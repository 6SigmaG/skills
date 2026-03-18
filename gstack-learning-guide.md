# gstack 深度学习指南

> **版本：** 基于 gstack v0.6.4.0 | **作者：** Garry Tan (Y Combinator CEO)
> **定位：** 将 Claude Code 变成一个虚拟工程团队的开源 Skills 框架

---

## 目录

- [第一层：基础篇 — 理解 gstack 是什么](#第一层基础篇--理解-gstack-是什么)
  - [1.1 一句话理解 gstack](#11-一句话理解-gstack)
  - [1.2 前置知识：Claude Code Skills 机制](#12-前置知识claude-code-skills-机制)
  - [1.3 gstack 解决了什么问题](#13-gstack-解决了什么问题)
  - [1.4 13 个 Skill 角色全景图](#14-13-个-skill-角色全景图)
  - [1.5 安装与快速体验](#15-安装与快速体验)
  - [1.6 核心工作流：一个功能从想法到上线](#16-核心工作流一个功能从想法到上线)
  - [1.7 关键概念速查表](#17-关键概念速查表)
- [第二层：高阶篇 — 架构设计与内部原理](#第二层高阶篇--架构设计与内部原理)
  - [2.1 整体架构：两层系统](#21-整体架构两层系统)
  - [2.2 Browse 浏览器引擎：核心技术深度拆解](#22-browse-浏览器引擎核心技术深度拆解)
  - [2.3 SKILL.md 模板系统：Prompt 即代码](#23-skillmd-模板系统prompt-即代码)
  - [2.4 测试金字塔：三层验证体系](#24-测试金字塔三层验证体系)
  - [2.5 设计哲学拆解](#25-设计哲学拆解)
  - [2.6 如何参考 gstack 开发自己的 Skills](#26-如何参考-gstack-开发自己的-skills)
  - [2.7 项目文件结构完整地图](#27-项目文件结构完整地图)

---

# 第一层：基础篇 — 理解 gstack 是什么

## 1.1 一句话理解 gstack

**gstack 是一套 Claude Code 的 Skill 集合，它把一个 AI 助手变成 13 个不同角色的"虚拟团队成员"。**

你可以这样理解：

| 没有 gstack | 有 gstack |
|---|---|
| 你有一个通用 AI 助手 | 你有 CEO、工程经理、设计师、QA、发布工程师... |
| 你说"帮我加个功能"，AI 直接写代码 | 你说 `/plan-ceo-review`，AI 先像 CEO 一样重新思考这个功能的本质 |
| 代码写完直接提交 | 经过 review → QA（真实浏览器测试）→ ship → 文档更新 |
| AI 是你的副驾驶 | AI 是你的 **团队** |

## 1.2 前置知识：Claude Code Skills 机制

在理解 gstack 之前，你需要知道 Claude Code 的 Skills 是怎么工作的：

### 什么是 Skill？

Skill 就是一个 **Markdown 文件**（`SKILL.md`），放在特定目录下，Claude Code 会自动发现它。

```
~/.claude/skills/          ← Claude Code 全局 Skills 目录
├── gstack/                ← gstack 主目录
│   ├── SKILL.md           ← /browse skill（浏览器）
│   ├── review/
│   │   └── SKILL.md       ← /review skill
│   ├── ship/
│   │   └── SKILL.md       ← /ship skill
│   └── ...
├── review -> gstack/review    ← 符号链接，让 Claude Code 发现它
├── ship -> gstack/ship
└── ...
```

### SKILL.md 的结构

每个 SKILL.md 包含两部分：

```markdown
---
name: review              ← Skill 名称（用户输入 /review 触发）
version: 1.1.0
description: |            ← 描述，让 Claude Code 知道什么时候使用
  Find bugs that pass CI but blow up in production...
allowed-tools:            ← 这个 Skill 被允许使用的工具
  - Bash
  - Read
  - Edit
  - Write
---

（下面是给 Claude 的详细指令 — 纯 Markdown）
```

**核心原理：** SKILL.md 本质上是一个**超级详细的 System Prompt**。当你输入 `/review` 时，Claude Code 读取这个文件，然后按照里面的指令执行。

## 1.3 gstack 解决了什么问题

### 问题 1：AI 写代码没有"流程"

普通使用 Claude Code：你说一句，它做一句。没有 review，没有测试，没有设计审查。
这就像一个程序员写完代码直接推到生产环境 — 迟早出事。

**gstack 的解决方案：** 给 AI 一套完整的软件工程流程。

```
想法 → CEO思考 → 工程规划 → 设计审查 → 编码 → Code Review → QA测试 → 发布 → 文档更新
        ↑           ↑           ↑                    ↑          ↑        ↑        ↑
  /plan-ceo   /plan-eng   /plan-design         /review      /qa    /ship  /document-release
```

### 问题 2：AI 没有"眼睛"

Claude Code 能写代码，但它看不到网页长什么样。它不知道按钮是不是真的能点，表单是不是真的能提交。

**gstack 的解决方案：** 内置一个持久化的 Chromium 浏览器。
`/qa` 和 `/browse` 可以打开真实网页、点击按钮、填表单、截图。

### 问题 3：AI 生成的代码有"AI 味"

AI 生成的 UI 有明显的套路：紫色渐变、三列图标网格、通用的 hero section...
有经验的人一眼就能看出来"这是 AI 生成的"。

**gstack 的解决方案：** `/plan-design-review` 和 `/design-review` 专门检测 "AI Slop"（AI 垃圾模式），并给出修复建议。

## 1.4 13 个 Skill 角色全景图

gstack 的 13 个 Skill 按照软件开发流程分为 **4 个阶段**：

### 阶段一：规划（Plan Mode）

| Skill | 角色 | 做什么 | 什么时候用 |
|---|---|---|---|
| `/plan-ceo-review` | CEO / 创始人 | 重新思考问题本质，找到"10 星产品"，提供 4 种模式：扩展/选择性扩展/保持/缩减 | 开始做新功能之前 |
| `/plan-eng-review` | 工程经理 | 锁定架构、数据流、状态机、边界情况，画 ASCII 图表 | CEO review 之后，开始编码之前 |
| `/plan-design-review` | 高级设计师 | 80 项设计审查，检测 AI Slop，推断你的设计系统 | 有 UI 变更的计划时 |
| `/design-consultation` | 设计伙伴 | 从零开始构建完整设计系统：字体、颜色、间距、布局 | 新项目或没有设计系统时 |

### 阶段二：审查（Review）

| Skill | 角色 | 做什么 | 什么时候用 |
|---|---|---|---|
| `/review` | Staff 工程师 | 找出通过 CI 但在生产环境会爆炸的 Bug，自动修复明显问题 | 写完代码之后、提交之前 |
| `/design-review` | 会编码的设计师 | 在真实网站上运行 80 项视觉审查，然后逐个修复 | 功能实现后，想修 UI 问题时 |

### 阶段三：测试（QA）

| Skill | 角色 | 做什么 | 什么时候用 |
|---|---|---|---|
| `/browse` | QA 工程师 | 给 AI 一个真实浏览器，能导航、点击、截图 | 需要浏览网页时 |
| `/qa` | QA 负责人 | 测试你的应用，发现 Bug，修复它们，生成回归测试 | 功能完成后，想全面测试 |
| `/qa-only` | QA 报告员 | 和 `/qa` 一样测试，但只报告不修改代码 | 只想要 Bug 报告时 |
| `/setup-browser-cookies` | 会话管理员 | 从真实浏览器导入 Cookie，测试登录后的页面 | 测试需要登录的页面前 |

### 阶段四：发布（Ship）

| Skill | 角色 | 做什么 | 什么时候用 |
|---|---|---|---|
| `/ship` | 发布工程师 | 同步 main，运行测试，审计覆盖率，推送代码，创建 PR | 准备发布时 |
| `/retro` | 工程经理 | 团队感知的周回顾：每人贡献、测试健康度、发布节奏 | 每周末或想复盘时 |
| `/document-release` | 技术作家 | 对比 diff 更新所有文档：README、ARCHITECTURE、CHANGELOG 等 | `/ship` 之后、合并之前 |

### 辅助 Skill

| Skill | 做什么 |
|---|---|
| `/gstack-upgrade` | 检查并升级 gstack 到最新版本 |

## 1.5 安装与快速体验

### 前置要求

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) — Anthropic 官方 CLI
- [Git](https://git-scm.com/)
- [Bun](https://bun.sh/) v1.0+ — JavaScript 运行时

### 安装（30 秒）

在 Claude Code 中粘贴：

```
安装 gstack: 运行 git clone https://github.com/garrytan/gstack.git ~/.claude/skills/gstack && cd ~/.claude/skills/gstack && ./setup
```

Claude 会自动完成安装。

### 快速体验

```bash
# 1. 对任意功能想法运行 CEO review
/plan-ceo-review

# 2. 对有改动的分支运行 code review
/review

# 3. 对 staging URL 运行 QA 测试
/qa https://staging.myapp.com

# 4. 准备发布
/ship
```

## 1.6 核心工作流：一个功能从想法到上线

以"给卖家加照片上传功能"为例，完整走一遍 gstack 工作流：

```
你：我想给卖家加照片上传功能。

你：/plan-ceo-review
Claude（CEO 模式）：
  "照片上传"不是真正的功能。真正的任务是帮卖家创建能卖出去的商品列表。
  如果我们能从照片自动识别产品、拉取规格和价格对比、自动生成标题描述呢？
  那是 10 星产品。"上传照片"只有 3 星。
  [提出 8 个扩展提案，你挑选 5 个，3 个放到 backlog]

你：/plan-eng-review
Claude（工程经理模式）：
  ┌─────────┐    ┌──────────┐    ┌──────────┐    ┌─────────┐
  │ Upload  │───▶│ Classify │───▶│ Enrich   │───▶│ Draft   │
  │ (sync)  │    │ (async)  │    │ (async)  │    │ (async) │
  └─────────┘    └──────────┘    └──────────┘    └─────────┘
  [架构图、状态图、数据流图]
  [14 个测试场景、6 个失败模式、3 个安全问题]

你：批准计划，退出 Plan Mode。
  Claude 写了 2400 行代码，跨 11 个文件 — 大约 8 分钟。

你：/review
Claude（Staff 工程师模式）：
  [自动修复] 上传失败时 S3 孤儿文件清理
  [自动修复] listings.status 缺少索引
  [询问] hero 图片选择的竞态条件 → 你说修复
  3 个问题 — 2 个自动修复，1 个手动确认后修复。

你：/qa https://staging.myapp.com
Claude（QA 模式）：
  [打开真实浏览器，登录，上传照片，点击各种流程]
  上传 → 分类 → 丰富 → 生成: 端到端 ✓
  发现 Bug: 第二次上传时预览不清除 — 已修复
  自动生成回归测试。

你：/ship
Claude（发布工程师模式）：
  测试: 42 → 51 (+9 new)
  覆盖率: 14/14 代码路径 (100%)
  PR: github.com/you/app/pull/42
```

**一个功能。七个命令。** Agent 重新定义了产品方向、画了架构图、写了 2400 行代码、发现了竞态条件、自动修复了两个问题、用真实浏览器做了 QA、发现并修复了一个你不知道的 Bug、写了 9 个测试。

## 1.7 关键概念速查表

| 概念 | 解释 |
|---|---|
| **Skill** | 一个 SKILL.md 文件，定义 Claude Code 的特殊行为模式 |
| **SKILL.md** | 给 Claude 的超级 Prompt，包含详细的工作指令 |
| **Browse / $B** | gstack 内置的无头浏览器命令行工具 |
| **Ref（@e1, @e2）** | 浏览器中可交互元素的引用标识符，不用写 CSS 选择器 |
| **Snapshot** | 页面可访问性树的快照，展示所有可交互元素 |
| **Plan Mode** | Claude Code 的规划模式，先想清楚再动手 |
| **Preamble** | 每个 Skill 开头的通用启动代码块 |
| **Review Readiness Dashboard** | 发布前的检查面板，显示哪些审查已完成 |
| **AI Slop** | AI 生成的明显套路化 UI 模式（渐变、网格、通用文案） |
| **Boil the Lake** | gstack 的核心哲学：AI 让完整实现的成本接近零，所以永远选最完整的方案 |
| **Conductor** | 第三方工具，可以并行运行多个 Claude Code 会话 |

---

# 第二层：高阶篇 — 架构设计与内部原理

## 2.1 整体架构：两层系统

gstack 本质上是两个独立但互补的系统：

```
┌────────────────────────────────────────────────────────────────┐
│                    第一层：Skill 层（纯 Markdown）              │
│                                                                │
│   12 个 SKILL.md 文件（由 .tmpl 模板生成）                      │
│   · 每个 Skill 是一个角色 Prompt                                │
│   · 零代码，纯文本                                              │
│   · 通过 {{PLACEHOLDER}} 模板系统从源码自动生成                   │
│                                                                │
├────────────────────────────────────────────────────────────────┤
│                    第二层：Browse 引擎（TypeScript/Bun）         │
│                                                                │
│   编译后的二进制 → HTTP 服务器 → Chromium 守护进程               │
│   · CLI 客户端 (cli.ts)                                        │
│   · HTTP 服务器 (server.ts, Bun.serve)                         │
│   · 浏览器管理器 (browser-manager.ts, Playwright)               │
│   · Cookie 导入系统 (cookie-import-browser.ts)                  │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

**为什么分两层？**

- **Skill 层**可以被任何人修改（只需要会写 Markdown），不需要编程知识
- **Browse 引擎**提供浏览器能力，是唯一需要编译的部分
- 大部分 Skill 不需要 Browse 引擎也能工作（如 `/review`、`/ship`、`/retro`）

## 2.2 Browse 浏览器引擎：核心技术深度拆解

### 为什么需要自建浏览器？

Claude Code 自身没有"看网页"的能力。gstack 通过内置一个持久化 Chromium 浏览器解决了这个问题。

### 架构图

```
Claude Code                     gstack Browse
─────────                      ─────────────
                               ┌───────────────────────┐
  Tool call: $B snapshot -i    │  CLI (编译后的二进制)    │
  ─────────────────────────→   │  · 读取状态文件          │
                               │  · POST /command        │
                               │    到 localhost:PORT     │
                               └──────────┬────────────┘
                                          │ HTTP
                               ┌──────────▼────────────┐
                               │  Server (Bun.serve)    │
                               │  · 分发命令             │
                               │  · 操控 Chromium        │
                               │  · 返回纯文本           │
                               └──────────┬────────────┘
                                          │ CDP (Chrome DevTools Protocol)
                               ┌──────────▼────────────┐
                               │  Chromium (无头模式)    │
                               │  · 持久化标签页          │
                               │  · Cookie 跨命令保持     │
                               │  · 30 分钟空闲超时       │
                               └────────────────────────┘
```

### 关键设计决策

#### 1. 守护进程模式（vs 每次冷启动）

| 方案 | 延迟 | 状态 |
|---|---|---|
| 每次冷启动 Chromium | 2-3 秒/命令 | 每次丢失（Cookie、登录态全没） |
| **守护进程（gstack 选择）** | **~100ms/命令** | **持久化（Cookie、标签页保留）** |

第一次调用启动浏览器约 3 秒，之后每次命令只需 ~100-200ms。30 分钟无活动自动关闭。

#### 2. 为什么用 Bun 而不是 Node.js？

| 原因 | 详细说明 |
|---|---|
| **编译二进制** | `bun build --compile` 生成一个 ~58MB 可执行文件，无需 node_modules |
| **内置 SQLite** | Cookie 解密需要读 Chromium 的 SQLite 数据库，Bun 原生支持 |
| **原生 TypeScript** | 开发时直接运行 .ts 文件，无需编译步骤 |
| **内置 HTTP 服务器** | `Bun.serve()` 轻量、快速，不需要 Express |

#### 3. Ref 系统（@e1, @e2, @c1）

这是 gstack 最精妙的设计之一。

**问题：** AI 要和网页交互，怎么指定"点哪个按钮"？

**传统方案：** CSS 选择器（`#submit-btn`）或 XPath — 但 AI 很容易猜错。

**gstack 的方案：** Ref 引用系统。

```bash
# 1. 运行 snapshot，获取可交互元素列表
$B snapshot -i
# 输出：
#   @e1 [heading] "Welcome" [level=1]
#   @e2 [textbox] "Email"
#   @e3 [button] "Submit"

# 2. 用 @ref 操作元素（不需要猜选择器）
$B fill @e2 "test@example.com"
$B click @e3
```

**内部实现原理：**

```
1. Agent 运行: $B snapshot -i
2. Server 调用 Playwright 的 page.accessibility.snapshot()
3. 解析器遍历 ARIA 可访问性树
4. 为每个元素分配 @e1, @e2, @e3... 编号
5. 为每个 Ref 构建 Playwright Locator: getByRole(role, { name }).nth(index)
6. 存储 Map<string, RefEntry> 映射

后续操作：
7. Agent 运行: $B click @e3
8. Server 查找 @e3 → 获取 Locator → locator.click()
```

**为什么用 Locator 而不是 DOM 注入（如注入 `data-ref="@e1"`）？**

| 问题 | 解释 |
|---|---|
| CSP 策略 | 很多网站的安全策略禁止外部脚本修改 DOM |
| React/Vue 水合 | 前端框架的协调机制会删掉注入的属性 |
| Shadow DOM | 从外部无法操作 Shadow Root 内的元素 |

Playwright Locator 在 DOM 之外工作，不修改页面，不受这些限制。

#### 4. Ref 生命周期与过期检测

```
· 页面导航（framenavigated 事件）→ 所有 Ref 自动清除（必须重新 snapshot）
· SPA 路由切换（无导航事件）→ resolveRef() 做异步 count() 检查
  - count === 0 → 抛出 "Ref @e3 is stale — element no longer exists"
  - count > 0 → 正常使用
  - 开销：~5ms（vs Playwright 默认 30 秒超时）
```

#### 5. 安全模型

| 层面 | 措施 |
|---|---|
| 网络 | 绑定 localhost，不可从外部访问 |
| 认证 | 每次启动生成随机 UUID Token，文件权限 0o600（只有 owner 可读） |
| Cookie | 在内存中解密，从不写入磁盘明文；从不显示 Cookie 值 |
| 注入防护 | 浏览器列表硬编码，路径不接受用户输入，用参数数组而非字符串拼接调用命令 |

#### 6. 命令分类系统

所有浏览器命令按副作用分为三类：

```typescript
READ 命令（无副作用，可安全重试）：
  text, html, links, console, cookies, accessibility, forms, ...

WRITE 命令（会改变页面状态，不可幂等）：
  goto, click, fill, press, scroll, select, upload, ...

META 命令（服务器级操作）：
  snapshot, screenshot, tabs, chain, stop, restart, ...
```

这个分类驱动了命令分发逻辑：
```typescript
if (READ_COMMANDS.has(cmd))  → handleReadCommand(cmd, args, bm)
if (WRITE_COMMANDS.has(cmd)) → handleWriteCommand(cmd, args, bm)
if (META_COMMANDS.has(cmd))  → handleMetaCommand(cmd, args, bm, shutdown)
```

#### 7. 错误设计：面向 AI Agent 而非人类

```
传统报错：          "Element not found"
gstack 报错：       "Element not found or not interactable.
                     Run `snapshot -i` to see available elements."

传统报错：          "Timeout"
gstack 报错：       "Navigation timed out after 30s. The page may be
                     slow or the URL may be wrong."
```

每个错误消息都告诉 Agent **下一步该做什么**。Agent 能自行恢复，无需人类干预。

#### 8. 日志架构

```
三个环形缓冲区（各 50,000 条，O(1) 写入）：
  Console 日志  → CircularBuffer → 每秒异步写入 .gstack/browse-console.log
  Network 日志  → CircularBuffer → 每秒异步写入 .gstack/browse-network.log
  Dialog 日志   → CircularBuffer → 每秒异步写入 .gstack/browse-dialog.log
```

**设计优势：**
- HTTP 处理永远不会被磁盘 I/O 阻塞
- 内存有界（50K × 3 缓冲区）
- 磁盘文件为追加模式，外部工具可读
- 命令读取的是内存缓冲区，不是磁盘

## 2.3 SKILL.md 模板系统：Prompt 即代码

### 问题：手写文档会和代码脱节

如果 SKILL.md 手动维护，迟早会出现：文档说有某个命令，但代码已经删了；代码加了新命令，但文档没更新。

### 解决方案：从源码自动生成

```
SKILL.md.tmpl          ← 人工编写的 Prompt 模板（含占位符）
       ↓
gen-skill-docs.ts      ← 构建脚本（读取源码元数据）
       ↓
SKILL.md               ← 自动生成的最终文件（提交到 git）
```

### 占位符系统

| 占位符 | 数据来源 | 生成内容 |
|---|---|---|
| `{{COMMAND_REFERENCE}}` | `commands.ts` | 按分类的命令表格 |
| `{{SNAPSHOT_FLAGS}}` | `snapshot.ts` | Flag 参考和示例 |
| `{{PREAMBLE}}` | `gen-skill-docs.ts` | 通用启动代码：更新检查、会话追踪、贡献者模式 |
| `{{BROWSE_SETUP}}` | `gen-skill-docs.ts` | 浏览器二进制发现 + 设置指令 |
| `{{BASE_BRANCH_DETECT}}` | `gen-skill-docs.ts` | 动态检测 PR 目标分支 |
| `{{QA_METHODOLOGY}}` | `gen-skill-docs.ts` | 共享 QA 方法论块 |
| `{{DESIGN_METHODOLOGY}}` | `gen-skill-docs.ts` | 共享设计审查方法论 |
| `{{REVIEW_DASHBOARD}}` | `gen-skill-docs.ts` | Review 就绪仪表盘 |
| `{{TEST_BOOTSTRAP}}` | `gen-skill-docs.ts` | 测试框架检测与自动搭建 |

### 为什么提交生成文件而不是运行时生成？

1. **Claude 在加载 Skill 时读取 SKILL.md** — 没有构建步骤
2. **CI 可以验证新鲜度** — `gen:skill-docs --dry-run` + `git diff --exit-code`
3. **git blame 有效** — 可以追踪每个命令何时添加

### Preamble（前言）系统

每个 Skill 都以一个通用的 `{{PREAMBLE}}` 代码块开始，在一次 bash 调用中处理 4 件事：

1. **更新检查** — 是否有新版本
2. **会话追踪** — 记录活跃会话数；3+ 个会话时进入"ELI16 模式"（每个问题都重新说明上下文）
3. **贡献者模式** — 是否开启自动 Bug 报告
4. **AskUserQuestion 格式** — 统一的提问格式：上下文、问题、推荐选项

### 模板编写规则（重要！）

如果你想开发自己的 Skill，这些是关键规则：

```
✅ 用自然语言传递状态（不要用 shell 变量跨代码块）
✅ 动态检测分支名（不要硬编码 main）
✅ 每个 bash 代码块独立可运行
✅ 用英语表达条件逻辑（不要嵌套 if/elif/else）

❌ 不要在代码块间传递 shell 变量（每个块是独立 shell）
❌ 不要硬编码分支名
❌ 不要用复杂的 bash 条件分支
```

## 2.4 测试金字塔：三层验证体系

| 层级 | 命令 | 成本 | 速度 | 验证内容 |
|---|---|---|---|---|
| **Tier 1 — 静态验证** | `bun test` | 免费 | <5 秒 | 解析 SKILL.md 中所有 `$B` 命令，对照命令注册表验证；检查 snapshot flag；验证 SKILL.md 正确性 |
| **Tier 2 — E2E 测试** | `bun run test:e2e` | ~$3.85 | ~20 分钟 | 启动真实 Claude 会话，运行每个 Skill，扫描错误 |
| **Tier 3 — LLM 评审** | `bun run test:evals` | ~$0.15 | ~30 秒 | Claude Sonnet 对生成的文档评分：清晰度/完整度/可操作性 |

### Tier 1 的精妙之处

它从 SKILL.md 中提取每一个 `$B` 命令调用，然后对照 `commands.ts` 中的命令注册表验证。如果模板里写了一个不存在的命令，测试立刻失败。

### Tier 2 的 Session Runner

E2E 测试的核心难题：如何在测试中运行一个完整的 Claude Code 会话？

```
解决方案（session-runner.ts）：
1. 把 Prompt 写入临时文件（避免 shell 转义问题）
2. 用 sh -c 'cat prompt | claude -p --output-format stream-json --verbose' 启动子进程
3. 从 stdout 流式读取 NDJSON 以获取实时进度
4. 设置超时竞赛
5. 解析完整的 NDJSON 转录为结构化结果
```

### Diff-based 测试选择

每个测试声明它依赖的文件。运行 `bun run test:e2e` 时，系统检查你的 git diff，只运行依赖文件被修改的测试。只改了 `/retro` 的分支跑 2 个测试而不是 31 个。

## 2.5 设计哲学拆解

### 哲学 1：Boil the Lake（煮干湖水）

> AI 让完整实现的边际成本接近零。当你可以选择 80% 方案（省 70 行代码）或 100% 方案时，**永远选 100%**。

| 任务类型 | 人类团队时间 | CC+gstack 时间 | 压缩比 |
|---|---|---|---|
| 脚手架/模板 | 2 天 | 15 分钟 | ~100x |
| 写测试 | 1 天 | 15 分钟 | ~50x |
| 功能实现 | 1 周 | 30 分钟 | ~30x |
| Bug 修复 + 回归测试 | 4 小时 | 15 分钟 | ~20x |
| 架构/设计 | 2 天 | 4 小时 | ~5x |

**Lake vs Ocean：**
- Lake（湖）= 可以煮干 = 100% 测试覆盖、完整功能、所有边界情况
- Ocean（海洋）= 不要尝试 = 从头重写系统、修改你不控制的依赖

### 哲学 2：认知模式切换

gstack 的核心洞察：**同一个 LLM，给它不同的角色 Prompt，产出质量天差地别。**

- CEO 模式 = 发散思维，重新定义问题
- 工程经理模式 = 收敛思维，锁定架构
- Staff 工程师模式 = 偏执思维，找漏洞
- QA 模式 = 系统思维，完整测试

这不是 Magic，而是 Prompt Engineering 的工业化实践。

### 哲学 3：Fix-First（先修复再汇报）

`/review` 的发现不是列出来就完了：
- 明显的机械修复（死代码、N+1 查询）→ **自动修复**
- 真正有歧义的问题（安全、竞态条件）→ **询问你**

### 哲学 4：Latent Space Activation（潜在空间激活）

`/plan-ceo-review` 不是给 Claude 一个检查清单。它提到真实人物和框架：

> "Bezos 的单向门决策、Grove 的偏执扫描、Munger 的逆向思维、Chesky 的 11 星体验..."

这不是让 Claude 背公式。这是**触发 LLM 训练数据中关于这些人思维方式的深层模式**。指令是"内化这些，不要列举它们"。

### 哲学 5：错误消息面向 Agent

传统软件的错误消息是给人看的。gstack 的错误消息是给 AI Agent 看的。每条消息都包含"下一步该做什么"的具体指令。

## 2.6 如何参考 gstack 开发自己的 Skills

### 最小可行 Skill

创建文件 `~/.claude/skills/my-skill/SKILL.md`：

```markdown
---
name: my-skill
version: 0.1.0
description: |
  描述你的 Skill 做什么。Claude Code 用这个决定什么时候建议使用它。
allowed-tools:
  - Bash
  - Read
  - Edit
---

# My Skill

## 步骤 1：了解情况

先读取项目的 README.md 和 package.json 来了解项目上下文。

## 步骤 2：做某件事

（你的详细指令...）

## 步骤 3：验证

运行测试确保一切正常：
```bash
npm test
```
```

这就是全部。一个 Markdown 文件。

### 从 gstack 学到的 Skill 设计模式

#### 模式 1：角色扮演（Persona）

```markdown
# 你是一个偏执的 Staff 工程师
你的工作不是赞美代码，而是找出会在生产环境爆炸的问题。
```

gstack 的每个 Skill 都在开头定义了 Claude 应该"扮演"的角色。

#### 模式 2：Preamble（通用启动块）

```markdown
## 前言（首先运行）

```bash
# 收集环境信息
echo "BRANCH: $(git branch --show-current)"
echo "PROJECT: $(basename $(git rev-parse --show-toplevel))"
```

在你的 Prompt 指令开始之前，先让 Agent 了解当前环境。
```

#### 模式 3：AskUserQuestion 格式一致性

```markdown
每次提问必须包含：
1. 上下文重述（项目名 + 当前分支 + 当前任务）
2. 简单解释（聪明的 16 岁高中生能看懂）
3. 推荐选项 + 原因
4. 选项列表（A/B/C）
```

#### 模式 4：自检与反馈循环

```markdown
做完操作后，验证结果。
如果失败了，诊断原因并重试。
不要盲目继续。
```

#### 模式 5：模板占位符（DRY）

如果多个 Skill 共享同一段逻辑（如 QA 方法论），提取为模板占位符，在构建时注入。

### 进阶：添加浏览器能力

如果你的 Skill 需要浏览网页：

```markdown
allowed-tools:
  - Bash
  - Read

## 设置

```bash
# 检查 browse 二进制是否可用
B=~/.claude/skills/gstack/browse/dist/browse
if [ -x "$B" ]; then
  echo "READY: $B"
else
  echo "需要先安装 gstack 的 browse 工具"
fi
```

## 使用浏览器

```bash
$B goto https://example.com
$B snapshot -i
$B click @e3
$B screenshot /tmp/result.png
```
```

## 2.7 项目文件结构完整地图

```
gstack/                              ← 项目根目录
│
├── 📋 文档文件
│   ├── README.md                    ← 用户入口，安装指南
│   ├── ARCHITECTURE.md              ← 技术架构设计文档
│   ├── BROWSER.md                   ← 浏览器命令完整参考
│   ├── CHANGELOG.md                 ← 版本更新日志
│   ├── CLAUDE.md                    ← 给 Claude Code 的项目上下文
│   ├── CONTRIBUTING.md              ← 贡献者指南
│   ├── TODOS.md                     ← 统一的项目 Backlog
│   ├── VERSION                      ← 当前版本号 (0.6.4.0)
│   ├── LICENSE                      ← MIT 许可证
│   └── docs/
│       ├── skills.md                ← 13 个 Skill 的深度指南
│       └── images/                  ← README 中的截图
│
├── 🏗 构建与配置
│   ├── package.json                 ← npm 配置：脚本、依赖
│   ├── conductor.json               ← Conductor 多会话集成配置
│   ├── setup                        ← 一键安装脚本（bash）
│   ├── SKILL.md                     ← /browse skill（自动生成 ← SKILL.md.tmpl）
│   └── SKILL.md.tmpl                ← /browse skill 模板（手工编辑此文件）
│
├── 🌐 browse/ — 浏览器引擎
│   ├── src/
│   │   ├── cli.ts                   ← CLI 入口：解析命令，发 HTTP 请求
│   │   ├── server.ts                ← HTTP 服务器：接收命令，操控浏览器
│   │   ├── browser-manager.ts       ← 核心：管理 Chromium 实例、Ref 映射
│   │   ├── commands.ts              ← 命令注册表（唯一数据源）
│   │   ├── snapshot.ts              ← Snapshot 系统：ARIA 树 → @ref
│   │   ├── read-commands.ts         ← 只读命令实现
│   │   ├── write-commands.ts        ← 写入命令实现
│   │   ├── meta-commands.ts         ← 元命令实现
│   │   ├── config.ts                ← 路径和配置解析
│   │   ├── buffers.ts               ← 环形缓冲区（日志）
│   │   ├── find-browse.ts           ← 二进制发现工具
│   │   ├── cookie-import-browser.ts ← 浏览器 Cookie 解密与导入
│   │   ├── cookie-picker-routes.ts  ← Cookie 选择器 HTTP 路由
│   │   └── cookie-picker-ui.ts      ← Cookie 选择器 Web UI
│   ├── test/                        ← 浏览器集成测试（166+）
│   │   ├── commands.test.ts         ← 命令测试
│   │   ├── snapshot.test.ts         ← Snapshot 测试
│   │   ├── config.test.ts           ← 配置测试
│   │   └── ...
│   └── dist/                        ← 编译输出（gitignored）
│       ├── browse                   ← 编译后的二进制（~58MB）
│       └── .version                 ← git commit SHA
│
├── 🎭 14 个 Skill 目录（每个含 SKILL.md + SKILL.md.tmpl）
│   ├── plan-ceo-review/             ← CEO/创始人模式
│   ├── plan-eng-review/             ← 工程经理模式
│   ├── plan-design-review/          ← 高级设计师模式
│   ├── design-consultation/         ← 设计伙伴模式
│   ├── review/                      ← Staff 工程师模式
│   │   ├── SKILL.md / SKILL.md.tmpl
│   │   ├── checklist.md             ← 审查清单 + Fix-First 规则
│   │   ├── greptile-triage.md       ← Greptile 评论分类规则
│   │   └── TODOS-format.md          ← TODO 格式规范
│   ├── design-review/               ← 会编码的设计师模式
│   ├── qa/                          ← QA 负责人模式
│   │   ├── SKILL.md / SKILL.md.tmpl
│   │   └── qa-report-template.md    ← QA 报告模板
│   ├── qa-only/                     ← QA 报告员模式（只报告不修复）
│   ├── ship/                        ← 发布工程师模式
│   ├── retro/                       ← 工程经理回顾模式
│   ├── document-release/            ← 技术作家模式
│   ├── setup-browser-cookies/       ← 会话管理员模式
│   └── gstack-upgrade/              ← 自升级 Skill
│
├── 🔧 bin/ — CLI 工具
│   ├── dev-setup                    ← 开发模式：创建符号链接
│   ├── dev-teardown                 ← 退出开发模式
│   ├── gstack-config                ← 配置管理（get/set/list）
│   ├── gstack-diff-scope            ← 分析 diff 影响范围
│   ├── gstack-slug                  ← 计算 owner-repo slug
│   └── gstack-update-check          ← 检查更新（带缓存）
│
├── 📜 scripts/ — 构建与开发工具
│   ├── gen-skill-docs.ts            ← 模板 → SKILL.md 生成器
│   ├── skill-check.ts               ← Skill 健康仪表盘
│   ├── dev-skill.ts                 ← 监听模式：改模板自动重新生成
│   ├── eval-watch.ts                ← 实时 E2E 测试仪表盘
│   ├── eval-list.ts                 ← 列出所有 eval 运行记录
│   ├── eval-compare.ts              ← 对比两次 eval 运行
│   ├── eval-summary.ts              ← 跨运行统计汇总
│   └── eval-select.ts               ← 预览 diff 会触发哪些测试
│
└── 🧪 test/ — 测试套件
    ├── skill-validation.test.ts     ← Tier 1：静态验证
    ├── gen-skill-docs.test.ts       ← Tier 1：生成器质量
    ├── skill-parser.test.ts         ← Tier 1：命令解析
    ├── touchfiles.test.ts           ← Tier 1：测试依赖完整性
    ├── skill-e2e.test.ts            ← Tier 2：E2E 测试
    ├── skill-llm-eval.test.ts       ← Tier 3：LLM 评审
    └── helpers/
        ├── session-runner.ts        ← E2E：claude -p 子进程运行器
        ├── eval-store.ts            ← eval 结果持久化
        ├── llm-judge.ts             ← LLM 评审调用
        ├── skill-parser.ts          ← SKILL.md 命令提取器
        └── touchfiles.ts            ← 测试 ↔ 文件依赖映射
```

---

## 附录：快速对照表

### gstack 技术栈

| 技术 | 用途 |
|---|---|
| **Bun** | JavaScript 运行时 + 包管理器 + 编译器 |
| **TypeScript** | Browse 引擎源码语言 |
| **Playwright** | 浏览器自动化库（by Microsoft） |
| **Chromium** | 无头浏览器实例 |
| **Markdown** | Skill 定义格式 |
| **Anthropic SDK** | LLM 评审测试（devDependency） |

### 常用命令

| 命令 | 用途 |
|---|---|
| `./setup` | 一键安装（构建二进制 + 链接 Skills） |
| `bun install` | 安装依赖 |
| `bun run build` | 生成文档 + 编译二进制 |
| `bun test` | 运行免费测试（<5 秒） |
| `bun run gen:skill-docs` | 从模板重新生成 SKILL.md |
| `bun run skill:check` | Skill 健康仪表盘 |
| `bun run dev:skill` | 监听模式开发 |
| `bin/dev-setup` | 进入开发模式 |
| `bin/dev-teardown` | 退出开发模式 |

---

> **这份文档基于 gstack v0.6.4.0 源码逐文件分析生成。**
> 每个技术细节都已对照 ARCHITECTURE.md、CLAUDE.md、CONTRIBUTING.md、package.json、setup 脚本、gen-skill-docs.ts、browse/src/ 源码进行验证。
