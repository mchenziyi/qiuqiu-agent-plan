# 🏀 球球学习日志 — V0 第 1 周：Agent 基础

> **阶段目标：** 跑通 LLM + Tool 循环，理解 Agent 的本质。
> **完成时间：** 第 1 天

---

## ✅ 完成内容

### Phase 0：名词扫盲

认识了 5 个核心概念：

| 概念 | 一句话理解 |
|------|-----------|
| **Agent** | 为了得到更准的答案，做一系列思考和动手操作的系统 |
| **Tool** | 函数。以前人决定调不调，现在模型自己决定 |
| **Memory** | 让 LLM 记住上下文，不瞎编 |
| **Planning** | 拆大任务到小粒度，越简单 LLM 越可靠 |
| **Skill** | 知识 + 范式，约束 Agent 不发散 |

**核心公式：** `Agent = LLM（大脑）+ Tool（手）+ Memory（记忆）+ Planning（拆解）`

---

### V0 第一件事：调通 LLM

用 Go 调用 DeepSeek API，确认链路可用。

```go
config := openai.DefaultConfig(apiKey)
config.BaseURL = "https://api.deepseek.com"
client := openai.NewClientWithConfig(config)

resp, err := client.CreateChatCompletion(ctx, openai.ChatCompletionRequest{
    Model:    "deepseek-chat",
    Messages: []openai.ChatCompletionMessage{
        {Role: "user", Content: "用一句话回答：什么是 Agent？"},
    },
})
fmt.Println(resp.Choices[0].Message.Content)
```

**输出：** `Agent（智能体）是指能够自主感知环境、做出决策并采取行动以实现特定目标的人工智能系统。`

---

### V0 第 1 周：Tool Calling + Agent Loop

#### 三个内置工具

| 工具 | 功能 | 参数 |
|------|------|------|
| `read_file` | 读取文件内容 | path（文件路径） |
| `write_file` | 写入内容到文件 | path、content |
| `run_shell` | 执行 shell 命令 | command（Windows cmd 语法） |

#### Agent 核心循环

```go
func (a *Agent) Run(ctx context.Context, userInput string) (string, error) {
    messages = append(messages, userInput)

    for i := 0; i < maxLoops; i++ {
        // ① 把消息 + 工具定义发给 LLM
        resp := a.client.CreateChatCompletion(ctx, req)

        // ② LLM 返回：没有 ToolCall → 最终答案，退出循环
        if len(msg.ToolCalls) == 0 {
            return msg.Content
        }

        // ③ LLM 返回：有 ToolCall → 执行工具
        for _, tc := range msg.ToolCalls {
            tool := a.tools[tc.Function.Name]
            result := tool.Execute(tc.Function.Arguments)
            // ④ 把工具结果加回消息历史
            messages = append(messages, toolResult{Content: result})
        }
        // ⑤ 继续循环，让 LLM 看到工具执行结果
    }
}
```

#### 运行测试

**输入：** `创建一个 test.txt 写入 Hello`

**执行过程（3 轮 LLM 对话）：**

```
第 1 轮：用户需求 + 工具列表 → LLM
        LLM → write_file({"path":"test.txt","content":"Hello"})
        → 代码执行 → 文件已写入（5 字节）

第 2 轮：上一轮对话 + write_file 结果 → LLM
        LLM → run_shell({"command":"type test.txt"})
        → 代码执行 → Hello

第 3 轮：上一轮对话 + type 结果 → LLM
        LLM → 最终答案 → 退出循环
```

**输出：** `文件已成功创建，内容为 Hello`

---

## 🧠 关键理解

### Agent 的本质

> **Agent = 注册工具 → 循环 { 调 LLM → 解析 ToolCall → 执行 → 结果喂回去 }**

不是 LLM "主动"调用工具——是 LLM **请求**调工具，你的代码实际执行。

### 每次任务 = 多次 HTTP 调用

一次 Agent 任务背后可能是 2~5 次 API 调用，每次都在花 token。这是后续 Planning（减少无效调用）和 Cache（复用结果）的动机。

### 工具结果为什么要喂回给 LLM

LLM 没有记忆。它说了"调 write_file"，但不知道执行结果。必须把结果塞回去，它才能**基于结果决定下一步**。这就是 `Observation`（观察）——整个循环能运转的关键。

---

## 📁 项目结构

```
D:\AgentDemo\
├── go.mod
├── go.sum
└── main.go    ← Agent 核心代码（~180 行）
```

- 使用 `github.com/sashabaranov/go-openai` SDK
- 通过 `DEEPSEEK_API_KEY` 环境变量配置密钥
- DeepSeek BaseURL: `https://api.deepseek.com`

---

## 🔜 下一步

**V0 第 2 周：上下文管理** — 支持连续对话，让 Agent 能记住多轮交互的历史。

```text
Phase 0  名词扫盲 ✅
V0 第 1 周   Tool Calling + Agent Loop ✅
  ↓
V0 第 2 周   上下文管理 ← 下一个
V0 第 3 周   Tool 设计
V0 第 4 周   回顾与重构
```

---

## 📚 参考资源

- [runoob AI Agent 教程](https://www.runoob.com/ai-agent/ai-agent-tutorial.html)
- [runoob AI Agent 核心组件](https://www.runoob.com/ai-agent/ai-agent-core.html)
- [DeepSeek API 文档](https://api-docs.deepseek.com)
