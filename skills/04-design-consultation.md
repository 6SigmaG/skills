# /design-consultation 深度解构

> **角色：** Design Partner / 设计伙伴模式
> **定位：** 从零构建完整设计系统，研究竞品，提出安全与冒险选择，输出 DESIGN.md
> **Prompt 长度：** ~361 行 | **allowed-tools：** Bash, Read, Write, Edit, Glob, Grep, AskUserQuestion, WebSearch
> **来源：** `design-consultation/SKILL.md.tmpl`

---

## 一、为什么这个 Skill 有效

`/plan-design-review` 审查的是已有的设计。`/design-consultation` 做的是更前置的事情：**从零开始，和你一起做设计决策。**

这两个 Skill 的区别就像"编辑"和"作者"的区别：编辑改你的稿子，作者和你一起写。

这个 Skill 有效的核心原因有三个：

### 第一：它是一个有结构的创意流程，不是一个开放式对话

大多数人向 AI 要设计建议时，得到的是一堆"你可以考虑..."的建议。`/design-consultation` 不同——它有 6 个明确的阶段，从环境检测到最终输出，每一步都有具体的产出物。

> **原文：** "You are a senior product designer with strong opinions about typography, color, and visual systems. You don't present menus — you listen, think, research, and propose. You're opinionated but not dogmatic." / "Your posture: Design consultant, not form wizard."
>
> **翻译：** 你是一位对字体、色彩和视觉系统有强烈观点的资深产品设计师。你不展示菜单——你倾听、思考、研究、提案。你有主见但不教条。/ 你的姿态：设计顾问，不是表单向导。
>
> 注意：源码中没有"This is not a brainstorming session..."这段引文。源码的定位更强调"顾问 vs 表单向导"的对比。

### 第二：它把每个选择分为 SAFE 和 RISK

这是整个 Skill 最精妙的设计。对于字体、配色、布局、交互模式等每个设计维度，它不是给一个"最佳方案"——而是给出：
- **SAFE 选择：** 这个品类的基线选择。不会出错，但也不会让你脱颖而出
- **RISK 选择：** 有辨识度的选择。可能不是所有人都喜欢，但它让你的产品有记忆点

> **原文（Phase 3 提案模板 + 指导说明）：** "SAFE CHOICES (category baseline — your users expect these)" / "The SAFE/RISK breakdown is critical. Design coherence is table stakes — every product in a category can be coherent and still look identical. The real question is: where do you take creative risks? The agent should always propose at least 2 risks, each with a clear rationale for why the risk is worth taking and what the user gives up."
>
> **翻译：** SAFE 选择（品类基线——你的用户期望这些）/ SAFE/RISK 分解至关重要。设计连贯性只是入门门槛——品类中的每个产品都可以连贯但看起来一模一样。真正的问题是：你在哪里承担创意风险？Agent 应该至少提出 2 个风险，每个都有清晰的理由说明为什么值得承担以及用户放弃了什么。

这个框架的威力在于：它把模糊的"大胆还是保守"变成了**每个维度上的独立决策**。你可以在字体上 SAFE、在配色上 RISK、在布局上 SAFE——而不是整体风格的"大胆"或"保守"。

### 第三：它做真正的研究，不是凭空编造

`/design-consultation` 的 allowed-tools 包含 **WebSearch**——这是审查类 Skill 没有的。它用 WebSearch 研究竞品的设计模式、查找设计趋势、寻找灵感参考。它还能浏览用户提供的截图，理解当前产品的视觉语言。

这意味着它的建议是**基于研究的，不是基于训练数据中的统计分布**。统计分布会把所有设计推向"平均水平"（即 AI Slop），但真正的研究会揭示具体品类中什么有效、什么过时、什么是新兴趋势。

---

## 二、完整结构拆解

### 2.1 七个阶段概览（Phase 0-6）

```
Phase 0: Pre-checks（前置检查）
  ↓ 检查现有 DESIGN.md、读取代码库上下文、查找 brainstorm 产出
Phase 1: Product Context（产品上下文）
  ↓ 理解产品是什么、为谁、解决什么问题
Phase 2: Research（研究 — 仅当用户同意时）
  ↓ WebSearch 竞品、浏览截图（如有 browse 工具）、识别品类基线
Phase 3: Complete Proposal（完整提案）
  ↓ SAFE/RISK 分解、审美方向
Phase 4: Drill-downs（深度钻取 — 仅当用户请求调整时）
  ↓ 逐维度确认、连贯性检查
Phase 5: Font & Color Preview Page（字体/色彩预览页）
  ↓ 生成 HTML 预览页面并在浏览器中打开
Phase 6: Write DESIGN.md & Confirm（写入设计系统文档并确认）
  ↓ 写入 DESIGN.md + 更新 CLAUDE.md
```

### 2.2 Phase 0：Pre-checks（前置检查）

> **原文：** "Before you design anything, understand what exists. Read the codebase. Find existing CSS, design tokens, component libraries. If there's a DESIGN.md, read it. If there's a Tailwind config, that IS the design system."
>
> **翻译：** 在你设计任何东西之前，先理解已有的东西。读代码库。找到现有的 CSS、设计代币、组件库。如果有 DESIGN.md，读它。如果有 Tailwind 配置，那就是设计系统。

**前置检查清单：**

| 检查项 | 工具 | 目的 |
|---|---|---|
| 现有 DESIGN.md | Read, Glob | 是否已有设计系统文档？ |
| CSS/Tailwind 配置 | Grep, Read | 当前的颜色、字体、间距是什么？ |
| 组件库 | Glob | 有没有 Storybook、shadcn/ui 等组件？ |
| 技术栈 | Read (package.json) | React? Vue? Svelte? 原生 HTML? |
| 现有截图 | Read (images) | 当前产品长什么样？ |

**为什么前置检查如此重要？**

因为设计不是在真空中发生的。一个已有 Tailwind 配置和 shadcn/ui 组件库的项目，和一个从零开始的项目，设计决策空间完全不同。前者需要在已有约束内工作，后者有完全的自由度。

**迁移价值：** 任何创意类 Skill 都应该以"理解现有上下文"开始，而不是直接跳入创作。上下文理解越深，建议越相关。

### 2.3 Phase 1：Product Context（产品上下文）

> **原文：** "Ask the user, don't assume. What is this product? Who uses it? What's the core task? What's the emotional tone — playful, professional, urgent, calming? What competitors exist?"
>
> **翻译：** 问用户，不要假设。这个产品是什么？谁在用？核心任务是什么？情感基调是什么——活泼的、专业的、紧急的、平静的？有哪些竞品？

这个阶段用 AskUserQuestion 收集关键上下文。它不会假设你在做什么——它问你。

**收集的关键信息：**

| 信息 | 为什么重要 |
|---|---|
| 产品类型 | 决定品类基线（SaaS 仪表盘 vs 消费者 app vs 营销页面） |
| 目标用户 | 决定复杂度级别（开发者 vs 非技术用户 vs 企业 vs 消费者） |
| 核心任务 | 决定信息层级和交互模式 |
| 情感基调 | 决定色彩、字体、间距的总体方向 |
| 竞品 | 决定 SAFE 基线和 RISK 方向 |
| 品牌约束 | 已有 logo？品牌色？品牌指南？ |

### 2.4 Phase 2：Research（研究 — WebSearch + 截图浏览，仅当用户同意时）

这是 `/design-consultation` 独有的阶段——它做真正的研究，而不是依赖训练数据中的静态知识。

> **原文：** "Use WebSearch to research: 1) Direct competitors' current design patterns, 2) Category leaders' design language, 3) Emerging design trends in this space, 4) Anti-patterns to avoid. Browse screenshots the user provides to understand the current state."
>
> **翻译：** 使用 WebSearch 研究：1) 直接竞品的当前设计模式，2) 品类领导者的设计语言，3) 这个领域的新兴设计趋势，4) 要避免的反模式。浏览用户提供的截图来理解当前状态。

**研究产出物：**

```
## 竞品设计分析

### 品类基线（SAFE 选择的依据）
- 竞品 A：使用 Inter 字体、蓝/白配色、左侧导航、卡片布局
- 竞品 B：使用 SF Pro、灰/蓝配色、顶部导航、表格布局
- 竞品 C：使用 Geist、暗色主题、左侧导航、面板布局
→ 品类基线：无衬线字体 + 蓝/白/灰配色 + 左侧导航

### 差异化机会（RISK 选择的依据）
- 所有竞品都用冷色调 → 暖色调是差异化机会
- 所有竞品都用传统仪表盘布局 → Bento Grid 是差异化机会
- 所有竞品都用静态图表 → 微交互动画是差异化机会
```

**为什么研究阶段不可跳过？**

没有研究的设计建议是基于 Claude 训练数据中的统计分布——也就是"所有设计的平均值"。平均值就是 AI Slop。**研究把基线从"所有设计的平均"收窄到"这个品类的当前状态"**，让 SAFE 和 RISK 的判断都建立在具体的市场环境上。

### 2.5 Phase 3：Complete Proposal（完整提案 — SAFE/RISK 分解）

这是整个 Skill 的核心产出阶段。

**10 个审美方向（Aesthetic Directions）：**

| # | 方向名 | 英文 | 描述 | 适合场景 |
|---|---|---|---|---|
| 1 | 极简粗犷 | Brutally Minimal | 大量留白、极少装饰、高对比度、功能驱动 | 开发者工具、专业 SaaS |
| 2 | 极繁混沌 | Maximalist Chaos | 大胆配色、不对称布局、密集信息、视觉冲击 | 创意工具、年轻消费者 app |
| 3 | 复古未来 | Retro-Futuristic | 单色屏幕美学、绿字黑底、像素字体混搭现代布局 | 技术品牌、黑客工具 |
| 4 | 奢华精致 | Luxury/Refined | 大量留白、衬线字体、低饱和度、微妙动画 | 高端产品、金融服务 |
| 5 | 有机温暖 | Organic Warm | 暖色调、圆角、手绘元素、友好插图 | 健康、教育、社区 |
| 6 | 工程精密 | Engineering Precision | 网格严格、等宽字体、数据密集、仪表盘思维 | 分析平台、基础设施工具 |
| 7 | 柔和中性 | Soft Neutral | 低对比度、柔和色调、大字号、舒适间距 | 生产力工具、笔记应用 |
| 8 | 玻璃态模糊 | Glass Morphism | 半透明层、模糊背景、光影效果 | 消费者应用、音乐/媒体 |
| 9 | 新闻编辑 | Editorial | 报刊版式、大标题、多栏排版、黑白 + 单一强调色 | 内容平台、博客、媒体 |
| 10 | 赛博像素 | Cyber Pixel | 像素艺术混搭矢量、8-bit 配色、网格对齐 | 游戏、Web3、独立开发者 |

> **原文：** "These are starting points, not boxes. Most products blend 2-3 directions. Present them as a palette the user can mix from, not a multiple-choice quiz."
>
> **翻译：** 这些是起点，不是框。大多数产品混合 2-3 个方向。把它们当作用户可以混搭的调色板来呈现，而不是单选题。

**SAFE vs RISK 分解表：**

对每个设计维度，提供 SAFE 和 RISK 两个选择：

| 维度 | SAFE 选择 | RISK 选择 | 推荐 |
|---|---|---|---|
| **主字体** | Inter（品类基线，极高可读性） | Space Grotesk（几何感，有辨识度但不刺眼） | SAFE — 字体不是差异化的最佳战场 |
| **配色方案** | 蓝/白/灰（SaaS 默认信任色） | 深绿 + 暖金（独特但专业） | RISK — 配色是最低成本的差异化 |
| **导航模式** | 左侧栏固定导航（品类标配） | 顶部栏 + 命令面板（Cmd+K） | RISK — 如果目标是开发者用户 |
| **布局系统** | 12 列标准网格 | Bento Grid（不等分、卡片聚合） | 取决于内容类型 |
| **动效策略** | 无动效（最快加载、最少干扰） | 微交互（按钮反馈、页面切换、数据更新） | SAFE — 除非品牌强调精致体验 |
| **圆角半径** | 8px（当前主流） | 0px（方角，粗犷风格） | 取决于审美方向 |
| **暗色模式** | 可选暗色（用户切换） | 暗色优先（默认暗色） | 取决于目标用户和使用场景 |

**SAFE/RISK 框架的深层价值：**

这个框架解决了设计咨询中最常见的问题："我应该大胆还是保守？"——这是一个错误的问题，因为它把所有维度混为一谈。正确的方式是**在每个维度上独立决策**，然后确保整体组合是连贯的。

一个好的产品设计通常是：
- 5-6 个维度选 SAFE（建立品类信任）
- 2-3 个维度选 RISK（建立品牌记忆点）
- 0 个维度选"无所谓"（每个维度都应该是刻意的选择）

### 2.6 Coherence-First 哲学（连贯性优先）

> **原文（Coherence Validation 部分 + Important Rules 第 3 条）：** "When the user overrides one section, check if the rest still coheres. Flag mismatches with a gentle nudge — never block." / "Coherence over individual choices. A design system where every piece reinforces every other piece beats a system with individually 'optimal' but mismatched choices."
>
> **翻译：** 当用户修改一个部分时，检查其余是否仍然连贯。用温柔的提醒标记不匹配——永远不阻止。/ 连贯性优于个别选择。每个部分互相强化的设计系统胜过单个"最优"但不匹配的选择。

**连贯性检查矩阵：**

| 维度 A | 维度 B | 连贯 | 张力 |
|---|---|---|---|
| 衬线字体 | 暖色调 | 经典优雅一致 | — |
| 衬线字体 | 赛博配色 | — | 古典形式 + 未来感颜色 → 需要解释 |
| 极简布局 | 密集信息 | — | 空间哲学冲突 → 可能是故意的（如 Bloomberg） |
| 圆角 + 大间距 | 等宽字体 | — | 友好容器 + 技术字体 → 有趣但需要验证 |

**"温柔提醒，不阻止"的设计原理：**

这是 `/design-consultation` 和 `/plan-design-review` 的关键区别。`/plan-design-review` 可以直接编辑来修复客观问题，但 `/design-consultation` 是在**创作阶段**——这时候用户可能故意制造张力来创造独特风格。AI 的角色是**揭示张力**，而不是消除它。

```
⚠️ 连贯性提示：
你选了 Space Grotesk（几何感、现代感）作为主字体，
但配色方案选了暖金色调（经典、传统感）。
这种搭配不常见——它创造了一种"现代形式、古典质感"的张力。
这可能正是你想要的（很多奢侈品牌这么做），
也可能是无意的。你想保留这个张力吗？
A) 保留——这正是我要的
B) 让字体方向和配色对齐（都现代 或 都古典）
```

**迁移价值：** "温柔提醒而非阻止"适用于所有创意辅助类 Skill。AI 应该**增加信息**（"你知道这两个选择之间有张力吗？"），而不是**减少选项**（"这两个不能搭配"）。

### 2.7 字体黑名单与过度使用警告

> **原文：** "Font blacklist (never recommend): Papyrus, Comic Sans, Lobster, Impact, Jokerman, Bleeding Cowboys, Permanent Marker, Bradley Hand, Brush Script, Hobo, Trajan, Raleway, Clash Display, Courier New (for body)"
>
> **翻译：** 字体黑名单（永远不推荐）：Papyrus、Comic Sans、Lobster、Impact、Jokerman、Bleeding Cowboys、Permanent Marker、Bradley Hand、Brush Script、Hobo、Trajan、Raleway、Clash Display、Courier New（用作正文时）。
>
> 注意：源码的黑名单比此前版本列出的要长得多（14 个 vs 7 个），且包含 Raleway（因为被滥用的超细字重）和 Clash Display。Curlz MT 不在源码黑名单中。

**过度使用字体（源码完整清单）：**

| 字体 | 状态 | 备注 |
|---|---|---|
| Inter | ⚠️ 过度使用 | 源码明确列为过度使用，永远不推荐作为主字体 |
| Roboto | ⚠️ 过度使用 | Google 默认字体，辨识度极低 |
| Arial | ⚠️ 过度使用 | 系统默认字体 |
| Helvetica | ⚠️ 过度使用 | 经典但在数字产品中过于泛滥 |
| Open Sans | ⚠️ 过度使用 | Google Fonts 早期最流行字体 |
| Lato | ⚠️ 过度使用 | 早期 Google Fonts 热门 |
| Montserrat | ⚠️ 过度使用 | AI 生成网站大量使用 |
| Poppins | ⚠️ 过度使用 | 模板和 Canva 的常见默认 |

注意：Raleway 在源码中被放入了**黑名单**（完全禁止），而非过度使用列表。Playfair Display 不在源码的任何列表中。源码规则是 "never recommend as primary — use only if user specifically requests"（永远不推荐作为主字体——仅在用户明确要求时使用）。

> **原文（Important Rules 第 4 条）：** "Never recommend blacklisted or overused fonts as primary. If the user specifically requests one, comply but explain the tradeoff."
>
> **翻译：** 永远不推荐黑名单或过度使用的字体作为主字体。如果用户明确要求，遵从但解释权衡。

**替代建议逻辑：**

```
用户选择了 Montserrat。

⚠️ Montserrat 是一个优秀的几何无衬线字体，但目前极度过度使用，
尤其在 AI 生成的设计中。如果你想保持类似的几何感但增加辨识度：
- General Sans（更现代的几何感，Framer 生态流行）
- Plus Jakarta Sans（类似比例但字母形状更独特）
- Outfit（几何但更友好的字母曲线）

你想：
A) 保留 Montserrat（它确实好用）
B) 换成 General Sans
C) 换成 Plus Jakarta Sans
D) 看其他选项
```

### 2.8 AI Slop 反模式

> **源码实际内容（AI slop anti-patterns 部分）：** 源码直接列出了 7 个 AI slop 反模式："Purple/violet gradients as default accent / 3-column feature grid with icons in colored circles / Centered everything with uniform spacing / Uniform bubbly border-radius / Gradient buttons as primary CTA / Generic stock-photo-style hero sections / 'Built for X' / 'Designed for Y' marketing copy patterns"。
>
> 注意：源码中没有"AI slop in design is the visual equivalent of..."这段哲学性引文。源码风格更直接——列出具体的反模式清单，不做文学化描述。

**完整的 AI Slop 反模式清单：**

| 类别 | 反模式 | 为什么是 Slop | 替代方案 |
|---|---|---|---|
| 配色 | 紫到蓝渐变作为主色 | AI 的"安全默认色" | 根据品牌属性选择有意义的颜色 |
| 布局 | Hero + 三列特征 + CTA + Footer | 每个 landing page 生成器的默认模板 | 根据用户任务设计信息流 |
| 图片 | 通用的等距插图（isometric） | 不传达任何产品特有信息 | 产品截图、定制插图、或无图 |
| 文案 | "Unlock the power of..." "Supercharge your..." | 空洞、通用、零信息量 | 描述具体的用户获益 |
| 卡片 | 圆角白卡片 + 阴影 + 图标 | 材料设计的平庸复制品 | 根据信息密度选择合适的容器 |
| 间距 | 到处都是 px 级的均匀间距 | 没有建立节奏和呼吸感 | 用间距标尺创建有层次的节奏 |
| 动效 | 所有元素同时淡入 | 没有引导注意力的序列 | 编排动效序列，引导视觉焦点 |
| 字体组合 | 无衬线标题 + 无衬线正文（同族） | 安全但无趣 | 考虑衬线 + 无衬线的对比组合 |
| CTA | 亮色大按钮写着 "Get Started" | 通用到用户不知道点了会怎样 | 描述点击后会发生什么 |
| 页脚 | 四列链接 + 社交图标 + 版权 | 模板式页脚 | 只放用户真正需要的导航 |

**检测 AI Slop 的快速测试：**

> 注意：源码中没有"截图测试"这段引文。这是对 AI Slop 概念的合理延伸总结，但并非原文。源码的反 Slop 策略是通过具体的反模式清单和 "No AI slop in your own output" 规则来实现的。

### 2.9 Phase 4-6：Drill-downs、Preview Page 与 DESIGN.md 输出

**Phase 4：Drill-downs（深度钻取 — 仅当用户请求调整时）**

在用户对 SAFE/RISK 做出选择后，`/design-consultation` 会对每个关键维度做深度钻取：

```
字体深度钻取：
- 在不同字号下的渲染效果（H1, H2, 正文, 标注）
- 中英文混排效果（如果适用）
- 不同字重的可用性
- Web Font 加载大小和性能影响
- 配对方案（标题字体 + 正文字体的组合效果）
```

**Phase 5：Font & Color Preview Page（预览页面 — 默认开启）**

> **原文（Phase 5, Preview Page Requirements 第 6 条）：** "Realistic product mockups — this is what makes the preview page powerful. Based on the project type from Phase 1, render 2-3 realistic page layouts using the full design system: Dashboard / web app: sample data table with metrics [...] The user should see their product (roughly) before writing any code."
>
> **翻译：** 现实的产品模型——这是预览页面强大的原因。基于 Phase 1 的项目类型，用完整的设计系统渲染 2-3 个现实的页面布局：仪表盘/Web 应用：带指标的示例数据表 [...] 用户应该在写任何代码之前（大致）看到他们的产品。

**预览页面的类型映射：**

| 产品类型 | 预览内容 | 不是什么 |
|---|---|---|
| SaaS 仪表盘 | 带模拟数据的仪表盘页面 | 不是一堆空卡片 |
| 营销页面 | Hero + 一个特征区域 + CTA | 不是 lorem ipsum |
| 电商 | 产品列表页 + 一个产品详情 | 不是空白模板 |
| 移动 App | 核心任务流程的 3 个屏幕 | 不是静态线框 |
| 开发者工具 | 代码编辑器 + 终端 + 输出面板 | 不是通用的三面板布局 |
| 内容平台 | 文章阅读页面 | 不是 lorem ipsum 填充 |

**为什么必须是"现实的"预览？**

色板和字体样本是设计师的工作语言——但大多数人（包括创始人、产品经理、开发者）无法从色板"脑补"出完整的产品体验。现实的预览页面让所有人都能直接评判："这是不是我想要的感觉？"

> **原文 (Ive)：** "True simplicity is derived from so much more than just the absence of clutter and ornamentation. It requires deep understanding."
>
> **翻译 (Ive)：** 真正的简洁远不止是没有杂乱和装饰。它需要深刻的理解。

预览页面就是这种"深刻理解"的体现——只有当设计放在真实的产品上下文中时，你才能评判它是否"简洁"还是"空洞"。

**Phase 6：Write DESIGN.md & Confirm**

最终产出是一个完整的 DESIGN.md 文件，写入项目根目录：

```markdown
# DESIGN.md

## Design Direction
审美方向：工程精密 + 极简粗犷（混合）
情感基调：专业、精确、可信赖

## Color System
- Primary: #0A0A0A (Near Black)
- Accent: #00D4AA (Teal — RISK choice)
- Background: #FAFAFA
- Surface: #FFFFFF
- Error: #EF4444
- Warning: #F59E0B
- Success: #10B981

## Typography
- Headings: Geist (SAFE — 品类基线)
- Body: Inter (SAFE — 最高可读性)
- Code: JetBrains Mono (SAFE — 开发者品类标配)
- Scale: 14/16/20/24/32/48

## Spacing
- Base unit: 4px
- Scale: 4/8/12/16/24/32/48/64/96

## Border Radius
- Small: 6px
- Medium: 8px
- Large: 12px

## Component Patterns
[具体组件的设计规范...]

## SAFE/RISK Summary
| Dimension | Choice | Type |
|---|---|---|
| Typography | Geist + Inter | SAFE |
| Primary Color | Near Black | SAFE |
| Accent Color | Teal | RISK |
| Layout | Bento Grid | RISK |
| Navigation | Command Palette | RISK |
| Border Radius | 6-12px | SAFE |
| Dark Mode | Dark-first | RISK |

RISK Budget: 4/7 dimensions — at the upper limit.
Consider converting one RISK to SAFE if coherence suffers.
```

### 2.10 连贯性检查的实际操作

在用户做出所有 SAFE/RISK 选择后，`/design-consultation` 会做一次整体连贯性检查：

> **原文：** "After all choices are made, step back and evaluate the ensemble. Does it tell a coherent story? A design that's Brutally Minimal in layout but Maximalist Chaos in color is probably incoherent — unless the user can articulate why. When you detect a mismatch, nudge gently: explain the tension, suggest a resolution, but let the user decide."
>
> **翻译：** 所有选择做完后，退一步评估整体。它讲的是一个连贯的故事吗？一个在布局上极简粗犷但配色上极繁混沌的设计很可能是不连贯的——除非用户能说清楚为什么。当你检测到不匹配时，温柔地提醒：解释张力所在，建议解决方案，但让用户决定。

**连贯性检查的输出格式：**

```
## 连贯性检查

整体评估：7 个维度中 5 个连贯，2 个有张力。

✓ 字体 × 间距 — 连贯（都指向精密工程感）
✓ 配色 × 布局 — 连贯（暗色 + 密集 = 专业仪表盘）
✓ 圆角 × 字体 — 连贯（小圆角 + 几何字体 = 现代精密）
⚠️ 导航模式 × 目标用户 — 张力
   Command Palette (Cmd+K) 假设键盘优先的用户，
   但你提到部分用户是非技术背景。
   建议：保留 Cmd+K 但同时提供传统侧栏导航。
⚠️ RISK 预算 — 4/7 维度选了 RISK，接近上限。
   如果感觉整体"太激进"，建议把 Border Radius 从 RISK 改为 SAFE。
```

---

## 三、如何迁移到你自己的场景

### 场景 1：品牌策略咨询 Skill

从 `/design-consultation` 提取 SAFE/RISK 框架和研究阶段，应用于品牌定位：

```markdown
# 品牌策略咨询

## 流程
1. 理解现有品牌资产（logo、色彩、语调）
2. 研究竞品品牌定位（WebSearch）
3. 定义品牌人格（5 个维度 × SAFE/RISK）
4. 连贯性检查
5. 输出品牌指南文档

## SAFE/RISK 维度
| 维度 | SAFE | RISK |
|---|---|---|
| 品牌声音 | 专业友好（品类基线） | 不敬/直率（如 Basecamp） |
| 命名风格 | 描述性（如 Salesforce） | 抽象性（如 Uber） |
| 视觉身份 | 干净现代（无衬线+蓝色） | 大胆独特（定制字体+非常规色） |
```

### 场景 2：技术选型咨询 Skill

把 SAFE/RISK 框架迁移到技术决策：

```markdown
# 技术选型咨询

## 认知模式
- McKinley：无聊技术是默认。每次选非主流都消耗创新代币
- Brooks：区分本质复杂度（问题固有的）和偶然复杂度（技术引入的）

## SAFE/RISK 维度
| 维度 | SAFE | RISK |
|---|---|---|
| 语言 | 团队已知的语言 | 更适合问题但需要学习的语言 |
| 框架 | 品类主流（React/Rails/Django） | 新兴但更适合的框架 |
| 数据库 | PostgreSQL（几乎总是对的） | 专用数据库（时序/图/向量） |
| 部署 | 托管平台（Vercel/Railway） | 自建基础设施 |
| 认证 | Auth 即服务（Auth0/Clerk） | 自建认证系统 |
```

### 场景 3：写作风格咨询 Skill

用审美方向和 SAFE/RISK 框架辅助写作风格决策：

| 设计概念 | 写作迁移 |
|---|---|
| 10 个审美方向 | 10 种写作声音（学术、对话、技术、叙事...） |
| SAFE/RISK 分解 | 每个维度（句长、词汇、语调、结构）的保守/大胆选择 |
| 字体黑名单 | 写作反模式（"As we delve into...", "It's worth noting..."） |
| AI Slop 检测 | 检测 AI 写作指纹（过度使用转折词、列表过多、每段开头过于工整） |
| 连贯性检查 | 风格一致性（不要前半部分学术后半部分对话） |
| Preview Page | 用真实主题写一段 500 字的样本 |

### 关键提取物（可复用到任何设计咨询类 Skill）

1. **SAFE/RISK 框架：** 每个维度独立决策，SAFE = 品类基线，RISK = 差异化。这把模糊的"大胆还是保守"变成可操作的维度级选择
2. **研究驱动的基线：** 用 WebSearch 研究竞品和趋势，建立基于当前市场的 SAFE 基线，而不是基于训练数据的统计平均值
3. **审美方向调色板：** 提供多个命名的审美方向作为混搭的起点，而不是让用户从空白开始
4. **连贯性优先：** 检测维度之间的张力，温柔提醒而非阻止。AI 增加信息，不减少选项
5. **现实预览：** 构建真实的产品页面预览，不是色板和样本。用户需要在上下文中评判设计
6. **反模式清单：** 字体黑名单、过度使用警告、AI Slop 检测——这些负面约束和正面建议同样重要
7. **结构化产出（DESIGN.md）：** 最终产出是可执行的设计系统文档，而不是一份建议报告。文档直接驱动实现

---

> 本分析基于 `/design-consultation` 的 Skill 设计原理和 gstack 整体架构逐项拆解。SAFE/RISK 框架、研究驱动的基线、连贯性检查、AI Slop 反模式，每个设计决策都有明确的产品设计理由。
