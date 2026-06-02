# Swift/SwiftUI 最优三方库指南

> 🎯 **本章目标**：在 18 个常用开发类别中，各推荐一个最优方案（优先原生，其次最成熟的第三方库），每个推荐附带选型理由和实战代码示例，帮你快速做出技术选型决策，避免"选择困难症"。

---

## 选型原则

```text
四条铁律：

1. 原生优先：Apple 已提供成熟方案的场景，不引入第三方依赖
2. 成熟优先：社区验证多年、文档完善、持续维护的库优先
3. 轻量优先：功能单一、无传递依赖的库优先
4. 最小化原则：每引入一个依赖 = 增加一份维护成本和安全风险
```

> 💡 **提示**：本章与 [依赖管理与开源库](../06-项目实战/依赖管理与开源库.md) 互补——那章讲"怎么管理依赖"，这章讲"选哪个依赖"。

---

## 总览表

| # | 类别 | 推荐方案 | 类型 | 一句话理由 |
|---|------|---------|------|-----------|
| 1 | 网络请求 | Alamofire | 第三方 | Swift 网络请求事实标准 |
| 2 | 图片加载 | Kingfisher | 第三方 | 一行代码完成加载+缓存 |
| 3 | JSON 解析 | Codable | 原生 | 零依赖、编译期类型安全 |
| 4 | 依赖注入 | Factory | 第三方 | 最轻量现代的 Swift DI |
| 5 | 代码生成 | Sourcery | 第三方 | 最强大的元编程工具 |
| 6 | 导航/架构 | TCA | 第三方 | 最完整的 SwiftUI 架构方案 |
| 7 | Keychain | KeychainAccess | 第三方 | 一行代码存取 Keychain |
| 8 | 日志 | OSLog + swift-log | 原生 | Instruments 深度集成 |
| 9 | 分析/崩溃 | Firebase | 第三方 | 免费无限制的分析+崩溃收集 |
| 10 | 图表 | Swift Charts | 原生 | SwiftUI 声明式图表，零依赖 |
| 11 | 日期时间 | Date + FormatStyle | 原生 | iOS 15+ 原生已足够强大 |
| 12 | 数据验证 | ValidatedPropertyKit | 第三方 | 声明式属性验证 |
| 13 | 权限管理 | PermissionsSwiftUI | 第三方 | SwiftUI 一站式权限管理 |
| 14 | SwiftUI 补全 | SwiftUIX | 第三方 | 填补 SwiftUI API 空白 |
| 15 | 测试/Mock | Swift Testing + Mockolo | 原生+第三方 | 新一代测试+自动 Mock |
| 16 | 数据库 | GRDB.swift | 第三方 | 最完善的 Swift SQLite 封装 |
| 17 | 响应式 | Combine | 原生 | 与 SwiftUI 天然集成 |
| 18 | Toast/提示 | AlertToast | 第三方 | 最简洁的 SwiftUI Toast |

---

## 1. 网络请求 — Alamofire

| 项目 | 详情 |
|------|------|
| GitHub | https://github.com/Alamofire/Alamofire |
| Stars | ~41k |
| 许可证 | MIT |

Swift 生态中网络请求的事实标准。封装 `URLSession`，提供链式 API、拦截器、重试策略、上传下载进度、证书绑定等。

```text
什么时候用 Alamofire：
✅ 项目中网络请求较多（>5 个接口），需要统一的请求/响应处理
✅ 需要请求拦截器、重试策略、证书绑定等高级功能
✅ 团队已有 Alamofire 使用经验

什么时候用 URLSession：
✅ 只有 1-2 个简单接口，不值得引入依赖
✅ 对包大小极度敏感
✅ ios-claude-skills 规范要求"原生优先"
```

```swift
import Alamofire

struct APIClient {
    static let shared = APIClient()

    func fetchUsers() async throws -> [User] {
        try await AF.request("https://api.example.com/users")
            .validate(statusCode: 200..<300)
            .serializingDecodable([User].self)
            .value
    }

    func createUser(name: String, email: String) async throws -> User {
        try await AF.request("https://api.example.com/users", method: .post,
                             parameters: ["name": name, "email": email],
                             encoding: JSONEncoding.default)
            .validate()
            .serializingDecodable(User.self)
            .value
    }

    func uploadImage(_ data: Data) async throws -> URL {
        try await AF.upload(multipartFormData: { formData in
            formData.append(data, withName: "file", fileName: "photo.jpg", mimeType: "image/jpeg")
        }, to: "https://api.example.com/upload")
        .validate()
        .serializingDecodable(UploadResponse.self)
        .value
        .url
    }
}
```

---

## 2. 图片加载 — Kingfisher

| 项目 | 详情 |
|------|------|
| GitHub | https://github.com/onevcat/Kingfisher |
| Stars | ~23k |
| 许可证 | MIT |

由王巍（onevcat）开发，支持网络图片下载、内存/磁盘二级缓存、渐进式加载、图片处理（圆角/缩放/滤镜），SwiftUI 原生支持。

```text
为什么不用 AsyncImage：
SwiftUI 的 AsyncImage 没有磁盘缓存，每次重启 App 都重新下载
Kingfisher 提供内存+磁盘二级缓存，离线也能显示图片
```

```swift
import Kingfisher

struct UserAvatarView: View {
    let urlString: String

    var body: some View {
        KFImage(URL(string: urlString))
            .placeholder { ProgressView() }
            .resizable()
            .scaledToFill()
            .frame(width: 80, height: 80)
            .clipShape(Circle())
            .onFailure { error in
                print("图片加载失败: \(error)")
            }
    }
}

struct CachedImageView: View {
    let url: String

    var body: some View {
        KFImage(URL(string: url))
            .setProcessor(DownsamplingImageProcessor(size: CGSize(width: 200, height: 200)))
            .scaleFactor(UIScreen.main.scale)
            .cacheOriginalImage()
            .fade(duration: 0.25)
            .resizable()
            .aspectRatio(contentMode: .fill)
    }
}
```

---

## 3. JSON 解析 — Codable（原生）

Swift 4 引入的原生协议，零依赖、编译期类型安全。社区已形成共识：**新项目优先 Codable，SwiftyJSON 逐渐退出主流**。

```swift
struct User: Codable, Identifiable {
    let id: Int
    let name: String
    let email: String
    let avatarURL: String?

    enum CodingKeys: String, CodingKey {
        case id, name, email
        case avatarURL = "avatar_url"
    }
}

let decoder = JSONDecoder()
decoder.keyDecodingStrategy = .convertFromSnakeCase
decoder.dateDecodingStrategy = .iso8601

let users = try decoder.decode([User].self, from: jsonData)

let encoder = JSONEncoder()
encoder.keyEncodingStrategy = .convertToSnakeCase
encoder.outputFormatting = [.prettyPrinted, .sortedKeys]
let data = try encoder.encode(users)
```

---

## 4. 依赖注入 — Factory

| 项目 | 详情 |
|------|------|
| GitHub | https://github.com/hmlongco/Factory |
| Stars | ~2k |
| 许可证 | MIT |

2024-2025 年社区增长最快的 Swift DI 框架。比 Swinject 更轻量、更现代，原生支持 SwiftUI 的 `@Injected` 属性包装器。

```swift
import Factory

extension Container {
    var userService: Factory<UserServiceProtocol> {
        Factory { UserServiceImpl() }
    }
    var apiClient: Factory<APIClient> {
        Factory { APIClient() }.singleton
    }
    var repository: Factory<DataRepository> {
        Factory { DataRepository(apiClient: Container.shared.apiClient()) }
    }
}

class UserListViewModel: ObservableObject {
    @Injected(\.userService) var userService
    @Injected(\.repository) var repository

    @Published var users: [User] = []

    func load() async {
        users = (try? await userService.fetchUsers()) ?? []
    }
}

Container.shared.userService.register { MockUserService() }
```

---

## 5. 代码生成 — Sourcery

| 项目 | 详情 |
|------|------|
| GitHub | https://github.com/krzysztofzablocki/Sourcery |
| Stars | ~7.6k |
| 许可证 | MIT |

Swift 元编程工具，通过 Stencil 模板自动生成样板代码。与 Swift Macros 互补：宏处理编译期简单展开，Sourcery 处理大规模批量生成。

```text
Sourcery vs Swift Macros：

Swift Macros：编译期展开，适合简单场景（#unwrap、#observable）
Sourcery：构建前批量生成，适合复杂场景（批量 Mock、AutoEquatable、DTO 映射）

推荐：简单用宏，复杂用 Sourcery，两者不冲突
```

```swift
// 源码中标记
// sourcery: AutoMockable
protocol UserServiceProtocol {
    func fetchUser(id: Int) async throws -> User
    func updateUser(_ user: User) async throws -> Bool
}

// 运行 Sourcery 自动生成 MockUserService
// sourcery --sources ./Sources --templates ./Templates --output ./Generated
```

> 💡 **补充**：资源类型安全访问推荐 [R.swift](https://github.com/mac-cain13/R.swift)（~9.4k Stars）或 [SwiftGen](https://github.com/SwiftGen/SwiftGen)（~9.3k Stars）。

---

## 6. 导航/架构 — TCA

| 项目 | 详情 |
|------|------|
| GitHub | https://github.com/pointfreeco/swift-composable-architecture |
| Stars | ~12k |
| 许可证 | MIT |

Point-Free 团队出品的 SwiftUI 架构框架，提供状态驱动的导航、依赖注入、可测试的 Reducer 模式。SwiftUI 中最成熟的类型安全路由方案。

```text
TCA 适合什么项目：
✅ 中大型项目，需要严格的架构约束
✅ 导航逻辑复杂（深层链接、条件导航）
✅ 需要高测试覆盖率
✅ 团队多人协作，需要统一架构模式

TCA 不适合什么项目：
❌ 小型/原型项目（过度设计）
❌ 团队不熟悉函数式编程范式
❌ 纯 UIKit 项目
```

```swift
import ComposableArchitecture

@Reducer
struct FeatureList {
    @ObservableState
    struct State {
        var items: [Item] = []
        var path = StackState<FeatureDetail.State>()
    }

    enum Action {
        case onAppear
        case itemsLoaded([Item])
        case path(StackAction<FeatureDetail.State, FeatureDetail.Action>)
    }

    @Dependency(\.itemClient) var itemClient

    var body: some ReducerOf<Self> {
        Reduce { state, action in
            switch action {
            case .onAppear:
                return .run { send in
                    let items = try await itemClient.fetch()
                    await send(.itemsLoaded(items))
                }
            case .itemsLoaded(let items):
                state.items = items
                return .none
            case .path:
                return .none
            }
        }
        .forEach(\.path, action: \.path)
    }
}
```

---

## 7. Keychain — KeychainAccess

| 项目 | 详情 |
|------|------|
| GitHub | https://github.com/kishikawakatsumi/KeychainAccess |
| Stars | ~7.9k |
| 许可证 | MIT |

将繁琐的 Security 框架 API 简化为一行代码的存取操作，支持所有 Keychain Item 类型和 Keychain Sharing Group。

```swift
import KeychainAccess

let keychain = Keychain(service: "com.example.app")
    .synchronizable(true)
    .accessGroup("group.com.example")

try keychain.set("secret-token-123", key: "accessToken")
try keychain.set("user@example.com", key: "email")

let token = try keychain.get("accessToken")

try keychain.remove("accessToken")

for (key, value) in keychain {
    print("\(key): \(value)")
}
```

---

## 8. 日志 — OSLog + swift-log（原生）

Apple 官方方案。`Logger`（iOS 14+）与 Xcode Instruments 深度集成，`swift-log` 提供可插拔的 `LogHandler` 协议。

```swift
import os

let logger = Logger(subsystem: "com.example.app", category: "network")

logger.info("请求开始: \(url)")
logger.debug("响应数据: \(data.count) bytes")
logger.error("请求失败: \(error.localizedDescription)")

logger.log(level: .default, "重试第 \(attempt) 次")
```

> 💡 **提示**：`os_signpost` 支持性能追踪，可在 Instruments 中可视化分析，这是第三方日志库做不到的。

---

## 9. 分析/崩溃 — Firebase

| 项目 | 详情 |
|------|------|
| GitHub | https://github.com/firebase/firebase-ios-sdk |
| Stars | ~5.7k |
| 许可证 | Apache 2.0 |

Analytics 免费、无限制；Crashlytics 是 iOS 崩溃收集的事实标准。两者深度集成，SPM 原生支持。

```swift
import FirebaseAnalytics
import FirebaseCrashlytics

Analytics.logEvent("purchase_completed", parameters: [
    "item_id": "sku_12345",
    "item_name": "Premium Plan",
    "price": 9.99
])

Analytics.setUserProperty("premium", forName: "subscription_tier")

Crashlytics.crashlytics().setCustomValue("user_123", forKey: "user_id")
Crashlytics.crashlytics().setCustomValue(true, forKey: "is_premium")

Crashlytics.crashlytics().record(error: NSError(
    domain: "AppError", code: 1001,
    userInfo: [NSLocalizedDescriptionKey: "支付失败"]
))
```

---

## 10. 图表 — Swift Charts（原生）

iOS 16+ 原生图表框架，声明式 API 与 SwiftUI 深度集成。自动支持深色模式、Dynamic Type、VoiceOver 无障碍。

```swift
import Charts

struct RevenueChartView: View {
    let data: [MonthlyRevenue]

    var body: some View {
        Chart(data) { item in
            BarMark(
                x: .value("月份", item.month),
                y: .value("营收", item.revenue)
            )
            .foregroundStyle(.blue.gradient)
        }
        .chartYAxis {
            AxisMarks(position: .leading)
        }
        .frame(height: 250)
        .padding()
    }
}

struct MonthlyRevenue: Identifiable {
    let id = UUID()
    let month: String
    let revenue: Double
}
```

> 💡 **补充**：需要支持 iOS 15 以下或更复杂图表（散点图、雷达图），推荐 [DGCharts](https://github.com/danielgindi/Charts)（~27k Stars）。

---

## 11. 日期时间 — Date + FormatStyle（原生）

iOS 15+ 的 `FormatStyle` API 已足够强大，支持多语言、多地区、相对时间格式，无需第三方库。

```swift
let now = Date()

now.formatted(date: .long, time: .shortened)
now.formatted(.dateTime.year().month(.wide).day())
now.formatted(.relative(presentation: .named))

now.addingTimeInterval(-3600).formatted(.relative(presentation: .numeric))

let date = try Date("2025-06-02", strategy: .date)
let dateTime = try Date("2025-06-02T15:30:00Z", strategy: .iso8601)
```

> 💡 **补充**：如需复杂日期计算（时区转换、日历运算），推荐 [SwiftDate](https://github.com/malcommac/SwiftDate)（~7.7k Stars）。

---

## 12. 数据验证 — ValidatedPropertyKit

| 项目 | 详情 |
|------|------|
| GitHub | https://github.com/SvenTiigi/ValidatedPropertyKit |
| Stars | ~1.1k |
| 许可证 | MIT |

通过属性包装器声明式地验证值，与 SwiftUI 表单天然契合，零传递依赖。

```swift
import ValidatedPropertyKit

struct RegistrationForm {
    @Validated(!.isEmpty)
    var username: String = ""

    @Validated(.email)
    var email: String = ""

    @Validated(.count(8...))
    var password: String = ""

    var isValid: Bool {
        _username.isValid && _email.isValid && _password.isValid
    }
}

struct RegistrationView: View {
    @State private var form = RegistrationForm()

    var body: some View {
        Form {
            TextField("用户名", text: $form.username)
            if !form._username.isValid {
                Text("用户名不能为空").foregroundStyle(.red).font(.caption)
            }

            TextField("邮箱", text: $form.email)
            if !form._email.isValid {
                Text("请输入有效邮箱").foregroundStyle(.red).font(.caption)
            }

            SecureField("密码", text: $form.password)
            if !form._password.isValid {
                Text("密码至少 8 位").foregroundStyle(.red).font(.caption)
            }

            Button("注册") { /* ... */ }
                .disabled(!form.isValid)
        }
    }
}
```

---

## 13. 权限管理 — PermissionsSwiftUI

| 项目 | 详情 |
|------|------|
| GitHub | https://github.com/jevonmao/PermissionsSwiftUI |
| Stars | ~1.4k |
| 许可证 | MIT |

SwiftUI 原生权限请求库，一站式管理相机/相册/定位/通知等系统权限，自动检测权限状态。

```swift
import PermissionsSwiftUI

struct OnboardingView: View {
    @State private var showModal = false

    var body: some View {
        VStack {
            Text("欢迎使用 App")
            Button("设置权限") { showModal = true }
        }
        .JMPermissions(showModal: $showModal, for: [
            .camera,
            .location,
            .photo,
            .notification
        ])
        .setHeaderTitle("需要以下权限")
        .setHeaderSubtitle("以提供完整功能体验")
    }
}
```

---

## 14. SwiftUI 补全 — SwiftUIX

| 项目 | 详情 |
|------|------|
| GitHub | https://github.com/SwiftUIX/SwiftUIX |
| Stars | ~6.9k |
| 许可证 | MIT |

填补 SwiftUI 官方 API 的空白，提供 `VisualEffectView`、`SearchBar`、`NavigationController` 等缺失组件。优先使用官方方案，仅补充缺失部分。

```swift
import SwiftUIX

struct BlurOverlayView: View {
    var body: some View {
        ZStack {
            Color.blue.opacity(0.3)
            VisualEffectView(effect: UIBlurEffect(style: .systemMaterial))
        }
    }
}

struct SearchableListView: View {
    @State var searchText = ""
    let items: [Item]

    var body: some View {
        VStack {
            SearchBar(text: $searchText)
            List(filteredItems) { item in
                Text(item.name)
            }
        }
    }

    var filteredItems: [Item] {
        items.filter { searchText.isEmpty || $0.name.contains(searchText) }
    }
}
```

---

## 15. 测试/Mock — Swift Testing + Mockolo

| 项目 | 详情 |
|------|------|
| Mockolo GitHub | https://github.com/uber/mockolo |
| Mockolo Stars | ~1.4k |
| 许可证 | MIT |

Swift Testing 是 Apple WWDC23 推出的新一代测试框架（替代 XCTest）。Mockolo 是 Uber 开源的 Mock 生成器。

```swift
import Testing

struct UserServiceTests {
    @Test func fetchUserReturnsValidUser() async throws {
        let mockClient = MockAPIClient()
        mockClient.fetchUserHandler = { _ in User(id: 1, name: "Test") }

        let service = UserService(client: mockClient)
        let user = try await service.fetchUser(id: 1)

        #expect(user.name == "Test")
        #expect(user.id == 1)
        #expect(mockClient.fetchUserCallCount == 1)
    }

    @Test(.tags(.networking))
    func concurrentRequestsHandled() async {
        // ...
    }
}

// Mockolo 注解 → 自动生成 Mock
/// @mockable
protocol APIClientProtocol {
    func fetchUser(id: Int) async throws -> User
    func updateUser(_ user: User) async throws -> Bool
}
```

---

## 16. 数据库 — GRDB.swift

| 项目 | 详情 |
|------|------|
| GitHub | https://github.com/groue/GRDB.swift |
| Stars | ~7k |
| 许可证 | MIT |

Swift SQLite 库中最完善的方案。SwiftData（iOS 17+）虽是未来方向，但当前 API 仍不够成熟，复杂查询受限。GRDB 支持 Migration、FTS5 全文搜索、数据库加密、Value Observation 响应式查询。

```swift
import GRDB

struct Player: Codable, FetchableRecord, PersistableRecord {
    var id: Int64?
    var name: String
    var score: Int
}

var migrator = DatabaseMigrator()
migrator.registerV1("initial") { db in
    try db.create(table: "player") { t in
        t.autoIncrementedPrimaryKey("id")
        t.column("name", .text).notNull()
        t.column("score", .integer).notNull()
    }
}

let dbQueue = try DatabaseQueue(path: "/path/to/db.sqlite")
try migrator.migrate(dbQueue)

try dbQueue.write { db in
    try Player(name: "Alice", score: 100).insert(db)
    try Player(name: "Bob", score: 200).insert(db)
}

let topPlayers: [Player] = try dbQueue.read { db in
    try Player.filter(Column("score") > 50)
        .order(Column("score").desc)
        .limit(10)
        .fetchAll(db)
}

let observation = ValueObservation.tracking { db in
    try Player.fetchAll(db)
}
let cancellable = observation.start(in: dbQueue) { error in
    // 错误处理
} onChange: { (players: [Player]) in
    // 数据变化时自动回调
}
```

---

## 17. 响应式 — Combine（原生）

Apple 官方响应式框架，与 SwiftUI 天然集成。2025 年的共识：**新项目 Combine + async/await，不再引入 RxSwift**。

```swift
import Combine

class SearchViewModel: ObservableObject {
    @Published var searchText = ""
    @Published var results: [Item] = []

    private var cancellables = Set<AnyCancellable>()

    init(service: SearchService) {
        $searchText
            .debounce(for: .milliseconds(300), scheduler: RunLoop.main)
            .removeDuplicates()
            .filter { !$0.isEmpty }
            .flatMap { query in
                service.search(query: query)
                    .catch { _ in Just([]) }
            }
            .receive(on: DispatchQueue.main)
            .assign(to: &$results)
    }
}
```

---

## 18. Toast/提示 — AlertToast

| 项目 | 详情 |
|------|------|
| GitHub | https://github.com/elai950/AlertToast |
| Stars | ~2k |
| 许可证 | MIT |

纯 SwiftUI Toast 组件，两行代码展示提示，支持成功/错误/警告/加载样式，轻量无传递依赖。

```swift
import AlertToast

struct ContentView: View {
    @State private var showSuccess = false
    @State private var showError = false
    @State private var showLoading = false

    var body: some View {
        VStack(spacing: 20) {
            Button("保存成功") { showSuccess = true }
            Button("操作失败") { showError = true }
            Button("加载中") {
                showLoading = true
                DispatchQueue.main.asyncAfter(deadline: .now() + 2) {
                    showLoading = false
                }
            }
        }
        .toast(isPresenting: $showSuccess) {
            AlertToast(displayMode: .banner(.pop), type: .complete(.green), title: "保存成功!")
        }
        .toast(isPresenting: $showError) {
            AlertToast(displayMode: .banner(.pop), type: .error(.red), title: "操作失败")
        }
        .toast(isPresenting: $showLoading) {
            AlertToast(displayMode: .alert, type: .loading, title: "加载中...")
        }
    }
}
```

---

## 按需选型速查

### 我的项目需要什么？

```text
刚起步的小项目（1-3 个页面）：
→ Codable + URLSession + OSLog + Swift Testing
→ 尽量不引入第三方依赖

中等项目（5-15 个页面）：
→ + Kingfisher（图片加载）
→ + KeychainAccess（安全存储）
→ + AlertToast（提示组件）
→ + Firebase（分析+崩溃）

大型项目（15+ 页面，团队协作）：
→ + Alamofire（统一网络层）
→ + Factory（依赖注入）
→ + TCA 或 MVVM + Combine（架构）
→ + GRDB（本地数据库）
→ + Sourcery + Mockolo（代码生成+测试）
```

### 原生 vs 第三方决策树

```text
需要某个功能
  ↓
Apple 有原生方案吗？
  ├─ 有 → 原生方案够用吗？
  │        ├─ 够用 → 用原生 ✅
  │        └─ 不够用 → 引入第三方
  └─ 没有 → 有成熟的第三方库吗？
             ├─ 有 → Stars > 1k 且持续维护？
             │        ├─ 是 → 引入 ✅
             │        └─ 否 → 自己实现或找替代
             └─ 没有 → 自己实现
```

---

## 本章小结

- **7 个类别已有原生最优解**：Codable、Swift Charts、Combine、OSLog、Date+FormatStyle、Swift Testing、SwiftUI NavigationStack
- **11 个类别推荐第三方库**：Alamofire、Kingfisher、Factory、Sourcery、TCA、KeychainAccess、Firebase、ValidatedPropertyKit、PermissionsSwiftUI、SwiftUIX、GRDB、AlertToast、Mockolo
- 原生优先、成熟优先、轻量优先、最小化原则——四个维度做选型
- 项目规模决定依赖数量：小项目 0-2 个，中项目 3-5 个，大项目 6-10 个
- 每引入一个依赖就增加一份维护成本，能用原生就不用第三方

---

[← 上一章：SwiftUI 性能优化专题](SwiftUI性能优化专题.md) | [下一章：Core Data 入门（选读） →](Core-Data入门选读.md)
