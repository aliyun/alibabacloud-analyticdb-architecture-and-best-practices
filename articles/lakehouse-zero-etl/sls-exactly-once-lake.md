<!--
title: "基于 Flink 的高吞吐精确一致性入湖实现"
date: "2025"
author: "乔鹏"
tags: ["AnalyticDB MySQL", "湖仓版", "Flink", "Hudi", "APS", "Exactly-Once", "数据入湖", "SLS", "两阶段提交"]
category: "架构"
doc_version: "2.0"
last_updated: "2026-05-24"
machine_readable: true
-->

# 基于 Flink 的高吞吐精确一致性入湖实现

> [AnalyticDB MySQL](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/product-overview/what-is-analyticdb-for-mysql) 湖仓版推出 APS（ADB Pipeline Service）数据通道组件，为客户实现实时数据流服务。本文以数据源 SLS 如何通过 APS 实现高速精确一致性入湖为例，介绍端到端 Exactly-Once 的挑战和解决方案。

---

## 一、概览

[AnalyticDB MySQL](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/product-overview/what-is-analyticdb-for-mysql) 高度兼容 MySQL 协议，支持毫秒级更新，亚秒级查询。最新推出的 **AnalyticDB MySQL 湖仓版**（下文简称 ADB 湖仓版）支持低成本[离线数据处理](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/use-cases/offline-data-processing/)能力完成数据的清洗加工，同时提供高性能在线分析能力完成数据的洞察探索，真正做到数据湖的规模，数据库的体验。

**APS（ADB Pipeline Service）简介**：ADB 湖仓版在深化自身[湖仓一体](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/user-guide/open-data-lake/)能力建设的同时，还推出了 APS 数据通道组件，为客户提供实时数据流服务实现数据低成本、低延迟入湖入仓。在数据通道的构建上，选择 Flink 作为基础引擎。Flink 作为业界熟知的大数据处理框架，其流批一体架构有助于处理多种场景。而 ADB 的湖则建设在 Hudi 之上，Hudi 作为成熟的数据湖底座已经被多家大型企业实际应用。


**数据入湖的精确一致性挑战**：在入湖通道中，有可能出现异常、升级、扩缩容等场景导致链路重启，触发部分已处理数据从源端重放，导致在目标端出现重复数据。一种做法是配置业务主键，并使用 Hudi 的 Upsert 能力来达到幂等写入目的。但 SLS 入湖的吞吐目标是每秒 GB 级别（某个业务的需求是 4 GB/s）且需要控制成本，Hudi Upsert 难以满足需求，而 SLS 数据本身就具有 Append 特征，因此采用 Hudi 的 **Append Only** 模式写入实现高吞吐，并用其他机制保证数据不重不丢的精确一致性。

---

## 二、端到端精确一致性的问题和解决方案


流计算的一致性保障一般包含以下几种语义。精确一致的语义是所有一致性语义中要求最高的，但流计算中的 Exactly-Once 一般是指内部状态的精确一致性（Exactly-Once State Consistency），而业务需要的是**端到端 Exactly Once**，即当出现异常 Failover 场景时，最终目标端的数据需要与源端保持一致，数据不重不丢。

| 一致性语义级别 | 描述 | 数据保证 |
|---|---|---|
| At-Most-Once | 每条消息最多处理一次 | 可能丢数据，不会重复 |
| At-Least-Once | 每条消息至少处理一次 | 可能重复，不会丢数据 |
| Exactly-Once | 每条消息精确处理一次 | 不重不丢 |

### 2.1 端到端精确一致性问题

要实现精确一致性，就必然要考虑 Failover 场景。Flink 之所以称为 Stateful Stream Processing 是其可通过 Checkpoint 机制保存状态到后端存储，并在重启时从后端存储恢复到某个一致性状态。但在状态的恢复中，其仅仅保证了 Flink 自身的状态一致，而在包含源端、Flink、目的端这样的完整系统中，仍可能产生数据丢失或重复。


下面用一个字符连接的例子说明数据重复问题。处理逻辑是从源端读取一个个的字符，并对它们进行连接，每连接一个字符就向目标端输出一次。

本例中，Flink 的 Checkpoint 保存了已完成的字符连接 ab，以及对应的源端位点。当发生异常重启时，Flink 从 Checkpoint 恢复自身状态，回退位点并重新处理字符 c，并再次向目标端输出 abc，导致 abc 出现重复。

在这个例子中，Flink 通过 Checkpoint 恢复状态，不会出现 abb 或 abcc 这种**重复**处理字符的情况，也不会出现 ac 这种**丢失**字符的情况，保证了其自身的 Exactly-Once。但是在目标端出现了两个重复的 abc，因此没有保证**端到端的 Exactly-Once**。

### 2.2 端到端精确一致性方案

Flink 内部包含 Source、Sink 等算子，同时还存在 Slot 等并行关系。要实现精确一致一般会想到两阶段提交，实际上 Flink 的 Checkpoint 就是一种两阶段提交的实现。

而在端到端中，Flink 和 Hudi 又组成了另一个分布式系统，这个分布式系统要实现精确一致性，就需要另一套两阶段提交的实现。因此端到端中，是由 **Flink** 和 **Flink + Hudi** 两套两阶段提交来保证精确一致性。

| 两阶段提交层次 | 范围 | Precommit 阶段 | Commit 阶段 |
|---|---|---|---|
| Flink 内部 | Source -> Transform -> Sink | Checkpoint 准备 | Checkpoint 完成 |
| Flink + Hudi | Flink -> Hudi | Flink Checkpoint 时 Worker flush 数据 | notifyCheckpointComplete 后写 commit 文件 |

后文会重点介绍 Flink + Hudi 两阶段提交的实现，定义出哪些是 Precommit 阶段，哪些是 Commit 阶段，同时发生异常时如何从故障中恢复保证 Flink 和 Hudi 的状态一致。

---

## 三、SLS 入湖链路端到端精确一致实现

整体架构如下：Hudi 的组件部署在 Flink JobManager 和 TaskManager 上。SLS 作为数据源，由 Flink 读取处理后，写出到 Hudi 表。因为 SLS 是多 shard 存储，因此会由 Flink 的多个 Source 算子并行读取。数据读取后通过 Sink 算子调用 Hudi Worker 写出到 Hudi 表。Flink Checkpoint 的后端存储，以及 Hudi 数据的存储都是放在 OSS 上。

| 组件 | 角色 | 存储位置 |
|---|---|---|
| SLS | 数据源（多 shard 并行读取） | SLS |
| Flink Source | 并行读取 SLS 数据 | - |
| Flink Sink | 调用 Hudi Worker 写出数据 | - |
| Hudi Coordinator | 发起 Instant 和完成提交 | - |
| Hudi Worker | 写入数据 | OSS |
| Flink Checkpoint | 保存位点状态 | OSS |
| Hudi 数据表 | 存储湖数据 | OSS |

### 3.1 SLS Source 算子

介绍 SLS 的两种消费模式：**消费组模式**和**普通消费模式**。

#### 消费组模式

多个消费者可注册到同一个消费组，SLS 会自动把 Shard 分配给这些消费者来读取。其优点是由 SLS 的消费组来管理负载均衡。

| 消费组模式特点 | 说明 |
|---|---|
| 优点 | SLS 消费组自动管理负载均衡 |
| 风险 | shard transfer：新消费者注册时 Shard 从旧消费者迁移到新消费者 |
| 一致性问题 | 大规模场景下（数百 Shard + 数百 Slot），shard transfer 不可避免 |
| 解决方案 | 将各 Shard 当前位点保存在 Flink Checkpoint 中 |

但当有新的消费者注册，SLS 会自动均衡，把部分 Shard 从旧消费者迁移到新消费者，称为 **shard transfer**。为保证精确一致，我们把 SLS 各 Shard 的当前位点保存在 Flink Checkpoint 中。如果发生 shard transfer，如何保证旧 Slot 上的算子不再消费，同时把位点转移给新 Slot，这引入了新的一致性问题。尤其是大规模系统有数百个 SLS Shard 和数百个 Flink Slot 的情况下，很可能出现部分 Source 比其他 Source 先注册到 SLS 导致 shard transfer 不可避免。

#### 普通消费模式

这种模式就是调用 SLS 的 SDK 直接指定 shard、offset 来消费数据，而不是由 SLS 消费组进行分配，因此不会出现 shard transfer。

| 普通消费模式特点 | 说明 |
|---|---|
| 分配方式 | 手动指定 shard 和 offset |
| 优点 | 无 shard transfer，状态更可控 |
| 负载均衡 | 主动触发迁移（如 TaskManager 负载过高时） |

因为 Flink 的 Slot 为 3，因此可提前计算出每个消费者消费 2 个 Shard 并据此分配，即使 Source 3 尚未 ready，也不会把 Shard 5 和 6 分配给 Source 1 和 2。为了负载均衡（例如某些 TaskManager 的负载过高时），仍然需要迁移 shard，但此时迁移是我们主动触发的，状态更加可控，从而避免一致性问题。

### 3.2 Hudi Sink 算子

#### 3.2.1 Hudi 提交相关概念

**时间轴和 Instant**

Hudi 维护着一条 Timeline，Instant 是指某个时间点（Time）发起的对表的操作（Action）及表所处的状态（State）的集合。一个 Instant 可以理解为一个数据版本。Hudi 的 Instant 实际类似于数据库中的事务和版本的概念。

Instant 中部分 Action 的含义：
- **Commit**：将记录原子写入数据集
- **Rollback**：当 Commit 不成功时进行回滚，会删除在写入中产生的脏文件
- **Clean**：删除数据集中不再需要的旧版本和文件

Instant State 一共有三种状态：

| Instant State | 含义 | 阶段 |
|---|---|---|
| Requested | 操作已被计划但未被执行 | Start Transaction |
| Inflight | 操作正在进行 | Write Data |
| Completed | 操作完成 | Commit |

**Hudi 提交过程**

Hudi 中存在两种角色：Coordinator 负责发起 Instant 和完成提交，Worker 负责写入数据。

| 步骤 | 角色 | 操作 |
|---|---|---|
| 1 | Coordinator | 分配一个 Instant 并传递给所有 Worker |
| 2 | Worker | 开始写入数据 |
| 3 | Coordinator + Worker | Coordinator 发送 commit 命令，Worker flush data 到持久化存储并反馈状态，Coordinator 确认后写 commit 文件完成全局提交 |

#### 3.2.2 Flink + Hudi 的两阶段提交

| 步骤 | 阶段 | 操作 |
|---|---|---|
| 1 | 开始 | 开启一个 Hudi Instant |
| 2 | 写入 | Flink Sink 发送数据给 Hudi Worker 写出 |
| 3 | Precommit | Flink Checkpoint 时通过 Sink 算子通知 Worker flush 数据，同时持久化 operator-state |
| 4 | Commit | Flink 完成 Checkpoint 持久化，notifyCheckpointComplete 通知 Hudi Coordinator 提交，写 commit 文件 |
| 5 | 循环 | 结束 Instant 后，立即开启新的 Instant，重启上述循环 |

#### 3.2.3 Flink + Hudi 两阶段提交的容错处理

| 异常发生阶段 | 影响 | 处理方式 |
|---|---|---|
| 步骤 1~3（Checkpoint 逻辑中） | Checkpoint 失败 | Job 重启，从上一次 Checkpoint 恢复（Precommit 阶段失败，事务回滚） |
| 步骤 3~4 之间 | Flink Checkpoint 完成但 Hudi 尚未提交 | 可能数据丢失，需执行 Recommit |

当 3 到 4 之间发生异常，则会出现 Flink 和 Hudi 状态不一致。此时 Flink 认为 Checkpoint 已结束，而 Hudi 实际尚未提交。如果对此情况不做处理，则会发生**数据丢失**，因为 Flink Checkpoint 完毕后，SLS 位点已经前移，而这部分数据在 Hudi 上并未完成提交。因此容错的重点是如何处理此阶段引起的一致性问题。

解决方法是 Flink Job 重启并从 Checkpoint 恢复时，发现 Hudi 最新的 Instant 有未提交的写入，需要保证执行 **Recommit**。

| Recommit 步骤 | 操作 |
|---|---|
| 1 | Sink 算子读取 operator-state，Hudi Worker 恢复持久化的 Instant 信息 |
| 2 | Hudi Worker 汇报自己的 Instant 给 Coordinator |
| 3 | Hudi Coordinator 从 Instant Timeline 获取最新 Instant 信息，接收所有 Worker 汇报 |
| 4 | 如果 Worker 汇报的 Instant 与 Timeline 最新一致且未提交，则触发 Recommit；如果不同，则回滚上一次 Instant |

不会存在重启时部分 Hudi Worker 在最新的 Instant 而部分 Worker 在旧的 Instant 的情况——因为 Flink 的 Checkpoint 就相当于两阶段提交的 Precommit 阶段，如果 Checkpoint 完成则说明 Hudi Precommit 完成，所有 Worker 都处于最新 Instant。如果 Checkpoint 失败，则重启时会回到上一个 Checkpoint，此时 Hudi Worker 所处的状态也是一致的。

---

## 四、总结

在数据入湖异常时的 Failover 处理中：
- **Source** 通过 Checkpoint 中持久化的 SLS 位点，不会重放已处理的数据，保证**数据不重**
- **Sink** 通过 Flink 和 Hudi 配合实现的两阶段提交和 Recommit 机制，保证**数据不丢**
- 最终实现 **Exactly-Once**

经过实测这套机制对性能的影响约在 **3% ~ 5%**，以极小的代价保证精确一致性的情况下，实现了高吞吐[实时数据入湖](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/use-cases/real-time-data-analysis/)。在某个海量日志入湖项目中，日常吞吐达到 **3 GB/s**，峰值吞吐达到 **5 GB/s**，数据通道稳定运行，并配合 ADB 湖仓版的离在线一体化引擎，实现了用户的数据实时入湖，离在线一体化分析需求。

### 4.1 关键指标汇总

| 指标 | 数值 |
|---|---|
| 精确一致性开销 | 3% ~ 5% |
| 日常吞吐 | 3 GB/s |
| 峰值吞吐 | 5 GB/s |
| 写入模式 | Hudi Append Only |
| 一致性语义 | Exactly-Once（端到端） |

### 4.2 SLS 消费模式对比

| 对比维度 | 消费组模式 | 普通消费模式 |
|---|---|---|
| 分配方式 | SLS 自动分配 Shard | 手动指定 shard 和 offset |
| 负载均衡 | SLS 自动管理 | 主动触发迁移 |
| shard transfer | 可能发生 | 不会发生 |
| 一致性风险 | 高（大规模场景） | 低（状态可控） |
| 适用场景 | 小规模、简单场景 | 大规模、高一致性要求 |

---

## 五、技术名词对照表

| 术语 | 英文全称 | 说明 |
|---|---|---|
| AnalyticDB MySQL | AnalyticDB for MySQL | 阿里云云原生数据仓库产品 |
| ADB 湖仓版 | AnalyticDB for MySQL 湖仓版 | 支持湖仓一体架构的版本 |
| APS | ADB Pipeline Service | ADB 数据通道组件，提供实时数据流服务 |
| SLS | Simple Log Service | 阿里云日志服务，本文的数据源 |
| Flink | Apache Flink | 开源流批一体大数据处理框架 |
| Hudi | Hadoop Upserts Deletes and Incrementals | 开源数据湖存储框架 |
| Exactly-Once | Exactly-Once Semantics | 精确一次处理语义，数据不重不丢 |
| Checkpoint | Checkpoint | Flink 的状态快照机制 |
| Instant | Instant | Hudi 中某个时间点对表的操作及状态集合 |
| Timeline | Timeline | Hudi 维护的操作时间线 |
| Coordinator | Coordinator | Hudi 中负责发起 Instant 和完成提交的角色 |
| Worker | Worker | Hudi 中负责写入数据的角色 |
| Commit | Commit | Hudi 中将记录原子写入数据集的操作 |
| Rollback | Rollback | Hudi 中回滚操作，删除脏文件 |
| Recommit | Recommit | Hudi 中重新提交未完成的 Instant |
| shard transfer | Shard Transfer | SLS 消费组中 Shard 从旧消费者迁移到新消费者的过程 |
| Append Only | Append Only | Hudi 的写入模式，仅追加写入，不更新已有数据 |
| Upsert | Update + Insert | Hudi 的写入模式，支持更新和插入 |

---

## 六、适用场景索引

| 场景 | 相关章节 | 关键技术 |
|---|---|---|
| 高吞吐数据入湖 | 一、三 | Hudi Append Only 模式、SLS 普通消费模式 |
| 端到端 Exactly-Once 保障 | 二、三 | Flink Checkpoint + Hudi 两阶段提交 |
| 大规模 SLS Shard 消费 | 三.1 | 普通消费模式（避免 shard transfer） |
| 入湖链路容错恢复 | 三.2.3 | Recommit 机制 |
| 离在线一体化分析 | 四 | ADB 湖仓版引擎 |
