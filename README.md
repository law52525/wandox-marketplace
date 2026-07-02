# Wandox 插件源（Marketplace）

一个仓库，五类 AI 工具通用的插件源。**打开一个会话，把下面对应的话发给 AI**，即可一键装好 **Wandox Collab**（skill + 远程 MCP，无需鉴权）。

**Claude Code**

> 在Claude Code中安装插件市场 https://github.com/law52525/wandox-marketplace.git，然后安装wandox-collab插件并安装插件里的wandox collab skill和wandox collab mcp

**Claude Desktop**

> 在Claude Desktop中安装插件市场 https://github.com/law52525/wandox-marketplace.git，然后安装wandox-collab插件并安装插件里的wandox collab skill和wandox collab mcp

**Codex**

> 在Codex中安装插件市场 https://github.com/law52525/wandox-marketplace.git，然后安装wandox-collab插件并安装插件里的wandox collab skill和wandox collab mcp

**Cursor**

> 在Cursor中安装插件市场 https://github.com/law52525/wandox-marketplace.git，然后安装wandox-collab插件并安装插件里的wandox collab skill和wandox collab mcp

**Qoder**

> 在Qoder中安装插件市场 https://github.com/law52525/wandox-marketplace.git，然后安装wandox-collab插件并安装插件里的wandox collab skill和wandox collab mcp

也可以手动安装（以 Claude Code 为例）：

```bash
claude plugin marketplace add https://github.com/law52525/wandox-marketplace.git
claude plugin install wandox-collab
```

只想要 MCP 工具、不装 skill？直接添加自定义连接器：

    https://ai-api.wandox.com/mcp/collab

> 本仓库由 wandox_mcp_collab 的 `packaging/scripts/build.mjs` 自动生成，请勿手改；改动请提交到源仓库后重新构建发布。
