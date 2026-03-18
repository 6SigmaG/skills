# /browse 深度解构

> **角色：** QA 工程师（QA Engineer）
> **定位：** 给 AI Agent 一个真实的 Chromium 浏览器——持久化、亚秒级、50+ 命令
> **Prompt 长度：** ~484 行（自动生成自 SKILL.md.tmpl）
> **allowed-tools：** Bash, Read, AskUserQuestion
> **来源：** `SKILL.md.tmpl` + `browse/src/` 14 个 TypeScript 文件

---

## 一、为什么这个 Skill 是 gstack 的基础设施

`/browse` 不是一个"角色 Skill"——它是一个**能力 Skill**。其他 Skill（`/qa`、`/qa-only`、`/design-review`、`/setup-browser-cookies`）都依赖它提供的浏览器能力。

没有 `/browse`，Claude Code 是"半盲"的——它能写代码但看不到网页。有了 `/browse`，Agent 能导航、点击、填表、截图、断言元素状态、检查控制台错误。

**核心技术决策：** gstack 没有用 MCP 协议或 WebSocket 流，而是选择了最简单的架构——**编译后的 CLI 二进制 + localhost HTTP + 持久化 Chromium 守护进程**。每个命令就是一次 HTTP POST，返回纯文本。

---

## 二、架构拆解

### 2.1 守护进程模型

```
Claude Code                     gstack Browse
─────────                      ─────────────
                               ┌──────────────────────┐
  Bash: $B snapshot -i    →    │  CLI 二进制 (58MB)     │
                               │  读 .gstack/browse.json│
                               │  POST /command         │
                               └──────────┬────────────┘
                                          │ HTTP + Bearer Token
                               ┌──────────▼────────────┐
                               │  Bun.serve() 服务器    │
                               │  命令分发               │
                               └──────────┬────────────┘
                                          │ Playwright (CDP)
                               ┌──────────▼────────────┐
                               │  Chromium（无头）       │
                               │  持久化 tabs + cookies  │
                               │  30 分钟空闲自动退出    │
                               └────────────────────────┘
```

**为什么是守护进程？** 每次冷启动 Chromium 需要 2-3 秒。一次 QA 会话有 20+ 命令，冷启动模式会浪费 40-60 秒纯启动时间，且每次丢失 Cookie/登录态。守护进程模式：首次 ~3 秒，后续每命令 ~100ms，状态跨命令保持。

**状态文件：** `.gstack/browse.json`（原子写入，权限 0o600）
```json
{ "pid": 12345, "port": 34567, "token": "uuid-v4",
  "startedAt": "...", "binaryVersion": "abc123" }
```

CLI 读此文件找到服务器。如果 PID 已死或版本不匹配，自动启动新服务器。

### 2.2 Ref 系统——AI 怎么"指"网页元素

**问题：** AI 要点击按钮，怎么告诉浏览器"点哪个"？

**传统方案：** CSS 选择器或 XPath——AI 容易猜错。

**gstack 方案：** 先拍快照（snapshot），给每个可交互元素分配编号（@e1, @e2...），然后用编号操作。

```bash
$B snapshot -i           # 拍快照，列出所有可交互元素
# @e1 [heading] "Welcome" [level=1]
# @e2 [textbox] "Email"
# @e3 [button] "Submit"

$B fill @e2 "test@example.com"   # 用编号操作
$B click @e3
```

**内部实现：**
1. 调用 `page.accessibility.snapshot()` 获取 ARIA 可访问性树
2. 遍历树，为每个元素分配 @e 编号
3. 为每个 Ref 构建 Playwright Locator：`getByRole(role, { name }).nth(index)`
4. 存储 `Map<string, RefEntry>`

**为什么不注入 DOM 属性（如 `data-ref`）？**
- CSP（内容安全策略）可能禁止 DOM 修改
- React/Vue 水合会删掉注入的属性
- Shadow DOM 从外部不可达

Playwright Locator 在 DOM 外部工作，不修改页面。

**过期检测：** SPA 可以在不触发导航的情况下改变页面。`resolveRef()` 在使用前做 `count()` 检查（~5ms），元素不存在则立即报错，而非等 30 秒超时。

**@c refs（cursor-interactive）：** `-C` flag 找到 ARIA 树遗漏的可点击元素（`cursor: pointer` 的 div、`onclick` 属性等），分配 @c1, @c2 编号。

### 2.3 Snapshot 系统

Snapshot 是 `/browse` 最核心的命令，有 8 个 flag：

| Flag | 作用 |
|---|---|
| `-i` | 只显示可交互元素（按钮、链接、输入框）+ @e refs |
| `-c` | 紧凑模式（去除空结构节点） |
| `-d N` | 限制树深度 |
| `-s sel` | 限定 CSS 选择器范围 |
| `-D` | 与上次 snapshot 的 diff（统一 diff 格式） |
| `-a` | 生成带红色标注框和 ref 标签的截图 |
| `-o path` | 标注截图的输出路径 |
| `-C` | 扫描 cursor-interactive 元素（@c refs） |

所有 flag 可自由组合。例如：`$B snapshot -i -a -C -o /tmp/annotated.png`

### 2.4 命令分类（READ / WRITE / META）

50+ 命令按副作用分三类：

**READ（无副作用，可安全重试）：**
text, html, links, console, cookies, accessibility, forms, attrs, css, js, is, network, perf, storage, eval

**WRITE（改变页面状态，不可幂等）：**
goto, click, fill, press, scroll, select, type, hover, upload, cookie, cookie-import, cookie-import-browser, header, useragent, viewport, wait, dialog-accept, dialog-dismiss

**META（服务器级操作）：**
snapshot, screenshot, responsive, pdf, diff, chain, tabs, newtab, tab, closetab, status, stop, restart

这个分类驱动命令分发：
```typescript
if (READ_COMMANDS.has(cmd))  → handleReadCommand(cmd, args, bm)
if (WRITE_COMMANDS.has(cmd)) → handleWriteCommand(cmd, args, bm)
if (META_COMMANDS.has(cmd))  → handleMetaCommand(cmd, args, bm, shutdown)
```

### 2.5 安全模型

| 层面 | 措施 |
|---|---|
| 网络 | 绑定 localhost，不可外部访问 |
| 认证 | 每次启动生成 UUID Token，状态文件 0o600 权限 |
| Cookie | 内存解密，不写磁盘明文，不显示 Cookie 值 |
| 注入防护 | 浏览器列表硬编码，用参数数组不用字符串拼接 |

### 2.6 面向 Agent 的错误设计

每条错误消息都告诉 Agent **下一步该做什么**：
```
"Element not found or not interactable. Run `snapshot -i` to see available elements."
"Navigation timed out after 30s. The page may be slow or the URL may be wrong."
"Use 'browse select' instead of 'click' for dropdown options."
```

崩溃恢复：Chromium 崩溃时服务器立即退出。CLI 在下次命令时检测到死服务器，自动重启。简单可靠。

---

## 三、SKILL.md 中的关键工作流模式

模板提供了 10 个即用的 QA 工作流：

1. **测试用户流程**（登录、注册、结账）—— goto → snapshot -i → fill → click → snapshot -D → is visible → screenshot
2. **验证部署** —— goto → text → console → network → is visible → screenshot
3. **端到端 dogfooding** —— snapshot -i -a -o → snapshot -C → click → snapshot -D → console
4. **响应式测试** —— responsive（自动 3 个尺寸）或手动 viewport → screenshot
5. **文件上传** —— upload @ref /path/to/file
6. **表单验证** —— 空提交 → snapshot -D（显示错误）→ 填写 → 重新提交 → snapshot -D（显示成功）
7. **对话框测试** —— dialog-accept → click 触发 → dialog 查看 → snapshot -D
8. **认证页面** —— cookie-import-browser → goto 登录后页面
9. **页面对比** —— diff url1 url2
10. **多步链** —— echo '[["goto","url"],["snapshot","-i"],...]' | $B chain

---

## 四、迁移价值

### 可复用的架构模式

1. **守护进程 + HTTP + Token 认证**：任何需要持久状态的 CLI 工具都可以用这个模式
2. **Ref 系统**：基于可访问性树的元素引用，比 CSS 选择器更稳定
3. **READ/WRITE/META 命令分类**：清晰的副作用边界，方便重试和调试
4. **面向 Agent 的错误消息**：每条错误附带恢复指令

### 为什么不用 MCP？

ARCHITECTURE.md 明确说明：

> "MCP adds JSON schema overhead per request and requires a persistent connection. Plain HTTP + plain text output is lighter on tokens and easier to debug."

HTTP + 纯文本 = 更少的 token 消耗 + 可以用 curl 调试。

---

> 本分析基于 `SKILL.md.tmpl`（484 行）+ `browse/src/` 全部 14 个 TypeScript 文件 + `ARCHITECTURE.md` 源码分析。
