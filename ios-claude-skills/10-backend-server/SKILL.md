---
name: backend-server
description: 涉及后端搭建、API 开发、数据库设计、服务器部署、Supabase、Vapor、Cloudflare Workers、Firebase、认证、BaaS、Serverless、RLS 策略、Edge Functions、Sign in with Apple 服务端验证的任务
---

# 后端服务开发

## 方案选型

| 方案 | 语言 | 适合场景 | 免费额度 | 学习成本 |
|------|------|---------|---------|---------|
| Supabase | SQL + TypeScript | 快速出产品、CRUD 为主 | 500MB 数据库 + 1GB 存储 | 低 |
| Cloudflare Workers | TypeScript | 轻量 API、全球加速 | 10 万请求/天 | 中 |
| Vapor | Swift | Swift 全栈、共享 Model | 需自备服务器 | 高 |
| Firebase | TypeScript / Dart | Google 生态、实时同步 | Spark 计划免费 | 中 |

**选择原则：**
- 独立开发者首选 **Supabase**（开箱即用，PostgreSQL + RLS 天然适合移动端）
- 追求极致轻量和全球边缘部署选 **Cloudflare Workers**
- 想前后端共享 Swift 代码选 **Vapor**
- 已在 Google 生态内选 **Firebase**

---

## Supabase 规范

### 项目初始化
```bash
npx supabase init
npx supabase login
npx supabase link --project-ref <project-id>
```

### 数据库 & RLS
- **所有业务表必须启用 RLS**，禁止 `public` schema 表无策略暴露
- 策略命名：`{操作}_{表名}_{角色}`，如 `select_profiles_authenticated`
- 禁止在策略中使用 `true`（即允许所有访问），必须明确条件：

```sql
-- ✅ 正确：只允许用户访问自己的数据
CREATE POLICY select_profiles_authenticated ON profiles
  FOR SELECT TO authenticated
  USING (auth.uid() = user_id);

-- ❌ 禁止：允许所有人访问
CREATE POLICY select_profiles ON profiles
  FOR SELECT USING (true);
```

- 视图必须设置 `security_invoker = true`：
```sql
CREATE VIEW public.user_stats WITH (security_invoker = true) AS
  SELECT * FROM stats WHERE user_id = auth.uid();
```

- **禁止暴露 `service_role` key 到客户端**，仅限服务端 / Edge Functions 使用

### Edge Functions
- 函数放在 `supabase/functions/` 目录，每个函数一个子目录
- 命名：`kebab-case`，如 `send-push-notification`
- 必须验证用户身份：

```typescript
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2'

Deno.serve(async (req) => {
  const authHeader = req.headers.get('Authorization')
  if (!authHeader) {
    return new Response(JSON.stringify({ error: 'Unauthorized' }), { status: 401 })
  }

  const supabase = createClient(
    Deno.env.get('SUPABASE_URL') ?? '',
    Deno.env.get('SUPABASE_ANON_KEY') ?? '',
    { global: { headers: { Authorization: authHeader } } }
  )

  const { data: { user } } = await supabase.auth.getUser()
  if (!user) {
    return new Response(JSON.stringify({ error: 'Invalid token' }), { status: 401 })
  }

  // 业务逻辑...
})
```

- 环境变量通过 `supabase secrets set` 管理，禁止硬编码
- CORS 处理：统一在函数入口处理 preflight

### Auth 集成
- iOS 端使用 `supabase-swift` SDK
- Sign in with Apple 流程：
  1. iOS 端获取 `authorizationCode` + `identityToken`
  2. 发送到 Supabase Auth 的 `signInWithIdToken` 接口
  3. Supabase 自动验证 Apple JWT 并创建/关联用户
- Token 刷新：SDK 自动处理，监听 `authStateChange` 更新 UI 状态

### 迁移管理
```bash
npx supabase migration new <name>    # 创建迁移
npx supabase db push                 # 推送到远程
npx supabase db pull                 # 拉取远程变更
```
- 迁移文件禁止手动修改已推送的文件（只能追加新迁移）
- 种子数据放在 `supabase/seed.sql`

---

## Cloudflare Workers 规范

### 项目初始化
```bash
npm create cloudflare@latest my-api -- --framework=hono
```

### Hono 路由模式
```typescript
import { Hono } from 'hono'
import { cors } from 'hono/cors'
import { bearerAuth } from 'hono/bearer-auth'

const app = new Hono()

app.use('*', cors({
  origin: ['*'],
  allowMethods: ['GET', 'POST', 'PUT', 'DELETE'],
  allowHeaders: ['Content-Type', 'Authorization'],
}))

app.use('/api/*', bearerAuth({ token: env.API_TOKEN }))

app.get('/api/user/:id', async (c) => {
  const id = c.req.param('id')
  const db = c.env.DB
  const user = await db.prepare('SELECT * FROM users WHERE id = ?').bind(id).first()
  if (!user) return c.json({ error: 'Not found' }, 404)
  return c.json({ data: user })
})

export default app
```

### D1 数据库
- Schema 放在 `wrangler.toml` 同级 `schema.sql`
- 迁移命令：
```bash
npx wrangler d1 migrations create <db-name> <migration-name>
npx wrangler d1 migrations apply <db-name>
```
- 禁止在 Worker 中拼接 SQL 字符串，必须用参数化查询 `.bind()`

### 部署
```bash
npx wrangler deploy
```
- 环境变量通过 `wrangler secret put <KEY>` 管理
- 多环境：`wrangler.toml` 中定义 `[env.staging]` / `[env.production]`

---

## Vapor 规范

### 项目初始化
```bash
vapor new MyAPI --template=api
cd MyAPI
open Package.swift
```

### 目录结构
```
Sources/
├── App/
│   ├── Controllers/        # 路由控制器
│   ├── Models/             # Fluent ORM 模型
│   ├── Migrations/         # 数据库迁移
│   ├── Middleware/          # 中间件
│   ├── configure.swift      # 服务配置
│   └── routes.swift         # 路由注册
└── Run/
    └── main.swift
```

### 控制器规范
```swift
struct UserController: RouteCollection {
    func boot(routes: RoutesBuilder) throws {
        let users = routes.grouped("api", "v1", "users")
        users.get(use: index)
        users.post(use: create)
        users.group(":userID") { user in
            user.get(use: show)
            user.put(use: update)
            user.delete(use: delete)
        }
    }

    func index(req: Request) async throws -> [User.Public] {
        try await User.query(on: req.db).all().map { $0.toPublic() }
    }
}
```

### Sign in with Apple 服务端验证
```swift
import JWT

func verifyAppleToken(_ token: String) async throws -> AppleIdentityToken {
    let jwtVerifier = JWTVerifier.es256(key: applePublicKey)
    let payload = try JWT<AppleIdentityToken>.verify(token, verifier: jwtVerifier)
    guard payload.issuer == "https://appleid.apple.com",
          payload.audience.contains(bundleId) else {
        throw Abort.unauthorized
    }
    return payload
}
```

### 共享代码（iOS ↔ Server）
- 将 Model / DTO 放在独立 SPM 包中：
```
SharedKit/
├── Package.swift
└── Sources/
    └── SharedKit/
        ├── Models/     # 共享数据模型
        └── DTOs/       # 请求/响应结构体
```
- iOS 和 Vapor 项目都通过 SPM 引用此包

### 部署
- Docker 多阶段构建：
```dockerfile
FROM swift:5.9-jammy AS build
WORKDIR /build
COPY . .
RUN swift build -c release

FROM swift:5.9-jammy-slim
COPY --from=build /build/.build/release/Run /app/Run
CMD ["/app/Run"]
```
- 推荐平台：Fly.io / Railway / 自有 VPS

---

## 认证规范

### Sign in with Apple（全流程）

**iOS 端：**
1. 调用 `ASAuthorizationAppleIDProvider` 获取 `authorizationCode`
2. 将 `authorizationCode`（Base64 编码）+ `identityToken` 发送到后端

**服务端：**
1. 用 `authorizationCode` 向 Apple 换取 `refresh_token`（仅首次）
2. 验证 `identityToken` 的 JWT 签名（Apple 公钥）
3. 校验 `iss == "https://appleid.apple.com"` + `aud == bundleId`
4. 提取 `sub`（Apple 用户唯一 ID）作为关联键
5. 存储用户信息，签发自有 JWT 返回 iOS 端

**Token 刷新：**
- Apple `refresh_token` 永不过期（除非用户撤销）
- 定期调用 Apple Revoke Endpoints 检查用户是否撤销授权

### JWT 规范
- 算法：**ES256**（ECDSA + P-256），禁止 RS256
- Access Token 有效期：**15 分钟**
- Refresh Token 有效期：**30 天**，支持轮换
- Token 载荷最小化，只放 `sub`（用户 ID）+ `exp`，业务数据查库获取

### OAuth2 PKCE（iOS 原生认证推荐）
- 适用于第三方 OAuth（非 Apple）
- 流程：iOS 生成 `code_verifier` + `code_challenge` → 授权码换 Token 时携带 `code_verifier`
- 禁止在客户端存储 `client_secret`

---

## 数据库设计规范

### 命名约定

| 对象 | 规范 | 示例 |
|------|------|------|
| 表名 | 复数，snake_case | `users`, `user_profiles` |
| 字段名 | snake_case | `created_at`, `user_id` |
| 主键 | `id`（UUID 优先） | `id UUID DEFAULT gen_random_uuid()` |
| 外键 | `{关联表单数}_id` | `user_id REFERENCES users(id)` |
| 布尔字段 | `is_` / `has_` 前缀 | `is_active`, `has_verified_email` |
| 时间字段 | `_at` 后缀 | `created_at`, `updated_at`, `deleted_at` |
| 索引 | `idx_{表名}_{字段名}` | `idx_users_email` |

### 必备字段
每张业务表必须包含：
```sql
id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
```
- 软删除用 `deleted_at TIMESTAMPTZ`，禁止物理删除用户数据
- `updated_at` 通过数据库触发器自动更新

### 索引策略
- 外键字段必须加索引
- 高频查询字段加复合索引（遵循最左前缀原则）
- 唯一约束用 `UNIQUE` 约束而非应用层检查
- 禁止在 `TEXT` 大字段上建 B-tree 索引，需要搜索用全文索引或 pg_trgm

---

## iOS ↔ 后端协作规范

### API 响应格式（统一）
```json
{
  "code": 0,
  "message": "success",
  "data": { }
}
```

错误响应：
```json
{
  "code": 40001,
  "message": "参数校验失败",
  "data": {
    "errors": [
      { "field": "email", "message": "邮箱格式不正确" }
    ]
  }
}
```

### 错误码约定

| 范围 | 含义 | 示例 |
|------|------|------|
| 0 | 成功 | `0` |
| 400xx | 客户端参数错误 | `40001` 参数校验失败 |
| 401xx | 认证错误 | `40101` Token 过期, `40102` 无权限 |
| 404xx | 资源不存在 | `40401` 用户不存在 |
| 409xx | 冲突 | `40901` 重复注册 |
| 500xx | 服务端错误 | `50001` 内部异常 |

### API 版本化
- URL 路径版本：`/api/v1/users`（推荐，简单直观）
- 破坏性变更必须升版本，旧版本至少保留 90 天
- 非破坏性变更（新增字段）不升版本

### 分页规范
```json
{
  "data": [...],
  "pagination": {
    "page": 1,
    "per_page": 20,
    "total": 150,
    "total_pages": 8
  }
}
```
- 默认 `per_page = 20`，最大 `100`
- 游标分页（大数据量）：用 `cursor` + `limit` 替代 `page`

---

## 安全规范

### 通用
- **所有 API 必须走 HTTPS**，禁止 HTTP
- CORS 白名单明确指定域名，禁止 `Access-Control-Allow-Origin: *`（开发环境除外）
- 速率限制：认证接口 10 次/分钟，业务接口 60 次/分钟
- 请求体大小限制：默认 1MB，文件上传接口单独配置

### 密码 & Token
- 密码哈希：**Argon2id**（禁止 MD5 / SHA 系列）
- API Key 管理：环境变量注入，禁止写入代码仓库
- Key 轮换：至少每 90 天轮换一次，支持无停机轮换

### SQL 注入防护
- 参数化查询，禁止字符串拼接 SQL
- ORM 默认参数化，原生查询必须 `.bind()`

---

## 部署 & CI/CD

### 环境管理
- 三个环境：`development` → `staging` → `production`
- 环境变量通过各平台 Secret 管理器注入，禁止写入配置文件
- 数据库迁移先在 staging 验证，再推到 production

### Supabase 部署
```bash
npx supabase db push                    # 推送迁移
npx supabase functions deploy <name>    # 部署 Edge Function
```

### Cloudflare Workers 部署
```bash
npx wrangler deploy                     # 部署
npx wrangler d1 migrations apply <db>   # 推送 D1 迁移
```

### Vapor 部署
```bash
docker build -t my-api .
docker push registry/my-api:latest
fly deploy                              # Fly.io
```

### CI/CD 检查清单
- [ ] 所有测试通过
- [ ] 数据库迁移无破坏性变更（或已兼容旧版本）
- [ ] 环境变量已配置
- [ ] API 文档已更新
- [ ] 日志和监控已配置
