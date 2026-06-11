# 第 0 章：名词扫盲——Agent 到底是什么

> **本章对应 Phase 0，纯概念，不写代码。**

---

## 🎯 预期收获

学完这一章，你能：

- 用自己的话解释 Agent、Tool、Memory、Planning、Skill 这 5 个词
- 知道 Agent 和 Chatbot 的本质区别
- 理解 Agent = LLM + Tool + Memory + Planning 这个公式

---

## 🧠 核心思路

不要一开始就写代码。先认识这 5 个词，后面写代码时碰到不会慌。

### ① Agent（智能体）

**一句话：** 一个有大脑（LLM）、能动手（Tool）、能记事（Memory）、能规划（Planning）的系统。不是聊天框。

Agent 和 Chatbot 的本质区别：

| | Chatbot | Agent |
|--|---------|-------|
| 你说"读一下 config.json" | 它编一个答案 | 它真的去读文件 |
| 能力边界 | 只能说话 | 能行动 |

### ② Tool（工具）

**一句话：** Tool 就是函数。以前人决定调不调，现在模型自己决定。

Agent 用 Tool 是为了得到真实答案，而不是编造。

### ③ Memory（记忆）

**一句话：** 让 LLM 记住上下文，不瞎编。

没有 Memory 的 Agent，每次对话都是"第一次见你"。

### ④ Planning（规划）

**一句话：** 把"做个登录功能"拆成"分析路由 → 设计数据库 → 写代码 → 测试"。

越简单的任务 LLM 越容易执行。Planning 就是把复杂任务拆到 LLM 能一次处理的粒度。

### ⑤ Skill（技能）

**一句话：** Tool 是能力（能做什么），Skill 是知识 + 范式（怎么做得更好）。

Skill 约束 Agent 不发散——切到架构师模式它出文档，切到审查模式它找 bug。

---

## 🛠️ 动手实现

这一章不动手。只动嘴：

遮住上面的解释，对每个词说一句自己的话。能说清楚就算过。

**核心公式：** Agent = LLM（大脑）+ Tool（手）+ Memory（记忆）+ Planning（拆解）

---

## ✍️ 你自己试试

1. 用你的话向一个不懂技术的朋友解释：Agent 和 ChatGPT 有什么区别？
2. 如果 Agent 没有 Tool，它能做什么？不能做什么？
3. 如果 Agent 没有 Memory，连续问 3 个相关的问题会出什么状况？
4. 你觉得"Skill"和"Tool"的区别是什么？举个例子说明。

---

## ✅ 完成标准

- [ ] 能用自己的话解释 Agent、Tool、Memory、Planning、Skill
- [ ] 不会混淆 Tool 和 Function Calling
- [ ] 知道 Agent ≠ Chatbot

**预计时间：** 1 小时
