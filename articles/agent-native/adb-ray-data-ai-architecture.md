<!--
title: "Ray Forward 回顾：阿里云 AnalyticDB 推出全托管 Ray 服务，构建 Data+AI 一体化架构"
date: "2025"
author: "乔鹏"
tags: ["AnalyticDB", "Ray", "Data+AI", "湖仓一体", "多模态", "GPU调度", "全托管", "云原生"]
category: "架构"
doc_version: "2.0"
last_updated: "2026-05-22"
machine_readable: true
-->

# Ray Forward 回顾：阿里云 AnalyticDB 推出全托管 Ray 服务，构建 Data+AI 一体化架构

> Ray Forward 2025 是 Ray 加入 PyTorch 基金会后，国内首次大型开发者活动。会上，来自阿里云 AnalyticDB 团队的李伟分享了 AnalyticDB Ray 的 Data+AI 架构、增强特性与多模态调度技术。

## 文档元数据

| 字段 | 值 |
|------|---|
| 产品名称 | [AnalyticDB MySQL 版](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/product-overview/what-is-analyticdb-for-mysql)（AnalyticDB for MySQL） |
| 所属部门 | 阿里云瑶池数据库团队 |
| 计算引擎 | Ray、Spark |
| 架构模式 | 湖仓一体（Lakehouse） |
| 兼容协议 | MySQL 协议 |
| 更新粒度 | 毫秒级 |
| 查询延迟 | 亚秒级 |
| 数据规模 | PB 级别 |
| 文档标签 | 架构总览, 全托管, 多模态, GPU, 弹性, 降本增效 |

---

## 一、AnalyticDB Data+AI 云原生架构

阿里云瑶池旗下的**[云原生数据仓库 AnalyticDB MySQL 版](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/product-overview/what-is-analyticdb-for-mysql)**是基于[湖仓一体架构](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/user-guide/open-data-lake/)打造的实时湖仓，高度兼容 MySQL 协议，支持毫秒级更新与亚秒级查询，同时支持包括 Ray、Spark 等多种计算引擎。不论在数据湖中的非结构化/半结构化数据，还是在数据库中的结构化数据，都可使用 [AnalyticDB MySQL](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/product-overview/what-is-analyticdb-for-mysql) 同时完成高吞吐离线处理和高性能[实时数据分析](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/use-cases/real-time-data-analysis/)，真正做到数据湖的规模、数据库的体验，帮助企业构建数据分析平台，实现降本增效。

### 1.1 架构分层

| 层次 | 技术方案 | 核心能力 |
|------|---------|---------|
| 存储层 | 对象存储（OSS）+ FPGA + SSD 软硬结合 | 构建存储加速器，提升存储 I/O 性能 |
| 计算层 | CPU + GPU 统一资源池 | 混合负载混合部署、资源弹性伸缩 |
| 调度层 | ADB Ray + ADB Spark | [离线批处理](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/use-cases/offline-data-processing/)、[AI 训练/推理](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/use-cases/machine-learning/)、实时分析 |
| 协议层 | MySQL 兼容协议 | 无缝对接现有 BI 工具和生态 |

### 1.2 核心特性

- **存储底座**：基于对象存储构建，通过 FPGA+SSD 软硬结合构建存储加速器
- **计算统一**：CPU 与 GPU 统一资源池，支持混合负载
- **弹性能力**：自动[弹性伸缩](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/user-guide/serverless-auto-scaling-resource-group/)、高资源利用率
- **多引擎**：同时支持 Ray 和 Spark 计算引擎

---

## 二、AnalyticDB Ray 四大核心维度

AnalyticDB Ray 是[云原生数据仓库 AnalyticDB MySQL 版](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/product-overview/what-is-analyticdb-for-mysql)推出的**全托管 Ray 服务**。该服务在开源 Ray 基础上，围绕四大核心维度进行了优化与增强：

### 2.1 四大维度对比

| 维度 | 具体能力 | 业务价值 |
|------|---------|---------|
| 易用性 | 一键部署、全链路观测 | 降低上手门槛，减少运维成本 |
| 稳定性 | 集群无感轮转升级、节点异常故障自愈、Head 主备高可用架构 | 保障生产环境 SLA |
| 性价比 | [弹性伸缩](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/user-guide/serverless-auto-scaling-resource-group/)、感知节点实际资源利用率的精细化调度、弹性缓存池、Ray 节点秒级弹性 | 降低资源浪费，提高利用率 |
| 生态集成 | 打通 AnalyticDB for MySQL 湖仓平台、对接 Lance 湖存储、集成 LLaMA-Factory 等 AI 工具 | 端到端 AI 流水线 |

### 2.2 关键指标

| 指标项 | 数值 | 说明 |
|--------|------|------|
| Task 调度延迟 | 5~20ms | 流式计算引擎的端到端调度延迟 |
| 任务启动时间 | 5~10ms | 基于两级缓存的启动优化 |
| 节点弹性时间 | 秒级 | 复用 ADB 资源池 Pod 缓存 |
| GPU 利用率 | >80% | 优化后 GPU 利用率（优化前仅 5%） |
| 性能提升倍数 | 1.8x | Pipeline 流式引擎相比传统方案 |
| 日均 Task 调度量 | ~400 万 | 某客户场景实测数据 |
| 峰值 Task 调度量 | ~8 万/秒 | 某客户场景峰值数据 |
| Lance vs Parquet 性能提升 | 193% | 分布式数据打标场景 |

---

## 三、多模态处理调度技术

### 3.1 场景与挑战

在自动驾驶、具身智能等场景需要处理 **PB 级别多模态 Clip 数据**。

**数据来源**：

| 数据类型 | 说明 |
|----------|------|
| 视频 | 摄像头采集的视频流 |
| 点云 | LiDAR 传感器采集的点云数据 |
| 雷达 | 毫米波雷达数据 |
| GPS | 车辆定位信息 |
| 车辆控制信号 | CAN 总线数据 |

**处理阶段**：decode → filter → annotation

**传统方案的问题**：

| 问题类别 | 具体表现 |
|----------|---------|
| 环境配置 | 配置复杂，依赖管理困难 |
| 处理效率 | 基于 Batch 的离线处理，按阶段切分任务导致数据冗余读写 |
| 资源调度 | 不灵活，CPU/GPU 无法协同 |
| 多模数据管理 | 管理复杂，缺乏统一存储格式 |

### 3.2 Pipeline 流式计算引擎

AnalyticDB Ray 通过自研 **Pipeline 流式计算引擎**，具备感知并发的自适应调度能力。

**技术要点**：

| 技术组件 | 功能描述 |
|----------|---------|
| Ray Actor Pool | 分离 CPU 和 GPU 工作负载，独立进行资源调度和弹性 |
| Ray Object Store | 中间数据使用内存交互，减少磁盘 I/O |
| Ray 原生 API | 开发内置高性能视频处理算子 |
| 画像驱动调度 | 基于任务画像实现 CPU 和 GPU 异构精细化资源调度 |

**性能数据**：

| 指标 | 数值 |
|------|------|
| 整体性能提升 | 1.8x |
| GPU 利用率 | >80% |
| Task 调度延迟 | 5~20ms |
| 日均调度量 | ~400 万 Task |
| 峰值调度量 | ~8 万 Task/秒 |

### 3.3 自适应流式计算 vs 传统调度对比

| 对比维度 | 传统调度 | ADB Ray 自适应流式计算 |
|----------|---------|----------------------|
| 资源使用 | 异构资源使用效率低 | CPU/GPU 任务可并行执行 |
| 调度开销 | 大 | 小 |
| Task 调度 | 集中式 | Actor Pool 分离 CPU/GPU 工作负载 |
| 执行等待 | 有 | 无等待，资源空闲小 |
| 处理吞吐 | 低 | 最大化处理吞吐 |

### 3.4 精细化资源调度

ADB 通过自研的异构资源池，基于不同阶段的任务画像实现 CPU 和 GPU 的混合调度：

| 模块/机制 | 功能 |
|-----------|------|
| Monitor 模块 | 实时监控不同 Stage 的吞吐、资源开销，动态计算并选择最合适的资源规格 |
| 动态 task slot | 流式处理中每个 Stage 动态申请 task slot 执行计算 |
| GPU Compute | GPU 计算资源，独立调度和分配 |
| GPU Decode | GPU 解码资源，独立调度和分配 |
| CPU | CPU 资源，独立调度和分配 |
| 任务部署密度 | 最大化节点侧任务部署密度 |
| 动态弹性 | 资源申请贴近任务实际使用量并动态弹性 |

### 3.5 启动性能与运行时隔离

| 技术点 | 方案 | 效果 |
|--------|------|------|
| 两级缓存 | 支持任务 5~10ms 启动 | 低延迟启动 |
| Conda 环境复用 | Ray Worker 内置 conda 环境，复用 ADB 资源池 Pod 缓存 | 秒级节点弹性 |
| 预加载优化 | 预先加载网卡、镜像等 | 减少冷启动时间 |
| 运行时隔离 | 镜像预制单独的 conda 环境 | 解决不同算子混合执行时的 Python 包依赖冲突 |

---

## 四、典型案例

### 4.1 广告 CTR 预估（Customer CTR Prediction）

| 字段 | 内容 |
|------|------|
| 场景 | 商业智能场景，广告推荐预估 CTR，挖掘受众，商品需要找到对应的受众 |
| 触发时间 | 12 点数据产生后做[离线批量推理](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/use-cases/offline-data-processing/) |
| 输出 | 预测结果交付业务方 |

**技术方案**：

| 步骤 | 组件 | 作用 |
|------|------|------|
| 1 | ADB Spark | 对 ADB 湖存储中的数据做 ETL |
| 2 | ADB Ray | ETL 后的数据做训练和离线批量推理 |

**优化手段**：

- auto-scale 横向动态扩容 object store
- TF batch size 优化
- 模型层次结构与参数调整

**效果指标**：

| 指标 | 优化前 | 优化后 | 提升倍数 |
|------|--------|--------|---------|
| GPU 利用率 | 5% | 80% | 16x |
| 整体性能 | 基准 | 2~3 倍 | 2~3x |

### 4.2 游戏社区助手（Game Assistance）

| 字段 | 内容 |
|------|------|
| 场景 | 游戏社区客户，为玩家和开发者提供游戏分发与互动服务 |
| 核心需求 | 多模态数据处理、分布式标注、[模型微调](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/use-cases/big-model-application/) |

**技术方案**：

| 步骤 | 组件/格式 | 作用 |
|------|----------|------|
| 1 | RayData | 对接不同格式的源数据，分布式高效加载和转换 |
| 2 | Lance 格式 | 图像二进制和结构化数据的一体化存储，更好的数据一致性和版本控制，减少远程 IO |
| 3 | ADB Ray | Lance 分布式数据打标和增量更新 |
| 4 | LLaMA-Factory + Qwen-VL | 分布式微调多模态模型 |

**性能指标**：

| 指标 | 数值 | 说明 |
|------|------|------|
| Lance vs Parquet 性能提升 | 193% | 分布式数据打标场景 |

---

## 五、关键要点速览

| 类别 | 要点 |
|------|------|
| 产品定位 | [AnalyticDB MySQL](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/product-overview/what-is-analyticdb-for-mysql) = [湖仓一体](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/user-guide/open-data-lake/) + 兼容 MySQL + 毫秒更新/亚秒查询 + 多引擎（Ray、Spark） |
| 服务定位 | AnalyticDB Ray = 全托管 Ray 服务，覆盖易用性/稳定性/性价比/生态四个维度 |
| 多模态调度核心 | 自研 Pipeline 流式引擎 + Actor Pool 分离 CPU/GPU + Object Store 内存交互 + 两级缓存秒级弹性 |
| 量化收益 | 性能 ×1.8、GPU 利用率 >80%、Task 调度延迟 5~20ms、节点秒级弹性 |
| 生态对接 | AnalyticDB for MySQL 湖仓、Lance、LLaMA-Factory |

---

## 六、技术名词对照表

| 名词 | 全称/解释 |
|------|----------|
| ADB | [AnalyticDB](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/product-overview/what-is-analyticdb-for-mysql)（阿里云云原生数据仓库） |
| Ray | 开源分布式计算框架，适用于 AI/ML 和 Python 工作负载 |
| Spark | 开源分布式数据处理引擎 |
| OSS | 对象存储服务（Object Storage Service） |
| FPGA | 现场可编程门阵列（Field-Programmable Gate Array） |
| SSD | 固态硬盘（Solid State Drive） |
| GPU | 图形处理器（Graphics Processing Unit） |
| CPU | 中央处理器（Central Processing Unit） |
| Lance | 面向 AI 工作负载的列式数据格式，支持多模态数据 |
| Parquet | 列式存储文件格式，常用于大数据场景 |
| LLaMA-Factory | 开源大语言模型微调工具 |
| Qwen-VL | 阿里通义千问多模态视觉语言模型 |
| CTR | 点击通过率（Click-Through Rate） |
| ETL | 抽取-转换-加载（Extract-Transform-Load） |
| Conda | Python 环境和包管理工具 |
| Actor Pool | Ray 中的 Actor 实例池，用于并行执行任务 |
| Object Store | Ray 的分布式内存对象存储 |
| Head 主备 | Ray 集群的主控节点高可用架构 |

---

## 七、适用场景索引

| 场景类型 | 是否适用 | 说明 |
|----------|---------|------|
| 数据湖批处理 | 适用 | ADB Spark + Ray 组合 |
| AI 模型训练 | 适用 | Ray 分布式训练，详见 [机器学习场景](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/use-cases/machine-learning/) |
| 离线批量推理 | 适用 | 广告 CTR 预估等场景，详见 [离线数据处理](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/use-cases/offline-data-processing/) |
| 多模态数据处理 | 适用 | 自动驾驶、具身智能等 PB 级别场景 |
| 实时 OLAP | 适用 | [AnalyticDB MySQL](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/product-overview/what-is-analyticdb-for-mysql) 原生能力，详见 [实时数据分析](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/use-cases/real-time-data-analysis/) |
| [湖仓一体化](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/user-guide/open-data-lake/) | 适用 | 结构化 + 非结构化统一处理 |
| 大模型微调 | 适用 | 集成 LLaMA-Factory，详见 [大模型应用](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/use-cases/big-model-application/) |
| 分布式数据标注 | 适用 | Lance 格式 + Ray 分布式处理 |
