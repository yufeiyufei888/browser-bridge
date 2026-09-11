# browser-bridge

> **让任何 AI Agent 用上你的真实浏览器。** 一条命令接上 CDP —— 自带登录态、跨平台、跨 Agent。

`browser-bridge` 是一个 **SKILL.md 形态的技能包**：DSH / Claude Code / Codex / 以及任何会读 `SKILL.md` 的 Agent 都能加载它。它给 Agent 补上两件自带工具缺的东西：**联网策略** + **直接操控你日常浏览器的能力**（Chrome DevTools Protocol，天然携带 Cookie 与登录态）。

脚本零依赖：纯 Node.js ESM（Node **22+** 原生 WebSocket），Windows / macOS / Linux 通用。

---

## 能力

| 能力 | 说明 |
|---|---|
| 联网方式自动选择 | WebSearch / WebFetch / curl / 第三方 Markdown 化服务 / 浏览器 CDP，按场景组合 |
| 直连日常浏览器 | 不启动独立浏览器，直接用你已登录的 Chrome / Edge / Chromium |
| 页面任意 JS | `eval` 读写 DOM、提取数据、穿透 Shadow DOM 与 iframe |
| 三种点击 | JS 点击 / CDP 真实鼠标事件 / 文件上传（绕过系统文件对话框） |
| 截图 | 整页或视频当前帧落盘 PNG，交给支持多模态的模型看图 |
| 本地书签 / 历史检索 | 公网搜不到的目标（内网系统、SSO 后台）可从本机浏览器记录里定位 |
| 并行分治 | 多个子 Agent 共享一个 proxy，按 tab 隔离并行调研 |
| 站点经验 | 按域名累积操作经验（URL 模式、平台特征、已知陷阱），跨会话复用 |

## 什么时候用它，什么时候不用

- **不用**：静态文章、文档、公开页面 —— Agent 自带的 WebSearch / WebFetch 更快更省 token。
- **用它**：需要登录态的页面、JS 动态渲染、反爬站点、需要点击/填表/上传/翻页的交互流程、需要截图或视频帧、以及上面那些方式都拿不到内容时的兜底。

## 安装

### 1. DSH

```powershell
$dst = "$env:USERPROFILE\.dsh\skills\browser-bridge"
git clone https://github.com/yufeiyufei888/browser-bridge.git $dst
```

DSH 会扫描用户级技能根（`~/.dsh/skills`、`~/.agents/skills`）与项目级技能根（`<项目>/.dsh/skills`、`<项目>/.agents/skills`），放进去即对所有项目生效。**无需重启**，新会话的技能目录会自动出现 `browser-bridge`。

### 2. Claude Code

```bash
git clone https://github.com/yufeiyufei888/browser-bridge.git ~/.claude/skills/browser-bridge
# 或者作为插件市场安装：
claude plugin marketplace add https://github.com/yufeiyufei888/browser-bridge
claude plugin install browser-bridge@browser-bridge --scope user
```

### 3. Codex CLI

```bash
git clone https://github.com/yufeiyufei888/browser-bridge.git ~/.codex/skills/browser-bridge
```

### 4. 其他 Agent（通用）

把仓库放到该 Agent 的技能目录，或直接告诉它「读取这个仓库的 `SKILL.md` 并遵循指引」。技能根目录由 Agent 自己解析（`CLAUDE_SKILL_DIR` / 加载时注入的 base directory / 你读取 `SKILL.md` 的目录），文档里一律记作 `<SKILL_DIR>`。

> 不想用 git？在 GitHub 页面点 **Code → Download ZIP** 解压到对应技能目录即可。

## 前置条件（三步）

1. **Node.js 22+**（`node -v` 检查）。
2. **开启浏览器远程调试**：在你常用的浏览器里打开
   - Chrome：`chrome://inspect/#remote-debugging`
   - Edge：`edge://inspect/#remote-debugging`
   勾选 **Allow remote debugging for this browser instance**（可能需要重启浏览器）。首次有调试客户端连接时浏览器会弹一次「是否允许远程调试」，点**允许**。
3. **自检**：

```bash
node "<SKILL_DIR>/scripts/check-deps.mjs"
```

期望输出类似：

```
node: ok (v22.x)
browser: ok (Microsoft Edge, port 9222) [config.env 偏好]
proxy: ready (Microsoft Edge)
```

退出码：`0` 可用；`2` 需要选择默认浏览器（写入 `config.env` 的 `WEB_ACCESS_BROWSER`，合法值 `chrome` / `edge`）；`1` 按 stdout 的提示处理。

`config.env` 首次运行会从 `templates/config.env.template` 自动生成，已被 `.gitignore` 忽略（个人配置不入库）。

## 快速验证

```bash
# 列出用户已打开的标签页
curl -s http://localhost:3456/targets

# 新建后台标签页并读标题（PowerShell 用 curl.exe）
curl -s -X POST --data-raw 'https://example.com' http://localhost:3456/new
curl -s -X POST "http://localhost:3456/eval?target=<ID>" -d 'document.title'

# 用完关掉自己开的标签页（不动用户的标签）
curl -s "http://localhost:3456/close?target=<ID>"
```

## Proxy API 速查

| 端点 | 方法 | 说明 |
|---|---|---|
| `/health` | GET | 状态：是否已连上浏览器、连的是哪个浏览器 |
| `/targets` | GET | 列出所有页面标签 |
| `/new` | POST body=URL | 新建后台标签（自动等待加载），返回 targetId |
| `/navigate?target=` | POST body=URL | 导航 |
| `/info?target=` | GET | 标题 / URL / readyState |
| `/eval?target=` | POST body=JS | 执行任意 JS |
| `/click?target=` | POST body=选择器 | JS 点击 |
| `/clickAt?target=` | POST body=选择器 | CDP 真实鼠标点击 |
| `/setFiles?target=` | POST body=JSON | 给 file input 塞本地文件路径 |
| `/screenshot?target=&file=` | GET | 截图落盘 PNG |
| `/scroll?target=&y=` 或 `&direction=bottom` | GET | 滚动（触发懒加载） |
| `/close?target=` | GET | 关闭标签 |

完整 API 与 JS 提取模式见 [`references/cdp-api.md`](references/cdp-api.md)。

## 平台差异（Windows 必读）

Windows PowerShell 5.1 与 POSIX shell 有三处关键差异，文档里都给了两种写法：

1. **`curl` 是 `Invoke-WebRequest` 的别名** → 必须写 `curl.exe`，否则报参数错误。
2. **没有 `pkill`** → 结束 proxy 用：

```powershell
Get-CimInstance Win32_Process -Filter "Name='node.exe'" |
  Where-Object { $_.CommandLine -match 'cdp-proxy' } |
  ForEach-Object { Stop-Process -Id $_.ProcessId -Force }
```

3. **引号会被吃掉** → 传含引号 / 中文 / 多行的 JS 时，先写 **UTF-8 无 BOM** 文件再 `--data-binary "@文件"`：

```powershell
[IO.File]::WriteAllText("$env:TEMP\expr.js", 'document.querySelectorAll("a").length', (New-Object Text.UTF8Encoding($false)))
curl.exe -s -X POST --data-binary "@$env:TEMP\expr.js" "http://localhost:3456/eval?target=<ID>"
```

更多见 [`docs/windows-powershell.md`](docs/windows-powershell.md)。

## 已知限制与排错

| 现象 | 原因 / 处理 |
|---|---|
| `/screenshot` 一直不返回（30s 超时） | 目标页所在浏览器窗口被**最小化或遮挡**时，Chrome/Edge 不再产生合成帧。先把窗口还原再截图 |
| 需要长期后台截图 | 给浏览器快捷方式加 `--disable-backgrounding-occluded-windows --disable-features=CalculateNativeWinOcclusion` 后重启（保留 inspect toggle 与登录态） |
| `find-url.mjs` 提示找不到 `sqlite3` | 该脚本用外部 `sqlite3` CLI 读书签/历史：Windows `winget install sqlite.sqlite`，macOS `brew install sqlite` |
| `check-deps` 报「浏览器不一致」并硬退出 | 3456 端口被**另一个浏览器**的 proxy 占着。先结束旧 proxy 再重跑，或在 `config.env` 固定浏览器 |
| 每次连接都弹「是否允许远程调试」 | 浏览器对**每个新的调试客户端**都要授权。复用本技能常驻的 proxy，不要另开 WebSocket 直连调试端口 |
| 网站提示「内容不存在 / 页面不见了」 | 不一定是真的不存在，可能是访问方式问题（URL 缺隐式参数、被反爬）—— 换 GUI 交互路径再试 |

## 安全与合规

- **它操作的是你的真实浏览器**（含已登录的账号）。只在你信任的目标站点上使用。
- 部分平台对浏览器自动化检测严格，**存在账号被限流或封禁的风险** —— 强烈建议用不重要的账号操作社交平台。
- 脚本不向任何第三方服务回传数据；`find-url.mjs` 只读本机浏览器的书签/历史文件。使用后果由使用者自负。

## 目录结构

```
browser-bridge/
├── SKILL.md                     # 技能说明（Agent 读这个）
├── scripts/
│   ├── cdp-proxy.mjs            # 常驻 HTTP → CDP 代理
│   ├── check-deps.mjs           # 环境自检 + 确保 proxy 就绪
│   ├── browser-discovery.mjs    # 跨平台浏览器发现与选择
│   ├── find-url.mjs             # 本地书签/历史检索（需 sqlite3）
│   └── match-site.mjs           # 站点经验匹配
├── references/
│   ├── cdp-api.md               # CDP API 详解与 JS 提取模式
│   ├── migration-2.5.3.md       # 上游 API 迁移说明
│   └── site-patterns/           # 按域名累积的站点经验
├── templates/config.env.template
├── docs/                        # 安装与平台差异
├── CHANGELOG.md · LICENSE · NOTICE.md
```

## 归属与许可

- 上游：[**web-access**](https://github.com/eze-is/web-access) · MIT · 作者 **一泽Eze**（commit `33eef84`）
- 本项目：**browser-bridge** —— 跨平台 / 跨 Agent 的打包与文档适配。`scripts/`、`references/`、`templates/` 与上游**逐字节一致**，未改动任何逻辑。
- 许可：MIT（见 [`LICENSE`](LICENSE)），改动明细见 [`NOTICE.md`](NOTICE.md)。

> 如果你已经装了上游 `web-access`，**两者能力重叠，选一个即可**（同一个 proxy 端口，同时启用会互相干扰）。
