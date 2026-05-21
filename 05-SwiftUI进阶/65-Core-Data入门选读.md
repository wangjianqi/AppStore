# 65-Core Data 入门（选读）

## 本章目标

- 理解 Core Data 的定位与价值，以及它与 SwiftData 的关系
- 掌握 Core Data 四层架构的职责与协作方式
- 学会在 Xcode 中创建 .xcdatamodeld 数据模型
- 熟练进行 CRUD 操作，包括 NSFetchRequest、NSPredicate、NSSortDescriptor
- 理解数据关系的建立与维护（一对一、一对多、多对多）
- 了解数据迁移的两种方式：轻量迁移与 Mapping Model
- 能将 Core Data 集成到 SwiftUI 项目中
- 了解 Core Data + CloudKit 同步的基本配置
- 能根据项目需求在 Core Data 与 SwiftData 之间做出选择

---

## 1. Core Data 概述

### 1.1 为什么还要学 Core Data

SwiftData 已经在 iOS 17 推出，为什么还要花时间学 Core Data？原因很现实：

- **大量存量项目**使用 Core Data，维护和迭代绕不开它
- **企业级 App** 对数据层有精细控制需求，Core Data 更成熟
- **CloudKit 同步**在 Core Data 上更稳定，SwiftData 的 CloudKit 支持仍在完善
- **兼容性**：需要支持 iOS 16 及以下时，Core Data 是唯一选择

> 💡 **生活类比**：SwiftData 就像高铁——快、新、方便；Core Data 就像绿皮火车——老、慢，但线路覆盖广，哪儿都能到。你要去的地方如果没有高铁站，绿皮火车就是唯一选择。

### 1.2 Core Data 与 SwiftData 的关系

Core Data 是 Apple 自 iOS 3 就推出的对象图与持久化框架，SwiftData 本质上是 Core Data 的 Swift 原生封装层：

```
┌─────────────────────────────┐
│        SwiftData            │  ← Swift 原生 API（iOS 17+）
│  @Model / @Query / ModelContext │
├─────────────────────────────┤
│        Core Data            │  ← 底层引擎（iOS 3+）
│  NSManagedObject / NSManagedObjectContext │
├─────────────────────────────┤
│         SQLite              │  ← 默认存储引擎
└─────────────────────────────┘
```

SwiftData 的 `ModelContainer` 内部就是 `NSPersistentContainer`，`ModelContext` 内部就是 `NSManagedObjectContext`。学 Core Data 也是在深入理解 SwiftData 的底层。

### 1.3 何时选择 Core Data

| 场景 | 推荐 |
|------|------|
| 新项目，最低支持 iOS 17+ | SwiftData |
| 新项目，需要兼容 iOS 16 及以下 | Core Data |
| 老项目已用 Core Data | 继续用 Core Data |
| 需要 CloudKit 同步且要求稳定 | Core Data（NSPersistentCloudKitContainer） |
| 需要精细控制 SQL、迁移、并发 | Core Data |
| 快速原型、个人小项目 | SwiftData |

> ⚠️ **提醒**：本章为选读内容。如果你的项目完全基于 iOS 17+ 且使用 SwiftData，可以跳过本章。但了解 Core Data 有助于你更好地理解 SwiftData 的底层机制。

---

## 2. Core Data 架构

Core Data 采用经典的四层架构，每层各司其职：

```
┌──────────────────────────────────────────┐
│         NSManagedObjectContext           │  ← 你最常打交道的层
│         （托管对象上下文 / 工作台）          │
├──────────────────────────────────────────┤
│       NSPersistentStoreCoordinator       │  ← 协调层
│       （持久化存储协调器 / 仓库管理员）       │
├──────────────┬───────────────────────────┤
│NSManagedObjectModel│  Persistent Store   │
│（数据模型 / 蓝图）  │  （存储 / 仓库）       │
└──────────────┴───────────────────────────┘
```

> 💡 **生活类比**：把 Core Data 想象成一家工厂——`NSManagedObjectModel` 是产品蓝图，`NSPersistentStoreCoordinator` 是仓库管理员，`NSManagedObjectContext` 是车间工人的工作台，`NSManagedObject` 是正在加工的产品零件。

### 2.1 NSManagedObjectModel

数据模型的"蓝图"，定义了有哪些 Entity、Attribute 和 Relationship。通常由 `.xcdatamodeld` 文件编译生成。

### 2.2 NSPersistentStoreCoordinator

连接"蓝图"和"仓库"的桥梁，负责：
- 管理一个或多个 Persistent Store（默认是 SQLite）
- 处理数据的读写调度
- 协调多个 Context 的并发访问

### 2.3 NSManagedObjectContext

你日常操作的核心，相当于一个**暂存区**：
- 所有增删改查都在 Context 上进行
- 修改不会立即写入磁盘，需要调用 `save()` 才持久化
- 支持撤销/重做（undo/redo）
- 可以创建多个 Context 实现并发

### 2.4 NSManagedObject

Core Data 中的数据实体基类，对应数据库中的一行记录。每个托管对象都"生活"在某个 Context 中。

```swift
@NSManaged var title: String
@NSManaged var createdAt: Date
@NSManaged var isDone: Bool
```

`@NSManaged` 告诉编译器：这个属性的 getter/setter 由 Core Data 在运行时动态提供。

---

## 3. 创建数据模型

### 3.1 创建 .xcdatamodeld 文件

1. Xcode → File → New → File → Data Model
2. 命名后得到 `.xcdatamodeld` 文件
3. 打开后进入可视化编辑器，可以添加 Entity、Attribute、Relationship

### 3.2 Entity / Attribute / Relationship

| 概念 | 说明 | 类比 |
|------|------|------|
| Entity | 实体，对应一个数据表 | Excel 中的一个 Sheet |
| Attribute | 属性，对应一列 | Excel 中的一列 |
| Relationship | 关系，对应表与表之间的关联 | Sheet 之间的超链接 |

### 3.3 数据类型映射表

| Core Data 类型 | Swift 类型 | 说明 |
|----------------|-----------|------|
| String | String | 字符串 |
| Integer 16 | Int16 | 短整数 |
| Integer 32 | Int32 | 默认整数 |
| Integer 64 | Int64 | 长整数 |
| Float | Float | 单精度浮点 |
| Double | Double | 双精度浮点 |
| Boolean | Bool | 布尔值 |
| Date | Date | 日期时间 |
| Binary Data | Data | 二进制数据 |
| Transformable | 任意（需 Codable） | 自定义类型 |
| URI | URL | 统一资源标识 |

> 💡 **提示**：对于枚举类型，通常存为 Integer 或 String，再在 NSManagedObject 子类中提供计算属性转换。

### 3.4 生成 NSManagedObject 子类

Xcode 提供两种方式：

| 方式 | 说明 |
|------|------|
| 自动生成（推荐） | Editor → Create NSManagedObject Subclass，Xcode 自动维护 |
| 手动定义 | 自己写类和 `@NSManaged` 属性，灵活但需手动同步 |

自动生成的代码示例：

```swift
import Foundation
import CoreData

@objc(TodoItem)
public class TodoItem: NSManagedObject {
}

extension TodoItem {
    @NSManaged public var title: String
    @NSManaged public var isDone: Bool
    @NSManaged public var createdAt: Date
    @NSManaged public var priority: Int16
}
```

> ⚠️ **注意**：自动生成的文件标记为 `//  This file was automatically generated`，不要手动修改它们，否则下次重新生成会覆盖。扩展逻辑应写在 Category/Extension 中。

---

## 4. CRUD 操作

### 4.1 初始化 NSPersistentContainer

```swift
import CoreData

let persistentContainer: NSPersistentContainer = {
    let container = NSPersistentContainer(name: "TodoModel")
    container.loadPersistentStores { description, error in
        if let error = error {
            fatalError("Core Data 加载失败: \(error)")
        }
    }
    return container
}()
```

> 💡 **生活类比**：`NSPersistentContainer` 就像一个已经建好的仓库——名字叫"TodoModel"，里面货架（Store）已经摆好，随时可以存取货物。

### 4.2 Create（创建）

```swift
func createTodo(title: String, context: NSManagedObjectContext) {
    let todo = TodoItem(context: context)
    todo.title = title
    todo.isDone = false
    todo.createdAt = Date()
    todo.priority = 1

    do {
        try context.save()
    } catch {
        context.rollback()
        print("保存失败: \(error)")
    }
}
```

### 4.3 Read（读取）

#### 基础查询

```swift
func fetchAllTodos(context: NSManagedObjectContext) -> [TodoItem] {
    let request: NSFetchRequest<TodoItem> = TodoItem.fetchRequest()
    request.sortDescriptors = [NSSortDescriptor(key: "createdAt", ascending: false)]

    do {
        return try context.fetch(request)
    } catch {
        print("查询失败: \(error)")
        return []
    }
}
```

#### NSPredicate 过滤

```swift
let request: NSFetchRequest<TodoItem> = TodoItem.fetchRequest()
request.predicate = NSPredicate(format: "isDone == NO")
```

常用 NSPredicate 语法：

| 语法 | 示例 | 说明 |
|------|------|------|
| 比较 | `"priority > %d", 1` | 大于 |
| 字符串匹配 | `"title CONTAINS[cd] %@"`, "会议" | 包含，不区分大小写 |
| 日期范围 | `"createdAt > %@"`, someDate | 日期比较 |
| 逻辑组合 | `"isDone == NO AND priority >= %d", 2` | 与 |
| IN | `"priority IN %@"`, [0, 2] | 在集合中 |
| BEGINSWITH | `"title BEGINSWITH %@"`, "紧急" | 前缀匹配 |

#### NSSortDescriptor 排序

```swift
request.sortDescriptors = [
    NSSortDescriptor(key: "isDone", ascending: true),
    NSSortDescriptor(key: "priority", ascending: false),
    NSSortDescriptor(key: "createdAt", ascending: false)
]
```

### 4.4 Update（更新）

```swift
func toggleDone(todo: TodoItem, context: NSManagedObjectContext) {
    todo.isDone.toggle()
    do {
        try context.save()
    } catch {
        context.rollback()
        print("更新失败: \(error)")
    }
}
```

### 4.5 Delete（删除）

```swift
func deleteTodo(todo: TodoItem, context: NSManagedObjectContext) {
    context.delete(todo)
    do {
        try context.save()
    } catch {
        context.rollback()
        print("删除失败: \(error)")
    }
}
```

### 4.6 CRUD 速查表

| 操作 | 核心代码 | 说明 |
|------|---------|------|
| 创建 | `TodoItem(context: ctx)` + `ctx.save()` | 创建对象并保存 |
| 读取 | `ctx.fetch(request)` | 通过 NSFetchRequest 查询 |
| 更新 | 修改属性 + `ctx.save()` | 直接赋值后保存 |
| 删除 | `ctx.delete(obj)` + `ctx.save()` | 从上下文中删除 |

> ⚠️ **关键提醒**：所有修改操作后都必须调用 `context.save()`，否则数据只在内存中，不会写入磁盘。如果保存失败，调用 `context.rollback()` 回滚。

---

## 5. 数据关系处理

### 5.1 关系类型

| 关系类型 | 标记 | 示例 |
|---------|------|------|
| 一对一 | To One | 用户 ↔ 个人资料 |
| 一对多 | To Many | 列表 → 多个任务 |
| 多对多 | To Many（双向） | 学生 ↔ 课程 |

> 💡 **生活类比**：一对一就像一个人只有一张身份证；一对多就像一个班级有很多学生；多对多就像一个学生选多门课，一门课也有多个学生。

### 5.2 建立关系

在 `.xcdatamodeld` 中添加 Relationship 后，生成的代码：

```swift
extension TodoList {
    @NSManaged public var name: String
    @NSManaged public var items: NSSet
}

extension TodoItem {
    @NSManaged public var title: String
    @NSManaged public var list: TodoList?
}
```

使用时通过 KVC 风格的便捷方法操作多对多/一对多关系：

```swift
let list = TodoList(context: context)
list.name = "工作"

let todo = TodoItem(context: context)
todo.title = "写周报"
todo.list = list

list.mutableItems?.add(todo)
```

### 5.3 逆关系（Inverse Relationship）

逆关系是 Core Data 的核心概念——如果 A 指向 B，B 也应该指回 A：

```
TodoList.items  ←(inverse)→  TodoItem.list
```

> ⚠️ **必须设置逆关系**：不设置逆关系会导致数据不一致、内存泄漏，以及难以排查的 Bug。Core Data 依赖逆关系来维护对象图的完整性。

### 5.4 删除规则（Delete Rule）

| 规则 | 说明 | 适用场景 |
|------|------|---------|
| Cascade（级联） | 删除父对象时，子对象一并删除 | 删除列表时，任务也删除 |
| Nullify（置空） | 删除父对象时，子对象的外键置 nil | 删除分类时，文章保留但分类为空 |
| Deny（拒绝） | 如果子对象存在，禁止删除父对象 | 有订单时禁止删除客户 |
| No Action（无操作） | 删除父对象，子对象不变（危险） | 极少使用，需手动维护一致性 |

```swift
let list = TodoList(context: context)
list.name = "购物清单"

let item1 = TodoItem(context: context)
item1.title = "牛奶"
item1.list = list

let item2 = TodoItem(context: context)
item2.title = "面包"
item2.list = list

try context.save()
```

如果 `items` 的删除规则是 Cascade，删除 `list` 时 `item1` 和 `item2` 也会被删除。

---

## 6. 数据迁移

App 迭代时数据模型会变化，Core Data 需要迁移来保证旧数据不丢失。

> 💡 **生活类比**：数据迁移就像搬家——旧房子（旧模型）里的家具（数据）要搬到新房子（新模型）里，位置可能变了，但东西不能丢。

### 6.1 版本管理

1. 选中 `.xcdatamodeld` 文件
2. Editor → Add Model Version
3. 新版本基于旧版本创建，可以修改 Entity/Attribute
4. 设置新版本为 Current Model Version

### 6.2 轻量迁移（Lightweight Migration）

适用于"安全"的改动，Core Data 自动推断映射：

| 自动支持的改动 | 示例 |
|---------------|------|
| 新增属性 | 加 `priority: Int16` |
| 删除属性 | 去掉 `tempNote: String` |
| 重命名属性 | `name` → `title`（需设置 Renaming ID） |
| 修改属性类型 | 小范围类型变化（如 Int16 → Int32） |

配置方式——在加载 Store 时添加选项：

```swift
let container = NSPersistentContainer(name: "TodoModel")

let description = container.persistentStoreDescriptions.first
description?.setOption(true as NSNumber,
                      forKey: NSMigratePersistentStoresAutomaticallyOption)
description?.setOption(true as NSNumber,
                      forKey: NSInferMappingModelAutomaticallyOption)

container.loadPersistentStores { desc, error in
    if let error = error {
        fatalError("迁移失败: \(error)")
    }
}
```

### 6.3 Mapping Model（手动映射）

当改动较复杂（类型变化、数据需要转换逻辑）时，需要创建 Mapping Model：

1. File → New → File → Mapping Model
2. 选择源模型版本和目标模型版本
3. Xcode 自动生成默认映射，你可以修改 Entity Mapping 和 Attribute Mapping
4. 添加 Value Expression 进行数据转换

```swift
let container = NSPersistentContainer(name: "TodoModel")
container.loadPersistentStores { desc, error in
    if let error = error {
        fatalError("迁移失败: \(error)")
    }
}
```

> ⚠️ **注意**：使用 Mapping Model 时，不要同时开启 `NSInferMappingModelAutomaticallyOption`，否则会冲突。

### 6.4 迁移策略选择

| 场景 | 策略 |
|------|------|
| 新增/删除属性 | 轻量迁移（自动） |
| 属性重命名 | 轻量迁移 + Renaming ID |
| 属性类型变化（如 String → Int） | Mapping Model |
| 数据需要转换逻辑 | Mapping Model |
| 多版本跨越 | 逐步迁移或自定义迁移策略 |

---

## 7. Core Data 与 SwiftUI 集成

### 7.1 环境注入

在 App 入口将 `NSManagedObjectContext` 注入 SwiftUI 环境：

```swift
import SwiftUI
import CoreData

@main
struct TodoApp: App {
    let persistenceController = PersistenceController.shared

    var body: some Scene {
        WindowGroup {
            ContentView()
                .environment(\.managedObjectContext,
                              persistenceController.container.viewContext)
        }
    }
}

struct PersistenceController {
    static let shared = PersistenceController()

    let container: NSPersistentContainer

    init(inMemory: Bool = false) {
        container = NSPersistentContainer(name: "TodoModel")

        if inMemory {
            container.persistentStoreDescriptions.first?.url = URL(fileURLWithPath: "/dev/null")
        }

        container.loadPersistentStores { description, error in
            if let error = error {
                fatalError("Core Data 初始化失败: \(error)")
            }
        }
    }
}
```

### 7.2 @FetchRequest

`@FetchRequest` 是 Core Data 在 SwiftUI 中的桥梁，自动查询并驱动视图刷新：

```swift
struct TodoListView: View {
    @FetchRequest(
        sortDescriptors: [NSSortDescriptor(keyPath: \TodoItem.createdAt, ascending: false)],
        animation: .default
    )
    private var items: FetchedResults<TodoItem>

    @Environment(\.managedObjectContext) private var viewContext

    var body: some View {
        List {
            ForEach(items) { item in
                HStack {
                    Image(systemName: item.isDone ? "checkmark.circle.fill" : "circle")
                        .foregroundStyle(item.isDone ? .green : .gray)
                        .onTapGesture {
                            item.isDone.toggle()
                            try? viewContext.save()
                        }
                    Text(item.title)
                        .strikethrough(item.isDone)
                }
            }
            .onDelete(perform: deleteItems)
        }
    }

    func deleteItems(at offsets: IndexSet) {
        for index in offsets {
            viewContext.delete(items[index])
        }
        try? viewContext.save()
    }
}
```

### 7.3 带过滤条件的 @FetchRequest

```swift
@FetchRequest(
    sortDescriptors: [NSSortDescriptor(keyPath: \TodoItem.priority, ascending: false)],
    predicate: NSPredicate(format: "isDone == NO"),
    animation: .default
)
private var activeItems: FetchedResults<TodoItem>
```

### 7.4 动态查询

当查询条件需要动态变化时，不能用 `@FetchRequest` 的属性初始化方式，改用 `init` 方法：

```swift
struct FilteredTodoView: View {
    let keyword: String

    @FetchRequest private var items: FetchedResults<TodoItem>

    init(keyword: String) {
        self.keyword = keyword
        _items = FetchRequest<TodoItem>(
            sortDescriptors: [NSSortDescriptor(keyPath: \TodoItem.createdAt, ascending: false)],
            predicate: NSPredicate(format: "title CONTAINS[cd] %@", keyword),
            animation: .default
        )
    }

    var body: some View {
        List(items) { item in
            Text(item.title)
        }
    }
}
```

### 7.5 添加数据的 Sheet

```swift
struct AddTodoView: View {
    @Environment(\.managedObjectContext) private var viewContext
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
                        let todo = TodoItem(context: viewContext)
                        todo.title = title
                        todo.isDone = false
                        todo.createdAt = Date()
                        try? viewContext.save()
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

---

## 8. Core Data + CloudKit 同步

### 8.1 NSPersistentCloudKitContainer

Apple 提供了 `NSPersistentCloudKitContainer`，它是 `NSPersistentContainer` 的子类，自动将 Core Data 数据同步到 iCloud：

```swift
let container = NSPersistentCloudKitContainer(name: "TodoModel")
container.loadPersistentStores { description, error in
    if let error = error {
        fatalError("CloudKit 容器加载失败: \(error)")
    }
}
```

只需把 `NSPersistentContainer` 换成 `NSPersistentCloudKitContainer`，其他代码完全不变！

> 💡 **生活类比**：就像给本地仓库装了一个自动云备份——你在本地做的任何改动，都会悄悄同步到云端，其他设备也能看到。

### 8.2 自动同步配置

```swift
let container = NSPersistentCloudKitContainer(name: "TodoModel")

let storeDescription = container.persistentStoreDescriptions.first
storeDescription?.cloudKitContainerOptions = NSPersistentCloudKitContainerOptions(
    containerIdentifier: "iCloud.com.example.TodoApp"
)
storeDescription?.setOption(true as NSNumber,
                            forKey: NSPersistentHistoryTrackingKey)

container.loadPersistentStores { desc, error in
    if let error = error {
        fatalError("加载失败: \(error)")
    }
}
```

### 8.3 同步前提条件

| 条件 | 说明 |
|------|------|
| 开启 iCloud Capability | Xcode → Signing & Capabilities → iCloud |
| 添加 CloudKit 容器 | 在 iCloud Capability 中勾选 CloudKit |
| 用户登录 iCloud | 设备上必须登录 iCloud 账户 |
| 网络可用 | 同步需要网络连接 |
| 数据模型兼容 | Entity 名称不能含特殊字符，Attribute 类型有限制 |

### 8.4 冲突处理

CloudKit 采用**最后写入胜出（Last Write Wins）**策略：

- 同一记录在多设备同时修改时，最后同步的版本覆盖之前的
- Core Data 通过 `NSPersistentHistoryTracking` 追踪变更时间
- 对于复杂冲突，可以实现自定义合并策略：

```swift
container.persistentStoreDescriptions.first?.setOption(
    NSMergePolicy.mergeByPropertyObjectTrump as NSObject,
    forKey: NSMergePolicyOption
)
```

| 合并策略 | 说明 |
|---------|------|
| `mergeByPropertyStoreTrump` | Store（磁盘）数据优先 |
| `mergeByPropertyObjectTrump` | 内存中的对象优先 |
| `errorMergePolicy` | 冲突时抛出错误（默认） |
| `overwriteMergePolicy` | 内存对象完全覆盖 Store |

> ⚠️ **注意**：CloudKit 同步有延迟（通常几秒到几分钟），不适合实时性要求极高的场景。同步失败时 Core Data 会自动重试，但你需要处理网络不可用时的离线状态。

---

## 9. Core Data vs SwiftData 对比

### 9.1 全面对比表

| 对比项 | Core Data | SwiftData |
|--------|-----------|-----------|
| 模型定义 | .xcdatamodeld 可视化编辑器 | `@Model` 宏，纯 Swift 代码 |
| 查询方式 | NSFetchRequest + NSPredicate | `@Query` + `#Predicate` |
| 上下文操作 | NSManagedObjectContext | ModelContext |
| 存储配置 | NSPersistentContainer | ModelContainer |
| 类型安全 | ❌ 字符串键值，运行时才报错 | ✅ 编译期检查 |
| 代码量 | 多（需手动管理托管对象） | 少（宏自动生成样板代码） |
| 学习曲线 | 陡峭 | 平缓 |
| SwiftUI 集成 | `@FetchRequest`（需手动桥接） | `@Query`（原生支持） |
| 最低系统版本 | iOS 3+ | iOS 17+ |
| 数据迁移 | 轻量迁移 + Mapping Model | 轻量迁移 + VersionedSchema |
| CloudKit 同步 | NSPersistentCloudKitContainer（成熟稳定） | ModelContainer（仍需完善） |
| 并发控制 | 多 Context + 队列，精细控制 | 自动管理，简化 API |
| 调试工具 | SQLDebug、Core Data Instruments | 同样支持 |
| 社区资源 | 丰富（十几年积累） | 逐渐增多 |

### 9.2 何时用哪个

| 场景 | 推荐 | 原因 |
|------|------|------|
| 全新 iOS 17+ 项目 | SwiftData | 代码简洁，开发效率高 |
| 需兼容 iOS 16 及以下 | Core Data | SwiftData 不可用 |
| 已有 Core Data 项目 | 继续用 Core Data | 迁移成本高，收益有限 |
| 需要 CloudKit 同步 | Core Data | 更成熟稳定 |
| 需要精细并发控制 | Core Data | 多 Context 更灵活 |
| 快速原型开发 | SwiftData | 声明式语法更高效 |
| 大型企业级 App | Core Data | 久经考验，可控性强 |

### 9.3 从 Core Data 迁移到 SwiftData 的路径

Apple 提供了渐进式迁移方案：

1. **保持 .xcdatamodeld 文件**：SwiftData 可以直接读取 Core Data 的模型文件
2. **使用 `ModelContainer(for:)` 加载**：传入 Core Data 的 `NSManagedObjectModel`
3. **逐步替换 API**：将 `@FetchRequest` 替换为 `@Query`，`NSManagedObjectContext` 替换为 `ModelContext`
4. **最终移除 .xcdatamodeld**：全部改为 `@Model` 宏定义

```swift
let coreDataModel = NSManagedObjectModel(contentsOf: modelURL)!
let container = try ModelContainer(for: coreDataModel)
```

> ⚠️ **迁移建议**：不要为了追新而迁移。如果现有 Core Data 代码运行良好，没有维护痛点，就继续使用。迁移的收益主要体现在新代码的编写效率上。

---

## 本章小结

| 知识点 | 关键内容 |
|--------|----------|
| Core Data 概述 | 老牌持久化框架，存量项目多，CloudKit 同步成熟 |
| 四层架构 | Model（蓝图）→ Coordinator（协调）→ Context（工作台）→ ManagedObject（数据） |
| 创建数据模型 | .xcdatamodeld 可视化编辑，自动生成 NSManagedObject 子类 |
| CRUD 操作 | 创建用 `init(context:)`，查询用 `NSFetchRequest`，修改后必须 `save()` |
| 数据关系 | 一对一/一对多/多对多，必须设逆关系，注意删除规则 |
| 数据迁移 | 轻量迁移（自动推断）+ Mapping Model（手动映射） |
| SwiftUI 集成 | `@FetchRequest` + 环境注入 `managedObjectContext` |
| CloudKit 同步 | `NSPersistentCloudKitContainer`，自动同步，Last Write Wins |
| vs SwiftData | Core Data 成熟稳定兼容广，SwiftData 简洁现代但限 iOS 17+ |

Core Data 虽然学习曲线陡峭，但它是 iOS 开发的"基本功"——理解了 Core Data，你不仅能维护存量项目，也能更深入地理解 SwiftData 的底层原理。选对工具，事半功倍！🎯
