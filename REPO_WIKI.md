# FreeLLMAPI 仓库 Wiki

## 概览

FreeLLMAPI 是一个 OpenAI 兼容的代理平台，目标是将多个免费 LLM 提供商聚合在一个统一的 `/v1/chat/completions` 接口后面。它包含：

- 一个 Express + SQLite 的后端服务
- 一个 React + Vite 的管理仪表盘
- 一套统一 Key 认证和路由逻辑
- Docker Compose 支持
- 多个免费 LLM 提供商的路由与降级机制

## 仓库结构

```
freellmapi/
├── client/           # 前端管理仪表盘
├── server/           # 后端 API 服务与代理
├── shared/           # 共享类型与包
├── docs/             # 文档静态资源
├── docker/           # Docker 相关说明
├── package.json      # 根工作区配置
├── package-lock.json # 锁定依赖
├── Dockerfile        # 单镜像生产构建
├── docker-compose.yml# 推荐部署方式
├── .env.example      # 环境变量模板
├── README.md         # 项目总说明
├── NPM_SECURITY.md   # 当前漏洞状态说明
└── repo-assets/      # 仓库图片与资源
```

## 根目录说明

- `package.json`
  - 定义了 monorepo 工作区：`shared`、`server`、`client`
  - 常用脚本：
    - `npm run dev`：并行启动 server 和 client
    - `npm run test`：执行 server 和 client 的测试
    - `npm run build`：构建 server 和 client

- `.env.example`
  - `ENCRYPTION_KEY`：必须提供，用于加密存储上游 API Key
  - `PORT`：后端服务端口，默认 `3001`
  - `HOST_BIND`：Docker 外网访问绑定地址
  - `PROXY_RATE_LIMIT_RPM`：/v1 接入速率限制
  - `DASHBOARD_ORIGINS`：允许从浏览器访问的额外 origin

- `Dockerfile` / `docker-compose.yml`
  - 项目提供单镜像和 Compose 启动方式
  - 生产环境建议使用 `docker compose up -d`
  - 默认仅绑定 `127.0.0.1`，外网需要 `HOST_BIND=0.0.0.0`

## 子项目说明

### server/

后端应用包，主要职责：

- 提供 OpenAI 兼容的代理接口 `/v1/*`
- 提供管理仪表盘后台 API `/api/*`
- 存储与加密用户配置的 provider keys
- 路由请求到多个 LLM 提供商并实现自动降级
- 统计 Key 健康、速率、令牌消耗、请求分析

主要文件和目录：

- `server/package.json`
  - `devDependencies` 包含 `tsx`、`typescript`、`vitest`
  - `dependencies` 包含 `express`、`drizzle-orm`、`better-sqlite3`、`helmet`、`zod`

- `server/src/index.ts`
  - 启动入口，初始化数据库并监听服务端口
  - 调用 `startHealthChecker()` 进行健康检查任务

- `server/src/app.ts`
  - Express 应用创建器
  - 注册了以下路由：
    - `/api/auth`
    - `/api/keys`
    - `/api/models`
    - `/api/fallback`
    - `/api/analytics`
    - `/api/health`
    - `/api/settings`
    - `/v1` OpenAI 兼容代理
    - `/v1` Responses API shim
  - 使用 `helmet`、`cors`、`express.json`
  - 静态服务 `client/dist`

- `server/src/routes/`
  - `auth.ts`：仪表盘登录、初始化、会话检查
  - `keys.ts`：管理 provider keys 和统一 API key
  - `models.ts`：列出可用模型
  - `fallback.ts`：回退链管理
  - `analytics.ts`：请求统计和指标
  - `health.ts`：Key 健康检查状态
  - `settings.ts`：服务配置与参数
  - `proxy.ts`：OpenAI 兼容代理核心路由
  - `responses.ts`：OpenAI Responses API shim

- `server/src/db/`
  - 数据库入口与 SQLite 初始化
  - 负责统一 API key、provider key、模型、分析数据持久化

- `server/src/services/`
  - 路由和限流服务
  - `router.ts`：主请求路由决策逻辑
  - `ratelimit.ts`：速率与冷却机制
  - `health.ts`：定期探测 Key 状态

- `server/src/lib/`
  - 辅助工具函数，如内容解析、加密、消息格式化等

- `server/src/middleware/`
  - `requireAuth.ts`：仪表盘 API 认证
  - `rateLimit.ts`：/v1 代理速率限制
  - `errorHandler.ts`：统一错误处理

### client/

前端管理仪表盘，基于 React + Vite + Tailwind。主要功能：

- 登录与会话管理
- 添加/管理 provider keys
- 修改回退链顺序
- Playground 测试请求
- Analytics 报表与统计

主要文件：

- `client/package.json`
  - `dependencies` 包含 `react`、`react-dom`、`react-router-dom`、`@tanstack/react-query`、`shadcn`、`tailwindcss`
  - `scripts` 包含 `dev`、`build`、`lint`、`preview`

- `client/src/App.tsx`
  - 前端路由与页面入口
  - 定义了 `Playground`、`Keys`、`Fallback`、`Analytics` 四个页面
  - 包含 Dark Mode 切换与 `AuthGate`

- `client/src/pages/`
  - `AnalyticsPage.tsx`：展示请求统计与模型表现
  - `FallbackPage.tsx`：编辑回退链顺序
  - `KeysPage.tsx`：管理 provider keys 和统一 API key
  - `PlaygroundPage.tsx`：测试请求与反馈

- `client/src/components/`
  - 复用 UI 组件，如 `auth-gate.tsx`、`markdown.tsx`、`page-header.tsx`
  - `ui/` 目录内为通用按钮、输入框、表格、卡片等 UI 组件

- `client/src/lib/`
  - `api.ts`：前端与后端 API 的请求封装
  - `utils.ts`：共享工具函数

### shared/

共享包，用于保存通用类型定义。主要文件：

- `shared/package.json`
- `shared/types.ts`
  - 包含 `ChatMessage`、API 请求/响应类型等

## 关键架构与工作流

### 统一 API Key

项目对外提供一个统一 API key，客户端使用该 key 调用 `/v1/chat/completions`。真正的上游 provider key 全部加密存储在 SQLite 中，避免客户端泄露上游凭证。

### 路由与回退机制

代理核心逻辑位于 `server/src/services/router.ts`，它负责：

- 根据请求模型选择最佳可用模型
- 检查每个 provider/key 是否在限额内
- 处理 `429`、`5xx`、超时等错误并自动降级
- 使用“Sticky sessions”保持会话连贯性

如果请求模型为 `auto`，则由系统自动选择最佳提供商。

### 模型数据与免费额度静态来源

供应商模型列表、能力排序和免费额度信息并非来自实时 API，而是以静态种子数据保存在后端数据库初始化逻辑中：

- 数据源文件：`server/src/db/index.ts`
- 主要函数：`seedModels(db)`
- 存储字段包括：
  - `platform`：提供商标识
  - `model_id`：上游模型 ID
  - `display_name`：展示名称
  - `intelligence_rank`：该模型在其提供商内部的智能排序
  - `speed_rank`：该模型的速度排序值
  - `size_label`：跨提供商的能力层级（例如 `Frontier`、`Large`、`Medium`、`Small`）
  - `rpm_limit`、`rpd_limit`、`tpm_limit`、`tpd_limit`：限制值
  - `monthly_token_budget`：近似免费额度说明
  - `context_window`：上下文窗口大小

这些静态数据在 `seedModels` 时插入数据库，并通过一组 `migrateModels*()` 函数持续更新以保持当前免费 tier 与实际可用性。

### `sort by` 逻辑在哪里

排序逻辑由回退链管理路由实现：`server/src/routes/fallback.ts`。

- `POST /api/fallback/sort/intelligence`
  - 优先按跨提供商能力层级排序：
    - `Frontier` 最优
    - `Large`
    - `Medium`
    - `Small`
    - 其他标签最后
  - 同一能力层级内部再按 `m.intelligence_rank ASC` 排序
  - 该逻辑通过 SQL 常量 `INTELLIGENCE_TIER` 实现
  - 原因：每个提供商内部的 `intelligence_rank` 仅表示该提供商内部的相对顺序，不能直接跨提供商比较

- `POST /api/fallback/sort/speed`
  - 直接按 `m.speed_rank ASC` 排序
  - 越小表示速度越快

- `POST /api/fallback/sort/budget`
  - 使用 `monthly_token_budget` 的字符串级别排序
  - 目前实现为固定 CASE 表达式，按预定义预算区间升序排序

这三个排序预设都在 `server/src/routes/fallback.ts` 中的 `SORT_PRESETS` 对象里声明。

### 额外说明

- `server/src/routes/models.ts` 也会按 `intelligence_rank` 返回 `/v1/models` 列表
- 回退链显示页面会把 `intelligenceRank`、`speedRank`、`monthlyTokenBudget` 一起下发给前端
- `server/src/db/index.ts` 中还包含 `migrateModelsV3Ranks(db)`，它基于 2026 年实际 benchmark 对 `intelligence_rank` 重新排序

### 参考位置

- 模型静态数据：`server/src/db/index.ts`
- 回退排序逻辑：`server/src/routes/fallback.ts`
- `/v1/models` 元数据：`server/src/routes/proxy.ts`
- 智能排序规则说明：`server/src/routes/fallback.ts` 的 `INTELLIGENCE_TIER` 和 `SORT_PRESETS`

### 测试覆盖

`server/src/__tests__/routes/fallback.test.ts` 包含 `sort/intelligence` 和 `sort/speed` 的单元测试，确保排序行为稳定。

### 模型权重与排序准则

系统使用两类静态排序字段：

- `size_label`：跨提供商的能力层级，用于统一比较不同平台的模型
- `intelligence_rank`：提供商内部的模型强度排序值，值越小代表越强
- `speed_rank`：模型速度排序值，值越小代表越快

`size_label` 的层级顺序为：

1. `Frontier`
2. `Large`
3. `Medium`
4. `Small`
5. 其他标签（如未识别标签）

这意味着在 `sort/intelligence` 预设里，优先比较模型的能力层级，而不是直接比较不同提供商的 `intelligence_rank`。

例如，`Frontier` 模型始终优先于 `Large` 或 `Medium` 模型，即使某个 `Large` 模型的 `intelligence_rank` 更小。只有在同一 `size_label` 内，系统才会使用 `m.intelligence_rank ASC` 作为次级排序。

这种设计源于项目对“跨平台能力归一化”的考虑：

- `intelligence_rank` 只在每个提供商内部有意义
- `size_label` 表示全局的能力层级，是跨供应商排序的主轴

`sort/speed` 则仅按 `speed_rank ASC` 排序，适合优先选择响应更快的模型。`speed_rank` 的值越低，代表模型在本项目中的实际速度越快。

`sort/budget` 使用 `monthly_token_budget` 的预定义区间排序，按预算从大到小排列，以便优先使用更大免费额度的模型。

这个章节可以作为 Wiki 的补充说明，帮助后续阅读者理解回退链智能排序背后的原理。

### 管理与仪表盘认证

仪表盘后台 API 均在 `/api/*` 下，并由 `requireAuth` 保护。登录会话使用 scrypt 哈希和 session token 存储。

### OpenAI 兼容性

后端实现了常见的 OpenAI 接口：

- `POST /v1/chat/completions`
- `GET /v1/models`
- `POST /v1/responses`

其中 `responses` 为兼容 `Codex CLI` 的格式 shim。

## 运行与开发

### 本地开发

```bash
git clone https://github.com/tashfeenahmed/freellmapi.git
cd freellmapi
npm install
cp .env.example .env
ENCRYPTION_KEY="$(node -e 'console.log(require("crypto").randomBytes(32).toString("hex"))')"
printf "ENCRYPTION_KEY=%s\nPORT=3001\n" "$ENCRYPTION_KEY" > .env
npm run dev
```

- 启动后，前端仪表盘默认访问 `http://localhost:5173`
- 代理接口默认访问 `http://localhost:3001/v1`

### Docker 启动

```bash
ENCRYPTION_KEY=$(openssl rand -hex 32)
printf "ENCRYPTION_KEY=%s\nPORT=3001\n" "$ENCRYPTION_KEY" > .env
docker compose up -d
```

如果需要外网访问：

```bash
HOST_BIND=0.0.0.0 docker compose up -d
```

注意：仅在受信任网络中使用 `0.0.0.0`。

### 构建生产包

```bash
npm run build
node server/dist/index.js
```

## 主要 API 端点

### 仪表盘后台接口

- `POST /api/auth/setup`：首次初始化仪表盘管理员账号
- `POST /api/auth/login`：登录
- `GET /api/auth/status`：检查登录状态
- `GET /api/keys`：获取 provider keys 列表
- `POST /api/keys`：新增 provider key
- `PUT /api/keys/:id`：更新 provider key
- `DELETE /api/keys/:id`：删除 provider key
- `GET /api/models`：获取可用模型配置
- `GET /api/fallback`：获取当前回退链
- `PUT /api/fallback`：保存回退链顺序
- `GET /api/analytics`：获取请求分析数据
- `GET /api/health`：获取 Keys 健康状态
- `GET /api/settings`：获取当前设置
- `PUT /api/settings`：保存设置

### 代理接口

- `POST /v1/chat/completions`
- `GET /v1/models`
- `POST /v1/responses`

> 这些接口使用独立的统一 API key 认证，与仪表盘登录不同。

## 运行环境与依赖

- Node.js 20+
- npm
- Docker / Docker Compose
- SQLite 存储

## 重要配置与注意点

- `ENCRYPTION_KEY` 是必需的，不存在时后端启动会失败
- `DASHBOARD_ORIGINS` 可用于允许非默认 dev host 访问仪表盘
- `PROXY_RATE_LIMIT_RPM` 用于防止 /v1 代理被滥用
- 前端静态文件在生产模式由后端直接服务

## 已知安全与维护状态

仓库根目录已存在 `NPM_SECURITY.md`，记录当前依赖审计状态与已修复漏洞。当前项目仍需关注 `drizzle-kit` / `esbuild` 链式依赖问题，但核心高危漏洞已经修复。

## 贡献与扩展点

该仓库欢迎以下改进：

- 添加 `embeddings`、`images`、`audio` 等 OpenAI 兼容端点
- 支持更多免费 LLM 提供商
- 改进多租户 / 多用户认证
- 增强 rate limit、token budget 规则
- 优化仪表盘 UI 与 Analytics 报表

## 额外说明

- `client/` 使用 `vite` 与 `@tanstack/react-query`
- `server/` 使用 `express` 与 `drizzle-orm`
- `shared/` 提供跨前后端类型共享
- `docker/README.md` 详细说明了 Docker 部署步骤

---

此 Wiki 适合作为项目快速上手与架构介绍文档。要继续补充更多细节，可在 `REPO_WIKI.md` 中增加「数据模型」、「数据库模式」、「provider 适配器列表」和「路由决策流程」章节。