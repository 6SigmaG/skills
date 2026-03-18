# /review 深度解构

> **角色：** Staff Engineer / 高级审查模式
> **定位：** Pre-landing PR 审查，两轮扫描 + 自动修复机械问题
> **Prompt 长度：** ~195 行（不含模板变量展开） | **allowed-tools：** Bash, Read, Edit, Write, Grep, Glob, AskUserQuestion
> **来源：** `review/SKILL.md.tmpl`

---

## 一、为什么这个 Skill 有效（即使你审查的不是 Pull Request）

这个 Skill 最反直觉的设计决策是：**它不是只读的审查工具，它会动手修代码。**

传统的代码审查工具——无论是人类的还是 AI 的——都遵循"看了说，说了等"的模式：发现问题 → 留评论 → 等作者修。/review 打破了这个模式，引入了 **Fix-First 原则**：能机械修复的（死代码、N+1、过期注释），直接改；需要判断的（安全、竞态条件），才停下来问。

这个设计有效的三个原因：

1. **消除评论往返的摩擦** — 传统审查中 60%+ 的评论是机械性的（"这行死代码删掉"、"这个注释过期了"）。Fix-First 直接消灭了这些低价值往返
2. **两轮扫描建立了优先级层级** — 不是把所有问题混在一起，而是先 CRITICAL（能导致事故的）、再 INFORMATIONAL（能改善质量的）。审查者的注意力被正确分配
3. **跨 diff 追踪让 enum 审查真正有效** — 它不只看 diff 里的代码，它追踪 enum 值穿过整个代码库的所有 switch 语句和白名单。这是人类审查者最容易遗漏的地方

**核心洞察：** 最好的代码审查不是找到最多问题的，而是**在正确的时机用正确的方式处理问题的**。机械问题就该机械修复，判断问题才值得讨论。

---

## 二、完整结构拆解

### 2.1 角色定义与姿态

**原文：**
```
You are running the /review workflow. Analyze the current branch's diff against
the base branch for structural issues that tests don't catch.
```

**中文翻译：**
你正在运行 /review 工作流。分析当前分支相对于基础分支的 diff，找出测试无法捕获的结构性问题。

**设计原理分析：**

注意这段姿态定义的克制——它没有赋予 Claude 一个华丽的头衔（如 "Staff Engineer"），而是直接定义了**任务边界**："structural issues that tests don't catch"（测试无法捕获的结构性问题）。这条指令精确地告诉 Claude：你的价值不在于重复测试已经做的事，而在于发现测试盲区。

结合 checklist.md 中的 Suppressions 部分（"DO NOT flag" 清单），/review 通过**负面约束**来控制噪音——明确列出不应该标记的问题类型（如无害的冗余、阈值变更、已在 diff 中解决的问题等）。这种"告诉 AI 什么不重要"的策略，和告诉它什么重要同样关键——甚至更关键，因为 AI 的默认倾向是面面俱到。

**迁移价值：** 任何审查类 Skill 都应该明确说"你不管什么"。Suppressions 清单是一种比泛泛说"不要挑剔风格"更精确的实现方式。

### 2.2 两轮扫描系统（Two-Pass Review）

这是 /review 的核心架构。所有问题被分为两个严格分离的层级。

**Pass 1 — CRITICAL（关键问题）**

**原文（来自 checklist.md）：**
```
Pass 1 (CRITICAL): SQL & Data Safety, Race Conditions & Concurrency,
LLM Output Trust Boundary, Enum & Value Completeness
```

具体而言，checklist.md 对四个 CRITICAL 类别展开如下：
- **SQL & Data Safety**：字符串插值 SQL、TOCTOU 竞态、绕过验证的 update_column、N+1 查询
- **Race Conditions & Concurrency**：无唯一约束的 read-check-write、无唯一索引的 find_or_create_by、非原子状态转换、html_safe 用于用户控制数据（XSS）
- **LLM Output Trust Boundary**：LLM 生成的值（邮件、URL、名称）未经格式验证就写入数据库；结构化工具输出未经类型/形状检查就写入数据库
- **Enum & Value Completeness**：新 enum 值引入后追踪所有消费者（READ CODE OUTSIDE THE DIFF）

**中文翻译：**
第一轮——关键问题：
- SQL 与数据安全：SQL 中的字符串插值、TOCTOU 竞态（应为原子 WHERE + update_all）、绕过验证的 update_column、N+1 查询
- 竞态条件与并发：无唯一约束的读取-检查-写入模式、无唯一数据库索引的 find_or_create_by、非原子状态转换、html_safe 用于用户控制数据
- LLM 输出信任边界：LLM 生成值未经验证写入数据库或传给 mailer；结构化工具输出未经类型检查写入数据库
- Enum 与值完整性：新 enum 值/状态字符串/层级名称/类型常量引入后，追踪每个消费者（读取 diff 之外的代码）

**设计原理分析：**

这四个 CRITICAL 类别不是随意选择的，它们精确对应了**最常导致生产事故的四类问题**：

| 类别 | 对应的事故类型 | 为什么人类审查者容易遗漏 |
|---|---|---|
| SQL 安全 | 数据泄露、注入攻击 | 参数化在大多数框架中是默认的，但 ORM 的 raw query escape hatch 容易被忽视 |
| 竞态条件 | 数据不一致、重复扣款 | 单线程思维——审查者脑中跑的是单请求流程，不是并发场景 |
| LLM 信任边界 | XSS、提示注入、代码执行 | 这是新兴的攻击面，大多数审查清单还没有覆盖 |
| Enum 完整性 | 未处理的分支、静默忽略 | diff 里看不到——新 enum 值加了，但受影响的 switch 语句在其他文件里 |

特别值得注意的是 **LLM 信任边界**。这是一个极其前瞻性的审查维度。大多数代码审查工具（包括人类的审查清单）还没有把 LLM 输出视为不可信输入。但 /review 明确把它列为 CRITICAL——因为在 AI 时代，LLM 输出就是新的 "用户输入"，需要同等级别的清洗和验证。

**Enum 追踪的跨 diff 设计：**

**原文：**
```
Enum completeness: new enum value added but switch/case statements elsewhere
in the codebase not updated (READ CODE OUTSIDE THE DIFF)
```

**中文翻译：**
Enum 完整性：添加了新的 enum 值但代码库其他地方的 switch/case 语句没有更新（读取 diff 之外的代码）

这条指令中的括号 `(READ CODE OUTSIDE THE DIFF)` 是整个 Skill 最关键的设计决策之一。它显式告诉 Claude：**不要只看改了什么，要追踪改动的涟漪效应**。

在实际操作中，Claude 会这样执行：
1. 在 diff 中发现新的 enum 值
2. 用 Grep 搜索整个代码库中引用这个 enum 类型的 switch/case 语句
3. 检查每个 switch 是否处理了新值
4. 没处理的标记为 CRITICAL

这是人类审查者最容易遗漏的问题类型——因为 GitHub 的 diff 视图只展示改动的文件，审查者看不到其他文件中未更新的 switch 语句。

**Pass 2 — INFORMATIONAL（信息性问题）**

**原文（来自 checklist.md）：**
```
Pass 2 (INFORMATIONAL): Conditional Side Effects, Magic Numbers & String
Coupling, Dead Code & Consistency, LLM Prompt Issues, Test Gaps,
Completeness Gaps, Crypto & Entropy, Time Window Safety,
Type Coercion at Boundaries, View/Frontend
```

**中文翻译：**
第二轮——信息性问题。质量改进（共 10 个类别）：
- 条件副作用：分支中遗忘的副作用，日志与实际行为不符
- 魔法数字与字符串耦合：裸数字字面量应命名为常量，错误消息字符串被其他地方用作查询过滤器
- 死代码与一致性：赋值未读变量、版本不匹配、CHANGELOG 描述不准确、过期注释
- LLM Prompt 问题：Prompt 中 0 索引列表（LLM 会返回 1 索引）、Prompt 声明的工具与实际不匹配、多处声明的 token 限制可能漂移
- 测试缺口：负路径测试只断言类型不断言副作用、缺少 `.expects(:something).never`、安全特性缺少端到端测试
- 完整性缺口：快捷实现本可在 30 分钟内完成完整版、测试覆盖缺口属于"lake 不是 ocean"级别
- 加密与熵：截断数据而非哈希、rand() 用于安全场景而非 SecureRandom、非恒定时间比较
- 时间窗口安全：日期键查找假设"今天"覆盖 24 小时、相关功能间时间窗口不匹配
- 类型强制转换边界：跨 Ruby→JSON→JS 边界时类型可能变化、哈希输入未调用 .to_s
- 视图/前端：partial 中的内联 style 块、视图中 O(n*m) 查找、Ruby 端 .select{} 本可为 WHERE 子句

**设计原理分析：**

Pass 2 和 Pass 1 的分离不仅是优先级排序——它是一种**认知负载管理**。

如果把安全问题和代码风格问题混在一起，审查者（无论人类还是 AI）会陷入"注意力平均分配"的陷阱。一个 SQL 注入和一个魔法数字得到相同的权重。两轮扫描确保 Claude 先用 100% 的注意力处理安全问题，然后再切换到质量问题。

**迁移价值：** 任何审查类 Skill 都应该把问题分成至少两个层级。即使不是代码审查——比如审查合同、审查设计稿——"致命问题"和"改善问题"的分离都是有效的。

### 2.3 Fix-First 模式

**原文（来自 checklist.md 的 Fix-First Heuristic）：**
```
AUTO-FIX (agent fixes without asking):     ASK (needs human judgment):
├─ Dead code / unused variables            ├─ Security (auth, XSS, injection)
├─ N+1 queries (missing .includes())      ├─ Race conditions
├─ Stale comments contradicting code       ├─ Design decisions
├─ Magic numbers → named constants         ├─ Large fixes (>20 lines)
├─ Missing LLM output validation           ├─ Enum completeness
├─ Version/path mismatches                 ├─ Removing functionality
├─ Variables assigned but never read       └─ Anything changing user-visible
└─ Inline styles, O(n*m) view lookups        behavior
```

**中文翻译：**
修复优先启发式：
- 自动修复（Agent 直接修，不问）：死代码/未使用变量、N+1 查询、与代码矛盾的过期注释、魔法数字改为命名常量、缺失的 LLM 输出验证、版本/路径不匹配、赋值未读变量、内联样式和 O(n*m) 视图查找
- 询问（需要人类判断）：安全问题（认证、XSS、注入）、竞态条件、设计决策、大型修复（>20 行）、Enum 完整性、删除功能、任何改变用户可见行为的修复

**设计原理分析：**

Fix-First 模式的精妙在于它的**分类标准**：不是按严重性分（严重的问也不一定需要问），而是按**是否需要人类判断**分。

| 类型 | 为什么自动修复 | 为什么询问 |
|---|---|---|
| 死代码 | 删除死代码永远是正确的 | — |
| N+1 查询 | 修复方式通常是机械的（eager load） | — |
| 过期注释 | 删除或更新，没有判断空间 | — |
| — | — | 安全边界：可能涉及业务权衡 |
| — | — | 竞态条件：修复方式有多种，各有取舍 |
| — | — | API 契约变更：影响下游消费者 |

这个分类暗含了一个深层原则：**如果一个问题只有一个合理的解决方案，就不需要讨论。讨论本身是一种浪费。** 只有在存在多个合理选择、需要权衡取舍时，才需要人类参与。

**ASK 的批量呈现：**

**原文：**
```
Batch all ASK items together. Present them as a numbered list with your
recommendation for each. Don't interleave ASK items with AUTO-FIX actions.
```

**中文翻译：**
把所有 ASK 项目批量呈现。作为编号列表呈现，每项附带你的建议。不要把 ASK 项目和自动修复动作交错混合。

这个设计避免了"审查疲劳"——如果 Claude 每发现一个问题就停下来问，用户会在第三个问题时开始无脑点"同意"。批量呈现让用户可以一次性审视所有需要判断的问题，做出更一致的决策。

### 2.4 Greptile 集成与四类分诊

**原文（来自 SKILL.md.tmpl Step 5 Greptile 部分）：**
```
Before replying to any comment, run the Escalation Detection algorithm from
greptile-triage.md to determine whether to use Tier 1 (friendly) or Tier 2
(firm) reply templates.

1. VALID & ACTIONABLE comments: included in findings — follow Fix-First flow
2. FALSE POSITIVE comments: Present via AskUserQuestion with options
3. VALID BUT ALREADY FIXED comments: Reply using Already Fixed template — no
   AskUserQuestion needed
4. SUPPRESSED comments: Skip silently
```

**中文翻译：**
在回复任何评论之前，运行 greptile-triage.md 中的升级检测算法，决定使用第一级（友好）还是第二级（坚定）回复模板。

四种分类及处理方式：
1. 有效且可操作——纳入审查发现，走 Fix-First 流程
2. 假阳性——用 AskUserQuestion 让用户决定（回复 Greptile / 顺手修 / 忽略）
3. 有效但已修复——用"已修复"模板自动回复，不需要问用户
4. 已抑制——静默跳过（来自之前分诊的已知假阳性）

**设计原理分析：**

Greptile 是一个 AI 代码审查工具，它会在 PR 上自动留下评论。/review 的设计不是忽略这些评论或与之竞争，而是**对其进行分诊（triage）**。

四类分诊 + 两级回复模板解决了一个真实的团队动态问题：当 AI 审查工具（Greptile）留下了不准确或过度谨慎的评论时，如何回复？直接忽略会让团队不信任工具；逐条反驳又太耗时。模板提供了预制的回复方式，并且每条回复都要求包含证据（内联 diff、代码引用、重新排序建议）。

特别值得注意的是"VALID BUT ALREADY FIXED"类别——不需要问用户，自动回复并附上修复的 commit SHA。这是**消除冗余交互**的设计：如果 Claude 已经在 Fix-First 阶段修复了 Greptile 指出的问题，就没必要再问用户怎么处理。

**迁移价值：** 这个模式适用于任何需要和其他审查工具（或审查者）协作的场景。关键是：不要覆盖别人的评论，而是对其分诊和补充。用分类系统替代逐条处理，用模板替代即兴回复。

### 2.5 设计审查精简版（Design Review Lite）

源模板中，Step 4.5 通过 `{{DESIGN_REVIEW_LITE}}` 模板变量引入精简版设计审查。这个模板变量的具体内容在另一个共享模板文件中定义，此处不可见。但模板说明了关键规则：

**原文：**
```
Include any design findings alongside the findings from Step 4. They follow the
same Fix-First flow in Step 5 — AUTO-FIX for mechanical CSS fixes, ASK for
everything else.
```

**中文翻译：**
将设计发现与 Step 4 的发现一起包含。它们遵循相同的 Fix-First 流程——机械性 CSS 修复自动执行，其他一切则询问。

**设计原理分析：**

设计审查精简版的巧妙之处在于它**不是一个独立的设计审查工具**，而是嵌入在代码审查流程中的。当 diff 中检测到前端文件变更时，它自动激活。发现的设计问题和代码问题统一进入 Fix-First 流程，不需要单独的处理管道。

这解决了一个常见的组织问题：设计审查和代码审查是分离的流程，前端代码经常在两个流程之间的缝隙中"漏网"——设计师审查了 mockup，工程师审查了逻辑，但没人检查**实现是否匹配设计意图的细节**。

### 2.6 完整执行流程

/review 的执行流程是严格有序的：

**Step 1：检查分支状态**
```
1. Run `git branch --show-current` to get the current branch.
2. If on the base branch, output: "Nothing to review — you're on the base branch
   or have no changes against it." and stop.
3. Run `git fetch origin <base> --quiet && git diff origin/<base> --stat` to check
   if there's a diff. If no diff, output the same message and stop.
```
获取当前分支，如果在基础分支上或没有 diff 则停止。

**Step 2：读取审查清单**
```
Read `.claude/skills/review/checklist.md`.
If the file cannot be read, STOP and report the error. Do not proceed
without the checklist.
```
读取 `.claude/skills/review/checklist.md` 获取审查标准。如果无法读取，停下来报告错误，不继续。

注意这里的**硬性依赖设计**——清单不是可选的，没有清单就不审查。这确保了审查标准的一致性，同时让标准可配置。不同项目有不同的审查重点——金融项目关注精度、医疗项目关注隐私、游戏项目关注性能。通过读取项目级别的清单文件，/review 可以适配任何项目。

**Step 2.5：检查 Greptile**（注意：源模板中编号为 Step 2.5，不是 Step 3）
```
Read `.claude/skills/review/greptile-triage.md` and follow the fetch, filter,
classify, and escalation detection steps.
If no PR exists, `gh` fails, API returns an error, or there are zero Greptile
comments: Skip this step silently. Greptile integration is additive — the
review works without it.
```
读取 greptile-triage.md 并执行分诊流程。如果没有 PR、gh 失败、API 报错或零评论——静默跳过。Greptile 集成是**附加功能**，审查不依赖它。

**Step 3：获取完整 diff**（源模板中为 Step 3）

**原文：**
```
Fetch the latest base branch to avoid false positives from stale local state:
git fetch origin <base> --quiet
Run `git diff origin/<base>` to get the full diff. This includes both committed
and uncommitted changes against the latest base branch.
```

**中文翻译：**
拉取最新基础分支以避免过期本地状态导致的假阳性。运行 `git diff origin/<base>` 获取完整 diff，包括已提交和未提交的变更。

**设计原理分析：**

这条规则看似简单，实际上是防止一类非常具体的 AI 错误：**过早结论**。

Claude（和所有 LLM）有一个倾向——读到第一个文件的变更后就开始形成观点，然后用确认偏误去解释后续文件。强制"读完再说"打破了这个倾向。

这也是为什么 /review 能发现跨文件问题（如 enum 完整性）——因为它在开始分析之前已经看到了所有文件的变更。

**Step 4：两轮审查**（源模板中为 Step 4）
执行 Pass 1（CRITICAL）和 Pass 2（INFORMATIONAL），如上文所述。

**Step 4.5：设计审查（条件触发）**
当 diff 包含前端文件时，自动运行 `{{DESIGN_REVIEW_LITE}}` 模板内容。发现的设计问题和代码问题一起进入 Fix-First 流程。

**Step 5：Fix-First 执行**（源模板中为 Step 5，含子步骤 5a-5d）

源模板把这个步骤拆成了四个精确的子步骤：
- **Step 5a**: 分类每个发现为 AUTO-FIX 或 ASK
- **Step 5b**: 自动修复所有 AUTO-FIX 项目，每个输出一行：`[AUTO-FIXED] [file:line] Problem → what you did`
- **Step 5c**: 将所有 ASK 项目批量呈现在一个 AskUserQuestion 中（3 个以下可以单独问）
- **Step 5d**: 执行用户批准的修复

关键细节：如果没有 ASK 项目（全部是 AUTO-FIX），完全跳过提问。

**Greptile 评论处理**也嵌入在 Step 5 中，按四种分类分别处理：VALID & ACTIONABLE（进入 Fix-First 流程）、VALID BUT ALREADY FIXED（自动回复，不需问用户）、FALSE POSITIVE（让用户决定是否回复 Greptile）、SUPPRESSED（静默跳过）。

**Step 5.5：TODOS 交叉参考**

**原文：**
```
Read `TODOS.md` in the repository root (if it exists). Cross-reference the PR
against open TODOs:
- Does this PR close any open TODOs?
- Does this PR create work that should become a TODO?
- Are there related TODOs that provide context for this review?
If TODOS.md doesn't exist, skip this step silently.
```

**中文翻译：**
读取仓库根目录的 TODOS.md（如果存在）。将 PR 与待办项交叉参考：
- 这个 PR 是否关闭了任何待办项？
- 这个 PR 是否创建了应该成为待办项的工作？
- 是否有相关待办项为本次审查提供上下文？
如果 TODOS.md 不存在，静默跳过。

**设计原理分析：**

这一步连接了 /review 和更大的项目管理生态。TODOS.md 是 gstack 中的"技术债务登记簿"。交叉参考确保：
1. PR 没有"忘记"做已承诺的工作
2. PR 做的新工作和已知的技术债务不冲突
3. 审查者有完整的上下文——不只是"这个 PR 做了什么"，还有"这个 PR 在整体计划中处于什么位置"

注意"如果不存在就静默跳过"——这与 /ship 中"不存在就停下来询问"形成对比。/review 是审查工具，TODOS.md 是可选上下文；/ship 是发布工具，TODOS.md 是推荐的项目管理实践。

**Step 5.6：文档过期检查**

**原文：**
```
Cross-reference the diff against documentation files. For each .md file in the
repo root (README.md, ARCHITECTURE.md, CONTRIBUTING.md, CLAUDE.md, etc.):
1. Check if code changes affect features described in that doc file.
2. If the doc was NOT updated but the code it describes WAS changed, flag as
   INFORMATIONAL: "Documentation may be stale... Consider running /document-release."
This is informational only — never critical.
```
检查文档相对于代码变更是否过期。这只是信息性问题，不是关键问题。建议的修复方式是运行 `/document-release`。

### 2.7 关键运行规则

**原文（Important Rules 节）：**
```
- Read the FULL diff before commenting. Do not flag issues already addressed
  in the diff.
- Fix-first, not read-only. AUTO-FIX items are applied directly. ASK items are
  only applied after user approval. Never commit, push, or create PRs — that's
  /ship's job.
- Be terse. One line problem, one line fix. No preamble.
- Only flag real problems. Skip anything that's fine.
- Use Greptile reply templates from greptile-triage.md. Every reply includes
  evidence. Never post vague replies.
```

**中文翻译：**
规则：
1. 评论前读完整 diff。不要标记 diff 中已经解决的问题
2. 修复优先，不是只读。AUTO-FIX 直接执行，ASK 只在用户批准后执行。**绝不提交、推送或创建 PR——那是 /ship 的工作**
3. 简洁。一行描述问题，一行描述修复。不要开场白
4. 只标记真实问题。没问题的就跳过
5. 使用 greptile-triage.md 中的回复模板。每条回复都包含证据。绝不发空泛回复

**设计原理分析：**

规则 3 和 4 共同解决了 AI 代码审查中最大的问题：**噪音**。规则 2 中"绝不提交/推送/创建 PR"则划清了 /review 和 /ship 的职责边界——/review 是审查工具，/ship 才是发布工具。

没有这些约束的 AI 审查工具会生成这样的评论：
> "这个函数看起来不错！不过我注意到你可能想考虑一下这里是否有潜在的性能影响，虽然我不完全确定在当前上下文中这是否真的是一个问题..."

有了规则 3 和 4，输出变成：
> "N+1 query on line 42. Fixed: added `.includes(:user)` to the scope."

前者浪费审查者的时间，后者直接解决问题。

**规则 4 特别值得深入分析：** "如果你不确定它是 bug，它就不是 bug" 是一条**高置信度过滤器**。它要求 Claude 只在确信时才报告。这会让 Claude 漏掉一些真实问题吗？会。但它显著减少了假阳性——而在代码审查中，假阳性的成本（审查疲劳、信任下降）远高于假阴性的成本（偶尔漏掉一个非关键问题）。

### 2.8 审查清单参考文件

/review 依赖两个外部参考文件：

| 文件 | 用途 |
|---|---|
| `review/checklist.md` | 项目特定的审查标准——每个项目可以定制自己的重点 |
| `review/greptile-triage.md` | Greptile 评论的分诊标准——什么级别的评论需要什么级别的回应 |

**设计原理分析：**

将审查标准外置到文件中（而非硬编码在 Prompt 里）是一个关键的可维护性决策。标准会随项目演化——新的安全威胁出现、新的框架引入、新的团队规范确立。通过外置文件，更新标准不需要修改 Skill 本身。

---

## 三、如何迁移到你自己的场景

### 场景 1：非代码文档审查

/review 的两轮扫描 + Fix-First 模式可以直接迁移到文档审查：

```markdown
# 文档审查 Skill

## Pass 1 — CRITICAL
- 事实错误：和代码/API 不一致的描述
- 缺失的警告：危险操作没有标注
- 版本不匹配：文档描述的和当前版本行为不同

## Pass 2 — INFORMATIONAL
- 过期截图
- 拼写/语法错误
- 缺少示例代码

## Fix-First
- AUTO-FIX：拼写、格式、明显过期的版本号
- ASK：内容重组、删除整个章节、修改 API 描述
```

### 场景 2：设计稿审查

```markdown
# 设计审查 Skill

## 姿态
你是一位有品味且务实的设计审查者。只标记真实问题，不要为了显得
仔细而挑毛病。

## Pass 1 — CRITICAL
- 可访问性违规：对比度不足、无键盘导航、缺少 alt 文本
- 状态缺失：没有空状态、错误状态、加载状态的设计
- 信息架构矛盾：导航结构和用户心智模型不匹配

## Pass 2 — INFORMATIONAL
- 间距不一致
- 颜色使用不在设计系统规范内
- 缺少动效说明
```

### 场景 3：安全审查专项

如果你的项目对安全要求更高，可以扩展 Pass 1：

```markdown
## Pass 1 — CRITICAL（安全专项版）
- 所有原始 CRITICAL 项目
- 认证绕过：任何绕过认证中间件的路径
- 权限升级：普通用户能触达管理员功能吗
- 密钥泄露：硬编码的 token、密码、API key
- 日志中的 PII：个人信息写入了日志吗
- SSRF：用户输入被用于构造内部 URL 吗
```

### 关键提取物（可复用到任何审查类 Skill）

1. **两轮扫描分离**：致命问题和改善问题不要混在一起。认知负载管理是审查质量的关键
2. **Fix-First 不是可选的**：如果你的审查 Skill 只输出评论不做修复，你在浪费 AI 的动手能力。分类标准是"是否需要人类判断"，不是"是否严重"
3. **读完再说**：强制 AI 在阅读全部上下文后再输出。过早结论是 LLM 审查中最常见的错误模式
4. **高置信度过滤**：宁可漏掉可疑问题，也不要报告假阳性。审查疲劳是真实威胁
5. **跨 diff 追踪**：显式指令 AI 用 Grep/Glob 搜索改动的涟漪效应——这是 AI 审查者相对于人类审查者最大的优势
6. **外置审查标准**：把标准放在可编辑的文件里，不要硬编码在 Prompt 中
7. **与其他工具协作**：不要忽略其他审查工具的输出，对其分诊和补充
8. **简洁是尊重**：代码审查评论中的客套话不是礼貌，是噪音。直接说问题，直接给方案

---

> 本分析基于 `review/SKILL.md.tmpl` 完整源码逐段拆解，结合 `review/checklist.md` 和 `review/greptile-triage.md` 的参考文件设计。
