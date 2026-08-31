---
name: wechat-published-verify
description: 核对公众号"文章索引/清单"与微信后台「发表记录」是否一致。通过 Edge/Chrome DevTools 协议(CDP) 自动抓取后台已发表文章列表，与本地索引逐条比对；含连接排障与降级方案（端口未开、旧进程未清、沙箱隔离、历史快照、截图核对）。当用户要"核实已发文章、核对索引与后台、查漏发/错标、验证发表记录"时使用。
---

# 公众号后台发表记录抓取与索引核对

## 何时用
- 你手上有一份"计划/已发文章清单"（如 200 问索引、系列目录、发布排期），想确认微信后台实际发表了哪些。
- 要查某篇是否真发了、状态标没标错、有没有漏发、标题对不对得上。
- 典型场景：系列收官时把"索引 74 篇"和后台"发表记录"逐条对账。

## 核心结论（先记住这 5 条）
1. **CDP 连接品牌无关**：Edge / Chrome 都是 Chromium 内核，`connectOverCDP('127.0.0.1:PORT')` 都能连，不必区分浏览器。注意：这和"Edge 复制 HTML 进微信丢格式"是**两码事**——CDP 是读 DOM，不碰剪贴板，故不受那个 bug 影响。
2. **必须带参启动**：普通双击打开的浏览器**不会**监听调试端口。必须用 `msedge --remote-debugging-port=9222` 之类带参启动。
3. **必须先杀干净旧进程**：Edge 关窗口后仍驻留后台 `msedge.exe`；不杀干净，新命令只是又开了个窗口、参数被旧实例忽略。加 `--user-data-dir` 强制开全新实例最稳。
4. **AI 侧可能连不到你桌面的端口**：AI 的 Bash/Node 常跑在隔离网络命名空间，即便你本地端口开了，AI 侧 `curl 127.0.0.1:PORT` 仍可能是 closed。此时直接转"截图核对"，别死磕端口。
5. **每次运行都有三道必须人工过的"授权门"**（AI 无法代点任何一道，故本 skill **不能无人值守运行**，每次都要你坐在电脑前逐步确认）：
   - **① 微信后台登录/CDP 授权**：登录 `mp.weixin.qq.com` 时可能需要管理员扫码或点确认；CDP 抓取时也可能弹出授权确认页。
   - **② WorkBuddy 危险操作授权**：AI 调用 Bash（沙箱外网络请求、本地进程操作等）时，WorkBuddy 会弹出权限确认页/卡，每次都要你手动点"同意"。（见 `assets/workbuddy-authorization-screenshot.png`）
   - **③ Git 凭据助手选择器（CredentialHelperSelector）**：AI 执行 `git push` 等 HTTPS 操作时，Git Credential Manager 可能弹出 `Select a credential helper` 窗口，让你选 `no helper / manager / wincred` 并决定是否"Always use this from now on"。**必须手动点选**：建议选 `wincred`（与 Windows 凭据管理器中的 `git:https://github.com` 条目匹配最稳），勾上 `Always use this from now on` 再点 `Select`。选错或不点，push 会卡住或失败。（见 `assets/git-credential-helper-selector.png`）

## 标准流程

### A. 抓后台（自动，优先）
1. 任务管理器**杀光**所有 `msedge.exe` / `chrome.exe`（含后台驻留）。
2. `Win+R` 运行：
   ```
   msedge --remote-debugging-port=9222 --user-data-dir="C:\Temp\edge-debug"
   ```
3. 新 Edge 里打开 `mp.weixin.qq.com` 登录，进「内容管理 → 发表记录」。
3.5. **手动通过授权页（每次必做）**：登录或抓取时微信后台会弹**授权/确认页**（你刚才那次也出现了），必须**你本人在浏览器里手动点"同意 / 确认 / 扫码"**。未经授权页面不会真正进入「发表记录」列表，脚本也抓不到数据。确认授权通过、能看到列表后，再让 AI 继续。
4. **本地验证端口**：在你 Edge 地址栏输 `http://127.0.0.1:9222/json/version`，能出 JSON（含 `"Browser"` 字段）即端口真开。
5. 跑抓取脚本 `query_published.js`（来自 `wechat-published-history-cdp` 技能）：自动翻页抓全部已发文章（标题 + 真实链接 + 阅读/赞/分享数据），落盘 `published_catalog.json`。

### B. 核对（索引 vs catalog）
- 从索引 HTML 提取篇目（题号 + 标题），按板块分组。
- 对每篇提取**核心关键词**，去 catalog 标题里搜——发表标题多是营销式，与索引标题不一致，靠主题词匹配（如"二十四节气""四大发明""满族""旗袍"）。
- 输出三态：✅ 对得上 / ❌ catalog 无此篇（可能漏发）/ ⚠️ 索引标错（如状态列错位、篇数矛盾）。

### C. 降级方案（连不上时，按稳度排序）
- **截图核对（最稳）**：让用户在后台「发表记录」多翻几页截图，AI 拿清单逐条比对。不依赖端口、不依赖 AI 侧网络。
- **历史快照**：搜旧工作区是否已有 `published_catalog.json` / `published_items.json`（之前抓过会留档）。⚠️ **注意时间边界**——快照只能证明"该时间点之前"的发表情况，覆盖不到之后新文。我们曾用 8/1 快照核 8/23–24 才生成的文章，结果之 190–200 全部零命中，无法核最新。

## 排障速查表
| 现象 | 根因 | 解决 |
|---|---|---|
| 端口 closed，进程列表无 `remote-debugging` | 普通窗口启动，未带参 | 带 `--remote-debugging-port` 重启 |
| 带参仍 closed | 旧进程未杀净，参数被旧实例忽略 | 杀光 msedge.exe + 加 `--user-data-dir` |
| 用户本地能访问 JSON，AI 侧 closed | 沙箱网络隔离 | 转截图核对 |
| catalog 搜不到最新篇 | 快照陈旧 | 重抓最新 / 截图核对 |
| 端口已开却抓不到数据 | 授权页弹出后未手动确认，脚本在等待 | 回浏览器查是否有待确认的授权页，手动点"同意" |
| git push 卡住 / 弹出 Select a credential helper | Git 未指定默认凭据助手 | 手动选 `wincred` 并勾 `Always use this from now on` 后点 `Select`；或命令显式加 `-c credential.helper=wincred` |

## 关键坑（踩过才懂）
- **别混淆两件事**：① CDP 读 DOM（品牌无关，稳）；② 剪贴板复制 HTML 进微信（Edge 有丢样式 bug，必须 Chrome）。前者用于"核对"，后者用于"发布"，互不影响。
- **索引内部先自查**：标题篇数 vs 正文篇数矛盾、状态列错位，往往比连后台更早发现错（我们那次标题写"全 68 篇"、正文写"全 74 篇"，之 195–200 状态列错填成板块名）。
- **未认证订阅号落地方式**：外链会失效，核对结论别写成"点超链接"，改为"回复关键词 / 进合集"；合集（专辑）才是稳定的聚合入口。

## 依赖
- `wechat-published-history-cdp` 技能（提供 `query_published.js` 抓取脚本）。
- Node + `playwright-core`（脚本已依赖，品牌无关 `connectOverCDP`）。

## 产物约定
- 核对清单：`索引篇目核对清单.md`（由索引 HTML 提取，标注内部不一致 + 每篇命中状态）。
- 抓取结果：`published_catalog.json` / `.md`（后台真实发表记录）。
