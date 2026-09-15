---
name: portfolio-sweep-review
description: 复盘组合研究（参数扫描）结果：按轴归因多组参数组合的收益与回撤差异、解释差距来源、推荐参数簇并给出理由与失效条件。需 wukong-quant-read MCP。
version: 1.1.0
author: blueenergy
license: MIT
metadata:
  hermes:
    tags: [quant, portfolio, research, review, A-share, MCP, wukong]
prerequisites:
  mcp_servers: [wukong-quant-read]
---

# 组合研究复盘

对一次组合研究（`portfolio research` 参数扫描）跑出的多组参数组合做复盘：**先归因，再推荐**。

组合研究在 `stock_scores` 之上回答：因子有没有预测力、Top-N / 权重 / 成本是否合适、风控叠加能否改善回撤。它跑出的是一张网格 —— 几十到几百组参数组合，每组一条回测路径。

## 触发词

| 说法 | 动作 |
|------|------|
| 组合研究复盘 / 分析组合研究 / 复盘参数扫描 | `portfolio_research_jobs` 定位最近完成的任务 → 复盘 |
| 复盘研究任务 {job_id} | 直接复盘指定任务 |
| 哪个组合好 / 这堆参数选哪个 / 为什么差这么多 | 同上，重点出推荐与失效条件 |

## 工具

只用这两个只读 MCP 工具，**不要**用 `execute_code` / `terminal` / `curl` 直连 quant API，也不要猜 REST endpoint：

| 工具 | 用途 |
|------|------|
| `portfolio_research_jobs` | 列出研究任务，定位 `job_id`（可按 status 过滤） |
| `portfolio_research_result` | 取结果 + 按轴归因（axis_effects / axis_marginals / axes / coverage） |

流程：`portfolio_research_jobs` → 挑 `status=completed` 且有 `result_id` 的那个 → `portfolio_research_result(job_id)`。

若用户没指定任务，先列出再让他确认；不要擅自复盘一个 `failed` 的任务。

`job_id` 在界面上可能只显示前 8 位；工具要完整 UUID，前缀不匹配就先列出任务确认。

## 怎么读归因数据

**按这个顺序读**，每一步都影响后面能不能信：

### 1. `coverage` —— 先看这个，它决定后面哪些数字能用

```json
{"row_count": 200, "row_count_total": 324, "truncated": true, "rows_source": "mongo_preview"}
```

结果文档只持久化按 `risk_adjusted_score` 降序的**前 200 行**，真网格可能大得多。`truncated: true` 意味着**丢掉的是分数最差的那批**，于是：

- 各档保留比例严重不均（实测某网格 `regime` 两档是 67 / 133，真值各半）
- 因此 `axis_marginals` 的均值、`global_best_row` **系统性偏高**
- 配对 `axis_effects` 受影响小得多（它在组内对比），但配对数会参差

`truncated` 为真时：**结论以配对为准，边际均值只作参考，且必须在报告里声明**。

### 2. `axes` —— 真正被扫描的维度

`source: derived` 的轴是 `sweep_view` **没声明**、由数据发现的维度（实测 `variant` 6 档在变却不在 `sweep_axes` 里）。它们和声明轴同等重要 —— 漏看会把一个被扫描的维度当成固定参数。

### 3. `axis_effects` —— 配对比较（**结论强度的唯一来源**）

固定其余**全部变化维度**后，比较同一组合内某轴两个取值。`delta` 一律是 `a − b`：

| 读数 | 含义 |
|------|------|
| `verdict == "a_dominates"`（`a_worse_count == 0`） | a 在所有配对中不劣于 b，无反转 —— 最强的一类结论 |
| `verdict == "b_dominates"`（`a_worse_count == n_pairs`） | b 完胜 a，同上 |
| `verdict == "mixed"` | 该轴**无一致方向**，不要当结论 |
| `verdict == "no_difference"` | 逐对完全相同；是"无差异"，不是"a 完胜" |

`a_worse_count` 对收益和回撤**同义**：收益上 `delta < 0` 是 a 收益更低，回撤上 `delta < 0` 是 a 的回撤更负（更差）。所以回撤指标的 `a_dominates` 表示 a 回撤更小。

两个指标要分开下结论：**收益上 mixed、回撤上 b_dominates 是完全正常的组合**，而且往往是最有落地价值的结论（"这档几乎不花收益就降了回撤"）。

`pairs` 为空的轴带 `unpaired_reason`：

- `collinear_with_other_axes` —— 该轴与别的维度共线，控制之后没有残差变异。**这是"不可单独归因"，不是"无影响"**，不要写成"该参数无作用"。
- `ambiguous_groups` —— 组内仍有未受控维度（数据异常），该组不可比已丢弃。

### 4. `axis_marginals` —— 边际聚合（**有偏，仅作参考**）

每个轴取值的组内均值 + **该档行数 `observations`**。先看 `observations` 均衡不均衡 —— 不均衡就直接说明截断偏差有多大，别急着解读均值。

边际均值和配对冲突时**一律以配对为准**。

### 5. `stability` —— 注意，这**不是**边际聚合

它按**全部轴** groupby，每组只剩一个 combo（`observations` 恒为 1），只是一张按 `index_excess_return` 排序的明细表。不要把它当"每档均值"读。

### 6. `global_best_row` —— 带选择偏差，只用来定位

按 `risk_adjusted_score` 选出，网格越大 max 越乐观。它的作用是**定位**（看最优区长什么样），不是**推荐**。

## 分析纪律（硬约束）

每一条都对应一个已知的误判，逐条遵守：

1. **先看 `coverage`，再决定信什么。** 截断时边际均值有偏，别拿它下结论。

2. **先按轴拆，再看单行。** 网格里累计收益从大幅亏损到翻倍，**不是回测随机** —— 回测无 RNG，同一 `combo_key` 重跑数字不变。差异来自参数改了持有路径，再叠加 Top-N 厚尾和网格里挑最大值。

3. **推荐簇，不推荐 argmax。** 不要输出「推荐第 37 组，累计 +95%」。要输出「推荐 X 轴取 A、Y 轴取 B 这一簇」，并说明簇内是否稳健。

4. **`unpaired_reason` 是"不可归因"，不是"无影响"。** 共线的轴要如实说"本网格无法把它和 XX 分开"，不要顺着边际均值给它编一个方向。

5. **累计收益不可跨平均仓位直接比。** regime 空仓组约 49% 暴露 vs 满仓 100%。跨组比较用 `index_excess_cumulative_return`，并同时看 `sharpe` 与 `max_drawdown`。若某组收益高但暴露也高，明说它只是承担了更多风险。

6. **区分收益结论与回撤结论，各自下。** 工具在两个指标上分别给 verdict，不要合并成一句"该参数更好"。

7. **收益要看年段分布。** 一组的高收益可能绝大部分来自某一段行情，或单只股票贡献两成净利。收益集中就明说，这直接降低推荐强度。

8. **区分「策略有 alpha」与「窗口好」。** 大部分组在单边上涨段都是正的，这解释不了排名。要看的是**同一窗口内组间差异**，以及是否有跨窗口一致性。**换一个数据窗口，结论可能反转 —— 这是常态，不是异常**（实测同一套网格换个窗口后，原先"调仓间隔是收益主因"的结论不再成立，只剩回撤上的结论）。

9. **结论必须标明是样本内。** 组合研究全是样本内结果。推荐要落到预设还需另外的验证，不要在复盘里承诺实盘效果。

10. **不要臆造未提供的数据。** 没有的字段就说没有，不要推算市场原因。若要用市场段解释，只做定性说明并标明是推测。

## 输出契约

固定四段，缺一不可：

```markdown
## 结论
一句话：这次网格的主导因素是什么，推荐哪一簇。附 coverage 声明（是否截断）。

## 轴归因
| 轴 | 主导? | 收益 verdict | 回撤 verdict | 证据 |
表格逐轴给出，证据列引用实际数字（n_pairs / median_delta / a_worse_count）。
不可归因的轴单列一行写明 unpaired_reason。
明确写：哪些轴主导、哪些轴无效、哪些轴不可归因。

## 差异原因
收益差距的分解：参数路径（调仓/止盈/仓位）如何影响换手与暴露，
哪部分差异只是暴露度不同，哪部分在同一暴露下仍然成立。
若收益集中在某年段或某几只票，在此点明。

## 推荐
- **推荐簇**：轴 → 取值（不要给单行索引/单行 combo）
- **理由**：引用上面的配对证据（verdict 与 n_pairs），而非「累计收益最高」
- **失效条件**：什么情况下这个推荐不成立（数据水位变化、窗口切换、
  某轴被证明依赖其他轴配置、某轴实为共线等）
- **样本内声明**
```

## 硬约束

- 不给单一 argmax 组合当推荐；推荐必须是一个簇。
- 不给实盘承诺；不把样本内结论说成预期收益。
- `coverage.truncated` 为真时必须声明边际均值可能偏高。
- 引用数字必须来自工具返回，不得编造。
- 用户要「选为候选 / 保存预设」时，说明这是 Dashboard 上的操作，本 skill 只做分析、不写库。
