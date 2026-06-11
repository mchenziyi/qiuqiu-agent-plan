# 第 3 章：Coding Agent——让 Agent 能改代码

> **本章对应 V2，让 Agent 能精确修改文件、提交 Git、验证编译。**
> **代码 Tag：`v2`**

---

## 🎯 预期收获

学完这一章，你能：

- 让 Agent 精确编辑文件（而不是整文件替换）
- 自动 `git commit` 每次修改
- 修改导致编译错误时能自动回滚

---

## 🧠 核心思路

```
用户说"给 main 函数加一行日志"
  ↓
1. 读取 main.go → 找到 main 函数位置
2. 精确插入一行代码（只改那一行，不碰其他）
3. go build 验证编译
4. 编译通过 → git commit
5. 编译失败 → git checkout 回滚
```

---

## 🛠️ 动手实现

### 精确编辑工具

```go
func NewEditFileBlockTool() Tool {
    Execute: func(args string) string {
        // 读文件
        // 找旧代码 → 找不到则拒绝
        // 旧代码出现多次则拒绝（防止改错位置）
        // 替换 → 写回
    }
}
```

核心设计：**找不到或找到多处就拒绝。** 宁可不改，也不乱改。

### Git 工具

```go
func NewGitCommitTool() Tool {
    Execute: func(args string) string {
        exec.Command("git", "add", ".").Run()
        exec.Command("git", "commit", "-m", message).Run()
    }
}

func NewGitRevertFileTool() Tool {
    Execute: func(args string) string {
        exec.Command("git", "checkout", "--", path).Run()
    }
}
```

### 验证 + 回滚机制

```go
// 修改前先确保有干净的 git 状态
// 修改后自动 go build
// 编译失败 → git checkout -- 回滚
```

---

## ✍️ 你自己试试

1. 让 Agent 修改一个不存在的文件，看它会怎么处理？
2. 如果没有 `edit_file_block`，只有 `write_file`，LLM 改一行代码会怎样？
3. 试试让 Agent 同时改 2 个文件，看它是一次性改完还是分两次 commit？

---

## ✅ 完成标准

- [ ] Agent 能精确修改文件的指定位置
- [ ] 每次修改自动 git commit
- [ ] 编译错误时自动回滚
- [ ] 你能说出精确编辑和整文件替换的区别

**预计时间：** 3 天
