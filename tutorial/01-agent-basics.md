# 第 1 章：Agent 基础——手写第一个 Agent Loop

> **本章对应 V0，从零开始写一个能调 LLM、能调用工具的 Agent。**
> **代码 Tag：`v0`**

---

## 🎯 预期收获

学完这一章，你能：

- 用 Go 调通一个 LLM API（DeepSeek）
- 实现 Agent 核心循环：`调 LLM → 有 ToolCall 就执行 → 再调 LLM → 直到返回`
- 让 Agent 支持连续对话
- 理解工具设计的三个基本原则

---

## 🧠 核心思路

Agent 的本质就是一个 for 循环：

```go
for {
    调 LLM（带着历史消息 + 工具列表）
    if LLM 没要求调工具 {
        return LLM 的回答  // 任务完成
    }
    if LLM 要求调工具 {
        执行工具 → 把结果放回消息历史
        continue  // 继续循环
    }
}
```

没有魔法，没有复杂架构，就是一个循环。

---

## 🛠️ 动手实现

### 第 1 步：调通 LLM

用一个最简单的 Go 程序确认你的 API Key 是好的：

```go
package main

import (
    "context"
    "fmt"
    "os"
    openai "github.com/sashabaranov/go-openai"
)

func main() {
    apiKey := os.Getenv("DEEPSEEK_API_KEY")
    config := openai.DefaultConfig(apiKey)
    config.BaseURL = "https://api.deepseek.com"
    client := openai.NewClientWithConfig(config)

    resp, _ := client.CreateChatCompletion(context.Background(),
        openai.ChatCompletionRequest{
            Model: "deepseek-chat",
            Messages: []openai.ChatCompletionMessage{
                {Role: "user", Content: "用一句话回答：什么是 Agent？"},
            },
        })
    fmt.Println(resp.Choices[0].Message.Content)
}
```

**预期输出：** Agent 的定义。

### 第 2 步：添加 Tool 定义

```go
type Tool struct {
    Name        string      // 工具名，LLM 通过它调用
    Description string      // 描述，告诉 LLM 什么时候用
    Parameters  any         // 参数定义（JSON Schema）
    Execute     func(string) string // 执行函数，返回 LLM 友好的结果
}
```

实现三个工具：`read_file`、`write_file`、`run_shell`。

每个工具的结构都一样：Name + Description + Parameters + Execute。

### 第 3 步：实现 Agent Loop

```go
func (a *Agent) Run(ctx context.Context, userInput string) (string, error) {
    a.messages = append(a.messages, userInput)

    for i := 0; i < maxLoops; i++ {
        resp := a.client.CreateChatCompletion(ctx, req)

        if len(msg.ToolCalls) == 0 {
            return msg.Content  // 没有 ToolCall → 最终答案
        }

        for _, tc := range msg.ToolCalls {
            tool := a.tools[tc.Function.Name]
            result := tool.Execute(tc.Function.Arguments)
            // 把工具结果放回消息历史
            a.messages = append(a.messages, toolResult)
        }
    }
}
```

### 第 4 步：支持连续对话

把 `messages` 从局部变量移到 Agent 结构体上，变成字段：

```go
type Agent struct {
    messages []openai.ChatCompletionMessage  // 持续累积
}

func (a *Agent) Run(ctx, input) {
    a.messages = append(a.messages, input)  // 追加到已有历史
    // ...
}
```

这样每次 Run 都带着全部历史，LLM 能"记住"之前的对话。

### 第 5 步：工具设计三原则

1. **命名即文档** — `list_directory` 比 `run_shell("dir")` 好
2. **参数越少越好** — 1-2 个参数，LLM 填错概率最低
3. **返回值对 LLM 友好** — 返回"文件 xxx 不存在"，不是 "no such file"

---

## ✍️ 你自己试试

1. 把 `maxLoops` 设成 3，问一个需要 5 步工具调用的任务，看 Agent 会怎样
2. 给 `run_shell` 加一个参数 `timeout_seconds`，让 LLM 可以控制命令超时
3. 如果 LLM 连续 3 次调用同一个工具，你觉得说明什么问题？
4. 试着让 Agent 连续问 5 个相关的问题，观察 messages 是怎么增长的

---

## ✅ 完成标准

- [ ] 能调通 LLM API，拿到回复
- [ ] Agent Loop 能跑通至少 2 轮工具调用
- [ ] 不会无限循环（maxLoops 生效）
- [ ] 支持连续多轮对话
- [ ] 每个工具遵守三原则

**预计时间：** 1 周
