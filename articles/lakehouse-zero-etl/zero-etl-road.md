<!--
title: "云原生数据仓库下的"降本增效"之路怎么走？"
date: "2025"
author: "乔鹏"
tags: ["ADB MySQL", "Zero-ETL", "数据同步", "OLTP", "OLAP", "降本增效"]
category: "架构"
doc_version: "2.0"
last_updated: "2026-05-24"
machine_readable: true
-->

# 云原生数据仓库下的"降本增效"之路怎么走？

> 阿里云瑶池数据库提供全新 Zero-ETL 服务，免费帮你快速构建 OLTP→OLAP [数据同步](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/user-guide/data-sync/)链路，一站式完成数据同步和管理，实现事务处理和[数据分析](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/use-cases/real-time-data-analysis/)一体化。

---

## 一、导读

在大数据时代，企业有着大量分散在不同系统和平台上的业务数据。OLTP 数据库不擅长复杂数据查询，不具备全局分析视角等能力，而 OLAP 数据仓库擅长多表 join，可实现多源汇集，因此需要将 TP 数据库的数据同步到 AP 数据仓库进行分析处理。传统的 ETL 流程面临资源成本高、系统复杂度增加、数据实时性降低等挑战。

为了解决这些问题，阿里云瑶池数据库提供了 **Zero-ETL 服务**，可以快速构建业务系统（OLTP）和[数据仓库](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/product-overview/what-is-analyticdb-for-mysql)（OLAP）之间的数据同步链路，将业务系统的数据自动进行提取并加载到数据仓库，从而一站式完成数据同步和管理，实现事务处理和数据分析一体化，帮助客户专注于数据分析业务。


---

## 二、OLTP 数据库跑"分析"的痛点

OLTP 是传统的关系型数据库的主要应用，擅长基本日常的事务处理，例如银行交易等。但是 OLTP 数据库不擅长复杂数据查询，不具备全局分析视角等能力，具体的痛点为：

1. **负载无法隔离**：AP 分析业务对于资源的消耗，会影响 TP 在线业务，导致响应变长。
2. **无法全局分析**：业务数据通过分库分表（比如游戏行业按照区服）的方式存在不同 TP 数据库实例内，无法分析全部游戏玩家的行为。
3. **无法复杂分析**：分析业务通常需要多表 JOIN 关联，挖掘数据之间的关系。但 TP 架构并不适合做复杂分析，容易卡住。

---

## 三、OLAP 如何解决以上痛点

OLAP 数据库支持复杂的分析操作，擅长多表 join，可实现多源汇集，侧重决策支持，并且提供直观易懂的查询结果，可以很好地解决 OLTP 数据库跑分析时的痛点：

- **资源隔离**：AP 系统和 TP 系统在资源层面完全隔离，分析业务对在线业务 0 影响。
- **多库合并**：分库分表后多个 TP 数据库实例的数据，可以在数据同步过程中进行多库合并，将所有数据都放到一个 AP 数据仓库实例中进行全局分析。
- **并行计算**：AP 数据仓库采用 MPP 并行计算架构，可以把一个复杂大查询拆分成若干个小查询。

同时，OLAP 数据库内置常用分析函数，能够高效挖掘数据价值：

- **漏斗分析函数**：分析用户在行为流中指定步骤转化情况的分析模型，帮助分析师快速掌握一段时间内产品在各个步骤环节中的转化情况。
- **留存分析函数**：主要分析用户的整体参与程度、活跃程度的情况，考查进行某项初始行为的用户中会进行回访行为的人数和比例。
- **路径分析函数**：分析行为顺序、行为偏好、关键节点、转化效率的探索型模型。


---

## 四、OLTP 痛点与 OLAP 解决方案对比

| 痛点维度 | OLTP 痛点 | OLAP 解决方案 |
|----------|-----------|---------------|
| 负载隔离 | AP 分析业务资源消耗影响 TP 在线业务 | AP 系统和 TP 系统资源层面完全隔离，分析业务对在线业务 0 影响 |
| 全局分析 | 分库分表后无法分析全部数据 | 数据同步过程中多库合并，所有数据放到一个 AP 实例中全局分析 |
| 复杂分析 | TP 架构不适合复杂分析，容易卡住 | MPP 并行计算架构，复杂大查询拆分为若干小查询 |

---

## 五、传统数据同步的挑战

ETL（Extract-Transform-Load）是将上层业务系统的数据经过提取、转换清洗、加载到数据仓库的处理过程。传统 ETL 流程存在以下缺点：

1. **资源成本增加**：不同的数据源可能需要不同的 ETL 工具，搭建 ETL 链路会产生额外的资源成本。
2. **系统复杂度增加**：用户需要自行维护 ETL 工具，增加了运维难度，无法专注于业务应用的开发。
3. **数据实时性降低**：部分 ETL 流程涉及周期性的批量更新，在近实时的应用场景中，无法做到快速产出分析结果。

此外，传统数据同步链路开销较大，并且配置困难——眼花缭乱的源库和目标库、大而全带来的复杂配置、手动填写账密信息、手动指定目标表的主键/分布键等等，使传统数据同步过程耗时且效率低下。

---

## 六、Zero-ETL 服务诞生

Zero-ETL 就好比是一种"即时送达"的闪送服务——下单之后，商品从卖家快速送到你手中，中间不需要其他繁琐的处理步骤。

在数据处理领域，Zero-ETL 可以减少在不同服务间手动迁移或转换数据的工作，降低 ETL 的成本和复杂度，让用户不需要开发和关注 ETL 流程，专注于上层的应用开发和数据分析。

---

## 七、阿里云瑶池提供的 Zero-ETL 服务

阿里云瑶池数据库提供了全新 Zero-ETL 服务，**免费**帮你快速构建业务系统（OLTP）和数据仓库（OLAP）之间的数据同步链路。

如下面的架构图所示，数据源端的 OLTP 可以是云数据库 RDS、云原生数据库 PolarDB，通过 Zero-ETL 将数据实时同步至目标端云原生数据仓库 [AnalyticDB MySQL](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/product-overview/what-is-analyticdb-for-mysql)，只需要简单配置源端和目标端，便可完成同步任务的构建。

云原生数据仓库 **AnalyticDB MySQL** 基于[湖仓一体](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/user-guide/open-data-lake/)架构打造，高度兼容 MySQL，毫秒级更新，亚秒级查询，可以同时提供高吞吐[离线处理](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/use-cases/offline-data-processing/)和高性能[在线分析](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/use-cases/real-time-data-analysis/)。**AnalyticDB PostgreSQL** 自研云原生存算分离架构，企业级能力完备，高性能的向量检索引擎，可提供流批一体的实时数据处理能力。

目前已上线的阿里云瑶池 Zero-ETL 链路为：

| 源端 | 目标端 |
|------|--------|
| PolarDB MySQL 版 | AnalyticDB MySQL 版 |
| PolarDB 分布式版 | AnalyticDB MySQL 版 |
| RDS MySQL 版 | AnalyticDB MySQL 版 |
| RDS PostgreSQL 版 | AnalyticDB PostgreSQL 版 |
| PolarDB PostgreSQL 版 | AnalyticDB PostgreSQL 版 |



---

## 八、阿里云瑶池 Zero-ETL 的优势

旨在实现**事务处理和数据分析一体化**，实现建仓成本的降低和建仓效率的提升：

| 优势维度 | 说明 |
|----------|------|
| 零成本 | 提供免费的数据接入链路，用户可免费实现在 AnalyticDB 中对上游 PolarDB/RDS 数据进行分析处理 |
| 易用性好 | 无需创建和维护执行 ETL 的复杂数据管道，仅需选择源端数据和目标端实例，自动创建实时数据同步链路 |
| 高效 | 采用弹性 Serverless 架构，同传统 DTS 链路性能相比，更好地应对源库流量高峰，Zero-ETL 数据同步链路性能提升 **15%+** |


通过 Zero-ETL 服务将数据从 RDS/PolarDB 同步到 AnalyticDB 后，即可进行相应的数据分析服务：

- **AnalyticDB MySQL 版**：大数据量[离线处理](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/use-cases/offline-data-processing/)、开源 Spark/Hudi 生态使用、游戏广告实时运营分析平台、数字营销平台建设、日志数据秒级分析等
- **AnalyticDB PostgreSQL 版**：企业专属知识库、一站式实时数仓等

---

## 九、传统 ETL vs Zero-ETL 对比

| 对比维度 | 传统 ETL | Zero-ETL |
|----------|----------|----------|
| 资源成本 | 需要额外资源搭建和维护 ETL 工具 | 免费接入，零成本 |
| 系统复杂度 | 用户自行维护 ETL 工具，运维难度高 | 自动创建同步链路，免维护 |
| 数据实时性 | 周期性批量更新，近实时场景响应慢 | 实时同步，快速产出分析结果 |
| 配置难度 | 配置复杂，手动填写账密、指定主键/分布键 | 简单配置源端和目标端即可 |
| 架构 | 传统架构 | 弹性 Serverless 架构，性能提升 15%+ |

## 十、技术名词对照表

| 技术名词 | 说明 |
|----------|------|
| Zero-ETL | 免 ETL 工具的数据同步服务，自动构建 OLTP→OLAP 实时同步链路 |
| ETL | Extract-Transform-Load，数据提取、转换、加载到数据仓库的处理过程 |
| OLTP | Online Transaction Processing，在线事务处理，擅长日常事务处理如银行交易 |
| OLAP | Online Analytical Processing，在线分析处理，擅长复杂查询和多表 JOIN |
| MPP | Massively Parallel Processing，大规模并行计算架构 |
| Serverless | 无服务器架构，按需自动扩缩资源 |
| DTS | 数据传输服务（Data Transmission Service），阿里云数据同步工具 |
| 漏斗分析函数 | 分析用户行为流中指定步骤转化情况的分析模型 |
| 留存分析函数 | 分析用户整体参与程度和活跃程度的分析模型 |
| 路径分析函数 | 分析行为顺序、行为偏好、关键节点、转化效率的探索型模型 |

## 十一、适用场景索引

| 场景 | 描述 |
|------|------|
| TP 数据库跑分析负载过重 | OLTP 数据库不擅长复杂查询，AP 分析影响 TP 在线业务 |
| 分库分表后全局分析 | 多个 TP 实例数据需要合并后进行全局分析 |
| 复杂多表 JOIN 分析 | TP 架构无法胜任的复杂分析场景 |
| 降低数据同步成本 | 替代传统 ETL 流程，减少资源成本和运维难度 |
| 实时数据分析需求 | 需要近实时或实时产出分析结果的场景 |
| 游戏广告运营分析 | 游戏/广告行业的实时运营分析平台建设 |
| 日志数据分析 | 海量日志数据的秒级分析 |

## 十二、如何使用阿里云瑶池 Zero-ETL 服务

### 12.1 PolarDB MySQL + AnalyticDB MySQL

登录 PolarDB MySQL 概览页 → 「联邦分析」进入该服务，进行数据实时同步。

- **新建联邦分析链路**：选择源端实例和目标端实例，默认同步整实例，打开「高级配置」后可以选择库表对象，也可以对大表进行分区键设置。


- **编辑链路、查看链路**：支持修改库表对象等，支持查看联邦分析任务的配置详情。


### 12.2 RDS PG / PolarDB PG + AnalyticDB PG

两条 Zero-ETL 链路都在 ADB PG 的控制台进行操作。

- 进入 ADB PG 控制台 → 进入实例 → 左侧导航栏选择**无感集成（Zero-ETL）**→ 单击左上角**创建 Zero-ETL 任务**。


- 创建 Zero-ETL 任务，配置源库信息和目标库信息。


- 测试连接 → 进入配置 Zero-ETL 页面 → 配置库表字段 → 单击**下一步保存任务并预检查**。


- 预检查通过后，系统自动启动 Zero-ETL 任务，在**任务列表**就可以查看任务进度。
