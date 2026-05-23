---
name: logging-monitoring
description: 涉及日志、崩溃监控、Crashlytics、OSLog、Analytics 埋点、用户行为追踪、APM、性能监控、错误上报的任务
---

# 日志与监控

## 日志系统

### OSLog（推荐）

```swift
import os.log

enum AppLog {
    static let general = Logger(subsystem: "com.app", category: "General")
    static let network = Logger(subsystem: "com.app", category: "Network")
    static let storage = Logger(subsystem: "com.app", category: "Storage")
    static let ui = Logger(subsystem: "com.app", category: "UI")
}
```

### 日志级别

```swift
AppLog.general.debug("调试信息: \(detail)")
AppLog.general.info("常规信息: 用户登录成功")
AppLog.network.warning("网络警告: 请求超时，将重试")
AppLog.storage.error("存储错误: CoreData 保存失败 - \(error)")
AppLog.general.fault("严重错误: 数据损坏")
```

| 级别 | 用途 | Release 可见 | 性能影响 |
|------|------|-------------|---------|
| debug | 开发调试 | ❌ | 极低（编译时移除） |
| info | 关键流程 | ✅ | 低 |
| warning | 异常但可恢复 | ✅ | 低 |
| error | 功能失败 | ✅ | 中 |
| fault | 严重错误 | ✅ | 中 |

### 规范
- **禁止使用 `print()`**，统一用 `Logger`
- subsystem 用 Bundle Identifier，category 按模块划分
- 日志中**禁止包含敏感数据**（Token、密码、身份证号）
- Debug 日志用 `#if DEBUG` 或 `Logger.debug`（Release 自动过滤）
- 日志格式：`{操作} - {对象} - {结果/原因}`

### Console 过滤
```
subsystem:com.app AND category:Network
level:error OR level:fault
```

---

## 崩溃监控

### Firebase Crashlytics

#### 配置
1. SPM 添加 `firebase-ios-sdk`，Product 选 `FirebaseCrashlytics`
2. Build Phase 添加脚本：
```bash
"${BUILD_DIR%/Build/*}/SourcePackages/checkouts/firebase-ios-sdk/Crashlytics/run"
```
3. `Info.plist` 添加 `FirebaseCrashlyticsCollectionEnabled = NO`（延迟初始化）

#### 初始化

```swift
import FirebaseCore
import FirebaseCrashlytics

func application(_ application: UIApplication,
    didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
    FirebaseApp.configure()
    Crashlytics.crashlytics().setCrashlyticsCollectionEnabled(true)
    return true
}
```

#### 自定义键值

```swift
Crashlytics.crashlytics().setUserID(userId)
Crashlytics.crashlytics().setCustomValue("premium", forKey: "subscription_tier")
Crashlytics.crashlytics().setCustomValue(true, forKey: "has_completed_onboarding")
```

#### 记录非致命异常

```swift
do {
    try performRiskyOperation()
} catch {
    Crashlytics.crashlytics().record(error: error)
}
```

#### 添加日志上下文

```swift
Crashlytics.crashlytics().log("进入支付页面")
Crashlytics.crashlytics().log("产品 ID: \(productId)")
```

### 规范
- **Release 必须启用崩溃监控**，Debug 可关闭
- `setUserID` 使用匿名 ID，**禁止使用邮箱或手机号**
- 关键业务流程添加 `log()` 上下文，崩溃时能看到用户操作路径
- 非致命错误用 `record(error:)`，**禁止用 `fatalError()` 代替**
- 崩溃报告在下次启动时上传，测试时用 `Crashlytics.crashlytics().sendUnsentReports()`

---

## Analytics 埋点

### 事件设计规范

| 类别 | 事件名格式 | 示例 |
|------|-----------|------|
| 页面浏览 | `page_{name}` | `page_home`, `page_settings` |
| 用户操作 | `action_{object}_{verb}` | `action_paywall_tap`, `action_photo_capture` |
| 业务事件 | `{domain}_{event}` | `purchase_success`, `login_complete` |
| 错误事件 | `error_{domain}` | `error_network_timeout`, `error_payment_failed` |

### 埋点参数规范

```swift
struct AnalyticsEvent {
    let name: String
    let parameters: [String: Any]?

    static func pageView(_ page: String, extra: [String: Any]? = nil) -> AnalyticsEvent {
        AnalyticsEvent(name: "page_\(page)", parameters: extra)
    }

    static func action(_ object: String, _ verb: String, extra: [String: Any]? = nil) -> AnalyticsEvent {
        AnalyticsEvent(name: "action_\(object)_\(verb)", parameters: extra)
    }
}
```

### 埋点服务封装

```swift
protocol AnalyticsServiceProtocol {
    func logEvent(_ event: AnalyticsEvent)
    func setUserProperty(_ value: String, for key: String)
    func setUserID(_ id: String)
}

final class AnalyticsService: AnalyticsServiceProtocol {
    private let providers: [AnalyticsProvider]

    init(providers: [AnalyticsProvider]) {
        self.providers = providers
    }

    func logEvent(_ event: AnalyticsEvent) {
        providers.forEach { $0.logEvent(event) }
    }

    func setUserProperty(_ value: String, for key: String) {
        providers.forEach { $0.setUserProperty(value, for: key) }
    }

    func setUserID(_ id: String) {
        providers.forEach { $0.setUserID(id) }
    }
}
```

### Firebase Analytics 集成

```swift
final class FirebaseAnalyticsProvider: AnalyticsProvider {
    func logEvent(_ event: AnalyticsEvent) {
        Analytics.logEvent(event.name, parameters: event.parameters)
    }

    func setUserProperty(_ value: String, for key: String) {
        Analytics.setUserProperty(value, forName: key)
    }

    func setUserID(_ id: String) {
        Analytics.setUserID(id)
    }
}
```

### 必须埋点的事件

| 事件 | 参数 | 时机 |
|------|------|------|
| `app_launch` | `is_cold_start`, `launch_time_ms` | App 启动 |
| `page_{name}` | `source` | 页面展示 |
| `paywall_shown` | `trigger_source` | Paywall 展示 |
| `purchase_attempt` | `product_id`, `price` | 点击购买 |
| `purchase_success` | `product_id`, `transaction_id` | 购买成功 |
| `purchase_failed` | `product_id`, `error_code` | 购买失败 |
| `error_{domain}` | `error_code`, `error_message` | 错误发生 |

### 规范
- **禁止在埋点参数中包含 PII**（个人身份信息：邮箱、手机号、身份证号）
- 用户 ID 使用匿名 UUID，不使用邮箱或手机号
- 埋点参数值类型统一：数值用 `Int`/`Double`，字符串用 `String`，**禁止混用**
- 埋点调用放在 ViewModel 层，**禁止在 VC 中直接调用**
- 新增埋点必须在文档中记录事件名和参数说明

---

## 性能监控

### Firebase Performance

```swift
import FirebasePerformance

func measureNetworkRequest(url: URL) {
    let metric = HTTPMetric(url: url, httpMethod: .get)
    metric.start()

    URLSession.shared.dataTask(with: url) { _, _, _ in
        metric.stop()
    }.resume()
}

func measureCustomTrace(name: String, block: () -> Void) {
    let trace = Performance.startTrace(name: name)
    block()
    trace.stop()
}
```

### 自定义性能指标

```swift
func measureStartup() {
    let trace = Performance.startTrace(name: "app_startup")
    trace.setValue?(1, forAttribute: "is_cold_start")

    DispatchQueue.main.async {
        trace.stop()
    }
}
```

### 关键指标监控

| 指标 | 监控方式 | 告警阈值 |
|------|---------|---------|
| 冷启动时间 | Firebase Trace | > 2s |
| API 响应时间 | HTTPMetric | > 3s (P95) |
| 崩溃率 | Crashlytics | > 1% |
| ANR 率 | 自定义监控 | > 0.5% |
| 内存峰值 | 自定义监控 | > 300MB |

---

## 用户反馈

### 反馈收集

```swift
struct FeedbackCollector {
    static func promptReview() {
        guard let scene = UIApplication.shared.connectedScenes.first as? UIWindowScene else { return }
        SKStoreReviewController.requestReview(in: scene)
    }

    static func collectFeedback(userId: String, message: String, screenshot: UIImage?) async throws {
        var parameters: [String: Any] = [
            "user_id": userId,
            "message": message,
            "app_version": Bundle.main.appVersion,
            "os_version": UIDevice.current.systemVersion,
            "device_model": UIDevice.current.model
        ]
        if let screenshot, let data = screenshot.jpegData(compressionQuality: 0.5) {
            parameters["screenshot_size"] = data.count
        }
        try await NetworkService.shared.post("/api/feedback", parameters: parameters)
    }
}
```

### 评分请求规范
- 使用 `SKStoreReviewController`，系统控制弹窗频率（每年最多 3 次）
- **禁止在用户遇到错误时请求评分**
- 请求时机：用户完成核心操作后（如成功导出、完成购买）
- **禁止自定义评分弹窗 UI**，必须用系统 API

---

## 规范

### 日志规范
- 日志保留 7 天，自动清理
- Release 日志级别 ≥ info，Debug 包含 debug
- 日志文件大小上限 10MB
- **禁止在日志中记录完整请求/响应体**（可能含敏感数据）

### 崩溃监控规范
- Release 必须启用，Debug 可选
- 崩溃率目标 < 0.5%
- 每次发版前检查 Crashlytics 未解决崩溃
- 崩溃修复优先级 > 新功能开发

### 埋点规范
- 埋点代码与业务代码分离（通过 Service 层调用）
- 埋点必须有文档记录（事件名、参数、触发时机）
- 新功能开发时同步设计埋点方案
- **禁止在 Release 中使用调试埋点**（如按钮点击坐标）

### 隐私合规
- 埋点数据传输必须 HTTPS
- 用户可关闭数据收集（GDPR / 隐私合规）
- App 首次启动展示隐私政策，用户同意后才启用 Analytics
- 中国大陆：需符合《个人信息保护法》，收集数据前获取用户同意

---

## 已知陷阱

- **Crashlytics 在 Debug 模式下默认不上报**，测试时需手动触发
- **Firebase Analytics 事件名最大 40 字符**，参数名最大 40 字符，参数值最大 100 字符
- **Firebase Analytics 事件参数最多 25 个**
- **OSLog 的 `debug` 级别在 Release 中编译时被移除**，不能用于 Release 诊断
- **`SKStoreReviewController` 的弹窗由系统控制**，调用后不一定弹出
- **Firebase Performance 自动监控需要添加 `FirebasePerformance` 框架**，不是默认包含的
- **Crashlytics 的 `log()` 最多保留最近 64 条**，超出会丢弃旧的
