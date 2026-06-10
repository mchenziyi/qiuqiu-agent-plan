# 🏀 球球（Qiú Qiú）6 个月 Agent 产品开发路线

> **定位：** 有 Golang 开发经验，Agent 开发经验为 0，目标是做出自己的 Agent 产品。
> **原则：** 每个月产出一个可运行版本。**不要先读完全部再动手——边做边看，做完一个阶段再看下一个。**

---

## 🧭 使用方式（先读这一节）

这是一份**边做边解锁的任务地图**，不是一份阅读材料。

```text
正确用法：
① 读当前阶段的"目标 + 认知跃迁"（3 分钟）
② 看"第一件事"→ 打开 IDE 写代码 ✅
③ 卡住了？查"常见失败点"
④ 跑通后，用"回顾问题"复盘
⑤ 确认"完成标准"全部打勾 → 进入下一阶段

错误用法：
先读完全部 V0~V5 → 觉得自己懂了 → 什么都没写
```

---

## ✅ 适用人群

如果你符合**所有条件**，这条路适合你：

- [ ] 会编程（Go 或至少能读懂 Go）
- [ ] 用过 ChatGPT / Claude（知道 LLM 能做什么、不能做什么）
- [ ] **没写过 Agent**（没亲手跑通过 LLM → Tool → Observation 循环）
- [ ] 目标是做自己的 Agent 产品，不是调 API 拼工作流

**不符合？** 第一条不满足：先学 Go 基础再回来。后面几条不满足：不影响，直接开始。

---

## 📊 总路线图

```text
第 1 月  Agent 基础     → 球球 V0    LLM + Tool 循环跑通
第 2 月  Planning       → 球球 V1    拆解任务、规划执行
第 3 月  Coding Agent   → 球球 V2    自己改代码、提交 Git
第 4 月  Runtime        → 球球 V3    Event Log、Checkpoint、状态恢复
第 5 月  MCP 生态       → 球球 V4    外部插件、MCP 工具
第 6 月  Skill 体系     → 球球 V5    专业能力包切换
```

---

# 🗓️ 第 1 个月：Agent 基础（最重要）

## 🎯 一句话任务

让 Agent 能回答"帮我读取当前目录文件列表"——它会真的去读文件，而不是编一个答案。

## 🧠 认知跃迁

> 学完这一层，你**第一次真正理解：** LLM 不是聊天工具，而是**带工具调用能力的执行器**。Agent 的本质是 `LLM → Action → Observation → LLM` 的循环，不是一问一答。

## 🔄 触发条件：为什么需要这一阶段？

如果你用过 ChatGPT，你会发现它不会帮你读文件、不会执行命令、不会真正"做事"。V0 就是让 LLM **第一次拿到工具**——从这里开始，LLM 不再只是说话，而是能行动。

---

## ✋ 第一件事（打开 IDE 第一行代码）

**写一个 Go 程序，只做一件事：把用户输入发给 LLM，打印返回结果。**

这一步跟 Agent 无关，但它让你确认三件事：
1. ✅ API Key 配置正确
2. ✅ 网络连通
3. ✅ LLM SDK 调通

### 完整代码（直接复制可用）

```go
package main

import (
    "context"
    "fmt"
    "os"

    openai "github.com/sashabaranov/go-openai"
)

func main() {
    apiKey := os.Getenv("OPENAI_API_KEY") // 或 DEEPSEEK_API_KEY
    if apiKey == "" {
        fmt.Println("请设置环境变量 OPENAI_API_KEY")
        return
    }

    client := openai.NewClient(apiKey)

    // 如果用 DeepSeek，需要设置自定义 BaseURL：
    // client = openai.NewClientWithConfig(openai.DefaultConfig(apiKey).
    //     WithBaseURL("https://api.deepseek.com"))

    resp, err := client.CreateChatCompletion(
        context.Background(),
        openai.ChatCompletionRequest{
            Model: openai.GPT4oMini, // DeepSeek 用 "deepseek-chat"
            Messages: []openai.ChatCompletionMessage{
                {Role: "user", Content: "用一句话回答：什么是 Agent？"},
            },
        },
    )
    if err != nil {
        fmt.Println("调用失败:", err)
        return
    }

    fmt.Println(resp.Choices[0].Message.Content)
}
```

### 运行方式

```bash
# 初始化模块
go mod init qiuqiu

# 安装 SDK
go get github.com/sashabaranov/go-openai

# 设置 API Key
$env:OPENAI_API_KEY="sk-xxx"   # Windows PowerShell
export OPENAI_API_KEY="sk-xxx"  # Linux / Mac

# 运行
go run main.go
```

**如果报错：** 先看 API Key 是否设对了，再看网络能不能通。搞不定去问 AI："go-openai 调用报错 xxx，怎么解决？"

---

## 第 1 周：Tool Calling

搞懂 LLM 是怎么"知道有工具可以用"的——Function Calling。

### 学习资料

- [OpenAI Function Calling Guide](https://platform.openai.com/docs/guides/function-calling)（快速过一遍概念即可）

### 要理解的概念

```text
Tool       → 一个可以被 LLM 调用的函数
Schema     → 用 JSON Schema 告诉 LLM "这个工具有什么参数"
Arguments  → LLM 决定调用时传的参数（JSON 格式）
Result     → 工具执行后返回给 LLM 的内容
```

### 动手实验：实现三个工具

```go
// 定义工具结构
type Tool struct {
    Name        string
    Description string
    Parameters  any  // JSON Schema
    Execute     func(args string) string
}

// 实现三个工具
func ReadFile(path string) string {
    data, err := os.ReadFile(path)
    if err != nil { return fmt.Sprintf("错误: %v", err) }
    return string(data)
}

func WriteFile(path, content string) string {
    err := os.WriteFile(path, []byte(content), 0644)
    if err != nil { return fmt.Sprintf("错误: %v", err) }
    return "写入成功"
}

func RunShell(command string) string {
    cmd := exec.Command("sh", "-c", command)
    out, err := cmd.CombinedOutput()
    if err != nil { return fmt.Sprintf("错误: %v\n输出: %s", err, out) }
    return string(out)
}
```

### 🎯 最小验证

手动构造一个包含 Tool Call 的 LLM 响应（伪造的 JSON），验证你的调度器能正确解析参数并执行。这一步不要真的调 LLM，先确保调度逻辑正确。

---

## 第 2 周：Agent Loop

把上周的工具装进一个循环里。

### 完整最小 Agent（核心骨架）

```go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "os"

    openai "github.com/sashabaranov/go-openai"
)

// Tool 定义
type Tool struct {
    Name        string
    Description string
    Parameters  any
    Execute     func(string) string
}

// Agent 结构
type Agent struct {
    client  *openai.Client
    model   string
    tools   map[string]Tool
    messages []openai.ChatCompletionMessage
}

func NewAgent(apiKey, model string) *Agent {
    return &Agent{
        client:  openai.NewClient(apiKey),
        model:   model,
        tools:   make(map[string]Tool),
        messages: make([]openai.ChatCompletionMessage, 0),
    }
}

func (a *Agent) RegisterTool(t Tool) {
    a.tools[t.Name] = t
}

// 核心循环
func (a *Agent) Run(ctx context.Context, userInput string) (string, error) {
    a.messages = append(a.messages, openai.ChatCompletionMessage{
        Role: "user", Content: userInput,
    })

    maxLoops := 10
    for i := 0; i < maxLoops; i++ {
        resp, err := a.client.CreateChatCompletion(ctx,
            openai.ChatCompletionRequest{
                Model:    a.model,
                Messages: a.messages,
                Tools:    a.toolDefinitions(),  // 注册的工具定义
            },
        )
        if err != nil {
            return "", fmt.Errorf("LLM 调用失败: %w", err)
        }

        msg := resp.Choices[0].Message
        a.messages = append(a.messages, msg)

        // 没有 Tool Call → 最终答案
        if len(msg.ToolCalls) == 0 {
            return msg.Content, nil
        }

        // 有 Tool Call → 执行
        for _, tc := range msg.ToolCalls {
            fmt.Printf("🔧 调用工具: %s(%s)\n", tc.Function.Name, tc.Function.Arguments)
            tool, ok := a.tools[tc.Function.Name]
            if !ok {
                return "", fmt.Errorf("未知工具: %s", tc.Function.Name)
            }
            result := tool.Execute(tc.Function.Arguments)
            a.messages = append(a.messages, openai.ChatCompletionMessage{
                Role:       "tool",
                Content:    result,
                ToolCallID: tc.ID,
            })
        }
    }

    return "", fmt.Errorf("达到最大循环次数 %d，Agent 未完成", maxLoops)
}

// 把工具定义转成 LLM 可识别的格式
func (a *Agent) toolDefinitions() []openai.Tool {
    var tools []openai.Tool
    for _, t := range a.tools {
        schema, _ := json.Marshal(t.Parameters)
        var params map[string]any
        json.Unmarshal(schema, &params)
        tools = append(tools, openai.Tool{
            Type: "function",
            Function: &openai.FunctionDefinition{
                Name:        t.Name,
                Description: t.Description,
                Parameters:  params,
            },
        })
    }
    return tools
}
```

**不需要完全理解每一行——先跑起来。** 跑起来之后，你再回去读不理解的部分。

### 🎯 最小验证

让 Agent 跑一个需要调用 2 次工具的请求，比如"先读文件 config.json，再根据内容写一个 summary.txt"。验证循环能连续工作 2 轮以上。

---

## 第 3 周：上下文管理

### 要理解的概念

```text
Messages   → 发给 LLM 的整个对话历史
Context    → LLM 能"看到"的所有 token
Token      → 计价单位（1K token ≈ $0.001~0.01）
History    → 多轮对话累积 → Token 爆炸 💸
```

### 实现

把 agent.messages 的持久化加上：

```go
type Session struct {
    ID        string
    Messages  []openai.ChatCompletionMessage
    CreatedAt time.Time
}

// 保存到文件
func (s *Session) Save(path string) error {
    data, _ := json.Marshal(s)
    return os.WriteFile(path, data, 0644)
}

// 从文件恢复
func (s *Session) Load(path string) error {
    data, _ := os.ReadFile(path)
    return json.Unmarshal(data, s)
}
```

### 🎯 最小验证

连续问 3 个有关联的问题（比如先"读取 config.json"、再"这个文件配了什么端口"），验证 Agent 能记住上下文。

---

## 第 4 周：Tool 设计

不要继续堆工具。停下来思考：**好的 Tool 长什么样？**

| 坏工具 | 好工具 |
|--------|--------|
| `DoEverything(input)` | `ReadFile(path)` |
| `Process(data)` — 名字模糊 | `SearchFiles(pattern)` — 名字就是功能 |
| 返回原始数据（JSON 大块） | 返回 LLM 能直接理解的文本结果 |

### 好工具三原则

1. **单一职责** — 一个工具只做一件事
2. **自描述名** — 名字 + 参数让 LLM 一看就懂
3. **返回值友好** — LLM 拿到结果直接能用，不需要二次解析

### 🎯 最小验证

重新审视你前三周写的所有工具，对每一个工具回答：它符合三条原则吗？不符合就重构。

---

## ✅ V0 完成标准

**满足所有条件才能进入第 2 个月：**

- [ ] 能调通 LLM：第一件事的代码跑通了，API 链路没问题
- [ ] 能稳定调用工具：Agent 连续 5 次请求都正确触发工具
- [ ] 能处理 3 次以上连续 Loop（LLM → Tool → LLM → Tool → ...）
- [ ] 不会无限循环（maxLoops 安全阀生效）
- [ ] 工具调用失败时 Agent 不会崩溃（会说"这个工具出错了"而不是 panic）
- [ ] Agent 能记住多轮对话的上下文
- [ ] 每个工具都遵循"单一职责"原则
- [ ] 没有使用任何 Agent 框架（LangGraph / Vercel AI SDK）——自己写 Loop

### 代码量参考

V0 的代码应该在 **200~400 行**之间。如果你超过 800 行，说明过度设计了。如果不到 100 行，可能缺了核心逻辑。

---

## 🔥 V0 常见失败点

| 问题 | 现象 | 解法 |
|------|------|------|
| LLM 不调用工具 | LLM 直接编答案 | 检查 Tool Schema 格式；先用一个最简单的工具（如 `Ping()`）验证链路 |
| 无限循环 | Agent 不停调工具，永远不 finish | 加 `maxLoops` 安全阀（10~15）；检查 tool result 是否正常返回 |
| JSON 参数解析失败 | LLM 返回的参数 decode 报错 | 打印原始响应看看 LLM 到底返回了什么；schema 里加 `"strict": true` |
| Token 爆炸 | 跑几轮后消息太长，LLM 开始丢信息 | 每次调完打印 token 消耗；学会看 token 用量 |
| 工具命名太抽象 | LLM 不知道该用哪个工具 | `DoTask()` → 改成 `ReadFile()`、`SearchCode()` |
| API Key 配置 | 每次都要手动设环境变量 | 写一个 `.env` 文件，用 `godotenv` 加载 |

---

## 🔄 V0 回顾

**做完 V0 后，花 30 分钟回答这几个问题：**

1. **我解决了什么问题？** —— 把 LLM 和工具连接成了一个闭环
2. **我之前哪里想错了？** —— 很多人一开始以为 Agent 就是"问一句答一句"，现在你知道差在哪了
3. **Agent 和 Chatbot 的本质区别是什么？** —— 一个是带工具的循环，一个是一次问答
4. **如果现在重新写 V0，我会怎么改？** —— 记录你的想法，V3 重构时会用到

---

# 🗓️ 第 2 个月：Planning

## 🎯 一句话任务

让 Agent 能回答"给我的 Go 项目加一个健康检查接口"——它会先拆成 3~5 个步骤，然后一步一步执行完。

## 🧠 认知跃迁

> 学完这一层，你**第一次真正理解：** Agent 不是回答问题，而是**拆解任务的系统**。一个复杂任务（"加登录功能"）需要被拆成多个子步骤（分析路由 → 检查数据库 → 实现 JWT → 测试），Agent 才能可靠地完成。

## 🔄 触发条件：为什么需要这一阶段？

**你发现：** LLM 直接处理复杂任务时开始"胡言乱语"——它会跳过步骤、重复步骤、或者做完第一步就声称完成了。**不是 LLM 变蠢了，是一个 LLM 调用解决不了多步骤问题。** 你需要把任务拆碎，让 LLM 一次只处理一件事。

---

## ✋ 第一件事（打开 IDE 第一行代码）

**写一个函数，把"增加登录功能"这样的目标，解析成 3~5 条步骤列表。**

先不跑 Agent，只跑规划器：

```go
// 第一步：让 LLM 帮你把目标拆成步骤
func GeneratePlan(goal string) ([]string, error) {
    prompt := fmt.Sprintf(`
你是一个项目规划专家。
请把以下目标拆成 3-8 个具体步骤。
每个步骤一句话，按执行顺序排列。

目标：%s

只输出步骤，每行一个，不要序号。`, goal)

    resp, _ := client.CreateChatCompletion(ctx,
        openai.ChatCompletionRequest{
            Model: "gpt-4o-mini",
            Messages: []openai.ChatCompletionMessage{
                {Role: "user", Content: prompt},
            },
        })

    steps := strings.Split(resp.Choices[0].Message.Content, "\n")
    return steps, nil
}
```

**运行它：** 输入"给 Go 项目加健康检查接口"，看看 LLM 给出的步骤有没有遗漏。

---

## 第 1 周：理解 Task / Plan / Subtask

### 核心概念

```text
Goal  →  用户说"我要做什么"
Plan  →  Agent 拆成的步骤列表
Task  →  每一步的具体操作单元
```

### 示例：Goal → Plan

用户说"增加登录功能"，Agent 生成：

```text
1. 分析现有路由 → 读取 router.go
2. 检查数据库用户表 → 读取 schema.sql
3. 实现 JWT 生成和验证 → 写 auth.go
4. 添加登录接口 → 改 router.go
5. 编译测试 → go build && go test
```

### 🎯 最小验证

让 LLM 对 3 个不同任务生成步骤列表。不执行，只看步骤质量——是不是按顺序的？有没有漏关键步骤？

---

## 第 2 周：实现 Todo Manager

```go
type TaskStatus string
const (
    TaskPending TaskStatus = "pending"
    TaskRunning TaskStatus = "running"
    TaskDone    TaskStatus = "done"
    TaskFailed  TaskStatus = "failed"
)

type Task struct {
    ID          string
    Description string
    Status      TaskStatus
    Result      string   // 执行结果
}

type Plan struct {
    Goal  string
    Tasks []Task
}

// 添加任务
func (p *Plan) AddTask(desc string) {
    p.Tasks = append(p.Tasks, Task{
        ID:          fmt.Sprintf("t%d", len(p.Tasks)+1),
        Description: desc,
        Status:      TaskPending,
    })
}

// 获取下一个待执行的任务
func (p *Plan) NextTask() *Task {
    for i := range p.Tasks {
        if p.Tasks[i].Status == TaskPending {
            return &p.Tasks[i]
        }
    }
    return nil
}
```

### 🎯 最小验证

手动创建一个 Plan（3 个任务），标记一个完成，验证 `NextTask()` 返回下一个待办。

---

## 第 3 周：实现任务执行器

把 Todo Manager 和 Agent 结合起来。Agent 按 Plan 执行 Task，每执行完一个把结果传回 LLM，LLM 决定下一步。

```go
func (a *Agent) ExecutePlan(ctx context.Context, plan Plan) error {
    for {
        task := plan.NextTask()
        if task == nil {
            return nil // 所有任务完成
        }

        task.Status = TaskRunning
        result, err := a.Run(ctx, task.Description)
        if err != nil {
            task.Status = TaskFailed
            task.Result = err.Error()
            return err
        }

        task.Status = TaskDone
        task.Result = result
        fmt.Printf("✅ %s: %s\n", task.ID, task.Description)
    }
}
```

### 🎯 最小验证

一个 3 步的 Plan 能完整执行完，Agent 不跳过步骤、不重复步骤。

---

## 第 4 周：加入失败重试与动态重规划

Task 执行失败时：

```go
func (a *Agent) ExecutePlanWithRetry(ctx context.Context, plan Plan, maxRetries int) error {
    for {
        task := plan.NextTask()
        if task == nil {
            return nil
        }

        var lastErr error
        for attempt := 0; attempt <= maxRetries; attempt++ {
            task.Status = TaskRunning
            _, err := a.Run(ctx, task.Description)
            if err == nil {
                task.Status = TaskDone
                break
            }
            lastErr = err
            fmt.Printf("⚠️  第 %d 次重试: %s\n", attempt+1, task.Description)
        }

        if lastErr != nil {
            task.Status = TaskFailed
            fmt.Printf("❌ 任务失败，尝试重新规划: %s\n", task.Description)
            // 让 LLM 重新规划未完成的部分
            newPlan, _ := GeneratePlan(fmt.Sprintf(
                "原目标：%s\n失败步骤：%s\n请重新规划剩余步骤", plan.Goal, task.Description))
            // ... 替换 plan 中未完成的任务
        }
    }
}
```

### 🎯 最小验证

让一个 Task 故意失败（比如读不存在的文件），验证 Agent 会重试或重新规划。

---

## ✅ V1 完成标准

- [ ] Agent 能根据用户请求生成 3 步以上的 Plan
- [ ] Plan 能按顺序执行，每步执行完通知 LLM 结果
- [ ] 执行过程中不会跳过任何 Task
- [ ] Task 执行失败后会重试（至少 1 次）
- [ ] 重试仍然失败时，能触发重新规划
- [ ] 你能说出"静态规划"和"动态规划"的区别

---

## 🔥 V1 常见失败点

| 问题 | 现象 | 解法 |
|------|------|------|
| 生成的 Plan 质量差 | 步骤太粗或太细 | 给 LLM 一个 Plan 示例（few-shot 提示）；限制步骤 3~8 步 |
| 先执行再规划 | Agent 还没列计划就开始做事 | 在 Agent Loop 里先强制输出 Plan，确认后再执行 |
| 规划错了不会回头 | 第一步错了，后面全错 | 每次 Task 完成后让 LLM 验证上一步的结果 |
| 任务依赖没处理 | Task B 依赖 Task A 的输出，但顺序错了 | 在 Task 结构体加 `DependsOn []string` |

---

## 🔄 V1 回顾

- **Planning 的本质是什么？** —— 把复杂问题拆成 LLM 能单步处理的子问题
- **Claude Code 为什么要做 Planning？** —— 保证 Agent 不会在一件事上钻牛角尖
- **最大的坑是什么？** —— LLM 的规划可能错了，需要能动态调整

---

# 🗓️ 第 3 个月：Coding Agent

## 🎯 一句话任务

让 Agent 能回答"给这个函数加错误处理"——它会自己找到文件、精确修改、然后提交 Git。

## 🧠 认知跃迁

> 学完这一层，你**第一次真正理解：** 让 LLM 直接返回整段代码然后你手动替换——**这是最蠢的做法**。真正的 Coding Agent 核心不是"生成代码"，而是**定位 → 精确编辑 → 验证回滚**三件套。

## 🔄 触发条件：为什么需要这一阶段？

**你发现：** Agent 能"写代码"了，但它是整文件替换——改一行就要传整个文件，token 烧得飞快，而且经常把没让改的部分也改坏了。**你需要一种精确、安全、可追溯的方式让 Agent 改代码。**

---

## ✋ 第一件事（打开 IDE 第一行代码）

**写一个函数：给定文件路径、行号、新内容，修改文件指定行。**

```go
// 第一步：不用 LLM，先手动实现精确编辑
func ReplaceLine(filePath string, lineNumber int, newContent string) error {
    data, err := os.ReadFile(filePath)
    if err != nil {
        return err
    }
    lines := strings.Split(string(data), "\n")
    if lineNumber < 0 || lineNumber >= len(lines) {
        return fmt.Errorf("行号 %d 超出范围（共 %d 行）", lineNumber, len(lines))
    }
    lines[lineNumber] = newContent
    return os.WriteFile(filePath, []byte(strings.Join(lines, "\n")), 0644)
}
```

**自己测试：** 写一个测试文件，改它的一行，验证只改了那一行，其他行没变。

---

## 第 1 周：文件选择

不要整个项目都发给 LLM。你需要一个**文件选择策略**：

```go
// 最简单的方案：让 LLM 先列出需要的文件
func (a *Agent) SelectFiles(task string, allFiles []string) []string {
    prompt := fmt.Sprintf("任务：%s\n\n项目文件列表：%s\n\n只输出需要读取的文件路径，每行一个。",
        task, strings.Join(allFiles, "\n"))
    resp, _ := a.llm.Chat(ctx, prompt)
    // 解析 LLM 回复，提取文件路径
    return parseFileList(resp)
}
```

更高级的方案：基于 import graph 选择（搜索"go import graph"了解）。

### 🎯 最小验证

Agent 能根据"给 handlers/user.go 的 Login 函数加日志"定位到 `handlers/user.go`。

---

## 第 2 周：精确编辑

```go
type EditType int
const (
    EditReplace EditType = iota  // 替换文本块
    EditInsert                   // 在某行后插入
    EditDelete                   // 删除文本块
)

type Edit struct {
    Type    EditType
    File    string
    Anchor  string   // 定位文本
    Content string   // 新内容
}

func ApplyEdit(e Edit) error {
    data, _ := os.ReadFile(e.File)
    text := string(data)

    switch e.Type {
    case EditReplace:
        // 找到 Anchor 并替换成 Content
        if !strings.Contains(text, e.Anchor) {
            return fmt.Errorf("找不到锚定文本: %s", e.Anchor)
        }
        text = strings.Replace(text, e.Anchor, e.Content, 1)

    case EditInsert:
        // 找到 Anchor 所在行，在它后面插入 Content
        lines := strings.Split(text, "\n")
        for i, line := range lines {
            if strings.Contains(line, e.Anchor) {
                // 在后面插入
                lines = append(lines[:i+1], append([]string{e.Content}, lines[i+1:]...)...)
                break
            }
        }
        text = strings.Join(lines, "\n")

    case EditDelete:
        text = strings.Replace(text, e.Anchor, "", 1)
    }

    return os.WriteFile(e.File, []byte(text), 0644)
}
```

### 🎯 最小验证

Agent 在指定函数的开始插入 `log.Println("enter")`，不删不改其他代码。

---

## 第 3 周：Git 集成

```go
func GitStatus() (string, error) {
    out, err := exec.Command("git", "status", "--short").Output()
    return string(out), err
}

func GitCommit(message string) error {
    _, err := exec.Command("git", "commit", "-am", message).Output()
    return err
}

func GitRevert(file string) error {
    _, err := exec.Command("git", "checkout", "--", file).Output()
    return err
}
```

**安全策略：** 每次修改前自动 commit 或 stash，改完验证通过才保留。

```go
func (a *Agent) SafeEdit(edits []Edit) error {
    // 1. 修改前保存状态
    exec.Command("git", "stash", "push", "--include-untracked").Run()

    // 2. 应用修改
    for _, e := range edits {
        ApplyEdit(e)
    }

    // 3. 验证编译
    if err := exec.Command("go", "build").Run(); err != nil {
        GitRevertAll() // 编译失败 → 回滚
        return fmt.Errorf("编译失败，已回滚: %w", err)
    }

    // 4. 提交
    return GitCommit(a.lastIntent)
}
```

### 🎯 最小验证

Agent 修改代码后自动 commit，commit message 有意义。故意改坏一个文件，验证自动回滚。

---

## 第 4 周：修改验证 + 综合集成

改完代码后自动验证三步走：

```go
func (a *Agent) VerifyChanges() error {
    steps := []struct{
        name string
        cmd  *exec.Cmd
    }{
        {"编译检查", exec.Command("go", "build")},
        {"代码规范", exec.Command("go", "vet", "./...")},
        {"运行测试", exec.Command("go", "test", "./...")},
    }

    for _, step := range steps {
        if out, err := step.cmd.CombinedOutput(); err != nil {
            return fmt.Errorf("%s 失败: %s\n输出: %s", step.name, err, out)
        }
    }
    return nil
}
```

### 🎯 最小验证

Agent 改了一个会导致编译错误的文件 → 自动检测到 → 自动回滚到修改前状态。

---

## ✅ V2 完成标准

- [ ] Agent 能根据描述定位到要修改的文件
- [ ] 支持至少 2 种编辑模式（替换 / 插入）
- [ ] 编辑是精确的行级修改，不是整文件替换
- [ ] 每次修改自动 git commit
- [ ] 修改导致编译错误时能自动回滚
- [ ] 你能说出 Aider 的"Search/Replace"和"Whole File"编辑的区别

---

## 🔥 V2 常见失败点

| 问题 | 现象 | 解法 |
|------|------|------|
| 替换错了位置 | LLM 给的 Anchor 匹配不到或匹配到多个 | 要求 LLM 用前后 2-3 行作为上下文锚定 |
| 整文件覆盖 | 一个函数改动导致整个文件被 LLM 重写 | 优先 Search/Replace，禁止 Whole File |
| Git 冲突 | Agent 改的文件别人也在改 | 修改前先 `git pull --rebase` |
| 改完不验证 | Agent 改了代码就以为完成了 | 必须加编译/测试验证步骤 |
| 一次改太多文件 | Agent 同时改 5 个文件，出问题难定位 | 一次只改 1 个文件，commit 后再改下一个 |

---

## 🔄 V2 回顾

- **精确编辑为什么比整文件替换好？** —— token 更少、出错更少、可追溯
- **Git 在 Coding Agent 里扮演什么角色？** —— 安全网 + 审计日志 + 回滚
- **如果让球球支持多语言（Go + Python + JS），改文件逻辑需要变吗？** —— 编辑模式通用，但语法解析和编译验证不同

---

# 🗓️ 第 4 个月：Runtime

## 🎯 一句话任务

Agent 运行时崩溃了，重启后能从中断的地方继续——就像游戏存档一样。

## 🧠 认知跃迁

> 学完这一层，你**第一次真正理解：** 程序的"当前状态"只是历史上所有 Event 的累计结果。**保存 Event = 保存状态**。这不是理论，是一个你亲手写出来的系统。

## 🔄 触发条件：为什么需要这一阶段？

**你发现：** Agent 跑一半崩溃了，所有状态丢了，要从头再来——而且最气人的是，你**完全不知道上次跑的时候 LLM 说了什么、做了什么决策**。你需要一个系统，让 Agent 的每一步都**可追溯、可恢复**。

---

## ✋ 第一件事（打开 IDE 第一行代码）

**写一个事件结构体和一个保存函数：**

```go
// 第一步：定义一个 Event，保存到 JSON 文件
type Event struct {
    ID        string    `json:"id"`
    Type      string    `json:"type"`      // "user_message" | "tool_call" | "tool_result" | "assistant_message"
    Data      string    `json:"data"`      // 事件内容
    Timestamp time.Time `json:"timestamp"`
}

// 追加写入 JSON 文件（每行一个 Event）
func AppendEvent(path string, e Event) error {
    data, _ := json.Marshal(e)
    f, _ := os.OpenFile(path, os.O_APPEND|os.O_CREATE|os.O_WRONLY, 0644)
    defer f.Close()
    f.WriteString(string(data) + "\n")
    return nil
}
```

**手动测试：** 构造 3 个 Event，写入文件，读回来验证内容一致。

---

## 第 1-2 周：Event Sourcing

### 三个核心概念

```text
Event   → "发生了什么"——不可变的事实记录
Reducer → 根据 Event 更新状态的纯函数
Replay  → 从空状态重放所有 Event 重建最新状态
```

### 实现

```go
type EventType string
const (
    EventUserMessage    EventType = "user_message"
    EventToolCall       EventType = "tool_call"
    EventToolResult     EventType = "tool_result"
    EventAssistantMsg   EventType = "assistant_message"
    EventError          EventType = "error"
)

type Event struct {
    ID        string    `json:"id"`
    SessionID string    `json:"session_id"`
    Type      EventType `json:"type"`
    Data      string    `json:"data"`
    Timestamp time.Time `json:"timestamp"`
    PrevID    string    `json:"prev_id"` // 上一步 Event ID，形成链
}

// Event Store（简单版：JSON 文件）
type EventStore struct {
    path string
}

func (s *EventStore) Append(e Event) error {
    data, _ := json.Marshal(e)
    f, _ := os.OpenFile(s.path, os.O_APPEND|os.O_CREATE|os.O_WRONLY, 0644)
    defer f.Close()
    f.WriteString(string(data) + "\n")
    return nil
}

func (s *EventStore) Load(sessionID string) ([]Event, error) {
    data, _ := os.ReadFile(s.path)
    var events []Event
    for _, line := range strings.Split(string(data), "\n") {
        if line == "" { continue }
        var e Event
        json.Unmarshal([]byte(line), &e)
        if e.SessionID == sessionID {
            events = append(events, e)
        }
    }
    return events, nil
}

// Reducer：根据 Event 更新状态
type State struct {
    Messages []openai.ChatCompletionMessage
    ToolResults map[string]string
}

func Reduce(state State, event Event) State {
    switch event.Type {
    case EventUserMessage:
        state.Messages = append(state.Messages, openai.ChatCompletionMessage{
            Role: "user", Content: event.Data,
        })
    case EventToolResult:
        state.ToolResults[event.ID] = event.Data
    }
    return state
}

// Replay：从空状态重建
func Replay(events []Event) State {
    state := State{ToolResults: make(map[string]string)}
    for _, e := range events {
        state = Reduce(state, e)
    }
    return state
}
```

> **建议：** 动手前花 30 分钟看 Reasonix 的 Event 定义——它的分类是验证过的，可以避免设计出偏差太大的方案。

### 🎯 最小验证

一个完整的 Agent 对话结束后，能从 Event Log 重建全过程。每个 Event 的时间戳和内容都对得上。

---

## 第 3 周：Checkpoint

```go
type Checkpoint struct {
    ID        string    `json:"id"`
    SessionID string    `json:"session_id"`
    State     State     `json:"state"`
    EventID   string    `json:"event_id"`   // 最后一个 Event 的 ID
    CreatedAt time.Time `json:"created_at"`
}

func SaveCheckpoint(sessionID string, state State, lastEventID string) error {
    cp := Checkpoint{
        ID:        uuid.New().String(),
        SessionID: sessionID,
        State:     state,
        EventID:   lastEventID,
        CreatedAt: time.Now(),
    }
    data, _ := json.Marshal(cp)
    return os.WriteFile(fmt.Sprintf("checkpoint_%s.json", sessionID), data, 0644)
}

func LoadCheckpoint(sessionID string) (*State, error) {
    data, err := os.ReadFile(fmt.Sprintf("checkpoint_%s.json", sessionID))
    if err != nil { return nil, err }
    var cp Checkpoint
    json.Unmarshal(data, &cp)
    return &cp.State, nil
}
```

**策略：** 每 3 步或每次工具调用后保存。

### 🎯 最小验证

进程退出后重启，加载最后的 Checkpoint，Agent 能继续对话而不是从头开始。

---

## 第 4 周：Session Replay（时光机）

```go
func (s *Session) Replay() (State, error) {
    events, err := s.eventStore.Load(s.sessionID)
    if err != nil { return State{}, err }

    state := State{ToolResults: make(map[string]string)}
    for i, e := range events {
        fmt.Printf("[%d] %s: %s\n", i, e.Type, e.Data[:min(50, len(e.Data))])
        state = Reduce(state, e)
    }
    return state, nil
}
```

### 🎯 最小验证

从 Event Log 完整重放一次 Session，每一步的 LLM 回复和工具结果都能复现。

---

## ✅ V3 完成标准

- [ ] 每一步 Agent 操作都记录了 Event
- [ ] Event Store 能按 Session 查询
- [ ] 支持从 Event Log 重放整个 Session
- [ ] 进程崩溃后重启，能从最近 Checkpoint 恢复
- [ ] 你能解释 Event Sourcing 和"普通日志"的区别
- [ ] 你能画出球球的 Event 流转图

---

## 🔥 V3 常见失败点

| 问题 | 现象 | 解法 |
|------|------|------|
| Event 太多 | 一次对话产生上千个 Event，文件膨胀 | 定期做 Snapshot（合并历史）+ 只保留事件链的头部 |
| Replay 不一致 | 重放时某个 Event 的结果跟当初不同 | 工具调用的**完整结果**也要序列化到 Event 里 |
| Checkpoint 太大 | 保存了整个 State 和历史消息 | 只保存最近的 N 条消息 + Event Log 引用 |
| 过度设计 | 一开始就想要完美的 Event 存储 | 先用 JSON 文件，V3 之后再考虑 SQLite |
| 忘记记录 Error | Agent 出错了没有 Event 记录 | Error 也是一种 Event，必须记录 |

---

## 🔄 V3 回顾

- **Event Sourcing 在 Agent 系统里的价值？** —— 可追溯、可重放、可审计
- **为什么不每次重新调 LLM 来恢复状态？** —— 重新调结果可能不一样（LLM 非确定性），而且浪费 token
- **如何回答：Agent 出问题了，怎么回到上一个正确的状态？** —— Checkpoint restore 或 Event Replay

---

# 🗓️ 第 5 个月：MCP 生态

## 🎯 一句话任务

让球球能调用"GitHub 创建 Issue"——不是写代码调 API，而是接入一个外部的 MCP Server。

## 🧠 认知跃迁

> 学完这一层，你**第一次真正理解：** 工具接入应该是一个**协议问题**，不是代码问题。你不需要为每个工具写适配器——只要对方实现了 MCP，你的 Agent 就能直接用。

## 🔄 触发条件：为什么需要这一阶段？

**你发现：** 球球的内置工具越来越多了（读文件、写文件、Shell、Git、搜索……），但每个新工具你都要**自己写代码实现**。如果球球想接入 GitHub、浏览器、数据库、Figma……每个都要写一套适配代码。**你需要一个标准协议，让任意工具即插即用。**

---

## ✋ 第一件事（打开 IDE 第一行代码）

**安装一个现成的 MCP Server，让球球连接它。**

```bash
# 安装 MCP Inspector（调试工具）
npx @modelcontextprotocol/inspector

# 安装 Filesystem MCP Server
npx @modelcontextprotocol/server-filesystem
```

**你的代码任务：** 用 Go MCP SDK 写一个最简 Client，连接到这个 Server 并调用一个工具。

```go
// 最简 MCP Client（示意）
package main

import (
    "fmt"
    "os/exec"
)

func main() {
    // 启动 MCP Server 进程（stdio 模式）
    cmd := exec.Command("npx", "-y", "@modelcontextprotocol/server-filesystem", ".")
    stdin, _ := cmd.StdinPipe()
    stdout, _ := cmd.StdoutPipe()
    cmd.Start()

    // 发送 JSON-RPC 请求
    request := `{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"read_file","arguments":{"path":"."}}}`
    stdin.Write([]byte(request + "\n"))

    // 读取响应
    buf := make([]byte, 4096)
    n, _ := stdout.Read(buf)
    fmt.Println(string(buf[:n]))

    cmd.Wait()
}
```

**不用跑通也行**——先理解 MCP 的通信模型（JSON-RPC over stdio）。第 1 周有的是时间调试。

---

## 第 1 周：接入现成的 MCP Server

### 学习资料

- [Model Context Protocol 规范](https://modelcontextprotocol.io/specification) — 读一下"Overview"就够了

### 动手

用 Go MCP SDK 连接官方的 MCP Server：

```go
import mcp "github.com/metoro-io/mcp-golang"

// 连接 Filesystem MCP Server
client, _ := mcp.NewClient("npx", "-y", "@modelcontextprotocol/server-filesystem", ".")
result, _ := client.CallTool("read_file", map[string]any{"path": "README.md"})
fmt.Println(result)
```

### 🎯 最小验证

球球通过 MCP 调用 Filesystem Server 的 `read_file` 工具。不经过你手写文件读取代码。

---

## 第 2 周：自己写一个 MCP Server

```go
// 最简 MCP Server：暴露一个 calculator 工具
package main

import (
    "fmt"
    "math"
    mcp "github.com/metoro-io/mcp-golang"
)

func main() {
    server := mcp.NewServer()

    server.RegisterTool("calculate", func(args struct {
        A float64 `json:"a"`
        B float64 `json:"b"`
        Op string  `json:"op"`
    }) (string, error) {
        var result float64
        switch args.Op {
        case "add": result = args.A + args.B
        case "sub": result = args.A - args.B
        case "mul": result = args.A * args.B
        case "div": result = args.A / args.B
        default: return "", fmt.Errorf("未知操作: %s", args.Op)
        }
        return fmt.Sprintf("%f", result), nil
    })

    server.ServeStdio() // 通过 stdio 通信
}
```

### 🎯 最小验证

启动你的 MCP Server，用 MCP Inspector 连上它，调用 `calculate` 工具。

---

## 第 3 周：写一个有用的 MCP Server

选一个对你有实际价值的：

```text
- Browser MCP：截图网页、执行 JS、获取 DOM
- Database MCP：查询 SQLite
- Docker MCP：管理容器
```

### 🎯 最小验证

球球通过你的 MCP Server 完成一个实际任务（比如"截图当前网页"）。

---

## 第 4 周：实现 Plugin Loader

```go
type MCPPlugin struct {
    Name   string
    Server *mcp.Client
    Tools  []Tool
}

type PluginConfig struct {
    Name    string
    Command string
    Args    []string
}

func (a *Agent) LoadPlugins(configs []PluginConfig) error {
    for _, cfg := range configs {
        client, err := mcp.NewClient(cfg.Command, cfg.Args...)
        if err != nil {
            return fmt.Errorf("加载插件 %s 失败: %w", cfg.Name, err)
        }

        // 获取该 Server 暴露的所有工具
        tools, _ := client.ListTools()
        for _, t := range tools {
            a.RegisterTool(Tool{
                Name:        fmt.Sprintf("%s_%s", cfg.Name, t.Name),
                Description: t.Description,
                Execute:     func(args string) string {
                    result, _ := client.CallTool(t.Name, args)
                    return result
                },
            })
        }
    }
    return nil
}
```

### 🎯 最小验证

在配置文件里新增一个 MCP Server，重启球球后自动出现新的工具，不需要改代码。

---

## ✅ V4 完成标准

- [ ] 球球能连接至少 1 个现成的 MCP Server 并使用它的工具
- [ ] 你亲手写了一个 MCP Server（哪怕只有 1 个工具）
- [ ] Plugin Loader 支持在配置里注册新的 MCP Server
- [ ] 不同 MCP Server 的同名工具不会冲突（命名空间隔离）
- [ ] 你能解释 MCP 和"直接调 REST API"的区别

---

## 🔥 V4 常见失败点

| 问题 | 现象 | 解法 |
|------|------|------|
| 协议细节卡住 | JSON-RPC 格式不匹配 | 先用 `mcp-inspector` 调试 Server，再集成 Client |
| 工具冲突 | 两个 Server 暴露同名工具 | 加前缀命名空间：`github_create_issue` vs `filesystem_read_file` |
| 安全风险 | MCP Server 可以执行任意操作 | 给插件声明需要的权限（只读/读写）；运行在沙箱里 |
| 进程管理 | MCP Server 进程挂了球球不知道 | 加心跳检测；Server 挂了自动重启 |

---

## 🔄 V4 回顾

- **MCP 解决了什么问题？** —— 统一工具的接入标准，N 个 Agent 不用写 N 套接入代码
- **MCP 和 Function Calling 是什么关系？** —— Function Calling 是 LLM 调用工具的**内部机制**，MCP 是工具**发现和调用的对外协议**
- **为什么不是 REST API？** —— MCP 支持双向通信（Server 主动通知）、资源发现、生命周期管理

---

# 🗓️ 第 6 个月：Skill 体系

## 🎯 一句话任务

在同一个 Agent 里，切换到"架构师模式"输出设计文档，切换到"代码审查模式"检查安全问题——不需要改一行代码。

## 🧠 认知跃迁

> 学完这一层，你**第一次真正理解：** Agent 的专业能力 = **提示词 + 工具权限 + 行为规则** 的三元组组合。Skill 不是插件，是 Agent 的"人格切换"。

## 🔄 触发条件：为什么需要这一阶段？

**你发现：** 球球什么都能做，但做什么都"一个风格"——让它审查代码，它也像写代码一样直接改；让它做架构设计，它直接开写代码而不是先画图。**你需要给球球"人格切换"的能力，不同任务用不同行为模式。**

---

## ✋ 第一件事（打开 IDE 第一行代码）

**定义一个 Skill 结构体，写一个 Apply 方法切换 Agent 的行为。**

```go
// 第一步：定义 Skill 结构
type Skill struct {
    Name        string   `json:"name"`
    Description string   `json:"description"`
    SystemPrompt string  `json:"system_prompt"`  // 核心：专业提示词
    Tools       []string `json:"tools"`           // 该 Skill 能用的工具
}

// 应用 Skill
func (a *Agent) ApplySkill(s Skill) {
    a.SystemPrompt = s.SystemPrompt
    // 只加载该 Skill 声明的工具
    for name := range a.tools {
        delete(a.tools, name)
    }
    for _, name := range s.Tools {
        if t, ok := a.allTools[name]; ok {
            a.tools[name] = t
        }
    }
}
```

**手动测试：** 创建两个 Skill（一个"温和模式"、一个"严格模式"），切换后问同一个问题，看回复风格是否不同。

---

## 第 1 周：设计 Skill 系统

```go
type Skill struct {
    Name        string    `json:"name"`
    Description string    `json:"description"`
    SystemPrompt string   `json:"system_prompt"`
    ToolWhitelist []string `json:"tool_whitelist"`  // 该 Skill 能用哪些工具
    Rules       []Rule    `json:"rules"`            // 行为规则
}

type Rule struct {
    Name        string `json:"name"`
    Description string `json:"description"`
}

type SkillRegistry struct {
    skills map[string]Skill
}

func (r *SkillRegistry) Register(s Skill) {
    r.skills[s.Name] = s
}

func (r *SkillRegistry) Get(name string) (Skill, bool) {
    s, ok := r.skills[name]
    return s, ok
}
```

### 🎯 最小验证

注册 2 个 Skill，切换后 Agent 的回复风格明显不同，可用工具集也随之变化。

---

## 第 2-4 周：内置 Skill 开发

每个 Skill 开发约 1 周。

### 架构师模式

```go
var ArchitectSkill = Skill{
    Name:        "architect",
    Description: "你是一个资深架构师，擅长分析和设计系统",
    SystemPrompt: `你是一个资深软件架构师。
在写任何代码之前，你必须：
1. 分析现有系统结构和代码组织
2. 提出至少 2 种方案并比较优劣
3. 输出架构决策记录（ADR）
4. 确认方案后再开始实施`,
    ToolWhitelist: []string{"read_file", "search_files", "list_directory"},
    Rules: []Rule{
        {Name: "必须有 ADR", Description: "每次架构决策必须记录原因和备选方案"},
        {Name: "方案比较", Description: "至少提出 2 种方案并列出优劣"},
    },
}
```

### 代码审查模式

```go
var CodeReviewSkill = Skill{
    Name:        "code_review",
    Description: "你是一个代码审查专家",
    SystemPrompt: `你是一个代码审查专家。
审查代码时你必须：
1. 列出每个问题的严重级别（Critical / Major / Minor）
2. 对每个问题给出修改建议和修改后的代码
3. 先分析影响范围，再给出修改方案`,
    ToolWhitelist: []string{"read_file", "search_files", "git_diff"},
    Rules: []Rule{
        {Name: "严重级别", Description: "每个问题必须标记 Critical/Major/Minor"},
        {Name: "影响分析", Description: "改之前必须分析影响范围"},
    },
}
```

### 前端设计模式

```go
var FrontendSkill = Skill{
    Name:        "frontend_design",
    Description: "你是一个前端架构师，熟悉组件设计和交互",
    SystemPrompt: `你是一个前端架构师。
设计 UI 时你必须：
1. 考虑组件拆分和复用
2. 考虑可访问性（a11y）
3. 考虑响应式布局
4. 输出组件树和状态管理方案`,
    ToolWhitelist: []string{"read_file", "search_files", "browser_screenshot"},
    Rules: []Rule{
        {Name: "可访问性", Description: "每个组件必须考虑 aria 标签和键盘导航"},
        {Name: "响应式", Description: "设计必须适配移动端和桌面端"},
    },
}
```

---

### 研究参考（到这个阶段再看）

到了第 6 个月，你已经亲手写过了全部 V0~V4，这时候才值得看：

- [Reasonix](https://github.com/esengine/DeepSeek-Reasonix) — 看它的 Skill 系统和 Runtime 设计
- [Claude Code](https://docs.anthropic.com/en/docs/claude-code/overview) — 看它的 Behavior 配置
- [Superpowers](https://github.com/esengine/superpowers) — 看别人怎么设计 Skill 集合

你会发现**大部分设计你都能理解，甚至能指出哪里跟你的球球不一样**。

### 🎯 最小验证

在同一个任务上（比如"分析这个项目的架构"），切换三个不同的 Skill，输出应该明显不同——架构师模式输出文档，审查模式列出问题，前端模式关注 UI 结构。

---

## ✅ V5 完成标准

- [ ] Skill 定义包含 SystemPrompt、ToolWhitelist、Rules
- [ ] Skill 可以热切换（不重启进程）
- [ ] 至少 3 个 Skill（架构师 / 代码审查 / 前端设计）正常工作
- [ ] 切换 Skill 后可用工具集跟着变化
- [ ] 你能设计一个新的 Skill 并注册进去
- [ ] 你能向别人解释 Skill 和 Plugin（MCP）的区别

---

## 🔥 V5 常见失败点

| 问题 | 现象 | 解法 |
|------|------|------|
| Skill 切换后差异不明显 | 架构师和程序员模式回答差不多 | 检查 SystemPrompt 是否足够**具体**和**有约束力** |
| 多个 Skill 同时激活 | 架构师模式突然开始审查代码 | 一个 Agent 实例一次只能激活一个 Skill |
| Skill 和 MCP 搞混 | 以为 Skill 是用来接入外部服务的 | **Skill = 行为配置（怎么做）**，**MCP = 工具来源（用什么做）** |
| ToolWhitelist 不生效 | 架构师模式还在用"写文件"工具 | 切换 Skill 时不仅要加工具，还要**移除不在白名单里的工具** |
| Skill 配置太复杂 | 定义 Skill 需要写大量 JSON | 用 Go 结构体硬编码，先不做外部配置文件 |

---

## 🔄 V5 回顾

- **Skill 和 MCP 的区别是什么？** —— Skill 是"怎么做事"，MCP 是"用什么做事"。两个正交维度
- **为什么把 Skill 放在最后一个月？** —— 因为 Skill 是对前面所有能力的**编排**，只有写完了 V0-V4，才真正理解要编排什么
- **如果现在让你重新设计球球的 Skill 系统，你会怎么改？** —— 记录你的想法，这是你自己的架构决策了

---

# 👋 总结

## 你的能力变化

```text
第 1 个月后：你理解 Agent 循环的本质，能自己写一个最小 Agent
第 2 个月后：你的 Agent 能拆解复杂任务，按计划执行
第 3 个月后：你的 Agent 能自己改代码、提交 Git、验证编译
第 4 个月后：你的 Agent 有"内存"了——Event Log + Checkpoint
第 5 个月后：你的 Agent 能接入任何 MCP 工具
第 6 个月后：你的 Agent 有"人格"了——不同 Skill 不同行为
```

## 最终能力模型

```text
Agent Runtime 设计     ★★★★★
Coding Agent 设计      ★★★★★
MCP 生态               ★★★★★
Go 系统开发            ★★★★★
Skill 系统设计         ★★★★
AI 产品设计            ★★★★
Prompt 工程            ★★★★
```

---

# 📚 附录

## 📅 每天投入建议

```text
工作日：1～2 小时
周末：  4～6 小时
```

没有时间压力，保持节奏即可。不想学的时候不用强迫——Agent 开发不是考试，少学两天不会"掉队"。

## 📖 学习资料优先级

### 优先级最高（按学习顺序）

1. **OpenAI Function Calling** — 第 1 个月看，Agent 的地基
2. **MCP** — 第 5 个月看，现代 Agent 的生态标准
3. **Aider** — 第 3 个月看设计思想，不用通读源码
4. **Reasonix** — 第 4-6 个月看 Event 定义和 Skill 系统

### 优先级一般

5. LangGraph — 了解概念即可，Go 路线不需要深入
6. OpenAI Agents SDK — 看设计思想，不用深入

### 暂时不用学

7. CrewAI / AutoGen / Dify / Coze — 目标是设计 Agent Runtime，不是搭工作流平台

## 🔧 推荐 Go 库

| 用途 | 库 |
|------|----|
| LLM SDK | `github.com/sashabaranov/go-openai` |
| MCP SDK | `github.com/metoro-io/mcp-golang` 或 `github.com/mark3labs/mcp-go` |
| 环境变量 | `github.com/joho/godotenv` |

---

# 🎯 最后的话

> **第一个月结束时，必须写出一个 1000 行以内的球球 V0。**
>
> 哪怕代码很丑都没关系。
>
> Agent 开发最关键的突破点不是学会概念，而是第一次亲手把 `LLM → Tool → Observation → LLM` 这个循环跑起来。**跑通之后，后面所有的 Planner、Event Sourcing、MCP、Skill，都会变得容易理解得多。**
>
> **别读完。去写。**

---

> **路线图维护：** 这份文档会随球球产品的演进而更新。如果你读到这里时时间已经过去了几个月，某些技术选型可能已经变了——但"先跑通 Loop，再迭代架构"这个顺序不会变。
