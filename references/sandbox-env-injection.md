# 排障：cdp-proxy 起不来 / 检测不到浏览器（run_code 沙箱环境变量缺失）

> 适用：在 **DSH run_code 沙箱**、**Codex** 或任何 **process.env 被清空**的执行环境中
> 调用 `cdp-proxy.mjs` 时，报「浏览器没连上」「mismatch」「detectAll 返回空」，
> HTTP fetch `http://localhost:3456/targets` 超时。

## 根因

`scripts/browser-discovery.mjs` 用 `process.env.LOCALAPPDATA` 构造浏览器的
`DevToolsActivePort` 文件路径（Windows）：

```js
// browser-discovery.mjs (节选)
const localAppData = process.env.LOCALAPPDATA || '';
// ...
{ id: 'edge', devToolsPath: path.join(localAppData, 'Microsoft/Edge/User Data/DevToolsActivePort') }
```

在 **run_code 沙箱**中，`process.env` 是**空对象**，
`process.env.LOCALAPPDATA === undefined`，
于是路径退化为**相对路径** `Microsoft/Edge/User Data/DevToolsActivePort`：

| 环境 | `process.env.LOCALAPPDATA` | 构造出的路径 | `fs.existsSync` |
|------|------------------------------|--------------|------------------|
| 正常终端 | `C:\Users\xxx\AppData\Local` | 绝对路径 | ✅ true |
| **run_code 沙箱** | `undefined` | **相对路径** | ❌ **false** |

`detectAll()` 因此**跳过 Edge**，`selectBrowser()` 返回 `mismatch`，
`cdp-proxy` 启动后无法连接，HTTP API 超时。

**同样的问题影响**：`USERPROFILE`、`APPDATA`、`HOME` 等所有
`process.env.*` 取值——只要沙箱清空环境变量，`path.join(undefined, ...)`
就会产生相对路径。

## 修复：spawn 时注入环境变量

`cdp-proxy.mjs` 是被 spawn 的**子进程**，它读的是自己的 `process.env`。
调用方必须在 spawn 时显式注入：

```js
import { spawn } from 'child_process';

const child = spawn('node', ['/path/to/cdp-proxy.mjs'], {
  detached: true,
  stdio: 'ignore',
  windowsHide: true,
  env: {
    ...process.env,                     // 透传调用方环境（若有）
    LOCALAPPDATA: 'C:\\Users\\yufei\\AppData\\Local',  // ← 关键
    USERPROFILE:  'C:\\Users\\yufei',
    APPDATA:      'C:\\Users\\yufei\\AppData\\Roaming',
    HOME:         'C:\\Users\\yufei',  // git 等也需要
  },
});
child.unref();
```

**要点**：
1. `LOCALAPPDATA` 是 Windows 上 `browser-discovery` 找 `DevToolsActivePort` 的**必需**变量。
2. `USERPROFILE`/`HOME` 是 git、`os.homedir()` 等需要的（否则 git 报 `$HOME not set`）。
3. 路径值用**绝对路径**，不要依赖 cwd。

## 完整启动流程（DSH run_code 场景）

```js
const { execSync, spawn } = await import('child_process');

// 1. 杀掉旧 proxy（Windows 无 pkill）
execSync('powershell -NoProfile -Command "Get-CimInstance Win32_Process -Filter \"Name='node.exe'\" | Where-Object { $_.CommandLine -match 'cdp-proxy' } | ForEach-Object { Stop-Process -Id $_.ProcessId -Force }"');

// 2. 注入环境变量并启动
const child = spawn('node', ['scripts/cdp-proxy.mjs'], {
  detached: true, stdio: 'ignore', windowsHide: true,
  env: {
    ...process.env,
    LOCALAPPDATA: 'C:\\Users\\yufei\\AppData\\Local',
    USERPROFILE:  'C:\\Users\\yufei',
    APPDATA:      'C:\\Users\\yufei\\AppData\\Roaming',
    HOME:         'C:\\Users\\yufei',
  },
});
child.unref();

// 3. 等待启动 + WS 握手（5-15 秒）
await new Promise(r => setTimeout(r, 12000));

// 4. 验证
const targets = await fetch('http://127.0.0.1:3456/targets').then(r => r.json());
console.log('标签页数:', targets.length);
```

## 诊断：如果仍然连不上

### 1. 确认 Edge 开了远程调试
`%LOCALAPPDATA%\Microsoft\Edge\User Data\DevToolsActivePort` 文件**存在且第 1 行是端口号**：

```
50506
/devtools/browser/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

如果文件不存在 → Edge 没开远程调试，需在 `edge://inspect/#remote-debugging` 勾选
**「Allow remote debugging for this browser instance」**（见
`edge-remote-debugging-consent.md`）。

### 2. 确认端口可连
```js
const net = await import('net');
const isOpen = await new Promise(resolve => {
  const s = net.createConnection(50506, '127.0.0.1');
  s.once('connect', () => { s.destroy(); resolve(true); });
  s.once('error', () => { s.destroy(); resolve(false); });
});
```

### 3. 首次 WS 连接需授权
Edge/Chrome 第一次 WebSocket 连接会弹「是否允许远程调试」对话框，
**用户点一次「允许」**后，后续 WS 连接自动复用。**未点允许前 WS 会超时**。

### 4. 用 curl.exe 而非 fetch
PowerShell 的 `curl` 是 `Invoke-WebRequest` 别名，参数不兼容。Windows 上必须
`curl.exe`。在 Node 里用 `execSync('curl.exe ...')` 或进程内 `fetch` 都可，
但 `fetch` 在 proxy 未就绪时会**超时**（无报错详情），不如 `curl.exe` 直观。

## 关联文档

- `edge-remote-debugging-consent.md` —— Edge consent 弹窗反复弹出/堆叠卡死
- `SKILL.md` 前置检查 —— `check-deps.mjs` 的浏览器发现逻辑
- `scripts/browser-discovery.mjs` —— `detectAll()` / `selectBrowser()` 源码

## 影响范围

任何 **process.env 被清空**的执行环境：
- DSH run_code 沙箱
- Codex 后台 eval
- CI/CD 容器（未注入环境变量）
- 其他 Agent 的受限执行环境

**通用规则**：spawn 子进程读环境变量时，**永远显式注入必需的环境变量**，
不要假设 `process.env` 完整。
