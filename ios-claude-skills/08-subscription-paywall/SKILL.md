---
name: subscription-paywall
description: 涉及订阅、付费墙、IAP、会员权益、试用、StoreKit、Paywall 页面的任务
---

# 订阅 & Paywall

## 技术栈
- **StoreKit 2**（iOS 15+）
- 产品配置：App Store Connect → Subscriptions
- 本地权益验证：`Transaction.currentEntitlements`
- 服务端验证（可选）：JWS Transaction 发送到后端校验

## 订阅产品结构
```swift
enum SubscriptionProduct: String, CaseIterable {
    case monthly  = "com.app.subscription.monthly"
    case yearly   = "com.app.subscription.yearly"
    // 按项目实际 Bundle ID 修改
}
```

## 标准购买流程
```swift
// 1. 加载产品
let products = try await Product.products(for: SubscriptionProduct.allCases.map(\.rawValue))

// 2. 发起购买
let result = try await product.purchase()

// 3. 处理结果
switch result {
case .success(let verification):
    guard case .verified(let transaction) = verification else { return }
    await transaction.finish()
    // 更新本地权益状态
case .userCancelled:
    break  // 用户取消，不报错
case .pending:
    // 等待家长批准等异步场景
@unknown default:
    break
}
```

## 权益检查
```swift
// App 启动时 & 前台时检查
func checkEntitlement() async -> Bool {
    for await result in Transaction.currentEntitlements {
        if case .verified(let transaction) = result {
            if transaction.productType == .autoRenewable {
                return transaction.revocationDate == nil
            }
        }
    }
    return false
}
```

## Paywall 页面规范
- 展示顺序：**价值主张 → 功能对比 → 价格 → CTA 按钮**
- 必须包含：
  - [ ] 订阅价格 + 周期（清晰可见）
  - [ ] 免费试用时长（如有）
  - [ ] 「恢复购买」按钮
  - [ ] 隐私政策 + 用户协议链接
  - [ ] 订阅说明文字（App Store 要求）
- CTA 按钮文案推荐：
  - 有试用："免费试用 X 天" / "7 天免费体验"
  - 无试用："立即订阅" / "解锁全部功能"
- **禁止：** 误导性试用提示、隐藏价格、自动关闭倒计时

## 订阅说明文字模板（App Store 要求）
```
订阅说明：
• 订阅费用将在确认购买时从您的 Apple ID 账户扣除
• 订阅将自动续期，除非您在当前周期结束前至少 24 小时关闭自动续订
• 续订费用将在当前周期结束前 24 小时内扣除
• 您可以在 App Store 账户设置中管理或取消订阅
• 免费试用期未使用的部分将在购买订阅时作废
```

## 权益门控
```swift
// 功能入口统一检查
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

## 分析埋点（必须）
- `paywall_shown`：展示 Paywall
- `paywall_purchase_tapped`：点击购买
- `purchase_success`：购买成功（含产品 ID）
- `purchase_failed`：购买失败（含错误类型）
- `restore_tapped`：点击恢复购买
