# /plan-design-review 深度解构

> **角色：** Senior Designer / 资深设计师模式
> **定位：** 设计质量审查、0-10 评分与修复循环、AI Slop 检测、交互状态完整性
> **Prompt 长度：** ~288 行 | **allowed-tools：** Read, Edit, Grep, Glob, Bash, AskUserQuestion
> **来源：** `plan-design-review/SKILL.md.tmpl`

---

## 一、为什么这个 Skill 有效

大多数设计审查工具做的事情是：列出一堆问题，然后交给你自己去修。`/plan-design-review` 做的完全不同——**它直接改你的计划，改完重新评分，直到每个维度都达到 8 分以上。**

这个 Skill 有效的核心原因有三个：

### 第一：它激活的是设计大师的审美直觉，不是设计规范手册

Prompt 中引用了 12 个认知模式，来自 Dieter Rams、Don Norman、Jony Ive、Julie Zhuo、Joe Gebbia 等设计领域最有影响力的思想家。当 Claude 读到 Rams 的名字时，它训练数据中关于"少即是多"哲学的数百万字文本被激活。它不是在应用一条规则——它是在**以 Rams 的审美标准审视设计**。

> **原文：** "These aren't a checklist — they're how you see. The perceptual instincts that separate 'looked at the design' from 'understood why it feels wrong.' Let them run automatically as you review."
>
> **翻译：** 这些不是清单——它们是你观察的方式。把"看了设计"和"理解了为什么感觉不对"区分开的感知直觉。在审查过程中让它们自动运行。

### 第二：它用量化评分把"品味"变成可操作的流程

设计审查中最大的问题是主观性——"我觉得不好"没有行动力。`/plan-design-review` 用 0-10 分的评分系统，加上"差距 → 修复 → 重新评分"的闭环，把主观判断变成了**可追踪、可验证的改进过程**。

### 第三：它直接编辑计划文件，而不是生成一份独立的审查报告

这是与 `/plan-ceo-review` 的关键区别。`/plan-ceo-review` 只读不写（没有 Edit/Write），纯粹生成审查意见。而 `/plan-eng-review` 有 Write 权限（用于写测试计划文件），`/plan-design-review` 有 Edit 权限，**直接在计划文件上改**。（注意：`/plan-design-review` 没有 Write 权限，只能编辑已有文件，不能创建新文件——这是刻意的约束。）

为什么？因为设计反馈如果不立即落地，就会被遗忘。"把间距改成 16px"如果只写在审查报告里，执行者可能漏掉。但如果直接改在计划文件里，它就成了执行规范的一部分。

**这是 edit-in-place workflow（就地编辑工作流）的核心价值：审查即改进，不是审查加改进。**

---

## 二、完整结构拆解

### 2.1 认知模式（12 个设计思维激活）

| # | 模式名（源码原名） | 来源 | 原文核心（逐字引用） | 中文含义 | 应用场景 |
|---|---|---|---|---|---|
| 1 | 看系统而非屏幕 | — | "Never evaluate in isolation; what comes before, after, and when things break." | 永远不要孤立评估；要看之前、之后、以及出错时会怎样 | 审查时始终关注上下文和转场 |
| 2 | 共情即模拟 | — | "Not 'I feel for the user' but running mental simulations: bad signal, one hand free, boss watching, first time vs. 1000th time." | 不是"我同情用户"，而是运行心理模拟：差信号、一只手空、老板在看、第一次 vs 第一千次 | 评估真实使用场景下的体验 |
| 3 | 层级即服务 | — | "Every decision answers 'what should the user see first, second, third?' Respecting their time, not prettifying pixels." | 每个决策回答"用户该先看到什么、然后看什么？"尊重用户时间，而非美化像素 | 信息架构审查 |
| 4 | 约束崇拜 | — | "Limitations force clarity. 'If I can only show 3 things, which 3 matter most?'" | 限制迫使清晰。"如果只能展示 3 样，哪 3 样最重要？" | 评估界面是否做了减法 |
| 5 | 问题反射 | — | "First instinct is questions, not opinions. 'Who is this for? What did they try before this?'" | 第一反应是问题而非意见。"这是给谁的？他们之前试过什么？" | 审查前先理解上下文 |
| 6 | 边界情况偏执 | — | "What if the name is 47 chars? Zero results? Network fails? Colorblind? RTL language?" | 名字 47 个字符？零结果？网络断了？色盲？RTL 语言？ | 交互状态覆盖审查 |
| 7 | "我会注意到吗"测试 | — | "Invisible = perfect. The highest compliment is not noticing the design." | 不可见 = 完美。最高赞美是没有注意到设计的存在 | 评估设计是否自然无痕 |
| 8 | 有原则的品味 | Zhuo | "'This feels wrong' is traceable to a broken principle. Taste is *debuggable*, not subjective." | "感觉不对"可追溯到一个被违反的原则。品味是可调试的，不是主观的 | 评分时必须给出原则依据 |
| 9 | 减法为默认 | Rams, Maeda | "'As little design as possible' (Rams). 'Subtract the obvious, add the meaningful' (Maeda)." | "尽可能少的设计"（Rams）。"减去显而易见的，加上有意义的"（Maeda） | 看到杂乱时先减再加 |
| 10 | 时间维度设计 | Norman | "First 5 seconds (visceral), 5 minutes (behavioral), 5-year relationship (reflective) — design for all three simultaneously." | 前 5 秒（本能）、5 分钟（行为）、5 年关系（反思）——同时为三者设计 | 用户旅程的多时间尺度评估 |
| 11 | 为信任而设计 | Gebbia | "Every design decision either builds or erodes trust. Strangers sharing a home requires pixel-level intentionality." | 每个设计决策要么建立要么侵蚀信任。陌生人共享住所需要像素级的用心 | 审查每个交互是否建立信任 |
| 12 | 故事板旅程 | Gebbia | "Before touching pixels, storyboard the full emotional arc. The 'Snow White' method: every moment is a scene with a mood." | 动手画像素之前，先做完整情感弧线的故事板。"白雪公主"方法：每个时刻都是有情绪的场景 | 用户旅程的情感维度 |

注意：源码中**没有**"AI Slop 雷达"作为认知模式之一。AI Slop 风险检测是作为 Pass 4 独立审查环节和 Design Principles 第 5 条存在的，而非认知模式的一部分。源码也没有"一致性偏好"、"情感弧线意识"、"可及性即基线"作为独立的认知模式——这些是 Design Principles 和审查环节中的内容，被前一版本错误地归入了认知模式列表。

**认知模式的应用映射：**

> **原文：** "When reviewing a plan, empathy as simulation runs automatically. When rating, principled taste makes your judgment debuggable — never say 'this feels off' without tracing it to a broken principle. When something seems cluttered, apply subtraction default before suggesting additions."
>
> **翻译：** 审查计划时，共情即模拟自动运行。评分时，有原则的品味让你的判断可调试——永远不要说"感觉不对"而不追溯到一个被违反的原则。当某个东西看起来杂乱时，先应用减法默认，再建议添加。

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

> **原文（The 0-10 Rating Method）：** "Pattern: 1. Rate: 'Information Architecture: 4/10' 2. Gap: 'It's a 4 because the plan doesn't define content hierarchy...' 3. Fix: Edit the plan to add what's missing 4. Re-rate: 'Now 8/10 — still missing mobile nav hierarchy' 5. AskUserQuestion if there's a genuine design choice to resolve 6. Fix again → repeat until 10 or user says 'good enough, move on'"
>
> **翻译：** 模式：1. 评分："信息架构：4/10" 2. 差距："是 4 分因为计划没有定义内容层级..." 3. 修复：编辑计划补充缺失内容 4. 重新评分："现在 8/10——仍缺少移动端导航层级" 5. 如果有真正的设计选择需要解决则 AskUserQuestion 6. 再次修复 → 重复直到 10 分或用户说"够好了，继续"

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

> **原文：** "Rate 0-10: Does the plan define what the user sees first, second, third? FIX TO 10: Add information hierarchy to the plan. Include ASCII diagram of screen/page structure and navigation flow. Apply 'constraint worship' — if you can only show 3 things, which 3?"
>
> **翻译：** 评分 0-10：计划是否定义了用户先看到什么、然后看到什么、再看到什么？修复到 10 分：在计划中添加信息层级。包含屏幕/页面结构和导航流程的 ASCII 图表。应用"约束崇拜"——如果只能展示 3 样东西，是哪 3 样？

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

> **原文：** "Rate 0-10: Does the plan specify loading, empty, error, success, partial states? FIX TO 10: Add interaction state table to the plan [...] For each state: describe what the user SEES, not backend behavior. Empty states are features — specify warmth, primary action, context."
>
> **翻译：** 评分 0-10：计划是否指定了加载、空、错误、成功、部分状态？修复到 10 分：在计划中添加交互状态表 [...] 对每种状态：描述用户看到什么，而非后端行为。空状态是功能——指定温暖感、主要操作和上下文。

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

> **原文（Pass 4: AI Slop Risk）：** "Rate 0-10: Does the plan describe specific, intentional UI — or generic patterns? FIX TO 10: Rewrite vague UI descriptions with specific alternatives." 以及 Design Principles 第 5 条: "AI slop is the enemy. Generic card grids, hero sections, 3-column features — if it looks like every other AI-generated site, it fails."
>
> **翻译：** 评分 0-10：计划描述的是具体的、有意图的 UI，还是通用模式？修复到 10 分：用具体的替代方案重写模糊的 UI 描述。/ AI 垃圾是敌人。通用卡片网格、hero 区域、三列特征——如果看起来像其他所有 AI 生成的网站，那就失败了。

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

> **原文（Pass 5）：** "Rate 0-10: Does the plan align with DESIGN.md? FIX TO 10: If DESIGN.md exists, annotate with specific tokens/components. If no DESIGN.md, flag the gap and recommend /design-consultation. Flag any new component — does it fit the existing vocabulary?"
>
> **翻译：** 评分 0-10：计划是否与 DESIGN.md 对齐？修复到 10 分：如果 DESIGN.md 存在，用具体的设计代币/组件来标注。如果没有 DESIGN.md，标记为缺口并推荐运行 /design-consultation。标记任何新组件——它是否适合现有的组件词汇？

| 检查项 | 审查什么 |
|---|---|
| 色彩系统 | 是否只用了设计系统中定义的颜色？有没有"差不多的蓝"？ |
| 字体层级 | H1-H6、正文、标注是否遵循统一的字体比例？ |
| 间距系统 | 是否使用了一致的间距标尺（4px/8px 网格）？ |
| 组件复用 | 这个新组件是否已有类似的组件？为什么不复用？ |
| 交互模式 | 按钮样式、表单样式、弹窗样式是否与现有一致？ |

#### Pass 6：Responsive & Accessibility（响应式与可及性）

> **原文（Design Principles 第 7 条 + Pass 6）：** "Accessibility is not optional. Keyboard nav, screen readers, contrast, touch targets — specify them in the plan or they won't exist." / Pass 6: "Add a11y: keyboard nav patterns, ARIA landmarks, touch target sizes (44px min), color contrast requirements."
>
> **翻译：** 可及性不是可选项。键盘导航、屏幕阅读器、对比度、触摸目标——在计划中指定它们，否则它们不会存在。/ 添加无障碍：键盘导航模式、ARIA 地标、触摸目标大小（最小 44px）、颜色对比度要求。

| 检查项 | 标准 |
|---|---|
| 对比度 | 文字与背景至少 4.5:1（WCAG AA） |
| 键盘导航 | Tab 顺序合理，焦点可见，所有操作可用键盘完成 |
| 屏幕阅读器 | ARIA 标签、语义化 HTML、图片 alt 文本 |
| 响应式 | 移动端 (320px)、平板 (768px)、桌面 (1024px+) 三个断点都有合理布局 |
| 触摸目标 | 可点击元素最小 44x44px |
| 色盲友好 | 不只用颜色传达信息（错误状态不只是红色） |

#### Pass 7：Unresolved Design Decisions（未解决的设计决策）

> **原文（Pass 7）：** "Surface ambiguities that will haunt implementation:" 以及表格模板 "DECISION NEEDED | IF DEFERRED, WHAT HAPPENS"，示例："What does empty state look like? | Engineer ships 'No items found.'"
>
> **翻译：** 暴露会困扰实现的歧义。每个未解决的决策都有一列"如果延后会怎样"——例如"空状态长什么样？→ 工程师会上线'没有找到项目。'"

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

> **原文（意译自源码设计意图）：** "The output of this skill is a better plan, not a document about the plan." / "Fix: Edit the plan to add what's missing"
>
> **翻译：** 这个 Skill 的产出是一个更好的计划，而不是一份关于计划的文档。发现差距时，直接编辑计划补充缺失的内容。

注意：源码中没有单独一段完整的"edit-in-place 宣言"，但这个理念体现在整个 Skill 的设计中——它有 Edit 权限（无 Write），每个审查环节的模式都是"评分 → 差距 → 修复 → 重新评分"。

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

> **设计意图（从源码行为推导）：** 源码通过 Gap→Fix→Re-rate 循环明确了修复的范围——补充缺失的状态、流程、规格（如交互状态表中的 EMPTY/PARTIAL）。同时，源码中"Escape hatch"规则指出：只有存在真正有意义的设计选择权衡时才 AskUserQuestion——这隐含了"客观遗漏直接修，主观选择问用户"的边界。
>
> 注意：源码中没有"Fix gaps in completeness..."这段原文，上述是对源码设计模式的准确总结。

这条规则划定了 AI 审查的边界：**可以补充遗漏，不能替代品味**。这是一个非常重要的设计决策——它防止了 AI 把所有设计都"审查"成同一种风格。

**迁移价值：** edit-in-place 模式适用于任何"审查 + 改进"的场景。关键是明确定义什么可以直接改、什么需要问用户。一个好的经验法则：**客观问题（缺失的状态、违反的规范）可以直接改；主观问题（审美偏好、风格选择）必须问。**

### 2.5 已达标部分的快速通过

> **原文：** "Re-run loop: invoke /plan-design-review again → re-rate → sections at 8+ get a quick pass, sections below 8 get full treatment."
>
> **翻译：** 重新运行循环：再次调用 /plan-design-review → 重新评分 → 8+ 的环节快速通过，低于 8 的环节完整审查。

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
