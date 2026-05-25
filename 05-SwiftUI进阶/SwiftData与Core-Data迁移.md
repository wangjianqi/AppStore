# SwiftData 与 Core Data 迁移

> 🎯 **本章目标**：理解 SwiftData 与 Core Data 的关系与差异，掌握从 Core Data 迁移到 SwiftData 的完整流程，学会数据模型映射、渐进式迁移策略，能够处理迁移中的常见问题。

---

## 1. 为什么需要迁移

### 1.1 SwiftData 是 Core Data 的现代替代品

自 iOS 3 时代起，Core Data 一直是 Apple 平台数据持久化的核心框架。近十五年来，它经历了 Objective-C 到 Swift 的语言变迁，但 API 风格始终带有浓厚的 Objective-C 痕迹：字符串键值、手动托管对象管理、可视化编辑器……这些在 Swift 时代显得格格不入。

SwiftData 在 iOS 17 横空出世，它不是从零开始的新框架，而是 Core Data 的 Swift 原生封装。底层仍然是那个久经考验的 Core Data 引擎，但上层 API 彻底拥抱了 Swift 的现代特性：宏、属性包装器、值类型、结构化并发。

### 1.2 SwiftData 的核心优势

| 优势 | 说明 |
|------|------|
| 纯 Swift API | 用 `@Model` 宏定义模型，告别 .xcdatamodeld 文件 |
| 类型安全 | `#Predicate` 在编译期检查，不再有字符串拼错的运行时崩溃 |
| 声明式查询 | `@Query` 自动驱动 SwiftUI 视图刷新，无需手动管理 NSFetchedResultsController |
| 代码量骤减 | 宏自动生成样板代码，一个 `@Model` 类顶得上 Core Data 的 Entity + NSManagedObject 子类 + Category |
| 与 SwiftUI 深度集成 | `@Query`、`@Environment(\.modelContext)` 原生支持 |
| Schema 即代码 | 数据模型就是 Swift 代码，可以享受版本控制、代码审查、重构工具的全部好处 |

### 1.3 何时应该迁移，何时应该暂缓

迁移不是必须的。Core Data 仍然被 Apple 官方维护和支持，如果你的项目运行良好，没有迫切理由，完全可以继续使用 Core Data。

**应该迁移的情况**：

- 项目最低部署目标已经是 iOS 17+
- 团队希望减少样板代码、提升开发效率
- 新功能模块希望用 SwiftData 实现
- 项目正在进行大规模重构

**应该暂缓的情况**：

- 项目需要兼容 iOS 16 及以下
- Core Data + CloudKit 同步已经稳定运行
- 项目使用了复杂的 NSMigrationPolicy 自定义迁移逻辑
- 团队对 Core Data 非常熟悉，短期内没有学习新框架的带宽

### 1.4 迁移决策表

| 场景 | 建议 | 原因 |
|------|------|------|
| 新项目，最低 iOS 17+ | 直接用 SwiftData | 无历史包袱，享受现代 API |
| 新项目，需兼容 iOS 16 | 用 Core Data | SwiftData 不支持低版本 |
| 老项目，Core Data 简单模型 | 迁移到 SwiftData | 迁移成本低，收益明显 |
| 老项目，Core Data 复杂模型+自定义迁移 | 暂缓迁移 | 迁移风险高，收益不确定 |
| 老项目，已用 CloudKit 同步 | 暂缓迁移 | SwiftData CloudKit 支持尚不完善 |
| 老项目，正在大规模重构 | 顺势迁移 | 重构本身就是改数据层的好时机 |
| 企业级 App，数据安全要求高 | 评估后决定 | Core Data 更成熟，SwiftData 更简洁 |

> 💡 **提示**：迁移不是非黑即白的选择。你可以采用渐进式策略，在新模块中使用 SwiftData，老模块继续使用 Core Data，逐步过渡。

---

## 2. SwiftData 与 Core Data 对比

### 2.1 数据模型定义方式对比

**Core Data**：使用 .xcdatamodeld 可视化编辑器，数据模型以 XML 格式存储在 `.xcdatamodeld` 包中。

```swift
import CoreData

@objc(User)
public class User: NSManagedObject {
    @NSManaged public var name: String
    @NSManaged public var age: Int16
    @NSManaged public var createdAt: Date
    @NSManaged public var posts: NSSet?
}

extension User {
    @nonobjc public class func fetchRequest() -> NSFetchRequest<User> {
        return NSFetchRequest<User>(entityName: "User")
    }
}
```

**SwiftData**：使用 `@Model` 宏，纯 Swift 代码定义。

```swift
import SwiftData

@Model
class User {
    var name: String
    var age: Int
    var createdAt: Date
    @Relationship(deleteRule: .cascade) var posts: [Post]

    init(name: String, age: Int, createdAt: Date = .now) {
        self.name = name
        self.age = age
        self.createdAt = createdAt
        self.posts = []
    }
}
```

### 2.2 查询方式对比

| 对比项 | Core Data | SwiftData |
|--------|-----------|-----------|
| 基本查询 | `NSFetchRequest` | `@Query` 或 `FetchDescriptor` |
| 过滤条件 | `NSPredicate`（字符串） | `#Predicate`（类型安全） |
| 排序 | `NSSortDescriptor` | `SortDescriptor` |
| SwiftUI 集成 | 需要 `@FetchRequest` 或手动管理 | `@Query` 原生支持 |
| 分页 | `fetchLimit` + `fetchOffset` | `fetchLimit` + `fetchOffset` |

```swift
// Core Data 查询
let request = NSFetchRequest<User>(entityName: "User")
request.predicate = NSPredicate(format: "age > %d", 18)
request.sortDescriptors = [NSSortDescriptor(key: "name", ascending: true)]
let users = try context.fetch(request)

// SwiftData 查询
var descriptor = FetchDescriptor<User>(
    predicate: #Predicate { $0.age > 18 },
    sortBy: [SortDescriptor(\User.name)]
)
let users = try modelContext.fetch(descriptor)
```

### 2.3 关系处理对比

| 对比项 | Core Data | SwiftData |
|--------|-----------|-----------|
| 一对一 | `@NSManaged var profile: Profile?` | `var profile: Profile?` |
| 一对多 | `@NSManaged var posts: NSSet?` | `var posts: [Post]` |
| 多对多 | `@NSManaged var tags: NSSet?` | `var tags: [Tag]` |
| 删除规则 | 在 .xcdatamodeld 中配置 | `@Relationship(deleteRule: .cascade)` |
| 反向关系 | 在编辑器中手动连线 | `@Relationship(inverse: \Post.author)` |
| 类型安全 | ❌ NSSet 需手动转型 | ✅ 原生 Swift 数组 |

### 2.4 迁移机制对比

| 对比项 | Core Data | SwiftData |
|--------|-----------|-----------|
| 轻量迁移 | `NSMigratePersistentStoresAutomaticallyOption` | 自动（默认行为） |
| 映射模型 | .xcmappingmodel 文件 | `VersionedSchema` |
| 自定义迁移 | `NSEntityMigrationPolicy` 子类 | `SchemaMigrationPlan` + `CustomMigration` |
| 版本管理 | .xcdatamodeld 内多版本 | `VersionedSchema` 枚举 |
| 迁移回滚 | 支持 | 有限支持 |

### 2.5 性能对比

在大多数场景下，SwiftData 和 Core Data 的性能几乎相同，因为 SwiftData 底层就是 Core Data。但在某些边界情况下存在差异：

| 场景 | Core Data | SwiftData | 说明 |
|------|-----------|-----------|------|
| 大量数据批量插入 | `NSBatchInsertRequest` | `ModelContext.insert` | Core Data 有专用批量 API |
| 大量数据批量删除 | `NSBatchDeleteRequest` | 无直接等价 API | Core Data 更高效 |
| 批量更新 | `NSBatchUpdateRequest` | 无直接等价 API | Core Data 更高效 |
| 常规 CRUD | 性能相当 | 性能相当 | 底层相同 |
| 内存占用 | 相当 | 相当 | 底层相同 |

> ⚠️ **警告**：如果你的项目大量使用 `NSBatchInsertRequest`、`NSBatchDeleteRequest`、`NSBatchUpdateRequest`，迁移前需要确认 SwiftData 是否有替代方案，否则可能面临性能回退。

### 2.6 底层关系

SwiftData 实际上是 Core Data 的 Swift 封装，这一点至关重要：

```
┌──────────────────────────────────┐
│          SwiftData API           │  ← @Model / @Query / ModelContext
├──────────────────────────────────┤
│          Core Data 引擎          │  ← NSManagedObject / NSManagedObjectContext
├──────────────────────────────────┤
│            SQLite                │  ← 默认持久化存储
└──────────────────────────────────┘
```

`ModelContainer` 内部创建 `NSPersistentContainer`，`ModelContext` 内部持有 `NSManagedObjectContext`。这意味着：

- SwiftData 的存储文件格式与 Core Data 兼容
- 两者可以共享同一个 SQLite 数据库
- SwiftData 的迁移本质上还是 Core Data 的迁移机制

> 💡 **提示**：理解这个底层关系是成功迁移的关键。SwiftData 不是"另一个数据库"，而是"同一数据库的新接口"。

---

## 3. 迁移策略选择

### 3.1 全量迁移

一次性将所有 Core Data 代码替换为 SwiftData 代码。

**适用场景**：

- 数据模型简单，Entity 数量少于 5 个
- 项目规模小，团队人数少于 3 人
- 有充足的测试时间和回滚方案

**优点**：

- 代码库干净，不存在两套 API 共存的情况
- 迁移完成后维护成本低
- 一次性解决所有兼容性问题

**缺点**：

- 风险集中，出问题影响面大
- 需要较长的开发+测试周期
- 回滚困难

### 3.2 渐进式迁移

按模块逐步替换 Core Data 代码为 SwiftData 代码。

**适用场景**：

- 数据模型复杂，Entity 数量多
- 项目规模大，需要控制风险
- 团队资源有限，无法一次性完成

**优点**：

- 风险分散，每个模块独立验证
- 可以边迁移边发布，不影响正常迭代
- 出问题只影响单个模块

**缺点**：

- 过渡期代码库中两套 API 共存，增加复杂度
- 迁移周期长
- 需要处理两套 API 之间的数据共享

### 3.3 双写策略

过渡期同时使用 Core Data 和 SwiftData，新数据同时写入两套存储，最终切换到 SwiftData。

**适用场景**：

- 用户数据非常重要，不能有任何丢失
- 需要灰度验证 SwiftData 的稳定性
- 有充足的存储空间和性能余量

**优点**：

- 安全性最高，可以随时回退
- 可以对比两套数据验证一致性
- 适合用户量大的生产环境

**缺点**：

- 代码复杂度最高
- 双写带来性能开销
- 过渡期维护成本高

### 3.4 策略对比表

| 策略 | 风险 | 复杂度 | 迁移周期 | 适用项目规模 | 数据安全 |
|------|------|--------|----------|-------------|----------|
| 全量迁移 | 高 | 低 | 短 | 小型 | 中 |
| 渐进式迁移 | 中 | 中 | 中 | 中大型 | 高 |
| 双写策略 | 低 | 高 | 长 | 大型 | 最高 |

> 💡 **提示**：对于大多数中小型项目，推荐渐进式迁移。它平衡了风险和复杂度，让你可以在迁移过程中持续交付。

---

## 4. 数据模型映射

### 4.1 Core Data Entity → SwiftData @Model

Core Data 的每个 Entity 对应 SwiftData 的一个 `@Model` 类：

```swift
// Core Data Entity: Item
// 属性: name (String), price (Double), createdAt (Date)

// SwiftData 对应模型
@Model
class Item {
    var name: String
    var price: Double
    var createdAt: Date

    init(name: String, price: Double, createdAt: Date = .now) {
        self.name = name
        self.price = price
        self.createdAt = createdAt
    }
}
```

### 4.2 属性类型映射表

| Core Data 类型 | Swift 类型（SwiftData） | 说明 |
|---------------|------------------------|------|
| String | String | 直接映射 |
| Integer 16 | Int16 | 直接映射 |
| Integer 32 | Int32 | 直接映射 |
| Integer 64 | Int64 | 直接映射 |
| Float | Float | 直接映射 |
| Double | Double | 直接映射 |
| Boolean | Bool | 直接映射 |
| Date | Date | 直接映射 |
| Binary Data | Data | 直接映射 |
| Transformable | 任意 Codable 类型 | 需实现 Codable |
| URI | URL | 直接映射 |
| UUID | UUID | 直接映射 |

> ⚠️ **警告**：Core Data 中 Integer 32 在 NSManagedObject 中映射为 `Int32`，但很多开发者习惯在 Swift 代码中用 `Int` 接收。迁移时务必确认类型一致，否则可能导致数据截断。

### 4.3 关系映射

**一对一关系（ToOne）**：

```swift
// Core Data
@NSManaged var profile: Profile?

// SwiftData
var profile: Profile?
```

**一对多关系（ToMany）**：

```swift
// Core Data
@NSManaged var posts: NSSet?

// SwiftData
@Relationship(deleteRule: .cascade, inverse: \Post.author)
var posts: [Post]
```

**多对多关系**：

```swift
// Core Data
@NSManaged var tags: NSSet?

// SwiftData
var tags: [Tag]
```

关系删除规则映射：

| Core Data 删除规则 | SwiftData 删除规则 | 说明 |
|-------------------|-------------------|------|
| No Action | .noAction | 不做任何处理 |
| Nullify | .nullify | 将反向关系置为 nil |
| Cascade | .cascade | 级联删除 |
| Deny | .deny | 如果有关联对象则拒绝删除 |

### 4.4 NSManagedObjectModel → ModelContainer 配置

Core Data 使用 `NSPersistentContainer` 配置存储，SwiftData 使用 `ModelContainer`：

```swift
// Core Data
let container = NSPersistentContainer(name: "MyApp")
container.loadPersistentStores { _, error in
    if let error = error {
        fatalError("Unresolved error: \(error)")
    }
}

// SwiftData
let container = try ModelContainer(for: Item.self, User.self, Post.self)
```

如果需要指定存储 URL（例如使用 Core Data 的现有数据库文件）：

```swift
let url = FileManager.default.urls(for: .applicationSupportDirectory, in: .userDomainMask).first!
let storeUrl = url.appendingPathComponent("MyApp.sqlite")

let config = ModelConfiguration(url: storeUrl)
let container = try ModelContainer(for: Item.self, configurations: config)
```

### 4.5 VersionedSchema 与 Schema 迁移

SwiftData 使用 `VersionedSchema` 管理数据模型的版本演进：

```swift
enum MyAppSchemaV1: VersionedSchema {
    static var versionIdentifier = Schema.Version(1, 0, 0)

    static var models: [any PersistentModel.Type] {
        [Item.self]
    }

    @Model
    class Item {
        var name: String
        var createdAt: Date

        init(name: String, createdAt: Date = .now) {
            self.name = name
            self.createdAt = createdAt
        }
    }
}

enum MyAppSchemaV2: VersionedSchema {
    static var versionIdentifier = Schema.Version(2, 0, 0)

    static var models: [any PersistentModel.Type] {
        [Item.self]
    }

    @Model
    class Item {
        var name: String
        var price: Double
        var createdAt: Date

        init(name: String, price: Double = 0, createdAt: Date = .now) {
            self.name = name
            self.price = price
            self.createdAt = createdAt
        }
    }
}
```

使用 `SchemaMigrationPlan` 定义迁移计划：

```swift
enum MyAppMigrationPlan: SchemaMigrationPlan {
    static var schemas: [any VersionedSchema.Type] {
        [MyAppSchemaV1.self, MyAppSchemaV2.self]
    }

    static var stages: [MigrationStage] {
        [migrateV1toV2]
    }

    static let migrateV1toV2 = MigrationStage.custom(
        fromVersion: MyAppSchemaV1.self,
        toVersion: MyAppSchemaV2.self,
        willMigrate: { context in
            let items = try context.fetch(FetchDescriptor<MyAppSchemaV1.Item>())
            for item in items {
                let migratedItem = MyAppSchemaV2.Item(
                    name: item.name,
                    price: 0,
                    createdAt: item.createdAt
                )
                context.insert(migratedItem)
                context.delete(item)
            }
            try context.save()
        },
        didMigrate: { context in
        }
    )
}
```

在 `ModelContainer` 中使用迁移计划：

```swift
let container = try ModelContainer(
    for: MyAppSchemaV2.Item.self,
    migrationPlan: MyAppMigrationPlan.self
)
```

> 💡 **提示**：轻量迁移（如添加新属性、重命名属性）SwiftData 会自动处理，不需要手动编写 `MigrationStage`。只有涉及数据转换的复杂迁移才需要自定义迁移逻辑。

---

## 5. 迁移实战步骤

### Step 1：创建 SwiftData 模型

对照 Core Data 的 .xcdatamodeld 文件，逐个创建对应的 `@Model` 类。

假设 Core Data 中有以下 Entity：

- `User`：name (String)、email (String)、createdAt (Date)
- `Post`：title (String)、content (String)、createdAt (Date)、author (关系→User)

创建 SwiftData 模型：

```swift
import SwiftData

@Model
class User {
    var name: String
    var email: String
    var createdAt: Date
    @Relationship(deleteRule: .cascade, inverse: \Post.author)
    var posts: [Post]

    init(name: String, email: String, createdAt: Date = .now) {
        self.name = name
        self.email = email
        self.createdAt = createdAt
        self.posts = []
    }
}

@Model
class Post {
    var title: String
    var content: String
    var createdAt: Date
    var author: User?

    init(title: String, content: String, createdAt: Date = .now, author: User? = nil) {
        self.title = title
        self.content = content
        self.createdAt = createdAt
        self.author = author
    }
}
```

### Step 2：配置 ModelContainer 使用 Core Data 存储文件

关键一步：让 SwiftData 读取 Core Data 已有的 SQLite 文件。

```swift
import SwiftData

@main
struct MyApp: App {
    var container: ModelContainer

    init() {
        let storeURL = Self.coreDataStoreURL
        let config = ModelConfiguration(url: storeURL)
        do {
            container = try ModelContainer(
                for: User.self, Post.self,
                configurations: config
            )
        } catch {
            fatalError("Failed to create ModelContainer: \(error)")
        }
    }

    static var coreDataStoreURL: URL {
        let appSupport = FileManager.default.urls(
            for: .applicationSupportDirectory,
            in: .userDomainMask
        ).first!
        return appSupport.appendingPathComponent("MyApp.sqlite")
    }

    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .modelContainer(container)
    }
}
```

> ⚠️ **警告**：确保 `ModelConfiguration` 的 URL 指向 Core Data 实际使用的 SQLite 文件路径。不同 App 的 Core Data 存储路径可能不同，请先打印 `NSPersistentContainer` 的 `persistentStoreDescriptions` 确认路径。

### Step 3：数据迁移脚本

对于简单的模型映射，SwiftData 可以直接读取 Core Data 的 SQLite 文件。但对于复杂情况（属性重命名、类型变更等），需要编写迁移脚本。

```swift
enum MigrationV1toV2: SchemaMigrationPlan {
    static var schemas: [any VersionedSchema.Type] {
        [SchemaV1.self, SchemaV2.self]
    }

    static var stages: [MigrationStage] {
        [migrateV1toV2]
    }

    static let migrateV1toV2 = MigrationStage.custom(
        fromVersion: SchemaV1.self,
        toVersion: SchemaV2.self,
        willMigrate: { context in
            let oldUsers = try context.fetch(FetchDescriptor<SchemaV1.User>())
            for oldUser in oldUsers {
                let newUser = SchemaV2.User(
                    name: oldUser.name,
                    email: oldUser.email,
                    createdAt: oldUser.createdAt
                )
                for oldPost in oldUser.posts {
                    let newPost = SchemaV2.Post(
                        title: oldPost.title,
                        content: oldPost.content,
                        createdAt: oldPost.createdAt,
                        author: newUser
                    )
                    context.insert(newPost)
                }
                context.insert(newUser)
                context.delete(oldUser)
            }
            try context.save()
        },
        didMigrate: { context in
        }
    )
}

enum SchemaV1: VersionedSchema {
    static var versionIdentifier = Schema.Version(1, 0, 0)
    static var models: [any PersistentModel.Type] { [User.self, Post.self] }

    @Model
    class User {
        var name: String
        var email: String
        var createdAt: Date
        var posts: [Post]
        init(name: String, email: String, createdAt: Date = .now) {
            self.name = name; self.email = email; self.createdAt = createdAt; self.posts = []
        }
    }

    @Model
    class Post {
        var title: String
        var content: String
        var createdAt: Date
        var author: User?
        init(title: String, content: String, createdAt: Date = .now, author: User? = nil) {
            self.title = title; self.content = content; self.createdAt = createdAt; self.author = author
        }
    }
}

enum SchemaV2: VersionedSchema {
    static var versionIdentifier = Schema.Version(2, 0, 0)
    static var models: [any PersistentModel.Type] { [User.self, Post.self] }

    @Model
    class User {
        var name: String
        var email: String
        var avatarURL: String
        var createdAt: Date
        @Relationship(deleteRule: .cascade, inverse: \Post.author)
        var posts: [Post]
        init(name: String, email: String, avatarURL: String = "", createdAt: Date = .now) {
            self.name = name; self.email = email; self.avatarURL = avatarURL
            self.createdAt = createdAt; self.posts = []
        }
    }

    @Model
    class Post {
        var title: String
        var content: String
        var createdAt: Date
        var author: User?
        init(title: String, content: String, createdAt: Date = .now, author: User? = nil) {
            self.title = title; self.content = content; self.createdAt = createdAt; self.author = author
        }
    }
}
```

### Step 4：验证数据完整性

迁移完成后，务必验证数据的完整性和一致性：

```swift
func verifyMigration(context: ModelContext) throws {
    let userCount = try context.fetchCount(FetchDescriptor<User>())
    let postCount = try context.fetchCount(FetchDescriptor<Post>())

    print("Users after migration: \(userCount)")
    print("Posts after migration: \(postCount)")

    let users = try context.fetch(FetchDescriptor<User>())
    for user in users {
        if user.name.isEmpty {
            print("Warning: Found user with empty name, id: \(user.persistentModelID)")
        }
        if user.posts.count > 0 {
            for post in user.posts {
                if post.author !== user {
                    print("Error: Post author mismatch for post: \(post.title)")
                }
            }
        }
    }
}
```

建议在迁移后增加以下检查：

- Entity 数量是否与迁移前一致
- 关系是否完整（双向关系是否正确建立）
- 必填字段是否为空
- 数据类型是否正确转换

### Step 5：切换到 SwiftData

验证通过后，移除 Core Data 相关代码：

1. 删除 .xcdatamodeld 文件
2. 删除 `NSPersistentContainer` 相关代码
3. 将所有 `NSFetchRequest` 替换为 `@Query` 或 `FetchDescriptor`
4. 将所有 `NSManagedObject` 子类替换为 `@Model` 类
5. 更新 SwiftUI 视图中的数据获取方式

```swift
// 迁移前（Core Data）
struct UserListView: View {
    @FetchRequest(
        sortDescriptors: [NSSortDescriptor(keyPath: \User.createdAt, ascending: false)],
        animation: .default
    )
    private var users: FetchedResults<User>

    var body: some View {
        List(users) { user in
            Text(user.name)
        }
    }
}

// 迁移后（SwiftData）
struct UserListView: View {
    @Query(sort: \User.createdAt, order: .reverse)
    private var users: [User]

    var body: some View {
        List(users) { user in
            Text(user.name)
        }
    }
}
```

### 完整代码示例

以下是一个完整的从 Core Data 迁移到 SwiftData 的 App 入口：

```swift
import SwiftUI
import SwiftData

@main
struct MigratedApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .modelContainer(for: [User.self, Post.self])
    }
}

struct ContentView: View {
    @Environment(\.modelContext) private var modelContext
    @Query(sort: \User.createdAt, order: .reverse) private var users: [User]

    var body: some View {
        NavigationStack {
            List(users) { user in
                VStack(alignment: .leading) {
                    Text(user.name)
                        .font(.headline)
                    Text(user.email)
                        .font(.subheadline)
                        .foregroundStyle(.secondary)
                    Text("\(user.posts.count) posts")
                        .font(.caption)
                        .foregroundStyle(.tertiary)
                }
            }
            .navigationTitle("Users")
            .toolbar {
                Button("Add Sample") {
                    let user = User(name: "Test User", email: "test@example.com")
                    let post = Post(title: "Hello", content: "World", author: user)
                    modelContext.insert(user)
                    modelContext.insert(post)
                }
            }
        }
    }
}
```

---

## 6. 常见问题与解决方案

### 6.1 迁移后数据丢失

**症状**：迁移完成后，查询结果为空或部分数据缺失。

**原因**：

- `ModelConfiguration` 的 URL 指向了错误的 SQLite 文件
- 迁移脚本中遗漏了某些 Entity 的数据转换
- 自定义迁移的 `willMigrate` 中先删除了旧数据但未正确插入新数据

**解决方案**：

```swift
static let migrateV1toV2 = MigrationStage.custom(
    fromVersion: SchemaV1.self,
    toVersion: SchemaV2.self,
    willMigrate: { context in
        let oldItems = try context.fetch(FetchDescriptor<SchemaV1.Item>())
        for oldItem in oldItems {
            let newItem = SchemaV2.Item(
                name: oldItem.name,
                price: 0,
                createdAt: oldItem.createdAt
            )
            context.insert(newItem)
        }
        try context.save()
        for oldItem in oldItems {
            context.delete(oldItem)
        }
        try context.save()
    },
    didMigrate: { _ in }
)
```

> ⚠️ **警告**：务必在迁移前备份用户数据。可以在迁移开始前将 SQLite 文件复制到备份目录，一旦迁移失败可以从备份恢复。

### 6.2 关系断裂问题

**症状**：迁移后，一对多关系的反向引用为 nil，或多对多关系丢失关联。

**原因**：

- 迁移脚本中只创建了主对象，未设置关系
- `@Relationship` 的 `inverse` 参数配置错误
- Core Data 的有序关系（ordered）在 SwiftData 中处理不当

**解决方案**：

```swift
willMigrate: { context in
    let oldUsers = try context.fetch(FetchDescriptor<SchemaV1.User>())
    for oldUser in oldUsers {
        let newUser = SchemaV2.User(
            name: oldUser.name,
            email: oldUser.email,
            createdAt: oldUser.createdAt
        )
        context.insert(newUser)

        for oldPost in oldUser.posts {
            let newPost = SchemaV2.Post(
                title: oldPost.title,
                content: oldPost.content,
                createdAt: oldPost.createdAt,
                author: newUser
            )
            context.insert(newPost)
        }
    }
    try context.save()
}
```

> 💡 **提示**：设置关系时，优先从"多"的一方设置反向引用（如 `post.author = user`），SwiftData 会自动维护双向关系。

### 6.3 性能下降

**症状**：迁移后 App 启动变慢、列表滚动卡顿、数据操作延迟。

**原因**：

- `@Query` 默认获取所有属性，包括大字段（如图片的 Binary Data）
- 关系未设置懒加载，一次性加载了所有关联数据
- 迁移脚本在主线程执行了大量数据操作

**解决方案**：

```swift
// 只查询需要的字段
var descriptor = FetchDescriptor<User>(
    predicate: #Predicate { $0.name.isEmpty == false }
)
descriptor.fetchLimit = 50

// 大字段使用外部存储
@Model
class Document {
    var title: String
    @Attribute(.externalStorage)
    var fileData: Data?
}
```

### 6.4 iOS 17 以下版本的兼容处理

**症状**：在 iOS 16 及以下设备上崩溃。

**原因**：SwiftData 仅支持 iOS 17+，无法在低版本系统上运行。

**解决方案**：使用条件编译和运行时检查实现双版本兼容：

```swift
if #available(iOS 17, *) {
    // 使用 SwiftData
    let container = try ModelContainer(for: User.self)
} else {
    // 使用 Core Data
    let container = NSPersistentContainer(name: "MyApp")
    container.loadPersistentStores { _, error in
        if let error = error {
            fatalError("\(error)")
        }
    }
}
```

> 💡 **提示**：如果需要兼容 iOS 16 及以下，建议暂缓迁移到 SwiftData，继续使用 Core Data。双版本兼容的维护成本远高于等待最低部署目标升级到 iOS 17 后再迁移。

### 6.5 回退方案

迁移不可能永远一帆风顺，必须有回退方案：

1. **迁移前备份**：在执行迁移前复制 SQLite 文件到备份目录

```swift
func backupCoreDataStore() -> URL? {
    let storeURL = Self.coreDataStoreURL
    let backupURL = storeURL.deletingLastPathComponent()
        .appendingPathComponent("MyApp_backup.sqlite")
    do {
        if FileManager.default.fileExists(atPath: storeURL.path) {
            try FileManager.default.copyItem(at: storeURL, to: backupURL)
            return backupURL
        }
    } catch {
        print("Backup failed: \(error)")
    }
    return nil
}
```

2. **版本标记**：使用 UserDefaults 记录迁移状态

```swift
func markMigrationCompleted() {
    UserDefaults.standard.set(true, forKey: "SwiftDataMigrationCompleted")
}

func isMigrationCompleted() -> Bool {
    UserDefaults.standard.bool(forKey: "SwiftDataMigrationCompleted")
}
```

3. **回退逻辑**：如果迁移失败，从备份恢复 Core Data 存储

```swift
func rollbackToCoreData(backupURL: URL) {
    let storeURL = Self.coreDataStoreURL
    do {
        try FileManager.default.removeItem(at: storeURL)
        try FileManager.default.copyItem(at: backupURL, to: storeURL)
        UserDefaults.standard.set(false, forKey: "SwiftDataMigrationCompleted")
    } catch {
        print("Rollback failed: \(error)")
    }
}
```

---

## 小结

本章详细介绍了从 Core Data 迁移到 SwiftData 的完整流程。以下是核心知识点总结：

| 知识点 | 核心内容 |
|--------|---------|
| 迁移动机 | SwiftData 是 Core Data 的现代封装，提供纯 Swift API、类型安全、声明式查询 |
| 底层关系 | SwiftData 底层就是 Core Data，共享 SQLite 存储格式 |
| 迁移策略 | 全量迁移（小项目）、渐进式迁移（中大型）、双写策略（大型/高风险） |
| 模型映射 | Entity → @Model、属性类型一一对应、关系用 @Relationship 声明 |
| 版本管理 | VersionedSchema 定义版本、SchemaMigrationPlan 管理迁移阶段 |
| 迁移步骤 | 创建模型→配置容器→编写迁移脚本→验证数据→切换代码 |
| 数据丢失 | 备份先行、迁移脚本先插入后删除、验证数量一致性 |
| 关系断裂 | 从"多"方设置反向引用、检查 @Relationship 的 inverse 参数 |
| 性能问题 | 使用 fetchLimit、外部存储大字段、避免主线程大量操作 |
| 兼容性 | iOS 17 以下无法使用 SwiftData，建议暂缓迁移而非双版本兼容 |
| 回退方案 | 迁移前备份 SQLite、UserDefaults 标记迁移状态、失败时从备份恢复 |

> 💡 **提示**：迁移的核心原则是"安全第一"。无论选择哪种策略，都要确保用户数据不会丢失，并且有可靠的回退方案。

← [Core Data 入门（选读）](./Core-Data入门选读.md) | [SwiftUI Charts 数据可视化](./SwiftUI-Charts数据可视化.md) →