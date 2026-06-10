# 🏀 球球学习日志 — V0 第 3 周：Tool 设计

> **阶段目标：** 优化工具设计，让 LLM 更容易理解和使用工具——命名清晰、参数简洁、返回值友好。
> **完成时间：** 第 2 天

---

## ✅ 完成内容

### 核心优化：三个设计原则

#### ① 命名即文档

工具名要让 LLM **一看就知道什么时候用它**：

```go
// ❌ 以前：run_shell 什么都能干，LLM 绕一道
run_shell({"command": "dir"})

// ✅ 现在：list_directory 专门列目录，LLM 直接调
list_directory({"path": "."})
```

#### ② 参数越少越好

所有工具参数控制在 1-2 个，LLM 填错概率低：

| 工具 | 参数 |
|------|------|
| `read_file` | `path` |
| `write_file` | `path`, `content` |
| `list_directory` | `path` |
| `run_shell` | `command` |

#### ③ 返回值对 LLM 友好

工具返回的内容是**给 LLM 看的**，不是给人看的：

```go
// ❌ 以前：原始错误，LLM 要猜
"open xxx: no such file or directory"

// ✅ 现在：友好描述，LLM 直接拿去回答用户
"读文件失败：找不到文件 xxx，请检查路径是否正确"
```

---

### 新增工具：list_directory

从 `run_shell("dir")` 独立出来，让 LLM 列目录时不用绕一道 shell：

```go
// list_directory 的输出：
目录 . 的内容：

📁 子目录：
  .codegraph
  .reasonix

📄 文件：
  main.go（9454 字节）
  readme.md（500 字节）
```

### run_shell 降级为"兜底工具"

在 Description 中引导 LLM **优先用专用工具**，只有搞不定时才用 shell：

```
Description: "优先使用 read_file、write_file、list_directory 等专用工具，
              只有它们搞不定时才用这个"
```

---

## 🔬 运行测试：4 轮对话

### 第 1 轮（list_directory 独立效果）

```
🧑 你: 列出当前目录
🔧 调用工具: list_directory({"path": "."})
📦 结果: 目录 . 的内容...
🤖 球球: 当前目录下的内容如下...
```

**→ 直接调 list_directory，没再绕 run_shell("dir") ✅**

### 第 2 轮（返回值友好效果）

```
🧑 你: 读取 readme
🤖 球球: 当前目录下没有 readme 文件。
         不过有一个类似介绍的 Markdown 文件——我是球球介绍.md，要读取它吗？
```

**→ 找不到文件时不再是报错，而是推荐了相似的文件 ✅**

### 第 3 轮（连续对话 + 自主推荐）

```
🧑 你: 那就读取你说的这个吧
🔧 调用工具: read_file({"path": "我是球球介绍.md"})
📦 结果: 文件 我是球球介绍.md（共 1698 字节）的内容如下...
🤖 球球: 以下是内容...（整理成表格输出）
```

**→ LLM 记得自己刚才推荐过什么，直接读取 ✅**

### 第 4 轮（write_file 正常）

```
🧑 你: 创建一个 hello.txt 写你好
🔧 调用工具: write_file({"path": "hello.txt", "content": "你好"})
📦 结果: 文件 hello.txt 已写入成功（共 6 字节）
🤖 球球: 已创建 hello.txt 文件
```

---

## 🧠 关键理解

### LLM 就是"人"

你总结得比我准确：

> **LLM 其实就和人一样，所以给 LLM 的东西最好做到见名知意，减少 LLM 瞎猜。**
>
> Tool 设计两个关键点：
> 1. **Tool 名要清晰** → LLM 一看就知道什么时候用
> 2. **输出结果要清晰** → LLM 一看就知道结果是什么

### 专用工具 > 通用工具

`list_directory` 比 `run_shell("dir")` 好，因为：

- LLM 不需要猜"列目录该用什么 shell 命令"
- 返回值格式化好了，LLM 直接拿去用
- 不依赖 shell 环境（Windows/Linux 差异由代码处理）

---

## 📁 项目结构

```
D:\AgentDemo\
├── go.mod
├── go.sum
└── main.go    ← 4 个工具，约 250 行，全中文注释
```

### 当前工具清单

| 工具 | 用途 | 说明 |
|------|------|------|
| `read_file` | 读取文件内容 | 返回值友好，失败时提示路径问题 |
| `write_file` | 写入文件 | 返回写入字节数 |
| `list_directory` | 列出目录 | 区分文件和子目录，带大小信息 |
| `run_shell` | 执行命令 | 降级为兜底，优先用上面三个 |

---

## ✅ V0 完成进度

| 周 | 内容 | 状态 |
|----|------|------|
| 第 1 周 | Tool Calling + Agent Loop | ✅ |
| 第 2 周 | 上下文管理（连续对话） | ✅ |
| 第 3 周 | Tool 设计（命名+返回值优化） | ✅ |
| 第 4 周 | 回顾与重构 | ⏳ 下一步 |

---

## 📚 参考资源

- [runoob AI Agent 工具设计](https://www.runoob.com/ai-agent/ai-agent-core.html) — 工具分类与设计
