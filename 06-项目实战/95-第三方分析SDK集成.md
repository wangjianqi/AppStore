# 95-第三方分析SDK集成

> 🎯 **本章目标**：学会在 iOS 项目中集成主流第三方分析和监控 SDK，掌握数据埋点的方法论，了解隐私合规要求下的 SDK 使用策略。

---

## 1. 为什么需要第三方 SDK

### 1.1 三大核心需求

iOS 开发中，第三方 SDK 主要解决三大类需求：

| 需求类别 | 说明 | 代表 SDK |
|----------|------|----------|
| 数据分析 | 了解用户行为、留存、转化 | Firebase Analytics、Umeng、Mixpanel |
| 崩溃监控 | 实时捕获和上报崩溃 | Firebase Crashlytics、Sentry、Bugly |
| 性能监控 | 监控 App 启动时间、网络请求、卡顿 | Firebase Performance、Sentry Performance |

> 💡 生活类比：第三方 SDK 就像商场的"监控系统"——
> - **数据分析**：统计每天有多少人进来、哪些区域最受欢迎、顾客停留多久
> - **崩溃监控**：发现哪里出了事故，第一时间报警
> - **性能监控**：监测电梯速度、空调温度，确保体验舒适

### 1.2 自建 vs 第三方

| 维度 | 自建 | 第三方 SDK |
|------|------|------------|
| 开发成本 | 高（需要开发+维护） | 低（集成即可） |
| 数据控制 | 完全自主 | 数据在第三方平台 |
| 功能完整度 | 需要逐步建设 | 开箱即用 |
| 隐私风险 | 自主可控 | 需要评估合规性 |
| 维护成本 | 持续投入 | SDK 更新由第三方负责 |
| 数据量级 | 适合小规模 | 适合大规模 |
| 适合团队 | 有数据团队的大公司 | 中小团队 |

### 1.3 国内 vs 海外 SDK 选择

| 市场 | 分析 SDK | 崩溃监控 | 说明 |
|------|----------|----------|------|
| 国内 | Umeng、TalkingData | Bugly、腾讯Bugly | 国内服务器，合规友好 |
| 海外 | Firebase Analytics、Mixpanel | Crashlytics、Sentry | 全球覆盖，功能强大 |
| 双市场 | Firebase + Umeng | Crashlytics + Bugly | 分别集成，数据互通 |

⚠️ **警告**：如果你的 App 面向中国大陆用户，数据必须存储在中国境内服务器，否则违反《个人信息保护法》和《数据安全法》。

---

## 2. Firebase Analytics 集成

### 2.1 Firebase 概述

Firebase 是 Google 提供的移动开发平台，Analytics 是其核心组件之一。

**Firebase Analytics 特点：**

| 特点 | 说明 |
|------|------|
| 免费 | 无限的事件上报量 |
| 自动事件 | 自动收集常见用户行为 |
| 用户细分 | 按属性、行为划分用户群 |
| 与其他 Firebase 集成 | 与 Crashlytics、Remote Config 等无缝集成 |
| BigQuery 导出 | 原始数据导出到 BigQuery 进行深度分析 |
| A/B 测试 | 与 Firebase A/B Testing 集成 |

### 2.2 Firebase 项目创建

**步骤：**

1. 访问 [Firebase Console](https://console.firebase.google.com)
2. 点击 "添加项目"
3. 输入项目名称
4. 选择是否启用 Google Analytics
5. 选择 Analytics 账号
6. 创建项目

**注册 iOS App：**

1. 在项目概览页点击 iOS 图标
2. 输入 Bundle ID（必须与 Xcode 项目一致）
3. 输入 App 昵称和 App Store ID
4. 下载 `GoogleService-Info.plist`

### 2.3 SPM 安装 Firebase

**在 Xcode 中添加 Firebase SDK：**

1. File → Add Package Dependencies
2. 输入 `https://github.com/firebase/firebase-ios-sdk`
3. 选择版本规则（Up to Next Major Version）
4. 选择需要的 Product：

| Product | 说明 | 必选 |
|---------|------|------|
| FirebaseAnalytics | 核心分析 | ✅ |
| FirebaseCrashlytics | 崩溃监控 | 按需 |
| FirebasePerformance | 性能监控 | 按需 |
| FirebaseRemoteConfig | 远程配置 | 按需 |
| FirebaseMessaging | 推送通知 | 按需 |
| FirebaseAuth | 身份认证 | 按需 |
| FirebaseFirestore | 云数据库 | 按需 |

### 2.4 CocoaPods 安装 Firebase

```ruby
# Podfile
pod 'FirebaseAnalytics'
pod 'FirebaseCrashlytics'
pod 'FirebasePerformance'
pod 'FirebaseMessaging'
pod 'FirebaseRemoteConfig'
```

```bash
pod install
```

### 2.5 Firebase 初始化

**添加 GoogleService-Info.plist：**

1. 将下载的 `GoogleService-Info.plist` 拖入 Xcode 项目
2. 确保添加到所有 Target
3. 确认 Target Membership 已勾选

**在 AppDelegate 中初始化：**

```swift
import UIKit
import FirebaseCore

@main
class AppDelegate: UIResponder, UIApplicationDelegate {

    func application(
        _ application: UIApplication,
        didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
    ) -> Bool {
        FirebaseApp.configure()
        return true
    }
}
```

**SwiftUI App 初始化：**

```swift
import SwiftUI
import FirebaseCore

@main
struct MyApp: App {
    init() {
        FirebaseApp.configure()
    }

    var body: some Scene {
        WindowGroup {
            ContentView()
        }
    }
}
```

### 2.6 事件追踪

Firebase Analytics 提供两种事件：**自动收集事件**和**自定义事件**。

**自动收集的事件：**

| 事件名 | 触发时机 |
|--------|----------|
| first_open | 用户首次打开 App |
| app_update | App 更新后首次打开 |
| session_start | 新会话开始 |
| user_engagement | App 在前台活跃 |
| os_update | 设备系统更新 |
| screen_view | 页面切换（需配置） |

**自定义事件：**

```swift
import FirebaseAnalytics

Analytics.logEvent("purchase_completed", parameters: [
    "item_id": "sku_12345",
    "item_name": "Premium Plan",
    "item_category": "subscription",
    "price": 68.0,
    "currency": "CNY"
])
```

**推荐的事件命名规范：**

| 规则 | 说明 | 示例 |
|------|------|------|
| 使用下划线 | 分隔单词 | `purchase_completed` |
| 全小写 | 避免大写字母 | ✅ `button_clicked` ❌ `buttonClicked` |
| 字母数字下划线 | 不使用特殊字符 | ✅ `level_3_completed` ❌ `level-3-completed` |
| 最多 40 字符 | 事件名长度限制 | - |
| 最多 25 个参数 | 单个事件参数限制 | - |

### 2.7 用户属性

用户属性用于描述用户特征，可以在分析时按属性筛选。

```swift
import FirebaseAnalytics

Analytics.setUserProperty("premium", forName: "user_type")
Analytics.setUserProperty("beijing", forName: "city")
Analytics.setUserProperty("18-24", forName: "age_group")
```

**预定义用户属性：**

| 属性名 | 说明 |
|--------|------|
| sign_up_method | 注册方式 |
| user_type | 用户类型（免费/付费） |

**自定义用户属性限制：**

| 限制 | 值 |
|------|-----|
| 最多属性数 | 25 个 |
| 属性名最长 | 24 字符 |
| 属性值最长 | 36 字符 |

### 2.8 用户 ID 设置

设置用户 ID 可以跨设备追踪用户行为：

```swift
Analytics.setUserID("user_12345")
```

⚠️ **警告**：用户 ID 不能包含个人身份信息（PII），如邮箱、手机号。应使用匿名化的 ID。

### 2.9 Screen 追踪

**UIKit 中自动追踪：**

```swift
import FirebaseAnalytics

class BaseViewController: UIViewController {
    override func viewDidAppear(_ animated: Bool) {
        super.viewDidAppear(animated)
        Analytics.logEvent(
            AnalyticsEventScreenView,
            parameters: [
                AnalyticsParameterScreenName: screenName,
                AnalyticsParameterScreenClass: String(describing: type(of: self))
            ]
        )
    }

    var screenName: String {
        return String(describing: type(of: self))
    }
}
```

**SwiftUI 中追踪：**

```swift
import SwiftUI
import FirebaseAnalytics

struct ContentView: View {
    var body: some View {
        TabView {
            HomeView()
                .onAppear {
                    Analytics.logEvent(
                        AnalyticsEventScreenView,
                        parameters: [
                            AnalyticsParameterScreenName: "home",
                            AnalyticsParameterScreenClass: "HomeView"
                        ]
                    )
                }

            ProfileView()
                .onAppear {
                    Analytics.logEvent(
                        AnalyticsEventScreenView,
                        parameters: [
                            AnalyticsParameterScreenName: "profile",
                            AnalyticsParameterScreenClass: "ProfileView"
                        ]
                    )
                }
        }
    }
}
```

### 2.10 电商事件示例

```swift
enum AnalyticsEvent {
    static func viewItem(id: String, name: String, category: String, price: Double) {
        Analytics.logEvent(AnalyticsEventViewItem, parameters: [
            AnalyticsParameterItemID: id,
            AnalyticsParameterItemName: name,
            AnalyticsParameterItemCategory: category,
            AnalyticsParameterPrice: price,
            AnalyticsParameterCurrency: "CNY"
        ])
    }

    static func addToCart(id: String, name: String, quantity: Int, price: Double) {
        Analytics.logEvent(AnalyticsEventAddToCart, parameters: [
            AnalyticsParameterItemID: id,
            AnalyticsParameterItemName: name,
            AnalyticsParameterQuantity: quantity,
            AnalyticsParameterPrice: price,
            AnalyticsParameterCurrency: "CNY"
        ])
    }

    static func beginCheckout(total: Double, items: [[String: Any]]) {
        Analytics.logEvent(AnalyticsEventBeginCheckout, parameters: [
            AnalyticsParameterValue: total,
            AnalyticsParameterCurrency: "CNY",
            AnalyticsParameterItems: items
        ])
    }

    static func purchase(orderId: String, total: Double, tax: Double, shipping: Double) {
        Analytics.logEvent(AnalyticsEventPurchase, parameters: [
            AnalyticsParameterTransactionID: orderId,
            AnalyticsParameterValue: total,
            AnalyticsParameterCurrency: "CNY",
            AnalyticsParameterTax: tax,
            AnalyticsParameterShipping: shipping
        ])
    }
}
```

---

## 3. Firebase Crashlytics 集成

### 3.1 Crashlytics 概述

Firebase Crashlytics 是业界最流行的 iOS 崩溃监控 SDK，提供实时崩溃报告。

**核心功能：**

| 功能 | 说明 |
|------|------|
| 实时崩溃报告 | 崩溃发生后秒级上报 |
| 崩溃堆栈 | 符号化的崩溃调用栈 |
| 非致命异常 | 捕获 Error 级别的问题 |
| 自定义日志 | 崩溃前记录上下文日志 |
| 用户标识 | 标记受影响的用户 |
| 趋势分析 | 崩溃率变化趋势 |
| 无需 dSYM | 自动管理符号文件（推荐上传） |

### 3.2 Crashlytics 安装

**SPM 方式：**

在 Xcode 的 Package Dependencies 中添加 `FirebaseCrashlytics`。

**CocoaPods 方式：**

```ruby
pod 'FirebaseCrashlytics'
```

### 3.3 Crashlytics 初始化和配置

**在 AppDelegate 中启用：**

```swift
import UIKit
import FirebaseCore
import FirebaseCrashlytics

@main
class AppDelegate: UIResponder, UIApplicationDelegate {

    func application(
        _ application: UIApplication,
        didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
    ) -> Bool {
        FirebaseApp.configure()

        Crashlytics.crashlytics().setCrashlyticsCollectionEnabled(true)

        return true
    }
}
```

**Build Phase 添加上传脚本：**

在 Xcode 的 Build Phases 中添加 Run Script：

```bash
"${BUILD_DIR%/Build/*}/SourcePackages/checkouts/firebase-ios-sdk/Crashlytics/run"
```

**SPM 路径可能不同，使用以下方式查找：**

```bash
# 在 Build Phase 的 Run Script 中
if [ "${CONFIGURATION}" = "Debug" ]; then
    echo "Skip Crashlytics upload in Debug"
else
    "${BUILD_DIR%/Build/*}/SourcePackages/checkouts/firebase-ios-sdk/Crashlytics/run"
fi
```

### 3.4 dSYM 上传

dSYM 文件用于符号化崩溃堆栈，将内存地址映射为可读的函数名。

**自动上传（推荐）：**

Crashlytics 的 Run Script 会在构建时自动上传 dSYM。

**手动上传：**

```bash
# 使用 Firebase CLI
firebase crashlytics:symbols:upload --app=YOUR_APP_ID path/to/dSYMs

# 使用 upload-symbols 脚本
"${PODS_ROOT}/FirebaseCrashlytics/upload-symbols" \
    -gsp "${PROJECT_DIR}/GoogleService-Info.plist" \
    -p ios \
    "${DWARF_DSYM_FOLDER_PATH}/${DWARF_DSYM_FILE_NAME}"
```

**在 Xcode Build Settings 中确保：**

| 设置 | 值 |
|------|-----|
| Debug Information Format | DWARF with dSYM File |
| Strip Debug Symbols During Copy | NO（Release 也建议 NO） |

### 3.5 自定义日志和键值

```swift
import FirebaseCrashlytics

Crashlytics.crashlytics().log("User started checkout flow")
Crashlytics.crashlytics().log("Selected payment method: Alipay")
Crashlytics.crashlytics().log("Order total: ¥68.00")

Crashlytics.crashlytics().setCustomValue("premium", forKey: "user_type")
Crashlytics.crashlytics().setCustomValue(3, forKey: "items_in_cart")
Crashlytics.crashlytics().setCustomValue(true, forKey: "has_subscription")
```

### 3.6 用户标识

```swift
Crashlytics.crashlytics().setUserID("user_12345")
```

⚠️ **警告**：用户 ID 不应包含 PII（个人身份信息）。使用匿名化的用户标识。

### 3.7 非致命异常上报

```swift
import FirebaseCrashlytics

do {
    try performRiskyOperation()
} catch {
    Crashlytics.crashlytics().record(error: error)
}

struct PaymentError: Error {
    let code: String
    let message: String
}

let paymentError = PaymentError(
    code: "PAYMENT_FAILED",
    message: "Alipay payment gateway timeout"
)
Crashlytics.crashlytics().record(error: paymentError)
```

### 3.8 Crashlytics 与 Analytics 的集成

Crashlytics 和 Analytics 共享用户 ID 和事件数据：

```swift
// 设置用户 ID（两边同步）
Analytics.setUserID("user_12345")
Crashlytics.crashlytics().setUserID("user_12345")

// Analytics 事件会自动关联到 Crashlytics 崩溃报告
// 在 Firebase Console 中可以看到崩溃前的用户行为路径
```

---

## 4. Sentry 集成

### 4.1 Sentry 概述

Sentry 是一个开源的错误监控和性能追踪平台，支持多语言多平台。

**Sentry vs Crashlytics 对比：**

| 特性 | Sentry | Crashlytics |
|------|--------|-------------|
| 开源 | ✅ 可自建 | ❌ 仅云服务 |
| 错误监控 | ✅ 崩溃 + 非致命错误 | ✅ 崩溃 + 非致命错误 |
| 性能监控 | ✅ 内置 APM | ❌ 需 Firebase Performance |
| Release 追踪 | ✅ 按版本追踪 | ✅ 按版本追踪 |
| 自定义标签 | ✅ 丰富 | ✅ 基本 |
| 面包屑 | ✅ 详细的上下文 | ✅ 自定义日志 |
| 告警规则 | ✅ 灵活配置 | ✅ 基本 |
| 集成生态 | ✅ 200+ 集成 | ✅ Firebase 生态 |
| 免费额度 | 5K 错误/月 | 无限 |
| 数据存储 | 可自建 | Google 服务器 |

### 4.2 Sentry 安装

**SPM 方式：**

1. File → Add Package Dependencies
2. 输入 `https://github.com/getsentry/sentry-cocoa`
3. 选择版本

**CocoaPods 方式：**

```ruby
pod 'Sentry', :git => 'https://github.com/getsentry/sentry-cocoa.git', :tag => '8.x.x'
```

### 4.3 Sentry 初始化

```swift
import UIKit
import Sentry

@main
class AppDelegate: UIResponder, UIApplicationDelegate {

    func application(
        _ application: UIApplication,
        didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
    ) -> Bool {
        SentrySDK.start { options in
            options.dsn = "https://examplePublicKey@o0.ingest.sentry.io/0"
            options.debug = false
            options.environment = "production"
            options.releaseName = "com.example.myapp@1.0.0"
            options.tracesSampleRate = 0.2
            options.profilesSampleRate = 0.2
            options.enableAutoSessionTracking = true
            options.sessionTrackingIntervalMillis = 30000
        }

        return true
    }
}
```

**配置说明：**

| 配置项 | 说明 | 推荐值 |
|--------|------|--------|
| dsn | 项目数据源地址 | 必填 |
| debug | 调试模式 | Debug: true, Release: false |
| environment | 环境标识 | development / staging / production |
| releaseName | 版本标识 | BundleID@Version |
| tracesSampleRate | 性能追踪采样率 | 0.1-0.5（生产环境） |
| profilesSampleRate | 性能分析采样率 | 0.1-0.2 |
| enableAutoSessionTracking | 自动会话追踪 | true |

### 4.4 错误捕获

```swift
import Sentry

// 捕获 Error
do {
    try fetchUserData()
} catch {
    SentrySDK.capture(error: error)
}

// 捕获自定义消息
SentrySDK.capture(message: "Payment gateway returned unexpected response")

// 捕获带上下文的错误
let event = Event(level: .error)
event.message = SentryMessage(formatted: "API request failed")
event.tags = ["endpoint": "/api/v1/users", "method": "GET"]
event.extra = ["status_code": 500, "response_time": 3.2]
event.fingerprint = ["api-error", "/api/v1/users"]
SentrySDK.capture(event: event)
```

### 4.5 面包屑（Breadcrumbs）

面包屑记录崩溃前的用户操作路径：

```swift
import Sentry

SentrySDK.addBreadcrumb(crumb: Breadcrumb(
    level: .info,
    category: "navigation",
    message: "User opened product detail",
    data: ["product_id": "sku_12345", "source": "search"]
))

SentrySDK.addBreadcrumb(crumb: Breadcrumb(
    level: .info,
    category: "action",
    message: "User tapped buy button",
    data: ["product_id": "sku_12345", "price": 68.0]
))

SentrySDK.addBreadcrumb(crumb: Breadcrumb(
    level: .info,
    category: "network",
    message: "API request started",
    data: ["url": "/api/v1/orders", "method": "POST"]
))
```

### 4.6 用户信息

```swift
import Sentry

let user = User(userId: "user_12345")
user.email = nil
user.username = "zhang_san"
user.data = [
    "subscription_type": "premium",
    "registration_date": "2024-01-15",
    "city": "beijing"
]
SentrySDK.setUser(user)
```

### 4.7 性能监控

```swift
import Sentry

// 自动追踪 UIViewController 加载时间（默认开启）
// 自动追踪网络请求（需配置）

// 手动创建 Transaction
let transaction = SentrySDK.startTransaction(
    name: "checkout_flow",
    operation: "ui.action"
)

let addToCartSpan = transaction.startChild(
    operation: "db.query",
    description: "Add item to cart"
)
addToCartSpan.setData(value: "sku_12345", key: "item_id")
addToCartSpan.finish()

let paymentSpan = transaction.startChild(
    operation: "http.request",
    description: "Process payment"
)
paymentSpan.setData(value: "alipay", key: "payment_method")
paymentSpan.finish()

transaction.finish()
```

### 4.8 Release 追踪

```swift
// 在 Sentry 初始化时设置 releaseName
SentrySDK.start { options in
    options.releaseName = "com.example.myapp@\(appVersion)"
}

// 在 CI 中设置
// SENTRY_RELEASE=com.example.myapp@1.0.0
```

**在 Sentry Dashboard 中：**

- 按版本查看崩溃率
- 对比不同版本的崩溃情况
- 发现新版本引入的回归问题
- 设置发布健康度指标

---

## 5. 数据埋点方法论

### 5.1 埋点设计原则

> 💡 核心原则：**先想清楚要回答什么问题，再决定埋什么点。**

**埋点设计四步法：**

```
Step 1: 明确业务问题
    → "用户为什么在注册流程中流失？"

Step 2: 定义关键指标
    → 注册转化率、每一步的完成率、停留时间

Step 3: 设计事件和属性
    → registration_step_viewed、registration_step_completed、registration_failed

Step 4: 实施和验证
    → 埋点代码、测试验证、数据校验
```

### 5.2 事件设计

**事件分类：**

| 类型 | 说明 | 示例 |
|------|------|------|
| 曝光事件 | 元素展示给用户 | `page_viewed`、`item_impressed` |
| 点击事件 | 用户交互操作 | `button_clicked`、`cell_tapped` |
| 业务事件 | 核心业务动作 | `purchase_completed`、`registration_done` |
| 系统事件 | 系统级动作 | `app_crashed`、`api_timeout` |

**事件命名规范：**

```
格式：[对象]_[动作]

示例：
page_viewed          → 页面被查看
button_clicked       → 按钮被点击
item_added_to_cart   → 商品被加入购物车
order_submitted      → 订单被提交
payment_completed    → 支付完成
```

### 5.3 属性设计

属性为事件提供上下文信息，让数据更丰富、更有分析价值。

**属性分类：**

| 类型 | 说明 | 示例 |
|------|------|------|
| 对象属性 | 事件作用的对象 | item_id、page_name |
| 来源属性 | 事件的触发来源 | source、entry_point |
| 结果属性 | 事件的结果 | result、error_code |
| 环境属性 | 事件发生的环境 | app_version、network_type |
| 用户属性 | 事件关联的用户特征 | user_type、vip_level |

**属性设计原则：**

| 原则 | 说明 | 示例 |
|------|------|------|
| 必要性 | 只记录有分析价值的属性 | ✅ item_id ❌ internal_flag |
| 可枚举 | 尽量使用可枚举的值 | ✅ "alipay" ❌ "用户选择了支付宝" |
| 一致性 | 同一概念使用同一属性名 | ✅ 统一用 item_id ❌ 混用 product_id |
| 简洁性 | 属性值简短明确 | ✅ "premium" ❌ "Premium Subscription Plan" |

### 5.4 用户画像

用户画像是通过事件和属性构建的用户特征标签。

```swift
struct UserProfile {
    var userType: String
    var vipLevel: Int
    var registrationDate: String
    var totalPurchases: Int
    var totalSpent: Double
    var preferredCategory: String
    var lastActiveDate: String
    var deviceModel: String
    var city: String
}

extension UserProfile {
    func applyToAnalytics() {
        Analytics.setUserProperty(userType, forName: "user_type")
        Analytics.setUserProperty("\(vipLevel)", forName: "vip_level")
        Analytics.setUserProperty(preferredCategory, forName: "preferred_category")
        Analytics.setUserProperty(city, forName: "city")
    }
}
```

### 5.5 埋点文档模板

每个事件都应有文档记录：

```markdown
### 事件：purchase_completed

**描述**：用户完成一次购买

**触发时机**：支付成功回调后

**属性**：

| 属性名 | 类型 | 必填 | 说明 | 示例 |
|--------|------|------|------|------|
| order_id | String | ✅ | 订单号 | "ORD_20240115_001" |
| item_id | String | ✅ | 商品 ID | "sku_12345" |
| item_name | String | ✅ | 商品名称 | "Premium Plan" |
| price | Double | ✅ | 价格 | 68.0 |
| currency | String | ✅ | 币种 | "CNY" |
| payment_method | String | ✅ | 支付方式 | "alipay" |
| is_first_purchase | Bool | ❌ | 是否首购 | true |
| discount_amount | Double | ❌ | 优惠金额 | 10.0 |

**关联事件**：begin_checkout → payment_started → purchase_completed
```

---

## 6. 埋点代码的组织和封装

### 6.1 埋点管理器设计

```swift
import FirebaseAnalytics
import FirebaseCrashlytics

final class AnalyticsManager {
    static let shared = AnalyticsManager()

    private init() {}

    func initialize() {
        FirebaseApp.configure()
        Crashlytics.crashlytics().setCrashlyticsCollectionEnabled(true)
    }

    func logEvent(_ event: AnalyticsEvent) {
        Analytics.logEvent(event.name, parameters: event.parameters)
    }

    func setUserProperty(_ value: String?, for name: String) {
        Analytics.setUserProperty(value, forName: name)
    }

    func setUserID(_ id: String?) {
        Analytics.setUserID(id)
        Crashlytics.crashlytics().setUserID(id)
    }
}
```

### 6.2 事件枚举定义

```swift
protocol AnalyticsEventProtocol {
    var name: String { get }
    var parameters: [String: Any]? { get }
}

enum AppEvent: String, AnalyticsEventProtocol {
    case pageViewed = "page_viewed"
    case buttonClicked = "button_clicked"
    case itemViewed = "item_viewed"
    case itemAddedToCart = "item_added_to_cart"
    case purchaseCompleted = "purchase_completed"
    case searchPerformed = "search_performed"
    case shareClicked = "share_clicked"
    case loginCompleted = "login_completed"
    case registrationCompleted = "registration_completed"

    var name: String { rawValue }

    var parameters: [String: Any]? { nil }
}

enum PageName: String {
    case home = "home"
    case productDetail = "product_detail"
    case cart = "cart"
    case checkout = "checkout"
    case profile = "profile"
    case settings = "settings"
}

enum ButtonName: String {
    case addToCart = "add_to_cart"
    case buyNow = "buy_now"
    case share = "share"
    case login = "login"
    case register = "register"
}
```

### 6.3 带参数的事件

```swift
struct PageViewedEvent: AnalyticsEventProtocol {
    let pageName: String
    let source: String?

    var name: String { "page_viewed" }

    var parameters: [String: Any]? {
        var params: [String: Any] = ["page_name": pageName]
        if let source = source {
            params["source"] = source
        }
        return params
    }
}

struct PurchaseCompletedEvent: AnalyticsEventProtocol {
    let orderId: String
    let itemId: String
    let itemName: String
    let price: Double
    let currency: String
    let paymentMethod: String

    var name: String { "purchase_completed" }

    var parameters: [String: Any]? {
        return [
            "order_id": orderId,
            "item_id": itemId,
            "item_name": itemName,
            "price": price,
            "currency": currency,
            "payment_method": paymentMethod
        ]
    }
}
```

### 6.4 使用示例

```swift
AnalyticsManager.shared.logEvent(
    PageViewedEvent(pageName: PageName.productDetail.rawValue, source: "search")
)

AnalyticsManager.shared.logEvent(
    PurchaseCompletedEvent(
        orderId: "ORD_001",
        itemId: "sku_12345",
        itemName: "Premium Plan",
        price: 68.0,
        currency: "CNY",
        paymentMethod: "alipay"
    )
)
```

### 6.5 埋点与业务代码解耦

使用协议和扩展将埋点与业务逻辑分离：

```swift
protocol Trackable {
    var trackEvent: AnalyticsEventProtocol { get }
}

extension UIViewController {
    func trackPageView() {
        if let trackable = self as? Trackable {
            AnalyticsManager.shared.logEvent(trackable.trackEvent)
        }
    }
}

class ProductDetailViewController: UIViewController, Trackable {
    var trackEvent: AnalyticsEventProtocol {
        PageViewedEvent(pageName: PageName.productDetail.rawValue, source: nil)
    }

    override func viewDidAppear(_ animated: Bool) {
        super.viewDidAppear(animated)
        trackPageView()
    }
}
```

### 6.6 埋点调试

```swift
#if DEBUG
extension AnalyticsManager {
    func logEvent(_ event: AnalyticsEventProtocol) {
        print("📊 Analytics Event: \(event.name)")
        if let params = event.parameters {
            print("   Parameters: \(params)")
        }
        Analytics.logEvent(event.name, parameters: event.parameters)
    }
}
#endif
```

**Firebase DebugView：**

```bash
# 启用 Analytics Debug 模式
# 在 Xcode 的 Scheme 中添加启动参数
-FIRAnalyticsDebugEnabled
```

在 Firebase Console → DebugView 中可以实时查看事件流。

---

## 7. SDK 对包大小的影响

### 7.1 各 SDK 包大小参考

| SDK | 增量大小（ARM64） | 说明 |
|-----|-------------------|------|
| Firebase Analytics | ~2.5 MB | 包含 Analytics Core |
| Firebase Crashlytics | ~1.5 MB | 崩溃监控 |
| Firebase Performance | ~1.0 MB | 性能监控 |
| Sentry | ~3.0 MB | 错误+性能监控 |
| Umeng Analytics | ~2.0 MB | 国内分析 |
| Bugly | ~1.0 MB | 崩溃监控 |
| Mixpanel | ~1.5 MB | 分析 |

### 7.2 减小包大小影响

| 策略 | 说明 |
|------|------|
| 按需集成 | 只集成需要的模块，不集成全家桶 |
| SPM 精细选择 | SPM 可以选择具体的 Product |
| App Thinning | 利用 App Thinning 自动裁剪不需要的架构 |
| 延迟加载 | 非核心 SDK 延迟到首次使用时初始化 |
| 动态库 | 使用动态库而非静态库（权衡启动时间） |

### 7.3 分析包大小

```bash
# 查看 App 大小
# 在 Xcode Organizer 中查看 App Store 大小

# 使用命令行分析
xcodebuild -project MyApp.xcodeproj \
    -scheme MyApp \
    -configuration Release \
    -archivePath build/MyApp.xcarchive \
    archive

# 查看 IPA 内容
unzip -l MyApp.ipa

# 查看各框架大小
find build/MyApp.xcarchive -name "*.framework" -exec du -sh {} \; | sort -rh
```

---

## 8. 隐私合规

### 8.1 国内隐私法规

| 法规 | 要点 |
|------|------|
| 《个人信息保护法》 | 明确同意、最小必要、目的限制 |
| 《数据安全法》 | 数据分类分级、跨境传输限制 |
| 《网络安全法》 | 数据存储在中国境内 |
| App 违法违规收集使用个人信息专项治理 | 不得强制索权、不得超范围收集 |

### 8.2 用户同意管理

```swift
final class ConsentManager {
    static let shared = ConsentManager()

    enum ConsentType: String {
        case analytics = "analytics"
        case crashReporting = "crash_reporting"
        case performanceMonitoring = "performance_monitoring"
        case personalizedAds = "personalized_ads"
    }

    private var consents: [ConsentType: Bool] = [:]

    var hasAnalyticsConsent: Bool {
        consents[.analytics] ?? false
    }

    var hasCrashReportingConsent: Bool {
        consents[.crashReporting] ?? false
    }

    func updateConsent(_ type: ConsentType, granted: Bool) {
        consents[type] = granted

        switch type {
        case .analytics:
            Analytics.setAnalyticsCollectionEnabled(granted)
        case .crashReporting:
            Crashlytics.crashlytics().setCrashlyticsCollectionEnabled(granted)
        case .performanceMonitoring:
            Performance.shared.isPerformanceCollectionEnabled = granted
        case .personalizedAds:
            break
        }
    }

    func loadSavedConsents() {
        let defaults = UserDefaults.standard
        for type in ConsentType.allCases {
            let key = "consent_\(type.rawValue)"
            if defaults.object(forKey: key) != nil {
                updateConsent(type, granted: defaults.bool(forKey: key))
            }
        }
    }

    func saveConsents() {
        let defaults = UserDefaults.standard
        for (type, granted) in consents {
            defaults.set(granted, forKey: "consent_\(type.rawValue)")
        }
    }
}
```

### 8.3 数据最小化

**原则：只收集实现功能所必需的数据。**

| 原则 | 说明 | 示例 |
|------|------|------|
| 目的限制 | 只为明确目的收集数据 | ✅ 分析用户留存 ❌ 收集通讯录"备用" |
| 数据最小化 | 只收集必要的数据 | ✅ 匿名用户 ID ❌ 收集邮箱和手机号 |
| 存储限制 | 不保留超过必要时间的数据 | ✅ 保留 90 天 ❌ 永久保留 |
| 访问限制 | 限制数据访问权限 | ✅ 仅数据团队可访问 ❌ 全公司可访问 |

### 8.4 隐私清单声明

Apple 从 2024 年 4 月起要求所有 SDK 声明隐私清单（PrivacyInfo.xcprivacy）。

**创建 PrivacyInfo.xcprivacy：**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>NSPrivacyTracking</key>
    <false/>
    <key>NSPrivacyTrackingDomains</key>
    <array/>
    <key>NSPrivacyCollectedDataTypes</key>
    <array>
        <dict>
            <key>NSPrivacyCollectedDataType</key>
            <string>NSPrivacyCollectedDataTypeProductInteraction</string>
            <key>NSPrivacyCollectedDataTypeLinked</key>
            <false/>
            <key>NSPrivacyCollectedDataTypeTracking</key>
            <false/>
            <key>NSPrivacyCollectedDataTypePurposes</key>
            <array>
                <string>NSPrivacyCollectedDataTypePurposeAnalytics</string>
            </array>
        </dict>
        <dict>
            <key>NSPrivacyCollectedDataType</key>
            <string>NSPrivacyCollectedDataTypeCrashData</string>
            <key>NSPrivacyCollectedDataTypeLinked</key>
            <false/>
            <key>NSPrivacyCollectedDataTypeTracking</key>
            <false/>
            <key>NSPrivacyCollectedDataTypePurposes</key>
            <array>
                <string>NSPrivacyCollectedDataTypePurposeAppFunctionality</string>
            </array>
        </dict>
    </array>
    <key>NSPrivacyAccessedAPITypes</key>
    <array>
        <dict>
            <key>NSPrivacyAccessedAPIType</key>
            <string>NSPrivacyAccessedAPICategoryFileTimestamp</string>
            <key>NSPrivacyAccessedAPITypeReasons</key>
            <array>
                <string>C617.1</string>
            </array>
        </dict>
        <dict>
            <key>NSPrivacyAccessedAPIType</key>
            <string>NSPrivacyAccessedAPICategoryUserDefaults</string>
            <key>NSPrivacyAccessedAPITypeReasons</key>
            <array>
                <string>CA92.1</string>
            </array>
        </dict>
    </array>
</dict>
</plist>
```

**常用数据类型声明：**

| 数据类型 | 说明 | Firebase Analytics | Crashlytics | Sentry |
|----------|------|-------------------|-------------|--------|
| ProductInteraction | 产品交互数据 | ✅ | - | - |
| CrashData | 崩溃数据 | - | ✅ | ✅ |
| PerformanceData | 性能数据 | - | - | ✅ |
| DeviceID | 设备标识 | ✅ | ✅ | ✅ |
| LocationData | 位置数据 | ❌ | ❌ | ❌ |

### 8.5 App Tracking Transparency (ATT)

如果 SDK 需要追踪用户跨 App 行为，必须先获取 ATT 授权：

```swift
import AppTrackingTransparency
import AdSupport

func requestTrackingAuthorization() {
    ATTrackingManager.requestTrackingAuthorization { status in
        switch status {
        case .authorized:
            print("用户同意追踪")
        case .denied:
            print("用户拒绝追踪")
        case .notDetermined:
            print("用户未决定")
        case .restricted:
            print("追踪受限")
        @unknown default:
            break
        }
    }
}
```

⚠️ **警告**：Firebase Analytics 和 Crashlytics **不需要** ATT 授权，因为它们不进行跨 App 追踪。但如果你的 App 同时集成了广告 SDK（如 Google Ads），则需要 ATT 授权。

### 8.6 App Store 隐私标签

在 App Store Connect 中，需要声明你的 App 收集了哪些数据：

| 数据类型 | Firebase Analytics | Crashlytics | Sentry |
|----------|-------------------|-------------|--------|
| 标识符 | ✅ 设备 ID | ✅ 设备 ID | ✅ 设备 ID |
| 使用数据 | ✅ 分析数据 | - | ✅ 性能数据 |
| 诊断数据 | - | ✅ 崩溃数据 | ✅ 错误数据 |
| 位置 | ❌ | ❌ | ❌ |
| 联系方式 | ❌ | ❌ | ❌ |

---

## 9. 多 SDK 的初始化策略

### 9.1 初始化顺序

推荐的 SDK 初始化顺序：

```
1. Firebase App（基础配置）
2. Crashlytics（尽早初始化，捕获启动崩溃）
3. Analytics（用户行为追踪）
4. Performance（性能监控）
5. 其他业务 SDK
```

### 9.2 延迟初始化

非关键 SDK 可以延迟初始化，减少启动时间：

```swift
final class SDKInitializer {
    static let shared = SDKInitializer()

    private let initQueue = DispatchQueue(label: "com.app.sdkinit", qos: .utility)

    func initializeCriticalSDKs() {
        FirebaseApp.configure()
        Crashlytics.crashlytics().setCrashlyticsCollectionEnabled(true)
    }

    func initializeNonCriticalSDKs() {
        initQueue.async {
            Analytics.setAnalyticsCollectionEnabled(true)
            Performance.shared.isPerformanceCollectionEnabled = true
            self.initializeSentry()
            self.initializeOtherSDKs()
        }
    }

    private func initializeSentry() {
        SentrySDK.start { options in
            options.dsn = "https://examplePublicKey@o0.ingest.sentry.io/0"
            options.environment = AppEnvironment.current.rawValue
            options.releaseName = "com.example.myapp@\(AppVersion.current)"
            options.tracesSampleRate = 0.2
        }
    }

    private func initializeOtherSDKs() {
        // 其他 SDK 初始化
    }
}
```

**在 AppDelegate 中使用：**

```swift
func application(
    _ application: UIApplication,
    didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
) -> Bool {
    SDKInitializer.shared.initializeCriticalSDKs()

    DispatchQueue.main.asyncAfter(deadline: .now() + 1.0) {
        SDKInitializer.shared.initializeNonCriticalSDKs()
    }

    return true
}
```

### 9.3 SDK 初始化监控

```swift
final class SDKInitMonitor {
    static let shared = SDKInitMonitor()

    private var initTimes: [String: TimeInterval] = [:]
    private let startTime = CACurrentMediaTime()

    func measureInit(name: String, block: () -> Void) {
        let start = CACurrentMediaTime()
        block()
        let elapsed = CACurrentMediaTime() - start
        initTimes[name] = elapsed
        print("⏱ SDK [\(name)] 初始化耗时: \(String(format: "%.2f", elapsed * 1000))ms")
    }

    func report() {
        let totalElapsed = CACurrentMediaTime() - startTime
        print("⏱ SDK 总初始化耗时: \(String(format: "%.2f", totalElapsed * 1000))ms")
        for (name, time) in initTimes.sorted(by: { $0.value > $1.value }) {
            print("   \(name): \(String(format: "%.2f", time * 1000))ms")
        }
    }
}
```

**使用方式：**

```swift
SDKInitMonitor.shared.measureInit(name: "Firebase") {
    FirebaseApp.configure()
}

SDKInitMonitor.shared.measureInit(name: "Crashlytics") {
    Crashlytics.crashlytics().setCrashlyticsCollectionEnabled(true)
}

SDKInitMonitor.shared.measureInit(name: "Sentry") {
    SentrySDK.start { options in
        options.dsn = "https://examplePublicKey@o0.ingest.sentry.io/0"
    }
}

SDKInitMonitor.shared.report()
```

### 9.4 条件初始化

根据环境决定是否初始化 SDK：

```swift
enum AppEnvironment: String {
    case development
    case staging
    case production
}

struct SDKConfig {
    static let current = SDKConfig()

    var environment: AppEnvironment {
#if DEBUG
        return .development
#else
        return .production
#endif
    }

    var shouldInitAnalytics: Bool {
        environment != .development || UserDefaults.standard.bool(forKey: "enable_analytics_debug")
    }

    var shouldInitCrashlytics: Bool {
        environment != .development
    }

    var shouldInitPerformance: Bool {
        environment == .production
    }

    var analyticsSampleRate: Double {
        environment == .production ? 1.0 : 0.0
    }

    var sentryTracesSampleRate: Double {
        environment == .production ? 0.2 : 0.0
    }
}
```

---

## 10. 自建分析系统的考量

### 10.1 何时考虑自建

| 场景 | 建议 |
|------|------|
| 数据合规要求极高 | 考虑自建 |
| 需要完全控制数据 | 考虑自建 |
| 有专门的数据团队 | 可以自建 |
| 数据分析需求简单 | 使用第三方 |
| 团队规模小 | 使用第三方 |
| 需要快速上线 | 使用第三方 |

### 10.2 自建系统架构

```
客户端 → 数据采集 SDK → 数据网关 → 消息队列 → 数据处理 → 数据仓库 → 可视化
   │                        │
   │                        └── 数据校验、去重、格式化
   │
   └── 轻量 SDK，只负责采集和上报
```

**核心组件：**

| 组件 | 说明 | 技术选型 |
|------|------|----------|
| 数据采集 SDK | 嵌入 App 的 SDK | Swift |
| 数据网关 | 接收上报数据 | Nginx / Go |
| 消息队列 | 缓冲和分发数据 | Kafka / RabbitMQ |
| 数据处理 | 清洗和聚合 | Flink / Spark |
| 数据仓库 | 存储分析数据 | ClickHouse / BigQuery |
| 可视化 | 数据展示 | Grafana / 自建 Dashboard |

### 10.3 自建 SDK 设计

```swift
final class CustomAnalytics {
    static let shared = CustomAnalytics()

    private let queue = DispatchQueue(label: "com.app.analytics", qos: .utility)
    private var eventBuffer: [Event] = []
    private let maxBufferSize = 20
    private let flushInterval: TimeInterval = 30

    private init() {
        startPeriodicFlush()
    }

    struct Event: Codable {
        let name: String
        let parameters: [String: String]?
        let timestamp: Int64
        let sessionId: String
        let deviceId: String
        let appVersion: String
        let platform: String
    }

    func logEvent(name: String, parameters: [String: String]? = nil) {
        queue.async {
            let event = Event(
                name: name,
                parameters: parameters,
                timestamp: Int64(Date().timeIntervalSince1970 * 1000),
                sessionId: self.currentSessionId,
                deviceId: self.deviceId,
                appVersion: self.appVersion,
                platform: "iOS"
            )
            self.eventBuffer.append(event)

            if self.eventBuffer.count >= self.maxBufferSize {
                self.flush()
            }
        }
    }

    private func flush() {
        guard !eventBuffer.isEmpty else { return }

        let eventsToSend = eventBuffer
        eventBuffer.removeAll()

        sendEvents(eventsToSend) { [weak self] success in
            if !success {
                self?.queue.async {
                    self?.eventBuffer.insert(contentsOf: eventsToSend, at: 0)
                }
            }
        }
    }

    private func sendEvents(_ events: [Event], completion: @escaping (Bool) -> Void) {
        guard let url = URL(string: "https://analytics.example.com/v1/events"),
              let data = try? JSONEncoder().encode(events) else {
            completion(false)
            return
        }

        var request = URLRequest(url: url)
        request.httpMethod = "POST"
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")
        request.httpBody = data

        URLSession.shared.dataTask(with: request) { _, response, error in
            let success = error == nil && (response as? HTTPURLResponse)?.statusCode == 200
            completion(success)
        }.resume()
    }

    private func startPeriodicFlush() {
        DispatchQueue.global(qos: .utility).async {
            while true {
                Thread.sleep(forTimeInterval: self.flushInterval)
                self.queue.async {
                    self.flush()
                }
            }
        }
    }
}
```

### 10.4 自建 vs 第三方对比

| 维度 | 自建 | 第三方 |
|------|------|--------|
| 初始成本 | 高（开发+测试+部署） | 低（集成即可） |
| 维护成本 | 持续投入 | SDK 更新免费 |
| 数据控制 | 完全自主 | 数据在第三方 |
| 功能完整度 | 需逐步建设 | 开箱即用 |
| 合规性 | 自主可控 | 需评估 |
| 扩展性 | 自定义 | 受限于平台 |
| 团队要求 | 需要数据团队 | 无特殊要求 |

💡 **提示**：对于大多数中小团队，建议先用第三方 SDK 快速验证业务，等数据量和合规要求达到一定规模后再考虑自建。

---

## 11. 国内常用 SDK 补充

### 11.1 友盟（Umeng）

**特点：**

| 特点 | 说明 |
|------|------|
| 国内最流行 | 市场占有率最高 |
| 合规友好 | 数据存储在中国境内 |
| 功能全面 | 分析+推送+分享 |
| 免费版够用 | 基础分析功能免费 |

**集成方式：**

```ruby
# Podfile
pod 'UMCommon', '~> 7.0'
pod 'UMDevice', '~> 3.0'
pod 'UMPush', '~> 3.0'
```

```swift
import UMCommon

func application(
    _ application: UIApplication,
    didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
) -> Bool {
    UMConfigure.initWithAppkey("your_app_key", channel: "AppStore")
    UMConfigure.setLogEnabled(false)
    return true
}
```

### 11.2 腾讯 Bugly

**特点：**

| 特点 | 说明 |
|------|------|
| 崩溃监控 | 专注崩溃和异常 |
| 国内服务器 | 数据存储在中国 |
| 免费使用 | 基础功能免费 |
| 腾讯生态 | 与微信等集成 |

**集成方式：**

```ruby
# Podfile
pod 'Bugly', '~> 2.5'
```

```swift
import Bugly

func application(
    _ application: UIApplication,
    didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
) -> Bool {
    Bugly.start(withAppId: "your_app_id")
    return true
}
```

### 11.3 国内海外双 SDK 策略

如果 App 同时面向国内和海外市场：

```swift
final class DualAnalyticsManager {
    static let shared = DualAnalyticsManager()

    private let isChinaMainland: Bool = {
        let locale = Locale.current
        return locale.region?.identifier == "CN"
    }()

    func initialize() {
        if isChinaMainland {
            UMConfigure.initWithAppkey("umeng_key", channel: "AppStore")
        } else {
            FirebaseApp.configure()
        }
    }

    func logEvent(name: String, parameters: [String: Any]? = nil) {
        if isChinaMainland {
            MobClick.event(name, attributes: parameters)
        } else {
            Analytics.logEvent(name, parameters: parameters)
        }
    }
}
```

---

## 12. SDK 集成检查清单

### 12.1 集成前检查

| 检查项 | 说明 |
|--------|------|
| 隐私合规评估 | 评估 SDK 收集的数据是否符合法规 |
| 包大小影响 | 评估 SDK 对包大小的影响 |
| 性能影响 | 评估 SDK 对启动时间和运行时性能的影响 |
| 数据安全 | 评估 SDK 的数据传输和存储安全性 |
| 维护状态 | 检查 SDK 是否持续维护 |
| 社区活跃度 | 检查 GitHub Star、Issue 响应速度 |

### 12.2 集成后验证

| 检查项 | 说明 |
|--------|------|
| 事件上报 | 验证事件是否正确上报 |
| 崩溃捕获 | 验证崩溃是否正确捕获 |
| 数据准确性 | 验证上报数据与实际一致 |
| 隐私清单 | 检查 PrivacyInfo.xcprivacy 是否完整 |
| ATT 授权 | 检查是否正确处理 ATT |
| 启动时间 | 测量 SDK 对启动时间的影响 |
| 内存占用 | 测量 SDK 对内存的影响 |

### 12.3 上线前检查

| 检查项 | 说明 |
|--------|------|
| Debug 模式关闭 | 确保 SDK 不在 Debug 模式运行 |
| 日志关闭 | 关闭 SDK 调试日志 |
| 采样率设置 | 设置合理的采样率 |
| 环境标识 | 设置正确的 environment |
| 版本标识 | 设置正确的 releaseName |
| 隐私政策更新 | 更新隐私政策中的 SDK 说明 |
| App Store 隐私标签 | 在 App Store Connect 中声明数据收集 |

---

## 本章小结

| 主题 | 核心要点 |
|------|----------|
| 为什么需要 SDK | 数据分析、崩溃监控、性能监控是 App 运营的三大基础 |
| Firebase Analytics | 免费强大，自动事件+自定义事件+用户属性，海外首选 |
| Firebase Crashlytics | 实时崩溃报告，dSYM 符号化，与 Analytics 集成 |
| Sentry | 开源可自建，错误+性能一体化监控，灵活的告警规则 |
| 埋点方法论 | 先想问题再埋点，事件+属性+用户画像三要素 |
| 代码封装 | AnalyticsManager + Event 枚举，埋点与业务解耦 |
| 包大小 | 按需集成、SPM 精细选择、延迟初始化 |
| 隐私合规 | 用户同意、数据最小化、隐私清单、ATT、App Store 隐私标签 |
| 多 SDK 初始化 | 关键 SDK 先初始化、非关键延迟初始化、条件初始化 |
| 自建系统 | 合规要求高时考虑，需要专门的数据团队 |
| 国内 SDK | 友盟+Bugly 是国内标配，双市场需要双 SDK 策略 |

> 💡 一句话总结：第三方 SDK 是 App 的"感知器官"——选对 SDK、埋好数据、守住隐私，才能让数据真正为业务服务。

← [代码签名与证书管理深入](./94-代码签名与证书管理深入.md) | [App Store Connect 完整配置](../07-上架准备/97-App-Store-Connect完整配置.md) →
