# /autoplan 深度解构

> **角色：** 审查流水线（Review Pipeline）
> **定位：** 一条命令完成 CEO → 设计 → 工程审查，用 6 条决策原则自动替代人类判断
> **allowed-tools：** Bash, Read, Write, Edit, Glob, Grep（注意：**没有 AskUserQuestion**）
> **来源：** `autoplan/SKILL.md`

---

## 一、为什么这个 Skill 存在

**问题：** 完整运行 /plan-ceo-review → /plan-design-review → /plan-eng-review 需要回答 15-30 个中间问题。这很彻底，但对已经知道自己要什么的用户来说太慢了。

**传统方案：** 跳过审查，直接编码。代价是错过架构问题和设计缺陷。

**gstack 的方案：** 用编码的决策原则自动回答大部分中间问题，只把"品味决策"（两个选项都合理、边界情况、Codex 不同意）留给用户在最终审批门控处一次性决定。

---

## 二、4 阶段工作流

```
Phase 1: CEO 审查（策略与范围）
  ├── 从磁盘加载 plan-ceo-review/SKILL.md
  ├── 用 6 条决策原则自动回答
  └── 输出：前提挑战 + 范围决策
           │
           ▼
Phase 2: 设计审查（条件性——无 UI 则跳过）
  ├── 从磁盘加载 plan-design-review/SKILL.md
  └── 输出：设计评分 + AI slop 检测
           │
           ▼
Phase 3: 工程审查 + Codex
  ├── 从磁盘加载 plan-eng-review/SKILL.md
  ├── 架构图、测试矩阵、失败模式
  ├── Codex 独立审查（如可用）
  └── 输出：架构方案 + 测试计划
           │
           ▼
Pre-Gate 验证
  ├── 检查所有必需输出是否完整
  └── 缺失 → 补产出（最多 2 次重试）
           │
           ▼
Phase 4: 最终审批门控
  ├── 计划摘要
  ├── 决策审计轨迹（N 个自动决策 + K 个需你选择）
  ├── 品味决策列表（逐个呈现）
  └── 审查评分
```

---

## 三、6 条决策原则

autoplan 不是"跳过审查"——它用这 6 条原则**替代你的判断**（不是跳过分析）：

1. **完整性优先：** 两个选项都可行时，选更完整的
2. **范围保持：** 默认不扩展也不缩减范围
3. **安全优先：** 安全相关问题选最保守方案
4. **简单优先：** 架构选择选最简单的
5. **测试完整：** 测试相关选全覆盖
6. **Codex 尊重：** Codex 和 Claude 不一致时，标记为品味决策

---

## 四、关键设计：没有 AskUserQuestion

autoplan 的 `allowed-tools` 中**故意没有 AskUserQuestion**。这迫使它必须用原则自动决策，而不是"不确定就问用户"。唯一的用户交互在最终门控处。

---

## 五、与其他 Skill 的关系

| 关系 | 说明 |
|------|------|
| `/plan-ceo-review` | autoplan 从磁盘读取并执行 |
| `/plan-design-review` | autoplan 从磁盘读取并执行（条件性） |
| `/plan-eng-review` | autoplan 从磁盘读取并执行 |
| `/codex` | 工程审查阶段集成 Codex 独立审查 |
| `/office-hours` | autoplan 可以在 CEO 审查前先运行 /office-hours |

---

## 六、Prompt Engineering 可迁移模式

### 模式 1：编码决策原则
把"需要人类判断"的场景分为"可编码"和"真正需要品味"两类——大部分"判断"其实有规则可循。

### 模式 2：审计轨迹
每个自动决策记录原因——用户可以事后审查并推翻。

### 模式 3：故意不给 AskUserQuestion
通过工具限制强制自动化行为——比语言约束更可靠。
