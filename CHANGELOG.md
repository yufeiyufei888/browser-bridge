# Changelog

## 1.0.1 — 2026-09-11

**文档修订版 —— 脚本零改动**（`scripts/`、`references/`、`templates/` 与上游 web-access 仍逐字节一致）。

- 子 Agent 段落：明确 **模型默认继承主对话当前模型**。DSH 0.1.5-rc.1 起，子代理路由实时取自父会话的 request header（`dsh-subagent/lib/index.js` 的 `parentAgentOptionsForDelegation`），早期「继承创建时模型、不随 UI 切换」的缺陷已修复，因此不再建议一律强制指定模型。
- 截图说明：去掉具体工具名，改为「落盘后交给模型的多模态输入」（原生多模态读图，无需第三方视觉 API），便于跨 Agent 通用。
- `NOTICE.md` 同步记录以上改动。
## 1.0.0 — 2026-09-11

首个发行版，基线为上游 [web-access](https://github.com/eze-is/web-access) v2.5.4（commit `33eef84`）。

**打包与适配**

- 技能更名为 `browser-bridge`，`SKILL.md` 改为跨 Agent 通用描述（技能根目录占位符 `<SKILL_DIR>`、两种 shell 方言对照表）。
- 新增 `README.md` / `README.en.md` / `docs/`：DSH、Claude Code、Codex、通用 Agent 的安装与自检步骤。
- 新增 Windows PowerShell 5.1 适配说明与实测记录（`docs/windows-powershell.md`）：`curl.exe`、无 `pkill`、UTF-8 无 BOM 文件 + `--data-binary`。
- 新增「已知限制与排错」：截图与浏览器窗口最小化、`find-url.mjs` 的 `sqlite3` 依赖、proxy 端口被另一浏览器占用。
- 新增 MIT `LICENSE`（保留上游版权）与 `NOTICE.md`（逐文件溯源）。

**未改动**

- `scripts/`、`references/`、`templates/` 与上游逐字节一致，逻辑零改动。
