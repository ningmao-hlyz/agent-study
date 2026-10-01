# 横向对比

按 [总纲](overview.md) 的八个模块逐行对齐。日期 2026-10-01。

## 主表

| 模块 | Pi 0.87.1 | Claude Code | Codex | Hermes | Grok | DSH |
| --- | --- | --- | --- | --- | --- | --- |
| 1 身份与目标 | 编程助手 | 软件工程交互 Agent | 共享工作区协作者，完整处理目标 | 通用 Agent；真实执行与验证交付 | 自主软件工程 Agent，保留全部要求 | coding agent；Web GUI |
| 2 工具语义·编辑 | `edits[]` 原文件匹配、唯一、不重叠 | `old_string`/`new_string`；先 Read；`replace_all` | 嵌套 `apply_patch` | `patch` 模糊 9 策略 + 自动语法检查 | `search_replace` 唯一/`replace_all`；剔除行号前缀 | 字面唯一替换，可 `replace_all` |
| 2 工具语义·命令 | `bash` | `Bash`；auto mode 下条件放宽 | `functions.exec` 内 `exec_command` | `terminal` 声明环境保持 | `run_terminal_command` 带 `block_until_ms` | `bash` 每次新 shell，超时转后台 |
| 2 工具语义·形态 | 4 个独立工具 | 31 个顶层工具 | JS 包装入口 + 嵌套工具 | 24 顶层 + 4 延迟加载 | 26 顶层 + 动态 MCP | 26 个顶层工具 |
| 3 行为规则 | `<rules>` 专注操作约定 | 规划先对齐；提问只用于真正属于用户的决策 | 推进到底、授权复用、连续任务 | 不止于 stub/计划；真实工具输出支持产物 | 动作/问答/审查三分 | 观察前提；不忙轮询 |
| 4 环境与交付 | 工作目录 | 终端 Markdown；`file_path:line_number` | 可点击文件链接；commentary/final 分工 | plain CLI 纯文字路径；cron 本地保存 | Markdown；`/tmp` 临时文件 | Web GUI `present` 卡片；内联展示 |
| 5 安全与授权 | 未见专门段落 | 拒绝需调整；难撤销/外向动作加条件 | 复用授权；外发需显式要求或 skill 授权 | 凭据走专用工具，不进上下文 | 可逆性 + 影响对象分档；三层区分最直接 | 沙箱拒绝是 policy denial；受控重试 vs 绕过 |
| 6 记忆与状态条款 | 未见 | 一条事实一张卡片 + `[[name]]` 双链 + `MEMORY.md` 索引 | `create_goal` / `get_goal` / `update_goal`，带 token 预算与轮数上限；无记忆库 | `MEMORY.md` 2,200 字符 + `USER.md` 1,375 字符冻结快照；方法走 SKILL.md | 有 `user_rules` 注入，无 memory 协议 | `goal` 持久目标 + 自动续跑轮次；无长期记忆协议 |
| 7 条件性段落 | 较少 | auto mode / plan mode / Artifact 例外 | Default 模式 + 多 Agent 限制 | one-shot | 工具不可用时的替代验证 | workspace-write / ask；GUI 热更新条件 |
| 8 布局 | system 块 + 工具定义分列 | 四 Block + cached 标记；工具区 89.5% | 六个 developer Block；工具区 59.2% | 一个 system 块 | system + 三条 user 消息 | 一个 Block + user 内运行时上下文；插件化组装 |

"工具区占比"按换行统一后的字符数统计，不是 token 计数，不能跨模型比成本；工具数量也要先统一口径（顶层入口 / 嵌套函数 / 全部能力不可混算）。

## 六家最值得记的三条

1. **接口决定规则文字。** 三种编辑接口对应三套不同强调（Pi 多区域联合约束 / Claude Code 先读与去行号 / Hermes 模糊容错），这不是厂商风格差异，是 schema 表达不了的东西必须用文字补。
2. **环境写法都是"当前值"。** 六家都声明本次的平台、权限模式与交付渠道。Codex 的 read-only + `never`、DSH 的 workspace-write + `ask`、Hermes 的 plain terminal，全是配置快照。
3. **只有 DSH 有官方文档明确"插件化组装"。** 因此不能凭导出是一整块就推断生成时也是一整块。

## 差异解释记录

| 现象 | 为什么 |
| --- | --- |
| Pi 的规则与 `edit` 描述都强调原文件匹配 | 帮助模型构造符合接口语义的参数 |
| Pi 是 `edits[]`，Claude Code 是单组字符串 + `replace_all` | 接口不同，参数构造指引只能不同 |
| Claude Code 声明索引式记忆、规划、委派、成果与调度 | 更多长任务状态与交付机制带来额外规则 |
| Codex 强调授权复用，Claude Code 有详尽规划对齐 | 交互控制方式不同，影响确认次数与完成率 |
| Codex 有多 Agent 接口，但当前策略限制主动创建 | 能力存在不等于本次策略允许 |
| Hermes 把任务经验分流到 skills，memory 只存稳定事实 | 加载范围决定知识归属与预算策略 |
| Hermes CLI 不渲染 Markdown，默认 cron 无实时交付 | 渠道决定展示与通知指引 |
| Hermes `patch` 模糊匹配，Pi / Claude Code 强调精确匹配 | 编辑策略在容错与误匹配之间取舍 |
| Grok 顶部 system 与 `user_rules` 内容重复 | 两层来源不同：厂商默认 vs 用户定制 |
| DSH 官方文档确认插件式组装，附件却是一整块 | 导出截面不代表生成结构 |

## 三种跨会话机制

这四种机制解决的不是同一个问题：

| 机制 | 载体 | 面向什么 | 何时写入 | 怎么读回来 |
| --- | --- | --- | --- | --- |
| Claude Code 记忆卡片 | 一条事实一个文件 + `[[name]]` 双链 + `MEMORY.md` 索引 | **关于用户和环境的事实** | 模型判断"值得记"时显式写 | 索引每会话注入，`description` 用于判相关性 |
| Codex 目标持久 | 会话内的目标、约束与授权状态 | **没做完的事** | 由任务自然产生，不落盘成文件 | 压缩或新会话后继续携带 |
| DSH `goal` | goal 列表与续跑调度 | **没做完的事** | 显式创建 | 目标列表 + 重新拉起执行 |
| Hermes 经验提炼 | SKILL.md（自动创建、后台整理） | **做事的方法** | 复杂任务完成后自主提炼 | skills 索引按任务加载 |

三点结论：

1. **"记忆"是三个不同问题的合称**——记住事实、不丢目标、复用方法。三家的载体和数据形状完全不同，把它们笼统对比"谁的记忆好"没有意义。
2. **写入时机是主要分野。** Claude Code 靠模型当下判断一条事实值不值得记；Hermes 在任务完成后回头提炼；Codex 和 DSH 则不是"记录"，而是**让没完成的目标不在压缩中蒸发**。
3. **成本控制方式不同。** Claude Code 用"索引常驻 + 正文按需"；Hermes 用硬字符预算卡住每轮注入的 memory，把方法挪到按需加载的 skills；Codex 与 DSH 不额外注入，代价是目标必须靠自己持续维护。

## 记忆是怎么被召回的

| Agent | 载体 | 召回方式 | 粒度 |
| --- | --- | --- | --- |
| Claude Code | 一条事实一个文件 | `MEMORY.md` 索引每会话注入；正文靠 `[[name]]` 双链打开；frontmatter 的 `description` 一行用于判相关性；召回内容以 `<system-reminder>` 的背景身份出现 | 索引全量 + 正文按需 |
| Hermes | `~/.hermes/memories/` 下的 `MEMORY.md` / `USER.md` | 冻结快照每轮整体注入（2,200 / 1,375 字符）；方法走 SKILL.md 三级加载：`skills_list()` 索引 → `skill_view(name)` 正文 → `skill_view(name, file_path)` 引用；历史会话走 `session_search`（SQLite 全文索引，返回消息原文，不做摘要） | 事实全量 + 技能按需 + 会话检索 |
| Grok | `user_rules` | 随 user 消息注入，无独立召回机制 | 全量 |
| Codex | `create_goal` / `get_goal` / `update_goal` | 按 goal 读取目标、状态与剩余预算；**不是"记事实"而是"续任务"** | 单个目标对象 |
| DSH | `goal` | 读取当前会话 goal 的 id 与 revision，支持 pause / resume / 标记 blocked | 单个目标对象 |
| Pi | 无 | — | — |

三点：

1. **内置记忆没有一家用向量检索。** 用的是"全量注入 + 全文检索 + 按需展开"三种朴素手段的排列组合；语义检索要靠外部提供方（Hermes 可接 Honcho）。
2. **决定效果的不是检索算法，而是注入时机。** Hermes 的冻结快照是明确取舍：同一会话内前缀稳定、可缓存，代价是刚写下的记忆本次会话看不到、下次才生效——源码里的注释就是 "mid-session writes hit disk but never change the prompt"。
3. **"索引常驻 + 正文按需"是唯一被重复采用的结构。** Claude Code 用 `MEMORY.md` 索引，Hermes 用 `skills_list()` 索引，都由一行摘要决定要不要展开正文——**先判断相关性，再付出上下文成本**。

## 跨主题维度

记忆的实现细节（召回算法、存储格式、预算策略）、权限与沙箱的执行、上下文压缩、委派与调度，各自留给后续主题。上面两处只讲提示词层面的**归属与载体**。