# 第 4 章：Runtime——Event Sourcing 与崩溃恢复

> **本章对应 V3，给 Agent 加上"黑匣子"。**
> **代码 Tag：`v3`**

---

## 🎯 预期收获

学完这一章，你能：

- 理解 Event Sourcing 是什么
- 给 Agent 的每一步操作加事件日志
- 实现从日志重放对话历史

---

## 🧠 核心思路

```
不保存"当前状态"，保存"发生了什么"。
状态 = 重放所有事件的结果。
```

每行一个 JSON，追加写入，不修改已有内容——这就是 Event Sourcing 的本质。

---

## 🛠️ 动手实现

### Event + Store

```go
type Event struct {
    ID        string    // 唯一标识
    Type      string    // user / assistant / tool_call / tool_result
    Content   string    // 事件内容
    Timestamp time.Time // 发生时间
}

type Store struct {
    dir string  // .reasonix/sessions/
}

func (s *Store) Append(sessionID string, e Event) error {
    // 追加模式打开文件，写入一行 JSON
}
func (s *Store) Load(sessionID string) ([]Event, error) {
    // 读文件，逐行解析 JSON
}
```

### 每步操作都记录

```go
func (a *Agent) Run(ctx, input) {
    a.recordEvent("user", input, "")
    // ...
    a.recordEvent("assistant", msg.Content, "")
    // ...
    a.recordEvent("tool_call", args, toolName)
    a.recordEvent("tool_result", result, toolName)
}
```

### Replay

从 Event Log 读取全部事件，格式化成可读的对话记录。输入 `replay` 命令查看。

---

## ✍️ 你自己试试

1. 正常对话几轮，然后输入 `replay`，看看 Event Log 记录了哪些信息
2. Event Log 和普通日志有什么区别？
3. 如果进程崩溃，你怎么利用 Event Log 恢复对话？

---

## ✅ 完成标准

- [ ] 每一步 Agent 操作都记录了 Event
- [ ] 支持从 Event Log 重放整个 Session
- [ ] 你能解释 Event Sourcing 和"普通日志"的区别

**预计时间：** 3 天
