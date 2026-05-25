# Swift 并发迁移实战

> 🎯 **本章目标**：掌握从传统回调模式到 Swift 并发（async/await、Actor）的渐进式迁移方法，学会处理 Completion Handler → async/await、Delegate → Actor、全局变量 → Actor 隔离等典型迁移场景，建立系统化的迁移工作流。

---

## 1. 迁移策略概述

### 1.1 为什么需要迁移

传统回调模式在小型项目中看似简单，但随着项目规模增长，三大痛点会越来越明显：

**回调地狱（Callback Hell）**

```swift
func loadUserProfile(userId: String, completion: @escaping (Result<User, Error>) -> Void) {
    fetchUser(id: userId) { result in
        switch result {
        case .success(let user):
            self.fetchAvatar(url: user.avatarURL) { avatarResult in
                switch avatarResult {
                case .success(let avatar):
                    self.fetchPreferences(userId: user.id) { prefResult in
                        switch prefResult {
                        case .success(let prefs):
                            let profile = UserProfile(user: user, avatar: avatar, prefs: prefs)
                            completion(.success(profile))
                        case .failure(let error):
                            completion(.failure(error))
                        }
                    }
                case .failure(let error):
                    completion(.failure(error))
                }
            }
        case .failure(let error):
            completion(.failure(error))
        }
    }
}
```

三层嵌套已经难以阅读，实际项目中五、六层嵌套并不罕见。而用 async/await 重写后，同样的逻辑变得线性清晰：

```swift
func loadUserProfile(userId: String) async throws -> UserProfile {
    let user = try await fetchUser(id: userId)
    let avatar = try await fetchAvatar(url: user.avatarURL)
    let prefs = try await fetchPreferences(userId: user.id)
    return UserProfile(user: user, avatar: avatar, prefs: prefs)
}
```

**错误处理复杂**

传统回调中，每一层都需要手动传递错误，遗漏任何一个分支都会导致回调永不执行。async/await 的 `try/catch` 机制让错误自动向上传播，不再需要手动逐层传递。

**难以调试**

回调代码的调用栈是断裂的——每个闭包都是独立的栈帧，崩溃时很难追踪完整的调用链。async/await 保持连续的调用栈，调试时能看到完整的异步调用路径。

### 1.2 渐进式迁移原则

> 💡 **提示**：迁移最忌讳"推倒重来"。正确的做法是逐模块、逐方法替换，确保每一步都可编译、可测试。

渐进式迁移的核心原则：

1. **新旧共存**：保留旧接口，新增 async 版本，逐步切换调用方
2. **自底向上**：先迁移底层工具方法，再迁移上层业务逻辑
3. **逐模块推进**：完成一个模块的迁移并验证后，再开始下一个
4. **编译器为友**：利用 Swift 编译器的并发警告指导迁移方向

### 1.3 迁移优先级

推荐按以下顺序迁移，因为底层迁移完成后，上层迁移会更容易：

| 优先级 | 层级 | 原因 | 典型迁移内容 |
|---|---|---|---|
| 🥇 第一 | 网络层 | 最独立、最常被调用、收益最大 | URLSession 回调 → async/await |
| 🥈 第二 | 数据层 | 依赖网络层，迁移后可简化存储逻辑 | 数据库回调 → Actor 隔离 |
| 🥉 第三 | 业务逻辑层 | 依赖数据层，迁移后可简化流程编排 | 多步骤异步操作 → async let |
| 4️⃣ 第四 | UI 层 | 依赖所有下层，迁移范围最大 | Delegate → async/await、@MainActor |

### 1.4 迁移风险评估

| 风险类型 | 风险等级 | 描述 | 应对策略 |
|---|---|---|---|
| 编译错误激增 | 🟡 中 | Swift 6 严格模式下大量警告变错误 | 先设为 Swift 5 模式，逐步提升检查级别 |
| 第三方库不兼容 | 🔴 高 | 未支持 Sendable 的第三方库导致编译错误 | 使用 @preconcurrency 导入、extension 桥接 |
| 行为差异 | 🟡 中 | async/await 执行语义与回调不同 | 编写测试验证行为一致性 |
| 性能回退 | 🟢 低 | Actor 隔离引入额外调度开销 | 性能测试对比，关键路径用 nonisolated |
| 团队学习成本 | 🟡 中 | 团队成员对 async/await 不熟悉 | 内部培训、代码评审中指导 |

---

## 2. Completion Handler → async/await

### 2.1 经典回调模式的问题

回顾一个典型的网络请求回调：

```swift
class APIClient {
    func request<T: Decodable>(
        url: URL,
        responseType: T.Type,
        completion: @escaping (Result<T, Error>) -> Void
    ) {
        let task = URLSession.shared.dataTask(with: url) { data, response, error in
            if let error = error {
                completion(.failure(error))
                return
            }
            guard let httpResponse = response as? HTTPURLResponse,
                  (200...299).contains(httpResponse.statusCode) else {
                completion(.failure(APIError.invalidResponse))
                return
            }
            guard let data = data else {
                completion(.failure(APIError.noData))
                return
            }
            do {
                let decoded = try JSONDecoder().decode(T.self, from: data)
                completion(.success(decoded))
            } catch {
                completion(.failure(error))
            }
        }
        task.resume()
    }
}
```

问题总结：

| 问题 | 说明 |
|---|---|
| 嵌套层级深 | 多个 guard + completion 调用，代码缩进严重 |
| 忘记调用 completion | 某个分支遗漏 completion，调用方永远等待 |
| 错误传递繁琐 | 每个错误分支都要手动调用 completion(.failure) |
| 无法使用 try/catch | 错误处理散落在各个闭包中 |
| 取消困难 | 需要手动管理 task 引用和取消逻辑 |

### 2.2 withCheckedContinuation 桥接回调

`withCheckedContinuation` 是将现有回调 API 桥接到 async/await 的核心工具。它将回调的"未来结果"包装成一个可 await 的值：

```swift
class APIClient {
    func request<T: Decodable>(url: URL, responseType: T.Type) async -> T {
        await withCheckedContinuation { continuation in
            self.request(url: url, responseType: responseType) { result in
                switch result {
                case .success(let value):
                    continuation.resume(returning: value)
                case .failure:
                    fatalError("Unexpected failure in non-throwing continuation")
                }
            }
        }
    }
}
```

> ⚠️ **警告**：`withCheckedContinuation` 中，`continuation.resume` 必须且只能调用一次。调用零次会导致永久挂起，调用多次会触发运行时崩溃。编译器会在调试模式下检查这一约束。

### 2.3 withCheckedThrowingContinuation 桥接可失败的回调

对于可能失败的回调，使用 `withCheckedThrowingContinuation`：

```swift
class APIClient {
    func request<T: Decodable>(url: URL, responseType: T.Type) async throws -> T {
        try await withCheckedThrowingContinuation { continuation in
            self.request(url: url, responseType: responseType) { result in
                switch result {
                case .success(let value):
                    continuation.resume(returning: value)
                case .failure(let error):
                    continuation.resume(throwing: error)
                }
            }
        }
    }
}
```

两个 Continuation 的区别：

| 特性 | withCheckedContinuation | withCheckedThrowingContinuation |
|---|---|---|
| 是否可抛出错误 | 否 | 是 |
| continuation.resume | `resume(returning:)` | `resume(returning:)` 和 `resume(throwing:)` |
| 适用场景 | 不会失败的回调 | 网络请求、文件 IO 等可失败操作 |
| 函数签名 | `async -> T` | `async throws -> T` |

### 2.4 URLSession 回调 → async/await 迁移

URLSession 从 iOS 15 起原生支持 async/await，迁移非常简单：

**迁移前：**

```swift
func fetchUser(id: String, completion: @escaping (Result<User, Error>) -> Void) {
    let url = URL(string: "https://api.example.com/users/\(id)")!
    let task = URLSession.shared.dataTask(with: url) { data, response, error in
        if let error = error {
            completion(.failure(error))
            return
        }
        guard let data = data else {
            completion(.failure(APIError.noData))
            return
        }
        do {
            let user = try JSONDecoder().decode(User.self, from: data)
            completion(.success(user))
        } catch {
            completion(.failure(error))
        }
    }
    task.resume()
}
```

**迁移后：**

```swift
func fetchUser(id: String) async throws -> User {
    let url = URL(string: "https://api.example.com/users/\(id)")!
    let (data, response) = try await URLSession.shared.data(from: url)
    guard let httpResponse = response as? HTTPURLResponse,
          (200...299).contains(httpResponse.statusCode) else {
        throw APIError.invalidResponse
    }
    return try JSONDecoder().decode(User.self, from: data)
}
```

代码行数从 16 行减少到 7 行，且逻辑更清晰。

### 2.5 第三方库回调 → async/await 封装

当第三方库只提供回调接口时，用 extension 添加 async 版本：

```swift
extension ThirdPartyImageLoader {
    func loadImage(from url: URL) async throws -> UIImage {
        try await withCheckedThrowingContinuation { continuation in
            self.loadImage(from: url) { result in
                switch result {
                case .success(let image):
                    continuation.resume(returning: image)
                case .failure(let error):
                    continuation.resume(throwing: error)
                }
            }
        }
    }
}
```

> 💡 **提示**：建议将桥接代码放在独立的 extension 文件中，与第三方库的原始代码分离，方便后续库升级时移除桥接层。

### 2.6 迁移前后代码对比

| 维度 | 回调模式 | async/await 模式 |
|---|---|---|
| 代码行数 | 多（每个错误分支都需要处理） | 少（错误自动传播） |
| 嵌套层级 | 深（每个异步操作增加一层） | 浅（线性流程） |
| 错误处理 | 手动传递 completion(.failure) | try/catch 自动传播 |
| 可读性 | 低（需要追踪闭包） | 高（类似同步代码） |
| 调试体验 | 差（调用栈断裂） | 好（连续调用栈） |
| 取消支持 | 需手动管理 | 通过 Task.cancel() 统一管理 |

---

## 3. Delegate → Actor

### 3.1 Delegate 模式的并发问题

Delegate 模式是 iOS 开发中最常见的设计模式之一，但在并发环境下存在隐患：

```swift
protocol DownloadManagerDelegate: AnyObject {
    func downloadManager(_ manager: DownloadManager, didProgress progress: Double)
    func downloadManager(_ manager: DownloadManager, didFinishWith data: Data)
    func downloadManager(_ manager: DownloadManager, didFailWithError error: Error)
}

class DownloadManager {
    weak var delegate: DownloadManagerDelegate?
    private var activeDownloads: [String: DownloadTask] = [:]

    func startDownload(url: String) {
        let task = DownloadTask(url: url)
        activeDownloads[url] = task
        task.onProgress = { [weak self] progress in
            self?.delegate?.downloadManager(self!, didProgress: progress)
        }
    }
}
```

问题分析：

| 问题 | 说明 |
|---|---|
| 线程安全 | `activeDownloads` 可能被多个线程同时读写 |
| Delegate 回调线程不确定 | 回调可能在任意线程执行，UI 更新需要手动切主线程 |
| 状态管理分散 | 下载状态散落在多个回调方法中 |
| 可选值链式调用 | `delegate?.method()` 可能静默失败 |

### 3.2 何时用 Actor 替代 Delegate

并非所有 Delegate 都需要迁移为 Actor。以下决策表帮助你判断：

| 场景 | 建议 | 原因 |
|---|---|---|
| Delegate 仅用于 UI 更新 | 保留 Delegate + @MainActor | UI 更新本身就在主线程 |
| Delegate 管理共享可变状态 | 迁移为 Actor | Actor 天然保护可变状态 |
| Delegate 涉及多步异步流程 | 迁移为 Actor + async/await | 流程编排更清晰 |
| 系统框架的 Delegate（如 UITableView） | 保留 | 系统框架不支持 Actor |
| 单一回调的简单场景 | 迁移为 async/await | 更简洁 |

### 3.3 Actor 隔离的基本概念

Actor 通过隔离机制保证线程安全——同一时刻只有一个任务能访问 Actor 的可变状态：

```swift
actor DownloadManager {
    private var activeDownloads: [String: DownloadTask] = [:]

    func startDownload(url: String) {
        let task = DownloadTask(url: url)
        activeDownloads[url] = task
    }

    func cancelDownload(url: String) {
        activeDownloads[url]?.cancel()
        activeDownloads.removeValue(forKey: url)
    }

    func getActiveDownloadCount() -> Int {
        activeDownloads.count
    }
}
```

与 class 的关键区别：

| 特性 | class | actor |
|---|---|---|
| 引用类型 | 是 | 是 |
| 线程安全 | 否（需手动加锁） | 是（自动隔离） |
| 方法调用 | 直接同步调用 | 需 await 异步调用 |
| 可变状态保护 | 无 | Actor 隔离自动保护 |
| 适用场景 | UI 控制器、简单数据模型 | 共享可变状态、并发访问 |

### 3.4 迁移步骤与代码示例

以一个位置管理器为例，展示从 Delegate 到 Actor 的完整迁移过程：

**第一步：识别共享状态和回调**

```swift
class LocationManager: NSObject, CLLocationManagerDelegate {
    private let manager = CLLocationManager()
    private var lastLocation: CLLocation?
    var onLocationUpdate: ((CLLocation) -> Void)?

    override init() {
        super.init()
        manager.delegate = self
        manager.startUpdatingLocation()
    }

    func locationManager(_ manager: CLLocationManager, didUpdateLocations locations: [CLLocation]) {
        lastLocation = locations.last
        onLocationUpdate?(locations.last!)
    }
}
```

**第二步：创建 Actor 封装**

```swift
actor LocationRepository {
    private var lastLocation: CLLocation?
    private let manager = CLLocationManager()
    private var continuation: AsyncStream<CLLocation>.Continuation?

    var currentLocation: CLLocation? {
        lastLocation
    }

    func start() -> AsyncStream<CLLocation> {
        AsyncStream { continuation in
            self.continuation = continuation
        }
    }

    func updateLocation(_ location: CLLocation) {
        lastLocation = location
        continuation?.yield(location)
    }
}
```

**第三步：创建桥接层**

```swift
class LocationBridge: NSObject, CLLocationManagerDelegate {
    private let repository: LocationRepository
    private let manager = CLLocationManager()

    init(repository: LocationRepository) {
        self.repository = repository
        super.init()
        manager.delegate = self
    }

    func startUpdating() {
        manager.startUpdatingLocation()
    }

    func locationManager(_ manager: CLLocationManager, didUpdateLocations locations: [CLLocation]) {
        if let location = locations.last {
            Task {
                await repository.updateLocation(location)
            }
        }
    }
}
```

**第四步：在业务层使用**

```swift
@MainActor
class MapViewModel: ObservableObject {
    @Published var currentLocation: CLLocation?

    private let repository = LocationRepository()
    private var bridge: LocationBridge?

    func startTracking() async {
        let bridge = LocationBridge(repository: repository)
        self.bridge = bridge
        bridge.startUpdating()

        let stream = await repository.start()
        for await location in stream {
            self.currentLocation = location
        }
    }
}
```

### 3.5 Delegate 与 Actor 共存的过渡期方案

在迁移过程中，Delegate 和 Actor 需要共存一段时间。推荐使用**适配器模式**：

```swift
protocol LegacyLocationDelegate: AnyObject {
    func didUpdateLocation(_ location: CLLocation)
}

actor LocationAdapter {
    weak var delegate: (any LegacyLocationDelegate)?

    func forwardLocation(_ location: CLLocation) {
        Task { @MainActor in
            delegate?.didUpdateLocation(location)
        }
    }
}
```

> 💡 **提示**：过渡期不要删除旧的 Delegate 接口，而是让新代码使用 Actor，旧代码继续使用 Delegate。待所有调用方迁移完毕后，再移除 Delegate 相关代码。

---

## 4. 全局变量与单例 → Actor 隔离

### 4.1 全局可变状态的并发风险

全局变量是并发安全的头号敌人。在 Swift 6 严格模式下，全局可变状态会直接报编译错误：

```swift
var appConfig: [String: String] = [:]

func updateConfig(_ key: String, value: String) {
    appConfig[key] = value
}

func readConfig(_ key: String) -> String? {
    appConfig[key]
}
```

这段代码在多线程环境下存在数据竞争——一个线程正在写入时，另一个线程可能正在读取。

### 4.2 单例模式的线程安全问题

传统的单例模式在 Swift 中虽然利用了 `static let` 的线程安全初始化，但单例内部的可变状态并不安全：

```swift
class UserManager {
    static let shared = UserManager()
    private var currentUser: User?
    private var loginHistory: [Date] = []

    private init() {}

    func login(_ user: User) {
        currentUser = user
        loginHistory.append(Date())
    }

    func logout() {
        currentUser = nil
    }

    func getCurrentUser() -> User? {
        currentUser
    }
}
```

`currentUser` 和 `loginHistory` 可能被多个线程同时访问，导致数据竞争。

### 4.3 @GlobalActor 与 @MainActor

Swift 提供了全局 Actor 来隔离全局状态：

**@MainActor** 是最常用的全局 Actor，所有标记了 @MainActor 的代码都在主线程执行：

```swift
@MainActor
class UserManager {
    static let shared = UserManager()
    private var currentUser: User?
    private var loginHistory: [Date] = []

    private init() {}

    func login(_ user: User) {
        currentUser = user
        loginHistory.append(Date())
    }

    func logout() {
        currentUser = nil
    }

    func getCurrentUser() -> User? {
        currentUser
    }
}
```

**自定义 @GlobalActor** 适用于需要在特定串行域执行的场景：

```swift
@globalActor
actor DataActor {
    static let shared = DataActor()
}

@DataActor
class DataStore {
    private var cache: [String: Data] = [:]

    func set(_ data: Data, forKey key: String) {
        cache[key] = data
    }

    func data(forKey key: String) -> Data? {
        cache[key]
    }
}
```

@MainActor 与自定义 @GlobalActor 的选择：

| 场景 | 选择 | 原因 |
|---|---|---|
| UI 相关的状态 | @MainActor | 必须在主线程访问 |
| 数据缓存 | 自定义 @GlobalActor | 不需要主线程，避免阻塞 UI |
| 网络请求管理 | 自定义 @GlobalActor | 网络回调不在主线程 |
| 配置信息 | 自定义 @GlobalActor | 全局访问但与 UI 无关 |

### 4.4 迁移全局状态到 Actor

将全局变量迁移到 Actor 的步骤：

**迁移前：**

```swift
var featureFlags: [String: Bool] = [:]
var apiEndpoint: String = "https://api.example.com"
var requestTimeout: Double = 30.0
```

**迁移后：**

```swift
actor AppConfiguration {
    static let shared = AppConfiguration()

    private var featureFlags: [String: Bool] = [:]
    private var apiEndpoint: String = "https://api.example.com"
    private var requestTimeout: Double = 30.0

    func setFeatureFlag(_ key: String, value: Bool) {
        featureFlags[key] = value
    }

    func isFeatureEnabled(_ key: String) -> Bool {
        featureFlags[key] ?? false
    }

    func getAPIEndpoint() -> String {
        apiEndpoint
    }

    func setAPIEndpoint(_ endpoint: String) {
        apiEndpoint = endpoint
    }

    func getTimeout() -> Double {
        requestTimeout
    }

    func setTimeout(_ timeout: Double) {
        requestTimeout = timeout
    }
}
```

> ⚠️ **警告**：迁移全局变量后，所有访问都需要 `await`，这会影响调用方代码。建议先添加 Actor 封装，再逐步迁移调用方，而不是一次性替换所有访问点。

### 4.5 迁移单例到 Actor

单例迁移有两种策略：

**策略一：将 class 改为 actor**

```swift
actor UserManager {
    static let shared = UserManager()

    private var currentUser: User?
    private var loginHistory: [Date] = []

    private init() {}

    func login(_ user: User) {
        currentUser = user
        loginHistory.append(Date())
    }

    func logout() {
        currentUser = nil
    }

    func getCurrentUser() -> User? {
        currentUser
    }

    func getLoginHistory() -> [Date] {
        loginHistory
    }
}
```

调用方变化：

```swift
let user = await UserManager.shared.getCurrentUser()
await UserManager.shared.login(newUser)
```

**策略二：保留 class，标记 @GlobalActor**

```swift
@MainActor
class UserManager {
    static let shared = UserManager()

    private var currentUser: User?
    private var loginHistory: [Date] = []

    private init() {}

    func login(_ user: User) {
        currentUser = user
        loginHistory.append(Date())
    }
}
```

两种策略对比：

| 维度 | actor 单例 | @GlobalActor class 单例 |
|---|---|---|
| 线程安全 | 自动保证 | 依赖全局 Actor 的调度 |
| 调用方式 | 需要 await | 需要 await（跨 Actor 边界时） |
| 继承支持 | actor 不支持继承 | class 支持继承 |
| 适用场景 | 纯数据管理 | 需要 ObserableObject 等 class 特性 |

---

## 5. 第三方库兼容处理

### 5.1 第三方库未支持 async/await 时的桥接方案

许多第三方库尚未提供 async/await 接口。这时需要自行桥接，但要注意桥接的正确性：

```swift
extension Alamofire.Session {
    func request(_ urlRequest: URLRequestConvertible) async throws -> Data {
        try await withCheckedThrowingContinuation { continuation in
            self.request(urlRequest).validate().responseData { response in
                switch response.result {
                case .success(let data):
                    continuation.resume(returning: data)
                case .failure(let error):
                    continuation.resume(throwing: error)
                }
            }
        }
    }
}
```

> ⚠️ **警告**：某些第三方库的回调可能不在主线程执行。如果回调中涉及 UI 更新，桥接时需要确保切换到 @MainActor。

### 5.2 使用 extension 添加 async 接口

为第三方库类型添加 async 接口的最佳实践：

```swift
extension KingfisherManager {
    @MainActor
    func loadImage(with resource: Resource) async throws -> UIImage {
        try await withCheckedThrowingContinuation { continuation in
            self.retrieveImage(with: resource) { result in
                switch result {
                case .success(let value):
                    continuation.resume(returning: value.image)
                case .failure(let error):
                    continuation.resume(throwing: error)
                }
            }
        }
    }
}
```

要点：

1. 将桥接方法放在独立的 extension 中
2. 标记正确的 Actor 隔离（如 @MainActor）
3. 方法命名与原库保持一致的风格
4. 在文件头部注释说明这是临时桥接，待库更新后移除

### 5.3 Sendable 兼容性处理

第三方库的类型可能不遵循 Sendable，在 Swift 6 严格模式下会报错。处理方式：

**方式一：@preconcurrency 导入**

```swift
@preconcurrency import SomeThirdPartyLibrary
```

`@preconcurrency` 告诉编译器："我知道这个库可能不满足 Sendable 要求，暂时不要对它的类型强制检查"。这是最常用的过渡方案。

**方式二：@unchecked Sendable 包装**

```swift
struct UncheckedSendableBox<T>: @unchecked Sendable {
    let value: T
}
```

> ⚠️ **警告**：`@unchecked Sendable` 绕过了编译器的安全检查，你需要自己保证类型的线程安全。只在确认类型实际安全（如不可变值）时使用。

**方式三：nonisolated(nonsending) 闭包**

```swift
func processWithThirdParty(_ block: nonisolated(nonsending) () -> Void) {
    block()
}
```

Swift 5.9 引入的 `nonisolated(nonsending)` 标记闭包不会跨越 Actor 边界传递数据，适用于不涉及共享状态的回调。

三种方式对比：

| 方式 | 安全性 | 适用场景 | 风险 |
|---|---|---|---|
| @preconcurrency 导入 | 中 | 第三方库整体不兼容 | 可能隐藏真实问题 |
| @unchecked Sendable | 低 | 确认类型实际安全 | 绕过编译器检查，需人工保证 |
| nonisolated(nonsending) | 高 | 闭包不跨域传递 | 仅适用于 Swift 5.9+ |

### 5.4 常见第三方库的迁移状态

| 库名 | async/await 支持 | Sendable 支持 | 桥接建议 |
|---|---|---|---|
| Alamofire | ✅ 5.6+ 原生支持 | ✅ 已适配 | 直接使用 async 接口 |
| Kingfisher | ✅ 7.0+ 原生支持 | 🟡 部分适配 | 使用 async 接口，注意 Sendable 警告 |
| SnapKit | N/A（同步 API） | ✅ | 无需桥接 |
| SwiftyJSON | ❌ 未支持 | ❌ 未适配 | @preconcurrency + extension 桥接 |
| Realm | ✅ 10.30+ 原生支持 | 🟡 部分适配 | 使用 async 接口，注意 Actor 隔离 |
| Firebase | ✅ 大部分 API 支持 | 🟡 部分适配 | 使用 async 接口，@preconcurrency 导入 |

> 💡 **提示**：第三方库的并发支持状态变化很快，建议定期查看库的 Release Notes 和 GitHub Issues，及时移除不再需要的桥接代码。

---

## 6. 迁移检查清单

### 6.1 逐模块迁移检查表

| 检查项 | 状态 | 说明 |
|---|---|---|
| Completion Handler 已替换为 async/await | ☐ | 保留旧接口作为过渡 |
| withCheckedContinuation 桥接正确 | ☐ | continuation.resume 只调用一次 |
| URLSession 使用原生 async API | ☐ | iOS 15+ 可直接使用 |
| 第三方库回调已桥接 | ☐ | extension 中添加 async 版本 |
| Delegate 已评估是否需要迁移为 Actor | ☐ | 仅管理共享状态的 Delegate 需要迁移 |
| Actor 隔离替代了手动加锁 | ☐ | 移除 NSLock、DispatchQueue 同步调用 |
| 全局可变状态已迁移到 Actor | ☐ | 或标记 @GlobalActor |
| 单例已迁移为 actor 或 @GlobalActor class | ☐ | 根据是否需要 class 特性选择 |
| @preconcurrency 导入已标注 | ☐ | 为不兼容的第三方库添加 |
| Sendable 一致性已检查 | ☐ | 跨 Actor 传递的类型遵循 Sendable |
| @MainActor 标记已添加 | ☐ | UI 相关代码标记 @MainActor |
| 测试已通过 | ☐ | 迁移后功能行为不变 |

### 6.2 编译警告处理策略

迁移过程中会遇到大量并发相关警告，建议按以下策略处理：

| 警告类型 | 严重程度 | 处理方式 |
|---|---|---|
| Sendable 一致性缺失 | 🔴 高 | 为类型添加 Sendable 或使用 @preconcurrency |
| Actor 隔离违规 | 🔴 高 | 添加 await 或标记正确的 Actor |
| @MainActor 缺失 | 🟡 中 | 为 UI 相关代码添加 @MainActor |
| 全局变量并发不安全 | 🔴 高 | 迁移到 Actor 或标记 @GlobalActor |
| 闭包捕获非 Sendable | 🟡 中 | 检查闭包是否跨域传递，必要时添加 @Sendable |
| nonisolated 访问 | 🟡 中 | 确认访问安全性，添加 nonisolated 或 await |

> 💡 **提示**：建议在迁移初期将并发检查设为 `minimal`，完成基础迁移后逐步提升到 `targeted`，最终达到 `complete`。在 Xcode 中：Build Settings → Swift Compiler - Concurrency → Strict Concurrency Checking。

### 6.3 测试验证要点

迁移后必须验证以下方面：

1. **功能正确性**：迁移后的行为与迁移前完全一致
2. **错误传播**：错误能正确传播到调用方
3. **取消行为**：Task 取消后资源正确释放
4. **主线程安全**：UI 更新仍在主线程执行
5. **性能对比**：关键路径的响应时间无明显回退

```swift
func testFetchUserMigration() async throws {
    let user = try await apiClient.fetchUser(id: "123")
    XCTAssertEqual(user.id, "123")
}

func testFetchUserErrorPropagation() async {
    do {
        _ = try await apiClient.fetchUser(id: "invalid")
        XCTFail("Should throw error")
    } catch {
        XCTAssertTrue(error is APIError)
    }
}
```

### 6.4 常见迁移错误与修复

| 错误信息 | 原因 | 修复方式 |
|---|---|---|
| `Expression is 'async' but is not marked with 'await'` | 调用 Actor 方法未加 await | 添加 await |
| `Non-sendable type crossed actor boundary` | 跨 Actor 传递了非 Sendable 类型 | 让类型遵循 Sendable 或使用 @preconcurrency |
| `Main actor-isolated property can not be mutated from a nonisolated context` | 在非主线程上下文修改 @MainActor 属性 | 将方法标记为 @MainActor 或使用 await |
| `Actor-isolated property 'x' can not be referenced from a nonisolated context` | 在 Actor 外部直接访问属性 | 通过 Actor 的 public 方法间接访问 |
| `Capture of 'self' with non-sendable type in a @Sendable closure` | @Sendable 闭包捕获了非 Sendable 的 self | 将 self 标记为 Sendable 或使用 @preconcurrency |
| `Global variable 'x' is not safe` | 全局变量缺少隔离 | 迁移到 Actor 或标记 @GlobalActor |

---

## 小结

本章系统讲解了从传统回调模式到 Swift 并发的渐进式迁移方法。以下是核心知识点总结：

| 知识领域 | 核心要点 | 关键 API / 模式 |
|---|---|---|
| 迁移策略 | 渐进式、自底向上、逐模块推进 | 迁移优先级：网络层 → 数据层 → 业务层 → UI 层 |
| Completion Handler 迁移 | 用 Continuation 桥接，逐步替换 | withCheckedContinuation、withCheckedThrowingContinuation |
| URLSession 迁移 | iOS 15+ 直接使用原生 async API | URLSession.shared.data(from:) |
| 第三方库回调桥接 | extension 添加 async 接口 | extension + withCheckedThrowingContinuation |
| Delegate → Actor | 仅管理共享状态的 Delegate 需迁移 | Actor 隔离 + AsyncStream + 桥接层 |
| Delegate 共存 | 适配器模式过渡 | 旧代码用 Delegate，新代码用 Actor |
| 全局变量迁移 | 封装到 Actor 或标记 @GlobalActor | actor、@globalActor |
| 单例迁移 | actor 单例或 @GlobalActor class | actor Shared {}、@MainActor class |
| 第三方库兼容 | @preconcurrency 导入、@unchecked Sendable | @preconcurrency import、nonisolated(nonsending) |
| Sendable 兼容 | 确保跨域类型遵循 Sendable | Sendable 协议、@Sendable 闭包 |
| 编译警告 | 按严重程度分批处理 | minimal → targeted → complete |
| 测试验证 | 功能、错误传播、取消、主线程、性能 | async 测试方法、XCTAssert 系列 |

迁移不是一蹴而就的工程，而是一个持续改进的过程。关键原则是：**每一步都可编译、可测试、可回退**。利用 Swift 编译器的并发警告作为迁移的路标，逐步将项目推向并发安全的目标。

← [Swift 6 与严格并发迁移](./Swift-6与严格并发迁移.md) | [ARC 与内存管理](./ARC与内存管理.md) →