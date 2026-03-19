# /careful 深度解构

> **角色：** 安全守卫（Safety Guardrails）
> **定位：** 在执行危险命令前弹出警告，防止误操作
> **类型：** 工具型 Skill（防护层，非角色型）
> **allowed-tools：** Bash
> **来源：** `careful/SKILL.md`

---

> **结构说明：** 本文采用简化结构，因为 /careful 是一个纯防护型 Skill，没有复杂的角色策略或多阶段工作流。

## 一、为什么需要这个 Skill

**问题：** AI Agent 拥有 Bash 权限后，一条 `rm -rf /` 或 `git push --force` 就可能造成不可逆损失。Claude Code 本身有安全检查，但不是专门针对开发工作流优化的。

**gstack 的方案：** 一个轻量级的命令拦截层——匹配已知危险模式，命中时弹出警告而非直接阻止。用户可以覆盖警告继续执行（"我知道我在干什么"）。

**设计哲学：** 安全而非安全剧场。不是禁止危险操作，而是确保你是**故意的**。

---

## 二、防护范围

### 2.1 拦截的危险模式

| 类别 | 示例 |
|------|------|
| 递归删除 | `rm -rf`, `rm -r` |
| 数据库破坏 | `DROP TABLE`, `DROP DATABASE`, `TRUNCATE` |
| Git 历史改写 | `git push --force`, `git reset --hard` |
| Kubernetes 删除 | `kubectl delete` |
| Docker 清理 | `docker rm -f`, `docker system prune` |

### 2.2 安全例外（不触发警告）

常见的构建产物目录会被自动排除：

```
node_modules/  .next/  dist/  __pycache__/
.cache/  build/  .turbo/  coverage/
```

**为什么有例外？** `rm -rf node_modules` 是 Node.js 开发中最常见的操作之一。如果每次清理 node_modules 都弹警告，开发者很快就会关掉整个 Skill。**好的安全系统是高信噪比的**——只在真正危险的时候响。

---

## 三、技术实现

### 3.1 Hook 机制

```
用户说 "删掉旧数据"
        │
        ▼
Claude 生成 Bash 命令
        │
        ▼
┌─────────────────────────┐
│ pre-tool hook (Bash)     │
│ 检查命令是否匹配危险模式  │
├─────────────────────────┤
│ 匹配？                   │
│ ├── 是 → 返回警告        │
│ │      用户确认后放行     │
│ └── 否 → 直接执行        │
└─────────────────────────┘
```

### 3.2 激活方式

通过自然语言触发：
- "be careful"
- "safety mode"
- "prod mode"
- "careful mode"

---

## 四、与 /freeze 和 /guard 的关系

| Skill | 防护类型 | 范围 |
|-------|---------|------|
| `/careful` | 危险命令警告 | 所有 Bash 命令 |
| `/freeze` | 文件编辑范围限制 | Edit/Write 工具 |
| `/guard` | 两者组合 | Bash + Edit/Write |

三个 Skill 形成了一个渐进式安全体系：最轻量的 `/careful`，中等的 `/freeze`，最严格的 `/guard`。

---

## 五、Prompt Engineering 可迁移模式

### 模式 1：模式匹配 + 警告而非阻止
拦截危险操作但保留用户覆盖权。这比硬阻止更实用——开发者需要偶尔执行危险操作。

### 模式 2：高信噪比例外
主动排除常见安全操作，避免"狼来了"效应。

### 模式 3：自然语言激活
用户不需要记住精确的命令，用自然语言描述意图即可。
