---
name: postclose-review
description: 盘后组合复盘：读个人账户快照 + 市场/评分/估值/事件，做收益归因、计划对照与数据质量对账，输出 draft 笔记。需 wukong-quant-read MCP。
version: 0.2.0
author: blueenergy
license: MIT
metadata:
  hermes:
    tags: [quant, portfolio, postclose, review, A-share, MCP, wukong]
prerequisites:
  mcp_servers: [wukong-quant-read]
  inputs: [portfolio-snapshot.json]
---

# 盘后组合复盘

对**个人账户**做收盘后复盘：先对账，再归因，最后对照已有计划。输出是一份 `status: draft` 的研究 artifact，不是交易指令。

## 触发词

| 说法 | 动作 |
|------|------|
| 盘后复盘 / 收盘复盘 / 组合复盘 | 用最近交易日跑一次 |
| 复盘 2026-10-30 | 指定交易日 |
| 为什么今天组合跌了 / 今天怎么样 | 同上，重点做归因 |

## 输入

**V0.2 首选**：`mcp__wukong-quant-read__portfolio_snapshot`

返回 `account`（total_asset / market_value / cash / synced_at）、`positions[]`（symbol / name / qty / avg_price / last_price / market_value / weight / unrealized_pnl）、`equity_history[]`、`data_quality`（含 `warnings`）。不需要本地文件。

**V0.1 回退**：本地只读导出，不写库：

```
~/trading/agent-runs/<YYYY-MM-DD>/portfolio-snapshot.json
```

必填字段：`data_as_of`、`account`、`positions[]`、`equity_history[]`、`data_quality`。

不要用 curl 直连 REST；工具不可用时报告工具名/参数/错误。

## 工具白名单

只允许 `wukong-quant-read` 的只读工具：

| 工具 | 用途 | 注意 |
|------|------|------|
| `portfolio_snapshot` | 账户/持仓/权益快照 | 先看 `data_quality.warnings`；需要 `X-User-Token` |
| `market_summary` | 当日市场总结 | 用 `analysis_date` 对齐交易日 |
| `ladder_daily` | 连板/情绪 | 只取摘要字段，不要整份塞进上下文 |
| `market_events_weekly` | 未来事件 | 只取 importance ≥ 3 |
| `stock_score_detail` | 六维评分 | **代码必须带后缀**（`600114.SH`） |
| `stock_valuation_metrics` | 估值/财务 | 先看 `financial.indicator_period` 是否最新 |
| `stock_industry_comparison` | 同业分位 | SW L2 可能过宽，只作粗参照 |
| `earnings_signals` | 业绩预告/快报 | 0 条也是结论 |
| `sector_latest` | 板块缓存 | **先看 `trade_date`；陈旧就不用** |
| `user_watchlist_stocks` | 自选股 | 401 = token 过期，提示重新 `user_login` |

**禁止**：`execute_code`、`terminal`、`curl` 直连、写库、下单、改计划、改评分。

## 流程

1. **Preflight**：`date` 确认今天/最近交易日；快照是否存在且 `data_as_of` 对齐；MCP 3002 是否可达；个人化工具是否 401。
2. **读快照**：优先 `portfolio_snapshot`，失败再退回本地 JSON；校验持仓数、权重和 ≈ 1、现金、`synced_at`；`total_asset=0` 时用 `market_value + cash` 并标注；逐条读 `data_quality.warnings`。
3. **读 vault**：对每只持仓 grep 现有笔记与计划（`holdings/`、`research/`、`closed-lines`），把计划 ID/链接写进 artifact。
4. **取数**：`market_summary`、`ladder_daily`、`market_events_weekly`；逐只 `stock_score_detail`；主仓 `stock_valuation_metrics` + `stock_industry_comparison`；可选 `earnings_signals`。
5. **对账**：
   - 用 `volume_price` 的上一交易日收盘算个股当日盈亏与贡献，不要用 `trader_account_equity` 的行情直接当收盘。
   - `updated_at` 是 **UTC**，不是 CST；08:00 UTC = 16:00 CST。检查 `source_synced_at` / `source_synced_age_seconds`：source sync 在收盘前，这一行就不是收盘终值。
   - 检查 `equity_history` 每行的 `updated_at`、`is_stale`；盘前/盘中行不能当收盘。
   - 检查 `sector_latest.trade_date` 是否落后；落后 > 2 个交易日即标为 unusable。
   - 检查 `stock_valuation_metrics.financial.indicator_period`；H1 数据不能当 Q3。
   - 检查 `forward_pe` 的一致预期日期；财报后未更新的 consensus 不能当估值支撑。
6. **计划对照**：若 vault 有减仓/持有计划，逐条对照目标权重、触发价、已执行/未执行；没有计划就写明“无既有计划”。
7. **写 artifact**：固定输出段，`status: draft`。

## 输出契约

```markdown
---
run_id:
workflow: postclose-review
version: 0.1
status: draft
data_as_of:
generated_at:
account:
snapshot:
tools: []
cost_usd:
---
# 结论（≤3 条）
# 组合表现与归因
| 标的 | 股数 | 成本 | 收盘 | 市值 | 权重 | 当日盈亏 | 贡献 |
# 市场/行业背景
# 个股要点
# 计划对照与下一交易日
# 数据质量与缺口
# 反证：什么会改变结论
# 待确认
# 证据表
| claim | value | source tool/file | timestamp |
```

## 硬约束

1. **每个数字都要有来源**（工具名/文件/时间戳）；没有就写“未取证”，不要估算后当事实。
2. **陈旧数据不用**：`sector_latest` 落后 > 2 个交易日、`equity_history` 的 `source_synced_at` 在收盘前、`financial` 非最新报告期，都必须显式标注。`updated_at` 是 UTC，不要按 CST 读。
3. **不臆造因果**：行业 beta、个股 alpha、资金流解释要分开；推测必须标“推测”。
4. **没有新信息就写没有**：如果只是复述 16:30 市场总结或昨天的持仓笔记，明确写“本次无新增 material 信息”。
5. **不写交易指令**：只对照既有计划；新建议最多写到“观察点/待确认”。
6. **不写生产库、不下单、不改计划**；只写 `~/trading/agent-runs/` 下的 draft。
7. **状态只能由人推进**：agent 永远写 `draft`；`reviewed/accepted` 由用户改。
8. **工具失败要报告**：工具名、参数、错误原文；不要 fallback 到未声明的 REST。

## 数据质量清单（每次必查）

- [ ] `sector_latest.trade_date` 是否 ≥ `data_as_of - 2` 个交易日
- [ ] `equity_history` 最新行 `updated_at`（UTC）换算 CST 是否在收盘后；`source_synced_at` / `source_synced_age_seconds` 是否在收盘后；`is_stale`
- [ ] `equity_history` 最新行 total 与 account 快照差异 > 0.5% 时标警（16:00 初值与最终 sync 的差）
- [ ] `stock_valuation_metrics.financial.indicator_period` 是否最新报告期
- [ ] `forward_pe` 的 consensus 是否在最新财报后更新
- [ ] `stock_score_detail` 是否带 `.SH/.SZ`
- [ ] 个人化工具是否 401
- [ ] `earnings_signals` 是否 0 条
- [ ] `financial_mainbz` 是否缺失（影响分部分析）

## V0.2 工具

`portfolio_snapshot` 已上线（`quantFinance` `8addeb5`）：

- 输入用 `mcp__wukong-quant-read__portfolio_snapshot`，不再依赖本地 JSON
- 返回 `account` + `positions` + `equity_history` + `data_as_of` + `data_quality`
- 工具已内置：`total_asset` 为 0 时按 `market_value + cash` 计算；账户同步过期；`trader_account_equity` 与账户快照差异 > 0.5%；缺失名称（含 ETF 回退 `etf_basic` / `fund_basic`）
- 调度侧新增：16:10 板块/概念分析、21:30 收盘终值权益快照

## 关联

- [[projects/trading/holdings/600114-reduction-plan-20261030]]
- [[projects/trading/holdings/000338-000951-truck-sleeve-20261030]]
- [[projects/trading/holdings/stock-600114-dongmu-20261026]]
- `wukong-skills/portfolio-sweep-review`（同类“只读 + 固定契约”范式）
