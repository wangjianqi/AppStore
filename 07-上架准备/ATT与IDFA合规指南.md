# ATT 与 IDFA 合规指南

> 🎯 **本章目标**：理解 App Tracking Transparency 框架与 IDFA 的工作原理，掌握 ATT 授权请求的正确实现方式，了解不同授权状态下的广告标识符使用策略，规避审核被拒风险。

---

## 1. ATT 与 IDFA 概述

### 什么是 IDFA

IDFA（Identifier for Advertisers）是 Apple 为每台 iOS 设备分配的唯一标识符，专门用于广告追踪与归因。它允许广告主和广告网络识别用户跨应用的行为，从而实现精准广告投放和效果衡量。

IDFA 的核心特征：

- 每台设备全局唯一，但用户可以随时重置
- 同一设备上所有 App 获取到的 IDFA 值相同
- 用户在"设置 → 隐私 → 跟踪"中可全局关闭或重置
- 在 iOS 14.5 之前，IDFA 默认可用；之后需要用户明确授权

### 什么是 ATT

ATT（App Tracking Transparency）是 Apple 在 iOS 14.5（2021 年 4 月正式生效）引入的隐私框架，要求 App 在访问 IDFA 之前必须先获得用户的明确授权。这一框架的核心是 `ATTrackingManager` 类，它提供了标准的授权请求弹窗。

ATT 弹窗的呈现方式由系统统一控制，开发者无法自定义弹窗 UI，但可以在弹窗出现前展示自定义的引导界面来解释为何需要追踪权限。

### ATT 出现的背景：iOS 14.5 的隐私变革

在 iOS 14.5 之前，IDFA 默认对所有 App 可用，用户只能在系统设置中全局关闭追踪。这意味着绝大多数用户并不知道自己的行为正在被跨应用追踪，隐私保护形同虚设。

Apple 在 WWDC 2020 上宣布了 ATT 政策，原定随 iOS 14 发布，后推迟至 iOS 14.5（2021 年 4 月 26 日）正式强制执行。这一变革对移动广告行业产生了深远影响：

- 广告归因精度大幅下降，从确定性归因转向概率性归因
- 广告收入普遍下滑，尤其是依赖个性化广告的免费 App
- 推动了 SKAdNetwork 等隐私友好的归因方案发展
- 促使行业向"隐私优先"的方向转型

### IDFA vs 其他标识符对比

| 标识符 | 唯一性 | 跨应用 | 用户可控 | 用途 | ATT 要求 |
|--------|--------|--------|----------|------|----------|
| IDFA | 设备级唯一 | ✅ 可跨应用追踪 | 可重置/可关闭 | 广告追踪与归因 | ✅ 必须 |
| IDFV | 同一开发者旗下唯一 | ❌ 仅同一开发者 | 卸载后可能变化 | 内部分析 | ❌ 不需要 |
| 广告渠道标识符（OAID/Android ID） | 设备级唯一 | ✅ 可跨应用 | 因平台而异 | 安卓广告追踪 | N/A |
| 设备指纹 | 理论上唯一 | ✅ 可跨应用 | 用户难以控制 | 风控与反欺诈 | ⚠️ Apple 明确禁止 |

> 💡 **提示**：IDFV（Identifier for Vendor）是同一开发者在同一设备上的共享标识符。如果你的 App 有多个应用且属于同一开发者账号，IDFV 可以作为跨应用追踪的替代方案，但无法跨不同开发者追踪。

---

## 2. ATT 授权请求实现

### Info.plist 配置：NSUserTrackingUsageDescription

在请求 ATT 授权之前，必须在 Info.plist 中添加 `NSUserTrackingUsageDescription` 键，提供一段面向用户的说明文字。这段文字会显示在系统授权弹窗中，直接影响用户的授权决策。

```xml
<key>NSUserTrackingUsageDescription</key>
<string>我们需要跨应用追踪您的活动，以便为您推荐更相关的广告内容并提供更精准的个性化体验。</string>
```

> ⚠️ **警告**：如果未配置 `NSUserTrackingUsageDescription` 就调用 `requestTrackingAuthorization`，App 会直接崩溃。说明文字不得为空，且必须清晰解释追踪目的，否则可能被 App Store 审核拒绝。

### ATTrackingManager.AuthorizationStatus 四种状态

```swift
import AppTrackingTransparency

enum AuthorizationStatus: UInt {
    case notDetermined = 0
    case restricted = 1
    case denied = 2
    case authorized = 3
}
```

| 状态 | 含义 | IDFA 可用性 |
|------|------|-------------|
| `notDetermined` | 用户尚未被询问，首次安装默认状态 | ❌ 不应访问 |
| `restricted` | 设备受限（如家长控制），无法授权 | ❌ 不可访问 |
| `denied` | 用户明确拒绝或系统全局关闭追踪 | ❌ 返回全零 |
| `authorized` | 用户明确授权 | ✅ 可正常访问 |

### requestTrackingAuthorization 调用时机

调用时机是 ATT 合规的关键。过早请求会降低授权率，过晚请求则可能影响广告加载。

**推荐时机：**

1. 用户完成注册或首次引导流程后
2. 用户主动触发需要广告追踪的功能时（如点击"个性化推荐"）
3. 在展示自定义引导界面之后

**应避免的时机：**

1. App 启动后立即弹出（用户体验极差，授权率极低）
2. 在系统弹窗上叠加自定义弹窗（审核会被拒）
3. 在后台或不可见状态下请求

```swift
import AppTrackingTransparency
import AdSupport

func requestATTIfNeeded() {
    let status = ATTrackingManager.trackingAuthorizationStatus
    guard status == .notDetermined else { return }

    ATTrackingManager.requestTrackingAuthorization { newStatus in
        DispatchQueue.main.async {
            switch newStatus {
            case .authorized:
                let idfa = ASIdentifierManager.shared().advertisingIdentifier.uuidString
                print(idfa)
            case .denied, .restricted, .notDetermined:
                break
            @unknown default:
                break
            }
        }
    }
}
```

### SwiftUI 中封装 ATT 请求

```swift
import SwiftUI
import AppTrackingTransparency
import AdSupport

class ATTManager: ObservableObject {
    @Published var authorizationStatus: ATTrackingManager.AuthorizationStatus =
        ATTrackingManager.trackingAuthorizationStatus

    func requestAuthorization() {
        ATTrackingManager.requestTrackingAuthorization { [weak self] status in
            DispatchQueue.main.async {
                self?.authorizationStatus = status
            }
        }
    }

    var idfa: String {
        guard authorizationStatus == .authorized else { return "" }
        return ASIdentifierManager.shared().advertisingIdentifier.uuidString
    }
}

struct OnboardingView: View {
    @StateObject private var attManager = ATTManager()

    var body: some View {
        VStack(spacing: 24) {
            Text("个性化推荐")
                .font(.title)
                .bold()

            Text("允许跨应用追踪可以帮助我们为您推荐更感兴趣的内容，您随时可以在系统设置中更改此选择。")
                .font(.body)
                .multilineTextAlignment(.center)
                .padding(.horizontal, 32)

            if attManager.authorizationStatus == .notDetermined {
                Button("允许追踪") {
                    attManager.requestAuthorization()
                }
                .buttonStyle(.borderedProminent)

                Button("暂不") {
                    attManager.authorizationStatus = .denied
                }
                .buttonStyle(.bordered)
            } else {
                Text("当前授权状态：\(statusText)")
                    .font(.footnote)
                    .foregroundStyle(.secondary)
            }
        }
    }

    private var statusText: String {
        switch attManager.authorizationStatus {
        case .authorized: return "已授权"
        case .denied: return "已拒绝"
        case .restricted: return "受限"
        case .notDetermined: return "未决定"
        @unknown default: return "未知"
        }
    }
}
```

### 授权引导界面设计

研究表明，先展示自定义引导界面解释追踪目的，再弹出系统 ATT 弹窗，可以显著提高授权率。这种"先解释再请求"的策略被称为"预授权引导"（Pre-permission Prompt）。

**设计原则：**

1. **透明告知**：清晰说明追踪什么数据、用于什么目的
2. **用户利益**：强调授权对用户的价值（如更好的推荐、更少的无关广告）
3. **尊重选择**：提供"暂不"选项，不强迫用户
4. **视觉友好**：使用插画或图标辅助说明，避免大段文字
5. **时机恰当**：在用户完成核心操作、对 App 产生信任后展示

> 💡 **提示**：自定义引导界面的"允许"按钮应触发系统 ATT 弹窗，而非直接标记为已授权。切勿在自定义界面中模拟系统弹窗的外观和行为，否则会被审核拒绝。

---

## 3. 不同授权状态下的策略

### authorized：正常使用 IDFA

当用户授权后，可以正常获取 IDFA 并用于以下场景：

- 广告归因：追踪广告点击到安装的转化
- 受众定向：基于用户行为构建广告受众
- 频次控制：限制同一用户看到同一广告的次数
- 广告去重：防止同一转化被重复计算

```swift
import AdSupport

func fetchIDFA() -> String {
    let manager = ASIdentifierManager.shared()
    guard manager.isAdvertisingTrackingEnabled else { return "" }
    return manager.advertisingIdentifier.uuidString
}
```

### denied：使用 IDFV 或归因替代方案

当用户拒绝授权后，IDFA 返回全零值 `00000000-0000-0000-0000-000000000000`，此时应采用替代方案：

**方案一：使用 IDFV**

```swift
import UIKit

func fetchIDFV() -> String {
    return UIDevice.current.identifierForVendor?.uuidString ?? ""
}
```

IDFV 的局限在于只能标识同一开发者旗下的用户，无法跨开发者追踪。

**方案二：SKAdNetwork 归因**

使用 Apple 提供的 SKAdNetwork 进行安装归因，无需 IDFA 即可获得归因数据。

**方案三：自建归因体系**

通过登录账号体系实现用户识别，结合服务端归因模型进行广告效果衡量。

### notDetermined：首次启动的引导策略

`notDetermined` 状态意味着用户尚未被询问。这是提高授权率的黄金窗口，应精心设计引导流程：

1. 延迟请求：不要在 App 启动时立即弹出，等待用户完成核心操作
2. 预授权引导：先展示自定义界面解释价值，再触发系统弹窗
3. 场景触发：在用户明确需要个性化功能时请求
4. A/B 测试：对不同引导文案和时机进行测试，优化授权率

### restricted：受限设备的处理

`restricted` 状态通常出现在家长控制或企业设备管理场景下。此时：

- 无法请求授权，系统不会弹出 ATT 弹窗
- IDFA 不可用
- 应优雅降级，使用不依赖追踪的替代方案
- 不要反复尝试请求授权

```swift
func handleTrackingStatus() {
    let status = ATTrackingManager.trackingAuthorizationStatus
    switch status {
    case .authorized:
        let idfa = ASIdentifierManager.shared().advertisingIdentifier.uuidString
        configureAdNetwork(with: idfa)
    case .denied, .restricted:
        configureAdNetworkWithoutTracking()
    case .notDetermined:
        showPrePermissionOnboarding()
    @unknown default:
        configureAdNetworkWithoutTracking()
    }
}
```

### 策略对比表

| 授权状态 | IDFA | 推荐策略 | 广告影响 |
|----------|------|----------|----------|
| authorized | ✅ 可用 | 正常使用 IDFA 进行归因和定向 | 个性化广告，收入最高 |
| denied | ❌ 全零 | 使用 IDFV + SKAdNetwork | 非个性化广告，收入下降 |
| notDetermined | ❌ 未请求 | 展示引导界面后请求授权 | 暂时无法追踪 |
| restricted | ❌ 不可用 | 使用 IDFV + SKAdNetwork，优雅降级 | 非个性化广告 |

---

## 4. ATT 与广告变现

### ATT 对广告收入的影响

ATT 政策实施后，行业数据普遍显示广告收入受到显著影响：

| 指标 | ATT 前水平 | ATT 后变化 |
|------|-----------|-----------|
| ATT 授权率（全球） | N/A | 约 25%-40% |
| CPM（千人展示成本） | 基准 | 下降 15%-30% |
| 个性化广告 eCPM | 基准 | 下降 30%-50% |
| 非个性化广告 eCPM | 基准 | 约为个性化的 30%-50% |
| 归因精度 | 确定性归因 | 大幅下降 |

> 💡 **提示**：中国市场的 ATT 授权率普遍高于全球平均水平，部分报告显示可达 40%-55%，这与国内用户对个性化推荐的接受度较高有关。

### SKAdNetwork 概述：ATT 时代的归因方案

SKAdNetwork 是 Apple 提供的隐私友好型归因框架，无需用户授权即可进行广告归因。其核心机制如下：

1. **广告展示**：App 展示广告时，广告平台注册一个印象（Impression）
2. **安装转化**：用户安装并首次打开 App 后，系统验证安装
3. **回传数据**：系统在 24-48 小时后向广告平台回传归因数据
4. **数据限制**：回传数据不包含用户级标识，仅提供聚合级信息

```swift
import StoreKit

func registerAppForAdNetworkAttribution() {
    SKAdNetwork.registerAppForAdNetworkAttribution()
}

func updateConversionValue(_ value: Int) {
    SKAdNetwork.updateConversionValue(value)
}
```

### SKAdNetwork 2.0+ 的改进

| 版本 | 关键改进 |
|------|----------|
| SKAdNetwork 1.0 | 基础安装归因，6 位转化值 |
| SKAdNetwork 2.0 | 支持 View-Through 归因（浏览转化） |
| SKAdNetwork 2.1 | 增加防欺诈机制，限制无效回传 |
| SKAdNetwork 2.2 | 支持多层转化值（0-63），延迟回传 |
| SKAdNetwork 4.0 | 三层转化值结构，支持多次回传，更细粒度的归因数据 |

SKAdNetwork 4.0 的三层转化值结构：

- **细粒度值**（Fine-grained）：0-63，对应不同转化事件
- **粗粒度值**（Coarse）：low/medium/high，在细粒度值无法回传时降级使用
- **锁定窗口**：支持多次回传，提供更丰富的转化数据

### 国内广告平台的 ATT 适配情况

| 广告平台 | ATT 适配状态 | SKAdNetwork 支持 | 备注 |
|----------|-------------|-----------------|------|
| 穿山甲（字节跳动） | ✅ 已适配 | ✅ 支持 | 提供预授权引导组件 |
| 优量汇（腾讯） | ✅ 已适配 | ✅ 支持 | 支持 C2C 归因 |
| 百度联盟 | ✅ 已适配 | ✅ 支持 | 支持 OAID 映射 |
| 快手联盟 | ✅ 已适配 | ✅ 支持 | 短视频场景优化 |
| Mintegral | ✅ 已适配 | ✅ 支持 | 出海场景推荐 |
| AdMob（Google） | ✅ 已适配 | ✅ 支持 | UAC 归因适配 |

### 广告变现 App 的 ATT 最佳实践

1. **延迟初始化广告 SDK**：在 ATT 授权完成后再初始化，确保 SDK 能获取到 IDFA
2. **分层广告策略**：授权用户展示个性化广告（高 eCPM），未授权用户展示非个性化广告
3. **优化引导文案**：A/B 测试不同文案，找到最高授权率的表述
4. **场景化请求**：在用户与广告交互时请求授权，而非启动时
5. **监控授权率**：持续追踪不同版本和场景下的授权率变化

```swift
func initializeAdSDK() {
    let status = ATTrackingManager.trackingAuthorizationStatus

    switch status {
    case .authorized:
        AdSDK.initialize(personalizedAds: true)
    case .denied, .restricted:
        AdSDK.initialize(personalizedAds: false)
    case .notDetermined:
        showATTOnboarding { granted in
            AdSDK.initialize(personalizedAds: granted)
        }
    @unknown default:
        AdSDK.initialize(personalizedAds: false)
    }
}
```

---

## 5. 审核合规要点

### 何时必须请求 ATT 授权

以下场景**必须**请求 ATT 授权：

1. 访问 IDFA 用于广告追踪或归因
2. 将设备级数据与第三方共享用于定向广告
3. 使用第三方广告 SDK（如 AdMob、穿山甲）展示个性化广告
4. 在 App 中嵌入追踪像素或第三方分析工具追踪跨应用行为
5. 收集用户行为数据用于构建广告受众画像

### 何时不需要请求 ATT 授权

以下场景**不需要** ATT 授权：

1. 仅使用 IDFV 进行同一开发者旗下的内部分析
2. 基于用户登录账号的第一方数据分析（不跨应用追踪）
3. 使用 SKAdNetwork 进行归因（无需 IDFA）
4. App 内部的行为分析（不与第三方共享设备级数据）
5. 欺诈检测和安全防护（Apple 明确豁免）

> ⚠️ **警告**：如果你的 App 包含第三方广告 SDK，即使你本人不直接使用 IDFA，SDK 也可能访问它。这种情况下仍需请求 ATT 授权，否则审核会被拒。务必检查所有第三方 SDK 的文档，确认其是否访问 IDFA。

### 常见被拒原因与应对

| 被拒原因 | 具体表现 | 应对方案 |
|----------|----------|----------|
| 未配置 NSUserTrackingUsageDescription | 调用 ATT API 但未提供说明 | 在 Info.plist 中添加完整的说明文字 |
| 说明文字模糊或误导 | 如"需要权限"等笼统描述 | 明确说明追踪目的和对用户的价值 |
| 强制要求授权 | 拒绝授权后 App 无法正常使用 | 确保拒绝授权后核心功能仍可用 |
| 自定义弹窗模拟系统弹窗 | 自定义 UI 与系统 ATT 弹窗外观相似 | 自定义引导界面应与系统弹窗有明显区别 |
| 在后台请求授权 | App 进入后台后弹出 ATT 弹窗 | 仅在前台且用户可见时请求 |
| 违规使用设备指纹 | 收集设备信息拼接生成唯一标识 | 不使用任何形式的设备指纹替代 IDFA |
| 未授权但访问 IDFA | 未调用 ATT 就直接读取 IDFA | 确保先检查授权状态再访问 |

### 隐私清单中 ATT 的声明

从 2024 年春季起，Apple 要求在隐私清单（PrivacyInfo.xcprivacy）中声明 App 使用了哪些隐私 API 及其用途。涉及 ATT 的声明包括：

```xml
<key>NSPrivacyTracking</key>
<true/>

<key>NSPrivacyTrackingDomains</key>
<array>
    <string>ads.example.com</string>
    <string>tracking.example.com</string>
</array>

<key>NSPrivacyCollectedDataTypes</key>
<array>
    <dict>
        <key>NSPrivacyCollectedDataType</key>
        <string>NSPrivacyCollectedDataTypeAdvertisingData</string>
        <key>NSPrivacyCollectedDataTypeLinked</key>
        <true/>
        <key>NSPrivacyCollectedDataTypeTracking</key>
        <true/>
        <key>NSPrivacyCollectedDataTypePurposes</key>
        <array>
            <string>NSPrivacyCollectedDataTypePurposeAnalytics</string>
            <string>NSPrivacyCollectedDataTypePurposeAdvertising</string>
        </array>
    </dict>
</array>
```

- `NSPrivacyTracking`：声明 App 是否进行追踪
- `NSPrivacyTrackingDomains`：列出追踪涉及的域名
- `NSPrivacyCollectedDataTypes`：详细声明收集的数据类型、是否关联用户身份、是否用于追踪

### 与 PIPL 合规的交叉考虑

中国《个人信息保护法》（PIPL）对个人信息处理有严格要求，与 ATT 合规存在交叉：

| 维度 | ATT 要求 | PIPL 要求 | 交叉处理 |
|------|----------|-----------|----------|
| 知情同意 | 系统弹窗获取授权 | 单独同意 + 明确告知 | ATT 弹窗 + 隐私政策双重告知 |
| 目的限制 | 说明追踪用途 | 不得超出约定目的 | NSUserTrackingUsageDescription 需与隐私政策一致 |
| 最小必要 | 仅追踪必要数据 | 收集最小必要信息 | 不收集与广告无关的设备数据 |
| 数据跨境 | 无特别要求 | 跨境传输需安全评估 | 广告数据回传境外需合规 |
| 撤回同意 | 系统设置中可关闭 | 用户可随时撤回 | 提供便捷的撤回入口 |

> 💡 **提示**：对于面向中国市场的 App，建议同时满足 ATT 和 PIPL 的要求。在隐私政策中明确说明 IDFA 的收集目的和方式，并提供独立的同意机制，而非仅依赖 ATT 系统弹窗。

---

## 小结

| 主题 | 核心要点 |
|------|----------|
| IDFA 概述 | 广告标识符，跨应用追踪，用户可重置，iOS 14.5 后需 ATT 授权 |
| ATT 框架 | 四种授权状态，系统弹窗不可自定义，必须配置 Usage Description |
| 授权请求 | 延迟请求 + 预授权引导，时机选择影响授权率，SwiftUI 封装 |
| 授权策略 | authorized 用 IDFA，denied 用 IDFV/SKAdNetwork，notDetermined 需引导 |
| 广告变现 | 授权率影响 eCPM，分层广告策略，SKAdNetwork 替代归因 |
| 审核合规 | 必须请求场景 vs 豁免场景，常见被拒原因，隐私清单声明 |
| PIPL 交叉 | 双重告知，目的限制，最小必要，数据跨境，撤回同意 |

← [隐私与权限合规](./隐私与权限合规.md) | [隐私清单（Privacy Manifest）](./隐私清单Privacy-Manifest.md) →