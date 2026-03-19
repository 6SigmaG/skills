# /unfreeze 深度解构

> **角色：** 编辑锁解除（Unlock）
> **定位：** 清除 /freeze 设置的编辑范围限制
> **类型：** 工具型 Skill（/freeze 的逆操作）
> **allowed-tools：** Bash
> **来源：** `unfreeze/SKILL.md`

---

> **结构说明：** 本文极度精简，因为 /unfreeze 是一个单一操作的工具。

## 一、做什么

删除 `~/.gstack/freeze-dir.txt` 状态文件，使 freeze hook 不再阻止任何编辑。

---

## 二、工作流程

```
用户输入 /unfreeze
       │
       ▼
检查 freeze-dir.txt 是否存在
       │
       ├── 存在 → 删除文件
       │         报告 "之前的限制目录是 X"
       │         "编辑范围已解除"
       │
       └── 不存在 → "当前没有活跃的 freeze"
```

---

## 三、技术细节

**Hook 仍然注册。** `/unfreeze` 只删除状态文件，不注销 hook。Hook 在检测不到状态文件时自动放行所有操作。

这意味着可以快速在 freeze/unfreeze 之间切换，而不需要重新安装 hook。

---

## 四、使用场景

| 场景 | 操作序列 |
|------|---------|
| 完成聚焦开发 | `/freeze src/api/` → 开发 → `/unfreeze` |
| 切换工作范围 | `/unfreeze` → `/freeze src/frontend/` |
| 需要改全局配置 | `/unfreeze` → 修改 → `/freeze src/api/` |
