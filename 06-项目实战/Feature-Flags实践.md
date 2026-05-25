# Feature Flags 实践

> 🎯 **本章目标**：理解 Feature Flags 的核心概念与价值，掌握在 iOS 项目中实现 Feature Flags 的方法，学会灰度发布与 A/B 测试的基础设施搭建。

---

## Feature Flags 概述

### 什么是 Feature Flags

Feature Flags（功能开关）是一种通过配置控制功能启用或禁用的技术手段。开发者将功能的开关逻辑从代码的条件判断中抽离出来，交给外部配置管理，从而实现**无需发版即可控制功能的开启与关闭**。

传统开发流程中，功能的上线与下线完全依赖 App Store 审核与发版周期。而 Feature Flags 打破了这一限制，让团队可以在运行时动态控制功能状态。

### 生活类比

Feature Flags 就像电灯开关——不需要重新布线（重新发版），只需拨动开关（修改配置），就能控制灯的亮灭（功能的开关）。你甚至可以装一个调光器（灰度百分比），让灯光慢慢变亮（逐步放量）。

### 核心价值

| 价值 | 说明 |
|------|------|
| 降低发布风险 | 新功能默认关闭，验证无误后再开启，避免"一发布就回滚" |
| 灰度发布 | 按百分比或用户分批放量，控制影响范围 |
| A/B 测试 | 不同用户看到不同功能变体，用数据驱动决策 |
| 紧急功能关闭 | 线上出现严重问题时，一键关闭功能，无需紧急发版 |

### Feature Flags 的类型

| 类型 | 说明 | 典型场景 |
|------|------|----------|
| Release Flag | 控制未完成功能对用户的可见性，发布后移除 | 新功能开发中，合入主分支但不暴露 |
| Ops Flag | 运维开关，用于紧急关闭或降级功能 | 服务端接口异常时关闭相关功能 |
| Experiment Flag | A/B 测试开关，控制实验分组 | 测试两种推荐算法的转化率差异 |
| Permission Flag | 权限开关，控制特定用户群体的功能访问 | VIP 用户专属功能、内测功能 |

---

## 本地 Feature Flags 实现

### 基于 plist 配置的简单方案

最基础的 Feature Flags 实现方式是将开关配置写在 plist 文件中，App 启动时读取配置。

首先创建 `FeatureFlags.plist` 文件：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>enable_dark_theme</key>
    <false/>
    <key>enable_new_onboarding</key>
    <true/>
    <key>enable_payment_v2</key>
    <false/>
</dict>
</plist>
```

然后封装读取逻辑：

```swift
final class FeatureFlagManager {
    static let shared = FeatureFlagManager()

    private var flags: [String: Bool] = [:]

    private init() {
        loadFlags()
    }

    private func loadFlags() {
        guard let url = Bundle.main.url(forResource: "FeatureFlags", withExtension: "plist"),
              let data = try? Data(contentsOf: url),
              let plist = try? PropertyListSerialization.propertyList(from: data, options: [], format: nil) as? [String: Bool] else {
            return
        }
        flags = plist
    }

    func isEnabled(_ key: String) -> Bool {
        flags[key] ?? false
    }
}
```

### 基于 UserDefaults 的运行时开关

plist 方案的局限是只读，无法在运行时修改。结合 UserDefaults 可以实现运行时动态切换：

```swift
final class FeatureFlagManager {
    static let shared = FeatureFlagManager()

    private let defaults = UserDefaults.standard
    private var localFlags: [String: Bool] = [:]

    private init() {
        loadLocalFlags()
    }

    private func loadLocalFlags() {
        guard let url = Bundle.main.url(forResource: "FeatureFlags", withExtension: "plist"),
              let data = try? Data(contentsOf: url),
              let plist = try? PropertyListSerialization.propertyList(from: data, options: [], format: nil) as? [String: Bool] else {
            return
        }
        localFlags = plist
    }

    func isEnabled(_ key: String) -> Bool {
        if defaults.object(forKey: key) != nil {
            return defaults.bool(forKey: key)
        }
        return localFlags[key] ?? false
    }

    func setFlag(_ key: String, enabled: Bool) {
        defaults.set(enabled, forKey: key)
        NotificationCenter.default.post(name: .featureFlagDidChange, object: nil, userInfo: ["key": key])
    }

    func resetFlag(_ key: String) {
        defaults.removeObject(forKey: key)
    }
}

extension Notification.Name {
    static let featureFlagDidChange = Notification.Name("featureFlagDidChange")
}
```

> 💡 **提示**：UserDefaults 的值会持久化，适合开发调试阶段使用。生产环境中应避免让用户随意修改 Flag 值。

### Swift Enum 类型安全的 Flags 定义

使用字符串 key 容易出错，通过 Enum 可以实现类型安全：

```swift
enum FeatureFlag: String, CaseIterable {
    case darkTheme = "enable_dark_theme"
    case newOnboarding = "enable_new_onboarding"
    case paymentV2 = "enable_payment_v2"
    case aiRecommendation = "enable_ai_recommendation"

    var isOn: Bool {
        FeatureFlagManager.shared.isEnabled(self.rawValue)
    }
}

final class FeatureFlagManager {
    static let shared = FeatureFlagManager()

    private let defaults = UserDefaults.standard
    private var localFlags: [String: Bool] = [:]

    private init() {
        loadLocalFlags()
    }

    private func loadLocalFlags() {
        guard let url = Bundle.main.url(forResource: "FeatureFlags", withExtension: "plist"),
              let data = try? Data(contentsOf: url),
              let plist = try? PropertyListSerialization.propertyList(from: data, options: [], format: nil) as? [String: Bool] else {
            return
        }
        localFlags = plist
    }

    func isEnabled(_ flag: FeatureFlag) -> Bool {
        let key = flag.rawValue
        if defaults.object(forKey: key) != nil {
            return defaults.bool(forKey: key)
        }
        return localFlags[key] ?? false
    }

    func setFlag(_ flag: FeatureFlag, enabled: Bool) {
        defaults.set(enabled, forKey: flag.rawValue)
        NotificationCenter.default.post(name: .featureFlagDidChange, object: nil)
    }
}
```

使用时非常简洁：

```swift
if FeatureFlag.darkTheme.isOn {
    applyDarkTheme()
}
```

### SwiftUI 中的 Feature Flags 使用

SwiftUI 的声明式语法与 Feature Flags 天然契合，可以直接在 View 层做条件渲染：

```swift
struct HomeView: View {
    var body: some View {
        VStack {
            if FeatureFlag.newOnboarding.isOn {
                NewOnboardingView()
            } else {
                LegacyOnboardingView()
            }
        }
        .onReceive(NotificationCenter.default.publisher(for: .featureFlagDidChange)) { _ in
        }
    }
}
```

对于需要动态响应 Flag 变化的场景，可以封装一个响应式的 Flag 观察器：

```swift
final class FeatureFlagObserver: ObservableObject {
    @Published var darkTheme: Bool = FeatureFlag.darkTheme.isOn
    @Published var newOnboarding: Bool = FeatureFlag.newOnboarding.isOn

    init() {
        NotificationCenter.default.addObserver(
            self,
            selector: #selector(flagsDidChange),
            name: .featureFlagDidChange,
            object: nil
        )
    }

    @objc private func flagsDidChange() {
        darkTheme = FeatureFlag.darkTheme.isOn
        newOnboarding = FeatureFlag.newOnboarding.isOn
    }
}

struct SettingsView: View {
    @StateObject private var flags = FeatureFlagObserver()

    var body: some View {
        List {
            Toggle("深色主题", isOn: $flags.darkTheme)
                .onChange(of: flags.darkTheme) { newValue in
                    FeatureFlagManager.shared.setFlag(.darkTheme, enabled: newValue)
                }
            Toggle("新引导流程", isOn: $flags.newOnboarding)
                .onChange(of: flags.newOnboarding) { newValue in
                    FeatureFlagManager.shared.setFlag(.newOnboarding, enabled: newValue)
                }
        }
    }
}
```

---

## 远程 Feature Flags

### 为什么需要远程控制

本地 Feature Flags 只能修改设备上的配置，无法做到"改一个配置，所有用户立即生效"。远程 Feature Flags 的核心优势：

- **无需发版**即可开关功能，响应时间从"天级"缩短到"分钟级"
- **统一管控**所有用户的 Flag 状态，避免配置不一致
- **精细化控制**可以按地区、用户群、百分比等维度分发

### Firebase Remote Config

Firebase Remote Config 是 Google 提供的远程配置服务，支持条件化配置下发：

```swift
import FirebaseRemoteConfig

final class RemoteConfigManager {
    static let shared = RemoteConfigManager()

    private let remoteConfig = RemoteConfig.remoteConfig()

    private init() {
        let settings = RemoteConfigSettings()
        settings.minimumFetchInterval = 3600
        remoteConfig.configSettings = settings

        remoteConfig.setDefaults(fromPlist: "FeatureFlags")
    }

    func fetchAndActivate(completion: @escaping (Bool) -> Void) {
        remoteConfig.fetch(withExpirationDuration: 3600) { [weak self] status, error in
            guard let self, status == .success else {
                completion(false)
                return
            }
            self.remoteConfig.activate { _, error in
                completion(error == nil)
            }
        }
    }

    func boolValue(for flag: FeatureFlag) -> Bool {
        remoteConfig[flag.rawValue].boolValue
    }

    func stringValue(for key: String) -> String {
        remoteConfig[key].stringValue ?? ""
    }
}
```

在 App 启动时拉取远程配置：

```swift
@main
struct MyApp: App {
    init() {
        RemoteConfigManager.shared.fetchAndActivate { success in
            if success {
                NotificationCenter.default.post(name: .featureFlagDidChange, object: nil)
            }
        }
    }

    var body: some Scene {
        WindowGroup {
            ContentView()
        }
    }
}
```

### 国内替代方案

由于 Firebase 在国内访问不稳定，以下是常用的国内替代方案：

| 方案 | 提供方 | 特点 | 免费额度 |
|------|--------|------|----------|
| Firebase Remote Config | Google | 功能完善，生态成熟 | 每日 1M 次请求 |
| 腾讯移动分析 MTAX | 腾讯 | 国内稳定，集成简单 | 基础版免费 |
| 神策数据 | 神策 | A/B 测试能力强，企业级 | 需商务洽谈 |
| GrowingIO | GrowingIO | 可视化配置，运营友好 | 需商务洽谈 |
| 自建远程配置 | 自研 | 完全可控，定制化强 | 服务器成本 |
| Apollo / Nacos | 开源 | 配置中心，适合后端驱动 | 自部署免费 |

> 💡 **提示**：对于个人开发者或小团队，推荐使用自建远程配置方案——一个简单的 JSON 接口配合 CDN 缓存即可满足大部分需求。

### 远程配置的缓存与降级策略

远程配置依赖网络，必须设计好缓存与降级策略，确保离线或网络异常时 App 依然可用：

```swift
final class FeatureFlagManager {
    static let shared = FeatureFlagManager()

    private let defaults = UserDefaults.standard
    private var localFlags: [String: Bool] = [:]
    private var cachedRemoteFlags: [String: Bool] = [:]

    private enum CacheKey {
        static let remoteFlags = "cached_remote_feature_flags"
        static let lastFetchTime = "last_remote_fetch_timestamp"
    }

    private init() {
        loadLocalFlags()
        loadCachedRemoteFlags()
    }

    private func loadLocalFlags() {
        guard let url = Bundle.main.url(forResource: "FeatureFlags", withExtension: "plist"),
              let data = try? Data(contentsOf: url),
              let plist = try? PropertyListSerialization.propertyList(from: data, options: [], format: nil) as? [String: Bool] else {
            return
        }
        localFlags = plist
    }

    private func loadCachedRemoteFlags() {
        if let data = defaults.data(forKey: CacheKey.remoteFlags),
           let decoded = try? JSONDecoder().decode([String: Bool].self, from: data) {
            cachedRemoteFlags = decoded
        }
    }

    private func cacheRemoteFlags(_ flags: [String: Bool]) {
        cachedRemoteFlags = flags
        if let data = try? JSONEncoder().encode(flags) {
            defaults.set(data, forKey: CacheKey.remoteFlags)
            defaults.set(Date().timeIntervalSince1970, forKey: CacheKey.lastFetchTime)
        }
    }

    func isEnabled(_ flag: FeatureFlag) -> Bool {
        let key = flag.rawValue

        if let remoteValue = cachedRemoteFlags[key] {
            return remoteValue
        }

        if defaults.object(forKey: key) != nil {
            return defaults.bool(forKey: key)
        }

        return localFlags[key] ?? false
    }

    func updateFromRemote(_ remoteFlags: [String: Bool]) {
        cacheRemoteFlags(remoteFlags)
        NotificationCenter.default.post(name: .featureFlagDidChange, object: nil)
    }
}
```

> ⚠️ **警告**：远程配置的优先级必须高于本地配置，否则远程修改将无法生效。但本地配置作为兜底，确保在远程配置不可用时 App 仍有合理的行为。

### 实时更新机制

Firebase Remote Config 默认不是实时的，需要主动 fetch。如果需要实时性，可以采用以下方案：

```swift
final class FeatureFlagSyncManager {
    private var timer: Timer?

    func startPolling(interval: TimeInterval = 300) {
        timer = Timer.scheduledTimer(withTimeInterval: interval, repeats: true) { _ in
            RemoteConfigManager.shared.fetchAndActivate { success in
                if success {
                    NotificationCenter.default.post(name: .featureFlagDidChange, object: nil)
                }
            }
        }
    }

    func stopPolling() {
        timer?.invalidate()
        timer = nil
    }
}
```

对于更高实时性要求的场景，可以使用 WebSocket 长连接或 Server-Sent Events（SSE）推送配置变更。

---

## 灰度发布实践

### 灰度发布的核心思想

灰度发布（Canary Release）是指将新功能逐步开放给部分用户，在确认稳定后再全量发布。核心思想是**缩小爆炸半径**——即使新功能有问题，也只影响小部分用户。

### 基于用户 ID 的灰度策略

通过对用户 ID 取模，可以精确控制哪些用户看到新功能：

```swift
struct GrayscaleStrategy {
    static func shouldEnable(feature: FeatureFlag, userId: String) -> Bool {
        let hashValue = abs(userId.hashValue)
        let bucket = hashValue % 100

        let allowList = getAllowList(for: feature)
        if allowList.contains(userId) {
            return true
        }

        let percentage = getPercentage(for: feature)
        return bucket < percentage
    }

    private static func getPercentage(for feature: FeatureFlag) -> Int {
        switch feature {
        case .aiRecommendation:
            return 10
        default:
            return 100
        }
    }

    private static func getAllowList(for feature: FeatureFlag) -> Set<String> {
        switch feature {
        case .aiRecommendation:
            return ["user_test_001", "user_test_002"]
        default:
            return []
        }
    }
}
```

使用方式：

```swift
let userId = currentUser.id
if GrayscaleStrategy.shouldEnable(feature: .aiRecommendation, userId: userId) {
    showAIRecommendation()
} else {
    showClassicRecommendation()
}
```

### 基于百分比的灰度策略

百分比灰度更灵活，可以动态调整放量比例：

```swift
final class GrayscaleManager {
    struct GrayscaleRule {
        let feature: FeatureFlag
        var percentage: Int
        var allowList: Set<String>
        var blockList: Set<String>
    }

    private var rules: [FeatureFlag: GrayscaleRule] = [:]

    static let shared = GrayscaleManager()

    private init() {
        rules = [
            .aiRecommendation: GrayscaleRule(
                feature: .aiRecommendation,
                percentage: 5,
                allowList: [],
                blockList: []
            ),
            .paymentV2: GrayscaleRule(
                feature: .paymentV2,
                percentage: 20,
                allowList: ["vip_user_001"],
                blockList: []
            )
        ]
    }

    func shouldEnable(feature: FeatureFlag, userId: String) -> Bool {
        guard let rule = rules[feature] else { return false }

        if rule.blockList.contains(userId) {
            return false
        }

        if rule.allowList.contains(userId) {
            return true
        }

        let hashValue = abs(userId.hashValue)
        let bucket = hashValue % 100
        return bucket < rule.percentage
    }

    func updatePercentage(feature: FeatureFlag, percentage: Int) {
        rules[feature]?.percentage = max(0, min(100, percentage))
    }

    func addToAllowList(feature: FeatureFlag, userId: String) {
        rules[feature]?.allowList.insert(userId)
    }

    func addToBlockList(feature: FeatureFlag, userId: String) {
        rules[feature]?.blockList.insert(userId)
    }
}
```

> 💡 **提示**：使用 `hashValue % 100` 的方式确保同一用户每次判断结果一致，避免用户反复看到功能开关切换。但 `hashValue` 在不同 Swift 版本间可能不一致，生产环境建议使用更稳定的哈希算法（如 MD5 或 SHA256）。

### SwiftUI 中的灰度逻辑

在 SwiftUI 中，可以封装一个通用的灰度视图修饰符：

```swift
struct GrayscaleFeature<TrueContent: View, FalseContent: View>: View {
    let feature: FeatureFlag
    let userId: String
    let trueContent: () -> TrueContent
    let falseContent: () -> FalseContent

    var body: some View {
        if GrayscaleManager.shared.shouldEnable(feature: feature, userId: userId) {
            trueContent()
        } else {
            falseContent()
        }
    }
}

struct HomeView: View {
    let userId: String

    var body: some View {
        ScrollView {
            VStack {
                GrayscaleFeature(
                    feature: .aiRecommendation,
                    userId: userId
                ) {
                    AIRecommendationSection()
                } falseContent: {
                    ClassicRecommendationSection()
                }
            }
        }
    }
}
```

### 灰度发布的监控与回退

灰度发布必须配套监控，否则无法判断新功能是否正常：

```swift
struct GrayscaleMetrics {
    var feature: FeatureFlag
    var exposedUsers: Int = 0
    var errorCount: Int = 0
    var crashCount: Int = 0
    var avgLoadTime: TimeInterval = 0

    var errorRate: Double {
        guard exposedUsers > 0 else { return 0 }
        return Double(errorCount) / Double(exposedUsers)
    }

    var shouldRollback: Bool {
        errorRate > 0.05 || crashCount > 3
    }
}

final class GrayscaleMonitor {
    static let shared = GrayscaleMonitor()

    private var metrics: [FeatureFlag: GrayscaleMetrics] = [:]

    func recordExposure(feature: FeatureFlag) {
        metrics[feature, default: GrayscaleMetrics(feature: feature)].exposedUsers += 1
    }

    func recordError(feature: FeatureFlag) {
        metrics[feature, default: GrayscaleMetrics(feature: feature)].errorCount += 1
        checkAndRollback(feature: feature)
    }

    func recordCrash(feature: FeatureFlag) {
        metrics[feature, default: GrayscaleMetrics(feature: feature)].crashCount += 1
        checkAndRollback(feature: feature)
    }

    private func checkAndRollback(feature: FeatureFlag) {
        guard let m = metrics[feature], m.shouldRollback else { return }
        logRollback(feature: feature, metrics: m)
        GrayscaleManager.shared.updatePercentage(feature: feature, percentage: 0)
    }

    private func logRollback(feature: FeatureFlag, metrics: GrayscaleMetrics) {
        print("灰度回退: \(feature.rawValue), 错误率=\(metrics.errorRate), 崩溃数=\(metrics.crashCount)")
    }
}
```

> ⚠️ **警告**：灰度回退机制必须自动化。线上问题每多存在一分钟，就多影响一批用户。设置合理的阈值（如错误率 > 5% 自动回退），比人工判断更及时。

---

## Feature Flags 最佳实践

### Flag 命名规范

好的命名能让团队一眼看出 Flag 的用途：

```swift
enum FeatureFlag: String, CaseIterable {
    case enableDarkTheme = "enable_dark_theme"
    case enableNewOnboarding = "enable_new_onboarding"
    case enablePaymentV2 = "enable_payment_v2"
    case enableAIRecommendation = "enable_ai_recommendation"
}
```

命名规则：

- 前缀统一使用 `enable_` 表示开关
- 使用下划线分隔单词（snake_case）
- 名称应描述功能而非实现，如 `enable_payment_v2` 而非 `enable_stripe_integration`
- 避免使用否定词，如用 `enable_legacy_mode` 而非 `disable_new_mode`

### Flag 的生命周期管理

每个 Feature Flag 都有明确的生命周期：

| 阶段 | 说明 | 持续时间 |
|------|------|----------|
| 创建 | 新功能开发时创建 Flag，默认关闭 | 开发开始时 |
| 启用 | 功能验证通过后开启 Flag | 测试/灰度阶段 |
| 全量 | 功能对所有用户开放 | 灰度完成 |
| 清理 | 移除 Flag 代码，功能成为默认行为 | 全量后 1-2 周 |

```swift
enum FeatureFlag: String, CaseIterable {
    case enableDarkTheme = "enable_dark_theme"
    case enableNewOnboarding = "enable_new_onboarding"

    var lifecycle: FlagLifecycle {
        switch self {
        case .enableDarkTheme:
            return .active(since: "2025-01-15")
        case .enableNewOnboarding:
            return .scheduledForRemoval(deadline: "2025-06-01")
        }
    }
}

enum FlagLifecycle {
    case active(since: String)
    case scheduledForRemoval(deadline: String)
    case permanent
}
```

### 技术债务：过期 Flag 的清理

Feature Flags 是有成本的技术债务。每个 Flag 都增加了代码的分支复杂度，长期不清理会导致：

- 代码可读性下降，`if-else` 分支越来越多
- 测试组合爆炸，每个 Flag 的开/关都需要测试
- 维护成本上升，新人难以理解代码逻辑

清理策略：

```swift
protocol FeatureFlagCleanable {
    var removalDeadline: Date { get }
    var isOverdue: Bool { get }
}

extension FeatureFlag: FeatureFlagCleanable {
    var removalDeadline: Date {
        let formatter = DateFormatter()
        formatter.dateFormat = "yyyy-MM-dd"
        switch self {
        case .enableNewOnboarding:
            return formatter.date(from: "2025-06-01") ?? Date.distantFuture
        default:
            return Date.distantFuture
        }
    }

    var isOverdue: Bool {
        Date() > removalDeadline
    }
}
```

> 💡 **提示**：建议在 CI 流水线中加入 Flag 过期检查，当 Flag 超过清理期限时发出警告。也可以在代码审查时将过期 Flag 作为必须处理项。

### 测试策略：如何测试不同 Flag 组合

Feature Flags 增加了测试的排列组合数量，需要系统化的测试策略：

```swift
import XCTest

final class FeatureFlagTests: XCTestCase {
    private var flagManager: FeatureFlagManager!

    override func setUp() {
        super.setUp()
        flagManager = FeatureFlagManager.shared
    }

    func testDarkThemeEnabled() {
        flagManager.setFlag(.darkTheme, enabled: true)
        XCTAssertTrue(FeatureFlag.darkTheme.isOn)
    }

    func testDarkThemeDisabled() {
        flagManager.setFlag(.darkTheme, enabled: false)
        XCTAssertFalse(FeatureFlag.darkTheme.isOn)
    }

    func testGrayscaleStrategy() {
        let strategy = GrayscaleStrategy.self

        let testUserId = "user_test_001"
        XCTAssertTrue(strategy.shouldEnable(feature: .aiRecommendation, userId: testUserId))

        let normalUserId = "normal_user_999"
        let result = strategy.shouldEnable(feature: .aiRecommendation, userId: normalUserId)
        XCTAssertNotNil(result as Any)
    }

    func testFlagCombination() {
        flagManager.setFlag(.darkTheme, enabled: true)
        flagManager.setFlag(.newOnboarding, enabled: false)

        XCTAssertTrue(FeatureFlag.darkTheme.isOn)
        XCTAssertFalse(FeatureFlag.newOnboarding.isOn)
    }

    override func tearDown() {
        for flag in FeatureFlag.allCases {
            flagManager.resetFlag(flag.rawValue)
        }
        super.tearDown()
    }
}
```

测试要点：

- 每个 Flag 单独测试开启和关闭两种状态
- 测试 Flag 组合的边界场景
- 测试灰度策略的白名单和百分比逻辑
- 测试远程配置降级场景（网络不可用时回退本地配置）

### 安全考虑：客户端 Flag 可被篡改

客户端的 Feature Flags 存在被篡改的风险：

- 越狱设备可以直接修改 UserDefaults 或 plist 文件
- 中间人攻击可以篡改远程配置的响应
- 逆向工程可以绕过 Flag 检查逻辑

安全防护措施：

```swift
final class SecureFeatureFlagManager {
    static let shared = SecureFeatureFlagManager()

    private let keychain = KeychainHelper.shared

    func isEnabled(_ flag: FeatureFlag) -> Bool {
        if let serverFlag = fetchServerSideFlag(flag) {
            return serverFlag
        }

        if isSecurityCritical(flag) {
            return false
        }

        return FeatureFlagManager.shared.isEnabled(flag)
    }

    private func fetchServerSideFlag(_ flag: FeatureFlag) -> Bool? {
        return nil
    }

    private func isSecurityCritical(_ flag: FeatureFlag) -> Bool {
        switch flag {
        case .paymentV2:
            return true
        default:
            return false
        }
    }
}
```

> ⚠️ **警告**：涉及付费、权限、安全相关的 Flag（如支付功能开关）必须在服务端判断，不能仅依赖客户端 Flag。客户端 Flag 只应控制 UI 展示逻辑，核心业务逻辑应由服务端保障。

---

## 小结

| 知识点 | 核心内容 |
|------------|------------|
| Feature Flags 概念 | 通过配置控制功能开关，无需发版 |
| 本地实现方案 | plist + UserDefaults + Enum 类型安全 |
| 远程 Feature Flags | Firebase Remote Config / 国内替代方案 |
| 缓存与降级 | 远程 > 本地缓存 > plist 兜底 |
| 灰度发布策略 | 用户 ID 白名单 + 百分比放量 |
| 灰度监控 | 错误率、崩溃数、自动回退 |
| Flag 命名规范 | `enable_` 前缀 + snake_case + 描述功能 |
| Flag 生命周期 | 创建 → 启用 → 全量 → 清理 |
| 技术债务管理 | 设置清理期限，CI 过期检查 |
| 测试策略 | 单 Flag 测试 + 组合测试 + 灰度策略测试 |
| 安全考虑 | 安全关键 Flag 必须服务端判断 |

Feature Flags 是现代 iOS 开发中不可或缺的基础设施。从最简单的 plist 配置到远程配置中心，从手动开关到自动灰度回退，每一步都在降低发布风险、提升迭代效率。掌握 Feature Flags，就是掌握了"安全地快速迭代"的能力。

← [Xcode Cloud CI/CD](./Xcode-Cloud-CI-CD.md) | [Fastlane 自动化构建](./Fastlane自动化构建.md) →