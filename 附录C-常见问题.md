# 附录 C：常见问题排查

12 个高频问题，每个给原因分析和具体操作步骤。

---

## 1. 命令被拒绝执行

**现象：** 输入一个 Bash 命令，Claude Code 拒绝执行，弹出红色提示或直接说"无法执行"。

**原因分析：** BashTool 有四层安全拦截。从外到内：
- 第一层：命令注入检测（20+ 检测器，识别管道注入、反引号、$() 嵌套等）
- 第二层：sed 白名单（只允许行打印和简单替换两种写法，其他全部拦截）
- 第三层：路径越界检测（30+ 命令验证，防止读写项目目录外的文件）
- 第四层：只读命令安全标志白名单（标记为安全的只读命令才能跳过权限确认）

**操作步骤：**
1. 看拒绝信息判断是哪一层。如果提示"blocked by policy"是前两层，"requires approval"是后两层
2. 前两层拦的没法绕——改用其他方式实现。比如 `curl | bash` 被封，就先 `curl -o script.sh url` 下载，然后单独 `bash script.sh`
3. 第三层拦的（模型觉得危险），换一种更明确的写法。比如 `rm -rf ./temp/` 比 `rm -rf *` 更容易通过
4. 第四层拦的，在 `alwaysAllowRules` 里加白名单。详见第 5 章

---

## 2. sed 命令报错

**现象：** 写了 sed 替换命令但被拒绝，提示不允许执行。

**原因分析：** BashTool 对 sed 有白名单限制。只允许两种写法：
- 写法 1：`sed -n 'Np'`（行打印，包括 `sed -n 'X,Yp'` 打印范围）
- 写法 2：`sed 's/pattern/replacement/flags'`（简单替换，只允许 `/` 作分隔符，标志只允许 g/p/i/I/m/M/1-9）

所有其他 sed 用法都被封锁——w/W 写文件、e/E 执行命令、花括号代码块、-e 多表达式等全部拦截。这是为了防止 sed 的复杂特性被利用做命令注入。

**操作步骤：**
1. 如果你只需要查看特定行，`sed -n '10,20p' file.txt` 是允许的
2. 简单替换也可以用 sed：`sed 's/old/new/g' file.txt`，但分隔符只能用 `/`，标志只能用 g/p/i/I/m/M/1-9
3. 复杂的 sed 用法（多表达式、花括号块、写文件等）不要尝试——直接用 Edit 工具代替，给出 old_string 和 new_string，比 sed 更安全也更准确

---

## 3. MCP 工具"消失"了

**现象：** 配置了 MCP 服务器，也显示 connected，但 Claude 说找不到工具或者不知道怎么调用。

**原因分析：** MCP 工具默认延迟加载。模型在 system prompt 里只看到工具名字和简短描述，没有完整的参数定义（参数名、类型、是否必填都不知道）。

**操作步骤：**
1. 让 Claude 用 ToolSearch 搜索工具名。比如说"搜索 github 的 create_pull_request 工具定义"
2. 搜到完整定义后，同一个会话里不需要再搜——已缓存
3. 如果你控制 MCP 服务器代码，在工具的 `_meta` 里设 `"anthropic/alwaysLoad": true` 跳过延迟加载
4. 如果是第三方 MCP 服务器，你控制不了 _meta，只能接受延迟加载。可以在 CLAUDE.md 里写"使用 XX MCP 工具前先用 ToolSearch 获取定义"

---

## 4. 对话变慢，回复质量下降

**现象：** 聊了很久之后，回复速度变慢，内容开始重复或遗忘之前的约定。比如你说过"用 TypeScript"但它开始写 JavaScript。

**原因分析：** 接近上下文窗口上限。两个因素叠加：
- 上下文太长导致注意力分散——模型在 500K tokens 里找关键信息的能力是递减的
- 压缩机制已经开始丢弃信息——Context Collapse 可能把你之前的关键约定折叠成了一句泛泛的摘要

**操作步骤：**
1. 立即手动执行 `/compact`，压缩当前对话
2. 压缩后检查——重新提一次关键约定，比如"继续用 TypeScript strict mode"
3. 如果质量已经很差，新开一个会话。第一条消息写清楚上下文："我在做 XX 项目，关键约束是 YY，刚完成了 ZZ，现在要做 AA"
4. 以后养成习惯：每完成一个子任务就 /compact 一次，不要等到 300K+ 再处理。详见第 6 章

---

## 5. 费用异常偏高

**现象：** 一个看起来不复杂的任务，花了远超预期的 token 量和费用。

**原因分析：** 最常见的原因是 **spawn Agent**。spawn 创建全新的上下文，不共享 prompt cache。如果你的操作触发了多次 spawn（比如并行处理 5 个文件，每个 spawn 一个 Agent），每次都是从零开始——system prompt、工具定义、CLAUDE.md 全部重新加载，cache 命中率为零。

另一个常见原因：MCP 工具返回了超大结果。比如一个数据库查询 MCP 返回了 10 万行数据，全部变成了 token。

**操作步骤：**
1. 查看 transcript 文件（Ctrl+O 打开），确认是不是 spawn 导致的。transcript 里能看到每个 Agent 的创建和 token 消耗
2. 如果是 spawn 导致，改用 fork。fork 继承父进程上下文，能吃 prompt cache，成本低很多。详见第 3 章
3. 如果是 MCP 返回数据过大，在 MCP 服务器端加分页或限制返回量
4. 检查是否有循环——Claude 有时候会陷入"修改→测试失败→修改→测试失败"的循环，每轮都消耗大量 token。在 CLAUDE.md 里加"如果连续 3 次修改同一个文件仍然失败，停下来问我"

---

## 6. auto 模式频繁要求确认

**现象：** 明明设了 auto 模式，Claude Code 还是经常停下来问你"是否允许执行"。

**原因分析：** auto 模式不是"全部放行"。背后有 AI 分类器实时判断每个工具调用的风险。如果分类器连续判定多个操作有风险，会自动回退到更保守的模式——等于 auto 临时降级成了 default。

另一种可能：你的 `alwaysAskRules` 匹配范围太广，覆盖了 auto 模式的自动批准。

**操作步骤：**
1. 检查 `alwaysAskRules` 是否过宽。比如 `"Bash(*)"` 会匹配所有 Bash 命令——这相当于取消了 auto 模式
2. 用 `alwaysAllowRules` 把高频且安全的操作加白：`["Read", "Glob", "Grep", "Bash(git status)", "Bash(git diff *)"]`
3. 如果是分类器回退，说明你的操作序列被判定为高风险。换一种更安全的实现方式，或者在一系列安全操作后再尝试
4. 检查是否有 PreToolUse Hook 返回了非 "approved" 的结果——Hook 返回其他值等于拒绝

---

## 7. CLAUDE.md 不生效

**现象：** 在 CLAUDE.md 里写了"用中文回复"，但 Claude 还是用英文。或者写了"不要创建 README"，但它还是创建了。

**原因分析：** 3 种可能：
- 被高优先级配置覆盖——企业级或系统级的 CLAUDE.md 有矛盾的规则
- 文件太长被截断——超过一定长度后面的内容注意力权重极低
- CLAUDE.md 放错了位置——没被自动发现

**操作步骤：**
1. 确认文件位置。Claude Code 按这个顺序查找：项目根目录 `CLAUDE.md` → `.claude/CLAUDE.md` → `.claude/rules/*.md` → `CLAUDE.local.md`
2. 确认文件被加载了。让 Claude 回答"你看到了哪些 CLAUDE.md 规则"，它会列出当前生效的配置
3. 检查长度。CLAUDE.md 控制在 500-1000 字以内。超长的 CLAUDE.md 后半部分的规则容易被"忽视"——不是没看到，是注意力被前面的内容占了
4. 检查优先级冲突。4 层优先级从低到高：系统级 → 用户级 → 项目级 → 本地级。如果两层都有"回复语言"规则，高优先级的生效
5. 把最重要的规则放在文件最前面。模型对开头的内容注意力最高

---

## 8. MCP 服务器连不上（状态 failed）

**现象：** 配置了 MCP 服务器，启动 Claude Code 后状态显示 failed。

**原因分析：** 4 种常见原因：启动命令错误、依赖没安装、环境变量缺失、端口冲突。

**操作步骤：**
1. 在终端手动运行启动命令。比如 `npx -y @modelcontextprotocol/server-github`。如果报 "not found"，是 npm 包名错了或网络问题
2. 检查 Node.js 版本。大多数 MCP 服务器要求 Node 18+。`node --version` 确认
3. 检查 settings.json 里的 `env` 字段。GitHub MCP 必须有 `GITHUB_TOKEN`，Slack MCP 必须有 `SLACK_BOT_TOKEN`
4. 检查 JSON 格式。最常见的错误：多了逗号（最后一个键值对后面不能有逗号）、少了引号、args 不是数组格式
5. 开 verbose 模式看详细日志：settings.json 里加 `"verbose": true`，重启 Claude Code 后看 MCP 握手过程的报错信息

---

## 9. fork Agent 失败或行为异常

**现象：** 用 fork 创建子 Agent 报错，或者子 Agent 完全不按预期执行。

**原因分析：** fork 有限制：
- fork 不能嵌套——fork 出来的 Agent 不能再 fork
- 上下文太大时 fork 可能失败——fork 继承父进程上下文，如果父进程已经接近上限，fork 后直接超了
- fork Agent 的工具权限继承父进程，但如果父进程在 plan 模式，fork Agent 也是 plan 模式（只读不写）

**操作步骤：**
1. 确认你不是在 fork 的 Agent 里再 fork。如果需要多层并行，用 spawn 代替（spawn 创建独立上下文）
2. 如果报"context too large"，先 `/compact` 压缩主对话再 fork
3. 如果 fork Agent 行为异常（比如不执行修改操作），检查当前的 permissionMode。在 plan 模式下 fork 出来的 Agent 也是只读的
4. 给 fork Agent 的 prompt 写清楚目标——不要写"基于之前的讨论修复 bug"，要写"修复 src/auth.ts 中 validateToken 函数的空指针问题，当 token 为 null 时返回 false"

---

## 10. 权限规则不生效

**现象：** 配了 `alwaysAllowRules` 但工具还是要确认。或者配了 `alwaysDenyRules` 但命令还是能执行。

**原因分析：** 配置来源优先级问题。7 层来源从低到高：enterprise → managed → user global → project local → dynamic → claudeai proxy → plugin。高层的规则覆盖低层。

**操作步骤：**
1. 确认规则写在了哪个配置文件。用 `claude config list` 查看当前生效的所有配置
2. 检查是否有高优先级来源的矛盾规则。企业级的 `alwaysDenyRules` 无法被项目级的 `alwaysAllowRules` 覆盖
3. 检查规则格式。`"Bash(rm *)"` 和 `"Bash(rm -rf *)"` 是不同的匹配模式——前者匹配 `rm` 开头的所有命令，后者只匹配 `rm -rf` 开头的
4. 检查 deny > ask > allow 的优先级。如果同一个操作同时匹配了 deny 和 allow，deny 生效
5. 修改后需要重启 Claude Code 才能生效——settings.json 的修改不是实时热加载的

---

## 11. 快捷键没反应

**现象：** 按了快捷键但什么都没发生。比如按 Enter 不提交消息，或 Ctrl+C 不中断。

**原因分析：** Claude Code 的快捷键按 Context 分层。同一个键在不同 Context 下行为完全不同。你可能在错误的 Context 层按了键。

**操作步骤：**
1. 确认当前在哪个 Context。Chat 层（输入框有光标）/ Settings 层（设置界面）/ Confirmation 层（确认框）/ Autocomplete 层（补全菜单）
2. 按 Escape 通常能回到 Chat 层——从 Settings、Autocomplete、Confirmation 都能回去
3. 如果是 Vim 模式导致的问题：按 Escape 确保在 NORMAL 模式，然后 `i` 进入 INSERT 模式再输入
4. 检查自定义 keybindings 是否冲突。`~/.claude/keybindings.json` 里的自定义键位会覆盖默认绑定
5. 终端本身可能吃了键位。比如 macOS Terminal 的 Ctrl+C 可能被终端拦截而不是传给 Claude Code。换用 iTerm2 或 Warp 通常能解决

---

## 12. Vim 模式输入异常

**现象：** 开了 Vim 模式后，打字的内容被当成命令。比如想输入 "delete" 但按 d 触发了删除操作。

**原因分析：** Vim 模式下默认是 NORMAL 模式——所有按键都是命令。必须先按 `i` 进入 INSERT 模式才能正常输入文字。这跟 Vim 编辑器的行为完全一致。

**操作步骤：**
1. 按 Escape 确保回到 NORMAL 模式（状态确认）
2. 按 `i` 进入 INSERT 模式，然后正常输入
3. 如果状态机卡住了（不确定当前在哪个模式），连按 3 次 Escape，保证回到 NORMAL，再按 `i`
4. 检查是否有自定义 keybindings 跟 Vim 键位冲突。比如你把 `i` 绑定到了其他命令，Vim 的 `i` 就失效了
5. 如果确实不需要 Vim 模式，在设置里关掉。Vim 模式适合 Vim 用户，如果你平时不用 Vim，这个模式只会增加困惑
