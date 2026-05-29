<!--
title: "深入浅出 SQL 优化器原理"
date: "2025"
author: "乔鹏"
tags: ["SQL优化器", "CBO", "RBO", "Cascades", "基数估计", "直方图"]
category: "架构"
doc_version: "2.0"
last_updated: "2026-05-24"
machine_readable: true
-->

# 深入浅出 [AnalyticDB MySQL](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/product-overview/what-is-analyticdb-for-mysql) SQL 优化器原理

> SQL 优化器是数据库、数据仓库、大数据等相关领域中最复杂的内核模块之一，它是影响查询性能的关键因素。本文由浅入深地带你了解 SQL 优化器的技术原理。

---

## 一、起源

1979 年，第一款基于 SQL 的商业关系型数据库管理系统 Oracle V2 问世，也标志着第一款商用的 SQL 优化器诞生。理论上，成熟的优化器原型，更早可以追溯到 IBM 的 System-R 项目。现今，很多开源数据库和大数据优化器还是沿用 System-R 原型。

---

## 二、从一条 SQL 开始

SQL（Structured Query Language）是一种结构化的查询语言。它只描述了用户需要什么样的数据，而没有告诉数据库该如何执行。这使得有很多优化空间蕴含在 SQL 改写中。

```sql
/* 查询五年级学生的平均年龄 */
SELECT avg(s.age)
FROM students s
JOIN classes c ON s.cls_id = c.id
WHERE c.grade = 5
```

这个查询有两种执行方式。第一种，直接按用户写的顺序执行：先做 INNER JOIN，然后过滤（只保留五年级学生），最后聚合求平均。第二种，对 SQL 进行适当改写后再执行：

```sql
SELECT avg(s.age)
FROM students s
JOIN (
  SELECT id
  FROM classes
  WHERE grade = 5  /* 下推过滤条件 */
) c ON s.cls_id = c.id
```

改写后的 SQL，在 INNER JOIN 运算前就完成了过滤运算。这样参与 JOIN 运算的数据量会更少，查询效率也会更快。优化器主要工作就是在保证等价的前提下，尽可能让查询更快执行。

优化器将抽象语法树（AST）先转换为初始的逻辑计划（Logical Plan），一般这个过程中会同时进行基本的检查工作，比如：表存不存在、权限够不够等。逻辑计划经过一系列等价改写，并选择每个算子如何执行，最终得到一个可执行的物理计划（Physical Plan）。

---

## 三、基本优化原理

### 3.1 基于规则的优化（RBO）

RBO（Rule-Based Optimizer），优化器中有很大一部分工作是基于规则进行的。简单地说，就是我认为什么样的形式是更优的，我就把 SQL 改写成什么样。比如上文提到的下推过滤条件，就是一种规则。

这类优化有很多：下推、裁剪、简化表达式、解关联等。还可以利用隐藏在 SQL 中的信息，做更深层次的优化：

```sql
/* 查询第一个班级学生的平均年龄 */
SELECT avg(s.age)
FROM students s
JOIN classes c ON s.cls_id = c.id
WHERE c.id = 1
```

除了将过滤条件下推外，还可以为 students 表生成一个条件 `s.cls_id = 1`，因为通过 JOIN 条件，我们知道 `s.cls_id` 等价于 `c.id`。这样又可以更早地过滤掉一部分数据，减少 JOIN 运算量。

到这里，对 SQL 优化有了一些基本的感觉。但是这里有一个容易被忽视的问题：这两张表进行 JOIN 的时候，到底谁在左，谁在右？

一般来说，INNER JOIN 的实现会有 HASH JOIN。JOIN 右端数据用来构建 HashSet，左端数据用来查找 HashSet。大家一定会把 classes 表放右边——因为 classes 表显然更小，HashSet 构建效率高，占用内存少，查询效率也更快。

但优化器并不知道 students 和 classes 的业务关系，也不知道它们之间的数据量有数量级的差距。这个时候基于代价的优化就派上用场了。

### 3.2 基于代价的优化（CBO）

CBO（Cost-Based Optimizer），SQL 优化中大部分收益是来自 CBO 的优化。因为很多很有效的优化手段是没法 100% 确定有收益的，这个时候就需要通过估计执行代价来评估。优化器中 CBO 是标配，不同产品之间，只是实现程度深浅的问题。

CBO 中一个很核心的问题，就是怎么估算。这需要引入**统计信息（Statistics）**概念。统计信息是事先对表里的数据进行分析，收集到的信息。大部分数据库都支持手动执行 ANALYZE 命令来收集。像 [AnalyticDB](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/product-overview/what-is-analyticdb-for-mysql) 这类商业数据仓库产品，一般都支持自动收集。

这个例子要用到的统计信息是 NDV（The Number of Distinct Values），然后还有一个均匀假设（Uniformity Assumption）。假设学生有 1000 人，班级有 5 个，班级人数均匀分布。

根据估算结果，优化器会把 students 放左边，用 classes 表去构建 HashSet。这是一个简单又典型的事实表和维度表关系。如果把条件换成 `s.cls_id >= 1`，就需要再引入范围的统计信息 Min & Max。再复杂点，如果实际人数是不均匀的，有些班级甚至可能没人，那么左右关系就需要互换。应对实际中非均匀的模型（大部分业务数据不是均匀分布的），需要引入**直方图统计信息**来解决。

#### 3.2.1 直方图

直方图（Histogram）基本是商业数仓必备的能力。直方图一般分为等宽和等高，等高应用会比较多些，因为能更好应对极值。

上图这个查询条件，如果只使用基本的统计信息（Min=1, Max=5, NDV=5），估计结果误差达 71%，严重高估。使用直方图，估计结果为 350，是完全准确的。

实际运用中，我们不可能为每种值都建立一个桶（精准直方图），一般会限制桶的数量来减少直方图计算开销和存储开销。这时候多种值会被划入一个桶，划分的方式就是等宽与等高。一般 100~300 个桶，就能比较好地在估算误差和开销之间权衡了。

在 AnalyticDB 中会自动识别，选择要建立怎样的直方图。对于 NDV 很小的情况，建立精准直方图就再合适不过了。即便不建立精准直方图，AnalyticDB 也会识别一些热点值，让它们单独一个桶，以增加估算精度。

#### 3.2.2 低估误差

实际中还需要估计范围、NDV 等信息经过每个算子后的变化。这个一般由优化器中叫 Cardinality Estimation（CE）的模块负责。除此以外，还需要根据 CE 提供的信息 + Cost Model，估计每个算子的代价（CPU/IO/MEM/NET）。

虽然有误差，但是误差并不一定会影响计划选择。导致误差的具体原因通常是这几个方面：

1. **估算能力不足**：比如不支持直方图，那就不能有效应对非均匀数据
2. **统计信息本身有误差**：收集时使用了近似算法，以及信息的时效性导致
3. **列与列之间有一定联系**：这层联系不得而知，影响估计结果
4. **累积误差**：经过每个算子后，推导出的统计信息存在误差，这些误差会层层累积
5. **确实难以估计的表达式**：比如没有常量前缀的 LIKE 表达式

[AnalyticDB](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/product-overview/what-is-analyticdb-for-mysql) 会自动分析 SQL 复杂程度，决定是否对复杂的 filter 进行动态采样，以提高计划质量。同时还有一些运行态自动调整计划的技术（Adaptive Query Processing），可以纠正计划。

### 3.3 搜索框架

要选出代价最低的计划，涉及到怎么高效率找出所有可能的计划，并选出代价最低的。最经典的方案是 System-R 风格，以 RBO + cost-based join reorder（bottom-up DP）为主，优点就是简单高效，像 MySQL 这种 OLTP 数据库就采用的这种方案。缺点就是容易陷入局部最优，而非全局最优。

商业数据库/数仓和开源产品，基本都或多或少参考 **Cascades 理论**构建搜索框架。比如 SQLServer、Snowflake、GreenplumDB 和 Calcite。AnalyticDB 的 CBO 也是以该理论为基础来构建的。

Cascades 跟传统的框架比，主要有这么几个优势：

- 搜索方向从 bottom-up 变成 top-down，有更多裁剪机会，效率更高
- 分层搜索（先搜逻辑再搜物理）到统一搜索（逻辑和物理混着一起搜），能更早触发裁剪，效率高
- 框架对优化器工作进行高度抽象后，可并行搜索，扩展性也更强

整个 Cascades 工作原理就是一个树型 DP（Dynamic Programing），通过记忆化搜索（Memoization Search）方式从上至下（top-down）推进。

#### 核心概念

- **Expression**：算子表达式，比如 JOIN 是一种 Expression，Filter 是一种 Expression。可以是逻辑的，也可以是物理的。
- **GroupExpression**：和 Expression 类似，只不过它的子节点是 Group。这样更加抽象，目的是减少表达式数量。
- **Group**：计划树中的一个点，用来归纳重复的信息到一起。包含等价的 GroupExpression、输出的统计信息、代价下限、输出的属性信息等。
- **Winner**：存储了特定请求的最优解，是记忆化搜索的体现。
- **PropertyEnforcer**：当子节点无法满足父节点属性要求时，强行插入一个算子来满足要求。
- **Memo**：搜索空间，用来存储 Group。
- **RULE**：分两类——Transformation rules（逻辑转逻辑）和 Implementation rules（逻辑转物理）。
- **TASK**：驱动上面这些东西运转的各种 Task，本质上是对递归遍历树过程的一个抽象。

#### 关键 Task 流程

- **OptimizeGroup**：对特定 Group 发起特定请求，是每个 Group 优化的入口。
- **OptimizeGroupExpression**：对特定逻辑表达式进行优化，主要应用 transformation 和 implementation 两种规则。
- **ExploreGroup**：被 OptimizeGroupExpression 调用，探索 Group 内可能生成的所有等价逻辑表达式。
- **ApplyRule**：输入是逻辑表达式和规则，根据规则类型得到新的逻辑或物理表达式。
- **OptimizeInput**：输入是一个物理表达式和属性要求，向所有子 Group 发起 OptimizeGroup 请求。

#### 分支裁剪

**上界裁剪**：当向子 Group 请求的过程中，不断收集到 winner，如果当前累加的 cost 已经超过上界，那么可以直接停止搜索。上界随着第一个满足要求的物理表达式得到一个 cost，被当作初始上界，并通过请求上下文带给其它任务。

**下界裁剪**：根据 cost model + 逻辑属性，直接确定一个 Group 的最小开销（下界）。如果 `sum(subGroup.costLowerBound) + GroupExpression.cost > upper bound` 成立，那根本不需要对子 Group 进行任何探索，直接就可以停了。

这些裁剪技术在 [AnalyticDB](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/product-overview/what-is-analyticdb-for-mysql) 中都有应用。经过测试，在 TPC-DS 中，大部分查询搜索空间都有减少，有些甚至可以减少近 50% 的搜索空间。

---

## 四、结语

在数据库和大数据等相关领域，查询优化十分重要。实际生产中的问题，远比本文提到的要复杂。[AnalyticDB MySQL](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/product-overview/what-is-analyticdb-for-mysql) 作为国内领先的云原生数仓和 TPC-DS 世界记录保持者，在查询优化技术上不断投入和创新。

---

## 五、结构化数据表

### 5.1 SQL 优化器演进历程

| 阶段 | 年份 | 代表系统 | 优化方式 |
|------|------|---------|---------|
| 早期原型 | 1970s | IBM System-R | RBO + bottom-up DP |
| 商业化 | 1979 | Oracle V2 | 基于规则的商用优化器 |
| 现代开源 | 2000s+ | MySQL, Calcite | RBO + cost-based join reorder |
| 现代商业 | 2000s+ | SQLServer, Snowflake, AnalyticDB | Cascades 框架 |

### 5.2 优化器两种执行方式对比

| 执行方式 | 执行顺序 | 性能特点 |
|---------|---------|---------|
| 原始执行 | JOIN → 过滤 → 聚合 | 参与 JOIN 数据量大，效率低 |
| 优化后执行 | 过滤 → JOIN → 聚合 | 参与 JOIN 数据量小，效率高 |

### 5.3 RBO vs CBO 对比

| 对比维度 | RBO（基于规则） | CBO（基于代价） |
|---------|---------------|---------------|
| 决策依据 | 预定义规则（下推、裁剪等） | 统计信息 + 代价模型 |
| 确定性 | 100% 确定规则有收益 | 通过估算评估，非 100% 确定 |
| 适用场景 | 简单等价改写 | JOIN 顺序选择、复杂查询 |
| 依赖数据 | 不需要统计数据 | 需要 NDV、Min/Max、直方图等 |
| 优化收益 | 基础优化 | 主要收益来源 |

### 5.4 常见 RBO 优化规则

| 规则类型 | 说明 | 示例 |
|---------|------|------|
| 下推（Pushdown） | 将过滤条件下推至数据源附近 | WHERE 条件下推到 JOIN 前 |
| 裁剪（Pruning） | 裁剪不需要的数据或列 | 分区裁剪、列裁剪 |
| 简化表达式 | 简化布尔表达式或算术表达式 | `1=1` 消除 |
| 解关联（Decorrelation） | 将子查询转换为 JOIN | EXISTS 转 SEMI JOIN |
| 条件推导 | 利用 JOIN 条件推导新过滤条件 | `c.id=1` 推导 `s.cls_id=1` |

### 5.5 统计信息类型与作用

| 统计信息 | 英文全称 | 作用 | 适用场景 |
|---------|---------|------|---------|
| NDV | Number of Distinct Values | 估算不同值数量，用于选择率计算 | 等值条件估算 |
| Min & Max | Minimum & Maximum | 确定数据范围 | 范围条件估算 |
| 直方图（等宽） | Histogram (Equal Width) | 应对非均匀分布 | 一般场景 |
| 直方图（等高） | Histogram (Equal Height) | 更好应对极值 | 有极端值的场景 |
| 精准直方图 | Exact Histogram | 每个值一个桶 | NDV 很小的场景 |
| 热点值 | Hot Values | 热点值单独一个桶 | 增加估算精度 |

### 5.6 直方图估算精度对比

| 估算方式 | 估计结果 | 误差率 | 说明 |
|---------|---------|-------|------|
| 基本统计信息（Min/Max/NDV） | 严重高估 | 71% 误差 | 无法应对非均匀分布 |
| 直方图 | 350（准确） | 0% | 精准反映数据分布 |

### 5.7 基数估计误差来源

| 误差来源 | 原因描述 | 影响程度 |
|---------|---------|---------|
| 估算能力不足 | 不支持直方图等高级统计信息 | 高 |
| 统计信息误差 | 近似算法、时效性问题 | 中 |
| 列间关联 | 列与列之间的联系未知 | 中 |
| 累积误差 | 多算子误差层层累积 | 高 |
| 难估计表达式 | 无常量前缀的 LIKE 等 | 低 |

### 5.8 搜索框架对比

| 框架类型 | 搜索方向 | 优点 | 缺点 | 代表系统 |
|---------|---------|------|------|---------|
| System-R 风格 | bottom-up | 简单高效 | 易陷入局部最优 | MySQL |
| Cascades | top-down | 裁剪机会多、效率高、可扩展 | 实现复杂 | SQLServer, Snowflake, AnalyticDB |

### 5.9 Cascades 核心概念

| 概念 | 英文 | 说明 |
|------|------|------|
| Expression | Expression | 算子表达式（JOIN, Filter 等） |
| GroupExpression | GroupExpression | 子节点为 Group 的表达式，减少数量 |
| Group | Group | 计划树中的点，归纳等价信息 |
| Winner | Winner | 特定请求的最优解（记忆化搜索） |
| PropertyEnforcer | PropertyEnforcer | 强插算子满足属性要求 |
| Memo | Memo | 搜索空间，存储 Group |
| RULE | Rule | Transformation / Implementation 规则 |
| TASK | Task | 驱动递归遍历树的抽象 |

### 5.10 Cascades 关键 Task 流程

| Task 名称 | 触发方式 | 主要职责 |
|----------|---------|---------|
| OptimizeGroup | 优化入口 | 对特定 Group 发起特定请求 |
| OptimizeGroupExpression | Group 优化 | 应用 transformation 和 implementation 规则 |
| ExploreGroup | 被 OptimizeGroupExpression 调用 | 探索所有等价逻辑表达式 |
| ApplyRule | 规则应用 | 根据规则类型得到新的逻辑/物理表达式 |
| OptimizeInput | 物理表达式优化 | 向子 Group 发起 OptimizeGroup 请求 |

### 5.11 分支裁剪技术

| 裁剪类型 | 原理 | 触发条件 | 效果 |
|---------|------|---------|------|
| 上界裁剪 | 累加 cost 超过上界即停止 | 首个物理表达式提供初始上界 | 减少无效搜索 |
| 下界裁剪 | Group 最小开销 + 表达式 cost > 上界 | `sum(subGroup.LB) + Expr.cost > UB` | TPC-DS 搜索空间减少近 50% |

### 5.12 误差纠正机制

| 机制 | 说明 | 触发条件 |
|------|------|---------|
| 动态采样 | 对复杂 filter 进行采样提高计划质量 | SQL 复杂程度达到阈值 |
| Adaptive Query Processing | 运行态自动调整计划 | 执行过程中发现计划不佳 |

---

## 六、技术名词对照表

| 技术名词 | 英文全称/缩写 | 说明 |
|---------|-------------|------|
| SQL | Structured Query Language | 结构化查询语言 |
| AST | Abstract Syntax Tree | 抽象语法树，SQL 解析后的树形结构 |
| Logical Plan | Logical Plan | 逻辑执行计划，不涉及物理执行细节 |
| Physical Plan | Physical Plan | 物理执行计划，包含具体执行方式 |
| RBO | Rule-Based Optimizer | 基于规则的优化器 |
| CBO | Cost-Based Optimizer | 基于代价的优化器 |
| NDV | Number of Distinct Values | 不同值的数量统计信息 |
| CE | Cardinality Estimation | 基数估计，估算算子输出行数 |
| Cost Model | Cost Model | 代价模型，估算 CPU/IO/MEM/NET 开销 |
| Cascades | Cascades Framework | 自顶向下的优化器搜索框架理论 |
| DP | Dynamic Programming | 动态规划算法 |
| Memoization Search | Memoization Search | 记忆化搜索，缓存已计算结果 |
| HashSet | Hash Set | 哈希集合，用于 HASH JOIN |
| HASH JOIN | Hash Join | 基于哈希表的 JOIN 算法 |
| INNER JOIN | Inner Join | 内连接，只返回匹配的行 |
| TPC-DS | Transaction Processing Performance Council - Decision Support | 决策支持基准测试 |
| OLTP | Online Transaction Processing | 在线事务处理 |
| Ad-hoc | Ad-hoc Query | 临时查询，非预定义的查询 |
| Adaptive Query Processing | Adaptive Query Processing | 自适应查询处理，运行时调整计划 |

---

## 七、适用场景索引

| 场景编号 | 场景描述 | 对应章节 | 相关技术 |
|---------|---------|---------|---------|
| S-001 | JOIN 前过滤条件下推 | 第二节 | RBO 下推规则 |
| S-002 | 事实表与维度表 JOIN 顺序选择 | 3.2 节 | CBO + NDV 统计信息 |
| S-003 | 非均匀数据分布导致估算偏差 | 3.2.1 节 | 直方图统计信息 |
| S-004 | 极值场景下的数据估算 | 3.2.1 节 | 等高直方图 |
| S-005 | 热点值精准估算 | 3.2.1 节 | 热点值单独桶 / 精准直方图 |
| S-006 | 复杂 filter 估算不准确 | 3.2.2 节 | 动态采样 |
| S-007 | 运行时计划不佳需调整 | 3.2.2 节 | Adaptive Query Processing |
| S-008 | 大规模查询搜索空间过大 | 3.3 节 | Cascades + 分支裁剪 |
| S-009 | TPC-DS 查询优化 | 3.3 节 | 上下界裁剪减少 50% 搜索空间 |
| S-010 | 子查询优化 | 3.1 节 | 解关联（Decorrelation） |
| S-011 | JOIN 条件推导新过滤条件 | 3.1 节 | 条件推导规则 |
