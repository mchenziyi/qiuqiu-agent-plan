# 🏀 球球 V4 完结总结 — MCP 生态篇

> **目标：让球球能接入外部工具，不需要改代码、不需要重新编译。**
> **核心理解：MCP = 可插拔的工具包。启动时连接外部 Server，工具自动注册。**

---

## 🎯 V4 解决了什么问题

V0~V3 的球球加工具只能改代码：

```go
// 加一个工具就要改 main.go，重新编译
agent.RegisterTool(NewReadFileTool())
```

V4 之后：

```go
// 启动时自动加载，不改代码，不重新编译
agent.LoadMCPPlugin("filesystem", "npx", "-y", "@modelcontextprotocol/server-filesystem", ".")
// 14 个外部工具自动注册完毕
```

---

## 🧱 V4 核心改动

### MCPClient 结构体

```go
type MCPClient struct {
    Name   string
    client *client.Client
    tools  []Tool
}
```

### ConnectMCPServer — 连接外部 MCP Server

```go
func ConnectMCPServer(name, command string, args ...string) (*MCPClient, error) {
    // ① 启动 MCP Server 子进程（stdio 通信）
    mcpClient, _ := client.NewStdioMCPClient(command, nil, args...)

    // ② 发送初始化请求（MCP 协议握手）
    initReq := mcp.InitializeRequest{}
    mcpClient.Initialize(ctx, initReq)

    // ③ 发现工具列表
    toolsResp, _ := mcpClient.ListTools(ctx, toolsReq)

    // ④ 把 MCP 工具包装成球球的 Tool，注册进去
    for _, mt := range toolsResp.Tools {
        t := Tool{Name: mt.Name, Description: mt.Description, ...}
    }
}
```

### LoadMCPPlugin — 一行代码加载

```go
func (a *Agent) LoadMCPPlugin(name, command string, args ...string) error {
    mc, err := ConnectMCPServer(name, command, args...)
    for _, t := range mc.tools {
        a.RegisterTool(t)
    }
}
```

---

## 🔬 运行测试

### 启动日志

```
🔌 正在加载 MCP 插件...
🔌 已加载 MCP Server [filesystem]：14 个工具
🤖 球球 V4（MCP 生态）已启动...
```

**一行代码，14 个外部工具自动可用。**

### 工具命名空间

MCP 工具自动加前缀防止冲突：

```
filesystem_read_file
filesystem_write_file
filesystem_list_directory
filesystem_search_files
...
```

和内置工具共存，互不干扰。

---

## 🧠 V4 学到的核心知识

### ① MCP = 可插拔的工具包

你说的完全对。MCP 的本质就是：

> **一个标准协议，让外部工具可以即插即用。不需要改 Agent 的代码。**

没有 MCP：加工具 = 改代码 + 重新编译
有 MCP：加工具 = 启动一个 Server 进程

### ② MCP 怎么工作的

MCP 没有魔法，就是 **JSON-RPC over stdio**：

```text
Agent (Client)                    MCP Server (子进程)
    │                                   │
    ├── 初始化握手 ─────────────────►   │
    │◄── 确认版本                     │
    │                                   │
    ├── "你有什么工具？" ───────────►   │
    │◄── "我有 read_file、write…"      │
    │                                   │
    ├── "调 read_file，参数..." ────►   │
    │◄── "返回结果..."                │
```

两边通过 stdin/stdout 传 JSON 消息。Server 可以用任何语言写。

### ③ 工具自动发现

`ListTools` 是 MCP 协议的一部分。Agent 启动时调用一次，就知道这个 Server 暴露了哪些工具、参数是什么。**不需要在 Agent 代码里硬编码。**

### ④ MCP 的价值不在技术，在生态

MCP 不是难的技术（JSON-RPC + stdio），它的价值是：

- **标准** — 所有 MCP Server 用同一套协议，Agent 不用为每个工具写适配器
- **生态** — 官方有 filesystem、github、puppeteer……社区有几千个 Server
- **语言无关** — Server 用 Python/Go/TypeScript 写的都可以，Agent 不关心

---

## 📊 代码量

```
V4 总代码：约 600 行
新增代码：约 80 行（MCP Client + Connect + LoadPlugin）
新增外部工具：14 个（来自 MCP filesystem Server）
内置工具总数：6 个
```

---

## ✅ 整体进度

| 阶段 | 内容 | 状态 |
|------|------|------|
| V0 | Agent 基础（Loop + Tool + 上下文） | ✅ |
| V1 | Planning（拆步骤） | ✅ |
| V2 | Coding Agent（精确编辑 + Git） | ✅ |
| V3 | Runtime（Event Sourcing） | ✅ |
| V4 | MCP 生态 | ✅ |
| V5 | Skill 体系 | ⏳ 下一步 |

---

## 🔜 下一步：V5 — Skill 体系

V5 是规划的最后一个月。让球球支持"人格切换"——切到架构师模式出设计文档，切到审查模式找 bug。

---

## 📚 学习笔记

| 文件 | 内容 |
|------|------|
| `v0-complete.md` | Agent 基础篇 |
| `v1-complete.md` | Planning 篇 |
| `v2-complete.md` | Coding Agent 篇 |
| `v3-complete.md` | Runtime 篇 |
| `v4-complete.md` | **本文 — MCP 生态篇** |
| `README.md` | 完整 6 个月规划 |

---

## ✍️ 最后

**V4 你掌握的核心：**

> **MCP = 可插拔的工具包。一个标准协议，让外部工具即插即用。**
>
> V0~V3 做的所有工具，都可以通过 MCP 的方式交给外部 Server 实现。Agent 本身只负责"编排"和"决策"，不负责"执行"——这才是现代 Agent 的架构。
