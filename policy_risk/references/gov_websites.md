# 政府网站可访问性实测记录

> **时点快照，非长期事实。** 记录时间：2026-06。站点行为会变，引用前先复核当前状态，
> 不要据此直接断言某站永久不可用。

结论：gov.cn 及各部委子域名的**政策内容页**普遍使用 JS 客户端重定向，`curl` 无法取得正文，
因此 terminal + curl 不能当作 `web_fetch` 的替代方案。

## 实测结果（2026-06）

| 网站 | 政策内容页 | 当时状态 |
|------|-----------|-----------|
| `www.gov.cn` | 所有政策文件页（如 `/zhengce/content/...`） | JS redirect，无法获取正文 |
| `www.mee.gov.cn` | 生态环境部政策页 | JS redirect，无法获取正文 |
| `www.ndrc.gov.cn` | 发改委政策页 | JS redirect，无法获取正文 |
| `www.miit.gov.cn` | 工信部政策页 | JS redirect，无法获取正文 |
| `www.moa.gov.cn` | 农业部政策页 | JS redirect，无法获取正文 |
| `www.mofcom.gov.cn` | 商务部政策页 | JS redirect / 404 |
| `www.ccgp.gov.cn` | 政府采购公告页 | 404 |
| `zycg.gov.cn` | 政府采购页 | 未验证 |
| `www.cac.gov.cn` | 网信办 | 未验证 |
| `www.chinatax.gov.cn` | 税务总局 | 未验证 |

搜索引擎侧：Google 对 curl 返回 JavaScript challenge 页；Bing 部分返回 RSS 或重定向，内容不可用。

## 技术原因

政策内容页返回的 HTML 只有一段重定向脚本：

```html
<script>window.location.href="https://www.gov.cn";</script>
```

`curl -sL` 跟随后落到导航首页而不是目标政策页，所以拿到的是首页 HTML，不是政策正文。
能执行 JS 的抓取方式（`browser_navigate`）不受此影响。

## 替代路径

1. MCP 工具最可靠：`macro_analysis`、`sector_latest` 提供宏观与板块政策方向
2. 东方财富/同花顺等平台内政策数据库搜索关键词
3. 行业协会网站有时直接提供政策文本
4. 已有具体政策链接时，用 `browser_navigate` 打开取正文
5. 新闻 API（如已配置）作为线索来源，仍需回溯原文

## 抓不到原文时的输出要求

- 该条政策标 `[待核验]`，不要用模型记忆补条款内容
- 报告开头写明政策原文检索受阻及受影响范围
- 明确列出需要研究员手动核实的政策条款
