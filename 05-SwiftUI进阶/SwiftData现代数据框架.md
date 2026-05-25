# 58 - SwiftData 现代数据框架

## 本章目标

- 理解 SwiftData 是什么，以及它和 Core Data 的关系
- 学会用 `@Model` 宏定义数据模型，包括属性和关系
- 掌握 ModelContainer 与 ModelContext 的配置和使用
- 熟练进行 CRUD（创建、读取、更新、删除）操作
- 会用 `@Query` 和 `#Predicate` 进行数据查询与过滤
- 了解数据迁移的基本方式
- 能将 SwiftData 集成到 SwiftUI 项目中，实现数据持久化
- 完成一个待办清单 App 的实战练习

---

## 1. SwiftData 是什么？

想象一下：你写了一个待办清单 App，添加了十几条任务，结果一关 App 再打开——全没了！😱 这就是因为数据没有**持久化**（保存到本地）。

SwiftData 就是 Apple 在 iOS 17 推出的**现代数据持久化框架**，它是 Core Data 的 Swift 原生替代品。

### 核心特点

| 特点 | 说明 |
|------|------|
| Swift 原生 | 用 Swift 宏和属性包装器，告别 .xcdatamodeld 文件 |
| 声明式语法 | 用 `@Model` 定义模型，就像用 SwiftUI 写界面一样直观 |
| 类型安全 | 编译期检查，不再有字符串拼错的运行时崩溃 |
| 与 SwiftUI 深度集成 | `@Query` 自动驱动视图刷新 |
| iOS 17+ | 最低支持 iOS 17、macOS 14、watchOS 10 等 |

> 💡 **生活类比**：Core Data 就像手动挡汽车——功能强大但操作复杂；SwiftData 就像自动挡——踩油门就走，底层还是那台发动机。

> ⚠️ **重要提醒**：SwiftData 要求 iOS 17+，如果你的 App 需要支持更低版本，仍需使用 Core Data。

---

## 2. SwiftData vs Core Data 对比

| 对比项 | Core Data | SwiftData |
|--------|-----------|-----------|
| 模型定义 | .xcdatamodeld 可视化编辑器 | `@Model` 宏，纯 Swift 代码 |
| 查询方式 | NSFetchRequest + NSPredicate | `@Query` + `#Predicate` |
| 上下文操作 | NSManagedObjectContext | ModelContext |
| 存储配置 | NSPersistentContainer | ModelContainer |
| 类型安全 | ❌ 字符串键值，运行时才报错 | ✅ 编译期检查 |
| 代码量 | 多（需手动管理托管对象） | 少（宏自动生成样板代码） |
| 学习曲线 | 陡峭 | 平缓 |
| SwiftUI 集成 | 需手动桥接 | 原生支持 |
| 最低系统版本 | iOS 3+ | iOS 17+ |
| 数据迁移 | 轻量迁移 + 手动映射模型 | 轻量迁移 + VersionedSchema |

> 💡 **简单总结**：新项目优先选 SwiftData；老项目或需兼容 iOS 16 及以下的，继续用 Core Data。

---

## 3. 定义数据模型

### 3.1 @Model 宏

在 Core Data 时代，你需要打开 .xcdatamodeld 文件，用可视化编辑器点来点去。SwiftData 只需要加一个 `@Model` 宏：

```swift
import SwiftData

@Model
class TodoItem {
    var title: String
    var isDone: Bool
    var createdAt: Date

    init(title: String, isDone: Bool = false, createdAt: Date = .now) {
        self.title = title
        self.isDone = isDone
        self.createdAt = createdAt
    }
}
```

就这么简单！`@Model` 宏会在编译时自动帮你：
- 让类遵循 `PersistentModel` 协议
- 生成所有属性的 getter/setter（带持久化支持）
- 添加 Observable 一致性（视图自动刷新）

### 3.2 支持的属性类型

| 类型分类 | 具体类型 |
|----------|----------|
| 基本类型 | Int, Double, Float, String, Bool |
| 日期 | Date |
| 数据 | Data |
| UUID | UUID |
| 枚举 | 遵循 `Codable` 的枚举 |
| 结构体 | 遵循 `Codable` 的结构体 |
| 关系 | 其他 `@Model` 类型的实例或集合 |

### 3.3 特殊属性标注

```swift
@Model
class TodoItem {
    var title: String
    @Attribute(.unique) var id: UUID
    @Attribute(.externalStorage) var attachment: Data?
    @Transient var tempCache: String = ""
    @Relationship(deleteRule: .cascade) var subtasks: [SubTask] = []

    init(title: String, id: UUID = UUID()) {
        self.title = title
        self.id = id
    }
}
```

| 标注 | 作用 |
|------|------|
| `@Attribute(.unique)` | 该属性值唯一，不可重复 |
| `@Attribute(.externalStorage)` | 数据存储在外部文件，适合大文件 |
| `@Transient` | 不持久化，仅内存中使用 |
| `@Relationship` | 定义关系及删除规则 |

### 3.4 关系（Relationship）

关系就像现实世界中事物之间的联系：一个人有多本书，一本书属于一个人。

#### 一对一

```swift
@Model
class User {
    var name: String
    @Relationship(deleteRule: .cascade) var profile: Profile?

    init(name: String) {
        self.name = name
    }
}

@Model
class Profile {
    var avatar: Data?
    var bio: String
    var user: User?

    init(bio: String) {
        self.bio = bio
    }
}
```

#### 一对多

```swift
@Model
class Category {
    var name: String
    @Relationship(deleteRule: .cascade, inverse: \TodoItem.category)
    var items: [TodoItem] = []

    init(name: String) {
        self.name = name
    }
}

@Model
class TodoItem {
    var title: String
    var category: Category?

    init(title: String) {
        self.title = title
    }
}
```

#### 多对多

```swift
@Model
class Tag {
    var name: String
    @Relationship(deleteRule: .nullify, inverse: \TodoItem.tags)
    var todos: [TodoItem] = []

    init(name: String) {
        self.name = name
    }
}

@Model
class TodoItem {
    var title: String
    @Relationship(deleteRule: .nullify, inverse: \Tag.todos)
    var tags: [Tag] = []

    init(title: String) {
        self.title = title
    }
}
```

#### 删除规则对比

| 删除规则 | 说明 | 适用场景 |
|----------|------|----------|
| `.cascade` | 删除父对象时，子对象也一起删除 | 用户删除 → 个人资料也删 |
| `.nullify` | 删除父对象时，子对象的关系属性置为 nil | 标签删除 → 待办还在，标签字段变 nil |
| `.deny` | 如果还有子对象关联，则拒绝删除父对象 | 有订单的客户不允许删除 |
| `.noAction` | 不做任何操作（慎用，可能产生悬挂引用） | 极少使用 |

> ⚠️ **注意**：定义双向关系时，务必用 `inverse:` 参数指定反向关系的 keyPath，否则 SwiftData 无法正确维护关系。

---

## 4. ModelContainer 与 ModelContext

### 4.1 生活类比

| 概念 | 类比 | 说明 |
|------|------|------|
| ModelContainer | 仓库 | 决定数据存在哪、存哪些类型 |
| ModelContext | 仓库管理员 | 负责具体的存取、修改操作 |

### 4.2 配置 ModelContainer

#### 最简配置

```swift
import SwiftData

@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .modelContainer(for: TodoItem.self)
    }
}
```

#### 多模型配置

```swift
.modelContainer(for: [TodoItem.self, Category.self, Tag.self])
```

#### 自定义配置

```swift
import SwiftData

@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .modelContainer(for: TodoItem.self) { result in
            if case .failure(let error) = result {
                fatalError("Failed to create ModelContainer: \(error)")
            }
        }
    }
}
```

#### 高级配置：Schema + ModelConfiguration

```swift
let schema = Schema([TodoItem.self, Category.self, Tag.self])
let config = ModelConfiguration(
    "TodoApp",
    schema: schema,
    isStoredInMemoryOnly: false,
    allowsSave: true,
    groupContainer: .none,
    cloudKitDatabase: .automatic
)
let container = try ModelContainer(for: schema, configurations: [config])
```

| ModelConfiguration 参数 | 说明 |
|------------------------|------|
| `isStoredInMemoryOnly` | `true` 则数据仅存内存（适合测试） |
| `allowsSave` | `false` 则只读 |
| `groupContainer` | App Group 共享数据 |
| `cloudKitDatabase` | CloudKit 同步设置 |

### 4.3 获取 ModelContext

#### 方式一：环境自动注入

`.modelContainer(for:)` 会自动把 ModelContext 注入环境：

```swift
struct ContentView: View {
    @Environment(\.modelContext) private var modelContext

    var body: some View {
        Text("Hello")
    }
}
```

#### 方式二：从 Container 获取

```swift
let context = container.mainContext
```

### 4.4 ModelContext 常用操作

```swift
@Environment(\.modelContext) private var modelContext

func saveData() throws {
    try modelContext.save()
}

func deleteData(_ item: TodoItem) {
    modelContext.delete(item)
}

func fetchAll() throws -> [TodoItem] {
    try modelContext.fetch(FetchDescriptor<TodoItem>())
}

func undoChange() {
    modelContext.undoManager?.undo()
}
```

> 💡 **提示**：SwiftData 的 ModelContext 默认会在合适的时机自动保存，大多数情况下你不需要手动调用 `save()`。但如果你需要确保数据立即落盘，可以显式调用。

---

## 5. CRUD 操作

### 5.1 Create（创建）

```swift
@Environment(\.modelContext) private var modelContext

func addTodo(title: String) {
    let item = TodoItem(title: title)
    modelContext.insert(item)
}
```

就这么两行！创建对象 + 插入上下文，数据就持久化了。

### 5.2 Read（读取）

#### 基础读取：FetchDescriptor

```swift
let descriptor = FetchDescriptor<TodoItem>(
    sortBy: [SortDescriptor(\.createdAt, order: .reverse)]
)
let items = try modelContext.fetch(descriptor)
```

#### 条件过滤：#Predicate

```swift
let predicate = #Predicate<TodoItem> { $0.isDone == false }
let descriptor = FetchDescriptor<TodoItem>(predicate: predicate)
let unfinishedItems = try modelContext.fetch(descriptor)
```

#### 排序：SortDescriptor

```swift
let descriptor = FetchDescriptor<TodoItem>(
    sortBy: [
        SortDescriptor(\.isDone),
        SortDescriptor(\.createdAt, order: .reverse)
    ]
)
```

> 💡 **生活类比**：`FetchDescriptor` 就像你去图书馆借书时填的检索单——告诉管理员你要什么条件、按什么排序。

### 5.3 Update（更新）

SwiftData 的更新超级简单——直接改属性值就行！

```swift
func toggleDone(item: TodoItem) {
    item.isDone.toggle()
}
```

因为 `@Model` 宏让类变成了 Observable，属性修改会自动被 ModelContext 追踪，无需额外操作。

### 5.4 Delete（删除）

```swift
func deleteItem(item: TodoItem) {
    modelContext.delete(item)
}
```

#### 批量删除

```swift
func deleteAll() throws {
    try modelContext.delete(model: TodoItem.self)
}
```

#### 配合 SwiftUI 的 swipeActions

```swift
List {
    ForEach(items) { item in
        TodoRow(item: item)
    }
    .onDelete(perform: deleteItems)
}

func deleteItems(at offsets: IndexSet) {
    for index in offsets {
        modelContext.delete(items[index])
    }
}
```

### CRUD 速查表

| 操作 | 代码 | 说明 |
|------|------|------|
| 创建 | `modelContext.insert(item)` | 插入新对象 |
| 读取 | `modelContext.fetch(descriptor)` | 按条件查询 |
| 更新 | `item.property = newValue` | 直接赋值，自动追踪 |
| 删除 | `modelContext.delete(item)` | 删除单个对象 |
| 批量删除 | `modelContext.delete(model: Type.self)` | 删除某类型全部 |

---

## 6. 数据查询与过滤

### 6.1 @Query 属性包装器

`@Query` 是 SwiftData 与 SwiftUI 的桥梁，它自动查询数据并驱动视图刷新：

```swift
struct TodoListView: View {
    @Query var items: [TodoItem]

    var body: some View {
        List(items) { item in
            Text(item.title)
        }
    }
}
```

### 6.2 @Query 排序

```swift
@Query(sort: \TodoItem.createdAt, order: .reverse)
var items: [TodoItem]
```

多字段排序：

```swift
@Query(sort: [
    SortDescriptor(\TodoItem.isDone),
    SortDescriptor(\TodoItem.createdAt, order: .reverse)
])
var items: [TodoItem]
```

### 6.3 @Query 过滤

```swift
@Query(filter: #Predicate<TodoItem> { $0.isDone == false })
var unfinishedItems: [TodoItem]
```

### 6.4 #Predicate 构建器

`#Predicate` 是 Swift 5.9 引入的宏，用于构建类型安全的查询条件。

#### 基础比较

```swift
#Predicate<TodoItem> { $0.isDone == false }
#Predicate<TodoItem> { $0.title == "买牛奶" }
#Predicate<TodoItem> { $0.createdAt > Date.now.addingTimeInterval(-86400 * 7) }
```

#### 逻辑组合

```swift
#Predicate<TodoItem> {
    $0.isDone == false && $0.title.contains("紧急")
}
```

#### 多条件用变量

```swift
let keyword = "会议"
let isDone = false

let predicate = #Predicate<TodoItem> {
    $0.isDone == isDone && $0.title.contains(keyword)
}
```

#### #Predicate 支持的运算符

| 类别 | 运算符 |
|------|--------|
| 比较 | `==`, `!=`, `>`, `<`, `>=`, `<=` |
| 逻辑 | `&&`, `\|\|`, `!` |
| 字符串 | `.contains()`, `.hasPrefix()`, `.hasSuffix()` |
| 集合 | `.isEmpty`, `.contains()` |
| 可选值 | `== nil`, `!= nil` |

> ⚠️ **限制**：`#Predicate` 中不能使用自定义函数、闭包、流程控制语句（if/switch/for），只能用上述运算符和系统提供的方法。

### 6.5 完整 @Query 用法速查

```swift
@Query var allItems: [TodoItem]

@Query(sort: \TodoItem.createdAt) var sortedItems: [TodoItem]

@Query(filter: #Predicate<TodoItem> { !$0.isDone })
var activeItems: [TodoItem]

@Query(
    filter: #Predicate<TodoItem> { !$0.isDone },
    sort: \TodoItem.createdAt,
    order: .reverse
) var filteredAndSorted: [TodoItem]
```

---

## 7. 数据迁移

App 迭代过程中，数据模型难免会变。SwiftData 提供了迁移机制，确保旧数据不会丢失。

### 7.1 生活类比

数据迁移就像搬家——房子（模型）变了，家具（数据）得跟着搬到新格局里。

### 7.2 轻量迁移（Lightweight Migration）

当你只做了"安全"的改动时，SwiftData 会自动迁移，无需额外代码：

| 自动迁移支持的改动 | 示例 |
|-------------------|------|
| 新增属性 | 加一个 `priority: Int = 0` |
| 删除属性 | 去掉 `tempNote: String` |
| 重命名属性 | `name` → `title`（需特殊标注） |
| 修改默认值 | `isDone` 默认值从 `true` 改为 `false` |

新增属性时给默认值即可：

```swift
@Model
class TodoItem {
    var title: String
    var isDone: Bool
    var createdAt: Date
    var priority: Int = 0

    init(title: String, isDone: Bool = false, createdAt: Date = .now, priority: Int = 0) {
        self.title = title
        self.isDone = isDone
        self.createdAt = createdAt
        self.priority = priority
    }
}
```

### 7.3 手动迁移（VersionedSchema）

当改动较复杂（如属性类型变化、数据需要转换）时，需要手动迁移。

#### 步骤一：定义版本化 Schema

```swift
import SwiftData

enum TodoItemSchemaV1: VersionedSchema {
    static var versionIdentifier = Schema.Version(1, 0, 0)

    @Model
    class TodoItem {
        var title: String
        var isDone: Bool
        var createdAt: Date

        init(title: String, isDone: Bool = false, createdAt: Date = .now) {
            self.title = title
            self.isDone = isDone
            self.createdAt = createdAt
        }
    }
}

enum TodoItemSchemaV2: VersionedSchema {
    static var versionIdentifier = Schema.Version(2, 0, 0)

    @Model
    class TodoItem {
        var title: String
        var isDone: Bool
        var createdAt: Date
        var priority: Priority

        enum Priority: Int, Codable {
            case low, medium, high
        }

        init(title: String, isDone: Bool = false, createdAt: Date = .now, priority: Priority = .medium) {
            self.title = title
            self.isDone = isDone
            self.createdAt = createdAt
            self.priority = priority
        }
    }
}
```

#### 步骤二：定义迁移计划

```swift
enum TodoItemMigrationPlan: SchemaMigrationPlan {
    static var schemas: [any VersionedSchema.Type] {
        [TodoItemSchemaV1.self, TodoItemSchemaV2.self]
    }

    static var stages: [MigrationStage] {
        [migrateV1toV2]
    }

    static let migrateV1toV2 = MigrationStage.custom(
        fromVersion: TodoItemSchemaV1.self,
        toVersion: TodoItemSchemaV2.self,
        willMigrate: { context in
            let items = try context.fetch(FetchDescriptor<TodoItemSchemaV1.TodoItem>())
            for item in items {
                let newItem = TodoItemSchemaV2.TodoItem(
                    title: item.title,
                    isDone: item.isDone,
                    createdAt: item.createdAt,
                    priority: .medium
                )
                context.delete(item)
                context.insert(newItem)
            }
            try context.save()
        },
        didMigrate: nil
    )
}
```

#### 步骤三：使用迁移计划

```swift
@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .modelContainer(
            for: TodoItem.self,
            migrationPlan: TodoItemMigrationPlan.self
        )
    }
}
```

### 7.4 迁移策略选择

| 场景 | 策略 |
|------|------|
| 仅新增/删除属性 | 轻量迁移（自动） |
| 属性重命名 | 轻量迁移 + `@Attribute(.rename(from:))` |
| 属性类型变化 | 手动迁移 |
| 数据需要转换逻辑 | 手动迁移 |
| 多版本跨越 | 手动迁移（定义多个 stage） |

> ⚠️ **最佳实践**：在 App 上线前就规划好迁移方案。上线后修改模型时，务必测试迁移流程，避免用户数据丢失。

---

## 8. 与 SwiftUI 集成

### 8.1 @Query 驱动列表

SwiftData 与 SwiftUI 的集成是"零摩擦"的——`@Query` 自动获取数据，数据变化时视图自动刷新：

```swift
struct TodoListView: View {
    @Query(sort: \TodoItem.createdAt, order: .reverse)
    var items: [TodoItem]

    @Environment(\.modelContext) private var modelContext

    var body: some View {
        List {
            ForEach(items) { item in
                HStack {
                    Image(systemName: item.isDone ? "checkmark.circle.fill" : "circle")
                        .onTapGesture { item.isDone.toggle() }
                    Text(item.title)
                        .strikethrough(item.isDone)
                }
            }
            .onDelete(perform: deleteItems)
        }
    }

    func deleteItems(at offsets: IndexSet) {
        for index in offsets {
            modelContext.delete(items[index])
        }
    }
}
```

### 8.2 自动刷新机制

```
数据变化 → ModelContext 检测到变更 → 通知 @Query → SwiftUI 重新渲染视图
```

你**不需要**手动调用 `objectWillChange.send()` 或 `state = state`，一切都是自动的！

### 8.3 添加数据的 Sheet

```swift
struct AddTodoView: View {
    @Environment(\.modelContext) private var modelContext
    @Environment(\.dismiss) private var dismiss

    @State private var title = ""

    var body: some View {
        NavigationStack {
            Form {
                TextField("任务标题", text: $title)
            }
            .navigationTitle("新建任务")
            .toolbar {
                ToolbarItem(placement: .confirmationAction) {
                    Button("保存") {
                        let item = TodoItem(title: title)
                        modelContext.insert(item)
                        dismiss()
                    }
                }
                ToolbarItem(placement: .cancellationAction) {
                    Button("取消") { dismiss() }
                }
            }
        }
    }
}
```

### 8.4 完整 App 骨架

```swift
import SwiftUI
import SwiftData

@main
struct TodoApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .modelContainer(for: TodoItem.self)
    }
}

struct ContentView: View {
    @State private var showAddSheet = false

    var body: some View {
        NavigationStack {
            TodoListView()
                .navigationTitle("待办清单")
                .toolbar {
                    ToolbarItem(placement: .primaryAction) {
                        Button(action: { showAddSheet = true }) {
                            Image(systemName: "plus")
                        }
                    }
                }
                .sheet(isPresented: $showAddSheet) {
                    AddTodoView()
                }
        }
    }
}
```

---

## 9. 实战示例：给待办清单 App 添加 SwiftData 持久化

让我们把前面学到的知识串起来，做一个完整的待办清单 App。

### 9.1 定义模型

```swift
import Foundation
import SwiftData

@Model
class TodoItem {
    var title: String
    var isDone: Bool
    var priority: Int
    var createdAt: Date
    @Relationship(deleteRule: .cascade) var subtasks: [SubTask] = []

    var priorityLabel: String {
        switch priority {
        case 0: return "低"
        case 1: return "中"
        case 2: return "高"
        default: return "中"
        }
    }

    init(title: String, isDone: Bool = false, priority: Int = 1, createdAt: Date = .now) {
        self.title = title
        self.isDone = isDone
        self.priority = priority
        self.createdAt = createdAt
    }
}

@Model
class SubTask {
    var title: String
    var isDone: Bool
    var parent: TodoItem?

    init(title: String, isDone: Bool = false) {
        self.title = title
        self.isDone = isDone
    }
}
```

### 9.2 App 入口

```swift
import SwiftUI
import SwiftData

@main
struct TodoApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .modelContainer(for: [TodoItem.self, SubTask.self])
    }
}
```

### 9.3 主列表视图

```swift
struct ContentView: View {
    @Environment(\.modelContext) private var modelContext
    @Query(
        filter: #Predicate<TodoItem> { $0.isDone == false },
        sort: [SortDescriptor(\TodoItem.priority, order: .reverse)]
    )
    private var activeItems: [TodoItem]

    @Query(filter: #Predicate<TodoItem> { $0.isDone == true })
    private var doneItems: [TodoItem]

    @State private var showAddSheet = false

    var body: some View {
        NavigationStack {
            List {
                Section("进行中") {
                    ForEach(activeItems) { item in
                        TodoRowView(item: item)
                    }
                    .onDelete(perform: deleteActive)
                }

                Section("已完成") {
                    ForEach(doneItems) { item in
                        TodoRowView(item: item)
                    }
                    .onDelete(perform: deleteDone)
                }
            }
            .navigationTitle("待办清单")
            .toolbar {
                ToolbarItem(placement: .primaryAction) {
                    Button(action: { showAddSheet = true }) {
                        Image(systemName: "plus")
                    }
                }
            }
            .sheet(isPresented: $showAddSheet) {
                AddTodoView()
            }
        }
    }

    func deleteActive(at offsets: IndexSet) {
        for index in offsets {
            modelContext.delete(activeItems[index])
        }
    }

    func deleteDone(at offsets: IndexSet) {
        for index in offsets {
            modelContext.delete(doneItems[index])
        }
    }
}
```

### 9.4 行视图

```swift
struct TodoRowView: View {
    let item: TodoItem

    var body: some View {
        HStack {
            Button(action: { item.isDone.toggle() }) {
                Image(systemName: item.isDone ? "checkmark.circle.fill" : "circle")
                    .foregroundStyle(item.isDone ? .green : .gray)
            }
            .buttonStyle(.plain)

            VStack(alignment: .leading) {
                Text(item.title)
                    .strikethrough(item.isDone)
                Text("优先级：\(item.priorityLabel)")
                    .font(.caption)
                    .foregroundStyle(.secondary)
            }

            Spacer()

            Text(item.subtasks.filter { !$0.isDone }.count > 0
                 ? "\(item.subtasks.filter { !$0.isDone }.count) 个子任务"
                 : "")
                .font(.caption2)
                .foregroundStyle(.secondary)
        }
    }
}
```

### 9.5 添加任务视图

```swift
struct AddTodoView: View {
    @Environment(\.modelContext) private var modelContext
    @Environment(\.dismiss) private var dismiss

    @State private var title = ""
    @State private var priority = 1

    var body: some View {
        NavigationStack {
            Form {
                TextField("任务标题", text: $title)

                Picker("优先级", selection: $priority) {
                    Text("低").tag(0)
                    Text("中").tag(1)
                    Text("高").tag(2)
                }
                .pickerStyle(.segmented)
            }
            .navigationTitle("新建任务")
            .toolbar {
                ToolbarItem(placement: .confirmationAction) {
                    Button("保存") {
                        guard !title.isEmpty else { return }
                        let item = TodoItem(title: title, priority: priority)
                        modelContext.insert(item)
                        dismiss()
                    }
                }
                ToolbarItem(placement: .cancellationAction) {
                    Button("取消") { dismiss() }
                }
            }
        }
    }
}
```

### 9.6 运行效果

| 操作 | 效果 |
|------|------|
| 打开 App | 显示空列表 |
| 点击 + | 弹出新建表单 |
| 输入标题，选择优先级，点保存 | 任务出现在"进行中"列表 |
| 点击圆圈 | 任务移到"已完成" |
| 左滑删除 | 任务被删除 |
| 关闭 App 再打开 | 数据依然存在 ✅ |

---

## 10. 常见问题与最佳实践

### 常见问题

#### Q1：数据没有持久化，重启 App 后丢失？

检查是否正确配置了 `.modelContainer(for:)`，以及 `isStoredInMemoryOnly` 是否被误设为 `true`。

#### Q2：@Query 查询结果不更新？

确保你使用的是 `@Query` 而不是手动 `fetch`。手动 `fetch` 的结果不会自动刷新视图。

#### Q3：#Predicate 编译报错？

`#Predicate` 中不支持：
- 自定义函数调用
- 可选链 `?.`
- 流程控制语句
- 闭包

只使用支持的运算符和方法即可。

#### Q4：迁移后 App 崩溃？

- 检查迁移计划中 `versionIdentifier` 是否正确递增
- 确保 `willMigrate` 中删除旧对象、插入新对象
- 在模拟器上测试时，可以先删除 App 重新安装

#### Q5：如何调试 SwiftData？

在 Scheme 的启动参数中添加：
```
-com.apple.CoreData.SQLDebug 1
```
这样可以在控制台看到 SQL 语句，方便排查问题。

### 最佳实践

| 实践 | 说明 |
|------|------|
| ✅ 新项目优先用 SwiftData | 代码简洁，开发效率高 |
| ✅ 给新增属性设默认值 | 方便轻量迁移自动处理 |
| ✅ 使用 `@Relationship` 的 `inverse` 参数 | 确保双向关系正确维护 |
| ✅ 选择合适的删除规则 | 避免误删或悬挂引用 |
| ✅ 测试迁移流程 | 每次改模型都要测试从旧版本迁移 |
| ✅ 大量数据用分页 | `FetchDescriptor` 支持 `fetchLimit` 和 `fetchOffset` |
| ❌ 不要在 #Predicate 中写复杂逻辑 | 提前过滤或简化条件 |
| ❌ 不要在视图初始化时手动 fetch | 用 `@Query` 自动管理 |
| ❌ 不要忘记 `.modelContainer` | 没有容器，SwiftData 无法工作 |

### 性能优化小贴士

```swift
var descriptor = FetchDescriptor<TodoItem>(
    predicate: #Predicate { $0.isDone == false },
    sortBy: [SortDescriptor(\.createdAt)]
)
descriptor.fetchLimit = 50
descriptor.fetchOffset = 0
```

| 优化手段 | 代码 |
|----------|------|
| 分页加载 | `descriptor.fetchLimit = 50` |
| 偏移跳过 | `descriptor.fetchOffset = 100` |
| 只读不追踪 | `descriptor.prefetchingRelationships = []` |
| 后台上下文 | 使用 `ModelContext(container)` 在后台线程操作 |

---

## 小结

| 知识点 | 关键内容 |
|--------|----------|
| SwiftData 是什么 | Core Data 的 Swift 原生替代，iOS 17+，声明式语法 |
| 模型定义 | `@Model` 宏 + 属性标注 + 关系定义 |
| 容器与上下文 | `ModelContainer` 配置存储，`ModelContext` 执行操作 |
| CRUD | insert / fetch / 直接赋值 / delete |
| 查询过滤 | `@Query` + `#Predicate` + `SortDescriptor` |
| 数据迁移 | 轻量迁移（自动）+ VersionedSchema（手动） |
| SwiftUI 集成 | `@Query` 自动刷新、`.modelContainer` 注入 |
| 实战 | 待办清单 App 完整持久化方案 |

SwiftData 让数据持久化变得像写 SwiftUI 界面一样简单——声明你想要什么，框架帮你实现。掌握了 SwiftData，你的 App 就能真正"记住"用户的数据了！🎉

← [动画与手势](./动画与手势.md) | [@Observable 与 Observation 框架](./Observable与Observation框架.md) →
