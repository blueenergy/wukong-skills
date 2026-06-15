---
name: wukong-quant
description: 通过悟空量化 MCP 接口调用 A 股深度分析、连板天梯等功能。需配置 wukong-quant-read 和 wukong-quant-actions MCP 服务器。
version: 1.0.0
author: blueenergy
license: MIT
metadata:
  hermes:
    tags: [quant, stock, A-share, MCP, wukong]
prerequisites:
  mcp_servers: [wukong-quant-read, wukong-quant-actions]
---

# 悟空量化

通过 wukong-quant MCP 工具访问 A 股量化数据。

## 触发词

| 说法 | 动作 |
|------|------|
| 悟空，深度分析 {股票代码} | 提交深度分析任务并等待结果 |
| 悟空，今日天梯 | 拉取连板天梯数据 |
| 悟空，今日涨停分析 / 悟空，涨停全景 | 获取涨停全景叙事（LLM 生成，盘后 19:00 后可用） |
| 悟空，今日市场总结 / 悟空，大盘分析 | 获取 AI 大盘分析（情绪/板块/资金/风险，交易日三次自动更新） |
| 悟空，全球市场 / 悟空，外盘简报 | 获取全球市场简报（美股/港股/外汇/大宗/宏观联动） |
| 悟空，热股分析 / 悟空，今日热股 | 获取当日最新热股榜 AI 分析（同花顺/东方财富） |
| 悟空，宏观分析 / 悟空，中国宏观 | 获取中国宏观 AI 深度分析（PMI/CPI/PPI/M2/GDP/LPR） |
| 悟空，美国宏观 / 悟空，美联储 | 获取美国利率市场深度分析 |
| 悟空，板块分析 / 悟空，今日概念 | 获取最新板块/概念 AI 分析（盘中热门方向点评） |
| 悟空，财报信号 / 悟空，业绩异动 | 扫描近期业绩预告/快报/券商评级上调信号 |
| 悟空，评分排行 / 悟空，{hs300/a500} 评分 | 查询指定指数评分排行榜 Top N |
| 悟空，查 {股票代码} 的评分 | 查询单股六维量化评分详情 |
| 悟空，智能选股 / 财务筛选 {条件} | 多条件财务筛选（营业利润/营收同比、绝对值门槛）附申万行业 |
| 悟空，2026Q1 外资投行进入前十大流通股东 | 查询指定报告期高盛/摩根士丹利/瑞银/Barclays 等进入前十大流通股东的股票 |
| 悟空，查 {股票代码} 的历史分析 | 查最近历史分析 |
| 悟空，登录 / 悟空，我要登录 | 用户登录，获取 JWT token 并提示如何配置 |
| 悟空，我的账户 / 悟空，我的信息 | 查看当前登录用户账户信息 |
| 悟空，我的自选股 / 悟空，自选股评分 | 获取自选股列表 + 最新量化评分排行 |
| 悟空，自选股加 {代码} / 添加到自选 | 将指定股票添加到自选股 |
| 悟空，自选股删 {代码} / 移除自选 | 从自选股中删除指定股票 |
| 悟空，自选股历史 / 我的分析记录 | 查询自选股历史深度分析记录 |

## 深度分析工作流

当用户说 **"悟空，深度分析 {symbol}"**：

1. 调用 `mcp_wukong_quant_actions_deep_analysis`，参数 `symbol={symbol}`（服务端自动等待完成，通常 60-180 秒）
2. `success=true` 时，**原文完整输出**以下字段，不得摘要压缩：

   **核心结论**
   - `analysis.investment_advice` — 投资建议（buy/hold/sell）
   - `analysis.risk_level` — 风险等级
   - `analysis.confidence_score` — 信心评分
   - `analysis.key_points` — 关键结论列表

   **价格与技术面**
   - `analysis.technical_analysis` — 技术面分析
   - `analysis.support_level` — 支撑位
   - `analysis.resistance_level` — 阻力位
   - `analysis.risk_price_levels` — 风险价位详情（止损/止盈参考价）

   **筹码与量化**
   - `analysis.chip_analysis` — 筹码分析
   - `analysis.quant_score_cross_check` — 量化评分交叉验证

   **展望**
   - `analysis.short_term_forecast` — 短期预测
   - `analysis.mid_term_forecast` — 中期预测
   - `analysis.long_term_forecast` — 长期预测

   **股权与风控**
   - `analysis.equity_risk_user_visible` — 股权质押与股东结构风控摘要

   **修订信息**（若有）
   - `analysis.revision_log` — 评审修订记录

   若某字段不存在或为空，跳过该字段；若所有字段均不存在，将 `analysis` 完整 JSON 原样展示
3. `success=false` 时，输出 `error` 字段；若 `status=timeout`，告知用户可用 `task_id` 继续查询

## 登录与个人化功能

### 登录流程

当用户说 **"悟空，登录"** 或 **"悟空，我要登录"**：

1. 询问用户用户名和密码
2. 调用 `mcp_wukong_quant_actions_user_login`，参数 `username={用户名}` `password={密码}`
3. 成功后展示 `token` 字段，并输出以下提示：

   > 登录成功！请将以下 token 填入 `.vscode/mcp.json` 的 `wukong-quant-read`
   > 和 `wukong-quant-actions` 服务器 headers 中：
   > ```
   > "X-User-Token": "<token>"
   > ```
   > 保存后执行 **Ctrl+Shift+P → Developer: Reload Window** 使配置生效。
   > token 有效期 8 小时，到期后重新登录即可。

4. token 配置生效后，即可使用 `user_profile`、`user_watchlist_stocks`、`user_watchlist_add`、`user_watchlist_history` 等个人化工具

### 个人化工具

- **我的账户**：调用 `mcp_wukong_quant_read_user_profile`（需 X-User-Token）
  - 返回：`username`、`email`、`full_name`、`service_level`、`created_at`、`last_login`

- **我的自选股评分**：调用 `mcp_wukong_quant_read_user_watchlist_stocks`（可选参数 `strategy`）
  - 返回自选股列表 + 每只股票最新六维量化评分，按综合分降序排列
  - 若自选股为空，提示用户可使用 `user_watchlist_add` 工具添加

- **添加自选股**：调用 `mcp_wukong_quant_actions_user_watchlist_add`，参数 `symbol={股票代码}`

- **删除自选股**：调用 `mcp_wukong_quant_actions_user_watchlist_remove`，参数 `symbol={股票代码}`

- **自选股历史分析**：调用 `mcp_wukong_quant_read_user_watchlist_history`（可选参数 `symbol`、`limit`）
  - 不指定 symbol 则返回所有自选股最近分析记录，指定则筛选单只

> **注意**：以上个人化工具均需在 MCP headers 中设置 `X-User-Token`。若未设置或 token 已过期，工具会返回 401 错误并提示重新登录。

- **查历史**：调用 `mcp_wukong_quant_read_analysis_history`，参数 `symbol`
- **连板天梯**：调用 `mcp_wukong_quant_read_ladder_daily`
- **涨停分析**：调用 `mcp_wukong_quant_read_ladder_narrative`（可选参数 `date=YYYYMMDD`，默认最新）
  - 返回字段：`headline`（标题）、`sentiment_signal`（情绪信号）、`narrative_markdown`（完整 Markdown 叙事，原文输出）
- **AI 大盘分析**：调用 `mcp_wukong_quant_read_market_summary`（可选参数 `date=YYYY-MM-DD`，默认最新交易日）
  - 交易日 10:00 / 11:30 / 15:30 自动更新；读缓存，不触发 LLM
- **全球市场简报**：调用 `mcp_wukong_quant_read_global_market`（无需参数，自动取最新）
  - 包含：美股、港股、外汇、大宗商品、宏观联动分析、A 股启示；原文完整输出 `analysis` 字段
- **热股分析**：调用 `mcp_wukong_quant_read_hot_stock`（可选参数 `source=ths`（默认）或 `source=dc`）
  - 返回：`analysis`（AI 全文点评）、`rank_time`（榜单时刻）、`stock_count`、`trade_date`
  - 原文完整输出 `analysis` 内容，不得压缩
- **宏观 AI 分析**：调用 `mcp_wukong_quant_read_macro_analysis`（可选参数 `scope=china_macro`（默认）或 `scope=us_macro`）
  - 返回最新宏观分析结果，读缓存，不触发 LLM
  - 原文完整输出分析正文，不得压缩
- **板块/概念 AI 分析**：调用 `mcp_wukong_quant_read_sector_latest`（可选参数 `source=dc`（默认）或 `source=ths`）
  - 返回当日热门板块/概念消内容要和 AI 点评，读缓存，不触发 LLM
  - 原文完整输出 `analysis` 内容，不得压缩
- **财报异动信号**：调用 `mcp_wukong_quant_read_earnings_signals`（可选参数 `days=7`，默认扫描最近7天）
  - 返回 `signals` 列表，每条包含：`ts_code`、`signal_type`（业绩预告/快报/卖方研报）、`core_metric`、`date`、`reason`
  - 按日期倒序输出，如有多条可选择重点展示
- **股票评分排行榜**：调用 `mcp_wukong_quant_read_stock_ranking`
  - 参数：`index_code`（hs300/a500/csi500/csi1000/star50，默认 hs300）、`top`（取前 N 名，默认 20）、`strategy`（balanced/aggressive/conservative）
  - 返回 `ranking` 列表，每条包含：`rank`、`symbol`、`name`、`industry`、`score`、六维子评分
  - 原文完整输出前 20 名，包含六维评分
- **单股量化评分详情**：调用 `mcp_wukong_quant_read_stock_score_detail`，参数 `symbol`
  - 返回：`composite_score`（三策略得分）、`cycle/value/fundamental/growth/technical/money_flow_score`（六维子分）
- **外资投行前十大流通股东筛选**：调用 `mcp_wukong_quant_read_foreign_top10_floatholders`
  - 参数：`period`（报告期 YYYYMMDD；2026Q1 → `20260331`）、`holders`（可选，逗号分隔；默认高盛/摩根士丹利/瑞银/Barclays）、`new_only`（可选，是否仅看较上一期新进）、`previous_period`（可选）
  - 用户说“进入前十大流通股东”时默认 `new_only=false`；用户明确说“新进”时传 `new_only=true`
  - 返回 `results`，每条包含：`ts_code`、`name`、`industry`、`holder_name`、`norm_label`、`holder_rank`、`hold_amount`、`hold_ratio`、`ann_date`、`end_date`
  - 输出时先按 `by_holder` 总结各机构命中数量，再列出股票；结果很多时展示前 30 条并说明 `matched_count`
- **智能选股（多条件财务筛选）**：调用 `mcp_wukong_quant_read_stock_screen`
  - 参数：`period`（YYYYMMDD 或 `latest`）、`filters`（JSON 字符串）、`report_type`（1累计/2单季，默认1）、`include_sw_industry`、`exclude_st`、`exclude_negative_base`、`sort_by`、`top_n`
  - 字段白名单：`operate_profit`（营业利润）、`revenue`（主营业务收入，默认）、`total_revenue`（营业总收入）、`n_income`（净利润）、`n_income_attr_p`（归母净利润）、`total_profit`（利润总额）、`oper_cost`（营业成本）
  - 返回：`matched`（命中总数）、`summary.industries_distribution`（申万 L1 分布）、`results[]`（`ts_code`/`name`/`sw_l1`/`sw_l2`/`sw_l3` + 每个 filter 字段的当期值与 YoY）
  - 输出时先报行业分布总结，再列出名单，名单超过 20 只可截断并说明

### LLM 参数翻译指南（stock_screen）

自然语言 → `filters` JSON 时必须遵守：

- **百分数 → 小数**：“同比超 35%” → `value: 0.35`，不是 `35`
- **期次**：2026Q1 → `20260331`；H1/中报 → `20260630`；Q3 → `20260930`；年报 → `20261231`；“最新业绩” → `period="latest"`
- **累计 vs 单季**：默认 `report_type=1`（行业惯例累计同比）；用户明说“单季”才传 `2`
- **营收字段**：默认 `revenue`；用户明说“营业总收入”才用 `total_revenue`
- **比较符**：“超过”/“大于” → `gt`；“不低于”/“至少” → `gte`；“小于” → `lt`；“不高于” → `lte`
- **主要金额单位**：营收/利润中“一亿”=`100000000`，“五千万”=`50000000`；不要输入带单位的字符串

#### Few-shot 示例

**例 1**：“找 2026 一季报 营业利润同比 超 35%、营收同比 超 30%，同时 2024 一季报 营业利润 大于 1 亿 的股票”
```
period=20260331
report_type=1
filters=[
  {"field":"operate_profit","metric_type":"yoy","op":"gt","value":0.35},
  {"field":"revenue","metric_type":"yoy","op":"gt","value":0.30},
  {"field":"operate_profit","metric_type":"abs","op":"gt","value":100000000,"base_period":"20240331"}
]
```

**例 2**：“最新一期 归母净利润同比 不低于 50% 的 A 股”
```
period=latest
filters=[
  {"field":"n_income_attr_p","metric_type":"yoy","op":"gte","value":0.50}
]
```

**例 3**：“2025 年报 营业总收入 超 100 亿、净利润同比 大于 20% 的股票”
```
period=20251231
report_type=1
filters=[
  {"field":"total_revenue","metric_type":"abs","op":"gt","value":10000000000},
  {"field":"n_income","metric_type":"yoy","op":"gt","value":0.20}
]
```

## MCP 服务器要求

本技能依赖两个 MCP 服务器：

- `wukong-quant-read`：只读工具，查询行情、天梯、市场分析、评分、自选股信息等。
- `wukong-quant-actions`：动作工具，提交深度分析、登录、添加/删除自选股等有副作用操作。

拆分 read/actions 后，工具名前缀也随 server name 区分：

- 只读工具：`mcp_wukong_quant_read_*`
- 动作工具：`mcp_wukong_quant_actions_*`

### 公网服务

| MCP server | transport | url |
|------------|-----------|-----|
| wukong-quant-read | http | `https://www.wukongquant.top/api/mcp/read` |
| wukong-quant-actions | http | `https://www.wukongquant.top/api/mcp/actions` |

### 本地部署

本地 Hermes 与 quantFinance 同在 `quant-network` 时，直接连接两个 MCP 容器：

| MCP server | transport | url |
|------------|-----------|-----|
| wukong-quant-read | http | `http://quant-mcp:3002/api/mcp` |
| wukong-quant-actions | http | `http://quant-mcp-actions:3003/api/mcp` |

### 请求头

两个 MCP server 都需要配置：

| 请求头名称 | 值 |
|------------|----|
| `X-Hermes-Quant-Key` | 公网服务使用内部密钥；本地部署使用 `quantFinance/.env` 中的 `HERMES_QUANT_INTERNAL_KEY` |
| `X-User-Token` | 通过 `user_login` 工具获取的 JWT token（可选，仅个人化端点需要） |

自签名证书场景可按 Hermes MCP 配置支持情况设置 `ssl_verify=false`。
