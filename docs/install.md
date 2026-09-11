# 安装与升级

## 通用前提

1. **Node.js 22+**：`node -v`。低于 22 时 `cdp-proxy.mjs` 会退回 `ws` 模块，需要额外安装依赖。
2. **开启远程调试**：Chrome `chrome://inspect/#remote-debugging` / Edge `edge://inspect/#remote-debugging` → 勾选 **Allow remote debugging for this browser instance**。
3. **浏览器偏好**（可选）：首次运行 `check-deps.mjs` 会在技能根目录生成 `config.env`。写入 `WEB_ACCESS_BROWSER=edge`（或 `chrome`）即固定默认浏览器；留空则每次启动询问。

## DSH

```powershell
git clone https://github.com/yufeiyufei888/browser-bridge.git "$env:USERPROFILE\.dsh\skills\browser-bridge"
node "$env:USERPROFILE\.dsh\skills\browser-bridge\scripts\check-deps.mjs"
```

- DSH 扫描的技能根：项目级 `<项目>/.dsh/skills`、`<项目>/.agents/skills`；用户级 `~/.dsh/skills`、`~/.agents/skills`（外加随包内置目录）。**不扫描** `~/.claude/skills` 与 `~/.codex/skills`。
- 放到用户级目录 → 所有项目、所有新会话都能用；新建会话时会自动出现在技能目录里，无需重启 DSH。
- 想只给某个项目用：放到该项目的 `.dsh/skills/browser-bridge`。

## Claude Code

```bash
git clone https://github.com/yufeiyufei888/browser-bridge.git ~/.claude/skills/browser-bridge
```

或作为插件市场安装（仓库内含 `.claude-plugin/`）：

```bash
claude plugin marketplace add https://github.com/yufeiyufei888/browser-bridge
claude plugin install browser-bridge@browser-bridge --scope user
```

Claude Code 会设置 `CLAUDE_SKILL_DIR` 环境变量，文档里的 `<SKILL_DIR>` 就是它。

## Codex CLI

```bash
git clone https://github.com/yufeiyufei888/browser-bridge.git ~/.codex/skills/browser-bridge
```

## 其他 Agent（通用）

SKILL.md 是事实标准：把仓库内容放到该 Agent 的技能目录，或直接把 `SKILL.md` 的内容交给它。技能根目录由 Agent 自己解析（`CLAUDE_SKILL_DIR` / 加载时注入的 base directory / 读取 `SKILL.md` 时所处的目录）。

## 升级

```bash
cd <技能目录>/browser-bridge && git pull
```

上游若发布新版本：本项目的 `scripts/` 是上游文件的原样副本，可用下列方式同步（保持逐字节一致，便于溯源）：

```bash
# 以 web-access 为上游，拉取脚本与本项目对比
git clone https://github.com/eze-is/web-access /tmp/web-access
diff -r /tmp/web-access/scripts ./scripts
```

## 卸载

1. 删除技能目录（如 `~/.dsh/skills/browser-bridge`）。
2. 结束常驻 proxy（见下节 `pkill` / `Stop-Process`）。
3. 如需彻底关闭，在浏览器里取消勾选远程调试开关。

## 自检与排错

| 输出 | 含义 | 处理 |
|---|---|---|
| `node: ok` | Node 版本满足 | — |
| `browser: ok (Chrome|Edge, port N)` | 找到已开启远程调试的浏览器 | — |
| `browser: needs decision`（exit 2） | 有多个浏览器开着调试，且未设偏好 | 写入 `config.env` 或加 `--browser edge` |
| `browser: 未连接`（exit 1） | 没有任何浏览器开启远程调试 | 打开 `chrome://inspect/#remote-debugging` 勾选开关 |
| `browser: 不一致`（exit 1） | 3456 端口被另一个浏览器的 proxy 占用 | 先结束旧 proxy 再重跑 |
| `proxy: ready` | proxy 已就绪，可以开始操作页面 | — |
