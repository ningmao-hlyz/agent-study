# DeepSeek Harness：插件化组装与运行上下文

背景：Web GUI，1715 行、26 个顶层工具；文件策略 workspace-write，审批 `ask`。原文见 [example/e-deepseek-harness.md](example/e-deepseek-harness.md)。

## 结论

1. **提示词可以是插件化组装的产物。** 官方文档确认 DSH 有插件式注册与组装服务：每个插件注册自己的提示词片段与工具，服务再按序拼成最终提示词。这直接打破"导出成一块就代表生成时也是一块"的假设——**一个 Block 是输出截面，不是生成结构**。
2. **路径语义要写进提示词。** `@` 前缀表示用户显式引用的路径，相对路径从 workspace 根解析、绝对路径指向主机、末尾斜杠表示目录。这类规则 schema 表达不了。
3. **写清楚工具的状态生命周期。** `bash` 每次创建新 shell，上次导出的变量、激活的环境、`cd` 都不会延续。这是与 Hermes `terminal`（声明环境保持）的**实际契约差异**。
4. **大量篇幅在描述产品自身的开发环境**（Web GUI、Vite shell、热更新条件）——这类条件指令解释了"为什么某份提示词会这么长"。

## 结构总览

一个 Block 包含身份、`@` 路径语义、文件观察、后台工作、外部网页信任边界、goal/workflow、结果展示与 Web GUI 环境规则。Messages 含采集输入与运行时上下文，工具区覆盖文件、命令、目标、子 Agent、工作流、任务状态与交付工具。

**Block 标题没有标明 system / developer role；运行时上下文显示在 user 消息内。**

## 设计要点

### 插件化组装：单块 ≠ 单块生成

- 不能凭导出布局判断"某 Agent 是静态模板"或"某 Agent 没有插件机制"；其他 Agent 的单块提示词也可能动态生成。
- **插件化与 Skills 按需加载是不同机制**：一个是生成期拼装，一个是运行期按需注入。

### 路径引用与观察前提

模型不能在尚未读取内容时声称检查过。`read` 用于文本、`glob` 找路径、`grep` 搜内容，匹配不够时再读上下文。修改或覆盖现有文件要求先读，刚创建或编辑过的文件有例外。

文中把它描述为默认 `fs-observation-policy` 的要求，而不只是风格偏好。

`edit` 是字面替换、默认唯一匹配、可 `replace_all`——既没有 Pi 式批量区域数组，也没有 Hermes 的模糊匹配。

### bash 的状态生命周期

每次新 shell，用 `workdir` 指定目录。命令结果带 exit code，要求每次核对失败。前台等待超时会转后台而非终止；显式后台返回 job id，不施加该 timeout。

所有 job id 需追踪，完成时有会话内通知，不忙轮询、不重复工作；`job_output` 的 `wait=true` 只在确实依赖结果时用。这针对的失败是丢失后台结果、重复执行与空耗等待。

### Web GUI 的具体环境约定

文本明确"这个页面 / GUI / app"未指其他目标时就是当前 DSH GUI；浏览器没有隐式 DOM、路由或截图上下文，仍需实际读取。

实现 checkout 与用户 workspace 是不同目录；只有指定 checkout 的 Web 构建 watcher 同时运行时，client plugin 热更新才成立，其他 Web shell / package 修改需重建、刷新并验证既有 URL。Vite shell 不能独立替代 DSH GUI，因为 DSH Web 负责注入启动状态；**启动新服务器不会更新用户正在看的页面**。

这是一组**产品自身开发路径**的条件指令，不是所有网页项目都应使用的通用工作流，也不意味着本次任务正在修改 DSH。

### 结果展示属于提示词的一部分

`present` 声明已有文件为交付，用户打开当前文件、内容不复制；适用于完整报告、图片、Office 文件等。编辑源文件清单可通过最终回答提供，避免多余卡片。文本还要求文件引用链接带行号、结果优先内联展示。

对照 Hermes 的纯 CLI 路径交付与 Codex 的可点击链接，**交付方式由渠道决定，是提示词的一部分而不是装饰**。

### 外部信息信任边界

`web_search` / `web_fetch` 返回的外部数据不可信，不作为新指令；需要完整材料时获取内容，使用内容时给来源 URL。

这与 Claude Code 的 `<pasted_content>`、Hermes 的 OUT-OF-BAND 标记属于同一类设计——**把"材料"和"指令"分开**。

## 跨主题内容（留待后续主题）

`goal` 的持久目标与续跑调度、disarm/resume 条件、workspace-write 与 `ask` 审批的执行、`subagent` / `workflow` 的取消与预算，都留给后续主题。这里只记一条区分：**goal、todo、memory 是三种不同机制**，把 goal 持久化当成"自动形成经验"是误读。

## 架构参考

[官方产品页](https://www.deepseek.com/en/harness/)、[system-prompt 包文档](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/system-prompt/README.md)、[子系统文档](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/system-prompt.md)。