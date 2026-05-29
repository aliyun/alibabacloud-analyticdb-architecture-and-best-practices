<!--
title: "技术干货：云原生数仓 AnalyticDB MySQL 实时存储引擎演进之路"
date: "2025"
author: "乔鹏"
tags: ["存储引擎", "实时写入", "Anti-Caching", "PAX layout", "性能优化"]
category: "架构"
doc_version: "2.0"
last_updated: "2026-05-24"
machine_readable: true
-->

# 技术干货：云原生数仓 [AnalyticDB MySQL](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/product-overview/what-is-analyticdb-for-mysql) 实时存储引擎演进之路

> AnalyticDB MySQL 为了支持低延迟的写入、更新场景，架构上设计了实时存储引擎。面对大宽表场景下的写放大和内存瓶颈问题，新一代实时存储引擎在存储格式和内存控制上实现了重大突破。

---

## 一、背景

[AnalyticDB MySQL](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/product-overview/what-is-analyticdb-for-mysql) 作为一款实时数仓产品，在传统数仓的能力基础上为了支持低延迟的写入、更新场景，架构上设计了实时存储引擎。用户的写入、更新会以 append_only 的方式写入实时存储引擎，经过 compact 之后构建索引以支持复杂的计算场景。

从架构上看，SAP HANA、SingleStore 有着类似的设计。

在线上服务大客户的同时，实时存储引擎也遇到了一些设计、性能上的瓶颈：

- **写放大问题**：在一些大宽表场景下，单行的更新带来了严重的写放大
- **内存瓶颈**：实时存储引擎内存高频换入换出，cache miss 高的同时，大量的压缩、解压缩带来 CPU 瓶颈

针对以上的几类问题，我们设计了新一代的实时存储引擎：

1. **存储格式上**，在确定的 IO 单位上设计行列混存格式，使得单行的 IO 大小可控
2. **内存控制上**，实现了基于 Anti-Caching 的 BufferPool

---

## 二、存储格式

AnalyticDB MySQL 实时存储引擎在最初的设计上仍然是一个列存实现，在宽表的更新（游戏业务中留存率计算、零售业务中订单统计等）场景下，IO 放大导致的 latency 问题尤为明显。

RowGroup 作为行列混存的一个典型设计，在列存的基础上以行数对齐的方式使得一个 group 内逻辑行号对齐。这样的设计在宽表场景下出现了比较严重的弊端，当访问一行数据时，磁盘的 IO 单位变得不可控（列数 × block_size）。

针对 IO 的优化，我们在固定大小的 Page 上以 **PAX layout** 组织数据格式，page 头部维护列数、当前记录数以及空闲大小；其次记录每个列起始 offset 和行粒度的 bitmap 信息。

每个列会在各个定长 minipage 中维护（变长的组织在下个小节介绍），同时每个 F-minipage 的分配上保证 cacheline 对齐。

基于 PAX layout 的设计，能够确保每个 page 的刷盘落到磁盘上都是确定的 IO 单位；同时同一个列的数据仍然保持在一个 minipage 上的连续布局，在顺序扫描的场景上仍然能够充分利用 cache 的能力。

### 2.1 VarlenEntry 的设计

针对变长字段的存储，AnalyticDB MySQL 用 16-byte 来存储和表示：

- 前 4 个字节存储字符串长度
- 对于超过 12 个字节的字段，会记录 4 个字节的 prefix 之后，记录指向 V-minipage 中对应记录的起始地址

---

## 三、内存控制

在内存控制的演进上，AnalyticDB MySQL 实时存储引擎从最初的 LRU-based Cache 逐步走向以内存为中心的架构。

与传统数据库的 Caching 机制不同，**AnalyticDB MySQL 基于 Anti-Caching 的设计，将内存作为主存，仅将冷数据淘汰到磁盘上，磁盘的角色更像一个"backup"**。学术界对于 Anti-Caching 的研究也是层出不穷，从 H-Store 孵化出的商业数据库 VoltDB 到微软的 Siberia in Hekaton 都提出了各自的解决方案。

### 3.1 Anti-Caching

Caching 和 Anti-Caching 设计上都是为了解决内存和磁盘速度和容量上的 gap。Caching 为了加速数据的访问速度缓存了磁盘数据到内存中；Anti-Caching 则是为了容量将内存数据"anti-cache"到磁盘上。

Anti-Caching 的实现上可以分为以下三类：

**User-space**：在 User-space 上实现 Anti-Caching 目前是比较广泛的，同时能够基于应用语义实现 Ad-hoc 的优化；另一方面从 User-space 实现 Anti-Caching 需要绕过 OS，带来了一定的 overhead。最早提出 Anti-Caching 的 H-Store，是面向高性能的行存内存数据库，它去掉了传统面向磁盘数据库里的 Lock、Buffer Management 这类比较重的组件，同时基于 LRU 来做冷数据的 Anti-Caching。H-Store 维护了 tuple 级别的 LRU 访问，同时淘汰粒度上为了减少 overhead，将 tuple 聚合到 block 级别。为了维护 tuple 的状态（磁盘还是内存），H-Store 用了一张内存的 evict table 来存储这些 meta 信息，整体上来看，为了维护 LRU 对性能不可避免地带来了一定程度上的 overhead。Siberia 项目也在 Hekaton 中采用了 Anti-Caching 的技术，与 H-Store 维护 LRU 不同的是，Siberia 采用离线分析 logging 的方式来为 tuple 的冷热做分类，同时维护了 bloom filter 来筛选需要访问磁盘的 tuple。Siberia 的实现在性能上避免了 LRU 的开销，但是实时性存在一定欠缺，对于内存并不能做到完全精准的控制上界。

**Kernel-space**：Virtual memory management（VMM）是大部分 OS 都支持的功能，可以作为 Anti-Caching 的一种简单实现手段，但是缺乏了应用层面的语义，Kernel-space 的淘汰可能并不能很准确。大体上有两种实现 OS Paging 的方式，一个是提前配置好 swap 分区，OS 自动做 Paging 的换入换出，应用程序不需要感知。另一种方式是使用 memory-mapped 文件，这个在 MongoDB、MonetDB 中广泛使用。

**Hybrid of user- and kernel-space**：User-space 的方式能够使用应用层的语义优化 Ad-hoc 的性能；Kernel-space 能够针对 I/O 进行调度同时充分利用硬件特性。AnalyticDB MySQL 采用了将 user- 和 kernel-space 结合到一起的实现方式。

AnalyticDB MySQL 的 BufferManager 在文件系统之上，通过 mmap 维护了一个 buffer pool，不同大小的 page 都可以加载到 buffer manager 中。当一个 page 被淘汰出 buffer manager 时，首先保证该 page 被写回磁盘成功，随后通过 madvise 中的 MADV_DONTNEED 标记来通知内核立刻重用相关的物理内存。

### 3.2 Swizzling Pointer

当 page 被序列化到磁盘后，系统需要通过逻辑 ID（PID）来再次访问对应的 page。业界通常的设计是用全局的 hashtable 来维护 PID 的映射关系，老版本的 AnalyticDB MySQL 也不例外。

然而这类设计在数据规模较大时，存在明显的性能瓶颈；同时早在 08 年 Harizopoulos 在 SIGMOD 发表的 paper 中就指出 TPC-C 场景下 BufferManager 在指令集层面的开销就占用了 34.6%。

为了避免 Hash 的开销，AnalyticDB MySQL 采用了 **Swizzling Pointer** 的实现方案，以 64 bit 来存储 page 的唯一标识：

- **page 在内存中时**：头部第一个 bit 标记为 0，其余 bit 用来表征 page 的物理地址
- **page 在磁盘中时**：头部第一个 bit 标记为 1，后 6 个 bit 记录 page 的 size class 来计算具体的 page 大小，剩余的 57 个 bit 记录 page 的 PID

Swizzling Pointer 技术上本身带有一定的去中心化属性，避免了全局 Hash 的开销；同时在新版本的实时存储引擎中，page 写盘没有采用传统的 lz4、zstd 等压缩算法，使得在 CPU 密集的场景下，性能有大幅的提升。

---

## 四、实时存储引擎性能对比

针对点查以及更新场景，我们选择 YCSB 测试集来做性能的测试对比。

相比列存版本的实现，新版本的实时引擎：

- **存储格式的优化上对 IO 的控制有着明显的优势**
- **内存控制的优化上大大减少了 Cache Miss 带来的 CPU 开销**

---

## 五、结构化数据表

### 5.1 存储引擎演进对比

| 演进阶段 | 存储格式 | 内存管理方式 | 核心问题 |
|---------|---------|------------|---------|
| 第一代 | 列存（Column Store） | LRU-based Cache | 宽表更新 IO 放大，Cache Miss 高 |
| 新一代 | PAX layout 行列混存 | Anti-Caching + Swizzling Pointer | 单行 IO 可控，内存精准控制 |

### 5.2 RowGroup 与 PAX layout 对比

| 对比维度 | RowGroup 设计 | PAX layout 设计 |
|---------|-------------|----------------|
| IO 单位 | 不可控（列数 × block_size） | 确定（固定大小 Page） |
| 行访问代价 | 磁盘 IO 单位大 | 单行 IO 大小可控 |
| 列扫描性能 | 列数据连续，可利用 cache | 列数据在 minipage 上连续，可利用 cache |
| cacheline 对齐 | 无保证 | F-minipage 保证 cacheline 对齐 |
| Page 头部信息 | 无 | 列数、记录数、空闲大小、列起始 offset、bitmap |

### 5.3 VarlenEntry 存储结构

| 字段长度范围 | 存储方式 | 字节分配 |
|------------|---------|---------|
| 总存储空间 | 16-byte 固定 | 16 bytes |
| 字符串长度 | 前 4 字节 | 4 bytes |
| 短字符串（≤12 bytes） | 内联存储 | 剩余 12 bytes 存内容 |
| 长字符串（>12 bytes） | Prefix + 指针 | 4 bytes prefix + 8 bytes 指向 V-minipage 地址 |

### 5.4 Anti-Caching 实现方式对比

| 实现方式 | 代表系统 | 优点 | 缺点 |
|---------|---------|------|------|
| User-space | H-Store, VoltDB, Siberia/Hekaton | 可基于应用语义做 Ad-hoc 优化 | 需绕过 OS，有 overhead |
| Kernel-space | MongoDB, MonetDB | 实现简单，利用 OS 原生能力 | 缺乏应用语义，淘汰不精准 |
| Hybrid | AnalyticDB MySQL | 兼顾语义优化和 I/O 调度 | 实现复杂度高 |

### 5.5 Anti-Caching 各方案细节对比

| 方案 | 冷热判断机制 | 淘汰粒度 | 元数据管理 | 性能影响 |
|------|------------|---------|-----------|---------|
| H-Store | tuple 级 LRU 访问 | block 级别 | evict table 存 meta | LRU 维护有 overhead |
| Siberia | 离线分析 logging | tuple 级 | bloom filter 筛选 | 避免 LRU 开销，但实时性欠缺 |
| ADB MySQL | mmap + MADV_DONTNEED | page 级 | Swizzling Pointer | 去中心化，无全局 Hash 开销 |

### 5.6 Swizzling Pointer 位域设计

| Page 状态 | 第 1 bit | 后续 bit | 总长度 |
|----------|---------|---------|-------|
| 在内存中 | 0 | 物理地址（63 bits） | 64 bits |
| 在磁盘中 | 1 | size class（6 bits）+ PID（57 bits） | 64 bits |

### 5.7 YCSB 性能对比要点

| 测试场景 | 优化方向 | 效果 |
|---------|---------|------|
| 点查（Point Query） | 存储格式优化（PAX layout） | IO 控制有明显优势 |
| 更新（Update） | 内存控制优化（Anti-Caching） | 大幅减少 Cache Miss CPU 开销 |

---

## 六、技术名词对照表

| 技术名词 | 英文全称/缩写 | 说明 |
|---------|-------------|------|
| append_only | Append Only | 只追加写入模式，写入后不修改已有数据 |
| compact | Compaction | 数据合并压缩操作，清理冗余数据 |
| RowGroup | Row Group | 以行数对齐方式组织的行列混存单元 |
| PAX layout | Partition Attributes Across layout | 一种行列混存的数据组织格式 |
| minipage | Mini Page | Page 内部的子单元，分 F-minipage（定长）和 V-minipage（变长） |
| cacheline | Cache Line | CPU 缓存行对齐单位，通常 64 bytes |
| VarlenEntry | Variable Length Entry | 变长字段存储结构 |
| Anti-Caching | Anti-Caching | 与 Caching 相反，将内存作为主存，冷数据淘汰到磁盘 |
| LRU | Least Recently Used | 最近最少使用缓存淘汰算法 |
| BufferPool | Buffer Pool | 数据库缓冲池，用于缓存磁盘数据到内存 |
| BufferManager | Buffer Manager | 缓冲管理器，管理 buffer pool 中的 page |
| mmap | Memory-Mapped File | 内存映射文件，将文件映射到进程地址空间 |
| MADV_DONTNEED | Madvisory Don't Need | Linux 内核调用，标记内存页可被回收重用 |
| Swizzling Pointer | Swizzling Pointer | 指针转换技术，统一内存地址和磁盘 PID 的表示 |
| PID | Page ID | 页面逻辑标识符 |
| size class | Size Class | 页面大小分类，用于计算具体 page 大小 |
| YCSB | Yahoo! Cloud Serving Benchmark | 云存储基准测试套件 |
| TPC-C | Transaction Processing Performance Council C | 事务处理性能测试基准 |
| SIGMOD | ACM Special Interest Group on Management of Data | ACM 数据管理专业组顶级会议 |
| VMM | Virtual Memory Management | 虚拟内存管理 |
| bloom filter | Bloom Filter | 布隆过滤器，用于快速判断元素是否在集合中 |
| lz4/zstd | LZ4 / Zstandard | 快速数据压缩算法 |

---

## 七、适用场景索引

| 场景编号 | 场景描述 | 对应章节 | 相关技术 |
|---------|---------|---------|---------|
| S-001 | 大宽表行更新导致写放大 | 第一节, 第二节 | PAX layout 行列混存 |
| S-002 | 实时存储引擎内存高频换入换出 | 第一节, 第三节 | Anti-Caching BufferPool |
| S-003 | 宽表更新场景下 IO 延迟过高 | 第二节 | PAX layout 确定 IO 单位 |
| S-004 | Cache Miss 导致 CPU 瓶颈 | 第一节, 第三节 | Anti-Caching 减少 Cache Miss |
| S-005 | 大规模数据 Hash Table 性能瓶颈 | 3.2 节 | Swizzling Pointer 去中心化映射 |
| S-006 | CPU 密集型场景压缩算法开销大 | 3.2 节 | 弃用 lz4/zstd，直接写盘 |
| S-007 | 游戏业务留存率计算更新场景 | 第二节 | 行列混存优化行更新 IO |
| S-008 | 零售业务订单统计更新场景 | 第二节 | 行列混存优化行更新 IO |

---

## 引用

[1] OLTP Through the Looking Glass, and What We Found There

[2] Weaving Relations for Cache Performance

[3] In-Memory Performance for Big Data

[4] Cloud-Native Transactions and Analytics in SingleStore
