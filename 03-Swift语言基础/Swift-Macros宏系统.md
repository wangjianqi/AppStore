# Swift Macros 宏系统

> 🎯 **本章目标**：理解 Swift 5.9 引入的宏系统，学会使用内置宏简化代码，掌握自定义宏的开发方法，了解宏在 SwiftUI 和 SwiftData 中的应用。

---

## 1. 宏是什么

### 1.1 从生活类比理解宏

宏就像**模板合同**——你不需要每次手写一份完整的合同，只需要填写关键信息，模板会自动帮你生成标准格式的文档。Swift 宏在编译时自动生成代码，让你用简短的声明代替冗长的样板代码。

### 1.2 编译时代码生成 vs 运行时

Swift 宏与运行时代码有本质区别：

| 特性 | 宏（编译时） | 函数/类（运行时） |
|------|------------|-----------------|
| 执行时机 | 编译时 | 程序运行时 |
| 代码可见性 | 展开后成为源代码的一部分 | 作为编译后的二进制存在 |
| 性能开销 | 零运行时开销 | 有函数调用等开销 |
| 调试 | 可以查看展开后的代码 | 正常调试 |
| 灵活性 | 固定模式，编译时确定 | 运行时可动态决定 |

```swift
@Observable
class ViewModel {
    var count = 0
    var name = ""
}
```

上面这段代码在编译时，`@Observable` 宏会自动为 `ViewModel` 生成观察机制所需的所有代码——包括属性变更通知、存储管理等。你不需要手写任何观察逻辑。

### 1.3 宏的工作流程

```
源代码（含宏） → 编译器识别宏 → 调用宏插件（独立进程） → 生成新代码 → 合并到源代码 → 继续编译
```

关键点：
1. **宏插件运行在独立进程中**——不会污染编译器进程
2. **宏只能读取声明信息**——不能读取函数体
3. **宏生成的是代码文本**——最终会被正常编译
4. **宏是幂等的**——相同输入总是产生相同输出

---

## 2. Swift 宏的设计哲学

### 2.1 与 C 宏的区别

| 特性 | C 宏（#define） | Swift 宏 |
|------|---------------|---------|
| 本质 | 文本替换 | 结构化代码生成 |
| 类型安全 | ❌ 无类型检查 | ✅ 生成代码参与类型检查 |
| 调试 | ❌ 难以调试 | ✅ 可查看展开结果 |
| 作用域 | 全局文本替换 | 限定在声明处 |
| 安全性 | 容易产生意外替换 | 独立进程，受沙盒限制 |
| 递归 | 支持（危险） | 不支持 |
| 副作用 | 可能多次求值 | 不会 |

```c
#define MAX(a, b) ((a) > (b) ? (a) : (b))

int x = 1, y = 2;
int z = MAX(x++, y++);
```

上面 C 宏的问题：`x++` 和 `y++` 会被展开后多次求值，导致非预期行为。

Swift 宏不会有这个问题——它是结构化的代码生成，不是文本替换。

### 2.2 Swift 宏的设计原则

1. **类型安全**：宏生成的代码必须通过类型检查
2. **可审计**：开发者可以查看宏展开后的代码
3. **隔离执行**：宏插件在独立进程中运行，不能访问文件系统或网络
4. **声明式**：宏描述"要什么"，而不是"怎么做"
5. **可组合**：多个宏可以同时作用于同一声明

### 2.3 宏的分类体系

```
Swift 宏
├── 自由宏（Freestanding）
│   ├── 表达式宏（Expression）    #colorLiteral(...)
│   ├── 声明宏（Declaration）    #Preview { ... }
│   └── 代码项宏（CodeItem）     #warning(...)
│
└── 附着宏（Attached）
    ├── 附属宏（Accessor）       @ObservationIgnored
    ├── 成员宏（Member）         @Observable
    ├── 属性包装宏（Peer）       @Dependency
    ├── 遵循宏（Conformance）    @Codable
    └── 扩展宏（Extension）      @AddCompletionHandler
```

---

## 3. 内置宏详解

### 3.1 #Preview 宏

`#Preview` 是 SwiftUI 开发中最常用的宏，它简化了预览代码的编写：

**传统方式**：

```swift
struct ContentView_Previews: PreviewProvider {
    static var previews: some View {
        ContentView()
    }
}
```

**使用 #Preview**：

```swift
#Preview {
    ContentView()
}
```

带参数的预览：

```swift
#Preview("浅色模式") {
    ContentView()
        .preferredColorScheme(.light)
}

#Preview("深色模式") {
    ContentView()
        .preferredColorScheme(.dark)
}

#Preview("iPhone 15 Pro") {
    ContentView()
}
```

多个预览：

```swift
struct UserCard_Previews: View {
    let users = [
        User(name: "张三", avatar: "person.fill"),
        User(name: "李四", avatar: "person.crop.circle")
    ]

    var body: some View {
        ScrollView {
            VStack {
                ForEach(users) { user in
                    UserCard(user: user)
                }
            }
        }
    }
}

#Preview("用户卡片") {
    UserCard_Previews()
}
```

💡 **提示**：`#Preview` 是自由声明宏，它会在编译时生成一个遵循 `PreviewProvider` 协议的结构体。

### 3.2 @Observable 宏

`@Observable` 是 Swift 5.9 引入的观察框架的核心宏，用于替代 `ObservableObject`：

**传统方式（Combine）**：

```swift
class ViewModel: ObservableObject {
    @Published var count = 0
    @Published var name = ""
}

struct ContentView: View {
    @ObservedObject var viewModel = ViewModel()

    var body: some View {
        Text("\(viewModel.count)")
    }
}
```

**使用 @Observable**：

```swift
@Observable
class ViewModel {
    var count = 0
    var name = ""
}

struct ContentView: View {
    var viewModel = ViewModel()

    var body: some View {
        Text("\(viewModel.count)")
    }
}
```

`@Observable` 宏自动做了什么：

1. 为每个存储属性生成 getter/setter（使用 `ObservationTracking`）
2. 添加 `$observationRegistrar` 存储属性
3. 生成 `withObservationTracking` 方法
4. 自动在属性变更时触发观察通知

```swift
@Observable
class ViewModel {
    var count = 0
    var name = ""

    @ObservationIgnored
    var cachedData: Data?
}
```

`@ObservationIgnored` 让属性不参与观察机制——适用于缓存、临时数据等不需要触发 UI 更新的属性。

### 3.3 @Model 宏（SwiftData）

`@Model` 是 SwiftData 框架的核心宏，用于定义数据模型：

```swift
import SwiftData

@Model
class TodoItem {
    var title: String
    var isCompleted: Bool
    var createdAt: Date
    var priority: Int

    init(title: String, isCompleted: Bool = false, createdAt: Date = .now, priority: Int = 0) {
        self.title = title
        self.isCompleted = isCompleted
        self.createdAt = createdAt
        self.priority = priority
    }
}
```

`@Model` 宏自动做了什么：

1. 让类遵循 `PersistentModel` 协议
2. 让类遵循 `Observable` 协议
3. 为每个存储属性生成持久化支持
4. 添加 `Schema` 描述信息
5. 生成 `BackingData` 管理

```swift
@Model
class TodoItem {
    #Unique<TodoItem>([\.title])

    var title: String
    var isCompleted: Bool = false
    var category: Category?

    @Relationship(deleteRule: .cascade)
    var subtasks: [SubTask] = []
}
```

### 3.4 @Codable 宏

Swift 5.10 引入了 `@Codable` 宏，用于自动生成 `Codable` 实现：

```swift
@Codable
struct User: Codable {
    let name: String
    let age: Int
    let email: String?
}
```

`@Codable` 宏支持自定义键名和默认值：

```swift
@Codable
struct APIResponse {
    @CodingKey("user_name")
    var name: String

    @CodingKey("user_age")
    var age: Int

    @Default("unknown")
    var email: String
}
```

### 3.5 其他内置宏

| 宏 | 类型 | 用途 |
|----|------|------|
| `#Preview` | 自由声明宏 | SwiftUI 预览 |
| `@Observable` | 附着成员宏 | 观察机制 |
| `@Model` | 附着成员宏 | SwiftData 模型 |
| `@Codable` | 附着成员宏 | 自动 Codable |
| `#colorLiteral` | 自由表达式宏 | 颜色字面量 |
| `#imageLiteral` | 自由表达式宏 | 图片字面量 |
| `#fileLiteral` | 自由表达式宏 | 文件字面量 |
| `#selector` | 自由表达式宏 | Objective-C 选择器 |
| `#keyPath` | 自由表达式宏 | 键路径 |
| `#warning` | 自由代码项宏 | 编译警告 |
| `#error` | 自由代码项宏 | 编译错误 |
| `#sourceLocation` | 自由代码项宏 | 源码位置 |
| `@TestSuite` | 附着宏 | Swift Testing |
| `@Test` | 附着宏 | Swift Testing |

```swift
#warning("TODO: 处理边界情况")

#if DEBUG
#error("请在 Release 配置中编译")
#endif
```

---

## 4. 附着宏（Attached Macros）

### 4.1 附着宏概述

附着宏附加在某个声明上，根据声明的信息生成代码。它们用 `@` 前缀标记：

```swift
@MemberMacro
struct MyStruct {
    var property: String
}
```

### 4.2 成员宏（Member Macro）

成员宏为类型添加新的成员声明：

```swift
@attached(member, names: named(init), named(allItems))
macro EquatableRepresentation() = #externalMacro(module: "MyMacros", type: "EquatableRepresentationMacro")
```

`@Observable` 就是成员宏——它为类添加观察相关的成员。

### 4.3 附属宏（Accessor Macro）

附属宏为属性添加自定义访问器（getter/setter）：

```swift
@attached(accessor)
macro UserDefault(key: String) = #externalMacro(module: "MyMacros", type: "UserDefaultMacro")

class Settings {
    @UserDefault(key: "theme")
    var theme: String = "light"
}
```

展开后可能生成：

```swift
class Settings {
    var theme: String {
        get {
            UserDefaults.standard.string(forKey: "theme") ?? "light"
        }
        set {
            UserDefaults.standard.set(newValue, forKey: "theme")
        }
    }
}
```

### 4.4 属性包装宏（Peer Macro）

属性包装宏在声明旁边生成新的声明：

```swift
@attached(peer, names: overloaded)
macro Dependency<T>() = #externalMacro(module: "MyMacros", type: "DependencyMacro")

class ViewController {
    @Dependency
    var apiClient: APIClient
}
```

### 4.5 遵循宏（Conformance Macro）

遵循宏为类型添加协议遵循：

```swift
@attached(conformance)
macro Codable() = #externalMacro(module: "MyMacros", type: "CodableMacro")

@Codable
struct User {
    var name: String
    var age: Int
}
```

### 4.6 扩展宏（Extension Macro）

扩展宏为类型添加扩展：

```swift
@attached(extension, conformances: CaseIterable, names: named(allCases))
macro AllCases() = #externalMacro(module: "MyMacros", type: "AllCasesMacro")

@AllCases
enum Direction {
    case up, down, left, right
}
```

### 4.7 附着宏的组合

一个宏可以同时具备多种附着能力：

```swift
@attached(member, names: named(init))
@attached(conformance)
macro DictionaryCodable() = #externalMacro(module: "MyMacros", type: "DictionaryCodableMacro")
```

`@Observable` 实际上是一个组合宏：

```swift
@attached(member, names: named(_$observationRegistrar), named(access))
@attached(memberAttribute)
@attached(conformance)
public macro Observable() = #externalMacro(module: "Observation", type: "ObservableMacro")
```

---

## 5. 自由宏（Freestanding Macros）

### 5.1 自由宏概述

自由宏不依附于任何声明，独立使用，用 `#` 前缀标记：

```swift
#Preview {
    ContentView()
}
```

### 5.2 表达式宏（Expression Macro）

表达式宏生成一个表达式，可以用在任何需要表达式的地方：

```swift
@freestanding(expression)
macro Stringify<T>(_ value: T) -> String = #externalMacro(module: "MyMacros", type: "StringifyMacro")

let result = #Stringify(1 + 2)
```

展开后：

```swift
let result = ("1 + 2", 1 + 2)
```

### 5.3 声明宏（Declaration Macro）

声明宏生成声明，如结构体、类、函数等：

```swift
@freestanding(declaration, names: named(Foo))
macro CreateFoo() = #externalMacro(module: "MyMacros", type: "CreateFooMacro")

#CreateFoo
```

### 5.4 代码项宏（Code Item Macro）

代码项宏生成可以在函数体或类型体中使用的代码项：

```swift
@freestanding(codeItem)
macro AddLog(_ message: String) = #externalMacro(module: "MyMacros", type: "AddLogMacro")

func process() {
    #AddLog("开始处理")
}
```

---

## 6. 开发自定义宏

### 6.1 宏项目结构

开发自定义宏需要一个独立的 Swift Package，因为宏插件运行在独立进程中：

```
MyMacroPackage/
├── Package.swift
├── Sources/
│   ├── MyMacros/          ← 宏实现（编译为宏插件）
│   │   └── StringifyMacro.swift
│   └── MyMacroClient/     ← 使用宏的代码
│       └── main.swift
└── Tests/
    └── MyMacroTests/
        └── StringifyMacroTests.swift
```

### 6.2 Package.swift 配置

```swift
let package = Package(
    name: "MyMacroPackage",
    platforms: [.macOS(.v10_15), .iOS(.v13)],
    products: [
        .library(name: "MyMacros", targets: ["MyMacros"]),
    ],
    dependencies: [
        .package(url: "https://github.com/swiftlang/swift-syntax", from: "509.0.0"),
    ],
    targets: [
        .macro(
            name: "MyMacroPlugin",
            dependencies: [
                .product(name: "SwiftSyntaxMacros", package: "swift-syntax"),
                .product(name: "SwiftCompilerPlugin", package: "swift-syntax"),
            ]
        ),
        .target(name: "MyMacros", dependencies: ["MyMacroPlugin"]),
        .testTarget(
            name: "MyMacroTests",
            dependencies: [
                "MyMacroPlugin",
                .product(name: "SwiftSyntaxMacrosTestSupport", package: "swift-syntax"),
            ]
        ),
    ]
)
```

⚠️ **警告**：宏插件目标（`.macro`）会被编译为独立的可执行文件，运行在编译器的子进程中。不要在宏插件中引入重型依赖。

### 6.3 实现表达式宏：#Stringify

**宏声明（在 MyMacros 模块中）**：

```swift
@freestanding(expression)
public macro Stringify<T>(_ value: T) -> (T, String) = #externalMacro(
    module: "MyMacroPlugin",
    type: "StringifyMacro"
)
```

**宏实现（在 MyMacroPlugin 模块中）**：

```swift
import SwiftSyntaxMacros
import SwiftSyntax

public struct StringifyMacro: ExpressionMacro {
    public static func expansion(
        of node: some FreestandingMacroExpansionSyntax,
        in context: some MacroExpansionContext
    ) -> ExprSyntax {
        guard let argument = node.arguments.first?.expression else {
            fatalError("Stringify 宏需要一个参数")
        }

        return "(\(argument), \(literal: argument.description))"
    }
}
```

**注册宏插件**：

```swift
import SwiftCompilerPlugin

@main
struct MyMacroPlugin: CompilerPlugin {
    let providingMacros: [Macro.Type] = [
        StringifyMacro.self,
    ]
}
```

**使用宏**：

```swift
import MyMacros

let (value, description) = #Stringify(1 + 2)
print(value)
print(description)
```

### 6.4 实现成员宏：@AddCompletionHandler

这个宏为异步函数自动添加完成处理器版本：

**宏声明**：

```swift
@attached(member, names: arbitrary)
public macro AddCompletionHandler() = #externalMacro(
    module: "MyMacroPlugin",
    type: "AddCompletionHandlerMacro"
)
```

**宏实现**：

```swift
public struct AddCompletionHandlerMacro: MemberMacro {
    public static func expansion(
        of node: AttributeSyntax,
        providingMembersOf declaration: some DeclGroupSyntax,
        in context: some MacroExpansionContext
    ) -> [DeclSyntax] {
        let methods = declaration.memberBlock.members.compactMap { member -> FunctionDeclSyntax? in
            guard let function = member.decl.as(FunctionDeclSyntax.self),
                  function.signature.effectSpecifiers?.asyncClause != nil else {
                return nil
            }
            return function
        }

        return methods.map { method in
            let name = method.name.text
            let parameters = method.signature.parameterClause.parameters
            let returnType = method.signature.returnClause?.type

            var completionHandlerParams: [String] = []
            for param in parameters {
                let firstName = param.firstName.text
                let type = param.type.description
                completionHandlerParams.append("\(firstName): \(type)")
            }

            if let returnType = returnType {
                completionHandlerParams.append("completion: @escaping (\(returnType)) -> Void")
            } else {
                completionHandlerParams.append("completion: @escaping () -> Void")
            }

            let paramList = completionHandlerParams.joined(separator: ", ")

            var callArgs: [String] = []
            for param in parameters {
                let firstName = param.firstName.text
                callArgs.append("\(firstName): \(firstName)")
            }
            let callArgList = callArgs.joined(separator: ", ")

            let body: String
            if returnType != nil {
                body = """
                Task {
                    let result = await \(name)(\(callArgList))
                    completion(result)
                }
                """
            } else {
                body = """
                Task {
                    await \(name)(\(callArgList))
                    completion()
                }
                """
            }

            return """
            func \(name)Completion(\(paramList)) {
                \(body)
            }
            """
        }
    }
}
```

**使用宏**：

```swift
@AddCompletionHandler
class APIClient {
    func fetchUser(id: String) async -> User {
    }
}
```

展开后自动生成：

```swift
class APIClient {
    func fetchUser(id: String) async -> User {
    }

    func fetchUserCompletion(id: String, completion: @escaping (User) -> Void) {
        Task {
            let result = await fetchUser(id: id)
            completion(result)
        }
    }
}
```

### 6.5 实现附属宏：@UserDefault

**宏声明**：

```swift
@attached(accessor)
public macro UserDefault<T>(key: String, defaultValue: T) = #externalMacro(
    module: "MyMacroPlugin",
    type: "UserDefaultMacro"
)
```

**宏实现**：

```swift
public struct UserDefaultMacro: AccessorMacro {
    public static func expansion(
        of node: AttributeSyntax,
        providingAccessorsOf declaration: some DeclSyntaxProtocol,
        in context: some MacroExpansionContext
    ) -> [AccessorDeclSyntax] {
        guard let varDecl = declaration.as(VariableDeclSyntax.self),
              let binding = varDecl.bindings.first,
              let identifier = binding.pattern.as(IdentifierPatternSyntax.self)?.identifier.text else {
            return []
        }

        let keyExpr = node.arguments?.first?.expression
        let defaultExpr = node.arguments?.dropFirst().first?.expression

        return [
            """
            get {
                UserDefaults.standard.object(forKey: \(keyExpr!)) as? T ?? \(defaultExpr!)
            }
            """,
            """
            set {
                UserDefaults.standard.set(newValue, forKey: \(keyExpr!))
            }
            """
        ]
    }
}
```

**使用宏**：

```swift
class AppSettings {
    @UserDefault(key: "app_theme", defaultValue: "light")
    var theme: String

    @UserDefault(key: "launch_count", defaultValue: 0)
    var launchCount: Int
}
```

### 6.6 实现遵循宏：@DictionaryCodable

```swift
@attached(conformance)
@attached(member)
public macro DictionaryCodable() = #externalMacro(
    module: "MyMacroPlugin",
    type: "DictionaryCodableMacro"
)
```

这个宏让结构体自动支持从/到字典的编解码：

```swift
@DictionaryCodable
struct Config {
    var apiKey: String
    var timeout: Double
    var retries: Int
}

let config = Config(from: ["apiKey": "abc", "timeout": 30.0, "retries": 3])
let dict = config.toDictionary()
```

### 6.7 宏类型总结

| 宏类型 | 协议 | 生成内容 | 示例 |
|--------|------|---------|------|
| ExpressionMacro | `ExpressionMacro` | 表达式 | `#Stringify(...)` |
| DeclarationMacro | `DeclarationMacro` | 声明 | `#Preview { ... }` |
| CodeItemMacro | `CodeItemMacro` | 代码项 | `#warning(...)` |
| AccessorMacro | `AccessorMacro` | 属性访问器 | `@UserDefault` |
| MemberMacro | `MemberMacro` | 类型成员 | `@Observable` |
| MemberAttributeMacro | `MemberAttributeMacro` | 成员属性 | `@Observable` |
| PeerMacro | `PeerMacro` | 伴随声明 | `@Dependency` |
| ConformanceMacro | `ConformanceMacro` | 协议遵循 | `@Codable` |
| ExtensionMacro | `ExtensionMacro` | 扩展 | `@AddEquatable` |

---

## 7. 宏在 SwiftUI 中的应用

### 7.1 #Preview 简化预览

`#Preview` 是 SwiftUI 开发中使用频率最高的宏：

```swift
struct WeatherView: View {
    let temperature: Double
    let condition: String

    var body: some View {
        VStack {
            Text("\(Int(temperature))°")
                .font(.system(size: 72, weight: .bold))
            Text(condition)
                .font(.title2)
        }
    }
}

#Preview("晴天") {
    WeatherView(temperature: 28, condition: "☀️ 晴")
}

#Preview("雨天") {
    WeatherView(temperature: 15, condition: "🌧️ 雨")
}

#Preview("雪天") {
    WeatherView(temperature: -5, condition: "❄️ 雪")
}
```

### 7.2 @Observable 替代 ObservableObject

```swift
import SwiftUI
import Observation

@Observable
class TodoListViewModel {
    var items: [TodoItem] = []
    var filter: FilterType = .all
    var isLoading = false

    var filteredItems: [TodoItem] {
        switch filter {
        case .all: items
        case .completed: items.filter(\.isCompleted)
        case .pending: items.filter { !$0.isCompleted }
        }
    }

    func loadItems() async {
        isLoading = true
        defer { isLoading = false }
        items = await apiClient.fetchItems()
    }
}

struct TodoListView: View {
    @State private var viewModel = TodoListViewModel()

    var body: some View {
        Group {
            if viewModel.isLoading {
                ProgressView()
            } else {
                List(viewModel.filteredItems) { item in
                    TodoRow(item: item)
                }
            }
        }
        .task {
            await viewModel.loadItems()
        }
    }
}
```

💡 **提示**：使用 `@Observable` 后，SwiftUI 会自动追踪 `body` 中实际访问的属性，只有这些属性变化时才会重新渲染——比 `@Published` 更精确。

### 7.3 @Observable 与 @State

```swift
struct ContentView: View {
    @State private var viewModel = TodoListViewModel()

    var body: some View {
        TodoListView(viewModel: viewModel)
    }
}
```

SwiftUI 5.0+ 中，`@State` 可以直接用于 `@Observable` 对象，不再需要 `@StateObject`。

### 7.4 @Bindable 宏

`@Bindable` 不是一个宏，但与 `@Observable` 配合使用：

```swift
struct EditTodoView: View {
    @Bindable var todo: TodoItem

    var body: some View {
        Form {
            TextField("标题", text: $todo.title)
            Toggle("已完成", isOn: $todo.isCompleted)
        }
    }
}
```

`@Bindable` 为 `@Observable` 对象创建绑定，让你可以用 `$` 前缀创建双向绑定。

### 7.5 宏与 View Modifier

可以创建宏来简化常见的 View Modifier 模式：

```swift
@freestanding(expression)
macro StyledCard<Content: View>(_ content: Content) -> some View = #externalMacro(
    module: "MyMacroPlugin",
    type: "StyledCardMacro"
)
```

---

## 8. 宏在 SwiftData 中的应用

### 8.1 @Model 宏详解

```swift
import SwiftData

@Model
class Article {
    var title: String
    var content: String
    var publishedAt: Date
    var isDraft: Bool
    var tags: [String]

    var author: Author?
    @Relationship(deleteRule: .cascade, inverse: \Comment.article)
    var comments: [Comment] = []

    init(title: String, content: String, publishedAt: Date = .now, isDraft: Bool = true, tags: [String] = []) {
        self.title = title
        self.content = content
        self.publishedAt = publishedAt
        self.isDraft = isDraft
        self.tags = tags
    }
}

@Model
class Author {
    var name: String
    var articles: [Article] = []

    init(name: String) {
        self.name = name
    }
}

@Model
class Comment {
    var text: String
    var createdAt: Date
    var article: Article?

    init(text: String, createdAt: Date = .now) {
        self.text = text
        self.createdAt = createdAt
    }
}
```

### 8.2 @Model 宏生成的代码

`@Model` 宏在编译时自动生成以下内容：

1. **PersistentModel 协议遵循**：让类型可以被 SwiftData 管理
2. **Observable 协议遵循**：属性变更自动触发 UI 更新
3. **持久化属性访问器**：每个属性的 getter/setter 都与 SwiftData 的 `BackingData` 关联
4. **Schema 元数据**：描述模型结构的元信息
5. **初始化器支持**：支持从 `BackingData` 初始化

### 8.3 @Attribute 和 @Relationship

```swift
@Model
class Product {
    @Attribute(.unique)
    var sku: String

    var name: String
    var price: Double

    @Attribute(.transformable(by: "NSSecureUnarchiveFromData"))
    var metadata: [String: Any]?

    @Relationship(deleteRule: .cascade)
    var variants: [ProductVariant] = []

    @Relationship(deleteRule: .nullify, inverse: \Category.products)
    var category: Category?
}
```

| 删除规则 | 说明 |
|---------|------|
| `.cascade` | 删除父对象时，自动删除关联对象 |
| `.nullify` | 删除父对象时，将关联对象的引用设为 nil |
| `.deny` | 如果有关联对象存在，拒绝删除父对象 |
| `.noAction` | 不做任何操作（需手动处理） |

### 8.4 @Transient

```swift
@Model
class User {
    var name: String
    var email: String

    @Transient
    var temporaryCache: [String: Any] = [:]
}
```

`@Transient` 标记的属性不会被持久化到数据库中。

### 8.5 SwiftData 模型迁移

```swift
import SwiftData

@Model
class User {
    var name: String
    var email: String

    var age: Int

    init(name: String, email: String, age: Int = 0) {
        self.name = name
        self.email = email
        self.age = age
    }
}
```

SwiftData 支持轻量级迁移——添加新属性时，如果提供了默认值，迁移会自动进行。

⚠️ **警告**：更复杂的迁移（如重命名属性、合并模型）需要手动编写迁移计划。

---

## 9. 宏的调试和排错

### 9.1 查看宏展开结果

在 Xcode 中查看宏展开后的代码：

1. 右键点击使用了宏的代码
2. 选择 **Expand Macro**（展开宏）
3. Xcode 会显示宏生成的完整代码

也可以通过命令行查看：

```bash
swift build -Xswiftc -dump-macro-expansions
```

这会在构建输出中打印所有宏的展开结果。

### 9.2 常见编译错误

| 错误信息 | 原因 | 解决方案 |
|---------|------|---------|
| `macro expansion failed` | 宏实现中发生错误 | 检查宏实现的逻辑 |
| `external macro implementation type 'XXX' not found` | 宏插件未正确链接 | 检查 Package.swift 中的模块名和类型名 |
| `macro 'XXX' is not attached to a valid declaration` | 宏用在了不支持的声明上 | 检查宏的 `@attached` 规则 |
| `ambiguous use of macro` | 多个同名宏冲突 | 使用完整模块名限定 |

### 9.3 宏插件的调试

宏插件是独立的可执行文件，可以单独调试：

1. 在 Xcode 中，为宏插件目标添加断点
2. 在使用宏的 target 的 Build Settings 中，设置 `SWIFT_EXEC` 为宏插件的可执行文件路径
3. 或者使用 `lldb` 直接附加到宏插件进程

更实用的调试方法——在宏实现中添加诊断信息：

```swift
public struct MyMacro: MemberMacro {
    public static func expansion(
        of node: AttributeSyntax,
        providingMembersOf declaration: some DeclGroupSyntax,
        in context: some MacroExpansionContext
    ) -> [DeclSyntax] {
        context.diagnose(Diagnostic(
            node: node,
            message: MacroDiagnostic.debug("正在处理：\(declaration.description)")
        ))

        return []
    }
}
```

### 9.4 宏测试

使用 `SwiftSyntaxMacrosTestSupport` 测试宏：

```swift
import SwiftSyntaxMacrosTestSupport
import XCTest
@testable import MyMacroPlugin

final class StringifyMacroTests: XCTestCase {
    let testMacros: [String: Macro.Type] = [
        "Stringify": StringifyMacro.self,
    ]

    func testStringify() {
        assertMacroExpansion(
            """
            let (value, description) = #Stringify(1 + 2)
            """,
            expandedSource: """
            let (value, description) = (1 + 2, "1 + 2")
            """,
            macros: testMacros
        )
    }

    func testStringifyWithString() {
        assertMacroExpansion(
            """
            let result = #Stringify("hello")
            """,
            expandedSource: """
            let result = ("hello", "\\"hello\\"")
            """,
            macros: testMacros
        )
    }
}
```

💡 **提示**：宏测试是保证宏正确性的关键。每次修改宏实现后都应该运行测试，确保展开结果符合预期。

---

## 10. 宏的局限性和注意事项

### 10.1 宏的局限性

| 局限 | 说明 |
|------|------|
| 不能读取函数体 | 宏只能看到声明信息，不能看到函数的实现代码 |
| 不能递归 | 宏不能在展开结果中再使用自身 |
| 不能跨文件 | 宏只能看到当前声明所在文件的部分信息 |
| 编译时间增加 | 宏插件是独立进程，启动和通信有开销 |
| 调试困难 | 宏展开后的代码不在源文件中，调试体验不如手写代码 |
| 生态不成熟 | 自定义宏的开发工具和最佳实践仍在发展中 |

### 10.2 性能影响

宏的编译时间开销主要来自：

1. **宏插件进程启动**：每个宏插件需要启动一个独立进程
2. **进程间通信**：编译器与宏插件之间的序列化/反序列化
3. **代码生成**：宏实现本身的计算时间

优化建议：

```swift
@Observable
class ViewModel {
    var count = 0
    var name = ""
}
```

- 避免在一个文件中使用大量宏
- 宏插件应该尽量轻量
- 使用 `swift-syntax` 的缓存机制

### 10.3 宏与代码可读性

宏虽然减少了样板代码，但也带来了可读性挑战：

```swift
@Observable
@Model
@Codable
class User {
    var name: String
    var age: Int
}
```

这段代码看起来很简洁，但三个宏加在一起可能生成了数百行代码。开发者需要理解每个宏做了什么。

**建议**：
- 在团队中建立宏的使用规范
- 为自定义宏编写详细文档
- 使用 Xcode 的"Expand Macro"功能查看生成代码
- 不要过度使用宏——如果手写代码更清晰，就不要用宏

### 10.4 宏与版本兼容性

| Swift 版本 | 宏支持 |
|-----------|--------|
| Swift 5.9 | 宏系统首次引入，支持基本宏类型 |
| Swift 5.10 | 改进宏诊断，新增 `@Codable` |
| Swift 6.0 | 增强宏能力，改进宏插件性能 |

⚠️ **警告**：宏依赖 `swift-syntax` 库，该库的版本必须与 Swift 编译器版本匹配。升级 Xcode 时可能需要同步更新 `swift-syntax` 依赖。

### 10.5 宏 vs 其他代码生成方式

| 方式 | 优点 | 缺点 |
|------|------|------|
| Swift 宏 | 类型安全、编译时检查、IDE 集成 | 学习曲线陡、调试困难 |
| Sourcery | 灵活、模板语法简单 | 外部工具、需额外构建步骤 |
| Gyro | 专为 Core Data 设计 | 局限性大 |
| 手写代码 | 完全可控、易调试 | 样板代码多、维护成本高 |

💡 **提示**：对于简单的代码生成需求（如 Equatable、Hashable），优先使用 Swift 宏。对于复杂的跨文件代码生成，Sourcery 可能更合适。

---

## 11. AI 辅助宏开发

### 11.1 AI 辅助场景

| 场景 | AI 提示词示例 |
|------|-------------|
| 理解宏展开 | "帮我展开 @Observable 宏，看看它生成了什么代码" |
| 编写宏声明 | "帮我声明一个成员宏，为结构体自动生成单例模式" |
| 实现宏逻辑 | "帮我实现一个 AccessorMacro，为属性添加线程安全的访问器" |
| 调试宏错误 | "我的宏编译报错 'external macro not found'，怎么排查？" |
| 宏测试 | "帮我为 StringifyMacro 编写测试用例" |

### 11.2 AI 辅助实现自定义宏

**需求**：实现一个 `@Singleton` 宏，自动为类生成单例模式。

**AI 生成的宏声明**：

```swift
@attached(member, names: named(shared), named(init))
public macro Singleton() = #externalMacro(
    module: "MyMacroPlugin",
    type: "SingletonMacro"
)
```

**AI 生成的宏实现**：

```swift
public struct SingletonMacro: MemberMacro {
    public static func expansion(
        of node: AttributeSyntax,
        providingMembersOf declaration: some DeclGroupSyntax,
        in context: some MacroExpansionContext
    ) -> [DeclSyntax] {
        let className = declaration.as(ClassDeclSyntax.self)?.name.text ?? "Unknown"

        return [
            """
            static let shared = \(raw: className)()
            """,
            """
            private override init() {}
            """
        ]
    }
}
```

**使用宏**：

```swift
@Singleton
class DatabaseManager {
    func execute(_ query: String) -> [Row] {
        return []
    }
}

let db = DatabaseManager.shared
db.execute("SELECT * FROM users")
```

### 11.3 AI 辅助宏调试

当宏出现问题时，可以请 AI 帮忙：

1. **提供错误信息**：将完整的编译错误发给 AI
2. **提供宏代码**：包括声明和实现
3. **提供使用代码**：展示如何使用宏的

AI 可以帮你：
- 检查宏声明与实现是否匹配
- 检查 `swift-syntax` API 使用是否正确
- 分析展开结果是否符合预期
- 建议修复方案

### 11.4 AI 辅助宏迁移

从旧模式迁移到宏模式：

```swift
// 旧模式：手写 ObservableObject
class OldViewModel: ObservableObject {
    @Published var count = 0
    @Published var name = ""
}

// AI 建议的新模式：使用 @Observable
@Observable
class NewViewModel {
    var count = 0
    var name = ""
}
```

💡 **提示**：AI 可以批量帮你将 `ObservableObject` + `@Published` 迁移为 `@Observable`，将 `PreviewProvider` 迁移为 `#Preview`。

---

## 本章小结

| 主题 | 核心要点 | 关键语法 |
|------|---------|---------|
| 宏的本质 | 编译时代码生成，类型安全 | `@Observable`, `#Preview` |
| 设计哲学 | 与 C 宏不同，结构化、安全、可审计 | 独立进程、沙盒执行 |
| 内置宏 | #Preview, @Observable, @Model, @Codable | `#Preview { ... }`, `@Model class` |
| 附着宏 | 附加在声明上生成代码 | `@attached(member)`, `@attached(accessor)` |
| 自由宏 | 独立使用，生成表达式或声明 | `@freestanding(expression)`, `#macroName` |
| 自定义宏 | 需要 Swift Package，宏插件独立进程 | `.macro` target, `CompilerPlugin` |
| SwiftUI 应用 | #Preview 简化预览，@Observable 替代 ObservableObject | `@State` + `@Observable` |
| SwiftData 应用 | @Model 定义数据模型，自动持久化 | `@Model`, `@Relationship`, `@Transient` |
| 调试 | Expand Macro 查看展开结果，宏测试 | `assertMacroExpansion` |
| 局限性 | 不能读函数体、不能递归、编译时间增加 | 合理使用，不过度依赖 |

> 💡 **学习建议**：宏系统是 Swift 5.9+ 最重要的新特性之一。建议从使用内置宏（`#Preview`、`@Observable`、`@Model`）开始，逐步理解宏的工作原理。当你发现项目中反复出现相同的样板代码模式时，再考虑开发自定义宏。记住：**宏的目的是消除重复，而不是炫技**。

← [可选值 Optional 深入](./可选值Optional深入.md) | [Property Wrappers 与 Result Builder](./Property-Wrappers与Result-Builder.md) →
