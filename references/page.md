# node-page — 纯前端 SPA + GitHub Pages

> 本文件是 `node-app-starter` 的 **node-page** 路线。通用流程（目录检查、依赖安装、文档骨架、验收纪律）见 `SKILL.md`；本文件给出该路线的完整模板、部署方式与验收清单。
>
> 构建、检查、测试统一由 **vp（Vite+）** 驱动。不在 `package.json` 里声明 `packageManager`，由 vp 依据 lockfile 自动检测。

## 概览

生成一个可立即开发、可一键部署到 GitHub Pages 的纯前端骨架：

```
<project>/
├── package.json                     # 单包（非 workspace）
├── pnpm-lock.yaml
├── tsconfig.json
├── vite.config.ts                   # vp 配置：base 由 CI 注入，fmt/lint
├── index.html                       # 资源引用使用 %BASE_URL%
├── postcss.config.js
├── tailwind.config.js               # shadcn/ui 标准配置
├── components.json                  # shadcn CLI 配置
├── .gitignore
├── .nvmrc
├── public/
│   ├── favicon.svg
│   └── .nojekyll                    # 原样拷贝到 dist/
├── src/
│   ├── main.tsx
│   ├── App.tsx
│   ├── components/
│   │   └── ui/                      # shadcn/ui 基础组件
│   ├── hooks/
│   ├── lib/
│   │   └── utils.ts                 # cn()
│   ├── data/                        # 静态数据（TS/JSON 常量）
│   └── styles/
│       └── globals.css              # Tailwind + shadcn/ui CSS 变量
├── .github/
│   └── workflows/
│       └── deploy.yml               # 构建 + 发布 Pages
└── README.md / AGENTS.md / docs/    # 由 scaffold-docs 生成
```

## 架构决策（路线内置约定）

| # | 决策 | 理由 |
|---|------|------|
| P1 | 单包，不建 workspace | 没有第二个包时 monorepo 只是纯开销 |
| P2 | vp build（Vite 8 + Rolldown）+ React 19 + TypeScript | 与 node-monorepo 的 `apps/web` 同构，日后升级全栈可整体平移 |
| P3 | **默认 UI 栈**：Tailwind 3.4 + shadcn/ui（Radix + CVA + CSS 变量） | 与全栈路线共享组件与主题；用户已指定其它 UI 方案时以其为准（见「替换默认前端栈」） |
| P4 | GitHub Actions：`upload-pages-artifact` + `deploy-pages` | 官方推荐路径；不再用 `gh-pages` 分支推产物 |
| P5 | `base` 由 CI 注入的 `PAGES_BASE_PATH` 决定，本地恒为 `/` | 不对仓库名或域名做任何推断；自定义域只需改仓库变量 |
| P6 | 路由默认 HashRouter；需要干净 URL 时用 `404.html` 复制法 | Pages 是纯静态托管，没有服务端重写 |
| P7 | `public/.nojekyll` | 防止 Jekyll 忽略 `_` 开头的构建产物 |
| P8 | 数据全静态（`src/data/` 常量、`public/` JSON、localStorage 或外部 API） | 无服务端；构建期抓取失败会让部署变红，必须显式确认 |
| P9 | 环境变量仅 `VITE_*` 且视为公开 | Vite 在构建期内联，任何密钥都会进 bundle |
| P10 | `vp check`（格式 + lint + 类型）进 CI | Pages 部署没有回滚成本，静态错误必须在部署前拦住 |

## 使用时机（触发条件）

- 用户要一个纯前端站点：工具站、展示页、个人主页、小游戏、纯本地数据的应用。
- 用户明确要求 GitHub Pages / 免费托管 / 无服务器部署。
- 用户已有静态数据（JSON、Markdown、常量），不需要写入数据库。

如果用户提到登录、数据库、上传、实时推送——回到 `SKILL.md` 重新选路。

## Phase 1：收集参数

已有明确答案则跳过：

1. **项目名称**（小写短横线，如 `my-site`）→ 同时决定 GitHub 仓库名与 `base`。
2. **站点形态**（决定的是仓库变量 `PAGES_BASE_PATH`，不是代码）：
   - 项目页 `https://<user>.github.io/<repo>/` → 变量**留空**，CI 自动取 `/<repo>/`（**默认**）
   - 用户页 `https://<user>.github.io/` → 变量设为 `/`
   - 自定义域（如 `https://demo.example.com/`）→ 变量设为 `/`
3. **项目描述**（一句话，用于 README / `<title>` / meta description）。
4. **是否需要多页路由**（默认单页；需要则启用 P6 的路由方案）。
5. **是否使用自定义域**（是 → 在仓库 Settings → Pages 配置域名，并把 `PAGES_BASE_PATH` 设为 `/`；不需要在仓库里放 `CNAME` 文件）。

## Phase 2：生成文件

占位符：`<NAME>` = 项目名，`<BASE>` = `/` 或 `/<NAME>/`。

### 根目录文件

**package.json**
```json
{
  "name": "<NAME>",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "vp dev",
    "build": "vp build",
    "preview": "vp preview",
    "check": "vp check"
  },
  "dependencies": {
    "@radix-ui/react-dialog": "^1.1.23",
    "@radix-ui/react-dropdown-menu": "^2.1.24",
    "@radix-ui/react-scroll-area": "^1.2.18",
    "@radix-ui/react-select": "^2.3.7",
    "@radix-ui/react-slot": "^1.3.3",
    "@radix-ui/react-tabs": "^1.1.21",
    "class-variance-authority": "^0.7.1",
    "clsx": "^2.1.1",
    "lucide-react": "^0.475.0",
    "react": "^19.2.8",
    "react-dom": "^19.2.8",
    "tailwind-merge": "^3.6.0"
  },
  "devDependencies": {
    "@tailwindcss/typography": "^0.5.20",
    "@types/node": "^22.17.0",
    "@types/react": "^19.2.18",
    "@types/react-dom": "^19.2.7",
    "@vitejs/plugin-react": "^6.1.1",
    "autoprefixer": "^10.5.5",
    "postcss": "^8.5.28",
    "tailwindcss": "^3.4.19",
    "typescript": "^5.9.3",
    "vite-plus": "^1.3.0"
  }
}
```

**tsconfig.json**
```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "jsx": "react-jsx",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "types": ["vite-plus/client", "node"],
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "paths": { "@/*": ["./src/*"] }
  },
  "include": ["src", "vite.config.ts"]
}
```

**vite.config.ts**
```ts
import { defineConfig } from 'vite-plus'
import react from '@vitejs/plugin-react'
import path from 'node:path'

// 本地开发与预览恒为根路径；
// CI 注入 PAGES_BASE_PATH 决定线上前缀：项目页 /<repo>/，自定义域或用户页 /
// 代码不做任何域名或仓库名推断
const base = process.env.PAGES_BASE_PATH || '/'

export default defineConfig({
  base,
  plugins: [react()],
  resolve: {
    alias: { '@': path.resolve(import.meta.dirname, 'src') },
  },
  server: { host: '0.0.0.0', port: 5173 },
  preview: { host: '0.0.0.0', port: 4173 },
  fmt: { semi: false, singleQuote: true },
  lint: {
    // 必须显式排除，否则 vp lint 会扫描 node_modules
    ignorePatterns: ['node_modules/**', 'dist/**'],
    options: { typeAware: true, typeCheck: true },
  },
})
```

**index.html**
```html
<!doctype html>
<html lang="zh-CN">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title><NAME></title>
    <meta name="description" content="<PROJECT_DESCRIPTION>" />
    <!-- 必须用 %BASE_URL%，否则部署到 /<repo>/ 子路径后图标 404 -->
    <link rel="icon" href="%BASE_URL%favicon.svg" />
  </head>
  <body class="min-h-screen bg-background text-foreground antialiased">
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

**postcss.config.js**
```js
export default {
  plugins: {
    tailwindcss: {},
    autoprefixer: {},
  },
}
```

**tailwind.config.js**

使用标准 shadcn/ui Tailwind 配置：`darkMode: 'class'`、CSS 变量驱动的语义色板、`--radius` 圆角变量、`@tailwindcss/typography` 插件，`content` 为 `['./index.html', './src/**/*.{ts,tsx}']`。

**components.json**（shadcn CLI）
```json
{
  "$schema": "https://ui.shadcn.com/schema.json",
  "style": "new-york",
  "rsc": false,
  "tsx": true,
  "tailwind": {
    "config": "tailwind.config.js",
    "css": "src/styles/globals.css",
    "baseColor": "slate",
    "cssVariables": true,
    "prefix": ""
  },
  "aliases": {
    "components": "@/components",
    "utils": "@/lib/utils",
    "ui": "@/components/ui",
    "lib": "@/lib",
    "hooks": "@/hooks"
  },
  "iconLibrary": "lucide"
}
```

**.gitignore**
```
# 依赖
node_modules/

# 构建产物
dist/

# 日志
*.log

# 缓存
.cache/
.vite/

# 系统文件
.DS_Store
Thumbs.db

# 环境变量
.env
.env.local

# IDE
.vscode/
.idea/
```

**.nvmrc**
```
22
```

**public/favicon.svg** — 极简占位 SVG（`viewBox="0 0 32 32"`，一个圆角方块 + 首字母或简单图形）。

**public/.nojekyll** — 空文件，必须提交到仓库。

### src/

**src/main.tsx**
```tsx
import React from 'react'
import ReactDOM from 'react-dom/client'
import { App } from './App'
import './styles/globals.css'

ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>,
)
```

**src/App.tsx** — 最小占位：
```tsx
export function App() {
  return (
    <main className="flex min-h-screen items-center justify-center">
      <h1 className="text-2xl font-bold"><NAME> — Ready</h1>
    </main>
  )
}
```

**src/lib/utils.ts** — `cn()` 工具函数（`clsx` + `tailwind-merge`），与 shadcn/ui 约定一致。

**src/styles/globals.css** — shadcn/ui 标准 CSS 变量（`@tailwind base/components/utilities` + `@layer base` 主题变量），内容参照 shadcn/ui 官方 starter。

**src/data/** — 静态数据放这里，用 `import` 直接打进 bundle；大批量数据放 `public/data/`，运行时用 `` fetch(`${import.meta.env.BASE_URL}data/xxx.json`) `` 读取。**不要**写死以 `/` 开头的绝对路径。

### 可选：多页路由

默认不装路由。用户需要多页时：

```bash
pnpm add react-router-dom
```

在 `src/main.tsx` 外层包 `<HashRouter>`，路由地址形如 `https://<user>.github.io/<repo>/#/about`。HashRouter 在 Pages 上**永远可用**，刷新深链接不会 404。

若坚持要干净 URL（`/about` 而非 `/#/about`），改用 `BrowserRouter`，并让构建产物带一份 `404.html`：

```json
"build": "tsc --noEmit && vite build && node --input-type=module -e \"import { copyFileSync } from 'node:fs'; copyFileSync('dist/index.html', 'dist/404.html')\""
```

Pages 对未知路径返回 `404.html`，SPA 启动后接管路由。注意这种方式返回的 HTTP 状态码仍是 404（对 SEO 不友好）。需要 SEO 的站点不要走本路线。

### .github/workflows/deploy.yml

`public/.nojekyll` 会被 Vite 原样拷进 `dist/`；工作流里再 `touch` 一次是兜底。`setup-vp` 必须写**精确版本**（不要用已冻结的 `v1` 标签），版本号从 setup-vp 的 releases 页面取。

```yaml
name: Deploy to GitHub Pages

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: false

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6

      # setup-vp 一并处理 Node、包管理器和依赖安装；
      # 不要再写 setup-node / pnpm-action-setup / pnpm install
      - uses: voidzero-dev/setup-vp@v1.21.2
        with:
          node-version: '24'
          cache: true

      - name: Check
        run: vp check

      # base 只在这里注入，代码里不做推断：
      #   项目页：变量未设置 → 默认 /<repo>/
      #   自定义域 / 用户页：把仓库变量 PAGES_BASE_PATH 设为 /
      - name: Build
        run: vp build
        env:
          PAGES_BASE_PATH: ${{ vars.PAGES_BASE_PATH || format('/{0}/', github.event.repository.name) }}

      - name: Disable Jekyll
        run: touch dist/.nojekyll

      - uses: actions/configure-pages@v5
        with:
          enablement: true

      - uses: actions/upload-pages-artifact@v3
        with:
          path: dist

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - id: deployment
        uses: actions/deploy-pages@v4
```

## Phase 3：安装依赖

```bash
# 首次使用先安装 vp（macOS / Linux）；Windows 见官方安装页
curl -fsSL https://vite.plus | bash

# vp 依据 lockfile 自动检测包管理器并代理安装
vp install

# 统一格式（重排 package.json 依赖顺序、换行等），保证 vp check 全绿
vp check --fix
```

网络受限时先设镜像：`pnpm config set registry https://registry.npmmirror.com`（vp 会沿用该配置）。

## Phase 4：首次部署

```bash
# 若目录已在 git 仓库内，跳过 git init
git init -b main
git add -A
git commit -m "chore(repo): 初始化纯前端项目骨架"
gh repo create <NAME> --public --source=. --remote=origin --push
```

`actions/configure-pages` 带 `enablement: true` 会在仓库未开通 Pages 时自动开通（需要 `pages: write` 权限）。若组织策略禁止自动开通，到 **Settings → Pages → Source** 手动选择 **GitHub Actions** 后重跑工作流。

- 项目页地址：`https://<user>.github.io/<NAME>/`（变量留空即可）。
- 用户页 / 自定义域：在 **Settings → Variables → Actions** 新建仓库变量 `PAGES_BASE_PATH`，值填 `/`，再按 GitHub 文档配置 DNS。不需要在仓库里放 `CNAME` 文件，`base` 也不写死在代码里。

## Phase 5：验收

- [ ] `pnpm check` 零错误（`vp check`：格式 + lint + 类型）。
- [ ] `pnpm dev` 后访问 `http://localhost:5173/`（根路径，无前缀）页面正常，favicon 不 404。
- [ ] 无环境变量构建：`pnpm build`，`dist/index.html` 中资源引用为 `/assets/...`。
- [ ] 模拟项目页：`PAGES_BASE_PATH=/<NAME>/ pnpm build`，资源引用变为 `/<NAME>/assets/...`。
- [ ] 模拟自定义域：`PAGES_BASE_PATH=/ pnpm build`，资源引用回到 `/assets/...`。
- [ ] `dist/` 含 `index.html`、`assets/`、`.nojekyll` 与 `favicon.svg`。
- [ ] `pnpm preview` 后访问 `http://localhost:4173/` 正常，刷新不 404。
- [ ] push 后 Actions 的 build / deploy 均绿色，`page_url` 可访问。
- [ ] 深链接直接访问并刷新，页面仍能渲染（HashRouter 或 `404.html`）。
- [ ] 仓库内无密钥；所有 `VITE_*` 变量都确认可公开。

## 常见坑

| 现象 | 原因 | 修正 |
|------|------|------|
| 部署后白屏、assets 全 404 | 自定义域下仍按项目页拼了 `/<repo>/`，或仓库名大小写不一致 | 自定义域/用户页把仓库变量 `PAGES_BASE_PATH` 设为 `/`；项目页保持留空 |
| 图标 / 图片 404 | `index.html` 或代码里写死 `/xxx` | 用 `%BASE_URL%`（HTML）或 `import.meta.env.BASE_URL`（TS） |
| 构建产物里的 `_` 文件丢失 | Jekyll 处理了静态站点 | 提交 `public/.nojekyll` |
| 深链接刷新 404 | Pages 没有服务端重写 | HashRouter，或构建时复制 `404.html` |
| deploy job 报权限错误 | 缺少 `pages: write` / `id-token: write` | 补齐 workflow `permissions` |
| 部署被中断后站点半新半旧 | `cancel-in-progress: true` | 保持 `false`，让发布串行完成 |
| 运行时读取 public JSON 404 | 用了根路径的绝对地址 | `` fetch(`${import.meta.env.BASE_URL}data/x.json`) `` |

## 检查与格式化

`vp check` 一次完成格式化检查（Oxfmt）、lint（Oxlint）和类型检查：

```bash
vp check        # CI 用：只检查，失败即退出非 0
vp check --fix  # 本地用：格式化并自动修复
vp fmt          # 只格式化
vp lint         # 只 lint
```

模板里已经固定 `fmt: { semi: false, singleQuote: true }`，与模板代码风格一致；如果你的团队偏好其它风格，改 `vite.config.ts` 的 `fmt` 块即可。`lint.ignorePatterns` 必须保留，否则 `vp lint` 会扫描 `node_modules`。

## 替换默认前端栈

Tailwind + shadcn/ui 只是**默认推荐**，不是本路线的前提。用户已指定其它方案时，保留以下不变量，其余按用户选择替换：

- 构建与检查仍走 vp：`vp dev` / `vp build` / `vp preview` / `vp check`。
- `vite.config.ts` 仍从 `vite-plus` 导入 `defineConfig`，`base` 仍由 `PAGES_BASE_PATH` 决定。
- `tsconfig` 的 `types` 从 `vite-plus/client` 取 Vite 环境类型。
- Pages 部署约束不变：`.nojekyll`、`base` 前缀、深链接回退。

替换时的具体动作：

| 用户选择 | 需要改什么 |
|---|---|
| 其它 CSS 方案（Tailwind 4 / CSS Modules / vanilla-extract） | 删掉 tailwind/postcss 配置与 `globals.css`，保留 `src/styles/` 入口 |
| 其它组件库（MUI / Ant Design / Ark UI） | 删除 `components/ui/` 与 `lib/utils.ts` 的 shadcn 约定，按库的 provider 结构接入 |
| 其它框架（Vue / Svelte / Solid） | 换插件（`@vitejs/plugin-vue` 等）、去掉 `@types/react`、把 `jsx` 改为框架对应配置；目录结构与部署流程不变 |
| 不使用 React | 同上；`vp` 本身与框架无关 |

不确定时先按默认方案生成，再在 README 里写明替换路径，不要自行混搭两套 UI 方案。

## 边界

- **不涉及** 服务端、数据库、鉴权、上传、定时任务。出现这些需求时改走 `references/monorepo.md`。
- **不涉及** SSR / SSG / SEO 优化。React SPA 在 Pages 上只有客户端渲染；确需 SEO 请明确告知用户并换用静态生成器方案。
- 需要后端时，本路线的 `src/` 可平移到 monorepo 的 `apps/web/`，`base` 改回 `'/'`。
