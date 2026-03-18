# /retro 深度解构

> **角色：** 工程经理（Eng Manager — 回顾模式）
> **定位：** 团队感知的周回顾，基于 commit 数据的定量分析 + 定性叙事
> **Prompt 长度：** ~513 行 | **allowed-tools：** Bash, Read, Write, Glob, Grep, AskUserQuestion
> **来源：** `retro/SKILL.md.tmpl`

---

## 一、为什么这个 Skill 有效

大多数回顾（retro）是定性的——大家坐在一起说"感觉这周挺忙"。`/retro` 的核心创新是：**从 commit 历史中提取定量数据，然后基于数据写叙事**。

每一句赞美都锚定在具体的 commit 上。每一个改进建议都有数据支撑。不是"你做得很好"，而是"你本周 32 个 commit，+2.4k LOC，测试比例 41%，最大交付是 cookie 导入系统"。

### 为什么 AI 特别适合做这件事？

1. **数据收集**：12 条 git 命令并行执行，人类需要 30 分钟手工汇总
2. **模式识别**：从 commit 时间戳推断工作 session、识别深夜编码模式、计算连续交付天数
3. **叙事生成**：把枯燥的统计数据转化为有人情味的回顾报告
4. **多人分析**：自动识别每个贡献者，分别给出赞美和成长建议

---

## 二、完整结构拆解

### 2.1 参数解析

```
/retro           → 默认 7 天
/retro 24h       → 最近 24 小时
/retro 14d       → 最近 14 天
/retro 30d       → 最近 30 天
/retro compare   → 与上一期对比
/retro compare 14d → 14 天窗口 + 对比
```

### 2.2 Step 1：数据收集（12 条并行 git 命令）

```bash
git shortlog --summary --numbered --since="7 days ago"    # 每人 commit 数
git log --since="7 days ago" --format="%H %an" --stat     # 每 commit 的文件/LOC
git log --since="7 days ago" --format="%aI"               # 时间戳（session 检测）
git log --since="7 days ago" --diff-filter=M --name-only  # 热点文件
# ... 共 12 条命令
```

**设计原理：** 并行执行最大化效率。对大型 repo 来说，12 条 git 命令串行可能需要 30 秒，并行只需最慢那条的时间。

### 2.3 Step 2-8：指标计算

| 指标 | 数据源 | 含义 |
|---|---|---|
| Commits | shortlog | 总提交数 |
| Contributors | shortlog | 贡献者数 |
| LOC +/- | diff --stat | 代码增减 |
| Test ratio | 文件名模式匹配 | 测试文件 vs 生产文件比例 |
| Sessions | 时间戳聚类 | 工作会话数（45 分钟间隔为分界） |
| LOC/active hour | LOC ÷ 活跃时间 | 编码效率 |
| Streak | 连续有 commit 的天数 | 连续交付天数 |
| Hotspots | 文件变更频率 | 变更 5+ 次的文件 = 代码热点 |
| PR sizes | diff --stat per PR | 小(<100)/中(100-500)/大(500-1500)/超大(1500+) |
| Fix ratio | commit 类型分析 | fix commit 占比（>50% 是警告信号） |
| Test health | 测试文件计数 + 回归测试 commit | 测试健康度趋势 |
| Backlog health | TODOS.md 分析 | P0/P1 数量、本期完成/新增数量 |

### 2.4 Session 检测（认知会话）

**算法：** 45 分钟间隔作为会话分界线。

```
commit 9:00 → commit 9:15 → commit 9:45 → [60 分钟空白] → commit 10:45 → commit 11:00
|________________ Session 1 (45 min) ________________|                |__ Session 2 (15 min) __|
```

**会话分类：**
- Deep（深度）：50+ 分钟
- Medium（中度）：20-50 分钟
- Micro（微型）：<20 分钟

**设计原理：** 45 分钟阈值基于认知科学——大多数人的深度工作在被打断后需要 15-25 分钟重新进入状态。45 分钟的空白几乎一定意味着"停下来做别的事了"。

### 2.5 Step 9：团队成员分析

**对"你"（当前用户）的分析：** 完整的第一人称深度剖析——commit 数、LOC、测试比例、高峰时段、最大交付、做得好的地方。

**对队友的分析（每人 2-3 句）：**

```
### Alice
12 个 commit 集中在 app/services/。每个 PR 低于 200 LOC——纪律性很强。
Praise: 连续 3 天交付支付系统重构，测试比例 45%。
Opportunity: 测试比例整体 12%——在支付变得更复杂之前值得投入。
```

**关键规则：**

> **原文：** "Praise must be specific and anchored in actual commits, never comparative. 'Great week' is banned. 'Shipped the cookie import system with 18 unit tests in one focused push' is good."
>
> **翻译：** 赞美必须具体且锚定在实际 commit 上，绝不能比较。"这周做得很好"是禁止的。"在一次集中冲刺中交付了 cookie 导入系统并附带 18 个单元测试"是好的。

**AI 协作追踪：** 如果 commit 有 `Co-Authored-By` trailer（AI 辅助的标志），单独统计 AI 辅助 commit 数量。

### 2.6 Step 12-13：历史对比与持久化

**JSON 快照：** 每次运行保存到 `.context/retros/{YYYY-MM-DD}-{seq}.json`，包含所有指标、作者统计、版本范围、连续天数等。

**对比模式：** `/retro compare` 加载上一期快照，计算每个指标的 delta（↑/↓），生成对比叙事。

### 2.7 Step 14：叙事输出

**第一行：Tweetable Summary（可发推特的一句话总结）**

```
Week of Mar 1: 47 commits (3 contributors), 3.2k LOC, 38% tests,
12 PRs, peak: 10pm | Streak: 47d
```

**后续章节：**
- Summary Table（指标总表）
- Trends（趋势）
- Time & Session Patterns（时间与会话模式）
- Shipping Velocity（交付速度）
- Code Quality Signals（代码质量信号）
- Test Health（测试健康度）
- Focus & Highlights（聚焦与亮点）
- Your Week（你的一周）
- Team Breakdown（团队拆解——每人）
- Top 3 Team Wins（团队 3 大胜利）
- 3 Things to Improve（3 个改进点）
- 3 Habits for Next Week（下周 3 个习惯）

### 2.8 语气设计

> **原文：** "Encouraging but candid. Specific and concrete — always anchored in commits. Skip generic praise. Frame improvements as leveling up. Genuine 1:1 tone for praise, investment-framing for growth suggestions."
>
> **翻译：** 鼓励但坦诚。具体且实在——永远锚定在 commit 上。跳过空泛赞美。把改进建议框架为"升级"。赞美用真诚的一对一语气，成长建议用"投资"框架。

---

## 三、迁移价值

### 可复用的设计模式

1. **数据驱动叙事**：先收集定量数据，再基于数据写定性分析。AI 擅长这种"数据→故事"的转化
2. **Session 检测算法**：任何基于时间戳的活动分析都可以用"间隔阈值 → 会话分割 → 分类"这个模式
3. **赞美锚定规则**：所有正面评价必须附具体证据。这个规则可以迁移到任何评审/反馈类 Skill
4. **Tweetable Summary**：强制把复杂分析压缩为一句话。对任何报告类 Skill 都有价值
5. **JSON 持久化 + 对比**：保存历史数据 → 下次运行加载 → 计算 delta → 趋势可视化

### 迁移场景

**个人周报 Skill：** 保留 session 检测、commit 分析、Tweetable Summary，去掉团队分析部分。

**团队健康仪表盘 Skill：** 保留团队分析、测试健康度、Backlog Health，增加 Jira/Linear 集成。

---

> 本分析基于 `retro/SKILL.md.tmpl`（513 行）源码逐段分析。
