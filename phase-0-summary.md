# 📖 Phase 0：名词扫盲 — 总结

> 目标：认全 Agent 领域的关键名词，不要求理解原理，只要求"见过这些词"。

---

## 🧠 5 个核心概念

### ① Agent（智能体）

**一句话：** 一个有大脑（LLM）、能动手（Tool）、能记事（Memory）、能规划（Planning）的系统。不是聊天框。

**学到的东西：**

> Agent 为了得到更准的答案，愿意去做一系列思考和动手操作，而不是像 Chatbot 一样只说话。Agent 是"为了得到更准的答案，做了一系列思考和动手操作的系统"。

---

### ② Tool（工具）

**一句话：** Agent 用来跟世界交互的"把手"——读文件、查数据库、发请求、执行命令。本质上就是一个函数，但 LLM 能"看到它"并"决定用它"。

**学到的东西：**

> Tool 其实就是一些函数。只不过在过去的开发中，函数是由人决定要不要调用的；在 Agent 中，是由模型自主决定要不要使用。Agent 用 Tool 就是为了给出真实答案，而不是编造。

---

### ③ Memory（记忆）

**一句话：** Agent 的"记事本"——短期记忆（当前对话上下文）+ 长期记忆（存到数据库里）。没有 Memory 的 Agent 每次对话都是"第一次见你"。

**学到的东西：**

> Memory 能够解决 LLM 能否将上下文联系起来的问题。如果没有 Memory，相当于每次和 LLM 对话都是一次新的对话，LLM 不知道之前我们做了什么。没有上下文的 LLM 可能会乱编造。Memory 就是支持我们在和 LLM 对话时能够针对一个问题不停地修补和完善。

---

### ④ Planning（规划）

**一句话：** 把"帮我做个登录功能"拆成"1. 分析路由 → 2. 设计数据库 → 3. 写代码 → 4. 测试"。复杂任务 LLM 一步做不完，需要拆成子任务一步步做。

**学到的东西：**

> 在日常的任务中其实都很复杂，Agent 接收到的指令其实很笼统，所以需要 Planning 将任务分析并拆分成 LLM 可执行的、颗粒度更细的任务。越简单的任务 LLM 越容易执行。并且有些子任务其实是可以并行的，所以 Planning 可以让 LLM 更完美地完成任务，甚至缩短完成任务的时间。如果没有 Planning，LLM 可能理解不了复杂任务，也可能因为复杂任务有大量上下文、LLM 没法承载这么多上下文，导致任务中后期上下文丢失，进而胡乱执行。

---

### ⑤ Skill（技能）

**一句话：** Agent 的"专业模式"——切到架构师模式它会出设计文档，切到审查模式它会找 bug。本质上是换一套提示词 + 工具组合。

**学到的东西：**

> Tool 是 Agent 能够调用的能力，Skill 更像是 Agent 拥有的知识。Skill 可以让 Agent 在执行某一类任务的时候有一定的范式，可以约束 Agent 去调用相应的 Tool，而不是让 Agent 自己去发散地想要用什么 Tool。

---

## 🎯 一句话记住全部

> **Agent = LLM（大脑）+ Tool（手）+ Memory（记忆）+ Planning（拆解）**
>
> **Skill 是在这之上加了一层"人格切换"。**

---

## ✅ 完成标准

- [x] 能用自己的话解释 Agent、Tool、Memory、Planning、Skill 这 5 个词
- [x] 不会混淆"Tool"和"Function Calling"（Tool 是函数本身，Function Calling 是 LLM 调用函数的机制）
- [x] 知道 Agent ≠ Chatbot（Agent 能行动，Chatbot 只能说话）

**完成时间：** 约 1 小时

---

## 📚 参考资源

- [runoob AI Agent 教程首页](https://www.runoob.com/ai-agent/ai-agent-tutorial.html) — 整个教程的入口
- [AI Agent 简介](https://www.runoob.com/ai-agent/ai-agent-intro.html)
- [AI Agent 核心组件](https://www.runoob.com/ai-agent/ai-agent-core.html)

---

## 🔜 下一步

进入 **V0：Agent 基础**——写 Go 程序调通 LLM，实现第一个 Agent Loop。

```
Phase 0（名词扫盲） ✅
  ↓
V0  第 1 个月  Agent 基础   LLM + Tool 循环跑通
V1  第 2 个月  Planning     拆解任务、规划执行
V2  第 3 个月  Coding Agent 自己改代码、提交 Git
V3  第 4 个月  Runtime      Event Log、Checkpoint、状态恢复
V4  第 5 个月  MCP 生态     外部插件、MCP 工具
V5  第 6 个月  Skill 体系   专业能力包切换
```
