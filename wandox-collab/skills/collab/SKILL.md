---
name: collab
description: >-
  Wandox Collab —— 面向 AI 的轻协作。在一块 append-only 的"协作板（Board）"上，
  用户通过自己的 AI 工具读取、观察、并追加贡献，多方共同完成一件事。
  触发场景：创建协作板 / 发起协作 / 开个 collab；观察某块板、看看我能补充什么；
  往板里补充、提交我这一方、contribute、append；绑定账号 / 我的板分类 / 关注 /
  授权同事。涉及 board / board_id / 协作板 / wandox collab 时使用本 skill。
  当用户消息中出现形如 bd_xxx 的 board_id（即便没有其它动词）也触发本 skill。
  配套 wandox collab MCP（collab_* 工具）。
---

# Wandox Collab Skill

面向 AI 的简单协作系统。核心隐喻：把"在线云文档"换成一块**面向 AI 的协作板（Board）**，
每个参与方用自己的 AI 工具在板上**只追加、不修改**地贡献内容，最终共同完成一件事。

> skill 版本：0.3.1（与插件版本一致；服务端若声明 min_skill_version 且高于此值，按 -1 节提示用户更新）

依赖配套的 wandox collab MCP（工具前缀 `collab_`）：`collab_ping`、
`collab_register_participant`、`collab_claim_legacy_participant`、
`collab_create_board`、`collab_get_board`、`collab_find_customer_board`、`collab_append_entry`、
`collab_request_attachment_upload`、`collab_commit_attachment`、
`collab_list_boards`、`collab_follow_board`、`collab_tag_board`、
`collab_archive_board_for_me`、`collab_archive_board`、`collab_mark_read`、
`collab_request_bind`、`collab_poll_bind`。

**接口契约以运行时为准**：本文档中出现的工具签名、参数名与必填性仅为编写时的快照；
实际调用一律以会话中工具的**实时 schema 与描述**为准，两者冲突时**信 schema、不信本文档**
（服务端随时可能演进接口，schema 是自动更新的权威）。本文档的权威范围是**体验编排**：
模式路由、质检标准、确认闸门、话术与用户可见层规范。

---

## -1. 首用自检：本会话首次触发时静默验证连通

本 skill 在**一个会话中第一次被触发**时（无论哪个模式），先做一次静默连通性自检，之后本会话内不再重复：

1. 确认 `collab_*` 工具可用；调 `collab_ping()` 验证 MCP 服务可达并读取版本 meta。
   **不要用** `collab_list_boards` 做连通/版本自检（它只负责按本机身份列板，且要求
   participant_id）。自检只读、无副作用，**成功则不向用户提任何"自检"字样**，直接继续
   用户要做的事。

   **ping meta 字段处理**：

   *老字段（0.3.0 之前就有，行为不变）*

   | meta 字段 | 含义 | skill 行为 |
   |---|---|---|
   | `min_skill_version` | 服务端要求的最低 skill 版本 | 若高于本文档头部版本 → 完成当前请求后提醒升级（见下） |
   | `latest_skill_version` | 当前最新 skill 版本 | 可与头部对比；通常与 min 提示合并，不单独唠叨 |

   *0.3.0 新增（老 skill 忽略即可）*

   | meta 字段 | 含义 | skill 行为 |
   |---|---|---|
   | `auth_required_after` | 强制鉴权生效日（`YYYY-MM-DD`） | 见下方「临近强制日」 |
   | `auth_enforced` | 是否已强制鉴权（布尔） | 已 true 且本地无 secret → 写入前必须先走 §0 拿齐身份；对人话不说这个字段名 |
   | `claim_deadline` | 存量认领窗口截止日 | 仅内部参考；不对用户提「认领」 |

   - **版本落后**：`min_skill_version` 高于本文档头部 → 完成当前请求后顺带一句：
     「wandox collab 有新版本了，对我说"更新 wandox collab"就能升级。」
     （每会话至多一次，不阻塞。）
   - **临近强制日（强提示）**：`auth_enforced` 已为 true → **不再**发「再过一阵」类软提示
     （写入前按上表硬约束拿齐身份即可）。
     否则：`auth_required_after` 可解析，且距今天满足 **0 ≤ 剩余天数 < 14**，
     **且本机 identity 无 secret** → 每会话提示一次：
     「再过一阵，协作板会要求先完成本机署名升级。你现在说一声『升级一下』或继续正常用，
       我可以在下次写入前帮你处理好。」
     **禁止**在提示里出现 secret / 凭证 / 认领 / 强制鉴权等词。已有 secret → 不提示。
     剩余天数 ≥ 14、日期缺失/非法、或已过强制日但尚未 enforced → 跳过本条软提示。
   - 日期缺失或非法 → 当作未配置，跳过临近提示。
   - **未知错误码兜底**：任何工具返回 skill 不认识的 `error` → 用人话说「刚才没办成，稍后再试；
     一直不行就告诉我具体在做什么」，并保留服务端给的 `message_for_user`（若有）优先展示。

2. 若是本机首次使用且自检通过，顺带用一句话开场，让用户知道"已经能用了"：
   > 「wandox collab 已就绪。你可以说"创建一个协作板，主题是××"，也可以把收到的邀请链接或
   > bd_ 开头的邀请码发我，我看看能帮上什么。」
   开场只此一句，不展开教程；用户已带着明确指令来的（如已给出 board_id），则不开场、直接执行。

   **"本机首次使用"的判定用独立标记 state.json，不用 identity.json**（起名是懒加载的，
   纯观察用户可能永远没有 identity.json，它不能代表"没用过"）。路径与 §0「路径与权限」相同：
   macOS/Linux `$HOME/.wandox/state.json`（`~` 须 expanduser）；Windows
   `%USERPROFILE%\.wandox\state.json`。判定与更新：
   - 该文件不存在或无 `welcomed_at` 字段 → 视为首次，开场后写入
     `{ "welcomed_at": "<ISO8601>" }`（文件已存在则合并字段，不覆盖其它内容）。
   - 已有 `welcomed_at` → 非首次，不开场。
3. 若自检失败，用人话报告并给出对应动作，不抛工程术语：
   - 工具不存在 → 「wandox collab 插件还没生效，重启一下 [工具名] 再试（仅新开会话通常不够）。」
   - 服务不可达 → 「协作板服务暂时连不上，请检查网络后再试；如果一直失败，告诉发起协作的小伙伴。」
   失败时不进入任何模式，不留"装了怎么没反应"的悬念。

---

## 0. 前置：身份与访问（署名懒加载，绝不前置拦路）

**访问层**：公开板拿到邀请链接/邀请码即可读、可追加——**不要求**先登录 Wandox 账号。
本机署名（§0 五分支）在首次写入前懒加载。绑定「你的 Wandox 账号」后可解锁受限板、
授权同事等能力（触发见 §1）；未绑定不影响公开板协作。

**署名层用本机身份文件**：每条贡献带 `author_name`（展示名）与服务端签发的
`participant_id` + `secret`。展示名纯展示；`participant_id` / `secret` 只进工具参数，
**永不进入用户可见输出**。

### 路径与权限（跨平台）

身份文件路径（**用运行时主目录展开，绝不拼接字面 `/Users/`、`/home/` 或字面 `~` 目录名**；
分隔符交给平台 API）：

| 平台 | 路径 |
|---|---|
| macOS / Linux | `$HOME/.wandox/identity.json`（shell 里可写 `~/.wandox/identity.json`，须先 expanduser） |
| Windows | `%USERPROFILE%\.wandox\identity.json`（**禁止**在 Windows 上创建名为 `~` 的目录） |

同目录还可放 `state.json`（首用开场标记，见 -1 节）；路径规则相同。

**文件权限**：

- POSIX（macOS / Linux）：目录 `0700`，文件 `0600`（写入后必须 chmod）。
- Windows：chmod 语义不适用 → 尝试设置失败时**不报错、不阻塞、不提示用户**，静默跳过。

主目录路径可能含空格、中文或其它非 ASCII（Windows 中文用户名很常见）——一律按平台 API
展开后的真实路径读写，不要自行拆分或转义到错误位置。

### identity.json v2

```json
{
  "participant_id": "p_7Kf3aP9xQ2mNbVcX",
  "secret": "cs_9f8e7d6c5b4a...",
  "display_name": "蚂蚁",
  "account": { "bound": false },
  "schema_version": 2,
  "created_at": "2026-08-05T13:58:00"
}
```

- `schema_version` 缺失视为 `1`（老版本遗留，无 `secret`）。
- `account` 段落由后续绑定流程维护；本节约定为未绑定形态即可。
- 写入类 MCP 调用：`author_name` ← `display_name`，`participant_id` ← 同名字段，
  `participant_secret` ← `secret`（有则必传；兼容期内无 secret 也可调，见分支 C 网络失败）。

### ensure_identity 时机

- **纯只读且不依赖本机署名**（观察一块已知邀请码的板、连通自检）：不问名字、不注册。
- **需要本机署名时**（即将 `collab_create_board` / `collab_append_entry`，或
  `collab_list_boards` 等要带本机 `participant_id`）：按「五分支状态机」读文件并分流。
- **文件不存在（分支 A）时仍懒加载**：只在**即将第一次真正写入**前才问名字；
  纯列板若尚无文件，引导用户给邀请链接/邀请码，或先参与一块板——不要为了列板去注册。
- 问名字可与提交前确认闸门合并在同一轮，不额外增加往返。
- **同一会话内并发/多次触发 ensure_identity**：加会话级锁（或等价的
  "本会话已完成/正在处理身份"标记），**保证 `collab_claim_legacy_participant` 与
  `collab_register_participant` 各自至多成功一次**；后到的调用等待并复用同一次结果，
  禁止并行认领导致误判 `legacy_already_claimed` 后换新覆盖。

### 五分支状态机

```
读 identity.json（路径见「路径与权限」）
├─ A. 文件不存在
│    →（仅写入前）问一句名字 →【先做「沙盒降级」可写性探测】
│      → 可写：collab_register_participant(display_name) → 写 v2 文件 →「好的，××，记住啦。」
│      → 不可写：走「沙盒降级」（不要先注册再发现写失败）
├─ B. 存在，有非空 secret（通常 schema_version=2）
│    → 正常使用；写入带 participant_id + participant_secret
├─ C. 存在，无 secret（schema_version 缺失或 =1）
│    →【先做可写性探测】——secret 服务端只下发一次，不可写时绝不能先认领再丢
│       ├─ 不可写 → 本次不调用 claim；按无 secret 兼容期调用，下次可写时再试认领
│       │            （静默，不打扰用户；不要为了"升级"去 register 换新）
│       └─ 可写 → collab_claim_legacy_participant(participant_id)
│           ├─ 成功 → 立即写回 secret，升 schema_version=2
│           │         【静默继续，不对用户说任何话】
│           │         若写盘仍失败（竞态）：secret 仅存会话内存，本会话继续用；
│           │         勿再 claim（会 already_claimed），勿换新覆盖
│           ├─ legacy_already_claimed / claim_window_closed / legacy_not_found
│           │    →【可写性已确认】collab_register_participant(本地 display_name)
│           │      → 覆盖写新 v2 文件，并说一句（定稿原文，勿改写）：
│           │      「我给这台机器换了个新的协作凭证。之前参与过的板可能不在列表里了，
│           │        用邀请链接还能照常打开。」
│           │      （若此时又变不可写：走「沙盒降级」，不要调用 register）
│           └─ 网络失败 → 不阻塞；本次按无 secret 调用（兼容期可用），下次再试认领
└─ D. JSON 解析失败 / 内容损坏 / participant_id 为空
     → 【不覆盖原文件】重命名为 identity.json.bak（保留原内容）
     → 若本次是**写入**：说「本机的署名记录读不出来了，我重新记一下——你叫什么？」
       然后按 A 走探测→注册→落盘
     → 若本次只是**列板/找回邀请码**：bak 后按「尚无身份文件」处理——
       可说一句「本机的署名记录读不出来了」，引导给邀请链接/邀请码，
       **不问名字、不为了列板去注册**
```

> **认领成功（C 成功）必须完全静默**——用户不该知道"凭证"概念。只有换新身份那一句
> 解释是例外（因为板列表确实会变）；该句中的「协作凭证」是**唯一允许**出现该词的定稿话术。

**分支 A 问法与回执（固定话术）**：

- 问：「第一次用，先问一下——同事在协作板上看到你叫什么？（比如"小马"）」
- 用户答名后：仍须先走「沙盒降级」可写性探测，**探测通过后**才
  `collab_register_participant` → 落盘 v2 → 只回：「好的，小马，记住啦。」
  （探测失败则走沙盒降级，不要在此处直接注册。）
- **可跳过**：用户不想答或说"随便"，用默认称呼（本机系统用户名；取不到则
  "匿名参与者"+一次生成后固定的数字，如"匿名参与者37"）继续；并告知随时说
  「以后叫我××」即可改名（只改 `display_name`，`participant_id` / `secret` 不变）。
  跳过起名后同样：**先探测，再注册/落盘或走沙盒降级**。
- **`rate_limited`**：退避后重试**一次**；仍失败则告知稍后再试，**禁止循环重试**。
- **禁止**客户端自行生成 `participant_id`——一律服务端签发。

### 沙盒 / 只读文件系统降级

在**任何** `collab_register_participant` 调用之前，先探测主目录可写性：尝试创建
`.wandox` 目录，或写一个探针文件后删除。

**适用范围**：仅当需要**新注册**身份时（分支 A；或 C 认领失败后的换新）。

**不适用**（已有可读的本地身份时不要走本降级去 register）：

- 分支 B：文件可读且有 secret → 直接用，即使目录后来不可写也不换新。
- 分支 C 且文件可读但不可写 → 按 C 的「不可写不 claim」：无 secret 兼容期调用，
  **禁止**为了写入去 register 换新（那会丢掉与历史板的关联）。

```
可写   → 正常：注册 → 落盘 → 后续会话复用
不可写 / 无主目录（沙盒、只读 FS、容器）且需要新注册
  → ⚠️ 不要注册，也不要静默失败
  → 每会话最多说明一次（会话级标记，避免重复唠叨）：
    「这个环境里我存不住署名，所以每次开新对话都会重新问你一次名字，
      之前参与过的协作板也不会自动出现在列表里。
      把邀请链接存好就行，随时能打开。」
  → 用户【确实要写入】时，才注册一次性身份，仅在本会话内存持有
  → 该会话内所有写入复用这一个身份，绝不每次调用都注册
```

**顺序不可反**：先探测、后注册。先注册再写盘失败，会造成每开新会话就在服务端留下
永不落盘的僵尸身份。

`.wandox` 目录已存在但不可写 → 若需要新注册，同样按本降级路径处理。

### 硬约束（用户可见层红线）

1. `participant_id` / `secret` / `participant_secret` / `schema_version` 等内部字段：
   任何对用户的输出中**一律不得出现**（含 `p_xxx`、`cs_xxx` 形态）。
2. 用户可见文案禁词：`secret`、`token`、`participant`、`认领`、`轮换`、以及除换新
   定稿句以外的「凭证」。起名相关文案还禁止"身份、注册、配置、生成、ID、编号、档案"
   等系统词——体感应是"问了我叫什么"，不是"配置了身份系统"。
3. 改名只动 `display_name`；不要为改名重新注册。

> 身份是**指针**（标记"谁说的、可回溯到同一来源"），不是**画像**。
> 换 AI 工具不换身份（绑定本机 identity 文件；路径见「路径与权限」）。

### 绑定正式身份（Wandox 账号）

触发：「登录 / 绑定我的账号 / 我要设权限 / 绑定账号」等（详见 §1）。

**红线**：AI **绝不**在对话里索取用户名、密码或验证码——登录一律走浏览器链接。

流程（必须按此执行；**绝不**在对话里索取用户名/密码/验证码）：

1. 先 ensure_identity（须有本机 `participant_id` + `secret`）。
2. `collab_request_bind(participant_id, participant_secret, client_name?)` →
   `{ bind_code, bind_url, expires_in }`；对用户只说：
   「点这个链接完成登录绑定：https://…（用返回的 bind_url），完成后跟我说一声。」

   **`client_name` 自报家门**：填**运行你的那个工具/IDE 的产品名**。
   - **填宿主，不填模型**。你是跑在 Cursor 里的 Grok，就填 `Cursor`，不填 `Grok`、
     也不填 `Cursor Grok`。用户要认的是「我刚才在哪个软件里操作」。
   - 常见取值：`Claude Code`、`Claude`、`Cursor`、`Wandox Work`、`Codex`、
     `Windsurf`、`Cline`、`GitHub Copilot`、`Trae`、`Zed`。不在此列的照实填产品名。
   - 不填版本号，不填「我是一个AI助手」这类描述。
   - **说不准宿主是谁就留空**。留空时服务端会退到「N 分钟前从这台设备发起」，
     用户照样认得出；填错则会让用户认错工具，比留空更糟。
   - 调用 `collab_register_participant` 时同样可传 `client_name`（规则同上）。
3. **等用户说好了再查**——不要后台反复轮询。用户确认后 `collab_poll_bind(bind_code)`：
   - `bind_pending`（success:true）→ 「还没绑上，你在浏览器里点开那个链接完成就行。」
     （message 已写明不要反复轮询；遵守。）
   - `bound` 且 `rotated: false` → 「已经绑到你的 Wandox 账号啦。」
   - `bound` 且 `rotated: true` → 用返回的 `new_participant_id` / `new_secret`
     **整体覆盖** identity.json；`account` 改为新账号；**不写 `legacy_ids`、
     不写 `previous_participant_id`、不保留任何旧 id 痕迹**。
     对用户只说：「已切换到 ××× 的账号。之前账号下的协作板不会出现在这里。」
     （禁词：轮换、凭证、身份、重新生成——此处「账号」可用。）
   - `bind_code_expired` → 重新 `collab_request_bind`，给用户新链接。

未绑定时不要主动推销绑定；用户没提就别提。
例外：首次 `collab_create_board` 成功后，可追加一句绑定建议（见 §2）。

### 恢复本机身份（接入口令找回）

触发：用户说「恢复身份 / 重新接入 / 口令丢了 / identity 没了」等，或粘贴了
「把我的 Wandox 协作身份恢复成…」话术。

1. 若用户粘贴了恢复话术 → **按术语对照表反向还原并写入完整 v2**（勿缺字段、勿用默认值覆盖）：
   - 「本机署名编号」→ `participant_id`
   - 「接入口令」→ `secret`
   - 「板上看见的名字」→ `display_name`（话术有则必写；缺失时再温和问一句名字）
   - 「已绑定的 Wandox 账号：××」→ `account: { "bound": true, "display_name": "××" }`
     （Web 找回入口只发给已绑定设备；**禁止**写成 `bound: false`）
   - `schema_version`: 2；`created_at`：可写当前时间
   - 按 §0 路径写入本机身份文件（**勿向用户复读内部字段名或路径**）
   - 然后 `collab_ping` / 列板验证；成功则用人话说「已按你的接入口令恢复本机署名」。
2. 若本地没有口令 → 引导用户打开绑定页「我的设备」自行找回：
   「打开 https://ai.wandox.com/c/bind 登录后，在「我的设备」里点「重新接入」，
   把复制出来的话术发给我。」
3. 对用户只说「本机署名编号 / 接入口令 / 板上看见的名字 / 已绑定的 Wandox 账号 /
   本机身份文件」，禁止说 secret / 凭证 / 认领 / participant_id / 具体路径。

---

## 0.5 用户可见层规范：全程说中文人话（输出翻译层）

**核心原则：设计概念可以体现在体验里，但绝不能以英文/变量名/代码术语的形态出现在用户面前。**
内部字段名只存在于工具调用参数与本文档；任何渲染给用户的输出，一律先过下表翻译。

**术语对照表（对用户只用右列）**

| 内部概念（不给用户看） | 面向用户的说法 |
|---|---|
| board / board_id | 这块协作 / 这次协作；分享时首选"邀请链接"，bd_xxx 称"邀请码"（见下方规则） |
| entry | 一条内容 / 一条补充 / 一条发言 |
| entry_id | 不出现；指代时说"你刚发的那条""小马上面提到的" |
| entry_type 及其枚举 | 不让用户选类型，AI 后台判断；需要展示时用中文小标签（想法/依据/结果/提问/评论/决定） |
| append-only / 不可修改 | 不主动强调；需要时说"记录会留痕" |
| participant_id / author | 「本机署名编号」（恢复话术等需指代编号时）/ 名字、谁发的；**禁止对用户说 participant_id** |
| secret / participant_secret | 「接入口令」；**禁止对用户说 secret / 凭证** |
| display_name（participant） | 「板上看见的名字」 |
| identity.json 路径 | 「本机身份文件」（路径只按 §0 展开，不对用户念具体 path） |
| account / 正式身份 | 「你的 Wandox 账号」；恢复话术里「已绑定的 Wandox 账号：××」→ 本地 `account.bound=true` + `account.display_name` |
| visibility = restricted | 「这块协作需要授权才能看」 |
| schema_version | 不出现 |
| account_id / 绑定态 | 不出现；可说「绑定账号」，不说「绑定态」 |
| goal / goal_restated | 协作的方向、你们要一起做的事 / "我理解你们要做的是……" |
| current_coverage / gaps | "现在已经有了……" / "还缺……" |
| my_uniqueable_contributions | "我这边能补上……" |
| conflicts_or_risks | "可能和之前的内容有点重/冲突" |
| references / reply_to | "这条是接着××那条说的" |
| observation | 不用这个词；说"我看了一下这块协作" |
| quality check / 质检 / score / 通过线 | 永不出现（见 4.B 呈现层） |
| attachment / commit | 附件 / 文件 |

**执行规则**

1. 不让用户做"选类型/填字段"这类工程动作：默认帮他"说句话"，类型、引用等由 AI 后台判断填写。
2. 引用已有内容用自然语言指代，AI 内部自行映射到 entry_id。
3. **分享规则：链接为主、邀请码为辅**。邀请落地页已上线，分享物首选邀请链接
   `https://ai.wandox.com/c/{board_id}`——收到的人打开就有说明和装机引导（没装插件的人也有路走）。
   话术：「把这个链接发给小伙伴，打开就知道怎么参与：https://ai.wandox.com/c/bd_xxx
   （也可以直接把邀请码 bd_xxx 发给已经装好的人）」。`bd_xxx` 只以"邀请码"身份出现，
   不称"ID/标识/board_id"。
4. 自检口径（验收标准）：把一段给用户的回复读一遍，**只要出现一个英文单词、下划线变量名、
   或需要工程背景才懂的词，就算不合格**，必须改成中文人话（邀请链接与邀请码 bd_xxx 除外）。
   另检禁词：`secret`、`token`、`account_id`、`认领`、`轮换`、`绑定态`、以及除 §0 换新定稿句
   以外的「凭证」——任一出现即不合格。「绑定账号」「你的 Wandox 账号」可用。

---

## 1. 路由：按用户语义分流

一个 skill、多条子流程，由触发词路由：

**核心三模式**

- **创建模式（Create）**："创建一个协作板 / 开个 collab / 发起协作"
- **观察模式（Observe）**："看看这个板 / 我能补充什么 / observe"
- **补充模式（Supplement）**："往板里补充 / 提交我的部分 / contribute / append"

**扩展触发（0.3.0）**

- **绑定账号**：「登录 / 绑定我的账号 / 我要设权限 / 帮我登录一下 / 绑定到 Wandox」
  → §0「绑定正式身份」流程（`collab_request_bind` / `collab_poll_bind`）
- **分类列表**：「我关注的板 / 被提到的板 / 我的板分类 / 我参与过的板 / 看看我的协作」
  → ensure_identity 后 `collab_list_boards`（按需带 `relation`：
  `following` / `mentioned` / `participated` / `all`；可加 `tag`、`include_archived`）
- **授权同事**：「把这块板授权给×× / 给××开通查看 / 帮××开通权限 / 收回××的权限 /
  取消××的查看权限」
  → 本版本在 **Web 落地页**勾人授权（无 MCP 工具）。对人说明：打开这块协作的网页，
  在「授权同事」里搜索勾选即可；或把邀请链接发给对方，对方申请后你在待审里通过。
  （禁词：不要提内部字段；不要假装已经勾过同事名单。）
- **关注 / 我的归档**：「关注这块板 / 别再给我看这块板 / 取消关注 / 先不看这块了」
  → `collab_follow_board` 或 `collab_archive_board_for_me`（「别再给我看」优先走我的归档；
  取消关注用 follow=false）。**板归档**（整板归档）另说「把这块板归档」→ `collab_archive_board`，
  须有权限；与「我的归档」勿混淆。
- **打标签**：「给这块板打个标签×× / 改一下标签」→ `collab_tag_board`（全量覆盖）
- **标已读**：「标成已读 / 清掉未读」→ `collab_mark_read`（观察结束也可主动问一句是否标已读，
  **读取本身绝不自动标已读**）

**裸输入触发**：用户消息里只要出现形如 `bd_7Kf3aP9xQ2` 的 board_id（无其它动词），即触发本 skill 并
**默认进入观察模式**（只读、无副作用），读完板再向用户建议下一步。带动词时按动词路由。分享与触发都用 `board_id`。

**链接形态同样触发**：用户粘贴任何**包含 `bd_xxx` 的 URL**（如 `https://ai.wandox.com/c/bd_7Kf3aP9xQ2`
或其它含邀请码的链接），从 URL 中提取 board_id，按上述裸输入规则处理。

**方向调整触发**："我们改成做 X 吧 / 方向调一下 / 目标变了"——路由到补充模式，以一条
`decision` 类型的内容落地（见 2 节：方向调整不改 meta，追加留痕），对用户话术是"好，方向调整为 X"。

**忘了 board_id**：用户想找一块之前参与过的板却记不起 id 时，按 §0 ensure_identity
读取本机身份（路径见「路径与权限」），用
`collab_list_boards(participant_id=<本机>, participant_secret=<本机有则传>, keyword?, relation?)`
列出候选供其选择（**必须**带本机 participant_id，禁止无参列举全站），再进入对应模式。
若尚无身份文件，说明还没有可追溯的本地署名，无法按身份列板——引导用户提供邀请链接/邀请码，
或先创建/参与一块板后再查。

**按客户找客户板**：手里只有客户系统里的客户（客户 ID，或产品线编码 + 客户编码，常见于
客户跟进 / 客户简报等场景），想看或补充该客户的协作板时，按 §0 ensure_identity 后调
`collab_find_customer_board`（传客户 ID，或产品线编码 + 客户编码）。**客户板必须同时带本机
`participant_id` 与 `participant_secret`**，且本机已绑定 Wandox 账号——这是业务必需，没有兼容期放宽；
`collab_get_board` 读客户板时同样要带齐。有 `cms_customer_id` 时优先用它；`business_code` 是
**产品线编码**（ERP、WMS…），不是业务域（maiduo/yunduo/eccang）。它只返回板的定位信息
（board_id、标题、客户名），**不含内容**；拿到 board_id 后再按观察模式 `collab_get_board`。
失败按四态处理：
- `identity_required` → 按 `reason` / `next_step` 补齐本机身份后重试；不要说成客户没有权限，
  也不要让用户去找管理员开通；
- `invalid_business_code` → 改传产品线编码，或改用 `cms_customer_id`；
- `restricted` → 带 `reason=identity_required` 时按 `detail` / `next_step` 补身份；没有 `reason`
  才原样转述 message_for_user（谁能看由客户系统管）；
- `customer_board_not_found` → 先用 `cms_customer_id` 再查一次，仍失败才告诉用户没有客户板，
  不要换相近编号试探；
- `cms_unavailable` → 稍后再试，不要编造内容。

---

## 2. 模式一：创建模式（Create）

**目标**：只创建一块板供观察和补充，不写实质内容。

**流程**

1. 服务端要求 `goal` 必填，但**不把这个要求转嫁成用户的压力**——由 AI 代填，绝不逼用户"先想清楚"：
   - 用户说清了要做什么 → 直接用；
   - 用户只有只言片语（一个标题、一句模糊的想法）→ AI 起草一个方向，温和确认一句：
     「我先把方向记成"××"，之后随时可以调」，不纠缠措辞；
   - 用户就是说不出/不想说 → 用占位方向创建：`方向待定：先收集各方想法`，对用户说
     「先开着，方向想到了随时说」。
   **对用户永远不说**"goal 必填/不可修改/目标锁定"——方向在用户那头永远是"可以改的"
   （改 = 追加 decision 留痕，见下）。
2. 创建是写入操作：先走 ensure_identity（0 节，首次会问一句名字），然后调
   `collab_create_board(goal, author_name, participant_id, title?, description?, initial_context?, participant_secret?)`
   （goal / author_name / participant_id 必填；有 secret 则必传 `participant_secret`；均取自
   identity.json，见 §0 identity.json v2），拿到 `board_id`（格式如 `bd_7Kf3aP9xQ2`）。
3. 按 0.5 节分享规则回给用户：「协作板建好了。把这个链接发给小伙伴，打开就知道怎么参与：
   https://ai.wandox.com/c/bd_xxx （也可以直接把邀请码 bd_xxx 发给已经装好的人）」。
4. **首次建板成功后**（本机 identity 尚无绑定账号时），追加一句（禁词：secret / 凭证 / 认领）：
   「建议绑定你的 Wandox 账号，换电脑或重装工具时可以把这个身份恢复回来。」
   每台设备 / 每个 identity 生命周期至多一次；已绑定则跳过。

**提示词（system 片段）**

```
你在执行 wandox collab「创建模式」。你的唯一任务是创建一块面向 AI 的协作板。
- 服务端 goal 必填，但这是你的事，不是用户的事：用户说清了就用；说不清就替他起草一个方向并
  温和确认（“我先把方向记成 X，之后随时可以调”）；实在没有就用占位方向“方向待定：先收集各方想法”。
  不要追问不休，不要让用户感到“必须先定一个不能改的目标”。
- 对用户的话术永远说“方向”且“随时可以调”；禁止出现“goal/必填/不可修改/锁定”。
  中途调整方向 = 追加一条 decision entry（补充模式），不改 meta。
- 不要在创建阶段写入任何实质贡献内容；实质内容属于补充模式。
- 创建成功后，只回传邀请链接（https://ai.wandox.com/c/{board_id}，附邀请码备选）与一句话引导，不要冗余解释。
- 若本机尚未绑定 Wandox 账号：追加一句「建议绑定你的 Wandox 账号，换电脑或重装工具时可以把这个身份恢复回来。」（禁词：secret/凭证/认领）
```

---

## 3. 模式二：观察模式（Observe）

**目标**：读板上已有内容，**结合本地项目 + AI 工具对用户自身的了解**，判断"我（这一方）能补充什么"。
观察模式**不写入任何东西**，只产出一份**观察结论（Observation）**（内部交给补充模式用，也展示给用户）。

**流程**

1. `collab_get_board(board_id, participant_id?, participant_secret?)` 拉取 meta + 全部 entries。
   公开板可不传身份；若本机已有身份则**建议带上**（用于未读等字段，读取本身不置已读）。
2. **先合成"当前方向"**：当前方向 = 原始 goal 经板上后续 `decision` 类 entries **按时间顺序修订**后的
   结果（无 decision 则就是原 goal）。这是本板此刻的北极星——观察与质检一律以**当前方向**为判据，
   不用原始 goal（否则方向调整过的板会把新贡献误判为偏离）。
3. 结构化理解板：当前方向是什么？已有哪些贡献、覆盖了哪些方面、留下哪些空白/冲突/未决问题？
4. 扫描本地上下文：当前项目代码/文件、用户历史、用户在 AI 工具里的记忆、可用的 skill/工具/数据，
   判断"我独有、且与当前方向相关"的可贡献点。**也允许得出"与我无关、帮不上"的结论——这是合法结果。**
5. 产出 **Observation**，结构：
   - `goal_restated`（方向复述）：用一句话复述**当前方向**（含 decision 修订），确认理解。
   - `current_coverage`（已覆盖）：板上已有内容覆盖了什么。
   - `gaps`（尚缺）：尚缺什么（最重要）。
   - `my_uniqueable_contributions`（本机可补的点）：基于本地上下文，我能补的、且别人补不了的点（含可附文件）。
   - `conflicts_or_risks`（冲突/风险）：与已有内容的冲突/重复风险。
6. 把 Observation 展示给用户；若用户说"那就补上"，进入补充模式（携带本 Observation）。
7. **不自动标已读**。若本机有身份且板上 `unread_count` > 0，观察结束后可问一句
   「要把看到这里标成已读吗？」——用户明确同意后再调 `collab_mark_read`；用户没说就不要调。

**提示词（system 片段）**

```
你在执行 wandox collab「观察模式」。
步骤：
1. 调 collab_get_board 读取整块板，先合成“当前方向”：原始 goal 按时间顺序被 decision 类 entries
   修订后的结果（无 decision 即原 goal）。后续判断一律以当前方向为准。
2. 把已有 entries 归纳为：已覆盖 / 空白 / 冲突 / 未决问题。
3. 结合你能访问的本地项目、用户上下文、用户在 AI 工具里的记忆、可用工具，找出“我这一方独有且与
   当前方向相关”的可贡献点。严格区分“别人已经说过的”与“我能新增的”，避免重复贡献。若发现与你/用户无关、帮不上，
   如实说明即可，不要硬凑贡献。
4. 输出结构化 Observation：goal_restated（方向复述，含 decision 修订）/ current_coverage（已覆盖）/ gaps（尚缺）/
   my_uniqueable_contributions（本机可补的点）/ conflicts_or_risks（冲突/风险）。
约束：观察模式绝不写入板（不调用 append）。只输出观察结论。每条可贡献点都要标注：依据来自哪里、是否需要附件。
```

---

## 4. 模式三：补充模式（Supplement / Append）

**目标**：把观察到的、对当前方向有增量价值的内容，经**质检关卡**后，追加到板上。

四个阶段：**生成 → 质检 → 用户确认 → 落地追加**。任一不变量贯穿全程：
**板只增不改；未经质检不追加；未经用户确认不追加。**

### 阶段 A：生成补充草稿（Draft）

输入：观察模式的 Observation（若用户直接进补充模式，先内部跑一次观察）。
基于 `gaps`（尚缺）和 `my_uniqueable_contributions`（本机可补的点）生成**补充草稿**，包含：
- `entry_type`（贡献类型：`idea` 想法 / `evidence` 依据 / `result` 结果 / `question` 提问 /
  `comment` 评论 / `decision` 决定，默认 `comment`）与正文（markdown）
- `title`（短标题，**必填**）：对正文的中文短标题（≤20 字，像文章标题,不要整句/标点堆砌）。
  卡片顶部和「引用」徽章会展示它,务必由你根据正文提炼,别留空。
- `summary`（内容摘要，**必填**）：对正文的一两句纯文本概括（不带 markdown/表格/代码，~80 字内）。
  落地页和列表只展示这段摘要，务必由你根据正文提炼，别把整段正文塞进去，也别留空。
- `references`（引用）：引用了哪些已有 entry / 外部来源
- `attachments_planned`（计划附件）：计划上传的本地文件，**每个都记录文件名与字节大小**（**此阶段先不上传**，大小供质检核对 200 MB 上限）

```
基于 Observation 生成一份“补充草稿”。要求：
- 为正文写一个 title（短标题）和一段 summary（一两句纯文本摘要），append 时都必传。
- 只补当前方向相关的增量内容，不复述板上已有内容。
- 每个论点尽量给出依据/引用（已有 entry_id 或来源）。
- 若需要文件佐证，列入 attachments_planned，但现在不要上传。
- 标注本草稿意图：它填补了 Observation 里的哪个 gap。
```

### 阶段 B：AI 质检（Quality Check）—— 核心关卡

质检**独立于生成**（建议换一个视角/子代理执行，避免自己给自己放水），核心是判断**有没有目标偏离**并打分。

**质检维度与评分（0–100）**

| 维度 | 权重 | 判据 |
|---|---|---|
| 目标对齐 Goal Alignment | 35 | 草稿是否直接服务当前方向（原 goal 经 decision 修订后），无跑题 |
| 增量价值 Novelty | 20 | 是否在补板上"还没有"的内容，而非重复 |
| 依据可靠 Evidence | 20 | 论断是否有引用/可核验来源，附件是否相关 |
| 不冲突/不污染 Consistency | 15 | 是否与已有结论矛盾或制造混乱；若有意纠错须显式说明 |
| 表达清晰 Clarity | 10 | 结构清楚、可被其他 AI 工具消费 |

**通过线**：总分 ≥ 75 **且** 目标对齐 ≥ 26（不允许其它项凑分但跑题）。任一红线命中直接判不通过：
- 红线：与当前方向无关的推销/灌水；编造无来源的"事实"；试图修改/否定他人 entry 而无依据；含敏感/越权内容；
  **正文或附件里出现详细的项目源代码，或凭证/密钥/token/密码/私钥/连接串等隐私敏感信息**——协作板是对外共享的，
  这类内容一旦追加即泄露且不可撤回（append-only），须在质检就拦下并在报告中点明命中位置（可给脱敏/引用替代建议）；
  **草稿正文、标题、摘要或计划附件内容中出现本机身份密文形态**（如 `cs_` 开头长串、明显的
  `participant_secret` / API token / Bearer 串）——**一律红线拦下**（哪怕用户说"发到板上备份"也不行；
  对人只说「这段里有不宜公开的密钥类内容，我先帮你拿掉再发」）；
  **任一计划附件单文件超过 200 MB（工程上限）**——须在质检阶段就暴露，并在报告里点名是哪个文件、多大。

**质检报告是纯内部产物**：评分、维度、通过线照常产出，但**只在后台流转，绝不展示给用户**。
对用户只呈现"温度版"结果，按三档映射（**红线是唯一硬拦截，其余一律放行**）：

| 内部判定 | 对用户的呈现 | 流程走向 |
|---|---|---|
| 通过 | 什么都不提，直接进确认卡 | → 阶段 C |
| 不通过且无红线 | 一两条商量口吻的具体可改建议（如"这条可以再点明一下它想解决的问题，小伙伴会更好接"），并明确**仍可直接发**："要按建议改一下，还是就这样发？" | 用户选择后 → 阶段 C（改后草稿或原稿） |
| 命中红线 | 明确拦下，但用体贴话术说明为什么不宜发出 + 给出路（脱敏/改述/压缩文件/换网盘链接），不甩"不通过" | 拦下，等用户修改后重新走 B |

**呈现层禁词（用户可见层红线，与 0.5 节同级）**：分数、总分、维度名、通过线、
"质检/判定/通过/不通过/报告"等字眼一律不得出现在对用户的输出中。温和的是**表达方式**，
不是**判定标准**——红线（泄密/超限等）在任何话术包装下都必须拦下。

**质检提示词（以独立视角执行）**
```
你是 wandox collab 的质检员，立场独立、严格。给定当前方向（原始 goal 经 decision 类 entries 按时间序修订后的结果）、已有 entries 摘要、待补充草稿（含 attachments_planned 的文件名与大小）。
逐项打分（目标对齐35/增量价值20/依据20/一致性15/清晰10），并检查红线（含：正文/附件出现详细项目源代码或凭证/密钥/token/密码/私钥/连接串等隐私敏感信息；草稿或附件中出现 cs_ 开头密文/API token/Bearer 串；任一计划附件单文件 > 200 MB）。
判定规则：总分≥75 且 目标对齐≥26 且 无红线命中 → 通过；否则不通过。附件超限属红线，必须在此暴露、报告中点名文件与大小。
只依据“是否服务于当前方向、是否有增量、是否有依据、是否制造冲突”评判，不要因为写得漂亮就放水。
输出结构化质检报告（含各维度分、红线、结论；不通过时给出主要偏离点）。不要替用户做追加决定。
注意：你的报告是纯内部产物，交回主流程按“三档呈现层”翻译给用户，不会原样展示。
```

### 阶段 C：观察+补充摘要，向用户确认

进入确认前，输出一份**确认卡**（全程按 0.5 节人话，不出现分数/质检字样）：
- **我看了一下这块协作**：方向是什么、现在还缺什么（精简两三句）
- **我准备补充的内容**：要点 + 附件清单 + "这条是接着××那条说的"
- 明确询问："确认发到协作板吗？（确认 / 改改 / 算了）"
- **软锁（板已归档）**：若 `get_board` 的 `meta.archived_at` 非空，确认卡里**额外多问一句**：
  「这块协作已标记完成，仍要补充吗？」用户明确同意后再 append。
  追加成功后服务端会自动清空归档标记（响应可带 `board_archived: true`），
  不要求用户再点「重新开启」。

**未得到用户明确"确认"前，不调用 append。**

```
向用户输出确认卡（按 0.5 节翻译层，禁止出现内部字段名与质检字样）：
①我看了一下这块协作：方向 + 还缺什么（精简）②准备补充的要点 + 附件清单 + 自然语言引用。
若 meta.archived_at 非空：多问「这块协作已标记完成，仍要补充吗？」
最后一句话请求确认：确认 / 改改 / 算了。在用户明确确认前，不得调用 collab_append_entry。
```

### 阶段 D：落地追加（Commit）——正文先行，附件后绑

（走到这一步的附件都已通过质检的 200 MB 上限检查。）**流程顺序：先落 entry，再传附件**——
附件在预留时就绑定 `entry_id`，从根源杜绝孤儿附件。

**D0 · 落地前预检**（此时还来得及无损退出）：逐一确认 `attachments_planned` 中每个文件
存在、可读、大小与质检时一致。任一预检不过 → **不 append**，向用户说明哪个文件有问题。

**D1 · 落正文**：`collab_append_entry(board_id, content, author_name, participant_id, entry_type?, references?[, reply_to], participant_secret?)` → 拿到 `entry_id`。
- **`author_name` 与 `participant_id` 必填**；有 secret 则必传 `participant_secret`。由 skill
  按 §0 读取本机身份文件后显式传入（服务端不自动读取本地身份；路径见「路径与权限」）。
- 正文中对附件的表述用不依赖上传时序的写法（如"附件：xxx"），不要写"已上传"。

**D2 · 传附件**（每个附件依次执行，字节不经过 MCP/模型）：
1. `collab_request_attachment_upload(board_id, entry_id, display_name, mime?, size?, participant_id?, participant_secret?)` →
   拿到 `attachment_id` 与 `upload_url`（附件预留即绑定该 entry）。有本机身份则**必传**
   `participant_id`；有 secret 则必传 `participant_secret`（与 D1 同源）。
2. skill 用本机 shell 把文件**直接 PUT 到 `upload_url`**（如 `curl -T <file> <upload_url>`）——字节直传 OSS。
3. `collab_commit_attachment(attachment_id, participant_id?, participant_secret?)` → 定稿，附件已绑定到 entry。
   同样：有身份则传，与 D1 一致。

**D3 · 失败处理（红线：补传不重发）**：正文已追加且不可撤回，附件上传失败时——
- 对**同一 `entry_id`** 重试 D2（重新 request → PUT → commit），最多自动重试 2 次；
- **禁止再 append 一条新 entry** 来"重发"，否则板上出现重复内容；
- 仍失败则人话交代："内容已经发到协作板了，但文件 xxx 没传上；稍后对我说'把附件补传一下'我再试。"
  （entry_id 记在会话里，补传时继续用它走 D2。）

成功后回传 `entry_id`，按 0.5 节话术一句话告知"已发到协作板，小伙伴打开邀请链接或凭邀请码就能看到"。

```
用户已确认。执行顺序（先正文后附件，附件绑定 entry_id）：
D0 预检 attachments_planned 每个文件（存在/可读/大小一致），不过则不 append 并说明。
D1 collab_append_entry 落正文（author_name / participant_id 必填，有 secret 则传 participant_secret；显式取自本机身份）→ 拿到 entry_id。
D2 对每个附件：collab_request_attachment_upload(..., participant_id / participant_secret 与 D1 同源)
   → 本机 curl -T 直传 upload_url → collab_commit_attachment(..., 同样带身份) 定稿。
D3 上传失败：对同一 entry_id 重试（最多 2 次）；禁止重发新 entry；仍失败则告知用户可稍后让我补传。
成功后只回传一句话确认，不赘述。
```

---

## 5. 状态机（补充模式）

```
观察结论 ─► A生成草稿 ─► B质检(后台) ─┬─ 红线命中 ─► 体贴拦下+给出路 ─► 用户改后回到 B
                                     ├─ 不通过无红线 ─► 温和建议(仍可直发) ─► C确认卡
                                     └─ 通过 ─► C确认卡 ─┬─ 算了/改改 ─► 结束 / 回到 A
                                                        └─ 确认 ─► D0预检 ─► D1落正文 ─► D2传附件 ─► 完成
                                                                    │                    └─ 失败 ─► 对同一 entry 补传（不重发）
                                                                    └─ 预检不过 ─► 不 append，说明后结束
```

## 6. 不变量（任何时候都要守住）

- 板**只增不改**：绝不调用任何修改/删除已有 entry 的操作（MCP 也不提供）。
- 公开板访问 permissionless：拿到邀请链接/邀请码即可读；写入须带本机身份
  （`participant_id`，有 secret 则传 `participant_secret`）。
- 展示名只进署名；`participant_id` / `secret` 只进工具参数、永不进用户可见层；
  绑定本机 identity 文件，换 AI 工具不换身份（路径见 §0）。
- 环境不可写时**不反复注册**身份（§0 沙盒降级）。
- AI **绝不**索取用户名 / 密码 / 验证码；登录与绑定一律走浏览器链接（§0 绑定流程）。
- 读取**不自动置已读**；需用户同意后显式调 `collab_mark_read`。
- 观察**不写**、补充**才写**。
- 未经质检不追加；**红线是唯一硬拦截**，其余质检结果只产生建议不拦路；质检结果 ≠ 自动追加，**用户确认是最后闸门**。
- 质检的分数、维度、判定字眼与内部字段名**永不进入用户可见输出**（0.5 节 + 4.B 呈现层禁词）。
- 附件**懒上传、正文先行**：草稿阶段只登记；用户确认后先落 entry，附件预留即绑定 entry_id 再上传。
- 附件上传失败**只对原 entry 补传，绝不重发新 entry**（板 append-only，重发即重复内容）。
