# /ship 深度解构

> **角色：** Release Engineer / 发布工程师模式
> **定位：** 全自动化发布流水线——从测试到 PR 创建，中间只在必要时停下
> **Prompt 长度：** ~609 行（gstack 最长模板） | **allowed-tools：** Read, Grep, Glob, Bash, Edit, Write, AskUserQuestion
> **来源：** `ship/SKILL.md.tmpl`

---

## 一、为什么这个 Skill 有效（即使你发布的不是代码）

/ship 是 gstack 中最长、最复杂的 Skill，但它的核心哲学可以用一句话概括：**除非遇到需要人类判断的事情，否则不要停下来。**

这个 Skill 有效的原因不是它自动化了多少步骤——而是它**精确定义了什么时候必须停下来**：

1. **基础分支确认** — 发布到错误的分支是不可逆的
2. **合并冲突** — 冲突解决需要业务判断
3. **测试失败** — 失败原因需要人类诊断
4. **ASK 审查项目** — 来自内嵌的 /review 流程
5. **MINOR/MAJOR 版本升级** — 语义化版本的公共 API 契约
6. **Greptile 决策** — 来自外部审查工具的问题
7. **TODOS.md 缺失** — 技术债务登记簿不存在时需要确认

除了这七个停止点之外，/ship **完全自主运行**。它不会问"要我运行测试吗？"——它直接运行。它不会问"要我生成 CHANGELOG 吗？"——它直接生成。

**核心洞察：** 自动化的最高形态不是"全自动"（那叫脚本），而是"在精确的时机交还控制权"。/ship 知道自己什么时候需要人类，什么时候不需要——这让它既安全又高效。

**为什么这个模式可以迁移到非代码场景：**

/ship 的底层模式是一个**带检查点的流水线**。这个模式适用于任何复杂的多步骤流程：
- 财务结算流水线：自动对账 → 自动生成报表 → 在异常金额处停下来 → 自动归档
- 内容发布流水线：自动格式化 → 自动生成缩略图 → 在法律合规检查处停下来 → 自动发布
- 招聘流水线：自动筛简历 → 自动排期 → 在候选人特殊情况处停下来 → 自动发 offer

---

## 二、完整结构拆解

### 2.1 整体架构：七阶段流水线

/ship 的执行流程分为七个大阶段，每个阶段内部又有多个子步骤。这是 gstack 中最具工程感的 Skill——它不像 /plan-ceo-review 那样是思维框架，它是一条**真正的发布流水线**。

```
阶段 1: Pre-flight（预检）
    ↓
阶段 2: Merge Base（合并基准分支）
    ↓
阶段 3: Test & Coverage（测试 + 覆盖率审计）
    ↓
阶段 4: Pre-landing Review（上线前审查）
    ↓
阶段 5: Version & CHANGELOG（版本号 + 变更日志）
    ↓
阶段 6: TODOS.md（技术债务管理）
    ↓
阶段 7: Commit & PR（提交 + 创建 PR）
```

### 2.2 阶段 1：Pre-flight（预检）

**原文：**
```
Pre-flight checks:
1. Check review dashboard for blocking issues
2. Verify clean working tree
3. Identify base branch
4. Confirm no uncommitted changes
```

**中文翻译：**
预检：
1. 检查审查仪表盘是否有阻塞问题
2. 验证工作区干净
3. 识别基础分支
4. 确认没有未提交的变更

**设计原理分析：**

Pre-flight 的第一步不是检查代码——而是**检查审查仪表盘**。这意味着 /ship 不是独立运行的，它是 gstack 工作流的下游消费者。如果之前的 /review 留下了未解决的阻塞问题，/ship 不会继续。

这种"流程间依赖"的设计确保了发布不会绕过审查。在很多团队中，"来不及审查就先发了"是事故的主要来源。/ship 通过把审查检查嵌入预检阶段，从架构上杜绝了这种绕行。

### 2.3 阶段 2：Merge Base（合并基准分支）

**原文：**
```
Merge base branch into current branch. If conflicts arise, STOP and
present conflicts to user for resolution.
```

**中文翻译：**
将基础分支合并到当前分支。如果出现冲突，停下来并向用户展示冲突以供解决。

这是七个停止点之一。合并冲突不能自动解决，因为冲突的两侧代码各自的意图需要人类判断。

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

**原文：**
```
If the diff contains prompt changes, run eval suites. Prompt changes without
eval results are not shippable.
```

**中文翻译：**
如果 diff 包含 prompt 变更，运行评估套件。没有评估结果的 prompt 变更不可发布。

**设计原理分析：**

这条规则反映了一个 AI 时代的新现实：**Prompt 变更和代码变更一样需要测试，但需要不同类型的测试。** 单元测试不能验证 prompt 的效果——你需要运行 eval suite（评估套件），用一组预定义的输入检查输出质量。

"没有评估结果的 prompt 变更不可发布"是一条硬规则，和"没有测试的代码变更不可发布"同级。这在 2024-2025 年的 AI 工程实践中是前沿性的认知。

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

### 2.5 阶段 4：Pre-landing Review（上线前审查）

**原文：**
```
Run pre-landing review using checklist and Greptile integration.
This is a condensed version of /review embedded within /ship.
```

**中文翻译：**
使用清单和 Greptile 集成运行上线前审查。这是嵌入在 /ship 中的精简版 /review。

**设计原理分析：**

/ship 没有假设你之前已经运行过 /review——它在发布流程中嵌入了一个精简版的审查。这是**防御性设计**：即使工程师跳过了 /review 直接跑 /ship，关键的安全和质量检查仍然会执行。

这也意味着如果你之前已经运行过 /review 并修复了所有问题，这个阶段会快速通过。它不会重复之前的工作——它检查当前代码状态。

### 2.6 阶段 5：版本号 + CHANGELOG

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

**原文：**
```
Auto-generate CHANGELOG from commit messages and PR descriptions.
Group changes by: Added, Changed, Fixed, Removed, Security.
```

**中文翻译：**
从提交信息和 PR 描述自动生成 CHANGELOG。按以下分组：新增、变更、修复、移除、安全。

**设计原理分析：**

自动生成 CHANGELOG 解决了一个普遍问题：**没有人喜欢手写 CHANGELOG**，所以大多数项目的 CHANGELOG 要么不存在，要么严重过期。通过自动化这个步骤，/ship 确保每次发布都有准确的变更日志。

分组方式遵循 [Keep a Changelog](https://keepachangelog.com/) 的标准格式。这不是随意选择——它是行业共识，意味着生成的 CHANGELOG 对任何开发者都是可读的。

### 2.7 阶段 6：TODOS.md 管理

**原文：**
```
Check TODOS.md. If it doesn't exist, STOP and ask if one should be created.
Cross-reference completed TODO items with this release.
Mark completed items. Flag items that should have been completed but weren't.
```

**中文翻译：**
检查 TODOS.md。如果不存在，停下来询问是否应该创建。将已完成的 TODO 项目与本次发布交叉参考。标记已完成的项目。标记应该已完成但未完成的项目。

**设计原理分析：**

TODOS.md 在 gstack 中扮演着"技术债务登记簿"的角色。/ship 在这里做的事情是**维护系统记忆**——确保团队承诺要做的事情不会被遗忘。

"如果不存在就停下来询问"是一个有趣的设计选择。它没有自动创建——因为 TODOS.md 的存在与否反映了团队的工作方式。自动创建一个空的 TODOS.md 没有意义；询问用户给了团队选择的权利。

### 2.8 阶段 7：Bisectable Commits + Push + PR

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
Only stop for: base branch confirmation, merge conflicts, test failures,
ASK review items, MINOR/MAJOR version bumps, Greptile decisions,
missing TODOS.md.
Everything else: just do it.
```

**中文翻译：**
只在以下情况停下：基础分支确认、合并冲突、测试失败、ASK 审查项目、MINOR/MAJOR 版本升级、Greptile 决策、TODOS.md 缺失。其他所有事情：直接做。

"Everything else: just do it" 这五个词是整个 Skill 的灵魂。它把默认行为从"等待许可"翻转为"主动执行"。

### 2.10 流水线中各阶段的关系图

```
Pre-flight ──→ 有阻塞问题？ ──→ [停止] 解决后重新开始
    │
    ↓ 无阻塞
Merge Base ──→ 有冲突？ ──→ [停止] 用户解决冲突
    │
    ↓ 无冲突
Test Bootstrap → Run Tests ──→ 失败？ ──→ [停止] 展示失败
    │                                        上下文
    ↓ 通过
Eval Suites ──→ (仅 prompt 变更)
    │
    ↓
Coverage Audit ──→ ☆☆☆ 缺口？ ──→ 自动生成测试
    │
    ↓
Pre-landing Review ──→ ASK 项目？ ──→ [停止] 批量展示
    │
    ↓
Version Bump ──→ MINOR/MAJOR？ ──→ [停止] 询问用户
    │
    ↓ PATCH/MICRO 自动
CHANGELOG 自动生成
    │
    ↓
TODOS.md ──→ 不存在？ ──→ [停止] 询问是否创建
    │
    ↓
Bisectable Commits → Push → Create PR ──→ 完成
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
2. **七个停止点原则**：只在"需要人类判断"的地方停下。判断标准是"结果是否可预测"——可预测就自动化，不可预测就询问
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
