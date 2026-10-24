# dnhy-space 架构说明

## 组成

| 目录 | 技术栈 | 职责 |
| --- | --- | --- |
| `backend/` | ABP 10.x（.NET 10）+ EF Core + PostgreSQL | 领域模型、REST API、Identity/OpenIddict 认证、文件存储 |
| `web/` | Next.js 16（App Router）+ Tailwind | 公开站点：时间线、日记/随想详情、评论与点赞 |
| `admin/` | Vben Admin 5（Vue 3 + Vite + Ant Design Vue） | 管理后台：内容管理、日程管理、评论管理、仪表盘 |

## 认证

```
管理员浏览器 ──(password grant)──> ABP /connect/token ──> Bearer token ──> ABP /api/*（管理员接口）
访客浏览器  ──(GitHub OAuth)──> Next.js Auth.js ──> 服务端签发 5 分钟访客 JWT
                                                     └── Authorization: Bearer (Visitor 方案) ──> ABP /api/comments|/api/likes
```

- 管理员：ABP Identity + OpenIddict 密码模式，后台保存 `access_token` / `refresh_token` 并自动刷新。
- 访客：独立的 `Visitor` JwtBearer 方案，校验 Next.js 以 `VISITOR_JWT_SECRET` 签发的 HS256 令牌（issuer `dnhy-space-web`，audience `dnhy-space-api`，claims: `github_id` / `github_login` / `name` / `picture` / `role`）。
- 业务接口均为显式控制器（`/api/posts`、`/api/comments`、`/api/likes`、`/api/schedules`、`/api/dashboard`、`/api/files`），通过 `[Authorize(AuthenticationSchemes = ...)]` 指定方案，不再依赖 ABP 约定式路由。

## 数据模型

```
Post                     ScheduleItem            Comment                  PostLike
----                     ------------            -------                  --------
Id                       Id                      Id                       Id
Type (Thought|Diary)     Title                   PostId                   PostId
Title                    StartTime               AuthorGithubId           GithubUserId
ContentMarkdown          EndTime                 AuthorLogin              CreationTime
Summary                  AllDay                  AuthorName
Visibility (Private|     Notes                   AuthorAvatarUrl          unique(PostId, GithubUserId)
  Public)                IsCompleted             Content
OccurredAt
```

`ScheduleItem` 仅所有者可访问；`Post` 的私密项对匿名与访客均返回 404。

时间统一以本地时间（`Timing:TimeZone` = `Asia/Shanghai`）存储与展示，数据库启用 `Npgsql.EnableLegacyTimestampBehavior`，因此时间列是 `timestamp without time zone`。

## 数据流

- 公开页面：Next.js Server Component → `web/src/lib/api.ts`（`apiGet` / `apiGetOrNull`，服务端调用 ABP 内网地址）→ 渲染；写操作后调用 `revalidatePath` 刷新缓存。
- 后台页面：Vben → `src/api/request.ts` 中的 `requestClient`（Bearer）→ ABP。开发环境经 Vite 代理 `/api-proxy`，生产环境经 nginx 同源反代，避免 CORS。
- 访客写操作：浏览器 → Next.js Route Handler（`web/src/app/api/**`）→ 校验 Auth.js session → `createVisitorToken` 签发访客 JWT → ABP。

## 目录约定

- 后端按 ABP 分层：`Domain` / `Application.Contracts` / `Application` / `EntityFrameworkCore` / `HttpApi` / `HttpApi.Host` / `DbMigrator`。
- 前台业务代码放 `web/src/`，服务端数据访问集中在 `web/src/lib/api/`。
- 后台业务代码放 `admin/apps/web-antd/src/`，接口封装在 `src/api/`，页面在 `src/views/`。
