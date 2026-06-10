# 🏀 球球 V3 完结总结 — Runtime 篇

> **目标：让 Agent 的每一步操作都记录下来，崩溃后能从日志恢复状态。**
> **核心理解：Event Sourcing = 追加写 JSON + 读回来重放。**

---

## 🎯 V3 解决了什么问题

V0~V2 的 Agent 能做很多事了，但有一个致命问题：**跑着跑着进程崩溃了，状态全丢。** LLM 说过什么、调过什么工具、结果是什么——全部丢失。

V3 给球球加上了"黑匣子"：

```
每步操作 → 追加写入 JSON Lines 文件
进程崩溃 → 读文件重放 → 恢复现场
```

---

## 🧱 V3 核心改动

### Event 结构体：每步操作变成一条不可变记录

```go
type Event struct {
    ID        string    // 唯一 ID
    Type      string    // user / assistant / tool_call / tool_result / error
    Content   string    // 内容
    ToolName  string    // 工具名
    Timestamp time.Time // 发生时间
}
```

### EventStore：追加写入 JSON Lines 文件

```go
type EventStore struct {
    dir string  // .reasonix/sessions/
}

// 追加一条事件（不会改已有内容）
func (s *EventStore) Append(sessionID string, event Event) error {
    f, _ := os.OpenFile(path, os.O_APPEND|os.O_CREATE|os.O_WRONLY, 0644)
    f.WriteString(json + "\n")
}

// 读取某个 session 的全部事件
func (s *EventStore) Load(sessionID string) ([]Event, error) {
    // 按行读取 JSON，返回 []Event
}
```

### Agent 每步操作都记录

```go
func (a *Agent) Run(...) {
    a.recordEvent("user", userInput, "")          // 用户说话
    // ... 调 LLM ...
    a.recordEvent("assistant", msg.Content, "")    // LLM 回复
    // ... 调工具 ...
    a.recordEvent("tool_call", args, toolName)     // 工具调用
    a.recordEvent("tool_result", result, toolName) // 工具结果
}
```

### Replay：从 Event Log 重放对话

```go
func (a *Agent) Replay(sessionID string) (string, error) {
    events := a.store.Load(sessionID)
    // 格式化成可读的对话记录
}
```

---

## 🔬 运行测试

### 对话过程

```
你: 列个目录
你: 创建一个 test.txt 写 test
你: replay
```

### Event Log 实际文件内容（`.reasonix/sessions/session_xxx.jsonl`）

```json
{"type":"user",      "content":"请执行：列出指定目录内容"}
{"type":"assistant", "content":"请告诉我您要列出哪个目录的内容？"}
{"type":"user",      "content":"请执行：用write_file创建test.txt"}
{"type":"assistant", "content":"好的，我来创建 test.txt"}
{"type":"tool_call",  "tool_name":"write_file", "content":"{\"path\":\"test.txt\",\"content\":\"test\"}"}
{"type":"tool_result","tool_name":"write_file", "content":"文件 test.txt 已写入（4 字节）"}
{"type":"assistant", "content":"已完成！已创建 test.txt 文件"}
```

**每行一条，追加写入，不修改已有内容。**

### Replay 输出

```
📋 Session session_xxx 操作记录（共 9 条事件）：
1. 🧑 [user] 请执行：列出指定目录内容
2. 🤖 [assistant] 请告诉我您要列出哪个目录的内容？
3. 🧑 [user] 请执行：用write_file创建test.txt
4. 🤖 [assistant] 好的，我来创建...
5. 🔧 [tool_call] write_file: {"path":"test.txt","content":"test"}
6. 📦 [tool_result] write_file: 文件 test.txt 已写入（4 字节）
7. 🤖 [assistant] 已完成！
✅ 重放完成
```

---

## 🧠 V3 学到的核心知识

### ① Event Sourcing 的本质

> **不保存"当前状态"，保存"发生了什么"。状态 = 重放所有事件的结果。**

```text
普通日志：记录错误信息
Event Log：记录全部操作，能重放重建状态
```

### ② JSON Lines 就是逐行追加的 JSON

每行一条独立 JSON 对象，全部是追加写入，不修改已有内容。**够用、简单、可读。**

### ③ Event Log 不只是"容灾"

崩溃恢复是它的价值之一，但不是唯一的。Event Log 还可以：

- **调试** — 回放看看 LLM 当时是怎么想的
- **审计** — 查 Agent 做了哪些操作
- **统计** — 数数一轮对话调了几次 LLM、花了多少 token
- **学习** — 收集成功的 Agent 行为链路，未来可以生成 Skill（这是你之前提到的思路）

### ④ 跟 Reasonix 的做法一致

你之前问"主流的做法是什么"——Reasonix 用的也是 JSON Lines，存储方式跟球球一样：

```text
~/.reasonix/sessions/<session-id>.jsonl
```

---

## 📊 代码量

```
V3 总代码：约 550 行（含注释）
新增代码：约 60 行
  - Event / EventStore：40 行
  - recordEvent / Replay：20 行
新增工具：0 个（纯架构改动，不涉及工具）
当前工具总数：8 个
```

---

## 📁 项目结构

```
D:\AgentDemo\
├── go.mod / go.sum
├── main.go              ← Agent 核心代码
└── .reasonix/
    └── sessions/
        └── session_xxx.jsonl  ← Event Log（V3 新增）
```

---

## ✅ 整体进度

| 阶段 | 内容 | 状态 |
|------|------|------|
| V0 | Agent 基础（Loop + Tool + 上下文） | ✅ |
| V1 | Planning（拆步骤） | ✅ |
| V2 | Coding Agent（精确编辑 + Git） | ✅ |
| V3 | Runtime（Event Log + Replay） | ✅ |
| V4 | MCP 生态 | ⏳ 下一步 |
| V5 | Skill 体系 | ⏳ |

---

## 🔜 下一步：V4 — MCP 生态

V3 的球球有"黑匣子"了。但它的工具全是写死在代码里的——加一个新工具就要改代码、重新编译。

V4 让球球支持 **MCP 协议**：外部的 MCP Server 可以即插即用，不用改球球的代码。

---

## 📚 学习笔记

| 文件 | 内容 |
|------|------|
| `v0-complete.md` | Agent 基础篇 |
| `v1-complete.md` | Planning 篇 |
| `v2-complete.md` | Coding Agent 篇 |
| `v3-complete.md` | **本文 — Runtime 篇** |
| `README.md` | 完整 6 个月规划 |

---

## ✍️ 最后

**V3 你掌握的核心：**

> **Event Sourcing = 追加写 JSON + 读回来重放。**
>
> 不是数据库，不是复杂架构，就是一个 JSON Lines 文件。
>
> 这一步是你从"调 API"走向"设计系统"的分水岭——你已经能设计 Agent 的状态管理层了。
