# 🏀 球球 Agent 系统架构总图

> 从 Phase 0 到 V5 的完整知识图谱，一张图看懂整个开发流程。

---

## 一、整体架构分层

```mermaid
graph TB
    subgraph 用户层
        CLI["🧑 终端输入<br/>exit / replay / use &lt;skill&gt;"]
    end

    subgraph 入口层
        MAIN["main.go<br/>初始化 + 启动 + 命令分发"]
    end

    subgraph 核心层
        AGENT["agent/agent.go<br/>Agent 结构体<br/>Run() / GeneratePlan() / ExecutePlan()"]
    end

    subgraph 能力层
        TOOL["tool/tool.go<br/>内置工具<br/>read_file / write_file / edit_file_block / git_commit / run_shell..."]
        MCP["mcp/client.go<br/>外部 MCP Server<br/>filesystem / github / ..."]
        SKILL["skill/skill.go<br/>Skill 切换<br/>architect / code_review / frontend_design"]
    end

    subgraph 基础设施层
        EVENT["event/store.go<br/>Event Log (.jsonl)<br/>Append / Load / Replay"]
        LLM["LLM 服务<br/>DeepSeek API"]
    end

    CLI -->|文本输入| MAIN
    MAIN -->|初始化| AGENT
    AGENT -->|注入 SystemPrompt| SKILL
    AGENT -->|过滤工具白名单| SKILL
    AGENT -->|注册内置工具| TOOL
    AGENT -->|注册 MCP 工具| MCP
    AGENT -->|记录每一步事件| EVENT
    AGENT -->|CreateChatCompletion| LLM

    MCP -->|stdio / JSON-RPC| FILESYSTEM["filesystem_mcp<br/>（独立进程）"]
```

---

## 二、Agent 核心循环（Run 函数）

```mermaid
flowchart TD
    START(["用户输入"]) --> A["记录 Event: user"]
    A --> B["构建 messages<br/>（如果有 Skill 则插入 SystemPrompt）"]
    B --> C["调 LLM<br/>CreateChatCompletion<br/>messages + tools"]
    C --> D{"LLM 返回了 ToolCall？"}
    
    D -->|否| E["记录 Event: assistant"]
    E --> F["返回最终答案"]
    F --> END(["结束"])

    D -->|是| G["遍历每个 ToolCall"]
    G --> H["记录 Event: tool_call"]
    H --> I{"这个 Tool 是否<br/>在当前 Skill 的白名单中？"}
    I -->|不在| J["标记为未知工具"]
    J --> C
    I -->|在| K["tool.Execute(args)"]
    K --> L["记录 Event: tool_result"]
    L --> M["把结果追加到 messages"]
    M --> N{"循环次数 < 15？"}
    N -->|是| C
    N -->|否| O["返回超时错误"]
    O --> END
```

---

## 三、Planning 流程

```mermaid
flowchart LR
    GOAL(["用户目标"]) --> PLAN["GeneratePlan<br/>调 LLM 把目标拆成步骤"]
    PLAN --> STEPS["步骤列表<br/>[1. 读取文件, 2. 分析, 3. 输出]"]
    STEPS --> EXEC["ExecutePlan<br/>遍历步骤"]
    EXEC --> STEP1["Step 1 → Run('请执行：读取文件')"]
    STEP1 --> CHECK1{"成功？"}
    CHECK1 -->|是| STEP2["Step 2 → Run('请执行：分析')"]
    CHECK1 -->|否| FAIL["❌ 执行失败"]
    STEP2 --> CHECK2{"成功？"}
    CHECK2 -->|是| STEP3["Step 3 → Run('请执行：输出')"]
    CHECK2 -->|否| FAIL
    STEP3 --> CHECK3{"成功？"}
    CHECK3 -->|是| DONE["🎉 全部完成"]
    CHECK3 -->|否| FAIL
```

---

## 四、事件存储与重放（Runtime）

```mermaid
flowchart TD
    subgraph 写入路径
        W_START(["Agent 执行操作"]) --> USER["recordEvent('user', ...)"]
        USER --> ASSISTANT["recordEvent('assistant', ...)"]
        ASSISTANT --> TOOL_CALL["recordEvent('tool_call', ...)"]
        TOOL_CALL --> TOOL_RESULT["recordEvent('tool_result', ...)"]
        TOOL_RESULT --> APPEND["Store.Append()<br/>追加写入 .reasonix/sessions/xxx.jsonl"]
    end

    subgraph 读取路径
        R_START(["输入 replay 命令"]) --> LOAD["Store.Load(sessionID)<br/>读取全部事件"]
        LOAD --> FORMAT["event.Replay()<br/>格式化成可读文本"]
        FORMAT --> OUTPUT["打印对话回顾"]
    end

    subgraph 文件格式
        FILE["session_xxx.jsonl<br/>{'type':'user','content':'...'}<br/>{'type':'assistant','content':'...'}<br/>{'type':'tool_call','tool_name':'write_file',...}<br/>{'type':'tool_result','tool_name':'write_file',...}"]
    end

    APPEND --> FILE
    LOAD --> FILE
```

---

## 五、MCP 集成流程

```mermaid
sequenceDiagram
    participant Main as main.go
    participant MCP as mcp/client.go
    participant Server as MCP Server (独立进程)
    participant Agent as agent/agent.go

    Main->>MCP: mcp.Connect("filesystem", "npx", "-y", "@modelcontextprotocol/server-filesystem", ".")
    MCP->>Server: 启动子进程（stdio 管道）
    Server-->>MCP: 进程启动
    MCP->>Server: Initialize（协议握手）
    Server-->>MCP: 确认版本
    MCP->>Server: ListTools
    Server-->>MCP: 返回工具列表（14 个）
    MCP->>MCP: 包装成 tool.Tool 格式（加前缀 filesystem_）
    MCP-->>Main: 返回 *MCPClient
    Main->>Agent: RegisterMCPTools("filesystem", tools)
    Note over Agent: 工具注册完成<br/>LLM 可以调用 filesystem_read_file、<br/>filesystem_write_file 等
    
    Agent->>Agent: Run()
    Agent->>MCP: tool.Execute() → CallTool
    MCP->>Server: CallTool("read_file", {path:"..."})
    Server-->>MCP: 返回文件内容
    MCP-->>Agent: 返回字符串结果
```

---

## 六、Skill 切换机制

```mermaid
flowchart TD
    START(["用户输入：use architect"]) --> PARSE["解析 skill 名"]
    PARSE --> MATCH{"匹配到内置 Skill？"}
    
    MATCH -->|是| APPLY["Agent.ApplySkill()"]
    APPLY --> SET_SP["设置 SystemPrompt<br/>'你是一个资深架构师...'"]
    APPLY --> SET_TW["设置 ToolWhitelist<br/>[read_file, list_directory]"]
    APPLY --> SET_SK["记录 currentSkill"]
    SET_SP --> DONE_SK["🎯 切换到 architect 模式"]
    SET_TW --> DONE_SK
    SET_SK --> DONE_SK

    MATCH -->|否| ERR["❌ 未找到 Skill"]
    
    DONE_SK --> WAIT["等待下轮用户输入"]
    WAIT --> RUN["Run() 时<br/>自动注入 SystemPrompt<br/>只暴露白名单内的工具"]
    RUN --> LLM_CALL["LLM 按架构师风格回答问题"]
```

---

## 七、完整调用链（一次典型对话）

```mermaid
flowchart TD
    U["🧑 用户：列出当前目录"] --> AG["Agent.GeneratePlan()"]
    AG --> PLAN["生成计划：[1步] 列出目录内容"]
    PLAN --> EX["Agent.ExecutePlan()"]

    EX --> R["Agent.Run('请执行：列出目录内容')"]
    
    R --> REC1["recordEvent('user', ...)"]
    REC1 --> MSG["构建 messages<br/>+ SystemPrompt（如果有 Skill）"]
    MSG --> LLM1["调 DeepSeek API"]
    LLM1 --> TCD{"返回 ToolCall？"}
    
    TCD -->|是 list_directory| EXEC["执行 tool.Execute()"]
    EXEC --> REC2["recordEvent('tool_result', ...)"]
    REC2 --> LLM2["把结果喂回给 LLM"]
    LLM2 --> TCD2{"还有 ToolCall？"}

    TCD2 -->|否| FINAL["LLM 输出最终答案"]
    FINAL --> REC3["recordEvent('assistant', ...)"]
    REC3 --> OUT["🤖 球球：当前目录的内容如下..."]
    
    OUT --> TRIM["trimMessages()<br/>超 100 条则截断"]
    TRIM --> LOG["Event 追加写入 .jsonl 文件"]
    LOG --> DONE_TALK["🎉 全部完成"]
```

---

## 八、包依赖关系

```mermaid
graph TD
    MAIN["main.go"] --> AGENT["agent"]
    MAIN --> MCP["mcp"]
    MAIN --> SKILL["skill"]
    MAIN --> EVENT["event"]
    MAIN --> TOOL["tool"]

    AGENT --> TOOL
    AGENT --> EVENT
    AGENT --> SKILL
    AGENT --> LLM_LIB["go-openai<br/>（DeepSeek API）"]

    MCP --> TOOL
    MCP --> MCP_LIB["mcp-go<br/>（MCP 协议）"]

    SKILL -->|无外部依赖| SELF["skill 包自带 3 个 Skill"]

    EVENT -->|无外部依赖| OS["os（文件读写）"]
```

---

## 九、知识图谱一览

```mermaid
mindmap
  root((球球 Agent))
    V0_基础
      Agent_Loop
      工具调用
      内存管理
      上下文
    V1_规划
      Plan_拆解
      步骤执行
      Plan_修复
    V2_编码
      精确编辑
      Git_集成
      编译验证
      自动回滚
    V3_运行时
      Event_Sourcing
      JSON_Lines
      Replay
    V4_MCP
      可插拔工具
      协议发现
      外部进程
      生态
    V5_Skill
      SystemPrompt
      工具白名单
      人格切换
      架构师_审查_前端
```

---

> 这份图谱覆盖了球球从 Phase 0 到 V5 的全部核心概念和流程。优化阶段（路线一）和阅读 Reasonix 源码（路线二）时，可以随时回到这张图定位自己当前在看哪一层。
