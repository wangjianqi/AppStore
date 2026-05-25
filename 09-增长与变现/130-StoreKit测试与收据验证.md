# 130-StoreKit 测试与收据验证

## 本章目标

- 理解内购测试为什么是独立于开发之外的关键话题
- 掌握三种测试环境（Xcode / Sandbox / Production）的特点与适用场景
- 学会使用 StoreKit Configuration File 和 SKTestSession 进行本地测试
- 了解 Sandbox 沙盒测试的完整流程与常见坑
- 掌握 App Store Server API V2 和 Server Notifications V2 的集成方式
- 深入理解收据验证的本地与服务端两种方案
- 学会管理订阅生命周期中的各种状态变化
- 能够编写内购自动化测试并在 CI 中运行

---

## 一、内购测试概述

### 1.1 为什么内购测试是独立话题

想象你开了一家餐厅——菜谱写好了（代码实现了），但开张之前必须经过卫生检查（审核）、试营业（沙盒测试）、正式营业（生产环境）。每一步的规则都不同，你不能用试营业的方式去做正式营业的事。

内购测试之所以是独立话题，原因有三：

| 原因 | 说明 |
|------|------|
| **涉及真实金钱** | 内购一旦出错，用户被多扣钱，后果远超普通 Bug |
| **苹果审核严格** | 内购相关被拒是 App Store 审核被拒的重灾区 |
| **环境差异大** | 三种测试环境行为各异，代码在不同环境下表现不同 |

> ⚠️ **血泪教训**：很多开发者在 Sandbox 里一切正常，上线后才发现收据验证逻辑有漏洞，导致用户付费后无法解锁功能，差评如潮。

### 1.2 三种测试环境对比

| 特性 | Xcode StoreKit Testing | Sandbox 沙盒环境 | Production 生产环境 |
|------|----------------------|------------------|-------------------|
| **运行位置** | Xcode 模拟器/真机 | TestFlight 或沙盒账号真机 | App Store 正式用户 |
| **需要网络** | ❌ 不需要 | ✅ 需要 | ✅ 需要 |
| **需要 App Store Connect 配置** | ❌ 不需要 | ✅ 需要 | ✅ 需要 |
| **需要沙盒账号** | ❌ 不需要 | ✅ 需要 | ❌（真实 Apple ID） |
| **支付速度** | 即时 | 几秒 | 几秒 |
| **订阅加速** | ✅ 大幅加速 | ✅ 加速（见下表） | ❌ 真实时间 |
| **退款测试** | ✅ 支持 | ✅ 支持 | 用户主动申请 |
| **收据验证** | 本地验证 | 沙盒验证服务器 | 生产验证服务器 |
| **适合阶段** | 开发期 | 上线前测试 | 正式运营 |
| **可自动化** | ✅ 完全支持 | ❌ 手动为主 | ❌ 不可测试 |

> 💡 **最佳实践**：开发阶段用 Xcode StoreKit Testing 快速迭代，上线前用 Sandbox 做完整回归，上线后用 App Store Server API 监控真实交易。

### 1.3 订阅时间加速对照

Sandbox 环境下，订阅周期会被大幅压缩，方便测试续期逻辑：

| 实际周期 | Sandbox 加速后 | Xcode StoreKit Testing |
|----------|---------------|----------------------|
| 1 周 | 3 分钟 | 可自定义 |
| 1 个月 | 5 分钟 | 可自定义 |
| 2 个月 | 10 分钟 | 可自定义 |
| 3 个月 | 15 分钟 | 可自定义 |
| 6 个月 | 30 分钟 | 可自定义 |
| 1 年 | 1 小时 | 可自定义 |

---

## 二、Xcode StoreKit Testing

### 2.1 StoreKit Configuration File 创建

StoreKit Configuration File 是 Xcode 提供的本地内购测试配置文件，就像一个"模拟收银台"——你可以在里面定义商品、价格、订阅组，完全不依赖 App Store Connect。

**创建步骤：**

1. Xcode → File → New → File
2. 选择 **StoreKit Configuration File**
3. 命名（如 `Products.storekit`）并保存到项目

**配置商品：**

点击配置文件底部的 `+` 按钮，可以添加：

| 商品类型 | 说明 |
|----------|------|
| Consumable In-App Purchase | 消耗型内购 |
| Non-Consumable In-App Purchase | 非消耗型内购 |
| Auto-Renewable Subscription | 自动续期订阅 |
| Non-Renewing Subscription | 非续期订阅 |

每个商品需要填写：
- Reference Name（引用名称）
- Product ID（产品标识符，需与代码中一致）
- Price（价格）

> 💡 **关键设置**：在 Scheme 中选择 Edit Scheme → Run → Options → StoreKit Configuration，选择你的 `.storekit` 文件，这样运行 App 时才会使用本地测试环境。

### 2.2 SKTestSession API

`SKTestSession` 是 Xcode 12+ 提供的测试 API，可以在测试代码中程序化控制 StoreKit 行为——就像你拥有收银台的遥控器，可以随时模拟购买、退款、续期。

```swift
import StoreKitTest

@MainActor
func testPurchaseFlow() async throws {
    let session = try SKTestSession(configurationFileNamed: "Products")

    session.disableDialogs = true
    session.clearTransactions()

    let product = try await Product.products(for: ["com.example.premium"]).first!
    let result = try await product.purchase()

    switch result {
    case .success(let verification):
        switch verification {
        case .unverified:
            XCTFail("交易验证失败")
        case .verified(let transaction):
            XCTAssertEqual(transaction.productID, "com.example.premium")
            XCTAssertEqual(transaction.purchasedAmount, 6_00)
            await transaction.finish()
        }
    case .userCancelled:
        XCTFail("用户不应取消")
    case .pending:
        XCTFail("交易不应待定")
    @unknown default:
        break
    }
}
```

### 2.3 测试购买与退款

```swift
@MainActor
func testRefundFlow() async throws {
    let session = try SKTestSession(configurationFileNamed: "Products")
    session.disableDialogs = true
    session.clearTransactions()

    let product = try await Product.products(for: ["com.example.premium"]).first!
    let result = try await product.purchase()

    case let .success(verification) = result:
        if case .verified(let transaction) = verification {
            await transaction.finish()
        }

    let transactions = session.allTransactions()
    XCTAssertEqual(transactions.count, 1)

    try session.refundTransaction(transactions[0].identifier)

    let refunded = session.allTransactions()
    XCTAssertEqual(refunded[0].revocationDate, nil)
}
```

### 2.4 测试订阅续期

```swift
@MainActor
func testSubscriptionRenewal() async throws {
    let session = try SKTestSession(configurationFileNamed: "Products")
    session.disableDialogs = true
    session.clearTransactions()

    let product = try await Product.products(for: ["com.example.monthly_sub"]).first!
    let result = try await product.purchase()

    if case .success(let verification) = result,
       case .verified(let transaction) = verification {
        await transaction.finish()
    }

    try session.advanceSubscriptionRenewal(by: 1)

    for await verification in Transaction.updates {
        if case .verified(let transaction) = verification {
            XCTAssertTrue(transaction.productID == "com.example.monthly_sub")
            await transaction.finish()
            break
        }
    }
}
```

### 2.5 测试优惠兑换

```swift
@MainActor
func testOfferRedemption() async throws {
    let session = try SKTestSession(configurationFileNamed: "Products")
    session.disableDialogs = true

    let discount = Product.SubscriptionOffer(
        id: "intro_offer",
        type: .introductory,
        paymentMode: .freeTrial,
        price: 0,
        period: .init(value: 7, unit: .day)
    )

    let product = try await Product.products(for: ["com.example.monthly_sub"]).first!
    let result = try await product.purchase(options: [])

    if case .success(let verification) = result,
       case .verified(let transaction) = verification {
        XCTAssertEqual(transaction.offer?.type, .introductory)
        await transaction.finish()
    }
}
```

> 💡 **Xcode 14+** 支持在 StoreKit Configuration File 中直接配置优惠信息（Introductory Offer、Promotional Offer），无需在 App Store Connect 中设置即可本地测试。

---

## 三、Sandbox 沙盒测试环境

### 3.1 沙盒账号创建

沙盒账号就像"演习用的假身份证"——它模拟真实用户购买，但不会真正扣款。

**创建步骤：**

1. 登录 [App Store Connect](https://appstoreconnect.apple.com)
2. 进入"用户和访问"→"沙盒"→"测试账户"
3. 点击 `+` 创建新测试账户
4. 填写信息（邮箱不能是真实 Apple ID，建议用 `test+xxx@example.com` 格式）

> ⚠️ **重要限制**：
> - 每个沙盒账号只能在一个 App Store 区域使用
> - 沙盒账号创建后**不能修改地区**
> - 一个 Apple ID 最多创建 100 个沙盒账号
> - 沙盒账号的购买不会真实扣款

### 3.2 Sandbox vs Production 差异

| 差异点 | Sandbox | Production |
|--------|---------|------------|
| 支付方式 | 不扣款，无支付弹窗确认 | 真实扣款，Face ID/密码确认 |
| 订阅续期 | 加速（1个月=5分钟） | 真实时间 |
| 自动续期 | 最多续期 6 次后自动取消 | 按用户设置持续续期 |
| 退款 | 可在设置中模拟退款 | 用户通过 Apple 支持申请 |
| 收据验证 URL | `https://sandbox.itunes.apple.com/verifyReceipt` | `https://buy.itunes.apple.com/verifyReceipt` |
| App Store Connect 显示 | 不显示 | 显示在销售和趋势中 |
| 订阅管理 | 设置 → Sandbox Account | 设置 → Apple ID → 订阅 |

### 3.3 沙盒测试流程

```
1. 在 App Store Connect 创建内购产品
        ↓
2. 创建沙盒测试账号
        ↓
3. 设备上登录沙盒账号（设置 → App Store → Sandbox Account）
        ↓
4. 运行 App（Xcode 真机 或 TestFlight）
        ↓
5. 触发购买流程
        ↓
6. 验证收据（使用沙盒验证 URL）
        ↓
7. 检查交易状态
        ↓
8. 测试退款、续期等场景
```

### 3.4 常见沙盒问题

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| "Cannot connect to iTunes Store" | 沙盒账号未登录或过期 | 重新登录沙盒账号 |
| 购买弹窗显示"Environment: Sandbox" | 正常现象，说明在沙盒环境 | 无需处理 |
| 同一商品重复购买失败 | 沙盒环境下非消耗型只能买一次 | 在设备上"恢复购买"或换沙盒账号 |
| 订阅不续期 | 沙盒最多续期 6 次 | 换新沙盒账号重新测试 |
| 收据验证返回 21002 | 收据格式错误 | 确保使用 Base64 编码原始数据 |
| "You've already purchased this" | 非消耗型重复购买 | 调用恢复购买 API |
| 沙盒账号被锁定 | 多次切换地区或异常操作 | 创建新沙盒账号 |

> ⚠️ **沙盒账号一旦在中国区创建，无法切换到美区。** 如果需要测试不同地区的定价，必须为每个地区创建单独的沙盒账号。

---

## 四、App Store Server API

### 4.1 V2 接口概述

App Store Server API V2 是苹果在 2021 年推出的服务端 API，就像一个"后台管理台"——你不需要依赖客户端上报，可以主动查询和管理用户的交易状态。

| 接口 | 用途 | 方法 |
|------|------|------|
| `GET /inApps/v1/transactions/{transactionId}` | 查询交易历史 | GET |
| `GET /inApps/v1/subscriptions/{subscriptionGroupId}` | 查询订阅状态 | GET |
| `GET /inApps/v1/refund/lookup` | 查询用户退款记录 | GET |
| `POST /inApps/v1/notifications/test` | 发送测试通知 | POST |
| `GET /inApps/v1/history/{originalTransactionId}` | 获取交易历史 | GET |

### 4.2 JWT 认证

调用 Server API 需要使用 JWT（JSON Web Token）认证，这就像进入办公楼的门禁卡——每次请求都要带上这个卡才能通行。

```swift
import CryptoKit
import Foundation

struct AppStoreServerAPI {
    let keyID: String
    let issuerID: String
    let bundleID: String
    let privateKey: P256.Signing.PrivateKey

    func generateJWT() throws -> String {
        let header: [String: Any] = [
            "alg": "ES256",
            "kid": keyID,
            "typ": "JWT"
        ]

        let now = Int(Date().timeIntervalSince1970)
        let payload: [String: Any] = [
            "iss": issuerID,
            "iat": now,
            "exp": now + 3600,
            "aud": "appstoreconnect-v1",
            "bid": bundleID
        ]

        let headerData = try JSONSerialization.data(withJSONObject: header)
        let payloadData = try JSONSerialization.data(withJSONObject: payload)

        let headerB64 = headerData.base64EncodedString()
        let payloadB64 = payloadData.base64EncodedString()
        let unsignedToken = "\(headerB64).\(payloadB64)"

        let signature = try privateKey.signature(for: unsignedToken.data(using: .utf8)!)
        let signatureB64 = signature.rawRepresentation.base64EncodedString()

        return "\(unsignedToken).\(signatureB64)"
    }
}
```

> ⚠️ **安全提醒**：私钥（.p8 文件）必须保存在服务端，**绝不能**放入客户端代码或提交到 Git 仓库。

### 4.3 查询订阅状态

```swift
extension AppStoreServerAPI {
    func checkSubscriptionStatus(
        originalTransactionId: String
    ) async throws -> SubscriptionStatus {
        let jwt = try generateJWT()
        var request = URLRequest(
            url: URL(string: "https://api.appstoreconnect.apple.com/inApps/v1/subscriptions/\(originalTransactionId)")!
        )
        request.setValue("Bearer \(jwt)", forHTTPHeaderField: "Authorization")

        let (data, response) = try await URLSession.shared.data(for: request)

        guard let httpResponse = response as? HTTPURLResponse,
              httpResponse.statusCode == 200 else {
            throw ServerAPIError.invalidResponse
        }

        let decoded = try JSONDecoder().decode(
            SubscriptionResponse.self,
            from: data
        )
        return decoded.data.first!.lastTransactions.first!
    }
}
```

### 4.4 App Store Server Notifications V2

Server Notifications V2 是苹果主动推送给你的服务端的事件通知，就像快递到达后的短信提醒——你不需要反复去查，苹果会主动告诉你发生了什么。

| 通知类型 | 说明 | 触发时机 |
|----------|------|----------|
| `DID_RENEW` | 订阅续期成功 | 自动续期扣款成功 |
| `DID_FAIL_TO_RENEW` | 续期失败 | 扣款失败（余额不足等） |
| `DID_CHANGE_RENEWAL_STATUS` | 续期状态变更 | 用户关闭/开启自动续期 |
| `REFUND` | 退款 | 用户申请退款成功 |
| `REVOKE` | 撤销 | 家庭共享成员失去访问权 |
| `DID_CHANGE_RENEWAL_PREF` | 续期偏好变更 | 用户切换订阅计划 |
| `CONSUMPTION_REQUEST` | 消费请求 | 用户申请退款，苹果征求开发者意见 |

**通知处理流程：**

```
苹果服务器 → HTTPS POST → 你的服务端 /webhooks/appstore
        ↓
1. 验证 JWT 签名（确保来自苹果）
        ↓
2. 解码通知负载
        ↓
3. 根据 notificationType 分发处理
        ↓
4. 更新数据库中的用户订阅状态
        ↓
5. 返回 HTTP 200 给苹果（确认接收）
```

> 💡 **V2 通知使用 JWT 签名**，不再像 V1 那样需要通过收据验证来确认真实性，安全性大幅提升。苹果会在收不到 200 响应时重试，最多重试 3 次。

---

## 五、收据验证 Receipt Validation

### 5.1 本地验证 vs 服务端验证

就像验证钞票真假——你可以自己看水印（本地验证），也可以送到银行用专业设备检测（服务端验证）。

| 对比项 | 本地验证 | 服务端验证 |
|--------|----------|------------|
| 安全性 | ⭐⭐ 较低，可被越狱设备绕过 | ⭐⭐⭐⭐⭐ 高，服务端可控 |
| 实现复杂度 | 中等（ASN.1 解析） | 较高（需要后端服务） |
| 响应速度 | 快（无网络请求） | 较慢（依赖网络） |
| 离线支持 | ✅ 支持 | ❌ 不支持 |
| 防篡改能力 | ❌ 越狱后可替换收据 | ✅ 服务端验证无法伪造 |
| 适用场景 | 非消耗型、低价值商品 | 消耗型、订阅、高价值商品 |
| 苹果推荐 | ❌ 不推荐单独使用 | ✅ 推荐方案 |

> ⚠️ **苹果官方建议**：对于任何涉及真实金钱的交易，都应使用服务端验证。本地验证只能作为辅助手段。

### 5.2 App Receipt 结构

App Receipt 是苹果签发的购买凭证，就像超市购物小票——上面记录了你买了什么、花了多少钱、什么时候买的。

```
App Receipt（PKCS#7 容器）
├── 签名信息（苹果 CA 签名）
└── Receipt Payload（ASN.1 编码）
    ├── Bundle ID
    ├── Application Version
    ├── Original Application Version
    ├── Receipt Creation Date
    ├── In-App Purchase Receipts[]
    │   ├── Quantity
    │   ├── Product ID
    │   ├── Transaction ID
    │   ├── Original Transaction ID
    │   ├── Purchase Date
    │   ├── Expiration Date（订阅）
    │   └── Cancellation Date（退款）
    └── ...
```

### 5.3 ASN.1 解析（本地验证）

本地验证需要解析 ASN.1 格式的收据，这就像拆解一个俄罗斯套娃——一层层剥开才能看到里面的数据。

```swift
import Security

enum ReceiptAttributeType: Int {
    case bundleID = 2
    case appVersion = 3
    case opaqueValue = 4
    case receiptHash = 5
    case inAppPurchase = 17
    case originalAppVersion = 19
    case expirationDate = 21
}

struct InAppPurchaseReceipt {
    let quantity: Int
    let productID: String
    let transactionID: String
    let purchaseDate: Date
    let expirationDate: Date?
    let cancellationDate: Date?
}

func parseReceipt(data: Data) -> [InAppPurchaseReceipt] {
    var purchases: [InAppPurchaseReceipt] = []

    guard let payload = extractPayload(from: data) else { return purchases }

    var pos = payload.startIndex
    while pos < payload.endIndex {
        guard let attribute = parseAttribute(at: pos, in: payload) else { break }
        pos = attribute.nextPosition

        if attribute.type == ReceiptAttributeType.inAppPurchase.rawValue {
            let iap = parseInAppPurchase(from: attribute.value)
            purchases.append(iap)
        }
    }

    return purchases
}
```

> 💡 **StoreKit 2 的改变**：StoreKit 2 使用 `Transaction` API 替代了传统的收据验证。`Transaction.updates` 和 `Product.purchase()` 返回的 `VerificationResult` 已经内置了验证逻辑，大多数场景下不再需要手动解析 ASN.1 收据。

### 5.4 服务端验证完整流程

```
客户端                           服务端                          苹果服务器
  │                               │                               │
  │  1. 获取 App Receipt          │                               │
  │  ────────────────────>        │                               │
  │                               │                               │
  │  2. Base64 编码收据           │                               │
  │  ──────────────────────────────────>                         │
  │                               │                               │
  │                               │  3. 发送验证请求               │
  │                               │  ──────────────────────────>  │
  │                               │                               │
  │                               │  4. 返回验证结果               │
  │                               │  <──────────────────────────  │
  │                               │                               │
  │                               │  5. 解析结果，更新数据库       │
  │                               │  ────                         │
  │                               │                               │
  │  6. 返回验证结果给客户端       │                               │
  │  <──────────────────────────  │                               │
  │                               │                               │
  │  7. 解锁/拒绝功能             │                               │
  │  ────                         │                               │
```

**服务端验证代码示例（Swift + Vapor）：**

```swift
import Vapor

struct ReceiptController: RouteCollection {
    func boot(routes: RoutesBuilder) throws {
        let receipts = routes.grouped("api", "receipt")
        receipts.post("verify", use: verifyReceipt)
    }

    func verifyReceipt(req: Request) async throws -> ReceiptResponse {
        let body = try req.content.decode(ReceiptRequest.self)

        let isSandbox = body.environment == "sandbox"
        let baseURL = isSandbox
            ? "https://sandbox.itunes.apple.com/verifyReceipt"
            : "https://buy.itunes.apple.com/verifyReceipt"

        let payload: [String: Any] = [
            "receipt-data": body.receiptData,
            "password": ProcessInfo.processInfo.environment["APP_SECRET"] ?? "",
            "exclude-old-transactions": true
        ]

        let responseData = try await sendVerificationRequest(
            to: baseURL,
            payload: payload,
            on: req
        )

        let result = try JSONSerialization.jsonObject(with: responseData) as! [String: Any]
        let status = result["status"] as! Int

        if status == 21007 {
            return try await verifyWithSandbox(receiptData: body.receiptData, on: req)
        }

        guard status == 0 else {
            throw Abort(.badRequest, reason: "Receipt validation failed: \(status)")
        }

        return try parseReceiptResponse(result)
    }
}
```

> ⚠️ **状态码 21007**：当生产环境验证沙盒收据时，苹果会返回 21007。此时应自动重试沙盒验证 URL。这是最常见的"收据验证失败"原因之一。

**常见验证状态码：**

| 状态码 | 含义 | 处理方式 |
|--------|------|----------|
| 0 | 验证成功 | 正常处理 |
| 21002 | 收据数据格式错误 | 检查 Base64 编码 |
| 21003 | 收据认证失败 | 收据被篡改或无效 |
| 21004 | 共享密钥不匹配 | 检查 App Secret |
| 21005 | 苹果服务器暂不可用 | 稍后重试 |
| 21006 | 订阅已过期 | 仍可获取最后续期信息 |
| 21007 | 沙盒收据发到了生产服务器 | 重试沙盒 URL |
| 21008 | 生产收据发到了沙盒服务器 | 重试生产 URL |

---

## 六、订阅状态管理

### 6.1 StoreKit 2 的订阅状态模型

订阅就像健身房会员——你办了卡，但卡的状态会变：正常、快到期、已过期、宽限期内、被退款……每种状态对应不同的处理逻辑。

### 6.2 Product.SubscriptionInfo.Status

```swift
func checkSubscriptionStatus(productID: String) async throws {
    let product = try await Product.products(for: [productID]).first!
    let statuses = try await product.subscription?.status ?? []

    for status in statuses {
        switch status.state {
        case .subscribed:
            print("✅ 用户已订阅")
        case .expired:
            print("❌ 订阅已过期")
        case .inBillingRetryPeriod:
            print("⚠️ 计费重试中，仍可使用")
        case .inGracePeriod:
            print("⏳ 宽限期内，仍可使用")
        case .revoked:
            print("🚫 订阅已被撤销（退款）")
        case .offered:
            print("🎁 用户有优惠待领取")
        @unknown default:
            print("❓ 未知状态")
        }
    }
}
```

### 6.3 Transaction.updates 监听

`Transaction.updates` 是一个异步序列，就像一个永远在线的广播——每当有新的交易或状态变化，它就会推送给你。

```swift
@MainActor
class SubscriptionManager: ObservableObject {
    @Published var isSubscribed = false
    @Published var subscriptionStatus: Product.SubscriptionInfo.Status?

    private var updateTask: Task<Void, Never>?

    func startListening() {
        updateTask = Task {
            for await verification in Transaction.updates {
                guard case .verified(let transaction) = verification else {
                    continue
                }

                await self.updateSubscriptionStatus(transaction)
                await transaction.finish()
            }
        }
    }

    func stopListening() {
        updateTask?.cancel()
    }

    private func updateSubscriptionStatus(_ transaction: Transaction) async {
        if transaction.productID == "com.example.premium_sub" {
            isSubscribed = transaction.revocationDate == nil
                && transaction.expirationDate?.compare(.now) == .orderedDescending
        }
    }
}
```

### 6.4 订阅生命周期状态处理

| 状态 | 含义 | 用户体验 | 处理策略 |
|------|------|----------|----------|
| `.subscribed` | 正常订阅中 | 完整功能 | 无需特殊处理 |
| `.expired` | 订阅已过期 | 功能受限 | 引导重新订阅 |
| `.inBillingRetryPeriod` | 扣款失败，重试中 | 保留功能 | 提示用户更新支付方式 |
| `.inGracePeriod` | 宽限期（苹果给几天缓冲） | 保留功能 | 温和提醒续费 |
| `.revoked` | 被撤销（退款） | 功能受限 | 立即撤销权限 |
| `.offered` | 有优惠可用 | 引导领取 | 展示优惠信息 |

### 6.5 完整订阅状态管理器

```swift
@MainActor
class SubscriptionStateManager: ObservableObject {
    @Published var currentStatus: SubscriptionState = .notSubscribed

    enum SubscriptionState {
        case notSubscribed
        case subscribed(expiryDate: Date)
        case inGracePeriod(expiryDate: Date)
        case inBillingRetry(expiryDate: Date)
        case expired
        case revoked
    }

    func refreshStatus() async {
        guard let product = try await Product.products(for: ["com.example.premium_sub"]).first,
              let subscription = product.subscription else {
            currentStatus = .notSubscribed
            return
        }

        let statuses = try await subscription.status
        guard let status = statuses.first else {
            currentStatus = .notSubscribed
            return
        }

        switch status.state {
        case .subscribed:
            if case .verified(let tx) = status.transaction {
                currentStatus = .subscribed(expiryDate: tx.expirationDate!)
            }
        case .inGracePeriod:
            if case .verified(let tx) = status.transaction {
                currentStatus = .inGracePeriod(expiryDate: tx.expirationDate!)
            }
        case .inBillingRetryPeriod:
            if case .verified(let tx) = status.transaction {
                currentStatus = .inBillingRetry(expiryDate: tx.expirationDate!)
            }
        case .expired:
            currentStatus = .expired
        case .revoked:
            currentStatus = .revoked
        default:
            currentStatus = .notSubscribed
        }
    }

    var hasAccess: Bool {
        switch currentStatus {
        case .subscribed, .inGracePeriod, .inBillingRetry:
            return true
        case .notSubscribed, .expired, .revoked:
            return false
        }
    }
}
```

> 💡 **宽限期和计费重试期**是苹果为开发者提供的"缓冲区"——在这两个阶段，用户仍然可以使用功能。如果你在这期间就撤销权限，用户体验会很差，可能导致永久流失。

---

## 七、测试自动化

### 7.1 XCTest + StoreKit

内购自动化测试就像给收银台装了自动检测仪——每次代码变更后自动验证购买流程是否正常。

```swift
import XCTest
import StoreKit
import StoreKitTest

@MainActor
final class StoreKitTests: XCTestCase {
    var session: SKTestSession!

    override func setUp() async throws {
        try await super.setUp()
        session = try SKTestSession(configurationFileNamed: "Products")
        session.disableDialogs = true
        session.clearTransactions()
    }

    override func tearDown() async throws {
        session = nil
        try await super.tearDown()
    }

    func testConsumablePurchase() async throws {
        let product = try await Product.products(for: ["com.example.coins_100"]).first!
        let result = try await product.purchase()

        if case .success(let verification) = result,
           case .verified(let transaction) = verification {
            XCTAssertEqual(transaction.productID, "com.example.coins_100")
            await transaction.finish()
        }
    }

    func testNonConsumablePurchaseAndRestore() async throws {
        let product = try await Product.products(for: ["com.example.premium"]).first!

        let result = try await product.purchase()
        if case .success(let verification) = result,
           case .verified(let transaction) = verification {
            await transaction.finish()
        }

        session.clearTransactions()

        var restored = false
        for await verification in Transaction.currentEntitlements {
            if case .verified(let transaction) = verification,
               transaction.productID == "com.example.premium" {
                restored = true
                await transaction.finish()
            }
        }
        XCTAssertTrue(restored, "应该能恢复已购买的非消耗型商品")
    }

    func testSubscriptionRenewalAndExpiry() async throws {
        let product = try await Product.products(for: ["com.example.monthly_sub"]).first!

        let result = try await product.purchase()
        if case .success(let verification) = result,
           case .verified(let transaction) = verification {
            await transaction.finish()
        }

        try session.advanceSubscriptionRenewal(by: 1)

        let statuses = try await product.subscription?.status ?? []
        XCTAssertEqual(statuses.count, 1)
        XCTAssertEqual(statuses.first?.state, .subscribed)

        try session.advanceSubscriptionRenewal(by: 6)

        let expiredStatuses = try await product.subscription?.status ?? []
        let hasExpired = expiredStatuses.contains { $0.state == .expired }
        XCTAssertTrue(hasExpired, "续期 6 次后沙盒订阅应自动过期")
    }

    func testRefundRevokesAccess() async throws {
        let product = try await Product.products(for: ["com.example.premium"]).first!

        let result = try await product.purchase()
        if case .success(let verification) = result,
           case .verified(let transaction) = verification {
            await transaction.finish()
        }

        let transactions = session.allTransactions()
        try session.refundTransaction(transactions[0].identifier)

        var stillEntitled = false
        for await verification in Transaction.currentEntitlements {
            if case .verified(let transaction) = verification,
               transaction.productID == "com.example.premium" {
                stillEntitled = true
            }
        }
        XCTAssertFalse(stillEntitled, "退款后不应再有访问权限")
    }
}
```

### 7.2 CI 中运行内购测试

在 CI 环境中运行内购测试需要特殊配置，因为 StoreKit Testing 依赖 Xcode 和模拟器：

```yaml
# GitHub Actions 示例
name: StoreKit Tests

on:
  push:
    paths:
      - 'Sources/**'
      - 'Tests/**'

jobs:
  storekit-tests:
    runs-on: macos-14
    steps:
      - uses: actions/checkout@v4

      - name: Select Xcode
        run: sudo xcode-select -s /Applications/Xcode_15.4.app

      - name: Run StoreKit Tests
        run: |
          xcodebuild test \
            -project MyApp.xcodeproj \
            -scheme MyApp \
            -destination 'platform=iOS Simulator,name=iPhone 15,OS=17.5' \
            -only-testing:MyAppTests/StoreKitTests \
            -skipPackagePluginValidation

      - name: Upload Test Results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: build/test-results
```

> ⚠️ **CI 注意事项**：
> - 必须使用 macOS runner（StoreKit Testing 不支持 Linux）
> - StoreKit Configuration File 必须包含在测试 target 中
> - `SKTestSession` 只能在测试环境中使用，不能在正式 App 代码中使用
> - CI 中的模拟器性能较低，测试超时时间应适当延长

### 7.3 测试覆盖率策略

| 测试类型 | 覆盖场景 | 工具 | 频率 |
|----------|----------|------|------|
| 单元测试 | 购买成功/失败/取消 | XCTest + SKTestSession | 每次提交 |
| 集成测试 | 完整购买→验证→解锁流程 | XCTest + SKTestSession | 每次提交 |
| 订阅测试 | 续期/过期/退款/宽限期 | XCTest + SKTestSession | 每次提交 |
| 沙盒回归 | 真实网络环境购买 | 手动 Sandbox 测试 | 每个版本发布前 |
| 通知测试 | Server Notifications V2 | 服务端日志验证 | 每个版本发布前 |
| 生产监控 | 真实用户交易异常 | App Store Server API | 持续运行 |

> 💡 **测试金字塔**：70% 自动化单元/集成测试 + 20% 沙盒手动测试 + 10% 生产监控，这是内购测试的最佳投入比例。

---

## 八、常见问题与最佳实践

### 8.1 测试账号管理

| 问题 | 建议 |
|------|------|
| 沙盒账号不够用 | 按 `项目_环境_序号` 命名，方便管理 |
| 沙盒账号被锁定 | 避免频繁切换地区，每个地区单独创建账号 |
| 忘记哪个账号买了什么 | 建立测试账号使用日志表格 |
| 多人共用沙盒账号冲突 | 每个测试人员分配独立账号 |
| 沙盒账号与真机 Apple ID 冲突 | 沙盒账号只在"设置→App Store→Sandbox"中登录，不要在主 Apple ID 处登录 |

### 8.2 收据验证失败排查清单

```
收据验证失败排查步骤：

1. □ 确认使用正确的验证 URL（沙盒 vs 生产）
2. □ 检查 Base64 编码是否正确（不要有多余换行）
3. □ 确认 App Secret 是否匹配
4. □ 检查状态码 21007 → 自动重试沙盒 URL
5. □ 确认收据未过期（收据有 expirationDate 字段）
6. □ 检查 Bundle ID 是否匹配
7. □ 确认网络连接正常（苹果服务器偶尔不稳定）
8. □ 检查是否在越狱设备上运行（收据可能被篡改）
9. □ 确认 App 版本与收据中的版本一致
10. □ 查看 Apple 系统状态页面是否有服务中断
```

### 8.3 苹果审核内购相关被拒原因

| 被拒原因 | 说明 | 解决方案 |
|----------|------|----------|
| 3.1.1 - 未使用 IAP | 虚拟商品/服务必须使用 IAP | 数字内容一律走内购 |
| 3.1.1 - 使用第三方支付 | App 内不得引导用户绕过 IAP | 移除任何第三方支付链接 |
| 3.1.2 - 订阅价格不清晰 | 未明确标注订阅价格和周期 | 在付费墙显著位置显示价格 |
| 3.1.2 - 未提供管理订阅入口 | 用户无法管理订阅 | 添加"管理订阅"链接 |
| 2.1 - 恢复购买缺失 | 非消耗型/订阅缺少恢复功能 | 添加"恢复购买"按钮 |
| 3.1.1 - 免费试用不明确 | 试用期后自动扣费未告知 | 明确标注"试用期后自动续费" |
| 2.1 - IAP 功能异常 | 购买后无法解锁功能 | 确保购买→验证→解锁链路完整 |

> ⚠️ **2024 年审核新动态**：苹果对"引导用户去网页购买"的审查更加严格。即使 App 不包含直接链接，如果 UI 设计暗示用户去网页购买，也可能被拒。

### 8.4 最佳实践总结

| 实践 | 说明 |
|------|------|
| **始终使用服务端验证** | 本地验证可被越狱绕过，消耗型商品尤其危险 |
| **处理所有订阅状态** | 不要只处理 subscribed/expired，忽略宽限期和计费重试 |
| **及时 finish 交易** | 未 finish 的交易会一直出现在队列中，导致重复处理 |
| **实现恢复购买** | 非消耗型和订阅必须提供恢复购买功能 |
| **监听 Transaction.updates** | 确保实时响应订阅状态变化 |
| **使用 Server Notifications V2** | 比客户端轮询更及时、更可靠 |
| **区分沙盒和生产环境** | 收据验证 URL、行为逻辑都要区分 |
| **记录所有交易日志** | 出现问题时可以快速定位 |
| **定期审计订阅状态** | 客户端和服务端状态可能不一致，定期同步 |
| **优雅处理网络失败** | 收据验证失败时不要立即撤销权限，给用户缓冲 |

---

## 本章小结

| 主题 | 核心要点 |
|------|----------|
| 内购测试概述 | 三种环境各有用途：Xcode 开发、Sandbox 上线前、Production 正式运营 |
| Xcode StoreKit Testing | Configuration File + SKTestSession 实现完全本地化、可自动化的内购测试 |
| Sandbox 沙盒测试 | 真实网络环境验证，注意沙盒账号管理和时间加速规则 |
| App Store Server API | V2 接口 + JWT 认证，主动查询交易和订阅状态 |
| Server Notifications V2 | 事件驱动的服务端通知，比客户端轮询更可靠 |
| 收据验证 | 优先服务端验证；StoreKit 2 的 Transaction API 简化了本地验证 |
| 订阅状态管理 | 6 种状态全覆盖，宽限期和计费重试期保留用户权限 |
| 测试自动化 | XCTest + SKTestSession 实现自动化，CI 中使用 macOS runner |
| 最佳实践 | 服务端验证、及时 finish、恢复购买、Server Notifications、日志记录 |

> 💡 内购测试不是"锦上添花"，而是"生死攸关"。一个内购 Bug 导致的差评和用户流失，可能比十个普通 Bug 加起来还严重。投入足够的时间在内购测试上，是值得的。

---

← [116-App Store Connect API 自动化](116-AppStoreConnect-API自动化.md) | [118-数据合规与隐私保护](118-数据合规与隐私保护.md) →

← [-内购与订阅模式实战](./129-内购与订阅模式实战.md) | [-付费墙与转化设计](./131-付费墙与转化设计.md) →
