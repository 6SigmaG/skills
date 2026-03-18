# /review 深度解构

> **角色：** Staff Engineer / 高级审查模式
> **定位：** Pre-landing PR 审查，两轮扫描 + 自动修复机械问题
> **Prompt 长度：** ~400 行 | **allowed-tools：** Read, Grep, Glob, Bash, Edit, Write
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
You are a Staff Engineer performing a pre-landing review. Your job is to catch
real problems before they hit production. You are not here to nitpick style —
you are here to prevent incidents.
```

**中文翻译：**
你是一位 Staff Engineer，正在做上线前审查。你的工作是在问题进入生产环境之前把它们抓住。你不是来挑剔代码风格的——你是来防止事故的。

**设计原理分析：**

这段姿态定义做了两件精准的事：

第一，它选择了 "Staff Engineer" 而不是 "Senior Engineer" 或 "Tech Lead"。在工程组织里，Staff Engineer 的特殊之处在于：他们不拥有项目，不管人，但他们有**跨团队的技术判断权**。这个角色定义告诉 Claude：你的视野应该超越单个 PR——要考虑系统级影响。

第二，"not here to nitpick style" 是一条**负面约束**。没有这条，Claude 会生成大量格式、命名、空行之类的低价值评论。这条指令让 Claude 把注意力集中在真正重要的问题上。

**迁移价值：** 任何审查类 Skill 都应该明确说"你不管什么"。告诉 AI 什么不重要，和告诉它什么重要同样关键——甚至更关键，因为 AI 的默认倾向是面面俱到。

### 2.2 两轮扫描系统（Two-Pass Review）

这是 /review 的核心架构。所有问题被分为两个严格分离的层级。

**Pass 1 — CRITICAL（关键问题）**

**原文：**
```
Pass 1 — CRITICAL. Stop-ship issues only:
- SQL safety: injection vectors, missing parameterization, raw string queries
- Race conditions: TOCTOU, double-submit, concurrent writes without locks
- LLM trust boundary: unsanitized model output used in SQL/HTML/shell/eval
- Enum completeness: new enum value added but switch/case statements elsewhere
  in the codebase not updated (READ CODE OUTSIDE THE DIFF)
```

**中文翻译：**
第一轮——关键问题。仅限阻止上线的问题：
- SQL 安全：注入向量、缺少参数化、原始字符串查询
- 竞态条件：TOCTOU（检查时间/使用时间）、重复提交、没有锁的并发写入
- LLM 信任边界：未清洗的模型输出被用在 SQL/HTML/shell/eval 中
- Enum 完整性：添加了新的 enum 值但代码库其他地方的 switch/case 语句没有更新（读取 diff 之外的代码）

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

**原文：**
```
Pass 2 — INFORMATIONAL. Quality improvements:
- Conditional side effects: mutations hidden inside boolean expressions
- Magic numbers: unexplained literals that should be named constants
- Dead code: unreachable branches, unused imports, commented-out blocks
- Test gaps: new code paths without corresponding test coverage
- View/frontend: accessibility issues, missing loading/error states
```

**中文翻译：**
第二轮——信息性问题。质量改进：
- 条件副作用：隐藏在布尔表达式中的状态变更
- 魔法数字：应该命名为常量的未解释字面量
- 死代码：不可达分支、未使用的导入、被注释掉的代码块
- 测试缺口：新代码路径没有对应的测试覆盖
- 视图/前端：可访问性问题、缺少加载/错误状态

**设计原理分析：**

Pass 2 和 Pass 1 的分离不仅是优先级排序——它是一种**认知负载管理**。

如果把安全问题和代码风格问题混在一起，审查者（无论人类还是 AI）会陷入"注意力平均分配"的陷阱。一个 SQL 注入和一个魔法数字得到相同的权重。两轮扫描确保 Claude 先用 100% 的注意力处理安全问题，然后再切换到质量问题。

**迁移价值：** 任何审查类 Skill 都应该把问题分成至少两个层级。即使不是代码审查——比如审查合同、审查设计稿——"致命问题"和"改善问题"的分离都是有效的。

### 2.3 Fix-First 模式

**原文：**
```
Fix-First pattern:
- AUTO-FIX: mechanical issues (dead code, N+1 queries, stale comments,
  unused imports, obvious typos). Just fix them. Don't ask.
- ASK: judgment calls (security boundaries, race condition mitigations,
  architectural changes, API contract changes). Present the problem,
  your recommendation, and ask for approval.
```

**中文翻译：**
修复优先模式：
- 自动修复：机械性问题（死代码、N+1 查询、过期注释、未使用的导入、明显的拼写错误）。直接修。不要问。
- 询问：判断性决策（安全边界、竞态条件缓解、架构变更、API 契约变更）。呈现问题、你的建议，然后请求批准。

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

### 2.4 Greptile 集成与两级升级

**原文：**
```
Check Greptile for existing review comments. Two-tier escalation:
- Tier 1 (friendly): "Thanks for flagging this! Here's what I found..."
- Tier 2 (firm): "This needs to be addressed before merge. The issue is..."
```

**中文翻译：**
检查 Greptile 的现有审查评论。两级升级：
- 第一级（友好）："感谢指出这个！这是我发现的..."
- 第二级（坚定）："这个在合并前必须解决。问题是..."

**设计原理分析：**

Greptile 是一个 AI 代码审查工具，它会在 PR 上自动留下评论。/review 的设计不是忽略这些评论或与之竞争，而是**对其进行分诊（triage）**。

两级回复模板解决了一个真实的团队动态问题：当 AI 审查工具（Greptile）留下了不准确或过度谨慎的评论时，如何回复？直接忽略会让团队不信任工具；逐条反驳又太耗时。两级模板提供了预制的回复方式：

- **第一级**用于 Greptile 的评论有道理但 Claude 已经处理了的情况——承认贡献，展示解决方案
- **第二级**用于 Greptile 发现了 Claude 也认为严重的问题——强化严重性，确保不被忽略

**迁移价值：** 这个模式适用于任何需要和其他审查工具（或审查者）协作的场景。关键是：不要覆盖别人的评论，而是对其分诊和补充。在引用 `review/greptile-triage.md` 作为参考时，Claude 得到了具体的分诊标准。

### 2.5 设计审查精简版（Design Review Lite）

**原文：**
```
For frontend changes, run a 20-item design checklist covering:
loading states, error states, empty states, responsive breakpoints,
accessibility (ARIA, keyboard nav, screen reader), animation performance,
color contrast, touch targets, RTL support...
```

**中文翻译：**
对于前端变更，运行一个 20 项设计清单，覆盖：加载状态、错误状态、空状态、响应式断点、可访问性（ARIA、键盘导航、屏幕阅读器）、动画性能、颜色对比度、触摸目标、RTL 支持...

**设计原理分析：**

这个 20 项清单的巧妙之处在于它**不是一个独立的设计审查工具**，而是嵌入在代码审查流程中的。当 diff 中检测到前端文件变更时，它自动激活。

这解决了一个常见的组织问题：设计审查和代码审查是分离的流程，前端代码经常在两个流程之间的缝隙中"漏网"——设计师审查了 mockup，工程师审查了逻辑，但没人检查**实现是否匹配设计意图的细节**。

### 2.6 完整执行流程

/review 的执行流程是严格有序的：

**Step 1：检查分支状态**
```
Check current branch. Identify base branch. Ensure working tree is clean.
```
确认分支状态，识别目标分支。确保工作区干净。

**Step 2：读取审查清单**
```
Read review/checklist.md for project-specific review criteria.
```
读取 `review/checklist.md` 获取项目特定的审查标准。

这一步的设计意图是**让审查标准可配置**。不同项目有不同的审查重点——金融项目关注精度、医疗项目关注隐私、游戏项目关注性能。通过读取项目级别的清单文件，/review 可以适配任何项目。

**Step 3：检查 Greptile**
```
Check Greptile for existing review comments. Triage using
review/greptile-triage.md criteria.
```
检查 Greptile 的现有评论。使用 `review/greptile-triage.md` 的标准进行分诊。

**Step 4：获取完整 diff**

**原文：**
```
Read the FULL diff before making any comments. Do not start commenting
after reading partial changes.
```

**中文翻译：**
在做任何评论之前，阅读完整的 diff。不要在读了部分变更后就开始评论。

**设计原理分析：**

这条规则看似简单，实际上是防止一类非常具体的 AI 错误：**过早结论**。

Claude（和所有 LLM）有一个倾向——读到第一个文件的变更后就开始形成观点，然后用确认偏误去解释后续文件。强制"读完再说"打破了这个倾向。

这也是为什么 /review 能发现跨文件问题（如 enum 完整性）——因为它在开始分析之前已经看到了所有文件的变更。

**Step 5：两轮审查**
执行 Pass 1（CRITICAL）和 Pass 2（INFORMATIONAL），如上文所述。

**Step 6：Fix-First 执行**
```
AUTO-FIX mechanical issues. Batch ASK items. Present for approval.
```
自动修复机械问题。批量呈现询问项目。等待批准。

**Step 7：TODOS 交叉参考**

**原文：**
```
Cross-reference with TODOS.md. Flag any TODO items that this PR should
have addressed but didn't.
```

**中文翻译：**
与 TODOS.md 交叉参考。标记这个 PR 应该处理但没有处理的 TODO 项目。

**设计原理分析：**

这一步连接了 /review 和更大的项目管理生态。TODOS.md 是 gstack 中的"技术债务登记簿"。交叉参考确保：
1. PR 没有"忘记"做已承诺的工作
2. PR 做的新工作和已知的技术债务不冲突
3. 审查者有完整的上下文——不只是"这个 PR 做了什么"，还有"这个 PR 在整体计划中处于什么位置"

**Step 8：文档过期检查**
```
Check if documentation is stale relative to the changes in this PR.
```
检查文档相对于这个 PR 的变更是否过期。

### 2.7 关键运行规则

**原文：**
```
Rules:
1. Read FULL diff before commenting
2. Fix-first, not read-only
3. Be terse — no filler words, no pleasantries in code comments
4. Only flag real problems — if you're not sure it's a bug, it's not a bug
```

**中文翻译：**
规则：
1. 评论前读完整 diff
2. 修复优先，不是只读
3. 简洁——代码评论中不要填充词、不要客套话
4. 只标记真实问题——如果你不确定它是 bug，它就不是 bug

**设计原理分析：**

规则 3 和 4 共同解决了 AI 代码审查中最大的问题：**噪音**。

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
