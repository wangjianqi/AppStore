---
name: project-architecture
description: 创建新文件、新模块、新功能、重构现有代码、设计数据流、MVVM 架构、依赖注入、模块划分的任务
---

# 项目架构 DNA

## 目录结构
```
App/
├── Features/              # 功能模块（每个模块自包含）
│   ├── Camera/
│   │   ├── CameraVC.swift
│   │   ├── CameraViewModel.swift
│   │   ├── CameraService.swift
│   │   └── Views/
│   │       ├── CameraPreviewView.swift
│   │       └── CameraControlView.swift
│   ├── Settings/
│   │   ├── SettingsVC.swift
│   │   ├── SettingsViewModel.swift
│   │   └── SettingsService.swift
│   └── Paywall/
│       ├── PaywallVC.swift
│       ├── PaywallViewModel.swift
│       └── Views/
│           ├── PlanCardView.swift
│           └── FeatureListView.swift
├── Core/                  # 跨模块基础设施
│   ├── Network/           # API 层
│   │   ├── NetworkService.swift
│   │   ├── APIEndpoint.swift
│   │   └── NetworkError.swift
│   ├── Storage/           # 本地持久化
│   │   ├── CoreDataStack.swift
│   │   ├── KeychainStorage.swift
│   │   └── UserDefaultsStorage.swift
│   ├── Analytics/         # 埋点
│   │   └── AnalyticsManager.swift
│   └── Extensions/        # Swift 扩展
│       ├── String+Localized.swift
│       ├── UIView+Constraints.swift
│       └── UIColor+Hex.swift
├── DesignSystem/          # UI 组件库
│   ├── AppColors.swift
│   ├── AppFonts.swift
│   ├── Layout.swift
│   └── Components/        # 可复用 UI 组件
│       ├── LoadingView.swift
│       ├── EmptyStateView.swift
│       └── PrimaryButton.swift
└── Resources/
    ├── Assets.xcassets
    ├── Localizable.strings
    └── Models/            # CoreML 模型文件
```

---

## 架构模式：MVVM

```
View (VC + UIView)
  ↕ 绑定（closure / Combine）
ViewModel（业务逻辑 + 状态）
  ↕ 调用
Service / Repository（数据层）
```

- **VC 职责：** 初始化视图、绑定 ViewModel、响应用户事件
- **ViewModel 职责：** 处理业务逻辑、持有状态、调用 Service
- **Service 职责：** 网络请求、本地存储、设备能力（相机、麦克风）
- **禁止跨层直接调用**（VC 不直接调 Service，ViewModel 不操作 UIView）

---

## ViewModel 完整模板

### Closure 绑定模式

```swift
import Foundation

protocol HomeViewModelProtocol {
    var onItemsUpdated: (() -> Void)? { get set }
    var onError: ((AppError) -> Void)? { get set }
    var onLoadingStateChanged: ((Bool) -> Void)? { get set }
    var items: [Item] { get }
    func loadData()
    func didSelectItem(at index: Int)
}

final class HomeViewModel: HomeViewModelProtocol {
    var onItemsUpdated: (() -> Void)?
    var onError: ((AppError) -> Void)?
    var onLoadingStateChanged: ((Bool) -> Void)?

    private(set) var items: [Item] = [] {
        didSet { onItemsUpdated?() }
    }

    private let service: ItemServiceProtocol
    private weak var navigation: HomeNavigation?

    init(service: ItemServiceProtocol, navigation: HomeNavigation) {
        self.service = service
        self.navigation = navigation
    }

    func loadData() {
        onLoadingStateChanged?(true)
        service.fetchItems { [weak self] result in
            self?.onLoadingStateChanged?(false)
            switch result {
            case .success(let items):
                self?.items = items
            case .failure(let error):
                self?.onError?(error)
            }
        }
    }

    func didSelectItem(at index: Int) {
        guard index < items.count else { return }
        navigation?.showDetail(itemID: items[index].id)
    }
}
```

### Combine 绑定模式

```swift
import Combine

final class HomeViewModel: ObservableObject {
    @Published var items: [Item] = []
    @Published var isLoading: Bool = false
    @Published var errorMessage: String?

    private let service: ItemServiceProtocol
    private var cancellables = Set<AnyCancellable>()

    init(service: ItemServiceProtocol) {
        self.service = service
    }

    func loadData() {
        isLoading = true
        service.fetchItemsPublisher()
            .receive(on: DispatchQueue.main)
            .sink { [weak self] completion in
                self?.isLoading = false
                if case .failure(let error) = completion {
                    self?.errorMessage = error.localizedDescription
                }
            } receiveValue: { [weak self] items in
                self?.items = items
            }
            .store(in: &cancellables)
    }
}
```

---

## Service 层模板

### 协议优先

```swift
protocol AuthServiceProtocol {
    func login(email: String, password: String) async throws -> User
    func logout() async throws
    func refreshToken() async throws -> String
    var currentUser: User? { get }
}

final class AuthService: AuthServiceProtocol {
    private let network: NetworkServiceProtocol
    private let keychain: KeychainStorage

    weak var currentUser: User?

    init(network: NetworkServiceProtocol, keychain: KeychainStorage) {
        self.network = network
        self.keychain = keychain
    }

    func login(email: String, password: String) async throws -> User {
        let request = AuthAPI.login(email: email, password: password)
        let response: AuthResponse = try await network.request(request)
        try keychain.saveToken(response.token)
        currentUser = response.user
        return response.user
    }

    func logout() async throws {
        try keychain.deleteToken()
        currentUser = nil
    }

    func refreshToken() async throws -> String {
        let request = AuthAPI.refreshToken
        let response: TokenResponse = try await network.request(request)
        try keychain.saveToken(response.token)
        return response.token
    }
}
```

---

## Repository 层模板

```swift
protocol ItemRepositoryProtocol {
    func fetchItems() async throws -> [Item]
    func saveItem(_ item: Item) async throws
    func deleteItem(id: String) async throws
}

final class ItemRepository: ItemRepositoryProtocol {
    private let network: NetworkServiceProtocol
    private let storage: CoreDataStack

    init(network: NetworkServiceProtocol, storage: CoreDataStack) {
        self.network = network
        self.storage = storage
    }

    func fetchItems() async throws -> [Item] {
        do {
            let remote: [ItemDTO] = try await network.request(ItemAPI.list)
            let items = remote.map { $0.toDomain() }
            try await cacheItems(items)
            return items
        } catch {
            return try await cachedItems()
        }
    }

    func saveItem(_ item: Item) async throws {
        try await network.request(ItemAPI.create(item))
    }

    func deleteItem(id: String) async throws {
        try await network.request(ItemAPI.delete(id: id))
    }

    private func cacheItems(_ items: [Item]) async throws {
        try await storage.performBackgroundTask { context in
            // 缓存逻辑
        }
    }

    private func cachedItems() async throws -> [Item] {
        let context = storage.viewContext
        // 读取缓存
        return []
    }
}
```

---

## 依赖注入

### 构造器注入（推荐）

```swift
// ✅ 正确：通过 init 注入
final class HomeVC: UIViewController {
    private let viewModel: HomeViewModel

    init(viewModel: HomeViewModel) {
        self.viewModel = viewModel
        super.init(nibName: nil, bundle: nil)
    }
}

// 组装
let service = AuthService(network: networkService, keychain: keychainStorage)
let viewModel = HomeViewModel(service: service, navigation: router)
let vc = HomeVC(viewModel: viewModel)
```

### 协议注入（测试场景）

```swift
final class HomeViewModel {
    private let service: ItemServiceProtocol

    // 生产环境注入真实 Service
    // 测试环境注入 Mock Service
    init(service: ItemServiceProtocol = ItemService()) {
        self.service = service
    }
}

// Mock
final class MockItemService: ItemServiceProtocol {
    var stubbedItems: [Item] = []
    func fetchItems(completion: @escaping (Result<[Item], Error>) -> Void) {
        completion(.success(stubbedItems))
    }
}

// 测试
let mockService = MockItemService()
mockService.stubbedItems = [Item(id: "1", title: "Test")]
let viewModel = HomeViewModel(service: mockService)
```

### 禁止的 DI 模式
- **禁止 Singleton 滥用**：只有真正的全局单例（CoreDataStack、KeychainStorage）才用 `.shared`
- **禁止 Service Locator**：不要创建全局容器手动 resolve
- **禁止在 ViewModel 中直接创建 Service 实例**，必须通过 init 注入

---

## 导航协议

```swift
protocol HomeNavigation: AnyObject {
    func showDetail(itemID: String)
    func showSettings()
    func showPaywall()
}

final class HomeRouter: HomeNavigation {
    private weak var navigationController: UINavigationController?

    init(navigationController: UINavigationController?) {
        self.navigationController = navigationController
    }

    func showDetail(itemID: String) {
        let service = ItemService(network: NetworkService.shared)
        let vm = DetailViewModel(service: service, itemID: itemID)
        let vc = DetailVC(viewModel: vm)
        navigationController?.pushViewController(vc, animated: true)
    }

    func showSettings() {
        let vm = SettingsViewModel()
        let vc = SettingsVC(viewModel: vm)
        navigationController?.pushViewController(vc, animated: true)
    }

    func showPaywall() {
        let vm = PaywallViewModel()
        let vc = PaywallVC(viewModel: vm)
        vc.modalPresentationStyle = .fullScreen
        navigationController?.present(vc, animated: true)
    }
}
```

- ViewModel 通过 `weak var navigation: HomeNavigation?` 持有导航协议
- **禁止 ViewModel 直接 import UIKit** 或创建 VC

---

## 命名约定

| 类型 | 规范 | 示例 |
|------|------|------|
| ViewController | XxxVC | `CameraVC`, `SettingsVC` |
| ViewModel | XxxViewModel | `CameraViewModel` |
| Service | XxxService | `CameraService`, `AuthService` |
| Repository | XxxRepository | `ItemRepository` |
| Protocol | XxxProtocol 或 Xxxable | `CameraServiceProtocol`, `Cacheable` |
| 导航协议 | XxxNavigation | `HomeNavigation`, `CameraNavigation` |
| 枚举 | 大写驼峰，case 小写 | `AppError.networkTimeout` |
| 常量 | enum + static let | `Layout.padding = 16` |
| 扩展文件 | Type+Feature | `String+Localized.swift` |

---

## 错误处理

### 统一错误枚举

```swift
enum AppError: LocalizedError {
    case network(NetworkError)
    case storage(StorageError)
    case permission(PermissionError)
    case business(code: Int, message: String)

    var errorDescription: String? {
        switch self {
        case .network(let error): return error.localizedDescription
        case .storage(let error): return error.localizedDescription
        case .permission(let error): return error.localizedDescription
        case .business(_, let message): return message
        }
    }
}

enum NetworkError: LocalizedError {
    case noConnection
    case timeout
    case serverError(Int)
    case unauthorized

    var errorDescription: String? {
        switch self {
        case .noConnection: return "网络连接失败，请检查网络后重试"
        case .timeout: return "请求超时，请稍后重试"
        case .serverError: return "服务异常，请稍后重试"
        case .unauthorized: return "登录已过期，请重新登录"
        }
    }
}
```

- 所有错误统一走 `AppError` 枚举（避免裸 `Error` 类型扩散）
- 用户可见错误必须有**中文提示文案**
- **禁止 `try!` 和强制解包 `!`**（除非有充分注释说明原因）
- 错误日志统一通过 `Logger.error()` 输出（不用 `print`）

---

## 数据流

### 状态管理
- ViewModel 用 `@Published` + Combine / closure 回调（选一种，全项目统一）
- **禁止在 VC 中持有业务状态**，状态只存在于 ViewModel

### 数据流向
```
用户操作 → VC → ViewModel.method()
ViewModel → Service.method()
Service → Network / Storage
Service → ViewModel (callback / async)
ViewModel.state → VC (binding / @Published)
```

### 存储优先级
- UserDefaults（轻量配置）> CoreData（结构化数据）> FileManager（文件）
- 敏感数据（token、密钥）：**必须存 Keychain**，禁止存 UserDefaults

---

## 新功能开发流程

1. 在 `Features/` 下建对应目录
2. 先定义 Protocol（接口），再写实现
3. ViewModel 先写，VC 后写
4. 单元测试文件与源文件同目录（`XxxViewModelTests.swift`）
5. 新增第三方库必须在 README 的依赖列表更新

### 新功能 Checklist

```markdown
- [ ] 创建 Features/Xxx/ 目录
- [ ] 定义 XxxServiceProtocol
- [ ] 实现 XxxService
- [ ] 定义 XxxNavigation 协议
- [ ] 实现 XxxViewModel（含 Protocol）
- [ ] 实现 XxxVC
- [ ] 实现 XxxRouter
- [ ] 在 SceneDelegate / TabBar 中注册路由
- [ ] 编写 XxxViewModelTests
- [ ] 更新 README 依赖列表（如有新库）
```

---

## 依赖管理
- 包管理器：**Swift Package Manager**（禁止新增 CocoaPods 依赖）
- 引入新库前评估：是否有系统 API 替代？维护是否活跃？

### 常用依赖评估

| 需求 | 推荐方案 | 禁止方案 |
|------|---------|---------|
| 自动布局 | SnapKit | PureLayout、Masonry |
| 图片加载 | Kingfisher | SDWebImage（除非已有） |
| 网络 | URLSession 封装 | Alamofire（除非已有） |
| 数据库 | CoreData / SwiftData | Realm（除非已有） |
| Keychain | 自封装 | KeychainAccess（除非已有） |

---

## 常见架构反模式

| 反模式 | 问题 | 正确做法 |
|--------|------|---------|
| Massive VC | VC 超过 400 行，含业务逻辑 | 拆分到 ViewModel + Service |
| ViewModel 操作 UIView | 违反 MVVM 分层 | ViewModel 只发状态，VC 响应更新 UI |
| 直接在 VC 中调 Service | 跳过 ViewModel 层 | VC → ViewModel → Service |
| Singleton 滥用 | 隐式依赖，难以测试 | 构造器注入 + Protocol |
| 散落的权益判断 | 多处 `if isPro` 判断 | 统一走 `SubscriptionManager` |
| 硬编码跳转 | VC 中直接 `pushViewController` | 走 Router / Navigation 协议 |
| 全局状态变量 | `var xxx` 在 AppDelegate | 封装到对应 Service/Manager |
| 循环引用 | closure 未 `[weak self]` | 检查所有 closure 和 delegate |

---

## 代码质量
- 单个文件不超过 400 行（超出考虑拆分）
- 单个函数不超过 50 行
- 禁止注释掉的废弃代码（直接删除，Git 有历史）
- `MARK: -` 分组：Properties / Lifecycle / Setup / Binding / Actions / Private
