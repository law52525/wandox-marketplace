# Wandox 插件源（Marketplace）

一个仓库，五类 AI 工具（Claude Code / Claude Desktop / Codex / Cursor / Qoder）通用的插件源。

**打开一个会话，把下面这段话发给 AI**，即可一键装好 **Wandox Collab**（skill + 远程 MCP，无需鉴权）：

> 安装插件市场 https://github.com/law52525/wandox-marketplace.git ，然后安装wandox-collab插件并安装插件里的wandox collab skill和wandox collab mcp。安装完成后请明确告诉我：1) 是否安装成功；2) 是否需要重启本工具或新开会话才能生效（如需要请提醒我操作）；3) 生效后我可以直接说什么来开始用，例如"创建一个协作板，主题是××，参与方是××"或"看看协作板 bd_xxx，我能补充什么"。

## 装好之后

多数工具需要**重启或新开一个会话**，插件的 skill 和 MCP 才会加载进来——装完"没反应"通常只是还没重启，不是没装好。

生效后新开会话，直接说一句话即可开始，例如：

- 创建一个协作板，主题是「××」，参与方是××和××
- 看看协作板 bd_xxx，我能补充什么
- 把我这部分结论提交到协作板 bd_xxx

验证是否装好：问 AI"你现在有 collab_ 开头的工具吗"，能列出 collab_create_board 等即为生效。

另外，第一次往协作板提交内容时，AI 会顺口问一句你怎么称呼（同事在板上看到的名字），回个名字就行；只问这一次，之后换工具也不再问。

也可以手动安装（以 Claude Code 为例）：

```bash
claude plugin marketplace add https://github.com/law52525/wandox-marketplace.git
claude plugin install wandox-collab
```

只想要 MCP 工具、不装 skill？直接添加自定义连接器：

    https://ai-api.wandox.com/mcp/collab

> 本仓库由 wandox_mcp_collab 的 `packaging/scripts/build.mjs` 自动生成，请勿手改；改动请提交到源仓库后重新构建发布。
