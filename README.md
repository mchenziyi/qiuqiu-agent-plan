# 球球（Qiú Qiú）6 个月 Agent 产品开发路线

> **定位：** 有 Golang 开发经验，Agent 开发经验为 0，目标是做出自己的 Agent 产品。
> **原则：** 每个月产出一个可运行版本，目标不是学知识，而是做产品。

---

## 总路线图

```text
第 1 月  Agent 基础     → 球球 V0
第 2 月  Planning       → 球球 V1
第 3 月  Coding Agent   → 球球 V2
第 4 月  Runtime        → 球球 V3
第 5 月  MCP 生态       → 球球 V4
第 6 月  Skill 体系     → 球球 V5
```

---

## 第 1 个月：Agent 基础（最重要）

**目标：** 理解 Agent 到底是什么。不要看太多理论，直接写。

### 第 1 周：Tool Calling

搞懂现代 Agent 的基础——Function Calling。

**学习资料：**
- [OpenAI Function Calling Guide](https://platform.openai.com/docs/guides/function-calling)

**要理解的概念：**

```text
Tool
Schema
Arguments
Tool Result
```

**动手实验：** 实现三个工具

```go
read_file(path)
write_file(path, content)
run_shell(command)
```

**成果：** 能跑通 `用户 → 模型 → Tool → 结果` 链路。

---

### 第 2 周：Agent Loop

实现 Agent 的核心循环：

```go
for {
    调模型

    如果是 ToolCall
        执行 Tool

    如果是 FinalAnswer
        break
}
```

**重点理解** 为什么叫 Agent——因为：

```text
LLM → Action → Observation → LLM
```

形成了循环。这就是 Agent 和 Chatbot 本质的区别。

**成果：** 球球 V0 诞生——能读文件、执行命令、回答问题。

---

### 第 3 周：上下文管理

**学习内容：**

```text
Messages
Context
Token
History
```

**实现：**

```go
Session
Conversation
```

**成果：** 支持连续聊天，理解"为什么 Agent 越来越贵"。

---

### 第 4 周：Tool 设计

不要继续堆工具。停下来思考：**好的 Tool 长什么样？**

```text
坏工具：DoEverything()
好工具：ReadFile()
好工具：SearchFiles()
好工具：RunTests()
```

**成果：** 初步具备 Tool 设计能力——知道怎么把功能拆成内聚、可描述的工具。

---

## 第 2 个月：Planning

**目标：** 理解 Claude Code 最核心的能力——规划与执行。

### 第 1 周：理解 Task / Plan / Subtask

**学习概念：**

```text
Goal → Plan → Tasks
```

**示例：** 用户说"增加登录功能"，Agent 生成：

```text
1. 分析路由
2. 分析数据库
3. 实现 JWT
4. 编译测试
```

---

### 第 2 周：实现 Todo Manager

```go
Task {
    ID
    Status
    Description
}
```

---

### 第 3 周：实现任务执行器

按顺序执行 Todo List 中的每个任务。

---

### 第 4 周：加入失败重试

任务执行失败后自动重试或重新规划。

**成果：** 球球 V1 具备**计划、执行、追踪**能力。

> **注意：** 真正的 Planning 不只是拆任务——还要支持执行中动态调整计划。如果第二步发现第一步错了，要能重新规划。

---

## 第 3 个月：Coding Agent

**目标：** 进入 Claude Code 的领域——让 Agent 能写代码。

### 学习对象

- [Aider](https://aider.chat) — 开源 AI 编程助手
- [OpenCode](https://opencode.ai) — 基于 AI 的代码工具

**重点研究方向：**

```text
文件选择——LLM 怎么知道该改哪个文件？
代码修改——怎么精确地改而不是整文件替换？
Git 集成——怎么让修改可追溯、可回滚？
```

---

### 实现 Read

```go
ReadFile()
ReadFolder()
```

---

### 实现 Edit（三种编辑模式）

```go
ReplaceBlock()  // 替换指定文本块
InsertAfter()   // 在某行后插入
DeleteBlock()   // 删除指定文本块
```

---

### 实现 Git 集成

```go
GitStatus()
GitCommit()
GitRevert()  // 改错了能回滚
```

**成果：** 球球 V2 能修改代码、运行测试、提交 Git。

---

## 第 4 个月：Runtime

**目标：** 开始学习架构——这是从"调 API"到"设计系统"的分水岭。

### Event Sourcing

**理解三个概念：**

```text
Event      → 发生了什么
Reducer    → 怎么更新状态
Replay     → 怎么重建历史
```

**实现事件类型：**

```go
UserMessageEvent
ToolCallEvent
ToolResultEvent
AssistantMessageEvent
```

---

### Checkpoint

```text
保存状态
恢复状态
```

**实现 Session Replay：** 基于 Event Log 重建任意时刻的 Agent 状态。

> **建议：** 动手前花 30 分钟看一下 Reasonix 的 Event 定义——它的分类是经过验证的，可以避免自己设计出偏差太大的方案。

**成果：** 球球 V3 支持 Session 管理、Event Replay、Checkpoint 恢复。

---

## 第 5 个月：MCP 生态

**目标：** 理解现代 Agent 生态——让球球能接入外部工具。

### 学习资料

- [Model Context Protocol](https://modelcontextprotocol.io)

---

### 第 1 周：接入现成的 MCP Server

```text
Filesystem MCP
GitHub MCP
```

---

### 第 2 周：自己写一个 MCP Server

```go
// Hello MCP Server — 最小的可运行示例
```

---

### 第 3 周：写一个有用的 MCP Server

```text
Browser MCP — 截图网页、执行 JS、获取 DOM
```

---

### 第 4 周：实现 Plugin Loader

让球球支持动态加载/卸载 MCP 插件，不需要重启进程。

**成果：** 球球 V4 支持动态工具加载。

---

## 第 6 个月：Skill 体系

**目标：** 球球从"一个 Agent"变成"Agent 平台"。

到了这个阶段，再看 Reasonix、Claude Code、Superpowers、CodeGraph 就会很容易理解——因为你已经亲手写过一遍类似的系统了。

### 实现 Role

```text
程序员
架构师
产品经理
测试工程师
```

### 实现 Skill

```text
frontend-design
architect
code-review
debug
```

### 体系结构

```go
Role → Skill → Tool
```

Skill 本质上是 `Prompt + Tool 白名单 + 行为规则` 的容器。

**成果：** 球球 V5 成为真正意义上的 Agent 产品。

---

## 每天投入建议

```text
工作日：1～2 小时
周末：  4～6 小时
```

没有时间压力，保持节奏即可。

---

## 学习资料优先级

### 优先级最高

1. **OpenAI Tool Calling** — Agent 的地基
2. **MCP** — 现代 Agent 的生态标准
3. **Aider** — Coding Agent 的参考实现
4. **Reasonix** — Agent Runtime 的架构参考

### 优先级一般

5. LangGraph — 了解概念即可，Go 路线不需要深入
6. OpenAI Agents SDK — 看设计思想，不用深入

### 暂时不用学

7. CrewAI
8. AutoGen
9. Dify
10. Coze

**原因：** 目标是设计 Agent Runtime，不是搭工作流平台。

---

## 最后的话

> **第一个月结束时，必须写出一个 1000 行以内的球球 V0。**
>
> 哪怕代码很丑都没关系。
>
> 因为 Agent 开发最关键的突破点不是学会概念，而是第一次亲手把：
>
> ```text
> LLM → Tool → Observation → LLM
> ```
>
> 这个循环跑起来。跑通之后，后面所有的 Planner、Memory、Reasonix、Claude Code 架构，都会变得容易理解得多。
