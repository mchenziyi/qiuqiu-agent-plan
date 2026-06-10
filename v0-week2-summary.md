# 🏀 球球学习日志 — V0 第 2 周：上下文管理

> **阶段目标：** 让 Agent 支持连续对话，messages 持续累积，记住多轮交互历史。
> **完成时间：** 第 2 天

---

## ✅ 完成内容

### 核心改动：messages 持久化

**之前：** 每次 `Run()` 新建局部变量 `messages`，用完就丢，无法连续对话。

**现在：** `messages` 挂在 Agent 结构体上，持续累积，跨 `Run()` 保留。

```go
type Agent struct {
    client   *openai.Client
    model    string
    tools    map[string]Tool
    messages []openai.ChatCompletionMessage // ← 对话历史，持续累积
}

func (a *Agent) Run(ctx context.Context, userInput string) (string, error) {
    // 追加用户输入到已有历史中
    a.messages = append(a.messages, openai.ChatCompletionMessage{
        Role: "user", Content: userInput,
    })
    // ... 后续循环同 V0 第 1 周
}
```

### 新增：交互式对话循环

支持用户在终端连续输入，直到输入 `exit` 退出：

```go
for {
    fmt.Print("🧑 你: ")
    scanner.Scan()
    input := scanner.Text()
    if input == "exit" { break }

    answer := agent.Run(ctx, input)  // 每次带着全部历史
    fmt.Printf("🤖 球球: %s\n", answer)

    agent.trimMessages()  // 控制消息数量，省 token
}
```

### 新增：消息截断

防止对话过长导致 token 浪费：

```go
const maxMessages = 100

func (a *Agent) trimMessages() {
    if len(a.messages) > maxMessages {
        // 保留第一条 + 最近的 maxMessages-1 条
        a.messages = append(
            []openai.ChatCompletionMessage{a.messages[0]},
            a.messages[len(a.messages)-maxMessages+1:]...,
        )
    }
}
```

### 代码风格

全部关键逻辑加了**简洁的中文注释**，方便后续阅读和修改。

---

## 🔬 运行测试：连续 4 轮对话

### 第 1 轮

```
🧑 你: 帮我创建一个txt文件，里面写我是球球
🔧 调用工具: write_file({"content": "我是球球", "path": "我是球球.txt"})
📦 结果: 已写入文件 我是球球.txt（12 字节）
🤖 球球: 已经帮你创建好了 我是球球.txt 文件 ✅
```

### 第 2 轮（验证上下文）

```
🧑 你: 刚刚写的文件的内容是什么
🔧 调用工具: read_file({"path": "我是球球.txt"})
📦 结果: 我是球球
🤖 球球: 文件的内容是：我是球球 🐶
```

**→ LLM 记得"刚才"指的是什么 ✅**

### 第 3 轮（复杂任务）

```
🧑 你: 帮我再写一个文档，把刚刚的内容写进去，要用markdown格式。然后再展开对内容的介绍。
🔧 调用工具: write_file({...})
📦 结果: 已写入文件 我是球球介绍.md（1030 字节）
🤖 球球: 已经创建好了 我是球球介绍.md ...
```

**→ LLM 根据历史自己生成了完整的 markdown 内容 ✅**

### 第 4 轮（修正 + 完善）

```
🧑 你: 球球是我的博美，你完善一下刚刚的那个markdown吧
🔧 调用工具: write_file({...})
📦 结果: 已写入文件 我是球球介绍.md（1698 字节）
🤖 球球: 完善了文档，包含球球的外貌、性格、日常、名字由来...
```

**→ LLM 理解"完善"指的是更新已有的文件 ✅**

### 第 5 轮

```
🧑 你: exit
👋 再见！
```

---

## 🧠 关键理解

### 连续对话的本质

> **messages 累积 + 每次全量发送 = Agent 的"记忆力"。**

不是 LLM 有记忆，是你的代码帮它保留了完整的对话历史。

### Token 管理的本质

1M 上下文窗口（DeepSeek）意味着不太可能"溢出"，但每次请求都带着数万 token 意味着**成本在累积**。截断策略的目的是**省 token 省钱**，不是防溢出。

### Go 做 Agent 的优势

结构体作为有状态对象，messages 作为字段自然累积——比 Python 的全局变量和 JavaScript 的闭包更清晰。

---

## 📁 项目结构

```
D:\AgentDemo\
├── go.mod
├── go.sum
└── main.go    ← Agent 核心代码（约 200 行，含中文注释）
```

---

## ✅ V0 完成进度

| 周 | 内容 | 状态 |
|----|------|------|
| 第 1 周 | Tool Calling + Agent Loop | ✅ |
| 第 2 周 | 上下文管理（连续对话） | ✅ |
| 第 3 周 | Tool 设计 | ⏳ 下一步 |
| 第 4 周 | 回顾与重构 | ⏳ |

---

## 📚 参考资源

- [runoob AI Agent 核心组件](https://www.runoob.com/ai-agent/ai-agent-core.html) — 记忆、规划等概念
- [DeepSeek API 文档](https://api-docs.deepseek.com) — 1M 上下文窗口说明
