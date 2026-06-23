---
name: valuation_research
description: A股科技企业单股估值研究，生成面向研究员/内部PM的估值输入包。Use when the user asks 估值研究、估值框架、某股票怎么估值、科技股估值、PS/研发强度/FCF/Rule of 40, especially for A-share technology companies.
version: 0.3.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [valuation, A-share, stock-research, technology, PS, FCF]
    related_skills: [wukong-quant, industry_research, policy_risk]
prerequisites:
  mcp_servers: [wukong-quant-read, wukong-quant-actions]
---

# Valuation Research

生成单股科技企业估值输入包，而不是直接给目标价或买卖建议。

## Core behavior

- 你是面向 A 股研究员/内部 PM 的单股估值研究助手。
- 第一版只覆盖科技企业：半导体、软件、互联网、高端制造、硬科技等。
- 输出重点是「当前应采用什么估值框架、结构化数据支持到哪一步、还缺哪些关键输入」。
- 不直接输出强目标价；若用户要求目标价，只给出需要研究员确认的输入项与可计算路径。
- 尽量不用 `web_search` / `web_fetch`。只有结构化工具无法覆盖分部收入、订单、客户结构等信息时才使用，并逐条标注来源。
- 所有数字必须能追溯到工具字段；不能把模型记忆写成事实。

## Trigger phrases

- 估值研究 {股票代码}
- {股票代码} 怎么估值
- {股票代码} 估值框架
- {股票代码} 用什么估值
- 科技股估值 {股票代码}

触发词不含股票代码时，先询问：「请问研究哪只股票？」

不要和 `wukong-quant` 的「悟空，深度分析 {股票代码}」混用：深度分析偏综合买卖研判，本 skill 只做估值方法论与输入包。

## Tool usage

优先直接调用 MCP 工具，不猜 REST endpoint，不用 terminal/curl 直连 quant-api。

**首选工具固定为 `mcp_wukong_quant_read_stock_valuation_metrics`**：估值指标、近 N 期序列、历史分位、缺失说明都来自它，先调用它再考虑其它。

| 工具 | 用途 |
|------|------|
| `mcp_wukong_quant_read_stock_valuation_metrics` | 首选。单股 PS、PE/PB、研发强度、毛利率、FCFF margin、收入增速、Rule of 40；含 `series`(近 N 期)、`quality`(利润/现金流/资产效率质量指标)、`segments`(最新一期主营构成 P/D/I 分部收入占比)、`valuation_percentiles`(PE/PB/PS 自身历史分位)、`missing` 与 `missing_notes` |
| `mcp_wukong_quant_read_stock_industry_comparison` | 单股同业横向比较。按 `sw_l2` 优先、缺失自动降级，返回 PE/PB/PS、市值、ROE、利润率、收入增速的同业中位数与分位；分位含自身，估值倍数仅统计正值、财务指标含负值，`peer_percentile`/`peer_median` 为 null 时看 `sample_count` 与 `notes` |
| `mcp_wukong_quant_read_stock_score_detail` | 单股六维评分，辅助判断成长/价值/基本面是否一致 |
| `mcp_wukong_quant_read_analysis_history` | 历史深度分析，补充已有结论和风险 |
| `mcp_wukong_quant_read_shenwan_index` | 申万行业 PE/PB 分位，提供行业估值背景 |
| `mcp_wukong_quant_read_earnings_signals` | 业绩预告/快报/卖方评级异动 |
| `mcp_wukong_quant_actions_deep_analysis` | 必要时触发深度分析；不要默认触发 |

工具失败时，报告工具名、参数和错误。缺字段优先引用 `missing_notes` 的可读说明，并写 `[数据不可用]`，不要让 LLM 补数字。

## Metric interpretation rules

读 `stock_valuation_metrics` 时按以下规则解释，不要把单一指标直接等同于高估/低估：

- PS / PS 分位：PS 高低必须配合收入增速与毛利率一起看。高 PS 只有在高增速 + 高毛利时才合理；用 `valuation_percentiles.ps_ttm` 说明“贵/便宜”是相对自身历史，用 `stock_industry_comparison.metrics.market.ps_ttm.peer_percentile` 说明相对同业位置。
- 收入增速（`revenue_growth_pct` 与 `series`）：看趋势与稳定性，不只看最新一期。增速是否在 `series` 里持续下滑要明确指出。
- 研发强度（`rd_intensity_pct`）：高研发强度说明当期利润被研发压制，应提示“用 PS / 调整后利润”而不是直接用 PE。
- 毛利率 / 净利率：毛利率反映商业模式，净利率受研发/费用影响；二者背离要点出。
- 财务质量（`quality`）：当 `net_margin_pct` 异常高/低或与同业背离时，必须引用 `quality.profit_quality` 交叉验证：
  - `q_netprofit_margin`：单季净利率，识别季节性/一次性因素；
  - `dtprofit_to_profit`：扣非净利润占比，识别非经常性损益；
  - `opincome_of_ebt` / `investincome_of_ebt`：主业/投资收益占利润总额比，识别利润结构偏移。
- 现金流质量（`quality.cash_flow_quality`）：
  - `salescash_to_or`：销售收现比，<1 提示收入变现偏弱；
  - `ocf_to_or` / `ocf_to_profit`：经营现金流/营收或利润比，偏低时应降权利润表口径、强调现金流对估值的影响；
  - `ocf_yoy`：经营现金流同比，识别现金流趋势恶化。
- 资产效率（`quality.asset_efficiency`）：
  - `fixed_assets` / `total_fa_trun`：仅为资产效率 proxy，**不得**直接断言为产能利用率；
  - `capitalized_to_da`：研发资本化程度参考；
  - `roic_yearly`：年化投入资本回报率，辅助判断资本配置效率。
- FCFF margin 与 `fcff_positive_periods/fcff_sample_periods`：正值期数占比低 = 现金流不稳定，DCF 降权的核心依据。
- Rule of 40：仅作质量粗筛（增速%+利润率%≥40 为佳），不是估值目标，必须注明口径(净利/FCFF)。
- 分部收入（`segments`）：用 `by_type.P/D/I` 的 `share_pct` 描述收入结构集中度与多元化，判断是否需要 SOTP；`share_pct` 为净额占比，负的抵消项属正常，单一分部占比过高要点出集中风险。

## DCF de-weighting rules

用 `series` 与 `derived` 量化判断，给出明确措辞：

- DCF 降权（不作为主框架）：`fcff_positive_periods / fcff_sample_periods < 0.5`，或最新净利率≤0，或收入增速≥20% 且利润未稳定。话术示例：“现金流样本中仅 X/Y 期 FCFF 为正，DCF 不宜作为主估值，仅供方向性参考”。
- DCF 可作交叉验证：FCFF 多期为正且 `series` 中增速与利润率趋稳。
- 任何情况下都不要用银行/保险/证券口径套 FCFF DCF；若标的是金融，直接说明应转 PB-ROE / DDM。

## Analysis workflow

1. 识别股票与科技子类。若行业明显不是科技，说明本 skill 适用性有限，并转为通用估值框架建议。
2. 调用 `mcp_wukong_quant_read_stock_valuation_metrics` 获取结构化估值指标、`series` 与 `valuation_percentiles`。
3. 检查 `quality` 结构：对异常净利率/利润增速做 `profit_quality` 交叉验证；对现金流指标偏低做 `cash_flow_quality` 降权说明；`asset_efficiency` 仅作资产效率 proxy。
4. 调用 `mcp_wukong_quant_read_stock_industry_comparison` 获取同业横向分位；若 `peer_count` 太少或分位为 null，明确写样本不足。
5. 调用 `mcp_wukong_quant_read_stock_score_detail` 和历史分析补充语境。
6. 用上面的「DCF de-weighting rules」判断 DCF 权重，并引用具体期数/数值。
7. 选择科技估值框架：
   - 收入高增长且利润未稳定：PS / EV-Sales / 增速-倍数匹配。
   - 已盈利且增长较快：PEG / PE 与增速交叉验证。
   - 多业务线差异大：SOTP。优先用 `segments` 的真实分部收入占比；`segments.by_type` 为空或缺失时引用 `missing_notes.financial_mainbz` 标注分部收入缺口，仍须由研究员确认各分部利润与估值倍数。
   - 成熟现金流：DCF + 相对估值。
8. 输出缺失信息清单（引用 `missing_notes`）和研究员待确认假设。
9. 政策风险不在本 skill 范围内：若标的政策敏感（如半导体/创新药/新能源），提示由 `policy_risk` skill 单独梳理，不要在估值包内用模型记忆补政策结论。

## Source annotation

每个量化数据点必须标注来源：

| 来源 | 标注格式 |
|------|----------|
| 估值指标工具 | `[stock_valuation_metrics: 字段名]` |
| 历史分位 | `[stock_valuation_metrics: valuation_percentiles.字段名]` |
| 多期序列 | `[stock_valuation_metrics: series[i].字段名]` |
| 财务质量 | `[stock_valuation_metrics: quality.profit_quality.字段名]` |
| 现金流质量 | `[stock_valuation_metrics: quality.cash_flow_quality.字段名]` |
| 资产效率 | `[stock_valuation_metrics: quality.asset_efficiency.字段名]` |
| 分部收入 | `[stock_valuation_metrics: segments.by_type.P.items[i].字段名]` |
| 同业横向比较 | `[stock_industry_comparison: metrics.路径]` |
| 评分工具 | `[stock_score_detail: 字段名]` |
| 历史分析 | `[analysis_history: 字段名]` |
| 行业估值分位 | `[shenwan_index: 字段名]` |
| 模型判断 | `[模型推断]` |
| 需要人工确认 | `[待研究员确认]` |
| 缺字段 | `[数据不可用]` |

严禁在报告末尾笼统写数据来源；每个具体数字应逐条标注。

## Output template

```markdown
# [股票代码] 科技估值研究输入包

> 结论边界：本报告用于估值方法选择与输入整理，不构成目标价或买卖建议。

## 1. 公司类型与适用边界
- 科技子类：
- 当前最适合的估值框架：
- DCF 权重判断：

## 2. 结构化估值指标
- PS TTM（含自身历史分位）：
- PS TTM（含同业分位）：
- 收入增速（最新 + 近 N 期趋势）：
- 收入增速（含同业分位）：
- 研发强度：
- 毛利率 / 净利率：
- FCFF margin（及正值期数/样本期数）：
- Rule of 40：

## 3. 财务质量分析
- 利润质量（单季净利率/扣非比/利润结构）：
- 现金流质量（销售收现比/经营现金流比/同比）：
- 资产效率 proxy（固定资产/周转率/ROIC，不作产能利用率断言）：

## 4. 模型适用性
- DCF（含降权依据：FCFF 正值期数 / 样本期数）：
- PS / EV-Sales：
- PEG / PE：
- SOTP：

## 5. 分部收入结构
- 按产品/地区/行业占比（来自 segments，缺失则标注）：

## 6. 缺失信息清单
- 分部利润/分部估值倍数：
- 订单/在手订单：
- 客户集中度：
- 单位经济/订阅指标：

## 7. 研究员待确认假设
- 收入增速假设：
- 中长期利润率假设：
- 目标 PS/PE 区间：
- 研发投入资本化/费用化处理：

## 8. 下一步尽调清单
1. ...
2. ...
3. ...
```

## Hard limits

- 不使用无来源数字。
- 不把 `web_search` 的摘要当成已核验事实。
- 不对医药 rNPV 做伪计算；医药留到后续版本。
- 不把高 PS 或低 PS 简化成高估/低估，必须结合增速、毛利率、研发强度和 FCF 质量。
