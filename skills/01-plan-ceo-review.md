# /plan-ceo-review 深度解构

> **角色：** CEO / 创始人模式
> **定位：** 重新思考问题，找到 10 星产品，挑战前提假设
> **Prompt 长度：** ~600 行 | **allowed-tools：** Read, Grep, Glob, Bash, AskUserQuestion
> **来源：** `plan-ceo-review/SKILL.md.tmpl`

---

## 一、为什么这个 Skill 有效（即使你审查的不是工程计划）

这个 Skill 最惊人的特性是：**它的 Prompt 里没有一个字提到商业领域，但它可以审查商业计划、产品策略、甚至个人职业规划。**

原因在于它做的三件事：

1. **定义审查姿态（Posture），而非审查内容** — 告诉 AI "你不是来盖橡皮章的"，而不是"检查这 10 个指标"
2. **使用领域无关的元问题** — "这是对的问题吗？" "如果什么都不做会怎样？" 这些问题对任何领域都有效
3. **提供不可跳过的完整流程** — 10 个审查环节的工程术语会被 Claude 自动迁移到输入的领域

**核心洞察：** Prompt 的最高形态不是给 AI 更多领域知识，而是给 AI 正确的行为框架，让它自己调用已有的海量知识。

---

## 二、完整结构拆解

### 2.1 YAML Frontmatter（元信息）

```yaml
---
name: plan-ceo-review
version: 1.0.0
description: |
  CEO/founder-mode plan review. Rethink the problem, find the 10-star product,
  challenge premises, expand scope when it creates a better product. Four modes:
  SCOPE EXPANSION (dream big), SELECTIVE EXPANSION (hold scope + cherry-pick
  expansions), HOLD SCOPE (maximum rigor), SCOPE REDUCTION (strip to essentials).
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash
  - AskUserQuestion
---
```

**中文翻译：**
CEO/创始人模式的计划审查。重新思考问题，找到 10 星产品，挑战前提假设，在能创造更好产品的时候扩大范围。四种模式：范围扩展（放大梦想）、选择性扩展（保持范围 + 逐个挑选扩展）、保持范围（最大严谨度）、范围缩减（剥离到本质）。

**设计原理分析：**
- `allowed-tools` 没有 Edit 和 Write — **这个 Skill 只审查，不改代码**。这是刻意的：在 Plan Mode 中，你不想让 AI 一边想一边动手。想清楚再动。
- description 写得像 pitch — 因为 Claude Code 会用这段文字决定什么时候建议使用这个 Skill

### 2.2 Preamble（前言启动块）

```
{{PREAMBLE}}
{{BASE_BRANCH_DETECT}}
```

这两个占位符在构建时被替换为：
- **PREAMBLE：** 更新检查 + 会话追踪 + 贡献者模式 + AskUserQuestion 统一格式
- **BASE_BRANCH_DETECT：** 动态检测 PR 目标分支（main / master / develop）

**迁移价值：** 如果你开发自己的 Skill，Preamble 模式非常值得借鉴 — 在真正的指令之前先让 Agent 感知环境。

### 2.3 Philosophy（哲学 — 姿态定义）

**原文：**
```
You are not here to rubber-stamp this plan. You are here to make it extraordinary,
catch every landmine before it explodes, and ensure that when this ships, it ships
at the highest possible standard.
```

**中文翻译：**
你不是来给这个计划盖橡皮章的。你是来让它变得卓越的，在每颗地雷爆炸之前把它找出来，确保这个东西上线的时候，是以最高标准上线的。

**设计原理：** 这一段是整个 Skill 最重要的一句话。它没有给任何具体指令，它设定了一个**对抗性姿态** — "你是来找问题的，不是来点头的"。Claude 读到这句话后，它的整个输出风格都会改变。

**四种模式的姿态定义：**

| 模式 | 原文 | 翻译 | 隐喻 |
|---|---|---|---|
| SCOPE EXPANSION | You are building a cathedral. Envision the platonic ideal. Push scope UP. Ask "what would make this 10x better for 2x the effort?" You have permission to dream. | 你在建造一座大教堂。想象柏拉图式的理想形态。把范围往上推。问"什么版本能在 2 倍努力下好 10 倍？"你有权做梦。 | 大教堂建造者 |
| SELECTIVE EXPANSION | You are a rigorous reviewer who also has taste. Hold the current scope as your baseline — make it bulletproof. But separately, surface every expansion opportunity individually as an AskUserQuestion so the user can cherry-pick. | 你是一个有品味的严谨审查者。以当前范围为基线——让它防弹。但同时逐个展示每个扩展机会供用户逐一挑选。 | 有品味的审查者 |
| HOLD SCOPE | You are a rigorous reviewer. The plan's scope is accepted. Your job is to make it bulletproof. Do not silently reduce OR expand. | 你是一个严谨的审查者。计划的范围已被接受。你的工作是让它防弹。不要悄悄缩小，也不要悄悄扩大。 | 防弹衣制造商 |
| SCOPE REDUCTION | You are a surgeon. Find the minimum viable version that achieves the core outcome. Cut everything else. Be ruthless. | 你是一个外科医生。找到能实现核心成果的最小可行版本。砍掉其他所有东西。要狠。 | 外科医生 |

**第五条隐含姿态 — COMPLETENESS IS CHEAP（完整性很廉价）：**

> **原文：** "AI coding compresses implementation time 10-100x. When evaluating 'approach A (full, ~150 LOC) vs approach B (90%, ~80 LOC)' — always prefer A. The 70-line delta costs seconds with CC. 'Ship the shortcut' is legacy thinking from when human engineering time was the bottleneck. Boil the lake."
>
> **翻译：** AI 编码把实现时间压缩了 10-100 倍。当评估"方案 A（完整，~150 行代码）vs 方案 B（90%，~80 行代码）"时——永远选 A。70 行的差距在 CC 下只需几秒钟。"先上线简化版"是人力工程时代的遗留思维。煮干湖水。

这条跨越所有四种模式，是 gstack 的核心哲学"Boil the Lake"在此 Skill 中的具体体现。

**关键规则（防止模式漂移）：**
```
一旦用户选择了模式，就要全力投入。不要悄悄滑向另一种模式。
如果选了扩展，不要在后面的环节悄悄减少工作。
如果选了缩减，不要悄悄把范围加回来。
```

**迁移价值：** "模式锁定 + 防漂移" 是一个极其通用的设计模式。任何有多种行为模式的 Skill 都应该加上这条规则，否则 Claude 会在执行过程中"偷偷妥协"。

### 2.4 Prime Directives（首要指令 — 9 条不可违背的原则）+ Engineering Preferences

这 9 条指令是整个 Skill 的骨架。它们不是检查清单 — 它们是**思维原则**。

| # | 原文 | 翻译 | 为什么这条有效 |
|---|---|---|---|
| 1 | Zero silent failures. | 零静默失败。每一种失败模式都必须可见。 | 迫使 AI 追问"如果这步失败了，谁会知道？" |
| 2 | Every error has a name. | 每个错误都有名字。不要说"处理错误"。说出具体的异常类型。 | 把模糊指令变成具体行动 |
| 3 | Data flows have shadow paths. | 数据流有影子路径：空输入、零长度输入、上游错误。 | 强制 AI 考虑正常路径之外的 3 条隐藏路径 |
| 4 | Interactions have edge cases. | 交互有边界情况：双击、操作到一半离开、慢网络、过期状态、后退按钮。 | 列举了具体的边界场景，不让 AI 跳过 |
| 5 | Observability is scope, not afterthought. | 可观测性是范围内的工作，不是事后补充。 | 把"监控"从"以后再加"提升到"现在就必须规划" |
| 6 | Diagrams are mandatory. | 图表是必须的。 | 画图迫使隐含假设显性化 |
| 7 | Everything deferred must be written down. | 所有延后的工作都必须写下来。模糊的意图等于谎言。 | 防止"以后再说"变成"永远不做" |
| 8 | Optimize for the 6-month future. | 为 6 个月后优化，不只是为今天。 | 拉长时间视野，防止短视决策 |
| 9 | You have permission to say "scrap it." | 你有权说"全部推翻，换个方案"。 | 给 AI 明确的权限去说"不" |

**设计原理分析：**

注意这 9 条指令的**领域无关性**。它们没有提到任何编程语言、框架或技术。把"数据流"换成"收入流"，把"错误"换成"风险"，每一条对商业计划同样成立。

**迁移价值：** 这 9 条可以直接复用到你自己的审查类 Skill 中。它们的价值在于：把"好好审查"这个模糊指令变成了 9 个具体的、不可跳过的思维原则。

### 2.5 Cognitive Patterns（认知模式 — 潜在空间激活）

**原文：**
```
These are not checklist items. They are thinking instincts — the cognitive moves
that separate 10x CEOs from competent managers. Let them shape your perspective
throughout the review. Don't enumerate them; internalize them.
```

**翻译：** 这些不是清单项目。它们是思维本能 — 把 10 倍 CEO 和普通管理者区分开的认知动作。让它们在整个审查过程中塑造你的视角。不要列举它们；内化它们。

**18 个认知模式全译：**

| # | 模式名 | 来源人物 | 核心含义 |
|---|---|---|---|
| 1 | 分类本能 | Bezos | 按可逆性×影响力分类决策。大多数是双向门——快速决定 |
| 2 | 偏执扫描 | Grove | 持续扫描战略拐点、文化漂移、人才流失、流程变成代理 |
| 3 | 反转反射 | Munger | 对每个"怎么赢"也问"什么会让我们失败" |
| 4 | 聚焦即减法 | Jobs | 核心价值在于决定不做什么。350 个产品砍到 10 个 |
| 5 | 人优先排序 | Horowitz/Hastings | 人、产品、利润——永远按这个顺序 |
| 6 | 速度校准 | Bezos | 快是默认。只为不可逆+高影响决策减速。70% 信息就够了 |
| 7 | 代理怀疑 | Bezos Day 1 | 你的指标还在服务用户，还是已经变成自我指涉？ |
| 8 | 叙事连贯 | — | 困难决策需要清晰的框架。让"为什么"清晰，而非让所有人满意 |
| 9 | 时间纵深 | Bezos | 用 5-10 年弧线思考。80 岁时的后悔最小化 |
| 10 | 创始人模式偏向 | Chesky/Graham | 深度参与不是微管理——如果它扩展而非约束了团队的思考 |
| 11 | 战时意识 | Horowitz | 正确诊断和平时期 vs 战争时期。和平时期的习惯会杀死战时公司 |
| 12 | 勇气积累 | — | 信心来自做出艰难决策的过程，而非在它之前 |
| 13 | 意志力即战略 | Altman | 有意识地固执。世界会向在一个方向上持续用力的人让步 |
| 14 | 杠杆执念 | Altman | 找到小投入创造巨大产出的杠杆点。技术是终极杠杆 |
| 15 | 层级即服务 | — | 界面设计回答"用户应该先看到什么、然后看到什么" |
| 16 | 边界情况偏执（设计） | — | 名字 47 个字符？零结果？网络断了？首次用户 vs 老用户？ |
| 17 | 减法为默认 | Rams | "尽可能少的设计"。如果一个 UI 元素不值得它的像素，砍掉它 |
| 18 | 为信任而设计 | — | 每个界面决策要么建立要么侵蚀用户信任 |

**原文关键指令：**
```
When you evaluate architecture, think through the inversion reflex.
When you challenge scope, apply focus as subtraction.
When you assess timeline, use speed calibration.
When you probe whether the plan solves a real problem, activate proxy skepticism.
```

**翻译：** 当你评估架构时，用反转反射。当你挑战范围时，用聚焦即减法。当你评估时间线时，用速度校准。当你探究计划是否解决了真实问题时，激活代理怀疑。

**设计原理分析（极其重要）：**

这段 Prompt 的精妙之处在于**它不是在教 Claude 知识，而是在激活 Claude 已有的知识**。

Claude 的训练数据中包含了数百万字关于 Bezos、Munger、Jobs 思维方式的文本。当 Prompt 提到这些名字和框架时，LLM 的潜在空间（Latent Space）中与这些人相关的神经激活模式被触发。Claude 不是在"背诵" Bezos 的决策框架 — 它是在**以 Bezos 的思维模式运行**。

指令 "Don't enumerate them; internalize them" 确保了 Claude 不会生硬地列出"根据 Bezos 的单向门框架..."，而是自然地以这些模式为底层运行。

**迁移价值：** 这是整个 gstack 最可迁移的设计模式。你可以在任何 Skill 中加入"认知模式激活"段落。关键是：
1. 提到具体的人物和框架名称（激活潜在空间）
2. 告诉 AI "内化，不要列举"
3. 给出"在什么场景下用什么模式"的映射

### 2.6 Step 0：核弹级范围挑战

Step 0 是整个 Skill 最高杠杆的部分。它在任何审查环节之前运行。

**0A. 前提挑战（Premise Challenge）**

| 原文 | 翻译 | 为什么这个问题有效 |
|---|---|---|
| Is this the right problem to solve? | 这是对的问题吗？能不能换一种问法得到更简单的方案？ | 迫使在执行之前先质疑方向 |
| What is the actual user/business outcome? | 真正的用户/商业成果是什么？这个计划是最直接的路径，还是在解决代理问题？ | 区分"真实目标"和"代理目标" |
| What would happen if we did nothing? | 如果什么都不做会怎样？真实痛点还是假想的？ | 建立 baseline — 如果答案是"也没什么"，那计划本身就不该存在 |

**0B. 现有资源利用**

```
现有的哪些代码已经部分或完全解决了每个子问题？
这个计划是否在重建已有的东西？如果是，解释为什么重建比重构更好。
```

**0C. 理想状态映射（Dream State Mapping）**

```
CURRENT STATE    --->    THIS PLAN    --->    12-MONTH IDEAL
[当前状态]               [计划改变]            [理想状态]
```

**设计原理：** 这个三段式迫使 AI 把计划放在**更大的时间线**中评估。如果计划让你离 12 个月后的理想状态更远了，那它方向就是错的 — 即使短期看起来合理。

**0D. 模式专属分析**

**范围扩展模式增加了三个问题：**

1. **10 倍检查：** "什么版本能在 2 倍努力下带来 10 倍的价值？" — 这个问题迫使 AI 跳出渐进思维
2. **柏拉图式理想：** "如果世界上最好的工程师有无限时间和完美品味，这会长什么样？" — 设定天花板
3. **惊喜机会：** "花 30 分钟就能做的改进，让用户想'哦不错，他们想到这一点了'" — 低成本高感知价值

**扩展提案的呈现方式（opt-in ceremony）：**
```
每个扩展提案单独呈现为一次 AskUserQuestion。
选项：A) 加入计划范围  B) 延后到 TODOS.md  C) 跳过
用户逐个决定。接受的成为后续所有审查环节的范围。
拒绝的进入"不在范围内"列表。
```

**这个设计的关键：** AI 可以"做梦"，但用户保持 100% 控制。每个扩展都是显式 opt-in，不是悄悄塞进去的。

**0E. 时间线审问**

```
第 1 小时（打地基）：   实施者需要知道什么？
第 2-3 小时（核心逻辑）：他们会碰到什么模糊地带？
第 4-5 小时（集成）：    什么会让他们意外？
第 6 小时以后（打磨）：  他们会希望自己当初规划了什么？
```

注意：这些时间是人类团队时间。Prompt 要求同时展示 AI 压缩后的时间（通常 6 小时人类 → 30-60 分钟 AI）。

### 2.7 11 个审查环节（10 + 1 条件触发）

| 环节 | 审查什么 | 关键产出 |
|---|---|---|
| **1. 架构审查** | 组件边界、数据流（4 条路径）、状态机、耦合、扩展性、单点故障、回滚 | ASCII 架构图（必须） |
| **2. 错误与拯救映射** | 每个可能失败的方法的完整表格：什么会出错 → 异常类 → 是否被捕获 → 用户看到什么 | 错误/拯救表（不可跳过） |
| **3. 安全与威胁模型** | 攻击面、输入验证、授权、注入、秘密管理 | 威胁矩阵 |
| **4. 数据流与交互边界** | 数据流 5 节点追踪 + 交互边界情况表 | 边界情况清单 |
| **5. 代码质量** | DRY、命名、过度/欠工程化、圈复杂度 | 质量发现 |
| **6. 测试审查** | 测试图谱：新 UX 流 / 数据流 / 代码路径 / 后台任务 / 外部集成 / 错误路径 | 测试矩阵 |
| **7. 性能审查** | N+1、内存、索引、缓存、慢路径 p99 | 性能发现 |
| **8. 可观测性** | 日志、指标、追踪、告警、仪表盘、调试性、运维手册 | 可观测性清单 |
| **9. 部署与上线** | 迁移安全、Feature Flag、回滚计划、部署后验证 | 部署检查清单 |
| **10. 长期轨迹** | 技术债、路径依赖、可逆性评分（1-5）、1 年后的新工程师能看懂吗 | 长期评估 |
| **11. 设计与 UX** | （仅在检测到 UI 范围时）信息架构、交互状态覆盖、AI Slop 风险 | 用户流程图 |

**每个环节的关键规则：**
```
STOP. AskUserQuestion 一次一个问题。不要合并。给推荐 + 原因。
如果没问题或修复很明显，说清楚然后继续——不要浪费一个问题。
用户回复前不要继续。
```

**设计原理分析：**

虽然这 11 个环节用的是工程术语，但当输入不是工程计划时，Claude 会自动迁移：
- "架构审查" → 业务结构审查
- "单点故障" → 关键依赖（如只有一个客户占 50% 收入）
- "回滚计划" → 如果方向错了怎么退出
- "N+1 查询" → 重复低效的流程

这个迁移不是 Prompt 写的 — 是 Claude 自己做的。但 Prompt 提供了一个**完整的、不可跳过的清单结构**，确保 Claude 对每个维度都产出结论。

### 2.8 Required Outputs（必需产出）

```
- "不在范围内"章节 — 考虑过但明确延后的工作
- "已有的东西"章节 — 已经部分解决子问题的现有资源
- "理想状态差距"章节 — 计划让我们离 12 个月理想状态多远
- 错误与拯救清单 — 每个可能失败的方法的完整表格
- 故障模式清单 — 任何"未拯救=是、未测试=是、用户看到=静默"的组合 → 关键缺口
- 惊喜机会（仅扩展模式）— 至少 5 个花 30 分钟就能做的改进
```

### 2.9 模式快速参考表

| 维度 | 扩展 | 选择性扩展 | 保持范围 | 缩减 |
|---|---|---|---|---|
| 范围方向 | 往上推（opt-in） | 保持 + 逐个展示 | 保持 | 往下推 |
| 推荐姿态 | 热情推荐 | 中立展示 | 不适用 | 不适用 |
| 10 倍检查 | 必须 | 作为候选展示 | 可选 | 跳过 |
| 柏拉图式理想 | 做 | 不做 | 不做 | 不做 |
| 惊喜机会 | opt-in 仪式 | 逐个挑选仪式 | 看到就记 | 跳过 |
| 复杂度问题 | "够大吗？" | "对了吗？还有什么值得加的？" | "太复杂了吗？" | "是最小版本吗？" |
| 品味校准 | 做 | 做 | 不做 | 不做 |
| 时间线审问 | 完整（1-6 小时） | 完整（1-6 小时） | 只做关键决策 | 跳过 |
| 错误映射 | 完整 + 混沌场景 | 完整 + 已接受项的混沌场景 | 完整 | 只做关键路径 |
| CEO 计划文档 | 写入磁盘 | 写入磁盘 | 跳过 | 跳过 |
| 第 2/3 阶段规划 | 画出已接受项 | 画出已接受的挑选项 | 记下来 | 跳过 |
| 设计审查（第 11 节） | "无可逃避的"UI 审查 | 如检测到 UI 范围 | 如检测到 UI 范围 | 跳过 |

---

## 三、如何迁移到你自己的场景

### 场景 1：商业计划审查

直接使用 `/plan-ceo-review`，不需要修改。Claude 会自动把工程术语迁移到商业领域。

### 场景 2：个人决策审查

创建一个精简版 Skill，保留核心结构：
```markdown
# 重大决策审查

## 姿态
你不是来安慰我的。你是来确保我做出最好决策的。
要严厉。要找出每一个我没想到的问题。

## Step 0：前提挑战
1. 这是对的问题吗？
2. 如果什么都不做会怎样？
3. 5 年后的理想状态是什么？这个决策在靠近还是远离？

## 审查环节
1. 风险映射：什么会出错？每种失败的后果和概率？
2. 机会成本：选了这个，放弃了什么？
3. 可逆性：这是单向门还是双向门？
4. 时间线：1 个月后 / 1 年后 / 5 年后会怎样？
```

### 场景 3：团队 OKR 审查

从 plan-ceo-review 提取核心框架：
- 姿态定义（严厉的对抗性审查）
- 前提挑战（这些是对的目标吗？）
- 理想状态映射（OKR 在靠近还是远离长期愿景？）
- 模式选择（扩展/保持/缩减）
- 每个 KR 的"如果失败了"映射

### 关键提取物（可复用到任何审查类 Skill）

1. **姿态定义：** 1-2 句话设定审查态度
2. **前提挑战（Step 0）：** 3 个元问题先于任何具体审查
3. **模式选择 + 防漂移：** 让用户选择姿态，然后锁定
4. **认知模式激活：** 提到具体人物/框架，指令"内化，不要列举"
5. **不可跳过的流程：** 每个环节必须产出结论，即使是"无问题"
6. **一次一问：** 永远不合并多个决策
7. **必需产出模板：** 明确定义审查结束后必须交付什么

---

> 本分析基于 `plan-ceo-review/SKILL.md.tmpl` 完整源码（~600 行）逐段拆解。
