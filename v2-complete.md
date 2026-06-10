# 🏀 球球 V2 完结总结 — Coding Agent 篇

> **目标：让 Agent 能精确修改代码、提交 Git，而不是整文件覆盖。**
> **核心发现：V2 = V1 + 3 个工具，循环没变，拆步骤没变。**

---

## 🎯 V2 解决了什么问题

V1 的 Agent 能拆任务了，但改代码只会 `write_file`——整文件覆盖。改一行就要传整个文件，token 浪费，还容易搞坏没让改的部分。

V2 让 Agent 改代码像人一样：**找到要改的地方，精确替换，验证编译，提交 Git，改错了能回滚。**

---

## 🧱 V2 核心改动：只加了 3 个工具

| 工具 | 作用 | 关键设计 |
|------|------|---------|
| `edit_file_block` | 找到一段旧代码，替换成新代码 | 找不到/找到多处就拒绝，**不会乱改** |
| `git_commit` | 提交所有变更 | 自动 `git add .` |
| `git_revert_file` | 恢复文件到上一个提交 | 改错了直接撤销 |

**Agent 循环、Plan 拆步骤、执行方式——一丁点没变。** V2 就是对"V1 + 新工具"的验证。

---

## 🔬 运行测试

### 输入

```
给 main.go 的 main 函数开头加一行 fmt.Println("球球 Coding Agent 启动")
```

### Agent 拆的 Plan

```
1. 读取 main.go 文件
2. 定位 main 函数开头
3. 编辑添加 fmt.Println
4. 提交 Git 变更
```

### 关键过程

**Step 3 编辑文件：**

```
第一次 edit_file_block → 失败（LLM 猜错了换行符格式）
第二次 edit_file_block → 成功 ✅（LLM 修正了缩进风格）
```

**工具本身没有问题，问题是 LLM 需要猜代码里的缩进风格。** 第一次猜错了，第二次自己修好了。

**Step 4 提交 Git：**

```
前几次 git commit -m "message" → 失败（cmd 引号解析问题）
最后 git commit -m adjusted → 成功 ✅
```

**cmd 的引号处理方式和 Git Bash 不一样，导致 LLM 生成的命令经常出错。**

---

## 🧠 V2 学到的核心知识

### ① V2 不是新架构，是 V1 + 新工具

```
V0：跑通 Agent Loop
V1：V0 + Planning（拆步骤）
V2：V1 + Coding 工具（edit、git）
```

**每一轮都是在上一轮的能力上叠加工具，核心循环不变。**

### ② 精确编辑的关键问题

`edit_file_block` 要"找到一段旧代码"——问题是 LLM 不一定知道代码里的缩进是空格还是 Tab、是几个空格。**解决方案是让 LLM 先读文件确认格式，再提交编辑。** 目前的 LLM 已经会自己读了再改。

### ③ Windows 的命令行是最大的坑

V0~V2 的测试在 Windows 上，`run_shell` 遇到了很多 cmd 特有的问题：

```text
findstr 语法跟 grep 不一样
git commit -m 的引号在 cmd 里被吃掉
路径里有空格会报错
```

**这不是 Agent 的问题，是操作系统差异的问题。** 换 Linux/Mac 会好很多。

### ④ 你的观察完全正确

> **"V2 就是加了几个工具来实现的。"**

对，而且往后 V3、V4、V5 都是这个模式——**Agent 架构不会变了，变的是工具列表越来越长。**

---

## 📊 代码量

```
V2 总代码：约 450 行（含注释）
新增代码：约 100 行（3 个工具）
新增工具：3 个（edit_file_block、git_commit、git_revert_file）
当前工具总数：8 个
```

### 当前工具清单

| 工具 | 用途 | 来源 |
|------|------|------|
| `list_directory` | 列出目录 | V0 |
| `read_file` | 读取文件 | V0 |
| `write_file` | 写入文件 | V0 |
| `count_file_chars` | 统计字符数 | V1 |
| `edit_file_block` | 精确编辑代码 | **V2** |
| `git_commit` | 提交 Git | **V2** |
| `git_revert_file` | 回滚文件 | **V2** |
| `run_shell` | 执行命令 | V0（兜底） |

---

## 🔜 下一步：V3 — Runtime

V2 的 Agent 能改代码了，但它有个问题：**跑着跑着崩溃了，状态全丢。**

V3 要解决这个问题——Event Log、Checkpoint、状态恢复。

---

## 📚 学习笔记

| 文件 | 内容 |
|------|------|
| `v0-complete.md` | Agent 基础篇 |
| `v1-complete.md` | Planning 篇 |
| `v2-complete.md` | **本文 — Coding Agent 篇** |
| `README.md` | 完整规划 |

---

## ✍️ 最后

**V2 你掌握的核心：**

> **Agent 的能力边界 = 工具列表。想让 Agent 能做什么，就给注册什么工具。**
>
> V2 不是新架构，是 V1 加了你需要的三个工具。

**剩下的问题（shell 命令优化、git 提交防错）是"优化 1"不是"缺失 0"，按你的计划——先走完 V0~V5，回头再优化。**
