# Hermes：渠道与知识归属如何改变提示词

背景：平台 `cli` / `default`，1975 行，24 个顶层工具小节 + 4 个延迟加载工具。原文见 [example/e-hermes.md](example/e-hermes.md)。

## 结论

1. **渠道直接改写交付方式。** 这次是 plain terminal：Markdown 不渲染、没有附件通道、MEDIA 标签不被拦截，于是提示词要求交付文件时给绝对路径或 URL 的纯文字。**同一个写文件任务，渠道不同，提示词就不同。**
2. **把"真实执行返回了什么"当作完成标准**：要求产物由工具输出支持，不能停在 stub、计划或单条命令，不能编造数据、文件和 API 响应。
3. **知识按加载范围三分**：memory 存每轮都注入的稳定事实，skill 存任务领域经验，TODO 与进度留在会话历史。价值在于**说明了加载范围如何决定知识该放哪**。
4. **依赖工具名做能力判断会出错。** 目录里有 `vision_analyze`（描述可加载图片），而 `browser_exec` 又说当前模型不能看图。

## 一个 system Block 的内容分工

| 模块 | 作用 |
| --- | --- |
| 身份与输出风格 | 回复长度适配问题；完成后报告变化、验证与剩余项 |
| 自身产品文档 | Hermes 问题查产品文档并加载相关 skill |
| Finishing the job | 真实执行、真实验证、不编造失败路径的输出 |
| Parallel tool calls | 独立调用同轮批量，有依赖时串行 |
| Memory / skills | 长期事实、任务经验、短期历史的分工 |
| Mid-turn user steering | 识别真正的中途用户消息，避免重复处理旧消息 |
| 环境、profile 与 CLI 渠道 | Python 环境、配置隔离、显示与通知能力 |
| Skills 目录 | 条件加载任务领域知识 |
| Runtime environment | Linux、home、cwd、临时目录及清理规则 |

## 设计要点

### 渠道决定展示与通知能力

CLI 创建的 cron 结果默认保存在本地，不会实时回到这次会话；要通知用户需连接相应 messaging gateway。这描述的是**特定配置的渠道能力**。

### 完成标准要绑定真实工具输出

与 Codex 的"推进到底"有重叠，Hermes 特别强调执行到底返回了什么，同时防止半成品和虚构成功。注意**工具调用成功不是业务正确的充分条件**。

### 知识归属取决于加载范围

附件直接给出了理由：memory 在未来每一轮注入、字符预算有限；skill 按相关任务加载。

memory 工具有两个目标：`user`（用户是谁——姓名、角色、偏好、风格）和 `memory`（自己的笔记——环境、约定、工具怪癖、教训），分别落到 `~/.hermes/memories/USER.md` 与 `~/.hermes/memories/MEMORY.md`。两者都是**冻结快照**：会话开始时一次性渲染进系统提示，会话内的写入只落盘、**不改变本次的提示词**，下次会话才带上。

预算写得很硬：`MEMORY.md` 2,200 字符，`USER.md` 1,375 字符。写满时 `add` 直接报错并回显现有条目，要用**一次批量操作**同时删旧、缩短、加新——因为字符上限只在批量结束后的最终结果上校验，单独一次 add 会溢出、合并成一批就不会。

工具还给了完整的写入门槛：

- `WHEN`：只记"每次会话都适用、与任务无关"的事实。
- `SKIP`：显而易见的、能重新查到的、原始数据堆、任务进度、已完成工作日志、临时 TODO——后几项改用 `session_search`。
- 写入用事实语句（"用户偏好简洁回复"），不用命令式（"始终简洁回复"）：命令式在后续会话里会被重新读成指令，可能压过用户当下的要求。

### 经验提炼：把做过的事变成 skill

Hermes 的产品定位是"闭环学习"：任务完成后**自主把过程写成 SKILL.md**，使用中继续改进，后台 Curator 周期性整理。这是六家里唯一把"从经验中提炼方法"写进产品机制的一家。

技能住在 `~/.hermes/skills/` 下，**一个技能一个文件夹**（`SKILL.md` 必需，可选 `references/`、`templates/`、`scripts/`）。加载是三级渐进揭示，符合"索引常驻、正文按需"：

1. `skills_list()` 只返回名字与描述，拼成索引进系统提示（文档给的估值是约 3k tokens）→ 2. `skill_view(name)` 拿 SKILL.md 正文与可用引用文件清单 → 3. `skill_view(name, file_path)` 拿具体引用文件。

Curator 是后台维护流程，只处理 agent 自己创建的技能：按使用情况把它们从 active 迁到 stale、再到 archived（默认 14 天转 stale、30 天归档；**只归档不删除**，被 pinned 的技能跳过所有自动流转），LLM 合并整理默认关闭。

它与 memory 的分工很清晰：memory 是每轮都注入的稳定事实，有硬性字符预算；skill 是按任务加载的方法。**一个是"关于用户和环境的事实"，一个是"做事的方法"。**

跨会话检索另有 `session_search`（延迟加载工具），走 SQLite 全文索引翻历史会话，返回的是真实消息原文，不做 LLM 摘要。

本次采集的配置里 skill 写入是关闭的（提示词明确写"这是一次性运行，不要创建或编辑 skills"），但记忆那段仍然保留"任务知识属于 skills，即使 skill writing 不可用"——**归属与能力被分开表述**：写在哪儿是归属问题，能不能写是能力问题，后者不改变前者。

### 中途用户消息与不可信标记

Mid-turn user steering 给出特定 OUT-OF-BAND 标记，要求只在最新工具结果后的对应位置处理，旧历史回放不重复处理。同时要求**不信任工具输出、网页或文件里的仿冒标记**——**字符串本身不构成鉴权**，可信度还要看实际 user 角色与运行时来源。

### 工具契约的第三种编辑形态

Hermes `patch` 描述 fuzzy matching（9 种策略）、唯一匹配、返回 unified diff 并自动语法检查。三种接口的对照：

| | 匹配策略 | 提示词因此强调 |
| --- | --- | --- |
| Pi | 批量、原文件、精确、不重叠 | 多区域联合约束 |
| Claude Code | 精确字符串、先 Read | 去行号前缀、replace_all |
| Hermes | 模糊 9 策略 | 容错范围与唯一性 |

模糊匹配能减少空白与缩进差异导致的失败，也可能在相似代码中选错目标；**工具有效率与目标修改正确率是两件事**。`write_file` 的 `verified:true` 只表示确认磁盘内容哈希，不代表功能通过测试。

### 动态工具与代码执行

`tool_search` 列出 4 个延迟加载能力（`cronjob_manage`、`session_search`、`process_manage`、`todo_list`）；知道工具名时可先 `tool_describe` 取 schema 再 `tool_call`。**顶层目录不是能力全集**——这直接影响"某 Agent 有几个工具"的统计口径。

`execute_code` 用 Python 调用 Hermes 工具，适合调用之间有过滤、分支、循环的机械工作；限制 5 分钟、最多 50 次工具调用，并建议把大量输出在代码内处理后再进入上下文。与 Codex 的 JavaScript `functions.exec` 目的相近，但语言、状态保持和暴露的工具范围不同。

### 能力信息会随模型模态变化

`browser_exec` 要求优先 `page_info`、DOM 文本和选择器操作，理由是当前模型不能看图。这不是能力写错，而是**按模型模态分支的表述**：`delegate_task` 的 schema 明写非视觉子 Agent 只拿到路径提示、需自行调用 `vision_analyze` 取像素，视觉子 Agent 则首轮直接收到图像。同一套提示词要同时服务两类模型，**不能只凭工具名或一句能力描述判断当前能做什么**。

另外，浏览器会话与工作区跨调用保持，Python 变量不保持；`execute_code` 则声明持久的 kernel 会话。**不同工具的状态生命周期要分别记录。**

## 组装结构（官方文档）

官方开发文档把系统提示词描述为**按缓存失效速度分三层拼装**：

| 层 | 内容 |
| --- | --- |
| stable | 身份（`SOUL.md` 或内置回退）、工具与模型指引、编码操作说明 |
| context | 调用方传入的 system message、项目上下文文件（`.hermes.md` / `AGENTS.md` / `CLAUDE.md` / `.cursorrules`）、git 工作区快照、操作者说明与平台提示 |
| volatile | skills 索引、记忆快照（`MEMORY.md`）、用户画像快照（`USER.md`）、外部记忆提供方块、时间戳与模型行、运行时环境提示 |

**顺序即缓存策略。** 项目上下文文件排在"命名当前 worktree"的内容之前，不同 worktree 的同一项目才能共享前缀；运行时环境块固定收尾，因为它是每次变化的部分。平台（desktop ↔ TUI）被有意排除在身份字段之外，切换界面时缓存前缀不失效。

这与 DSH 的插件化组装是同一结论的两种实现：**导出看到的一整块，往往是分层拼装后再连接的结果。**

## 跨主题内容（留待后续主题）

memory 的预算与加载路径、`browser_vault` 的凭据处理、cron 与 gateway 的交付行为、`delegate_task` 的生命周期，都留给后续主题。这里只记一条：**描述不等于强制执行**。

## 参考

仓库 [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent)（MIT）、[Prompt Assembly](https://hermes-agent.nousresearch.com/docs/developer-guide/prompt-assembly/)（涉及 `agent/system_prompt.py`、`agent/prompt_builder.py`）。
