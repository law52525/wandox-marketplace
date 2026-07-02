---
name: collab
description: >-
  Wandox Collab —— 面向 AI 的轻协作。在一块 append-only 的"协作板（Board）"上，
  用户通过自己的 AI 工具读取、观察、并追加贡献，多方共同完成一件事。
  触发场景：创建协作板 / 发起协作 / 开个 collab；观察某块板、看看我能补充什么；
  往板里补充、提交我这一方、contribute、append。涉及 board / board_id / 协作板 /
  wandox collab 时使用本 skill。当用户消息中出现形如 bd_xxx 的 board_id（即便没有其它动词）
  也触发本 skill。配套 wandox collab MCP（collab_* 工具）。
---

# Wandox Collab Skill

面向 AI 的简单协作系统。核心隐喻：把"在线云文档"换成一块**面向 AI 的协作板（Board）**，
每个参与方用自己的 AI 工具在板上**只追加、不修改**地贡献内容，最终共同完成一件事。

依赖配套的 wandox collab MCP（工具前缀 `collab_`）：`collab_create_board`、`collab_get_board`、
`collab_append_entry`、`collab_request_attachment_upload`、`collab_commit_attachment`、`collab_list_boards`。

---

## 0. 前置：身份与访问（每次启动都先做）

**访问层 permissionless**：`board_id` 本身就是凭证，拿到即可读、可追加。没有账号、登录、
角色、权限矩阵，没有创建者/协作者之分。

**署名层用轻量本地身份**：每条贡献带一个 `author` 署名，纯展示、不做鉴权。

任何模式启动前，先执行 **ensure_identity**：

1. 读取 `~/.wandox/identity.json`。
2. 若不存在：引导用户填一个 display name（如 `Caren / 后端`），生成 `participant_id`（uuid），
   写入 `~/.wandox/identity.json`，格式：
   ```json
   { "participant_id": "p_xxxx", "display_name": "Caren / 后端", "created_at": "<ISO8601>" }
   ```
3. 若已存在：静默读取，作为后续 `collab_append_entry` 的默认 `author`。

> 身份是**指针**（标记"谁说的、可回溯到同一来源"），不是**画像**（这人是什么角色/专长）。
> 本系统不引入职业角色——"某参与方此刻能贡献什么"由其本地 AI 工具临场推断，且随 board 演进变化。
> 换 AI 工具不换身份（绑定本机 `~/.wandox/identity.json`）。

---

## 1. 路由：按用户语义分流到三个模式

一个 skill、三个子流程，由触发词路由：

- **创建模式（Create）**："创建一个协作板 / 开个 collab / 发起协作"
- **观察模式（Observe）**："看看这个板 / 我能补充什么 / observe"
- **补充模式（Supplement）**："往板里补充 / 提交我的部分 / contribute / append"

**裸输入触发**：用户消息里只要出现形如 `bd_7Kf3aP9xQ2` 的 board_id（无其它动词），即触发本 skill 并
**默认进入观察模式**（只读、无副作用），读完板再向用户建议下一步。带动词时按动词路由。分享与触发都用 `board_id`。

**忘了 board_id**：用户想找一块之前参与过的板却记不起 id 时，用 `collab_list_boards`（可带 keyword 或本机
participant_id 过滤）列出候选供其选择，再进入对应模式。

---

## 2. 模式一：创建模式（Create）

**目标**：只创建一块板供观察和补充，不写实质内容。

**流程**

1. 确认 `goal`（**必填**），可选 `title` / `description` / `initial_context`。goal 必须明确——它是后续所有
   观察和质检的"北极星"，且创建后不可修改。**没有 goal 不创建板**：先追问用户这块板要协作完成什么。
2. 调 `collab_create_board`（goal 必填），拿到 `board_id`（格式如 `bd_7Kf3aP9xQ2`）。
3. 回给用户 `board_id`，并一句话引导："把这个 board_id 发给协作方，拿到的人即可读、可追加（permissionless）。"

**提示词（system 片段）**

```
你在执行 wandox collab「创建模式」。你的唯一任务是创建一块面向 AI 的协作板。
- 必须先确认 goal（这块板要协作完成什么）。goal 要具体、可判定，因为它将作为后续“偏离判据”，
  且创建后不可修改。用户没给 goal 就追问，不要在没有 goal 的情况下创建板。
- 若中途需要调整方向，不是改 goal，而是（在补充模式）追加一条 decision entry。
- 不要在创建阶段写入任何实质贡献内容；实质内容属于补充模式。
- 创建成功后，只回传 board_id 与一句话引导，不要冗余解释。
```

---

## 3. 模式二：观察模式（Observe）

**目标**：读板上已有内容，**结合本地项目 + AI 工具对用户自身的了解**，判断"我（这一方）能补充什么"。
观察模式**不写入任何东西**，只产出一份**观察结论（Observation）**（内部交给补充模式用，也展示给用户）。

**流程**

1. `collab_get_board(board_id)` 拉取 meta + 全部 entries。
2. 结构化理解板：goal 是什么？已有哪些贡献、覆盖了哪些方面、留下哪些空白/冲突/未决问题？
3. 扫描本地上下文：当前项目代码/文件、用户历史、用户在 AI 工具里的记忆、可用的 skill/工具/数据，
   判断"我独有、且与 goal 相关"的可贡献点。**也允许得出"与我无关、帮不上"的结论——这是合法结果。**
4. 产出 **Observation**，结构：
   - `goal_restated`（目标复述）：用一句话复述目标，确认理解。
   - `current_coverage`（已覆盖）：板上已有内容覆盖了什么。
   - `gaps`（尚缺）：尚缺什么（最重要）。
   - `my_uniqueable_contributions`（本机可补的点）：基于本地上下文，我能补的、且别人补不了的点（含可附文件）。
   - `conflicts_or_risks`（冲突/风险）：与已有内容的冲突/重复风险。
5. 把 Observation 展示给用户；若用户说"那就补上"，进入补充模式（携带本 Observation）。

**提示词（system 片段）**

```
你在执行 wandox collab「观察模式」。
步骤：
1. 调 collab_get_board 读取整块板，先抓住 goal。
2. 把已有 entries 归纳为：已覆盖 / 空白 / 冲突 / 未决问题。
3. 结合你能访问的本地项目、用户上下文、用户在 AI 工具里的记忆、可用工具，找出“我这一方独有且与 goal
   相关”的可贡献点。严格区分“别人已经说过的”与“我能新增的”，避免重复贡献。若发现与你/用户无关、帮不上，
   如实说明即可，不要硬凑贡献。
4. 输出结构化 Observation：goal_restated（目标复述）/ current_coverage（已覆盖）/ gaps（尚缺）/
   my_uniqueable_contributions（本机可补的点）/ conflicts_or_risks（冲突/风险）。
约束：观察模式绝不写入板（不调用 append）。只输出观察结论。每条可贡献点都要标注：依据来自哪里、是否需要附件。
```

---

## 4. 模式三：补充模式（Supplement / Append）

**目标**：把观察到的、对 goal 有增量价值的内容，经**质检关卡**后，追加到板上。

四个阶段：**生成 → 质检 → 用户确认 → 落地追加**。任一不变量贯穿全程：
**板只增不改；未经质检不追加；未经用户确认不追加。**

### 阶段 A：生成补充草稿（Draft）

输入：观察模式的 Observation（若用户直接进补充模式，先内部跑一次观察）。
基于 `gaps`（尚缺）和 `my_uniqueable_contributions`（本机可补的点）生成**补充草稿**，包含：
- `entry_type`（贡献类型：`idea` 想法 / `evidence` 依据 / `result` 结果 / `question` 提问 /
  `comment` 评论 / `decision` 决定，默认 `comment`）与正文（markdown）
- `references`（引用）：引用了哪些已有 entry / 外部来源
- `attachments_planned`（计划附件）：计划上传的本地文件，**每个都记录文件名与字节大小**（**此阶段先不上传**，大小供质检核对 200 MB 上限）

```
基于 Observation 生成一份“补充草稿”。要求：
- 只补 goal 相关的增量内容，不复述板上已有内容。
- 每个论点尽量给出依据/引用（已有 entry_id 或来源）。
- 若需要文件佐证，列入 attachments_planned，但现在不要上传。
- 标注本草稿意图：它填补了 Observation 里的哪个 gap。
```

### 阶段 B：AI 质检（Quality Check）—— 核心关卡

质检**独立于生成**（建议换一个视角/子代理执行，避免自己给自己放水），核心是判断**有没有目标偏离**并打分。

**质检维度与评分（0–100）**

| 维度 | 权重 | 判据 |
|---|---|---|
| 目标对齐 Goal Alignment | 35 | 草稿是否直接服务 board.goal，无跑题 |
| 增量价值 Novelty | 20 | 是否在补板上"还没有"的内容，而非重复 |
| 依据可靠 Evidence | 20 | 论断是否有引用/可核验来源，附件是否相关 |
| 不冲突/不污染 Consistency | 15 | 是否与已有结论矛盾或制造混乱；若有意纠错须显式说明 |
| 表达清晰 Clarity | 10 | 结构清楚、可被其他 AI 工具消费 |

**通过线**：总分 ≥ 75 **且** 目标对齐 ≥ 26（不允许其它项凑分但跑题）。任一红线命中直接判不通过：
- 红线：与 goal 无关的推销/灌水；编造无来源的"事实"；试图修改/否定他人 entry 而无依据；含敏感/越权内容；
  **正文或附件里出现详细的项目源代码，或凭证/密钥/token/密码/私钥/连接串等隐私敏感信息**——协作板是对外共享的，
  这类内容一旦追加即泄露且不可撤回（append-only），须在质检就拦下并在报告中点明命中位置（可给脱敏/引用替代建议）；
  **任一计划附件单文件超过 200 MB（工程上限）**——须在质检阶段就暴露，并在报告里点名是哪个文件、多大。

**质检报告（展示给用户）结构**
```
质检报告
- 总分：xx / 100（通过线 75）
- 各维度得分与简评
- 命中红线：无 / [列出]
- 结论：通过 / 不通过
- 若不通过：主要偏离点说明（为什么偏离 goal）
```

**分支**
- **不通过**：流程**到此结束**。用户最后看到的就是这份"不通过"报告 + 简要原因。
  不追加任何内容，不自动重试（提示用户"如需调整方向请重新发起"）。
- **通过**：进入阶段 C。

**质检提示词（以独立视角执行）**
```
你是 wandox collab 的质检员，立场独立、严格。给定 board.goal、已有 entries 摘要、待补充草稿（含 attachments_planned 的文件名与大小）。
逐项打分（目标对齐35/增量价值20/依据20/一致性15/清晰10），并检查红线（含：正文/附件出现详细项目源代码或凭证/密钥/token/密码/私钥/连接串等隐私敏感信息；任一计划附件单文件 > 200 MB）。
判定规则：总分≥75 且 目标对齐≥26 且 无红线命中 → 通过；否则不通过。附件超限属红线，必须在此暴露、报告中点名文件与大小。
只依据“是否服务于 goal、是否有增量、是否有依据、是否制造冲突”评判，不要因为写得漂亮就放水。
输出结构化质检报告（含各维度分、红线、结论；不通过时给出主要偏离点）。不要替用户做追加决定。
```

### 阶段 C：观察+补充摘要，向用户确认

质检通过后，输出一份**确认卡**：
- **观察到了什么**（Observation 精简版：goal、gaps）
- **将要补充什么**（草稿正文要点 + 计划附件清单 + 引用）
- **质检结论**（通过，xx 分）
- 明确询问："确认追加到协作板？（确认 / 修改 / 取消）"

**未得到用户明确"确认"前，不调用 append。**

```
向用户输出确认卡：①观察到什么（goal+gaps 精简）②将追加的内容要点+附件清单+引用 ③质检通过分数。
最后用一句话请求确认：确认 / 修改 / 取消。在用户明确确认前，不得调用 collab_append_entry。
```

### 阶段 D：落地追加（Commit）

（走到这一步的附件都已通过质检的 200 MB 上限检查。）用户确认后，先处理附件——**统一一条路径,不分大小,字节不经过 MCP/模型**：
1. `collab_request_attachment_upload(board_id, display_name, mime?, size?)` → 拿到 `attachment_id` 与 `upload_url`（带上 `size`，服务端可再兜底校验）。
2. skill 用本机 shell 把文件**直接 PUT 到 `upload_url`**（如 `curl -T <file> <upload_url>`）——字节直传 OSS。
3. `collab_commit_attachment(attachment_id)` → 确认入库,拿到最终 `url`。

每个附件重复 1–3,收集各 `attachment_id`。然后：

`collab_append_entry(board_id, content, entry_type, references, attachment_ids[, reply_to])`。
   **`author` 由 skill 读取本机 `~/.wandox/identity.json` 后显式传入**（服务端不自动读取本地身份）；
   若是针对某条已有 entry 的回应，带上 `reply_to`。
3. 回传 `entry_id`，一句话告知"已追加，他人凭 board_id 在观察模式即可看到"。

```
用户已确认。先处理 attachments_planned 中的每个附件（统一流程,不分大小）：
collab_request_attachment_upload → 本机 curl -T 直传 upload_url → collab_commit_attachment，拿到各 attachment_id。
再调用 collab_append_entry 一次性追加（正文含引用与附件，author 显式取自本机 identity.json）。
成功后只回传 entry_id 与一句话确认，不赘述。
```

---

## 5. 状态机（补充模式）

```
观察结论 ─► A生成草稿 ─► B质检 ─┬─ 不通过 ─► 展示“不通过质检报告” ─► 结束
                               └─ 通过 ─► C确认卡 ─┬─ 取消/修改 ─► 结束 / 回到 A
                                                  └─ 确认 ─► D上传附件 + append ─► 完成
```

## 6. 不变量（任何时候都要守住）

- 板**只增不改**：绝不调用任何修改/删除已有 entry 的操作（MCP 也不提供）。
- 访问 permissionless：board_id 即凭证，不做角色/权限判断。
- 身份只进署名、不进鉴权；绑定本机，换 AI 工具不换身份。
- 观察**不写**、补充**才写**。
- 未经质检不追加；质检通过 ≠ 自动追加，**用户确认是最后闸门**。
- 附件**懒上传**：草稿阶段只登记，确认后才上传到 OSS。
