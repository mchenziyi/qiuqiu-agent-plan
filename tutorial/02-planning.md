# 第 2 章：Planning——让 Agent 学会拆任务

> **本章对应 V1，在 Agent Loop 基础上加规划能力。**
> **代码 Tag：`v1`**

---

## 🎯 预期收获

学完这一章，你能：

- 让 Agent 先把复杂目标拆成步骤，再按顺序执行
- 实现 Plan 自我审视（拆完步骤 LLM 自己检查一遍）
- 实现动态重规划（某步失败后自动重新规划剩余步骤）

---

## 🧠 核心思路

复杂任务 LLM 一步做不完，需要拆成子任务。

```text
用户说"加健康检查接口"
  ↓
Agent 拆成步骤：1.读路由 → 2.读启动文件 → 3.写 health.go → 4.注册路由 → 5.编译
  ↓
按顺序执行：每步都是一次完整的 LLM 调用
  ↓
某步失败了 → 自动重新规划剩余步骤 → 继续执行
```

---

## 🛠️ 动手实现

### 核心结构体

```go
type Step struct {
    ID     int    `json:"id"`
    Desc   string `json:"desc"`
    Status string `json:"status"` // pending / running / done / failed
}

type Plan struct {
    Goal  string `json:"goal"`
    Steps []Step `json:"steps"`
}
```

### GeneratePlan — 让 LLM 拆步骤

把当前工具列表发给 LLM，确保拆出的步骤能用现有工具完成。

```go
func (a *Agent) GeneratePlan(ctx, goal string) (*Plan, error) {
    prompt = "可用工具：... \n 请把目标拆成 3~8 个步骤。只输出 JSON。"
    resp := a.client.CreateChatCompletion(ctx, req)
    // 解析 JSON → Plan
}
```

### ReviewPlan — 自我审视

拆完 Plan 后，让 LLM 自己检查一遍质量。有问题就返回修正版，没问题返回 "OK"。

```go
func (a *Agent) ReviewPlan(ctx, plan) (*Plan, error) {
    prompt = "请检查这个 Plan 有没有遗漏步骤、顺序是否合理..."
    if resp == "OK" { return plan, nil }
    // 否则解析 LLM 返回的修正 Plan
}
```

### ExecutePlan + RePlan — 执行与动态重规划

执行中某步失败，自动调 RePlan 重新规划后续步骤。

```go
func (a *Agent) ExecutePlan(ctx, plan) error {
    for i := 0; i < len(plan.Steps); i++ {
        _, err := a.Run(ctx, plan.Steps[i].Desc)
        if err != nil {
            newPlan := a.RePlan(ctx, plan, i)  // 重规划剩余步骤
            plan.Steps = append(plan.Steps[:i+1], newPlan.Steps...)
            continue
        }
    }
}
```

---

## ✍️ 你自己试试

1. 让 Agent 拆一个不依赖任何工具的任务（比如"写一首诗"），看它还会拆步骤吗？
2. 如果没有 ReviewPlan，直接执行，Plan 质量差时会发生什么？
3. 把 RePlan 去掉，某步失败后直接返回 error —— 对比体验有什么不同？

---

## ✅ 完成标准

- [ ] Agent 能根据用户请求生成 3 步以上的 Plan
- [ ] Plan 按顺序执行，每步完成通知 LLM 结果
- [ ] ReviewPlan 能修正有问题的步骤
- [ ] 某步失败后老自动重规划并继续执行

**预计时间：** 3 天
