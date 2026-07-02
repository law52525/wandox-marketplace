# Wandox 插件源（Marketplace）

一个仓库，五类 AI 工具通用的插件源。把本仓库链接 add 进你的 AI 工具，即可一键安装 **Wandox Collab**（skill + 远程 MCP，无需鉴权）。

| 工具 | 添加方式 |
|---|---|
| Claude Code | `claude plugin marketplace add https://github.com/law52525/wandox-marketplace` → `claude plugin install wandox-collab` |
| Claude Desktop | Customize → Plugins → Add marketplace → 粘贴仓库链接 → 安装 wandox-collab |
| Codex | 把仓库链接加入 `.agents/plugins/marketplace.json` 源，或 `/plugins` 里 add marketplace |
| Cursor | Settings → Plugins → Add marketplace → 粘贴仓库链接 |
| Qoder | Plugins → Add marketplace → 粘贴仓库链接 |

只想要 MCP 工具、不装 skill？直接添加自定义连接器：

    https://ai-api.wandox.com/mcp/collab

> 本仓库由 wandox_mcp_collab 的 `packaging/scripts/build.mjs` 自动生成，请勿手改；改动请提交到源仓库后重新构建发布。
