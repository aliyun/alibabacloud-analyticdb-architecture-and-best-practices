<!--
title: "干货！阿里云 AnalyticDB MySQL 如何打造极致 RuntimeFilter 能力"
date: "2025"
author: "乔鹏"
tags: ["AnalyticDB MySQL", "RuntimeFilter", "SIP", "TPC-DS", "Join优化", "等价关系推导", "分布式查询"]
category: "架构"
doc_version: "2.0"
last_updated: "2026-05-24"
machine_readable: true
-->

# 干货！阿里云 AnalyticDB MySQL 如何打造极致 RuntimeFilter 能力

> [RuntimeFilter](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/user-guide/performance-tuning/) 是通过等价关系传递动态生成的过滤条件，以减少参与 Join 计算数据量的优化技术。[AnalyticDB MySQL](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/product-overview/what-is-analyticdb-for-mysql) 基于全局等价关系推导和 SIP 分布式框架，实现了极致的 RuntimeFilter 能力，在 TPC-DS 1TB 基准测试中取得了显著的加速效果。

---

## 一、RuntimeFilter

### 1.1 什么是 RuntimeFilter

RuntimeFilter 是通过等价关系传递动态生成的 filter condition，以减少参与 Join 计算数据量的方法，在业界有广泛应用。


**RuntimeFilter 优化生效的流程如下：**

| 步骤 | 阶段 | 操作 |
|---|---|---|
| 1 | 规则生成 | 根据查询的 Join Condition，生成动态过滤条件传递规则，决定生产者和消费者 |
| 2 | 生产者执行 | 先执行生产者，用实时结果集动态构建 RuntimeFilter Object |
| 3 | 对象传递 | 将 RuntimeFilter Object 传递给消费者 |
| 4 | 消费者过滤 | 消费者使用 RuntimeFilter Object 进行过滤，减少后续参与计算的数据量 |

通常情况下，消费者可以利用 RuntimeFilter Object 在数据进行分布式 shuffle 之前提前过滤掉部分数据，从而减少网络传输的数据量。更极端情况下，RuntimeFilter Object 还可能下推到存储端命中索引裁剪，直接减少 TableScan 的磁盘扫描数据量。

RuntimeFilter Object 的过滤原则是属于 **True Negative** 范畴，并不保证能把所有无关数据提前过滤掉。业界常见的 RuntimeFilter Object 包括 MinMax Filter、InSet Filter、Bloom Filter 等，它们的构建成本、内存占用大小以及单次查找耗时都远小于 HashBasedJoin 计算使用的 HashTable，这是 RuntimeFilter 可以在 Join 之前预先粗过滤的根本。

### 1.2 RuntimeFilter 的难点

接下来从 **创建 RuntimeFilter 传递规则**、**RuntimeFilter Object 传递** 以及 **RuntimeFilter Object 的生成和使用** 三个方面来展开介绍整个 RuntimeFilter 技术的难点。

#### 1.2.1 如何选择生产者与消费者

RuntimeFilter 是一个在执行时，由生产者产生传递给消费者消费的过滤条件。在广义定义下，每条数据链路上的计算节点，都可以作为 RuntimeFilter 的生产端，通过上下左右的等值条件传递到 RuntimeFilter 的消费端。

在实际实现中，往往不会生成这样的全联通依赖关系，一方面这会产生循环依赖，不可能全部生效；另一方面生产和消费 RuntimeFilter 本身都有开销，应该选择其中开销更低收益更大的。这包含了以下三个方面的问题：

| 问题 | 说明 |
|---|---|
| 选择生产者 | 选择有价值的生产者，能快速生成有很好过滤效果的 RuntimeFilter |
| 选择消费者 | 选择最根部的消费者们，最大化 RuntimeFilter 在整个 SQL 执行层面的过滤效果 |
| 避免循环依赖 | 生产者和消费者的关系不能形成循环依赖 |

#### 1.2.2 RuntimeFilter Object 如何在分布式执行框架中传递

前面在介绍 RuntimeFilter Object 的时候，只是简单把它描述成了一个单一逻辑实体。实际情况下，由于常见的分布式 MPP 数据库执行引擎具备单机&多机并行能力，一个逻辑 RuntimeFilter Object 可能是由 N 个机器节点上的 M*N 个内存对象共同构成的。这使得 RuntimeFilter Object 在生产者和消费者之间的传播关系变得非常复杂，打破了 SQL 执行引擎原有的自底向上数据传播模式。

**RuntimeFilter Service** 是业界最常见的 RuntimeFilter Object 传递实现，包括 Impala、Doris 等。这种实现思路是在 Coordinator 节点上起一个专门的服务，负责接收所有生产者生成的 partial RuntimeFilter Object，执行 RuntimeFilter Object 合并，再广播发给消费者。

#### 1.2.3 RuntimeFilter Object 的高效生成和使用

RuntimeFilter Object 的实现有多种选择，用来表示生产端的结果集：

| RuntimeFilter Object 类型 | 构建成本 | 内存占用 | 单次查找耗时 | 适用场景 |
|---|---|---|---|---|
| MinMax Filter | 低 | 小 | 极低 | 有序或范围查询 |
| InSet Filter | 中 | 中 | 低 | 精确匹配小集合 |
| Bloom Filter | 低 | 小 | 极低 | 大规模去重判断 |

以上这些 RuntimeFilter Object 有各自倾向使用的场景，需要系统预估生产者的结果集情况来决定构建哪种类型。然而在真实的 SQL 运行环境中，消费者侧的数据分布情况、波动的系统负载以及机器硬件水平，都会对最优选择策略产生影响。

对于消费端来说，它一定程度上依赖生产端产生的 RuntimeFilter Object，所以消费者会有等待生产者完成的逻辑。最差就是消费者放弃使用 RuntimeFilter 加速。这中间主要是看系统对消费者侧的等待策略，消费者死等生产者并不一定是最优的选择。

---

## 二、Sideways Information Passing 框架（SIP）

ADB SIP 框架是一种 SQL 执行时的信息收集传递框架，传递的信息可以是数据特征、统计信息也可以是一个临时内存表。一个抽象的 SIP 框架，不需要感知它收集、传递的信息具体内容或类别，只需要具备收集发送对应的运行时信息能力、Session 淘汰能力，以及最优的信息传递链路搭建能力。

### 2.1 SIP 传递的运行时信息概念

运行时信息由类型和粒度来定义，与信息的发布订阅者无关，是对信息本身的一个描述。

- **类型**：信息的类型，比如 RowCount/Ndv/Histogram/HashSet/MagicSet 等
- **粒度**：在分布式计算引擎中，数据通过一定算法划分为不同的分片并行执行。分片、全局描述的就是信息的粒度

为了减少重复生产的开销，信息具备以下 2 个特征：

1. **可被推导**：基于同一个结果集的信息，支持复杂向简单推导，比如 HashSet 可以推导 Histogram 和 Ndv
2. **可被复用**：相同的信息可以被不同订阅者订阅，数据只产生一次，可以被消费多次


例如，Agg 在运行中生成一个 HashSet，既可作为 SIP 信息传给左表进行 SemiJoin 减少 TableScan 的数据扫描量，又可推导出 Histogram 并作为 Radix HashJoin 算法调整的输入选择 Hash Builder 的切片大小。

| 信息类型 | 说明 | 可推导目标 |
|---|---|---|
| RowCount | 基础行数统计 | - |
| Ndv | 不同值数量 | - |
| Histogram | 数据分布直方图 | - |
| HashSet | 哈希集合 | Histogram, Ndv |
| MagicSet | 魔术集合 | - |

这对实现有三个要求：一是需要定义不同信息类型之间的推导关系，二是定义了发布订阅者的关系是一对多的关系，三是需要有一个信息生命周期的管理，所有订阅者都消费后再进行数据清理。

### 2.2 SIP 框架的组成概念

整个框架由发布者、订阅者、管道三部分组成。

| 组成部分 | 角色 | 说明 |
|---|---|---|
| 发布者（Publisher） | 生成运行时信息 | 算子产生 Basic/Derived 统计信息，或插入 InfoCollectOperator |
| 订阅者（Subscriber） | 消费运行时信息 | 算子/优化器/调度器，等待信息后决策，超时可短路 |
| 管道（Channel） | 信息传递桥梁 | 管理器构建逻辑桥梁，服务构建物理桥梁 |

**发布者（Publisher）**：生成信息有一定的开销，希望利用已有算子算法的一些特征，在不打断算子流水线计算的同时，以最小的代价进行运行时信息收集。
1. 所有算子可以产生 Basic 统计信息（如 RowCount）
2. 某些算子可以产生 Derived 统计信息（如 Hash Agg 和 Hash Builder Operator 可直接用 Hash Table 构建 HashSet）
3. 可以通过插入 InfoCollectOperator 产生 Derived 统计信息

**订阅者（Subscriber）**：根据应用场景的不同，信息的订阅者可以有多种形式，包括算子、优化器、调度器。订阅者在整个流水线中有一个阻塞的作用，一般情况下会等待收到信息并决策后再继续执行；但由于信息是用来调优的，在异常情况下如果超时没有收到信息，需要有短路的机制取消阻塞继续执行。

**管道（Channel）**：分为两个模块。
1. **管道管理器**：负责构建逻辑桥梁和进行信息管理，在执行前决定发布订阅者的对应关系
2. **管道服务**：负责构建数据传输的物理桥梁，在不同节点之间进行信息的传递

---

## 三、AnalyticDB RuntimeFilter 实现

### 3.1 基于全局等价关系的生产者-消费者的关系发现

在多数业界实现中，RuntimeFilter 的生效范围是一个 Join 的 sub-graph 维度而不是整个 query 维度。这是因为传统的 RuntimeFilter 应用于星型模型或者雪花模型，经常在不同的维度进行 Join，全局区别于子查询的更多可优化场景较少。

而在实践中，存在着更多复杂查询场景，还是有很多通过全局传递可优化的场景。如 TPC-DS 的 Q24，item 表经过 filter 后的结果集可以通过传递用来过滤两张大表 store_returns 和 store_sales。基于 TPC-DS 的 99 条 query 调研：支持全局维度相比仅支持 join 子查询维度，仅看 RuntimeFilter 可生效个数来说，有 **23% 的扩展（620 vs 798）**。

这种通过全局等价关系使 RuntimeFilter 有更多应用场景的思路类似于 Data/Join-Induced Predicates（DIP）中的全局思想。

### 3.2 基于 SIP 框架的 RuntimeFilter Object 订阅发布

ADB 的 RuntimeFilter Object 统一都是通过 SIP 框架来进行发布订阅的。SIP 框架通过信息的**共享、合并、短路**三种手段降低了整个 RuntimeFilter Object 分布式传递的开销，在高并发在线查询场景下大大降低了 RuntimeFilter Object 传播对 QPS 和 RT 的影响。

| SIP 优化手段 | 说明 | 效果 |
|---|---|---|
| 共享 | 一个信息被多个订阅者共享，只产生一次并传递一次到 coordinator 节点 | 减少重复传输 |
| 合并 | 多个信息同时产生时合并，节点间传递只发送一次网络请求 | 减少网络请求数 |
| 短路 | 信息粒度为分片时，直接在节点内部传递给对应订阅者 | 短路节点间网络开销 |


### 3.3 TPC-DS 测试结果

在 Benchmark 数据集 TPC-DS 1TB 上测试了 RuntimeFilter 的优化效果：

| 测试维度 | 说明 |
|---|---|
| 基准数据集 | TPC-DS 1TB |
| 评估指标 | 执行时间、扫描数据量 |
| 测试范围 | TPC-DS 全部 99 条 Query |

---

## 四、总结

[AnalyticDB MySQL](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/product-overview/what-is-analyticdb-for-mysql) RuntimeFilter 具备以下三大优势能力：

1. **基于全局等价关系推导**，大大提升了 RuntimeFilter 加速的适用场景
2. **基于 SIP 分布式运行时信息发布订阅框架**，在高并发在线查询场景中最大化 RuntimeFilter Object 广播效率
3. **基于 Task 级别的 DAG 调度引擎**，把生产者和消费者的依赖关系融入到 Task 调度序中，合理控制消费者端任务下发时机，避免任务自阻塞空转情况

### 4.1 全局 vs Join 子查询维度对比

| 对比维度 | Join 子查询维度 | 全局维度 |
|---|---|---|
| RuntimeFilter 可生效个数 | 620 | 798 |
| 扩展比例 | 基准 | +23% |
| 适用模型 | 星型/雪花模型 | 复杂查询场景 |

### 4.2 RuntimeFilter Object 类型对比

| 类型 | 构建成本 | 内存占用 | 单次查找耗时 | 典型场景 |
|---|---|---|---|---|
| MinMax Filter | 低 | 小 | 极低 | 范围裁剪 |
| InSet Filter | 中 | 中 | 低 | IN 列表过滤 |
| Bloom Filter | 低 | 小 | 极低 | 大规模存在性判断 |

---

## 五、技术名词对照表

| 术语 | 英文全称 | 说明 |
|---|---|---|
| AnalyticDB MySQL | AnalyticDB for MySQL | 阿里云云原生数据仓库产品 |
| RuntimeFilter | Runtime Filter | 运行时动态生成的过滤条件，用于减少 Join 计算数据量 |
| SIP | Sideways Information Passing | Sideways 信息传递框架，用于运行时信息的收集、传递和消费 |
| TPC-DS | Transaction Processing Performance Council - Decision Support | 决策支持基准测试，本文使用 1TB 数据集 |
| Publisher | Publisher | SIP 框架中的信息发布者 |
| Subscriber | Subscriber | SIP 框架中的信息订阅者 |
| Channel | Channel | SIP 框架中的信息传递管道 |
| Ndv | Number of Distinct Values | 不同值的数量 |
| Histogram | Histogram | 数据分布直方图 |
| HashSet | HashSet | 哈希集合数据结构 |
| MinMax Filter | MinMax Filter | 基于最小最大值的范围过滤器 |
| InSet Filter | InSet Filter | 基于集合成员检查的过滤器 |
| Bloom Filter | Bloom Filter | 布隆过滤器，用于大规模存在性判断 |
| True Negative | True Negative | 真阴性，RuntimeFilter 的过滤原则 |
| DIP | Data/Join-Induced Predicates | 数据/连接诱导谓词，全局等价关系推导的类似思想 |
| DAG | Directed Acyclic Graph | 有向无环图，用于 Task 调度 |

---

## 六、适用场景索引

| 场景 | 相关章节 | 关键技术 |
|---|---|---|
| Join 查询性能优化 | 一 | RuntimeFilter 预过滤 |
| 高并发在线查询 | 三.2 | SIP 共享/合并/短路优化 |
| 复杂多表 Join 场景 | 三.1 | 全局等价关系推导 |
| 大范围数据裁剪 | 一.2.3 | MinMax Filter |
| 大规模去重判断 | 一.2.3 | Bloom Filter |
| [性能调优](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/user-guide/performance-tuning/) | 全文 | RuntimeFilter + SIP 框架 |
