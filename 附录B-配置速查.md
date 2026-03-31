# 附录 B：settings.json 配置速查

配置文件位置：
- **用户级**：`~/.claude/settings.json`（全局生效）
- **项目级**：`.claude/settings.json`（只对当前项目生效）

优先级：企业级 > 项目级 > 用户级。项目级配置会覆盖用户级的同名配置项。

下面按 8 个场景分组，每项含类型、作用、默认值和注意事项。

---

## 1. 权限配置

| 配置项 | 类型 | 作用 | 默认值 |
|--------|------|------|--------|
| permissionMode | string | 权限模式 | "default" |
| alwaysAllowRules | string[] | 永远自动批准的工具/命令模式 | [] |
| alwaysDenyRules | string[] | 永远拒绝的工具/命令模式 | [] |
| alwaysAskRules | string[] | 永远需要手动确认的工具/命令模式 | [] |

**permissionMode 可选值：** plan（只看不改）、default（读自动写确认）、acceptEdits（读写自动 Bash 确认）、auto（AI 判断）、bypassPermissions（全部跳过，必须配合沙箱）。

**三条规则的优先级：** alwaysDeny > alwaysAsk > alwaysAllow。即使你在 alwaysAllow 里加了某个命令，如果 alwaysDeny 也匹配到了，deny 优先。

规则支持通配符和参数匹配：
- `"Read"` 匹配所有 Read 操作
- `"Bash(git *)"` 匹配所有 git 命令
- `"mcp__github__*"` 匹配 GitHub MCP 所有工具

```json
{
  "alwaysAllowRules": ["Read", "Glob", "Grep", "Bash(git status)", "Bash(git diff *)"],
  "alwaysDenyRules": ["Bash(rm -rf *)", "Bash(curl * | bash)"],
  "alwaysAskRules": ["Bash(git push *)"]
}
```

这个配置的意思：读操作和 git 查看命令自动批准，rm -rf 和管道执行永远拒绝，git push 每次都问。

---

## 2. 模型配置

| 配置项 | 类型 | 作用 | 默认值 |
|--------|------|------|--------|
| model | string | 默认使用的模型 | 根据订阅等级不同 |
| opusplan | boolean | plan 模式时用 Opus | false |

**注意：** model 字段改的是默认模型。对话中用 `/model` 命令切换会临时覆盖这个配置。

`opusplan: true` 让你在 plan 模式下自动使用 Opus 模型思考。适合架构设计阶段——用 Opus 的深度思考能力做规划，用 Sonnet 做执行，两者切换实现成本和质量的平衡。

---

## 3. MCP 配置

| 配置项 | 类型 | 作用 | 默认值 |
|--------|------|------|--------|
| mcpServers | object | MCP 服务器配置 | {} |
| enableAllProjectMcpServers | boolean | 自动信任项目级 MCP 服务器 | false |

**enableAllProjectMcpServers 安全风险：** 设为 true 意味着打开任何项目都会自动启动该项目配置的 MCP 服务器。恶意仓库可以在 `.claude/settings.json` 里藏 MCP 服务器，你 clone 后打开就中招了。除非你只打开自己的项目，否则保持 false。

mcpServers 的详细配置格式见第 8 章。每个服务器至少需要 type、command、args 三个字段。

---

## 4. Hooks 配置

| 配置项 | 类型 | 作用 |
|--------|------|------|
| hooks | object | 按事件类型组织的 Hook 配置 |

hooks 是一个嵌套对象。第一层 key 是事件名（16 种之一），第二层是 Hook 数组。

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "if": "FileWrite",
        "command": "npx prettier --write $TOOL_INPUT_FILE_PATH",
        "timeout": 10000,
        "statusMessage": "Formatting..."
      }
    ],
    "SessionStart": [
      {
        "command": "echo 'Session started at $(date)'",
        "once": true
      }
    ]
  }
}
```

每个 Hook 的 `if` 字段决定触发条件。不写 `if` 就是该事件每次都触发。`once: true` 表示只执行一次（适合初始化操作）。`async: true` 后台运行不阻塞。

详见第 7 章 Hooks 部分。

---

## 5. UI 配置

| 配置项 | 类型 | 作用 | 默认值 |
|--------|------|------|--------|
| theme | string | 主题 | "auto" |
| expandedView | boolean | 展开模式，显示工具执行细节 | false |
| verbose | boolean | 详细模式，显示 debug 级别信息 | false |

**theme 可选值：** "light"、"dark"、"auto"（跟随系统）。

**expandedView 什么时候开：** 调试时。它会显示每个工具调用的完整输入参数和返回结果。日常用 false，不然屏幕太吵，有效信息被淹没在大量的工具输出细节里。

**verbose 什么时候开：** 排查 MCP 连接问题、Hook 不触发等疑难杂症时。开了之后会看到连接握手过程、权限判断逻辑、Skill 匹配过程等。日常开发不需要。

---

## 6. Memory 配置

| 配置项 | 类型 | 作用 | 默认值 |
|--------|------|------|--------|
| autoMemoryDirectory | string | 自定义记忆存储路径 | ~/.claude/projects/\<项目\>/memory/ |
| autoMemoryEnabled | boolean | 是否启用自动记忆 | true |

关掉 `autoMemoryEnabled` 后，Claude 不再自动保存任何新记忆，也不再加载已有记忆到 system prompt。但已有的记忆文件不会被删除——重新开启后恢复。

`autoMemoryDirectory` 可以指向自定义路径。一个用法：让多个相关项目共享记忆，把它们的 autoMemoryDirectory 指向同一个目录。但一般不需要——项目隔离的记忆更干净，不会出现 A 项目的偏好干扰 B 项目的情况。

---

## 7. 安全配置

| 配置项 | 类型 | 作用 | 默认值 |
|--------|------|------|--------|
| sandbox | boolean / string | 沙箱模式 | true |
| dangerouslySkipPermissions | boolean | 跳过所有权限检查 | false |

**sandbox** 控制文件系统沙箱。macOS 上默认用 Apple Sandbox 限制文件访问范围到项目目录。Linux 上用 Docker 或其他容器技术。关掉意味着 Claude Code 可以读写机器上的任何文件——包括 ~/.ssh、~/.aws 这些敏感目录。

**dangerouslySkipPermissions** 名字里的 "dangerously" 不是吓人的。设为 true 等于给了 Claude Code 完全的系统访问权限——无确认执行任何命令、读写任何文件、调用任何 MCP 工具。唯一合理的使用场景是在完全隔离的 Docker 容器里跑自动化任务。在你的开发机上开这个，相当于把 sudo 密码给了一个 AI。

---

## 8. 环境变量配置

| 配置项 | 类型 | 作用 | 默认值 |
|--------|------|------|--------|
| env | object | 注入环境变量到运行环境 | {} |

```json
{
  "env": {
    "GITHUB_TOKEN": "ghp_xxx",
    "NODE_ENV": "development",
    "CLAUDE_AUTOCOMPACT_PCT_OVERRIDE": "70"
  }
}
```

这些变量对 Claude Code 执行的所有 Bash 命令可见。也对 MCP 服务器的启动命令可见。

**安全提醒：** 不要把敏感 token 放在项目级配置里——项目级配置可能被 git 追踪。放在用户级配置（`~/.claude/settings.json`）更安全。

一个实用技巧：把 `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` 放在 env 里。这个环境变量控制自动压缩的触发百分比（见第 6 章）。Sonnet 用户设 60-70 能提前触发压缩，避免上下文过满导致质量下降。

还有一个常用变量 `EDITOR`，设成你喜欢的编辑器（vim、code、nano）。Ctrl+X → Ctrl+E 的外部编辑器就读这个变量。
