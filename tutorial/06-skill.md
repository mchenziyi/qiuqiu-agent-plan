# 第 6 章：Skill 体系——Agent 人格切换

> **本章对应 V5，让 Agent 支持"人格切换"。**
> **代码 Tag：`v5`**

---

## 🎯 预期收获

学完这一章，你能：

- 理解 Skill 的本质（SystemPrompt + 工具白名单 + 规则）
- 定义自己的 Skill（架构师、审查、前端设计）
- 让 Skill 从外部 JSON 文件加载

---

## 🧠 核心思路

```
Skill = SystemPrompt + ToolWhitelist + Rules
不是插件（MCP 才是），是 Agent 的行为配置。

切换 Skill = 换提示词 + 限制工具。不改代码，不重启。
```

Skill 和 MCP 的区别：

| 维度 | Skill | MCP |
|------|-------|-----|
| 回答"怎么做事" | ✅ | ❌ |
| 回答"用什么做" | ❌ | ✅ |

---

## 🛠️ 动手实现

### Skill 结构体

```go
type Skill struct {
    Name          string   // 技能名
    SystemPrompt  string   // 专业提示词（行为核心）
    ToolWhitelist []string // 允许的工具（空 = 全部）
    Rules         []Rule   // 行为规则
}
```

### ApplySkill — 切换

```go
func (a *Agent) ApplySkill(s skill.Skill) {
    a.sysPrompt = s.SystemPrompt
    a.activeTools = s.ToolWhitelist  // 限制工具
    fmt.Printf("切换到 [%s] 模式\n", s.Name)
}
```

### 内置三个 Skill

| Skill | 提示词核心 | 可用工具 |
|-------|-----------|---------|
| architect | 分析系统、对比方案、输出 ADR | 只读 |
| code_review | 标严重级别、分析影响 | 读 + 编辑 |
| frontend_design | 组件拆分、a11y、响应式 | 读 + 写 |

### 外部加载

把 Skill 定义放在 `~/.qiuqiu/skills/xxx.json`，启动时自动加载：

```json
{
  "name": "debug_expert",
  "system_prompt": "你是一个 Debug 专家...",
  "tool_whitelist": ["read_file", "search_files", "run_powershell"]
}
```

---

## ✍️ 你自己试试

1. 设计一个你自己的 Skill（比如"翻译专家"或"SQL 优化师"），用 JSON 文件加载
2. 如果两个 Skill 有同名工具但行为不同，怎么设计？
3. 如果 Agent 同时激活了两个 Skill，会出现什么问题？

---

## ✅ 完成标准

- [ ] Skill 切换后行为明显不同
- [ ] 至少 3 个 Skill 可正常工作
- [ ] 外部 JSON 文件加载的 Skill 也可用
- [ ] 能向别人解释 Skill 和 MCP 的区别

**预计时间：** 3 天
