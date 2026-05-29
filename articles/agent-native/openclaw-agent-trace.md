<!--
title: "OpenClaw 日志 x ADB MySQL：Agent Trace 诊断实战"
date: "2025"
author: "乔鹏"
tags: ["Agent", "Trace 诊断", "Token 归因", "Prompt 优化", "AI 可观测性", "大模型应用"]
category: "架构"
doc_version: "2.0"
last_updated: "2026-05-24"
machine_readable: true
-->

# OpenClaw 日志 x [ADB MySQL](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/product-overview/what-is-analyticdb-for-mysql)：Agent Trace 诊断实战

> 本文借助 ADB MySQL 的 Agent 日志可观测能力，在 SQL 引擎里闭环跑通从原始日志到 Token 归因与 Prompt 优化的完整实践，为生产级 Agent 构建专属评估体系。

---

## 一、背景

据 Gartner 预测，未来几年将有超过 40% 的 Agentic AI 项目因无法达成商业目标而被取消。核心原因之一：团队构建了**错位的评估体系**——过度依赖"幻觉率"等通用指标，无法捕获真正导致 Agent 表现失控的根因。而成功交付生产级 Agent 的研发团队做法恰恰相反：**大家选择深入剖析底层 Traces，从真实数据中推演出专属评估指标。**

在 OpenClaw 时代，Agent Trace 日志记录了用户 Query、模型推理、工具调用及最终输出的完整执行计划。本文借助 **[ADB MySQL](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/product-overview/what-is-analyticdb-for-mysql)** 的 Agent 日志可观测能力，在 SQL 引擎里闭环跑通这套最高 ROI 的 Agent 数据工程实践。

---

## 二、Agent 可观测性的三层困境，与 ADB MySQL 的解法

### 困境一：观测盲区——Trace 链路不可见

Agent 执行过程是非线性的：一次用户 Query 背后，可能触发 5 次工具调用、3 次模型推理、N 次重试。原始日志是扁平的行记录，没有任何工具能告诉你"这次任务到底发生了什么"。

→ **ADB MySQL 的解法**：窗口函数一行代码将线性日志重建为完整任务链，让每条 Trace 的执行路径从黑盒变透明。

---

### 困境二：TokenOps 成本失控——烧了多少钱，不知道烧在哪

50 万 Token 消耗里，多少是正常推理，多少在和错误 API 反复死磕？传统监控只能看总量，无法归因到具体失效类型。

→ **ADB MySQL 的解法**：直接在 SQL 中聚合各失效模式的 Token 消耗，精确回答"幻觉让我损失了多少钱、修哪个 Prompt 收益最大"。

---

### 困境三：失效无法定位——排障靠猜，经验无法沉淀

上下文窗口一清空，排障经验就蒸发。团队凭直觉猜测失败原因，同样的问题反复踩坑，高价值过程无人整理。

→ **ADB MySQL 的解法**：内置 `ai_classify` 和 `ai_generate` 函数，在 SQL 中直接调大模型完成失效分类与根因诊断，并自动生成 Prompt 优化建议——零 Python 代码，失效经验自动入库沉淀。

---

相比于 Python 代码的单机处理，ADB MySQL 强大的分布式计算和向量化执行引擎能够在秒级完成大规模 Agent 日志 Trace 的复杂解析任务，一站式建立从日志采集到语义提取的完整解决方案。

以下基于产研团队内部类似 OpenClaw 日常产出的真实日志数据，进行实操复现。

---

## 三、实战：从原始日志到 Token 归因与 Prompt 优化

### Step 1：从非结构化日志提取完整执行链路

Agent 日志是高度非结构化的，首先要将线性日志切分为有业务意义的"任务链"。在 ADB MySQL 中，**一个窗口函数即可完成**：

```sql
WITH TaskBoundaries AS (
    SELECT *,
           SUM(CASE WHEN role = 'user' THEN 1 ELSE 0 END)
               OVER (PARTITION BY session_id ORDER BY row_id) AS chain_id
    FROM openclaw_logs.openclaw_sessions
    WHERE role IS NOT NULL
),
TaskChains AS (
    SELECT
        CONCAT(session_id, '_', chain_id)  AS unique_chain_id,
        GROUP_CONCAT(... ORDER BY row_id SEPARATOR ' >>> ')  AS full_trace,
        COUNT(CASE WHEN tool_name IS NOT NULL THEN 1 END)    AS tool_usage_count,
        ...
    FROM TaskBoundaries
    GROUP BY session_id, chain_id
)
SELECT chain_id, session_id, tool_usage_count, LEFT(full_trace, 200) AS trace_preview
FROM TaskChains LIMIT 5;
```

> **执行结果**：1484 行日志 → 171 个完整任务链，292 次工具调用。

---

### Step 2：用 AI 函数自动标注失败模式

ADB MySQL 的 `ai_classify` 在 SQL 中直接调大模型完成**失效分类**，`ai_generate` 自动生成根因诊断——零 Python 代码：

```sql
WITH TaskBoundaries AS ( ... ),  -- 同 Step 1
TaskChains AS ( ... )
SELECT
    unique_chain_id,
    ai_classify('qwen_max_test', LEFT(full_trace, 600),
        '["死循环", "工具参数幻觉", "拒绝执行", "逻辑断裂", "成功解决"]'
    ) AS failure_label,
    ai_generate('qwen_max_test',
        CONCAT('你是OpenClaw AI诊断员。分析以下任务链...', LEFT(full_trace, 400))
    ) AS root_cause_notes
FROM TaskChains
WHERE tool_usage_count > 0 OR last_stop_reason IS NULL OR last_stop_reason != 'stop';
```

> **分析结果**：15% 的任务链存在失败风险，其中 10.5% 陷入"工具参数幻觉"。100% 的根因指向**工具返回数据质量问题**（API 密钥缺失、路径越界等），而非模型推理缺陷。

---

### Step 3：量化各失效模式的 Token 消耗

失效模式定义后，下一步量化它——**幻觉让我损失了多少 Token，修哪个 Prompt 收益最大？**

```sql
WITH TaskBoundaries AS ( ... ),
ChainTokens AS (
    SELECT CONCAT(session_id, '_', chain_id) AS unique_chain_id,
           SUM(IFNULL(total_tokens, 0))      AS chain_total_tokens
    FROM TaskBoundaries GROUP BY session_id, chain_id
)
SELECT a.failure_label, COUNT(*) AS task_count,
       ROUND(AVG(ct.chain_total_tokens)) AS avg_tokens,
       SUM(ct.chain_total_tokens)        AS total_tokens_burned
FROM openclaw_logs.t_ai_audit_results a
JOIN ChainTokens ct ON a.unique_chain_id = ct.unique_chain_id
GROUP BY a.failure_label
ORDER BY total_tokens_burned DESC;
```

结论令人震惊：

> **工具参数幻觉仅占 15% 的任务量，却烧掉了 3,161,237 Token——是全部成功任务总量的 3.27 倍！** 单条最高消耗达 958,743 Token。

进一步通过 `tokens_per_step` 下钻，可精准定位每条失败链路的单步消耗密度：

---

### Step 4：基于根因生成 Prompt 优化建议

Agent 失败分两种：规范失败（指令不清晰）和泛化失败（模型无法应用指令）。最优先动作：**先修 Prompt，别急着建评估器**。利用 `ai_generate`，一条 SQL 同时提取**原始指令**和**优化 Prompt** 做并排对比：

```sql
WITH TaskBoundaries AS ( ... ),
ChainTokens AS ( ... ),
FirstUserMsg AS (
    SELECT CONCAT(session_id, '_', chain_id) AS unique_chain_id,
           SUBSTRING_INDEX(content_text, CONCAT('```', CHAR(10), CHAR(10)), -1) AS original_prompt,
           ROW_NUMBER() OVER (PARTITION BY session_id, chain_id ORDER BY row_id) AS rn
    FROM TaskBoundaries WHERE role = 'user'
),
FailedChains AS (
    SELECT unique_chain_id, failure_label, root_cause_notes,
           ROW_NUMBER() OVER (PARTITION BY unique_chain_id ORDER BY created_at DESC) AS rn
    FROM openclaw_logs.t_ai_audit_results
    WHERE failure_label != '成功解决'
)
SELECT fc.unique_chain_id, LEFT(fu.original_prompt, 200) AS original_prompt,
       fc.root_cause_notes AS root_cause,
       ai_generate('qwen_max_test',
           CONCAT('你是Prompt优化专家。...原始指令：', LEFT(fu.original_prompt, 500),
                  '失败根因：', fc.root_cause_notes)
       ) AS optimized_prompt
FROM FailedChains fc
JOIN ChainTokens ct ON ...
JOIN FirstUserMsg fu ON ... AND fu.rn = 1
WHERE fc.rn = 1
ORDER BY ct.chain_total_tokens DESC LIMIT 3;
```

---

### 实战步骤与关键数据汇总

| 步骤 | 目标 | 核心技术 | 输入 | 输出 | 关键数据 |
|------|------|----------|------|------|----------|
| Step 1 | 提取完整执行链路 | 窗口函数 (SUM OVER + CASE) | 1484 行线性日志 | 171 个任务链 | 292 次工具调用 |
| Step 2 | 自动标注失败模式 | `ai_classify` + `ai_generate` | 任务链 Trace | 失效分类 + 根因诊断 | 15% 任务链有失败风险 |
| Step 3 | 量化 Token 消耗 | SQL JOIN + 聚合 | 失败模式 + Token 统计 | 各失效模式 Token 消耗 | 幻觉占 15% 任务量但消耗 3.27 倍 Token |
| Step 4 | 生成 Prompt 优化建议 | `ai_generate` + 多表 JOIN | 失败链 + 原始指令 | 优化后的 Prompt | 零 Python 代码，自动入库 |

### 失效模式分析结果

| 失效模式 | 任务占比 | Token 消耗占比 | 根因类型 | 修复优先级 |
|----------|----------|----------------|----------|------------|
| 工具参数幻觉 | 10.5% | 极高（316 万 Token） | 工具返回数据质量问题 | P0（修 Prompt 收益最大） |
| 其他失败模式 | ~4.5% | 待统计 | 待分析 | P1 |
| 成功解决 | ~85% | 正常 | - | - |

---

## 四、技术名词对照表

| 技术名词 | 解释说明 |
|----------|----------|
| AnalyticDB for MySQL (ADB MySQL) | 阿里云[云原生数据仓库](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/product-overview/what-is-analyticdb-for-mysql/) MySQL 版，支持 [AI 函数](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/use-cases/big-model-application/)（`ai_classify`、`ai_generate`）和大模型集成 |
| Agent Trace | Agent 执行过程的完整日志记录，包含用户 Query、模型推理、工具调用和最终输出 |
| TokenOps | Token 运营管理，关注 Token 消耗的成本归因和优化 |
| `ai_classify` | ADB MySQL 内置 AI 函数，在 SQL 中直接调用大模型进行分类任务 |
| `ai_generate` | ADB MySQL 内置 AI 函数，在 SQL 中直接调用大模型生成文本内容 |
| 窗口函数 (Window Function) | SQL 分析函数，如 SUM() OVER()，用于在不改变行数的情况下计算聚合值 |
| Qwen | 阿里云通义千问大语言模型，用于失效分类和根因诊断 |
| 失效模式 | Agent 执行失败的具体表现类型：死循环、工具参数幻觉、拒绝执行、逻辑断裂等 |
| 工具参数幻觉 | Agent 在调用工具时产生错误的参数，导致反复重试和 Token 浪费 |
| 规范失败 | 由于指令/Prompt 不清晰导致的 Agent 失败 |
| 泛化失败 | 模型无法将指令应用到新场景导致的失败 |
| Bootstrapping | 引导/自举过程，Agent 通过经验回放逐步自我优化 |
| 大模型应用 | 将大语言模型集成到业务流程中，详见 ADB [大模型应用场景](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/use-cases/big-model-application/) |

## 五、适用场景索引

| 场景编号 | 适用场景 | 典型特征 | 推荐方案 |
|----------|----------|----------|----------|
| S-01 | Agent Trace 可观测性建设 | 需要监控 Agent 执行过程，建立评估体系 | ADB MySQL + 窗口函数 + 日志采集 |
| S-02 | Token 成本归因分析 | Token 消耗大，需要精确到失效类型的成本分析 | ADB MySQL + `ai_classify` + SQL 聚合 |
| S-03 | Prompt 自动优化 | 需要持续优化 Agent Prompt，减少失败 | ADB MySQL + `ai_generate` + 定时任务 |
| S-04 | Agent 失效根因诊断 | 排障靠猜，需要自动化分类和归因 | ADB MySQL + `ai_classify` 自动标注 |
| S-05 | 生产级 Agent 评估体系 | 需要基于统计分布的失效分析而非简单指标 | ADB MySQL SQL 引擎 + AI 函数 |
| S-06 | Agent 经验沉淀 | 高价值 SOP 需要沉淀和复用 | ADB MySQL 数据库作为"经验回放缓冲区" |

---

## 六、结语

我们用 **4 条 SQL 走完了从原始日志到闭环修复的全链路**。三个 insight：

- **Agent 的失败是概率性的。** 同一条指令 10 次执行可能成功 7 次，每次失败路径不同。传统测试无效，**你需要基于统计分布的失效模式分析**——SQL 引擎天然擅长。
- **Token 异常是最好的异常检测器。** 16 条幻觉链路消耗 310 万 Token，是成功任务总和的 3.27 倍。Token 飙升意味着模型在"打转"，远比规则检测灵敏。
- **AI 可观测性的终局是"闭环"。** 数据库充当"反射弧"——失败 Trace 进去，优化 Prompt 出来。封装为定时任务，Agent 就获得了持续运转的**免疫系统**。

**不要把 AI 的命脉交给通用外部指标。** 死磕 Trace，围绕真实问题构建专属评估体系——ADB MySQL 把这套工程压缩成了几行 SQL，轻松嵌入到企业日常的 workflow 里。这为企业提供了一个绝佳的"经验回放缓冲区"，让高价值的 SOP 在 Agent 进行 Bootstrapping 的过程里得以沉淀。
