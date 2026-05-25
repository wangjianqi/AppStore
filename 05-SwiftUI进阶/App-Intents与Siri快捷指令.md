# App Intents 与 Siri 快捷指令

## 本章目标

- 理解 App Intents 的作用和适用场景
- 掌握 AppIntent 协议的定义方式
- 学会创建 App Shortcuts 并用短语触发
- 了解 Entity 与 Query 的数据建模方式
- 掌握 Spotlight 索引的基本用法
- 能为待办清单 App 添加完整的 Siri 快捷指令功能

---

## 1. App Intents 简介

### 什么是 App Intents？

想象一下：你走进一家餐厅，不需要看菜单、不需要跟服务员解释，只需要说一句"老规矩"，服务员就知道你要什么——这就是 **App Intents** 做的事。

App Intents 是 iOS 16+ 引入的框架，它让你的 App 功能可以被 **Siri、快捷指令 App、Spotlight 搜索** 直接调用，用户不需要打开你的 App 就能完成操作。

### 三大入口

| 入口 | 说明 | 举例 |
|------|------|------|
| Siri | 用户语音触发 | "嘿 Siri，添加一个待办" |
| 快捷指令 App | 用户手动创建自动化流程 | 在快捷指令中组合多个操作 |
| Spotlight | 搜索框直接显示 App 操作 | 搜索"待办"出现"添加待办" |

### 支持的系统功能

| 功能 | 最低系统版本 | 说明 |
|------|-------------|------|
| App Intents 框架 | iOS 16 | 核心 API |
| App Shortcuts | iOS 16 | 预定义的快捷指令 |
| Interactive Widgets | iOS 17 | 小组件上直接执行 Intent |
| App Intents 架构重构 | iOS 18 | 更简化的 API |

> 💡 App Intents 是 SiriKit 的继任者，苹果推荐新项目使用 App Intents。SiriKit 仍可用，但不会再有新功能更新。

---

## 2. App Intents vs SiriKit 对比

| 对比项 | App Intents | SiriKit |
|--------|------------|---------|
| 最低系统 | iOS 16 | iOS 10 |
| 定义方式 | Swift 协议 + 结构体 | Intent Definition 文件 (.intentdefinition) |
| 语言 | 纯 Swift，代码即文档 | 需要生成 Swift/ObjC 代码 |
| 自由度 | 任意操作，不限领域 | 仅限苹果预定义的领域（如消息、打车） |
| 快捷指令 | 自动出现在快捷指令 App | 需要捐赠（donate）才出现 |
| Spotlight | 原生支持 | 不支持 |
| 学习曲线 | 较低 | 较高 |
| 维护成本 | 低，代码即配置 | 高，需维护 .intentdefinition 文件 |

> ⚠️ 如果你的 App 只需要 SiriKit 预定义领域的功能（如发消息、打电话），且需要兼容 iOS 16 以下，可以继续用 SiriKit。否则，优先选择 App Intents。

### 生活类比

SiriKit 就像去自助餐厅——你只能选餐厅已经准备好的菜（预定义领域）。App Intents 就像点外卖——你想吃什么就点什么，完全自定义。

---

## 3. 定义 Intent：AppIntent 协议

### 最简 Intent

定义一个 Intent 就像填一张表：告诉系统"我能做什么"和"怎么做"。

```swift
import AppIntents

struct SayHelloIntent: AppIntent {
    static var title: LocalizedStringResource = "说你好"
    static var description = IntentDescription("让 App 说一句你好")

    func perform() async throws -> some IntentResult {
        return .result(dialog: "你好！欢迎使用待办清单！")
    }
}
```

### AppIntent 协议必填项

| 属性/方法 | 类型 | 说明 |
|-----------|------|------|
| `title` | `LocalizedStringResource` | Intent 的名称，用户可见 |
| `perform()` | 异步方法 | 执行具体操作，返回结果 |

### 可选配置

| 属性 | 类型 | 说明 |
|------|------|------|
| `description` | `IntentDescription` | 对 Intent 的详细描述 |
| `openAppWhenRun` | `Bool` | 执行时是否打开 App |
| `isDiscoverable` | `Bool` | 是否可被 Spotlight/快捷指令发现 |

### 带参数的 Intent

```swift
struct AddTodoIntent: AppIntent {
    static var title: LocalizedStringResource = "添加待办"
    static var description = IntentDescription("通过 Siri 或快捷指令添加一条待办事项")
    static var openAppWhenRun = false

    @Parameter(title: "待办内容", description: "要添加的待办事项")
    var content: String

    @Parameter(title: "优先级", default: .normal)
    var priority: TodoPriority

    func perform() async throws -> some IntentResult {
        TodoManager.shared.add(title: content, priority: priority)

        return .result(dialog: "已添加待办：\(content)")
    }
}
```

### @Parameter 详解

`@Parameter` 用来声明 Intent 的输入参数，就像函数的参数列表。

| 属性 | 说明 | 示例 |
|------|------|------|
| `title` | 参数的显示名称 | `"待办内容"` |
| `description` | 参数的描述 | `"要添加的待办事项"` |
| `default` | 默认值 | `.normal` |
| `requestValueDialog` | Siri 询问用户时的提示语 | `"你想添加什么待办？"` |

```swift
@Parameter(
    title: "待办内容",
    requestValueDialog: "你想添加什么待办？"
)
var content: String
```

### 参数类型支持

| 类型 | 说明 | 示例 |
|------|------|------|
| `String` | 文本 | 待办内容 |
| `Int / Double` | 数字 | 数量、金额 |
| `Bool` | 布尔 | 是否提醒 |
| `Date` | 日期 | 截止日期 |
| `URL` | 链接 | 相关链接 |
| `Custom Entity` | 自定义实体 | 选择某个待办项 |
| `Enum` | 枚举 | 优先级选择 |

> 💡 枚举类型需要遵循 `AppEnum` 协议才能在 Intent 中使用，后面会详细讲。

### AppEnum 协议

```swift
enum TodoPriority: String, AppEnum {
    case low
    case normal
    case high

    static var typeDisplayRepresentation = TypeDisplayRepresentation(name: "优先级")
    static var caseDisplayRepresentations: [TodoPriority: DisplayRepresentation] = [
        .low: "低",
        .normal: "普通",
        .high: "高"
    ]
}
```

| 必填项 | 说明 |
|--------|------|
| `typeDisplayRepresentation` | 枚举类型的显示名称 |
| `caseDisplayRepresentations` | 每个枚举值的显示名称 |

### IntentResult 返回值

`perform()` 方法的返回值决定了 Siri 会说什么、快捷指令会得到什么。

```swift
// 只返回对话
return .result(dialog: "已添加待办")

// 返回对话 + 值
return .result(value: newTodo, dialog: "已添加待办：\(newTodo.title)")

// 返回对话 + 值 + 视图
return .result(value: newTodo, dialog: "已添加待办") {
    TodoDetailView(todo: newTodo)
}
```

> ⚠️ `perform()` 是 `async` 方法，意味着你可以在里面执行异步操作（如网络请求），不需要额外的回调。

---

## 4. App Shortcuts：创建快捷指令

### 什么是 App Shortcuts？

App Shortcuts 是你为用户 **预定义好的快捷指令**。用户不需要自己创建，安装 App 后就能在快捷指令 App 中看到。

类比：App Intents 是食材，App Shortcuts 是预制菜——你帮用户搭配好了，用户直接用就行。

### 最简 App Shortcut

```swift
struct AddTodoShortcut: AppShortcut {
    static var intent: AddTodoIntent { AddTodoIntent() }

    static var phrases: [AppShortcutPhrase] = [
        "用\(.applicationName)添加待办",
        "在\(.applicationName)中新建任务"
    ]

    static var shortTitle: LocalizedStringResource = "添加待办"
    static var systemImageName = "plus.circle"
}
```

### AppShortcut 协议必填项

| 属性 | 说明 |
|------|------|
| `intent` | 关联的 Intent |
| `phrases` | 触发短语，Siri 语音识别用 |
| `shortTitle` | 快捷指令的短标题 |
| `systemImageName` | 图标（SF Symbols） |

### 注册 App Shortcuts

在 App 入口处用 `AppShortcutsProvider` 注册所有快捷指令：

```swift
struct TodoShortcuts: AppShortcutsProvider {
    static var appShortcuts: [AppShortcut] = [
        AddTodoShortcut(),
        ViewTodosShortcut()
    ]

    static var shortcutTileColor: ShortcutTileColor = .blue
}
```

然后在 App 的主结构体中声明：

```swift
@main
struct TodoApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
    }
}

// AppIntentsPackage 协议自动发现，无需额外注册
```

> 💡 `shortcutTileColor` 决定快捷指令在快捷指令 App 中的背景色。

### 短语设计建议

| 建议 | 说明 | 好的示例 | 差的示例 |
|------|------|---------|---------|
| 包含 App 名 | 用 `\(.applicationName)` | "用待办清单添加任务" | "添加任务" |
| 自然口语 | 像平时说话 | "帮我记一笔待办" | "执行添加待办 Intent" |
| 提供多个 | 不同用户习惯不同 | 提供 2-3 个短语 | 只提供一个 |
| 简洁 | 不要太长 | ≤ 8 个字 | "请帮我在待办清单应用中添加一条新的待办事项" |

> ⚠️ 短语中的 `\(.applicationName)` 会被替换为你的 App 名称。这是必须的——苹果要求至少一个短语包含 App 名称。

### 视觉展示

快捷指令在快捷指令 App 中会以卡片形式展示：

```swift
struct AddTodoShortcut: AppShortcut {
    static var intent: AddTodoIntent { AddTodoIntent() }

    static var phrases: [AppShortcutPhrase] = [
        "用\(.applicationName)添加待办"
    ]

    static var shortTitle: LocalizedStringResource = "添加待办"
    static var systemImageName = "plus.circle.fill"

    // 可选：自定义背景色
    static var shortcutTileColor: ShortcutTileColor = .green
}
```

---

## 5. Entity 与 Query：数据模型

### 为什么需要 Entity？

当你的 Intent 需要操作 App 中的 **具体数据**（如"选择一个待办项"），就需要 Entity。

类比：Intent 是动词（"删除"），Entity 是名词（"某个待办"），Query 是查找方式（"按标题搜索"）。

### 定义 Entity

```swift
struct TodoEntity: AppEntity {
    var id: UUID
    var title: String
    var isCompleted: Bool
    var priority: TodoPriority

    static var typeDisplayRepresentation = TypeDisplayRepresentation(name: "待办事项")
    static var defaultQuery = TodoEntityQuery()

    var displayRepresentation: DisplayRepresentation {
        DisplayRepresentation(title: "\(title)")
    }
}
```

### AppEntity 协议必填项

| 属性 | 说明 |
|------|------|
| `id` | 唯一标识，需遵循 `Hashable` |
| `typeDisplayRepresentation` | Entity 类型的显示名称 |
| `defaultQuery` | 默认查询，用于搜索和选择 |
| `displayRepresentation` | 单个实例的显示方式 |

### 定义 Query

Query 负责告诉系统如何 **搜索和列出** 你的 Entity：

```swift
struct TodoEntityQuery: EntityQuery {
    func entities(for identifiers: [UUID]) async throws -> [TodoEntity] {
        TodoManager.shared.todos
            .filter { identifiers.contains($0.id) }
            .map { $0.toEntity() }
    }

    func suggestedEntities() async throws -> [TodoEntity] {
        TodoManager.shared.todos
            .map { $0.toEntity() }
    }

    func entities(matching query: String) async throws -> [TodoEntity] {
        TodoManager.shared.todos
            .filter { $0.title.localizedCaseInsensitiveContains(query) }
            .map { $0.toEntity() }
    }
}
```

### EntityQuery 三个核心方法

| 方法 | 说明 | 触发时机 |
|------|------|---------|
| `entities(for:)` | 根据 ID 获取实体 | 快捷指令恢复、Siri 确认操作 |
| `suggestedEntities()` | 返回建议列表 | 用户未输入搜索词时 |
| `entities(matching:)` | 搜索匹配的实体 | 用户输入搜索词时 |

> 💡 三个方法都是 `async` 的，可以做网络请求。但建议优先使用本地数据，Siri 的响应时间有限。

### 在 Intent 中使用 Entity

```swift
struct CompleteTodoIntent: AppIntent {
    static var title: LocalizedStringResource = "完成待办"
    static var description = IntentDescription("将指定的待办标记为已完成")

    @Parameter(title: "待办事项")
    var todo: TodoEntity

    func perform() async throws -> some IntentResult {
        TodoManager.shared.complete(id: todo.id)
        return .result(dialog: "已完成待办：\(todo.title)")
    }
}
```

### EntityStringQuery（更简单的查询）

如果你的 Entity 只需要简单的字符串搜索，可以用 `EntityStringQuery`：

```swift
struct TodoEntityQuery: EntityStringQuery {
    func entities(for identifiers: [UUID]) async throws -> [TodoEntity] {
        TodoManager.shared.todos
            .filter { identifiers.contains($0.id) }
            .map { $0.toEntity() }
    }

    func entities(matching query: String) async throws -> [TodoEntity] {
        TodoManager.shared.todos
            .filter { $0.title.localizedCaseInsensitiveContains(query) }
            .map { $0.toEntity() }
    }
}
```

---

## 6. Spotlight 索引

### 让 App 内容出现在 Spotlight 搜索中

用户在 Spotlight 搜索时，如果你的 App 内容被索引了，就会出现在搜索结果中。点击结果可以直接跳转到 App 对应页面。

类比：Spotlight 索引就像给图书馆的每本书贴上标签，读者搜索关键词就能找到对应的书。

### 使用 CSSearchableItem

```swift
import CoreSpotlight

func indexTodo(_ todo: Todo) {
    let item = CSSearchableItem(
        uniqueIdentifier: todo.id.uuidString,
        domainIdentifier: "com.example.TodoApp.todos",
        attributeSet: attributeSet(for: todo)
    )

    CSSearchableIndex.default().indexSearchableItems([item]) { error in
        if let error = error {
            print("索引失败：\(error)")
        }
    }
}

private func attributeSet(for todo: Todo) -> CSSearchableItemAttributeSet {
    let attributes = CSSearchableItemAttributeSet(contentType: .text)
    attributes.title = todo.title
    attributes.contentDescription = "优先级：\(todo.priority.rawValue) · \(todo.isCompleted ? "已完成" : "未完成")"
    attributes.keywords = ["待办", "任务", todo.title]
    return attributes
}
```

### 删除索引

```swift
func deindexTodo(_ todo: Todo) {
    CSSearchableIndex.default().deleteSearchableItems(
        withIdentifiers: [todo.id.uuidString]
    ) { error in
        if let error = error {
            print("删除索引失败：\(error)")
        }
    }
}
```

### 处理 Spotlight 点击

在 App 的入口处处理用户点击 Spotlight 结果的事件：

```swift
@main
struct TodoApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
                .onContinueUserActivity(
                    CSSearchableItemActionType,
                    perform: { activity in
                        guard let idString = activity.userInfo?[
                            CSSearchableItemActivityIdentifier
                        ] as? String,
                              let id = UUID(uuidString: idString) else {
                            return
                        }
                        // 跳转到对应待办详情
                        NavigationManager.shared.navigateTo(todoID: id)
                    }
                )
        }
    }
}
```

### CSSearchableItemAttributeSet 常用属性

| 属性 | 说明 | 示例 |
|------|------|------|
| `title` | 标题 | "买牛奶" |
| `contentDescription` | 描述 | "优先级：高 · 未完成" |
| `keywords` | 关键词 | ["待办", "购物"] |
| `thumbnailData` | 缩略图 | 待办的图标 |
| `contentType` | 内容类型 | `.text` |

> ⚠️ Spotlight 索引有数量限制，建议只索引最重要的内容。索引过多会影响性能和搜索排名。

### App Intents 自动支持 Spotlight

从 iOS 16 开始，标记为 `isDiscoverable = true`（默认值）的 Intent 会 **自动** 出现在 Spotlight 搜索结果中，无需手动使用 `CSSearchableItem`。

```swift
struct AddTodoIntent: AppIntent {
    static var title: LocalizedStringResource = "添加待办"
    static var isDiscoverable = true  // 默认就是 true

    // ...
}
```

| 方式 | 适用场景 | 是否需要额外代码 |
|------|---------|----------------|
| App Intents 自动发现 | 让 Intent 操作出现在 Spotlight | 不需要，默认开启 |
| CSSearchableItem | 让 App 数据内容出现在 Spotlight | 需要手动索引 |

---

## 7. 实战示例：给待办清单 App 添加 Intent

### 项目结构

```
TodoApp/
├── Models/
│   ├── Todo.swift              // 数据模型
│   └── TodoManager.swift       // 数据管理
├── Intents/
│   ├── AddTodoIntent.swift     // 添加待办 Intent
│   ├── ViewTodosIntent.swift   // 查看待办 Intent
│   ├── CompleteTodoIntent.swift// 完成待办 Intent
│   └── TodoEntity.swift        // 待办 Entity + Query
├── Shortcuts/
│   └── TodoShortcuts.swift     // App Shortcuts 注册
└── Views/
    └── ContentView.swift       // 主界面
```

### 数据模型

```swift
import Foundation

struct Todo: Identifiable, Hashable {
    let id: UUID
    var title: String
    var isCompleted: Bool
    var priority: TodoPriority
    var createdAt: Date

    init(title: String, priority: TodoPriority = .normal) {
        self.id = UUID()
        self.title = title
        self.isCompleted = false
        self.priority = priority
        self.createdAt = Date()
    }

    func toEntity() -> TodoEntity {
        TodoEntity(
            id: id,
            title: title,
            isCompleted: isCompleted,
            priority: priority
        )
    }
}
```

```swift
import Foundation

enum TodoPriority: String, AppEnum {
    case low
    case normal
    case high

    static var typeDisplayRepresentation = TypeDisplayRepresentation(name: "优先级")
    static var caseDisplayRepresentations: [TodoPriority: DisplayRepresentation] = [
        .low: "低优先级",
        .normal: "普通优先级",
        .high: "高优先级"
    ]
}
```

```swift
import Foundation

@MainActor
class TodoManager: ObservableObject {
    static let shared = TodoManager()

    @Published var todos: [Todo] = []

    func add(title: String, priority: TodoPriority = .normal) -> Todo {
        let todo = Todo(title: title, priority: priority)
        todos.append(todo)
        return todo
    }

    func complete(id: UUID) {
        guard let index = todos.firstIndex(where: { $0.id == id }) else { return }
        todos[index].isCompleted = true
    }

    var pendingTodos: [Todo] {
        todos.filter { !$0.isCompleted }
    }
}
```

### Entity + Query

```swift
import AppIntents

struct TodoEntity: AppEntity {
    var id: UUID
    var title: String
    var isCompleted: Bool
    var priority: TodoPriority

    static var typeDisplayRepresentation = TypeDisplayRepresentation(name: "待办事项")
    static var defaultQuery = TodoEntityQuery()

    var displayRepresentation: DisplayRepresentation {
        DisplayRepresentation(title: "\(title)")
    }
}
```

```swift
import AppIntents

struct TodoEntityQuery: EntityQuery {
    func entities(for identifiers: [UUID]) async throws -> [TodoEntity] {
        await TodoManager.shared.todos
            .filter { identifiers.contains($0.id) }
            .map { $0.toEntity() }
    }

    func suggestedEntities() async throws -> [TodoEntity] {
        await TodoManager.shared.todos
            .map { $0.toEntity() }
    }

    func entities(matching query: String) async throws -> [TodoEntity] {
        await TodoManager.shared.todos
            .filter { $0.title.localizedCaseInsensitiveContains(query) }
            .map { $0.toEntity() }
    }
}
```

### Intent 1：添加待办

```swift
import AppIntents

struct AddTodoIntent: AppIntent {
    static var title: LocalizedStringResource = "添加待办"
    static var description = IntentDescription("通过 Siri 或快捷指令添加一条待办事项")
    static var openAppWhenRun = false

    @Parameter(title: "待办内容", requestValueDialog: "你想添加什么待办？")
    var content: String

    @Parameter(title: "优先级", default: TodoPriority.normal)
    var priority: TodoPriority

    func perform() async throws -> some IntentResult & ReturnsValue<TodoEntity> {
        let todo = await TodoManager.shared.add(
            title: content,
            priority: priority
        )
        return .result(
            value: todo.toEntity(),
            dialog: "已添加待办：\(content)"
        )
    }
}
```

### Intent 2：查看待办

```swift
import AppIntents

struct ViewTodosIntent: AppIntent {
    static var title: LocalizedStringResource = "查看待办列表"
    static var description = IntentDescription("查看当前未完成的待办事项")
    static var openAppWhenRun = true

    func perform() async throws -> some IntentResult {
        let pending = await TodoManager.shared.pendingTodos

        if pending.isEmpty {
            return .result(dialog: "没有未完成的待办，好好休息吧！")
        }

        let list = pending.map { "- \($0.title)" }.joined(separator: "\n")
        return .result(dialog: "你有 \(pending.count) 条未完成待办：\n\(list)")
    }
}
```

### Intent 3：完成待办

```swift
import AppIntents

struct CompleteTodoIntent: AppIntent {
    static var title: LocalizedStringResource = "完成待办"
    static var description = IntentDescription("将指定的待办标记为已完成")

    @Parameter(title: "待办事项", requestValueDialog: "你想完成哪个待办？")
    var todo: TodoEntity

    func perform() async throws -> some IntentResult {
        await TodoManager.shared.complete(id: todo.id)
        return .result(dialog: "已完成待办：\(todo.title)")
    }
}
```

### App Shortcuts 注册

```swift
import AppIntents

struct AddTodoShortcut: AppShortcut {
    static var intent: AddTodoIntent { AddTodoIntent() }

    static var phrases: [AppShortcutPhrase] = [
        "用\(.applicationName)添加待办",
        "在\(.applicationName)中新建任务",
        "用\(.applicationName)记一笔待办"
    ]

    static var shortTitle: LocalizedStringResource = "添加待办"
    static var systemImageName = "plus.circle.fill"
}

struct ViewTodosShortcut: AppShortcut {
    static var intent: ViewTodosIntent { ViewTodosIntent() }

    static var phrases: [AppShortcutPhrase] = [
        "用\(.applicationName)查看待办",
        "在\(.applicationName)中看看有什么待办"
    ]

    static var shortTitle: LocalizedStringResource = "查看待办"
    static var systemImageName = "list.bullet"
}

struct CompleteTodoShortcut: AppShortcut {
    static var intent: CompleteTodoIntent { CompleteTodoIntent() }

    static var phrases: [AppShortcutPhrase] = [
        "用\(.applicationName)完成待办"
    ]

    static var shortTitle: LocalizedStringResource = "完成待办"
    static var systemImageName = "checkmark.circle"
}

struct TodoShortcuts: AppShortcutsProvider {
    static var appShortcuts: [AppShortcut] = [
        AddTodoShortcut(),
        ViewTodosShortcut(),
        CompleteTodoShortcut()
    ]

    static var shortcutTileColor: ShortcutTileColor = .blue
}
```

### 主界面

```swift
import SwiftUI

struct ContentView: View {
    @StateObject private var manager = TodoManager.shared

    var body: some View {
        NavigationStack {
            List {
                ForEach(manager.todos) { todo in
                    HStack {
                        Image(systemName: todo.isCompleted
                              ? "checkmark.circle.fill"
                              : "circle")
                            .foregroundColor(todo.isCompleted ? .green : .gray)

                        VStack(alignment: .leading) {
                            Text(todo.title)
                                .strikethrough(todo.isCompleted)
                            Text(todo.priority.rawValue)
                                .font(.caption)
                                .foregroundColor(.secondary)
                        }
                    }
                }
            }
            .navigationTitle("待办清单")
        }
    }
}
```

---

## 8. 调试与测试

### 在快捷指令 App 中测试

1. **编译运行** App 到真机或模拟器
2. 打开系统 **快捷指令** App
3. 点击底部 **"快捷指令"** 标签
4. 找到你的 App 名称，应该能看到你定义的 Shortcuts
5. 点击某个 Shortcut，填写参数，运行

| 检查项 | 说明 |
|--------|------|
| Shortcut 是否出现 | 如果没出现，检查 `AppShortcutsProvider` 是否正确注册 |
| 参数是否正确 | 检查 `@Parameter` 的 `title` 和 `default` |
| 执行结果是否正确 | 检查 `perform()` 的逻辑 |
| 返回对话是否正确 | 检查 `.result(dialog:)` |

### 用 Siri 测试

1. 确保设备已开启 Siri
2. 对着设备说出你定义的短语，如"嘿 Siri，用待办清单添加待办"
3. Siri 会询问参数（如果未提供）
4. 确认执行结果

> ⚠️ Siri 语音识别可能不准确，尤其是中文短语。建议多试几次，并在短语中使用简洁自然的表达。

### 常见问题排查

| 问题 | 可能原因 | 解决方案 |
|------|---------|---------|
| Shortcut 没出现 | Provider 未注册 | 确认 `AppShortcutsProvider` 在 App target 中 |
| 参数显示为英文 | 缺少本地化 | 为 `title` 和 `description` 使用 `LocalizedStringResource` |
| Siri 无法识别短语 | 短语太复杂 | 简化短语，确保包含 App 名称 |
| 执行后无反应 | `perform()` 抛出异常 | 检查异步逻辑，添加 try-catch |
| Entity 列表为空 | Query 返回空数组 | 检查数据源是否正确 |

### 调试技巧

```swift
struct AddTodoIntent: AppIntent {
    // ...

    func perform() async throws -> some IntentResult {
        print("👉 AddTodoIntent 开始执行")
        print("👉 content = \(content)")
        print("👉 priority = \(priority)")

        let todo = await TodoManager.shared.add(
            title: content,
            priority: priority
        )

        print("👉 添加成功：\(todo.id)")
        return .result(value: todo.toEntity(), dialog: "已添加待办：\(content)")
    }
}
```

在 Xcode Console 中可以看到 print 输出，帮助你调试 Intent 的执行流程。

> 💡 也可以在 `perform()` 中设置断点，但 Siri 调试时断点可能不太稳定，建议优先使用 print。

---

## 9. 审核注意事项

### 必须遵守的规则

| 规则 | 说明 |
|------|------|
| 短语必须包含 App 名 | 至少一个短语使用 `\(.applicationName)` |
| 功能必须真实可用 | 不能定义空壳 Intent |
| 不能误导用户 | Intent 的 title 和 description 必须准确描述功能 |
| 不能收集隐私数据 | Intent 不能在后台偷偷收集用户信息 |
| 必须处理错误 | `perform()` 中的异常要妥善处理 |

### 常见被拒原因

| 被拒原因 | 解决方案 |
|---------|---------|
| Intent 功能与描述不符 | 确保 title/description 与实际行为一致 |
| 快捷指令崩溃 | 充分测试，处理所有边界情况 |
| 短语不包含 App 名 | 添加包含 `\(.applicationName)` 的短语 |
| 滥用系统权限 | 只请求必要的权限 |

### 最佳实践

| 实践 | 说明 |
|------|------|
| 提供有意义的对话 | Siri 的回复要自然、有用 |
| 处理空数据 | 列表为空时给出友好提示 |
| 异步操作加超时 | 网络请求不要无限等待 |
| 本地化 | 为所有用户可见字符串提供本地化 |
| 最少权限 | 只请求 Intent 必需的权限 |

> ⚠️ 提交审核前，务必在真机上完整测试所有 Intent 和 Shortcut，包括 Siri 语音触发和快捷指令 App 触发两种方式。

---

## 小结

| 概念 | 一句话总结 |
|------|-----------|
| App Intents | 让 App 功能暴露给 Siri / 快捷指令 / Spotlight 的框架 |
| AppIntent 协议 | 定义一个可执行的操作，包含 title + perform() |
| @Parameter | Intent 的输入参数，Siri 会询问用户 |
| AppEnum | 让自定义枚举可在 Intent 中使用 |
| AppEntity | 让自定义数据类型可在 Intent 中使用 |
| EntityQuery | 告诉系统如何搜索和列出 Entity |
| AppShortcut | 预定义的快捷指令，安装 App 即可用 |
| CSSearchableItem | 让 App 内容出现在 Spotlight 搜索中 |

核心流程：

```
定义 Entity + Query（数据层）
    ↓
定义 Intent（操作层）
    ↓
创建 AppShortcut（展示层）
    ↓
注册到 AppShortcutsProvider
    ↓
用户通过 Siri / 快捷指令 / Spotlight 触发
```

App Intents 是苹果生态的重要方向——随着 Siri 越来越智能，让你的 App 支持 App Intents，就是让用户更容易地使用你的 App。从最简单的 Intent 开始，逐步添加 Entity、Query 和 Shortcut，你的 App 就能和系统深度集成。

← [地图与定位](./地图与定位.md) | [相册与相机](./相册与相机.md) →
