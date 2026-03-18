# /document-release 深度解构

> **角色：** 技术作家（Technical Writer）
> **定位：** 在 `/ship` 之后、PR 合并之前，自动更新所有因代码变更而过时的文档
> **Prompt 长度：** ~356 行 | **allowed-tools：** Bash, Read, Write, Edit, Grep, Glob, AskUserQuestion
> **来源：** `document-release/SKILL.md.tmpl`

---

## 一、为什么需要这个 Skill

**问题：** 代码改了，文档没跟上。README 里的安装命令过时了。ARCHITECTURE.md 的组件图和实际代码不一致。CONTRIBUTING.md 的设置步骤跑不通。

这不是懒——是成本问题。以前更新文档的人力成本和写代码差不多。但 AI 把这个成本压缩到接近零。

**gstack 的方案：** `/document-release` 读取 diff，交叉对比每一个 .md 文件，自动更新事实性内容（文件路径、命令列表、项目结构），对有风险的叙事性修改提问。

---

## 二、完整结构拆解

### 2.1 定位：在流水线中的位置

```
编码 → /review → /qa → /ship → /document-release → PR 合并
                                      ↑
                                  你在这里
```

**关键约束：** 代码已提交，PR 已存在（或即将存在）。`/document-release` 不改代码——只改文档。

### 2.2 Step 1：预飞行与 Diff 分析

```bash
git diff <base>...HEAD --stat          # 变更概览
git log <base>..HEAD --oneline         # commit 历史
git diff <base>...HEAD --name-only     # 变更文件列表
find . -maxdepth 2 -name "*.md" ...    # 发现所有文档文件
```

**变更分类：** 新功能、行为变更、功能移除、基础设施变更。

### 2.3 Step 2：逐文件文档审计

对每个 .md 文件，交叉对比 diff：

| 文档 | 审计重点 |
|---|---|
| **README.md** | 功能描述是否完整？安装/设置/示例/故障排除是否仍然有效？ |
| **ARCHITECTURE.md** | 图表/组件/设计决策是否匹配代码？（保守——架构文档不常变） |
| **CONTRIBUTING.md** | 以新贡献者身份走一遍设置流程——命令是否准确？测试层级是否匹配？ |
| **CLAUDE.md** | 项目结构是否匹配文件树？命令/脚本/构建/测试指令是否当前？ |
| **其他 .md** | 确定目的和受众，交叉对比 diff |

**分类：**
- **Auto-update（自动更新）：** 事实性、明确正确的——直接修改
- **Ask user（询问用户）：** 叙事性、删除操作、安全相关、大段重写（>10 行）

### 2.4 Step 3-4：自动更新 + 询问有风险的修改

**自动更新规则：**

> 永远不要自动更新：引言、哲学、安全模型，或删除章节。

每个自动更新输出一行摘要：`README.md: 更新了 Skill 数量从 9 到 10，在表格中添加了新 Skill`。

**有风险的修改：** 每个用 AskUserQuestion 单独提问，附推荐和"C) 跳过"选项。

### 2.5 Step 5：CHANGELOG 语气润色（关键步骤）

这是 `/document-release` 最精妙的设计之一。

**核心规则：**

> **永远不要覆盖 CHANGELOG 条目。** 只润色措辞。保留所有内容。先读完整个 CHANGELOG 再改。

**润色标准（"售卖测试"）：**

> 每一条都应该让读者想"哦不错，我想试试这个"。
> 用"你现在可以..."开头，不用"重构了..."。
> 具体地说它做什么，不说实现细节。

**示例：**
```
# ❌ 原始（开发者视角）
- 重构了 gen-skill-docs.ts 中的模板解析器

# ✅ 润色后（用户视角）
- 你的 Skill 文档现在在构建时自动和源码同步——不再有过时的命令参考
```

**自动修正 vs 提问边界：** 小的语气调整自动做。如果重写会改变含义，用 AskUserQuestion。

### 2.6 Step 6：跨文档一致性

更新完单个文件后，做一次全局检查：

- README 功能 ↔ CLAUDE.md 描述
- ARCHITECTURE 组件 ↔ CONTRIBUTING 项目结构
- CHANGELOG 版本 ↔ VERSION 文件
- **可发现性：** 每个文档都应该从 README 或 CLAUDE.md 可达

事实性不一致自动修复。叙事性矛盾提问。

### 2.7 Step 7：TODOS.md 清理

这是 `/ship` Step 5.5 的补充第二轮：

- 检测已完成项（保守——需要 diff 中的明确证据）
- 更新过时的描述
- 从 TODO/FIXME/HACK/XXX 注释中捕获新的延后工作

### 2.8 Step 8：VERSION 检查

**关键规则：**

> **永远不要悄悄 bump VERSION。**

如果 VERSION 文件存在且未在分支上被 bump：
- 推荐：跳过（纯文档变更很少需要版本 bump）
- 选项：A) Bump PATCH  B) Bump MINOR  C) Skip

如果已被 bump：检查 CHANGELOG 条目是否覆盖了完整范围。如果有遗漏，提问。

### 2.9 Step 9：提交与输出

**空检查：** 如果没有任何文档文件被修改，直接退出："所有文档都是最新的。"

**安全提交：** 按文件名添加（不用 `git add -A`），推送，更新 PR body。

**输出：** 文档健康度摘要表

```
| 文件              | 状态                          |
|-------------------|-------------------------------|
| README.md         | Updated — 更新了 Skill 计数    |
| ARCHITECTURE.md   | Current — 无需更改            |
| CONTRIBUTING.md   | Updated — 修正了设置命令       |
| CHANGELOG.md      | Voice polished — 3 条润色      |
| VERSION           | Not bumped                    |
```

---

## 三、设计原理分析

### 为什么分离 /document-release 和 /ship？

`/ship` 已经做了版本 bump 和 CHANGELOG 生成。为什么还需要单独的文档 Skill？

因为 **ship 关注的是"代码准备好发布了吗"，document-release 关注的是"文档和代码一致吗"**。这是两个不同的关注点。把它们分开意味着：
1. 文档更新可以独立运行（比如在 PR review 阶段）
2. 文档审计可以做得更深（CONTRIBUTING 冒烟测试、跨文档一致性检查）
3. 失败不会阻塞发布

### CHANGELOG "售卖测试"的精妙

大多数 CHANGELOG 是给开发者看的技术日志。gstack 把它重新定义为**产品发布说明**——用户看了想"我要试试"。这个视角转变是微妙但强大的。

### "可发现性"作为文档质量指标

大多数文档审计只看"准确性"。gstack 增加了"可发现性"——每个文档必须从入口点（README 或 CLAUDE.md）可达。不可达的文档等于不存在。

---

## 四、迁移价值

### 可复用的设计模式

1. **Diff→文档交叉对比**：对任何"内容和源码需要保持同步"的场景有效（API 文档、配置说明、FAQ）
2. **Auto-update vs Ask 分类**：事实性变更自动做，叙事性变更提问。这个分类可以迁移到任何自动化编辑工作流
3. **CHANGELOG 售卖测试**：把技术日志写成产品发布说明。适用于任何面向用户的更新通知
4. **可发现性检查**：每个文档都应该从入口点可达
5. **保守的 CHANGELOG 处理**：只润色不覆盖，多重安全保护

### 迁移场景

**API 文档同步 Skill：** 保留 diff→文档交叉对比，替换文件列表为 API endpoint 文档。

**内部 Wiki 更新 Skill：** 保留跨文档一致性检查，增加 Confluence/Notion API 集成。

---

> 本分析基于 `document-release/SKILL.md.tmpl`（356 行）源码逐段分析。
