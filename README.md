# Wandox 插件源（Marketplace）

一个仓库，五类 AI 工具（Claude Code / Claude Desktop / Codex / Cursor / Qoder）通用的插件源。

**打开一个会话，把下面这段话发给 AI**，即可一键装好 **Wandox Collab**（skill + 远程 MCP，无需鉴权）：

> 安装插件市场 https://github.com/law52525/wandox-marketplace.git ，然后安装wandox-collab插件并安装插件里的wandox collab skill和wandox collab mcp。安装完成后请明确告诉我：1) 是否安装成功；2) 提醒我重启本工具来加载插件（绝大多数 AI 工具需要重启才生效，新开会话通常没用）；3) 生效后我可以直接说什么来开始用，例如"创建一个协作板，主题是××，参与方是××"或"看看协作板 bd_xxx，我能补充什么"。

## 装好之后

装完后**重启一次 AI 工具**，插件的 skill 和 MCP 才会加载进来（实测绝大多数工具仅新开会话不够）——装完"没反应"通常只是还没重启，不是没装好。

生效后新开会话，直接说一句话即可开始，例如：

- 创建一个协作板，主题是「××」，参与方是××和××
- 看看协作板 bd_xxx，我能补充什么
- 把我这部分结论提交到协作板 bd_xxx

验证是否装好：问 AI"你现在有 collab_ 开头的工具吗"，能列出 collab_create_board 等即为生效。

## 更新到新版本

和安装一样，发一句话给 AI 即可。

通过插件安装的：

> 更新wandox-collab插件到最新版本（插件市场 https://github.com/law52525/wandox-marketplace.git ），更新完成后提醒我重启本工具以加载新版本。

当年手动配置过 collab skill / MCP 的（没走插件）：

> 从 https://github.com/law52525/wandox-marketplace 仓库拉取最新的 wandox-collab/skills/collab/SKILL.md ，覆盖我本地手动配置的 collab skill 文件，并确认 MCP 地址是 https://ai-api.wandox.com/mcp/collab 。完成后告诉我改了哪里。

（小建议：手动配置没有更新通道，不妨趁这次迁移到插件安装——用最上面的安装提示词装一次、删掉手动配置，以后更新一句话搞定。）

MCP 是远程服务，无需更新——工具能力在服务端发布后自动对所有用户生效。

另外，第一次往协作板提交内容时，AI 会顺口问一句你怎么称呼（同事在板上看到的名字），回个名字就行；只问这一次，之后换工具也不再问。

也可以手动安装（以 Claude Code 为例）：

```bash
claude plugin marketplace add https://github.com/law52525/wandox-marketplace.git
claude plugin install wandox-collab
```

只想要 MCP 工具、不装 skill？直接添加自定义连接器：

    https://ai-api.wandox.com/mcp/collab

> 本仓库由 wandox_mcp_collab 的 `packaging/scripts/build.mjs` 自动生成，请勿手改；改动请提交到源仓库后重新构建发布。
