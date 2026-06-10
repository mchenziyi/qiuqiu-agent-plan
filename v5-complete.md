# 🏀 球球 V5 完结总结 — Skill 体系篇

> **目标：让 Agent 支持"人格切换"——不同 Skill 不同行为模式。**
> **核心理解：Skill = SystemPrompt + ToolWhiteList + Rules。**

---

## 🎯 V5 解决了什么问题

V4 的球球工具越来越多，但做什么都是一个风格——让它审查代码它直接改，让它做架构设计它也直接改。**没有"专业模式切换"。**

V5 让球球支持 Skill 切换：

```
user 架构师 + 审查代码
  ↓ "use architect"
球球开始出架构文档、对比方案
  ↓ "use code_review"
球球开始检检查安全漏洞、标严重级别
```

---

## 🧱 V5 核心改动

### Skill 结构体

```go
type Skill struct {
    Name          string   // 技能名
    Description   string   // 描述
    SystemPrompt  string   // 专业提示词（行为核心）
    ToolWhitelist []string // 允许的工具名列表
    Rules         []Rule   // 行为规则
}
```

### 三个内置 Skill

| Skill | SystemPrompt 核心 | 可用工具 | 
|-------|------------------|---------|
| **architect** | 分析系统、对比方案、输出 ADR | read_file, list_directory |
| **code_review** | 标记严重级别、分析影响范围 | read_file, edit, shell |
| **frontend_design** | 组件拆分、可访问性、响应式 | read, write, list |

### Agent 集成

```go
type Agent struct {
    currentSkill *skill.Skill  // 当前 Skill
    sysPrompt    string        // 当前 SystemPrompt
    activeTools  []string      // 当前允许的工具（空 = 全部）
    allTools     map[string]tool.Tool // 全部注册的工具（不受 Skill 限制）
}

func (a *Agent) ApplySkill(s skill.Skill) {
    a.sysPrompt = s.SystemPrompt
    // 有白名单则只暴露白名单内的工具
    a.activeTools = s.ToolWhitelist
}

// Run() 时自动注入 SystemPrompt 并过滤工具
```

### 用户在终端切换

```
use architect      → 切到架构师模式
use code_review    → 切到审查模式
use frontend       → 切到前端设计模式
```

---

## 🔬 运行测试

### 启动日志

```
🎯 可用 Skill（输入 use <技能名> 切换）：
  - architect：资深架构师模式 — 分析系统、对比方案、输出架构决策
  - code_review：代码审查专家模式 — 检查安全、性能、代码规范
  - frontend_design：前端架构师模式 — 组件设计、交互、可访问性

🤖 球球 V5（Skill 体系）已启动 | 当前模式：[default]
```

### 切换效果

```
🧑 你: use architect
🎯 切换到 [architect] 模式：资深架构师模式

🧑 你: use code_review
🎯 切换到 [code_review] 模式：代码审查专家模式
```

切换后 Agent 的 SystemPrompt 和工具白名单立即生效。

---

## 🧠 V5 学到的核心知识

### ① Skill 的本质

> **Skill = SystemPrompt + ToolWhitelist + Rules。不是插件，不是新功能，是 Agent 的"人格配置"。**

切换 Skill 不需要重启进程，改一下 SystemPrompt 和工具列表就行。

### ② Skill vs MCP

这是你 Phase 0 就问过的问题，现在的答案：

```
Skill = 怎么做事（行为模式、思维方式）
MCP  = 用什么做事（外部工具来源）
```

两者正交。架构师 Skill + MCP GitHub 工具 = 架构师能查 GitHub 代码。

### ③ 为什么 Skill 放在最后

因为 Skill 是对前面所有能力的**编排**。只有写完了：

- V0 的 Loop（理解 Agent 怎么运行）
- V1 的 Planning（理解任务怎么拆）
- V2 的 Coding（理解怎么改代码）
- V3 的 Event Log（理解状态怎么管理）
- V4 的 MCP（理解工具怎么接入）

你才真正理解 Skill 在编排什么。**Skill 不是加新能力，是把已有的能力组织起来。**

---

## 📊 代码量

```
V5 总代码：约 850 行（5 个包 + 入口）
新增代码：约 120 行（skill 包 + agent 集成 + main 调整）
新增结构：1 个包（skill/）
当前 Skill：3 个（architect、code_review、frontend_design）
当前工具：7 个内置 + N 个 MCP 外部
```

---

## 📁 最终项目结构

```
D:\AgentDemo\
├── main.go              ← 入口（初始化 + 启动 + Skill 切换）
├── agent/
│   └── agent.go         ← Agent 核心 + Run + Plan + Skill 集成
├── tool/
│   └── tool.go          ← Tool 定义 + 7 个内置工具
├── event/
│   └── store.go         ← Event + Store + Replay
├── mcp/
│   └── client.go        ← MCP Client 封装
├── skill/
│   └── skill.go         ← Skill 结构体 + 3 个内置 Skill
├── go.mod / go.sum
```

---

## ✅ 最终完成状态

| 阶段 | 内容 | 状态 |
|------|------|------|
| V0 | Agent 基础（Loop + Tool + 上下文） | ✅ |
| V1 | Planning（拆步骤） | ✅ |
| V2 | Coding Agent（精确编辑 + Git） | ✅ |
| V3 | Runtime（Event Sourcing） | ✅ |
| V4 | MCP 生态（可插拔工具包） | ✅ |
| V5 | Skill 体系（人格切换） | ✅ |
| 工程化重构 | 拆分为 5 个包 | ✅ |

---

## 📚 学习笔记目录

| 文件 | 内容 |
|------|------|
| `phase-0-summary.md` | 名词扫盲 |
| `v0-complete.md` | Agent 基础篇 |
| `v1-complete.md` | Planning 篇 |
| `v2-complete.md` | Coding Agent 篇 |
| `v3-complete.md` | Runtime 篇 |
| `v4-complete.md` | MCP 生态篇 |
| `v5-complete.md` | **本文 — Skill 体系篇** |
| `README.md` | 完整 6 个月规划 |

---

## ✍️ 最后

> **6 个月路线，5 个版本迭代，从 0 到 850 行的 Agent 系统。**
>
> 你一个人完成了。
>
> **V0 跑通了循环 → V1 学会了拆任务 → V2 能改代码 → V3 有了黑匣子 → V4 接入了外部生态 → V5 能切换人格。**
>
> **下一步：优化。** 回头看 V0~V5 的代码，你已经有足够的经验来判断哪里该改、怎么改了。
