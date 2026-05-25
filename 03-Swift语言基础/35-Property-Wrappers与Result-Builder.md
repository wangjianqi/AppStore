# 35-Property Wrappers 与 Result Builder

> 🎯 **本章目标**：深入理解 SwiftUI 声明式语法的底层原理，掌握 Property Wrappers 封装属性逻辑的方法，学会用 Result Builder 构建自定义 DSL。

---

## 1. Property Wrappers 原理

### 1.1 从生活类比理解 Property Wrapper

Property Wrapper 就像**智能插座**——你把普通电器插上去，智能插座就自动给它加上了定时开关、电量统计、远程控制等功能。电器本身不需要任何改造，但通过智能插座的包装，它获得了额外的能力。

Property Wrapper 把属性的"存储逻辑"和"访问逻辑"分离开来。你只需要声明属性的类型，存储和访问的细节由 Wrapper 负责。

### 1.2 为什么需要 Property Wrapper

在 Property Wrapper 出现之前，如果你想让多个属性共享同一种逻辑（比如延迟加载、线程安全、值验证），你需要为每个属性重复编写相同的代码：

```swift
struct User {
    private var _name: String = ""
    var name: String {
        get { _name.trimmingCharacters(in: .whitespaces) }
        set { _name = newValue.trimmingCharacters(in: .whitespaces) }
    }

    private var _email: String = ""
    var email: String {
        get { _email.trimmingCharacters(in: .whitespaces) }
        set { _email = newValue.trimmingCharacters(in: .whitespaces) }
    }

    private var _phone: String = ""
    var phone: String {
        get { _phone.trimmingCharacters(in: .whitespaces) }
        set { _phone = newValue.trimmingCharacters(in: .whitespaces) }
    }
}
```

使用 Property Wrapper 后：

```swift
@propertyWrapper
struct Trimmed {
    private var value: String = ""

    var wrappedValue: String {
        get { value }
        set { value = newValue.trimmingCharacters(in: .whitespaces) }
    }

    init(wrappedValue: String) {
        self.value = wrappedValue.trimmingCharacters(in: .whitespaces)
    }
}

struct User {
    @Trimmed var name: String = ""
    @Trimmed var email: String = ""
    @Trimmed var phone: String = ""
}
```

### 1.3 wrappedValue：核心接口

`wrappedValue` 是 Property Wrapper 的核心——它定义了被包装属性的读写行为：

```swift
@propertyWrapper
struct Clamped<Value: Comparable> {
    private var value: Value
    let range: ClosedRange<Value>

    var wrappedValue: Value {
        get { value }
        set { value = min(max(newValue, range.lowerBound), range.upperBound) }
    }

    init(wrappedValue: Value, _ range: ClosedRange<Value>) {
        self.range = range
        self.value = min(max(wrappedValue, range.lowerBound), range.upperBound)
    }
}
```

使用：

```swift
struct GameSettings {
    @Clamped(0...100) var volume: Int = 50
    @Clamped(0...10) var difficulty: Int = 5
}

var settings = GameSettings()
settings.volume = 150
print(settings.volume)

settings.difficulty = -3
print(settings.difficulty)
```

### 1.4 projectedValue：投影值

`projectedValue` 是 Property Wrapper 的第二个接口，通过 `$` 前缀访问。它提供了对 Wrapper 自身的引用，而不仅仅是包装的值：

```swift
@propertyWrapper
struct Validated<Value> {
    private var value: Value
    let validator: (Value) -> Bool

    var wrappedValue: Value {
        get { value }
        set {
            if validator(newValue) {
                value = newValue
            }
        }
    }

    var projectedValue: Bool {
        validator(value)
    }

    init(wrappedValue: Value, validator: @escaping (Value) -> Bool) {
        self.validator = validator
        self.value = validator(wrappedValue) ? wrappedValue : value
    }
}
```

使用：

```swift
struct FormInput {
    @Validated(validator: { $0.count >= 6 })
    var password: String = ""

    @Validated(validator: { $0.contains("@") })
    var email: String = ""
}

var input = FormInput()
input.password = "123"
print(input.password)
print($input.password)

input.password = "123456"
print(input.password)
print($input.password)
```

💡 **提示**：`$` 前缀访问的是 `projectedValue`，不是"可选值解包"。在 SwiftUI 中，`$` 前缀通常用于创建 Binding。

### 1.5 Property Wrapper 的编译器变换

当你写下：

```swift
struct User {
    @Trimmed var name: String = ""
}
```

编译器实际生成的是：

```swift
struct User {
    private var _name = Trimmed(wrappedValue: "")

    var name: String {
        get { _name.wrappedValue }
        set { _name.wrappedValue = newValue }
    }

    var $name: Trimmed {
        get { _name }
    }
}
```

⚠️ **警告**：编译器生成的存储属性名是 `_` + 原属性名。因此，在你的代码中不要使用 `_name` 这样的命名，以免与编译器生成的属性冲突。

### 1.6 Property Wrapper 的初始化

Property Wrapper 支持多种初始化方式：

```swift
@propertyWrapper
struct Delayed<T> {
    private var initializer: (() -> T)?
    private var _value: T?

    var wrappedValue: T {
        mutating get {
            if let value = _value {
                return value
            }
            let value = initializer!()
            _value = value
            initializer = nil
            return value
        }
        set {
            _value = newValue
            initializer = nil
        }
    }

    init(wrappedValue: @autoclosure @escaping () -> T) {
        self.initializer = wrappedValue
    }
}

struct ExpensiveObject {
    @Delayed var data: Data = loadData()
}
```

---

## 2. SwiftUI 中的 Property Wrappers

### 2.1 SwiftUI Property Wrappers 总览

| Property Wrapper | 用途 | 值类型 | 投影值（$） | 数据流向 |
|-----------------|------|--------|-----------|---------|
| `@State` | 视图本地状态 | 值类型 | `Binding<T>` | 视图内部 |
| `@Binding` | 引用外部状态 | 值类型 | `Binding<T>` | 父→子 |
| `@ObservedObject` | 观察外部对象 | 引用类型 | `Binding<T>` | 外部→视图 |
| `@StateObject` | 拥有并观察对象 | 引用类型 | `Binding<T>` | 视图拥有 |
| `@EnvironmentObject` | 从环境获取对象 | 引用类型 | `Binding<T>` | 环境→视图 |
| `@Environment` | 从环境读取值 | 任意类型 | — | 环境→视图 |
| `@AppStorage` | UserDefaults 包装 | 基本类型 | `Binding<T>` | 持久化 |
| `@FetchRequest` | Core Data 查询 | FetchedResults | `Binding<T>` | 数据库→视图 |
| `@Query` | SwiftData 查询 | [PersistentModel] | — | 数据库→视图 |
| `@FocusState` | 焦点状态 | 哈希类型 | `Binding<T>` | 视图内部 |
| `@ScaledMetric` | 动态字体缩放 | 数值类型 | — | 系统→视图 |
| `@SectionedFetchRequest` | 分组查询 | SectionedFetchResults | — | 数据库→视图 |
| `@Namespace` | 动画命名空间 | Namespace.ID | — | 视图内部 |
| `@GestureState` | 手势临时状态 | 值类型 | `GestureState<T>` | 手势→视图 |
| `@Observable` | Swift 5.9+ 观察 | 引用类型 | — | 对象→视图 |

### 2.2 @State 深入

`@State` 是 SwiftUI 最基础的 Property Wrapper，用于管理视图的本地状态：

```swift
struct CounterView: View {
    @State private var count = 0

    var body: some View {
        VStack {
            Text("计数：\(count)")
                .font(.title)
            HStack {
                Button("减少") { count -= 1 }
                Button("增加") { count += 1 }
            }
        }
    }
}
```

`@State` 的底层原理：

```swift
@propertyWrapper
struct State<Value> {
    var wrappedValue: Value { get nonmutating set }
    var projectedValue: Binding<Value> { get }
}
```

关键特性：
- **nonmutating set**：修改 `@State` 变量不需要 `mutating`，因为值存储在 SwiftUI 框架管理的特殊存储区
- **投影值为 Binding**：`$count` 返回 `Binding<Int>`，可以传递给子视图
- **值类型专用**：`@State` 适用于 `Int`、`String`、`Bool` 等值类型
- **视图拥有**：`@State` 的生命周期与视图绑定

⚠️ **警告**：`@State` 应该总是标记为 `private`。它的值由 SwiftUI 管理，不应该从外部直接访问。

### 2.3 @Binding 深入

`@Binding` 创建对状态的引用，不拥有数据，只是"借用"父视图的状态：

```swift
struct ToggleView: View {
    @Binding var isOn: Bool

    var body: some View {
        Toggle("开关", isOn: $isOn)
    }
}

struct ParentView: View {
    @State private var isEnabled = false

    var body: some View {
        VStack {
            Text(isEnabled ? "已开启" : "已关闭")
            ToggleView(isOn: $isEnabled)
        }
    }
}
```

`@Binding` 的底层原理：

```swift
@propertyWrapper
struct Binding<Value> {
    var wrappedValue: Value { get nonmutating set }
    var projectedValue: Binding<Value> { get }

    init(get: @escaping () -> Value, set: @escaping (Value) -> Void)
}
```

`Binding` 本质上是一对 getter/setter 闭包——它不存储值，只是提供了读写值的通道。

**自定义 Binding**：

```swift
struct FormView: View {
    @State private var username = ""

    var body: some View {
        TextField("用户名", text: Binding(
            get: { username },
            set: { newValue in
                username = newValue.filter { $0.isLetter || $0.isNumber }
            }
        ))
    }
}
```

### 2.4 @ObservedObject vs @StateObject

| 特性 | @ObservedObject | @StateObject |
|------|----------------|-------------|
| 所有权 | 不拥有，引用外部对象 | 拥有，负责创建和生命周期 |
| 初始化 | 外部传入 | 视图内部创建 |
| 重建时 | 可能被重新创建 | 保持不变 |
| 适用场景 | 父视图传入的对象 | 视图自己管理的对象 |

```swift
class UserManager: ObservableObject {
    @Published var currentUser: User?
    @Published var isLoggedIn = false
}

struct LoginView: View {
    @StateObject private var userManager = UserManager()

    var body: some View {
        if userManager.isLoggedIn {
            HomeView(userManager: userManager)
        } else {
            LoginFormView(userManager: userManager)
        }
    }
}

struct HomeView: View {
    @ObservedObject var userManager: UserManager

    var body: some View {
        Text("欢迎，\(userManager.currentUser?.name ?? "用户")")
    }
}
```

⚠️ **警告**：在视图中创建 ObservableObject 时，必须使用 `@StateObject` 而非 `@ObservedObject`。使用 `@ObservedObject` 创建对象会导致视图重建时对象被重新创建。

### 2.5 @EnvironmentObject

`@EnvironmentObject` 从 SwiftUI 环境中获取共享对象：

```swift
class ThemeManager: ObservableObject {
    @Published var isDarkMode = false
    @Published var accentColor: Color = .blue
}

struct MyApp: App {
    @StateObject private var themeManager = ThemeManager()

    var body: some Scene {
        WindowGroup {
            ContentView()
                .environmentObject(themeManager)
        }
    }
}

struct SettingsView: View {
    @EnvironmentObject var themeManager: ThemeManager

    var body: some View {
        Toggle("深色模式", isOn: $themeManager.isDarkMode)
    }
}
```

### 2.6 @Environment

`@Environment` 读取 SwiftUI 环境中的值（不需要 ObservableObject）：

```swift
struct ResponsiveView: View {
    @Environment(\.colorScheme) var colorScheme
    @Environment(\.horizontalSizeClass) var sizeClass
    @Environment(\.locale) var locale
    @Environment(\.dismiss) var dismiss

    var body: some View {
        VStack {
            Text(colorScheme == .dark ? "深色模式" : "浅色模式")
            Text(sizeClass == .compact ? "紧凑布局" : "常规布局")
            Text("当前语言：\(locale.identifier)")
            Button("关闭") { dismiss() }
        }
    }
}
```

**自定义 Environment 值**：

```swift
enum AppTheme {
    case standard, compact, accessible
}

private struct AppThemeKey: EnvironmentKey {
    static let defaultValue: AppTheme = .standard
}

extension EnvironmentValues {
    var appTheme: AppTheme {
        get { self[AppThemeKey.self] }
        set { self[AppThemeKey.self] = newValue }
    }
}

struct ThemedView: View {
    @Environment(\.appTheme) var theme

    var body: some View {
        Text("当前主题")
            .font(theme == .compact ? .caption : .body)
    }
}
```

### 2.7 @AppStorage

`@AppStorage` 是 UserDefaults 的 Property Wrapper 封装：

```swift
struct SettingsView: View {
    @AppStorage("username") var username: String = ""
    @AppStorage("launchCount") var launchCount: Int = 0
    @AppStorage("isDarkMode") var isDarkMode: Bool = false
    @AppStorage("fontSize") var fontSize: Double = 14.0
    @AppStorage("lastOpenDate") var lastOpenDate: Date?

    var body: some View {
        Form {
            TextField("用户名", text: $username)
            Stepper("启动次数：\(launchCount)", value: $launchCount)
            Toggle("深色模式", isOn: $isDarkMode)
            Slider(value: $fontSize, in: 10...24, step: 1) {
                Text("字体大小：\(Int(fontSize))")
            }
        }
    }
}
```

`@AppStorage` 支持的类型：

| 类型 | UserDefaults 存储 |
|------|-----------------|
| `String` | ✅ |
| `Int` | ✅ |
| `Double` | ✅ |
| `Bool` | ✅ |
| `URL` | ✅ |
| `Data` | ✅ |
| `Date` | 需要自定义 |
| `RawRepresentable` | ✅（枚举） |
| `Codable` | 需要扩展 |

**自定义 @AppStorage 支持 Codable**：

```swift
extension AppStorage {
    init(wrappedValue: Value, _ key: String, store: UserDefaults? = nil) where Value: Codable {
        let data = try? JSONEncoder().encode(wrappedValue)
        let stringValue = data.flatMap { String(data: $0, encoding: .utf8) }
        self.init(wrappedValue: wrappedValue, key, store: store)
    }
}
```

### 2.8 @FetchRequest（Core Data）

```swift
struct TodoListView: View {
    @FetchRequest(
        sortDescriptors: [SortDescriptor(\.createdAt, order: .reverse)],
        predicate: #Predicate<TodoItem> { $0.isCompleted == false },
        animation: .default
    )
    var items: FetchedResults<TodoItem>

    var body: some View {
        List(items) { item in
            Text(item.title ?? "")
        }
    }
}
```

### 2.9 @Query（SwiftData）

```swift
import SwiftData

struct ArticleListView: View {
    @Query(sort: \Article.publishedAt, order: .reverse)
    var articles: [Article]

    @Query(filter: #Predicate<Article> { $0.isDraft == false })
    var publishedArticles: [Article]

    var body: some View {
        List(articles) { article in
            VStack(alignment: .leading) {
                Text(article.title)
                    .font(.headline)
                Text(article.publishedAt.formatted(date: .abbreviated, time: .shortened))
                    .font(.caption)
                    .foregroundStyle(.secondary)
            }
        }
    }
}
```

### 2.10 @FocusState

```swift
struct LoginFormView: View {
    @FocusState private var focusedField: Field?
    @State private var username = ""
    @State private var password = ""

    enum Field: Hashable {
        case username, password
    }

    var body: some View {
        Form {
            TextField("用户名", text: $username)
                .focused($focusedField, equals: .username)

            SecureField("密码", text: $password)
                .focused($focusedField, equals: .password)

            Button("登录") {
                if username.isEmpty {
                    focusedField = .username
                } else if password.isEmpty {
                    focusedField = .password
                } else {
                    login()
                }
            }
        }
        .onAppear {
            focusedField = .username
        }
    }

    func login() {}
}
```

### 2.11 SwiftUI Property Wrapper 的演进

| Swift/SwiftUI 版本 | 新增 Property Wrapper |
|-------------------|---------------------|
| SwiftUI 1.0 (iOS 13) | @State, @Binding, @ObservedObject, @StateObject, @EnvironmentObject, @Environment |
| SwiftUI 2.0 (iOS 14) | @AppStorage, @ScaledMetric, @GestureState, @FocusedValue |
| SwiftUI 3.0 (iOS 15) | @FocusState, @Namespace 改进 |
| SwiftUI 4.0 (iOS 16) | @FetchRequest 改进, @SectionedFetchRequest |
| SwiftUI 5.0 (iOS 17) | @Bindable, @Query (SwiftData), @Observable |
| SwiftUI 5.1 (iOS 18) | @Entry (Environment 改进) |

💡 **提示**：SwiftUI 5.0+ 推荐使用 `@Observable` + `@State`/`@Bindable` 替代 `ObservableObject` + `@StateObject`/`@ObservedObject`。新方案更简洁、性能更好。

---

## 3. 自定义 Property Wrapper

### 3.1 实现步骤

1. 创建一个标记了 `@propertyWrapper` 的结构体或类
2. 实现 `wrappedValue` 属性
3. 可选：实现 `projectedValue` 属性
4. 实现 `init(wrappedValue:)` 初始化器

### 3.2 实战：线程安全的属性

```swift
@propertyWrapper
struct ThreadSafe<Value> {
    private var value: Value
    private let lock = NSLock()

    var wrappedValue: Value {
        get {
            lock.lock()
            defer { lock.unlock() }
            return value
        }
        set {
            lock.lock()
            defer { lock.unlock() }
            value = newValue
        }
    }

    init(wrappedValue: Value) {
        self.value = wrappedValue
    }
}

class DataCache {
    @ThreadSafe var items: [String: Data] = [:]
    @ThreadSafe var hitCount: Int = 0
}
```

### 3.3 实战：值验证

```swift
@propertyWrapper
struct Validated<Value> {
    private var value: Value
    private let rules: [(Value) -> Bool]
    private let errorMessage: String

    var wrappedValue: Value {
        get { value }
        set {
            if rules.allSatisfy({ $0(newValue) }) {
                value = newValue
            }
        }
    }

    var projectedValue: Bool {
        rules.allSatisfy { $0(value) }
    }

    init(wrappedValue: Value, rules: [(Value) -> Bool], errorMessage: String = "验证失败") {
        self.rules = rules
        self.errorMessage = errorMessage
        self.value = rules.allSatisfy({ $0(wrappedValue) }) ? wrappedValue : value
    }
}

struct RegistrationForm {
    @Validated(
        rules: [{ $0.count >= 8 }, { $0.contains(where: { $0.isNumber }) }, { $0.contains(where: { $0.isLetter }) }],
        errorMessage: "密码至少8位，需包含字母和数字"
    )
    var password: String = ""

    @Validated(
        rules: [{ $0.contains("@") && $0.contains(".") }],
        errorMessage: "邮箱格式不正确"
    )
    var email: String = ""
}
```

### 3.4 实战：自动通知变更

```swift
@propertyWrapper
struct ChangeLogged<Value: Equatable> {
    private var value: Value
    private var changeCount = 0
    private var lastChanged: Date?

    var wrappedValue: Value {
        get { value }
        set {
            if newValue != value {
                value = newValue
                changeCount += 1
                lastChanged = Date()
            }
        }
    }

    var projectedValue: ChangeLog {
        ChangeLog(changeCount: changeCount, lastChanged: lastChanged)
    }

    struct ChangeLog {
        let changeCount: Int
        let lastChanged: Date?
    }

    init(wrappedValue: Value) {
        self.value = wrappedValue
    }
}

struct Document {
    @ChangeLogged var title: String = ""
    @ChangeLogged var content: String = ""
}

var doc = Document()
doc.title = "第一章"
doc.title = "第二章"
doc.title = "第二章"
print(doc.$title.changeCount)
print(doc.$title.lastChanged!)
```

### 3.5 实战：UserDefaults 封装

```swift
@propertyWrapper
struct UserDefault<T> {
    let key: String
    let defaultValue: T
    let store: UserDefaults

    var wrappedValue: T {
        get { store.object(forKey: key) as? T ?? defaultValue }
        set { store.set(newValue, forKey: key) }
    }

    init(key: String, defaultValue: T, store: UserDefaults = .standard) {
        self.key = key
        self.defaultValue = defaultValue
        self.store = store
    }
}

extension UserDefault where T: Codable {
    init(key: String, defaultValue: T, store: UserDefaults = .standard) {
        self.key = key
        self.defaultValue = defaultValue
        self.store = store
    }

    var wrappedValue: T {
        get {
            if let data = store.data(forKey: key),
               let decoded = try? JSONDecoder().decode(T.self, from: data) {
                return decoded
            }
            return defaultValue
        }
        set {
            if let encoded = try? JSONEncoder().encode(newValue) {
                store.set(encoded, forKey: key)
            }
        }
    }
}

class AppSettings: ObservableObject {
    @UserDefault(key: "has_seen_onboarding", defaultValue: false)
    var hasSeenOnboarding: Bool

    @UserDefault(key: "app_theme", defaultValue: "system")
    var appTheme: String

    @UserDefault(key: "recent_searches", defaultValue: [])
    var recentSearches: [String]
}
```

### 3.6 实战：缓存属性

```swift
@propertyWrapper
struct Cached<Value> {
    private var value: Value?
    private var expirationDate: Date?
    private let ttl: TimeInterval
    private let loader: () -> Value

    var wrappedValue: Value {
        mutating get {
            if let value = value, let expiration = expirationDate, Date() < expiration {
                return value
            }
            let newValue = loader()
            self.value = newValue
            self.expirationDate = Date().addingTimeInterval(ttl)
            return newValue
        }
    }

    init(ttl: TimeInterval, loader: @escaping () -> Value) {
        self.ttl = ttl
        self.loader = loader
    }

    mutating func invalidate() {
        value = nil
        expirationDate = nil
    }
}

struct APIClient {
    @Cached(ttl: 300) var userProfile: UserProfile = {
        return fetchUserProfileFromNetwork()
    }
}
```

### 3.7 Property Wrapper 与泛型约束

```swift
@propertyWrapper
struct Sorted<ArrayElement: Comparable> {
    private var array: [ArrayElement]

    var wrappedValue: [ArrayElement] {
        get { array }
        set { array = newValue.sorted() }
    }

    init(wrappedValue: [ArrayElement]) {
        self.array = wrappedValue.sorted()
    }
}

struct Leaderboard {
    @Sorted var scores: [Int] = []
    @Sorted var names: [String] = []
}

var board = Leaderboard()
board.scores = [95, 72, 88, 100, 65]
print(board.scores)
```

### 3.8 Property Wrapper 的限制

| 限制 | 说明 |
|------|------|
| 不能参与协议遵循 | 包装后的类型与原始类型不同 |
| 不能用 `lazy` | Property Wrapper 本身处理延迟逻辑 |
| 不能用于计算属性 | 只能用于存储属性 |
| 投影值类型固定 | 一个 Wrapper 只能有一个 projectedValue 类型 |
| 不能嵌套 | `@WrapperA @WrapperB var x` 不支持 |
| 初始化顺序 | 多个 Property Wrapper 的初始化顺序不确定 |

---

## 4. Property Wrapper 与 $ 前缀

### 4.1 $ 的三种含义

在 Swift 中，`$` 前缀有三种不同的含义：

| 上下文 | 含义 | 示例 |
|--------|------|------|
| Property Wrapper | 访问 projectedValue | `$isOn` → `Binding<Bool>` |
| 闭包参数 | 匿名参数 | `$0`, `$1` |
| 键路径 | 引用属性路径 | `\.name` 等价于 `\Type.name` |

### 4.2 SwiftUI 中的 $ 与 Binding

在 SwiftUI 中，`$` 最常见的用途是从 `@State`、`@Binding` 等创建 `Binding`：

```swift
struct ContentView: View {
    @State private var text = ""
    @State private var isOn = false

    var body: some View {
        VStack {
            TextField("输入", text: $text)
            Toggle("开关", isOn: $isOn)
        }
    }
}
```

`$text` 的类型是 `Binding<String>`，它提供了对 `text` 属性的读写引用。

### 4.3 自定义投影值类型

投影值不一定是 `Binding`，可以是任何类型：

```swift
@propertyWrapper
struct Formatted<Value> {
    private var value: Value
    let formatter: (Value) -> String

    var wrappedValue: Value {
        get { value }
        set { value = newValue }
    }

    var projectedValue: String {
        formatter(value)
    }

    init(wrappedValue: Value, formatter: @escaping (Value) -> String) {
        self.value = wrappedValue
        self.formatter = formatter
    }
}

struct Invoice {
    @Formatted(formatter: { String(format: "¥%.2f", $0) })
    var amount: Double = 0

    @Formatted(formatter: { $0.uppercased() })
    var currency: String = "CNY"
}

var invoice = Invoice()
invoice.amount = 1234.5
print(invoice.amount)
print(invoice.$amount)

invoice.currency = "usd"
print(invoice.currency)
print(invoice.$currency)
```

### 4.4 投影值与观察模式

```swift
@propertyWrapper
struct Observed<Value: Equatable> {
    private var value: Value
    private var observers: [(Value, Value) -> Void] = []

    var wrappedValue: Value {
        get { value }
        set {
            let oldValue = value
            if newValue != oldValue {
                value = newValue
                observers.forEach { $0(oldValue, newValue) }
            }
        }
    }

    var projectedValue: Observed<Value> { self }

    mutating func observe(_ handler: @escaping (Value, Value) -> Void) {
        observers.append(handler)
    }

    init(wrappedValue: Value) {
        self.value = wrappedValue
    }
}

struct ShoppingCart {
    @Observed var itemCount: Int = 0
}

var cart = ShoppingCart()
cart.$itemCount.observe { old, new in
    print("商品数量从 \(old) 变为 \(new)")
}
cart.itemCount = 3
cart.itemCount = 5
```

---

## 5. Result Builder 原理

### 5.1 从生活类比理解 Result Builder

Result Builder 就像**乐高积木的组装台**——你把各种形状的积木放在台上，组装台会按照规则自动把它们拼接成一个完整的作品。在 SwiftUI 中，你把各种 View 放在 `VStack` 里，`ViewBuilder` 会自动把它们组合成一个复合 View。

### 5.2 Result Builder 是什么

Result Builder（原名 Function Builder）是 Swift 5.4 引入的特性，它允许你定义一种特殊的上下文，在这个上下文中，语句的执行结果会被自动收集和组合。

```swift
@resultBuilder
struct StringBuilder {
    static func buildBlock(_ components: String...) -> String {
        components.joined(separator: "\n")
    }
}

func buildMessage(@StringBuilder content: () -> String) -> String {
    content()
}

let message = buildMessage {
    "尊敬的用户："
    "您的订单已发货。"
    "预计3天内送达。"
    "感谢您的购买！"
}
print(message)
```

### 5.3 Result Builder 的核心方法

| 方法 | 用途 | 对应语法 |
|------|------|---------|
| `buildBlock(_:)` | 组合一组结果 | 顺序语句 |
| `buildOptional(_:)` | 处理可选结果 | `if` 无 `else` |
| `buildEither(first:)` | 处理 if 分支 | `if` - `else` |
| `buildEither(second:)` | 处理 else 分支 | `if` - `else` |
| `buildArray(_:)` | 处理循环结果 | `for` 循环 |
| `buildLimitedAvailability(_:)` | 处理可用性检查 | `if #available` |
| `buildExpression(_:)` | 转换单个表达式 | 表达式类型转换 |
| `buildFinalResult(_:)` | 转换最终结果 | 返回值类型转换 |

### 5.4 buildBlock：组合结果

```swift
@resultBuilder
struct TupleBuilder {
    static func buildBlock<each Content>(_ content: repeat each Content) -> (repeat each Content) {
        (repeat each content)
    }
}
```

### 5.5 buildExpression：类型转换

```swift
@resultBuilder
struct NumberBuilder {
    static func buildExpression(_ expression: Int) -> [Int] {
        [expression]
    }

    static func buildExpression(_ expression: Double) -> [Int] {
        [Int(expression)]
    }

    static func buildBlock(_ components: [Int]...) -> [Int] {
        components.flatMap { $0 }
    }
}

let numbers = NumberBuilder.buildBlock {
    1
    2.5
    3
    4.7
}
print(numbers)
```

### 5.6 buildOptional 和 buildEither：条件逻辑

```swift
@resultBuilder
struct ConditionalBuilder {
    static func buildBlock(_ components: String...) -> String {
        components.joined(separator: ", ")
    }

    static func buildOptional(_ component: String?) -> String {
        component ?? "无"
    }

    static func buildEither(first component: String) -> String {
        "✅ \(component)"
    }

    static func buildEither(second component: String) -> String {
        "❌ \(component)"
    }
}

func buildStatus(isSuccess: Bool) -> String {
    ConditionalBuilder.buildBlock {
        "状态报告"
        if isSuccess {
            "操作成功"
        } else {
            "操作失败"
        }
    }
}
```

### 5.7 buildArray：循环支持

```swift
@resultBuilder
struct ListBuilder {
    static func buildBlock(_ components: [String]...) -> [String] {
        components.flatMap { $0 }
    }

    static func buildExpression(_ expression: String) -> [String] {
        [expression]
    }

    static func buildArray(_ components: [[String]]) -> [String] {
        components.flatMap { $0 }
    }
}

func buildList(@ListBuilder content: () -> [String]) -> [String] {
    content()
}

let items = buildList {
    "第一项"
    for i in 2...5 {
        "第\(i)项"
    }
}
print(items)
```

---

## 6. SwiftUI 的 ViewBuilder 详解

### 6.1 ViewBuilder 的定义

`ViewBuilder` 是 SwiftUI 中最核心的 Result Builder，它让你可以在闭包中写多个 View，自动组合成一个复合 View：

```swift
@resultBuilder
struct ViewBuilder {
    static func buildBlock() -> EmptyView

    static func buildBlock<Content: View>(_ content: Content) -> Content

    static func buildBlock<C0: View, C1: View>(_ c0: C0, _ c1: C1) -> TupleView<(C0, C1)>

    static func buildBlock<C0: View, C1: View, C2: View>(_ c0: C0, _ c1: C1, _ c2: C2) -> TupleView<(C0, C1, C2)>

    // ... 最多支持 10 个子视图
}
```

### 6.2 ViewBuilder 如何工作

当你写下：

```swift
VStack {
    Text("标题")
    Text("副标题")
    Image(systemName: "star.fill")
}
```

编译器将其转换为：

```swift
VStack {
    ViewBuilder.buildBlock(
        Text("标题"),
        Text("副标题"),
        Image(systemName: "star.fill")
    )
}
```

返回类型是 `TupleView<(Text, Text, Image)>`——一个包含三个视图的元组视图。

### 6.3 ViewBuilder 与条件渲染

```swift
VStack {
    if isLoggedIn {
        Text("欢迎回来")
    } else {
        Button("登录") { login() }
    }
}
```

编译器转换为：

```swift
VStack {
    ViewBuilder.buildEither(
        first: ViewBuilder.buildBlock(Text("欢迎回来")),
        second: ViewBuilder.buildBlock(Button("登录") { login() })
    )
}
```

### 6.4 ViewBuilder 的 10 子视图限制

⚠️ **警告**：`ViewBuilder` 的 `buildBlock` 最多支持 10 个子视图。超过 10 个会编译报错。

**解决方案一：使用 Group**

```swift
VStack {
    Group {
        Text("1")
        Text("2")
        Text("3")
        Text("4")
        Text("5")
        Text("6")
        Text("7")
        Text("8")
        Text("9")
        Text("10")
    }
    Text("11")
}
```

**解决方案二：使用 ForEach**

```swift
VStack {
    ForEach(1...15, id: \.self) { i in
        Text("第 \(i) 项")
    }
}
```

**解决方案三：提取子视图**

```swift
struct HeaderView: View {
    var body: some View {
        VStack {
            Text("标题1")
            Text("标题2")
            Text("标题3")
        }
    }
}

struct ContentView: View {
    var body: some View {
        VStack {
            HeaderView()
            Text("内容1")
            Text("内容2")
        }
    }
}
```

💡 **提示**：提取子视图是最佳实践——不仅解决 10 子视图限制，还提高了代码的可读性和复用性。

### 6.5 ViewBuilder 与泛型

```swift
struct CardView<Content: View>: View {
    let content: Content

    init(@ViewBuilder content: () -> Content) {
        self.content = content()
    }

    var body: some View {
        content
            .padding()
            .background(Color.white)
            .cornerRadius(12)
            .shadow(radius: 4)
    }
}

CardView {
    Text("卡片标题")
        .font(.headline)
    Text("卡片内容")
        .font(.body)
}
```

### 6.6 ViewBuilder 与 some View

```swift
struct MyView: View {
    var body: some View {
        VStack {
            Text("Hello")
        }
    }
}
```

`some View` 是不透明返回类型——编译器知道具体的返回类型（如 `VStack<Text>`），但外部只能看到 `View` 协议。这保证了静态派发和编译时优化。

---

## 7. 自定义 Result Builder

### 7.1 实现步骤

1. 创建一个标记了 `@resultBuilder` 的结构体
2. 实现 `buildBlock` 方法（必需）
3. 可选：实现其他 build 方法

### 7.2 实战：HTML 构建器

```swift
@resultBuilder
struct HTMLBuilder {
    static func buildBlock(_ components: HTMLElement...) -> HTMLElement {
        HTMLElement(tag: "div", children: components)
    }

    static func buildEither(first component: HTMLElement) -> HTMLElement {
        component
    }

    static func buildEither(second component: HTMLElement) -> HTMLElement {
        component
    }

    static func buildOptional(_ component: HTMLElement?) -> HTMLElement {
        component ?? HTMLElement(tag: "span", text: "")
    }

    static func buildArray(_ components: [HTMLElement]) -> HTMLElement {
        HTMLElement(tag: "div", children: components)
    }
}

struct HTMLElement {
    let tag: String
    var text: String? = nil
    var attributes: [String: String] = [:]
    var children: [HTMLElement] = []

    func render() -> String {
        let attrs = attributes.map { " \($0.key)=\"\($0.value)\"" }.joined()
        if let text = text {
            return "<\(tag)\(attrs)>\(text)</\(tag)>"
        }
        let inner = children.map { $0.render() }.joined()
        return "<\(tag)\(attrs)>\(inner)</\(tag)>"
    }
}

func h1(_ text: String) -> HTMLElement {
    HTMLElement(tag: "h1", text: text)
}

func p(_ text: String) -> HTMLElement {
    HTMLElement(tag: "p", text: text)
}

func a(href: String, _ text: String) -> HTMLElement {
    HTMLElement(tag: "a", text: text, attributes: ["href": href])
}

func div(@HTMLBuilder content: () -> HTMLElement) -> HTMLElement {
    HTMLElement(tag: "div", children: [content()])
}

func buildPage(title: String, isLoggedIn: Bool) -> HTMLElement {
    div {
        h1(title)
        if isLoggedIn {
            p("欢迎回来！")
        } else {
            p("请先登录")
            a(href: "/login", "点击登录")
        }
        HTMLElement(tag: "footer", text: "© 2024")
    }
}

let page = buildPage(title: "我的网站", isLoggedIn: false)
print(page.render())
```

### 7.3 实战：SQL 查询构建器

```swift
@resultBuilder
struct SQLBuilder {
    static func buildBlock(_ components: SQLClause...) -> SQLClause {
        SQLClause(components: components)
    }

    static func buildOptional(_ component: SQLClause?) -> SQLClause {
        component ?? SQLClause(components: [])
    }

    static func buildEither(first component: SQLClause) -> SQLClause {
        component
    }

    static func buildEither(second component: SQLClause) -> SQLClause {
        component
    }
}

struct SQLClause {
    let components: [SQLClause]
    let clause: String?

    init(clause: String) {
        self.components = []
        self.clause = clause
    }

    init(components: [SQLClause]) {
        self.components = components
        self.clause = nil
    }

    func build() -> String {
        if let clause = clause {
            return clause
        }
        return components.map { $0.build() }.filter { !$0.isEmpty }.joined(separator: " ")
    }
}

func select(_ columns: String...) -> SQLClause {
    SQLClause(clause: "SELECT \(columns.joined(separator: ", "))")
}

func from(_ table: String) -> SQLClause {
    SQLClause(clause: "FROM \(table)")
}

func where_(_ condition: String) -> SQLClause {
    SQLClause(clause: "WHERE \(condition)")
}

func orderBy(_ column: String, direction: String = "ASC") -> SQLClause {
    SQLClause(clause: "ORDER BY \(column) \(direction)")
}

func limit(_ count: Int) -> SQLClause {
    SQLClause(clause: "LIMIT \(count)")
}

func buildQuery(@SQLBuilder content: () -> SQLClause) -> String {
    content().build()
}

let query = buildQuery {
    select("name", "age", "email")
    from("users")
    where_("age > 18")
    orderBy("name")
    limit(10)
}
print(query)
```

### 7.4 实战：配置文件构建器

```swift
@resultBuilder
struct ConfigBuilder {
    static func buildBlock(_ components: ConfigItem...) -> [ConfigItem] {
        components
    }

    static func buildOptional(_ component: [ConfigItem]?) -> [ConfigItem] {
        component ?? []
    }

    static func buildEither(first component: [ConfigItem]) -> [ConfigItem] {
        component
    }

    static func buildEither(second component: [ConfigItem]) -> [ConfigItem] {
        component
    }

    static func buildArray(_ components: [[ConfigItem]]) -> [ConfigItem] {
        components.flatMap { $0 }
    }
}

struct ConfigItem {
    let key: String
    let value: String
}

func setting(_ key: String, _ value: String) -> ConfigItem {
    ConfigItem(key: key, value: value)
}

func buildConfig(isProduction: Bool, features: [String]) -> [ConfigItem] {
    ConfigBuilder.buildBlock {
        setting("app_name", "MyApp")
        setting("version", "2.0.0")
        if isProduction {
            setting("api_url", "https://api.example.com")
            setting("log_level", "error")
        } else {
            setting("api_url", "http://localhost:8080")
            setting("log_level", "debug")
        }
        for feature in features {
            setting("feature_\(feature)", "enabled")
        }
    }
}

let config = buildConfig(isProduction: true, features: ["dark_mode", "notifications"])
config.forEach { print("\($0.key) = \($0.value)") }
```

---

## 8. Result Builder 与泛型结合

### 8.1 泛型 Result Builder

```swift
@resultBuilder
struct ArrayBuilder<Element> {
    static func buildExpression(_ expression: Element) -> [Element] {
        [expression]
    }

    static func buildExpression(_ expression: [Element]) -> [Element] {
        expression
    }

    static func buildBlock(_ components: [Element]...) -> [Element] {
        components.flatMap { $0 }
    }

    static func buildOptional(_ component: [Element]?) -> [Element] {
        component ?? []
    }

    static func buildEither(first component: [Element]) -> [Element] {
        component
    }

    static func buildEither(second component: [Element]) -> [Element] {
        component
    }

    static func buildArray(_ components: [[Element]]) -> [Element] {
        components.flatMap { $0 }
    }
}

func buildArray<Element>(@ArrayBuilder<Element> content: () -> [Element]) -> [Element] {
    content()
}

let mixedArray = buildArray {
    1
    [2, 3]
    if true {
        4
    }
    for i in 5...7 {
        i
    }
}
print(mixedArray)
```

### 8.2 类型安全的 DSL

```swift
@resultBuilder
struct LayoutBuilder {
    static func buildExpression(_ expression: some LayoutItem) -> [LayoutItem] {
        [expression]
    }

    static func buildBlock(_ components: [LayoutItem]...) -> [LayoutItem] {
        components.flatMap { $0 }
    }
}

protocol LayoutItem {
    var frame: CGRect { get }
    func render() -> String
}

struct Box: LayoutItem {
    let frame: CGRect
    let color: String

    func render() -> String {
        "Box(\(color)) at \(frame)"
    }
}

struct Spacer: LayoutItem {
    let frame: CGRect = .zero
    let size: CGFloat

    func render() -> String {
        "Spacer(\(size)pt)"
    }
}
```

### 8.3 Result Builder 与关联类型

```swift
protocol BuilderProtocol {
    associatedtype Result

    @resultBuilder
    static var builder: some ResultBuilder where Result == builder.Result { get }
}
```

### 8.4 Result Builder 与 async/await

```swift
@resultBuilder
struct AsyncSequenceBuilder<Element> {
    static func buildBlock(_ components: AsyncSequence<Element>...) -> AsyncSequence<Element> {
        AsyncSequenceChain(chains: components)
    }
}

struct AsyncSequenceChain<Element>: AsyncSequence {
    let chains: [AsyncSequence<Element>]

    func makeAsyncIterator() -> AsyncIterator {
        AsyncIterator(chains: chains)
    }

    struct AsyncIterator: AsyncIteratorProtocol {
        var chains: [AsyncSequence<Element>]
        var currentIterator: AsyncSequence<Element>.AsyncIterator?
        var currentIndex = 0

        mutating func next() async -> Element? {
            while currentIndex < chains.count {
                if currentIterator == nil {
                    currentIterator = chains[currentIndex].makeAsyncIterator()
                }
                if let element = await currentIterator?.next() {
                    return element
                }
                currentIterator = nil
                currentIndex += 1
            }
            return nil
        }
    }
}
```

---

## 9. 实战：完整案例

### 9.1 用 Property Wrapper 封装 UserDefaults（完整版）

```swift
@propertyWrapper
struct AppStorage<T> {
    let key: String
    let defaultValue: T
    let store: UserDefaults

    var wrappedValue: T {
        get {
            store.object(forKey: key) as? T ?? defaultValue
        }
        set {
            if let optionalValue = newValue as? AnyOptional, optionalValue.isNil {
                store.removeObject(forKey: key)
            } else {
                store.set(newValue, forKey: key)
            }
        }
    }

    var projectedValue: Binding<T> {
        Binding(
            get: { wrappedValue },
            set: { wrappedValue = $0 }
        )
    }

    init(key: String, defaultValue: T, store: UserDefaults = .standard) {
        self.key = key
        self.defaultValue = defaultValue
        self.store = store
    }
}

private protocol AnyOptional {
    var isNil: Bool { get }
}

extension Optional: AnyOptional {
    var isNil: Bool { self == nil }
}

extension AppStorage where T: RawRepresentable {
    init(key: String, defaultValue: T, store: UserDefaults = .standard) {
        self.key = key
        self.defaultValue = defaultValue
        self.store = store
    }

    var wrappedValue: T {
        get {
            if let rawValue = store.object(forKey: key) as? T.RawValue,
               let value = T(rawValue: rawValue) {
                return value
            }
            return defaultValue
        }
        set {
            store.set(newValue.rawValue, forKey: key)
        }
    }
}

extension AppStorage where T: Codable {
    init(key: String, defaultValue: T, store: UserDefaults = .standard) {
        self.key = key
        self.defaultValue = defaultValue
        self.store = store
    }

    var wrappedValue: T {
        get {
            if let data = store.data(forKey: key),
               let decoded = try? JSONDecoder().decode(T.self, from: data) {
                return decoded
            }
            return defaultValue
        }
        set {
            if let encoded = try? JSONEncoder().encode(newValue) {
                store.set(encoded, forKey: key)
            }
        }
    }
}
```

### 9.2 用 Result Builder 构建图表 DSL

```swift
@resultBuilder
struct ChartBuilder {
    static func buildBlock(_ components: ChartComponent...) -> [ChartComponent] {
        components
    }

    static func buildOptional(_ component: [ChartComponent]?) -> [ChartComponent] {
        component ?? []
    }

    static func buildEither(first component: [ChartComponent]) -> [ChartComponent] {
        component
    }

    static func buildEither(second component: [ChartComponent]) -> [ChartComponent] {
        component
    }

    static func buildArray(_ components: [[ChartComponent]]) -> [ChartComponent] {
        components.flatMap { $0 }
    }
}

protocol ChartComponent {
    func render(into context: DrawingContext)
}

struct Bar: ChartComponent {
    let value: Double
    let label: String
    let color: String

    func render(into context: DrawingContext) {
        print("  柱状图[\(label)] = \(value) (\(color))")
    }
}

struct Line: ChartComponent {
    let points: [Double]
    let label: String

    func render(into context: DrawingContext) {
        print("  折线[\(label)] = \(points)")
    }
}

struct Pie: ChartComponent {
    let slices: [(Double, String)]

    func render(into context: DrawingContext) {
        slices.forEach { print("  饼图切片[\($0.1)] = \($0.0)") }
    }
}

struct Legend: ChartComponent {
    let items: [(String, String)]

    func render(into context: DrawingContext) {
        items.forEach { print("  图例: \($0.0) → \($0.1)") }
    }
}

struct DrawingContext {
    var width: Int
    var height: Int
}

func bar(_ value: Double, label: String, color: String = "blue") -> ChartComponent {
    Bar(value: value, label: label, color: color)
}

func line(_ points: Double..., label: String) -> ChartComponent {
    Line(points: points, label: label)
}

func pie(_ slices: (Double, String)...) -> ChartComponent {
    Pie(slices: slices)
}

func legend(_ items: (String, String)...) -> ChartComponent {
    Legend(items: items)
}

func chart(width: Int = 800, height: Int = 600, @ChartBuilder content: () -> [ChartComponent]) {
    let context = DrawingContext(width: width, height: height)
    print("图表 (\(width)x\(height)):")
    content().forEach { $0.render(into: context) }
}

chart(width: 600, height: 400) {
    bar(85, label: "Swift")
    bar(72, label: "Python", color: "green")
    bar(90, label: "Rust", color: "orange")
    line(60, 75, 85, 90, label: "趋势")
    legend(("Swift", "蓝色"), ("Python", "绿色"), ("Rust", "橙色"))
}
```

### 9.3 Property Wrapper + Result Builder 组合

```swift
@propertyWrapper
struct Observable<T: Equatable> {
    private var value: T
    private var subscribers: [(T) -> Void] = []

    var wrappedValue: T {
        get { value }
        set {
            if newValue != value {
                value = newValue
                subscribers.forEach { $0(newValue) }
            }
        }
    }

    var projectedValue: Observable<T> { self }

    mutating func subscribe(_ handler: @escaping (T) -> Void) {
        subscribers.append(handler)
        handler(value)
    }

    init(wrappedValue: T) {
        self.value = wrappedValue
    }
}

@resultBuilder
struct BindingBuilder {
    static func buildBlock(_ components: BindingAction...) -> [BindingAction] {
        components
    }
}

struct BindingAction {
    let name: String
    let action: () -> Void
}

class FormViewModel {
    @Observable var username: String = ""
    @Observable var password: String = ""
    @Observable var isRememberMe: Bool = false

    func configureBindings(@BindingBuilder content: () -> [BindingAction]) {
        content().forEach { $0.action() }
    }
}
```

---

## 10. Property Wrapper 与 Result Builder 的关系

### 10.1 共同点

| 特性 | Property Wrapper | Result Builder |
|------|-----------------|---------------|
| 本质 | 编译器变换 | 编译器变换 |
| 目的 | 封装属性逻辑 | 构建领域特定语言 |
| 声明式 | ✅ | ✅ |
| 类型安全 | ✅ | ✅ |
| SwiftUI 基础 | ✅ @State, @Binding | ✅ @ViewBuilder |

### 10.2 协作关系

在 SwiftUI 中，Property Wrapper 和 Result Builder 密切协作：

```swift
struct ContentView: View {
    @State private var count = 0

    var body: some View {
        VStack {
            Text("\(count)")
            Button("增加") { count += 1 }
        }
    }
}
```

- `@State`（Property Wrapper）管理 `count` 的状态
- `VStack` 的闭包参数标记了 `@ViewBuilder`（Result Builder），自动组合子视图
- `$count`（projectedValue）创建 `Binding`，传递给需要双向绑定的子视图

### 10.3 在 SwiftUI 渲染管线中的角色

```
声明（Property Wrapper 管理状态）
    ↓
body 计算属性（ViewBuilder 组合视图）
    ↓
SwiftUI Diff 算法（比较新旧视图树）
    ↓
渲染更新（只更新变化的部分）
```

---

## 11. AI 辅助理解和使用高级特性

### 11.1 AI 辅助场景

| 场景 | AI 提示词示例 |
|------|-------------|
| 理解 SwiftUI Property Wrapper | "@State 和 @StateObject 有什么区别？什么时候用哪个？" |
| 自定义 Property Wrapper | "帮我写一个 Property Wrapper，实现属性变更时自动记录日志" |
| 理解 Result Builder | "解释 ViewBuilder 的工作原理，为什么 VStack 里可以写多个 View？" |
| 自定义 Result Builder | "帮我写一个 Result Builder，用于构建 JSON 配置" |
| 调试 Property Wrapper | "我的 @Binding 没有更新 UI，可能是什么原因？" |
| 代码重构 | "帮我把这段手写的 UserDefaults 代码重构为 Property Wrapper" |

### 11.2 AI 辅助理解 SwiftUI 数据流

AI 可以帮你理清 SwiftUI 中各种 Property Wrapper 的数据流向：

```
@State（视图拥有）→ @Binding（传递给子视图）
@StateObject（视图拥有）→ @ObservedObject（子视图观察）
@EnvironmentObject（环境注入）→ 任何子视图
@Environment（系统值）→ 任何视图
@AppStorage（UserDefaults）→ 任何视图
```

### 11.3 AI 辅助自定义 Property Wrapper

**需求**：实现一个 Property Wrapper，在 Debug 模式下打印属性变更日志。

**AI 生成的代码**：

```swift
@propertyWrapper
struct DebugLogged<Value: Equatable> {
    private var value: Value
    let label: String

    var wrappedValue: Value {
        get { value }
        set {
            #if DEBUG
            if newValue != value {
                print("[变更] \(label): \(value) → \(newValue)")
            }
            #endif
            value = newValue
        }
    }

    init(wrappedValue: Value, _ label: String) {
        self.label = label
        self.value = wrappedValue
    }
}

struct GameSettings {
    @DebugLogged("音量") var volume: Int = 50
    @DebugLogged("难度") var difficulty: Int = 3
    @DebugLogged("全屏") var isFullscreen: Bool = true
}
```

### 11.4 AI 辅助自定义 Result Builder

**需求**：实现一个 Result Builder，用于构建命令行参数。

**AI 生成的代码**：

```swift
@resultBuilder
struct ArgBuilder {
    static func buildBlock(_ components: Arg...) -> [String] {
        components.flatMap { $0.tokens }
    }

    static func buildOptional(_ component: [String]?) -> [String] {
        component ?? []
    }

    static func buildEither(first component: [String]) -> [String] {
        component
    }

    static func buildEither(second component: [String]) -> [String] {
        component
    }

    static func buildArray(_ components: [[String]]) -> [String] {
        components.flatMap { $0 }
    }
}

struct Arg {
    let tokens: [String]
}

func flag(_ name: String, value: String? = nil) -> Arg {
    if let value = value {
        return Arg(tokens: ["--\(name)", value])
    }
    return Arg(tokens: ["--\(name)"])
}

func positional(_ value: String) -> Arg {
    Arg(tokens: [value])
}

func buildCommand(name: String, @ArgBuilder content: () -> [String]) -> String {
    ([name] + content()).joined(separator: " ")
}

let command = buildCommand(name: "git") {
    positional("commit")
    flag("m", value: "Initial commit")
    flag("verbose")
}
print(command)
```

### 11.5 AI 辅助调试常见问题

**问题 1：@Binding 不更新 UI**

```swift
struct ChildView: View {
    var count: Int

    var body: some View {
        Text("\(count)")
    }
}
```

AI 诊断：`count` 应该是 `@Binding var count: Int`，否则子视图只是接收了值的副本，无法响应变化。

**问题 2：@StateObject 在 if 中被重新创建**

```swift
struct ContentView: View {
    @State var showDetail = false

    var body: some View {
        if showDetail {
            DetailView()
        }
    }
}

struct DetailView: View {
    @StateObject var viewModel = DetailViewModel()
}
```

AI 诊断：`DetailView` 在 `if` 条件变化时会被销毁和重建，`@StateObject` 也会重新创建。应使用 `@ObservedObject` 从父视图传入，或使用 `.sheet()` 等方式。

💡 **提示**：当遇到 SwiftUI 数据流问题时，把视图层级和 Property Wrapper 的使用方式告诉 AI，它能快速定位问题。

---

## 本章小结

| 主题 | 核心要点 | 关键语法 |
|------|---------|---------|
| Property Wrapper 原理 | wrappedValue + projectedValue，编译器自动变换 | `@propertyWrapper struct` |
| wrappedValue | 包装属性的读写接口 | `var wrappedValue: T { get set }` |
| projectedValue | 通过 $ 访问的投影值 | `var projectedValue: Binding<T>` |
| SwiftUI @State | 视图本地状态，投影为 Binding | `@State private var count = 0` |
| SwiftUI @Binding | 引用外部状态 | `@Binding var isOn: Bool` |
| SwiftUI @Observable | Swift 5.9+ 观察机制 | `@Observable class ViewModel` |
| 自定义 Property Wrapper | 封装属性逻辑，消除重复代码 | `@Trimmed var name: String` |
| Result Builder 原理 | buildBlock/buildEither/buildOptional | `@resultBuilder struct` |
| ViewBuilder | SwiftUI 视图组合的核心 | `VStack { ... }` |
| 自定义 Result Builder | 构建领域特定语言 | `@HTMLBuilder func buildPage()` |
| 实战应用 | UserDefaults 封装、DSL 构建 | Property Wrapper + Result Builder |

> 💡 **学习建议**：Property Wrapper 和 Result Builder 是 SwiftUI 声明式语法的两大支柱。理解它们的原理，不仅能帮你写出更好的 SwiftUI 代码，还能让你在面对编译器错误时不再困惑。建议先从理解 SwiftUI 内置的 Property Wrapper 开始，逐步尝试自定义 Property Wrapper，最后再挑战 Result Builder。记住：**这些特性的本质都是编译器变换——理解了编译器做了什么，一切就豁然开朗了**。

← [Swift Macros 宏系统](./34-Swift-Macros宏系统.md) | [SwiftUI 初体验：第一个项目](../04-SwiftUI入门/36-SwiftUI初体验.md) →
