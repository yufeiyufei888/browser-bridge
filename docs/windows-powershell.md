# Windows / PowerShell 差异与实测记录

本页记录在 **Windows + PowerShell 5.1** 下使用本技能时踩到并已验证的差异。POSIX 用户可忽略。

## 1. `curl` 是别名，必须写 `curl.exe`

```powershell
(Get-Command curl).CommandType   # Alias
(Get-Command curl).Definition    # Invoke-WebRequest
```

因此 `curl -s -X POST --data-raw '...'` 会直接报参数错误。把文档里所有 `curl` 换成 **`curl.exe`** 即可（Windows 10+ 自带）。

## 2. 没有 `pkill`

结束常驻 proxy：

```powershell
Get-CimInstance Win32_Process -Filter "Name='node.exe'" |
  Where-Object { $_.CommandLine -match 'cdp-proxy' } |
  ForEach-Object { Stop-Process -Id $_.ProcessId -Force }
```

## 3. 传 JS：引号会被 PowerShell 吃掉

简单表达式 `--data-raw 'document.title'` 可以，但含**引号 / 中文 / 多行**时会失败（PowerShell 会把内层引号当字面反斜杠传出去）。可靠做法是写 **UTF-8 无 BOM** 文件，再 `--data-binary "@文件"`：

```powershell
$expr = '"当前标题: " + document.title + " | 链接数=" + document.querySelectorAll("a").length'
[IO.File]::WriteAllText("$env:TEMP\expr.js", $expr, (New-Object Text.UTF8Encoding($false)))
curl.exe -s -X POST --data-binary "@$env:TEMP\expr.js" "http://localhost:3456/eval?target=<ID>"
```

> 必须用 `UTF8Encoding($false)`（无 BOM）。PS 5.1 的 `Set-Content -Encoding UTF8` 会写 BOM，BOM 出现在 JS 表达式开头会变成语法错误。

实测：`{"value":"中文测试 ok: Example Domain | a=1"}`。

## 4. `/screenshot` 与窗口最小化

浏览器窗口被**最小化或完全遮挡**时，CDP 的 `Page.captureScreenshot` 不返回，proxy 在 30s 后报 `CDP 命令超时`。`fromSurface:false`、`captureBeyondViewport` 都无效；Chrome 与 Edge 都会复现。

验证方法：`Browser.getWindowForTarget` 返回 `windowState`，最小化时为 `minimized`。

- 临时：截图前把浏览器窗口还原（可见即可）。
- 长期：给浏览器快捷方式追加 `--disable-backgrounding-occluded-windows --disable-features=CalculateNativeWinOcclusion` 后重启。inspect toggle 与登录态都不受影响。

纯 DOM 取文本（`/eval`）不受此限制。

## 5. 端口与浏览器绑定

proxy 默认监听 `127.0.0.1:3456`，并**在首次连上后固定该浏览器**。若 3456 已被「另一个浏览器」的 proxy 占用，`check-deps.mjs` 会硬错退出而不是静默降级：

```
proxy: 浏览器不一致 — 当前已连着 Microsoft Edge，但本次需要 chrome
```

处理：先结束旧 proxy（见第 2 节），或用 `config.env` 固定唯一默认浏览器。想同时用两个浏览器，可用 `CDP_PROXY_PORT=3457` 启动第二个实例，但本技能默认只用一个。

## 6. 授权弹窗

Chrome / Edge 对**每一个新的调试客户端**都会弹一次「是否允许远程调试？」。因此：

- 优先复用常驻 proxy（一次授权，长期有效），不要在脚本里另开 WebSocket 直连调试端口；
- 若弹窗反复出现，多半是某个进程反复新建连接；把浏览器里的远程调试开关关掉即可彻底停止。

## 7. 本机实测环境（供参考）

| 项 | 值 |
|---|---|
| OS | Windows 11（build 26100 级别） |
| Shell | Windows PowerShell 5.1.26100 |
| Node | v24.18.0 |
| 浏览器 | Chrome 153 / Edge 152，均开启 inspect toggle |
| 验证项 | `check-deps` exit 0；`/new`、`/eval`（含中文与引号）、`/screenshot`（窗口可见）、`/close` 全部通过 |
