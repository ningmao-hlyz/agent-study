# Agent 学习笔记

学习整个 Agent，当前只学 **System Prompt**。记忆、上下文构建、沙箱、权限等主题以后各自单独学。

## System Prompt

- [总纲：组成模块与学习框架](system-prompt/overview.md) —— 先读这篇
- 分析稿：[Pi](system-prompt/pi.md)、[Claude Code](system-prompt/claude-code.md)、[Codex](system-prompt/codex.md)、[Hermes](system-prompt/hermes.md)、[Grok](system-prompt/grok.md)、[DeepSeek Harness](system-prompt/deepseek-harness.md)
- [横向对比](system-prompt/comparison.md)
- 原始提示词：[system-prompt/example/](system-prompt/example/)（`e-` 前缀）
  - `e-pi.md` → [pi.md](system-prompt/pi.md)｜`e-claude-code.md` → [claude-code.md](system-prompt/claude-code.md)｜`e-codex.md` → [codex.md](system-prompt/codex.md)
  - `e-hermes.md` → [hermes.md](system-prompt/hermes.md)｜`e-grok.md` → [grok.md](system-prompt/grok.md)｜`e-deepseek-harness.md` → [deepseek-harness.md](system-prompt/deepseek-harness.md)

## 笔记约定

每篇分析稿统一按四段写：**结论 → 设计要点 → 可借鉴与代价 → 跨主题内容（留待后续主题）**，末尾附背景与来源。


本轮只学"提示词怎么写"，记忆、权限、沙箱等机制本身留给后续主题，在分析稿里只记录它们的表述方式。

版本与更新见 [CHANGELOG](CHANGELOG.md)。当前笔记 v0.3.5。
