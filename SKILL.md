---
name: node-app-starter
description: 从零搭建 Node.js Web 应用脚手架，内含两条路线——node-monorepo（pnpm workspace 全栈：Hono + React + SQLite/SSE，自托管）与 node-page（纯前端 SPA：React + Tailwind + shadcn/ui，GitHub Pages + Actions 部署）。构建与检查统一使用 vp（Vite+），不在 package.json 指定包管理器；默认前端方案可按需替换。当用户要求创建/初始化 Node 项目、搭建 monorepo 全栈骨架，或新建纯前端静态站点并部署到 GitHub Pages 时使用。
---

## 概览

本技能是一块**引路牌**，自身不生成代码。先判定路线，再读取对应的 reference，严格按其中的模板与验收清单执行。

| 路线 | Reference | 形态 | 部署 |
|------|-----------|------|------|
| **node-monorepo** | `references/monorepo.md` | pnpm workspace：Hono API + React SPA + shared 包 + SQLite + SSE | 自托管（`vp pack` → `node dist/index.js` / pm2） |
| **node-page** | `references/page.md` | 单包纯前端 SPA，无服务端、无数据库 | GitHub Pages + GitHub Actions（`base` 由 CI 注入） |

## 选路

按下表判定，命中即执行；**一次只生成一条路线**。

| 用户信号 | 路线 |
|----------|------|
| 需要后端 API / 数据库 / 鉴权 / 定时任务 / 文件上传 / SSE | node-monorepo |
| 需要一个进程同时托管前后端，部署到自己的服务器 | node-monorepo |
| 明确提到 monorepo / workspace / 多包 / 前后端分离 | node-monorepo |
| 纯静态展示、工具站、个人主页、纯本地数据（localStorage / 静态 JSON） | node-page |
| 明确提到 GitHub Pages / 免费托管 / 无服务器 / 纯前端 | node-page |

判据倾斜：**默认选 node-page**。只有当用户确实需要服务端能力时才升级到 node-monorepo——不要在需求里塞进「以后可能要用后端」的假设。

判定仍不清楚时，只问一个二选一问题：「这个项目需要后端和数据库吗？」需要 → monorepo；不需要 → page。不要罗列两条路线的区别让用户做题。

## 通用流程（两条路线一致）

1. **收集参数**：路线专属参数见各 reference 的 Phase 1。
2. **检查目录**：确认目标目录现状。已存在的文件一律不覆盖，只补齐缺失项。
3. **生成文件**：按 reference 中的模板逐文件写入，占位符全部替换。
4. **安装依赖**：先安装 vp（`curl -fsSL https://vite.plus | bash`），再执行 `vp install`。vp 依据 lockfile 自动检测并代理包管理器；下载走中国境内镜像（`registry.npmmirror.com`）。
5. **生成文档骨架**：调用 `scaffold-docs` 技能生成五件套（README / AGENTS / PRD / ARCHITECTURE / DECISIONS）。内容用本路线已确定的真实决策填充，不留空模板。
6. **验收**：逐条跑通 reference 末尾的验收清单，失败即修复，不以「应该能跑」结案。

## 共同基线

| 项 | 约定 |
|----|------|
| 包管理器 | **不在 `package.json` 写 `packageManager` 字段**；由 vp 依据 lockfile 自动检测并代理 |
| 构建 / 检查 | 统一走 **vp（Vite+）**：`vp dev` / `vp build` / `vp pack` / `vp check` / `vp test`（Node 服务进程的开发启动仍用 `tsx watch`） |
| 语言 | TypeScript，`strict: true`（monorepo 与纯前端同样适用） |
| Node | ≥ 22；CI 用 24 |
| 前端 | **默认** React 19 + Tailwind 3.4 + shadcn/ui（Radix + CVA + CSS 变量），非强制；用户指定其它栈时按 page.md 的替换表处理 |
| 版本 | 以 reference 中钉住的版本为准；升级前先验证，不要凭记忆改版本号 |
| 命名 | 项目名小写短横线；monorepo 由其推导 scope `@<name>` |

## 边界

- **不涉及** 具体业务逻辑（认证、聊天、支付、工具系统等）。产出是可立即开发的最小骨架。
- **不替代** 产品与技术决策。本技能内建的是已验证的默认值；项目有特殊约束时以项目约束为准。
- **不混搭** 两条路线。需要前端页面托管在全栈项目里，走 monorepo 的 `apps/web`，不要额外叠一个 page 项目。
- node-page 只覆盖客户端渲染的 SPA；需要 SEO / 预渲染的站点（博客、文档站）不在本技能范围内，应直接向用户说明并建议静态生成器方案。

## 文件导航

- `references/monorepo.md` — pnpm 全栈 monorepo 完整模板（Hono + React + SQLite + SSE）。
- `references/page.md` — 纯前端 SPA + GitHub Pages（Actions）完整模板。
