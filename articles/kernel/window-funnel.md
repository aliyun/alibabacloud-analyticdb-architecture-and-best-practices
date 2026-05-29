<!--
title: "10 倍性能提升：一文读懂 AnalyticDB 秒级漏斗分析函数"
date: "2025"
author: "乔鹏"
tags: ["AnalyticDB MySQL", "window_funnel", "漏斗分析", "营销分析", "用户行为", "AARRR", "OLAP"]
category: "架构"
doc_version: "2.0"
last_updated: "2026-05-24"
machine_readable: true
-->

# 10 倍性能提升：一文读懂 AnalyticDB 秒级漏斗分析函数

> 营销域中的洞察分析、智能圈人、经营报表等场景是 OLAP 分析型数据库的重要应用场景。[AnalyticDB MySQL](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/product-overview/what-is-analyticdb-for-mysql) 通过 window_funnel 函数实现了高效的漏斗分析，相比传统 SQL 实现有 10 倍以上的性能提升，查询性能不会随着漏斗层级的加深而变差。

---

## 一、业务挑战

营销域中的洞察分析/智能圈人/经营报表等场景是 OLAP 分析型数据库的重要应用场景。云原生数据仓库 [AnalyticDB MySQL](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/product-overview/what-is-analyticdb-for-mysql) 在淘宝、饿了么、菜鸟、优酷、盒马等业务的营销场景有比较长时间的积累和沉淀。

对于营销域的业务运营同学而言，"增长黑客"理论是一个耳熟能详的概念。运营同学一定希望当增长处于 AARRR 不同阶段时可以采取一定的措施和试验，来优化转化路径，挽回流失客户。这对数据产品的功能需求就是需要准确地计算出每个转化阶段的用户行为数据，也就是每个阶段的漏斗转化。另外，性能需求当然是越快越好了，毕竟谁也无法忍受前台 UI 一直处于 loading 状态。

---

## 二、技术挑战

过去数据库产品本身通常不会关心某一个具体业务场景如何实现，通常只会提供标准 SQL 语义的能力。假设我们有一份用户行为数据，包含"谁在什么地方做了什么"的全部信息，用户行为数据表 user_behavior 如下：

| 字段 | 说明 | 示例值 |
|---|---|---|
| uid | 用户 ID | 用户唯一标识 |
| ts | 时间戳 | 事件发生时间 |
| item_id | 商品 ID | 商品唯一标识 |
| event_type | 事件类型 | pv/fav/cart/buy |

用户行为类型共有四种：

| 事件类型 | 含义 | 说明 |
|---|---|---|
| pv | 页面浏览 | Page View，用户浏览商品页面 |
| fav | 收藏 | Favorite，用户收藏商品 |
| cart | 加购物车 | Add to Cart，用户将商品加入购物车 |
| buy | 购买 | Purchase，用户完成购买 |

### 2.1 粗粒度漏斗

第一种是放在数据报表首页给决策层看的，只关注每个事件的统计数据：

```sql
select  event_type,  count(distinct uid)
from  user_behavior
where  item_id = 3838928  and ts >= 1511540732  and ts <= 1512312625
group by  event_type
```

这种漏斗只能展示事件的粗粒度统计信息，无法分析出事件前后的因果关系和行为路径。

### 2.2 细粒度序列匹配漏斗

在实际业务中，更多的是需要满足非连续子序列匹配，比如 "pv...fav...cart...buy"：

| 漏斗层级 | 事件序列 | 含义 |
|---|---|---|
| Level 1 | pv | 浏览商品 |
| Level 2 | pv -> fav | 浏览后收藏 |
| Level 3 | pv -> fav -> cart | 浏览、收藏后加购物车 |
| Level 4 | pv -> fav -> cart -> buy | 完整转化路径 |

然而，在数据库产品中通常不会提供这种子序列匹配功能的聚合函数。一个可能的解法是通过字符串匹配函数来实现：

```sql
/* 将符合条件的数据转成事件标志 */
with t1 as (
  select uid, ts,
    case event_type
      when 'pv' then 'e1'
      when 'fav' then 'e2'
      when 'cart' then 'e3'
      when 'buy' then 'e4'
      else 'ex'
    end as event_code
  from user_behavior
)
/* 统计每个层级的用户数 */
select level, count(distinct uid)
from (
  select uid,
    case
      when event_lst like '%e1%e2%e3%e4%' then 'level_4'
      when event_lst like '%e1%e2%e3%' then 'level_3'
      when event_lst like '%e1%e2%' then 'level_2'
      when event_lst like '%e1%' then 'level_1'
      else 'level_0'
    end as level
  from (
    /* 将用户的事件聚合成事件序列 */
    SELECT uid,
      GROUP_CONCAT(event_code order by ts asc) as event_lst
    from t1
    group by uid
  )
)
group by level
```

**以上的实现涉及几个性能瓶颈：**

1. **组内排序聚合**：`GROUP_CONCAT(event_code order by ts asc)`，由于真实业务场景中存在干扰数据（如刷单用户有很多异常事件），导致计算量巨大
2. **字符串模糊匹配**：`case event_lst like '%e1%e2%e3%e4%'`，这一步会成为 CPU 的消耗大户，当匹配层级大于 5 个之后会极大影响查询性能
3. **类型转换、排序、分组**：整个计算过程中的这些操作也会极大降低执行效率

此外，这种实现 SQL 异常复杂，还没有结合其他用户属性做关联查询，扩展能力很有限。

---

## 三、AnalyticDB MySQL 优化方法

针对上述漏斗场景的痛点问题，AnalyticDB MySQL 针对性地引入了 **window_funnel** 函数。

### 3.1 函数说明

漏斗函数（window_funnel）可以搜索滑动时间窗口中的事件列表，并计算条件匹配的事件列表的最大长度。搜索事件列表，从第一个事件开始匹配，依次做最长、有序匹配，返回匹配的最大长度。一旦匹配失败，结束整个匹配。

假设在窗口足够大的条件下：
- 条件事件为 c1、c2、c3，而用户数据为 c1、c2、c3、c4，最终匹配到 c1、c2、c3，函数返回值为 3
- 条件事件为 c1、c2、c3，而用户数据为 c4、c3、c2、c1，最终匹配到 c1，函数返回值为 1
- 条件事件为 c1、c2、c3，而用户数据为 c4、c3，最终没有匹配到事件，函数返回值为 0

### 3.2 函数语法

```
window_funnel(window, mode, timestamp, cond1, cond2, ..., condN)
```

### 3.3 参数说明

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| window | integer | 是 | 滑动时间窗口大小（单位：毫秒） |
| mode | string | 是 | 匹配模式，如 'default' |
| timestamp | timestamp | 是 | 事件时间戳字段 |
| cond1...condN | condition | 是 | 事件条件表达式，按漏斗层级顺序排列 |

### 3.4 实现漏斗计算

基于 window_funnel 函数，实现漏斗计算逻辑的 SQL 如下：

```sql
select funnel_step, count(1)
from (
  /* 直接计算每个用户满足的行为序列 */
  select uid,
    window_funnel(
      cast(86400000 /* 滑动窗口大小，可根据业务配置 */ as integer),
      'default',
      ts,
      event_type = 'pv',
      event_type = 'fav',
      event_type = 'cart',
      event_type = 'buy'
    ) as funnel_step
  from user_behavior
  group by uid
)
group by funnel_step;
```

### 3.5 优势对比

相比使用标准 SQL 实现的方式：

1. **简化 SQL 逻辑**：window_funnel 函数将所有计算逻辑都封装到一个聚合函数中，极大简化 SQL 逻辑，降低业务实现复杂程度，利于代码维护和扩展
2. **滑动窗口支持**：支持滑动窗口设置，统计在时间窗口内满足行为序列的用户，用户可以灵活设置窗口大小，而标准 SQL 方式难以实现相同语义
3. **性能大幅提升**：对 user_behavior 表只有一次 group by，移除了分组、排序、聚合、类型转换、字符匹配等耗时操作

在相同实例下，两种实现方式的性能（执行时间）对比：

| 实现方式 | 性能特征 | 说明 |
|---|---|---|
| 传统 SQL（GROUP_CONCAT + LIKE） | 慢 | 组内排序聚合 + 字符串模糊匹配，CPU 消耗大 |
| window_funnel 函数 | 快 | 仅一次 group by，无排序/聚合/类型转换/字符匹配 |

> **测试环境**：C 系列（高性能版），4 组 worker 96 core
> **测试数据**：可通过 [天池数据集](https://tianchi.aliyun.com/dataset/649) 下载
> **测试结果**：可在 [AnalyticDB 产品页](https://www.aliyun.com/product/ApsaraDB/ads) 购买实例复现

---

## 四、总结

[AnalyticDB MySQL](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/product-overview/what-is-analyticdb-for-mysql) 内置的 window_funnel 函数可以实现高效的漏斗计算。相比于传统 SQL 的实现方式：

- **降低 SQL 复杂度**：所有逻辑封装在一个聚合函数中
- **更丰富的滑动窗口语义**：支持灵活配置窗口大小
- **更好的查询性能**：查询性能不会随着漏斗层级的加深而变差
- **10 倍以上性能提升**：对于漏斗层级很深的场景有 10 倍以上的性能提升，对最终用户而言无需等待，"立刻"就能得到要分析的结果

### 4.1 两种实现方式对比

| 对比维度 | 传统 SQL 实现 | window_funnel 函数 |
|---|---|---|
| SQL 复杂度 | 高（多层嵌套 + GROUP_CONCAT + LIKE） | 低（单一函数调用） |
| 滑动窗口 | 难以实现 | 原生支持 |
| 性能瓶颈 | 组内排序、字符串模糊匹配、类型转换 | 仅一次 group by |
| 层级加深影响 | 性能急剧下降 | 性能基本不变 |
| 性能提升 | 基准 | 10 倍以上 |

### 4.2 window_funnel 匹配示例

| 条件事件 | 用户数据 | 匹配结果 | 返回值 |
|---|---|---|---|
| c1, c2, c3 | c1, c2, c3, c4 | 匹配 c1, c2, c3 | 3 |
| c1, c2, c3 | c4, c3, c2, c1 | 仅匹配 c1 | 1 |
| c1, c2, c3 | c4, c3 | 无匹配 | 0 |

### 4.3 传统 SQL 实现的性能瓶颈

| 瓶颈 | 操作 | 影响 |
|---|---|---|
| 组内排序聚合 | GROUP_CONCAT(event_code order by event_time asc) | 干扰数据（如刷单用户）导致计算量巨大 |
| 字符串模糊匹配 | event_lst like '%e1%e2%e3%e4%' | CPU 消耗大户，层级 > 5 后严重影响性能 |
| 类型转换/排序/分组 | 整个计算过程中的辅助操作 | 降低执行效率 |

---

## 五、技术名词对照表

| 术语 | 英文全称 | 说明 |
|---|---|---|
| AnalyticDB MySQL | AnalyticDB for MySQL | 阿里云云原生数据仓库产品 |
| window_funnel | Window Funnel Function | 窗口漏斗分析函数，用于滑动时间窗口中的事件序列匹配 |
| OLAP | Online Analytical Processing | 联机分析处理，用于复杂数据查询和分析 |
| AARRR | Acquisition, Activation, Retention, Revenue, Referral | 增长黑客模型，用户生命周期五个阶段 |
| pv | Page View | 页面浏览事件 |
| fav | Favorite | 收藏事件 |
| cart | Add to Cart | 加购物车事件 |
| buy | Purchase | 购买事件 |
| GROUP_CONCAT | GROUP_CONCAT | MySQL 聚合函数，将组内字符串连接 |
| 滑动窗口 | Sliding Window | 在时间轴上滑动的固定大小窗口 |
| 漏斗分析 | Funnel Analysis | 分析用户在多步骤转化流程中的流失情况 |

---

## 六、适用场景索引

| 场景 | 相关章节 | 关键技术 |
|---|---|---|
| 营销域洞察分析 | 一 | window_funnel 漏斗函数 |
| 智能圈人 | 一 | 用户行为序列匹配 |
| 经营报表 | 一、三 | 漏斗转化率统计 |
| AARRR 增长分析 | 一 | 多阶段漏斗转化计算 |
| [实时数据分析](https://help.aliyun.com/zh/analyticdb/analyticdb-for-mysql/use-cases/real-time-data-analysis/) | 三 | window_funnel 秒级响应 |
