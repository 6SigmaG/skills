# gstack 深度解构与学习指南

> 基于 gstack v0.6.4.0 源码逐文件分析 | Garry Tan（Y Combinator CEO）
>
> 本文比你能找到的任何 gstack 介绍都更深入。它不仅解释 gstack **做什么**，更拆解它**为什么有效** — 从 Prompt Engineering 的底层原理到可以直接复用的设计模式。

---

## 目录

- [Layer 0：30 秒理解 gstack](#layer-030-秒理解-gstack)
- [Layer 1：先感受，再理解](#layer-1先感受再理解)
  - [1.1 一个功能从想法到上线](#11-一个功能从想法到上线)
  - [1.2 30 分钟动手实验](#12-30-分钟动手实验)
  - [1.3 13 个角色全景](#13-13-个角色全景)
- [Layer 2：为什么有效 — 设计原理深度拆解](#layer-2为什么有效--设计原理深度拆解)
  - [2.1 核心发现：姿态 > 内容](#21-核心发现姿态--内容)
  - [2.2 五个设计哲学](#22-五个设计哲学)
  - [2.3 Browse 浏览器引擎：关键工程决策](#23-browse-浏览器引擎关键工程决策)
  - [2.4 模板系统：Prompt 即代码](#24-模板系统prompt-即代码)
  - [2.5 三层测试金字塔](#25-三层测试金字塔)
- [Layer 3：从零开发你自己的 Skill](#layer-3从零开发你自己的-skill)
  - [3.1 Claude Code Skill 是什么](#31-claude-code-skill-是什么)
  - [3.2 最小可行 Skill：5 分钟写一个](#32-最小可行-skill5-分钟写一个)
  - [3.3 从 gstack 提取的 7 个设计模式](#33-从-gstack-提取的-7-个设计模式)
  - [3.4 完整开发流程：从构思到部署](#34-完整开发流程从构思到部署)
  - [3.5 常见误区](#35-常见误区)
- [附录：速查参考](#附录速查参考)

---

# Layer 0：30 秒理解 gstack

**gstack 把一个 AI 助手变成一支 13 人的虚拟工程团队。**

没有 gstack：你说"帮我加个功能"→ AI 直接写代码 → 直接提交。
有 gstack：CEO 重新思考问题 → 工程经理画架构图 → 写代码 → Staff 工程师找 Bug → QA 打开真实浏览器测试 → 发布工程师创建 PR → 技术作家更新文档。

七个命令。一个人。一支团队的产出。

---

# Layer 1：先感受，再理解

> **TL;DR** — 本层通过一个完整示例和一个动手实验，让你在 15 分钟内**体感**到 gstack 的认知模式切换效果，然后再看 13 个角色的全景。

## 1.1 一个功能从想法到上线

以"给卖家加照片上传功能"为例：

```
你：我想给卖家加照片上传功能。

你：/plan-ceo-review
Claude（CEO 模式）：
  "照片上传"不是真正的功能。真正的任务是帮卖家创建能卖出去的商品列表。
  如果我们能从照片自动识别产品、拉取规格和价格对比、自动生成标题描述呢？
  那是 10 星产品。"上传照片"只有 3 星。你选哪个？
  [提出 8 个扩展提案，你逐个决定接受/延后/跳过]

你：/plan-eng-review
Claude（工程经理模式）：
  ┌─────────┐    ┌──────────┐    ┌──────────┐    ┌─────────┐
  │ Upload  │───▶│ Classify │───▶│ Enrich   │───▶│ Draft   │
  │ (sync)  │    │ (async)  │    │ (async)  │    │ (async) │
  └─────────┘    └──────────┘    └──────────┘    └─────────┘
  [架构图、状态图、数据流图、14 个测试场景、6 个失败模式]

你：批准计划，退出 Plan Mode。
  Claude 写了 2400 行代码，跨 11 个文件 — 约 8 分钟。

你：/review
Claude（Staff 工程师模式）：
  [AUTO-FIXED] 上传失败时 S3 孤儿文件清理
  [AUTO-FIXED] listings.status 缺少索引
  [ASK] hero 图片选择的竞态条件 → 你确认修复
  3 个问题 — 2 个自动修复，1 个询问后修复。

你：/qa https://staging.myapp.com
Claude（QA 模式）：
  [打开真实 Chromium 浏览器，登录，上传照片，点击各流程]
  发现 Bug: 第二次上传时预览不清除 — 已修复并提交
  自动生成回归测试。

你：/ship
Claude（发布工程师模式）：
  测试: 42 → 51 (+9 new)  |  覆盖率: 14/14 路径 (100%)
  PR: github.com/you/app/pull/42
```

**关键观察：** 同一个 Claude，在不同 Skill 下表现截然不同。CEO 模式在发散，工程模式在收敛，Review 模式在找漏洞，QA 模式在系统测试。这不是 magic — 这是 Prompt Engineering 的工业化实践。

## 1.2 30 分钟动手实验

> 这是最高效的学习方式。gstack 的价值只能通过亲手体验来传递。

### 准备工作（5 分钟）

1. 确保你有 [Claude Code](https://docs.anthropic.com/en/docs/claude-code)、[Git](https://git-scm.com/)、[Bun](https://bun.sh/) v1.0+
2. 安装 gstack — 在 Claude Code 中粘贴：
```
安装 gstack: 运行 git clone https://github.com/garrytan/gstack.git ~/.claude/skills/gstack && cd ~/.claude/skills/gstack && ./setup
```

### 实验 1：感受 CEO 模式（5 分钟）

在你自己的任何项目中：
1. 让 Claude 先通过 README 或本地文件了解你的项目
2. 输入 `/plan-ceo-review`
3. 描述一个你正在考虑的功能或业务计划
4. 观察 Claude 的"姿态转变" — 它不再是一个顺从的助手，而是在挑战你的前提假设

**你可能会注意到的事情：**
- 它问的第一个问题是"这是对的问题吗？"而不是"怎么实现？"
- 即使你描述的不是工程项目（比如商业计划），它的审查框架依然有效
- 每个问题单独提问，不会一次扔给你 10 个

### 实验 2：感受 Review 模式（5 分钟）

在一个有代码变更的分支上：
1. 输入 `/review`
2. 观察它自动获取 diff、读取审查清单、运行两轮检查
3. 注意它区分了"自动修复"和"询问你"两种处理方式

### 实验 3：感受浏览器 QA（10 分钟）

如果你有一个可以访问的 staging 环境或本地 dev server：
1. 输入 `/qa http://localhost:3000`（替换为你的 URL）
2. 观察 Claude 打开真实浏览器、截图、点击按钮、填表单
3. 如果发现 Bug，它会修复代码、提交、重新验证

### 实验 4：读一个 Skill 的 Prompt（5 分钟）

```bash
cat ~/.claude/skills/gstack/plan-ceo-review/SKILL.md | head -100
```

带着你刚才的使用体验来读这个 Prompt。注意它**没有**教 Claude 任何领域知识 — 它定义的是"姿态"和"流程"。

## 1.3 13 个角色全景

gstack 的 13 个 Skill（技能指令）按软件开发生命周期排列：

```
                        ┌──────────────────────────────┐
                        │  /plan-ceo-review (CEO)      │
  规划阶段              │  /plan-eng-review (工程经理)   │
  Plan Mode             │  /plan-design-review (设计师) │
                        │  /design-consultation (设计)  │
                        └──────────┬───────────────────┘
                                   │
                        ┌──────────▼───────────────────┐
  编码阶段              │  Claude Code 正常编码          │
                        └──────────┬───────────────────┘
                                   │
                        ┌──────────▼───────────────────┐
  审查阶段              │  /review (Staff 工程师)       │
                        │  /design-review (设计审查)     │
                        └──────────┬───────────────────┘
                                   │
                        ┌──────────▼───────────────────┐
  测试阶段              │  /qa (QA + 修复)              │
  (用真实浏览器)        │  /qa-only (只报告不修复)       │
                        │  /browse (浏览器底层能力)      │
                        │  /setup-browser-cookies       │
                        └──────────┬───────────────────┘
                                   │
                        ┌──────────▼───────────────────┐
  发布阶段              │  /ship (发布 + PR)            │
                        │  /retro (周回顾)              │
                        │  /document-release (文档更新)  │
                        └──────────────────────────────┘
```

**关键洞察：Skill 之间有数据流。**
- `/plan-eng-review` 的测试计划会自动被 `/qa` 读取
- `/review` 的 Greptile 评论分类会传递给 `/ship`
- 所有 plan review 的结果会写入 Review Readiness Dashboard（审查就绪仪表盘），`/ship` 发布前会自动检查
- `/retro` 读取 commit 历史和 TODOS.md 来生成回顾报告

这不是 13 个独立工具，这是一个有状态的流水线。

### 各角色速览

| 阶段 | Skill | 一句话 |
|---|---|---|
| 规划 | `/plan-ceo-review` | 挑战前提假设，找到 10 星版本。4 种模式：扩展/选择性扩展/保持/缩减 |
| 规划 | `/plan-eng-review` | ASCII 架构图 + 失败模式 + 测试矩阵。强制隐含假设显性化 |
| 规划 | `/plan-design-review` | 7 个维度评分 0-10，逐个修复到 8+。检测 AI Slop |
| 规划 | `/design-consultation` | 从零构建设计系统。研究竞品、提出安全选择 vs 冒险选择 |
| 审查 | `/review` | 找 CI 抓不到但生产会爆的 Bug。明显的自动修复，有歧义的问你 |
| 审查 | `/design-review` | 80 项视觉审查 + 逐个修复。每个修复一个 commit，可二分 |
| QA | `/qa` | 打开真实浏览器测试，发现 Bug 就修复 + 提交 + 验证 + 生成回归测试 |
| QA | `/qa-only` | 同上但只报告不修复。给团队交一份干净的 Bug 报告 |
| QA | `/browse` | 底层浏览器能力。~100ms/命令，50+ 个命令 |
| QA | `/setup-browser-cookies` | 从 Chrome/Arc/Brave/Edge 导入 Cookie，测试登录后页面 |
| 发布 | `/ship` | 同步 main → 跑测试 → 审查覆盖率 → 推送 → 创建 PR。一个命令 |
| 发布 | `/retro` | 每人贡献分析、测试健康度、发布节奏、增长建议 |
| 发布 | `/document-release` | 对比 diff 更新所有过时文档 |

---

# Layer 2：为什么有效 — 设计原理深度拆解

> **TL;DR** — gstack 的核心秘密不是给 AI 领域知识，而是给 AI 一个**严格的认知姿态 + 领域无关的元问题 + 不可跳过的流程**。本层拆解 5 个设计哲学和 3 个核心技术系统。

## 2.1 核心发现：姿态 > 内容

这是理解 gstack **为什么有效**的钥匙。

我用 `/plan-ceo-review` 审查了一个商业计划 — 不是工程项目，是商业模式。这个 Skill 的 Prompt 里完全没有提到商业领域。但它在一分钟内给了一个真的能帮我提升收入的解决方案。

我仔细读了 Prompt 源码（约 600 行），发现它做了三件事：

### 第一：定义审查的姿态（Posture），而不是审查的内容

```markdown
你不是来给这个计划盖橡皮章的。你是来让它变得卓越的，
在每颗地雷爆炸之前把它找出来。
```

它没有告诉 AI 要审查什么指标。它告诉 AI 的是：**你要以什么态度面对这个计划** — 严厉的、对抗性的、不留情面的。

这个姿态放在工程项目上管用，放在商业计划上也管用。因为它约束的是 AI 的**行为模式**，跟领域无关。

四种姿态对应四种模式：

| 模式 | 姿态隐喻 | Prompt 原文 |
|---|---|---|
| 范围扩展 | 你在建造一座大教堂 | "You are building a cathedral. You have permission to dream." |
| 选择性扩展 | 你是有品味的严谨审查者 | "Hold scope as baseline, surface every expansion opportunity" |
| 保持范围 | 你是防弹衣制造商 | "Make it bulletproof. Do not silently reduce OR expand." |
| 范围缩减 | 你是外科医生 | "Find the minimum viable version. Cut everything else. Be ruthless." |

### 第二：Step 0 的问题是领域无关的元问题（Meta-Questions）

```
1. 这是对的问题吗？
2. 真正的用户/商业成果是什么？这个计划是最直接的路径，还是在解决一个代理问题？
3. 如果什么都不做会怎样？这是真实的痛点，还是假想的？
4. 12 个月后的理想状态是什么？这个计划在靠近还是远离那个状态？
```

没有一个词提到工程或商业。它们问的是更底层的事情：**你确定你在解决对的事情吗？**

当我把商业计划丢给这个工具时，Claude 读到的指令是"先挑战前提，再审查细节"，然后它用**自己已有的商业知识**按照这个指令执行。

### 第三：工程术语被 Claude 自动迁移到其他领域

10 个审查环节的原始语境是工程的："架构审查""单点故障""回滚计划"。但当输入是商业计划时：

| Prompt 原始术语 | Claude 自动迁移到商业领域 |
|---|---|
| 架构审查 | 业务结构审查 |
| 单点故障 | 业务依赖的单一渠道 |
| 回滚计划 | 如果这个方向不行怎么退出 |
| 数据流的影子路径 | 收入来源的脆弱性分析 |
| 错误恢复 | 失败后的止损方案 |

这个迁移不是 Prompt 写的，是 Claude 自己做的。但 Prompt 给了一个**不允许跳过任何环节的完整清单结构**，Claude 沿着这个结构逐步审查。

**这就是 gstack 的核心秘密：**

> 它没有给 AI 领域知识。它给了 AI 一个严厉的姿态、一组领域无关的元问题、和一个不允许跳过任何环节的流程。AI 自己填充领域知识。

### 为什么这比给 AI 领域知识更强？

因为你给的领域知识永远是有限的，而 LLM 训练数据中的知识是海量的。

如果你在 Prompt 里写"检查这 10 个商业指标"，Claude 就只检查那 10 个。但如果你写"用最大的严谨度检查每一个潜在失败点"，Claude 会调用它**所有**相关知识来执行这条指令。

**Prompt 的最高形态不是给 AI 更多信息，而是给 AI 正确的行为框架，让它自己调用已有的海量知识。**

## 2.2 五个设计哲学

### 哲学 1：Boil the Lake（煮干湖水） — 完整性原则

> AI 让完整实现的边际成本接近零。80% 方案和 100% 方案的差距可能只有 70 行代码 — 用 AI 写这 70 行只需几秒钟。**永远选 100%。**

| 任务类型 | 人类团队 | AI+gstack | 压缩比 |
|---|---|---|---|
| 脚手架/模板代码 | 2 天 | 15 分钟 | ~100x |
| 写测试 | 1 天 | 15 分钟 | ~50x |
| 功能实现 | 1 周 | 30 分钟 | ~30x |
| Bug 修复 + 回归测试 | 4 小时 | 15 分钟 | ~20x |

gstack 在每个 AskUserQuestion（提问环节）中都显示"完整度评分（Completeness Score）"：
- **10/10** = 完整实现（所有边界情况、完整覆盖）— 这是"湖"，可以煮干
- **7/10** = 覆盖正常路径但跳过部分边界 — 别选这个
- **3/10** = 快捷方式 — 绝对别选
- "海洋"（从头重写整个系统）不属于讨论范围

### 哲学 2：认知模式切换 — Latent Space Activation（潜在空间激活）

`/plan-ceo-review` 的 Prompt 里提到了真实的人和框架：

```markdown
## Cognitive Patterns — How Great CEOs Think
这些不是清单项目。它们是思维本能。不要列举它们；内化它们。

1. 分类本能 — 按可逆性 × 影响力分类每个决策（Bezos 单向门/双向门）
2. 偏执扫描 — 持续扫描战略拐点、文化漂移、人才流失（Grove）
3. 反转反射 — 对每个"我们怎么赢？"也问"什么会让我们失败？"（Munger）
4. 聚焦即减法 — Jobs 从 350 个产品砍到 10 个
5. 速度校准 — 快是默认。只为不可逆+高影响决策减速。70% 信息就够了（Bezos）
...
```

这不是让 Claude 背公式。指令明确说了"Don't enumerate them; internalize them."

**原理：** LLM 的训练数据中包含了大量关于 Bezos、Munger、Grove 思维方式的文本。提到这些名字和框架等于在 LLM 的潜在空间（Latent Space）中**激活**了这些思维模式的神经路径。Claude 不是在"背诵" Bezos 的决策框架 — 它是在**像 Bezos 那样思考**。

`/plan-eng-review` 同样激活了不同的工程思维：Brooks（本质复杂度 vs 偶然复杂度）、Beck（先让变更变容易，再做变更）、Google SRE（错误预算）。

### 哲学 3：Fix-First（先修后报） — 动作优先于报告

传统 Code Review 工具：列出 20 个发现，你自己决定怎么办。
gstack 的 `/review`：

```
[AUTO-FIXED] app/models/post.rb:42 — 死代码 → 已删除
[AUTO-FIXED] app/services/upload.rb:88 — N+1 查询 → 已优化
[ASK] app/services/payment.rb:47 — 竞态条件：并发扣款可能重复
  → A) 加数据库级锁  B) 跳过
```

分类规则写在 `review/checklist.md` 里：
- **机械修复**（死代码、N+1 查询、缺失索引）→ 自动修复，告诉你做了什么
- **需要判断**（安全问题、竞态条件、设计决策）→ 问你，给推荐选项

### 哲学 4：面向 Agent 的错误消息

gstack 的浏览器引擎不是给人用的 — 它是给 AI Agent 用的。每条错误消息都包含"下一步该做什么"：

```
传统：   "Element not found"
gstack： "Element not found or not interactable.
          Run `snapshot -i` to see available elements."

传统：   "Timeout"
gstack： "Navigation timed out after 30s.
          The page may be slow or the URL may be wrong."
```

Agent 读到这条消息后能**自行恢复**，不需要人类干预。这让 `/qa` 可以自动运行 20+ 步的测试流程而不中断。

### 哲学 5：一次提问，一个决策

gstack 的所有 Skill 共享一个统一的 AskUserQuestion 格式：

```
1. 重新定位：说明项目名、当前分支、当前任务（假设你 20 分钟没看这个窗口了）
2. 大白话解释：聪明的 16 岁高中生能听懂。不用函数名、不用行话
3. 推荐：推荐选择 X，因为___
4. 选项：A) ... B) ... C) ...
```

关键规则：**一个问题 = 一次提问。** 永远不把多个决策合并成一次提问。这确保了用户在多个会话窗口之间切换时不会迷路。

## 2.3 Browse 浏览器引擎：关键工程决策

> **TL;DR** — gstack 的浏览器不是每次冷启动的，而是一个持久化的 Chromium 守护进程（Daemon），通过 localhost HTTP 通信。这让每个命令从 2-3 秒降到 ~100ms，且 Cookie/登录态跨命令保持。

### 架构

```
Claude Code                     gstack Browse
─────────                      ─────────────
                               ┌──────────────────────┐
  Bash: $B snapshot -i    →    │  CLI (编译二进制 58MB) │
                               │  读 .gstack/browse.json│
                               │  POST /command         │
                               └──────────┬────────────┘
                                          │ HTTP + Bearer Token
                               ┌──────────▼────────────┐
                               │  Bun.serve() 服务器    │
                               │  命令分发 + Chromium 操控│
                               └──────────┬────────────┘
                                          │ Playwright (CDP)
                               ┌──────────▼────────────┐
                               │  Chromium（无头模式）   │
                               │  持久化 tabs + cookies  │
                               │  30 分钟空闲自动退出    │
                               └────────────────────────┘
```

### 为什么是守护进程而不是每次冷启动？

| | 冷启动模式 | 守护进程模式（gstack） |
|---|---|---|
| 每命令延迟 | 2-3 秒 | ~100ms |
| 20 步 QA 额外开销 | 40-60 秒 | ~0 |
| Cookie/登录态 | 每次丢失 | 跨命令保持 |
| localStorage | 每次丢失 | 跨命令保持 |

### Ref 系统：AI 怎么"指"网页上的元素？

这是 gstack 最精妙的设计。

**问题：** AI 要点击网页上的按钮，怎么告诉浏览器"点哪个"？用 CSS 选择器？AI 很容易猜错。

**gstack 的方案：** 先跑 `snapshot`（快照），给每个可交互元素分配一个编号（@e1, @e2...），然后用编号操作。

```bash
$B snapshot -i           # 拍快照
# @e1 [heading] "Welcome"
# @e2 [textbox] "Email"
# @e3 [button] "Submit"

$B fill @e2 "test@example.com"   # 用编号操作
$B click @e3
```

**为什么不在 DOM 里注入 `data-ref` 属性？**
- 很多网站的 CSP（内容安全策略）禁止外部修改 DOM
- React/Vue 的水合（Hydration）过程会删掉注入的属性
- Shadow DOM 从外部不可达

gstack 用 Playwright 的 Locator（定位器），基于 ARIA 可访问性树，完全在 DOM 外部工作。

**过期检测：** SPA 应用可以在不触发导航的情况下改变页面（如 React Router）。gstack 在使用每个 Ref 前做一次 `count()` 检查（~5ms），如果元素不存在了就立刻报错，而不是等 Playwright 的 30 秒超时。

### 安全模型

| 层面 | 措施 |
|---|---|
| 网络隔离 | 绑定 localhost，不可外部访问 |
| 认证 | 每次启动生成 UUID Token，状态文件权限 0o600 |
| Cookie 安全 | 内存中解密，从不写入磁盘明文，不显示 Cookie 值 |
| 注入防护 | 浏览器列表硬编码，用参数数组不用字符串拼接 |

### 为什么用 Bun 而不是 Node.js？

| 原因 | 细节 |
|---|---|
| 编译二进制 | `bun build --compile` → 单文件 58MB 可执行，无需 node_modules |
| 原生 SQLite | Cookie 解密需要读 Chromium 的 SQLite 数据库 |
| 原生 TypeScript | 开发时直接跑 .ts，无需编译 |
| 内置 HTTP | `Bun.serve()` 够用，不需要 Express |

## 2.4 模板系统：Prompt 即代码

> **TL;DR** — SKILL.md 不是手写的，而是从 `.tmpl` 模板 + 源码元数据自动生成的。这确保了 Prompt 中的命令参考永远和代码同步。

### 手写文档的问题

文档说有某个命令 → 代码已经删了 → Agent 执行失败。
代码加了新命令 → 文档没更新 → Agent 不知道可以用。

### gstack 的方案

```
SKILL.md.tmpl    ← 人工写的 Prompt + 占位符
       ↓
gen-skill-docs.ts ← 构建脚本（读源码元数据）
       ↓
SKILL.md          ← 自动生成，提交到 git
```

占位符示例：

| 占位符 | 数据来源 | 效果 |
|---|---|---|
| `{{COMMAND_REFERENCE}}` | `commands.ts` 注册表 | 命令存在则出现在文档，不存在则消失 |
| `{{SNAPSHOT_FLAGS}}` | `snapshot.ts` 元数据 | Flag 文档自动和解析器同步 |
| `{{PREAMBLE}}` | 生成器 | 所有 Skill 共享的启动代码 |
| `{{QA_METHODOLOGY}}` | 生成器 | `/qa` 和 `/qa-only` 共享 QA 方法论 |

**为什么提交生成文件？** 因为 Claude 在调用 Skill 时直接读 SKILL.md —— 没有构建步骤。同时 CI 可以用 `--dry-run` 验证文件是否最新。

### 模板编写的核心规则

如果你想修改 gstack 或写自己的模板化 Skill：

1. **用自然语言传递状态** — 每个 bash 代码块在独立 shell 中运行，变量不能跨块
2. **动态检测分支** — 不要硬编码 `main`，用 `{{BASE_BRANCH_DETECT}}`
3. **条件逻辑用英语** — "If X, do Y. Otherwise, do Z." 而不是嵌套 if/elif
4. **每个代码块独立可运行** — 不依赖上一个块的状态

## 2.5 三层测试金字塔

| 层级 | 成本 | 速度 | 验证什么 |
|---|---|---|---|
| **Tier 1** 静态验证 | 免费 | <5 秒 | 从 SKILL.md 提取所有 `$B` 命令，对照注册表验证 |
| **Tier 2** E2E 测试 | ~$3.85 | ~20 分钟 | 启动真实 Claude 会话（`claude -p` 子进程），运行每个 Skill |
| **Tier 3** LLM 评审 | ~$0.15 | ~30 秒 | Claude Sonnet 对文档评分：清晰度/完整度/可操作性，每项 ≥4/5 |

**精妙之处：** 95% 的问题被免费的 Tier 1 捕获。Tier 2/3 只用来验证"Skill 在真实 Claude 会话中是否正常工作"和"文档质量是否足够"。

**Diff-based 选择：** 每个测试声明它依赖的文件。只改了 `/retro` 的分支只跑 2 个测试，不是 31 个。`bun run eval:select` 可以预览。

---

# Layer 3：从零开发你自己的 Skill

> **TL;DR** — 本层是一个从零到部署的完整教程。你会学到 Skill 的机制、7 个可复用的设计模式、和 5 个常见误区。

## 3.1 Claude Code Skill 是什么

Skill 就是一个 Markdown 文件（`SKILL.md`），放在 `~/.claude/skills/` 目录下。

```
~/.claude/skills/
├── my-skill/
│   └── SKILL.md      ← Claude Code 发现这个文件 → 注册为 /my-skill
├── another-skill/
│   └── SKILL.md      ← 注册为 /another-skill
└── gstack/
    ├── review/
    │   └── SKILL.md   ← 通过符号链接注册为 /review
    └── ...
```

当你输入 `/my-skill` 时，Claude Code 读取这个 SKILL.md 文件，把它作为**额外的系统指令**注入到当前对话中。本质上就是一个可切换的超级 System Prompt。

## 3.2 最小可行 Skill：5 分钟写一个

```bash
mkdir -p ~/.claude/skills/my-reviewer
```

创建 `~/.claude/skills/my-reviewer/SKILL.md`：

```markdown
---
name: my-reviewer
version: 0.1.0
description: |
  审查当前分支的代码变更，专注于安全问题和性能问题。
allowed-tools:
  - Bash
  - Read
  - Grep
---

# 代码安全与性能审查

你是一个偏执的安全工程师。你的工作不是赞美代码，而是找出每一个潜在的安全漏洞和性能瓶颈。

## 步骤 1：获取变更

```bash
git diff main --stat
git diff main
```

## 步骤 2：安全审查

对 diff 中的每个文件，检查：
1. SQL 注入：任何字符串拼接的查询
2. XSS：任何未转义的用户输入渲染
3. 认证绕过：任何缺少权限检查的端点
4. 秘密泄露：任何硬编码的密钥或密码

## 步骤 3：性能审查

1. N+1 查询：循环中的数据库调用
2. 缺失索引：WHERE 条件中的未索引字段
3. 大量内存分配：一次性加载整个表

## 输出格式

对每个发现：
- [CRITICAL/HIGH/MEDIUM] 文件:行号 — 问题描述
- 建议修复方案
```

**测试它：** 在一个有代码变更的项目中输入 `/my-reviewer`。

**恭喜 — 你刚刚创建了一个 Skill。** 本质上就是这么简单。

## 3.3 从 gstack 提取的 7 个设计模式

### 模式 1：姿态定义（Posture Definition）

不要告诉 AI **检查什么**。告诉 AI **以什么态度检查**。

```markdown
# ❌ 弱 Prompt
检查代码中的安全问题、性能问题和可维护性问题。

# ✅ 强 Prompt（gstack 风格）
你不是来给这个代码盖橡皮章的。你是来阻止生产事故的。
想象你是 on-call 工程师，凌晨 3 点被这段代码叫醒了。
你会问什么问题？
```

姿态约束的是行为模式，不是知识范围。这让同一个 Prompt 可以跨领域工作。

### 模式 2：领域无关的元问题（Meta-Questions）

gstack Step 0 的问题没有提到任何具体领域：

```markdown
1. 这是对的问题吗？
2. 如果什么都不做会怎样？
3. 12 个月后的理想状态是什么？
```

这些问题迫使 AI 在执行任何审查之前先**挑战前提**。你可以在任何 Skill 中复用这个模式。

### 模式 3：环境感知的 Preamble（前言启动块）

```markdown
## 前言（首先运行）

```bash
echo "BRANCH: $(git branch --show-current 2>/dev/null || echo unknown)"
echo "PROJECT: $(basename $(git rev-parse --show-toplevel 2>/dev/null) || echo unknown)"
```

在指令开始前，先让 Agent 感知当前环境。gstack 的 Preamble 还做了：更新检查、多会话计数（3+ 个窗口时进入"简化解释"模式）、贡献者模式。
```

### 模式 4：一次一问（One Question Per Ask）

```markdown
## 关键规则
- 一个问题 = 一次提问。永远不要合并
- 假设用户 20 分钟没看这个窗口
- 每个问题附推荐选项 + 理由
- 用户回复前不要继续下一步
```

这保证了在多窗口切换时不会迷路。

### 模式 5：不可跳过的流程（Non-Skippable Checklist）

```markdown
## 关键规则
不允许跳过任何环节。如果某个环节没有发现问题，
明确说明"此环节无问题"然后继续。
```

这迫使 AI 对**每一步**都产出结论，而不是偷懒跳过看起来不重要的部分。

### 模式 6：Fix-First 分类

```markdown
对每个发现，分类为：
- AUTO-FIX（机械修复）：直接修复，输出 [AUTO-FIXED] 文件:行 问题 → 做了什么
- ASK（需要判断）：批量提问，附推荐
```

让 AI 自己能处理的问题先处理掉，只把真正需要人类判断的问题抛出来。

### 模式 7：模板占位符（DRY Across Skills）

如果多个 Skill 共享同一段逻辑：

```
# 在 .tmpl 模板中
{{QA_METHODOLOGY}}     ← 构建时从生成器注入
{{BROWSE_SETUP}}       ← 浏览器设置指令
{{BASE_BRANCH_DETECT}} ← 分支检测逻辑
```

一处修改，所有 Skill 同步更新。

## 3.4 完整开发流程：从构思到部署

### Step 1：确定你的 Skill 要解决什么问题

问自己：
- 我反复在做什么手动操作？
- 我希望 Claude 用什么**态度**做这件事？
- 这件事需要什么工具？（Bash? Edit? 浏览器?）

### Step 2：写 SKILL.md

```bash
mkdir -p ~/.claude/skills/my-skill
```

用 3.2 的最小模板开始。YAML frontmatter 里的 `allowed-tools` 决定了这个 Skill 能用什么工具。

### Step 3：迭代测试

1. 在 Claude Code 中输入 `/my-skill`
2. 观察 Claude 的行为是否符合你的预期
3. 修改 SKILL.md，重新测试
4. 重复直到满意

### Step 4：（可选）添加模板系统

如果你有多个 Skill 且它们共享逻辑，可以参考 gstack 的 `gen-skill-docs.ts` 搭建模板系统。但对大多数个人 Skill 来说，直接编辑 SKILL.md 就够了。

### Step 5：分享

把 Skill 目录复制到项目的 `.claude/skills/` 下并提交：
```bash
cp -r ~/.claude/skills/my-skill .claude/skills/my-skill
git add .claude/skills/my-skill
```

团队成员 `git clone` 后自动获得这个 Skill。

## 3.5 常见误区

### 误区 1：给 AI 太多领域知识，太少行为框架

```markdown
# ❌ 长长的检查清单
检查以下 47 个安全项：1) SQL 注入 2) XSS 3) CSRF ...

# ✅ 行为框架
你是安全审计员。假设每一行代码都是攻击面。
对每个输入，追问：它从哪来？验证了吗？失败时拒绝了吗？
```

47 项清单限制了 AI 的视野。行为框架释放了 AI 的全部知识。

### 误区 2：在 bash 代码块之间传递变量

```markdown
# ❌ 不工作（每个代码块是独立 shell）
```bash
BRANCH=$(git branch --show-current)
```
```bash
echo "当前分支是 $BRANCH"    # 变量丢失！
```

# ✅ 用自然语言传递
运行上面的命令获取分支名。在后续步骤中使用这个分支名。
```

### 误区 3：模式漂移（Mode Drift）

gstack 的一条关键规则：

```markdown
一旦用户选择了模式，就要全力投入。不要悄悄滑向另一种模式。
```

如果你的 Skill 有多种模式，必须显式锁定。否则 Claude 会在执行过程中"偷偷妥协" — 比如在"范围扩展"模式中悄悄缩减范围。

### 误区 4：一次提太多问题

```markdown
# ❌ 信息轰炸
这里有 8 个问题需要你决定...

# ✅ gstack 风格
一个问题 = 一次 AskUserQuestion。逐个来。
```

### 误区 5：忘记"如果什么都不做会怎样？"

大多数 Skill 直接跳入"怎么做"。gstack 教我们的是：先问"该不该做"。在你的 Skill 中加一个 Step 0 来挑战前提，可以避免大量无效工作。

---

# 附录：速查参考

### gstack 技术栈

| 技术 | 用途 |
|---|---|
| **Bun** | JS 运行时 + 编译器（生成二进制） |
| **TypeScript** | Browse 引擎源码 |
| **Playwright** | Chromium 浏览器自动化（by Microsoft） |
| **Markdown** | Skill 定义格式 |

### 常用命令

| 命令 | 用途 |
|---|---|
| `./setup` | 一键安装 |
| `bun run build` | 生成文档 + 编译二进制 |
| `bun test` | 免费测试（<5 秒） |
| `bun run gen:skill-docs` | 从模板重新生成 SKILL.md |
| `bin/dev-setup` / `bin/dev-teardown` | 进入/退出开发模式 |

### 核心文件

```
gstack/
├── SKILL.md.tmpl              ← /browse 的 Prompt 模板
├── setup                      ← 安装脚本
├── package.json               ← 依赖：playwright, diff
├── browse/src/                ← 浏览器引擎（14 个 TypeScript 文件）
│   ├── cli.ts                 ← CLI 入口
│   ├── server.ts              ← HTTP 服务器
│   ├── browser-manager.ts     ← Chromium + Ref 管理
│   ├── commands.ts            ← 命令注册表（唯一数据源）
│   └── snapshot.ts            ← ARIA 树 → @ref 映射
├── scripts/gen-skill-docs.ts  ← 模板生成器
├── {skill-name}/SKILL.md.tmpl ← 各 Skill 的 Prompt 模板（14 个）
├── review/checklist.md        ← /review 的审查清单 + Fix-First 规则
└── test/                      ← 三层测试套件
```

> 完整文件列表见 [GitHub 仓库](https://github.com/garrytan/gstack)。

### 关键概念中英对照

| 英文术语 | 中文解释 |
|---|---|
| Skill | 技能指令 — 一个 SKILL.md 文件定义的 Claude 行为模式 |
| Ref (@e1, @e2) | 引用标识符 — 浏览器中可交互元素的编号 |
| Snapshot | 快照 — 页面可访问性树的结构化输出 |
| Preamble | 前言 — 每个 Skill 开头的通用启动代码块 |
| Plan Mode | 规划模式 — Claude Code 的"先想后做"模式 |
| Boil the Lake | 煮干湖水 — 完整性原则：AI 时代完整方案的成本接近零 |
| AI Slop | AI 垃圾 — AI 生成的套路化 UI（紫色渐变、三列网格等） |
| Fix-First | 先修后报 — 机械问题自动修复，只问真正需要判断的 |
| Latent Space Activation | 潜在空间激活 — 提到特定人/框架来激活 LLM 的深层知识模式 |
| Review Readiness Dashboard | 审查就绪仪表盘 — 发布前显示哪些审查已完成 |
| Conductor | 指挥器 — 并行运行多个 Claude Code 会话的第三方工具 |

---

> **本文基于 gstack v0.6.4.0 源码逐文件分析。** 所有技术细节对照 ARCHITECTURE.md、CLAUDE.md、CONTRIBUTING.md、14 个 SKILL.md.tmpl 模板源码、browse/src/ 全部 14 个 TypeScript 文件验证。
