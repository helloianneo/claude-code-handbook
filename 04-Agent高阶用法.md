# 第 4 章：Agent 高阶用法

Claude Code 不是一个模型在干活，是一群模型在干活。你发一条指令，背后可能同时跑着 5 个 Agent，各自搜文件、读代码、做规划。调度得当，5 个 Agent 同时干活比你手动串行快 5 倍。调度不当，5 个 Agent 互相踩文件、重复搜索，token 烧完了事情还没干成。

这一章讲清楚怎么选 Agent、怎么写 prompt、怎么管生命周期，让它们干对事、不打架。

---

## 4.1 四种 Agent 对比

Claude Code 内置 4 种 Agent，加上 2 个特殊用途辅助 Agent。每种的模型、工具权限、适用场景完全不同，选错了要么浪费钱，要么任务直接失败。

**fork Agent**

触发方式：调用 Agent 工具时省略 `subagent_type` 参数，默认就是 fork。

模型：继承父进程的模型。父进程用 Opus，fork 出来也是 Opus。

核心优势：继承父进程的完整上下文，构建字节相同的 API 请求前缀，prompt cache 命中率极高。第 3 章讲过，这意味着 fork 比从头启动的 Agent 便宜得多——同样一个任务，fork 可能只花 general-purpose 十分之一的钱。

适用场景：简单独立任务、并行查询、需要复用当前对话上下文的工作。比如你正在调试一个 bug，想同时查 3 个可能的原因——fork 3 个 Agent 出去，每个都能看到你之前的调试过程，不需要重复解释背景。

不适用场景：需要干净上下文的任务。fork 继承了父进程所有的对话历史，如果你之前的对话里有误导性的信息，fork 出去的 Agent 也会被误导。

工具限制：全工具可用，但 fork 不能再 fork。内部通过检测 `<fork_boilerplate>` 标签来防止嵌套，你不需要管这个机制，只要知道 fork 只能一层。

**general-purpose Agent**

触发方式：设置 `subagent_type="general-purpose"`。

模型：继承父进程模型。

核心特点：从干净上下文开始，全工具可用。它是功能最完整的 Agent 类型，能调用所有工具，包括 Read、Write、Edit、Bash、Grep、Glob、Agent（可以再创建子 Agent）。

适用场景：复杂多步骤任务，比如"重构这个模块的错误处理逻辑"——需要先搜索、再分析、再修改多个文件。

不适用场景：简单查询。杀鸡用牛刀，token 消耗远高于 fork 或 Explore。一个简单的"找到所有用了 deprecated API 的文件"用 general-purpose，可能花 3000 token；用 Explore 只要 300。

**Explore Agent**

触发方式：设置 `subagent_type="Explore"`。

模型：Haiku。这是关键——Haiku 是最便宜最快的模型，一个 Explore 调用的成本大概是 Opus 的二十分之一。

适用场景：快速代码搜索、文件查找、回答代码结构问题。比如"这个项目的入口文件在哪""找到所有导入了 lodash 的文件"。

工具限制：Edit、Write、Agent 全被禁了。Explore Agent 只能看，不能改。这个限制是硬性的，你没法绕过。如果你让 Explore Agent "找到文件并修复 bug"，它能找到文件，但修复那一步会失败。

踩坑经历：有一次我让 Explore Agent "分析这个函数的问题并给出修复方案"，它分析得很好，但因为不能写文件，最后只给了一段文字建议。我还得再开一个 fork Agent 去执行修复。早知道直接用 fork，一步到位。教训是：如果任务链条里有任何写操作，别用 Explore。

**Plan Agent**

触发方式：设置 `subagent_type="Plan"`。

模型：继承父进程模型。

适用场景：架构设计、方案规划、实现计划输出。比如"设计一个缓存系统的方案"或者"把这个单体应用拆分成微服务需要几步"。

工具限制：Edit 和 Write 被禁了，但可以用 Read、Grep、Glob、Bash（只读命令）。它能读代码、搜索代码来辅助规划，但不能动手改。

不适用场景：任何需要执行的任务。Plan Agent 规划完了，你还得派别的 Agent 去执行。

**两个辅助 Agent**

`claude-code-guide`：使用指南查询 Agent。当你问"怎么配置 MCP"这类关于 Claude Code 本身的问题时，主 Agent 可能会调用它来查文档。日常你不需要直接用。

`statusline-setup`：状态栏配置 Agent。帮你设置终端状态栏显示 token 用量、模型信息等。同样是内部辅助用途。

**一张表总结：**

| Agent | 触发方式 | 模型 | 能写文件 | 能再创建子 Agent | 成本 |
|-------|---------|------|---------|----------------|------|
| fork | 省略 subagent_type | 继承（有 cache） | 能 | 不能 | 低 |
| general-purpose | subagent_type="general-purpose" | 继承（无 cache） | 能 | 能 | 高 |
| Explore | subagent_type="Explore" | Haiku | 不能 | 不能 | 极低 |
| Plan | subagent_type="Plan" | 继承 | 不能 | 看配置 | 中 |

---

## 4.2 Agent prompt 写法

Agent 被创建后，除了 fork 会继承上下文，其他 Agent 看不到你之前的对话。你给 Agent 写的 prompt，等于给一个刚入职的聪明同事写工作简报。

6 条规则，每条展开说：

**第 1 条：像给新同事写简报，不像给自己写备忘录。**

你脑子里的上下文，Agent 一个字都不知道。"修那个 bug"——哪个 bug？"改一下那个文件"——哪个文件？你觉得显而易见的东西，对 Agent 来说是完全空白。写 prompt 的时候假设对方什么都不知道，但很聪明。

**第 2 条：解释目标和你已经发现的东西。**

不要只说"修这个 bug"，要说"用户点击提交按钮后页面白屏，错误日志显示 TypeError at line 42 of submit.ts。我已经确认不是网络问题，API 返回了 200，问题在前端渲染层。需要找到根因并修复。"你排除过的方向，省得 Agent 重新走一遍弯路。

**第 3 条：给足够上下文让它做判断，不要写死每一步。**

调查类任务给问题，不给步骤。"为什么用户列表页加载要 8 秒？"比"先看 API 响应时间，再看数据库查询，再看前端渲染"好。Agent 可能发现你没想到的原因——比如是 CDN 配置问题，你写的步骤里根本没提到。

**第 4 条：查询类任务反过来，直接给具体命令。**

"在 src/ 目录下搜索所有 console.log 语句，列出文件路径和行号。"这种机械性任务不需要 Agent 发挥创造力，给明确指令效率更高。

**第 5 条：禁止偷懒委托。**

"based on your findings, fix the bug"这种写法是在赌博。你必须自己消化 Agent 的发现，想清楚修复方案，再写下一条明确的修改指令。把"分析"和"执行"拆成两步，中间加上你的判断。这是用 Agent 最容易犯的错误——以为扔出去就不用管了。

**第 6 条：Agent 的产出对用户不可见。**

Agent 跑完后，结果只回传给主 Agent。主 Agent 需要自己总结后回复用户。所以你在 Agent prompt 里不需要考虑排版美观、不需要加开场白，直接输出结论和关键信息就行。反过来，主 Agent 拿到结果后要做一件事：把多个 Agent 的结果整合成一个清晰的回复给用户。

---

## 4.3 后台 Agent 生命周期

Agent 不是发出去就不管了。它有完整的生命周期，理解这 4 个阶段能帮你避开大部分"Agent 跑丢了""Agent 结果去哪了"的问题。

**阶段 1：启动**

设置 `run_in_background: true`，Agent 一出生就在后台跑。不设置的话默认在前台，会阻塞主 Agent 直到完成。

**阶段 2：自动转后台**

前台 Agent 超过 120 秒没完成，系统自动把它转入后台（内部叫 `getAutoBackgroundMs`，默认值 120000 毫秒）。这个设计很聪明——你不需要预判哪些任务会很久。发出去就行，短任务前台直接返回，长任务自动转后台，你可以继续和主 Agent 对话。

**阶段 3：完成通知**

Agent 跑完后，通过一条 user-role message 通知主 Agent。消息内容类似"Agent XXX 已完成，结果是……"。这条消息会出现在你的对话流里，就像有人发了一条新消息一样。

**阶段 4：继续对话**

用 `SendMessage` 工具可以继续和已有的 Agent 对话，通过 Agent 的 `name` 参数寻址。不需要重新创建。比如你之前派了一个叫 "code-reviewer" 的 Agent 去审查代码，它返回了 3 个问题。你可以用 SendMessage 继续跟它说"第 2 个问题能不能详细解释一下"，不用重新创建一个新 Agent。

**举个完整的场景：**

你在重构一个支付模块。你派出 3 个 Explore Agent 分别搜索：订单处理逻辑、支付网关集成、退款流程。3 个 Agent 同时跑，大概 10 秒内全部返回结果。你看完结果，发现退款流程有个竞态条件。你再派一个 fork Agent 去修复这个竞态条件——因为你之前的对话已经包含了所有上下文，fork Agent 继承了这些信息，不需要重新搜索。fork Agent 修复并 commit 后，结果回传给你。整个过程大概 2 分钟，如果纯手动可能要 20 分钟。

---

## 4.4 并行策略

一条消息里同时发多个 Agent 调用等于并行执行。这是 Claude Code 的杀手锏。

**安全并行的规则：**

只读 Agent 可以大量并行。Explore Agent、搜索类任务，开 10 个都没问题。它们各自搜各自的，互不干扰。

写操作 Agent 要防冲突。两个 Agent 同时改同一个文件，结果不可预测。让它们各管各的文件，或者用 worktree 隔离（4.6 节讲）。

fork 不可嵌套。内部通过检测 `<fork_boilerplate>` 标签来防止递归调用。如果 fork 出去的 Agent 试图再 fork，会被拦截。这是安全设计——无限递归的 Agent 会把你的 token 额度瞬间烧光。

**fork 子进程的行为约束：**

fork 出去的 Agent 遵循一套严格的规则：不跟用户对话、不提问、不请求确认，直接执行任务。修改文件后必须 commit——不 commit 的修改可能会被其他并行 Agent 覆盖。最终报告不超过 500 词，直接给主 Agent，不给用户。

**最佳实践模式——扇出搜索 + 扇入执行：**

```
同时发出 3-5 个 Explore Agent → 各自搜索不同方向
                    ↓
主 Agent 汇总所有结果 → 做判断
                    ↓
发出 1-2 个 fork Agent → 基于汇总信息执行修改
```

踩坑经历：有一次我同时派了 3 个 fork Agent 去改 3 个文件，结果其中 2 个文件互相 import，Agent A 改了函数签名，Agent B 还在用旧签名，最后代码编译不过。后来我学会了：有依赖关系的修改必须串行，只有真正独立的修改才并行。

---

## 4.5 自定义 Agent

在 `~/.claude/agents/` 放 Markdown 文件是全局自定义 Agent，在项目根目录 `.claude/agents/` 放是项目级的。文件名就是 Agent 的调用名。

**一个完整的代码审查 Agent 示例：**

```yaml
---
name: my-reviewer
description: 代码审查专家，检查代码质量和安全问题
tools: [Read, Glob, Grep, Bash]
disallowedTools: [Write, Edit]
model: sonnet
effort: high
maxTurns: 50
permissionMode: plan
isolation: null
background: false
---

你是一个代码审查专家。检查以下方面：

1. 安全漏洞：SQL 注入、XSS、命令注入、硬编码密钥
2. 错误处理：是否有裸 catch、是否有未处理的 Promise rejection
3. 代码风格：命名一致性、函数长度、嵌套深度
4. 性能问题：N+1 查询、不必要的循环、大对象拷贝

输出格式：
- 严重问题（必须修复）：列出文件路径 + 行号 + 问题描述
- 改进建议：列出建议但不阻塞合并
- 总体评价：1-10 分
```

**frontmatter 完整字段清单：**

| 字段 | 作用 | 可选值 |
|------|------|-------|
| name | Agent 名称，用于寻址和调用 | 字符串 |
| description | 描述，帮助主 Agent 决定何时调用 | 字符串 |
| tools | 允许使用的工具白名单 | 工具名数组 |
| disallowedTools | 禁用的工具黑名单 | 工具名数组 |
| model | 使用的模型 | sonnet / opus / haiku / inherit |
| effort | 推理深度 | high / medium / low |
| permissionMode | 权限模式 | plan / bypassPermissions / default / acceptEdits / auto |
| maxTurns | 最大循环轮数 | 数字（默认无限制） |
| mcpServers | 关联的 MCP 服务器 | 服务器名数组 |
| hooks | 生命周期钩子 | 钩子配置对象 |
| skills | 关联的 Skills | Skill 名数组 |
| memory | 记忆来源 | user / project / local |
| background | 是否后台运行 | true / false |
| isolation | 隔离模式 | worktree / remote / null |
| initialPrompt | 启动时的初始指令 | 字符串 |

**来源优先级（从低到高）：** built-in → plugin → userSettings → projectSettings → flagSettings → policySettings。后面的覆盖前面的。意思是：如果你在项目的 `.claude/agents/` 里定义了一个跟内置 Agent 同名的 Agent，你的定义会覆盖内置的。但如果组织管理员通过 policySettings 定义了同名 Agent，管理员的会覆盖你的。

实际操作建议：把通用的 Agent（代码审查、测试运行）放在 `~/.claude/agents/` 全局生效。把项目特定的 Agent（比如"按我们项目的 API 规范写接口"）放在 `.claude/agents/` 跟代码一起版本管理。

---

## 4.6 Worktree 隔离

大型重构或实验性修改，你不希望 Agent 在主仓库上动刀。万一改坏了，回滚的成本太高。Worktree 隔离解决这个问题。

设置 `isolation: "worktree"`，Agent 会创建一个独立的 git worktree——相当于把整个仓库复制了一份。Agent 在这个副本上工作，主仓库不受任何影响。

Agent 跑完后，如果没有任何文件修改，worktree 自动清理，不留垃圾。如果有修改，Agent 会返回 worktree 路径和分支名，你自己决定要不要合并回主分支。

手动控制两个工具：`EnterWorktreeTool` 进入 worktree（name 参数支持字母数字加点、下划线、短横线，最长 64 字符），`ExitWorktreeTool` 退出（action 参数：`keep` 保留修改，`remove` 丢弃）。

**什么时候该用 worktree：**

重构超过 5 个文件的大改动——改坏了能整体丢弃。试验性方案——可能要全部回滚。多个 Agent 要同时改同一个仓库的不同区域——每个 Agent 一个 worktree，互不干扰，最后分别合并。

不用 worktree 的话，两个 Agent 改同一个文件会冲突。用了 worktree，每个 Agent 一个副本，这是唯一靠谱的并行写操作方案。

---

## 4.7 Agent 选择决策树

拿到一个任务，按这个顺序判断：

```
这个任务需要修改文件吗？
│
├─ 不需要
│  ├─ 只是搜索 / 探索代码？ → Explore（Haiku，最快最便宜）
│  └─ 需要规划 / 设计？     → Plan（继承模型，能读代码辅助规划）
│
└─ 需要
   ├─ 简单任务 + 当前上下文有帮助？ → fork（省钱，缓存复用）
   ├─ 复杂多步骤 + 需要干净上下文？ → general-purpose（完整能力）
   └─ 需要隔离（大改 / 实验 / 并行写）？→ 任意写操作 Agent + worktree
```

**成本排序（同等任务，从便宜到贵）：**

Explore（Haiku，约 0.01 美元/次） < fork（缓存复用，约 0.05 美元/次） < Plan（继承模型，约 0.10 美元/次） < general-purpose（继承模型 + 完整能力，约 0.15 美元/次）

以上数字是典型中等任务的估算，实际取决于任务复杂度和 token 用量。但比例关系是稳定的：每升一级，成本翻倍以上。

记住一个原则：能用 Explore 就不用 fork，能用 fork 就不用 general-purpose。选最便宜的能完成任务的 Agent 类型。

---
