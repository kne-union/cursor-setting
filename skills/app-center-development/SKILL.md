---
name: app-center-development
description: >-
  Standard workflow and conventions for apps hosted in the developer-document
  App Center (@kne/fastify-app-manager): template choice, /app/{name} mount
  path, server/env.schema.json, DB_TABLE_PREFIX, packApp, MCP deploy, plus the
  frontend rule that BizUnit Table / Modal / FormInfo / InfoPage replace antd
  Table / Modal / Form / Descriptions. Use when creating, developing, packing
  or deploying an 应用中心 app (fullstack-biz-app / fastify-app), or writing
  list, form, modal and detail pages in such an app.
---

# 应用中心应用开发

应用中心 = developer-document 内的 `@kne/fastify-app-manager`。应用以 zip 版本上传，由宿主用 PM2 托管，挂在 `/app/{name}` 下（也可绑定域名）。

规范细节以提示词为准，本 skill 只给流程与硬约束：

| 主题 | 提示词（`prompts/` 下，`npx @kne/npm-tools deployPrompts <type>` 部署） |
|---|---|
| 打包、env schema、MCP 部署 | `prompts-fastify-libs/应用中心部署规范.md` |
| 业务模块结构 | `prompts-remote-components/BizUnit使用指南.md`、`prompts-projects/创建业务模块.md` |
| 表单 | `prompts-remote-components/FormInfo使用指南.md` |
| 前端组件选型（BizUnit 优先于 antd） | `prompts-projects/前端组件使用规范.md` |

项目里已有 `prompts/` 时先读对应文件；组件 API 按 `kne-document-search` 走 MCP `search_document_index`，不凭记忆写。

## 1. 创建应用

```bash
npx @kne/npm-tools init <project-name>
```

按提示选模板：

| 模板 | 用途 |
|---|---|
| `@kne-template/fullstack-biz-app` | 全栈 BizUnit 业务应用（可选 Tenant / 客户端门户），首选 |
| `@kne-template/fastify-app` | 以后端为主的业务应用 |

生成后 `npm run init`（装 server 依赖、husky、部署 prompts），`npm start` 同时起前端与 server。

## 2. 开发约束

### 挂载路径

应用运行在 `/app/{name}` 下，宿主注入 `window.runtimePublicUrl`。

- 路由 basename、页面内跳转、静态资源统一用模板 `src/commons/publicUrl.js` 的 `getRouterBasename` / `withPublicUrl`
- 不要手写 `/app/xxx`，也不要用 `window.PUBLIC_URL` 当挂载前缀（它可能是 CDN 地址）
- 接口走相对 `/api/...`，网关会改写文本响应里的 `"/api/`、`"/static/`、`"/account/` 前缀，以及 `Location` 与 Cookie `Path`

### 环境变量

- server 读取的**所有**变量都在 `server/env.schema.json` 声明，`server.js` 用 `require('./env.schema.json')`，不内联 schema
- 每项写 `description`；能默认的写 `default`，`required` 只放缺了就不能跑的；密钥加 `"writeOnly": true`，不写真实默认值
- `PORT`、`DB_*`、`DB_TABLE_PREFIX`、`AUTH_MODE`、`OIDC_*`（以及未配置时的 `ORIGIN`）由应用中心注入，schema 里只给本地默认值

### 数据库

- 共享库下宿主注入 `DB_TABLE_PREFIX=t_{appName}_`；`@kne/fastify-sequelize@>=4.0.4` 会强制所有模型表名带该前缀
- 表名不要写死，也不要指望 `addModels({ prefix })` 绕开前缀
- 结构变更写成 `server/sql/*.sql` 迁移脚本（文件名只含 `[A-Za-z0-9_-.]`），部署时可选执行

### 登录

部署到应用中心时以 central 模式接入宿主统一登录（宿主注入 OIDC 配置），应用内不要另做一套登录或要求用户填 OIDC 密钥。

### 服务端与前端代码边界

`server/` 不得 `require` `src/` 里的文件：Docker 镜像与部署包运行时只有 `server/`。前后端共用的逻辑抽成 npm 包，两边各自声明依赖。

### 移动端

按 `responsive-utils-mobile`：SCSS 用 `resp.mobile-container`，JS 用 `useIsMobile`。

## 3. 前端组件规范：BizUnit 优先

写列表、表单、弹窗、详情前先读 `prompts-projects/前端组件使用规范.md`，按其中的选型顺序与对照表写：

1. 完整 CRUD 模块用 `components-admin:BizUnit`
2. 单独的列表 / 表单 / 弹窗 / 详情用 `components-core` 的 `TablePage`（静态表格 `TablePage@Table`）、`FormInfo`（`@useFormModal`）、`Modal@useModal`、`InfoPage`
3. `components-core:Table` 已废弃，新代码不要使用
4. 尽量不用 antd 的 `Table` / `Form` / `Modal` / `Descriptions`；确需退回时在代码旁注释原因

项目 `prompts/` 里没有这份文件时，执行 `npx @kne/npm-tools deployPrompts fastify-project` 刷新（或 MCP `get_prompt` 读 `@kne/prompts-projects`）。

## 4. 版本与打包

1. 改完升根 `package.json` 的 `version`（同一应用内版本号不能重复，`packApp` 默认取它）
2. `npm run build`（前端产物进 `build/`）
3. 打包并上传：

```bash
npx @kne/npm-tools packApp --upload --name <应用 name>
```

`packApp` 会检查包结构、`server/package.json` 无 `file:` / `link:`、`env.schema.json` 格式与 SQL 文件名，并自动排除 `node_modules`、`.env*`、本地 sqlite 等。不要手工打 zip。

## 5. 部署（MCP `user-developer-document`，管理员令牌）

1. `list_apps` / `get_app` 确认应用存在；没有则 `create_app`（`label`、`icon`、`isPublic`、`entries: [{ label, path }]` 决定应用中心展示与 PWA 入口）
2. `upload_app_version { name, version, fileId }`（`fileId` 来自上一步 `packApp --upload`）
3. `get_app_env { name, versionId }` 看缺哪些变量 → `save_app_env` 补齐（密钥保持 `********` 表示不改）
4. 有 SQL 时 `list_app_migrations` 确认待执行脚本
5. `deploy_app { name, version }`，等各阶段结果
6. 失败排查：`get_app_logs { name, stream: "err" }`、`list_app_tables` / `query_app_sql`（只读）、`get_app_load`

代码提交仍按仓库所属走 `development-workflow`（kne-union）或 `business-development-workflow`；部署前过一遍其中的本地调试泄漏门禁。

## 禁止

- `server/` require `src/`，或读取未在 `env.schema.json` 声明的变量
- 手写 `/app/{name}` 前缀，或用 `window.PUBLIC_URL` 拼页面路由
- 写死表名、绕开 `DB_TABLE_PREFIX`
- 列表 / 表单 / 弹窗 / 详情用 antd `Table` / `Form` / `Modal` / `Descriptions` 自拼（见 `前端组件使用规范.md` 的例外条件）
- 跳过 `packApp` 手工打包，或复用已上传过的版本号
- 部署包里带 `.env`、`node_modules`、本地数据库文件
