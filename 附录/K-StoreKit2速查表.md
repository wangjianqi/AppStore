# K-StoreKit 2 速查表

> 本速查表涵盖 StoreKit 2 (iOS 15+) 核心 API，适用于内购和订阅开发日常查阅。

---

## 1. Product 获取

### 1.1 核心 API

| API | 说明 | 返回类型 |
|-----|------|----------|
| `Product.products(for: [String])` | 批量获取产品信息 | `[Product]` |
| `Product.products(for: [String], timeout:)` | 带超时的获取 | `[Product]` |
| `Product.product(for: String)` | 获取单个产品 | `Product` |

### 1.2 Product 属性

| 属性 | 类型 | 说明 |
|------|------|------|
| `id` | `String` | 产品 ID（App Store Connect 配置） |
| `type` | `Product.ProductType` | `.consumable` / `.nonConsumable` / `.autoRenewable` / `.nonRenewable` |
| `displayName` | `String` | 显示名称 |
| `description` | `String` | 产品描述 |
| `displayPrice` | `String` | 格式化价格（含货币符号） |
| `price` | `Decimal` | 原始价格数值 |
| `subscription` | `Product.SubscriptionInfo?` | 订阅信息（仅订阅产品） |

### 1.3 产品类型枚举

| 类型 | 说明 | 示例 |
|------|------|------|
| `.consumable` | 消耗型 | 游戏金币、虚拟货币 |
| `.nonConsumable` | 非消耗型 | 去广告、解锁功能 |
| `.autoRenewable` | 自动续期订阅 | 会员月卡、年度订阅 |
| `.nonRenewable` | 非续期订阅 | 限时访问权限 |

```swift
import StoreKit

func loadProducts() async throws -> [Product] {
    let productIDs = [
        "com.app.coin100",
        "com.app.premium",
        "com.app.subscription.monthly"
    ]
    let products = try await Product.products(for: productIDs)
    for product in products {
        print("\(product.displayName) - \(product.displayPrice)")
        print("类型：\(product.type)")
    }
    return products
}
```

---

## 2. 购买流程

### 2.1 purchase() 方法

| 方法签名 | 说明 |
|----------|------|
| `purchase()` | 基本购买 |
| `purchase(options:)` | 带选项购买 |

### 2.2 PurchaseResult

| 结果 | 说明 |
|------|------|
| `.success(verification)` | 购买成功，需验证交易 |
| `.userCancelled` | 用户取消 |
| `.pending` | 交易待定（需家长批准等） |
| `.error(Error)` | 购买失败 |

### 2.3 购买选项

| 选项 | 说明 |
|------|------|
| `Product.PurchaseOption.offerID` | 促销优惠 ID |
| `Product.PurchaseOption.offerSigningKey` | 优惠签名 |
| `Product.PurchaseOption.appAccountToken` | 关联用户标识（UUID） |

```swift
func purchase(_ product: Product) async throws -> Transaction? {
    let result = try await product.purchase()

    switch result {
    case .success(let verification):
        let transaction = try checkVerified(verification)
        await transaction.finish()
        return transaction

    case .userCancelled:
        print("用户取消购买")
        return nil

    case .pending:
        print("交易待定，等待审批")
        return nil

    case .error(let error):
        print("购买失败：\(error)")
        return nil

    @unknown default:
        return nil
    }
}

func checkVerified<T>(_ result: VerificationResult<T>) throws -> T {
    switch result {
    case .unverified(_, let error):
        throw error
    case .verified(let safe):
        return safe
    }
}
```

---

## 3. 交易验证

### 3.1 VerificationResult

| 状态 | 说明 | 处理 |
|------|------|------|
| `.verified(Transaction)` | 验证通过 | 安全使用 |
| `.unverified(Transaction, error)` | 验证失败 | 不应信任，丢弃 |

### 3.2 Transaction 属性

| 属性 | 类型 | 说明 |
|------|------|------|
| `id` | `UInt64` | 交易唯一 ID |
| `productID` | `String` | 产品 ID |
| `purchaseDate` | `Date` | 购买时间 |
| `expirationDate` | `Date?` | 过期时间（订阅） |
| `revocationDate` | `Date?` | 撤销时间（退款） |
| `appAccountToken` | `UUID?` | 关联用户标识 |
| `originalID` | `UInt64` | 原始交易 ID |
| `isUpgraded` | `Bool` | 是否已升级 |

### 3.3 交易监听

| API | 说明 |
|-----|------|
| `Transaction.updates` | 异步序列，监听新交易 |
| `Transaction.currentEntitlements` | 当前有效权益 |
| `Transaction.all` | 所有历史交易 |

```swift
func listenForTransactions() -> Task<Void, Never> {
    Task.detached {
        for await result in Transaction.updates {
            do {
                let transaction = try self.checkVerified(result)
                self.updatePurchasedState(transaction)
                await transaction.finish()
            } catch {
                print("交易验证失败：\(error)")
            }
        }
    }
}

func getCurrentEntitlements() async -> [Transaction] {
    var entitlements: [Transaction] = []
    for await result in Transaction.currentEntitlements {
        if let transaction = try? checkVerified(result) {
            if transaction.revocationDate == nil {
                entitlements.append(transaction)
            }
        }
    }
    return entitlements
}
```

---

## 4. 订阅管理

### 4.1 Product.SubscriptionInfo

| 属性/方法 | 类型 | 说明 |
|-----------|------|------|
| `groupID` | `String` | 订阅组 ID |
| `subscriptionGroupID` | `String` | 订阅组 ID（Product 上） |
| `renewalInfo` | `Product.SubscriptionInfo.RenewalInfo` | 续期信息 |

### 4.2 RenewalInfo 属性

| 属性 | 类型 | 说明 |
|------|------|------|
| `willAutoRenew` | `Bool` | 是否自动续期 |
| `expirationDate` | `Date` | 到期时间 |
| `autoRenewPreference` | `String?` | 用户偏好的续期产品 |
| `gracePeriodExpirationDate` | `Date?` | 宽限期到期时间 |
| `offerID` | `String?` | 当前优惠 ID |
| `offerType` | `Product.SubscriptionOffer.OfferType?` | 优惠类型 |

### 4.3 订阅状态

| 状态 | 说明 |
|------|------|
| `Product.SubscriptionInfo.Status.active` | 活跃 |
| `expired` | 已过期 |
| `inGracePeriod` | 宽限期 |
| `inBillingRetryPeriod` | 计费重试期 |
| `revoked` | 已撤销 |

```swift
func checkSubscriptionStatus(for product: Product) async {
    guard let subscription = product.subscription else {
        print("非订阅产品")
        return
    }

    let statuses = try? await subscription.status
    for status in statuses ?? [] {
        switch status.state {
        case .active:
            print("订阅活跃")
        case .expired:
            print("订阅过期")
        case .inGracePeriod:
            print("宽限期中")
        case .inBillingRetryPeriod:
            print("计费重试中")
        case .revoked:
            print("订阅已撤销")
        @unknown default:
            break
        }

        if let renewalInfo = try? status.renewalInfo {
            let info = try? checkVerified(renewalInfo)
            print("自动续期：\(info?.willAutoRenew ?? false)")
        }
    }
}
```

### 4.4 订阅组状态

```swift
func getSubscriptionGroupStatus() async {
    let groupID = "com.app.premium_group"
    let statuses = try? await Product.SubscriptionInfo.status(for: groupID)

    let highestStatus = statuses?
        .filter { $0.state == .active }
        .sorted { $0.transaction.expirationDate ?? .distantPast > $1.transaction.expirationDate ?? .distantPast }
        .first

    if let status = highestStatus {
        print("最高级别订阅活跃")
    } else {
        print("无活跃订阅")
    }
}
```

---

## 5. 退款处理

### 5.1 退款请求 API

| API | 平台 | 说明 |
|-----|------|------|
| `refundRequestSheet(for:)` | iOS 15+ | 弹出退款申请界面 |
| `Transaction.revocationDate` | iOS 15+ | 退款撤销时间 |

### 5.2 RefundRequestSheet 结果

| 结果 | 说明 |
|------|------|
| `.success` | 退款请求已提交 |
| `.userCancelled` | 用户取消 |

```swift
func requestRefund(transactionID: UInt64) async {
    guard let windowScene = UIApplication.shared.connectedScenes.first as? UIWindowScene else {
        return
    }

    do {
        let result = try await Transaction.refundRequestSheet(
            for: transactionID,
            on: windowScene
        )
        switch result {
        case .success:
            print("退款请求已提交")
        case .userCancelled:
            print("用户取消退款")
        @unknown default:
            break
        }
    } catch {
        print("退款请求失败：\(error)")
    }
}
```

### 5.3 退款监听

```swift
func monitorRefunds() -> Task<Void, Never> {
    Task.detached {
        for await result in Transaction.updates {
            if let transaction = try? self.checkVerified(result) {
                if transaction.revocationDate != nil {
                    print("交易已被退款：\(transaction.productID)")
                    self.revokeAccess(for: transaction.productID)
                }
                await transaction.finish()
            }
        }
    }
}
```

---

## 6. StoreKit Configuration 文件

### 6.1 创建配置文件

| 步骤 | 操作 |
|------|------|
| 1 | Xcode → File → New → File → StoreKit Configuration File |
| 2 | 命名如 `StoreKitConfig.storekit` |
| 3 | 添加产品 ID、类型、价格 |
| 4 | 添加订阅组（如需） |

### 6.2 配置文件设置

| 设置项 | 说明 |
|--------|------|
| Product ID | 对应 App Store Connect 中的产品 ID |
| Reference Price | 参考价格 |
| Localized Display Name | 本地化显示名 |
| Subscription Group | 订阅所属组 |

### 6.3 启用本地测试

| 步骤 | 操作 |
|------|------|
| 1 | Scheme → Edit Scheme → Options → StoreKit Configuration |
| 2 | 选择 `.storekit` 文件 |
| 3 | 运行应用，购买将使用本地配置 |

---

## 7. StoreKit Testing in Xcode

### 7.1 StoreKit 管理器

| 操作 | 位置 |
|------|------|
| 打开管理器 | Xcode → Debug → StoreKit → Manage Transactions |
| 批准交易 | 右键 → Approve |
| 退款交易 | 右键 → Refund |
| 删除交易 | 右键 → Delete |
| 切换地区 | 管理器中选择 Storefront |

### 7.2 测试场景

| 场景 | 操作 |
|------|------|
| 正常购买 | 点击购买按钮 |
| 用户取消 | 系统弹窗点取消 |
| 交易待定 | 在管理器中设置为 Ask to Buy |
| 订阅续期 | 在管理器中快进时间 |
| 订阅过期 | 快进到过期日期 |
| 退款 | 管理器中 Refund |
| 计费重试 | 设置为 Billing Retry |
| 宽限期 | 设置 Grace Period |

### 7.3 XCTest 集成

```swift
import StoreKit
import XCTest

final class StoreKitTests: XCTestCase {
    var configuration: StoreKit.Configuration!

    override func setUp() async throws {
        configuration = StoreKit.Configuration(
            storefront: .china,
            testingEnabled: true
        )
        try await super.setUp()
    }

    func testPurchase() async throws {
        let products = try await Product.products(for: ["com.app.coin100"])
        XCTAssertEqual(products.count, 1)

        let product = products[0]
        let result = try await product.purchase()

        switch result {
        case .success(let verification):
            let transaction = try checkVerified(verification)
            XCTAssertEqual(transaction.productID, "com.app.coin100")
            await transaction.finish()
        default:
            XCTFail("购买应该成功")
        }
    }
}
```

---

## 8. 服务器通知

### 8.1 App Store Server API

| API | 方法 | 说明 |
|-----|------|------|
| Get Transaction History | `GET /inApps/v1/history/{transactionId}` | 获取交易历史 |
| Get Transaction Info | `GET /inApps/v1/transactions/{transactionId}` | 获取交易详情 |
| Get Refund History | `GET /inApps/v1/refund/lookup/{originalTransactionId}` | 获取退款历史 |
| Get All Subscription Statuses | `GET /inApps/v1/subscriptions/{transactionId}` | 获取订阅状态 |
| Send Consumption Information | `PUT /inApps/v1/transactions/consumption/{originalTransactionId}` | 发送消费信息 |
| Extend Subscription Renewal Date | `POST /inApps/v1/subscriptions/extend/{originalTransactionId}` | 延长订阅续期日期 |
| Get Notification History | `GET /inApps/v1/notifications` | 获取通知历史 |

### 8.2 V2 通知类型

| 通知类型 | 说明 |
|----------|------|
| `DID_RENEW` | 订阅续期成功 |
| `DID_FAIL_TO_RENEW` | 续期失败 |
| `DID_CHANGE_RENEWAL_STATUS` | 续期状态变更 |
| `SUBSCRIBED` | 新订阅 |
| `EXPIRED` | 订阅过期 |
| `GRACE_PERIOD_EXPIRED` | 宽限期过期 |
| `DID_CHANGE_RENEWAL_PREF` | 续期偏好变更 |
| `REFUND` | 退款 |
| `REVOKE` | 撤销（家庭共享） |
| `CONSUMPTION_REQUEST` | 消费请求 |
| `ONE_TIME_CHARGE` | 一次性购买 |
| `RENEWAL_EXTENDED` | 续期已延长 |
| `REFUND_DECLINED` | 退款被拒绝 |
| `PRICE_CHANGE` | 价格变更 |

### 8.3 JWT 验证

```swift
import CryptoKit

func verifyJWT(_ token: String) -> Bool {
    let parts = token.split(separator: ".")
    guard parts.count == 3 else { return false }

    let header = String(parts[0])
    let payload = String(parts[1])
    let signature = String(parts[2])

    let signedData = "\(header).\(payload)"
    let signedDataBytes = Data(base64Encoded: signedData.base64URL()) ?? Data()

    let signatureBytes = Data(base64Encoded: signature.base64URL()) ?? Data()

    let publicKey = getApplePublicKey()
    let isValid = publicKey.verify(
        signature: signatureBytes,
        for: signedDataBytes
    )

    return isValid
}
```

---

## 9. 收据验证关键步骤

### 9.1 客户端验证流程

| 步骤 | 操作 | 说明 |
|------|------|------|
| 1 | 获取交易 | `Transaction.updates` / `Transaction.currentEntitlements` |
| 2 | 验证签名 | `VerificationResult` 自动验证 | 
| 3 | 检查撤销 | `revocationDate == nil` |
| 4 | 处理权益 | 更新本地状态 |
| 5 | 完成交易 | `transaction.finish()` |

### 9.2 服务端验证流程

| 步骤 | 操作 | 说明 |
|------|------|------|
| 1 | 接收原始交易 ID | 客户端上报 |
| 2 | 调用 Server API | 获取交易详情 |
| 3 | 验证 JWT 签名 | 使用 Apple 公钥 |
| 4 | 检查 bundleID | 确认属于本应用 |
| 5 | 检查环境 | sandbox / production |
| 6 | 更新用户权益 | 写入数据库 |

### 9.3 关键安全注意事项

| 事项 | 说明 |
|------|------|
| 始终验证签名 | 不要信任未验证的交易 |
| 检查 bundleID | 防止跨应用攻击 |
| 检查环境 | sandbox 交易不应授予生产权益 |
| 及时 finish | 未 finish 的交易会持续出现 |
| 服务端为主 | 关键权益判断应在服务端完成 |

---

## 10. 常用代码模板

### 10.1 完整的 StoreManager

```swift
import StoreKit
import Observation

@Observable
class StoreManager {
    var products: [Product] = []
    var purchasedProductIDs: Set<String> = []
    var subscriptionGroupStatus: Product.SubscriptionInfo.Status?

    private var transactionListener: Task<Void, Never>?

    init() {
        transactionListener = listenForTransactions()
    }

    deinit {
        transactionListener?.cancel()
    }

    func loadProducts() async {
        do {
            let storeProducts = try await Product.products(for: productIDs)
            products = storeProducts.sorted { $0.price < $1.price }
        } catch {
            print("加载产品失败：\(error)")
        }
    }

    func purchase(_ product: Product) async -> Bool {
        do {
            let result = try await product.purchase(options: [])

            switch result {
            case .success(let verification):
                let transaction = try checkVerified(verification)
                updatePurchasedState(transaction)
                await transaction.finish()
                return true
            case .userCancelled, .pending:
                return false
            case .error(let error):
                print("购买错误：\(error)")
                return false
            @unknown default:
                return false
            }
        } catch {
            print("购买失败：\(error)")
            return false
        }
    }

    func updatePurchasedState(_ transaction: Transaction) {
        if transaction.revocationDate == nil {
            purchasedProductIDs.insert(transaction.productID)
        } else {
            purchasedProductIDs.remove(transaction.productID)
        }
    }

    func restorePurchases() async {
        do {
            try await AppStore.sync()
        } catch {
            print("恢复购买失败：\(error)")
        }
    }

    private func listenForTransactions() -> Task<Void, Never> {
        Task.detached { [weak self] in
            guard let self else { return }
            for await result in Transaction.updates {
                do {
                    let transaction = try self.checkVerified(result)
                    self.updatePurchasedState(transaction)
                    await transaction.finish()
                } catch {
                    print("交易验证失败：\(error)")
                }
            }
        }
    }

    private func checkVerified<T>(_ result: VerificationResult<T>) throws -> T {
        switch result {
        case .unverified(_, let error):
            throw error
        case .verified(let safe):
            return safe
        }
    }
}
```

### 10.2 订阅状态检查

```swift
func checkSubscriptionStatus() async {
    for await result in Transaction.currentEntitlements {
        guard let transaction = try? checkVerified(result) else { continue }
        guard transaction.productID.hasPrefix("com.app.subscription") else { continue }

        if transaction.revocationDate == nil {
            let isExpired: Bool
            if let expirationDate = transaction.expirationDate {
                isExpired = expirationDate < Date()
            } else {
                isExpired = false
            }

            if !isExpired {
                purchasedProductIDs.insert(transaction.productID)
            }
        }
    }
}
```

### 10.3 促销优惠签名（服务端）

```swift
import CryptoKit
import Foundation

func generateOfferSignature(
    keyID: String,
    productID: String,
    offerID: String,
    applicationUsername: String?,
    nonce: UUID,
    timestamp: Int64,
    privateKey: P256.Signing.PrivateKey
) throws -> String {
    let payload = [
        applicationUsername ?? "",
        nonce.uuidString,
        String(timestamp),
        productID,
        offerID
    ].joined(separator: "\u{0001}")

    let signature = try privateKey.signature(for: Data(payload.utf8))
    return signature.rawRepresentation.base64EncodedString()
}
```

---

## 11. 常见问题速查

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 产品列表为空 | 产品 ID 不匹配 / 未审核通过 | 检查 ID 拼写，确认 App Store Connect 状态 |
| 购买返回 `.userCancelled` | 用户主动取消 | 正常行为，无需处理 |
| 交易一直不 finish | 忘记调用 `transaction.finish()` | 始终在处理完成后调用 finish |
| 沙盒环境订阅立即过期 | 沙盒加速时间 | 正常行为，查看沙盒时间表 |
| `Transaction.updates` 无响应 | 未启动监听 / 网络问题 | 在 app 启动时启动监听 |
| 退款后权益未更新 | 未检查 `revocationDate` | 检查交易撤销日期 |
| 促销优惠签名无效 | 签名参数错误 / 密钥不匹配 | 检查 keyID、nonce、timestamp |
| 恢复购买无效 | 未调用 `AppStore.sync()` | 使用 sync() 或遍历 currentEntitlements |

### 沙盒订阅时间表

| 实际时长 | 沙盒时长 | 续期次数 |
|----------|----------|----------|
| 1 周 | 3 分钟 | 最多 6 次 |
| 1 个月 | 5 分钟 | 最多 6 次 |
| 2 个月 | 10 分钟 | 最多 6 次 |
| 3 个月 | 15 分钟 | 最多 6 次 |
| 6 个月 | 30 分钟 | 最多 6 次 |
| 1 年 | 1 小时 | 最多 6 次 |

---

> 💡提示：StoreKit 2 仅支持 iOS 15+。如需兼容 iOS 14 及更早版本，仍需使用原始 StoreKit 1 API。
