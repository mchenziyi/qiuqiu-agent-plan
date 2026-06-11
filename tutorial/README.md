# 🏀 球球 Agent 开发实战教程

> **从零开始，手写一个属于自己的 Agent 系统。**
>
> 不需要机器学习基础，不需要懂 Transformer，只需要会一点 Go。
> 整个教程基于真实项目 [QiuQiuPro](https://github.com/mchenziyi/QiuQiuPro) 的演进过程整理而成。

---

## 适合谁

- 会一点 Go（能读懂 Go 代码）
- 用过 ChatGPT / Claude，但想知道它背后怎么工作的
- 想自己做一个 Agent，但不知道从哪里开始

---

## 怎么用这个教程

每章结构：

```text
① 预期收获 — 这一章学完你能做什么
② 核心思路 — 这一章要解决什么问题
③ 动手实现 — 跟着步骤写代码
④ 你自己试试 — 不告诉答案的练习
```

代码版本：每章结尾的代码状态对应 QiuQiuPro 仓库的一个 Git tag：

```bash
git checkout v0   # 第 2 章结束时的代码
git checkout v1   # 第 3 章结束时的代码
# ...
```

---

## 目录

| 章 | 内容 | 对应阶段 | 代码 Tag |
|----|------|---------|----------|
| 第 0 章 | [名词扫盲——Agent 到底是什么](./00-noun-scanning.md) | Phase 0 | - |
| 第 1 章 | [Agent 基础——手写第一个 Agent Loop](./01-agent-basics.md) | V0 | `v0` |
| 第 2 章 | [Planning——让 Agent 学会拆任务](./02-planning.md) | V1 | `v1` |
| 第 3 章 | [Coding Agent——让 Agent 能改代码](./03-coding-agent.md) | V2 | `v2` |
| 第 4 章 | [Runtime——Event Sourcing 与崩溃恢复](./04-runtime.md) | V3 | `v3` |
| 第 5 章 | [MCP 生态——外部工具即插即用](./05-mcp.md) | V4 | `v4` |
| 第 6 章 | [Skill 体系——Agent 人格切换](./06-skill.md) | V5 | `v5` |
| 第 7 章 | [优化——把 Agent 打磨成产品](./07-optimization.md) | QiuQiuPro | `v6`（最新） |

---

## 推荐的学习节奏

```text
每天 1-2 小时，每周 1 章，2 个月完成全部 8 章。

第 0 章：1 天（纯概念）
第 1 章：1 周（核心代码量最大）
第 2 章：3 天（代码改动小）
第 3 章：3 天（主要加工具）
第 4 章：3 天（Event Sourcing）
第 5 章：3 天（MCP 客户端）
第 6 章：3 天（Skill 结构体）
第 7 章：1 周（按兴趣选做）
```

---

## 配套资源

| 资源 | 地址 |
|------|------|
| 完整项目代码 | [github.com/mchenziyi/QiuQiuPro](https://github.com/mchenziyi/QiuQiuPro) |
| 学习笔记仓库 | [github.com/mchenziyi/qiuqiu-agent-plan](https://github.com/mchenziyi/qiuqiu-agent-plan) |
| 架构总图 | [architecture-overview.md](../architecture-overview.md) |
| DeepSeek API 文档 | [api-docs.deepseek.com](https://api-docs.deepseek.com) |
| MCP 协议规范 | [modelcontextprotocol.io](https://modelcontextprotocol.io) |

---

## 关于本教程

本教程源于作者从零学习 Agent 开发的真实记录。作者起点只是"会一点 Go、没写过 Agent"——这也正是你现在的起点。
