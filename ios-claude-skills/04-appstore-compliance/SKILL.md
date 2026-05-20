---
name: appstore-compliance
description: 涉及订阅、支付、权限申请、隐私数据、用户引导、审核准备的任务
---

# App Store 审核合规

## 内购 & 订阅（IAP）
- 数字内容 / 会员订阅**必须走 StoreKit 2**（iOS 15+），禁止第三方支付
- **禁止**在 App 内出现外部支付链接、二维码、价格暗示
- 订阅购买前必须展示：**价格 + 周期 + 免费试用时长**（如有）
- 恢复购买按钮必须可见（审核员会点）
- Receipt 验证推荐服务端验证，禁止纯客户端验证
- 退订引导：只能引导到系统订阅管理页，禁止自定义取消流程

```swift
// StoreKit 2 标准购买流程
let result = try await product.purchase()
switch result {
case .success(let verification):
    // 验证 transaction
case .userCancelled: break
case .pending: break
}
```

## 权限申请规范
- **原则：在用户理解使用场景后再申请，禁止启动即弹权限**
- 每个权限必须有前置说明页（Pre-Permission Screen）
- `Info.plist` 描述必须具体说明功能，禁止模糊表述：
  - ❌ "需要访问相机"
  - ✅ "用于录制行车视频，视频仅存储在您的设备上"

| 权限 | Key | 必要性说明 |
|------|-----|-----------|
| 相机 | NSCameraUsageDescription | 说明具体拍摄用途 |
| 麦克风 | NSMicrophoneUsageDescription | 说明录音场景 |
| 相册读取 | NSPhotoLibraryUsageDescription | 说明读取原因 |
| 相册写入 | NSPhotoLibraryAddUsageDescription | 可仅申请写入权限 |
| 位置（使用时）| NSLocationWhenInUseUsageDescription | 禁止申请 Always（除导航类） |

## 隐私合规
- 使用 ATT 框架前不得访问 IDFA（`AppTrackingTransparency`）
- **PrivacyInfo.xcprivacy** 必须声明：
  - 收集的数据类型
  - 使用目的
  - 是否用于追踪
- 第三方 SDK 的隐私说明必须合并（Firebase / Amplitude / Crashlytics 等）
- 隐私政策 URL 必须在提交前可公开访问（不能是 localhost）

## 常见拒绝原因 & 对策

| 条款 | 原因 | 对策 |
|------|------|------|
| 2.1 | 崩溃 / 明显 Bug | 真机 Release 模式完整测试 |
| 4.3 | 功能同质化 | 准备差异化功能说明文档 |
| 4.0 | 功能描述不符 | 截图和描述必须与实际一致 |
| 5.1.1 | 数据收集未说明 | 检查 PrivacyInfo + 隐私政策 |
| 3.1.1 | 绕过 IAP | 移除所有外部支付引导 |
| 2.3.3 | 截图含虚假 UI | 截图必须是真实设备截图 |
| 1.4 | 色情 / 赌博内容 | 用户生成内容需有审核机制 |

## 提审 Checklist
- [ ] 所有权限 Info.plist 描述已填写
- [ ] 订阅价格、周期在购买页清晰展示
- [ ] 恢复购买按钮可见
- [ ] 隐私政策 URL 可访问
- [ ] PrivacyInfo.xcprivacy 已配置
- [ ] 截图为真实设备截图（非模拟器）
- [ ] Release 模式真机测试无崩溃
- [ ] App 描述与实际功能一致
- [ ] 年龄分级填写正确
