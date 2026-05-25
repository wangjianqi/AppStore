# NavigationStack 与 NavigationSplitView 深入

> 🎯 **本章目标**：深入理解 iOS 16+ 全新导航体系，掌握 NavigationStack 的编程式导航与 NavigationPath，学会 NavigationSplitView 的多栏布局，能够在 iPhone 和 iPad 上构建灵活的导航体验。

---

## 导航体系演进

### 从 NavigationView 到新导航体系

SwiftUI 诞生之初，导航能力由 `NavigationView` 和 `NavigationLink` 承担。开发者通过声明式的 `NavigationLink` 将视图串联起来，配合 `NavigationView` 提供的导航栏与转场动画完成页面跳转。然而，`NavigationView` 从设计之初就存在诸多局限，难以满足实际项目中的复杂导航需求。

iOS 16 引入了 `NavigationStack` 和 `NavigationSplitView`，彻底重构了 SwiftUI 的导航体系。`NavigationStack` 专注于线性栈式导航，提供编程式导航能力和类型安全的路由机制；`NavigationSplitView` 则专注于多栏并列布局，为 iPad 和 Mac 提供原生的分栏体验。

### NavigationView 的问题与局限

| 特性 | NavigationView（旧） | NavigationStack + NavigationSplitView（新） |
|---|---|---|
| 编程式导航 | 不支持，只能通过 NavigationLink 声明 | 支持 NavigationPath 编程式控制 |
| 类型安全路由 | 无，依赖字符串或泛型 | navigationDestination 提供类型安全路由 |
| 深层链接 | 难以实现，需嵌套多层 NavigationLink | 通过 NavigationPath 直接构建任意路径 |
| 导航状态管理 | 无法外部读取或控制导航栈 | NavigationPath 可观察、可序列化 |
| 多栏布局 | NavigationView 配合 .sidebar 样式，行为不可控 | NavigationSplitView 原生支持两栏/三栏 |
| iPad 适配 | 需手动处理不同尺寸类的布局 | 自动适配 iPhone/iPad/Mac |
| 栏可见性控制 | 无 API | columnVisibility 精细控制 |
| 导航转场自定义 | 不支持 | .navigationTransition() 自定义转场 |

### 新导航体系的核心设计思想

新导航体系遵循三个核心设计原则：

1. **关注点分离** — 栈式导航（NavigationStack）与分栏导航（NavigationSplitView）各司其职，不再用一个 `NavigationView` 模糊处理两种截然不同的导航模式。

2. **数据驱动** — 导航状态由数据（NavigationPath）驱动，而非由视图层级隐式决定。这意味着导航状态可以被观察、持久化、测试和恢复。

3. **类型安全** — 通过 `navigationDestination(for:)` 修饰符，路由目标与数据类型绑定，编译器能够在编译期检查路由的正确性，避免运行时崩溃。

---

## NavigationStack 深入

### 基本使用回顾

`NavigationStack` 的最简用法与 `NavigationView` 类似，包裹根视图并提供导航栏：

```swift
struct ContentView: View {
    var body: some View {
        NavigationStack {
            List(1...10, id: \.self) { index in
                NavigationLink("第 \(index) 项") {
                    DetailView(index: index)
                }
            }
            .navigationTitle("首页")
        }
    }
}

struct DetailView: View {
    let index: Int
    var body: some View {
        Text("详情页 \(index)")
            .navigationTitle("详情 \(index)")
    }
}
```

### navigationDestination 修饰符：类型安全的路由

`navigationDestination(for:destination:)` 是新导航体系的核心修饰符。它将数据类型与目标视图绑定，实现类型安全的路由：

```swift
struct Route: Hashable {
    let id: Int
    let title: String
}

struct ContentView: View {
    var body: some View {
        NavigationStack {
            List(1...5, id: \.self) { i in
                NavigationLink("跳转", value: Route(id: i, title: "页面 \(i)"))
            }
            .navigationTitle("首页")
            .navigationDestination(for: Route.self) { route in
                RouteDetailView(route: route)
            }
        }
    }
}

struct RouteDetailView: View {
    let route: Route
    var body: some View {
        VStack {
            Text(route.title)
            NavigationLink("下一页", value: Route(id: route.id + 1, title: "页面 \(route.id + 1)"))
        }
        .navigationTitle(route.title)
        .navigationDestination(for: Route.self) { route in
            RouteDetailView(route: route)
        }
    }
}
```

关键点：`navigationDestination` 应该附加在 `NavigationStack` 内部的视图上，而非 `NavigationStack` 本身。每个 `NavigationLink` 通过 `value` 参数传递路由数据，系统自动查找匹配的 `navigationDestination` 来渲染目标视图。

### 编程式导航：NavigationPath 的使用

`NavigationPath` 是一个类型擦除的导航栈容器，允许你在视图外部以编程方式控制导航：

```swift
struct ContentView: View {
    @State private var path = NavigationPath()

    var body: some View {
        NavigationStack(path: $path) {
            VStack(spacing: 20) {
                Button("跳转到第 1 页") {
                    path.append(Route(id: 1, title: "页面 1"))
                }
                Button("跳转到第 3 页（深层链接）") {
                    path.append(Route(id: 1, title: "页面 1"))
                    path.append(Route(id: 2, title: "页面 2"))
                    path.append(Route(id: 3, title: "页面 3"))
                }
                Button("返回根视图") {
                    path.removeLast(path.count)
                }
            }
            .navigationTitle("首页")
            .navigationDestination(for: Route.self) { route in
                RouteDetailView(route: route, path: $path)
            }
        }
    }
}

struct RouteDetailView: View {
    let route: Route
    @Binding var path: NavigationPath

    var body: some View {
        VStack(spacing: 20) {
            Text(route.title)
            Button("继续前进") {
                path.append(Route(id: route.id + 1, title: "页面 \(route.id + 1)"))
            }
            Button("返回上一页") {
                path.removeLast()
            }
            Button("返回首页") {
                path.removeLast(path.count)
            }
        }
        .navigationTitle(route.title)
        .navigationDestination(for: Route.self) { route in
            RouteDetailView(route: route, path: $path)
        }
    }
}
```

`NavigationPath` 支持以下操作：

| 操作 | 方法 | 说明 |
|---|---|---|
| 压入 | `path.append(value)` | 将新路由数据压入栈顶 |
| 弹出一个 | `path.removeLast()` | 移除栈顶元素 |
| 弹出多个 | `path.removeLast(k)` | 移除栈顶 k 个元素 |
| 清空栈 | `path.removeLast(path.count)` | 回到根视图 |
| 栈深度 | `path.count` | 当前导航栈的深度 |
| 是否为空 | `path.isEmpty` | 是否在根视图 |

### 基于值类型的路由 vs 基于字符串的路由

SwiftUI 的路由可以基于值类型（枚举、结构体），也可以基于字符串。两种方式各有优劣：

```swift
enum AppRoute: Hashable {
    case detail(id: Int)
    case profile(userId: String)
    case settings
}

struct ContentView: View {
    @State private var path = NavigationPath()

    var body: some View {
        NavigationStack(path: $path) {
            HomeView()
                .navigationDestination(for: AppRoute.self) { route in
                    switch route {
                    case .detail(let id):
                        DetailView(id: id)
                    case .profile(let userId):
                        ProfileView(userId: userId)
                    case .settings:
                        SettingsView()
                    }
                }
        }
    }
}
```

| 对比维度 | 值类型路由（枚举/结构体） | 字符串路由 |
|---|---|---|
| 类型安全 | 编译期检查，不会拼错 | 运行时匹配，容易出错 |
| 可扩展性 | 新增路由需修改枚举 | 字符串灵活但无约束 |
| 参数传递 | 关联值强类型 | 需手动解析字符串参数 |
| 适用场景 | 中大型项目 | 快速原型、简单项目 |
| 可维护性 | 高，重构时编译器辅助 | 低，依赖命名约定 |

> 💡 **提示**：推荐在正式项目中使用枚举路由，编译器会成为你最好的盟友。字符串路由虽然灵活，但容易在重构时遗漏，导致运行时导航失败。

### 深层链接：通过 NavigationPath 直接跳转到任意层级

深层链接是 `NavigationPath` 最强大的能力之一。你可以直接构建完整的导航路径，让用户一步到位到达目标页面：

```swift
struct DeepLinkHandler {
    static func handleDeepLink(url: URL, path: inout NavigationPath) {
        guard let host = url.host else { return }

        switch host {
        case "detail":
            if let id = url.pathComponents.last.flatMap(Int.init) {
                path.removeLast(path.count)
                path.append(AppRoute.detail(id: id))
            }
        case "profile":
            if url.pathComponents.count > 1 {
                let userId = url.pathComponents[1]
                path.removeLast(path.count)
                path.append(AppRoute.profile(userId: userId))
            }
        default:
            break
        }
    }
}

struct ContentView: View {
    @State private var path = NavigationPath()

    var body: some View {
        NavigationStack(path: $path) {
            HomeView()
                .navigationDestination(for: AppRoute.self) { route in
                    switch route {
                    case .detail(let id):
                        DetailView(id: id)
                    case .profile(let userId):
                        ProfileView(userId: userId)
                    case .settings:
                        SettingsView()
                    }
                }
                .onOpenURL { url in
                    DeepLinkHandler.handleDeepLink(url: url, path: &path)
                }
        }
    }
}
```

### 导航状态保存与恢复

`NavigationPath` 支持 `Codable`，可以将导航状态序列化和反序列化，实现导航状态的持久化：

```swift
struct AppState: Codable {
    var navigationPath: [AppRoute]
}

class NavigationState: ObservableObject {
    @Published var path = NavigationPath()
    private let saveKey = "NavigationState"

    func save() {
        let codablePath = path.codable
        if let data = try? JSONEncoder().encode(codablePath) {
            UserDefaults.standard.set(data, forKey: saveKey)
        }
    }

    func restore() {
        guard let data = UserDefaults.standard.data(forKey: saveKey) else { return }
        if let codablePath = try? JSONDecoder().decode(NavigationPath.CodableValue.self, from: data) {
            path = NavigationPath(codablePath)
        }
    }
}
```

> ⚠️ **警告**：`NavigationPath` 的 `codable` 属性要求所有路由类型都实现 `Codable`。如果你的路由枚举包含不满足 `Codable` 的关联值，编译时会报错。确保所有路由类型同时遵循 `Hashable` 和 `Codable`。

---

## NavigationSplitView 详解

### 两栏布局：sidebar + detail

`NavigationSplitView` 最基础的形态是两栏布局，左侧为侧边栏，右侧为详情页：

```swift
struct TwoColumnView: View {
    @State private var selectedItem: Item?

    var body: some View {
        NavigationSplitView {
            List(Item.samples) { item in
                NavigationLink(item.title, value: item)
            }
            .navigationTitle("侧边栏")
        } detail: {
            if let item = selectedItem {
                DetailView(item: item)
            } else {
                Text("请选择一个项目")
                    .foregroundStyle(.secondary)
            }
        }
    }
}
```

### 三栏布局：sidebar + content + detail

三栏布局增加了一个中间的内容栏，适合邮件、笔记等需要层级浏览的场景：

```swift
struct ThreeColumnView: View {
    @State private var selectedCategory: Category?
    @State private var selectedItem: Item?

    var body: some View {
        NavigationSplitView {
            List(Category.samples) { category in
                NavigationLink(category.name, value: category)
            }
            .navigationTitle("分类")
        } content: {
            if let category = selectedCategory {
                List(category.items) { item in
                    NavigationLink(item.title, value: item)
                }
                .navigationTitle(category.name)
            } else {
                Text("请选择分类")
                    .foregroundStyle(.secondary)
            }
        } detail: {
            if let item = selectedItem {
                ItemDetailView(item: item)
            } else {
                Text("请选择项目")
                    .foregroundStyle(.secondary)
            }
        }
    }
}
```

### columnVisibility 控制栏的显示

`NavigationSplitView` 提供了 `columnVisibility` 参数，允许你精确控制各栏的显示状态：

```swift
struct ColumnControlView: View {
    @State private var columnVisibility: NavigationSplitViewVisibility = .all
    @State private var selectedItem: Item?

    var body: some View {
        NavigationSplitView(columnVisibility: $columnVisibility) {
            List(Item.samples) { item in
                NavigationLink(item.title, value: item)
            }
            .navigationTitle("侧边栏")
            .toolbar {
                ToolbarItem(placement: .primaryAction) {
                    Button("切换侧边栏") {
                        withAnimation {
                            columnVisibility = columnVisibility == .all ? .detailOnly : .all
                        }
                    }
                }
            }
        } detail: {
            if let item = selectedItem {
                DetailView(item: item)
            } else {
                Text("请选择项目")
            }
        }
    }
}
```

`NavigationSplitViewVisibility` 的可选值：

| 值 | 说明 |
|---|---|
| `.all` | 显示所有栏 |
| `.doubleColumn` | 仅显示两栏（隐藏中间栏） |
| `.detailOnly` | 仅显示详情栏 |
| `.automatic` | 系统根据设备自动决定 |

### 在 iPhone 上的自适应行为

iPhone 屏幕较小，`NavigationSplitView` 会自动将多栏布局转换为栈式导航。侧边栏和内容栏会以全屏方式呈现，用户通过导航栏返回上一栏。这种自适应行为无需额外代码。

```swift
struct AdaptiveView: View {
    @State private var selectedItem: Item?

    var body: some View {
        NavigationSplitView {
            List(Item.samples) { item in
                NavigationLink(item.title, value: item)
            }
            .navigationTitle("项目列表")
        } detail: {
            if let item = selectedItem {
                DetailView(item: item)
            } else {
                Text("请选择项目")
                    .foregroundStyle(.secondary)
            }
        }
        .navigationSplitViewStyle(.automatic)
    }
}
```

> 💡 **提示**：在 iPhone 上，`.automatic` 样式等同于 `.balanced`，侧边栏以全屏方式覆盖详情页。你可以使用 `.navigationSplitViewStyle(.prominentDetail)` 让详情页更突出。

### 在 iPad 上的多栏体验

iPad 上 `NavigationSplitView` 默认以并排方式展示多栏。你可以通过 `navigationSplitViewStyle` 选择不同的展示风格：

| 样式 | 说明 | 适用场景 |
|---|---|---|
| `.automatic` | 系统根据设备和方向自动选择 | 大多数场景 |
| `.balanced` | 侧边栏和详情栏均衡显示 | 通用场景 |
| `.prominentDetail` | 详情栏占据更多空间 | 内容为主的场景 |

```swift
NavigationSplitView {
    SidebarView()
} detail: {
    DetailView()
}
.navigationSplitViewStyle(.prominentDetail)
```

### 在 Mac 上的侧边栏行为

在 macOS 上，`NavigationSplitView` 的侧边栏遵循 macOS 的标准行为：用户可以通过工具栏按钮切换侧边栏显示/隐藏，侧边栏支持拖拽调整宽度。`columnVisibility` 绑定会自动与 macOS 的侧边栏状态同步。

---

## 自定义导航转场

### .navigationTransition() 修饰符

iOS 18 引入了 `.navigationTransition()` 修饰符，允许你自定义 NavigationStack 内的页面转场动画：

```swift
struct CustomTransitionView: View {
    var body: some View {
        NavigationStack {
            ColorListView()
                .navigationTransition(.zoom(sourceID: selectedID, in: namespace))
        }
    }
}
```

SwiftUI 提供了以下内置转场：

| 转场类型 | 说明 |
|---|---|
| `.default` | 系统默认的滑入转场 |
| `.zoom(sourceID:in:)` | 从源视图缩放展开 |
| `.slide` | 滑动转场 |

### .navigationDestinationLink() 关联转场

通过将 `NavigationLink` 与转场修饰符配合，可以实现从列表项到详情页的无缝缩放动画：

```swift
struct ZoomTransitionView: View {
    @Namespace private var namespace
    @State private var selectedID: String?

    var body: some View {
        NavigationStack {
            ScrollView {
                LazyVGrid(columns: [GridItem(.adaptive(minimum: 150))]) {
                    ForEach(Photo.samples) { photo in
                        NavigationLink(value: photo) {
                            Image(photo.thumbnail)
                                .clipShape(RoundedRectangle(cornerRadius: 12))
                                .matchedTransitionSource(id: photo.id, in: namespace)
                        }
                    }
                }
            }
            .navigationTitle("照片")
            .navigationDestination(for: Photo.self) { photo in
                Image(photo.fullSize)
                    .navigationTransition(.zoom(sourceID: photo.id, in: namespace))
            }
        }
    }
}
```

### 自定义转场动画示例

你还可以结合 `.navigationTransition()` 和 `AnyNavigationTransition` 创建更复杂的转场效果：

```swift
struct SlideTransitionView: View {
    var body: some View {
        NavigationStack {
            List(Item.samples) { item in
                NavigationLink(value: item) {
                    HStack {
                        Image(systemName: item.icon)
                        Text(item.title)
                    }
                }
            }
            .navigationTitle("项目")
            .navigationDestination(for: Item.self) { item in
                ItemDetailView(item: item)
                    .navigationTransition(.slide)
            }
        }
    }
}
```

> ⚠️ **警告**：自定义导航转场需要 iOS 18+。如果你的应用需要支持 iOS 16/17，请使用条件编译或回退到默认转场。

---

## 导航架构设计

### MVVM 中的导航状态管理

在 MVVM 架构中，导航状态应该由 ViewModel 持有和管理，View 仅负责绑定和渲染：

```swift
class HomeViewModel: ObservableObject {
    @Published var path = NavigationPath()

    func goToDetail(id: Int) {
        path.append(AppRoute.detail(id: id))
    }

    func goToProfile(userId: String) {
        path.append(AppRoute.profile(userId: userId))
    }

    func goBack() {
        path.removeLast()
    }

    func goToRoot() {
        path.removeLast(path.count)
    }
}

struct HomeView: View {
    @StateObject private var viewModel = HomeViewModel()

    var body: some View {
        NavigationStack(path: $viewModel.path) {
            VStack(spacing: 20) {
                Button("查看详情") {
                    viewModel.goToDetail(id: 1)
                }
                Button("查看个人资料") {
                    viewModel.goToProfile(userId: "user_001")
                }
            }
            .navigationTitle("首页")
            .navigationDestination(for: AppRoute.self) { route in
                switch route {
                case .detail(let id):
                    DetailView(id: id)
                case .profile(let userId):
                    ProfileView(userId: userId)
                case .settings:
                    SettingsView()
                }
            }
        }
    }
}
```

### Coordinator 模式在 SwiftUI 中的实现

Coordinator 模式将导航逻辑从 ViewModel 中抽离，由专门的协调器统一管理页面跳转：

```swift
protocol Coordinator: ObservableObject {
    var path: NavigationPath { get set }
    func navigate(to route: AppRoute)
    func goBack()
    func goBackToRoot()
}

class AppCoordinator: Coordinator {
    @Published var path = NavigationPath()

    func navigate(to route: AppRoute) {
        path.append(route)
    }

    func goBack() {
        guard !path.isEmpty else { return }
        path.removeLast()
    }

    func goBackToRoot() {
        path.removeLast(path.count)
    }
}

struct CoordinatorView<CoordinatorType: Coordinator>: View {
    @ObservedObject var coordinator: CoordinatorType

    var body: some View {
        NavigationStack(path: $coordinator.path) {
            HomeView(coordinator: coordinator)
                .navigationDestination(for: AppRoute.self) { route in
                    routeView(for: route)
                }
        }
    }

    @ViewBuilder
    private func routeView(for route: AppRoute) -> some View {
        switch route {
        case .detail(let id):
            DetailView(id: id, coordinator: coordinator)
        case .profile(let userId):
            ProfileView(userId: userId, coordinator: coordinator)
        case .settings:
            SettingsView(coordinator: coordinator)
        }
    }
}
```

### 路由管理器的设计

对于大型项目，可以设计一个集中式的路由管理器，统一管理所有路由注册和跳转逻辑：

```swift
class Router: ObservableObject {
    @Published var path = NavigationPath()

    private var routeHandlers: [String: (Any) -> Void] = [:]

    func register<T: Hashable>(route: AppRoute, handler: @escaping (T) -> Void) {
        routeHandlers[route.hashValue.description] = { value in
            if let typedValue = value as? T {
                handler(typedValue)
            }
        }
    }

    func navigate(to route: AppRoute) {
        path.append(route)
    }

    func replace(with routes: [AppRoute]) {
        path.removeLast(path.count)
        for route in routes {
            path.append(route)
        }
    }

    func pop(to routeCount: Int) {
        let currentCount = path.count
        if routeCount < currentCount {
            path.removeLast(currentCount - routeCount)
        }
    }
}
```

### 大型项目的导航架构方案

在大型项目中，推荐采用以下分层导航架构：

| 层级 | 职责 | 组件 |
|---|---|---|
| 路由层 | 定义所有路由类型和参数 | AppRoute 枚举 |
| 协调层 | 管理导航流程和状态 | Coordinator 协议 |
| 视图层 | 绑定导航状态，渲染界面 | NavigationStack + navigationDestination |
| 持久层 | 保存和恢复导航状态 | NavigationPath.CodableValue |

```swift
enum AppRoute: Hashable, Codable {
    case home
    case detail(id: Int)
    case profile(userId: String)
    case settings
    case editProfile
}

class AppCoordinator: ObservableObject {
    @Published var path = NavigationPath()

    func handle(_ route: AppRoute) {
        switch route {
        case .home:
            path.removeLast(path.count)
        case .detail, .profile, .settings, .editProfile:
            path.append(route)
        }
    }

    func handleDeepLink(url: URL) {
        guard let host = url.host else { return }
        switch host {
        case "detail":
            if let id = url.pathComponents.last.flatMap(Int.init) {
                path.removeLast(path.count)
                path.append(AppRoute.detail(id: id))
            }
        case "profile":
            if url.pathComponents.count > 1 {
                path.removeLast(path.count)
                path.append(AppRoute.profile(userId: url.pathComponents[1]))
            }
        default:
            break
        }
    }
}
```

> 💡 **提示**：大型项目中，建议为每个功能模块创建独立的 Coordinator，再由一个主 Coordinator 统一协调。这样既保持了模块的独立性，又能在模块间实现灵活的跳转。

---

## 常见问题与最佳实践

### 导航栏自定义样式

SwiftUI 提供了多种导航栏样式和修饰符来自定义导航栏外观：

```swift
struct CustomNavBarView: View {
    var body: some View {
        NavigationStack {
            ContentView()
                .navigationTitle("首页")
                .navigationBarTitleDisplayMode(.inline)
                .toolbarColorScheme(.dark, for: .navigationBar)
                .toolbarBackground(.blue, for: .navigationBar)
                .toolbarBackgroundVisibility(.visible, for: .navigationBar)
                .toolbar {
                    ToolbarItem(placement: .topBarLeading) {
                        Button("编辑") { }
                    }
                    ToolbarItem(placement: .topBarTrailing) {
                        Button("添加") { }
                    }
                }
        }
    }
}
```

| 修饰符 | 说明 |
|---|---|
| `.navigationBarTitleDisplayMode(.large)` | 大标题模式 |
| `.navigationBarTitleDisplayMode(.inline)` | 内联标题模式 |
| `.toolbarBackground(.visible, for: .navigationBar)` | 显示导航栏背景 |
| `.toolbarColorScheme(.dark, for: .navigationBar)` | 导航栏深色方案 |
| `.toolbar(.hidden, for: .navigationBar)` | 隐藏导航栏 |

### 返回按钮行为控制

默认情况下，SwiftUI 的返回按钮会弹出导航栈的顶层视图。如果需要在返回时执行额外逻辑，可以使用 `toolbar` 自定义返回按钮：

```swift
struct DetailWithBackAction: View {
    @Environment(\.dismiss) private var dismiss
    @State private var hasUnsavedChanges = false
    @State private var showDiscardAlert = false

    var body: some View {
        ContentWebView()
            .navigationTitle("编辑")
            .navigationBarBackButtonHidden(hasUnsavedChanges)
            .toolbar {
                if hasUnsavedChanges {
                    ToolbarItem(placement: .topBarLeading) {
                        Button("返回") {
                            showDiscardAlert = true
                        }
                    }
                }
            }
            .alert("放弃更改？", isPresented: $showDiscardAlert) {
                Button("放弃", role: .destructive) { dismiss() }
                Button("取消", role: .cancel) { }
            }
    }
}
```

### TabView 与 NavigationStack 的嵌套

在 TabView 中，每个 Tab 应该拥有独立的 NavigationStack，避免导航状态混乱：

```swift
struct MainTabView: View {
    @State private var selectedTab = 0

    var body: some View {
        TabView(selection: $selectedTab) {
            NavigationStack {
                HomeView()
                    .navigationTitle("首页")
            }
            .tabItem {
                Label("首页", systemImage: "house")
            }
            .tag(0)

            NavigationStack {
                ExploreView()
                    .navigationTitle("探索")
            }
            .tabItem {
                Label("探索", systemImage: "magnifyingglass")
            }
            .tag(1)

            NavigationStack {
                ProfileView()
                    .navigationTitle("我的")
            }
            .tabItem {
                Label("我的", systemImage: "person")
            }
            .tag(2)
        }
    }
}
```

> ⚠️ **警告**：不要将 `NavigationStack` 放在 `TabView` 外层包裹所有 Tab。这会导致所有 Tab 共享同一个导航栈，切换 Tab 时导航状态会混乱。每个 Tab 必须有自己独立的 `NavigationStack`。

### 内存管理与导航栈泄漏

NavigationStack 中的视图在出栈后应该被释放。常见的内存泄漏原因包括：

1. **闭包循环引用** — NavigationLink 的 destination 闭包或 `navigationDestination` 闭包中强引用了父视图。

2. **定时器未停止** — 视图中的 Timer 或 Combine 订阅在视图出栈后仍在运行。

3. **观察者未移除** — NotificationCenter 观察者在 `onDisappear` 中未移除。

```swift
struct SafeDetailView: View {
    @State private var timer: Timer?
    @State private var count = 0

    var body: some View {
        Text("计数: \(count)")
            .navigationTitle("详情")
            .onAppear {
                timer = Timer.scheduledTimer(withTimeInterval: 1, repeats: true) { _ in
                    count += 1
                }
            }
            .onDisappear {
                timer?.invalidate()
                timer = nil
            }
    }
}
```

> 💡 **提示**：使用 Instruments 的 Allocations 工具检查导航栈的内存行为。反复 push/pop 同一页面，观察内存是否持续增长。如果增长，说明存在泄漏。

---

## 小结

| 知识点 | 核心内容 |
|---|---|
| 导航体系演进 | NavigationView → NavigationStack + NavigationSplitView，关注点分离、数据驱动、类型安全 |
| NavigationStack 基础 | 替代 NavigationView，支持声明式和编程式导航 |
| navigationDestination | 类型安全路由，将数据类型与目标视图绑定 |
| NavigationPath | 类型擦除的导航栈，支持 append/removeLast/count 编程式控制 |
| 值类型路由 vs 字符串路由 | 枚举路由编译期安全，字符串路由灵活但易出错 |
| 深层链接 | 通过 NavigationPath 直接构建任意导航路径 |
| 导航状态保存 | NavigationPath 支持 Codable，可序列化持久化 |
| NavigationSplitView 两栏 | sidebar + detail，适合主从布局 |
| NavigationSplitView 三栏 | sidebar + content + detail，适合层级浏览 |
| columnVisibility | 精细控制各栏显示状态 |
| 设备自适应 | iPhone 自动转栈式，iPad 并排展示，Mac 标准侧边栏 |
| 自定义转场 | .navigationTransition() + .zoom/.slide，iOS 18+ |
| MVVM 导航管理 | ViewModel 持有 NavigationPath，View 绑定渲染 |
| Coordinator 模式 | 抽离导航逻辑，统一管理跳转流程 |
| 路由管理器 | 集中式路由注册和跳转，适合大型项目 |
| 导航栏自定义 | toolbarBackground、toolbarColorScheme 等修饰符 |
| 返回按钮控制 | navigationBarBackButtonHidden + 自定义 toolbar |
| TabView 嵌套 | 每个 Tab 独立 NavigationStack，避免状态混乱 |
| 内存管理 | 注意闭包循环引用、定时器清理、观察者移除 |

← [自定义组件与样式系统](./自定义组件与样式.md) | [数据持久化方案](./数据持久化.md) →