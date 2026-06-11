# 第 5 章：MCP 生态——外部工具即插即用

> **本章对应 V4，让 Agent 能接入外部 MCP Server。**
> **代码 Tag：`v4`**

---

## 🎯 预期收获

学完这一章，你能：

- 理解 MCP 协议的核心原理（JSON-RPC over stdio）
- 让 Agent 连接外部的 MCP Server
- 把外部工具注册到 Agent 中，像内置工具一样调用

---

## 🧠 核心思路

MCP = Model Context Protocol，一个标准协议，让工具可以即插即用。

```
Agent (Client)                    MCP Server (独立进程)
    │                                   │
    ├── "你有什么工具？" ───────────►   │
    │◄── "我有 read_file、write…"      │
    ├── "调 read_file，参数..." ────►   │
    │◄── "返回结果..."                │
```

没有 MCP：加工具 = 改代码 + 重新编译。
有 MCP：加工具 = 启动一个 Server 进程。

---

## 🛠️ 动手实现

### Connect — 连接外部 Server

```go
func Connect(name, command string, args ...string) (*MCPClient, error) {
    mcpClient, _ := client.NewStdioMCPClient(command, nil, args...)
    mcpClient.Initialize(ctx, initReq) // 握手
    return &MCPClient{Name: name, client: mcpClient}, nil
}
```

### DiscoverTools — 发现工具

```go
func (c *MCPClient) DiscoverTools() ([]tool.Tool, error) {
    toolsResp := c.client.ListTools(ctx, req)
    // 把 MCP 工具包装成球球的 Tool 格式
    for _, mt := range toolsResp.Tools {
        t := tool.Tool{
            Name:        name + "_" + mt.Name,  // 加前缀防冲突
            Description: mt.Description,
            Execute:     func(args string) string { /* 调 CallTool */ },
        }
    }
}
```

### 配置文件驱动

MCP Server 列表从 `~/.qiuqiu/mcp_servers.json` 读取：

```json
[
  {"name": "filesystem", "command": "npx", "args": ["-y", "@modelcontextprotocol/server-filesystem", "."]}
]
```

---

## ✍️ 你自己试试

1. 如果没有工具名前缀（`filesystem_read_file`），两个 Server 都暴露了同名工具会怎样？
2. 断开 MCP Server 的网络/进程，看 Agent 调用这个工具时会发生什么？
3. 试着自己写一个最简单的 MCP Server（calculator）并接入球球

---

## ✅ 完成标准

- [ ] 球球能连接至少 1 个现成的 MCP Server
- [ ] MCP 工具名前缀防止冲突
- [ ] 配置文件驱动，不改代码

**预计时间：** 3 天
