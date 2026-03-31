# 第 8 章：MCP 集成指南

MCP 让 Claude Code 从"只能读写本地文件"变成"能操作任何有 API 的系统"。GitHub、Slack、数据库、飞书、你自己的后端——只要有人写了 MCP 服务器，Claude Code 就能直接调用。

配置得当的话，你永远不需要离开终端去浏览器操作 GitHub，也不需要手动复制 Slack 消息给 Claude。

---

## 8.1 MCP 一句话解释

MCP（Model Context Protocol）= 让 AI 使用外部工具和数据源的标准协议。

USB 让电脑连接键盘、鼠标、硬盘，设备厂商只需要实现 USB 标准。MCP 让 Claude Code 连接 GitHub、Slack、数据库，服务提供方只需要实现 MCP 标准。你不需要了解协议细节，只需要知道怎么配置和使用。

协议定义了三种资源类型：**工具**（tools，可执行的函数）、**资源**（resources，可读取的数据源）、**提示**（prompts，预定义的 prompt 模板即 Skill）。日常用到最多的是工具。

---

## 8.2 传输协议

MCP 服务器和 Claude Code 之间有 4 种连接方式：

**stdio（标准输入输出）：** 本地命令行启动一个进程，通过 stdin/stdout 通信。90% 的 MCP 服务器用这个方式。简单可靠，不需要网络，进程直接通信。所有 `npx -y @modelcontextprotocol/server-xxx` 形式的服务器都是 stdio。

**sse（Server-Sent Events）：** 通过 HTTP 长连接，服务器单向推送数据给客户端。适合远程托管的 MCP 服务——比如公司内网部署了一个 MCP 网关，你通过 URL 连接。优点是兼容性好（任何支持 HTTP 的环境都能用），缺点是单向流，客户端到服务器需要额外的 HTTP 请求。

**http（Streamable HTTP）：** MCP 协议的新标准传输方式，正在替代 sse。支持双向流，性能更好。如果 MCP 服务器支持 http，优先用 http 而不是 sse。

**ws（WebSocket）：** 全双工实时通信，延迟最低。适合需要实时交互的场景——比如实时协作编辑、流式数据处理。日常开发场景很少需要 ws。

> 【截图位置：MCP 状态面板截图，显示不同协议的连接状态】

**怎么选？** 本地工具用 stdio（不需要想），远程服务看提供方支持什么——有 http 用 http，只有 sse 就 sse。需要实时双向的罕见场景才考虑 ws。

---

## 8.3 配置方式

三种配置方式，按使用频率排序：

### 项目级配置（.claude/settings.json）

放在项目根目录的 `.claude/settings.json` 里，只对当前项目生效。适合团队共享的 MCP 配置——比如项目专用的数据库 MCP。

```json
{
  "mcpServers": {
    "github": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "ghp_你的token"
      }
    }
  }
}
```

4 个核心字段：
- `type`：传输协议，绝大多数情况是 `"stdio"`
- `command`：启动命令，通常是 `npx` 或具体的二进制路径
- `args`：传给启动命令的参数，数组格式
- `env`：环境变量，放 API token 等认证信息

踩坑提醒：`env` 里的 token 会被写入文件。如果 `.claude/settings.json` 被 git 追踪，你的 token 就泄露了。两种解决方案：把 `.claude/settings.json` 加进 `.gitignore`，或者把含 token 的配置放在用户级而非项目级。

### 用户级配置（~/.claude/settings.json）

格式完全一样，区别是全局生效，所有项目都能用。适合个人通用的 MCP 配置——GitHub MCP 你在所有项目里都用得到，放用户级一劳永逸。

```json
{
  "mcpServers": {
    "github": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "ghp_你的token"
      }
    },
    "slack": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@anthropic/mcp-server-slack"],
      "env": {
        "SLACK_BOT_TOKEN": "xoxb-你的token"
      }
    }
  }
}
```

### CLI 管理命令

不想手动编辑 JSON？用命令行：

```bash
# 添加 MCP 服务器（默认添加到用户级配置）
claude mcp add github -- npx -y @modelcontextprotocol/server-github

# 添加到项目级配置
claude mcp add github --project -- npx -y @modelcontextprotocol/server-github

# 列出所有已配置的 MCP 服务器
claude mcp list

# 删除
claude mcp remove github

# 带环境变量添加
claude mcp add github -e GITHUB_TOKEN=ghp_xxx -- npx -y @modelcontextprotocol/server-github
```

CLI 命令的好处是不会写错 JSON 格式。手动编辑 JSON 最常见的错误是漏逗号或多逗号——CLI 帮你避免这个问题。

---

## 8.4 七层配置来源

MCP 配置有 7 个来源，优先级从低到高：

```
enterprise → managed → user global → project local → dynamic → claudeai proxy → plugin
```

| 来源 | 说明 | 你会接触到吗 |
|------|------|-------------|
| enterprise | 企业 IT 管控 | 除非你在大公司，否则不会 |
| managed | 托管配置 | 同上 |
| **user global** | ~/.claude/settings.json | **你最常用的** |
| **project local** | .claude/settings.json | **项目级配置** |
| dynamic | 运行时动态注册 | 很少手动操作 |
| claudeai proxy | Claude.ai 代理 | 自动的，不用管 |
| plugin | 插件注册 | 安装插件时自动 |

高优先级覆盖低优先级。实操中你接触最多的是 user global 和 project local。

**如果同一个 MCP 服务器在 user global 和 project local 都配了，project local 的生效。** 这让你能在特定项目里覆盖全局配置——比如全局用个人 GitHub token，某个开源项目用团队 token。

---

## 8.5 连接状态排查

配好之后不一定能连上。Claude Code 启动时会尝试连接所有配置的 MCP 服务器，每个服务器有 5 种可能的状态：

| 状态 | 含义 | 怎么办 |
|------|------|--------|
| **connected** | 正常运行 | 不用管，一切正常 |
| **failed** | 连接失败 | 最常见的问题。见下面的排查步骤 |
| **needs-auth** | 需要 OAuth 认证 | 按终端提示完成浏览器认证流程。通常只需要认证一次 |
| **pending** | 正在重连 | 等 10 秒。如果一直 pending，重启 Claude Code |
| **disabled** | 已禁用 | 在 settings.json 中移除 `"disabled": true` 字段 |

**failed 状态排查 4 步法：**

第一步：在终端手动运行 MCP 服务器的启动命令。比如 `npx -y @modelcontextprotocol/server-github`，看是不是命令本身就报错。最常见的原因是 npx 找不到包（网络问题）或 Node.js 版本不兼容（要求 18+）。

第二步：检查 `env` 里的环境变量是否完整。GitHub MCP 没有 `GITHUB_TOKEN` 就会 failed。Slack MCP 没有 `SLACK_BOT_TOKEN` 也是。

第三步：检查 command 路径。如果你用的不是 npx 而是本地安装的二进制，确认路径是否正确。`which npx` 看看 npx 在哪。

第四步：看 Claude Code 的日志。启动时加 `--verbose` 或者设 `verbose: true`，日志里会有 MCP 连接的详细错误信息。

---

## 8.6 工具权限和延迟加载

### 权限命名空间

MCP 工具在权限系统里的命名格式是 `mcp__<server>__<tool>`。

比如 GitHub MCP 的 create_pull_request 工具 → 权限名是 `mcp__github__create_pull_request`。

配置自动批准所有 GitHub 操作：

```json
{
  "alwaysAllowRules": [
    "mcp__github__*"
  ]
}
```

通配符 `*` 匹配该服务器下所有工具。你也可以精细控制——只自动批准读操作，写操作仍需确认：

```json
{
  "alwaysAllowRules": [
    "mcp__github__get_*",
    "mcp__github__list_*",
    "mcp__github__search_*"
  ]
}
```

### 延迟加载

MCP 工具默认**延迟加载（deferred loading）**。模型在 system prompt 里只看到工具名和简短描述，没有完整的参数定义——参数名是什么、类型是什么、哪些必填哪些可选，都不知道。

调用前必须先通过 `ToolSearch` 拿到完整定义。这意味着每次调用一个**新的** MCP 工具，都要多走一步搜索。

为什么要这样设计？因为一个 MCP 服务器可能暴露 50 个工具，每个工具的完整定义 200 tokens，50 个就是 10K。如果所有 MCP 工具都在 system prompt 里全量展示，光工具定义就占掉几万 tokens。

延迟加载的代价：第一次调用某个 MCP 工具时速度慢一拍（多一轮 ToolSearch）。

解决办法：MCP 服务器端在工具的 `_meta` 中设置 `"anthropic/alwaysLoad": true`。设了这个标记的工具会跳过延迟加载，直接在 system prompt 里展示完整定义。但这需要 MCP 服务器端支持——第三方服务器你控制不了，只能接受延迟加载。

好消息：同一个工具在同一个会话里只需要 ToolSearch 一次，之后就缓存了。

---

## 8.7 三个最有用的 MCP 服务器

### 1. GitHub MCP

管理 PR、Issues、代码搜索、仓库操作。开发者用 Claude Code 几乎必装。

```json
{
  "mcpServers": {
    "github": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "ghp_你的token"
      }
    }
  }
}
```

**GitHub Token 怎么拿：** GitHub → Settings → Developer Settings → Personal Access Tokens → Generate new token (classic)。勾选 repo、issues、pull requests 权限。

典型用法：
- "帮我创建一个 PR，标题是 XX，描述写 YY"
- "看看 #123 这个 issue 的评论"
- "搜一下仓库里有没有类似的实现"
- "review 一下 PR #456 的 diff"

跟 `gh` CLI 的区别：gh 需要你记命令语法，MCP 让你用自然语言操作。Claude 自动把你的意图翻译成 API 调用。

### 2. Filesystem MCP

增强的文件操作——比内置的 Read/Write 多了目录树展示、文件移动、文件监控等能力。

```json
{
  "mcpServers": {
    "filesystem": {
      "type": "stdio",
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "/你的/项目/路径"
      ]
    }
  }
}
```

注意最后一个参数是**根目录限制**。MCP 服务器只能访问这个目录及其子目录。不设的话整个文件系统都暴露了——这是个安全风险，特别是 auto 模式下 Claude 可能读到你不想让它看的文件。

什么时候需要 Filesystem MCP？当你需要操作项目外的文件时。Claude Code 内置的 Read/Write 受沙箱限制，只能操作当前项目目录。Filesystem MCP 可以突破这个限制（在你指定的根目录范围内）。

### 3. Slack MCP

让 Claude Code 发消息到 Slack。最有用的场景：长任务跑完后自动通知你。

```json
{
  "mcpServers": {
    "slack": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@anthropic/mcp-server-slack"],
      "env": {
        "SLACK_BOT_TOKEN": "xoxb-你的token"
      }
    }
  }
}
```

**Slack Bot Token 怎么拿：** Slack API 官网 → Create New App → From scratch → 添加 Bot Token Scopes（chat:write, channels:read）→ Install to Workspace → 复制 Bot User OAuth Token。

配合 Hooks 的 asyncRewake 用效果极佳：

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "if": "Bash",
        "command": "if [ $TOOL_EXIT_CODE -eq 0 ]; then echo 'Task completed'; fi",
        "asyncRewake": true
      }
    ]
  }
}
```

长任务后台跑 → 完成后 Hook 唤醒模型 → 模型调用 Slack MCP 发通知 → 你在手机上看到"XX 任务完成了"。整个流程不需要你盯着终端。

我自己的用法是飞书 MCP（社区维护的）替代 Slack MCP。原理一样，配置格式差不多，国内用飞书更方便。

---
