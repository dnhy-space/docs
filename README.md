# dnhy-space

个人空间：日程管理 + Deadline 进度 + 随想 / 日记，按时间线展示。前台是 Next.js 站点，后台是 Vben Admin 5，接口由 ABP Framework（.NET 10）提供。

- 公开站点（时间线、详情、评论、点赞、Deadline 进度）：`web/`（Next.js 16 + Tailwind v4）
- 管理后台（写内容、管日程、管 Deadline、清理评论）：`admin/`（Vue 3 + Vite + Ant Design Vue）
- 接口服务（ABP + PostgreSQL + OpenIddict）：`backend/`

## 技术栈

| 模块 | 技术 |
| --- | --- |
| 后端 | ABP 10.7 / .NET 10 / EF Core 10 / PostgreSQL 17 / OpenIddict 7 |
| 前台 | Next.js 16（App Router、Turbopack）/ Tailwind CSS v4 / next-auth（GitHub）/ react-markdown |
| 后台 | Vben Admin 5.8 / Vue 3 / Vite 8 / Ant Design Vue / md-editor-v3 |
| 部署 | Docker Compose（postgres + migrator + api + web + admin） |

## 目录结构

```
backend/   ABP 分层解决方案（Domain / Application / EF Core / HttpApi.Host / DbMigrator）
web/       Next.js 公开站点
admin/     Vben Admin 5 管理后台（保留 apps/web-antd）
docs/      架构说明
```

## 本地开发

### 1. 数据库

```powershell
docker run -d --name dnhy-postgres -p 5433:5432 `
  -e POSTGRES_PASSWORD=postgres -e POSTGRES_DB=DnhySpace `
  -v dnhy_pgdata:/var/lib/postgresql/data postgres:17-alpine
```

### 2. 后端（http://localhost:44300）

```powershell
$env:DOTNET_ROOT="$env:USERPROFILE\.dotnet"
$env:PATH="$env:DOTNET_ROOT;$env:DOTNET_ROOT\tools;$env:PATH"

cd backend/src/DnhySpace.DbMigrator
dotnet run          # 建表 + 种子数据 + 管理员账号

cd ../DnhySpace.HttpApi.Host
dotnet run
```

Swagger：<http://localhost:44300/swagger>

### 3. 前台（http://localhost:3000）

```powershell
cd web
Copy-Item .env.example .env.local   # 若无
npm install
npm run dev
```

`web/.env.local` 关键项：

```
ABP_API_BASE_URL=http://localhost:44300
AUTH_URL=http://localhost:3000
AUTH_SECRET=<随机 32+ 字符>
VISITOR_JWT_SECRET=<与后端 VisitorToken:Secret 一致>
AUTH_GITHUB_ID=          # 留空则前台隐藏登录入口
AUTH_GITHUB_SECRET=
```

### 4. 后台（http://localhost:5666）

```powershell
cd admin
pnpm install
pnpm dev:antd
```

默认管理员：`admin` / `ChangeMe_123456`（由 DbMigrator 写入，可用 `App:AdminPassword` 或环境变量覆盖）。

## 认证模型

- **管理员**：ABP Identity + OpenIddict 密码模式。后台向 `/connect/token` 换取 `access_token` / `refresh_token`，请求携带 `Authorization: Bearer`。
- **访客**：前台用 GitHub 登录后，由 Next.js 用 `VISITOR_JWT_SECRET` 签发 5 分钟短令牌（HS256，claims：`github_id` / `github_login` / `name` / `picture` / `role`），后端以独立的 `Visitor` JwtBearer 方案校验；只能创建/删除自己的评论与点赞。
- 匿名访客仅可读取公开内容、评论列表与点赞数。

## 主要接口

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| GET | `/api/posts/public` | 公开时间线（分页、可按类型筛选；归档内容不返回） |
| GET | `/api/posts/public/{id}` | 公开详情（私密或已归档内容返回 404） |
| GET | `/api/posts/public/version` | 日记/随想版本号（前台轮询自动刷新） |
| GET/POST/PUT/DELETE | `/api/posts` | 管理员内容管理（可按 `IsArchived` 筛选） |
| POST | `/api/posts/{id}/archive`、`/unarchive` | 归档 / 取消归档（归档后仅后台可见） |
| GET | `/api/comments?PostId=` | 评论列表 |
| POST | `/api/comments` | 访客发表评论 |
| DELETE | `/api/comments/{id}/mine` | 访客删除自己的评论 |
| POST | `/api/likes/toggle` | 访客点赞/取消 |
| GET | `/api/likes/status/{postId}` | 点赞状态 |
| GET/POST/PUT/DELETE | `/api/schedules` | 管理员日程（含完成状态切换） |
| GET | `/api/deadlines/public` | 公开 Deadline 列表（含进度与状态） |
| GET | `/api/deadlines/public/version` | Deadline 版本号（前台轮询自动刷新） |
| GET/POST/PUT/DELETE | `/api/deadlines` | 管理员 Deadline 管理 |
| GET | `/api/dashboard/summary` | 后台统计 |
| POST | `/api/files/upload` | 图片上传（ABP BlobStoring FileSystem） |
| GET | `/api/files/{blobName}` | 图片读取（前台 `/api/files/*` 代理） |

## Docker 部署

```powershell
Copy-Item .env.example .env
# 按需修改 .env：密码、密钥、对外域名（API_PUBLIC_URL / WEB_PUBLIC_URL / ADMIN_PUBLIC_URL）
docker compose up -d --build
```

- 前台：<http://localhost:3000>
- 后台：<http://localhost:8081>
- 接口：<http://localhost:8080>
- `migrator` 是一次性容器，完成迁移后退出；`api` 通过 `service_completed_successfully` 等待它。
- 图片数据在 `dnhy_blobs` 卷，数据库在 `dnhy_pgdata` 卷。
- 后台容器内 nginx 把 `/api-proxy/**` 反代到 `api:8080`，因此后台与接口同源，无需额外 CORS。
- 若在前面再加一层 HTTPS 反向代理，把 `*_PUBLIC_URL` 改成 `https://...`，并保留 `AuthServer__DisableTransportSecurityRequirement=true`（容器内仍是 HTTP）。

## 上线前检查

1. 修改 `ADMIN_PASSWORD`、`AUTH_SECRET`、`VISITOR_JWT_SECRET`、`StringEncryption:DefaultPassPhrase`。
2. 在 GitHub 创建 OAuth App，回调地址填 `{WEB_PUBLIC_URL}/api/auth/callback/github`。
3. 把 `App:CorsOrigins`（compose 中由 `WEB_PUBLIC_URL` / `ADMIN_PUBLIC_URL` 生成）改成真实域名。
4. 迁移完成后可用管理员账号登录后台修改密码。

## 架构说明

更详细的模块划分与数据模型见 [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md)。
