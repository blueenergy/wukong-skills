---
name: valuation_research
description: A股科技企业单股估值研究，生成面向研究员/内部PM的估值输入包。Use when the user asks 估值研究、估值框架、某股票怎么估值、科技股估值、PS/研发强度/FCF/Rule of 40, especially for A-share technology companies.
version: 0.1.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [valuation, A-share, stock-research, technology, PS, FCF]
    related_skills: [wukong-quant, industry_research]
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

| 工具 | 用途 |
|------|------|
| `mcp_wukong_quant_read_stock_valuation_metrics` | 单股 PS、PE/PB、研发强度、毛利率、FCFF margin、收入增速、Rule of 40 |
| `mcp_wukong_quant_read_stock_score_detail` | 单股六维评分，辅助判断成长/价值/基本面是否一致 |
| `mcp_wukong_quant_read_analysis_history` | 历史深度分析，补充已有结论和风险 |
| `mcp_wukong_quant_read_shenwan_index` | 申万行业 PE/PB 分位，提供行业估值背景 |
| `mcp_wukong_quant_read_earnings_signals` | 业绩预告/快报/卖方评级异动 |
| `mcp_wukong_quant_actions_deep_analysis` | 必要时触发深度分析；不要默认触发 |

工具失败时，报告工具名、参数和错误。缺字段写 `[数据不可用]`，不要让 LLM 补数字。

## Analysis workflow

1. 识别股票与科技子类。若行业明显不是科技，说明本 skill 适用性有限，并转为通用估值框架建议。
2. 调用 `mcp_wukong_quant_read_stock_valuation_metrics` 获取结构化估值指标。
3. 调用 `mcp_wukong_quant_read_stock_score_detail` 和历史分析补充语境。
4. 判断 DCF 是否应降权：
   - FCFF 为负或波动大、研发投入高、收入高增长但利润未稳定时，DCF 降权。
   - FCFF 稳定为正、收入增速趋稳、利润率可解释时，DCF 可作为交叉验证。
5. 选择科技估值框架：
   - 收入高增长且利润未稳定：PS / EV-Sales / 增速-倍数匹配。
   - 已盈利且增长较快：PEG / PE 与增速交叉验证。
   - 多业务线差异大：SOTP，但必须标注分部收入缺口。
   - 成熟现金流：DCF + 相对估值。
6. 输出缺失信息清单和研究员待确认假设。

## Source annotation

每个量化数据点必须标注来源：

| 来源 | 标注格式 |
|------|----------|
| 估值指标工具 | `[stock_valuation_metrics: 字段名]` |
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
- PS TTM：
- 收入增速：
- 研发强度：
- 毛利率 / 净利率：
- FCFF margin：
- Rule of 40：

## 3. 模型适用性
- DCF：
- PS / EV-Sales：
- PEG / PE：
- SOTP：

## 4. 缺失信息清单
- 分部收入：
- 订单/在手订单：
- 客户集中度：
- 单位经济/订阅指标：

## 5. 研究员待确认假设
- 收入增速假设：
- 中长期利润率假设：
- 目标 PS/PE 区间：
- 研发投入资本化/费用化处理：

## 6. 下一步尽调清单
1. ...
2. ...
3. ...
```

## Hard limits

- 不使用无来源数字。
- 不把 `web_search` 的摘要当成已核验事实。
- 不对医药 rNPV 做伪计算；医药留到后续版本。
- 不把高 PS 或低 PS 简化成高估/低估，必须结合增速、毛利率、研发强度和 FCF 质量。
