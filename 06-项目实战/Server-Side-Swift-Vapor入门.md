# Server-Side Swift：Vapor 入门

> 🎯 **本章目标**：了解 Server-Side Swift 生态，掌握 Vapor 框架的基础使用，能够搭建简单的后端 API 服务，学会 iOS App 与自建后端的对接，理解独立开发者自建后端的场景与成本。

---

## 1. 为什么 iOS 开发者需要了解后端

### 1.1 独立开发者的痛点

作为 iOS 独立开发者，你在开发 App 时几乎不可避免地需要后端支持。常见的需求包括：

- **用户系统**：注册、登录、个人资料管理
- **数据同步**：多设备数据同步、云端备份
- **内容管理**：App 内的动态内容、配置下发
- **支付验证**：服务端收据验证、订阅状态管理
- **推送通知**：远程推送需要服务端发起

面对这些需求，独立开发者通常有三种选择：

| 方案 | 优点 | 缺点 |
|------|------|------|
| BaaS（如 Firebase、Supabase） | 快速上手、免运维 | 免费额度有限、国内访问不稳定、数据不在自己手里 |
| 第三方 API 服务 | 专业能力强 | 费用高、依赖第三方稳定性 |
| 自建后端 | 完全自主、灵活可控 | 需要后端知识、需要运维 |

> 💡 **提示**：BaaS 在原型阶段非常高效，但当你的 App 用户增长后，免费额度往往不够用，付费方案可能比自建后端更贵。以 Firebase 为例，Firestore 每月免费额度仅 5 万次读和 2 万次写，超过后按用量计费，一个小规模应用月费可能达到 $50-100。

### 1.2 Server-Side Swift 的优势

如果你已经熟悉 Swift，选择 Server-Side Swift 框架来构建后端有独特优势：

- **同一语言**：前后端都用 Swift，无需切换语言思维，降低学习成本
- **代码共享**：Model、验证逻辑、网络请求的请求/响应结构可以在 iOS 端和服务端共享
- **类型安全**：Swift 的强类型系统在服务端同样发挥作用，编译期捕获错误
- **性能优秀**：Swift 编译为原生代码，性能远超 Node.js、Python 等解释型语言
- **Xcode 开发**：可以在熟悉的 Xcode 中调试后端代码

> 💡 生活类比：Server-Side Swift 就像一位会双语的厨师——前端和后端是两个厨房，但你只需要一套"菜谱"（Swift），不用在两个厨房之间翻译菜谱。

### 1.3 Vapor vs 其他后端框架

| 维度 | Vapor (Swift) | Express (Node.js) | Flask (Python) | Gin (Go) |
|------|---------------|-------------------|----------------|----------|
| 语言 | Swift | JavaScript | Python | Go |
| 性能 | 高 | 中 | 低 | 高 |
| 学习曲线（对 iOS 开发者） | 低 | 中 | 中 | 高 |
| 生态成熟度 | 中 | 高 | 高 | 中 |
| 异步模型 | async/await | 回调/Promise | 协程 | goroutine |
| 部署便利性 | 中 | 高 | 高 | 高 |
| 类型安全 | 强 | 弱 | 弱 | 强 |
| 社区规模 | 中 | 大 | 大 | 中 |
| 适合场景 | iOS 全栈 | 快速原型 | 数据处理 | 高并发服务 |

对于 iOS 开发者来说，Vapor 的最大吸引力在于**语言一致性**——你可以用同一套 Swift 知识栈覆盖前后端开发。

### 1.4 适合自建后端的场景

并非所有 App 都需要自建后端。以下场景特别适合：

| 场景 | 说明 | 示例 |
|------|------|------|
| 服务端收据验证 | Apple 推荐收据验证在服务端完成 | 订阅型 App |
| 数据聚合 | 需要聚合多个数据源 | 天气 App、新闻聚合 |
| AI 能力集成 | 调用大模型 API 并做中间处理 | AI 对话 App |
| 用户内容平台 | UGC 内容存储与分发 | 社区、论坛 |
| 自定义业务逻辑 | 复杂的业务规则不适合 BaaS | 金融、工具类 App |

> ⚠️ **警告**：如果你的 App 只需要简单的数据存储和用户认证，BaaS 可能是更务实的选择。不要为了技术偏好而增加不必要的复杂度。

---

## 2. Vapor 环境搭建

### 2.1 Swift 环境确认

Vapor 4 需要 Swift 5.9 及以上版本。确认你的环境：

```bash
swift --version
```

如果你已经安装了 Xcode 15+，Swift 工具链已经包含在内。macOS 上无需额外安装。

### 2.2 Vapor Toolbox 安装

Vapor Toolbox 是 Vapor 的命令行工具，用于快速创建项目和管理：

```bash
brew install vapor
```

安装完成后验证：

```bash
vapor --help
```

> 💡 **提示**：如果 `brew install vapor` 找不到 formula，可以先执行 `brew tap vapor/tap`，然后再 `brew install vapor/tap/vapor`。

### 2.3 创建第一个 Vapor 项目

使用 Toolbox 创建项目：

```bash
vapor new MyAPI
cd MyAPI
```

创建过程中，Toolbox 会询问是否集成 Fluent（数据库 ORM）、Leaf（模板引擎）等。对于 API 项目，建议选择：

- Fluent：**Yes**
- Fluent database driver：**PostgreSQL**（生产环境推荐）
- Leaf：**No**（我们做 API，不需要模板）

然后打开 Xcode 项目：

```bash
open Package.swift
```

### 2.4 项目结构解析

```
MyAPI/
├── Package.swift          # SPM 依赖配置
├── Sources/
│   └── App/
│       ├── configure.swift    # 应用配置（数据库、中间件等）
│       ├── routes.swift       # 路由定义
│       ├── entrypoint.swift   # 入口点
│       ├── Controllers/       # 控制器目录
│       ├── Models/            # 数据模型目录
│       └── Migrations/        # 数据库迁移目录
├── Tests/                 # 测试目录
└── .env                   # 环境变量
```

关键文件说明：

| 文件 | 作用 |
|------|------|
| `configure.swift` | 注册数据库、中间件、服务等 |
| `routes.swift` | 定义路由与控制器映射 |
| `entrypoint.swift` | 应用启动入口 |
| `Package.swift` | 声明依赖和目标 |

### 2.5 运行与测试

在 Xcode 中选择 "Run" scheme，或使用命令行：

```bash
vapor run
```

默认启动在 `http://localhost:8080`。测试一下：

```bash
curl http://localhost:8080
```

```bash
curl http://localhost:8080/hello
```

> 💡 **提示**：开发时使用 `vapor run` 即可，Vapor 默认会监听文件变化。如果需要自动重载，可以配合 Xcode 的运行方案使用。

---

## 3. 路由与控制器

### 3.1 基本路由定义

Vapor 的路由定义非常直观。打开 `routes.swift`：

```swift
import Vapor

func routes(_ app: Application) throws {
    app.get { req async in
        "It works!"
    }

    app.get("hello") { req async -> String in
        "Hello, world!"
    }

    app.get("greet", ":name") { req async -> String in
        guard let name = req.parameters.get("name") else {
            throw Abort(.badRequest)
        }
        return "Hello, \(name)!"
    }
}
```

### 3.2 RESTful API 设计

RESTful API 遵循资源 + HTTP 动词的约定：

| HTTP 方法 | 路径 | 作用 |
|-----------|------|------|
| GET | /users | 获取用户列表 |
| GET | /users/:id | 获取单个用户 |
| POST | /users | 创建用户 |
| PUT | /users/:id | 更新用户 |
| DELETE | /users/:id | 删除用户 |

在 Vapor 中实现：

```swift
app.get("users") { req async -> [User] in
    try await User.query(on: req.db).all()
}

app.get("users", ":id") { req async -> User in
    guard let id = req.parameters.get("id", as: UUID.self) else {
        throw Abort(.badRequest)
    }
    guard let user = try await User.find(id, on: req.db) else {
        throw Abort(.notFound)
    }
    return user
}

app.post("users") { req async -> User in
    let user = try req.content.decode(User.self)
    try await user.save(on: req.db)
    return user
}

app.put("users", ":id") { req async -> User in
    guard let id = req.parameters.get("id", as: UUID.self) else {
        throw Abort(.badRequest)
    }
    guard let user = try await User.find(id, on: req.db) else {
        throw Abort(.notFound)
    }
    let updated = try req.content.decode(User.self)
    user.name = updated.name
    user.email = updated.email
    try await user.save(on: req.db)
    return user
}

app.delete("users", ":id") { req async -> HTTPStatus in
    guard let id = req.parameters.get("id", as: UUID.self) else {
        throw Abort(.badRequest)
    }
    guard let user = try await User.find(id, on: req.db) else {
        throw Abort(.notFound)
    }
    try await user.delete(on: req.db)
    return .noContent
}
```

### 3.3 控制器组织

当路由增多时，应该将逻辑移到控制器中。创建 `Controllers/UsersController.swift`：

```swift
import Vapor

struct UsersController: RouteCollection {
    func boot(routes: RoutesBuilder) throws {
        let users = routes.grouped("users")
        users.get(use: index)
        users.get(":id", use: show)
        users.post(use: create)
        users.put(":id", use: update)
        users.delete(":id", use: delete)
    }

    func index(req: Request) async throws -> [User] {
        try await User.query(on: req.db).all()
    }

    func show(req: Request) async throws -> User {
        guard let id = req.parameters.get("id", as: UUID.self) else {
            throw Abort(.badRequest)
        }
        guard let user = try await User.find(id, on: req.db) else {
            throw Abort(.notFound)
        }
        return user
    }

    func create(req: Request) async throws -> User {
        let user = try req.content.decode(User.self)
        try await user.save(on: req.db)
        return user
    }

    func update(req: Request) async throws -> User {
        guard let id = req.parameters.get("id", as: UUID.self) else {
            throw Abort(.badRequest)
        }
        guard let user = try await User.find(id, on: req.db) else {
            throw Abort(.notFound)
        }
        let updated = try req.content.decode(User.self)
        user.name = updated.name
        user.email = updated.email
        try await user.save(on: req.db)
        return user
    }

    func delete(req: Request) async throws -> HTTPStatus {
        guard let id = req.parameters.get("id", as: UUID.self) else {
            throw Abort(.badRequest)
        }
        guard let user = try await User.find(id, on: req.db) else {
            throw Abort(.notFound)
        }
        try await user.delete(on: req.db)
        return .noContent
    }
}
```

在 `routes.swift` 中注册控制器：

```swift
func routes(_ app: Application) throws {
    try app.register(collection: UsersController())
}
```

### 3.4 请求参数获取

Vapor 提供了多种方式获取请求参数：

```swift
app.get("search") { req async -> String in
    let keyword = req.query.get(String.self, at: "q")
    let page = req.query.get(Int?.self, at: "page") ?? 1
    return "Searching '\(keyword)', page \(page)"
}

app.post("items") { req async -> Item in
    let item = try req.content.decode(Item.self)
    let authHeader = req.headers.first(name: "Authorization")
    return item
}

app.get("users", ":userId", "posts", ":postId") { req async -> String in
    let userId = req.parameters.get("userId", as: UUID.self)
    let postId = req.parameters.get("postId", as: Int.self)
    return "User \(userId!.uuidString), Post \(postId!)"
}
```

### 3.5 JSON 响应

Vapor 的 Model 默认遵循 `Content` 协议，自动支持 JSON 序列化。你也可以自定义响应结构：

```swift
struct APIResponse<T: Content>: Content {
    let code: Int
    let message: String
    let data: T
}

struct UserDTO: Content {
    let id: UUID
    let name: String
    let email: String
}

app.get("users", ":id") { req async -> APIResponse<UserDTO> in
    guard let id = req.parameters.get("id", as: UUID.self) else {
        throw Abort(.badRequest)
    }
    guard let user = try await User.find(id, on: req.db) else {
        return APIResponse(code: 404, message: "User not found", data: UserDTO(id: UUID(), name: "", email: ""))
    }
    let dto = UserDTO(id: user.id!, name: user.name, email: user.email)
    return APIResponse(code: 200, message: "success", data: dto)
}
```

> 💡 **提示**：iOS 端可以定义与 `APIResponse<T>` 完全相同的 Swift 结构体，利用 Codable 直接解码服务端返回的 JSON，实现前后端类型共享。

---

## 4. 数据库与模型

### 4.1 Fluent ORM 概述

Fluent 是 Vapor 的官方 ORM 框架，提供类型安全的数据库操作。核心概念：

| 概念 | 说明 | 类比 iOS |
|------|------|----------|
| Model | 数据模型 | Core Data 的 NSManagedObject |
| Migration | 数据库结构变更 | Core Data 的 Lightweight Migration |
| Query | 查询构建器 | NSFetchRequest |
| Relation | 模型关系 | Core Data 的 Relationship |

### 4.2 数据库配置

**SQLite（开发环境推荐）**：

在 `Package.swift` 中添加依赖：

```swift
.package(url: "https://github.com/vapor/fluent-sqlite-driver.git", from: "4.0.0")
```

在 `configure.swift` 中配置：

```swift
import FluentSQLiteDriver

app.databases.use(.sqlite(.file("db.sqlite")), as: .sqlite)
```

**PostgreSQL（生产环境推荐）**：

```swift
.package(url: "https://github.com/vapor/fluent-postgres-driver.git", from: "2.0.0")
```

```swift
import FluentPostgresDriver

app.databases.use(
    .postgres(
        hostname: Environment.get("DATABASE_HOST") ?? "localhost",
        port: Environment.get("DATABASE_PORT").flatMap(Int.init) ?? PostgresConfiguration.ianaPortNumber,
        username: Environment.get("DATABASE_USERNAME") ?? "vapor",
        password: Environment.get("DATABASE_PASSWORD") ?? "password",
        database: Environment.get("DATABASE_NAME") ?? "vapor"
    ),
    as: .psql
)
```

> ⚠️ **警告**：生产环境务必使用 PostgreSQL 或 MySQL，SQLite 不支持并发写入，在高并发场景下会出现锁等待甚至数据丢失。

### 4.3 Model 定义

创建 `Models/User.swift`：

```swift
import Fluent
import Vapor

final class User: Model, Content {
    static let schema = "users"

    @ID(key: .id)
    var id: UUID?

    @Field(key: "name")
    var name: String

    @Field(key: "email")
    var email: String

    @Field(key: "password_hash")
    var passwordHash: String

    @Timestamp(key: "created_at", on: .create)
    var createdAt: Date?

    @Timestamp(key: "updated_at", on: .update)
    var updatedAt: Date?

    init() {}

    init(id: UUID? = nil, name: String, email: String, passwordHash: String) {
        self.id = id
        self.name = name
        self.email = email
        self.passwordHash = passwordHash
    }
}
```

常用属性包装器：

| 包装器 | 用途 | 对应数据库 |
|--------|------|------------|
| `@ID` | 主键 | PRIMARY KEY |
| `@Field` | 普通字段 | COLUMN |
| `@OptionalField` | 可选字段 | NULLABLE COLUMN |
| `@Timestamp` | 时间戳 | TIMESTAMP |
| `@Parent` | 外键关系 | FOREIGN KEY |
| `@Children` | 一对多关系 | — |
| `@Siblings` | 多对多关系 | 中间表 |

### 4.4 Migration 迁移

Migration 用于创建和修改数据库表结构。创建 `Migrations/CreateUser.swift`：

```swift
import Fluent

struct CreateUser: Migration {
    func prepare(on database: Database) -> EventLoopFuture<Void> {
        database.schema("users")
            .id()
            .field("name", .string, .required)
            .field("email", .string, .required)
            .field("password_hash", .string, .required)
            .field("created_at", .datetime)
            .field("updated_at", .datetime)
            .unique(on: "email")
            .create()
    }

    func revert(on database: Database) -> EventLoopFuture<Void> {
        database.schema("users").delete()
    }
}
```

在 `configure.swift` 中注册迁移：

```swift
app.migrations.add(CreateUser())
```

Vapor 启动时自动执行未运行的迁移。也可以手动运行：

```bash
vapor run migrate
```

### 4.5 CRUD 操作

```swift
let user = User(name: "张三", email: "zhangsan@example.com", passwordHash: "...")

try await user.save(on: req.db)

let fetched = try await User.find(user.id, on: req.db)

user.name = "李四"
try await user.save(on: req.db)

try await user.delete(on: req.db)

let allUsers = try await User.query(on: req.db).all()

let filtered = try await User.query(on: req.db)
    .filter(\.$name == "张三")
    .sort(\.$createdAt, .descending)
    .range(0..<10)
    .all()
```

### 4.6 关系定义

**一对多关系**：一个用户有多篇文章。

```swift
final class Post: Model, Content {
    static let schema = "posts"

    @ID(key: .id)
    var id: UUID?

    @Field(key: "title")
    var title: String

    @Field(key: "content")
    var content: String

    @Parent(key: "author_id")
    var author: User

    init() {}

    init(id: UUID? = nil, title: String, content: String, authorID: UUID) {
        self.id = id
        self.title = title
        self.content = content
        self.$author.id = authorID
    }
}
```

在 User 模型中添加反向关系：

```swift
@Children(for: \.$author)
var posts: [Post]
```

查询时加载关系：

```swift
let userWithPosts = try await User.query(on: req.db)
    .with(\.$posts)
    .find(id)
```

---

## 5. 认证与安全

### 5.1 JWT 认证实现

Vapor 通过 `jwt` 模块支持 JWT。在 `Package.swift` 中添加：

```swift
.package(url: "https://github.com/vapor/jwt.git", from: "4.0.0")
```

配置 JWT 签名器：

```swift
import JWT

let secret = Environment.get("JWT_SECRET") ?? "your-secret-key"
app.jwt.signers.use(.hs256(key: secret))
```

定义 Token 载荷：

```swift
struct UserToken: JWTPayload, Content {
    let sub: UUIDValueClaim
    let exp: ExpirationClaim
    let iat: IssuedAtClaim

    func verify(using signer: JWTSigner) throws {
        try exp.verifyNotExpired()
    }
}
```

### 5.2 用户注册与登录 API

```swift
struct UsersController: RouteCollection {
    func boot(routes: RoutesBuilder) throws {
        let users = routes.grouped("users")
        users.post("register", use: register)
        users.post("login", use: login)

        let protected = users.grouped(UserToken.authenticator())
        protected.get("me", use: me)
    }

    func register(req: Request) async throws -> User.Public {
        let input = try req.content.decode(RegisterInput.self)
        let existing = try await User.query(on: req.db)
            .filter(\.$email == input.email)
            .first()
        if existing != nil {
            throw Abort(.conflict, reason: "Email already registered")
        }
        let passwordHash = try Bcrypt.hash(input.password)
        let user = User(name: input.name, email: input.email, passwordHash: passwordHash)
        try await user.save(on: req.db)
        return user.publicRepresentation
    }

    func login(req: Request) async throws -> TokenResponse {
        let input = try req.content.decode(LoginInput.self)
        let user = try await User.query(on: req.db)
            .filter(\.$email == input.email)
            .first()
        guard let user = user else {
            throw Abort(.unauthorized, reason: "Invalid credentials")
        }
        guard try Bcrypt.verify(input.password, created: user.passwordHash) else {
            throw Abort(.unauthorized, reason: "Invalid credentials")
        }
        let token = try UserToken(
            sub: UUIDValueClaim(value: user.id!),
            exp: ExpirationClaim(value: Date().addingTimeInterval(3600)),
            iat: IssuedAtClaim(value: Date())
        )
        let signedToken = try req.jwt.signers.sign(token)
        return TokenResponse(token: signedToken, user: user.publicRepresentation)
    }

    func me(req: Request) async throws -> User.Public {
        let payload = try req.auth.require(UserToken.self)
        let user = try await User.find(payload.sub.value, on: req.db)
        guard let user = user else {
            throw Abort(.notFound)
        }
        return user.publicRepresentation
    }
}
```

辅助结构体：

```swift
struct RegisterInput: Content {
    let name: String
    let email: String
    let password: String
}

struct LoginInput: Content {
    let email: String
    let password: String
}

struct TokenResponse: Content {
    let token: String
    let user: User.Public
}

extension User {
    struct Public: Content {
        let id: UUID
        let name: String
        let email: String
    }

    var publicRepresentation: Public {
        Public(id: id!, name: name, email: email)
    }
}
```

### 5.3 Token 刷新机制

```swift
func refresh(req: Request) async throws -> TokenResponse {
    let payload = try req.auth.require(UserToken.self)
    let user = try await User.find(payload.sub.value, on: req.db)
    guard let user = user else {
        throw Abort(.notFound)
    }
    let newToken = try UserToken(
        sub: UUIDValueClaim(value: user.id!),
        exp: ExpirationClaim(value: Date().addingTimeInterval(3600)),
        iat: IssuedAtClaim(value: Date())
    )
    let signedToken = try req.jwt.signers.sign(newToken)
    return TokenResponse(token: signedToken, user: user.publicRepresentation)
}
```

iOS 端在 Token 即将过期时调用刷新接口：

```swift
class APIClient {
    private var token: String
    private var tokenExpiry: Date

    func refreshTokenIfNeeded() async throws {
        guard tokenExpiry < Date().addingTimeInterval(300) else { return }
        var request = URLRequest(url: URL(string: "https://api.example.com/users/refresh")!)
        request.httpMethod = "POST"
        request.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")
        let (data, _) = try await URLSession.shared.data(for: request)
        let response = try JSONDecoder().decode(TokenResponse.self, from: data)
        self.token = response.token
        self.tokenExpiry = Date().addingTimeInterval(3600)
    }
}
```

> 💡 **提示**：建议在 iOS 端使用 `URLProtocol` 或 `AsyncAwait` 中间件模式，在每次请求前自动检查并刷新 Token，避免手动管理。

### 5.4 CORS 配置

如果你的 iOS App 通过 WebView 或本地开发时需要跨域访问，需要配置 CORS：

```swift
let corsConfiguration = CORSMiddleware.Configuration(
    allowedOrigin: .all,
    allowedMethods: [.GET, .POST, .PUT, .DELETE, .OPTIONS],
    allowedHeaders: [.accept, .authorization, .contentType, .origin]
)
app.middleware.use(CORSMiddleware(configuration: corsConfiguration))
```

### 5.5 请求限流

Vapor 没有内置限流中间件，但可以轻松实现：

```swift
struct RateLimitMiddleware: Middleware {
    let limit: Int
    let window: TimeInterval

    func respond(to request: Request, chainingTo next: Responder) -> EventLoopFuture<Response> {
        let key = request.remoteAddress?.description ?? "unknown"
        let storage = request.application.storage
        let now = Date()

        if storage[RateLimitKey.self] == nil {
            storage[RateLimitKey.self] = [String: (count: Int, resetAt: Date)]()
        }

        var limits = storage[RateLimitKey.self]!
        if let entry = limits[key], now < entry.resetAt {
            if entry.count >= limit {
                return request.eventLoop.makeFailedFuture(Abort(.tooManyRequests))
            }
            limits[key] = (count: entry.count + 1, resetAt: entry.resetAt)
        } else {
            limits[key] = (count: 1, resetAt: now + window)
        }
        storage[RateLimitKey.self] = limits
        return next.respond(to: request)
    }
}

struct RateLimitKey: StorageKey {
    typealias Value = [String: (count: Int, resetAt: Date)]
}
```

> ⚠️ **警告**：上述限流实现基于内存存储，服务重启后计数器会重置。生产环境建议使用 Redis 存储计数器。

---

## 6. 部署与运维

### 6.1 本地开发环境

开发时直接使用 Xcode 运行即可。建议创建 `.env` 文件管理环境变量：

```
DATABASE_HOST=localhost
DATABASE_PORT=5432
DATABASE_USERNAME=vapor
DATABASE_PASSWORD=password
DATABASE_NAME=vapor_dev
JWT_SECRET=your-dev-secret-key
```

生产环境使用 `.env.production`：

```
DATABASE_HOST=your-prod-host
DATABASE_USERNAME=prod_user
DATABASE_PASSWORD=strong-prod-password
DATABASE_NAME=vapor_prod
JWT_SECRET=very-strong-production-secret
```

> ⚠️ **警告**：`.env` 文件包含敏感信息，务必添加到 `.gitignore`，不要提交到代码仓库。

### 6.2 Docker 容器化部署

创建 `Dockerfile`：

```dockerfile
FROM swift:5.9-jammy as build
WORKDIR /build
COPY Package.swift Package.resolved ./
COPY Sources ./Sources
RUN swift build -c release

FROM swift:5.9-jammy-slim
WORKDIR /app
COPY --from=build /build/.build/release/MyAPI ./MyAPI
COPY .env.production .env
EXPOSE 8080
ENTRYPOINT ["./MyAPI"]
CMD ["serve", "--env", "production", "--hostname", "0.0.0.0", "--port", "8080"]
```

创建 `docker-compose.yml`（包含 PostgreSQL）：

```yaml
version: '3.8'
services:
  app:
    build: .
    ports:
      - "8080:8080"
    depends_on:
      - db
    environment:
      - DATABASE_HOST=db
      - DATABASE_PORT=5432
      - DATABASE_USERNAME=vapor
      - DATABASE_PASSWORD=password
      - DATABASE_NAME=vapor_prod

  db:
    image: postgres:15
    environment:
      - POSTGRES_USER=vapor
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=vapor_prod
    volumes:
      - pgdata:/var/lib/postgresql/data
    ports:
      - "5432:5432"

volumes:
  pgdata:
```

启动：

```bash
docker-compose up -d
```

### 6.3 国内云服务选择

| 云服务商 | 轻量服务器起步价 | 优势 | 适合场景 |
|----------|------------------|------|----------|
| 阿里云 | ¥34/月 | 生态完善、文档丰富 | 通用型 |
| 腾讯云 | ¥30/月 | 新人优惠多 | 通用型 |
| 华为云 | ¥35/月 | 政企客户 | 企业应用 |
| Vultr | $5/月 | 海外节点、按小时计费 | 海外用户 |
| Railway | $5/月起 | Docker 一键部署 | 快速上线 |

> 💡 **提示**：对于独立开发者，阿里云或腾讯云的轻量应用服务器（2C2G）足以支撑数千日活用户的 API 服务。新用户通常有首年优惠，最低可到 ¥50/年。

### 6.4 域名与 HTTPS 配置

1. **购买域名**：在阿里云、腾讯云等平台购买 `.com` 或 `.cn` 域名
2. **ICP 备案**：国内服务器必须完成 ICP 备案（约 10-20 个工作日）
3. **DNS 解析**：将域名指向服务器 IP
4. **HTTPS 证书**：使用 Let's Encrypt 免费证书

使用 Caddy 自动配置 HTTPS（推荐）：

```
api.yourdomain.com {
    reverse_proxy localhost:8080
}
```

Caddy 会自动申请和续期 Let's Encrypt 证书，无需手动配置。

使用 Nginx + Certbot：

```nginx
server {
    listen 443 ssl;
    server_name api.yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/api.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.yourdomain.com/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### 6.5 日志与监控

Vapor 内置日志系统：

```swift
req.logger.info("User \(user.id!) logged in")
req.logger.error("Database connection failed: \(error)")
```

配置日志级别：

```swift
app.logger.logLevel = .info
```

推荐监控方案：

| 方案 | 类型 | 费用 | 说明 |
|------|------|------|------|
| Vapor 内置日志 | 日志 | 免费 | 基础文本日志 |
| Apple OSLog | 日志 | 免费 | macOS 原生 |
| Sentry | APM | 免费额度 | 错误追踪+性能监控 |
| Prometheus + Grafana | 监控 | 自建免费 | 全面的指标监控 |

### 6.6 成本估算

独立开发者自建后端的典型月度成本：

| 项目 | 方案 | 月费用 |
|------|------|--------|
| 云服务器（2C2G） | 阿里云轻量 | ¥34-50 |
| PostgreSQL | 自建（同服务器） | ¥0 |
| 域名 | .com | ¥5（均摊） |
| HTTPS 证书 | Let's Encrypt | ¥0 |
| CDN（可选） | 阿里云 CDN | ¥10-30 |
| 监控（可选） | Sentry 免费版 | ¥0 |
| **合计** | | **¥39-85/月** |

> 💡 **提示**：对比 Firebase Blaze 方案，同样 10 万日活用户规模，Firebase 月费可能达到 $100-300，而自建后端仅需 ¥50-100。用户规模越大，自建后端的成本优势越明显。

---

## 小结

本章介绍了 Server-Side Swift 与 Vapor 框架的核心知识，帮助你从 iOS 开发者扩展到全栈开发：

| 知识点 | 核心内容 | 关键要点 |
|--------|----------|----------|
| 为什么需要后端 | BaaS 限制、自建优势 | 用户增长后自建更经济 |
| Server-Side Swift 优势 | 同语言、代码共享、类型安全 | Vapor 是最成熟的 Swift 服务端框架 |
| 环境搭建 | Toolbox、项目结构 | 开发用 SQLite，生产用 PostgreSQL |
| 路由与控制器 | RESTful API、控制器组织 | 用 RouteCollection 组织路由 |
| 数据库与模型 | Fluent ORM、Migration | @属性包装器定义模型字段 |
| 关系定义 | @Parent、@Children、@Siblings | 用 `.with()` 预加载关系 |
| JWT 认证 | 注册、登录、Token 刷新 | Bcrypt 哈希密码、JWT 签发 Token |
| CORS 与限流 | 中间件模式 | 生产环境限流用 Redis |
| Docker 部署 | Dockerfile、docker-compose | 容器化部署简化运维 |
| 云服务选择 | 阿里云、腾讯云等 | 轻量服务器起步 ¥30-50/月 |
| HTTPS 配置 | Caddy/Nginx + Let's Encrypt | Caddy 自动管理证书 |
| 成本估算 | 月费 ¥39-85 | 规模越大自建越划算 |

Vapor 让 iOS 开发者用熟悉的 Swift 语言构建后端服务，降低了全栈开发的门槛。对于独立开发者来说，掌握 Vapor 意味着你可以不依赖第三方 BaaS，完全掌控自己的后端服务，同时享受 Swift 生态的类型安全和代码共享优势。

← [免费部署平台实战](../07-上架准备/免费部署平台实战.md) | [第三方分析SDK集成](./第三方分析SDK集成.md) →