# 59-@Observable 与 Observation 框架

## 本章目标

- 理解 Observation 框架的诞生背景与核心思想
- 掌握 `@Observable` 的基础用法与自动属性追踪机制
- 学会在 SwiftUI 视图中使用 `@Observable`、`@Bindable`、`@Environment`
- 理解 `@Observable` 与 `@State`/`@Binding` 的关系与选择策略
- 能够将现有 `ObservableObject` 代码迁移到 `@Observable`
- 通过实战项目巩固所学知识

---

## 1. Observation 框架简介

### 1.1 为什么需要 Observation 框架？

在 iOS 17 之前，SwiftUI 的状态管理主要依赖 `ObservableObject` + `@Published`。这套机制虽然能用，但有几个明显的痛点：

| 痛点 | 说明 |
|------|------|
| 粗粒度刷新 | `@Published` 修饰的属性一变，整个视图都会刷新，即使没用到那个属性 |
| 样板代码多 | 每个属性都要手动加 `@Published`，忘加就不会触发刷新 |
| 类型约束 | 必须是 `class`，且必须遵循 `ObservableObject` 协议 |
| 属性包装器多 | `@StateObject`、`@ObservedObject`、`@EnvironmentObject` 容易混淆 |

> 💡 **生活类比**：`ObservableObject` 就像一个老式广播站——不管你关不关心，只要它播了，所有人都得听。而 `@Observable` 更像智能推送——只给你看你订阅的内容，精准又高效。

### 1.2 Observation 框架是什么？

Observation 框架是 Apple 在 WWDC 2023 随 iOS 17 推出的全新状态管理方案。它的核心是 `@Observable` 宏，能够：

- **自动追踪**属性访问，视图只在实际使用的属性变化时才刷新
- **零样板代码**，不需要手动标记 `@Published`
- **与 SwiftUI 深度集成**，开箱即用
- **更简洁的 API**，减少属性包装器的种类

> ⚠️ **系统要求**：Observation 框架需要 **iOS 17.0+**、**macOS 14.0+**、**watchOS 10.0+**。如果你的 App 需要支持更低版本，暂时无法使用。

---

## 2. @Observable vs ObservableObject 对比表

| 对比项 | `ObservableObject`（旧） | `@Observable`（新） |
|--------|--------------------------|---------------------|
| 最低系统版本 | iOS 13+ | iOS 17+ |
| 声明方式 | `class MyClass: ObservableObject` | `@Observable class MyClass` |
| 标记可观察属性 | 每个属性加 `@Published` | 自动追踪，无需手动标记 |
| 视图中使用 | `@StateObject` / `@ObservedObject` | `@State` |
| 环境注入 | `@EnvironmentObject` | `@Environment` |
| 双向绑定 | 需要 `@Published` + `$` 前缀 | `@Bindable` |
| 刷新粒度 | 整个对象级别 | 属性级别（精确刷新） |
| 忘加标记的后果 | 属性变化不触发刷新 | 无此问题，自动追踪 |
| 值类型支持 | ❌ 仅 class | ❌ 仅 class（但体验更好） |
| 宏依赖 | 无 | 需要 `import Observation` |

> 💡 **一句话总结**：`@Observable` 是 `ObservableObject` 的全面升级版——更少代码、更精确刷新、更少出错。

---

## 3. @Observable 基础用法

### 3.1 修饰 class

使用 `@Observable` 宏非常简单，只需在 class 声明前加上 `@Observable`：

```swift
import SwiftUI
import Observation

@Observable
class UserProfile {
    var name: String = ""
    var age: Int = 0
    var avatar: String = ""
}
```

就这么简单！不需要 `ObservableObject` 协议，不需要 `@Published`，所有存储属性自动变为可观察的。

### 3.2 自动追踪属性变化

`@Observable` 的魔法在于**自动追踪**。SwiftUI 会在视图读取属性时记录依赖，属性变化时只刷新相关视图：

```swift
@Observable
class GameStore {
    var score: Int = 0
    var level: Int = 1
    var playerName: String = "玩家"
}

struct ScoreView: View {
    var store: GameStore

    var body: some View {
        Text("分数：\(store.score)")
        // ✅ 只有 score 变化时才会刷新
        // ❌ level 或 playerName 变化不会触发此视图刷新
    }
}

struct LevelView: View {
    var store: GameStore

    var body: some View {
        Text("等级：\(store.level)")
        // ✅ 只有 level 变化时才会刷新
    }
}
```

> 💡 **生活类比**：想象你在网上购物，你只关注商品的价格。价格变了你会收到通知，但商品的评价变了你不会收到通知——因为你没"订阅"评价。`@Observable` 就是这么智能。

### 3.3 排除不需要追踪的属性

有些属性不需要触发视图刷新（比如临时缓存），可以用 `@ObservationIgnored` 排除：

```swift
@Observable
class DataStore {
    var items: [String] = []       // ✅ 自动追踪
    var isLoading: Bool = false     // ✅ 自动追踪

    @ObservationIgnored
    var cache: [String: Data] = [:] // ❌ 不追踪，变化不触发刷新
}
```

> ⚠️ **注意**：`@ObservationIgnored` 修饰的属性变化时，不会触发任何视图刷新。请确保你真的不需要它被追踪。

---

## 4. 与 SwiftUI 视图集成

### 4.1 视图自动订阅变化

在 SwiftUI 中使用 `@Observable` 对象，最简单的方式是用 `@State` 持有：

```swift
struct ContentView: View {
    @State private var store = GameStore()

    var body: some View {
        VStack {
            Text("分数：\(store.score)")
            Button("加分") {
                store.score += 10
            }
        }
    }
}
```

当 `store.score` 变化时，SwiftUI 会自动刷新 `ContentView`。

### 4.2 精确刷新（对比 ObservableObject）

这是 `@Observable` 最大的优势。来看对比：

```swift
// ❌ 旧方式：ObservableObject
class OldStore: ObservableObject {
    @Published var score: Int = 0
    @Published var level: Int = 1
}

struct OldScoreView: View {
    @ObservedObject var store: OldStore

    var body: some View {
        print("OldScoreView 刷新了") // ⚠️ level 变化也会触发！
        return Text("分数：\(store.score)")
    }
}

// ✅ 新方式：@Observable
@Observable
class NewStore {
    var score: Int = 0
    var level: Int = 1
}

struct NewScoreView: View {
    var store: NewStore

    var body: some View {
        print("NewScoreView 刷新了") // ✅ 只有 score 变化才触发！
        return Text("分数：\(store.score)")
    }
}
```

| 场景 | ObservableObject | @Observable |
|------|-----------------|-------------|
| `score` 变化 | ScoreView 刷新 ✅ | ScoreView 刷新 ✅ |
| `level` 变化 | ScoreView 也刷新 ❌ | ScoreView 不刷新 ✅ |

> 💡 **性能提升**：在复杂界面中，精确刷新可以大幅减少不必要的视图重绘，提升 App 流畅度。

### 4.3 在子视图中传递

将 `@Observable` 对象传递给子视图时，直接用普通属性即可：

```swift
struct ParentView: View {
    @State private var store = GameStore()

    var body: some View {
        VStack {
            ScoreView(store: store)
            LevelView(store: store)
        }
    }
}

struct ScoreView: View {
    var store: GameStore  // 不需要任何属性包装器！

    var body: some View {
        Text("分数：\(store.score)")
    }
}

struct LevelView: View {
    var store: GameStore  // 不需要任何属性包装器！

    var body: some View {
        Text("等级：\(store.level)")
    }
}
```

> 💡 **关键区别**：旧方式需要 `@ObservedObject`，新方式直接用 `var` 就行。SwiftUI 会自动追踪属性访问。

---

## 5. @Bindable 属性包装器

### 5.1 为什么需要 @Bindable？

`@Observable` 对象的属性可以直接读取，但如果需要**双向绑定**（比如绑定到 `TextField`），就需要 `@Bindable`：

```swift
struct ProfileEditView: View {
    @Bindable var profile: UserProfile

    var body: some View {
        TextField("姓名", text: $profile.name)
        //                     ↑ 需要 $ 前缀创建 Binding
        //                     ↑ @Bindable 让这成为可能
    }
}
```

### 5.2 @Bindable 的使用场景

| 场景 | 是否需要 @Bindable | 示例 |
|------|-------------------|------|
| 读取属性 | ❌ 不需要 | `Text(store.score)` |
| 写入属性 | ❌ 不需要 | `store.score = 100` |
| 创建 `$` 绑定 | ✅ 需要 | `TextField("", text: $store.name)` |
| Toggle 绑定 | ✅ 需要 | `Toggle("开关", isOn: $store.isEnabled)` |
| Slider 绑定 | ✅ 需要 | `Slider(value: $store.volume)` |

### 5.3 表单场景实战

```swift
@Observable
class Settings {
    var username: String = ""
    var notificationsEnabled: Bool = true
    var volume: Double = 0.5
    var theme: String = "浅色"
}

struct SettingsFormView: View {
    @State private var settings = Settings()

    var body: some View {
        @Bindable var settings = settings

        Form {
            Section("个人信息") {
                TextField("用户名", text: $settings.username)
            }

            Section("偏好设置") {
                Toggle("通知", isOn: $settings.notificationsEnabled)
                Slider("音量", value: $settings.volume)
                Picker("主题", selection: $settings.theme) {
                    Text("浅色").tag("浅色")
                    Text("深色").tag("深色")
                }
            }
        }
    }
}
```

> 💡 **小技巧**：在视图内部使用 `@Bindable`，可以用局部变量声明 `@Bindable var settings = settings`，这样不需要修改函数签名。

> ⚠️ **注意**：`@Bindable` 只能用于 `@Observable` 修饰的 class，不能用于普通 class 或 struct。

---

## 6. 与 @State / @Binding 的关系与选择

### 6.1 四种状态管理方式一览

| 方式 | 适用类型 | 用途 | 拥有权 |
|------|---------|------|--------|
| `@State` | 值类型（struct/枚举） | 视图拥有的简单状态 | 视图拥有 |
| `@State` | `@Observable` class | 视图拥有的可观察对象 | 视图拥有 |
| `@Binding` | 值类型的双向绑定 | 子视图修改父视图的状态 | 不拥有 |
| `@Bindable` | `@Observable` class 的双向绑定 | 创建 `$` 绑定给 TextField 等 | 不拥有 |

### 6.2 选择决策流程

```
需要管理状态？
│
├─ 是简单值（String, Int, Bool）？
│   └─ 使用 @State
│       └─ 子视图需要修改？→ @Binding
│
├─ 是复杂对象（class）？
│   └─ 使用 @Observable + @State
│       └─ 需要 $ 绑定？→ @Bindable
│
└─ 需要跨视图层级共享？
    └─ 使用 environment() 注入
```

### 6.3 代码对比

```swift
// 场景 1：简单计数器 → 用 @State
struct CounterView: View {
    @State private var count = 0

    var body: some View {
        Button("点击：\(count)") { count += 1 }
    }
}

// 场景 2：复杂数据模型 → 用 @Observable + @State
@Observable
class TodoList {
    var items: [Todo] = []
    var filter: FilterType = .all
}

struct TodoListView: View {
    @State private var todoList = TodoList()

    var body: some View {
        List(todoList.items) { item in
            Text(item.title)
        }
    }
}

// 场景 3：子视图需要双向绑定 → 用 @Bindable
struct TodoEditView: View {
    @Bindable var todoList: TodoList

    var body: some View {
        TextField("搜索", text: $todoList.searchText)
    }
}
```

> ⚠️ **常见误区**：不要用 `@State` 包装一个非 `@Observable` 的 class。`@State` 对引用类型的追踪能力有限，只有 `@Observable` 才能实现属性级别的精确追踪。

---

## 7. 环境注入：environment() + @Environment

### 7.1 为什么需要环境注入？

当多个视图需要访问同一个 `@Observable` 对象时，逐层传递太麻烦。环境注入可以让对象"无处不在"：

```
AppView
├── HomeView       ← 需要 store
│   └── ItemView   ← 需要 store
├── SearchView     ← 需要 store
└── ProfileView    ← 需要 store
```

### 7.2 注入方式

```swift
@Observable
class AppStore {
    var currentUser: String = ""
    var isLoggedIn: Bool = false
    var theme: String = "浅色"
}

@main
struct MyApp: App {
    @State private var store = AppStore()

    var body: some Scene {
        WindowGroup {
            ContentView()
                .environment(store)  // ✅ 注入到环境中
        }
    }
}
```

### 7.3 在视图中读取

```swift
struct HomeView: View {
    @Environment(AppStore.self) var store  // ✅ 从环境中读取

    var body: some View {
        VStack {
            Text("欢迎，\(store.currentUser)")
            Button("切换主题") {
                store.theme = store.theme == "浅色" ? "深色" : "浅色"
            }
        }
    }
}

struct ProfileView: View {
    @Environment(AppStore.self) var store

    var body: some View {
        @Bindable var store = store  // 需要绑定时

        Form {
            TextField("用户名", text: $store.currentUser)
            Toggle("已登录", isOn: $store.isLoggedIn)
        }
    }
}
```

### 7.4 旧方式 vs 新方式对比

| 操作 | 旧方式（ObservableObject） | 新方式（@Observable） |
|------|---------------------------|----------------------|
| 注入 | `.environmentObject(store)` | `.environment(store)` |
| 读取 | `@EnvironmentObject var store: Store` | `@Environment(Store.self) var store` |
| 安全性 | 找不到时运行时崩溃 ⚠️ | 找不到时编译时/运行时报错 |
| 类型推断 | 需要显式声明类型 | 通过 `.self` 指定类型 |

> ⚠️ **重要区别**：`@EnvironmentObject` 如果忘记注入，会在运行时崩溃且很难排查。`@Environment(Store.self)` 更安全，错误信息更清晰。

### 7.5 可选环境值

如果某个 `@Observable` 对象不是所有视图都需要，可以用可选方式读取：

```swift
struct SomeView: View {
    @Environment(AppStore?.self) var store  // 可选，不存在时为 nil

    var body: some View {
        if let store {
            Text(store.currentUser)
        } else {
            Text("未登录")
        }
    }
}
```

---

## 8. 迁移指南：从 ObservableObject 迁移到 @Observable

### 8.1 迁移步骤总览

```
步骤 1：将 class 声明从 ObservableObject 改为 @Observable
步骤 2：删除所有 @Published
步骤 3：更新视图中的属性包装器
步骤 4：更新环境注入方式
步骤 5：测试验证
```

### 8.2 详细迁移对照

#### 步骤 1 & 2：改造 Model

```swift
// ❌ 迁移前
class WeatherStore: ObservableObject {
    @Published var temperature: Double = 0
    @Published var condition: String = "晴"
    @Published var city: String = "北京"
    @Published var isLoading: Bool = false
}

// ✅ 迁移后
@Observable
class WeatherStore {
    var temperature: Double = 0
    var condition: String = "晴"
    var city: String = "北京"
    var isLoading: Bool = false
}
```

#### 步骤 3：更新视图属性包装器

| 旧写法 | 新写法 |
|--------|--------|
| `@StateObject var store = WeatherStore()` | `@State var store = WeatherStore()` |
| `@ObservedObject var store: WeatherStore` | `var store: WeatherStore` |
| `@EnvironmentObject var store: WeatherStore` | `@Environment(WeatherStore.self) var store` |
| 需要 `$` 绑定 | 加 `@Bindable` |

```swift
// ❌ 迁移前
struct WeatherView: View {
    @StateObject private var store = WeatherStore()

    var body: some View {
        // ...
    }
}

// ✅ 迁移后
struct WeatherView: View {
    @State private var store = WeatherStore()

    var body: some View {
        // ...
    }
}
```

#### 步骤 4：更新环境注入

```swift
// ❌ 迁移前
ContentView()
    .environmentObject(store)

// ✅ 迁移后
ContentView()
    .environment(store)
```

### 8.3 迁移速查表

| 旧 API | 新 API | 说明 |
|---------|--------|------|
| `class Foo: ObservableObject` | `@Observable class Foo` | 宏替代协议 |
| `@Published var x` | `var x` | 自动追踪 |
| `@StateObject` | `@State` | 统一为 @State |
| `@ObservedObject` | `var`（无包装器） | 直接声明属性 |
| `@EnvironmentObject` | `@Environment(Foo.self)` | 更安全的环境读取 |
| `.environmentObject()` | `.environment()` | 更简洁的注入 |
| `$property`（绑定） | `@Bindable` + `$property` | 显式声明可绑定 |

> ⚠️ **迁移注意**：如果你的 App 需要支持 iOS 16 及以下，不能迁移！`@Observable` 最低要求 iOS 17。可以考虑用条件编译同时兼容两种方式。

---

## 9. 完整实战示例：用 @Observable 重构天气 App 数据层

### 9.1 数据模型

```swift
import Foundation
import Observation

struct WeatherData: Identifiable {
    let id = UUID()
    var city: String
    var temperature: Double
    var condition: String
    var humidity: Int
    var windSpeed: Double
}

@Observable
class WeatherStore {
    var weathers: [WeatherData] = []
    var selectedCity: String = "北京"
    var isLoading: Bool = false
    var errorMessage: String?

    @ObservationIgnored
    private var fetchTask: Task<Void, Never>?

    func fetchWeather(for city: String) {
        fetchTask?.cancel()
        isLoading = true
        errorMessage = nil

        fetchTask = Task {
            do {
                try await Task.sleep(for: .seconds(1))

                guard !Task.isCancelled else { return }

                let mockWeather = WeatherData(
                    city: city,
                    temperature: Double.random(in: 10...35),
                    condition: ["晴", "多云", "阴", "小雨"].randomElement()!,
                    humidity: Int.random(in: 30...90),
                    windSpeed: Double.random(in: 0...20)
                )

                if let index = weathers.firstIndex(where: { $0.city == city }) {
                    weathers[index] = mockWeather
                } else {
                    weathers.append(mockWeather)
                }

                isLoading = false
            } catch {
                errorMessage = "加载失败：\(error.localizedDescription)"
                isLoading = false
            }
        }
    }

    func removeWeather(at offsets: IndexSet) {
        weathers.remove(atOffsets: offsets)
    }
}
```

### 9.2 主视图

```swift
import SwiftUI

struct WeatherAppView: View {
    @State private var store = WeatherStore()

    var body: some View {
        NavigationStack {
            WeatherListView(store: store)
                .navigationTitle("天气")
                .toolbar {
                    ToolbarItem {
                        AddCityButton(store: store)
                    }
                }
        }
    }
}
```

### 9.3 天气列表视图

```swift
struct WeatherListView: View {
    var store: WeatherStore

    var body: some View {
        Group {
            if store.weathers.isEmpty {
                ContentUnavailableView(
                    "暂无天气数据",
                    systemImage: "cloud.sun",
                    description: Text("点击右上角添加城市")
                )
            } else {
                List {
                    ForEach(store.weathers) { weather in
                        NavigationLink {
                            WeatherDetailView(weather: weather, store: store)
                        } label: {
                            WeatherRow(weather: weather)
                        }
                    }
                    .onDelete { offsets in
                        store.removeWeather(at: offsets)
                    }
                }
            }
        }
        .overlay {
            if store.isLoading {
                ProgressView("加载中...")
            }
        }
        .alert("错误", isPresented: .constant(store.errorMessage != nil)) {
            Button("确定") { store.errorMessage = nil }
        } message: {
            Text(store.errorMessage ?? "")
        }
    }
}

struct WeatherRow: View {
    let weather: WeatherData

    var body: some View {
        HStack {
            Image(systemName: conditionIcon(weather.condition))
                .font(.title2)
                .foregroundStyle(.orange)

            VStack(alignment: .leading) {
                Text(weather.city)
                    .font(.headline)
                Text(weather.condition)
                    .font(.subheadline)
                    .foregroundStyle(.secondary)
            }

            Spacer()

            Text("\(Int(weather.temperature))°")
                .font(.title)
                .bold()
        }
        .padding(.vertical, 4)
    }

    func conditionIcon(_ condition: String) -> String {
        switch condition {
        case "晴": return "sun.max.fill"
        case "多云": return "cloud.sun.fill"
        case "阴": return "cloud.fill"
        case "小雨": return "cloud.rain.fill"
        default: return "cloud"
        }
    }
}
```

### 9.4 添加城市视图

```swift
struct AddCityButton: View {
    var store: WeatherStore

    @State private var newCity = ""
    @State private var showingAddSheet = false

    var body: some View {
        Button {
            showingAddSheet = true
        } label: {
            Image(systemName: "plus")
        }
        .sheet(isPresented: $showingAddSheet) {
            NavigationStack {
                Form {
                    TextField("输入城市名", text: $newCity)
                }
                .navigationTitle("添加城市")
                .toolbar {
                    ToolbarItem(.cancellationAction) {
                        Button("取消") { showingAddSheet = false }
                    }
                    ToolbarItem(.confirmationAction) {
                        Button("添加") {
                            store.fetchWeather(for: newCity)
                            newCity = ""
                            showingAddSheet = false
                        }
                    }
                }
            }
        }
    }
}
```

### 9.5 天气详情视图

```swift
struct WeatherDetailView: View {
    let weather: WeatherData
    var store: WeatherStore

    var body: some View {
        ScrollView {
            VStack(spacing: 24) {
                Image(systemName: conditionIcon(weather.condition))
                    .font(.system(size: 80))
                    .foregroundStyle(.orange)

                Text(weather.city)
                    .font(.largeTitle)
                    .bold()

                Text("\(Int(weather.temperature))°C")
                    .font(.system(size: 60, weight: .bold))

                Text(weather.condition)
                    .font(.title2)
                    .foregroundStyle(.secondary)

                Divider()

                HStack(spacing: 40) {
                    VStack {
                        Image(systemName: "drop.fill")
                            .foregroundStyle(.blue)
                        Text("湿度")
                            .font(.caption)
                            .foregroundStyle(.secondary)
                        Text("\(weather.humidity)%")
                            .font(.title3)
                            .bold()
                    }

                    VStack {
                        Image(systemName: "wind")
                            .foregroundStyle(.teal)
                        Text("风速")
                            .font(.caption)
                            .foregroundStyle(.secondary)
                        Text("\(Int(weather.windSpeed)) km/h")
                            .font(.title3)
                            .bold()
                    }
                }

                Button("刷新数据") {
                    store.fetchWeather(for: weather.city)
                }
                .buttonStyle(.borderedProminent)
            }
            .padding()
        }
        .navigationTitle(weather.city)
        .navigationBarTitleDisplayMode(.inline)
    }

    func conditionIcon(_ condition: String) -> String {
        switch condition {
        case "晴": return "sun.max.fill"
        case "多云": return "cloud.sun.fill"
        case "阴": return "cloud.fill"
        case "小雨": return "cloud.rain.fill"
        default: return "cloud"
        }
    }
}
```

> 💡 **实战要点**：
> - `WeatherStore` 用 `@Observable` 修饰，所有属性自动可观察
> - `isLoading` 和 `errorMessage` 不需要 `@Published`，变化自动触发刷新
> - `fetchTask` 用 `@ObservationIgnored` 排除追踪，因为它是内部实现细节
> - 视图中直接用 `var store: WeatherStore` 传递，无需额外属性包装器

---

## 10. 常见陷阱与最佳实践

### 10.1 常见陷阱

#### 陷阱 1：在 body 中创建 @Observable 对象

```swift
// ❌ 错误：每次刷新都会创建新对象
struct BadView: View {
    var body: some View {
        let store = WeatherStore()  // 每次刷新都重新创建！
        Text(store.selectedCity)
    }
}

// ✅ 正确：用 @State 持有
struct GoodView: View {
    @State private var store = WeatherStore()

    var body: some View {
        Text(store.selectedCity)
    }
}
```

#### 陷阱 2：忘记用 @Bindable 创建绑定

```swift
// ❌ 错误：无法创建 $ 绑定
struct BadFormView: View {
    var store: Settings

    var body: some View {
        TextField("用户名", text: $store.username)
        // ❌ 编译错误！var store 没有 $ 语法
    }
}

// ✅ 正确：加 @Bindable
struct GoodFormView: View {
    @Bindable var store: Settings

    var body: some View {
        TextField("用户名", text: $store.username)
        // ✅ 编译通过
    }
}
```

#### 陷阱 3：在 init 中直接修改 @Observable 属性

```swift
// ❌ 可能有问题：初始化期间修改属性
@Observable
class Config {
    var theme: String = "浅色"

    init() {
        self.theme = loadTheme()  // ⚠️ 初始化期间可能触发观察
    }
}

// ✅ 更安全：使用延迟初始化
@Observable
class Config {
    var theme: String = "浅色"

    func loadSavedTheme() {
        theme = loadTheme()  // ✅ 初始化完成后再调用
    }
}
```

#### 陷阱 4：混用 ObservableObject 和 @Observable

```swift
// ❌ 不要混用！
@Observable
class MixedStore: ObservableObject {  // ❌ 同时使用两种方式
    @Published var score: Int = 0     // ❌ 不需要 @Published
    var level: Int = 1
}

// ✅ 选择一种方式
@Observable
class CleanStore {
    var score: Int = 0
    var level: Int = 1
}
```

### 10.2 最佳实践

| 最佳实践 | 说明 |
|---------|------|
| 🎯 用 `@State` 拥有对象 | 视图创建的 `@Observable` 对象用 `@State` 持有 |
| 📦 用 `@Bindable` 做绑定 | 需要 `$` 语法时加 `@Bindable` |
| 🌍 用 `environment()` 共享 | 跨层级共享时用环境注入 |
| 🚫 用 `@ObservationIgnored` 排除 | 内部实现属性不需要追踪时排除 |
| 🔒 保持单一数据源 | 一个数据流只由一个 `@Observable` 对象管理 |
| 🧪 测试时直接实例化 | `@Observable` 对象不依赖 SwiftUI，可以独立测试 |
| 📱 检查最低系统版本 | `@Observable` 需要 iOS 17+，低版本用 `ObservableObject` |

### 10.3 性能优化技巧

```swift
@Observable
class LargeDataStore {
    var items: [Item] = []           // ✅ 整个数组变化时才刷新
    var filteredItems: [Item] {      // ✅ 计算属性，基于 items 自动追踪
        items.filter { $0.isActive }
    }

    @ObservationIgnored
    private var imageCache: [URL: Image] = [:]  // ✅ 缓存不需要追踪

    func updateItem(id: UUID, newValue: Item) {
        // ✅ 精确修改单个元素
        if let index = items.firstIndex(where: { $0.id == id }) {
            items[index] = newValue
        }
    }
}
```

> 💡 **性能提示**：`@Observable` 的精确刷新本身就是最大的性能优化。不需要像 `ObservableObject` 那样手动拆分 ObservableObject 来避免不必要的刷新。

---

## 小结

| 知识点 | 核心内容 |
|--------|---------|
| Observation 框架 | iOS 17+ 新状态管理方案，替代 `ObservableObject` |
| `@Observable` | 宏修饰 class，自动追踪属性变化，零样板代码 |
| 精确刷新 | 视图只在实际读取的属性变化时才刷新，性能更优 |
| `@Bindable` | 为 `@Observable` 对象创建 `$` 双向绑定 |
| `@State` 持有 | 视图拥有 `@Observable` 对象用 `@State` |
| `@Environment` | 环境注入共享 `@Observable` 对象，比 `@EnvironmentObject` 更安全 |
| 迁移 | `ObservableObject` → `@Observable`，`@Published` → `var`，`@StateObject` → `@State` |
| `@ObservationIgnored` | 排除不需要追踪的属性 |

> 💡 **学习建议**：如果你是新项目且最低支持 iOS 17，请直接使用 `@Observable`，无需再学 `ObservableObject`。如果需要兼容旧版本，先掌握 `ObservableObject`，等升级最低版本后再迁移。
