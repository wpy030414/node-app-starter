# node-monorepo — pnpm 全栈骨架

> 本文件是 `node-app-starter` 的 **node-monorepo** 路线。通用流程（目录检查、依赖安装、文档骨架、验收纪律）见 `SKILL.md`；本文件给出该路线的完整模板与验收清单。
>
> 构建、检查、测试统一由 **vp（Vite+）** 驱动：web 用 `vp build`，server 用 `vp pack`。不在 `package.json` 里声明 `packageManager`，由 vp 依据 lockfile 自动检测。

## 概览

生成一个可立即开发的全栈 monorepo 骨架：

```
<project>/
├── package.json                     # root：pnpm workspace scripts（dev/build/test）
├── pnpm-workspace.yaml              # packages: ['apps/*', 'packages/*'] + allowBuilds
├── vite.config.ts                   # vp 统一配置：fmt / lint（workspace 级）
├── tsconfig.base.json               # 共享 TS 编译基准
├── tsconfig.json                    # 根引用（files: []）
├── .gitignore
├── .env.example
├── README.md
├── AGENTS.md
├── apps/
│   ├── server/
│   │   ├── package.json             # @<name>/server
│   │   ├── tsconfig.json
│   │   ├── vite.config.ts           # vp pack（tsdown）配置，内联 shared
│   │   └── src/
│   │       └── index.ts             # Hono 入口（API + 托管 client/ 静态文件）
│   └── web/
│       ├── package.json             # @<name>/web
│       ├── tsconfig.json
│       ├── vite.config.ts           # vp build 配置：代理 /api，outDir → server/dist/client
│       ├── index.html
│       ├── postcss.config.js
│       ├── tailwind.config.js
│       └── src/
│           ├── main.tsx             # React 入口
│           ├── App.tsx
│           ├── components/ui/       # shadcn/ui 基础组件
│           ├── hooks/
│           ├── lib/
│           └── styles/globals.css   # Tailwind + shadcn/ui CSS 变量
├── packages/
│   └── shared/
│       ├── package.json             # @<name>/shared（exports → ./src/*.ts，零构建）
│       ├── tsconfig.json
│       └── src/
│           ├── types.ts
│           └── constants.ts
└── docs/
    ├── PRD.md
    ├── ARCHITECTURE.md
    ├── DECISIONS.md
    └── specs/
```

## 架构决策（路线内置约定）

| # | 决策 | 理由 |
|---|------|------|
| A1 | pnpm workspace，无外部编排工具 | 原生 `workspace:*` 协议足够，零额外依赖 |
| A2 | `@scope/shared` 用 TS 源码直引（`exports` → `./src/*.ts`） | 绕过 shared 包构建，vp pack 内联进 server bundle |
| A3 | vp pack（tsdown）构建 server（ESM），`deps.alwaysBundle` 内联 shared | 产物自包含，部署只需 `node dist/index.js` |
| A4 | vp build（Vite）前端 outDir → `apps/server/dist/client/` | 单服务进程托管一切；生产模式 Hono 直出静态文件 |
| A5 | 开发模式 Vite proxy `/api` → Hono | 前后端独立热重载，无需 CORS |
| A6 | root `package.json` 用 `concurrently` + `wait-on` 编排并行启动 | 一键 `pnpm dev` 同时拉起前后端 |
| A7 | **默认前端方案**：React + Vite + Tailwind 3.4 + shadcn/ui（Radix + CVA + CSS 变量） | 可访问、可主题化；用户已指定其它前端栈时以其为准，只保留 A4/A5 的构建与代理约束 |
| A8 | SSE 作为实时推送方案 | 标准 HTTP 协议，天然兼容代理/负载均衡，客户端自动重连 |
| A9 | SQLite（node:sqlite）+ Drizzle ORM 为默认数据库 | 零配置，单文件易备份；可选 PostgreSQL |
| A10 | Hono 4 为后端框架 | 轻量、TypeScript-first、标准 Web API，兼容任何 Node/http 部署 |
| A11 | 构建统一走 vp（Vite+）：web 用 `vp build`，server 用 `vp pack` | 一个 CLI 覆盖 dev/build/check/test，不再拼装 vite + tsup + vitest |
| A12 | 不在 `package.json` 声明 `packageManager` | 由 vp 依据 lockfile 自动检测并代理包管理器 |

> `vp dev` 是 Vite 开发服务器，不负责启动 Node 进程；server 的 dev 仍用 `tsx watch`，构建与检查才交给 vp。

## 使用时机（触发条件）

- 用户说「创建 Node 项目 / 新建 monorepo / 搭全栈骨架 / 初始化仓库」且提到前后端一体。
- 用户想启动一个自托管的 Web 应用项目（API + SPA）。
- 用户需要一个验证了技术可行性的脚手骨架，不想从头试错。

## 执行步骤

### Phase 1：收集参数

向用户确认以下信息（已有明确答案则跳过）：

1. **项目名称**（小写短横线，如 `my-app`）→ 推导出 scope `@my-app`
2. **项目描述**（一句话）
3. **服务端口**（默认 `11408`）
4. **是否需要 Docker / CI 配置**（默认勾选 `.xat/workflows/ci-cd.yml` 的 pm2 部署模板）

### Phase 2：生成文件

按下方模板逐文件生成。占位符 `<NAME>` = 项目名（短横线），`<SCOPE>` = `@<NAME>`，`<PORT>` = 端口号。

#### 根目录文件

**package.json**
```json
{
  "name": "<NAME>",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "concurrently -k -n server,web \"pnpm --filter <SCOPE>/server dev\" \"wait-on tcp:<PORT> && pnpm --filter <SCOPE>/web dev\"",
    "build": "pnpm --filter <SCOPE>/server build && pnpm --filter <SCOPE>/web build",
    "start": "pnpm --filter <SCOPE>/server start",
    "test": "pnpm -r test",
    "check": "pnpm -r check"
  },
  "devDependencies": {
    "concurrently": "^9.2.4",
    "typescript": "^5.9.3",
    "vite-plus": "^1.3.0",
    "wait-on": "^9.1.0"
  }
}
```

**pnpm-workspace.yaml**
```yaml
packages:
  - 'apps/*'
  - 'packages/*'

# pnpm 10+ 默认拦截依赖构建脚本；tsx 依赖 esbuild。
# pnpm 12 用 allowBuilds；pnpm 10/11 用 onlyBuiltDependencies: [esbuild]
allowBuilds:
  esbuild: true
```

**tsconfig.base.json**
```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true
  }
}
```

**tsconfig.json**（根）
```json
{ "extends": "./tsconfig.base.json", "files": [] }
```

**vite.config.ts**（根）— workspace 级 fmt / lint，`vp check` 会从各子包向上读取它：
```ts
import { defineConfig } from 'vite-plus'

export default defineConfig({
  fmt: { semi: false, singleQuote: true },
  lint: {
    // 必须显式排除，否则 vp lint 会扫描 node_modules
    ignorePatterns: ['**/node_modules/**', '**/dist/**'],
    options: { typeAware: true, typeCheck: true },
  },
})
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

**.env.example**
```
# ===== 服务端口 =====
PORT=<PORT>
```

#### apps/server/

**apps/server/package.json**
```json
{
  "name": "<SCOPE>/server",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "cross-env NODE_ENV=development tsx watch src/index.ts",
    "build": "vp pack",
    "start": "cross-env NODE_ENV=production node dist/index.js",
    "test": "vp test --passWithNoTests",
    "check": "vp check"
  },
  "dependencies": {
    "@hono/node-server": "^1.19.17",
    "<SCOPE>/shared": "workspace:*",
    "dotenv": "^16.6.1",
    "drizzle-orm": "1.0.0-rc.4",
    "hono": "^4.13.7"
  },
  "devDependencies": {
    "@types/node": "^22.17.0",
    "cross-env": "^10.1.0",
    "drizzle-kit": "1.0.0-rc.4",
    "tsx": "^4.23.13",
    "typescript": "^5.9.3",
    "vite-plus": "^1.3.0"
  }
}
```

**apps/server/tsconfig.json**
```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": { "types": ["node"] },
  "include": ["src"],
  "exclude": ["node_modules", "dist"]
}
```

**apps/server/vite.config.ts** — `pack` 块就是 tsdown 的配置入口（不要另建 `tsdown.config.ts`）：
```ts
import { defineConfig } from 'vite-plus'

export default defineConfig({
  pack: {
    entry: ['src/index.ts'],
    format: ['esm'],
    outDir: 'dist',
    clean: true,
    dts: false,
    platform: 'node',
    // tsdown 的 ESM 默认产物是 .mjs；这里固定为 .js，与 start 脚本一致
    outExtensions: () => ({ js: '.js' }),
    // 内联 @scope/shared，使 dist/ 自包含
    deps: { alwaysBundle: ['<SCOPE>/shared'] },
  },
})
```

**apps/server/src/index.ts** — 最小 Hono 启动器：
```ts
import { serve } from '@hono/node-server'
import { Hono } from 'hono'
import { readFileSync, existsSync } from 'node:fs'
import { join, dirname, resolve } from 'node:path'
import { fileURLToPath } from 'node:url'
import { config } from 'dotenv'

// 从仓库根加载 .env（开发模式 cwd=repo root；生产模式向上找到 pnpm-workspace.yaml）
const __dirname = dirname(fileURLToPath(import.meta.url))
const repoRoot = resolve(__dirname, '..', '..')
config({ path: join(repoRoot, '.env') })

const PORT = parseInt(process.env.PORT || '<PORT>', 10)

const app = new Hono()

// API 路由
app.get('/api/health', (c) => c.json({ status: 'ok' }))

// --- 生产模式：托管前端静态文件 ---
// dist/ 目录结构预期：dist/index.js + dist/client/（vp build 产物）
const clientDir = join(__dirname, 'client')
if (existsSync(clientDir)) {
  // 静态资源
  app.get('/assets/*', async (c) => {
    const filePath = join(clientDir, c.req.path)
    if (!existsSync(filePath)) return c.notFound()
    const ext = filePath.split('.').pop() || ''
    const mime: Record<string, string> = {
      js: 'application/javascript',
      mjs: 'application/javascript',
      css: 'text/css',
      html: 'text/html',
      svg: 'image/svg+xml',
      png: 'image/png',
      ico: 'image/x-icon',
      json: 'application/json',
      woff2: 'font/woff2',
      woff: 'font/woff',
    }
    return c.body(readFileSync(filePath), 200, {
      'Content-Type': mime[ext] || 'application/octet-stream',
    })
  })

  // SPA fallback：所有非 /api 路径返回 index.html
  app.get('*', (c) => {
    if (c.req.path.startsWith('/api')) return c.notFound()
    const html = readFileSync(join(clientDir, 'index.html'), 'utf-8')
    return c.html(html)
  })
}

console.log(`Server starting on port ${PORT}...`)
serve({ fetch: app.fetch, port: PORT })
```

#### apps/web/

**apps/web/package.json**
```json
{
  "name": "<SCOPE>/web",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "vp dev",
    "build": "vp build",
    "check": "vp check"
  },
  "dependencies": {
    "<SCOPE>/shared": "workspace:*",
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
    "dotenv": "^17.4.2",
    "postcss": "^8.5.28",
    "tailwindcss": "^3.4.19",
    "typescript": "^5.9.3",
    "vite-plus": "^1.3.0"
  }
}
```

**apps/web/tsconfig.json**
```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "jsx": "react-jsx",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "types": ["vite-plus/client", "node"],
    "paths": { "@/*": ["./src/*"] }
  },
  "include": ["src", "vite.config.ts"]
}
```

**apps/web/vite.config.ts**
```ts
import { defineConfig } from 'vite-plus'
import react from '@vitejs/plugin-react'
import path from 'node:path'
import dotenv from 'dotenv'

// 从 repo 根读 .env（向上两级）
dotenv.config({ path: path.resolve(import.meta.dirname, '../../.env') })
const apiPort = process.env.PORT || '<PORT>'

export default defineConfig({
  plugins: [react()],
  build: {
    outDir: '../server/dist/client',
    emptyOutDir: true,
  },
  resolve: {
    alias: { '@': path.resolve(import.meta.dirname, 'src') },
  },
  server: {
    host: '0.0.0.0',
    port: 5173,
    proxy: {
      '/api': { target: `http://localhost:${apiPort}`, changeOrigin: true },
    },
  },
})
```

**apps/web/postcss.config.js**
```js
export default {
  plugins: {
    tailwindcss: {},
    autoprefixer: {},
  },
}
```

**apps/web/tailwind.config.js**（shadcn/ui 标准配置）

使用标准 shadcn/ui Tailwind 配置，含 `darkMode: 'class'`、CSS 变量驱动的语义色板、`--radius` 圆角变量，以及 `@tailwindcss/typography` 插件。

**apps/web/index.html**
```html
<!doctype html>
<html lang="zh-CN">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title><NAME></title>
    <link rel="icon" href="/favicon.svg" />
  </head>
  <body class="min-h-screen bg-background text-foreground antialiased">
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

**apps/web/src/main.tsx**
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

**apps/web/src/App.tsx** — 最小占位：
```tsx
export function App() {
  return (
    <div className="flex min-h-screen items-center justify-center">
      <h1 className="text-2xl font-bold"><NAME> — Ready</h1>
    </div>
  )
}
```

**apps/web/src/styles/globals.css** — shadcn/ui 标准 CSS 变量（`@layer base` 含主题变量 + `@tailwind base/components/utilities`）。内容参照 shadcn/ui 官方 starter。

**apps/web/src/lib/utils.ts** — `cn()` 工具函数（`clsx` + `tailwind-merge`）。

#### packages/shared/

**packages/shared/package.json**
```json
{
  "name": "<SCOPE>/shared",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "exports": {
    "./types": "./src/types.ts",
    "./constants": "./src/constants.ts"
  },
  "scripts": {
    "check": "vp check"
  },
  "devDependencies": {
    "typescript": "^5.9.3",
    "vite-plus": "^1.3.0"
  }
}
```

**packages/shared/tsconfig.json**
```json
{ "extends": "../../tsconfig.base.json", "include": ["src"] }
```

**packages/shared/src/types.ts** — 项目基础类型占位。

**packages/shared/src/constants.ts** — 项目常量占位。

#### 文档骨架

使用 `scaffold-docs` 技能生成标准五件套（README / AGENTS / PRD / ARCHITECTURE / DECISIONS），内容基于本路线已陈述的架构决策填充，而非空模板。specs/ 目录创建 `.gitkeep` 等待具体模块。

#### 可选：CI/CD 配置

若用户需要，创建 `.xat/workflows/ci-cd.yml`（pm2 部署模板）：
```yaml
steps:
  - name: Build
    run: |
      avail_kb=$(awk '/MemAvailable/ {print $2}' /proc/meminfo)
      if [ "${avail_kb:-0}" -lt 4194304 ]; then
        export NODE_OPTIONS='--max-old-space-size=1024'
      fi
      vp install
      pnpm build
  - name: Run
    run: |
      if pm2 jlist 2>/dev/null | grep -q '"name":"<NAME>"'; then
        pm2 delete <NAME>
      fi
      pm2 start 'pnpm start' --name <NAME>
```

### Phase 3：安装依赖

文件生成完毕后：

```bash
# 首次使用先安装 vp（macOS / Linux）
curl -fsSL https://vite.plus | bash
# 由 vp 检测 lockfile 并代理包管理器；不要手写 packageManager 字段
vp install

# 统一格式：oxfmt 会重排 package.json 的依赖顺序、换行等
# （scoped 包名会影响字母序，生成后跑一次可保证 check 全绿）
pnpm --filter <SCOPE>/server exec vp check --fix
pnpm --filter <SCOPE>/web exec vp check --fix
```

pnpm 10+ 默认拦截依赖构建脚本：`pnpm-workspace.yaml` 里的 `allowBuilds` 必须保留，否则 tsx 的 esbuild 装不上。

### Phase 4：验收

- [ ] `pnpm check` 零错误（`vp check`：格式 + lint + 类型）
- [ ] `pnpm dev` 同时启动 Vite (:5173) 和 Hono (:<PORT>)，`/api/health` 返回 `{"status":"ok"}`
- [ ] `pnpm build` 成功，`apps/server/dist/` 包含 `index.js` + `client/`
- [ ] `apps/server/dist/index.js` 自包含：`grep <SCOPE>/shared` 无结果（shared 已内联）
- [ ] `pnpm test` 通过（无测试文件时以 `--passWithNoTests` 退出 0）
- [ ] `pnpm start` 生产模式直接访问 http://localhost:<PORT> 能看到前端页面

## 边界

- **不涉及** 具体业务逻辑（认证、聊天、工具系统等）。这是纯净骨架，业务代码由后续迭代添加。
- **不涉及** 数据库 schema 设计（仅有 Drizzle ORM 依赖声明，无 ddl/schema 文件）。
- **不替代** 产品/技术决策。本路线内建决策是已验证的可靠默认值；若项目有不同约束，以项目约束为准。
- 若目标目录已有内容，不得覆盖；改为叠加缺失文件。