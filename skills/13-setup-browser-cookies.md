# /setup-browser-cookies 深度解构

> **角色：** 会话管理员（Session Manager）
> **定位：** 从真实浏览器导入 Cookie 到无头 Chromium 会话，让 `/qa` 和 `/browse` 能测试登录后的页面
> **Prompt 长度：** ~74 行（gstack 中最短的 Skill）
> **allowed-tools：** Bash, Read, AskUserQuestion
> **来源：** `setup-browser-cookies/SKILL.md.tmpl`

---

## 一、为什么需要这个 Skill

**问题：** `/qa` 和 `/browse` 使用的是无头 Chromium 浏览器 — 它没有你的登录态。如果你要测试登录后的页面（如 GitHub Settings、后台管理面板、用户 Dashboard），你需要先登录。

**传统方案：** 在无头浏览器中走一遍登录流程（输入用户名、密码、处理 2FA）。这很慢、很脆弱，而且很多网站有反机器人检测。

**gstack 的方案：** 直接从你日常使用的真实浏览器（Chrome、Arc、Brave、Edge）中**导入已有的 Cookie**。你在 Chrome 里已经登录了 GitHub？一条命令就把这个登录态搬到无头浏览器里。

---

## 二、完整结构拆解

### 2.1 工作流程

```
用户输入 /setup-browser-cookies
        │
        ▼
检测本地安装的 Chromium 浏览器
（Comet, Chrome, Arc, Brave, Edge）
        │
        ▼
┌─────────────────────────────┐
│ 方式 A：交互式 Cookie 选择器  │
│ 打开一个 Web UI              │
│ 用户勾选要导入的域名          │
│ （只显示域名和数量，不显示值） │
└──────────┬──────────────────┘
           │
           ▼
┌─────────────────────────────┐
│ 方式 B：直接指定域名          │
│ $B cookie-import-browser     │
│    comet --domain .github.com│
└──────────┬──────────────────┘
           │
           ▼
Cookie 注入到 Playwright 会话
后续所有 $B 命令自动携带这些 Cookie
```

### 2.2 两种使用方式

**方式 A：交互式选择器（默认）**

```bash
$B cookie-import-browser
```

这会启动一个本地 Web UI（dark 主题，与 browse 服务器共用端口），展示所有检测到的浏览器和它们的 Cookie 域名。用户通过 + 按钮选择要导入的域名，通过垃圾桶按钮移除。

**原文：**
```
Tell the user: "Cookie picker opened — select the domains you want to import
in your browser, then tell me when you're done."
Wait for the user to confirm they've finished selecting.
```

**翻译：** 告诉用户："Cookie 选择器已打开 — 在浏览器中选择你要导入的域名，选完后告诉我。" 等用户确认选完。

**方式 B：直接导入指定域名**

```bash
$B cookie-import-browser comet --domain .github.com
```

跳过 UI，直接从指定浏览器导入指定域名的 Cookie。适合你已经知道需要哪些 Cookie 的场景。

### 2.3 验证步骤

导入后运行：
```bash
$B cookies
```

显示已导入的 Cookie 摘要（域名和数量）。

### 2.4 安全设计

| 安全措施 | 细节 |
|---|---|
| **只显示域名和数量** | Cookie 选择器 UI 和 `cookies` 命令都不显示 Cookie 的实际值 |
| **macOS Keychain 授权** | 首次导入会触发 macOS Keychain 弹窗，用户必须点击"允许"才能解密 Cookie |
| **内存中解密** | Cookie 值通过 PBKDF2 + AES-128-CBC 在内存中解密，从不写入磁盘明文 |
| **数据库只读** | 复制 Chromium 的 Cookie 数据库到临时文件再读取，不修改原浏览器的数据库 |
| **会话级缓存** | Keychain 密码和 AES 密钥缓存在服务器内存中，服务器退出后缓存消失 |

---

## 三、底层技术原理

### Cookie 解密流程（macOS）

Chromium 浏览器在 macOS 上用以下方式加密 Cookie：

```
1. 从 macOS Keychain 获取浏览器特定的密码
   （每个浏览器有不同的 Keychain 条目名）
2. 用 PBKDF2 从密码派生 AES-128-CBC 密钥
   （1003 次迭代，特定 salt）
3. 用 AES-128-CBC 解密 Cookie 值
4. 解密后的 Cookie 注入到 Playwright 的 BrowserContext
```

gstack 的 `cookie-import-browser.ts` 实现了这个完整流程。它支持的浏览器注册表是**硬编码**的（安全考虑 — 不接受用户输入的浏览器路径）：

| 浏览器 | Keychain 条目 | Cookie 数据库路径 |
|---|---|---|
| Comet | Comet Safe Storage | ~/Library/Application Support/Comet/Default/Cookies |
| Chrome | Chrome Safe Storage | ~/Library/Application Support/Google/Chrome/Default/Cookies |
| Arc | Arc Safe Storage | ~/Library/Application Support/Arc/User Data/Default/Cookies |
| Brave | Brave Safe Storage | ~/Library/Application Support/BraveSoftware/Brave-Browser/Default/Cookies |
| Edge | Microsoft Edge Safe Storage | ~/Library/Application Support/Microsoft Edge/Default/Cookies |

### Cookie 选择器 UI

`cookie-picker-ui.ts` 是一个自包含的 HTML/CSS/JS 页面（dark 主题，无外部依赖），通过 browse 服务器的 HTTP 路由提供。用户在真实浏览器中打开这个 UI，选择要导入的域名。

---

## 四、使用场景

### 场景 1：测试 GitHub 登录后的页面

```
/setup-browser-cookies github.com
/qa https://github.com/settings/profile
```

### 场景 2：测试公司内部管理后台

```
/setup-browser-cookies
（在 Cookie 选择器中选择 admin.mycompany.com）
/browse
$B goto https://admin.mycompany.com/dashboard
$B snapshot -i
$B screenshot /tmp/admin-dashboard.png
```

### 场景 3：QA 测试需要多个登录态

```
/setup-browser-cookies
（选择 myapp.com 和 stripe.com 的域名）
/qa https://staging.myapp.com
（Claude 现在可以测试支付流程，因为有 Stripe 的 Cookie）
```

---

## 五、迁移价值

### 可复用的设计模式

1. **交互式 + 直接模式双通道：** 复杂操作提供 UI，简单操作提供命令行。让用户根据场景选择
2. **安全第一的信息显示：** 只显示元数据（域名/数量），不显示敏感值（Cookie 内容）
3. **会话级状态：** 导入的 Cookie 在浏览器会话期间持续有效，30 分钟空闲后随服务器一起消失

### 限制

- **仅支持 macOS：** Keychain 解密只实现了 macOS。Linux（GNOME Keyring/kwallet）和 Windows（DPAPI）理论可行但未实现
- **仅支持 Chromium 系浏览器：** Firefox 使用不同的加密方案，不支持
- **需要用户授权：** 首次使用会弹出 Keychain 授权对话框

---

> 本分析基于 `setup-browser-cookies/SKILL.md.tmpl`（74 行）+ `browse/src/cookie-import-browser.ts`（16KB）+ `browse/src/cookie-picker-ui.ts`（17KB）+ `browse/src/cookie-picker-routes.ts`（8KB）源码分析。
