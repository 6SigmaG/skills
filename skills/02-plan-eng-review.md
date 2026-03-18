# /plan-eng-review 深度解构

> **角色：** Engineering Manager / 工程经理模式
> **定位：** 架构审查、失败模式分析、测试矩阵生成、完整性保障
> **Prompt 长度：** ~240 行 | **allowed-tools：** Read, Write, Grep, Glob, Bash, AskUserQuestion
> **来源：** `plan-eng-review/SKILL.md.tmpl`

---

## 一、为什么这个 Skill 有效

如果说 `/plan-ceo-review` 是"该不该做"的审查，那 `/plan-eng-review` 就是"能不能做好"的审查。它解决的是一个工程团队中最常见的问题：**计划看起来合理，但执行时才发现架构有洞、测试没覆盖、失败路径没想过。**

这个 Skill 有效的核心原因有三个：

### 第一：它激活的是工程领袖的思维模式，不是检查清单

Prompt 中引用了 15 个认知模式，每一个都来自真实的工程领袖或方法论。这不是装饰——这是**潜在空间激活（Latent Space Activation）**。当 Prompt 提到 Brooks 的"本质复杂度 vs 偶然复杂度"时，Claude 训练数据中关于《人月神话》的数百万字文本被激活。它不是在背诵 Brooks 的框架，它是在**以 Brooks 的方式审视架构**。

关键指令是：

> **原文：** "These are not additional checklist items. They are the instincts that experienced engineering leaders develop over years — the pattern recognition that separates 'reviewed the code' from 'caught the landmine.' Apply them throughout your review."
>
> **翻译：** 这些不是额外的检查清单。它们是经验丰富的工程领袖多年积累的直觉——把"审查了代码"和"发现了地雷"区分开的模式识别能力。在整个审查过程中应用它们。

### 第二：它用范围挑战作为入口，而不是直接开始审查

大多数工程审查工具会直接跳到"看代码"。但 `/plan-eng-review` 的第一件事是**质疑范围本身**——8+ 文件的变更触发复杂度警告，迫使你解释为什么不能拆分。这个设计来自一个深刻的工程洞察：**最昂贵的工程错误不是代码 Bug，而是做了不该做的事情。**

### 第三：它生成的测试计划是可消费的产物

`/plan-eng-review` 的测试矩阵不是写完就扔的文档——它被写入 `~/.gstack/projects/` 目录，供 `/qa` 直接读取和执行。这意味着规划阶段的思考直接流入测试阶段，不会在交接中丢失。

**这是 gstack 流水线设计的精髓：Skill 之间有数据流，不是孤立的工具。**

---

## 二、完整结构拆解

### 2.1 认知模式（15 个工程思维激活）

这是整个 Skill 的底层操作系统。15 个认知模式不是要 Claude 逐条检查的——它们塑造 Claude 在整个审查过程中的**思维底色**。

| # | 模式名 | 来源 | 原文核心（逐字引用） | 中文含义 | 应用场景 |
|---|---|---|---|---|---|
| 1 | 状态诊断 | Larson | "Teams exist in four states: falling behind, treading water, repaying debt, innovating. Each demands a different intervention." | 团队有四种状态：掉队、勉强维持、还债、创新。每种需要不同干预 | 审查计划时先问：这个团队处于什么状态？ |
| 2 | 爆炸半径直觉 | — | "Every decision evaluated through 'what's the worst case and how many systems/people does it affect?'" | 每个决策都要问"最坏情况是什么？影响多少系统/多少人？" | 评估变更影响范围 |
| 3 | 无聊即默认 | McKinley | "Every company gets about three innovation tokens." Everything else should be proven technology. | 每家公司大约只有三枚创新代币。其他一切都应该是成熟技术 | 看到新框架/新语言时触发质疑 |
| 4 | 渐进优于革命 | Fowler | "Strangler fig, not big bang. Canary, not global rollout. Refactor, not rewrite." | 绞杀者模式，而非大爆炸。金丝雀发布，而非全量上线。重构，而非重写 | 审查变更策略是否足够渐进 |
| 5 | 系统优于英雄 | — | "Design for tired humans at 3am, not your best engineer on their best day." | 为凌晨 3 点疲惫的人设计，而非为最佳状态的最优工程师设计 | 评估方案是否依赖个人英雄主义 |
| 6 | 可逆性偏好 | — | "Feature flags, A/B tests, incremental rollouts. Make the cost of being wrong low." | Feature flag、A/B 测试、渐进上线。让犯错成本尽可能低 | 部署和迁移策略审查 |
| 7 | 失败即信息 | Allspaw, Google SRE | "Blameless postmortems, error budgets, chaos engineering. Incidents are learning opportunities, not blame events." | 无责事后分析、错误预算、混沌工程。事故是学习机会，不是追责事件 | 失败模式分析时关注系统性弱点 |
| 8 | 组织结构即架构 | Skelton/Pais | "Conway's Law in practice. Design both intentionally." | 康威定律的实践。有意识地同时设计组织和架构 | 审查组件边界是否与团队结构匹配 |
| 9 | DX 即产品质量 | — | "Slow CI, bad local dev, painful deploys → worse software, higher attrition." | 慢 CI、差本地开发、痛苦部署 → 更差的软件、更高的流失 | 评估开发者体验是否被考虑 |
| 10 | 本质 vs 偶然复杂度 | Brooks | "Before adding anything: 'Is this solving a real problem or one we created?'" | 在加任何东西之前问："这是在解决真实问题还是我们自己创造的问题？" | 复杂度检查时追问复杂度的来源 |
| 11 | 两周气味测试 | — | "If a competent engineer can't ship a small feature in two weeks, you have an onboarding problem disguised as architecture." | 如果一个称职的工程师两周内无法交付小功能，你有一个伪装成架构的入职问题 | 评估系统的可上手性 |
| 12 | 胶水工作意识 | Reilly | "Recognize invisible coordination work. Value it, but don't let people get stuck doing only glue." | 识别隐形的协调工作。重视它，但别让人只做胶水工作 | 评估计划是否产生不可见的协调负担 |
| 13 | 先让变更变容易 | Beck | "Refactor first, implement second. Never structural + behavioral changes simultaneously." | 先重构，再实现。永远不要同时做结构性和行为性变更 | 架构审查时评估变更策略 |
| 14 | 在生产环境拥有你的代码 | Majors | "No wall between dev and ops." | 开发和运维之间没有墙 | 评估运维责任是否被规划 |
| 15 | 错误预算优于正常运行时间目标 | Google SRE | "SLO of 99.9% = 0.1% downtime *budget to spend on shipping*." | 99.9% SLO = 0.1% 的停机时间预算可以用来发布新功能 | 可靠性决策时用预算思维而非完美主义 |

**原文关键指令：**

> "When evaluating architecture, think 'boring by default.' When reviewing tests, think 'systems over heroes.' When assessing complexity, ask Brooks's question. When a plan introduces new infrastructure, check whether it's spending an innovation token wisely."
>
> **翻译：** 评估架构时，想"无聊即默认"。审查测试时，想"系统优于英雄"。评估复杂度时，问 Brooks 的问题。当计划引入新基础设施时，检查是否明智地花费了创新代币。

**设计原理分析：**

注意这些认知模式的选择是经过精心设计的。`/plan-ceo-review` 用的是 CEO 级别的思维模式（Bezos、Munger、Jobs），而 `/plan-eng-review` 用的是**工程领袖和方法论**（Brooks、Beck、Google SRE）。这不是随意的——不同角色需要激活不同的潜在空间区域。

**迁移价值：** 如果你创建面向特定职能的 Skill，选择**那个职能中最受尊重的思想家**作为认知模式来源。这些名字在训练数据中有巨大的影响面积。

### 2.2 范围挑战与复杂度检查

在进入任何具体审查之前，`/plan-eng-review` 先做一次**范围级别的质疑**。

**8+ 文件气味检测（Complexity Smell）：**

> **原文：** "Complexity check: If the plan touches more than 8 files or introduces more than 2 new classes/services, treat that as a smell and challenge whether the same goal can be achieved with fewer moving parts."
>
> **翻译：** 复杂度检查：如果计划涉及超过 8 个文件或引入超过 2 个新的类/服务，将其视为一个气味（smell），并质疑能否用更少的活动部件实现同样的目标。

**为什么是 8 个文件？**

这不是随意的数字。在代码审查的实践中，超过 8 个文件的 PR 几乎必然会有以下问题：
- 审查者的注意力无法覆盖所有文件之间的交互
- 回滚变得困难——你不确定能安全地只回滚部分变更
- 测试覆盖率容易出现盲区——每增加一个文件，交叉路径指数级增长

这个检查不是说"不能改 8 个以上文件"——它说的是"**你需要解释为什么**"。

**范围挑战的三个维度：**

| 维度 | 问题 | 为什么有效 |
|---|---|---|
| 必要性 | 这个变更解决的问题是真实的吗？有更简单的方案吗？ | Beck 的"先让变更变容易"——也许你需要的是重构，不是新功能 |
| 可分割性 | 能拆成更小的独立交付吗？ | 渐进交付原则——每个切片都能独立上线并交付价值 |
| 偶然复杂度 | 有多少复杂度是问题固有的？多少是设计引入的？ | Brooks 的本质 vs 偶然复杂度——如果大部分复杂度是偶然的，设计有问题 |

**迁移价值：** "N 文件气味检测"这个模式可以迁移到任何审查类 Skill 中。关键不是具体的数字，而是**为超出阈值的情况设一个强制解释点**。

### 2.3 四大审查区块

这是 `/plan-eng-review` 的核心审查流程。四个区块按从宏观到微观的顺序排列：

#### 区块一：Architecture（架构审查）

> **原文（来自 Documentation and diagrams 部分）：** "I value ASCII art diagrams highly — for data flow, state machines, dependency graphs, processing pipelines, and decision trees. Use them liberally in plans and design docs."
>
> **翻译：** 我非常重视 ASCII 图表——用于数据流、状态机、依赖图、处理管道和决策树。在计划和设计文档中大量使用它们。
>
> 注意："画不出来就是没理解"是对源码设计意图的合理推导，但并非原文。

**架构审查的子项：**

| 检查项 | 审查什么 | 关键问题 |
|---|---|---|
| 组件边界 | 每个组件的职责是否清晰、单一？ | "如果我把这个组件删掉，谁会受影响？" |
| 数据流（4 条路径） | 正常路径 + 空输入 + 上游错误 + 部分失败 | "数据从 A 到 B 的每一步，失败了会怎样？" |
| 状态管理 | 状态在哪里？谁拥有它？并发访问安全吗？ | "两个请求同时修改这个状态会怎样？" |
| 耦合度 | 组件之间的依赖是松的还是紧的？ | "改一个组件是否必须同时改其他组件？" |
| 扩展性 | 当负载增加 10 倍时会发生什么？ | "哪个组件先成为瓶颈？" |
| 单点故障 | 有没有一个组件挂了整个系统就挂了？ | "Redis 挂了会怎样？数据库只读了会怎样？" |
| 回滚策略 | 部署出问题后能否安全回滚？ | "如果有数据库迁移，回滚会丢数据吗？" |

**ASCII 图表的强制要求是精心设计的。** 画图迫使 Claude 把隐含的架构假设**显性化**。很多时候，画图的过程本身就能暴露设计缺陷——"等等，这两个组件之间怎么通信？我没定义过这个接口。"

#### 区块二：Code Quality（代码质量审查）

> **源码实际内容（Section 2: Code quality review）：** 该环节审查代码组织、DRY 违反、错误处理模式、技术债热点，以及过度/不足工程化。结合工程偏好中的 "I want code that's 'engineered enough'" 原则。
>
> 注意：源码中没有"Quality is not about style..."这段引文。以上是对源码审查内容的准确描述。

**代码质量审查的关键维度：**

| 维度 | 审查什么 | 判断标准 |
|---|---|---|
| DRY（不重复自己） | 是否有重复逻辑？ | 重复逻辑在 3+ 处出现就需要抽象 |
| 命名 | 变量、函数、类的命名是否准确传达意图？ | 读名字就知道它做什么，不需要看实现 |
| 过度工程 | 是否为不存在的需求设计了抽象？ | "这个抽象解决了今天的什么问题？" |
| 欠工程 | 是否为了快而跳过了必要的抽象？ | "下一个开发者改这段代码时会骂人吗？" |
| 圈复杂度 | 函数中的分支路径是否过多？ | 圈复杂度 > 10 需要拆分 |

这里的关键洞察来自 Beck：

> **原文 (Beck)：** "Make the change easy, then make the easy change."
>
> **翻译：** 先让变更变容易，再做那个容易的变更。

这意味着代码质量审查不只是看当前代码好不好——更要看**它是否让未来的变更变容易了**。

#### 区块三：Tests（测试审查）

测试审查是 `/plan-eng-review` 最有产出价值的环节，因为它生成的测试矩阵会被写入项目文件供 `/qa` 消费。

> **原文（Section 3: Test review）：** "Make a diagram of all new UX, new data flow, new codepaths, and new branching if statements or outcomes. For each, note what is new about the features discussed in this branch and plan. Then, for each new item in the diagram, make sure there is a JS or Rails test."
>
> **翻译：** 画出所有新 UX、新数据流、新代码路径和新的分支条件或结果的图表。对每项标注本分支和计划中有什么是新的。然后，对图表中的每一项，确保有 JS 或 Rails 测试。

**测试图谱（Test Topology）：**

| 测试类型 | 覆盖什么 | 例子 |
|---|---|---|
| UX 流测试 | 用户从页面 A 到页面 B 的完整流程 | 用户注册 → 邮件验证 → 首次登录 |
| 数据流测试 | 数据从输入到存储到输出的完整路径 | 上传照片 → 处理 → CDN → 显示 |
| 代码路径测试 | 函数/方法的每个分支路径 | if/else、switch 的每个分支 |
| 后台任务测试 | 异步任务的执行和失败处理 | 队列任务超时、重试、死信处理 |
| 外部集成测试 | 第三方 API 的正常/异常响应 | Stripe 支付成功、失败、超时 |
| 错误路径测试 | 每种错误类型的处理和用户反馈 | 网络断开、权限不足、数据格式错误 |

**测试计划写入 ~/.gstack/projects/ 的设计：**

```
~/.gstack/projects/{project-name}/
├── test-plan.md        ← /plan-eng-review 生成
├── test-results.md     ← /qa 执行后更新
└── review-status.md    ← 审查就绪仪表盘
```

这个设计让规划与执行之间产生了**有状态的数据流**。`/qa` 启动时会自动读取 `test-plan.md`，按照测试矩阵逐项执行，而不是凭空猜测应该测什么。

**迁移价值：** 如果你创建多个互相关联的 Skill，让前一个 Skill 的产出成为后一个 Skill 的输入。具体方式是通过文件系统（写入约定路径），而不是通过环境变量或内存——因为 Claude Code 的每个 Bash 块是独立 shell。

#### 区块四：Performance（性能审查）

> **源码实际内容（Section 4: Performance review）：** 审查 N+1 查询和数据库访问模式、内存使用、缓存机会、慢/高复杂度代码路径。
>
> 注意：源码中没有"Performance is not about making everything fast..."这段哲学性引文。源码的风格是直接列出检查项，不做抽象概括。

**性能审查的核心检查项：**

| 检查项 | 审查什么 | 典型问题 |
|---|---|---|
| N+1 查询 | 循环中的数据库调用 | `users.each { |u| u.posts.count }` → 1000 用户 = 1001 条 SQL |
| 内存使用 | 是否一次性加载大量数据？ | `User.all.map(&:name)` 把整张表加载到内存 |
| 索引缺失 | WHERE 条件中的字段有索引吗？ | `WHERE status = 'active' AND created_at > ?` 需要复合索引 |
| 缓存策略 | 重复计算是否被缓存？缓存是否会过期？ | 缓存穿透、缓存雪崩、缓存与数据库不一致 |
| 慢路径 p99 | 最慢的 1% 请求有多慢？ | 正常请求 50ms，但包含 100+ 子项的请求要 5 秒 |

这里的关键是 Google SRE 的**错误预算**思维：

> **原文（认知模式 #15）：** "SLO of 99.9% = 0.1% downtime *budget to spend on shipping*. Reliability is resource allocation (Google SRE)."
>
> **翻译：** 99.9% 的 SLO = 0.1% 的停机时间预算可以花在发布上。可靠性是资源分配问题。

这意味着性能审查不是追求极致性能——而是设定**务实的性能预算**，然后确保计划不会超出这个预算。

### 2.4 一次一问规则（One-Issue-Per-AskUserQuestion）

> **原文：** "STOP. For each issue found in this section, call AskUserQuestion individually. One issue per call. Present options, state your recommendation, explain WHY. Do NOT batch multiple issues into one AskUserQuestion. Only proceed to the next section after ALL issues in this section are resolved."
>
> **翻译：** 停。对本环节发现的每个问题，单独调用 AskUserQuestion。一次一个问题。展示选项，给出推荐，解释为什么。不要把多个问题合并到一个 AskUserQuestion 中。在本环节所有问题解决之前不要进入下一环节。

这条规则看似简单，但它解决了一个深层问题：**信息过载导致决策质量下降。**

当 AI 一次扔给你 5 个问题时，你的注意力被分散在 5 件事上。你倾向于对每个问题都做快速判断，而不是深入思考。但工程审查中的每个决策都可能有长期影响——匆忙做出的决定可能在 6 个月后变成技术债。

**一次一问的连锁效果：**
1. 用户被迫对每个问题深入思考
2. 每个决策都有完整的上下文（不会因为看到第 5 个问题时忘了第 1 个问题的上下文）
3. 决策历史清晰可追溯（每个 AskUserQuestion 就是一个决策点）
4. 多窗口切换时不会迷路（每个问题都有完整的重新定位信息）

**提问格式（统一规范）：**

```
1. 重新定位：项目名 | 分支 | 当前审查环节
2. 大白话解释：聪明的 16 岁高中生能听懂
3. 推荐：推荐选择 X，因为___
4. 选项：A) ... B) ... C) 跳过
```

**迁移价值：** 一次一问规则应该成为所有交互式 Skill 的默认行为。它的成本极低（多几次交互），但收益极高（决策质量显著提升）。

### 2.5 失败模式分析（Failure Mode Analysis）

这是 `/plan-eng-review` 最独特的设计之一——它要求**每条代码路径至少有一个现实的生产失败场景**。

> **原文（来自 Architecture review 和 Failure modes 部分）：** "For each new codepath or integration point, describe one realistic production failure scenario and whether the plan accounts for it." / "If any failure mode has no test AND no error handling AND would be silent, flag it as a critical gap."
>
> **翻译：** 对每个新的代码路径或集成点，描述一个现实的生产失败场景以及计划是否考虑了它。/ 如果任何失败模式没有测试、没有错误处理、且会静默发生，标记为关键缺口。
>
> 注意：源码中没有"It's 2 AM. PagerDuty fires..."这段叙事化描述。原文更简洁直接，要求的是"一个现实的生产失败场景"而非"故事"。

**为什么要求"故事"而不是"列表"？**

这是 Allspaw 的事后分析方法论在发挥作用。当你写"可能出现超时"时，这是抽象的、可以被跳过的。但当你写"凌晨 2 点，上传服务的 S3 连接超时，用户看到的是一个无限旋转的进度条，重试 3 次后静默失败，照片丢失但数据库记录显示上传成功"——这是具体的、痛苦的、不可忽视的。

**失败模式表格格式：**

| 代码路径 | 失败场景（故事） | 失败类型 | 用户看到什么 | 恢复策略 | 是否被捕获 | 是否被测试 |
|---|---|---|---|---|---|---|
| 照片上传 | S3 超时，重试耗尽，前端无反馈 | 静默失败 | 无限进度条 | 需要手动重试 | 否 → **关键缺口** | 否 |
| 照片处理 | 图片格式异常，处理器 OOM | 进程崩溃 | "处理失败"错误页 | 队列自动重试 | 是 | 是 |

**关键规则：**

> **原文（Failure modes 部分）：** "If any failure mode has no test AND no error handling AND would be silent, flag it as a **critical gap**."
>
> **翻译：** 如果任何失败模式没有测试、没有错误处理、且会静默发生，标记为**关键缺口**。

这个规则把失败模式分析从"参考信息"提升到了"阻塞条件"。如果有静默的、未捕获的、未测试的失败路径，审查不能通过。

**迁移价值：** "每条路径一个失败故事"的要求可以迁移到任何涉及风险评估的 Skill 中。故事比清单更有说服力，因为它们激活了 Claude 的叙事能力，产出更具体、更有用的分析。

### 2.6 Boil-the-Lake 完整性哲学

> **原文（Step 0, Completeness check）：** "With AI-assisted coding, the cost of completeness (100% test coverage, full edge case handling, complete error paths) is 10-100x cheaper than with a human team. If the plan proposes a shortcut that saves human-hours but only saves minutes with CC+gstack, recommend the complete version. Boil the lake."
>
> **翻译：** 使用 AI 辅助编码时，完整性的成本（100% 测试覆盖、完整边界处理、完整错误路径）比纯人力团队便宜 10-100 倍。如果计划提出的捷径省的是人力时间但用 CC+gstack 只省几分钟，推荐完整版本。煮干湖水。

"Boil the Lake"（煮干湖水）是 gstack 的核心哲学之一，但在 `/plan-eng-review` 中它有特殊的表现形式：

**完整性在工程审查中的含义：**

| 维度 | 80% 方案（不可接受） | 100% 方案（目标） |
|---|---|---|
| 测试覆盖 | 只测正常路径 | 正常路径 + 边界 + 错误路径 + 并发 |
| 错误处理 | `catch (e) { console.log(e) }` | 具名异常 + 用户可见错误 + 恢复策略 + 监控 |
| 文档 | "以后再写" | 接口文档 + 架构决策记录（ADR） |
| 可观测性 | "出了问题看日志" | 结构化日志 + 指标 + 追踪 + 告警 + 仪表盘 |
| 边界情况 | "正常使用不会遇到" | 空输入、超长输入、并发、网络中断、权限变更 |

**为什么"以后再补"是谎言：**

> **原文（来自 Prime Directives）：** "Everything deferred must be written down. Vague intentions are lies."
>
> **翻译：** 所有延后的工作都必须写下来。模糊的意图等于谎言。

工程团队中最常见的技术债来源不是"做了坏的决定"——而是"说了'以后再做'然后忘了"。`/plan-eng-review` 要求所有被推迟的工作必须写入明确的 TODOS.md，带有优先级和截止日期。

**完整度评分的应用：**

和 `/plan-ceo-review` 一样，每次 AskUserQuestion 都会显示完整度评分：

```
完整度评分：7/10
- 正常路径：覆盖 ✓
- 空输入处理：未规划 ✗
- 上游错误传播：未规划 ✗
- 并发安全：未评估 ✗
推荐：选 A（完整实现），因为缺失的 3 项总共约 40 行代码，AI 补完只需 1 分钟。
```

这个评分机制把"完整性"从抽象概念变成了**可量化、可比较的具体度量**。

### 2.7 审查产出物与 /qa 的数据流

`/plan-eng-review` 的最终产出不只是一份审查报告——它是一组结构化的产物（artifacts），设计为被下游 Skill 消费。

**产出物清单：**

| 产出物 | 写入位置 | 消费者 | 内容 |
|---|---|---|---|
| 架构图（ASCII） | 计划文件内 | 人类审查者 | 组件边界、数据流、状态机 |
| 测试计划 | `~/.gstack/projects/{project}/test-plan.md` | `/qa` | 测试矩阵、优先级、预期结果 |
| 失败模式清单 | 计划文件内 | `/review`, 人类 | 每条路径的失败场景和恢复策略 |
| 性能预算 | 计划文件内 | `/review` | p99 延迟目标、内存上限、查询预算 |
| 审查就绪状态 | `~/.gstack/projects/{project}/review-status.md` | `/ship` | 哪些审查已完成、哪些有未解决问题 |

**测试计划的格式规范：**

```markdown
# Test Plan: [Feature Name]
Generated by /plan-eng-review on [date]

## Critical Paths (must pass before merge)
- [ ] 用户上传照片 → 处理成功 → 显示在列表中
- [ ] 上传失败 → 用户看到明确错误 → 可重试

## Edge Cases (must pass before ship)
- [ ] 上传 0 字节文件 → 拒绝 + 错误消息
- [ ] 上传 50MB 文件（超过限制）→ 拒绝 + 大小提示
- [ ] 处理中断电 → 重启后自动重试

## Performance Assertions
- [ ] 单张照片上传 < 3s (p99)
- [ ] 10 张批量上传 < 15s (p99)
```

**迁移价值：** 如果你开发自己的 Skill 体系，**文件系统是最可靠的 Skill 间通信机制**。约定路径 + 约定格式，不需要任何特殊的 IPC 机制。Claude Code 的每个 Bash 块可以读写文件，这就是你需要的全部。

---

## 三、如何迁移到你自己的场景

### 场景 1：后端 API 审查 Skill

从 `/plan-eng-review` 提取核心框架，聚焦 API 特有的审查维度：

```markdown
# API 审查

## 姿态
你是一个偏执的 API 架构师。每个端点都是攻击面。每个响应都是承诺。

## 认知模式
- Brooks：这个 API 的复杂度是问题固有的还是设计引入的？
- Google SRE：这个 API 的 SLO 是什么？错误预算是多少？
- Beck：这个 API 设计让未来的变更变容易了吗？

## 审查区块
1. 契约审查：请求/响应格式是否有版本控制？
2. 错误处理：每个错误码是否有意义？客户端能用它做什么？
3. 幂等性：重复调用会产生副作用吗？
4. 速率限制：被滥用时会怎样？
5. 向后兼容：现有客户端会崩溃吗？
```

### 场景 2：数据管道审查 Skill

提取失败模式分析和数据流审查，适配数据工程场景：

```markdown
# 数据管道审查

## 姿态
你是凌晨 3 点的值班数据工程师。管道坏了。你需要在 15 分钟内知道坏在哪里、影响了什么、怎么修。

## 关键检查
1. 每个节点：上游延迟了会怎样？数据格式变了会怎样？
2. 完整性检查：怎么知道今天的数据是完整的？
3. 幂等性：重跑会产生重复数据吗？
4. 回填策略：发现 3 天前的数据有问题，能回填吗？
```

### 场景 3：非技术项目管理审查

把 4 个审查区块迁移到项目管理领域：

| 工程审查区块 | 项目管理迁移 |
|---|---|
| Architecture | 项目结构：工作分解结构（WBS）是否合理？依赖关系是否清晰？ |
| Code Quality | 交付质量：每个里程碑的验收标准是否明确？ |
| Tests | 风险测试：每个关键假设是否有验证计划？ |
| Performance | 资源效率：人力分配是否合理？关键路径在哪里？ |

### 关键提取物（可复用到任何工程审查类 Skill）

1. **认知模式激活：** 选择目标职能中最受尊重的思想家，提到具体名字和框架
2. **复杂度气味检测：** 为超出阈值的情况设强制解释点（8 文件、N 个依赖等）
3. **四区块结构：** 从宏观到微观——架构 → 质量 → 测试 → 性能
4. **一次一问：** 每个决策点单独提问，附推荐和理由
5. **失败故事：** 每条路径至少一个现实的生产失败场景，不是抽象的风险列表
6. **产出物流向下游：** 测试计划写入约定路径，供后续 Skill 消费
7. **完整性评分：** 量化当前方案的完整度，让"80% 够了"的想法无处躲藏
8. **防漂移机制：** 审查模式一旦确定就锁定，不允许在过程中悄悄降低标准

---

> 本分析基于 `/plan-eng-review` 的 Skill 设计原理和 gstack 整体架构逐项拆解。认知模式的选择、审查区块的排列、产出物的流转，每一个设计都有明确的工程理由。
