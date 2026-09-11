# 归属与改动说明（NOTICE）

## 上游

- 项目：[web-access](https://github.com/eze-is/web-access)
- 作者：**一泽Eze**
- 许可：MIT（声明于上游 README 与 SKILL.md frontmatter）
- 本发行版基线 commit：`33eef84a55b1919396a80e7a55650a07bb83f590`（v2.5.4，2026-08-19）

## 与上游逐字节一致的文件（未做任何修改，SHA256 前 12 位）

| 文件 | SHA256 |
|---|---|
| `scripts/cdp-proxy.mjs` | `E18DF1A9C971` |
| `scripts/check-deps.mjs` | `6F08C9FDA6CA` |
| `scripts/browser-discovery.mjs` | `30F62A5BAD76` |
| `scripts/find-url.mjs` | `B5738E490DC6` |
| `scripts/match-site.mjs` | `75A5BD66DB94` |
| `references/cdp-api.md` | `38F853CE21AD` |
| `references/migration-2.5.3.md` | `AA7AEC6C6231` |
| `templates/config.env.template` | `DD498CB0ED2C` |

> 也就是说：**本项目的全部能力都来自上游脚本**；browser-bridge 贡献的是打包、文档与跨平台/跨 Agent 的适配。

## 本发行版改了什么

| 文件 | 改动 |
|---|---|
| `SKILL.md` | 基于上游 SKILL.md 改编：`name` 改为 `browser-bridge`；新增「适配与使用前提」（技能根目录解析、两种 shell 方言对照、环境要求、不要接管全部联网）；命令示例统一用 `<SKILL_DIR>` 占位符；Proxy API 段按 PowerShell 改写并注明 POSIX 等价写法；子 Agent 段改为 Agent 无关并补「模型默认继承主对话当前模型」（DSH 0.1.5-rc.1 起该继承缺陷已修复）；截图说明去掉具体工具名，改为「交给模型多模态输入」；新增「已知限制与排错」「归属与许可」 |
| `README.md` / `README.en.md` | 新写：定位、能力、安装（DSH / Claude Code / Codex / 通用）、前置条件、快速验证、API 速查、平台差异、限制、安全与合规 |
| `docs/` | 新写：分 Agent 安装细节、Windows PowerShell 差异与实测记录 |
| `templates/config.env.template` | 上游文件原样保留（未覆盖） |
| `.claude-plugin/` | 新写：按本仓库名称生成 Claude Code 插件/市场清单 |
| `.gitignore` / `LICENSE` / `NOTICE.md` / `CHANGELOG.md` | 新写 |

## 商标与免责

本项目与上游作者无隶属或背书关系。Chrome、Microsoft Edge、DeepSeek、Claude、Codex 等名称归各自权利人所有。
