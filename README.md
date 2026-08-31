# wechat-published-verify

公众号后台「发表记录」抓取与索引核对的可复用流程。

## 解决什么痛点
系列文章做多了，常有一份"索引 / 清单 / 排期"要和微信后台实际发表记录对账：哪些真发了？状态标错没？有没有漏发？手动翻后台又慢又易漏。本技能把"自动抓后台 + 逐条核对 + 连接排障 + 降级方案"固化下来。

## 核心要点
- CDP 连接**品牌无关**（Edge / Chrome 同属 Chromium），`connectOverCDP` 都能连。
- 必须**带 `--remote-debugging-port` 启动浏览器**，且先杀干净旧进程（建议加 `--user-data-dir` 强制新实例）。
- AI 侧常连不到用户桌面端口 → 直接转**截图核对**最稳。
- 历史快照有**时间边界**，覆盖不到之后新文，别拿旧快照核最新。
- 别把"CDP 读 DOM"和"剪贴板复制丢格式"混为一谈。

## 用法
1. 让用户带参启动浏览器并登录后台「发表记录」（见 SKILL.md 标准流程 A）。
2. 跑 `wechat-published-history-cdp` 的 `query_published.js` 抓全部已发文章。
3. 从索引 HTML 提取篇目，关键词匹配 catalog，输出三态结论（✅/❌/⚠️）。
4. 连不上时降级：截图核对 或 复用历史快照（注意时效）。

## 依赖
- `wechat-published-history-cdp` 技能（抓取脚本）
- Node + playwright-core
