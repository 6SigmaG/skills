# /plan-design-review 深度解构

> **角色：** Senior Designer / 资深设计师模式
> **定位：** 设计质量审查、0-10 评分与修复循环、AI Slop 检测、交互状态完整性
> **Prompt 长度：** ~450 行 | **allowed-tools：** Read, Grep, Glob, Bash, Edit, Write, AskUserQuestion
> **来源：** `plan-design-review/SKILL.md.tmpl`

---

## 一、为什么这个 Skill 有效

大多数设计审查工具做的事情是：列出一堆问题，然后交给你自己去修。`/plan-design-review` 做的完全不同——**它直接改你的计划，改完重新评分，直到每个维度都达到 8 分以上。**

这个 Skill 有效的核心原因有三个：

### 第一：它激活的是设计大师的审美直觉，不是设计规范手册

Prompt 中引用了 12 个认知模式，来自 Dieter Rams、Don Norman、Jony Ive、Julie Zhuo、Joe Gebbia 等设计领域最有影响力的思想家。当 Claude 读到 Rams 的名字时，它训练数据中关于"少即是多"哲学的数百万字文本被激活。它不是在应用一条规则——它是在**以 Rams 的审美标准审视设计**。

> **原文：** "These are not design rules. They are taste instincts — the aesthetic reflexes that separate great designers from competent ones. Let them shape every evaluation. Don't cite them; embody them."
>
> **翻译：** 这些不是设计规则。它们是品味直觉——把伟大设计师和合格设计师区分开的审美反射。让它们塑造每次评估。不要引用它们；体现它们。

### 第二：它用量化评分把"品味"变成可操作的流程

设计审查中最大的问题是主观性——"我觉得不好"没有行动力。`/plan-design-review` 用 0-10 分的评分系统，加上"差距 → 修复 → 重新评分"的闭环，把主观判断变成了**可追踪、可验证的改进过程**。

### 第三：它直接编辑计划文件，而不是生成一份独立的审查报告

这是与 `/plan-ceo-review` 和 `/plan-eng-review` 最大的区别。那两个 Skill 只读不写——它们生成审查意见，由你决定是否修改。但 `/plan-design-review` 拿到 Edit 和 Write 权限，**直接在计划文件上改**。

为什么？因为设计反馈如果不立即落地，就会被遗忘。"把间距改成 16px"如果只写在审查报告里，执行者可能漏掉。但如果直接改在计划文件里，它就成了执行规范的一部分。

**这是 edit-in-place workflow（就地编辑工作流）的核心价值：审查即改进，不是审查加改进。**

---

## 二、完整结构拆解

### 2.1 认知模式（12 个设计思维激活）

| # | 模式名 | 来源 | 原文核心 | 中文含义 | 应用场景 |
|---|---|---|---|---|---|
| 1 | 减法为默认 | Rams | "Good design is as little design as possible. Less, but better." | 好的设计是尽可能少的设计。更少，但更好 | 每看到一个 UI 元素都问：删掉它会怎样？ |
| 2 | 时间维度设计 | Norman | "Design for the 5-second glance, the 5-minute task, and the 5-year relationship." | 为 5 秒扫视、5 分钟任务、5 年关系而设计 | 用户旅程评估时的三层时间框架 |
| 3 | 细节执念 | Ive | "True simplicity is derived from so much more than just the absence of clutter." | 真正的简洁远不止是没有杂乱 | 质疑表面的简洁是否隐藏了复杂性 |
| 4 | 有原则的品味 | Zhuo | "Good taste is not subjective. It's having clear principles and applying them consistently." | 好的品味不是主观的。它是拥有清晰的原则并一致地应用它们 | 评分时要求给出原则依据，不是"我觉得" |
| 5 | 为信任而设计 | Gebbia | "Every design decision either builds or erodes trust." | 每个设计决策要么建立信任，要么侵蚀信任 | 审查每个交互是否让用户感到安全 |
| 6 | 故事板旅程 | Gebbia | "Storyboard the entire journey. The pain points are in the transitions." | 把整个旅程做成故事板。痛点在转场中 | 不只看单个页面，看页面之间的过渡 |
| 7 | 一致性偏好 | — | "Inconsistency is cognitive tax. Every deviation needs justification." | 不一致就是认知税。每次偏离都需要理由 | 检查设计系统的遵从度 |
| 8 | 层级即服务 | — | "Hierarchy answers: what should the user see first, second, third?" | 层级回答的问题是：用户应该先看到什么、然后看到什么、再然后呢？ | 信息架构审查 |
| 9 | 边界情况偏执 | — | "47-character names? Zero results? Offline? First-timer vs power user?" | 47 个字符的名字？零结果？离线？新手 vs 老手？ | 交互状态覆盖审查 |
| 10 | 情感弧线意识 | — | "Users have feelings at every step. Map the emotional arc." | 用户在每一步都有情感。画出情感弧线 | 用户旅程的情感维度 |
| 11 | 可及性即基线 | — | "Accessibility is not a feature. It's the baseline." | 可及性不是功能。它是基线 | 对比度、键盘导航、屏幕阅读器 |
| 12 | AI Slop 雷达 | — | "If it looks like every other AI-generated UI, it's slop. Purple gradients, generic hero sections, three-column grids with icons — these are AI tells." | 如果它看起来像所有其他 AI 生成的 UI，那就是垃圾。紫色渐变、通用的 hero 区域、带图标的三列网格——这些是 AI 的指纹 | 检测并消除 AI 生成的套路化设计 |

**认知模式的应用映射：**

> **原文：** "When you evaluate information architecture, think through hierarchy-as-service. When you check interaction states, channel boundary-case paranoia. When you assess the emotional journey, apply Norman's time horizons. When something feels 'meh' but you can't say why, it's probably AI slop — activate the radar."
>
> **翻译：** 当你评估信息架构时，用层级即服务的方式思考。当你检查交互状态时，用边界情况偏执的方式。当你评估情感旅程时，用 Norman 的时间维度。当某个东西感觉"还行"但你说不出为什么不好时，它可能是 AI 垃圾——启动雷达。

**设计原理分析：**

注意第 12 个模式"AI Slop 雷达"的存在。这在其他 Skill 中不常见。它的存在是因为 `/plan-design-review` 审查的计划很可能是 AI 生成的——而 AI 生成的设计有非常明显的模式化特征。让 Claude 用一个专门的"反模式检测器"来审查自己（或其他 AI）的产出，这是一种**元认知设计**。

### 2.2 0-10 评分方法与 Gap→Fix→Re-rate 循环

这是 `/plan-design-review` 最核心的机制——把主观的设计审查变成可量化、可迭代的改进流程。

**评分标准：**

| 分数 | 含义 | 行动 |
|---|---|---|
| 0-3 | 严重问题，基本不可用 | 需要重大重设计 |
| 4-5 | 功能可用但体验差 | 需要显著改进 |
| 6-7 | 可接受但有明显改进空间 | 需要针对性修复 |
| 8-9 | 高质量，只有微调空间 | 快速通过，只做微调 |
| 10 | 卓越，堪称典范 | 跳过 |

**Gap→Fix→Re-rate 循环：**

> **原文：** "For each pass: Rate 0-10. If below 8, identify the specific gaps. Fix them by editing the plan directly. Re-rate. Repeat until 8+. Sections already at 8+ get a quick confirmation pass on re-run — don't re-do work that's already good."
>
> **翻译：** 对每个审查环节：评分 0-10。如果低于 8，识别具体差距。通过直接编辑计划来修复。重新评分。重复直到 8+。已经达到 8+ 的部分在重新运行时只做快速确认——不要重做已经做好的工作。

**循环示意：**

```
审查环节 X → 评分 6/10
  ↓
识别差距："缺少空状态设计""加载状态用了通用 spinner 而非骨架屏"
  ↓
直接编辑计划文件，补充缺失内容
  ↓
重新评分 → 8/10 ✓ → 进入下一个环节
```

**8+ 快速通过规则的设计原理：**

这条规则解决了一个实际问题：如果用户多次运行 `/plan-design-review`（比如改了一部分后重新审查），已经通过的环节不需要从头审查。这节省了时间，也避免了"过度审查"——反复雕琢一个已经足够好的部分，而忽略了真正需要改进的地方。

**迁移价值：** "评分 → 差距 → 修复 → 重新评分"这个循环是一个极其通用的模式。你可以在任何质量审查类 Skill 中使用它：
1. 定义评分维度和标准
2. 设定通过阈值（8/10 是一个好的默认值）
3. 低于阈值时强制修复
4. 高于阈值时快速通过

### 2.3 七个审查 Pass（审查环节）

七个审查环节按从结构到细节、从静态到动态的顺序排列：

#### Pass 1：Information Architecture（信息架构）

> **原文：** "Information architecture is the skeleton. If the skeleton is wrong, no amount of visual design can save it."
>
> **翻译：** 信息架构是骨骼。如果骨骼是错的，再多的视觉设计也救不了它。

| 检查项 | 审查什么 | 关键问题 |
|---|---|---|
| 内容层级 | 最重要的信息是否最突出？ | "用户 5 秒扫视能抓住核心信息吗？" |
| 导航结构 | 用户知道自己在哪里、能去哪里吗？ | "3 次点击内能到达任何核心功能吗？" |
| 分组逻辑 | 相关内容是否在一起？ | "用户找一个功能会在哪里找？它在那里吗？" |
| 命名 | 标签和标题是否清晰、无歧义？ | "用户读到这个词会想到什么？和实际功能一致吗？" |

#### Pass 2：Interaction States（交互状态覆盖表）

这是 `/plan-design-review` 最具操作性的环节。它用一张表格强制覆盖每个界面的 5 种状态：

**LOADING / EMPTY / ERROR / SUCCESS / PARTIAL 状态表：**

| 界面/组件 | LOADING | EMPTY | ERROR | SUCCESS | PARTIAL |
|---|---|---|---|---|---|
| 用户列表 | 骨架屏 3 行 | "还没有用户" + CTA | "加载失败" + 重试 | 用户卡片列表 | 部分加载 + "加载更多" |
| 搜索结果 | 输入时即时反馈 | "没有找到结果" + 建议 | "搜索出错" + 重试 | 结果列表 + 高亮 | 部分匹配 + 模糊提示 |
| 照片上传 | 进度条 + 百分比 | 拖拽区域 + 提示 | "上传失败" + 原因 + 重试 | 预览 + 确认 | 3/5 上传成功 + 失败项可重试 |

> **原文：** "Every UI component has at least 5 states. If the plan only describes SUCCESS, it's incomplete. Fill in the table. If the designer hasn't thought about EMPTY or PARTIAL, that's where the bugs will be."
>
> **翻译：** 每个 UI 组件至少有 5 种状态。如果计划只描述了 SUCCESS，它是不完整的。填完这张表。如果设计师没想过 EMPTY 或 PARTIAL，Bug 就在那里。

**为什么 PARTIAL 状态特别重要？**

大多数设计只考虑"全部成功"和"全部失败"。但现实中最常见的情况是**部分成功**：5 张照片上传了 3 张、10 条记录加载了 7 条、API 返回了数据但某些字段是 null。PARTIAL 状态的处理往往暴露了设计中最深的假设缺陷。

#### Pass 3：User Journey & Emotional Arc（用户旅程与情感弧线）

这个环节用 Norman 的时间维度框架审查用户体验：

**5 秒 / 5 分钟 / 5 年 框架：**

| 时间维度 | 审查什么 | 关键问题 |
|---|---|---|
| **5 秒**（第一印象） | 用户落地时看到什么？能立刻理解这是什么产品吗？ | "一个从没见过这个产品的人，5 秒内能说出它是干什么的吗？" |
| **5 分钟**（核心任务） | 用户能在 5 分钟内完成核心任务吗？过程中有摩擦点吗？ | "从注册到完成第一个有价值的操作需要多少步？" |
| **5 年**（长期关系） | 这个设计能随用户成长吗？会不会成为束缚？ | "高级用户 2 年后还会满意这个交互吗？" |

**情感弧线映射：**

> **原文 (Gebbia)：** "Storyboard the entire journey. The pain points are in the transitions."
>
> **翻译：** 把整个旅程做成故事板。痛点在转场中。

```
注册 → [期待] → 填表 → [焦虑: 要填这么多？] → 完成 → [成就感]
  → 首次使用 → [困惑: 从哪开始？] → 引导 → [安心] → 核心任务 → [满足]
```

审查不只看每个页面好不好——更看**页面之间的转场**是否自然。Gebbia 的洞察是：用户最容易流失的地方不是某个页面上，而是从一个页面跳到另一个页面时。

#### Pass 4：AI Slop Risk（AI 垃圾风险检测）

> **原文：** "AI-generated designs have tells. Like a poker player's tell, they reveal that no human designer made deliberate choices. Purple-to-blue gradients. Generic hero sections with stock imagery. Three-column grids with circular icons. Overly symmetrical layouts. If you see these patterns, flag them."
>
> **翻译：** AI 生成的设计有指纹。就像扑克玩家的 tell，它们暴露了没有人类设计师做过深思熟虑的选择。紫蓝渐变。带库存图片的通用 hero 区域。带圆形图标的三列网格。过度对称的布局。看到这些模式就标记它们。

**AI Slop 典型特征清单：**

| 特征 | 为什么是 Slop | 替代方案 |
|---|---|---|
| 紫蓝渐变背景 | 每个 AI 生成的 SaaS 页面都用这个 | 用品牌色或中性色 |
| 三列网格 + 圆形图标 | AI 的"安全默认值"，没有信息层级 | 根据内容重要性做非对称布局 |
| "Unlock the power of..." 标题 | 空洞的营销话术 | 用具体的价值主张 |
| 过度对称布局 | 视觉单调，没有焦点 | 刻意打破对称，创造视觉锚点 |
| 通用库存插图 | 不传达任何产品特有信息 | 用产品截图或定制插图 |
| 浅灰色细线分隔符到处都是 | 没有建立真正的视觉层级 | 用留白和字体大小建立层级 |

**这个环节存在的深层原因：**

在 AI 辅助设计的时代，最大的风险不是设计太差——而是设计"还行但没有灵魂"。AI 会趋向统计学上最常见的设计模式（因为那是训练数据中最多的），但这些模式恰恰是最没有辨识度的。伟大的设计需要**刻意的、有原则的偏离**。

#### Pass 5：Design System Alignment（设计系统对齐）

> **原文：** "Every component should trace back to the design system. If it can't, either the design system needs updating or the component needs redesigning."
>
> **翻译：** 每个组件都应该能追溯到设计系统。如果不能，要么设计系统需要更新，要么组件需要重新设计。

| 检查项 | 审查什么 |
|---|---|
| 色彩系统 | 是否只用了设计系统中定义的颜色？有没有"差不多的蓝"？ |
| 字体层级 | H1-H6、正文、标注是否遵循统一的字体比例？ |
| 间距系统 | 是否使用了一致的间距标尺（4px/8px 网格）？ |
| 组件复用 | 这个新组件是否已有类似的组件？为什么不复用？ |
| 交互模式 | 按钮样式、表单样式、弹窗样式是否与现有一致？ |

#### Pass 6：Responsive & Accessibility（响应式与可及性）

> **原文：** "Accessibility is not a feature. It's the baseline. If a screen reader user can't complete the core task, the design is broken — not 'needs improvement', broken."
>
> **翻译：** 可及性不是功能。它是基线。如果屏幕阅读器用户不能完成核心任务，设计就是坏的——不是"需要改进"，是坏的。

| 检查项 | 标准 |
|---|---|
| 对比度 | 文字与背景至少 4.5:1（WCAG AA） |
| 键盘导航 | Tab 顺序合理，焦点可见，所有操作可用键盘完成 |
| 屏幕阅读器 | ARIA 标签、语义化 HTML、图片 alt 文本 |
| 响应式 | 移动端 (320px)、平板 (768px)、桌面 (1024px+) 三个断点都有合理布局 |
| 触摸目标 | 可点击元素最小 44x44px |
| 色盲友好 | 不只用颜色传达信息（错误状态不只是红色） |

#### Pass 7：Unresolved Design Decisions（未解决的设计决策）

> **原文：** "Every plan has decisions that were deferred, compromised, or never made. Find them. List them. For each one: what's the default if no decision is made? Is that default acceptable?"
>
> **翻译：** 每个计划都有被延后的、妥协的、或从未做出的决策。找到它们。列出来。对每一个：如果不做决定，默认结果是什么？那个默认结果可接受吗？

这个环节是整个审查的**安全网**——它捕获前 6 个环节可能遗漏的问题。

```
未解决决策清单：
1. 移动端的表格如何展示？（默认：水平滚动 → 可能不可接受）
2. 暗色模式支持？（默认：不支持 → 取决于用户群体）
3. 国际化/RTL？（默认：只支持英文 LTR → 如果有中东用户则不可接受）
4. 离线状态？（默认：不支持 → 如果是移动 app 则不可接受）
```

### 2.4 Edit-in-Place 工作流

这是 `/plan-design-review` 与其他审查类 Skill 最大的区别。

> **原文：** "You have Edit and Write permissions. When you find a gap, don't just report it — fix it. Edit the plan directly. The plan IS the deliverable. When you're done, the plan should be better than when you started, not just annotated."
>
> **翻译：** 你有 Edit 和 Write 权限。当你发现差距时，不要只是报告——修复它。直接编辑计划。计划本身就是交付物。当你完成时，计划应该比开始时更好，而不只是多了些批注。

**为什么直接编辑而不是生成审查报告？**

传统审查流程：

```
审查者 → 审查报告 → 设计师读报告 → 设计师修改计划 → 审查者再审
```

gstack 审查流程：

```
/plan-design-review → 直接编辑计划 → 重新评分 → 计划已改进
```

中间环节的消除意味着：
1. **零信息损失** — 审查意见直接变成计划内容，不会在"翻译"过程中走样
2. **即时反馈** — 不需要等待设计师理解并执行审查意见
3. **可验证** — 每次编辑后立刻重新评分，能确认修复是否有效

**Edit-in-Place 的边界规则：**

> **原文：** "Fix gaps in completeness: add missing states, add missing flows, add missing specs. Do NOT override aesthetic choices unless they violate the design system or fail accessibility. Taste is the user's prerogative."
>
> **翻译：** 修复完整性差距：补充缺失的状态、缺失的流程、缺失的规格。不要覆盖审美选择，除非它违反了设计系统或可及性不达标。品味是用户的特权。

这条规则划定了 AI 审查的边界：**可以补充遗漏，不能替代品味**。这是一个非常重要的设计决策——它防止了 AI 把所有设计都"审查"成同一种风格。

**迁移价值：** edit-in-place 模式适用于任何"审查 + 改进"的场景。关键是明确定义什么可以直接改、什么需要问用户。一个好的经验法则：**客观问题（缺失的状态、违反的规范）可以直接改；主观问题（审美偏好、风格选择）必须问。**

### 2.5 已达标部分的快速通过

> **原文：** "Sections already at 8+ get a quick confirmation pass on re-run. Don't re-do work that's already good. Acknowledge the score, note any new context that might change it, and move on."
>
> **翻译：** 已经达到 8+ 的部分在重新运行时只做快速确认。不要重做已经做好的工作。确认分数，记录任何可能改变它的新上下文，然后继续。

这条规则解决了一个实际问题：用户可能多次运行 `/plan-design-review`——第一次获得初始审查，修改后再跑一次确认。如果每次都从零开始审查所有 7 个环节，不仅浪费时间，还可能产生不一致的评分（因为 LLM 的输出有随机性）。

**快速通过的流程：**

```
环节 1 信息架构：上次评分 9/10
  → 快速扫描：无新变更影响此环节
  → 确认 9/10 ✓ → 跳过

环节 2 交互状态：上次评分 5/10
  → 检测到计划已添加 EMPTY 和 ERROR 状态
  → 重新评分 → 8/10 ✓ → 继续

环节 3 用户旅程：上次评分 6/10
  → 检测到计划已添加情感弧线
  → 但新增的注册流程缺少 5 秒维度分析
  → 编辑补充 → 重新评分 → 8/10 ✓ → 继续
```

---

## 三、如何迁移到你自己的场景

### 场景 1：内容质量审查 Skill

从 `/plan-design-review` 提取评分循环机制，应用于内容审查：

```markdown
# 内容质量审查

## 认知模式
- Strunk & White：删掉每一个不必要的词
- Orwell：如果能用日常词汇，不要用专业术语
- Hemingway：短句是默认。长句需要理由。

## 审查环节（每环节 0-10 评分，< 8 则修复）
1. 清晰度：读者 5 秒内能理解核心信息吗？
2. 结构：逻辑流是否自然？读者会在哪里迷路？
3. 语气：是否与目标受众匹配？
4. 行动力：读完后读者知道该做什么吗？
5. AI Slop：是否有"In today's fast-paced world..."等空洞套话？

## 工作方式
直接编辑内容。不要生成审查报告。改完重新评分。
```

### 场景 2：产品需求文档（PRD）审查 Skill

把 7 个设计审查环节迁移到 PRD 领域：

| 设计审查环节 | PRD 迁移 |
|---|---|
| 信息架构 | 需求结构：核心需求 vs 边缘需求是否分层清晰？ |
| 交互状态（5 种） | 场景覆盖：正常流 / 异常流 / 边界条件 / 并发 / 降级 |
| 用户旅程（5s/5m/5y） | 价值时间线：即时价值 / 短期价值 / 长期价值 |
| AI Slop | PRD Slop：是否有模糊的"提升用户体验"而无具体度量？ |
| 设计系统对齐 | 技术对齐：是否与现有架构和技术栈一致？ |
| 响应式与可及性 | 约束条件：性能预算、合规要求、平台限制 |
| 未解决决策 | 开放问题：哪些关键决策还没做？默认结果是什么？ |

### 场景 3：演示文稿审查 Skill

提取视觉审查和情感弧线，应用于 PPT/演示审查：

```markdown
# 演示审查

## 姿态
你是 TED 演讲的导演。每一张幻灯片都需要赢得存在的权利。

## 认知模式
- Rams：如果删掉这张幻灯片，演示还成立吗？
- Norman（5s/5m）：观众 5 秒能理解这张幻灯片的核心信息吗？
- Gebbia（故事板）：幻灯片之间的过渡是否自然？

## 审查环节
1. 叙事弧线：故事有起承转合吗？高潮在哪里？
2. 视觉层级：每张幻灯片的视觉焦点是否明确？
3. 信息密度：每张幻灯片是否只传达一个核心信息？
4. AI Slop：是否有毫无信息量的装饰性幻灯片？
```

### 关键提取物（可复用到任何设计审查类 Skill）

1. **评分循环（Gap→Fix→Re-rate）：** 0-10 评分 + 低于阈值强制修复 + 重新评分。这把主观审查变成可量化的流程
2. **交互状态覆盖表（LOADING/EMPTY/ERROR/SUCCESS/PARTIAL）：** 强制覆盖每个组件的 5 种状态。大多数 Bug 藏在 EMPTY 和 PARTIAL 中
3. **时间维度审查（5s/5m/5y）：** 从三个时间尺度评估体验，防止只优化短期而牺牲长期
4. **AI Slop 检测：** 列出 AI 生成内容的典型模式，作为反模式清单
5. **Edit-in-Place：** 审查即改进，不是审查加改进。用 Edit/Write 权限直接修复差距
6. **快速通过已达标部分：** 8+ 的环节在重新运行时只确认不重审。节省时间，防止过度雕琢
7. **品味边界规则：** AI 可以补充遗漏，不能替代品味。客观问题直接改，主观问题问用户

---

> 本分析基于 `/plan-design-review` 的 Skill 设计原理和 gstack 整体架构逐项拆解。0-10 评分机制、七个审查环节、edit-in-place 工作流，每个设计决策都有明确的体验设计理由。
