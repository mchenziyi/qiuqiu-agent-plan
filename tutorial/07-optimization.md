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

**SubAgent 子 Agent（第 13 项）：**

### 解决了什么问题

主 Agent 在执行复杂任务时，经常需要中途去做一些独立子任务。如果直接在主 Agent 的对话历史里问，会污染上下文。SubAgent 就是让主 Agent **新开一个草稿纸**，独立算完拿结果回来。

### 设计思路

```
主 Agent（messages 装着"加健康检查接口"的全部上下文）
  │
  ├── 发现需要查 Gin 路由写法
  │     ↓
  ├── SpawnSubAgent("Gin 怎么注册路由")
  │     ↓
  │     子 Agent 新建 messages 独立执行
  │     返回结果
  │
  └── 主 Agent 拿到结果继续，messages 不受影响
```

关键点：
1. **共享 LLM 客户端** — 不重新创建，省资源
2. **共享工具列表** — 子 Agent 也能读写文件、搜索代码
3. **独立对话历史** — 子 Agent 有自己的 messages，执行完销毁，不污染主 Agent
4. **独立 session** — 子 Agent 有独立的 Event Log，可以单独 replay

### 代码实现

```go
func (a *Agent) SpawnSubAgent(ctx context.Context, task string) (string, error) {
    sub := &Agent{
        client:   a.client,                    // 共享 LLM 客户端
        model:    a.model,
        allTools: a.allTools,                  // 共享工具
        messages: make([]openai.ChatCompletionMessage, 0), // 全新对话历史
        store:    a.store,
        session:  fmt.Sprintf("%s_sub_%d", a.session, time.Now().UnixNano()),
        Quiet:    a.Quiet,
    }
    return sub.Run(ctx, task)
}
```

子 Agent 就是**一个完整的新 Agent**，跟主 Agent 唯一的区别是共享了 LLM 客户端和工具列表。

### 使用场景

| 场景 | 主 Agent 任务 | 子 Agent 任务 |
|------|-------------|-------------|
| 查文档 | 加 JWT 认证 | 查 golang-jwt 的用法 |
| 写测试 | 修改用户模块 | 为当前修改写单元测试 |
| 调研 | 重构支付模块 | 分析当前代码结构 |
| 独立验证 | 合并 PR | 检查代码安全性 |

### 当前限制与未来方向

| 限制 | 说明 | 改进方向 |
|------|------|---------|
| 串行 | 主 Agent 等子 Agent 返回后再继续 | 支持并行派发多个 SubAgent |
| 无通信 | 子 Agent 不能调用主 Agent 的工具 | 支持子 Agent 回调主 Agent |
| 手动 | 目前通过 /subagent 手动触发 | 让主 Agent 在 Plan 执行中自动派发 |

用法：`/subagent 查一下 flag 包怎么解析参数`
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
