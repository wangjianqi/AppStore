# App Clips 轻量版

## 本章目标

- 理解 App Clips 的概念和设计理念
- 掌握在 Xcode 中创建 App Clip Target 的方法
- 学会共享代码、条件编译、精简 UI 的开发技巧
- 了解 App Clip 的触发方式与启动流程
- 掌握 App Clip 与主 App 之间的数据共享方案
- 完成 Associated Domains、Smart App Banner 等链接配置
- 熟悉 App Store Connect 中 Clip 卡片的配置流程
- 了解审核要求与最佳实践

---

## 1. App Clips 是什么

### 1.1 一句话理解

> 💡 生活类比：App Clip 就像餐厅的"试吃小样"——你不需要买整份菜品（下载完整 App），就能先尝一口核心味道（体验核心功能），觉得好吃再点整份（下载完整 App）。

App Clips 是 iOS 14 引入的一种轻量级应用体验。用户无需从 App Store 下载完整 App，就能在需要的瞬间快速访问 App 的核心功能。

### 1.2 核心特点

| 特点 | 说明 |
|------|------|
| 无需下载 | 扫码/NFC 等方式直接打开，不用去 App Store 搜索下载 |
| 体积限制 | 最大 **10MB**（压缩后），必须精简 |
| 即用即走 | 用完即消失，不会长期驻留主屏幕 |
| 引导下载 | 体验后可一键安装完整 App |
| 系统集成 | 与 Apple Pay、Sign in with Apple 深度集成 |

### 1.3 App Clip 与主 App 的关系

```
┌─────────────────────────────────┐
│          主 App (完整版)          │
│  ┌───────────────────────────┐  │
│  │     App Clip (轻量版)      │  │
│  │   只包含核心功能，≤10MB    │  │
│  └───────────────────────────┘  │
│                                 │
│  其他完整功能、设置、历史记录等   │
└─────────────────────────────────┘
```

> ⚠️ App Clip 不是独立 App，它必须依附于一个主 App 存在。用户在 App Store 搜索不到单独的 App Clip。

---

## 2. 适用场景

### 2.1 典型场景一览

| 场景 | 举例 | 核心功能 |
|------|------|----------|
| 🍔 餐饮点餐 | 咖啡店扫码点单 | 浏览菜单 → 下单 → 支付 |
| 🚲 共享单车 | 扫码解锁单车 | 扫码 → 解锁 → 骑行 |
| 🅿️ 停车缴费 | 路边停车扫码缴费 | 查看费用 → 缴费 |
| 🎫 活动签到 | 展会/演唱会签到 | 扫码签到 → 获取信息 |
| 🛒 快速购物 | 便利店自助结账 | 扫商品 → 支付 |
| 🔑 门禁通行 | 酒店/公寓开门 | 验证身份 → 开门 |

### 2.2 适合做 App Clip 的功能特征

```
✅ 适合                          ❌ 不适合
─────────────────────────────    ─────────────────────────────
用户临时需要、快速完成            需要长时间沉浸使用
核心功能单一明确                  功能复杂、层级深
无需登录或快速登录                需要复杂注册流程
10MB 以内能实现                  依赖大量资源文件
线下场景触发                      纯线上浏览型应用
```

> 💡 判断标准：如果你的核心功能能在 **30 秒内完成**，那就非常适合做 App Clip。

---

## 3. 创建 App Clip Target

### 3.1 在 Xcode 中添加 Clip Target

**步骤：**

1. 打开你的 Xcode 项目
2. 点击菜单 `File → New → Target`
3. 选择 **App Clip** 模板
4. 填写配置信息：

| 配置项 | 说明 | 示例 |
|--------|------|------|
| Product Name | Clip Target 名称 | `MyAppClip` |
| Organization Identifier | 组织标识符 | `com.example` |
| Bundle Identifier | 自动生成 | `com.example.MyApp.Clip` |

> ⚠️ Clip 的 Bundle ID 必须是主 App Bundle ID 的子集。例如主 App 是 `com.example.MyApp`，Clip 就是 `com.example.MyApp.Clip`。

### 3.2 项目结构变化

添加 Clip Target 后，项目结构如下：

```
MyApp/
├── MyApp/                  # 主 App 代码
│   ├── AppDelegate.swift
│   ├── SceneDelegate.swift
│   └── Views/
├── MyAppClip/              # App Clip 代码
│   ├── ClipAppDelegate.swift
│   ├── ClipSceneDelegate.swift
│   └── Views/
├── Shared/                 # 共享代码（新建）
│   ├── Models/
│   ├── Services/
│   └── Utilities/
└── MyFramework/            # 共享框架（可选）
    ├── Models/
    └── Networking/
```

### 3.3 创建共享框架

为了让主 App 和 App Clip 复用代码，最佳做法是创建一个**共享框架**：

1. `File → New → Target`
2. 选择 **Framework**
3. 命名为 `Shared` 或 `Core`

```swift
// Shared/Models/CoffeeOrder.swift
import Foundation

public struct CoffeeOrder: Codable {
    public let id: String
    public let coffeeName: String
    public let size: Size
    public let price: Double

    public enum Size: String, Codable {
        case small, medium, large
    }

    public init(id: String, coffeeName: String, size: Size, price: Double) {
        self.id = id
        self.coffeeName = coffeeName
        self.size = size
        self.price = price
    }
}
```

然后在主 App 和 Clip Target 的 **Frameworks, Libraries, and Embedded Content** 中都添加这个框架。

> 💡 共享框架是 App Clip 开发中最重要的架构决策。把核心业务逻辑放在共享框架里，主 App 和 Clip 都调用它，避免代码重复。

---

## 4. App Clip 体验

### 4.1 触发方式

App Clip 可以通过以下方式触发：

| 触发方式 | 说明 | 用户操作 |
|----------|------|----------|
| 📷 扫码 | 扫描 App Clip Code 或二维码 | 相机扫描 |
| 📱 NFC | 手机靠近 NFC 标签 | 靠近即可 |
| 🌐 Safari | 网页中显示 Smart App Banner | 点击"打开" |
| 📍 地图 | Apple Maps 中显示 App Clip | 点击卡片 |
| 💬 信息 | iMessage 中分享链接 | 点击链接 |

### 4.2 App Clip Code

iOS 14.3 引入了专用的 **App Clip Code**，这是一种视觉标识：

```
┌──────────────┐
│   ╭──╮       │
│   │🍎│  ←──  │  App Clip Code
│   ╰──╯       │  (类似二维码但更美观)
│    ◇◇◇◇     │
│   ◇◇◇◇◇◇    │
│  ◇◇◇◇◇◇◇◇   │
└──────────────┘
```

> 💡 App Clip Code 同时支持视觉扫描（相机）和 NFC 触碰，一个码两种触发方式。

### 4.3 启动流程

```
用户扫码/NFC
    │
    ▼
系统识别 App Clip 链接
    │
    ▼
显示 App Clip 卡片（名称、描述、图片）
    │
    ▼
用户点击"打开" ──→ 下载 App Clip（后台，极快）
    │
    ▼
启动 Clip App ──→ 传入 URL（包含场景信息）
    │
    ▼
显示核心功能界面
```

### 4.4 处理启动 URL

App Clip 启动时会收到一个 URL，包含触发场景的信息：

```swift
// MyAppClip/ClipSceneDelegate.swift
import UIKit
import Shared

class ClipSceneDelegate: UIResponder, UIWindowSceneDelegate {
    var window: UIWindow?

    func scene(_ scene: UIScene,
               continue userActivity: NSUserActivity) {
        guard userActivity.activityType == NSUserActivityTypeBrowsingWeb,
              let incomingURL = userActivity.webpageURL else {
            return
        }

        handleClipURL(incomingURL)
    }

    private func handleClipURL(_ url: URL) {
        // 例如: https://example.com/order/coffee?id=123
        let pathComponents = url.pathComponents
        let queryItems = URLComponents(url: url, resolvingAgainstBaseURL: false)?.queryItems

        if pathComponents.contains("order") {
            let itemId = queryItems?.first(where: { $0.name == "id" })?.value
            showOrderScreen(itemId: itemId)
        }
    }

    private func showOrderScreen(itemId: String?) {
        // 直接展示点餐界面
    }
}
```

> ⚠️ 启动 URL 是 App Clip 获取场景上下文的关键。务必在 `scene(_:continue:)` 中正确解析并处理。

---

## 5. 核心功能开发

### 5.1 共享框架设计

推荐将以下内容放入共享框架：

| 层级 | 内容 | 是否共享 |
|------|------|----------|
| Model | 数据模型 | ✅ 共享 |
| Networking | API 请求 | ✅ 共享 |
| Service | 业务逻辑 | ✅ 共享 |
| Utility | 工具方法 | ✅ 共享 |
| UI 组件 | 通用组件 | ⚠️ 部分共享 |
| 页面 | 完整页面 | ❌ 各自实现 |

### 5.2 条件编译

使用编译标志区分主 App 和 Clip 代码：

**第一步：设置编译标志**

在 Clip Target 的 Build Settings 中：
- 搜索 `Active Compilation Conditionsitions`
- 添加 `APPCLIP`

**第二步：使用条件编译**

```swift
import Foundation

#if APPCLIP
print("运行在 App Clip 中")
#else
print("运行在主 App 中")
#endif
```

**实际应用示例——网络请求配置：**

```swift
// Shared/Services/APIService.swift
import Foundation

public class APIService {
    public static let shared = APIService()

    public var baseURL: String {
        #if APPCLIP
        return "https://api.example.com/v1/clip"
        #else
        return "https://api.example.com/v1"
        #endif
    }

    public func fetchMenu(completion: @escaping ([MenuItem]) -> Void) {
        let url = URL(string: "\(baseURL)/menu")!
        URLSession.shared.dataTask(with: url) { data, _, error in
            guard let data = data else { return }
            let menuItems = try? JSONDecoder().decode([MenuItem].self, from: data)
            completion(menuItems ?? [])
        }.resume()
    }
}
```

**实际应用示例——功能开关：**

```swift
// Shared/Services/FeatureFlag.swift
import Foundation

public struct FeatureFlag {
    public static var isFullApp: Bool {
        #if APPCLIP
        return false
        #else
        return true
        #endif
    }

    public static var canViewHistory: Bool { isFullApp }
    public static var canEditProfile: Bool { isFullApp }
    public static var canManagePayment: Bool { isFullApp }
    public static var canQuickOrder: Bool { true }
}
```

### 5.3 精简 UI

App Clip 的 UI 设计原则：

| 原则 | 说明 | 做法 |
|------|------|------|
| 聚焦核心 | 只展示最核心的功能 | 去掉侧边栏、设置页等 |
| 减少步骤 | 尽量少的操作完成任务 | 默认值、自动填充 |
| 快速支付 | 集成 Apple Pay | 一键支付 |
| 快速登录 | 使用 Sign in with Apple | 免注册 |
| 引导下载 | 适时提示安装完整 App | 横幅引导 |

**Clip 界面示例：**

```swift
// MyAppClip/Views/ClipOrderView.swift
import SwiftUI
import Shared

struct ClipOrderView: View {
    @State private var menuItems: [MenuItem] = []
    @State private var selectedItems: [MenuItem] = []

    var body: some View {
        NavigationView {
            VStack {
                List(menuItems) { item in
                    MenuItemRow(item: item, isSelected: selectedItems.contains(where: { $0.id == item.id })) {
                        toggleSelection(item)
                    }
                }

                if !selectedItems.isEmpty {
                    CheckoutButton(total: calculateTotal()) {
                        checkoutWithApplePay()
                    }
                }
            }
            .navigationTitle("快速点餐")
        }
        .onAppear {
            loadMenu()
        }
    }

    private func loadMenu() {
        APIService.shared.fetchMenu { items in
            DispatchQueue.main.async {
                self.menuItems = items
            }
        }
    }

    private func toggleSelection(_ item: MenuItem) {
        if let index = selectedItems.firstIndex(where: { $0.id == item.id }) {
            selectedItems.remove(at: index)
        } else {
            selectedItems.append(item)
        }
    }

    private func calculateTotal() -> Double {
        selectedItems.reduce(0) { $0 + $1.price }
    }

    private func checkoutWithApplePay() {
        // 调用 Apple Pay 完成支付
    }
}
```

> 💡 Clip UI 的核心思路：**一个屏幕完成一个任务**。不要让用户在多个页面间跳转。

---

## 6. App Clip 与主 App 数据共享

### 6.1 为什么需要数据共享

> 💡 生活类比：App Clip 就像你在酒店用临时房卡进了房间，但你想让正式会员卡（主 App）也能看到这次入住记录，就需要前台（App Group）帮忙同步信息。

常见场景：
- 用户在 Clip 中下单，安装主 App 后能看到订单历史
- 用户在 Clip 中登录，主 App 自动获得登录状态
- 用户在 Clip 中设置偏好，主 App 同步这些设置

### 6.2 App Group 配置

**第一步：在 Apple Developer 中创建 App Group**

| 步骤 | 操作 |
|------|------|
| 1 | 登录 [developer.apple.com](https://developer.apple.com) |
| 2 | 进入 Certificates, Identifiers & Profiles |
| 3 | 选择 Identifiers → App Groups |
| 4 | 创建 App Group，标识符格式：`group.com.example.myapp` |

**第二步：为主 App 和 Clip 的 Bundle ID 启用 App Group**

| 步骤 | 操作 |
|------|------|
| 1 | 选择主 App 的 Bundle ID |
| 2 | 勾选 App Groups 能力 |
| 3 | 分配刚才创建的 App Group |
| 4 | 对 Clip 的 Bundle ID 重复 1-3 步 |

**第三步：在 Xcode 中添加 App Group 能力**

1. 选择主 App Target → Signing & Capabilities → "+ Capability"
2. 添加 **App Groups**
3. 勾选对应的 Group
4. 对 Clip Target 重复以上步骤

### 6.3 使用 UserDefaults 共享数据

```swift
// Shared/Services/SharedDataStore.swift
import Foundation

public class SharedDataStore {
    public static let shared = SharedDataStore()

    private let defaults: UserDefaults

    public init() {
        let appGroupIdentifier = "group.com.example.myapp"
        defaults = UserDefaults(suiteName: appGroupIdentifier)!
    }

    public func saveOrder(_ order: CoffeeOrder) {
        var orders = fetchOrders()
        orders.append(order)
        if let data = try? JSONEncoder().encode(orders) {
            defaults.set(data, forKey: "saved_orders")
        }
    }

    public func fetchOrders() -> [CoffeeOrder] {
        guard let data = defaults.data(forKey: "saved_orders"),
              let orders = try? JSONDecoder().decode([CoffeeOrder].self, from: data) else {
            return []
        }
        return orders
    }

    public func saveUserToken(_ token: String) {
        defaults.set(token, forKey: "user_token")
    }

    public func fetchUserToken() -> String? {
        defaults.string(forKey: "user_token")
    }
}
```

### 6.4 使用共享文件容器

```swift
// Shared/Services/SharedFileManager.swift
import Foundation

public class SharedFileManager {
    public static let shared = SharedFileManager()

    public var sharedContainerURL: URL {
        let appGroupIdentifier = "group.com.example.myapp"
        guard let url = FileManager.default.containerURL(
            forSecurityApplicationGroupIdentifier: appGroupIdentifier
        ) else {
            fatalError("无法获取共享容器 URL")
        }
        return url
    }

    public func saveImage(data: Data, name: String) -> Bool {
        let fileURL = sharedContainerURL.appendingPathComponent(name)
        do {
            try data.write(to: fileURL)
            return true
        } catch {
            print("保存失败: \(error)")
            return false
        }
    }

    public func loadImage(name: String) -> Data? {
        let fileURL = sharedContainerURL.appendingPathComponent(name)
        return try? Data(contentsOf: fileURL)
    }
}
```

### 6.5 数据共享对照表

| 共享方式 | 适用数据 | 大小限制 | 复杂度 |
|----------|----------|----------|--------|
| UserDefaults (suiteName) | 简单键值对、小型配置 | KB 级 | ⭐ |
| 共享文件容器 | 图片、数据库文件 | MB 级 | ⭐⭐ |
| Keychain (kSecAttrAccessGroup) | 密码、令牌 | KB 级 | ⭐⭐⭐ |
| CoreData + 共享容器 | 结构化数据 | MB 级 | ⭐⭐⭐ |

> ⚠️ Keychain 共享需要额外配置 Keychain Group，且两个 Target 必须属于同一开发者账号。

---

## 7. 配置 App Clip 链接

### 7.1 Associated Domains 配置

App Clip 通过 Universal Link 触发，需要配置 Associated Domains。

**第一步：在 Xcode 中添加能力**

1. 选择 Clip Target → Signing & Capabilities
2. 添加 **Associated Domains**
3. 添加格式：`clipapp:example.com`（注意不是 `applinks:`）

> ⚠️ 主 App 用 `applinks:example.com`，Clip 用 `clipapp:example.com`，两者前缀不同！

**第二步：创建 Apple App Site Association (AASA) 文件**

在你的服务器上放置 JSON 文件：

```
https://example.com/.well-known/apple-app-site-association
```

文件内容：

```json
{
    "appclips": {
        "apps": [
            "ABCDE12345.com.example.MyApp.Clip"
        ]
    },
    "applinks": {
        "details": [
            {
                "appIDs": [
                    "ABCDE12345.com.example.MyApp"
                ],
                "components": [
                    {
                        "/": "/order/*"
                    }
                ]
            }
        ]
    }
}
```

| 字段 | 说明 |
|------|------|
| `appclips.apps` | Clip 的 Bundle ID（含 Team ID 前缀） |
| `applinks.details` | 主 App 的 Universal Link 配置 |
| `components` | 匹配的 URL 路径规则 |

> ⚠️ AASA 文件必须通过 HTTPS 提供，Content-Type 为 `application/json`，且不能有 `.json` 后缀。

### 7.2 Smart App Banner

在网页中添加 Smart App Banner，可以让 Safari 用户快速打开 App Clip：

```html
<!-- 在网页 <head> 中添加 -->
<meta
    apple-itunes-app="app-id=1234567890,
    app-clip-bundle-id=com.example.MyApp.Clip,
    affiliate-data=optional_affiliate_data,
    app-argument=https://example.com/order/coffee?id=123"
>
```

| 参数 | 说明 |
|------|------|
| `app-id` | 主 App 的 App Store ID |
| `app-clip-bundle-id` | Clip 的 Bundle ID |
| `app-argument` | 传递给 Clip 的 URL |

### 7.3 URL 路径设计建议

```
https://example.com/
├── order/          → Clip: 点餐页面
│   ├── coffee/     → Clip: 咖啡点餐
│   └── food/       → Clip: 食物点餐
├── parking/        → Clip: 停车缴费
└── about/          → 主 App: 关于页面（Clip 不处理）
```

> 💡 URL 路径要语义清晰。Clip 只处理自己负责的路径，其他路径由主 App 或网页处理。

---

## 8. App Store Connect 配置

### 8.1 Clip 卡片配置

App Clip 在触发时会显示一张卡片，需要在 App Store Connect 中配置：

**路径：** App Store Connect → 你的 App → App Clip → 体验

| 配置项 | 说明 | 要求 |
|--------|------|------|
| 标题 | Clip 卡片标题 | 简短，如"点杯咖啡" |
| 副标题 | 补充说明 | 如"30秒完成点单" |
| 默认 URL | 默认触发链接 | `https://example.com/order` |
| 图片 | 卡片展示图 | 1800×1200 px |

### 8.2 配置步骤

1. 登录 [App Store Connect](https://appstoreconnect.apple.com)
2. 选择你的 App
3. 左侧菜单找到 **App Clip**
4. 点击 **设置 App Clip**
5. 填写以下信息：

| 区域 | 内容 | 示例 |
|------|------|------|
| App Clip 体验标题 | 用户看到的名称 | "快速点餐" |
| 副标题 | 补充描述 | "扫码即点，无需下载" |
| 默认 URL | 入口链接 | `https://example.com/order` |
| 头部图片 | 卡片顶部图片 | 1800×1200 像素 |

### 8.3 多体验配置

一个 App Clip 可以配置多个"体验"，对应不同的 URL 路径：

| 体验 | URL | 标题 | 副标题 |
|------|-----|------|--------|
| 点餐 | `/order` | 快速点餐 | 扫码即点 |
| 停车 | `/parking` | 停车缴费 | 即停即付 |
| 签到 | `/checkin` | 活动签到 | 扫码入场 |

> ⚠️ 每个体验都需要单独的图片和描述，且图片规格必须严格符合要求。

---

## 9. 审核要求与最佳实践

### 9.1 审核要求

Apple 对 App Clip 的审核有额外要求：

| 要求 | 说明 |
|------|------|
| 功能完整 | Clip 必须能独立完成核心功能，不能只是主 App 的广告 |
| 体积限制 | 压缩后不超过 **10MB** |
| 无注册墙 | 不能强制用户注册才能使用核心功能 |
| Apple Pay 优先 | 涉及支付时应支持 Apple Pay |
| Sign in with Apple | 如需登录，必须支持 Sign in with Apple |
| 隐私合规 | 明确说明数据收集用途 |
| 位置权限 | 仅在必要时请求位置权限 |

### 9.2 最佳实践

#### 体积优化

```swift
// ❌ 不要在 Clip 中包含大量资源
// 不要把所有城市的门店数据打包进 Clip

// ✅ 按需加载
// Clip 只包含最基础的 UI，数据从服务器获取
func loadNearbyStores() {
    let url = URL(string: "\(API.baseURL)/stores/nearby?lat=\(lat)&lng=\(lng)")!
    // 从服务器获取附近门店
}
```

| 优化手段 | 说明 |
|----------|------|
| 按需加载资源 | 图片、数据从服务器获取 |
| 使用 Asset Catalog 的 On-Demand Resources | 延迟加载资源 |
| 精简第三方库 | 只引入必要的库 |
| 避免多语言全量打包 | 只包含必要语言 |
| 压缩图片资源 | 使用 WebP 或压缩 PNG |

#### 引导安装主 App

```swift
import SwiftUI
import StoreKit

struct ClipToFullAppBanner: View {
    @State private var showBanner = false

    var body: some View {
        VStack {
            if showBanner {
                HStack {
                    Image(systemName: "arrow.down.circle.fill")
                        .foregroundColor(.blue)
                    Text("获取完整版，享受更多功能")
                        .font(.subheadline)
                    Spacer()
                    Button("安装") {
                        presentAppStoreOverlay()
                    }
                    .buttonStyle(.borderedProminent)
                    .controlSize(.small)
                }
                .padding()
                .background(Color(.systemGray6))
                .cornerRadius(10)
                .padding(.horizontal)
            }
        }
        .onAppear {
            DispatchQueue.main.asyncAfter(deadline: .now() + 3) {
                showBanner = true
            }
        }
    }

    private func presentAppStoreOverlay() {
        let scene = UIApplication.shared.connectedScenes
            .first as? UIWindowScene
        SKOverlay(
            configuration: SKOverlay.AppClipConfiguration(
                position: .bottom
            )
        ).present(in: scene!)
    }
}
```

> 💡 使用 `SKOverlay` 可以在 Clip 内展示 App Store 下载浮层，这是 Apple 推荐的引导方式，比自定义弹窗体验更好。

#### 启动速度优化

```swift
// ❌ 不要在启动时做大量初始化
func application(_ application: UIApplication,
                 didFinishLaunchingWithOptions launchOptions: ...) -> Bool {
    DatabaseManager.shared.initialize()    // 耗时
    AnalyticsManager.shared.setup()        // 耗时
    CacheManager.shared.warmUp()           // 耗时
    return true
}

// ✅ 延迟初始化，只做最必要的
func application(_ application: UIApplication,
                 didFinishLaunchingWithOptions launchOptions: ...) -> Bool {
    #if APPCLIP
    // Clip 中只做最少的初始化
    APIService.shared.configure()
    #else
    // 主 App 做完整初始化
    DatabaseManager.shared.initialize()
    AnalyticsManager.shared.setup()
    CacheManager.shared.warmUp()
    #endif
    return true
}
```

### 9.3 常见审核被拒原因

| 被拒原因 | 解决方案 |
|----------|----------|
| Clip 只是主 App 的广告 | 确保 Clip 能独立完成核心功能 |
| 体积超过 10MB | 精简资源、按需加载 |
| 强制注册才能使用 | 允许游客模式或使用 Sign in with Apple |
| 不支持 Apple Pay | 涉及支付时集成 Apple Pay |
| 缺少隐私说明 | 在 Info.plist 中添加隐私描述 |
| URL 配置错误 | 检查 AASA 文件和 Associated Domains |

### 9.4 检查清单

在提交审核前，逐项确认：

- [ ] Clip 压缩后体积 ≤ 10MB
- [ ] 核心功能无需注册即可使用
- [ ] 支付场景已集成 Apple Pay
- [ ] 登录场景已集成 Sign in with Apple
- [ ] AASA 文件可正常访问
- [ ] Associated Domains 配置正确
- [ ] App Group 数据共享正常
- [ ] App Store Connect 卡片信息已填写
- [ ] 所有 Clip 体验均有对应图片
- [ ] 主 App 安装后能读取 Clip 产生的数据

---

## 小结

| 知识点 | 要点 |
|--------|------|
| App Clips 是什么 | 无需下载的轻量 App，10MB 限制，即用即走 |
| 适用场景 | 临时性、快速完成的核心功能（点餐、缴费、签到等） |
| 创建 Clip Target | Xcode 添加 App Clip Target，Bundle ID 为主 App 子集 |
| 触发方式 | 扫码、NFC、Safari、地图、信息 |
| 核心开发 | 共享框架复用代码、`#if APPCLIP` 条件编译、精简 UI |
| 数据共享 | App Group + UserDefaults/文件容器/Keychain |
| 链接配置 | Associated Domains (`clipapp:`)、AASA 文件、Smart App Banner |
| App Store Connect | 配置 Clip 卡片标题、图片、默认 URL |
| 审核要求 | 功能完整、体积合规、支持 Apple Pay 和 Sign in with Apple |

> 💡 App Clips 的核心哲学：**让用户在最需要的时刻，用最少的步骤，完成最核心的功能**。记住这个原则，你就能设计出优秀的 App Clip 体验。

← [TestFlight 测试](./TestFlight测试.md) | [App Extension 全景](./App-Extension全景.md) →
