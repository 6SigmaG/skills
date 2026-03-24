# /canary 深度解构

> **角色：** Release Reliability Engineer / 发布可靠性工程师
> **定位：** 部署后生产环境金丝雀监控——截图对比、控制台错误检测、性能回归告警
> **Prompt 长度：** ~531 行（自动生成自 SKILL.md.tmpl）
> **allowed-tools：** Bash, Read, Write, Glob, AskUserQuestion
> **来源：** `canary/SKILL.md.tmpl`（v1.0.0）

---

## 一、为什么这个 Skill 存在

CI 通过不代表生产没问题。/canary 的存在基于一个反复出现的现实：**代码在测试环境里工作，但在真实生产环境里悄悄坏掉**。

原文给出了三个典型场景：

> "a missing environment variable, a CDN cache serving stale assets, a database migration that's slower than expected on real data"

这三个场景的共同特征是——它们**不会被 CI 捕获**：CI 在受控环境里跑，没有真实用户流量、没有 CDN、没有真实数据规模。问题暴露的时机往往是部署后 10 分钟到几小时之间，等用户报告时已经影响了大量真实用户。

Skill 原文对工程师角色的定义一语中的：

> "Your job is to catch these in the first 10 minutes, not 10 hours."

/canary 的职责是把生产问题的发现窗口从"几小时"压缩到"10 分钟内"。

### 核心设计哲学：基于变化告警，而非绝对阈值

/canary 最重要的设计决策写在 Important Rules 里：

> "Alert on changes, not absolutes. Compare against baseline, not industry standards."

一个在基线中已有 3 个控制台错误的页面，监控期间仍有 3 个——**不告警**。同一页面出现第 4 个新错误——**立即告警**。

这个设计避免了"噪音驱逐信号"的问题：如果按行业标准告警，历史遗留问题会让监控系统持续鸣叫，直到没人再关注它。只对"变化"告警，让每一次告警都有意义。

---

## 二、工作流程

### 2.1 整体架构：七阶段流水线

```
Phase 1: Setup（目录初始化）
    ↓
Phase 2: Baseline Capture（--baseline 模式，部署前运行）
    ↓  [部署发生]
Phase 3: Page Discovery（自动发现要监控的页面）
    ↓
Phase 4: Pre-Deploy Snapshot（无 baseline 时建立参考点）
    ↓
Phase 5: Continuous Monitoring Loop（持续监控，每 60 秒一轮）
    ↓
Phase 6: Health Report（健康报告 + 文件持久化）
    ↓
Phase 7: Baseline Update（健康时可选更新基线）
```

### 2.2 两种运行模式

| 模式 | 命令 | 时机 | 作用 |
|---|---|---|---|
| 基线捕获 | `/canary <url> --baseline` | 部署**前** | 记录"已知好"的状态作为参照 |
| 监控模式 | `/canary <url>` | 部署**后** | 与基线对比，检测异常 |
| 自定义时长 | `/canary <url> --duration 5m` | 部署后 | 1m 到 30m 均可 |
| 指定页面 | `/canary <url> --pages /,/dashboard` | 部署后 | 跳过自动发现 |
| 快速检查 | `/canary <url> --quick` | 任何时候 | 单次扫描，不持续循环 |

**原文（Important Rules）：**
> "Baseline is king. Without a baseline, canary is a health check. Encourage `--baseline` before deploying."

没有基线，/canary 只能检测绝对异常（完全无法访问）。有了基线，才能检测相对回归（变慢了、多了错误）。这个两阶段设计将"健康检查"升级为"回归检测"。

### 2.3 详细执行流程图

```
用户输入 /canary <url> --baseline
         │
         ▼
    Phase 1: Setup
    eval gstack-slug
    mkdir -p .gstack/canary-reports/{baselines,screenshots}
         │
         ▼
    Phase 2: Baseline Capture（--baseline 模式）
    对每个页面执行：
      $B goto <page-url>
      $B snapshot -i -a -o baselines/<page>.png
      $B console --errors
      $B perf
      $B text
    → 写入 .gstack/canary-reports/baseline.json
    → 停止，告知用户："Baseline captured. Deploy, then run /canary <url>"

         │ [用户部署]
         │
         ▼
    用户输入 /canary <url>
         │
         ▼
    Phase 3: Page Discovery（无 --pages 参数时）
    $B goto <url>
    $B links
    → 提取 top 5 内部导航链接 + 首页
    → AskUserQuestion 确认监控页面列表
    选项：
      A) 监控这些发现的页面
      B) 添加更多页面
      C) 只监控首页（快速检查）
         │
         ▼
    Phase 4: Pre-Deploy Snapshot（无 baseline.json 时）
    对每个页面记录：控制台错误数、加载时间
    作为本次监控的参考基准
         │
         ▼
    Phase 5: Continuous Monitoring Loop
    ┌─────────────────────────────────────────────┐
    │  每 60 秒，对每个页面：                       │
    │  $B goto → $B snapshot -i -a -o             │
    │  → $B console --errors → $B perf            │
    │                                             │
    │  对比基线，按优先级分级：                      │
    │  CRITICAL: goto 返回错误或超时               │
    │  HIGH:     出现基线中没有的新控制台错误        │
    │  MEDIUM:   加载时间 > 基线 2x               │
    │  LOW:      出现新的 404 链接                 │
    │                                             │
    │  同一问题持续 2+ 次才触发告警（防误报）         │
    │                                             │
    │  CRITICAL/HIGH 触发时 → 立即 AskUserQuestion │
    └─────────────────────────────────────────────┘
         │
         ▼
    Phase 6: Health Report
    输出 CANARY REPORT 表格（每页状态 + 汇总）
    → 保存 .gstack/canary-reports/{date}-canary.md
    → 保存 .gstack/canary-reports/{date}-canary.json
    → 写 JSONL 摘要到 ~/.gstack/projects/$SLUG
         │
         ▼
    Phase 7: Baseline Update（仅健康时提供选项）
    → 将本次截图更新为新基线 + 刷新 baseline.json
```

---

## 三、关键监控机制

### 3.1 四级告警体系

/canary 定义了四级告警，每级有明确触发条件，严重程度递减：

| 级别 | 触发条件 | 原文描述 |
|---|---|---|
| **CRITICAL** | 页面完全无法加载（goto 错误或超时） | "Page load failure — goto returns error or timeout" |
| **HIGH** | 出现基线中没有的新控制台错误 | "New console errors — errors not present in baseline" |
| **MEDIUM** | 加载时间超过基线 2 倍 | "Performance regression — load time exceeds 2x baseline" |
| **LOW** | 出现基线中没有的新 404 链接 | "Broken links — new 404s not in baseline" |

**设计原理：** MEDIUM 的阈值是 **2x 基线**，而非某个固定毫秒数。1200ms 的页面，需到 2400ms 才告警；200ms 的页面，400ms 就告警。这体现了 Important Rules 中的原则：

> "Performance thresholds are relative. 2x baseline is a regression. 1.5x might be normal variance."

### 3.2 防误报机制：两次确认原则

**原文：**
> "Don't cry wolf. Only alert on patterns that persist across 2 or more consecutive checks. A single transient network blip is not an alert."

这个设计解决了监控系统最常见的问题：网络瞬时抖动导致的假阳性告警疲劳。用户被假阳性淹没后会停止关注真正的告警。"两次确认"以 60 秒代价换取了更高的信噪比。

### 3.3 CANARY ALERT 的标准化格式

当 CRITICAL 或 HIGH 告警触发时，必须立即通过 AskUserQuestion 通知用户，格式固定：

```
CANARY ALERT
════════════
Time:     [check #N at Xs]
Page:     [page URL]
Type:     [CRITICAL / HIGH / MEDIUM]
Finding:  [具体变化内容，要明确]
Evidence: [截图路径]
Baseline: [基线值]
Current:  [当前值]
```

`Evidence`（截图路径）是不可省略的。原文规则明确：

> "Screenshots are evidence. Every alert includes a screenshot path. No exceptions."

截图路径使告警可独立验证：用户不需要信任 AI 的文字描述，可以直接打开截图确认。

### 3.4 告警后的四个决策选项

告警触发后，AskUserQuestion 同时提供四个响应路径：

- **A) Investigate now** — 停止监控，深入调查此问题
- **B) Continue monitoring** — 可能是瞬时问题，等下一轮检查
- **C) Rollback** — 立即回滚部署
- **D) Dismiss** — 假阳性，继续监控

四个选项覆盖了实际场景中的所有决策路径。选项 C（回滚）的存在尤为关键——/canary 的最终目的不只是报告问题，而是让用户能在第一时间决定是否需要回滚。

### 3.5 Baseline JSON 结构

基线以 JSON 格式保存，每个页面记录截图路径、控制台错误数、加载时间三项数据：

```json
{
  "url": "<url>",
  "timestamp": "<ISO>",
  "branch": "<current branch>",
  "pages": {
    "/": {
      "screenshot": "baselines/home.png",
      "console_errors": 0,
      "load_time_ms": 450
    }
  }
}
```

`branch` 字段记录了基线捕获时的 Git 分支，使基线与代码版本绑定——知道"这个基线是在哪个分支、什么时候捕获的"，便于追溯。

### 3.6 Health Report 格式

监控结束后输出标准化报告，含每页详细状态和最终判决：

```
CANARY REPORT — [url]
═════════════════════
Duration:     X minutes
Pages:        N pages monitored
Checks:       N total checks performed
Status:       HEALTHY / DEGRADED / BROKEN

Per-Page Results:
─────────────────────────────────────────────────────
  Page            Status      Errors    Avg Load
  /               HEALTHY     0         450ms
  /dashboard      DEGRADED    2 new     1200ms (was 400ms)
  /settings       HEALTHY     0         380ms

Alerts Fired:  N (X critical, Y high, Z medium)
Screenshots:   .gstack/canary-reports/screenshots/

VERDICT: DEPLOY IS HEALTHY / DEPLOY HAS ISSUES — details above
```

报告同时保存为 `.gstack/canary-reports/{date}-canary.md` 和 `.gstack/canary-reports/{date}-canary.json` 两种格式，并将 JSONL 摘要写入 `~/.gstack/projects/$SLUG` 供后续仪表盘使用。

---

## 四、与其他 Skill 的关系

### 4.1 依赖 /browse 提供浏览器能力

/canary 完全依赖 browse 守护进程（`$B`）执行所有浏览器操作。在运行任何监控命令前，Skill 必须先通过 SETUP 检查确认 browse 二进制可用：

```bash
[ -x "$B" ] → READY（继续执行）
           → NEEDS_SETUP（告知用户需要一次性构建，等待确认）
```

browse 为 /canary 提供的关键命令：

| 命令 | 用途 |
|---|---|
| `$B goto <url>` | 导航到页面，检测加载失败（CRITICAL 告警源） |
| `$B snapshot -i -a -o <path>` | 截图（带标注），作为告警证据 |
| `$B console --errors` | 获取控制台错误，检测新错误（HIGH 告警源） |
| `$B perf` | 获取页面加载时间，检测性能回归（MEDIUM 告警源） |
| `$B links` | 提取页面链接（页面发现 + 404 检测） |
| `$B text` | 获取页面文本内容（基线捕获时用于内容快照） |

### 4.2 在发布流水线中的位置

/canary 是部署后的最后一道安全网，与其他 Skill 形成完整保障链：

```
/qa（功能验证，部署前）
    ↓
/ship（测试 + PR 创建）
    ↓
/setup-deploy（配置部署环境）
    ↓
/land-and-deploy（执行部署，Step 7 自动调用 /canary）
    ↓
/canary --baseline（部署前捕获基线）
    ↓  [实际部署]
/canary <url>（部署后持续监控，默认 10 分钟）
```

### 4.3 与 /qa 和 /benchmark 的区别

| 维度 | /qa | /canary | /benchmark |
|---|---|---|---|
| **时机** | 部署前 | 部署后 | 任何时候 |
| **环境** | 开发/测试环境 | 真实生产环境 | 通常是受控环境 |
| **操作** | 可以修改代码 | 只读，不修改代码 | 只读 |
| **目标** | 发现并修复 Bug | 检测生产回归、触发告警 | 建立性能基准 |
| **持续性** | 单次运行 | 循环（默认 10 分钟） | 单次或多次测量 |

/canary 的 Important Rules 明确：

> "Read-only. Observe and report. Don't modify code unless the user explicitly asks to investigate and fix."

---

## 五、Prompt Engineering 可迁移模式

### 模式一：基于变化的相对阈值告警

**核心思路：** 不设固定阈值，而是与自身历史基线对比，只告警"变化"。

**为什么有效：** 每个应用的"正常状态"不同。固定阈值（如"加载超过 2 秒告警"）对慢应用是噪音，对快应用又太宽松。相对阈值（"超过基线 2x"）自适应每个应用的实际情况，且对熟悉自身基线的工程师更直观——"比平时慢一倍"比"超过 2000ms"更有意义。

**可迁移场景：** 成本监控（比上周涨 50%？）、流量异常（比昨日同时段低 30%？）、任何需要监控"变化"而非"绝对值"的场景。

### 模式二：截图作为不可省略的告警证据

**核心思路：** 每一条告警必须附带截图路径，作为可独立验证的物证。规则明确"No exceptions"。

**为什么有效：** AI 的文字描述可能有误，但截图不会说谎。截图将告警从"主观声明"变成"客观证据"，使用户能独立验证，也使告警有更高的可信度。这个模式的成本极低——`$B snapshot -i -a -o <path>` 一条命令即可。

**可迁移场景：** 任何涉及视觉状态变化的监控或测试类 Skill——截图 + 路径是构建最轻量"证据链"的方式。

### 模式三：两阶段工作流（先捕获基线，再监控对比）

**核心思路：** 在状态变化前捕获"已知好"的状态，然后在变化后对比检测异常。

**为什么有效：** 没有基线，监控只能检测绝对异常（完全崩溃）。有了基线，才能检测相对回归（变慢了、多了错误、布局变了）。这个两阶段设计将"健康检查"（health check）升级为"回归检测"（regression detection），检测能力提升一个量级。

**可迁移场景：** A/B 测试、性能基准测试、视觉回归测试——任何"对比部署前后"的场景都适用此模式。

### 模式四：分级告警 + 即时决策菜单

**核心思路：** 告警分四级，高级别告警立即打断监控循环，并同时提供包含"回滚"在内的完整决策选项。

**为什么有效：** 单一告警级别导致两种极端——全部忽略（噪音太多）或全部紧急响应（不知轻重）。四级分类让响应与风险匹配。更重要的是，告警时直接提供决策选项（包括"回滚"），缩短了从发现问题到采取行动的时间。如果告警只说"有问题"，用户还需要单独决定下一步；告警同时呈现选项，决策过程内嵌于告警中。

**可迁移场景：** 任何需要人工介入决策的监控类 Skill——关键是告警时不只通知问题，还要同时呈现可选的应对路径。

### 模式五：速度优先，先监控再分析

**原文（Important Rules）：**
> "Speed matters. Start monitoring within 30 seconds of invocation. Don't over-analyze before monitoring."

**核心思路：** 监控类 Skill 的优先级是快速开始，而非完美准备。先监控，边监控边收集信息。

**为什么有效：** 部署后的前 10 分钟是最关键的发现窗口。如果 Skill 花 5 分钟"分析需要监控哪些页面"再开始，可能已经错过了 CRITICAL 告警。正确优先级是：30 秒内开始监控，用 Phase 3（页面发现）在第一轮检查后再细化。

**可迁移场景：** 任何时间敏感的 Skill——"先做，边做边完善"比"分析完再做"更适合紧迫场景。这与 /canary 的 read-only 原则配合良好：因为不会修改任何东西，快速开始的风险极低。

---

> 本分析基于 `canary/SKILL.md.tmpl`（v1.0.0，含 7 个执行阶段、4 级告警体系、两阶段基线管理机制、5 条 Important Rules）逐段拆解。/canary 是 gstack 发布流水线的最后一道安全网，将生产问题的发现窗口从"小时级"压缩到"分钟级"。
