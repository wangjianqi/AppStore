# iOS 版本适配指南

> 🎯 **本章目标**：掌握 iOS 版本适配的系统方法，学会设置最低部署目标与条件编译，理解废弃 API 的替换策略，建立每年适配新 iOS 版本的工作流。

---

## 为什么需要版本适配

### iOS 生态的碎片化现状

与 Android 生态相比，iOS 的版本碎片化程度相对较低，但每年 WWDC 发布新版本后，开发者仍需面对多版本并存的现实。以下是近年 iOS 版本的市场份额分布：

| iOS 版本 | 市场份额（2025 年初） | 发布年份 |
|----------|----------------------|----------|
| iOS 18   | ~52%                 | 2024     |
| iOS 17   | ~30%                 | 2023     |
| iOS 16   | ~12%                 | 2022     |
| iOS 15   | ~4%                  | 2021     |
| 更早版本  | ~2%                  | -        |

> 💡 **提示**：Apple 官方会在 App Store Connect 后台提供你应用的用户版本分布数据，这是选择最低部署目标的最可靠依据。

### 每年 WWDC 后的适配压力

每年 6 月 WWDC 后，Apple 会发布新一代 iOS 的开发者预览版，同时宣布一批废弃 API 和新特性。开发者面临的适配压力主要来自：

- **废弃 API 警告**：Xcode 升级后，项目可能突然出现大量废弃警告
- **新审核要求**：Apple 通常要求新提交的应用在次年春季前适配最新 SDK
- **新隐私政策**：每年几乎都有新的隐私相关变更，不做适配可能被拒审
- **UI 变化**：系统 UI 风格调整可能影响自定义界面的视觉效果

### 版本适配的核心原则

版本适配应遵循以下原则：

- **向前兼容**：代码在旧版本上正常运行，在新版本上不崩溃
- **渐进适配**：优先保证功能可用，再逐步采用新 API 优化体验
- **最小影响**：适配改动应尽量局限，避免大规模重构
- **数据驱动**：基于实际用户分布数据做决策，而非主观判断

---

## 最低部署目标策略

### 如何选择最低部署目标

最低部署目标（Minimum Deployment Target）决定了你的应用能支持的最低 iOS 版本。选择时需要权衡用户覆盖率和开发成本：

| 目标版本 | 覆盖用户比例 | 建议场景 |
|----------|-------------|----------|
| iOS 16   | ~96%        | 新项目推荐，可使用 SwiftUI 4.0+ 新特性 |
| iOS 17   | ~84%        | 需要使用 iOS 17 独有 API（如 TipKit、Observable） |
| iOS 18   | ~52%        | 仅适合面向早期采用者的应用 |
| iOS 15   | ~98%+       | 需要最大用户覆盖的企业应用 |

> ⚠️ **警告**：Apple 通常要求新提交的应用使用最新版 Xcode 构建，且最低部署目标不能低于当前版本前推两个大版本。例如 Xcode 16 时代，最低部署目标通常不能低于 iOS 15。

### Xcode 中设置 Deployment Target

在 Xcode 中设置最低部署目标的步骤：

1. 选择项目导航器中的项目文件
2. 选择对应的 Target
3. 在 General 标签页中找到 Minimum Deployments 区域
4. 选择 iOS 版本

也可以在 Build Settings 中搜索 "Deployment Target" 进行设置。

对于项目配置文件（`.pbxproj`），部署目标体现为 `IPHONEOS_DEPLOYMENT_TARGET` 键值：

```
IPHONEOS_DEPLOYMENT_TARGET = 16.0;
```

### SPM 包的最低部署目标

Swift Package Manager 包的最低部署目标在 `Package.swift` 中声明：

```swift
let package = Package(
    name: "MyLibrary",
    platforms: [
        .iOS(.v16),
        .macOS(.v13)
    ],
    products: [
        .library(name: "MyLibrary", targets: ["MyLibrary"])
    ],
    targets: [
        .target(name: "MyLibrary")
    ]
)
```

> 💡 **提示**：SPM 包的最低部署目标可以高于主应用的部署目标，但不能低于它。如果包的部署目标高于应用，应用需要提高部署目标或寻找替代方案。

### 多 Target 不同部署目标

大型项目可能包含主应用、扩展（Widget、Share Extension 等）和测试 Target，它们可以设置不同的部署目标：

```swift
if #available(iOS 17.0, *) {
    let widget = WidgetConfiguration()
} else {
    let legacyWidget = LegacyWidgetConfiguration()
}
```

在 Xcode 中，每个 Target 可以独立设置 Deployment Target。例如，主应用支持 iOS 16+，而 Widget Extension 要求 iOS 17+，这是合法的配置。

---

## 条件编译与 API 可用性

### #available 运行时检查

`#available` 是 Swift 中进行运行时 API 可用性检查的核心语法，它允许你在运行时判断当前系统是否满足指定版本要求：

```swift
if #available(iOS 17.0, *) {
    let tipView = TipView()
    present(tipView, animated: true)
} else {
    let alert = UIAlertController(
        title: "提示",
        message: "此功能需要 iOS 17 及以上版本",
        preferredStyle: .alert
    )
    present(alert, animated: true)
}
```

`*` 表示"所有其他平台"，这是必须写的通配符，确保代码在未来新平台上也能编译。

### if #available 语法详解

`if #available` 支持同时检查多个平台：

```swift
if #available(iOS 17.0, macOS 14.0, *) {
    print("支持新 API")
}
```

在 `if #available` 块内，编译器允许你使用该版本引入的新 API，而 `else` 块中只能使用低版本可用的 API。

`#available` 也可以用于 `guard` 语句：

```swift
guard #available(iOS 17.0, *) else {
    showFallbackUI()
    return
}
```

### @available 属性标注

`@available` 用于标注函数、类、协议等的可用性，告诉编译器和调用方该 API 从哪个版本开始可用：

```swift
@available(iOS 17.0, *)
func configureWidget() {
}

@available(iOS, introduced: 16.0, deprecated: 18.0, message: "请使用 configureWidget() 替代")
func configureLegacyWidget() {
}
```

`@available` 支持的参数：

| 参数 | 含义 |
|------|------|
| `introduced` | API 引入的版本 |
| `deprecated` | API 废弃的版本 |
| `obsoleted` | API 移除的版本 |
| `message` | 废弃时显示的提示信息 |
| `renamed` | API 重命名后的新名称 |
| `unavailable` | 标记为不可用 |

当整个类或协议仅在新版本可用时，可以标注在类级别：

```swift
@available(iOS 17.0, *)
class ModernViewController: UIViewController {
}
```

### #unavailable（Swift 5.6+）

Swift 5.6 引入了 `#unavailable`，它是 `#available` 的反向检查，让代码更简洁：

```swift
if #unavailable(iOS 17.0) {
    showFallbackUI()
    return
}
```

等价于：

```swift
if #available(iOS 17.0, *) {
} else {
    showFallbackUI()
    return
}
```

> 💡 **提示**：`#unavailable` 不需要写 `*` 通配符，因为它的语义已经明确——只关心"不可用"的情况。

### canImport 条件编译

`canImport` 用于判断某个模块是否可以导入，常用于跨平台代码或可选依赖：

```swift
#if canImport(SnapKit)
    import SnapKit
    makeConstraintsUsingSnapKit()
#else
    makeConstraintsUsingNSLayoutConstraint()
#endif
```

这在编写跨平台库时特别有用，可以根据平台自动选择可用的依赖。

### 目标平台条件编译

`#if os()` 用于根据不同操作系统编译不同代码：

```swift
#if os(iOS)
    let platform = "iOS"
#elseif os(macOS)
    let platform = "macOS"
#elseif os(watchOS)
    let platform = "watchOS"
#elseif os(tvOS)
    let platform = "tvOS"
#endif
```

还可以结合 `targetEnvironment` 判断模拟器环境：

```swift
#if targetEnvironment(simulator)
    print("运行在模拟器")
#else
    print("运行在真机")
#endif
```

以及判断调试/发布环境：

```swift
#if DEBUG
    let baseURL = "https://dev.api.example.com"
#else
    let baseURL = "https://api.example.com"
#endif
```

---

## 废弃 API 替换策略

### 如何发现废弃 API

发现废弃 API 的主要途径：

1. **Xcode 编译警告**：升级 Xcode 后编译项目，废弃 API 会产生 `-Wdeprecated-declarations` 警告
2. **Apple 官方文档**：每个 API 的文档页面都会标注废弃版本和替代方案
3. **Xcode Release Notes**：每个 Xcode 版本的发布说明中列出废弃 API
4. **API Diffs**：Apple 提供的 SDK Diffs 工具可以对比版本间的 API 变化

在 Xcode 中开启严格废弃警告：

```
Build Settings → Swift Compiler - Warning Policies → Deprecation Warnings → Error
```

### 常见废弃 API 替换表

#### iOS 16 废弃 API

| 废弃 API | 替代方案 | 说明 |
|----------|---------|------|
| `UNNotificationSettings` 部分属性 | `UNNotificationSettings` 新属性 | 通知设置获取方式调整 |
| `UICollectionView` FlowLayout 部分方法 | `UICollectionViewCompositionalLayout` | 建议迁移到组合布局 |
| `NSAttributedString` 键值属性 | `AttributedString`（Swift 原生） | Swift 5.5+ 新类型 |
| `LAContext` 旧评估策略 | `LAPolicy.deviceOwnerAuthenticationWithBiometrics` | 生物认证 API 更新 |

#### iOS 17 废弃 API

| 废弃 API | 替代方案 | 说明 |
|----------|---------|------|
| `UIApplication.shared.statusBarFrame` | `UIView.safeAreaLayoutGuide` | 状态栏帧获取废弃 |
| `UITabBar` 旧配置方式 | `UITabBarController` 新 API | Tab Bar 配置方式重构 |
| `NSLocationManager` 旧权限方法 | `CLLocationManager` 新授权流程 | 定位权限流程更新 |
| `SKPaymentQueue`（StoreKit 1） | StoreKit 2 (`Product`/`Transaction`) | 内购全面迁移到 StoreKit 2 |

#### iOS 18 废弃 API

| 废弃 API | 替代方案 | 说明 |
|----------|---------|------|
| `UIApplication.shared.openURL(_:)` 旧签名 | `UIApplication.shared.open(_:options:completionHandler:)` | URL 打开方式更新 |
| `UNUserNotificationCenter` 旧通知类别 | `UNNotificationCategory` 新初始化方式 | 通知类别配置更新 |
| `UIDevice` 部分电池属性 | `UIDevice` 新电池信息 API | 电池信息获取方式调整 |
| 旧版 `NSControl` 事件处理 | Swift 化的新回调 API | 控件事件处理现代化 |

### 渐进式替换方案

面对大量废弃 API，不建议一次性全部替换，应采用渐进式策略：

**第一阶段：评估与分类**

```swift
enum DeprecationLevel {
    case critical
    case warning
    case info
}

func classifyDeprecation(api: String) -> DeprecationLevel {
    if api.contains("StoreKit") { return .critical }
    if api.contains("statusBar") { return .warning }
    return .info
}
```

**第二阶段：优先处理关键废弃**

- 影响审核通过的（如隐私相关）
- 影响核心功能的（如 StoreKit 1 → 2）
- 已有明确替代方案的

**第三阶段：逐步替换次要废弃**

- UI 样式相关
- 辅助功能相关
- 非核心路径

### 兼容性封装层设计

当需要在多版本间保持兼容时，可以设计封装层：

```swift
enum NotificationAuthAdapter {
    @available(iOS 17.0, *)
    static func requestModernAuthorization() async throws -> Bool {
        let center = UNUserNotificationCenter.current()
        return try await center.requestAuthorization(options: [.alert, .badge, .sound])
    }

    static func requestAuthorization(
        options: UNAuthorizationOptions = [.alert, .badge, .sound],
        completionHandler: @escaping (Bool, Error?) -> Void
    ) {
        if #available(iOS 17.0, *) {
            Task {
                do {
                    let granted = try await requestModernAuthorization()
                    completionHandler(granted, nil)
                } catch {
                    completionHandler(false, error)
                }
            }
        } else {
            UNUserNotificationCenter.current()
                .requestAuthorization(options: options, completionHandler: completionHandler)
        }
    }
}
```

封装层的设计原则：

- 对外提供统一接口，隐藏版本差异
- 内部根据 `#available` 分发到不同实现
- 优先使用新 API，旧版本回退到兼容实现
- 封装层本身应有单元测试覆盖

---

## 新版本适配工作流

### WWDC 后的适配时间线

| 时间节点 | 任务 | 优先级 |
|----------|------|--------|
| WWDC 当周（6 月） | 安装新版 Xcode Beta，编译项目查看警告 | 高 |
| 7 月 | 评估新 API，确定适配范围，创建适配分支 | 高 |
| 8 月 | 完成核心 API 适配，处理废弃 API 替换 | 高 |
| 9 月（新版本发布） | 在真机上全面测试，修复兼容性问题 | 紧急 |
| 10-11 月 | 优化新版本体验，采用新特性 | 中 |
| 12 月-次年 1 月 | 完成全部适配，提交审核 | 高 |

> ⚠️ **警告**：Apple 通常要求从次年 4 月起，新提交的应用必须使用最新版 SDK 构建。不要拖延适配工作。

### 新 API 评估与选型

面对每年 WWDC 发布的大量新 API，需要理性评估：

**评估维度：**

| 维度 | 考量因素 |
|------|---------|
| 用户覆盖率 | 新 API 对应的最低版本用户占比 |
| 功能增益 | 新 API 相比旧方案有多大提升 |
| 迁移成本 | 替换旧代码需要多少工作量 |
| 稳定性 | 新 API 是否为 Beta，是否有已知问题 |
| 回退方案 | 旧版本是否有合理的替代实现 |

**选型原则：**

- 能显著提升用户体验的优先采用
- 迁移成本低且无风险的优先采用
- 隐私和安全性相关的必须采用
- 仅美化性质的可以延后

### 适配分支策略

推荐使用 Git 分支管理适配工作：

```
main (稳定版，当前线上版本)
  └── feature/ios18-adaptation (适配分支)
        ├── feature/ios18-control-center
        ├── feature/ios18-privacy-updates
        └── feature/ios18-deprecated-api-fix
```

分支策略要点：

- 从 `main` 创建适配总分支，避免影响线上版本
- 按功能模块拆分子分支，便于 Code Review
- 适配完成并测试通过后，合并回 `main`
- 保留适配分支一段时间，便于回滚

### 测试验证要点

适配过程中的测试重点：

1. **多版本真机测试**：至少在最低部署目标和最新版本各测一遍
2. **废弃 API 警告清零**：确保编译无废弃警告
3. **新 API 回退测试**：在旧版本上验证回退逻辑正确
4. **UI 适配测试**：检查新系统 UI 变化是否影响布局
5. **性能回归测试**：新 API 的性能表现是否优于旧方案

### 上线前的兼容性检查

上线前检查清单：

- [ ] 所有废弃 API 警告已处理
- [ ] 最低部署目标版本上功能正常
- [ ] 最新 iOS 版本上功能正常
- [ ] 新隐私权限已适配
- [ ] App Store 审核指南更新项已检查
- [ ] 第三方库已更新到兼容版本
- [ ] 截图已更新为新系统外观

---

## iOS 18 适配实战

### iOS 18 关键变化一览

iOS 18 带来了多项影响开发者的变化：

| 变更类别 | 具体内容 | 影响程度 |
|----------|---------|----------|
| 控制中心 | 支持第三方小组件 | 高 |
| 隐私 | 控制中心权限、增强的 App 沙盒 | 高 |
| UI | 新的 Tab Bar 设计、侧边栏变化 | 中 |
| Swift | Swift 6 并发模式默认开启 | 高 |
| StoreKit | StoreKit 2 进一步增强 | 中 |
| 通知 | 通知管理方式更新 | 低 |

### 控制中心小组件适配

iOS 18 允许第三方应用提供控制中心小组件，这是全新的入口：

```swift
import WidgetKit
import AppIntents

@available(iOS 18.0, *)
struct QuickActionControlWidget: ControlWidget {
    var body: some ControlWidgetConfiguration {
        StaticControlConfiguration(
            kind: "com.example.quickaction"
        ) {
            ControlWidgetButton(action: QuickActionIntent()) {
                Label("快捷操作", systemImage: "bolt.fill")
            }
        }
    }
}

@available(iOS 18.0, *)
struct QuickActionIntent: AppIntent {
    static var title: LocalizedStringResource = "执行快捷操作"

    func perform() async throws -> some IntentResult {
        return .result()
    }
}
```

> 💡 **提示**：控制中心小组件需要使用 `AppIntents` 框架，与 WidgetKit 的主屏幕小组件是不同的实现方式。

### 隐私变更适配

iOS 18 在隐私方面的重要变更：

**1. 增强的 App 沙盒**

```swift
if #available(iOS 18.0, *) {
    let access = FileAccessAdapter.requestSandboxAccess()
}
```

**2. 控制中心权限**

控制中心小组件需要声明权限：

```xml
<key>NSSupportsControlCenter</key>
<true/>
```

**3. 通讯录访问进一步收紧**

```swift
if #available(iOS 18.0, *) {
    let store = CNContactStore()
    let limitedAccess = store.authorizationStatus(for: .contacts)
    switch limitedAccess {
    case .limited:
        print("用户仅授权了部分联系人")
    case .authorized:
        print("用户授权了全部联系人")
    default:
        break
    }
}
```

### 废弃 API 替换

iOS 18 中需要重点关注的废弃 API 替换：

**URL 打开方式更新：**

```swift
if #available(iOS 18.0, *) {
    let url = URL(string: "https://example.com")!
    UIApplication.shared.open(url, options: [:], completionHandler: nil)
} else {
    let url = URL(string: "https://example.com")!
    UIApplication.shared.open(url, options: [:], completionHandler: nil)
}
```

**StoreKit 1 迁移到 StoreKit 2：**

```swift
@available(iOS 15.0, *)
func fetchProducts() async throws -> [Product] {
    return try await Product.products(for: ["com.example.product"])
}

@available(iOS 15.0, *)
func purchase(_ product: Product) async throws -> StoreKit.Transaction? {
    let result = try await product.purchase()
    switch result {
    case .success(let verification):
        let transaction = try checkVerified(verification)
        await transaction.finish()
        return transaction
    case .userCancelled:
        return nil
    case .pending:
        return nil
    @unknown default:
        return nil
    }
}

@available(iOS 15.0, *)
func checkVerified<T>(_ result: VerificationResult<T>) throws -> T {
    switch result {
    case .unverified:
        throw StoreError.failedVerification
    case .verified(let safe):
        return safe
    }
}

enum StoreError: Error {
    case failedVerification
}
```

### 新 API 使用注意事项

使用 iOS 18 新 API 时需要注意：

**1. Swift 6 并发安全**

```swift
@MainActor
class ViewModel: ObservableObject {
    @Published var items: [String] = []

    func loadData() async {
        items = await fetchRemoteData()
    }

    private func fetchRemoteData() async -> [String] {
        return ["Item 1", "Item 2", "Item 3"]
    }
}
```

**2. 新 Tab Bar API**

```swift
if #available(iOS 18.0, *) {
    let tabBarController = UITabBarController()
    tabBarController.mode = .tabSidebar
    tabBarController.trailingAccessoryView = createAccessoryView()
}

@available(iOS 18.0, *)
func createAccessoryView() -> UIView {
    let view = UIView()
    return view
}
```

**3. 新 Navigation API**

```swift
if #available(iOS 18.0, *) {
    let router = UINavigationRouting()
}
```

> ⚠️ **警告**：Swift 6 的严格并发检查可能导致大量编译错误。建议先在 Swift 5 模式下完成适配，再逐步迁移到 Swift 6 并发模式。可以在 Build Settings 中设置 `SWIFT_STRICT_CONCURRENCY` 为 `minimal` 开始过渡。

---

## 小结

本章介绍了 iOS 版本适配的完整方法论，从策略制定到实战操作：

| 主题 | 核心要点 |
|------|---------|
| 版本适配原则 | 向前兼容、渐进适配、最小影响、数据驱动 |
| 最低部署目标 | 基于用户覆盖率选择，新项目建议 iOS 16+ |
| 条件编译 | `#available` 运行时检查、`@available` 编译标注、`#unavailable` 反向检查 |
| 废弃 API 替换 | 评估分类、渐进替换、封装层设计 |
| 适配工作流 | WWDC 后立即启动、分支管理、多版本测试 |
| iOS 18 适配 | 控制中心小组件、隐私变更、Swift 6 并发、StoreKit 2 迁移 |

版本适配是 iOS 开发者的年度必修课。建立系统化的适配流程，能让你每年从容应对新版本带来的变化，而不是在审核截止前手忙脚乱。

← [Xcode 项目配置深入](./Xcode项目配置深入.md) | [依赖管理与开源库](./依赖管理与开源库.md) →