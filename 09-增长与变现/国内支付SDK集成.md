# 国内支付 SDK 集成

> 🎯 **本章目标**：掌握微信支付和支付宝支付的完整集成流程，理解服务端签名与验签机制，建立统一支付管理器架构，了解 Apple 审核对第三方支付的合规要求与边界。

---

## 一、国内支付生态概述

### 1.1 支付方式市场份额

中国移动互联网支付市场由两大巨头主导，形成了独特的双寡头格局：

| 支付方式 | 市场份额 | 月活用户 | 核心优势 | 典型场景 |
|----------|----------|----------|----------|----------|
| 微信支付 | ~40% | 13亿+ | 社交裂变、小程序生态 | 社交电商、线下扫码、小程序内购 |
| 支付宝 | ~55% | 10亿+ | 信用体系、金融生态 | 电商购物、生活缴费、信用支付 |
| Apple Pay | ~1% | 有限 | 系统级集成、NFC | 线下 NFC 支付、App 内快捷支付 |
| 云闪付 | ~4% | 3亿+ | 银联背景、银行补贴 | 大额转账、银行优惠活动 |

> 💡 **提示**：以上数据为估算值，实际份额随时间和场景动态变化。对于 iOS App 开发者而言，微信支付 + 支付宝的覆盖率已超过 95%，两者缺一不可。

### 1.2 为什么国内 App 必须支持微信/支付宝支付

国内用户已经形成了根深蒂固的移动支付习惯。不支持微信/支付宝支付，意味着放弃了绝大多数潜在付费用户：

- **用户心智**：国内用户看到"支付"二字，第一反应就是找微信或支付宝图标
- **支付门槛**：绑定银行卡需要填写卡号、手机号、验证码，流程繁琐；而微信/支付宝只需输入 6 位密码或刷脸即可完成
- **信任基础**：用户对微信/支付宝的安全机制有充分信任，但对陌生 App 的支付页面天然警惕
- **运营需求**：微信支付支持公众号/小程序联动推广，支付宝有花呗分期等金融工具，这些是原生支付无法提供的增值能力

### 1.3 Apple IAP vs 第三方支付的适用场景

这是国内 iOS 开发者最常遇到的核心问题——什么时候用 Apple IAP，什么时候可以用第三方支付：

| 场景 | Apple IAP（StoreKit） | 第三方支付（微信/支付宝） | 说明 |
|------|----------------------|------------------------|------|
| 虚拟货币/游戏币 | ✅ 必须 | ❌ 不允许 | 数字商品必须走 IAP |
| 解锁 App 功能/去广告 | ✅ 必须 | ❌ 不允许 | 数字内容解锁走 IAP |
| 订阅会员（纯数字内容） | ✅ 必须 | ❌ 不允许 | 订阅类数字内容走 IAP |
| 购买实物商品 | ❌ 不适用 | ✅ 推荐 | 实物电商可用第三方支付 |
| O2O 服务（打车/外卖） | ❌ 不适用 | ✅ 推荐 | 线上付款线下消费 |
| 知识付费课程 | ⚠️ 边界模糊 | ⚠️ 需谨慎 | 取决于内容交付形式 |
| 捐赠/打赏 | ⚠️ 视情况 | ⚠️ 需合规设计 | 读者对创作者的直接打赏有例外空间 |
| 充值后消费实体服务 | ⚠️ 可协商 | ✅ 常见做法 | 如充值话费、充加油卡 |

### 1.4 Apple 审核 3.1.1 条款详解

Apple《App Store 审核指南》3.1.1 条款是关于"应用内购买"的核心规则，其原文要点如下：

- **核心原则**：如果 App 内提供可解锁的功能性、订阅式或虚拟内容，则必须通过 IAP 提供
- **禁止行为**：不得包含指向外部购买机制的按钮、外部链接或其他行动号召
- **例外情况**：跨平台消费的商品（在 iOS 外也可使用的）、实物商品和服务、通过 Apple 批准的特定阅读器应用

> ⚠️ **警告**：3.1.1 条款的执行力度逐年加强。Apple 会通过机器扫描 + 人工审核双重手段检测违规行为，包括但不限于：代码中搜索"alipay"、"wechat"、"pay"等关键词、UI 中出现第三方支付图标、网络请求中包含第三方支付域名。

### 1.5 合规策略总览

针对国内开发者的实际业务场景，推荐以下分层合规策略：

| 业务类型 | 推荐方案 | 合规说明 |
|----------|----------|----------|
| 纯数字内容 App | StoreKit IAP | 无选择余地，必须走 IAP |
| 电商 App（卖实物） | 微信/支付宝 + Apple Pay | 实物商品不受 3.1.1 限制 |
| O2O 服务 App | 微信/支付宝 | 打车、外卖等服务类交易合规 |
| 混合型 App（内容+实物） | 数字内容走 IAP，实物走第三方 | 需要在 UI 和逻辑上严格隔离 |
| 读者打赏类 | 可申请特殊审批 | 需提前与 Apple 沟通确认 |

> 💡 **提示**：最安全的做法是在提交审核前，将所有涉及第三方支付的入口做条件编译或服务端开关控制，审核版本隐藏第三方支付通道，上线后再开启。但这属于灰色操作，存在被拒风险，需自行评估。

---

## 二、微信支付集成

### 2.1 商户平台注册与配置

#### 注册流程

1. 访问 [微信支付商户平台](https://pay.weixin.qq.com)，使用企业资质注册商户号
2. 完成企业实名认证（营业执照、法人身份证、银行账户）
3. 签署协议并缴纳保证金（根据行业不同，0~10 万元不等）
4. 在商户平台 → 产品中心 → App 支付，开通移动端支付权限
5. 获取关键参数：商户号（mch_id）、API 密钥（key）、AppID

#### 关键配置项

| 配置项 | 获取位置 | 用途 |
|--------|----------|------|
| AppID | 微信开放平台 → 应用详情 | 标识你的 App |
| mch_id | 商户平台 → 账户中心 | 商户唯一标识 |
| API Key | 商户平台 → 账户中心 → API 安全 | 用于签名计算 |
| Cert Path | 商户平台 → 账户中心 → API 安全 | 证书文件，用于退款等高级接口 |

> 💡 **提示**：API 密钥建议每 3 个月更换一次。密钥泄露意味着任何人都可以伪造支付订单，后果极其严重。

### 2.2 SDK 集成

#### 方案一：CocoaPods 集成

```ruby
pod 'WechatOpenSDK'
```

#### 方案二：SPM 集成

在 Xcode 中添加 Package Dependency：
- URL: `https://github.com/wechat-sdk/WeChatSDK`
- 选择 WechatOpenSDK-XCFramework 产品

#### 初始化配置

```swift
import WechatOpenSDK

class WXPayConfig {
    static let shared = WXPayConfig()

    func register(appId: String, universalLink: String) {
        WXApi.registerApp(appId, universalLink: universalLink)
    }
}
```

在 `AppDelegate` 或 App 入口处调用初始化：

```swift
@main
struct MyApp: App {
    init() {
        WXPayConfig.shared.register(
            appId: "your_wechat_appid",
            universalLink: "https://yourdomain.com/app/"
        )
    }

    var body: some Scene {
        WindowGroup {
            ContentView()
        }
    }
}
```

### 2.3 Universal Link 配置

Universal Link 是微信支付回调的关键环节，配置不正确会导致支付完成后无法返回 App。

#### 步骤一：创建 apple-app-site-association 文件

```json
{
    "applinks": {
        "apps": [],
        "details": [
            {
                "appIDs": ["TEAM_ID.com.yourcompany.appname"],
                "components": [
                    {
                        "/": "/app/*",
                        "comment": "Matches any URL whose path starts with /app/"
                    },
                    {
                        "/": "/pay/*",
                        "comment": "Matches payment callback URLs"
                    }
                ]
            }
        ]
    }
}
```

将此 JSON 文件上传到服务器 `https://yourdomain.com/.well-known/apple-app-site-association`（注意：无 .json 后缀，Content-Type 为 application/json）。

#### 步骤二：Xcode 配置

在 Xcode → Target → Signing & Capabilities 中：
- 添加 Associated Domains 能力
- 添加条目：`applinks:yourdomain.com`

#### 步骤三：微信开放平台配置

在微信开放平台 → 应用开发 → 开发配置中：
- 填写 Universal Links：`https://yourdomain.com/app/`
- 确保该域名已备案且支持 HTTPS

> ⚠️ **警告**：Universal Link 配置错误是最常见的微信支付问题。常见症状包括：支付后不跳回 App、iOS 提示"打开失败"、Safari 中验证 Universal Link 失败。务必使用 Apple 的 [App Search API Validation Tool](https://appsearchconnect.apple.com/) 验证配置是否正确。

### 2.4 支付流程详解

完整的微信支付流程包含 6 个步骤：

```
┌─────────────┐     ┌─────────────┐     ┌──────────┐     ┌──────────┐     ┌─────────┐     ┌─────────────┐
│   用户 App   │────▶│   你的服务端  │────▶│ 微信支付  │────▶│  微信 App │────▶│  用户    │────▶│ 微信支付服务端 │
│  发起支付请求 │     │  统一下单API │     │  下单响应  │     │  展示支付  │     │ 确认支付  │     │  异步通知回调  │
└─────────────┘     └─────────────┘     └──────────┘     └──────────┘     └─────────┘     └─────────────┘
       ①                   ②                  ③              ④               ⑤              ⑥
```

**① App 发起支付请求**

用户点击支付按钮，App 向自己的服务端发起下单请求，携带商品信息、金额、用户标识等参数。

**② 服务端调用统一下单 API**

服务端收到请求后，调用微信支付「统一下单」接口，传入商户号、AppID、金额、订单号、签名等信息。微信返回预支付交易会话标识（prepay_id）。

**③ 服务端返回支付参数给 App**

服务端将 prepay_id 以及二次签名的参数（appid、partnerid、prepayid、package、noncestr、timestamp、sign）返回给客户端 App。

**④ App 拉起微信支付**

客户端使用 SDK 的 `WXApi.sendReq()` 方法，构造 `PayReq` 对象并调起微信客户端展示支付界面。

**⑤ 用户完成支付**

用户在微信中确认支付（密码/指纹/人脸），微信处理扣款。

**⑥ 异步通知回调**

无论支付成功还是失败，微信都会向服务端的 notify_url 发送异步通知。服务端收到通知后进行验签、更新订单状态，再通知客户端结果。

### 2.5 SwiftUI 封装微信支付调用

```swift
import SwiftUI
import WechatOpenSDK

enum PaymentError: LocalizedError {
    case wechatNotInstalled
    case invalidParameter
    case payFailed(code: Int)
    case cancel
    case unknown

    var errorDescription: String? {
        switch self {
        case .wechatNotInstalled: return "未安装微信"
        case .invalidParameter: return "支付参数无效"
        case .payFailed(let code): return "支付失败(错误码: \(code))"
        case .cancel: return "用户取消支付"
        case .unknown: return "未知错误"
        }
    }
}

@MainActor
final class WechatPaymentManager: NSObject, ObservableObject, WXApiDelegate {
    static let shared = WechatPaymentManager()
    @Published var isProcessing = false

    private var continuation: CheckedContinuation<Void, Error>?

    override init() {
        super.init()
        WXApi.delegate = self
    }

    func pay(orderInfo: WechatOrderInfo) async throws {
        guard WXApi.isWXAppInstalled() else {
            throw PaymentError.wechatNotInstalled
        }

        isProcessing = true
        defer { isProcessing = false }

        try await withCheckedThrowingContinuation { continuation in
            self.continuation = continuation

            let req = PayReq()
            req.partnerId = orderInfo.partnerId
            req.prepayId = orderInfo.prepayId
            req.nonceStr = orderInfo.nonceStr
            req.timeStamp = UInt32(orderInfo.timeStamp) ?? 0
            req.package = orderInfo.package
            req.sign = orderInfo.sign

            let result = WXApi.send(req)
            if !result {
                continuation.resume(throwing: PaymentError.invalidParameter)
            }
        }
    }

    func onResp(_ resp: BaseResp) {
        guard let payResp = resp as? PayResp,
              let continuation = continuation else { return }

        switch payResp.errCode {
        case WXSuccess.rawValue:
            continuation.resume()
        case WXErrCodeUserCancel.rawValue:
            continuation.resume(throwing: PaymentError.cancel)
        default:
            continuation.resume(throwing: PaymentError.payFailed(code: payResp.errCode))
        }
        self.continuation = nil
    }
}

struct WechatOrderInfo {
    let partnerId: String
    let prepayId: String
    let nonceStr: String
    let timeStamp: String
    let package: String
    let sign: String
}

struct WechatPayButton: View {
    let amount: Decimal
    let orderId: String
    @StateObject private var paymentManager = WechatPaymentManager.shared
    @State private var showError = false
    @State private var errorMessage = ""

    var body: some View {
        Button(action: handlePayment) {
            HStack {
                Image(systemName: "message.fill")
                    .foregroundColor(.green)
                Text("微信支付 ¥\(amount)")
                    .font(.headline)
                    .foregroundColor(.white)
            }
            .frame(maxWidth: .infinity)
            .padding(.vertical, 14)
            .background(Color(red: 0.07, green: 0.62, blue: 0.46))
            .cornerRadius(10)
        }
        .disabled(paymentManager.isProcessing)
        .overlay {
            if paymentManager.isProcessing {
                ProgressView()
                    .tint(.white)
            }
        }
        .alert("支付失败", isPresented: $showError) {
            Button("确定", role: .cancel) {}
        } message: {
            Text(errorMessage)
        }
    }

    func handlePayment() {
        Task {
            do {
                let orderInfo = try await fetchPrepayParams(orderId: orderId)
                try await paymentManager.pay(orderInfo: orderInfo)
            } catch {
                errorMessage = error.localizedDescription
                showError = true
            }
        }
    }

    func fetchPrepayParams(orderId: String) async throws -> WechatOrderInfo {
        guard let url = URL(string: "https://api.yourserver.com/pay/wechat/prepay") else {
            throw URLError(.badURL)
        }
        var request = URLRequest(url: url)
        request.httpMethod = "POST"
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")
        let body: [String: Any] = ["orderId": orderId]
        request.httpBody = try? JSONSerialization.data(withJSONObject: body)

        let (data, _) = try await URLSession.shared.data(for: request)
        let json = try JSONSerialization.jsonObject(with: data) as? [String: Any]
        guard let dict = json?["data"] as? [String: String] else {
            throw PaymentError.invalidParameter
        }
        return WechatOrderInfo(
            partnerId: dict["partnerId"] ?? "",
            prepayId: dict["prepayId"] ?? "",
            nonceStr: dict["nonceStr"] ?? "",
            timeStamp: dict["timeStamp"] ?? "",
            package: dict["package"] ?? "Sign=WXPay",
            sign: dict["sign"] ?? ""
        )
    }
}
```

### 2.6 支付结果回调处理

微信支付的结果通过两个渠道获取：

| 渠道 | 触发时机 | 可靠性 | 说明 |
|------|----------|--------|------|
| SDK 回调（onResp） | 用户在微信完成操作后立即触发 | 仅供参考 | 可能因网络问题丢失 |
| 服务端异步通知 | 微信服务器主动推送 | **权威来源** | 以此为准更新订单状态 |

正确的处理方式是：SDK 回调仅用于 UI 反馈（显示成功/失败），最终订单状态以服务端异步通知为准。客户端应实现轮询或 WebSocket 来获取服务端确认后的订单状态。

### 2.7 常见问题排查

| 问题现象 | 可能原因 | 解决方案 |
|----------|----------|----------|
| 调用 sendReq 返回 false | 参数缺失或格式错误 | 检查所有字段是否非空，timestamp 是否为合法数字 |
| 微信未响应 / 无反应 | Universal Link 未生效 | 使用 Apple 验证工具检查 AASA 文件 |
| 支付后不回到 App | Universal Link 域名不匹配 | 确保 Xcode 中 Associated Domains 与微信后台一致 |
| 错误码 -1（通用错误） | 签名错误或参数问题 | 在商户平台使用签名校验工具对比签名 |
| 错误码 -2（用户取消） | 正常行为 | 用户主动取消，无需处理 |
| 回调 errCode 始终为 -2 | 未设置 WXApi.delegate | 确保在 App 启动时设置了 delegate 且未被释放 |
| iOS 17+ 回调不触发 | AppDelegate 适配问题 | 在 SceneDelegate 或 UIWindowScene 中正确处理 openURL |

> 💡 **提示**：微信支付官方提供了[SDK 调试工具](https://pay.weixin.qq.com/wiki/doc/apiv3/sdk/ios/index.shtml)，可以快速定位大部分集成问题。建议先跑通 Demo 再接入正式项目。

---

## 三、支付宝支付集成

### 3.1 开放平台配置

#### 注册与开通

1. 访问 [支付宝开放平台](https://open.alipay.com)，注册企业开发者账号
2. 创建应用，获取 APPID
3. 添加「App 支付」能力
4. 配置接口加密方式（RSA2），上传应用公钥
5. 等待审核通过（通常 1-3 个工作日）

#### 密钥体系

支付宝采用 RSA2（SHA256WithRSA）非对称加密体系：

| 密钥类型 | 生成方 | 保存位置 | 用途 |
|----------|--------|----------|------|
| 应用私钥 | 开发者本地生成 | 服务端安全存储 | 对请求参数签名 |
| 应用公钥 | 开发者本地生成 | 上传至支付宝开放平台 | 支付宝验证请求签名 |
| 支付宝公钥 | 支付宝生成 | 从开放平台下载 | 验证支付宝回调和响应 |

> ⚠️ **警告**：应用私钥绝对不能出现在客户端代码中！私钥只存在于服务端。客户端只负责拉起支付和接收结果，所有签名工作都在服务端完成。

### 3.2 SDK 集成步骤

#### CocoaPods 集成

```ruby
pod 'AlipaySDK-iOS'
```

#### Info.plist 配置

在 Info.plist 中添加 URL Scheme 和白名单配置：

```xml
<key>LSApplicationQueriesSchemes</key>
<array>
    <string>alipay</string>
    <string>alipays</string>
</array>

<key>CFBundleURLTypes</key>
<array>
    <dict>
        <key>CFBundleURLSchemes</key>
        <array>
            <string>your_app_scheme_from_alipay</string>
        </array>
    </dict>
</array>
```

#### 处理 Open URL 回调

支付宝通过 URL Scheme 回调支付结果，需要在 App 生命周期中拦截：

```swift
import AlipaySDK

@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .onOpenURL { url in
            AlipaySDK.defaultService().processOrder(withPaymentResult: url) { resultDic in
                let status = resultDic?["resultStatus"] as? String
                print("Alipay callback status: \(status ?? "unknown")")
            }
        }
    }
}
```

### 3.3 支付流程与微信支付的异同

| 对比维度 | 微信支付 | 支付宝支付 |
|----------|----------|------------|
| 回调方式 | Universal Link + WXApiDelegate | URL Scheme + block 回调 |
| 签名算法 | HMAC-SHA256（MD5 已废弃） | RSA2（SHA256WithRSA） |
| 统一下单 | 需要（获取 prepay_id） | 不需要（直接传 orderString） |
| SDK 体积 | 较小（~2MB） | 较大（~8MB，含安全组件） |
| 支付界面 | 微信 App 内展示 | 支付宝 App 或 H5 内展示 |
| 结果可靠性 | 以服务端通知为准 | 以服务端通知为准 |
| 沙箱环境 | 有独立沙箱 | 有独立沙箱 |
| 调试难度 | Universal Link 配置复杂 | 相对简单，URL Scheme 更直观 |

### 3.4 SwiftUI 封装支付宝调用

```swift
import SwiftUI
import AlipaySDK

enum AlipayPaymentError: LocalizedError {
    case alipayNotInstalled
    case invalidOrderString
    case payFailed(resultStatus: String)
    case cancel
    case unknown

    var errorDescription: String? {
        switch self {
        case .alipayNotInstalled: return "未安装支付宝"
        case .invalidOrderString: return "支付订单信息无效"
        case .payFailed(let s): return "支付失败(状态码: \(s))"
        case .cancel: return "用户取消支付"
        case .unknown: return "未知错误"
        }
    }
}

@MainActor
final class AlipayPaymentManager: ObservableObject {
    static let shared = AlipayPaymentManager()
    @Published var isProcessing = false

    func pay(orderString: String) async throws {
        isProcessing = true
        defer { isProcessing = false }

        try await withCheckedThrowingContinuation { (continuation: CheckedContinuation<Void, Error>) in
            AlipaySDK.defaultService().payOrder(orderString, fromScheme: "yourscheme") { resultDic in
                guard let dic = resultDic,
                      let status = dic["resultStatus"] as? String else {
                    continuation.resume(throwing: AlipayPaymentError.unknown)
                    return
                }

                switch status {
                case "9000":
                    continuation.resume()
                case "6001":
                    continuation.resume(throwing: AlipayPaymentError.cancel)
                case "6002":
                    continuation.resume(throwing: AlipayPaymentError.alipayNotInstalled)
                default:
                    continuation.resume(throwing: AlipayPaymentError.payFailed(resultStatus: status))
                }
            }
        }
    }
}

struct AlipayPayButton: View {
    let amount: Decimal
    let orderId: String
    @StateObject private var paymentManager = AlipayPaymentManager.shared
    @State private var showError = false
    @State private var errorMessage = ""

    var body: some View {
        Button(action: handlePayment) {
            HStack {
                Image(systemIcon: "alipay_logo")
                    .foregroundColor(.blue)
                Text("支付宝支付 ¥\(amount)")
                    .font(.headline)
                    .foregroundColor(.white)
            }
            .frame(maxWidth: .infinity)
            .padding(.vertical, 14)
            .background(Color(red: 0.16, green: 0.54, blue: 0.93))
            .cornerRadius(10)
        }
        .disabled(paymentManager.isProcessing)
        .overlay {
            if paymentManager.isProcessing {
                ProgressView()
                    .tint(.white)
            }
        }
        .alert("支付失败", isPresented: $showError) {
            Button("确定", role: .cancel) {}
        } message: {
            Text(errorMessage)
        }
    }

    func handlePayment() {
        Task {
            do {
                let orderString = try await fetchOrderString(orderId: orderId)
                try await paymentManager.pay(orderString: orderString)
            } catch {
                errorMessage = error.localizedDescription
                showError = true
            }
        }
    }

    func fetchOrderString(orderId: String) async throws -> String {
        guard let url = URL(string: "https://api.yourserver.com/pay/alipay/order") else {
            throw URLError(.badURL)
        }
        var request = URLRequest(url: url)
        request.httpMethod = "POST"
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")
        let body: [String: Any] = ["orderId": orderId]
        request.httpBody = try? JSONSerialization.data(withJSONObject: body)

        let (data, _) = try await URLSession.shared.data(for: request)
        let json = try JSONSerialization.jsonObject(with: data) as? [String: Any]
        guard let orderString = json?["data"] as? String, !orderString.isEmpty else {
            throw AlipayPaymentError.invalidOrderString
        }
        return orderString
    }
}
```

### 3.5 支付结果状态码说明

支付宝支付回调中的 `resultStatus` 字段含义：

| 状态码 | 含义 | 处理方式 |
|--------|------|----------|
| 9000 | 支付成功 | UI 显示成功，等待服务端最终确认 |
| 8000 | 正在处理中 | 展示"支付处理中"，轮询查询订单状态 |
| 4000 | 支付失败 | 提示用户重试 |
| 5000 | 重复请求 | 忽略，防止重复支付 |
| 6001 | 用户中途取消 | 静默处理或温和提示 |
| 6002 | 网络连接出错 | 提示用户检查网络后重试 |
| 4001 | 数据异常 | 记录日志，联系技术排查 |

> 💡 **提示**：状态码 9000 仅表示用户在支付宝侧完成了支付流程，不代表资金已到账。最终的订单成交状态必须以服务端收到的异步通知（notify_url）为准。客户端应在支付完成后轮询服务端订单状态接口，直到获得最终确认。

---

## 四、服务端签名与验签

### 4.1 为什么签名必须在服务端完成

这是支付安全的第一原则：**所有涉及密钥的操作必须在服务端完成**。

如果把签名逻辑放在客户端 App 中，攻击者可以通过以下方式破解：

- **逆向工程**：使用 Hopper、IDA Pro 等工具反编译二进制文件，提取硬编码的密钥
- **中间人抓包**：通过 Charles/Proxyman 抓取网络请求，分析签名算法和参数
- **内存注入**：运行时通过 Frida/Lldb 注入进程，读取内存中的敏感数据
- **重放攻击**：截获一次合法请求，修改金额后重复发送

一旦密钥泄露，攻击者可以伪造任意金额的订单、篡改回调通知，造成直接的资金损失。因此，签名密钥（无论是微信的 API Key 还是支付宝的应用私钥）只能存在于受控的服务端环境中。

### 4.2 微信支付签名算法（HMAC-SHA256）

微信支付 V3 版本采用 HMAC-SHA256 签名算法，签名流程如下：

**第一步：构造签名串**

签名串按以下规则拼接：

```
HTTP请求方法\n
URL路径\n
请求时间戳\n
请求随机串\n
请求报文主体\n
```

示例：

```
POST
/v3/pay/transactions/h5
1559156665
5K8264ILTKCH16CQ2502SI8ZNMTM67VS
{"mch_id":"YOUR_MCH_ID","out_trade_no":"201503192101","description":"Image形象店-深圳腾大校区-朱老师","amount":{"total":1},"notify_url":"https://www.weixin.qq.com/wxpay/pay.php"}
```

**第二步：生成签名**

使用 APIv3 密钥对签名串进行 HMAC-SHA256 计算，得到签名值：

```python
import hmac
import hashlib

def sign(message, api_key):
    return hmac.new(
        api_key.encode('utf-8'),
        message.encode('utf-8'),
        hashlib.sha256
    ).hexdigest()
```

**第三步：构建 Authorization 头**

```
WECHATPAY2-SHA256-RSA2048 mchid="1900009191",nonce_str="5K8264ILTKCH16CQ2502SI8ZNMTM67VS",signature="...",timestamp="1559156665",serial_no="..."
```

> 💡 **提示**：微信支付官方提供了各语言的 SDK（Java、PHP、Go、Python、Node.js 等），封装了签名和验签逻辑。强烈建议直接使用官方 SDK 而非自己实现签名算法，避免引入安全隐患。

### 4.3 支付宝签名算法（RSA2）

支付宝采用 RSA2（SHA256WithRSA）签名算法，流程如下：

**第一步：组装待签名字典**

将所有请求参数按 key 的 ASCII 码升序排列，排除 sign 和 sign_type 字段。

**第二步：拼接签名字符串**

将排序后的参数按 key=value 格式用 & 连接：

```
app_id=2021001162642894&biz_content={"subject":"订单","out_trade_no":"ORDER001","total_amount":"0.01"}&charset=utf-8&format=json&method=alipay.trade.app.pay&sign_type=RSA2&timestamp=2024-03-15 10:30:00&version=1.0
```

**第三步：RSA 私钥签名**

```python
from cryptography.hazmat.primitives import hashes, serialization
from cryptography.hazmat.primitives.asymmetric import padding
import base64

def rsa_sign(data_str, private_key_pem):
    private_key = serialization.load_pem_private_key(
        private_key_pem.encode(),
        password=None
    )
    signature = private_key.sign(
        data_str.encode('utf-8'),
        padding.PKCS1v15(),
        hashes.SHA256()
    )
    return base64.b64encode(signature).decode('utf-8')
```

**第四步：将签名加入请求参数**

将生成的签名值作为 sign 参数加入请求字典中。

### 4.4 服务端回调验签流程

当支付完成后，微信/支付宝会向服务端的 notify_url 发送异步通知。服务端必须验签才能确保通知的真实性。

#### 微信支付验签

```python
import hashlib
import hmac
import json
from cryptography.hazmat.primitives import serialization, hashes
from cryptography.hazmat.primitives.asymmetric import padding, utils
from cryptography.hazmat.backends import default_backend

def verify_wechat_notification(headers, body, api_key):
    timestamp = headers.get('Wechatpay-Timestamp', '')
    nonce = headers.get('Wechatpay-Nonce', '')
    signature = headers.get('Wechatpay-Signature', '')
    serial = headers.get('Wechatpay-Serial', '')

    message = f"{timestamp}\n{nonce}\n{body}\n"

    expected_sign = hmac.new(
        api_key.encode('utf-8'),
        message.encode('utf-8'),
        hashlib.sha256
    ).hexdigest()

    return expected_sign == signature
```

#### 支付宝验签

```python
from cryptography.hazmat.primitives.asymmetric import utils
from cryptography.hazmat.primitives import hashes, serialization
from cryptography.hazmat.primitives.asymmetric import padding
import base64

def verify_alipay_notification(params, alipay_public_key):
    sign = params.pop('sign', '')
    sign_type = params.pop('sign_type', '')

    sorted_params = sorted(params.items())
    sign_string = '&'.join([f"{k}={v}" for k, v in sorted_params])

    public_key = serialization.load_pem_public_key(
        alipay_public_key.encode(),
        backend=default_backend()
    )

    try:
        public_key.verify(
            base64.b64decode(sign),
            sign_string.encode('utf-8'),
            padding.PKCS1v15(),
            hashes.SHA256()
        )
        return True
    except Exception:
        return False
```

> ⚠️ **警告**：验签失败的通知必须直接丢弃，不能作为任何业务处理的依据。攻击者可能伪造回调通知来骗取发货或充值。

### 4.5 订单状态同步机制

由于网络的不确定性，客户端 SDK 回调和服务端异步通知之间可能存在延迟和不一致。推荐的同步架构如下：

```
客户端支付完成
      │
      ▼
  SDK 回调（UI 即时反馈：处理中...）
      │
      ▼
  客户端轮询服务端订单状态接口（每 2 秒一次，最多 30 次）
      │
      ├── 收到 SUCCESS → 显示成功页
      ├── 收到 FAILED → 显示失败页，引导重试
      └── 超时未确定 → 显示"支付结果确认中"，引导用户手动刷新
```

服务端侧的状态流转：

```
CREATED（创建）→ PROCESSING（支付中）→ SUCCESS（成功）/ FAILED（失败）/ REFUNDED（已退款）
```

每个状态的变更都必须写入数据库，并记录时间戳和触发原因，便于后续对账和争议处理。

### 4.6 防重复支付与幂等性设计

支付系统中最危险的情况之一是重复支付——用户因为网络抖动多次点击支付，或者恶意脚本重复发起请求。

#### 幂等性保障措施

| 层面 | 措施 | 实现方式 |
|------|------|----------|
| 客户端 | 支付按钮防抖 | 点击后禁用按钮，直到结果返回 |
| 客户端 | 本地订单缓存 | 相同 orderId 在短时间内不重复发起 |
| 服务端 | 订单号唯一约束 | 数据库 out_trade_no 字段加 UNIQUE 索引 |
| 服务端 | 分布式锁 | Redis SETNX 保证同一订单并发处理只有一个线程执行 |
| 服务端 | 状态机保护 | 只有 CREATED 状态的订单才能进入支付流程 |
| 微信/支付宝 | 平台层面去重 | 同一 out_trade_id 只能支付一次 |

#### 分布式锁示例

```python
import redis
import time

r = redis.Redis(host='localhost', port=6379, db=0)

def process_payment_with_lock(order_id):
    lock_key = f"pay:lock:{order_id}"
    locked = r.set(lock_key, "1", nx=True, ex=30)

    if not locked:
        raise Exception("订单正在处理中，请勿重复提交")

    try:
        order = get_order(order_id)
        if order.status != "CREATED":
            raise Exception(f"订单状态不允许支付: {order.status}")

        call_unified_order_api(order)
        update_order_status(order_id, "PROCESSING")
    finally:
        r.delete(lock_key)
```

---

## 五、统一支付管理器架构

### 5.1 PaymentService 协议设计

为了在 App 中统一管理多种支付渠道，我们需要定义一个抽象的支付服务协议：

```swift
import Foundation

public enum PaymentChannel: String, CaseIterable, Identifiable {
    case wechat = "wechat"
    case alipay = "alipay"
    case appleIAP = "apple_iap"

    public var id: String { rawValue }

    public var displayName: String {
        switch self {
        case .wechat: return "微信支付"
        case .alipay: return "支付宝"
        case .appleIAP: return "Apple Pay"
        }
    }

    public var iconName: String {
        switch self {
        case .wechat: return "message.fill"
        case .alipay: return "creditcard"
        case .appleIAP: return "applelogo"
        }
    }

    public var brandColor: Color {
        switch self {
        case .wechat: return Color(red: 0.07, green: 0.62, blue: 0.46)
        case .alipay: return Color(red: 0.16, green: 0.54, blue: 0.93)
        case .appleIAP: return .black
        }
    }
}

public enum PaymentResult: Equatable {
    case success(transactionId: String)
    case failed(error: String)
    case cancelled
    case pending

    public var isSuccess: Bool {
        if case .success = self { return true }
        return false
    }
}

public protocol PaymentServiceProtocol: AnyObject {
    var channel: PaymentChannel { get }
    var isAvailable: Bool { get }
    var isProcessing: Bool { get }

    func pay(orderId: String, amount: Decimal) async throws -> PaymentResult
    func checkAvailability() -> Bool
}
```

### 5.2 微信支付 / 支付宝 / Apple IAP 统一接口

基于上述协议，为每个支付渠道实现具体的服务类：

```swift
import StoreKit

final class WechatPaymentService: PaymentServiceProtocol {
    let channel: PaymentChannel = .wechat
    @Published var isProcessing = false

    var isAvailable: Bool {
        WXApi.isWXAppInstalled()
    }

    func checkAvailability() -> Bool {
        isAvailable
    }

    func pay(orderId: String, amount: Decimal) async throws -> PaymentResult {
        guard isAvailable else {
            return .failed(error: "未安装微信")
        }

        isProcessing = true
        defer { isProcessing = false }

        let orderInfo = try await fetchWechatPrepay(orderId: orderId)
        try await WechatPaymentManager.shared.pay(orderInfo: orderInfo)
        return .success(transactionId: orderId)
    }

    private func fetchWechatPrepay(orderId: String) async throws -> WechatOrderInfo {
        let url = URL(string: "https://api.yourserver.com/pay/wechat/prepay")!
        var request = URLRequest(url: url)
        request.httpMethod = "POST"
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")
        request.httpBody = try? JSONSerialization.data(withJSONObject: ["orderId": orderId])

        let (data, _) = try await URLSession.shared.data(for: request)
        let json = try JSONSerialization.jsonObject(with: data) as? [String: Any]
        guard let d = json?["data"] as? [String: String] else {
            throw NSError(domain: "WechatPay", code: -1, userInfo: [NSLocalizedDescriptionKey: "解析失败"])
        }
        return WechatOrderInfo(
            partnerId: d["partnerId"] ?? "",
            prepayId: d["prepayId"] ?? "",
            nonceStr: d["nonceStr"] ?? "",
            timeStamp: d["timeStamp"] ?? "",
            package: d["package"] ?? "Sign=WXPay",
            sign: d["sign"] ?? ""
        )
    }
}

final class AlipayPaymentService: PaymentServiceProtocol {
    let channel: PaymentChannel = .alipay
    @Published var isProcessing = false

    var isAvailable: Bool {
        true
    }

    func checkAvailability() -> Bool {
        true
    }

    func pay(orderId: String, amount: Decimal) async throws -> PaymentResult {
        isProcessing = true
        defer { isProcessing = false }

        let orderString = try await fetchAlipayOrder(orderId: orderId)
        try await AlipayPaymentManager.shared.pay(orderString: orderString)
        return .success(transactionId: orderId)
    }

    private func fetchAlipayOrder(orderId: String) async throws -> String {
        let url = URL(string: "https://api.yourserver.com/pay/alipay/order")!
        var request = URLRequest(url: url)
        request.httpMethod = "POST"
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")
        request.httpBody = try? JSONSerialization.data(withJSONObject: ["orderId": orderId])

        let (data, _) = try await URLSession.shared.data(for: request)
        let json = try JSONSerialization.jsonObject(with: data) as? [String: Any]
        guard let orderString = json?["data"] as? String else {
            throw NSError(domain: "Alipay", code: -1, userInfo: [NSLocalizedDescriptionKey: "解析失败"])
        }
        return orderString
    }
}

final class AppleIAPPaymentService: PaymentServiceProtocol {
    let channel: PaymentChannel = .appleIAP
    @Published var isProcessing = false

    var isAvailable: Bool {
        true
    }

    func checkAvailability() -> Bool {
        true
    }

    func pay(orderId: String, amount: Decimal) async throws -> PaymentResult {
        isProcessing = true
        defer { isProcessing = false }

        guard let product = try await Product.products(for: [orderId]).first else {
            return .failed(error: "商品不存在")
        }

        let result = try await product.purchase()

        switch result {
        case .success(let verification):
            switch verification {
            case .verified(let transaction):
                await transaction.finish()
                return .success(transactionId: transaction.transactionID)
            case .unverified(_, let error):
                return .failed(error: error.localizedDescription)
            }
        case .userCancelled:
            return .cancelled
        case .pending:
            return .pending
        @unknown default:
            return .failed(error: "未知错误")
        }
    }
}
```

### 5.3 支付方式选择策略

根据不同的业务场景，自动推荐最优的支付方式：

```swift
final class PaymentRouter {

    static func availableChannels(for scenario: PaymentScenario) -> [PaymentChannel] {
        switch scenario {
        case .physicalGoods:
            var channels: [PaymentChannel] = [.alipay, .wechat]
            if WechatPaymentService().isAvailable == false {
                channels.removeAll { $0 == .wechat }
            }
            return channels
        case .digitalContent:
            return [.appleIAP]
        case .donation:
            return [.alipay, .wechat]
        }
    }

    static func recommendedChannel(from available: [PaymentChannel]) -> PaymentChannel? {
        available.first
    }
}

enum PaymentScenario {
    case physicalGoods
    case digitalContent
    case donation
}
```

### 5.4 订单管理系统设计

一个健壮的订单管理系统是支付功能的基础设施：

```swift
struct OrderModel: Identifiable, Codable {
    let id: String
    let orderId: String
    let channel: PaymentChannel
    let amount: Decimal
    let subject: String
    var status: OrderStatus
    var createdAt: Date
    var paidAt: Date?
    var transactionId: String?

    enum OrderStatus: String, Codable {
        case created = "created"
        case processing = "processing"
        case paid = "paid"
        case failed = "failed"
        case refunded = "refunded"
    }
}

@MainActor
final class OrderManager: ObservableObject {
    static let shared = OrderManager()
    @Published var orders: [OrderModel] = []
    @Published var currentOrder: OrderModel?

    func createOrder(channel: PaymentChannel, amount: Decimal, subject: String) async throws -> OrderModel {
        let order = OrderModel(
            id: UUID().uuidString,
            orderId: "ORD\(Int(Date().timeIntervalSince1970))",
            channel: channel,
            amount: amount,
            subject: subject,
            status: .created,
            createdAt: Date(),
            paidAt: nil,
            transactionId: nil
        )
        orders.append(order)
        currentOrder = order
        return order
    }

    func updateOrderStatus(orderId: String, status: OrderModel.OrderStatus, transactionId: String? = nil) {
        if let index = orders.firstIndex(where: { $0.orderId == orderId }) {
            orders[index].status = status
            orders[index].transactionId = transactionId
            orders[index].paidAt = (status == .paid) ? Date() : nil
        }
    }

    func pollOrderStatus(orderId: String) async throws -> OrderModel.OrderStatus {
        guard let url = URL(string: "https://api.yourserver.com/order/\(orderId)/status") else {
            throw URLError(.badURL)
        }
        let (data, _) = try await URLSession.shared.data(from: url)
        let json = try JSONSerialization.jsonObject(with: data) as? [String: Any]
        let statusStr = json?["status"] as? String ?? "created"
        return OrderModel.OrderStatus(rawValue: statusStr) ?? .created
    }
}
```

### 5.5 SwiftUI 支付选择界面

```swift
struct PaymentSelectionView: View {
    let amount: Decimal
    let subject: String
    let scenario: PaymentScenario
    @StateObject private var orderManager = OrderManager.shared
    @StateObject private var router = PaymentRouterProxy()
    @State private var selectedChannel: PaymentChannel?
    @State private var showResult = false
    @State private var paymentResult: PaymentResult?
    @State private var isLoading = false

    var body: some View {
        ScrollView {
            VStack(spacing: 20) {
                orderSummarySection
                channelSelectionSection
                confirmButton
            }
            .padding()
        }
        .navigationTitle("确认支付")
        .navigationBarTitleDisplayMode(.inline)
        .alert("支付结果", isPresented: $showResult) {
            Button("完成", role: .cancel) {}
        } message: {
            Text(resultMessage)
        }
    }

    var orderSummarySection: some View {
        VStack(spacing: 12) {
            Text("¥\(amount)")
                .font(.system(size: 36, weight: .bold))
            Text(subject)
                .font(.subheadline)
                .foregroundColor(.secondary)
        }
        .frame(maxWidth: .infinity)
        .padding(.vertical, 24)
        .background(Color(.systemGroupedBackground))
        .cornerRadius(12)
    }

    var channelSelectionSection: some View {
        VStack(alignment: .leading, spacing: 12) {
            Text("选择支付方式")
                .font(.headline)

            ForEach(router.availableChannels) { channel in
                ChannelRow(
                    channel: channel,
                    isSelected: selectedChannel == channel
                ) {
                    selectedChannel = channel
                }
            }
        }
    }

    var confirmButton: some View {
        Button(action: handlePayment) {
            Text(isLoading ? "支付中..." : "确认支付 ¥\(amount)")
                .font(.headline)
                .foregroundColor(.white)
                .frame(maxWidth: .infinity)
                .padding(.vertical, 14)
                .background(selectedChannel != nil ? Color.blue : Color.gray)
                .cornerRadius(10)
        }
        .disabled(selectedChannel == nil || isLoading)
    }

    var resultMessage: String {
        guard let result = paymentResult else { return "" }
        switch result {
        case .success: return "支付成功！"
        case .failed(let e): return "支付失败：\(e)"
        case .cancelled: return "您已取消支付"
        case .pending: return "支付处理中，请稍后查看"
        }
    }

    func handlePayment() {
        guard let channel = selectedChannel else { return }
        isLoading = true

        Task {
            do {
                let order = try await orderManager.createOrder(
                    channel: channel,
                    amount: amount,
                    subject: subject
                )

                let service: PaymentServiceProtocol = switch channel {
                case .wechat: WechatPaymentService()
                case .alipay: AlipayPaymentService()
                case .appleIAP: AppleIAPPaymentService()
                }

                let result = try await service.pay(orderId: order.orderId, amount: amount)
                paymentResult = result
                showResult = true
            } catch {
                paymentResult = .failed(error: error.localizedDescription)
                showResult = true
            }
            isLoading = false
        }
    }
}

struct ChannelRow: View {
    let channel: PaymentChannel
    let isSelected: Bool
    let onTap: () -> Void

    var body: some View {
        Button(action: onTap) {
            HStack {
                Image(systemName: channel.iconName)
                    .foregroundColor(channel.brandColor)
                    .frame(width: 28)

                Text(channel.displayName)
                    .foregroundColor(.primary)

                Spacer()

                if isSelected {
                    Image(systemName: "checkmark.circle.fill")
                        .foregroundColor(.blue)
                }
            }
            .padding(.vertical, 12)
            .padding(.horizontal, 16)
            .background(isSelected ? Color.blue.opacity(0.08) : Color.clear)
            .cornerRadius(8)
            .overlay(
                RoundedRectangle(cornerRadius: 8)
                    .stroke(isSelected ? Color.blue : Color(.separator), lineWidth: isSelected ? 2 : 0.5)
            )
        }
        .buttonStyle(.plain)
    }
}

final class PaymentRouterProxy: ObservableObject {
    let availableChannels: [PaymentChannel]

    init(scenario: PaymentScenario = .physicalGoods) {
        self.availableChannels = PaymentRouter.availableChannels(for: scenario)
    }
}
```

### 5.6 支付状态机设计

支付过程是一个典型的有限状态机，明确定义状态转换规则可以有效避免边界情况的混乱：

```
                    ┌──────────┐
                    │  CREATED │ ◀──── 创建订单
                    └────┬─────┘
                         │ 发起支付
                         ▼
                    ┌──────────────┐
          ┌────────▶│  PROCESSING  │◀────────┐
          │         └──────┬───────┘         │
          │                │                 │
          │    ┌───────────┼───────────┐     │
          │    ▼           ▼           ▼     │
          │ ┌───────┐  ┌───────┐  ┌────────┐ │
          │ │ PAID  │  │FAILED │  │PENDING │ │
          │ └───┬───┘  └───────┘  └────┬───┘ │
          │     │                     │      │
          │     ▼                     ▼      │
          │ ┌──────────┐         ┌────────┐  │
          └─│ REFUNDED  │         │  PAID  │──┘
            └──────────┘         └────────┘
                                  (异步确认)
```

| 当前状态 | 允许的转换 | 触发条件 |
|----------|-----------|----------|
| CREATED | → PROCESSING | 用户发起支付 |
| PROCESSING | → PAID | 服务端确认支付成功 |
| PROCESSING | → FAILED | 支付失败/超时/用户取消 |
| PROCESSING | → PENDING | Apple IAP 待 Family Sharing 审批 |
| PAID | → REFUNDED | 用户申请退款成功 |
| PENDING | → PAID | 家庭共享审批通过 |

---

## 六、Apple 审核合规要点

### 6.1 3.1.1 条款深度解读

Apple《App Store 审核指南》3.1.1 条款是关于应用内购买的核心条款，完整解读如下：

**原文核心要求**：

> If you want to unlock features or functionality within your app, (by way of example: subscriptions, in-game currencies, game levels, access to premium content, or unlocking a full version), you must use In-App Purchase. Apps may not include buttons, external links, or other calls to action that direct customers to purchasing mechanisms other than In-App Purchase.

**逐句解读**：

| 条款原文 | 解读 | 违规示例 |
|----------|------|----------|
| unlock features or functionality within your app | App 内解锁任何功能 | 去广告、解锁高级版、VIP 权限 |
| subscriptions | 所有形式的订阅 | 月度/年度会员、内容订阅 |
| in-game currencies | 游戏内的虚拟货币 | 金币、钻石、点券 |
| game levels | 游戏关卡 | 买关卡、买道具 |
| access to premium content | 优质内容访问权 | 视频、音乐、文章、课程 |
| unlocking a full version | 解锁完整版 | Lite→Pro 升级 |
| must use In-App Purchase | **必须**使用 IAP | 没有任何例外（除非符合 Reader Rule） |
| no buttons, external links, or other calls to action | 不能有任何引导 | 包括文字链接、二维码、客服指引 |

### 6.2 数字商品 vs 实体商品的判断标准

这是审核中最常见的争议点——如何界定"数字商品"和"实体商品"：

| 商品类型 | 归类 | 判断依据 | 是否允许第三方支付 |
|----------|------|----------|------------------|
| 电子书 / 电子杂志 | 数字商品 | 纯数字化交付，无实体对应 | ❌ 必须走 IAP |
| 游戏道具 / 虚拟货币 | 数字商品 | 仅在 App 内使用 | ❌ 必须走 IAP |
| 在线课程视频 | 数字商品 | 纯线上内容 | ❌ 必须走 IAP |
| 去广告 / VIP 会员 | 数字商品 | 解锁 App 内功能 | ❌ 必须走 IAP |
| 实体商品（衣服/食品/电子产品） | 实物商品 | 需要物流配送 | ✅ 可用第三方支付 |
| O2O 服务（打车/外卖/家政） | 服务 | 线上付款线下履约 | ✅ 可用第三方支付 |
| 电影票 / 机票 / 酒店预订 | 服务凭证 | 最终消费在线下发生 | ✅ 可用第三方支付 |
| 充值话费 / 加油卡 | 实体服务关联 | 充值后在电信/石油系统消费 | ✅ 通常允许 |
| 知识付费（直播课+社群） | 边界模糊 | 如果含线下活动则有机会 | ⚠️ 需谨慎设计 |
| 读者打赏 | 特殊例外 | Reader Rule 适用 | ⚠️ 需提前沟通 |

> 💡 **Reader Rule**：如果你的 App 是纯粹的阅读器（如电子书阅读器、新闻聚合器），且内容完全由用户提供（UGC），那么用户对内容创作者的直接打赏/捐赠可以不走 IAP。但这个规则的适用范围非常窄，Apple 会严格审查。

### 6.3 常见被拒场景与应对

| 被拒场景 | 具体表现 | 应对策略 |
|----------|----------|----------|
| 引导外链支付 | App 内显示"前往网页支付享受优惠" | 删除所有外部支付相关文案和链接 |
| 虚拟货币第三方支付 | 游戏内用微信/支付宝买金币 | 全部改为 IAP 购买 |
| 订阅绕过 IAP | 显示"微信公众号订阅更便宜" | 删除价格对比，订阅只能走 IAP |
| 二维码收款 | App 内展示个人/商家收款码 | 移除二维码，改为 IAP |
| 客服暗示 | 客服回复"您可以淘宝购买" | 规范客服话术，严禁提及第三方支付 |
| 代码残留 | 审核员在代码中发现 alipay 相关字符串 | 使用条件编译彻底移除审核版本的第三方支付代码 |
| UI 遗留 | 审核模式下仍能看到支付宝图标 | 服务端下发开关，审核时关闭第三方支付入口 |
| 跨平台价差 | iOS 版比 Android 版贵很多 | 保持价格一致，或合理说明差异原因（Apple 抽成） |

### 6.4 审核时的支付功能处理

如果你确实需要在 App 中保留第三方支付功能（用于合规的实物/服务交易），以下是一些经过实践验证的策略：

**策略一：服务端开关控制**

```swift
struct FeatureFlag {
    static let thirdPartyPaymentEnabled: Bool = {
        #if DEBUG
        return true
        #else
        return Bundle.main.infoDictionary?["THIRD_PARTY_PAY_ENABLED"] as? String == "true"
        #endif
    }()
}
```

审核期间在服务端关闭开关，上线后开启。但这种方式有风险——Apple 可能会在审核中测试多种路径。

**策略二：条件编译**

```swift
#if APPSTORE_REVIEW
let availableChannels: [PaymentChannel] = [.appleIAP]
#else
let availableChannels: [PaymentChannel] = [.alipay, .wechat, .appleIAP]
#endif
```

通过 Build Configuration 区分审核构建和正式构建。更隐蔽但也更有风险。

**策略三：最稳妥的做法——严格按业务分离**

- 数字内容模块：只有 IAP，没有任何第三方支付的痕迹
- 电商/服务模块：独立 Tab 或独立入口，清晰标注"实物商品"/"服务预约"
- 两个模块之间没有交叉引导

> ⚠️ **警告**：任何试图欺骗审核的行为（包括上述策略一和策略二）都违反了 Apple Developer Program License Agreement。如果被发现，可能导致 App 被下架甚至开发者账号被封禁。本节仅供了解风险，不建议实际使用。

### 6.5 合规案例分享

**案例 A：知识付费 App（合规通过）**

某在线教育 App 同时销售录播课程（数字内容）和线下训练营（含实体物料）。

- 录播课程 → StoreKit IAP 购买
- 线下训练营 → 微信/支付宝支付（含教材邮寄）
- 两套系统完全隔离，UI 上明确区分「线上课程」和「线下集训营」
- 审核说明文档中详细列出了每种商品的交付形式
- **结果**：一次通过审核

**案例 B：电商 App（被拒后整改通过）**

某电商 App 在商品详情页同时展示了"App 内购买"和"微信支付更优惠"两个按钮。

- **被拒理由**：违反 3.1.1，引导用户使用外部支付
- **整改措施**：删除所有第三方支付入口，全部改为 IAP
- **后续调整**：向 Apple 申请 Reader Rule 例外被拒后，接受 IAP-only 方案
- **结果**：整改后通过

**案例 C：社交 App 打赏功能（经沟通后通过）**

某社交 App 的用户可以对发布的内容进行打赏。

- 初版提交：使用微信支付打赏 → 被拒（3.1.1）
- 申诉：引用 Reader Rule，强调内容为 UGC，打赏直接给创作者
- Apple 要求补充材料：证明内容完全由用户提供、平台不从打赏中抽成
- 补充证明后：通过审核
- **关键点**：打赏功能必须 100% 传递给内容创作者，平台不得从中获利

> 💡 **提示**：如果你的业务处于合规边界地带，建议在开发前先通过 [App Review Contact](https://developer.apple.com/contact/app-store/) 提交预审咨询，获得书面确认后再投入开发。这能节省大量返工成本。

---

## 小结

| 本章要点 | 一句话总结 |
|----------|-----------|
| 国内支付生态 | 微信支付 + 支付宝覆盖 95%+ 用户，两者必须同时支持 |
| IAP vs 第三方支付 | 数字内容必须走 IAP，实物/服务可用第三方支付 |
| 3.1.1 条款 | Apple 审核的红线，数字商品解锁功能严禁绕过 IAP |
| 微信支付集成 | SDK + Universal Link + 服务端签名 + 异步通知，四要素缺一不可 |
| 支付宝集成 | RSA2 签名 + URL Scheme 回调 + orderString 拉起支付，相对简单 |
| 签名安全 | 所有签名必须在服务端完成，密钥绝不能出现在客户端代码中 |
| 验签机制 | 服务端收到异步通知后必须验签，验签失败直接丢弃 |
| 幂等设计 | 分布式锁 + 订单状态机 + 唯一索引，三重保障防重复支付 |
| 统一支付管理器 | PaymentService 协议 + 多渠道实现 + 路由策略，一套代码管所有支付 |
| 审核合规 | 严格区分数字商品和实物商品，不存侥幸心理，被拒成本远高于合规成本 |

> 💡 **最后的建议**：支付功能的复杂性往往被低估。从 SDK 集成到签名验签，从状态管理到审核合规，每一个环节都可能成为坑。建议先跑通最小闭环（一个渠道 + 一个商品类型），再逐步扩展。记住——支付关系到用户的真金白银，稳定性和安全性永远排在第一位。

← [内购与订阅模式实战](./内购与订阅模式实战.md) | [StoreKit 测试与收据验证](./StoreKit测试与收据验证.md) →
