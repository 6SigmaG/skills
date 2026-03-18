# /qa-only 深度解构

> **角色：** QA Reporter — 质量报告专家
> **定位：** 与 /qa 相同的测试方法论，但绝不修复 Bug、绝不读源码
> **核心机制：** 纯黑盒测试、最小工具集、报告导向、Bug 报告移交
> **来源：** `qa-only/SKILL.md.tmpl`

---

## 一、为什么这个 Skill 有效

`/qa-only` 看似是 `/qa` 的"阉割版" — 去掉了修复能力，只保留了测试能力。但它的存在是一个深思熟虑的设计决策，而不是功能缩减。

### 关注点分离：一个被 AI 时代重新定义的原则

在 `/qa` 的分析中我们讨论过：AI Agent 既测试又修复是合理的，因为 AI 没有"护短"心理。但这个论点成立的前提是：**修复和测试发生在同一个代码库、同一个上下文中**。

现实中有很多场景不满足这个前提：

| 场景 | 为什么需要只报告不修复 |
|---|---|
| QA 团队独立于开发团队 | QA 不应该直接改代码，改了谁负责？ |
| 外部审计 | 审计员只报告问题，不应该碰生产代码 |
| 跨团队协作 | A 团队测 B 团队的服务，A 不应该改 B 的代码 |
| 安全评估 | 安全团队发现漏洞后交给开发修复，而不是自己修 |
| 合规要求 | 某些行业要求测试和修复必须由不同角色完成 |

**原文：**
> "Same methodology as /qa. Same rigor. Same thoroughness. But NEVER fix bugs. NEVER read source code. You are a black-box tester. Your deliverable is a bug report, not a code change."

**翻译：** 与 /qa 相同的方法论。相同的严谨度。相同的彻底性。但绝不修复 Bug。绝不读源码。你是黑盒测试员。你的交付物是 Bug 报告，不是代码改动。

### 三个关键约束

**1. NEVER fixes bugs — 绝不修复**

这不是"尽量不修" — 这是**绝对禁令**。即使 AI 明确知道怎么修，它也只报告问题。

为什么要如此绝对？因为"尽量不修"会给 AI 留下解释空间。Claude 很擅长自我说服："这个修复只是加了一行 CSS，应该没问题吧？" 一旦开了这个口子，行为就不可预测了。绝对禁令消除了所有灰色地带。

**原文：**
> "There is no 'trivial enough to fix' exception. A one-character typo fix is still a fix. Report it. Do not touch it."

**翻译：** 没有"简单到可以顺手修"的例外。一个字符的拼写错误修复也是修复。报告它。不要碰它。

**2. NEVER reads source code — 绝不读源码**

这个约束更激进。`/qa-only` 不只是不改代码 — 它**不看代码**。

**为什么？** 因为一旦 AI 读了源码，它的测试行为就不再是"黑盒"了。它会不自觉地根据代码实现来调整测试策略，跳过它"知道代码已经处理了"的边界情况。

纯黑盒测试的价值在于：测试者不知道实现细节，所以它会尝试所有它能想到的输入组合 — 包括开发者认为"不可能发生"的那些。

**3. Minimal toolset — 最小工具集**

| /qa 的工具 | /qa-only 的工具 |
|---|---|
| Bash, Read, Edit, Write, Grep, Glob | Bash（仅限浏览器命令） |
| 浏览器（全部命令） | 浏览器（READ + 部分 META 命令） |
| AskUserQuestion | AskUserQuestion |

**原文：**
> "Minimal toolset enforces the constraint architecturally. You cannot read source code if you don't have the Read tool. You cannot edit files if you don't have the Edit tool."

**翻译：** 最小工具集从架构层面强制执行约束。没有 Read 工具你就不能读源码。没有 Edit 工具你就不能编辑文件。

**设计原理（极其重要）：** 这是 gstack 的一个深层设计原则 — **用 allowed-tools 而不是用自然语言指令来强制行为约束。**

为什么？因为自然语言指令（"不要读源码"）是软约束 — Claude 在特定上下文压力下可能违反。但 allowed-tools 是硬约束 — Claude 物理上不能调用未授权的工具。

这等于：不是告诉保安"不要让任何人进入" — 而是**直接锁上门**。

---

## 二、完整结构拆解

### 2.1 测试方法论（共享自 /qa）

`/qa-only` 和 `/qa` 共享完全相同的测试方法论（通过模板系统的 `{{QA_METHODOLOGY}}` 占位符实现）：

- **三级测试深度：** Quick / Standard / Exhaustive
- **Diff-aware 模式：** 自动检测受影响页面（但不读源码来做依赖分析 — 只通过 URL 路由推断）
- **80 项检查清单：** 功能 + 视觉 + 性能 + 可访问性

唯一的区别：Diff-aware 在 `/qa-only` 中只能通过路由映射来推断影响范围（"改了 /api/users → 测试用户相关页面"），不能通过源码依赖分析来精确追踪。这是"不读源码"约束的代价 — 测试范围可能更宽泛（过度测试）但不会遗漏。

### 2.2 Bug 报告格式

`/qa-only` 的核心产出是一份结构化的 Bug 报告：

```markdown
# QA Report — 2026-03-15

## Summary
- Pages tested: 12
- Issues found: 7
- Critical: 1
- High: 2
- Medium: 3
- Low: 1

## ISSUE-001 [CRITICAL]
**Page:** /checkout
**Steps to reproduce:**
  1. Add item to cart
  2. Go to checkout
  3. Enter email with + sign (e.g., user+tag@example.com)
  4. Click "Pay Now"
**Expected:** Payment processes successfully
**Actual:** Form shows "Invalid email" error
**Screenshot:** [attached]
**Environment:** Chrome 120, macOS, 1440px viewport

## ISSUE-002 [HIGH]
**Page:** /dashboard
**Steps to reproduce:**
  1. Navigate to /dashboard as new user (0 data)
  2. Wait for page load
**Expected:** Empty state with onboarding prompt
**Actual:** Blank white screen, chart component throws error
**Screenshot:** [attached]
**Environment:** Chrome 120, macOS, 1440px viewport

... (more issues)
```

**每个 Bug 报告的必备要素：**

| 要素 | 为什么必须有 |
|---|---|
| 严重级别 | 让开发者知道先修哪个 |
| 精确复现步骤 | 开发者可以在 30 秒内重现 |
| 预期 vs 实际 | 消除"这是 feature 还是 bug"的争论 |
| 截图 | 视觉问题的唯一证据 |
| 环境信息 | 排除"我这边没问题"的借口 |

**设计原理：** 这个格式不是偶然的。每个要素都对应着 Bug 报告在交接过程中最常丢失的信息。传统 Bug 报告最大的问题是"无法重现" — 因为复现步骤不精确。`/qa-only` 用浏览器自动化保证了**每一步都可以精确重放**。

### 2.3 不读源码时如何做 Diff-aware？

这是 `/qa-only` 最有趣的技术挑战。`/qa` 可以读源码做依赖分析（"Button 组件被改了 → 所有使用 Button 的页面都需要测试"）。`/qa-only` 不能读源码，所以它用了一种更粗粒度但同样有效的方法：

```
Step 1: git diff --stat → 获取变更文件名列表（文件名不是源码）
Step 2: 从文件名推断影响范围
  - pages/checkout.tsx → 测试 /checkout
  - components/Button.tsx → 测试所有页面（保守策略）
  - api/users.ts → 测试用户相关功能
  - styles/global.css → 测试所有页面（保守策略）
Step 3: 生成测试计划
```

**关键设计决策：** 当不确定影响范围时，选择**过度测试而不是遗漏测试**。如果一个共享组件改了，测所有页面。这会比 `/qa` 的精确分析多花时间，但保证不遗漏。

---

## 三、/qa vs /qa-only 深度对比

| 维度 | /qa | /qa-only |
|---|---|---|
| **角色** | QA Lead（全权负责） | QA Reporter（只报告） |
| **发现 Bug 后** | 定位 → 修复 → 提交 → 验证 | 记录 → 截图 → 写报告 |
| **读源码** | 是 | 否 |
| **写代码** | 是 | 否 |
| **工具集** | 完整 | 最小 |
| **产出** | 修复后的代码 + 回归测试 | Bug 报告 |
| **交付对象** | 代码库（直接提交） | 人类开发者（读报告后自行修复） |
| **Diff-aware 精度** | 精确（源码依赖分析） | 粗粒度（文件名推断） |
| **适用团队** | 独立开发者、小团队 | 有 QA 岗位的团队、外部审计 |

**选择指南：**

```
你是唯一的开发者？          → /qa
你的团队 QA 和开发分离？     → /qa-only
你在做安全审计？            → /qa-only
你想让 AI 全自动跑完？      → /qa
你需要一份报告交给别人？     → /qa-only
```

---

## 四、用 allowed-tools 强制行为约束的设计模式

这是 `/qa-only` 贡献的最重要的可迁移设计模式。

### 自然语言约束 vs 工具约束

```markdown
# 方案 A：自然语言约束（弱）
你是一个只读审查员。不要修改任何文件。只报告问题。
allowed-tools:
  - Read
  - Edit    ← 物理上可以编辑，只靠指令阻止
  - Bash
  - Grep

# 方案 B：工具约束（强）
你是一个只读审查员。
allowed-tools:
  - Read    ← 可以读
  - Bash    ← 可以运行命令
  - Grep    ← 可以搜索
  # 没有 Edit、没有 Write → 物理上不可能修改文件
```

方案 A 依赖 Claude 的"自制力"。方案 B 依赖**架构约束**。

在安全工程中这叫"最小权限原则（Principle of Least Privilege）" — 不要给一个角色超出其职责所需的任何权限。gstack 把这个原则应用到了 AI Agent 的工具授权上。

### 如何设计你自己的"约束型 Skill"

1. **确定角色的职责边界** — 这个角色应该做什么？不应该做什么？
2. **把"不应该做什么"映射到工具** — 不应该改代码 → 去掉 Edit/Write。不应该读源码 → 去掉 Read/Grep
3. **在 allowed-tools 中只保留必需工具** — 宁可少给不多给
4. **用自然语言补充解释** — "你没有 Edit 工具，因为你的角色是报告者不是修复者"

**原文：**
> "The toolset IS the constraint. Natural language instructions are the explanation. If they conflict, the toolset wins."

**翻译：** 工具集就是约束。自然语言指令是解释。如果两者冲突，工具集赢。

---

## 五、如何迁移到你自己的场景

### 场景 1：安全审计 Skill（只报告不修复）

```markdown
---
name: security-audit
allowed-tools:
  - Bash    # 只用于运行扫描工具
  - Grep    # 搜索可疑模式
  # 没有 Edit/Write — 审计员不改代码
---

# 安全审计

你是外部安全审计员。你的任务是发现漏洞并写报告。
你没有修改代码的权限。这是故意的。

对每个发现：
- 严重级别（Critical/High/Medium/Low）
- 漏洞类型（OWASP Top 10 分类）
- 精确位置（文件:行号）
- 复现步骤
- 建议修复方案（描述，不是代码）
```

### 场景 2：性能审计 Skill（只测量不优化）

```markdown
---
name: perf-audit
allowed-tools:
  - Bash    # 运行 Lighthouse, ab, wrk 等工具
  # 没有 Read/Edit — 只测量，不看代码，不改代码
---

# 性能审计

你是性能测试工程师。测量，不优化。
产出是一份包含数据的性能报告，不是代码改动。
```

### 场景 3：用户体验走查（只体验不改进）

```markdown
---
name: ux-walkthrough
allowed-tools:
  - Bash    # 浏览器命令
---

# UX 走查

你是第一次使用这个产品的用户。
打开浏览器，按照直觉操作。
记录每一个"卡住了""不确定该点哪里""等太久了"的时刻。
不要建议修复方案。只记录体验。
```

### 关键提取物（可复用到任何"只报告不修复"的 Skill）

1. **用 allowed-tools 而不是自然语言做硬约束** — 工具集是铁门，指令是建议
2. **绝对禁令优于柔性建议** — "NEVER" 比 "try not to" 可靠 100 倍
3. **共享方法论，分离权限** — /qa 和 /qa-only 共享 `{{QA_METHODOLOGY}}`，只是工具集不同
4. **过度测试优于遗漏** — 不读源码时影响分析会更粗粒度，这是可接受的代价
5. **报告格式即交接协议** — Bug 报告的每个字段都是为了减少交接中的信息丢失

---

> 本分析基于 `/qa-only` Skill 的核心设计：黑盒测试约束、最小工具集强制、与 /qa 共享方法论的模板机制，以及"用架构而非指令强制行为"的设计原则。
