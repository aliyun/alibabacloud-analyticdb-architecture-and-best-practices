<!--
repository: adb-mysql-architecture-and-best-practices
product: AnalyticDB MySQL（云原生数据仓库）
product_url: https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/product-overview/what-is-analyticdb-for-mysql
team: 阿里云瑶池数据库
article_count: 16
format: machine-readable markdown (structured tables, glossaries, scenario indexes, official doc hyperlinks)
categories: 4
layout: articles organized into category subdirectories (agent-native, lakehouse-zero-etl, kernel, diagnosis)
last_updated: 2026-05-25
-->

# ADB-MySQL 架构与最佳实践

> 分享阿里云云原生数据仓库 **[AnalyticDB MySQL 版](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/product-overview/what-is-analyticdb-for-mysql)**（ADB-MySQL）相关的技术文档、架构解读与最佳实践。

本仓库收录与 ADB-MySQL 相关的技术文章，按 **Agent-Native（智能体原生）**、**Lakehouse & Zero-ETL（湖仓与下一代数据集成）**、**Kernel 产品功能** 和 **诊断和优化** 四个方向组织在 `articles/` 下的对应子目录，帮助开发者更好地理解和使用 ADB-MySQL 构建 Data+AI 一体化的数据分析平台。

---

## 分类总览

<!--
category_index:
  - id: 1
    name: Agent-Native
    cn_name: 智能体原生
    dir: articles/agent-native
    article_count: 4
  - id: 2
    name: Lakehouse & Zero-ETL
    cn_name: 湖仓与下一代数据集成
    dir: articles/lakehouse-zero-etl
    article_count: 4
  - id: 3
    name: Kernel 产品功能
    cn_name: 内核产品功能
    dir: articles/kernel
    article_count: 7
  - id: 4
    name: 诊断和优化
    cn_name: 诊断和优化
    dir: articles/diagnosis
    article_count: 1
-->

| 分类 | 主题方向 | 子目录 | 文章数 |
|------|---------|--------|--------|
| 1. Agent-Native（智能体原生） | Data+AI 一体化、Ray 全托管、Agent 可观测性、AI 助手运维 | [`articles/agent-native/`](articles/agent-native/) | 4 |
| 2. Lakehouse & Zero-ETL（湖仓与下一代数据集成） | 湖仓一体、Zero-ETL、Flink 高吞吐入湖 | [`articles/lakehouse-zero-etl/`](articles/lakehouse-zero-etl/) | 4 |
| 3. Kernel 产品功能 | 存储引擎、SQL 优化器、谓词下推、RuntimeFilter、Serverless、Iceberg、漏斗分析 | [`articles/kernel/`](articles/kernel/) | 7 |
| 4. 诊断和优化 | SQL 智能诊断 | [`articles/diagnosis/`](articles/diagnosis/) | 1 |

---

## 分类 1：Agent-Native（智能体原生）

> Data+AI 一体化基础设施 + 围绕 Agent 全生命周期（开发、运维、可观测）的能力建设。

| 标题 | 关键词 | 全文链接 |
|------|--------|----------|
| Ray Forward 回顾：阿里云 AnalyticDB 推出全托管 Ray 服务 | AnalyticDB Ray、Data+AI 架构、多模态调度、GPU 利用率 | [adb-ray-data-ai-architecture.md](articles/agent-native/adb-ray-data-ai-architecture.md) |
| 多模 ETL+ML 一体化：AnalyticDB+Ray 解锁仓内 AI 流水线 | AnalyticDB Ray、多模态 ETL、LLM 蒸馏、分布式微调 | [adb-ray-multimodal-etl-ml.md](articles/agent-native/adb-ray-multimodal-etl-ml.md) |
| OpenClaw 日志 x ADB MySQL：Agent Trace 诊断实战 | Agent 可观测性、Trace 链路、Token 归因、Prompt 优化 | [openclaw-agent-trace.md](articles/agent-native/openclaw-agent-trace.md) |
| ADB MySQL AI 助手重塑数据库诊断体验 | AI 助手、多轮问答、空间诊断、SQL 对比、CPU 分析 | [adb-mysql-ai-assistant.md](articles/agent-native/adb-mysql-ai-assistant.md) |

---

## 分类 2：Lakehouse & Zero-ETL（湖仓与下一代数据集成）

> 一份数据、统一管理；从传统 ETL 走向 Zero-ETL 与高吞吐、高一致性的入湖体系。

| 标题 | 关键词 | 全文链接 |
|------|--------|----------|
| 高效易用的数据同步：阿里云瑶池 Zero-ETL 服务 | Zero-ETL、数据同步、OLTP→OLAP、PolarDB、RDS | [zero-etl-sync.md](articles/lakehouse-zero-etl/zero-etl-sync.md) |
| 云原生数据仓库下的"降本增效"之路 | Zero-ETL、PolarDB 联邦分析、ADB PG 无感集成、免费链路 | [zero-etl-road.md](articles/lakehouse-zero-etl/zero-etl-road.md) |
| AnalyticDB MySQL 湖仓版公测发布 | 湖仓一体、羲和融合引擎、MPP+BSP、Serverless、Hudi | [adb-lakehouse.md](articles/lakehouse-zero-etl/adb-lakehouse.md) |
| 基于 Flink 的高吞吐精确一致性入湖实现 | APS、SLS、Flink + Hudi、两阶段提交、Exactly-Once、Recommit | [sls-exactly-once-lake.md](articles/lakehouse-zero-etl/sls-exactly-once-lake.md) |

---

## 分类 3：Kernel 产品功能

> 数据库内核能力建设：存储引擎、查询优化器、谓词下推、RuntimeFilter、Serverless 弹性、Iceberg 表服务、漏斗分析等。

| 标题 | 关键词 | 全文链接 |
|------|--------|----------|
| ADB Iceberg 表优化服务：让湖仓数据自动提速降本 | Iceberg、Table Service、冷热分层、Bitmap 索引 | [adb-iceberg-table-service.md](articles/kernel/adb-iceberg-table-service.md) |
| AnalyticDB MySQL 的 Serverless 弹性技术解析 | Serverless、ACU、池化调度、两级库存、离在线解耦 | [serverless-elasticity.md](articles/kernel/serverless-elasticity.md) |
| AnalyticDB MySQL 实时存储引擎演进之路 | PAX layout、行列混存、Anti-Caching、Swizzling Pointer、BufferPool | [realtime-storage-engine.md](articles/kernel/realtime-storage-engine.md) |
| 深入浅出 SQL 优化器原理 | RBO、CBO、直方图、Cardinality Estimation、Cascades | [sql-optimizer.md](articles/kernel/sql-optimizer.md) |
| AnalyticDB MySQL 过滤条件智能下推原理 | 谓词下推、全索引、selectivity、conjunction、代价模型 | [smart-predicate-pushdown.md](articles/kernel/smart-predicate-pushdown.md) |
| AnalyticDB MySQL 如何打造极致 RuntimeFilter 能力 | RuntimeFilter、SIP 框架、全局等价关系、Bloom Filter、TPC-DS | [runtime-filter.md](articles/kernel/runtime-filter.md) |
| 10 倍性能提升：AnalyticDB 秒级漏斗分析函数 | window_funnel、漏斗分析、滑动窗口、AARRR、营销场景 | [window-funnel.md](articles/kernel/window-funnel.md) |

---

## 分类 4：诊断和优化

> 面向用户体验的 SQL 智能诊断与调优能力。

| 标题 | 关键词 | 全文链接 |
|------|--------|----------|
| AnalyticDB「SQL 智能诊断」功能详解 | SQL 诊断、甘特图、执行计划、自诊断、数据倾斜、内存调优 | [sql-smart-diagnosis.md](articles/diagnosis/sql-smart-diagnosis.md) |

---

## 文档格式说明

每篇文章均采用机器友好格式，包含以下结构化元素：

| 元素 | 说明 |
|------|------|
| HTML 注释元数据 | 文件开头包含 title、date、author、tags、category 等字段，机器可解析 |
| 结构化表格 | 关键指标、架构分层、性能对比、参数规格等均以 Markdown 表格呈现 |
| 技术名词对照表 | 每篇文末提供术语全称和解释 |
| 适用场景索引 | 每篇文末提供使用场景与适用性映射 |
| 官网超链接 | 关键术语链接至 [AnalyticDB MySQL 官方文档](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/) |

---

## 目录结构

```
.
├── README.md                          # 仓库索引（本文件）
└── articles/                          # 文章正文，按分类组织
    ├── agent-native/                  # 1. 智能体原生
    │   ├── adb-ray-data-ai-architecture.md
    │   ├── adb-ray-multimodal-etl-ml.md
    │   ├── openclaw-agent-trace.md
    │   └── adb-mysql-ai-assistant.md
    ├── lakehouse-zero-etl/            # 2. 湖仓与下一代数据集成
    │   ├── zero-etl-sync.md
    │   ├── zero-etl-road.md
    │   ├── adb-lakehouse.md
    │   └── sls-exactly-once-lake.md
    ├── kernel/                        # 3. 内核产品功能
    │   ├── adb-iceberg-table-service.md
    │   ├── serverless-elasticity.md
    │   ├── realtime-storage-engine.md
    │   ├── sql-optimizer.md
    │   ├── smart-predicate-pushdown.md
    │   ├── runtime-filter.md
    │   └── window-funnel.md
    └── diagnosis/                     # 4. 诊断和优化
        └── sql-smart-diagnosis.md
```

> 文件名已移除数字编号；如需按发布时间检索，请查阅各文章开头 HTML 注释中的 `date` 字段，或使用 `git log --follow <path>` 查看完整迁移与提交历史。

---

## 相关链接

| 资源 | 链接 |
|------|------|
| ADB MySQL 产品概述 | https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/product-overview/what-is-analyticdb-for-mysql |
| ADB MySQL 官网 | https://www.aliyun.com/product/ApsaraDB/ads |
| ADB MySQL 用户指南 | https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/user-guide/ |
| ADB MySQL 最佳实践 | https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/use-cases/ |
