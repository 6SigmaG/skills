# /setup-deploy 深度解构

> **角色：** 部署配置器（Deploy Configurator）
> **定位：** 一次性检测部署平台并写入配置，让 /land-and-deploy 自动化运行
> **allowed-tools：** Bash, Read, Write, Edit, Glob, Grep, AskUserQuestion
> **来源：** `setup-deploy/SKILL.md`

---

## 一、为什么这个 Skill 存在

**问题：** /land-and-deploy 需要知道你的部署平台、生产 URL、健康检查端点和部署状态命令。每次部署都手动输入太麻烦。

**gstack 的方案：** 一次运行 /setup-deploy，自动检测平台并把配置写入 CLAUDE.md。之后所有 /land-and-deploy 运行都自动读取配置。

---

## 二、6 步工作流

```
Step 1: 检查现有配置
  └── CLAUDE.md 中是否已有 Deploy Configuration？
       ├── 有 → 展示并确认是否需要更新
       └── 没有 → 继续检测
           │
           ▼
Step 2: 平台检测
  ├── 配置文件检测（fly.toml, render.yaml, vercel.json...）
  ├── GitHub Actions 部署工作流检测
  └── 项目类型检测
           │
           ▼
Step 3: 平台特定设置
  ├── Fly.io: app 名称、区域、CLI 命令
  ├── Render: service ID、dashboard URL
  ├── Vercel: 项目名、自动部署确认
  ├── Netlify: site ID、构建命令
  ├── GitHub Actions: workflow 文件名、触发条件
  └── Custom: 用户提供命令和 URL
           │
           ▼
Step 4: 写入配置到 CLAUDE.md
  └── ## Deploy Configuration 节
           │
           ▼
Step 5: 验证
  └── 确认配置可读、URL 可达
           │
           ▼
Step 6: 摘要
  └── 展示配置，建议运行 /land-and-deploy 测试
```

---

## 三、支持的平台

| 平台 | 检测信号 | 部署方式 |
|------|---------|---------|
| Fly.io | `fly.toml` | `fly deploy` |
| Render | `render.yaml` | API / Dashboard |
| Vercel | `vercel.json` / `.vercel/` | 自动部署（push 触发） |
| Netlify | `netlify.toml` | 自动部署（push 触发） |
| GitHub Actions | `.github/workflows/*deploy*` | Workflow dispatch |
| Heroku | `Procfile` | `git push heroku` |
| Custom | 用户提供 | 自定义命令 |

---

## 四、与 /land-and-deploy 和 /ship 的关系

```
/setup-deploy（一次性）
  └── 写入 CLAUDE.md 配置
           │
           ▼
/ship（每次 PR）
  └── 创建 PR
           │
           ▼
/land-and-deploy（部署时）
  └── 读取 CLAUDE.md 配置 → 自动合并 → 自动部署 → 自动验证
```

---

## 五、Prompt Engineering 可迁移模式

### 模式 1：一次配置，永久使用
把环境检测结果持久化到项目文件中，避免每次重复检测。

### 模式 2：多平台自适应
同一个 Skill 支持多种平台，通过文件特征自动路由。
