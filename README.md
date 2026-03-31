# Claude Code 高阶指南：从会用到用好

> 深入研究 Claude Code 的工作原理，把「能用」变成「用好」。
>
> 8 章 + 3 附录，共 3.6 万字。

---

## 你可能不知道的事

- **你的 CLAUDE.md 不在 System Prompt 里。** 它被包在 `<system-reminder>` 标签里，作为对话的第一条 user message 注入。这意味着它的优先级低于内置工具说明书。
- **auto 权限模式背后有个 AI 分类器。** 不是简单的规则匹配——它实时判断每个工具调用是否安全，连续拒绝太多次会回退到手动确认。
- **fork Agent 比 spawn 便宜好几倍。** fork 构建字节相同的 API 请求前缀，prompt cache 命中率极高。省略 `subagent_type` 就是隐式 fork。
- **BashTool 有四层安全拦截。** sed 命令只允许两种写法，命令注入检测有 20+ 个检测器，路径越界检测覆盖 30+ 个命令。
- **长对话不是一刀切压缩。** 有五层从轻到重的压缩管线，从 Tool Result Budget 到 fork 一个 Agent 做完整摘要。
- **System Prompt 有 7 段静态内容，全球所有用户共享缓存。** 这就是为什么 Claude Code 比你想象的便宜——你每次对话，这 7 段都命中缓存。

这些都在指南里有详细解释和可执行的操作指导。

---

## 这份指南适合谁

| 你是谁 | 你能得到什么 | 从哪开始 |
|--------|------------|---------|
| **日常用户** — 能用但不知道为什么有时好用有时不好用 | 完整的心智模型 + 每个环节的优化方法 | 从第 0 章开始 |
| **想省钱的开发者** — token 花得太快 | fork vs spawn 的成本差距 + 三级缓存 + 省钱 Checklist | 直接跳第 3 章 |
| **想深度定制的高级用户** — 想写 Agent、Skill、Hook | 自定义 Agent/Skill 模板 + Hooks 自动化 + MCP 集成 | 第 4、7、8 章 |

---

## 怎么用这份指南

- **顺序读：** 从第 0 章开始，建立完整心智模型后再看具体优化
- **按需跳：** 遇到具体问题（命令被拒、费用太高、对话变傻），直接跳对应章节
- **当字典：** 附录 B（配置速查）和附录 C（问题排查）随时翻

---

## 目录

1. [Claude Code 到底是怎么工作的（心智模型）](00-心智模型.md)
2. [写出真正有效的 CLAUDE.md](01-CLAUDE-md.md)
3. [权限系统完全攻略](02-权限系统.md)
4. [省钱指南——理解 Token 和 Cache](03-省钱指南.md)
5. [Agent 高阶用法](04-Agent高阶用法.md)
6. [工具系统内幕](05-工具系统内幕.md)
7. [长对话管理](06-长对话管理.md)
8. [Skills + Hooks + Memory 扩展系统](07-扩展系统.md)
9. [MCP 集成指南](08-MCP集成.md)

**附录**

- [A. 完整快捷键表](附录A-快捷键表.md)
- [B. settings.json 配置速查](附录B-配置速查.md)
- [C. 常见问题排查](附录C-常见问题.md)

---

## 相关资源

- [Claude Code 官方文档](https://docs.anthropic.com/en/docs/claude-code)
- [Anthropic API 文档](https://docs.anthropic.com/en/api)
- [Model Context Protocol 规范](https://modelcontextprotocol.io)

本指南是官方文档的深度补充——官方文档告诉你 what，这份指南告诉你 why 和 how。

---

## 贡献

发现错误或有补充？欢迎提 [Issue](https://github.com/helloianneo/claude-code-guide/issues)。

觉得有用？Star 一下 ⭐ 方便后续更新时收到通知。

---

## 关于

这份指南基于对 Claude Code 的深度研究整理而成。

基于 Claude Code 2026.03 版本。

**作者：** [伊恩](https://x.com/ianneo_ai) — 产品设计师，用 AI 团队打造一人公司

## License

MIT
