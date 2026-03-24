# /benchmark 深度解构

> **角色：** 性能工程师（Performance Engineer）
> **定位：** 基于 browse 守护进程的性能回归检测——建立基线、逐 PR 对比、追踪性能趋势
> **核心原则：** "Measure, don't guess. Bundle size is the leading indicator."
> **allowed-tools：** Bash, Read, Write, Glob, AskUserQuestion
> **来源：** `benchmark/SKILL.md`（v1.0.0）

---

## 一、为什么这个 Skill 存在

**问题：** 性能退化不是一次大崩塌，而是"千刀万剐"——每个 PR 加 50ms，每次合并多 20KB，直到某天应用启动需要 8 秒，没有人知道是什么时候变慢的。

原文是这样描述的：

> "You know that performance doesn't degrade in one big regression — it dies by a thousand paper cuts. Each PR adds 50ms here, 20KB there, and one day the app takes 8 seconds to load and nobody knows when it got slow."

**传统方案的缺陷：**
- 手工跑 Lighthouse 是一次性快照，不能在 PR 维度做前后对比
- 没有自动化基线管理，"快了还是慢了"只靠感觉
- 第三方工具（如 CI 集成的 Lighthouse CI）需要额外部署和配置

**gstack 的方案：** 复用已有的 browse 守护进程（持久化 Chromium），通过浏览器原生 `performance.getEntries()` API 采集真实性能数据，将基线存为 JSON，每次跑完自动比较并给出回归诊断。

**核心判断：** Bundle size 比加载时间更可靠。加载时间随网络条件变化，bundle size 是确定性指标——"Track it religiously."

---

## 二、9 阶段工作流程

```
用户输入 /benchmark <url>
         │
         ▼
┌─────────────────────┐
│  Phase 1: Setup      │  eval gstack-slug
│  环境准备             │  mkdir -p .gstack/benchmark-reports/baselines
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  Phase 2: Page       │  自动发现导航页面，或 --pages 手动指定
│  Discovery           │  --diff 模式：git diff HEAD 推断受影响页面
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  Phase 3: Data       │  $B goto <url>
│  Collection          │  $B perf
│                      │  $B eval "JSON.stringify(performance
│  采集 6 大时序指标    │             .getEntriesByType('navigation')[0])"
│  资源列表 + Bundle   │  JS/CSS 资源过滤 + 网络摘要
└──────────┬──────────┘
           │
    ┌──────┴──────────┐
    │                 │
    ▼                 ▼
--baseline          已有 baseline
模式                继续比较
    │                 │
    ▼                 │
┌─────────────┐      │
│ Phase 4:    │      │
│ 写入         │      │
│ baseline    │      │
│ .json       │      │
└─────────────┘      │
                     ▼
          ┌─────────────────────┐
          │  Phase 5: Comparison │  逐指标 Delta 计算
          │  回归判定（3 档状态） │  OK / WARNING / REGRESSION
          └──────────┬──────────┘
                     │
                     ▼
          ┌─────────────────────┐
          │  Phase 6: Slowest    │  TOP 10 最慢资源 + 每条建议
          │  Resources           │  标注第三方脚本（← third-party）
          └──────────┬──────────┘
                     │
                     ▼
          ┌─────────────────────┐
          │  Phase 7: Budget     │  对照行业预算（FCP<1.8s, LCP<2.5s 等）
          │  Check               │  输出字母评级
          └──────────┬──────────┘
                     │
             --trend 模式？
          ┌──────────┘
          ▼
┌─────────────────────┐
│  Phase 8: Trend      │  读取历史 baseline 文件，绘制趋势表
│  Analysis            │  输出 "LCP doubled in 8 days" 类诊断
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  Phase 9: Save       │  写 .gstack/benchmark-reports/
│  Report              │  {date}-benchmark.md + .json（双格式）
└─────────────────────┘
```

---

## 三、六大时序指标

| 指标 | 计算来源 | 说明 |
|------|----------|------|
| **TTFB**（首字节时间） | `responseStart - requestStart` | 服务端响应速度 |
| **FCP**（首次内容绘制） | PerformanceObserver / paint entries | 用户看到内容的时刻 |
| **LCP**（最大内容绘制） | PerformanceObserver | Core Web Vitals 核心指标 |
| **DOM Interactive** | `domInteractive - navigationStart` | DOM 可交互时刻 |
| **DOM Complete** | `domComplete - navigationStart` | DOM 完全解析时刻 |
| **Full Load** | `loadEventEnd - navigationStart` | 所有资源加载完成 |

资源层面额外采集：
- `total_requests` — 请求总数
- `total_transfer_bytes` — 传输总字节
- `js_bundle_bytes` — JS 包总大小
- `css_bundle_bytes` — CSS 包总大小
- `largest_resources` — 前 N 个最大/最慢资源（name / size / duration）

---

## 四、回归判定阈值

这是 `/benchmark` 最核心的决策层——把原始数字变成可操作的诊断：

```
时序指标：
  >50% 增幅 OR >500ms 绝对增量  → REGRESSION（回归）
  >20% 增幅                      → WARNING（警告）
  其余                            → OK

Bundle 大小：
  >25% 增幅                      → REGRESSION
  >10% 增幅                      → WARNING

请求数量：
  >30% 增幅                      → WARNING
```

**设计哲学：** 使用相对阈值，不用绝对阈值。原文明确说：

> "Relative thresholds, not absolute. 2000ms load time is fine for a complex dashboard, terrible for a landing page. Compare against YOUR baseline."

一个仪表盘 2000ms 加载是合理的；同样 2000ms 的落地页就是问题。只有对照自己的基线才是公平的比较。

---

## 五、行业预算标准（Phase 7）

```
Metric              Budget      对应 Web Vitals 标准
──────────────────────────────────────────────────
FCP                 < 1.8s      Good
LCP                 < 2.5s      Good
Total JS            < 500KB     —
Total CSS           < 100KB     —
Total Transfer      < 2MB       —
HTTP Requests       < 50        —
```

评级公式：通过数 / 总指标数 → 字母评级（A = 全部通过，F = 全部失败）。Phase 7 同时显示 `WARNING (90%)` 这类接近失败的中间状态。

---

## 六、基线文件结构

写入路径：`.gstack/benchmark-reports/baselines/baseline.json`

```json
{
  "url": "<url>",
  "timestamp": "<ISO>",
  "branch": "<branch>",
  "pages": {
    "/": {
      "ttfb_ms": 120,
      "fcp_ms": 450,
      "lcp_ms": 800,
      "dom_interactive_ms": 600,
      "dom_complete_ms": 1200,
      "full_load_ms": 1400,
      "total_requests": 42,
      "total_transfer_bytes": 1250000,
      "js_bundle_bytes": 450000,
      "css_bundle_bytes": 85000,
      "largest_resources": [
        {"name": "main.js", "size": 320000, "duration": 180},
        {"name": "vendor.js", "size": 130000, "duration": 90}
      ]
    }
  }
}
```

**注意：** 基线文件记录 `branch` 字段。比较报告的标题行会显示 `Branch: [current] vs baseline ([baseline-branch])`，方便追踪基线建立于哪个分支。

---

## 七、趋势分析输出（--trend 模式）

`--trend` 模式读取 `.gstack/benchmark-reports/baselines/` 下的历史文件，输出趋势表（以下直接引用源文件示例数据）：

```
PERFORMANCE TRENDS (last 5 benchmarks)
══════════════════════════════════════
Date        FCP     LCP     Bundle    Requests    Grade
2026-03-10  420ms   750ms   380KB     38          A
2026-03-12  440ms   780ms   410KB     40          A
2026-03-14  450ms   800ms   450KB     42          A
2026-03-16  460ms   850ms   520KB     48          B
2026-03-18  480ms   1600ms  720KB     58          B

TREND: Performance degrading. LCP doubled in 8 days.
       JS bundle growing 50KB/week. Investigate.
```

趋势分析的价值：单次数据只能说明"现在"，历史数据才能发现渐进式退化。"每周 +50KB"这类模式在单次测量中完全不可见，在趋势图中一目了然。

---

## 八、调用参数全览

| 参数 | 行为 |
|------|------|
| `/benchmark <url>` | 完整性能审计，如有基线则自动对比 |
| `/benchmark <url> --baseline` | 仅采集基线（变更前运行） |
| `/benchmark <url> --quick` | 单次时序检查，不需要基线 |
| `/benchmark <url> --pages /,/dashboard,/api/health` | 手动指定多页面 |
| `/benchmark --diff` | 只对当前分支影响的页面做基准测试 |
| `/benchmark --trend` | 展示历史趋势（读取所有历史 baseline 文件） |

**`--diff` 模式的实现细节（源文件原文）：**
```bash
git diff $(gh pr view --json baseRefName -q .baseRefName 2>/dev/null \
  || gh repo view --json defaultBranchRef -q .defaultBranchRef.name 2>/dev/null \
  || echo main)...HEAD --name-only
```
优先用 `gh pr view` 获取 PR 的 base branch，回退到仓库默认分支，最后回退到 `main`。

---

## 九、五条核心规则（Important Rules）

源文件明确列出的不可破坏约束：

1. **Measure, don't guess.** 使用真实的 `performance.getEntries()` 数据，不用估算值
2. **Baseline is essential.** 没有基线只能报绝对值，无法检测回归；始终鼓励先建基线再改代码
3. **Relative thresholds, not absolute.** 对照自己的基线，而非套用行业绝对阈值
4. **Third-party scripts are context.** 标记第三方脚本（Google Analytics 等），但不把它们当主要优化目标——用户无法控制
5. **Read-only.** 只生成报告，不修改代码，除非用户明确要求

---

## 十、与其他 Skill 的关系

```
                        /browse（基础设施）
                             │
                    提供 $B goto / $B perf 能力
                             │
                             ▼
/qa ─────────────────────► /benchmark ◄──────── /canary
功能测试完成后              性能回归检测           Page Discovery
可触发性能验证              建立基线、对比 Delta   逻辑相同（同源注释：
                            输出回归诊断           "Same as /canary"）
                                 │
                                 ▼
                    .gstack/benchmark-reports/
                    历史 baseline JSON 文件
                                 │
                                 ▼
                         /benchmark --trend
                         读取历史文件，展示趋势

/ship（发布前） ──────► 可调用 /benchmark 做性能门控检查
```

| Skill | 关系说明 |
|-------|----------|
| `/browse` | `/benchmark` 完全依赖 browse 守护进程；`$B goto` 导航、`$B perf` 采集原始指标 |
| `/qa` | `/qa` 验证功能正确性；`/benchmark` 验证性能不退化，两者互补 |
| `/canary` | `--diff` 的 Page Discovery 与 `/canary` 实现相同（源文件注明 "Same as /canary"） |
| `/ship` | 发布前可运行 `/benchmark` 作为性能门控，确认无退化再合并 |

---

## 十一、Prompt Engineering 可迁移模式

### 模式 1：基线比较优于绝对阈值

```
Compare against YOUR baseline.
Relative thresholds, not absolute.
```

**价值：** 绝对阈值（"加载时间必须 < 2s"）无法适应不同复杂度的页面。基线比较把"好"的定义锚定在自身历史数据上，适用于任何需要"前后对比"的检测场景——性能、包大小、API 响应时间、测试覆盖率皆可。

**迁移方式：** 在任何监测类 Skill 中，存储 JSON 格式的基线快照，下次运行时自动加载并计算 Delta。

### 模式 2：确定性指标优先于波动性指标

```
Bundle size is the leading indicator.
Load time varies with network. Bundle size is deterministic.
```

**价值：** 在有多个候选指标时，选择最确定性的那个作为主要追踪目标。加载时间受网络、服务器负载、缓存状态干扰；bundle size 是纯粹的代码产物。

**迁移方式：** 识别你监测领域中的"确定性指标"优先追踪。代码质量中圈复杂度比测试通过率更确定；API 中 payload 大小比延迟更确定。

### 模式 3：三级状态输出（OK / WARNING / REGRESSION）

```
>50% OR >500ms absolute → REGRESSION
>20%                    → WARNING
else                    → OK
```

**价值：** 二元判断（Pass/Fail）要么太宽松（小退化被忽略）要么太严格（正常波动触发警报）。三级状态允许渐进式响应：WARNING 是"留意"，REGRESSION 是"必须处理"。

**迁移方式：** 在任何评分或检测 Skill 中，设计三档而非两档。中间档（WARNING）专用于"值得关注但不阻塞发布"的场景。

### 模式 4：第三方资源上下文标注

```
Flag them, but the user can't fix Google Analytics being slow.
Focus recommendations on first-party resources.
```

**价值：** 报告时区分"用户能影响的"和"用户无法控制的"。把第三方资源单独标注（`← third-party`），避免用户为无法修复的问题焦虑，同时保持报告完整性。

**迁移方式：** 在任何诊断型输出中，对每个发现附加"可行动性"标签：`← third-party（无法控制）` 或 `← first-party（建议优化）`。

### 模式 5：历史数据持久化驱动趋势发现

```
TREND: Performance degrading. LCP doubled in 8 days.
       JS bundle growing 50KB/week.
```

**价值：** 单次快照只能说明"现在"，历史数据才能发现"模式"。渐进式退化（每 PR +50ms）在单次测量中不可见，在趋势图中一目了然。

**迁移方式：** 任何定期运行的 Skill 都应将结果持久化，并提供 `--trend` 模式。文件命名规范：`{date}-{skill-name}.json`，存放于 `.gstack/{skill-name}-reports/` 下。这样 `--trend` 只需按日期排序读取目录下所有文件即可。

---

> 本分析基于 `benchmark/SKILL.md`（v1.0.0）全文，包括 Preamble 通用框架、Browse Setup 协议和 `/benchmark` 专属指令（9 个阶段、5 条核心规则、6 大指标定义）的完整解读。所有阈值数据、示例输出、bash 命令均直接引用源文件，未做添改。
