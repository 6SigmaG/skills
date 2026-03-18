# /ship 深度解构

> **角色：** Release Engineer / 发布工程师模式
> **定位：** 全自动化发布流水线——从测试到 PR 创建，中间只在必要时停下
> **Prompt 长度：** ~609 行（gstack 最长模板） | **allowed-tools：** Bash, Read, Write, Edit, Grep, Glob, AskUserQuestion, WebSearch
> **来源：** `ship/SKILL.md.tmpl`

---

## 一、为什么这个 Skill 有效（即使你发布的不是代码）

/ship 是 gstack 中最长、最复杂的 Skill，但它的核心哲学可以用一句话概括：**除非遇到需要人类判断的事情，否则不要停下来。**

这个 Skill 有效的原因不是它自动化了多少步骤——而是它**精确定义了什么时候必须停下来**：

1. **在基础分支上** — 直接中止，不是发布分支
2. **合并冲突** — 简单冲突（VERSION、schema.rb、CHANGELOG 排序）自动解决；复杂冲突停下来
3. **测试失败** — 失败原因需要人类诊断
4. **ASK 审查项目** — 来自内嵌的 /review 流程
5. **MINOR/MAJOR 版本升级** — 语义化版本的公共 API 契约
6. **Greptile 决策** — 来自外部审查工具的复杂修复或假阳性
7. **TODOS.md 缺失** — 技术债务登记簿不存在时需要确认是否创建
8. **TODOS.md 结构混乱** — 不符合推荐结构时询问是否重组

除了这八个停止点之外，/ship **完全自主运行**。它不会问"要我运行测试吗？"——它直接运行。它不会问"要我生成 CHANGELOG 吗？"——它直接生成。

**核心洞察：** 自动化的最高形态不是"全自动"（那叫脚本），而是"在精确的时机交还控制权"。/ship 知道自己什么时候需要人类，什么时候不需要——这让它既安全又高效。

**为什么这个模式可以迁移到非代码场景：**

/ship 的底层模式是一个**带检查点的流水线**。这个模式适用于任何复杂的多步骤流程：
- 财务结算流水线：自动对账 → 自动生成报表 → 在异常金额处停下来 → 自动归档
- 内容发布流水线：自动格式化 → 自动生成缩略图 → 在法律合规检查处停下来 → 自动发布
- 招聘流水线：自动筛简历 → 自动排期 → 在候选人特殊情况处停下来 → 自动发 offer

---

## 二、完整结构拆解

### 2.1 整体架构：八阶段流水线

/ship 的执行流程分为八个大步骤（源模板编号为 Steps 1-8，含多个子步骤如 2.5、3.25、3.4、3.5、3.75、5.5）。这是 gstack 中最具工程感的 Skill——它不像 /plan-ceo-review 那样是思维框架，它是一条**真正的发布流水线**。

```
Step 1: Pre-flight（预检 + 审查仪表盘）
    ↓
Step 2: Merge Base（合并基准分支）
  Step 2.5: Test Bootstrap（测试框架引导）
    ↓
Step 3: Run Tests（运行测试）
  Step 3.25: Eval Suites（条件：仅 prompt 变更）
  Step 3.4: Test Coverage Audit（测试覆盖率审计）
  Step 3.5: Pre-Landing Review（上线前审查）
  Step 3.75: Greptile Review（条件：PR 存在时）
    ↓
Step 4: Version Bump（版本号）
    ↓
Step 5: CHANGELOG（变更日志）
  Step 5.5: TODOS.md（技术债务管理）
    ↓
Step 6: Commit（可二分提交）
    ↓
Step 7: Push（推送）
    ↓
Step 8: Create PR（创建 PR）
```

### 2.2 阶段 1：Pre-flight（预检）

**原文（Step 1: Pre-flight）：**
```
1. Check the current branch. If on the base branch or the repo's default branch,
   abort: "You're on the base branch. Ship from a feature branch."
2. Run `git status` (never use `-uall`). Uncommitted changes are always
   included — no need to ask.
3. Run `git diff <base>...HEAD --stat` and `git log <base>..HEAD --oneline`
   to understand what's being shipped.
4. Check review readiness: {{REVIEW_DASHBOARD}}
```

**中文翻译：**
预检：
1. 检查当前分支。如果在基础分支上，中止
2. 运行 git status。未提交的变更总是被包含——不需要问
3. 运行 git diff 和 git log 了解要发布的内容
4. 检查审查就绪度（通过审查仪表盘）

**设计原理分析：**

两个关键设计决策值得注意：

第一，**未提交的变更总是被包含**——不停下来问"要先提交吗？"。这是非交互哲学的体现。

第二，Step 4 的**审查仪表盘检查**确保发布不会绕过审查。如果工程审查（Eng Review）未通过，/ship 会给用户三个选择：A）直接发布，B）中止去跑 /plan-eng-review，C）变更太小不需要审查。选择 A 或 C 会被持久化，同分支后续 /ship 不会再问。

### 2.3 阶段 2：Merge Base（合并基准分支）

**原文（Step 2）：**
```
git fetch origin <base> && git merge origin/<base> --no-edit

If there are merge conflicts: Try to auto-resolve if they are simple
(VERSION, schema.rb, CHANGELOG ordering). If conflicts are complex or
ambiguous, STOP and show them.

If already up to date: Continue silently.
```

**中文翻译：**
拉取并合并基础分支。如果有冲突：简单冲突（VERSION、schema.rb、CHANGELOG 排序）尝试自动解决。复杂或模糊的冲突停下来展示。

注意：这不是一刀切地"有冲突就停"。简单冲突（如版本号冲突——两边都递增了版本号）有明确的解决方式，可以自动化。只有需要业务判断的冲突才停下来。

### 2.4 阶段 3：测试 + 覆盖率审计（最复杂的阶段）

这是整个 /ship Skill 中最精心设计的阶段，也是它区别于简单 CI 脚本的地方。

**Step 3.1：Test Bootstrap**

**原文：**
```
Bootstrap test environment. Install dependencies. Verify test infrastructure
is operational before running any tests.
```

**中文翻译：**
引导测试环境。安装依赖。在运行任何测试之前验证测试基础设施可用。

**设计原理：** 一个看似简单但极其重要的步骤。多少次 "测试失败" 其实是 "环境没装好"？先验证基础设施，再运行测试，避免了大量虚假失败。

**Step 3.2：运行测试**

**原文：**
```
Run full test suite. If tests fail, STOP. Present failures with context.
Do not attempt to fix failing tests automatically.
```

**中文翻译：**
运行完整测试套件。如果测试失败，停下来。展示失败及其上下文。不要尝试自动修复失败的测试。

**设计原理分析：**

注意最后一句：**不要自动修复失败的测试**。这与 /review 的 Fix-First 模式形成了有趣的对比。在 /review 中，死代码、N+1 查询这些机械问题可以自动修复。但测试失败不行——因为测试失败可能意味着：
1. 代码有 bug（应该修代码，不是修测试）
2. 测试本身过期了（需要判断新行为是否正确）
3. 测试基础设施有问题（不是代码问题）

这三种情况需要完全不同的处理方式，自动修复会掩盖真实问题。

**Step 3.3：Eval Suites（仅 Prompt 变更时）**

**原文（Step 3.25: Eval Suites）：**
```
Evals are mandatory when prompt-related files change. Skip this step entirely
if no prompt files are in the diff.

Check if the diff touches prompt-related files:
- app/services/*_prompt_builder.rb
- app/services/*_generation_service.rb, *_writer_service.rb, *_designer_service.rb
- config/system_prompts/*.txt
- test/evals/**/*
... (more patterns)

/ship is a pre-merge gate, so always use full tier (Sonnet structural +
Opus persona judges).
```

**中文翻译：**
当 prompt 相关文件变更时，eval 是强制性的。如果 diff 中没有 prompt 文件则完全跳过。/ship 作为合并前的门控，总是使用 full tier（Sonnet 结构化 + Opus 角色评委）。

**设计原理分析：**

这个步骤的精细度值得注意——它不是简单地"有 prompt 变更就跑全部 eval"，而是：
1. 用文件路径模式精确识别 prompt 相关文件
2. 通过 eval runner 中声明的 `PROMPT_SOURCE_FILES` 找到受影响的 eval 套件
3. 顺序运行受影响的套件（不并行，因为每个需要 test lane）
4. 第一个失败就停止——不浪费 API 费用

三级 eval tier（fast/standard/full）的存在表明 eval 在开发过程中也被使用（fast tier 用于迭代，约 $0.07/次），但 /ship 强制使用 full tier（约 $1.27/次）。这在 AI 工程实践中是前沿性的认知——prompt 变更和代码变更一样需要测试，且发布前需要最严格的测试。

**Step 3.4：测试覆盖率审计（最精心的子步骤）**

这个子步骤是 /ship 中最独特的部分——它不只是看覆盖率数字，它**画图**。

**原文：**
```
Test coverage audit:
1. Trace every code path introduced or modified in this PR
2. Draw ASCII diagrams of execution paths
3. Map user flows to code paths
4. Check coverage with quality scoring:
   ★★★ = happy path + edge cases + error paths tested
   ★★☆ = happy path + some edge cases tested
   ★☆☆ = only happy path tested
   ☆☆☆ = no tests
5. Auto-generate tests for gaps where possible
```

**中文翻译：**
测试覆盖率审计：
1. 追踪这个 PR 中引入或修改的每一条代码路径
2. 画出执行路径的 ASCII 图
3. 将用户流映射到代码路径
4. 用质量评分检查覆盖率：
   ★★★ = 正常路径 + 边界情况 + 错误路径都有测试
   ★★☆ = 正常路径 + 部分边界情况有测试
   ★☆☆ = 只有正常路径测试
   ☆☆☆ = 没有测试
5. 在可能的情况下自动生成测试以填补缺口

**设计原理分析：**

这个设计的每一层都值得拆解：

**为什么画 ASCII 图？**

覆盖率数字（如 "87% 覆盖率"）是一个**欺骗性指标**。87% 可能意味着所有正常路径都测了但没有一条错误路径被测试。ASCII 图迫使 Claude 显式展示每条路径，让人类审查者可以一眼看到"哦，这条错误处理路径完全没有覆盖"。

```
示例 ASCII 图：
[用户提交表单] → [验证输入] → ★★★ 正常路径
                      ↓
              [验证失败] → ★★☆ 部分边界测试
                      ↓
              [服务器错误] → ☆☆☆ 无测试  ← 缺口！
```

**为什么用星级评分而不是百分比？**

百分比给人一种虚假的精确感。"83% 覆盖率"和"85% 覆盖率"的区别在实践中毫无意义。星级评分（★★★/★★☆/★☆☆/☆☆☆）提供了一个**粗粒度但更有用的信号**——它告诉你的是"这条路径的测试深度如何"，而不是"测试了百分之几的行数"。

这四个级别也对应了四种不同的行动：
- ★★★ = 不需要操心
- ★★☆ = 可以接受，但如果有时间应该补充
- ★☆☆ = 风险较高，应该在下一个迭代补充
- ☆☆☆ = 不可接受，必须现在补充

**自动生成测试：**

当发现覆盖率缺口时，/ship 不只是报告——它会**尝试自动生成测试**。这和 /review 的 Fix-First 模式一脉相承：能自动解决的就自动解决。生成的测试可能不完美，但它们提供了一个起点，比"请补充测试"这样的空洞评论有用得多。

**迁移价值：** 覆盖率审计的模式可以迁移到非代码场景。比如文档覆盖率审计：
- ★★★ = 有教程 + API 参考 + 故障排除指南
- ★★☆ = 有 API 参考 + 基本描述
- ★☆☆ = 只有自动生成的 API 参考
- ☆☆☆ = 无文档

### 2.5 Step 3.5：Pre-landing Review（上线前审查）

**原文（Step 3.5）：**
```
Review the diff for structural issues that tests don't catch.
1. Read `.claude/skills/review/checklist.md`. If unreadable, STOP.
2. Run `git diff origin/<base>` for full diff.
3. Apply checklist in two passes:
   Pass 1 (CRITICAL): SQL & Data Safety, LLM Output Trust Boundary
   Pass 2 (INFORMATIONAL): All remaining categories
4. Classify each finding as AUTO-FIX or ASK per Fix-First Heuristic.
5. Auto-fix all AUTO-FIX items.
6. If ASK items remain, present in ONE AskUserQuestion.
7. After all fixes: if ANY fixes applied, commit and STOP — tell user to
   run /ship again to re-test.
```

**中文翻译：**
审查 diff 中测试无法捕获的结构性问题。使用 checklist.md 两轮审查。自动修复机械问题。如果有修复，提交后停止——让用户重新运行 /ship 以重新测试。

**设计原理分析：**

/ship 没有假设你之前已经运行过 /review——它在发布流程中嵌入了一个精简版的审查。这是**防御性设计**：即使工程师跳过了 /review 直接跑 /ship，关键的安全和质量检查仍然会执行。

这也意味着如果你之前已经运行过 /review 并修复了所有问题，这个阶段会快速通过。它不会重复之前的工作——它检查当前代码状态。

### 2.6 Step 4 & 5：版本号 + CHANGELOG

**版本号格式：**

**原文：**
```
4-digit version format: MAJOR.MINOR.PATCH.MICRO
- PATCH and MICRO: auto-increment, no user input needed
- MINOR: ask user — indicates new features, backward compatible
- MAJOR: ask user — indicates breaking changes
```

**中文翻译：**
4 位版本格式：MAJOR.MINOR.PATCH.MICRO
- PATCH 和 MICRO：自动递增，不需要用户输入
- MINOR：询问用户——表示新功能，向后兼容
- MAJOR：询问用户——表示破坏性变更

**设计原理分析：**

标准语义化版本是三位（MAJOR.MINOR.PATCH）。/ship 使用四位，增加了 MICRO。这第四位的作用是**在不影响语义化版本含义的情况下区分不同的构建/部署**。

自动递增 PATCH 和 MICRO、但询问 MINOR 和 MAJOR 的设计精确地反映了这两类变更的性质：
- PATCH/MICRO 的增加不改变公共 API 契约——安全自动化
- MINOR/MAJOR 的增加改变了用户的期望——需要人类确认

**CHANGELOG 自动生成：**

**原文（Step 5）：**
```
Auto-generate the entry from ALL commits on the branch (not just recent ones).
Categorize changes into applicable sections:
  - ### Added — new features
  - ### Changed — changes to existing functionality
  - ### Fixed — bug fixes
  - ### Removed — removed features
Format: ## [X.Y.Z.W] - YYYY-MM-DD
```

**中文翻译：**
从分支上所有 commit 自动生成条目（不只是最近的）。按以下分组：Added（新增）、Changed（变更）、Fixed（修复）、Removed（移除）。格式：`## [X.Y.Z.W] - YYYY-MM-DD`。

**设计原理分析：**

自动生成 CHANGELOG 解决了一个普遍问题：**没有人喜欢手写 CHANGELOG**，所以大多数项目的 CHANGELOG 要么不存在，要么严重过期。通过自动化这个步骤，/ship 确保每次发布都有准确的变更日志。

分组方式遵循 [Keep a Changelog](https://keepachangelog.com/) 的标准格式。这不是随意选择——它是行业共识，意味着生成的 CHANGELOG 对任何开发者都是可读的。

### 2.7 Step 5.5：TODOS.md 管理

**原文（Step 5.5 摘要）：**
```
1. Check if TODOS.md exists. If not: AskUserQuestion (Create/Skip).
2. Check structure — if disorganized: AskUserQuestion (Reorganize/Leave).
3. Detect completed TODOs (fully automatic — match commits/diff against items).
   Be conservative: only mark if clear evidence in diff.
4. Move completed items to ## Completed section.
5. Output summary.
6. Defensive: if write fails, warn and continue. Never stop ship for TODOS failure.
```

**中文翻译：**
1. 检查 TODOS.md 是否存在。不存在则询问（创建/跳过）
2. 检查结构——不符合推荐结构则询问（重组/保持原样）
3. 自动检测已完成的 TODO（全自动，无用户交互）。保守策略：只在 diff 中有明确证据时才标记完成
4. 将已完成项目移到 Completed 区
5. 输出摘要
6. 防御性设计：如果写入失败，警告并继续。绝不因 TODOS 失败而停止发布

**设计原理分析：**

TODOS.md 在 gstack 中扮演着"技术债务登记簿"的角色。Step 5.5 的设计体现了几个原则：

- **分离自动和交互**：检测完成项是全自动的，但创建文件和重组结构需要询问
- **保守标记**：只在有明确证据时才标记完成，避免误报
- **防御性设计**：TODOS 是增值功能，不是核心功能——它的失败不应阻塞发布

### 2.8 Steps 6-8：Bisectable Commits + Push + PR

**Bisectable Commit 策略：**

**原文：**
```
Bisectable commit strategy — commit in this order:
1. Infrastructure changes (configs, dependencies, migrations)
2. Models and services (business logic)
3. Controllers and views (routing, presentation)
4. VERSION, CHANGELOG, TODOS.md (metadata)
```

**中文翻译：**
可二分提交策略——按以下顺序提交：
1. 基础设施变更（配置、依赖、迁移）
2. 模型和服务（业务逻辑）
3. 控制器和视图（路由、展示）
4. VERSION、CHANGELOG、TODOS.md（元数据）

**设计原理分析：**

这个提交顺序不是随意的——它遵循了**依赖图的拓扑排序**：每一层只依赖它下面的层。

为什么这很重要？因为 `git bisect`。当生产环境出现 bug 时，`git bisect` 通过二分搜索找到引入 bug 的提交。但如果提交不是 bisectable 的（即某些中间提交处于"半成品"状态，测试无法通过），`git bisect` 就会失效。

按依赖图顺序提交确保了每个中间状态都是一个**可编译、可测试、可运行的状态**。这让 `git bisect` 和 `git revert` 都能正常工作。

这四层顺序的另一个好处是**回滚粒度**。如果发布后发现视图层有 bug，你可以只回滚第三层提交，而不影响基础设施和业务逻辑。

**迁移价值：** 即使你不用 `git bisect`，分层提交也是一种好习惯。在非代码场景中，这等价于"按依赖关系排序发布步骤"——先发布底层变更，再发布上层变更，确保每一步都可以独立回滚。

**Push + PR 创建：**

**原文：**
```
Push to remote. Create PR with:
- Title: concise description of changes
- Body: auto-generated from CHANGELOG entries
- Labels: auto-applied based on change type
```

**中文翻译：**
推送到远程。创建 PR，包含：
- 标题：变更的简洁描述
- 正文：从 CHANGELOG 条目自动生成
- 标签：根据变更类型自动应用

### 2.9 非交互设计哲学

/ship 最重要的设计哲学是**非交互为默认**。让我们对比两种设计：

**传统 CI 工具的方式：**
```
要运行测试吗？ [Y/n]
要更新版本号吗？ [Y/n]
要生成 CHANGELOG 吗？ [Y/n]
选择版本类型：[patch/minor/major]
要推送吗？ [Y/n]
要创建 PR 吗？ [Y/n]
```

**/ship 的方式：**
```
[自动] 运行测试 ✓
[自动] 运行覆盖率审计 ✓
[自动] 运行上线前审查 ✓
[停止] 这个 PR 包含 API 变更。版本应该是 MINOR 还是 MAJOR？
[自动] 生成 CHANGELOG ✓
[自动] 分层提交 ✓
[自动] 推送 + 创建 PR ✓
```

**设计原理分析：**

传统方式的每个问题都在**消耗用户的决策能量**。而大多数问题的答案都是"是"——你当然要运行测试，你当然要生成 CHANGELOG。每个不必要的确认都是对用户时间和注意力的浪费。

/ship 的哲学是：**如果一个步骤的结果是可预测的，就不要问。只在结果不可预测时才停下来。** 这七个停止点的共同特征是：Claude 无法代替人类做出决策——因为决策涉及业务上下文、风险承受能力、或冲突解决策略。

**原文关键指令：**
```
You are running the `/ship` workflow. This is a non-interactive, fully
automated workflow. Do NOT ask for confirmation at any step. The user said
`/ship` which means DO IT. Run straight through and output the PR URL at the end.

Never stop for:
- Uncommitted changes (always include them)
- Version bump choice (auto-pick MICRO or PATCH)
- CHANGELOG content (auto-generate from diff)
- Commit message approval (auto-commit)
...

The goal is: user says `/ship`, next thing they see is the review + PR URL.
```

**中文翻译：**
你正在运行 /ship 工作流。这是一个非交互式、全自动化的工作流。任何步骤都不要请求确认。用户说了 /ship 就意味着去做。直接运行到底，最后输出 PR URL。

目标是：用户说 `/ship`，下一个看到的就是审查结果 + PR URL。

这段描述是整个 Skill 的灵魂。它把默认行为从"等待许可"翻转为"主动执行"——只有那八个明确列出的停止点才允许打断流程。

### 2.10 流水线中各阶段的关系图

```
Step 1: Pre-flight ──→ 在基础分支？ ──→ [中止]
    │               ──→ 审查未通过？ ──→ [停止] 用户选择继续/中止
    ↓
Step 2: Merge Base ──→ 复杂冲突？ ──→ [停止] 用户解决
    │                ──→ 简单冲突？ ──→ 自动解决
    ↓
Step 3: Run Tests ──→ 失败？ ──→ [停止] 展示失败
    │
    ↓ 通过
Step 3.25: Eval Suites ──→ (仅 prompt 变更时)
    │
    ↓
Step 3.4: Coverage Audit ──→ 自动生成测试
    │
    ↓
Step 3.5: Pre-landing Review ──→ ASK 项目？ ──→ [停止] 批量展示
    │                          ──→ 有修复？ ──→ [停止] 让用户重跑 /ship
    ↓
Step 3.75: Greptile ──→ 需要决策？ ──→ [停止] 询问用户
    │
    ↓
Step 4: Version Bump ──→ MINOR/MAJOR？ ──→ [停止] 询问用户
    │                 ──→ PATCH/MICRO → 自动
    ↓
Step 5: CHANGELOG 自动生成
    │
    ↓
Step 5.5: TODOS.md ──→ 不存在？ ──→ [停止] 询问是否创建
    │               ──→ 结构混乱？ ──→ [停止] 询问是否重组
    ↓
Steps 6-8: Bisectable Commits → Push → Create PR ──→ 输出 PR URL
```

---

## 三、如何迁移到你自己的场景

### 场景 1：非代码项目的发布流程

/ship 的"带检查点的流水线"模式可以直接迁移：

```markdown
# 内容发布 Skill

## 非交互默认
只在以下情况停下：
- 法律合规检查发现问题
- 与已发布内容的冲突
- 需要编辑审批的变更类型

## 流水线
1. [自动] 格式检查 + 修复
2. [自动] 链接验证
3. [自动] SEO 元数据生成
4. [条件停止] 法律合规扫描
5. [自动] 生成缩略图 + 社交媒体预览
6. [自动] 版本标记
7. [自动] 发布 + 通知相关团队
```

### 场景 2：简化版代码发布

如果你不需要 /ship 的全部复杂度，可以提取核心模式：

```markdown
# 精简发布 Skill

## 规则
1. 非交互为默认——只在测试失败和版本选择时停下
2. 测试通过是硬性要求，不可跳过
3. 分层提交：底层先于上层
4. CHANGELOG 自动生成

## 流程
1. 合并基础分支
2. 运行测试（失败则停止）
3. 自动递增版本号（破坏性变更则询问）
4. 生成 CHANGELOG
5. 提交、推送、创建 PR
```

### 场景 3：覆盖率审计独立使用

/ship 中的覆盖率审计模式可以独立提取为一个审查 Skill：

```markdown
# 覆盖率审计 Skill

## 步骤
1. 追踪 PR 中每条新的/修改的代码路径
2. 画 ASCII 执行路径图
3. 评分每条路径：
   ★★★ = 完整测试（正常 + 边界 + 错误）
   ★★☆ = 部分测试（正常 + 部分边界）
   ★☆☆ = 最小测试（仅正常路径）
   ☆☆☆ = 无测试
4. 为 ☆☆☆ 路径自动生成测试骨架
5. 为 ★☆☆ 路径建议额外测试用例
```

### 关键提取物（可复用到任何自动化流水线 Skill）

1. **非交互为默认**：列出精确的停止点清单。清单之外的一切都自动执行。"Everything else: just do it" 应该是每个自动化 Skill 的座右铭
2. **八个停止点原则**：只在"需要人类判断"的地方停下。判断标准是"结果是否可预测"——可预测就自动化，不可预测就询问
3. **测试覆盖率不等于覆盖率数字**：用 ASCII 图展示路径，用星级评分替代百分比。粗粒度但更有用的信号
4. **Bisectable commits**：按依赖图拓扑排序提交。每个中间状态都应该是可编译、可测试、可回滚的
5. **嵌入式审查**：不要假设上游流程已经执行。在关键流程中嵌入精简版的审查步骤作为安全网
6. **版本自动化的正确边界**：机械递增可以自动化，语义决策（这是新功能还是破坏性变更？）不能
7. **Prompt 变更 = 代码变更**：prompt 的修改需要 eval suite，就像代码修改需要单元测试一样
8. **CHANGELOG 作为自动副产品**：如果 CHANGELOG 需要手动维护，它就会过期。让它成为发布流程的自动输出
9. **技术债务登记簿**：TODOS.md 的交叉参考确保承诺不被遗忘——这是系统级记忆的实现
10. **防御性流水线设计**：Pre-flight 检查上游状态。不要信任人类会按正确顺序执行流程——在流程中用代码强制执行

---

> 本分析基于 `ship/SKILL.md.tmpl` 完整源码（~609 行）逐段拆解。/ship 是 gstack 中最长的模板，其复杂度反映了软件发布本身的复杂度——它不是在做简单的事情，而是在把复杂的事情变成可靠的自动化流程。
