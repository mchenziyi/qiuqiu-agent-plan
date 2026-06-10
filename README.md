# 🏀 球球（Qiú Qiú）6 个月 Agent 产品开发路线

> **定位：** 有 Golang 开发经验，Agent 开发经验为 0，目标是做出自己的 Agent 产品。
> **原则：** 每个月产出一个可运行版本。**不要先读完全部再动手——边做边看，做完一个阶段再看下一个。**

---

## 🧭 使用方式（先读这一段）

这是一份**边做边解锁的地图 + 任务系统**，不是一份"阅读材料"。

| 正确做法 | 错误做法 |
|----------|---------|
| 读完 V0 → 动手写 → 跑通 → 回顾 → 再读 V1 | 一口气读完 V0~V5 → 觉得自己懂了 → 什么都没做 |
| 卡住了先看"常见失败点" | 卡住了自己闷头查 3 天 |
| 完成任务目标就进入下一阶段 | 追求完美，在一个阶段反复打磨 |

**每阶段循环：**

```text
① 读本阶段目标（5 分钟）
② 看最小任务 → 动手写代码（核心）
③ 卡住了？查"常见失败点"
④ 跑通后，用"回顾问题"复盘
⑤ 确认"进入条件"满足 → 进入下一阶段
```

---

## ✅ 适用人群

如果你符合**所有条件**，这条路适合你：

- [ ] 会编程（Go 或至少能读懂 Go）
- [ ] 用过 ChatGPT / Claude（知道 LLM 能做什么、不能做什么）
- [ ] **没写过 Agent**（没亲手跑通过 `LLM → Tool → Observation` 循环）
- [ ] 目标是做自己的 Agent 产品，而不是调 API 拼工作流

**第一步：** 从 V0 开始。

---

## 📊 总路线图

```text
第 1 月  Agent 基础     → 球球 V0    从零跑通 LLM→Tool→Observation 循环
第 2 月  Planning       → 球球 V1    Agent 能拆解任务、规划执行步骤
第 3 月  Coding Agent   → 球球 V2    Agent 能自己改代码、提交 Git
第 4 月  Runtime        → 球球 V3    Event Log、Checkpoint、状态恢复
第 5 月  MCP 生态       → 球球 V4    支持外部插件、接入 MCP 工具
第 6 月  Skill 体系     → 球球 V5    专业能力包切换（架构师/审查/前端设计）
```

---

# 🗓️ 第 1 个月：Agent 基础（最重要）

> **一句话任务：** 让 Agent 能回答"帮我读取当前目录文件列表"——并且它会真的去读文件，而不是直接编一个答案。

**目标：** 理解 Agent 到底是什么。不要看太多理论，直接写。

---

### 第 1 周：Tool Calling

搞懂现代 Agent 的基础——Function Calling。

**学习资料：**
- [OpenAI Function Calling Guide](https://platform.openai.com/docs/guides/function-calling)

**要理解的概念：**

```text
Tool       → 一个可以被 LLM 调用的函数
Schema     → 告诉 LLM "这个工具有什么参数"
Arguments  → LLM 决定调用时传的参数
Result     → 工具执行后返回给 LLM 的内容
```

**动手实验：** 实现三个工具

```go
read_file(path)
write_file(path, content)
run_shell(command)
```

**🎯 最小验证：** 手动构造一个包含 Tool Call 的响应，验证你的工具调度器能正确解析参数并执行。

---

### 第 2 周：Agent Loop

实现 Agent 的核心循环：

```go
for {
    resp := llm.Chat(messages)

    if resp.IsToolCall {
        result := executeTool(resp.ToolCall)
        messages = append(messages, result)
        continue
    }

    if resp.IsFinalAnswer {
        return resp.Content
    }
}
```

**重点理解** 为什么叫 Agent——因为：

```text
LLM → Tool 调用 → 执行结果 → LLM（再次思考）
```

形成了**感知-行动-观察**循环。这是 Agent 和 Chatbot 本质的区别：Chatbot 只说话，Agent 会做事。

**🎯 最小验证：** 让 Agent 跑一个需要调用 2 次工具的请求（例如"先读文件 A，再根据内容写文件 B"），验证循环能连续工作。

---

### 第 3 周：上下文管理

**要理解的概念：**

```text
Messages   → 发给 LLM 的整个对话历史
Context    → LLM 能"看到"的所有内容
Token      → LLM 的计价和上下文长度单位
History    → 多轮对话的累积
```

**实现：**

```go
type Session struct {
    ID        string
    Messages  []Message  // 完整的对话历史
    CreatedAt time.Time
}

type Conversation struct {
    Sessions []Session
    Current  *Session
}
```

**🎯 最小验证：** 连续问 3 个有关联的问题（比如先"读取 config.json"、再"这个文件配了什么端口"），验证 Agent 能记住上下文。

---

### 第 4 周：Tool 设计

不要继续堆工具。停下来思考：**好的 Tool 长什么样？**

| 坏工具 | 好工具 |
|--------|--------|
| `DoEverything(input string)` — 一个工具做所有事 | `ReadFile(path)` — 一个工具只做一件事 |
| `Process(data string)` — 名字模糊 | `SearchFiles(pattern)` — 名字就是功能 |
| 工具返回原始数据 | 工具返回 LLM 能直接用的结果 |

**好工具三原则：**

1. **单一职责** — 一个工具只做一件事
2. **自描述** — 名字 + 参数让 LLM 一看就懂
3. **返回值友好** — 返回的是 LLM 能直接理解的内容，而不是需要二次解析的原始数据

**🎯 最小验证：** 重新审视你上周写的工具，能说出每个工具的优化点。

---

### ✅ V0 进入条件

**满足所有条件才能进入第 2 个月：**

- [ ] 能稳定调用工具：Agent 连续 5 次请求都能正确触发工具
- [ ] 能处理 3 次以上的连续 Loop（LLM → Tool → LLM → Tool → ...）
- [ ] 工具调用失败时 Agent 不会崩溃（会说"这个工具出错了"而不是 panic）
- [ ] Agent 能记住多轮对话的上下文
- [ ] 你的工具遵循"单一职责"原则
- [ ] 没有使用任何 Agent 框架（LangGraph / Vercel AI SDK 等）——自己写 Loop

### 🔥 V0 常见失败点

| 问题 | 现象 | 解法 |
|------|------|------|
| LLM 不调用工具 | LLM 直接编答案而不是调用你定义的 Tool | 检查 Tool Schema 格式；先让 LLM 调用一个最简单的 tool（如 `ping`）来验证链路 |
| 无限循环 | Agent 不停调用工具，永远不会 finish | 加 `maxLoops` 安全阀（建议 10~15）；检查 LLM 是否收到了正确的 tool result |
| JSON 格式错误 | LLM 返回的参数解析失败 | 打印原始响应看 LLM 到底返回了什么；在 schema 里加 `strict: true` |
| Token 爆炸 | 跑几轮后消息太长，LLM 开始丢信息 | 记录每次 LLM 调用的 token 消耗；思考怎么压缩历史（下一阶段会解决） |
| 工具设计太粗 | 一个工具做了太多事，LLM 不知道怎么用 | 拆成更小的工具；每个工具只做一件事 |

### 🔄 V0 回顾

跑通后，花 30 分钟回答这几个问题：

1. **我解决了什么问题？** —— 能把 LLM 和工具连接起来形成一个闭环
2. **我之前哪里想错了？** —— 很多人一开始以为 Agent 就是"问一句答一句"
3. **Agent 和 Chatbot 的本质区别是什么？** —— Chatbot 是单次问答，Agent 是带工具的循环
4. **如果现在让你重新写 V0，你会怎么改？** —— 记录你的想法，V3 会用到

---

# 🗓️ 第 2 个月：Planning

> **一句话任务：** 让 Agent 能回答"给我的 Go 项目加一个健康检查接口"——它会先拆成 3-5 个步骤，然后一步一步执行完。

**目标：** 理解 Claude Code 最核心的能力——规划与执行。

---

### 第 1 周：理解 Task / Plan / Subtask

**核心概念：**

```text
Goal  →  用户说"我要做什么"
Plan  →  Agent 拆成的步骤列表
Task  →  每一步的具体操作
```

**示例：** 用户说"增加登录功能"，Agent 生成：

```text
1. 分析路由 → 读取现有路由文件
2. 分析数据库 → 检查用户表结构
3. 实现 JWT → 编写 token 生成和验证
4. 编译测试 → 确认能编译通过
```

**🎯 最小验证：** 让 LLM 对一个任务生成步骤列表，不执行，只看步骤质量。

---

### 第 2 周：实现 Todo Manager

```go
type Task struct {
    ID          string
    Description string
    Status      TaskStatus // Pending | Running | Done | Failed
    DependsOn   []string   // 依赖的其他 Task ID
}

type Plan struct {
    Goal  string
    Tasks []Task
}
```

**🎯 最小验证：** 能手动创建 Plan、添加 Task、标记完成。

---

### 第 3 周：实现任务执行器

让 Agent 按 Plan 的顺序执行 Task。执行完一个 Task，把结果传回 LLM 决定下一步。

**🎯 最小验证：** 一个 3 步的 Plan 能完整执行完，Agent 不会跳过或重复步骤。

---

### 第 4 周：加入失败重试

Task 执行失败时：

1. **简单重试** — 重试同一 Task 最多 3 次
2. **重新规划** — 如果重试仍失败，让 LLM 重新生成 Plan
3. **跳过** — 某些 Task 失败不影响整体时，可以标记为 Skipped

**🎯 最小验证：** 让一个 Task 故意失败（比如读一个不存在的文件），验证 Agent 会重试或重新规划。

---

### ✅ V1 进入条件

- [ ] Agent 能根据用户请求生成 3 步以上的 Plan
- [ ] Plan 能按顺序执行，每步执行完通知 LLM 结果
- [ ] Task 执行失败后 Agent 会重试（至少 1 次）
- [ ] 你能说出"静态规划"和"动态规划"的区别

### 🔥 V1 常见失败点

| 问题 | 现象 | 解法 |
|------|------|------|
| 生成的 Plan 质量差 | LLM 拆出来的步骤太粗或太细 | 给 LLM 一个 Plan 示例（few-shot）；限制步骤数 3~8 步 |
| 先执行再规划 | Agent 还没列完计划就开始改代码 | 在 Loop 里先强制输出 Plan，确认后再执行 |
| 规划错了不会回头 | 第一步就走错了，后面全错 | 检查 LLM 是不是"假装"看到了正确的中间结果 |

### 🔄 V1 回顾

- **Planning 的本质是什么？** —— 把一个复杂问题拆成 LLM 能单步处理的子问题
- **Claude Code 为什么要做 Planning？** —— 它保证 Agent 不会在一个问题上钻牛角尖
- **最大的坑是什么？** —— LLM 的规划可能错了，需要能动态调整

---

# 🗓️ 第 3 个月：Coding Agent

> **一句话任务：** 让 Agent 能回答"给这个函数加错误处理"——它会自己找到文件、精确修改、然后提交 Git。

**目标：** 进入 Claude Code 的领域——让 Agent 能写代码。

---

### 学习对象（看设计思想，不用通读源码）

- [Aider](https://aider.chat) — 开源 AI 编程助手
- [OpenCode](https://opencode.ai) — 基于 AI 的代码工具

**要回答的三个核心问题：**

```text
1. LLM 怎么知道应该改哪个文件？（文件选择）
2. 怎么精确地改而不是整文件替换？（精确编辑）
3. 改错了怎么恢复？（安全回滚）
```

---

### 第 1 周：文件选择

不用把整个项目传给 LLM（token 爆炸）。你需要一个**文件选择策略**：

- 基于文件名搜索（最简单的方案）
- 基于 import graph 选择（更精确，但更复杂）
- 让 LLM 自己决定要先读哪些文件

**🎯 最小验证：** Agent 能根据"给 handlers/user.go 的 Login 函数加日志"找到正确文件。

---

### 第 2 周：精确编辑

三种编辑模式：

```go
// 替换文本块 —— 找到一段文本，替换成新的
ReplaceBlock(file, oldText, newText)

// 在某行后插入 —— 找到锚点行，在后面插入新代码
InsertAfter(file, anchorLine, code)

// 删除文本块 —— 找到一段文本，删除它
DeleteBlock(file, text)
```

**核心原则：** 不要整文件替换。**精确编辑的 token 消耗更低、错误率也更低。**

**🎯 最小验证：** Agent 能在文件的指定函数里插入一行 `log.Println("enter")`，不影响其他代码。

---

### 第 3 周：Git 集成

```go
GitStatus()           // 当前变更
GitDiff(file)         // 文件的未提交变更
GitCommit(message)    // 提交变更
GitRevert(file)       // 恢复某个文件到上一个提交
```

**安全策略：** 每次修改前自动 Stash 或 Commit，改完验证通过才保留。

**🎯 最小验证：** Agent 修改代码后自动 commit，且 commit message 有意义。

---

### 第 4 周：修改验证

改完代码后自动验证：

1. **编译检查** — `go build` 或 `go vet`
2. **测试运行** — 运行相关测试
3. **失败回滚** — 如果验证失败，`git checkout` 恢复文件

**🎯 最小验证：** Agent 改了一个会导致编译错误的文件 → 自动检测到 → 自动回滚。

---

### ✅ V2 进入条件

- [ ] Agent 能根据描述定位到要修改的文件
- [ ] 支持至少 2 种编辑模式（替换 / 插入）
- [ ] 每次修改自动 git commit
- [ ] 修改导致编译错误时能自动回滚
- [ ] 你能说出 Aider 的"Search/Replace 编辑"和"Whole File 编辑"的区别

### 🔥 V2 常见失败点

| 问题 | 现象 | 解法 |
|------|------|------|
| 替换错了位置 | LLM 给的 oldText 匹配到了多个位置或匹配不到 | 要求 LLM 用前后几行代码做上下文锚定 |
| 整文件覆盖 | 一个函数改动导致整个文件被 LLM 重写 | 优先用 Search/Replace，禁止 Whole File 编辑 |
| Git 冲突 | Agent 改的文件别人也在改 | 修改前先 `git pull` |
| 改完不验证 | Agent 改了代码就以为完成了 | 必须加编译/测试验证步骤 |

### 🔄 V2 回顾

- **精确编辑为什么比整文件替换好？** —— 少传 token，少出错
- **Git 在 Coding Agent 里扮演什么角色？** —— 安全网 + 追溯
- **如果让球球支持多语言（Go + Python + JS），改文件逻辑需要变吗？** —— 编辑模式通用，但语法解析不同

---

# 🗓️ 第 4 个月：Runtime

> **一句话任务：** Agent 运行时崩溃了，重启后能从中断的地方继续——就像游戏存档一样。

**目标：** 开始学习架构——这是从"调 API"到"设计系统"的分水岭。

---

### 第 1-2 周：Event Sourcing

**三个核心概念：**

```text
Event   → "发生了什么"——一个不可变的事实记录
Reducer → 根据 Event 更新当前状态的纯函数
Replay  → 从空状态开始，重放所有 Event 重建最新状态
```

**实现事件类型：**

```go
type Event struct {
    ID        string
    SessionID string
    Type      EventType // UserMessage | ToolCall | ToolResult | AssistantMessage | Error
    Data      any       // 事件负载
    Timestamp time.Time
    PrevID    string    // 上一步事件 ID，形成链
}
```

**Event 存储方案（从简单开始）：**

```text
V1: JSON 文件（每行一个 Event）
V2: SQLite（更可靠）
V3: ???（一年后再说）
```

> **建议：** 动手前花 30 分钟看一下 Reasonix 的 Event 定义——它的分类是经过验证的，可以避免自己设计出偏差太大的方案。

**🎯 最小验证：** 一个完整的 Agent 对话结束后，能从 Event Log 重建全过程。

---

### 第 3 周：Checkpoint

```go
type Checkpoint struct {
    ID        string
    SessionID string
    State     []Message   // 当前 Agent 的完整状态
    CreatedAt time.Time
    EventID   string      // 对应的最后一个 Event
}

func SaveCheckpoint(sessionID string, state State) Checkpoint
func LoadCheckpoint(id string) (State, error)
```

**Checkpoint 策略：** 每 N 步或每次工具调用后自动保存。

**🎯 最小验证：** 进程退出后重启，能加载最后的 Checkpoint 恢复对话。

---

### 第 4 周：Session Replay

基于 Event Log 的"时光机"：

```go
func (s *Session) Replay() {
    events := s.LoadEvents()
    state := NewState()
    for _, e := range events {
        state = Reduce(state, e)
    }
    return state
}
```

**🎯 最小验证：** 从 Event Log 完整重放一次 Session，每一步的 LLM 回复和工具结果都能复现。

---

### ✅ V3 进入条件

- [ ] 每一步 Agent 操作都记录了 Event
- [ ] 支持从 Event Log 重放整个 Session
- [ ] 进程崩溃后重启，能从最近的 Checkpoint 恢复
- [ ] 你能解释 Event Sourcing 和"普通日志"的区别
- [ ] 你能画出球球的 Event 流转图

### 🔥 V3 常见失败点

| 问题 | 现象 | 解法 |
|------|------|------|
| Event 太多 | 一次 Agent 对话产生上千个 Event，存储膨胀 | 定期合并（Snapshot）+ 只保留最近 N 天的 Event |
| Replay 不一致 | 重放时某个 Event 的结果跟当初不一样 | 工具调用的结果也要序列化到 Event 里 |
| Checkpoint 太大 | 保存了整个 State，包含大量历史消息 | 只保存最近的 N 条消息 + Event Log 的引用 |
| 过度设计 | 一开始就想要完美的 Event 存储方案 | 先用 JSON 文件，V3 之后再考虑 SQLite |

### 🔄 V3 回顾

- **Event Sourcing 在 Agent 系统里的价值在哪里？** —— 可追溯、可重放、可审计
- **为什么 Reasonix 选择 Event Sourcing 而不是每次重新调 LLM？** —— 一个是设计理念，一个是性能取舍
- **你现在能否回答：如果 Agent 出问题了，怎么回到上一个正确的状态？** —— Checkpoint restore 或者 Event Replay

---

# 🗓️ 第 5 个月：MCP 生态

> **一句话任务：** 让球球能调用"GitHub 创建 Issue"——不是你自己写代码调 API，而是接入一个外部的 MCP Server。

**目标：** 理解现代 Agent 的协议生态——让球球能接入外部工具。

---

### 学习资料

- [Model Context Protocol](https://modelcontextprotocol.io) — 协议规范

---

### 第 1 周：接入现成的 MCP Server

- 安装官方的 [Filesystem MCP Server](https://github.com/modelcontextprotocol/servers)
- 让球球作为 MCP Client 连接它
- 调用它的工具（读文件、写文件、搜索）

**🎯 最小验证：** 球球通过 MCP 调用一个外部工具的"Hello World"。

---

### 第 2 周：自己写一个 MCP Server

用 Go 写一个最简的 MCP Server：

- 注册 1-2 个工具
- 用标准 MCP 协议通信

**Go MCP SDK 选择：**
- [`github.com/metoro-io/mcp-golang`](https://github.com/metoro-io/mcp-golang)
- [`github.com/mark3labs/mcp-go`](https://github.com/mark3labs/mcp-go)

**🎯 最小验证：** 启动你的 MCP Server，用 MCP Inspector 能连上并调用工具。

---

### 第 3 周：写一个有用的 MCP Server

选一个对你有实际价值的：

```text
- Browser MCP（截图网页、获取 DOM）
- Database MCP（查询 SQLite）
- Docker MCP（管理容器）
```

**🎯 最小验证：** 球球通过你的 MCP Server 完成一个实际任务。

---

### 第 4 周：实现 Plugin Loader

球球启动时自动加载已注册的 MCP Server：

```go
type MCPPlugin struct {
    Name    string
    Tools   []Tool
    Client  *MCPClient
}

func (a *Agent) LoadPlugins(configs []PluginConfig) {
    for _, cfg := range configs {
        plugin := ConnectMCP(cfg)
        for _, tool := range plugin.ListTools() {
            a.RegisterTool(tool)
        }
    }
}
```

**🎯 最小验证：** 在球球的配置里加一个新的 MCP Server，重启后自动出现新的工具。

---

### ✅ V4 进入条件

- [ ] 球球能连接至少 1 个现成的 MCP Server 并使用它的工具
- [ ] 你亲手写了一个 MCP Server（哪怕只有 1 个工具）
- [ ] Plugin Loader 支持动态注册/注销
- [ ] 你能解释 MCP 和"直接调 REST API"的区别

### 🔥 V4 常见失败点

| 问题 | 现象 | 解法 |
|------|------|------|
| 协议细节卡住 | MCP 的 JSON-RPC 格式不匹配 | 先用 `mcp-inspector` 调试，再集成 |
| 工具冲突 | 两个 MCP Server 注册了同名工具 | 加命名空间：`github_create_issue`, `filesystem_read_file` |
| 安全风险 | MCP Server 可能执行危险操作 | 给 MCP 插件加权限声明（需要什么权限） |

### 🔄 V4 回顾

- **MCP 解决了什么问题？** —— 统一了工具的接入标准，避免 N 个 Agent 写 N 套接入代码
- **MCP 和 Function Calling 是什么关系？** —— Function Calling 是 LLM 调用工具的机制，MCP 是工具发现和调用的协议
- **为什么不是 REST API？** —— MCP 是双向的（Server 可以主动通知 Client）、支持资源发现

---

# 🗓️ 第 6 个月：Skill 体系

> **一句话任务：** 在同一个 Agent 里，切换到"架构师模式"它会输出设计文档，切换到"代码审查模式"它会检查安全问题。

**目标：** 球球从"一个 Agent"变成"Agent 平台"。

---

### 什么是 Skill

Skill 本质上是一个**配置包**：

```go
type Skill struct {
    Name        string
    Description string
    SystemPrompt string    // 专业提示词
    ToolWhitelist []string // 该 Skill 能用的工具
    Rules       []Rule     // 行为规则
}
```

**Skill 不是插件——它是 Agent 的一套"行为配置"。** 切换 Skill 不需要重启进程。

---

### 第 1 周：设计 Skill 系统

实现 Skill 注册和切换机制：

```go
func (a *Agent) ApplySkill(skill Skill) {
    a.SystemPrompt = skill.SystemPrompt
    a.EnabledTools = skill.ToolWhitelist
    a.Rules = skill.Rules
}
```

**🎯 最小验证：** 切换一个 Skill 后，Agent 的行为（回复风格、可用工具）明显改变。

---

### 第 2-4 周：内置 Skill 开发

以下三个 Skill 是核心，每个开发约 1 周。

**架构师模式**

```text
系统提示：你是一个资深架构师，擅长分析和设计
工具：读文件、搜索、输出文档
规则：必须输出架构决策记录（ADR）；必须分析至少 2 种方案
```

**代码审查模式**

```text
系统提示：你是一个代码审查专家
工具：读文件、搜索、执行测试
规则：每个问题标记严重级别（Critical/Major/Minor）；改前必须分析影响范围
```

**前端设计模式**

```text
系统提示：你是一个前端架构师，熟悉组件设计
工具：读文件、搜索、截图（MCP）
规则：输出前必须考虑可访问性和响应式
```

---

### 研究参考（到这个阶段再看）

到了第 6 个月，你已经亲手写过 Agent Loop、Planning、Coding、Event Sourcing、MCP 了。这时候再看：

- [Reasonix](https://github.com/esengine/DeepSeek-Reasonix) — 看它的 Skill 系统和 Runtime 设计
- [Claude Code](https://docs.anthropic.com/en/docs/claude-code/overview) — 看它的 Behavior 配置
- [Superpowers](https://github.com/esengine/superpowers) — 看别人怎么设计 Skill

你会发现**大部分设计你都能理解，甚至能指出哪里跟你的球球不一样**。

**🎯 最小验证：** 在同一个任务上，切换不同 Skill 得到明显不同的输出。

---

### ✅ V5 进入条件

- [ ] Skill 可以热切换（不重启进程）
- [ ] 至少 3 个 Skill 可正常工作
- [ ] Skill 可以声明自己需要的工具，Agent 自动加载
- [ ] 你能设计一个新的 Skill 并注册进去
- [ ] 你能向别人解释 Skill 和 Plugin（MCP）的区别

### 🔥 V5 常见失败点

| 问题 | 现象 | 解法 |
|------|------|------|
| Skill 不够"专业" | 切换前后差异不明显 | 检查 SystemPrompt 是否足够具体；给 Skill 配专属工具 |
| Skill 同时激活 | 架构师模式突然开始审查代码 | 一个 Agent 一次只能激活一个 Skill |
| Skill 和 MCP 搞混 | 以为 Skill 是用来接入外部服务的 | **Skill = 行为配置**，**MCP = 工具来源**，两个维度 |

### 🔄 V5 回顾

- **Skill 和 MCP 的区别是什么？** —— Skill 是"怎么做事"，MCP 是"用什么做事"
- **为什么要把 Skill 放在最后一个月？** —— 因为 Skill 是对前面所有能力的编排，你只有写完了 V0-V4，才真正理解要编排什么
- **如果现在让你重新设计 Skill 系统，你会怎么改？** —— 记录你的想法

---

# 📚 附录

## 📅 每天投入建议

```text
工作日：1～2 小时
周末：  4～6 小时
```

没有时间压力，保持节奏即可。

---

## 📖 学习资料优先级

### 优先级最高

1. **OpenAI Tool Calling** — Agent 的地基（第 1 个月看）
2. **MCP** — 现代 Agent 的生态标准（第 5 个月看）
3. **Aider** — Coding Agent 的参考实现（第 3 个月看）
4. **Reasonix** — Agent Runtime 的架构参考（第 4~6 个月看）

### 优先级一般

5. LangGraph — 了解概念即可，Go 路线不需要深入
6. OpenAI Agents SDK — 看设计思想，不用深入

### 暂时不用学

7. CrewAI
8. AutoGen
9. Dify
10. Coze

**原因：** 目标是设计 Agent Runtime，不是搭工作流平台。

---

# 🎯 最后的话

> **第一个月结束时，必须写出一个 1000 行以内的球球 V0。**
>
> 哪怕代码很丑都没关系。
>
> 因为 Agent 开发最关键的突破点不是学会概念，而是第一次亲手把：
>
> ```text
> LLM → Tool → Observation → LLM
> ```
>
> 这个循环跑起来。跑通之后，后面所有的 Planner、Memory、Event Sourcing、MCP、Skill，都会变得容易理解得多。
>
> **别读完。去写。**

---

> **路线图维护：** 这份文档会随球球产品的演进而更新。如果你读到这里时时间已经过去了几个月，某些技术选型可能已经变了——但"先跑通 Loop，再迭代架构"这个顺序不会变。
