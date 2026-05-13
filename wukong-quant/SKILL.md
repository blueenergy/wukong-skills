---
name: wukong-quant
description: 通过悟空量化 MCP 接口调用 A 股深度分析、连板天梯等功能。需配置 wukong-quant MCP 服务器。
version: 1.0.0
author: blueenergy
license: MIT
metadata:
  hermes:
    tags: [quant, stock, A-share, MCP, wukong]
prerequisites:
  mcp_servers: [wukong-quant]
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
| 悟空，查 {股票代码} 的历史分析 | 查最近历史分析 |

## 深度分析工作流

当用户说 **"悟空，深度分析 {symbol}"**：

1. 调用 `mcp_wukong_quant_hermes_deep_analysis_wait`，参数 `symbol={symbol}`（服务端自动等待完成，通常 60-180 秒）
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

## 其他工具

- **查历史**：调用 `mcp_wukong_quant_hermes_get_analysis_history`，参数 `symbol`
- **连板天梯**：调用 `mcp_wukong_quant_hermes_ladder_daily`
- **涨停分析**：调用 `mcp_wukong_quant_hermes_ladder_narrative`（可选参数 `date=YYYYMMDD`，默认最新）
  - 返回字段：`headline`（标题）、`sentiment_signal`（情绪信号）、`narrative_markdown`（完整 Markdown 叙事，原文输出）
- **AI 大盘分析**：调用 `mcp_wukong_quant_hermes_market_summary`（可选参数 `date=YYYY-MM-DD`，默认最新交易日）
  - 交易日 10:00 / 11:30 / 15:30 自动更新；读缓存，不触发 LLM

## MCP 配置

需在 `~/.hermes/config.yaml` 中添加：

```yaml
mcp_servers:
  wukong-quant:
    transport: http
    url: http://localhost:3001/mcp
    headers:
      X-Hermes-Quant-Key: "your-key-here"