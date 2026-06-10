# 🏀 球球 V1 完结总结 — Planning 篇

> **目标：让 Agent 能把复杂任务先拆成步骤，再一步步执行，而不是一次硬扛。**
> **核心思路：Plan → Execute，错了执行中再修正。**

---

## 🎯 V1 解决了什么问题

V0 的 Agent 能跑通单轮对话 + 工具调用了，但有个大问题：

```
你：帮我统计当前目录所有文件的字符数，写入 11.txt
球球 V0：一次性去做 → 中途忘了前面的结果 → 输出不全
```

**问题不是 LLM 不够聪明，是一个 LLM 调用解决不了多步骤任务。** V1 就是让 Agent 学会"先拆步骤，一步步做"。

---

## 🧱 V1 核心改动

### 新增：Plan + Step 结构体

```go
type Step struct {
    ID     int    // 步骤编号
    Desc   string // 步骤描述，如"列出当前目录所有文件"
    Status string // pending / running / done / failed
}

type Plan struct {
    Goal  string // 总目标
    Steps []Step // 步骤列表
}
```

### 新增：GeneratePlan — 让 LLM 拆任务

```go
func (a *Agent) GeneratePlan(ctx, goal string) (*Plan, error) {
    // 把目标 + 可用工具列表发给 LLM
    // LLM 返回一步步的 JSON 步骤列表
    // 解析成 Plan 返回
}
```

### 新增：ExecutePlan — 按步骤执行

```go
func (a *Agent) ExecutePlan(ctx, plan *Plan) error {
    for 每一步 {
        fmt.Printf("[%d/%d] %s\n", i, total, step.Desc)
        a.Run(ctx, "请执行：" + step.Desc)  // 每步调一次 LLM
        step.Status = "done"
    }
}
```

### 关键设计：拆步骤时告诉 LLM 有哪些工具可用

```
GeneratePlan 的 prompt 包含：
- 当前注册的所有工具列表（名称 + 描述）
- 要求：每步必须能用现有工具完成

→ LLM 不会拆出"读数据库"这种没有对应工具的步骤
```

---

## 🔬 运行测试

### 输入

```
帮我计算当前目录所有文本文件的字符数，然后把结果写到 11.txt 里
```

### Plan 拆解

```
1. 列出当前目录所有文件
2. 过滤出文本文件路径
3. 统计每个文件的字符数
4. 累加字符数
5. 将结果写入 11.txt
```

### 执行过程（5 步，每步独立调 LLM）

```
[1/5] 列出当前目录所有文件
  → list_directory(".") ✅

[2/5] 过滤出文本文件路径
  → run_shell("dir /b") → run_shell("for %i in (*.txt *.md)") ✅
  （shell 命令第一次写错了，LLM 自己换写法重试，最终成功）

[3/5] 统计第一个文件字符数
  → count_file_chars("hello.txt") ✅

[4/5] 累加字符数并继续统计
  → read_file(每个文件) → 计数 ✅

[5/5] 将结果写入 11.txt
  → count_file_chars → write_file ✅

🎉 全部完成！
```

---

## 🧠 V1 学到的核心知识

### ① Planning 的本质

> **不是让 LLM 一次性做对，而是把问题拆到 LLM 能一次做对的粒度。**

每一步骤都是一个独立的 LLM 调用，只关注一件事。这样每一步都容易做对。

### ② Plan 的第一次拆解不一定好——没关系

Plan 的"正确性"不是由第一次拆解决定的，而是由执行中能否及时修正决定的。**LLM 执行时发现不对会自己调整。**

### ③ 工具缺失 ≠ 卡死

LLM 会自己尝试不同的方式绕过（你亲眼看到了——shell 命令失败了 3 次，LLM 自己换写法直到成功）。

但更好的方式不是让 LLM 硬扛，而是**发现缺工具后自己写一个注册进去**。

```
闭源 Agent：靠穷举大量工具减少缺工具的概率
你的球球：缺工具 → 写一个注册进去
```

后者比前者更灵活。

### ④ Agent 架构本质

```
工具列表 + 当前任务 → LLM → 要不要调工具？→ 调 → 结果喂回去 → 继续
                                               → 不调 → 输出答案
```

**Planning 只是在这个循环前面加了一步：先拆任务，再进入循环。** 没有改变 Agent 的本质。

---

## 📊 代码量

```
V1 总代码：约 350 行（含注释）
新增代码：约 100 行
  - Plan/Step 结构体：10 行
  - GeneratePlan：40 行
  - ExecutePlan：30 行
  - main 主流程调整：20 行
新增工具：1 个（count_file_chars）
```

---

## 📁 当前工具清单

| 工具 | 用途 | 说明 |
|------|------|------|
| `list_directory` | 列出目录内容 | 区分文件和子目录，带大小 |
| `read_file` | 读取文件内容 | 返回文件内容和元信息 |
| `write_file` | 写入文件 | 覆盖写入，返回字节数 |
| `count_file_chars` | 统计文件字符数 | UTF-8 实际字符数 |
| `run_shell` | 执行 cmd 命令 | 兜底，其他工具搞不定时用 |

---

## 🔜 下一步：V2 — Coding Agent

V1 的 Agent 能拆任务了，但它拆出来的步骤还是"读文件、统计、写入"这类操作。

V2 要让 Agent **能自己改代码**——找到要修改的文件、精确编辑、提交 Git、验证编译。

---

## 📚 学习笔记

| 文件 | 内容 |
|------|------|
| `v0-complete.md` | V0 完结总结 — Agent 基础篇 |
| `v1-complete.md` | **本文 — Planning 篇** |
| `README.md` | 完整 6 个月规划 |

---

## ✍️ 最后

**V1 你掌握的核心：**

> **Agent 处理复杂任务 = 先拆步骤 + 每步独立执行。拆错了没关系，执行中能修正。**

**接下来的 V2~V5 都是在这个框架上加能力，不是改框架本身。**
