# /freeze 深度解构

> **角色：** 编辑锁管理员（Edit Lock）
> **定位：** 限制文件编辑范围到指定目录，防止 AI 修改不该动的文件
> **类型：** 工具型 Skill（防护层，非角色型）
> **allowed-tools：** Bash, AskUserQuestion
> **来源：** `freeze/SKILL.md`

---

> **结构说明：** 本文采用简化结构，因为 /freeze 是一个纯工具型 Skill。

## 一、为什么需要这个 Skill

**问题：** AI Agent 在修复 Bug 或添加功能时，可能"顺手"修改不相关的文件。比如你让它改前端组件，它可能同时"优化"了后端 API。在大型代码库中，这种越界编辑可能引入意外回归。

**gstack 的方案：** 设置一个"编辑围栏"——所有 Edit 和 Write 操作只允许在指定目录内进行。越界操作**直接阻止**（不是警告）。

**与 /careful 的区别：** `/careful` 是警告后放行，`/freeze` 是**硬阻止**。因为文件编辑比 Bash 命令更常见，如果每次都弹确认框，工作流会被严重打断。

---

## 二、工作流程

```
用户输入 /freeze
       │
       ▼
"你想限制编辑范围到哪个目录？"
       │
       ▼
用户输入路径（如 src/frontend）
       │
       ▼
转换为绝对路径
       │
       ▼
保存到 ~/.gstack/freeze-dir.txt
       │
       ▼
"编辑已限制到 /path/to/src/frontend/"
       │
       ▼
后续所有 Edit/Write 操作
       │
       ├── 目标在范围内 → ✓ 允许
       └── 目标在范围外 → ✗ 阻止
```

---

## 三、技术实现

### 3.1 Hook 机制

使用 Claude Code 的 pre-tool hook，在每次 Edit 或 Write 调用前执行 `check-freeze.sh`：

```bash
# 伪代码逻辑
target_path=$(realpath "$1")
freeze_dir=$(cat ~/.gstack/freeze-dir.txt)

if [[ "$target_path" != "$freeze_dir"* ]]; then
    echo "BLOCKED: Edit outside freeze boundary"
    exit 1
fi
```

### 3.2 重要限制

**只拦截 Edit 和 Write 工具，不拦截 Bash。**

这意味着 `sed`、`awk`、`echo >` 等通过 Bash 工具执行的文件修改**不会被阻止**。文档明确说明：

> "Freeze applies to Edit and Write tools only — Read, Bash, Glob, Grep are unaffected."

**这不是安全边界**，而是防意外的护栏。

### 3.3 解除方式

- 运行 `/unfreeze`
- 或结束当前会话

---

## 四、使用场景

| 场景 | 冻结目录 | 价值 |
|------|---------|------|
| 前端开发 | `src/frontend/` | 防止改后端 |
| Bug 修复 | `src/module-x/` | 聚焦修复范围 |
| 设计实现 | `src/components/` | 防止改业务逻辑 |
| 文档更新 | `docs/` | 防止改代码 |

---

## 五、Prompt Engineering 可迁移模式

### 模式 1：路径前缀匹配
用最简单的字符串前缀匹配实现范围控制——不需要复杂的权限模型。

### 模式 2：阻止 vs 警告的选择
高频操作（文件编辑）用硬阻止，低频操作（危险命令）用警告。根据操作频率选择防护强度。

### 模式 3：状态文件驱动
Hook 的行为由一个简单的状态文件控制——删除文件即解除限制。极其简单的状态管理。
