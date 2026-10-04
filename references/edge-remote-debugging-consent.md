# 排障：Edge「是否允许远程调试？」consent 弹窗反复弹出 / 堆叠卡死

> 适用：用 browser-bridge（或任何 CDP 客户端）连接 **Microsoft Edge 136+** 时，
> 反复弹出「是否允许远程调试？/ 某个外部应用希望完全控制此 Microsoft Edge 会话」，
> 无人点击时越堆越多、任务卡死。

## 根因

Edge 136 起，对**默认用户数据目录**的远程调试增加了一个 consent 确认。它是 **Chromium 原生对话框（Views 控件）**，有几个关键特性：

- **每次新的浏览器级 CDP 连接都可能触发一次**。客户端若在后台周期性重连（断线重连、健康检查），就会一次性弹出几十上百个，堆叠卡死。
- **自动化点不到**：它不是 OS 顶层窗口（Win32 枚举查无此窗口），也不在 Edge 的 UIA / MSAA 无障碍树里（扫遍全部 descendant 只有页面文字，没有「允许」按钮）。所以「找控件→Invoke」「找窗口→点按钮」这类方案在原理上全部失效。
- `RemoteDebuggingAllowed` 这类企业策略**只管「能不能用调试」这个开关，管不到这个 consent 弹窗**——设了也照弹（实测）。

## 关键事实（实测）

| 浏览器启动方式 | profile | consent 弹窗 |
|---|---|---|
| app 模式（`--app-id`/`--app-url`，常见于把网页当应用的启动器）+ 调试 | 默认 | **反复弹** |
| 默认 profile 通过 `edge://inspect` 的「Allow remote debugging」开关开调试 | 默认 | **反复弹** |
| **普通模式 + `--remote-debugging-port=<port>` 启动** | 默认 | **不弹** ✅ |
| 普通模式 + `--remote-debugging-port=<port>` 启动 | 独立 `--user-data-dir` | **不弹** ✅ |

结论：**以「普通模式 + 命令行调试端口」启动的实例不弹 consent**。分水岭不在 profile 是否默认，而在「是不是 app 模式 / 是不是用 inspect 开关开的调试」。

## 解法（已实测）

### 方案 1：让浏览器以「普通模式 + 调试端口」启动（保留登录态，推荐）

给你日常用的浏览器启动方式加上调试端口。Windows 上最简单的是改快捷方式：

```text
目标: "C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe" --remote-debugging-port=9222
```

之后这个实例的 `http://127.0.0.1:9222/json/version` 直接可用、CDP 可连、**全程不弹 consent**，且用的是你日常 profile（登录态保留）。

注意：不要让另一个「app 模式」实例同时开着调试——它会去抢写 `<User Data>\DevToolsActivePort`，把按该文件发现端口的客户端引到会弹窗的实例上。

### 方案 2：客户端固定连一个「不弹窗」的实例

如果你的自动化客户端支持指定端口，**固定连方案 1 那个端口**，不要按 `DevToolsActivePort` 自动发现（那个文件可能被 app 模式实例抢写）。

如果你希望给 `cdp-proxy` 加这个能力，可在 `discoverChromePort()` 开头加一段：设置一个环境变量（如 `CDP_FIXED_DEBUG_PORT`）后，先探测该端口的 `/json/version`，可用则直接返回它的 `webSocketDebuggerUrl`，跳过 `selectBrowser` 的自动发现（也就不会读到被 app 模式实例抢写的 `DevToolsActivePort`）。

## 不要走的路（都试过、无效）

- **UIA / Win32 找「允许」按钮**：consent 是 Chromium 原生对话框，不暴露在无障碍树，找不到。
- **computer-use 等「看见了再点」**：弹窗连发时追不上，仍堆叠；且依赖前台可见。
- **`RemoteDebuggingAllowed` 策略**：只控制调试开关，不抑制 consent。
- **杀进程**：app 模式实例往往由启动器（如各类桌面 Agent 的 Web 壳）自动重拉，治标不治本。

## 验证方法（确认修好）

1. 普通模式 + `--remote-debugging-port=9222` 启动 Edge。
2. 连续多次建立/断开浏览器级 CDP 连接（模拟断线重连）。
3. 用 Win32 `EnumWindows` 监视标题含「远程调试」的窗口计数 —— 应为 **0**。
