---
name: appstore-compliance
description: 涉及订阅、支付、权限申请、隐私数据、用户引导、审核准备、ATT、Privacy Manifest、隐私政策、ICP 备案的任务
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

---

## ATT（App Tracking Transparency）

### 完整请求流程

```swift
import AppTrackingTransparency
import AdSupport

final class TrackingManager {
    static let shared = TrackingManager()

    var isAuthorized: Bool {
        ATTrackingManager.trackingAuthorizationStatus == .authorized
    }

    func requestPermission(completion: @escaping (Bool) -> Void) {
        ATTrackingManager.requestTrackingAuthorization { status in
            DispatchQueue.main.async {
                switch status {
                case .authorized:
                    completion(true)
                case .denied, .restricted, .notDetermined:
                    completion(false)
                @unknown default:
                    completion(false)
                }
            }
        }
    }

    func checkStatus() -> ATTrackingManager.AuthorizationStatus {
        ATTrackingManager.trackingAuthorizationStatus
    }
}
```

### ATT 规范
- **禁止在 App 启动时立即请求 ATT**，必须在用户理解追踪目的后再请求
- 推荐时机：用户点击"个性化推荐"开关 / 进入含广告的页面 / 首次使用相关功能时
- `Info.plist` 中 `NSUserTrackingUsageDescription` 必须具体说明用途：
  - ❌ "需要追踪您的活动"
  - ✅ "用于向您推荐更感兴趣的内容，您可随时在设置中关闭"
- ATT 被拒绝后，禁止反复弹窗，提供设置页引导

### 前置说明页（Pre-ATT Screen）

```swift
final class TrackingPermissionVC: UIViewController {
    private let titleLabel = UILabel()
    private let descriptionLabel = UILabel()
    private let allowButton = UIButton()
    private let skipButton = UIButton()

    private func setupUI() {
        titleLabel.text = "个性化推荐"
        descriptionLabel.text = "允许追踪可以帮助我们为您推荐更感兴趣的内容，您可随时在系统设置中关闭此权限。"
        allowButton.setTitle("允许追踪", for: .normal)
        skipButton.setTitle("暂不开启", for: .normal)
    }

    @objc private func allowTapped() {
        TrackingManager.shared.requestPermission { [weak self] authorized in
            self?.dismiss(animated: true)
        }
    }

    @objc private func skipTapped() {
        dismiss(animated: true)
    }
}
```

---

## 权限申请规范

### 原则
- **在用户理解使用场景后再申请，禁止启动即弹权限**
- 每个权限必须有前置说明页（Pre-Permission Screen）
- `Info.plist` 描述必须具体说明功能，禁止模糊表述

### 权限列表

| 权限 | Key | 必要性说明 | 前置说明 |
|------|-----|-----------|---------|
| 相机 | NSCameraUsageDescription | 说明具体拍摄用途 | 展示拍摄功能预览 |
| 麦克风 | NSMicrophoneUsageDescription | 说明录音场景 | 展示录音功能说明 |
| 相册读取 | NSPhotoLibraryUsageDescription | 说明读取原因 | 展示选择照片场景 |
| 相册写入 | NSPhotoLibraryAddUsageDescription | 可仅申请写入权限 | 展示保存功能说明 |
| 位置（使用时）| NSLocationWhenInUseUsageDescription | 禁止申请 Always（除导航类） | 展示地图/定位功能 |
| 位置（始终）| NSLocationAlwaysAndWhenInUseUsageDescription | 仅导航/运动类 App | 必须先申请 WhenInUse |
| 通讯录 | NSContactsUsageDescription | 审核极严，必须有核心功能 | 展示邀请好友场景 |
| 通知 | 无需 Info.plist | 需代码请求 | 展示通知内容预览 |

### 权限请求封装

```swift
final class PermissionManager {
    static func requestCamera(completion: @escaping (Bool) -> Void) {
        let status = AVCaptureDevice.authorizationStatus(for: .video)
        switch status {
        case .notDetermined:
            AVCaptureDevice.requestAccess(for: .video) { granted in
                DispatchQueue.main.async { completion(granted) }
            }
        case .authorized:
            completion(true)
        case .denied, .restricted:
            showSettingsAlert()
            completion(false)
        @unknown default:
            completion(false)
        }
    }

    static func requestPhotoLibrary(completion: @escaping (Bool) -> Void) {
        let status = PHPhotoLibrary.authorizationStatus(for: .readWrite)
        switch status {
        case .notDetermined:
            PHPhotoLibrary.requestAuthorization(for: .readWrite) { newStatus in
                DispatchQueue.main.async { completion(newStatus == .authorized || newStatus == .limited) }
            }
        case .authorized, .limited:
            completion(true)
        case .denied, .restricted:
            showSettingsAlert()
            completion(false)
        @unknown default:
            completion(false)
        }
    }

    private static func showSettingsAlert() {
        guard let settingsURL = URL(string: UIApplication.openSettingsURLString) else { return }
        let alert = UIAlertController(
            title: "需要权限",
            message: "请在设置中开启权限以使用此功能",
            preferredStyle: .alert
        )
        alert.addAction(UIAlertAction(title: "去设置", style: .default) { _ in
            UIApplication.shared.open(settingsURL)
        })
        alert.addAction(UIAlertAction(title: "取消", style: .cancel))
        (UIApplication.shared.connectedScenes.first?.delegate as? SceneDelegate)?
            .window?.rootViewController?.present(alert, animated: true)
    }
}
```

---

## PrivacyInfo.xcprivacy 配置

### 必须声明的数据类型

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
            <string>NSPrivacyCollectedDataTypeEmailAddress</string>
            <key>NSPrivacyCollectedDataTypeLinked</key>
            <true/>
            <key>NSPrivacyCollectedDataTypeTracking</key>
            <false/>
            <key>NSPrivacyCollectedDataTypePurposes</key>
            <array>
                <string>NSPrivacyCollectedDataTypePurposeAppFunctionality</string>
            </array>
        </dict>
        <dict>
            <key>NSPrivacyCollectedDataType</key>
            <string>NSPrivacyCollectedDataTypePhotosAndVideos</string>
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
            <string>NSPrivacyAccessedAPITypeFileTimestamp</string>
            <key>NSPrivacyAccessedAPITypeReasons</key>
            <array>
                <string>DDA9B8E6</string>
            </array>
        </dict>
        <dict>
            <key>NSPrivacyAccessedAPIType</key>
            <string>NSPrivacyAccessedAPITypeDiskSpace</string>
            <key>NSPrivacyAccessedAPITypeReasons</key>
            <array>
                <string>E5F3D7A6</string>
            </array>
        </dict>
    </array>
</dict>
</plist>
```

### 第三方 SDK 合并
- Firebase / Amplitude / Crashlytics 等第三方 SDK 必须合并其隐私声明
- 检查方式：`grep -r "NSPrivacyCollectedDataTypes" Pods/` 或查看 SPM 依赖的 PrivacyInfo
- Apple 2024 年起强制要求，缺失会被拒审

---

## 隐私政策模板

### 必须包含的章节

```markdown
# 隐私政策

最后更新日期：YYYY年MM月DD日

## 1. 信息收集
我们收集以下信息：
- 注册信息：邮箱地址（用于账号注册）
- 使用数据：App 使用记录（用于改善体验）
- 设备信息：设备型号、操作系统版本（用于兼容性优化）

## 2. 信息使用
收集的信息仅用于：
- 提供 App 核心功能
- 改善用户体验
- 提供客户支持

## 3. 信息共享
我们不会出售您的个人信息。仅在以下情况共享：
- 获得您的明确同意
- 法律法规要求

## 4. 数据存储与安全
- 数据存储于 [服务器位置]
- 采用行业标准加密保护

## 5. 您的权利
- 访问您的个人数据
- 要求删除您的数据
- 撤回同意

## 6. 第三方 SDK
- Firebase Analytics：使用分析
- Crashlytics：崩溃报告

## 7. 儿童隐私
本 App 不面向 13 岁以下儿童。

## 8. 联系我们
邮箱：privacy@example.com
```

### 规范
- 隐私政策 URL 必须在提交前可公开访问（不能是 localhost）
- 中文 App 必须提供中文版隐私政策
- 更新隐私政策后需同步更新 App Store Connect 中的链接

---

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
| 2.5.6 | 未声明 API 原因 | 补充 PrivacyInfo.xcprivacy 的 API 声明 |
| 5.1.2 | 数据收集超出声明 | 确保 PrivacyInfo 覆盖所有收集项 |
| 3.2 | 重复 App | 增加差异化功能，避免模板化 |

---

## 国内上架额外要求

### ICP 备案
- App 上架前必须完成 ICP 备案
- 备案号必须在 App 内展示（通常放在"关于"页面）
- 域名备案与 App 备案需分开办理

### 实名认证
- 用户发布内容类 App 必须接入实名认证
- 收集手机号需在隐私政策中明确说明

### 内容审核
- 用户生成内容（UGC）必须有审核机制
- 敏感词过滤：接入第三方内容安全 API

---

## 提审 Checklist
- [ ] 所有权限 Info.plist 描述已填写
- [ ] ATT 请求有前置说明页
- [ ] 订阅价格、周期在购买页清晰展示
- [ ] 恢复购买按钮可见
- [ ] 隐私政策 URL 可访问
- [ ] PrivacyInfo.xcprivacy 已配置（含 API 声明）
- [ ] 截图为真实设备截图（非模拟器）
- [ ] Release 模式真机测试无崩溃
- [ ] App 描述与实际功能一致
- [ ] 年龄分级填写正确
- [ ] ICP 备案号已展示（国内上架）
- [ ] 第三方 SDK 隐私声明已合并
