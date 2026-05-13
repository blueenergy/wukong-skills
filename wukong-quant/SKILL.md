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
| 悟空，深度分析 {股票代码} | 提交深度分析任务并轮询结果 |
| 悟空，查 {股票代码} 的历史分析 | 查最近历史分析 |
| 悟空，今日天梯 | 拉取连板天梯数据 |

## 深度分析工作流

当用户说 **"悟空，深度分析 {symbol}"**：

1. 调用 `mcp_wukong_quant_hermes_deep_analysis_wait`，参数 `symbol={symbol}`（服务端自动等待完成，通常 60-180 秒）
2. `success=true` 时，**原文完整输出**以下字段，不得摘要压缩：
   - `analysis.investment_advice`（投资建议）
   - `analysis.technical_analysis`（技术面分析）
   - `analysis.key_points`（关键结论）
   - `analysis.short_term_forecast` / `mid_term_forecast` / `long_term_forecast`
   - `analysis.risk_level`、`analysis.support_level`、`analysis.resistance_level`
   - `analysis.confidence_score`（信心评分）

   若上述子字段不存在，则将 `analysis` 完整 JSON 原样展示给用户
3. `success=false` 时，输出 `error` 字段；若 `status=timeout`，告知用户可用 `task_id` 继续查询

## 其他工具

- **查历史**：调用 `mcp_wukong_quant_hermes_get_analysis_history`，参数 `symbol`
- **连板天梯**：调用 `mcp_wukong_quant_hermes_ladder_daily`

## MCP 配置

需在 `~/.hermes/config.yaml` 中添加：

```yaml
mcp_servers:
  wukong-quant:
    transport: http
    url: http://localhost:3001/mcp
    headers:
      X-Hermes-Quant-Key: "your-key-here"