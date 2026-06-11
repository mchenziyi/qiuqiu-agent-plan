# 第 7 章：优化——把 Agent 打磨成产品

> **本章对应 QiuQiuPro 优化阶段，基于 V5 代码做体验优化。**
> **代码 Tag：`v6`（最新代码）**

---

## 🎯 预期收获

学完这一章，你能：

- 理解"代码写完了"和"产品做好了"的区别
- 从 10 个优化点中挑适合自己项目的去实现
- 知道怎么给 Agent 加安全、加搜索、加命令系统

---

## 🧠 核心思路

V5 的 Agent 结构完整，但功能毛边。优化就是把这些毛边磨掉。

不是每个优化都必做——根据你的使用场景选。

---

## 🛠️ 优化清单（选做）

| # | 优化点 | 改了什么 | 难度 |
|---|--------|---------|------|
| 1 | **search_files** | 按文件名和内容搜索，大项目里找代码 | ⭐ |
| 2 | **PowerShell** | Windows 下优先用 PowerShell，cmd 引号问题多 | ⭐ |
| 3 | **API Key 自动保存** | 首次输入后自动保存，后续免配置 | ⭐ |
| 4 | **Plan 自我审视** | 拆完步骤 LLM 自己检查一遍 | ⭐⭐ |
| 5 | **动态重规划** | 执行失败后自动重新规划 | ⭐⭐ |
| 6 | **MCP 可配置** | 从 JSON 文件读取 MCP Server 列表 | ⭐ |
| 7 | **Skill 外部加载** | `~/.qiuqiu/skills/*.json` 自动加载 | ⭐ |
| 8 | **Glob + Grep** | 拆分搜索工具，LLM 更容易选对 | ⭐ |
| 9 | **安全拦截** | 高危操作执行前弹窗确认 | ⭐⭐ |
| 11 | **Checkpoint 快照** | 定期保存 messages 快照，加速崩溃恢复 | ⭐⭐ |
| 12 | **`--quiet` 安静模式** | `-q` 参数减少中间日志，只显示关键信息 | ⭐ |

### 每个优化的核心改动示例

**Checkpoint 快照（第 11 项）：**

```go
// agent.go — 自动保存 Checkpoint
const checkpointInterval = 5 // 每 5 次工具调用保存一次

func (a *Agent) SaveCheckpoint() {
    data, _ := json.Marshal(a.messages)
    a.store.SaveCheckpoint(a.session, a.lastEventID, string(data))
}

// agent.go — 启动时从 Checkpoint 恢复
a.RestoreFromCheckpoint()
```

**安静模式（第 12 项）：**

```go
// agent.go — 开关控制
type Agent struct {
    Quiet bool  // true 时隐藏中间日志
}

func (a *Agent) debugf(format string, args ...interface{}) {
    if !a.Quiet {
        fmt.Printf(format, args...)
    }
}
```

**安全拦截（第 9 项）：**

```go
// agent.go — 定义高危工具名单
var highRiskTools = map[string]bool{
    "write_file": true,
    "run_shell":  true,
}

// run.go — 执行前弹窗确认
if IsHighRiskTool(tc.Function.Name) {
    fmt.Print("确认执行？[Y/n] ")
    fmt.Scanln(&confirm)
    if confirm == "n" { continue }
}
```

**斜杠命令（第 10 项）：**

```go
// command/registry.go
type Command struct {
    Name        string
    Description string
    Handler     func(args string) bool
}
```

---

## ✍️ 你自己试试

1. 从 10 个优化中挑 3 个你觉得最重要的，按你的场景排个优先级
2. 如果要加一个 `/format` 命令让 LLM 格式化代码，怎么写？
3. 如果用户想跳过安全确认（`--yes` 模式），怎么加这个参数？

---

## ✅ 完成标准

- [ ] 你选择了适合自己场景的优化点
- [ ] 你自己加了一个之前没有的命令/工具
- [ ] 你能说出"优化"和"重写"的区别

**预计时间：** 选做，1 周起步
