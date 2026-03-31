# 第 1 章：写出真正有效的 CLAUDE.md

**读完这章你能得到什么：** 理解 CLAUDE.md 的真实注入机制，知道它为什么有时候不生效，掌握 4 层优先级的覆盖关系，学会用 @include 拆分长配置，拿到 3 个可以直接复制使用的实战模板。

你现在要做的就是打开你的 CLAUDE.md，按这章的方法重写一遍。

---

## 1.1 真实注入格式：它到底长什么样

上一章说了 CLAUDE.md 是第一条 user message，但"第一条 user message"到底是什么样子？看完整结构：

```
<system-reminder>
As you answer the user's questions, you can use the following context:
# claudeMd
Codebase and user instructions are shown below. Be sure to adhere to
these instructions. IMPORTANT: These instructions OVERRIDE any default
behavior and you MUST follow them exactly as written.

Contents of ~/.claude/CLAUDE.md (user's global instructions):
[你的全局 CLAUDE.md 内容]

Contents of /project/CLAUDE.md (project-level instructions):
[你的项目级 CLAUDE.md 内容]

# currentDate
Today's date is 2026-03-31.

      IMPORTANT: this context may or may not be relevant to your tasks.
      You should not respond to this context unless it is highly relevant
      to your task.
</system-reminder>
```

有 3 个关键点你必须注意。

**第 1 点：开头写了"OVERRIDE any default behavior"。** 这句话是让模型把你的 CLAUDE.md 当作高优先级指令来对待。但别高兴太早——它只是在 user message 层面的"高优先级"，对 system prompt 层面的规则依然无效。

打个比方：你是部门经理，你的指令对团队成员是"最高优先级"，但对董事会定下的公司规章，你改不了。

**第 2 点：结尾写了"may or may not be relevant"。** 这不是废话。它告诉模型：你自己判断这段内容跟当前任务有没有关系。

模型不会无条件执行你写的每一条规则——如果你在 CLAUDE.md 里写了 Python 的命名规范，但当前在写 TypeScript，模型可能判定这条规则"不相关"直接跳过。

**第 3 点：多层 CLAUDE.md 会被拼接在一起。** 全局的、项目级的、rules 目录下的，所有 CLAUDE.md 内容按优先级顺序拼成一大段文本，一起塞进这个 system-reminder 标签里。

它们共享同一个上下文窗口——所以总长度是有硬限制的。

当 CLAUDE.md 的指令和 system prompt 冲突时，谁赢？system prompt 赢。我一开始在 CLAUDE.md 里写"用 cat 命令读取文件内容"，结果模型每次都用 Read 工具。

反复试了几次才想明白——system prompt 第 5 段明确写了"专用工具优先于 Bash"，这条铁律不会被 user message 层的指令覆盖。把指令改成"读取文件时优先检查完整内容"后，模型照做了，因为这条不跟 system prompt 冲突。

---

## 1.2 4 层优先级：后面覆盖前面

CLAUDE.md 有 4 个层级，后加载的覆盖先加载的：

| 优先级 | 路径 | 用途 | 谁维护 |
|---|---|---|---|
| 最低 | `/etc/claude-code/CLAUDE.md` | 系统级 | 企业 IT 统一管理 |
| 低 | `~/.claude/CLAUDE.md` | 用户全局 | 你自己 |
| 高 | 项目根目录 `CLAUDE.md` / `.claude/CLAUDE.md` / `.claude/rules/*.md` | 项目级 | 团队 |
| 最高 | `CLAUDE.local.md` | 本地私有 | 你自己，不进 git |

**覆盖规则：** 后面覆盖前面。你在全局写了"用英文回复"，项目级写了"用中文回复"，最终模型用中文——项目级优先级更高。

**合并规则：** 同一层级的多个文件会被合并。项目根目录同时有 `CLAUDE.md` 和 `.claude/CLAUDE.md` 和 `.claude/rules/` 下 3 个文件，这 5 份内容全部拼在一起注入，不会互相覆盖。

我踩过一个坑：全局 CLAUDE.md 里写了 20 条通用规则（大约 1200 字），每个项目的 CLAUDE.md 又写了 15 条项目规则。加起来 2000 多字全部注入到对话里，模型对单条规则的遵循率明显下降。

后来把全局规则砍到 5 条核心偏好（200 字），项目规则控制在 800 字以内，遵循率马上回来了。全局配太多，会污染每一个项目的上下文空间。

**你现在要做的：** 打开你的全局 CLAUDE.md（`~/.claude/CLAUDE.md`），问自己——这些规则在过去 10 次对话中触发过几次？触发过 0 次的全部删掉。全局只保留跨所有项目都需要的偏好。

---

## 1.3 为什么你的 CLAUDE.md 有时候不生效

3 个原因，按出现频率排序。遇到规则不生效，按这个顺序排查。

### 原因 1：它是 user message，模型会自行判断相关性

还记得注入格式里那句"may or may not be relevant"吗？模型真的会判断。你在 CLAUDE.md 里写了"所有 React 组件用 function 声明不用 class"，但当前对话在讨论数据库 migration——模型判定这条规则跟当前任务无关，直接忽略了。

**排查方法：** 把规则写得跟你的实际工作场景强关联。不要写"React 组件用 function 声明"，写"在 components/ 目录下创建新文件时，必须使用 function 声明的 React 组件"。越具体的规则越不容易被判定为"不相关"。

### 原因 2：被更高优先级的文件覆盖

你在 `~/.claude/CLAUDE.md` 写了"commit message 用英文"，但项目的 `.claude/rules/git.md` 里写了"commit message 用中文"。项目级优先级更高，全局配置被覆盖了。

**排查方法：** 检查 4 个层级的所有文件，看是否有冲突的指令。特别注意 `.claude/rules/` 目录下可能有队友加的规则文件。你可以在对话中直接问模型"你现在看到的 CLAUDE.md 内容是什么"，它会把所有已加载的内容告诉你。

### 原因 3：写得太长，模型注意力被稀释

CLAUDE.md 占用的是对话上下文窗口。你塞了 3000 字的规则，模型需要在这 3000 字里找到跟当前任务相关的那几条。

注意力被稀释，重要规则的遵循率下降。

更严重的问题是空间挤占。对话上下文窗口总共就那么大（Opus 1M tokens、Sonnet 200K tokens），CLAUDE.md 占得越多，留给后续对话内容的空间越少。

你的 CLAUDE.md 写了 5000 字，相当于每轮对话都背着 5000 字的"包袱"跑——最终会更早触发上下文压缩，导致之前对话的细节被丢弃。

**排查顺序：**
1. 规则是否写得足够具体（排除原因 1）
2. 是否有更高优先级的文件覆盖了（排除原因 2）
3. 总字数是否超过 1500 字（排除原因 3）

---

## 1.4 @include 指令：把长配置拆成多个文件

CLAUDE.md 支持用 `@` 指令引入其他文件的内容，4 种语法：

| 语法 | 含义 | 示例 |
|---|---|---|
| `@path` | 相对于当前 CLAUDE.md 所在目录 | `@rules/style.md` |
| `@./relative` | 相对路径 | `@./docs/api-spec.md` |
| `@~/home` | 用户主目录 | `@~/shared-rules/typescript.md` |
| `@/absolute` | 绝对路径 | `@/etc/company/coding-standards.md` |

用途很直接：当 CLAUDE.md 超过 1000 字，按主题拆成多个文件用 @include 引入。拆分后每个文件独立维护，不同项目可以复用同一份规则。

比如你有一份 TypeScript 编码规范 300 字，所有 TS 项目都 @include 这同一份文件，改一处全部生效。

---

## 1.5 写法策略：因为是 user message，所以你应该这样写

### 不写角色设定

"你是一个资深 Python 开发者"——删掉。System prompt 里已经有完整的角色定义了（"You are Claude Code, Anthropic's official CLI"）。

你在 user message 里重复定义角色，模型会困惑到底听谁的。最终结果是两边都不完全遵守。

### 用 Markdown 标题分块

模型对结构化内容的解析效果远好于一坨纯文本。用 `#` `##` 标题把规则分组，每组不超过 5 条规则。模型内部的注意力机制对标题标记有更强的响应——分了块的规则比连续列表的遵循率高。

### 把最重要的规则放最前面

模型对上下文开头和结尾的内容注意力最高，中间的容易被稀释。你最在意的 3 条规则，放在 CLAUDE.md 的最开头。

### 甜蜜点：500-1000 字

低于 500 字说明你还没把关键规则写清楚。超过 1500 字效果开始递减——不是完全不生效，而是单条规则的遵循概率在下降。

我测过把 CLAUDE.md 从 2000 字砍到 800 字，模型对核心规则（比如"不自动 push"）的遵循率从大约 7 成提到了 9 成以上。

### 写具体规则，不写模糊描述

**反面例子：** "写高质量的代码"——模型不知道你说的"高质量"是什么意思。
**正面例子：** "函数不超过 30 行，超过就拆成子函数。每个 API 请求必须有 try-catch 错误处理。不引入新的第三方依赖除非我明确要求。"

每条规则需要通过一个测试：模型看完之后，能不能在没有任何歧义的情况下执行？如果你自己都没法判断某条规则有没有被遵守，模型也判断不了。

---

## 1.6 .claude/rules/ 多人协作

项目根目录下的 `.claude/rules/` 文件夹里可以放任意数量的 `.md` 文件，全部会被加载并合并到 CLAUDE.md 的内容中。

这个机制专门解决多人协作的痛点。团队 5 个人都去改同一个 CLAUDE.md，天天 merge conflict。

用 rules 目录，每个人管自己负责的 rule 文件：后端的人管 `api-patterns.md`，前端的人管 `ui-patterns.md`，QA 管 `testing-rules.md`。各改各的文件，零冲突。

个人偏好放 `CLAUDE.local.md`（在 .gitignore 里，不进版本控制），团队规则放 `.claude/rules/`（进 git，所有人共享）。这是 Claude Code 官方设计的协作分工方式。

---

## 1.7 3 个实战模板

### 模板 1：个人全局配置

文件路径：`~/.claude/CLAUDE.md`

对所有项目生效，只放跨项目通用的核心偏好。控制在 200 字以内。

```markdown
# 核心偏好

- 用中文回复，技术术语保留英文原文
- 不用 emoji
- 代码注释用英文

# 编码习惯

- 变量命名：TypeScript 用 camelCase，Python 用 snake_case
- 函数不超过 30 行，超过就拆
- 不主动引入新的第三方依赖

# Git 约定

- commit message 用英文，格式：type(scope): description
- 不要自动 push，等我确认
- 不要用 git checkout/restore 撤回改动
```

这份配置大约 160 字，覆盖了语言、编码、Git 3 个维度。注意最后一条——"不要用 git checkout/restore 撤回改动"——这是防止模型误操作丢失你未提交的工作。

### 模板 2：项目配置

文件路径：项目根目录 `CLAUDE.md`

告诉模型这个项目的技术栈、目录结构、关键约定。控制在 500 字以内。

```markdown
# 技术栈

- Next.js 14 (App Router) + TypeScript + Tailwind CSS
- 数据库：PostgreSQL + Prisma ORM
- 部署：Vercel
- 测试：Vitest + Testing Library

# 项目结构

- app/ — 页面和路由（App Router 约定）
- lib/ — 工具函数和业务逻辑
- components/ — UI 组件，按 atoms/molecules/organisms 三层组织
- prisma/ — 数据库 schema 和 migration

# 关键约定

- 组件用 function 声明，不用 arrow function export
- 状态管理只用 React hooks，不引入 Redux/Zustand/Jotai
- 所有数据库查询走 lib/db/ 下的封装函数，组件里不直接调 Prisma
- API 路由统一用 app/api/ 下的 route handler
- 新增 API 必须有入参校验（用 zod）

# 测试规范

- 改动涉及 lib/ 目录的函数必须有单元测试
- 测试文件放同目录 __tests__/ 下，命名 xxx.test.ts
- PR 前跑 pnpm test 确保全部通过
```

这份配置大约 350 字。模型读完就知道：用什么技术栈、代码放哪里、怎么写、怎么测。每条规则都是可执行、可验证的。

### 模板 3：多人协作项目

目录结构：

```
.claude/
├── CLAUDE.md                  # 索引文件，@include 各 rule
├── rules/
│   ├── api-conventions.md     # 后端同学维护（~200 字）
│   ├── ui-patterns.md         # 前端同学维护（~200 字）
│   └── testing.md             # QA 同学维护（~200 字）
```

`.claude/CLAUDE.md` 内容：

```markdown
# 项目规则

技术栈：Next.js 14 + TypeScript + PostgreSQL + Prisma

@./rules/api-conventions.md
@./rules/ui-patterns.md
@./rules/testing.md
```

`api-conventions.md` 示例：

```markdown
# API 约定

- RESTful 命名：GET /api/users, POST /api/users, GET /api/users/:id
- 响应格式统一：{ data, error, meta }
- 错误码用 HTTP 标准状态码，业务错误放 error.code 字段
- 每个 route handler 必须有 try-catch，catch 里返回 500 + 错误日志
- 数据库事务用 prisma.$transaction，不手写 BEGIN/COMMIT
```

每个人的 `CLAUDE.local.md`（不进 git）：

```markdown
# 我的偏好

- 我负责前端模块，改后端代码前先跟我确认
- 生成代码后自动运行 pnpm lint
- PR 描述用中文写
```

这种结构下，团队共享规则 3 个人各管一份文件，个人偏好各写各的，零冲突。所有 rules 文件加起来不超过 800 字，加上索引和 local 总共不超过 1000 字——正好在甜蜜点范围内。


---
