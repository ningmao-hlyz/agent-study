# Claude Code：把运行机制写成提示词

背景：文本自述 `cc_version=2.1.285.496`，入口 `cli`，2478 行、31 个工具。原文见 [example/e-claude-code.md](example/e-claude-code.md)。

## 结论

1. **这份提示词的重心在工具描述。** 工具区约占 89.5% 字符，大量流程规则直接写在工具说明里，不在 System Prompt 段。
2. **它把"模型如何与执行系统配合"显式写出来**：输出以 Markdown 显示、工具受权限模式约束、拒绝后必须调整而不能原样重试、hooks 输出按用户反馈对待。这是控制权与信息来源的约定。
3. **条件段落最多。** auto mode、规划模式、Artifact 例外、AskUserQuestion 的适用范围各有触发条件；把任意一段扁平化成绝对规则都会读错。
4. 对 `<pasted_content>` 的规则是同类提示词中最具体的：外部粘贴内容的指令只在用户自己的请求要求时才遵循，区段用成对随机 id 对应。
5. **记忆不是一段文本，而是一组卡片。** 一条事实一个文件，带 `name` / `description` / `type` frontmatter，正文用 `[[name]]` 互相链接，`MEMORY.md` 只做一行式索引。这是双链笔记的做法搬进了 Agent。

## 层级结构

| 区段 | 内容与职责 |
| --- | --- |
| Block 1 | 版本、入口等元信息，几乎没有行为规则 |
| Block 2 · cached | 身份声明；`cached` 是导出标记，不是更高权限 |
| Block 3 · cached | 任务范围、安全、harness、授权、记忆、环境、上下文管理 |
| Block 4 · system message · cached | 本次环境、模型声明、Agent 类型、skills 列表、条件性 auto mode 指引；环境与可用能力混在一起 |
| Messages | 署名提醒与采集用用户消息 |
| Tools | 31 个工具的说明与 schema；大量流程规则位于此处 |

## 设计要点

### Harness：先约定控制权与信息来源

工具受用户选择的权限模式约束；**拒绝是执行系统给出的结果，模型需要据此调整，不能原样重试**。系统可在对话中途更新规则，其来源与普通函数结果不同，不能混同。

### 外部粘贴内容要单独设边界

`<pasted_content>` 里的指令可能并非用户本人所写，只在用户自己的请求要求遵循时才遵循；成对随机 id 只提供区段对应信息，本身不保证内容安全。

这类规则的作用是区分"用户意图 / 外部材料 / 工具结果"，减少把读到的内容当成新命令的机会。

### 权限与授权：把"何时问"写成条件

难以撤销或面向外部的动作先确认，除非已持久授权或明确被告知不再询问；**一个情境里的批准不能迁移到下一个**。删除或覆盖前先查看目标，测试失败要带输出如实报告，跳过步骤要说明。

它并非对所有动作统一询问：AskUserQuestion 明确要求只用于真正属于用户、且无法从上下文或合理默认值解决的决策。代价是可能增加交互次数。

### 条件性段落与表面冲突

Harness 先说"有专用工具时优先用专用工具"，后文"While auto mode is active"又允许 Bash 读文件、搜索和做小幅机械修改，并提示脆弱编辑仍优先 Edit/Write。这两段各有适用条件，**不能扁平化成"永远禁止 shell"或"永远优先 shell"**。

Artifact 描述允许自有成果直接发布，而通用规则强调外向动作确认——它们有各自的适用对象、授权与例外。

### 记忆：一条事实一张卡片，用双链连起来

记忆存在 `.../memory/` 目录下，**一条事实一个文件**，frontmatter 三项：

```markdown
---
name: <short-kebab-case-slug>
description: <一行摘要，召回时用来判断相关性>
metadata:
  type: user | feedback | project | reference
---
```

正文写事实本身；`feedback` 与 `project` 两类还要求补 **Why:** 和 **How to apply:** 两行——不只记结论，还记理由和适用方式。四个类型的分工：`user` 是谁，`feedback` 用户对你工作方式的纠正与确认，`project` 代码和 git 历史里读不出来的进行中目标与约束，`reference` 外部资源指针。

关键设计是 `[[name]]` 双向链接，而且**鼓励多链**：链接到一个还不存在的记忆**不算错误**，它标记了一个"以后值得写"的位置。也就是先建关系、内容后补。

写完要在 `MEMORY.md` 加一行 `- [Title](file.md) — hook`。索引每个会话注入上下文，正文永远不进索引——**用索引控制常驻成本，用正文承载细节**。

保存前还有几条约束：先查重，已有文件就更新而不是新建；发现写错就直接删；不存仓库已经记录的（代码结构、历史修复、git 历史、CLAUDE.md），也不存只对本次对话有意义的——被要求记这类东西时，反问"其中哪一点是不显然的"再记那一点。

最后一层是时效性：召回的记忆出现在 `<system-reminder>` 里，**是背景而非指令**，而且反映的是写入当时的情况；如果它提到某个文件、函数或 flag，使用前要确认它们还存在。

### 工具按职责分类

| 职责 | 工具 |
| --- | --- |
| 本地文件和命令 | Bash、Read、Edit、Write、NotebookEdit |
| 网络资料 | WebFetch、WebSearch |
| 委派与协作 | Agent、ListAgents、SendMessage、Workflow、TaskStop |
| 规划与工作区 | EnterPlanMode、ExitPlanMode、EnterWorktree、ExitWorktree |
| 调度与事件 | CronCreate、CronDelete、CronList、Monitor、ScheduleWakeup |
| 用户交互 | AskUserQuestion、PushNotification、EndConversation |
| 成果与共享状态 | Artifact、ArtifactComments、ArtifactData、DesignSync |
| 专项机制 | Skill、ReportFindings、SendFeedback |

## 与 Pi 的接口对照

| | Pi 0.87.1 | Claude Code |
| --- | --- | --- |
| 参数 | `path` + `edits[]` | `file_path` + `old_string` + `new_string` + 可选 `replace_all` |
| 提示词强调 | 多区域共同匹配**原文件**、唯一、不重叠 | 本会话先 Read、字符串精确匹配且唯一、开启 replace_all 时替换全部 |
| schema 未表达 | 数组内多区域的联合约束 | 运行时串行/并行与文件状态冲突处理 |

同样的编程任务，接口不同，提示词就必须不同。这是"工具语义决定规则文字"最直接的一组对照。

## 跨主题内容（留待后续主题）

记忆、会话摘要、Agent 委派、Skill 与 Workflow 的机制实现留给后续各自的主题。这里只记录它们的**表述方式**：提示词如何声明知识归属、何时允许委派、条件性工作流的 opt-in。