---
name: subscription-paywall
description: 涉及订阅、付费墙、IAP、会员权益、试用、StoreKit、Paywall 页面、家庭共享、退款、沙盒测试的任务
---

# 订阅 & Paywall

## 技术栈
- **StoreKit 2**（iOS 15+）
- 产品配置：App Store Connect → Subscriptions
- 本地权益验证：`Transaction.currentEntitlements`
- 服务端验证（可选）：JWS Transaction 发送到后端校验

---

## 订阅产品结构

```swift
enum SubscriptionProduct: String, CaseIterable {
    case monthly  = "com.app.subscription.monthly"
    case yearly   = "com.app.subscription.yearly"
}

extension SubscriptionProduct {
    var displayName: String {
        switch self {
        case .monthly: return "月度会员"
        case .yearly: return "年度会员"
        }
    }

    var pricePerMonth: String {
        switch self {
        case .monthly: return "按月付费"
        case .yearly: return "按年付费（省 40%）"
        }
    }
}
```

---

## SubscriptionManager 完整封装

```swift
import StoreKit

final class SubscriptionManager {
    static let shared = SubscriptionManager()

    private(set) var isPro: Bool = false
    private(set) var activeSubscription: SubscriptionProduct?
    private(set) var expirationDate: Date?

    var onEntitlementChanged: ((Bool) -> Void)?

    private var transactionListener: Task<Void, Never>?

    private init() {
        transactionListener = listenForTransactions()
        Task { await checkEntitlement() }
    }

    deinit {
        transactionListener?.cancel()
    }

    func loadProducts() async throws -> [Product] {
        let productIDs = SubscriptionProduct.allCases.map(\.rawValue)
        return try await Product.products(for: productIDs)
    }

    func purchase(_ product: Product) async throws -> Bool {
        let result = try await product.purchase()

        switch result {
        case .success(let verification):
            let transaction = try checkVerification(verification)
            await transaction.finish()
            await updateEntitlement(transaction)
            return true
        case .userCancelled:
            return false
        case .pending:
            return false
        @unknown default:
            return false
        }
    }

    func restorePurchases() async {
        try? await AppStore.sync()
        await checkEntitlement()
    }

    func checkEntitlement() async {
        var isSubscribed = false
        var latestTransaction: StoreKit.Transaction?

        for await result in Transaction.currentEntitlements {
            if case .verified(let transaction) = result {
                if transaction.productType == .autoRenewable,
                   transaction.revocationDate == nil {
                    isSubscribed = true
                    if latestTransaction == nil || transaction.expirationDate ?? .distantPast > latestTransaction?.expirationDate ?? .distantPast {
                        latestTransaction = transaction
                    }
                }
            }
        }

        isPro = isSubscribed
        if let transaction = latestTransaction {
            activeSubscription = SubscriptionProduct(rawValue: transaction.productID)
            expirationDate = transaction.expirationDate
        }
        onEntitlementChanged?(isPro)
    }

    private func listenForTransactions() -> Task<Void, Never> {
        Task.detached { [weak self] in
            for await result in Transaction.updates {
                if case .verified(let transaction) = result {
                    await transaction.finish()
                    await self?.updateEntitlement(transaction)
                }
            }
        }
    }

    private func updateEntitlement(_ transaction: StoreKit.Transaction) async {
        isPro = transaction.revocationDate == nil
        activeSubscription = SubscriptionProduct(rawValue: transaction.productID)
        expirationDate = transaction.expirationDate
        onEntitlementChanged?(isPro)
    }

    private func checkVerification(_ result: VerificationResult<StoreKit.Transaction>) throws -> StoreKit.Transaction {
        switch result {
        case .unverified(_, let error):
            throw error
        case .verified(let transaction):
            return transaction
        }
    }
}
```

---

## 标准购买流程

```swift
// 1. 加载产品
let products = try await SubscriptionManager.shared.loadProducts()

// 2. 发起购买
let success = try await SubscriptionManager.shared.purchase(product)

// 3. 处理结果
if success {
    // 解锁功能
}
```

---

## Paywall 页面规范

### 展示顺序
**价值主张 → 功能对比 → 价格 → CTA 按钮**

### 必须包含
- [ ] 订阅价格 + 周期（清晰可见）
- [ ] 免费试用时长（如有）
- [ ] 「恢复购买」按钮
- [ ] 隐私政策 + 用户协议链接
- [ ] 订阅说明文字（App Store 要求）

### CTA 按钮文案推荐
- 有试用："免费试用 X 天" / "7 天免费体验"
- 无试用："立即订阅" / "解锁全部功能"

### 禁止
- 误导性试用提示
- 隐藏价格
- 自动关闭倒计时
- 虚假倒计时（"限时优惠"实际永不过期）

---

## 订阅说明文字模板（App Store 要求）

```
订阅说明：
• 订阅费用将在确认购买时从您的 Apple ID 账户扣除
• 订阅将自动续期，除非您在当前周期结束前至少 24 小时关闭自动续订
• 续订费用将在当前周期结束前 24 小时内扣除
• 您可以在 App Store 账户设置中管理或取消订阅
• 免费试用期未使用的部分将在购买订阅时作废
```

---

## 权益门控

```swift
func requiresPro(_ action: () -> Void) {
    if SubscriptionManager.shared.isPro {
        action()
    } else {
        presentPaywall()
    }
}
```

- **禁止在多处散落权益判断**，统一走 `SubscriptionManager`
- 免费用户体验：提供有限次数或降级功能，不直接锁死入口

---

## 已知陷阱

### 家庭共享

```swift
// 检查家庭共享资格
let eligibility = await transaction.eligibility
// .eligible → 家庭成员可用
// .notEligible → 仅购买者可用
```

- App Store Connect 中需开启"Family Sharing"
- 家庭共享的订阅，`Transaction.currentEntitlements` 会返回家庭成员的 transaction
- **陷阱**：家庭共享成员无法管理订阅（无法取消），只有购买者可以

### 退款处理

- Apple 退款后，`transaction.revocationDate` 不为 nil
- **必须检查 `revocationDate`**，否则退款用户仍可使用功能
- 退款通知：配置 App Store Server Notifications V2，服务端接收 `REFUND` 事件

```swift
// 正确的权益检查
if transaction.revocationDate == nil {
    // 有效订阅
} else {
    // 已退款，撤销权益
}
```

### 沙盒测试注意事项

| 问题 | 说明 |
|------|------|
| 订阅加速 | 沙盒中 1 个月 = 5 分钟，1 年 = 1 小时 |
| 最多续订 6 次 | 沙盒订阅最多自动续订 6 次 |
| 测试账号 | 需在 App Store Connect 创建沙盒测试账号 |
| 购买弹窗 | 沙盒购买显示"[沙盒环境]"标识 |
| StoreKit Configuration | Xcode 14+ 可用本地 StoreKit 配置文件测试，无需真机 |
| 退款测试 | 沙盒无法模拟退款，需用 StoreKit Testing 框架 |

### Transaction 监听丢失

- `Transaction.updates` 是异步流，App 被杀后可能丢失
- **必须在 App 启动时重新检查** `Transaction.currentEntitlements`
- 推荐配置 App Store Server Notifications V2，服务端兜底

### 其他陷阱

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| 购买后权益未更新 | 未监听 `Transaction.updates` | 启动时检查 + 持续监听 |
| 恢复购买无反应 | 未调用 `AppStore.sync()` | 恢复购买前先 sync |
| 订阅过期仍可用 | 未检查 `expirationDate` | 每次启动检查过期时间 |
| 沙盒购买卡住 | 沙盒环境不稳定 | 等待或换测试账号 |
| 价格显示错误 | 未使用 `Product.displayPrice` | 使用 `displayPrice` 而非 `price`（含货币符号） |
| 审核被拒 3.1.1 | 存在外部支付引导 | 移除所有外部支付链接/文字 |

---

## 分析埋点（必须）

| 事件名 | 触发时机 | 参数 |
|--------|---------|------|
| `paywall_shown` | 展示 Paywall | `source`: 来源页面 |
| `paywall_purchase_tapped` | 点击购买 | `product_id`: 产品 ID |
| `purchase_success` | 购买成功 | `product_id`, `is_trial`: 是否试用 |
| `purchase_failed` | 购买失败 | `product_id`, `error`: 错误类型 |
| `restore_tapped` | 点击恢复购买 | — |
| `restore_success` | 恢复成功 | `product_id` |
| `subscription_expired` | 订阅过期 | `product_id` |
