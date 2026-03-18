# /qa-only 深度解构

> **角色：** QA Engineer — 质量工程师（仅报告模式）
> **定位：** 与 /qa 共享 `{{QA_METHODOLOGY}}` 测试方法论，但绝不修复 Bug、绝不读源码
> **核心机制：** 报告导向、自然语言行为约束（非工具约束）、Bug 报告移交
> **allowed-tools：** Bash, Read, Write, AskUserQuestion
> **来源：** `qa-only/SKILL.md.tmpl`（v1.0.0）

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

**原文（模板头部和 Rule 11）：**
> "You are a QA engineer. Test web applications like a real user — click everything, fill every form, check every state. Produce a structured report with evidence. **NEVER fix anything.**"
>
> Rule 11: "Never fix bugs. Find and document only. Do not read source code, edit files, or suggest fixes in the report. Your job is to report what's broken, not to fix it. Use `/qa` for the test-fix-verify loop."

**翻译：** 你是一名 QA 工程师。像真实用户一样测试 Web 应用——点击所有东西、填写所有表单、检查所有状态。生成带证据的结构化报告。**绝不修复任何东西。**

规则 11：绝不修复 Bug。只发现和记录。不要读源码、编辑文件或在报告中建议修复方案。你的工作是报告什么坏了，而不是修它。要修复的话用 `/qa`。

### 三个关键约束

**1. NEVER fixes bugs — 绝不修复**

这不是"尽量不修" — 这是**绝对禁令**。即使 AI 明确知道怎么修，它也只报告问题。

为什么要如此绝对？因为"尽量不修"会给 AI 留下解释空间。Claude 很擅长自我说服："这个修复只是加了一行 CSS，应该没问题吧？" 一旦开了这个口子，行为就不可预测了。绝对禁令消除了所有灰色地带。

**2. Do not read source code — 不读源码**

这是一个**自然语言约束**，而非工具约束。Rule 11 明确说"Do not read source code"——但 `/qa-only` 的 allowed-tools 中**实际上包含 Read 工具**。

**为什么？** 因为 `/qa-only` 需要 Read 来读取配置文件、测试计划等非源码文件。"不读源码"是通过自然语言指令而非工具限制来约束的。

纯黑盒测试的价值在于：测试者不依赖实现细节，所以它会尝试所有它能想到的输入组合 — 包括开发者认为"不可能发生"的那些。

**3. 工具集对比**

| /qa 的工具 | /qa-only 的工具 |
|---|---|
| Bash, Read, Write, Edit, Glob, Grep, AskUserQuestion, WebSearch | Bash, Read, Write, AskUserQuestion |

**与 /qa 的区别：** `/qa-only` 缺少 Edit、Glob、Grep、WebSearch。没有 Edit 意味着不能编辑代码文件。没有 Grep/Glob 限制了源码搜索能力。但 Read 和 Write 是存在的——Read 用于读取配置和报告文件，Write 用于生成报告。

**设计原理分析：** `/qa-only` 的约束设计是**混合模式**：部分通过工具限制（去掉 Edit/Grep/Glob 限制了代码修改和搜索能力），部分通过自然语言指令（Rule 11 的 "Do not read source code" 约束了 Read 工具的使用范围）。这不是纯粹的"锁门"策略，而是"锁了部分门 + 立了规矩"的策略。

---

## 二、完整结构拆解

### 2.1 测试方法论（共享自 /qa）

`/qa-only` 和 `/qa` 共享完全相同的测试方法论（通过模板系统的 `{{QA_METHODOLOGY}}` 占位符实现）：

- **共享 QA 方法论：** 通过 `{{QA_METHODOLOGY}}` 模板变量引入（具体内容在共享模板中）
- **Diff-aware 模式：** 自动检测受影响页面（但不读源码来做依赖分析 — 只通过 URL 路由推断）

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
| **工具集** | Bash, Read, Write, Edit, Glob, Grep, AskUserQuestion, WebSearch | Bash, Read, Write, AskUserQuestion |
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

## 四、行为约束的混合设计模式

`/qa-only` 实际采用的是**混合约束策略**，而非纯工具约束。

### 三种约束层级

```markdown
# 层级 1：工具约束（硬约束）
去掉 Edit → 物理上不能修改文件内容
去掉 Grep/Glob → 限制了源码搜索能力

# 层级 2：自然语言约束（软约束）
Rule 11: "Do not read source code" → 有 Read 工具但被指令约束使用范围
Rule 11: "Do not suggest fixes" → 报告中不包含修复建议

# 层级 3：角色定义约束（上下文约束）
"You are a QA engineer" → 角色暗示了行为边界
"NEVER fix anything" → 加粗强调的行为禁令
```

`/qa-only` 的实际设计表明 gstack 并非总是采用纯工具约束。当一个工具（如 Read）有正当用途（读配置、读测试计划）但也有被滥用的可能（读源码），自然语言指令是务实的补充。

### 如何设计你自己的"约束型 Skill"

1. **确定角色的职责边界** — 这个角色应该做什么？不应该做什么？
2. **优先用工具约束实现硬性禁令** — 不应该改代码 → 去掉 Edit。这是最可靠的
3. **对需要部分权限的工具，用自然语言限制使用范围** — 保留 Read 但指令说"不读源码"
4. **在 allowed-tools 中只保留必需工具** — 宁可少给不多给
5. **角色定义强化约束** — "你是报告者不是修复者"提供行为上下文

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

1. **优先用 allowed-tools 做硬约束，自然语言做补充** — 工具限制是最可靠的，但当工具有正当多用途时，自然语言指令是务实的补充
2. **绝对禁令优于柔性建议** — "NEVER" 比 "try not to" 可靠 100 倍
3. **共享方法论，分离权限** — /qa 和 /qa-only 共享 `{{QA_METHODOLOGY}}`，只是工具集不同
4. **过度测试优于遗漏** — 不读源码时影响分析会更粗粒度，这是可接受的代价
5. **报告格式即交接协议** — Bug 报告的每个字段都是为了减少交接中的信息丢失

---

> 本分析基于 `/qa-only` Skill 的核心设计：黑盒测试约束、最小工具集强制、与 /qa 共享方法论的模板机制，以及"用架构而非指令强制行为"的设计原则。
