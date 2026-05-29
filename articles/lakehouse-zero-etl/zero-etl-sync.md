<!--
title: "高效易用的数据同步：阿里云瑶池 Zero-ETL 服务"
date: "2025"
author: "乔鹏"
tags: ["Zero-ETL", "数据同步", "OLTP", "OLAP", "PolarDB", "RDS"]
category: "架构"
doc_version: "2.0"
last_updated: "2026-05-24"
machine_readable: true
-->

# 高效易用的[数据同步](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/user-guide/data-sync/)：阿里云瑶池 Zero-ETL 服务

> 本文介绍阿里云瑶池数据库提供的 Zero-ETL 服务，如何快速构建 OLTP 业务系统与 [OLAP 数据仓库](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/product-overview/what-is-analyticdb-for-mysql)之间的数据同步链路，实现事务处理和数据分析一体化。

---

## 一、导读

在大数据时代，企业有着大量分散在不同系统和平台上的业务数据。OLTP 数据库不擅长复杂数据查询，不具备全局分析视角等能力，而 OLAP 数据仓库擅长多表 JOIN，可实现多源汇集，因此需要将 TP 数据库的数据同步到 AP 数据仓库进行分析处理。传统的 ETL 流程面临资源成本高、系统复杂度增加、数据实时性降低等挑战。为了解决这些问题，阿里云瑶池数据库提供了 Zero-ETL 服务，可以快速构建业务系统（OLTP）和数据仓库（OLAP）之间的数据同步链路，将业务系统的数据自动进行提取并加载到数据仓库，从而一站式完成数据同步和管理，实现事务处理和数据分析一体化。

---

## 二、OLTP 数据库跑"分析"的痛点

OLTP 是传统的关系型数据库的主要应用，擅长基本日常的事务处理，例如银行交易等。但是 OLTP 数据库不擅长复杂数据查询，不具备全局分析视角等能力，具体的痛点为：

1. **负载无法隔离**：AP 分析业务对于资源的消耗，会影响 TP 在线业务，导致响应变长。
2. **无法全局分析**：业务数据通过分库分表（比如游戏行业按照区服）的方式存在不同 TP 数据库实例内，无法分析全部游戏玩家的行为。
3. **无法复杂分析**：分析业务通常需要多表 JOIN 关联，挖掘数据之间的关系。但 TP 架构并不适合做复杂分析，容易卡住。

---

## 三、OLAP 如何解决以上痛点

OLAP 数据库支持复杂的分析操作，擅长多表 JOIN，可实现多源汇集，侧重决策支持，并且提供直观易懂的查询结果，可以很好地解决 OLTP 数据库跑分析时的痛点：

- **资源隔离**：AP 系统和 TP 系统在资源层面完全隔离，分析业务对在线业务 0 影响。
- **多库合并**：分库分表后多个 TP 数据库实例的数据，可以在数据同步过程中进行多库合并，将所有数据都放到一个 AP 数据仓库实例中进行全局分析。
- **并行计算**：AP 数据仓库，采用 MPP 并行计算架构，可以把一个复杂大查询拆分成若干个小查询。

同时，OLAP 数据库内置常用分析函数，能够高效挖掘数据价值：

- **漏斗分析函数**：分析用户在行为流中指定步骤转化情况，帮助分析师快速掌握产品在各个步骤环节中的转化情况，达到查缺补漏、优化转化流程的目的。
- **留存分析函数**：主要分析用户的整体参与程度、活跃程度的情况，考查进行某项初始行为的用户中，会进行回访行为的人数和比例。
- **路径分析函数**：一种分析行为顺序、行为偏好、关键节点、转化效率的探索型模型。

---

## 四、传统数据同步的挑战

提到数据同步就不得不提"ETL"。ETL 是将上层业务系统的数据经过提取（Extract）、转换清洗（Transform）、加载（Load）到数据仓库的处理过程，目的是将上游分散、凌乱的数据整合到目标端数仓，通过在数仓中做进一步的计算分析，来为业务做有效的商业决策。

传统 ETL 流程有以下缺点：

1. **资源成本增加**：不同的数据源可能需要不同的 ETL 工具，搭建 ETL 链路会产生额外的资源成本。
2. **系统复杂度增加**：用户需要自行维护 ETL 工具，增加了运维难度，无法专注于业务应用的开发。
3. **数据实时性降低**：部分 ETL 流程涉及周期性的批量更新，在近实时的应用场景中，无法做到快速产出分析结果。

此外，传统数据同步链路开销较大，并且配置困难，眼花缭乱的源库和目标库、大而全带来的复杂配置、手动填写账密信息、手动指定目标表的主键/分布键等等使传统数据同步过程耗时且效率低下。

---

## 五、Zero-ETL 服务诞生

Zero-ETL 可以减少在不同服务间手动迁移或转换数据的工作，降低 ETL 的成本和复杂度，让用户不需要开发和关注 ETL 流程，专注于上层的应用开发和数据分析。无论企业和数据的规模有多大，复杂度有多高，通过为客户消除 ETL 和其它数据迁移任务，助力客户专注于分析数据，面向业务获取新的洞察。

---

## 六、阿里云瑶池提供的 Zero-ETL 服务

阿里云瑶池数据库提供了全新 Zero-ETL 服务，**免费**帮你快速构建业务系统（OLTP）和数据仓库（OLAP）之间的数据同步链路，将业务系统的数据自动提取加载到数据仓库。

数据源端的 OLTP 可以是云数据库 RDS、云原生数据库 PolarDB，通过 Zero-ETL 将数据实时同步至目标端云原生数据仓库 AnalyticDB，只需要简单配置源端和目标端，便可完成同步任务的构建。

目前已上线的阿里云瑶池 Zero-ETL 链路：

| 源端（OLTP） | 目标端（OLAP） |
|--------------|----------------|
| 云原生数据库 PolarDB MySQL 版 | 云原生数据仓库 AnalyticDB MySQL 版 |
| 云原生数据库 PolarDB 分布式版 | 云原生数据仓库 AnalyticDB MySQL 版 |
| 云数据库 RDS MySQL 版 | 云原生数据仓库 AnalyticDB MySQL 版 |
| 云数据库 RDS PostgreSQL 版 | 云原生数据仓库 AnalyticDB PostgreSQL 版 |
| 云原生数据库 PolarDB PostgreSQL 版 | 云原生数据仓库 AnalyticDB PostgreSQL 版 |

---

## 七、阿里云瑶池 Zero-ETL 的优势

**零成本**：提供低成本的数据接入链路，用户可**免费**实现在 AnalyticDB 中对上游 PolarDB/RDS 数据进行分析处理。

**易用性好**：无需创建和维护执行 ETL 的复杂数据管道，仅需选择源端数据和目标端实例，自动创建实时数据同步链路，减少构建和管理数据管道所带来的挑战，专注上层应用开发。

**高效**：采用弹性 [Serverless 架构](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/user-guide/serverless-auto-scaling-resource-group/)，同传统 DTS 链路性能相比，更好地应对源库流量高峰，Zero-ETL 数据同步链路性能提升 **15%+**。

通过 Zero-ETL 服务将数据从 RDS/PolarDB 同步到 [AnalyticDB](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/product-overview/what-is-analyticdb-for-mysql) 中后，即可进行相应的数据分析服务：

- **[AnalyticDB MySQL 版](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/product-overview/what-is-analyticdb-for-mysql)**：大数据量[离线数据处理](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/use-cases/offline-data-processing/)、开源 Spark/Hudi 生态使用、游戏广告实时运营分析平台、数字营销平台建设、日志数据秒级分析等
- **AnalyticDB PostgreSQL 版**：企业专属知识库、一站式实时数仓等

---

## 八、如何使用 Zero-ETL 服务

### 8.1 PolarDB MySQL + AnalyticDB MySQL

登录 PolarDB MySQL 概览页 →「联邦分析」进入该服务，进行数据实时同步。

- **新建联邦分析链路**：选择源端实例和目标端实例，默认同步整实例，打开「高级配置」后可以选择库表对象，也可以对大表进行分区键设置。

- **编辑链路、查看链路**：支持修改库表对象等，支持查看联邦分析任务的配置详情。

### 8.2 RDS PG / PolarDB PG + AnalyticDB PG

RDS PG / PolarDB PG → ADB PG 的 Zero-ETL 链路都在 ADB PG 的控制台进行操作。

- 进入 ADB PG 控制台 → 实例 → 左侧导航栏选择**无感集成（Zero-ETL）**→ 单击**创建 Zero-ETL 任务**。

- 创建 Zero-ETL 任务，配置源库信息和目标库信息。

- 测试连接并进行下一步，进入配置 Zero-ETL 页面，配置库表字段，完成后单击**下一步保存任务并预检查**。

- 预检查通过后，系统自动启动 Zero-ETL 任务，在**任务列表**就可以查看任务进度。

---

## 九、传统 ETL vs Zero-ETL 对比

| 对比维度 | 传统 ETL | Zero-ETL |
|----------|----------|----------|
| 工具维护 | 需自行搭建和维护 ETL 工具 | 免维护，全托管服务 |
| 配置复杂度 | 手动配置源库/目标库、填写账密、指定主键/分布键 | 选择源端和目标端实例即可 |
| 成本 | 额外资源成本（ETL 服务器、工具授权） | 免费（仅目标端存储/计算费用） |
| 实时性 | 周期性批量更新，延迟较高 | 实时同步，延迟低 |
| 扩展性 | 新数据源需开发适配 | 支持 PolarDB/RDS 多种数据库 |
| 性能 | 传统 DTS 链路性能 | 比传统 DTS 提升 15%+ |

---

## 十、技术名词对照表

| 技术名词 | 解释说明 |
|----------|----------|
| Zero-ETL | 免 ETL 工具的数据同步服务，自动将 OLTP 数据库数据实时同步到 OLAP 数据仓库 |
| OLTP | Online Transaction Processing，在线事务处理，擅长日常事务处理（如银行交易），不擅长复杂分析 |
| OLAP | Online Analytical Processing，在线分析处理，擅长多表 JOIN、复杂分析和全局数据汇总 |
| AnalyticDB MySQL | 阿里云[云原生数据仓库](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/product-overview/what-is-analyticdb-for-mysql) MySQL 版，MPP 架构，支持 PB 级[实时数据分析](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/use-cases/real-time-data-analysis/) |
| AnalyticDB PostgreSQL | 阿里云云原生数据仓库 PostgreSQL 版，支持企业知识库和实时数仓 |
| ETL | Extract-Transform-Load，数据提取、转换清洗、加载到数据仓库的传统流程 |
| DTS | Data Transmission Service，阿里云数据传输服务，传统数据同步链路 |
| MPP | Massively Parallel Processing，大规模并行处理架构，将复杂查询拆分并行执行 |
| PolarDB MySQL 版 | 阿里云云原生关系型数据库 MySQL 版，兼容 MySQL 生态 |
| PolarDB 分布式版 | 阿里云分布式云原生数据库，支持水平扩展 |
| PolarDB PostgreSQL 版 | 阿里云云原生关系型数据库 PostgreSQL 版 |
| RDS MySQL | 阿里云关系型数据库 MySQL 版 |
| RDS PostgreSQL | 阿里云关系型数据库 PostgreSQL 版 |
| Serverless | 弹性无服务器架构，按需自动分配资源，详见[弹性伸缩](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/user-guide/serverless-auto-scaling-resource-group/) |
| 漏斗分析函数 | 分析用户在行为流中指定步骤转化情况的分析函数 |
| 留存分析函数 | 分析用户整体参与程度和活跃程度的分析函数 |
| 路径分析函数 | 分析行为顺序、行为偏好、关键节点、转化效率的探索型模型 |

---

## 十一、适用场景索引

| 场景编号 | 适用场景 | 典型特征 | 推荐方案 |
|----------|----------|----------|----------|
| S-01 | TP/AP 一体化 | 需要在 OLTP 业务基础上做全局数据分析 | PolarDB MySQL → AnalyticDB MySQL Zero-ETL |
| S-02 | 分库分表数据汇总 | 游戏行业按区服分库分表，需全局分析 | 多 PolarDB 实例 → ADB MySQL 合并 |
| S-03 | 实时运营分析 | 游戏广告实时运营，要求低延迟 | RDS/PolarDB → ADB MySQL 实时同步 |
| S-04 | 日志数据分析 | 应用日志、访问日志秒级分析 | RDS MySQL → ADB MySQL Zero-ETL |
| S-05 | 数字营销平台 | 多源数据汇集 + 复杂 JOIN 分析 | RDS/PolarDB → ADB MySQL |
| S-06 | 企业知识库 | 基于 PostgreSQL 生态的企业知识库 | RDS PG/PolarDB PG → ADB PG |
| S-07 | 一站式实时数仓 | 需要完整数仓链路 | RDS PG/PolarDB PG → ADB PG |
