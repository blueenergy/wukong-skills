---
name: stock-screening
description: Agent-framework agnostic stock screening via ScreenSpec and MCP tools stock_screen_schema / run_stock_screen.
version: 1.0.0
author: blueenergy
license: MIT
metadata:
  hermes:
    tags: [quant, stock, screening, MCP]
prerequisites:
  mcp_servers: [wukong-quant-read]
---

# 通用选股（ScreenSpec）

通过 MCP 工具执行**确定性**选股。Agent 只负责把自然语言翻译成 `ScreenSpec` JSON；**不得**自行计算振幅/涨跌幅或访问数据库。

## 必须先调用的工具

| 工具 (operation_id) | 用途 |
|---|---|
| `stock_screen_schema` | 合法指标、操作符、股票池 preset |
| `stock_screen_presets` | 版本化 universe 列表 |
| `run_stock_screen` | 执行 ScreenSpec（POST，body 即 ScreenSpec JSON） |

## 鉴权

- `X-Quant-Client-Key` 或兼容头 `X-Hermes-Quant-Key`
- 个人化写入操作不在本 skill 范围

## ScreenSpec 要点

- `universe.type`: `preset` | `sw_l2` | `manual` | `all_financial`
- 价格指标需 `start` + `end`（`latest` 可解析为最新交易日），`adjust` 默认 `hfq`
- 财务指标需 `period`（`latest` 或 YYYYMMDD），`report_type` 默认 1（累计）
- 百分数用小数：`35%` → `0.35`
- 过滤 `op`: `gt` | `gte` | `lt` | `lte`
- 不支持未在 schema 注册的字段；遇到报错应提示用户改条件
- `all_financial` 只能配财务指标；价格指标必须指定 `preset` / `sw_l2` / `manual` 股票池
- 省略 `sort` 时：有价格指标按 `range_amplitude` 降序；否则按第一个同比过滤字段降序

## 价格指标（provider: price）

| 字段 | 含义 |
|---|---|
| `range_amplitude` | 区间振幅 `(max_high - min_low) / min_low` |
| `peak_gain` | 前期最大涨幅（相对区间首日收盘） |
| `drawdown_from_peak` | 高点至今回撤（相对区间最高价） |
| `return_pct` | 区间涨跌幅 |

**注意**：振幅大不等于前期强势；筛「强势超跌」应同时看 `peak_gain` 与 `drawdown_from_peak`。

## 财务数据来源分层（provider: financial）

财报季早期正式财报很少（如 2026H1 在 8 月初只有 22 家）。缺正式财报的个股会按优先级回退，
每股只用一个来源，同比基数一律取去年的**正式财报**：

| `data_source` | 来源 | 可用字段 | 可靠性 |
|---|---|---|---|
| `formal` | 正式财报 | 全部 7 个字段 | 高 |
| `express` | 业绩快报 | `revenue` `operate_profit` `total_profit` `n_income_attr_p` | 中，未审计预披露值，与最终财报可差数个百分点 |
| `forecast` | 业绩预告 | 仅 `n_income_attr_p`（区间中值） | 低，仅供参考 |

- 开关：`include_express` / `include_forecast`，默认均为 `true`；`report_type=2`（单季）时自动不启用
- 按 `total_revenue` / `oper_cost` 筛选时，快报和预告都无法供数，这些个股会落选
- 每行带 `data_source` 与 `data_source_ann_date`，`summary.sources_distribution` 给出条数分布
- **引用 `express` / `forecast` 的数字时必须注明是预披露值**，不要说成「已披露财报显示」

## 同比可信度标注（yoy_quality）

扭亏和低基数个股本身是有价值的信号，**不会被剔除**，而是逐字段标注。每个 `{field}_yoy`
都附带 `{field}_yoy_base`（去年同期金额）和 `{field}_yoy_quality`：

| `yoy_quality` | 含义 | 播报方式 |
|---|---|---|
| `normal` | 去年同期为正、增速在合理量级 | 正常说增速 |
| `turnaround` | 去年同期 <=0、今年为正 | 说「**扭亏为盈**」并给出两期金额，不要念百分比 |
| `loss` | 两期均 <=0 | 说「**亏损收窄/扩大**」，正的同比不代表盈利 |
| `low_base` | 去年同期为正但极小，同比畸高 | 说「**低基数**」并给出两期金额 |

- `summary.yoy_quality_distribution` 按严重程度从高到低给出条数分布，**必须在总结里提及**
- 阈值：`low_base_yoy_threshold` 默认 10.0（即 |同比| >= 1000% 判为低基数）
- 若用户明确只要「真实增长」，可传 `exclude_negative_base: true` 剔除去年同期 <=0 的个股
- 排序不会因标注降权，所以榜首常是扭亏/低基数股——这正是必须逐条标注的原因

## 用例 1：科技硬件强势超跌（按振幅排序）

```json
{
  "universe": {"type": "preset", "value": "tech_hardware_v1"},
  "start": "20260401",
  "end": "latest",
  "adjust": "hfq",
  "exclude_st": true,
  "sort": [
    {"field": "range_amplitude", "order": "desc"},
    {"field": "peak_gain", "order": "desc"}
  ],
  "top_n": 100
}
```

可选过滤超跌：`{"field": "drawdown_from_peak", "op": "lte", "value": -0.2}`

## 用例 2：最新财报营收同比高增长（财务）

```json
{
  "universe": {"type": "all_financial", "value": "all"},
  "period": "latest",
  "report_type": 1,
  "filters": [
    {"field": "revenue", "metric_type": "yoy", "op": "gt", "value": 0.30}
  ],
  "sort": [{"field": "revenue_yoy", "order": "desc"}],
  "top_n": 50
}
```

## 用例 3：多条件财务（与 legacy `stock_screen` 等价）

```json
{
  "universe": {"type": "all_financial", "value": "all"},
  "period": "20260331",
  "report_type": 1,
  "filters": [
    {"field": "operate_profit", "metric_type": "yoy", "op": "gt", "value": 0.35},
    {"field": "revenue", "metric_type": "yoy", "op": "gt", "value": 0.30},
    {"field": "operate_profit", "metric_type": "abs", "op": "gt", "value": 100000000, "base_period": "20240331"}
  ],
  "top_n": 100
}
```

## 输出约定

- 先总结 `matched`（过滤并剔除 ST 后的命中数）/ `returned` 与 `summary.industries_distribution`
- 财务筛选还要报 `summary.sources_distribution` 与 `summary.yoy_quality_distribution`，
  说明多少条来自快报/预告、多少条是扭亏或低基数
- 缺行情或缺财报的标的会被剔除，不会以空指标行返回；结果为空即条件无匹配
- 列出前 20 条；超过则说明截断
- 展示 `window.as_of` / `start_trade_date` / `end_trade_date`（价格类）
- 不要编造未返回的数值

## MCP 端点（本地示例）

| Server | URL |
|---|---|
| wukong-quant-read | `http://localhost:3002/api/mcp` |

请求头：`X-Quant-Client-Key` 或 `X-Hermes-Quant-Key`
