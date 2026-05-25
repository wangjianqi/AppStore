# 快照测试与 UI 自动化测试深入

> 🎯 **本章目标**：掌握 swift-snapshot-testing 快照测试的使用方法，学会 XCUITest 的工程化实践（Page Object 模式、并行测试、Flaky test 处理），建立可靠的 UI 测试工作流。

---

## 为什么需要快照测试与 UI 自动化测试

### 单元测试的局限

单元测试擅长验证逻辑正确性，但对于 UI 层面的问题几乎无能为力。一个按钮的文字、颜色、间距、圆角是否正确，单元测试无法给出答案。即使你为 ViewModel 写了 100% 覆盖率的测试，用户看到的界面仍然可能因为一个约束错误而完全崩坏。

常见的 UI 问题包括：

- 布局在不同屏幕尺寸下错位或溢出
- 深色模式下文字与背景色对比度不足
- 动态字体导致文字被截断
- 本地化后文本溢出容器

这些问题只有通过视觉层面的验证才能发现，而快照测试正是为此而生。

### 快照测试的价值

快照测试的核心思路是：将视图渲染为图片，与之前保存的参考图片进行像素级对比。如果两者一致，测试通过；如果出现差异，测试失败并生成差异对比图。

这种方式的优势在于：

- **一次编写，全面覆盖**：一个快照测试可以同时验证布局、颜色、字体、间距等多个视觉属性
- **回归检测利器**：任何无意的视觉变更都会被立即捕获
- **文档化作用**：快照参考图本身就是 UI 的可视化文档
- **低成本高收益**：编写一个快照测试的成本远低于为每个视觉属性编写断言

### UI 自动化测试的价值

快照测试验证的是"看起来对不对"，而 UI 自动化测试验证的是"用起来对不对"。XCUITest 模拟用户的真实操作：点击、滑动、输入、导航，从启动 App 到完成核心流程，端到端地验证整个用户旅程。

UI 自动化测试的关键价值：

- 验证用户完整流程是否畅通
- 捕获页面间跳转、数据传递的集成问题
- 确保关键业务路径（注册、支付、提交）始终可用
- 在重构或架构调整后提供安全网

### 测试金字塔中的位置

| 测试类型 | 覆盖范围 | 执行速度 | 维护成本 | 示例 |
|---------|---------|---------|---------|------|
| 单元测试 | 单个函数/类 | 毫秒级 | 低 | ViewModel 逻辑验证 |
| 快照测试 | 单个视图/组件 | 秒级 | 中 | 登录按钮样式验证 |
| UI 自动化测试 | 完整用户流程 | 分钟级 | 高 | 注册→登录→购买流程 |
| 手动测试 | 全局体验 | 小时级 | 最高 | 探索性测试 |

按照测试金字塔原则，单元测试数量最多、UI 测试数量最少。但快照测试和 UI 自动化测试覆盖的是金字塔上层的关键场景，它们的价值不在于数量，而在于覆盖最核心的路径。

---

## swift-snapshot-testing 快照测试

### 库的安装与配置

swift-snapshot-testing 是 Point-Free 团队开源的快照测试框架，支持 SwiftUI、UIKit 视图、CALayer 等多种快照类型。

通过 Swift Package Manager 安装：

1. 在 Xcode 中选择 File → Add Package Dependencies
2. 输入仓库地址：`https://github.com/pointfreeco/swift-snapshot-testing`
3. 选择版本规则，推荐使用 1.17.0 及以上版本
4. 将 `SnapshotTesting` 添加到测试 Target 的依赖中

在 Package.swift 中声明：

```swift
dependencies: [
    .package(
        url: "https://github.com/pointfreeco/swift-snapshot-testing",
        from: "1.17.0"
    )
],
targets: [
    .testTarget(
        name: "MyAppTests",
        dependencies: [
            .product(name: "SnapshotTesting", package: "swift-snapshot-testing")
        ]
    )
]
```

### 基本快照测试编写

在测试文件中导入框架并编写快照测试：

```swift
import XCTest
import SnapshotTesting
import SwiftUI
@testable import MyApp

class LoginViewSnapshotTests: XCTestCase {
    override func setUp() {
        SnapshotTesting.isRecording = false
    }

    func testLoginView() {
        let view = LoginView()
        let controller = UIHostingController(rootView: view)
        assertSnapshot(of: controller, as: .image)
    }
}
```

首次运行时，由于没有参考图，需要将 `isRecording` 设为 `true` 来生成参考快照。参考图会保存在测试文件同目录下的 `__Snapshots__/LoginViewSnapshotTests/` 文件夹中，文件名与测试方法名一致。

> 💡 **提示**：参考图应纳入 Git 版本管理，这样团队成员和 CI 环境都能使用同一套参考图。

### SwiftUI 视图快照测试

SwiftUI 视图可以直接使用 `.image` 策略进行快照：

```swift
func testProfileCard() {
    let card = ProfileCard(
        user: User(
            name: "张三",
            avatar: "avatar_placeholder",
            bio: "iOS 开发者"
        )
    )
    assertSnapshot(of: card, as: .image(layout: .fixed(width: 375, height: 200)))
}
```

`layout` 参数控制视图的布局方式：

- `.device(config)`：模拟特定设备的尺寸
- `.fixed(width:height:)`：固定尺寸
- `.sizeThatFits`：根据内容自适应大小

对于需要环境值的视图，可以注入预览环境：

```swift
func testSettingsView() {
    let settings = SettingsView()
        .environment(\.locale, Locale(identifier: "zh_CN"))
        .environmentObject(AppSettings())

    assertSnapshot(of: settings, as: .image(layout: .device(config: .iPhone13)))
}
```

### 多设备/多外观快照

为了确保 UI 在不同设备和外观下都正确，可以批量生成快照：

```swift
func testHomeScreenMultipleDevices() {
    let devices: [(String, ViewImageConfig)] = [
        ("iPhone13", .iPhone13),
        ("iPhone13ProMax", .iPhone13ProMax),
        ("iPadPro11", .iPadPro11),
    ]

    for (name, config) in devices {
        assertSnapshot(
            of: HomeScreenView(),
            as: .image(layout: .device(config: config)),
            named: name
        )
    }
}
```

深色模式与浅色模式对比：

```swift
func testCardViewAppearances() {
    let appearances: [(String, UIUserInterfaceStyle)] = [
        ("light", .light),
        ("dark", .dark),
    ]

    for (name, style) in appearances {
        let controller = UIHostingController(rootView: CardView())
        controller.overrideUserInterfaceStyle = style
        assertSnapshot(
            of: controller,
            as: .image,
            named: name
        )
    }
}
```

> ⚠️ **警告**：多设备多外观快照会显著增加参考图数量和存储空间。建议只为关键页面生成多设备快照，普通组件使用固定尺寸即可。

### 快照差异对比与更新

当快照测试失败时，框架会在快照目录中生成三张图片：

- `testName.png`：当前渲染结果
- `testName@2x.png`：差异对比图（红色标记差异区域）
- `testName@3x.png`：参考图

差异对比图是判断变更是否合理的关键工具。如果差异是预期的（如设计稿更新），需要重新录制参考图：

```swift
override func setUp() {
    SnapshotTesting.isRecording = true
}
```

或将环境变量 `SNAPSHOT_TESTING_RECORD` 设为 `true` 后运行测试：

```bash
SNAPSHOT_TESTING_RECORD=true xcodebuild test \
    -scheme MyApp \
    -destination 'platform=iOS Simulator,name=iPhone 15'
```

### 自定义快照策略

除了 `.image`，swift-snapshot-testing 还支持多种快照策略：

```swift
func testViewModelState() {
    let viewModel = LoginViewModel()
    viewModel.email = "test@example.com"
    viewModel.password = "123456"
    assertSnapshot(of: viewModel, as: .json)
}

func testAttributedString() {
    let attributed = NSAttributedString(
        string: "Hello World",
        attributes: [
            .font: UIFont.boldSystemFont(ofSize: 16),
            .foregroundColor: UIColor.red
        ]
    )
    assertSnapshot(of: attributed, as: .image)
}
```

可以自定义快照策略来满足特殊需求：

```swift
extension Snapshotting where Value == UIView, Format == UIImage {
    static var precisionImage: Snapshotting {
        return .image(precision: 0.95, scale: nil)
    }
}
```

`precision` 参数允许一定程度的像素差异，取值 0 到 1 之间。0.95 表示允许 5% 的像素差异，适用于抗锯齿等微小渲染差异的场景。

### CI 中的快照测试

快照测试在 CI 中运行时需要注意以下几点：

1. **模拟器一致性**：CI 必须使用与开发环境相同型号的模拟器，不同模拟器的渲染结果可能存在细微差异
2. **参考图同步**：确保 CI 拉取到最新的参考图
3. **失败产物收集**：CI 失败时需要上传差异对比图作为 Artifact

在 Xcode Cloud 中的配置：

```yaml
# .xcode-cloud.yml
workflows:
  snapshot-tests:
    name: Snapshot Tests
    environment:
      xcode: "15.4"
      simulators:
        - "iPhone 15"
    actions:
      - test:
          scheme: MyApp
          testPlans:
            - SnapshotTests
    artifacts:
      - "__Snapshots__/**"
```

> 💡 **提示**：在 CI 中使用 `SNAPSHOT_TESTING_RECORD=false` 确保不会意外覆盖参考图。只在需要更新参考图时才开启录制模式。

---

## XCUITest 工程化实践

### Page Object 模式详解

Page Object 模式是 UI 自动化测试中最重要的设计模式。它将页面元素和操作封装为独立的对象，测试代码只与页面对象交互，不直接操作 UI 元素。

没有 Page Object 的测试代码：

```swift
func testLogin() {
    let app = XCUIApplication()
    app.textFields["email"].tap()
    app.textFields["email"].typeText("test@example.com")
    app.secureTextFields["password"].tap()
    app.secureTextFields["password"].typeText("123456")
    app.buttons["loginButton"].tap()
    XCTAssertTrue(app.staticTexts["welcome"].exists)
}
```

这种写法的问题：元素定位逻辑散落在每个测试中，一旦 UI 变更，所有测试都需要修改。

使用 Page Object 后：

```swift
class LoginPage {
    let app: XCUIApplication

    init(app: XCUIApplication) {
        self.app = app
    }

    private var emailField: XCUIElement {
        app.textFields["email"]
    }

    private var passwordField: XCUIElement {
        app.secureTextFields["password"]
    }

    private var loginButton: XCUIElement {
        app.buttons["loginButton"]
    }

    @discardableResult
    func enterEmail(_ email: String) -> LoginPage {
        emailField.tap()
        emailField.typeText(email)
        return self
    }

    @discardableResult
    func enterPassword(_ password: String) -> LoginPage {
        passwordField.tap()
        passwordField.typeText(password)
        return self
    }

    func tapLogin() -> HomePage {
        loginButton.tap()
        return HomePage(app: app)
    }

    func login(email: String, password: String) -> HomePage {
        return enterEmail(email)
            .enterPassword(password)
            .tapLogin()
    }
}
```

测试代码变得简洁且可读：

```swift
func testLogin() {
    let homePage = LoginPage(app: XCUIApplication())
        .login(email: "test@example.com", password: "123456")
    XCTAssertTrue(homePage.isDisplayed)
}
```

### 封装页面对象

为每个页面创建独立的 Page Object，并建立基类减少重复代码：

```swift
class BasePage {
    let app: XCUIApplication

    init(app: XCUIApplication) {
        self.app = app
    }

    func waitForElement(_ element: XCUIElement, timeout: TimeInterval = 5) -> Bool {
        let predicate = NSPredicate(format: "exists == true")
        let expectation = XCTNSPredicateExpectation(predicate: predicate, object: element)
        let result = XCTWaiter().wait(for: [expectation], timeout: timeout)
        return result == .completed
    }

    var isDisplayed: Bool {
        fatalError("Subclass must override")
    }
}
```

具体页面对象继承基类：

```swift
class HomePage: BasePage {
    private var greetingLabel: XCUIElement {
        app.staticTexts["welcome"]
    }

    private var profileButton: XCUIElement {
        app.buttons["profile"]
    }

    override var isDisplayed: Bool {
        greetingLabel.exists
    }

    func tapProfile() -> ProfilePage {
        profileButton.tap()
        return ProfilePage(app: app)
    }
}
```

> 💡 **提示**：页面对象的方法应返回下一个页面的实例，形成链式调用，这样可以在编译时就发现页面跳转的错误。

### 测试数据管理

硬编码测试数据会导致维护困难，应将测试数据集中管理：

```swift
enum TestData {
    enum User {
        static let validEmail = "test@example.com"
        static let validPassword = "Test123456!"
        static let invalidEmail = "invalid-email"
        static let shortPassword = "12"
    }

    enum Product {
        static let searchKeyword = "iPhone"
        static let outOfStockSKU = "SKU-00000"
    }
}
```

对于需要动态生成的数据，使用工厂方法：

```swift
struct UserFactory {
    static func makeUniqueUser() -> (email: String, password: String) {
        let timestamp = Int(Date().timeIntervalSince1970)
        return (
            email: "test_\(timestamp)@example.com",
            password: "Test\(timestamp)!"
        )
    }
}
```

### 等待策略优化

XCUITest 中最常见的问题是时序问题。元素尚未出现就尝试操作，导致测试失败。

基础等待方式：

```swift
func waitForElement(_ element: XCUIElement, timeout: TimeInterval = 10) -> Bool {
    let predicate = NSPredicate(format: "exists == true AND isHittable == true")
    let expectation = XCTNSPredicateExpectation(predicate: predicate, object: element)
    return XCTWaiter().wait(for: [expectation], timeout: timeout) == .completed
}
```

高级等待策略——等待特定状态：

```swift
func waitForLoadingComplete(timeout: TimeInterval = 15) {
    let activityIndicator = app.activityIndicators.element
    let predicate = NSPredicate(format: "exists == false")
    let expectation = XCTNSPredicateExpectation(predicate: predicate, object: activityIndicator)
    XCTWaiter().wait(for: [expectation], timeout: timeout)
}
```

> ⚠️ **警告**：避免使用 `sleep()` 硬编码等待。`sleep(3)` 在快速设备上浪费时间，在慢速设备上可能仍然不够。始终使用条件等待。

### 并行测试配置

当测试用例增多后，串行执行会消耗大量时间。XCUITest 支持在多个模拟器上并行运行测试。

通过 xcodebuild 启用并行测试：

```bash
xcodebuild test \
    -scheme MyApp \
    -destination 'platform=iOS Simulator,name=iPhone 15' \
    -parallel-testing-enabled YES \
    -parallel-testing-worker-count 4
```

在 Xcode Scheme 中配置：Edit Scheme → Test → Options → 勾选 Execute in Parallel。

并行测试的注意事项：

- 每个测试 Worker 使用独立的模拟器实例
- 测试之间不能有共享状态的依赖
- 测试数据必须相互独立，避免数据竞争
- 模拟器数量受机器性能限制，建议不超过 CPU 核心数

确保测试可并行化的关键原则：

```swift
class ParallelSafeTests: XCTestCase {
    let app = XCUIApplication()

    override func setUp() {
        continueAfterFailure = false
        app.launchArguments = ["--ui-testing", "--reset-state"]
        app.launch()
    }

    override func tearDown() {
        let resetButton = app.buttons["resetEnvironment"]
        if resetButton.exists {
            resetButton.tap()
        }
    }
}
```

通过 `--reset-state` 启动参数在 App 端重置状态，确保每个测试从干净的环境开始。

### 测试录制与手写的选择

Xcode 提供了测试录制功能：在测试方法中点击录制按钮，然后在模拟器上操作，Xcode 会自动生成对应的测试代码。

录制生成的代码示例：

```swift
let app = XCUIApplication()
app.buttons["登录"].tap()
app.textFields["请输入邮箱"].typeText("test@example.com")
app.secureTextFields["请输入密码"].typeText("123456")
app.buttons["确认登录"].tap()
```

录制与手写的对比：

| 维度 | 录制 | 手写 |
|------|------|------|
| 速度 | 快，几分钟即可生成 | 慢，需要设计封装 |
| 可维护性 | 差，元素定位硬编码 | 好，Page Object 封装 |
| 可读性 | 差，操作步骤堆砌 | 好，语义化方法调用 |
| 复用性 | 无 | 高，方法可复用 |
| 适用场景 | 快速原型、探索性验证 | 长期维护的测试套件 |

> 💡 **提示**：推荐的工作流是先用录制快速定位元素和操作路径，然后将录制结果重构为 Page Object 模式的手写代码。

---

## Flaky Test 处理

### 什么是 Flaky Test

Flaky Test 是指在代码没有变更的情况下，有时通过有时失败的测试。它是测试工程中最令人头疼的问题，会严重削弱团队对测试结果的信任。

Flaky Test 的危害：

- 开发者开始忽略测试失败，"再跑一次就好了"成为习惯
- CI 变得不可信，红色构建不再引起重视
- 测试套件的整体价值下降
- 浪费大量排查时间

### 常见 Flaky 原因分析

| 原因 | 表现 | 出现频率 |
|------|------|---------|
| 异步等待不足 | 元素未出现就操作 | 非常高 |
| 网络请求不确定 | 接口响应时间波动 | 高 |
| 测试数据冲突 | 并行测试间数据互相影响 | 中 |
| 动画干扰 | 动画未完成就点击 | 中 |
| 时间依赖 | 时区、日期相关逻辑 | 低 |
| 模拟器性能波动 | 超时但元素实际存在 | 中 |
| 推送/弹窗干扰 | 系统弹窗遮挡操作 | 低 |

### 防御性编写策略

编写 UI 测试时应始终采用防御性思维，预判可能出现的不稳定因素：

**策略一：显式等待而非隐式假设**

```swift
func testAddToCart() {
    let productCell = app.cells["product_001"]
    waitForElement(productCell)
    productCell.tap()

    let addToCartButton = app.buttons["addToCart"]
    waitForElement(addToCartButton)
    addToCartButton.tap()

    let cartBadge = app.staticTexts["cartBadge"]
    XCTAssertTrue(waitForElement(cartBadge), "Cart badge should appear after adding item")
}
```

**策略二：处理系统弹窗**

```swift
func handleSystemAlerts() {
    let alerts = app.alerts
    if alerts.count > 0 {
        alerts.buttons.firstMatch.tap()
    }
}

func allowNotificationsIfAsked() {
    let springboard = XCUIApplication(bundleIdentifier: "com.apple.springboard")
    let allowButton = springboard.buttons["Allow"]
    if allowButton.waitForExistence(timeout: 5) {
        allowButton.tap()
    }
}
```

**策略三：使用 Accessibility Identifier 而非文本定位**

```swift
app.buttons["loginButton"]
app.staticTexts["welcomeMessage"]
app.cells["productCell_001"]
```

而非：

```swift
app.buttons["登录"]
app.staticTexts["欢迎回来"]
app.cells["iPhone 15 Pro Max 256GB"]
```

文本会随本地化变化，而 Accessibility Identifier 始终稳定。

### 重试机制

对于难以完全消除的 Flaky，可以引入重试机制作为最后防线：

```swift
extension XCTestCase {
    func retryTest(maxAttempts: Int = 2, testBlock: () -> Void) {
        var lastError: Error?
        for attempt in 1...maxAttempts {
            do {
                try testBlock()
                return
            } catch {
                lastError = error
                print("Attempt \(attempt) failed: \(error)")
            }
        }
        if let error = lastError {
            XCTFail("Test failed after \(maxAttempts) attempts: \(error)")
        }
    }
}
```

Xcode 13+ 原生支持测试重试：在 Scheme 配置中勾选 "Retry test failures"，默认重试一次。

> ⚠️ **警告**：重试机制只是缓解手段，不是解决方案。每次重试成功都应记录下来，后续仍需排查根本原因。

### Flaky Test 隔离与标记

将已知 Flaky 的测试与稳定测试分离，避免影响整体 CI 结果：

```swift
class FlakyTestBase: XCTestCase {
    override func setUp() {
        continueAfterFailure = true
    }
}
```

在测试计划中创建两个测试套件：

- **Stable Suite**：所有稳定的测试，失败即阻断
- **Flaky Suite**：已知不稳定测试，失败仅警告

通过 Xcode Scheme 配置或测试计划实现：

1. Product → Scheme → Edit Scheme
2. Test → Test Plans → 创建多个 Test Plan
3. 将 Flaky 测试放入独立的 Test Plan
4. CI 中分别运行，Flaky 测试失败不阻断主流程

在 GitHub Actions 中可以这样处理：

```yaml
- name: Run Stable Tests
  run: |
    xcodebuild test \
      -scheme MyApp \
      -only-testing:MyAppUITests/StableSuite

- name: Run Flaky Tests
  continue-on-error: true
  run: |
    xcodebuild test \
      -scheme MyApp \
      -only-testing:MyAppUITests/FlakySuite
```

---

## CI 集成最佳实践

### Xcode Cloud 中的 UI 测试

Xcode Cloud 是 Apple 官方的 CI/CD 服务，原生支持 XCUITest：

```yaml
# .xcode-cloud.yml
version: 1
workflows:
  ui-tests:
    name: UI Tests
    environment:
      xcode: "16.0"
      simulators:
        - "iPhone 15"
        - "iPad (10th generation)"
    triggers:
      - branch: main
        event: push
    actions:
      - test:
          scheme: MyApp
          testPlans:
            - UITestPlan
    artifacts:
      - "*.xcresult"
```

Xcode Cloud 的 UI 测试注意事项：

- 免费额度为每月 25 小时，UI 测试消耗较快，需合理规划
- 模拟器型号由 Apple 管理，确保测试不依赖特定模拟器
- 测试失败时可在 Xcode Cloud 控制台查看截图和视频回放
- 支持并行测试，但会消耗更多额度

### GitHub Actions 配置

使用 GitHub Actions 运行 XCUITest：

```yaml
name: UI Tests

on:
  pull_request:
    branches: [main]

jobs:
  ui-tests:
    runs-on: macos-14
    steps:
      - uses: actions/checkout@v4

      - name: Select Xcode
        run: sudo xcode-select -s /Applications/Xcode_16.0.app

      - name: Run UI Tests
        run: |
          xcodebuild test \
            -project MyApp.xcodeproj \
            -scheme MyApp \
            -destination 'platform=iOS Simulator,name=iPhone 15,OS=18.0' \
            -resultBundlePath TestResults \
            -parallel-testing-enabled YES

      - name: Upload Test Results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: TestResults

      - name: Upload Snapshot Diffs
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: snapshot-diffs
          path: "**/__Snapshots__/**"
```

### 测试报告与通知

将 .xcresult 转换为可读报告：

```bash
xcrun xcresulttool get test-results summary \
    --path TestResults.xcresult
```

使用第三方工具生成 HTML 报告：

```bash
brew install chargepoint/xcparse/xcparse
xcparse logs --output ./logs TestResults.xcresult
xcparse screenshots --output ./screenshots TestResults.xcresult
```

在 GitHub Actions 中添加 Slack 通知：

```yaml
- name: Notify Slack
  if: failure()
  uses: slackapi/slack-github-action@v1
  with:
    payload: |
      {
        "text": "UI Tests failed on ${{ github.ref }}",
        "blocks": [
          {
            "type": "section",
            "text": {
              "type": "mrkdwn",
              "text": "❌ UI Tests failed\nCommit: ${{ github.sha }}\n<${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}|View Details>"
            }
          }
        ]
      }
```

### 测试覆盖率统计

在 xcodebuild 中启用覆盖率收集：

```bash
xcodebuild test \
    -scheme MyApp \
    -destination 'platform=iOS Simulator,name=iPhone 15' \
    -enableCodeCoverage YES \
    -resultBundlePath TestResults.xcresult
```

导出覆盖率报告：

```bash
xcrun xccov view --report TestResults.xcresult > coverage.txt
```

> 💡 **提示**：UI 测试的代码覆盖率通常比单元测试更"真实"，因为它覆盖的是实际用户路径。但不要追求 UI 测试的高覆盖率，重点覆盖核心业务路径即可。

### 快照测试的 CI 特殊处理

快照测试在 CI 中有特殊的挑战：

**挑战一：渲染差异**

不同 macOS 版本、不同 Xcode 版本的渲染引擎可能产生细微差异。解决方案：

- CI 使用与开发团队一致的 Xcode 版本
- 使用 `precision` 参数容忍微小差异
- 将参考图按 Xcode 版本分目录存储

```swift
func testCardView() {
    let xcodeVersion = ProcessInfo.processInfo.environment["XCODE_VERSION"] ?? "unknown"
    assertSnapshot(
        of: CardView(),
        as: .image(precision: 0.98),
        named: "xcode_\(xcodeVersion)"
    )
}
```

**挑战二：参考图管理**

参考图文件较大，频繁更新会增加仓库体积。解决方案：

- 使用 Git LFS 管理快照参考图
- 定期清理过时的参考图
- 在 PR 中审查参考图变更

```bash
git lfs track "__Snapshots__/**/*.png"
git lfs track "__Snapshots__/**/*.tiff"
```

**挑战三：CI 失败时的调试信息**

```yaml
- name: Upload Snapshot Failures
  if: failure()
  uses: actions/upload-artifact@v4
  with:
    name: snapshot-failures
    path: |
      **/__Snapshots__/**/*.png
      **/__Snapshots__/**/*.tiff
```

确保 CI 失败时能下载差异对比图，方便开发者判断是真实回归还是渲染差异。

---

## 小结

| 知识点 | 核心内容 | 关键实践 |
|--------|---------|---------|
| 快照测试价值 | 像素级 UI 回归检测 | 优先覆盖核心页面 |
| swift-snapshot-testing | 安装配置、多设备多外观快照 | 参考图纳入 Git 管理 |
| 快照策略 | image/json/自定义策略 | precision 参数处理渲染差异 |
| CI 快照测试 | 模拟器一致性、差异图收集 | Git LFS 管理参考图 |
| Page Object 模式 | 页面元素与操作封装 | 方法返回下一页面实例 |
| 测试数据管理 | 集中管理、工厂方法 | 避免硬编码测试数据 |
| 等待策略 | 条件等待替代 sleep | 等待 isHittable 而非 exists |
| 并行测试 | 多模拟器并行执行 | 测试间无共享状态依赖 |
| 测试录制 | 快速定位元素路径 | 录制后重构为 Page Object |
| Flaky Test | 时序、网络、数据冲突 | 防御性编写 + 条件等待 |
| 重试机制 | 自动重试失败测试 | 重试仅作缓解，需排查根因 |
| Flaky 隔离 | 稳定/不稳定测试分离 | Flaky 测试失败不阻断 CI |
| Xcode Cloud | 原生 CI/CD 支持 | 注意额度消耗 |
| GitHub Actions | 灵活配置、丰富生态 | 失败时上传差异图和日志 |
| 测试报告 | xcresult 解析与通知 | 集成 Slack/钉钉通知 |

快照测试和 UI 自动化测试是保障 App 质量的重要手段。快照测试以低成本实现视觉回归检测，UI 自动化测试验证端到端用户流程。两者结合，配合 Page Object 模式、防御性编写策略和完善的 CI 集成，能够构建出可靠且高效的 UI 测试工作流。记住：测试的价值不在于数量，而在于稳定和可信赖。

← [测试基础](./测试基础.md) | [无障碍与国际化](./无障碍与国际化.md) →