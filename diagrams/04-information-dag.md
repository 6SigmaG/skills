# L4：信息流与 DAG 层 — Skill 间的依赖与数据流

> Boyd 的时序视角 + Kay 的黑板架构 = 完整的信息流图。

---

## 1. 完整 DAG（有向无环图）

```
                            ┌─────────────────────┐
                            │   用户需求 / 代码变更  │
                            └──────────┬──────────┘
                                       │
                    ┌──────────────────┼──────────────────┐
                    │                  │                   │
                    ▼                  ▼                   ▼
          ┌─────────────┐   ┌──────────────┐   ┌──────────────────┐
          │/plan-ceo-    │   │/plan-eng-    │   │/plan-design-     │
          │  review      │   │  review      │   │  review          │
          │  (CEO 审查)  │   │  (EM 审查)   │   │  (设计师审查)     │
          └──────┬──────┘   └──────┬───────┘   └────────┬─────────┘
                 │                  │                     │
                 │         ┌───────┴────────┐            │
                 │         │                │            │
                 │         ▼                │            │
                 │  ┌────────────┐          │            │
                 │  │ test-plan  │          │            │
                 │  │ .md 文件   │          │            │
                 │  └─────┬──────┘          │            │
                 │        │                 │            │
                 │        │    ┌────────────┘            │
                 │        │    │                         │
                 ▼        ▼    ▼                         ▼
          ┌──────────────────────────┐    ┌────────────────────────┐
          │       /review            │    │   /design-consultation │
          │  (Staff 工程师审查)       │    │   (设计伙伴)            │
          │  读取 diff + 代码        │    │   生成 DESIGN.md       │
          └────────────┬─────────────┘    └───────────┬────────────┘
                       │                               │
          ┌────────────┼───────────────────────────────┤
          │            │                               │
          ▼            ▼                               ▼
   ┌─────────────┐  ┌──────────────┐         ┌──────────────┐
   │/design-     │  │   /qa        │         │/design-      │
   │ review      │  │ (QA 全循环)  │         │ review       │
   │(视觉审查)   │  │              │         │(视觉审查)    │
   └──────┬──────┘  └──────┬───────┘         └──────┬───────┘
          │                │                         │
          │         ┌──────┴──────┐                  │
          │         │             │                  │
          │         ▼             ▼                  │
          │  ┌───────────┐ ┌───────────┐            │
          │  │ /qa-only  │ │修复后的代码│            │
          │  │ (仅报告)  │ │           │            │
          │  └───────────┘ └─────┬─────┘            │
          │                      │                  │
          └──────────┬───────────┘──────────────────┘
                     │
                     ▼
              ┌─────────────┐
              │   /ship      │
              │ (发布流水线)  │
              │ 8 阶段       │
              └──────┬──────┘
                     │
                     ▼
           ┌──────────────────┐
           │/document-release │
           │ (文档更新)       │
           └────────┬─────────┘
                    │
                    ▼
           ┌──────────────────┐
           │    /retro        │ ─── 外循环反馈 ───┐
           │  (周回顾)        │                    │
           └──────────────────┘                    │
                                                   │
                    ┌──────────────────────────────┘
                    ▼
           回到顶部：影响下一轮 plan 的姿态选择
```

## 2. 基础设施依赖图

```
                 ┌──────────────────────────────────────┐
                 │          /browse （浏览器引擎）        │
                 │   Chromium 守护进程 + 50+ 命令         │
                 └──────┬─────────────────┬─────────────┘
                        │                 │
               ┌────────┘                 └────────┐
               │                                   │
               ▼                                   ▼
    ┌──────────────────┐               ┌──────────────────┐
    │/setup-browser-   │               │所有使用浏览器的   │
    │  cookies         │               │Skill:             │
    │  (Cookie 导入)   │               │  /qa              │
    └────────┬─────────┘               │  /qa-only         │
             │                         │  /design-review   │
             │  导入 Cookie 到         └──────────────────┘
             │  Chromium 守护进程
             │
             ▼
    ┌──────────────────┐
    │ 登录态页面可访问   │
    │ 无需重新登录       │
    └──────────────────┘
```

## 3. 文件系统黑板架构

```
Kay 的关键发现：Skill 不直接调用彼此，而是通过共享文件系统通信

                        ┌────────────────────┐
                        │   文件系统 "黑板"    │
                        │                    │
生产者                  │  共享状态            │  消费者
──────────             │  ──────────         │  ──────────
                        │                    │
/plan-eng-review ──────►│ test-plan.md       │──────► /qa
                        │                    │
/ship ─────────────────►│ CHANGELOG.md       │──────► /document-release
                        │                    │
/ship ─────────────────►│ VERSION            │──────► /document-release
                        │                    │
/retro ────────────────►│ .context/retros/   │──────► /retro (下一次)
                        │ {date}.json        │
                        │                    │
/design-consultation ──►│ DESIGN.md          │──────► /design-review
                        │                    │
/design-review ────────►│ design-baseline    │──────► /design-review (下一次)
                        │ .json              │
                        │                    │
/review ───────────────►│ git commits        │──────► /ship
                        │ (修复的 commit)     │
                        │                    │
/document-release ─────►│ CLAUDE.md          │──────► 所有 Skill
                        │ README.md          │       (上下文输入)
                        │                    │
/setup-browser-cookies─►│ .gstack/browse     │──────► /browse
                        │ .json (守护进程)    │
                        │                    │
/office-hours ────────►│ ~/.gstack/projects/ │──────► /plan-ceo-review
                        │ *-design-*.md      │       (设计文档作为输入)
                        │                    │
/investigate ─────────►│ root-cause.md      │──────► /qa (验证修复)
                        │ (根因分析)          │
                        │                    │
/codex ───────────────►│ codex-findings     │──────► /review (交叉比对)
                        │                    │
/freeze ──────────────►│ ~/.gstack/         │──────► /unfreeze
                        │ freeze-dir.txt     │       (读取并删除)
                        │                    │
/careful ─────────────►│ [pre-tool hook]    │──────► Bash (拦截检查)
                        │                    │
/guard ───────────────►│ [组合 careful +    │──────► /careful + /freeze
                        │  freeze]           │
                        │                    │
/gstack-upgrade ──────►│ VERSION            │──────► 自身 (版本比对)
                        └────────────────────┘

关键观察：
  1. 没有 Skill 知道其他 Skill 的存在 → 松耦合
  2. 通信介质是文件 → 可用标准工具检查和调试
  3. 文件格式是契约 → test-plan.md 的结构是隐式 API
  4. 用户决定执行顺序 → 晚绑定（Kay 的核心洞察）
  5. 安全 Skill 通过 hook 系统运作，不参与数据流但影响行为
  6. /office-hours 的设计文档通过文件系统自然流入 plan 阶段
```

## 4. 典型使用流程（按 Boyd 的时序视角）

### 流程 A：完整功能开发（最长路径）

```
Phase        Skill              输入              输出               耗时
──────       ──────             ──────            ──────            ──────
战略         /plan-ceo-review    需求文档          批准/拒绝/修改     10-15 min
战术         /plan-eng-review    批准的需求        架构 + 测试计划    15-25 min
设计         /design-consultation 需求 + 架构     DESIGN.md          20-30 min
编码         [人工/AI 编码]       架构 + 设计      代码变更           变化大
审查         /review             git diff         修复 + 发现报告    10-20 min
视觉审查     /design-review      运行中的页面     CSS 修复           10-15 min
测试         /qa                 代码 + 测试计划   修复 + 回归测试    15-30 min
发布         /ship               通过测试的代码    PR + CHANGELOG    10-15 min
文档         /document-release   PR diff          文档更新           5-10 min
回顾         /retro              上周 commit      回顾报告           5-10 min
```

### 流程 B：快速 bug 修复（最短路径）

```
/qa → [修复] → /ship
3 个 Skill，约 30 分钟
```

### 流程 C：设计优化（无代码变更）

```
/design-consultation → /plan-design-review → /design-review
3 个 Skill，约 45 分钟
```

### 流程 D：仅审计（不修改代码）

```
/qa-only → 生成报告 → 人工决定是否修复
1 个 Skill，约 15 分钟
```

## 5. 组合场景决策树

```
你有什么？                     推荐流程
──────────                    ──────────

产品方向不确定
  └── 需要重新审视方向          A0: office-hours → plan-ceo → plan-eng → ...

新功能需求
  ├── 大功能（>1 周）          A: plan-ceo → plan-eng → design-consult
  │                              → 编码 → review → codex → qa → ship → doc-release
  │
  ├── 中功能（1-3 天）          B: plan-eng → 编码 → review → qa → ship
  │
  └── 小功能（<1 天）           C: 编码 → review → ship

Bug 报告
  ├── 有复现步骤                D: qa（自动定位+修复+验证）→ ship
  ├── 无复现步骤                E: investigate（根因调试）→ qa（验证）→ ship
  ├── 复杂/深层 bug             E2: investigate → qa → ship
  └── 视觉 bug                  F: design-review（定位+修复）→ ship

设计改进
  ├── 从零开始                  G: design-consult → plan-design-review
  │                              → 编码 → design-review
  └── 已有页面优化              H: design-review（直接审查+修复）

高风险操作
  ├── 修改关键模块              I0: guard → 编码 → review → qa → ship
  └── 操作生产环境              I1: careful → 操作

发布
  ├── 常规发布                  I: ship → doc-release
  └── 发布 + 回顾               J: ship → doc-release → retro

周期性
  ├── 每周五                    K: retro
  └── 版本更新                  L: gstack-upgrade
```
