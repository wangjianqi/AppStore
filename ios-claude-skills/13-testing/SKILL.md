---
name: testing
description: 涉及单元测试、UI 测试、XCTest、Mock、Stub、Snapshot Testing、异步测试、测试覆盖率、TDD 的任务
---

# 测试

## 测试分层

| 层级 | 类型 | 占比 | 速度 | 目标 |
|------|------|------|------|------|
| ViewModel / Service | 单元测试 | 70% | 快（< 0.1s） | 逻辑正确性 |
| Repository / Storage | 集成测试 | 20% | 中（< 1s） | 数据层协作 |
| ViewController | UI 测试 | 10% | 慢（> 2s） | 关键用户流程 |

**原则：**
- ViewModel 和 Service 是测试重点，覆盖率目标 > 80%
- VC 不写单元测试（太脆弱），用 UI 测试覆盖关键路径
- Repository 层用内存数据库测试，不依赖真实 CoreData

---

## 项目配置

### 目录约定
```
AppTests/
├── ViewModels/                 # ViewModel 单元测试
│   └── CameraViewModelTests.swift
├── Services/                   # Service 单元测试
│   └── AuthServiceTests.swift
├── Repositories/               # Repository 集成测试
│   └── UserRepositoryTests.swift
├── UI/                         # UI 测试
│   └── LoginFlowTests.swift
├── Helpers/                    # 测试辅助
│   ├── MockNetworkService.swift
│   ├── StubData.swift
│   └── CoreDataTestStack.swift
└── Extensions/                 # 测试扩展
    └── XCTestCase+Async.swift
```

### 测试 Target 配置
- Unit Test Target：`AppTests`，Host Application 设为主 App
- UI Test Target：`AppUITests`，独立进程运行
- `@testable import App` 允许访问 internal 成员

---

## 单元测试

### ViewModel 测试模板

```swift
import XCTest
@testable import App

final class CameraViewModelTests: XCTestCase {
    private var sut: CameraViewModel!
    private var mockCameraService: MockCameraService!

    override func setUp() {
        super.setUp()
        mockCameraService = MockCameraService()
        sut = CameraViewModel(cameraService: mockCameraService)
    }

    override func tearDown() {
        sut = nil
        mockCameraService = nil
        super.tearDown()
    }

    func testStartCapture_whenAuthorized_callsServiceStart() async {
        mockCameraService.authorizationStatus = .authorized
        await sut.startCapture()
        XCTAssertTrue(mockCameraService.startCaptureCalled)
    }

    func testStartCapture_whenDenied_showsAlert() async {
        mockCameraService.authorizationStatus = .denied
        await sut.startCapture()
        XCTAssertTrue(sut.showPermissionAlert)
        XCTAssertFalse(mockCameraService.startCaptureCalled)
    }
}
```

### 规范
- 每个测试方法只验证一个行为
- 命名：`test{方法}_{条件}_{期望结果}`
- `setUp` 创建 SUT（System Under Test），`tearDown` 释放
- **禁止测试方法之间有依赖**，每个测试独立运行
- 测试方法必须 `async` 或使用 `XCTestExpectation`

---

## Mock 与 Stub

### Mock 模板（Protocol-Based）

```swift
protocol CameraServiceProtocol {
    var authorizationStatus: AVAuthorizationStatus { get }
    func requestAccess() async -> Bool
    func startCapture() async throws
    func stopCapture()
}

final class MockCameraService: CameraServiceProtocol {
    var authorizationStatus: AVAuthorizationStatus = .notDetermined
    var requestAccessResult = true
    var startCaptureCalled = false
    var stopCaptureCalled = false
    var startCaptureError: Error?

    func requestAccess() async -> Bool {
        return requestAccessResult
    }

    func startCapture() async throws {
        startCaptureCalled = true
        if let error = startCaptureError {
            throw error
        }
    }

    func stopCapture() {
        stopCaptureCalled = true
    }
}
```

### Stub 数据

```swift
enum StubData {
    static func makeUser(
        id: String = "stub-user-id",
        name: String = "测试用户",
        email: String = "test@example.com"
    ) -> User {
        User(id: id, name: name, email: email)
    }

    static func makeAPIResponse<T: Decodable>(from file: String) -> T {
        let url = Bundle(for: BundleFinder.self).url(forResource: file, withExtension: "json")!
        let data = try! Data(contentsOf: url)
        return try! JSONDecoder().decode(T.self, from: data)
    }
}

private class BundleFinder {}
```

### 规范
- Mock 放在 `Tests/Helpers/` 目录
- Mock 只模拟当前测试需要的行为，**禁止创建万能 Mock**
- Stub JSON 文件放 `Tests/Resources/` 目录
- **禁止在 Mock 中实现业务逻辑**，只记录调用和返回预设值

---

## 异步测试

### async/await 测试

```swift
func testFetchUser_success_returnsUser() async throws {
    mockNetworkService.mockResponse = StubData.makeUser()
    let user = try await sut.fetchUser(id: "123")
    XCTAssertEqual(user.name, "测试用户")
}
```

### XCTestExpectation（回调场景）

```swift
func testDelegateCallback_firesOnSuccess() {
    let expectation = expectation(description: "Delegate callback")
    mockDelegate.onSuccess = { user in
        XCTAssertEqual(user.name, "测试用户")
        expectation.fulfill()
    }
    sut.fetchUser(id: "123")
    waitForExpectations(timeout: 2)
}
```

### 规范
- `timeout` 默认 2 秒，网络相关测试可设 5 秒
- **禁止使用 `sleep()` 等待**，必须用 `XCTestExpectation` 或 `async/await`
- `waitForExpectations` 超时即测试失败

---

## CoreData 测试

### 内存数据库

```swift
final class CoreDataTestStack {
    lazy var persistentContainer: NSPersistentContainer = {
        let container = NSPersistentContainer(name: "CoreDataModels")
        let description = NSPersistentStoreDescription()
        description.type = NSInMemoryStoreType
        container.persistentStoreDescriptions = [description]
        container.loadPersistentStores { _, error in
            if let error { fatalError("测试 CoreData 加载失败: \(error)") }
        }
        return container
    }()

    var viewContext: NSManagedObjectContext { persistentContainer.viewContext }
}
```

### Repository 集成测试

```swift
final class UserRepositoryTests: XCTestCase {
    private var sut: UserRepository!
    private var testStack: CoreDataTestStack!

    override func setUp() {
        super.setUp()
        testStack = CoreDataTestStack()
        sut = UserRepository(coreDataStack: testStack)
    }

    func testSaveUser_persistsToDatabase() async throws {
        let user = try await sut.saveUser(name: "测试", email: "test@example.com")
        let fetched = try await sut.fetchUser(id: user.id)
        XCTAssertEqual(fetched?.name, "测试")
    }
}
```

---

## UI 测试

### 关键流程测试

```swift
final class LoginFlowTests: XCTestCase {
    let app = XCUIApplication()

    override func setUp() {
        continueAfterFailure = false
        app.launchArguments = ["--uitesting", "--reset-state"]
        app.launch()
    }

    func testLogin_withValidCredentials_navigatesToHome() {
        let emailField = app.textFields["email"]
        emailField.tap()
        emailField.typeText("test@example.com")

        let passwordField = app.secureTextFields["password"]
        passwordField.tap()
        passwordField.typeText("password123")

        app.buttons["login"].tap()

        let homeTitle = app.staticTexts["home_title"]
        XCTAssertTrue(homeTitle.waitForExistence(timeout: 5))
    }
}
```

### Launch Arguments 控制测试环境

```swift
// App 端
if ProcessInfo.processInfo.arguments.contains("--uitesting") {
    if ProcessInfo.processInfo.arguments.contains("--reset-state") {
        resetAppState()
    }
    if ProcessInfo.processInfo.arguments.contains("--mock-network") {
        NetworkService.shared = MockNetworkService()
    }
}
```

### 规范
- UI 测试只覆盖关键用户流程（登录、购买、核心操作）
- 使用 Accessibility Identifier 定位元素，**禁止用文本或索引定位**
- `continueAfterFailure = false`，一步失败后续不再执行
- UI 测试必须能在 CI 环境运行（无模拟器弹窗干扰）

---

## Snapshot Testing（可选）

### 配置
- SPM 依赖：`github.com/pointfreeco/swift-snapshot-testing`

### 用法

```swift
import SnapshotTesting

final class PaywallViewTests: XCTestCase {
    func testPaywallLayout() {
        let paywallVC = PaywallVC()
        paywallVC.loadViewIfNeeded()
        assertSnapshot(of: paywallVC, as: .image(on: .iPhone15))
    }

    func testPaywallDarkMode() {
        let paywallVC = PaywallVC()
        paywallVC.overrideUserInterfaceStyle = .dark
        paywallVC.loadViewIfNeeded()
        assertSnapshot(of: paywallVC, as: .image(on: .iPhone15))
    }
}
```

### 规范
- Snapshot 文件提交到 Git，CI 上对比差异
- 首次运行生成基准图，后续运行对比
- 不同设备尺寸分别测试：`.iPhone15`, `.iPhone15ProMax`, `.iPad`
- **禁止在 CI 上重新生成基准图**，必须人工审核后更新

---

## 测试覆盖率

### 配置
- Xcode → Scheme → Test → Options → Code Coverage ✅
- 目标：ViewModel > 80%，Service > 70%，整体 > 60%

### 排除项（不计入覆盖率）
- AppDelegate / SceneDelegate（生命周期代码）
- Extensions（纯语法糖）
- Models（纯数据结构）
- DesignSystem（UI 组件）

---

## CI 集成

### xcodebuild 命令

```bash
xcodebuild test \
    -workspace App.xcworkspace \
    -scheme App \
    -destination 'platform=iOS Simulator,name=iPhone 15,OS=latest' \
    -only-testing:AppTests \
    -resultBundlePath TestResults.xcresult \
    CODE_SIGNING_ALLOWED=NO
```

### 规范
- PR 合并前必须跑单元测试
- UI 测试可夜间定时跑，不阻塞合并
- 测试失败时输出具体断言信息，禁止只报 "test failed"

---

## 已知陷阱

- **`setUp` 在每个 test 方法前调用**，不是每个测试类调用一次
- **CoreData 内存数据库不支持 `NSPersistentHistoryTrack`**，测试时需跳过
- **UI 测试的 `typeText` 在中文输入法下可能失败**，测试前切换英文输入法
- **异步测试中 `async let` 并发可能导致测试顺序问题**，每个测试用 `await` 串行执行
- **Snapshot Testing 在不同 Xcode 版本间可能产生差异**，CI 和开发环境必须用同一版本
- **Mock 的 `called` 标志必须在 `setUp` 中重置**，否则测试间会污染
