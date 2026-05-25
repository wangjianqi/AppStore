# 个人信息保护法（PIPL）合规指南

> 🎯 **本章目标**：理解《个人信息保护法》的核心要求，掌握 App 开发中的合规要点，建立完整的合规检查清单，学会处理用户同意、数据跨境、用户权利等关键合规场景，规避因违规导致的下架风险。

---

## PIPL 概述

### 立法背景与意义

《中华人民共和国个人信息保护法》（简称 PIPL）于 2021 年 8 月 20 日经十三届全国人大常委会第三十次会议通过，**2021 年 11 月 1 日起正式施行**。这是中国首部全面规范个人信息保护的专门法律，标志着中国个人信息保护进入了有法可依的新时代。

对 App 开发者而言，PIPL 的施行意味着：

- 个人信息处理不再是"灰色地带"，有了明确的法律红线
- 违规成本大幅提升，最高可处 **5000 万元或上一年度营业额 5%** 的罚款
- 工信部定期通报违规 App，严重者直接下架
- App Store 审核也逐步加强对合规性的要求

### 核心原则

PIPL 确立了个人信息处理的五大核心原则：

| 原则 | 含义 | 对 App 开发的影响 |
|------|------|-------------------|
| **合法** | 处理个人信息必须有合法依据 | 需获得用户同意或符合其他法定情形 |
| **正当** | 不得以欺诈、诱骗等方式收集信息 | 不得通过误导性 UI 获取用户同意 |
| **必要** | 只收集实现功能所必需的最少信息 | 禁止过度收集，拒绝"捆绑授权" |
| **诚信** | 不得违背诚实信用原则 | 隐私政策必须真实、完整、与实际行为一致 |
| **公开透明** | 公开处理规则，保证用户知情权 | 隐私政策必须清晰可访问，不得隐藏 |

### 适用范围

PIPL 适用于以下情形：

1. **在中国境内**处理自然人个人信息的活动
2. 在中国境外，**以向境内自然人提供产品或服务为目的**的处理活动
3. 分析、评估境内自然人的行为

> 💡 **提示**：即使你的服务器部署在海外，只要 App 面向中国用户提供服务，就受 PIPL 管辖。

### 关键定义

| 术语 | 定义 | App 中的典型示例 |
|------|------|------------------|
| **个人信息** | 以电子或其他方式记录的与已识别或可识别的自然人有关的各种信息 | 用户名、手机号、设备 ID、位置信息 |
| **敏感个人信息** | 一旦泄露或非法使用，容易导致自然人人格尊严或人身、财产安全受到危害的个人信息 | 身份证号、生物识别、金融账户、行踪轨迹 |
| **个人信息处理者** | 在个人信息处理活动中自主决定处理目的、处理方式的组织或个人 | App 开发者/运营公司 |

### PIPL 与 GDPR 对比

| 对比维度 | PIPL（中国） | GDPR（欧盟） |
|----------|-------------|--------------|
| 施行时间 | 2021 年 11 月 1 日 | 2018 年 5 月 25 日 |
| 适用范围 | 境内处理 + 境外向境内提供服务 | 欧盟境内处理 + 境外向欧盟居民提供服务 |
| 合法依据 | 同意、合同履行、法定义务、突发公共卫生事件、公共利益、合理范围 | 同意、合同、法定义务、重大利益、公共利益、正当利益 |
| 敏感信息 | 需取得**单独同意** | 需取得**明确同意** |
| 跨境传输 | 需满足安全评估/标准合同/认证等条件 | 需满足充分性认定/标准合同/约束性企业规则等 |
| 用户权利 | 知情权、决定权、查阅权、复制权、更正权、删除权、可携带权 | 访问权、更正权、删除权、限制处理权、可携带权、反对权 |
| 最高罚款 | 5000 万元或上一年营业额 5% | 2000 万欧元或全球年营业额 4% |
| 数据本地化 | 关键信息基础设施运营者需境内存储 | 无强制本地化，但跨境需保障 |
| DPO 要求 | 处理信息达到规定数量需指定个人信息保护负责人 | 公共机构及核心业务涉及大规模监控或敏感数据需指定 DPO |

---

## App 开发者必须遵守的核心要求

### 最小必要原则

最小必要原则是 PIPL 对 App 开发者最核心的要求之一。它要求你**只收集实现功能所必需的最少信息**，不得因为"可能有用"就提前收集。

实际操作中，这意味着：

- 注册时只要求必要信息（手机号即可，不需要同时要身份证号）
- 功能触发时才请求对应权限（不用相机时不请求相机权限）
- 不收集与功能无关的设备信息（如天气预报 App 不需要通讯录）
- 后台不持续采集位置等敏感数据

### 知情同意原则

在收集用户个人信息前，必须**以清晰易懂的方式告知用户**并获得明确同意。同意必须是自愿的、具体的，不能是默认勾选或强制捆绑。

### 目的限制原则

收集的个人信息**只能用于告知用户的目的**，不得超出约定范围使用。如果需要用于新目的，必须重新获取用户同意。

### 安全保障原则

作为个人信息处理者，必须采取**必要的技术和管理措施**保护用户数据安全，包括：

- 数据传输加密（HTTPS/TLS）
- 数据存储加密
- 访问控制与权限管理
- 安全审计与日志记录
- 数据泄露应急预案

### 核心要求实操解读

| 要求 | 违规示例 | 合规做法 |
|------|----------|----------|
| 最小必要 | 天气 App 要求通讯录权限 | 仅请求位置权限，且允许手动输入城市 |
| 最小必要 | 注册时强制填写真实姓名、身份证 | 仅要求手机号验证，身份信息按需补充 |
| 知情同意 | 隐私政策默认勾选"我已阅读" | 用户主动点击"同意"按钮，且可查看完整政策 |
| 知情同意 | 一键授权所有权限 | 分场景逐项请求权限，用户可逐项拒绝 |
| 目的限制 | 收集位置用于导航，实际还用于广告推送 | 位置仅用于导航，广告推送需另行获取同意 |
| 目的限制 | 以"账户安全"为由收集通讯录 | 通讯录仅用于用户主动发起的社交功能 |
| 安全保障 | 用户密码明文存储 | 密码使用 bcrypt 加密存储，传输使用 HTTPS |
| 安全保障 | 任意员工可访问用户数据库 | 实施最小权限原则，操作需审批并留痕 |

> ⚠️ **警告**：工信部每月定期通报违规 App，"超范围收集个人信息"和"强制索权"是最常见的违规类型。被通报后通常限期整改，逾期未改将面临下架。

---

## 用户同意机制实现

### 首次启动时的隐私弹窗

根据 PIPL 和工信部要求，App 首次启动时必须通过弹窗等方式向用户展示隐私政策摘要，并获得用户**明确同意**后才能收集个人信息。

合规要点：

- 弹窗必须在任何个人信息收集行为之前展示
- 必须提供"同意"和"不同意"两个选项
- "不同意"不应导致 App 直接退出，应允许使用基本功能
- 隐私政策链接必须可点击查看全文

### 单独同意

处理**敏感个人信息**时，PIPL 要求取得用户的**单独同意**，不能与一般性同意捆绑在一起。单独同意意味着：

- 必须有独立的提示和确认流程
- 不能与其他授权混在一起
- 用户必须明确知道正在授权的是哪项敏感信息

### 同意的撤回机制

PIPL 赋予用户撤回同意的权利。App 必须：

- 提供便捷的撤回同意入口
- 撤回同意的操作不应比给出同意更复杂
- 撤回同意后应停止相应的信息处理活动
- 撤回同意不影响撤回前基于同意已进行的处理活动

### SwiftUI 实现隐私同意界面

```swift
import SwiftUI

struct PrivacyConsentView: View {
    @AppStorage("hasConsentedPrivacy") private var hasConsented = false
    @AppStorage("consentDate") private var consentDate: Double = 0
    @State private var showFullPolicy = false
    @State private var showDisagreeAlert = false
    let onConsent: (Bool) -> Void

    var body: some View {
        VStack(spacing: 24) {
            Image(systemName: "shield.checkered")
                .font(.system(size: 48))
                .foregroundStyle(.blue)

            Text("隐私政策提示")
                .font(.title2.bold())

            Text("在使用我们的服务前，请您仔细阅读并理解《隐私政策》。我们将严格按照法律法规保护您的个人信息安全。")
                .font(.body)
                .foregroundStyle(.secondary)
                .multilineTextAlignment(.center)
                .padding(.horizontal)

            Button {
                showFullPolicy = true
            } label: {
                Text("查看完整《隐私政策》")
                    .font(.subheadline)
                    .underline()
            }

            Spacer()

            VStack(spacing: 12) {
                Button {
                    hasConsented = true
                    consentDate = Date().timeIntervalSince1970
                    onConsent(true)
                } label: {
                    Text("同意并继续")
                        .font(.headline)
                        .frame(maxWidth: .infinity)
                        .padding()
                        .background(.blue)
                        .foregroundStyle(.white)
                        .clipShape(RoundedRectangle(cornerRadius: 12))
                }

                Button {
                    showDisagreeAlert = true
                } label: {
                    Text("不同意")
                        .font(.subheadline)
                        .foregroundStyle(.secondary)
                }
            }
            .padding(.horizontal)
        }
        .padding()
        .sheet(isPresented: $showFullPolicy) {
            NavigationStack {
                ScrollView {
                    Text(privacyPolicyText)
                        .padding()
                }
                .navigationTitle("隐私政策")
                .navigationBarTitleDisplayMode(.inline)
            }
        }
        .alert("不同意隐私政策", isPresented: $showDisagreeAlert) {
            Button("继续使用基础功能") {
                onConsent(false)
            }
            Button("退出", role: .destructive) {
                exit(0)
            }
        } message: {
            Text("不同意隐私政策将无法使用需要个人信息的功能，但您可以继续使用基础功能。")
        }
    }
}
```

### 同意记录的存储与审计

合规实践中，仅仅获取同意还不够，还需要**留存同意记录**以备审计。记录应包含：

- 用户标识
- 同意时间（精确到秒）
- 同意的具体内容版本（隐私政策版本号）
- 同意方式（弹窗点击、页面勾选等）
- 设备信息（设备型号、操作系统版本）

```swift
import Foundation

struct ConsentRecord: Codable {
    let userID: String
    let consentType: String
    let policyVersion: String
    let timestamp: Date
    let deviceModel: String
    let osVersion: String
}

class ConsentManager {
    static let shared = ConsentManager()
    private let defaults = UserDefaults.standard
    private let recordsKey = "consent_records"

    func recordConsent(userID: String, type: String, policyVersion: String) {
        let record = ConsentRecord(
            userID: userID,
            consentType: type,
            policyVersion: policyVersion,
            timestamp: Date(),
            deviceModel: UIDevice.current.model,
            osVersion: UIDevice.current.systemVersion
        )

        var records = fetchRecords()
        records.append(record)
        if let data = try? JSONEncoder().encode(records) {
            defaults.set(data, forKey: recordsKey)
        }
    }

    func fetchRecords() -> [ConsentRecord] {
        guard let data = defaults.data(forKey: recordsKey),
              let records = try? JSONDecoder().decode([ConsentRecord].self, from: data) else {
            return []
        }
        return records
    }

    func hasConsented(to type: String, policyVersion: String) -> Bool {
        fetchRecords().contains { $0.consentType == type && $0.policyVersion == policyVersion }
    }
}
```

### 不同场景下的同意策略

| 场景 | 同意方式 | 实现要点 |
|------|----------|----------|
| 首次启动 | 隐私弹窗 | 展示摘要 + 全文链接，同意后方可收集信息 |
| 注册账号 | 注册流程内确认 | 隐私政策链接置于注册按钮旁 |
| 请求位置权限 | 系统弹窗 + 自定义说明 | 先用自定义 UI 解释用途，再触发系统弹窗 |
| 收集生物识别 | 单独同意弹窗 | 独立提示，明确说明用途和存储方式 |
| 开启推送通知 | 系统弹窗 | 说明推送用途，允许用户拒绝 |
| 信息用于广告 | 独立开关 | 默认关闭，用户主动开启 |
| 隐私政策更新 | 弹窗提示变更内容 | 仅提示变更部分，重新获取同意 |

---

## 敏感个人信息处理

### 敏感个人信息的定义与范围

PIPL 第二十八条将以下类型的信息定义为敏感个人信息：

| 类别 | 具体示例 | App 中的常见场景 |
|------|----------|------------------|
| 生物识别 | 人脸、指纹、声纹、虹膜 | Face ID 登录、人脸核身 |
| 宗教信仰 | 宗教信仰信息 | 社交 App 个人资料 |
| 医疗健康 | 病历、健康数据、用药记录 | 健康 App、在线问诊 |
| 金融账户 | 银行卡号、支付密码、征信信息 | 支付 App、借贷 App |
| 行踪轨迹 | GPS 定位轨迹、出行记录 | 导航 App、打车 App |
| 未成年人信息 | 不满 14 周岁未成年人的信息 | 儿童教育 App、游戏 App |
| 其他 | 身份证号、护照号 | 实名认证、KYC |

### 处理敏感信息的额外要求

处理敏感个人信息时，PIPL 要求满足以下额外条件：

1. **具有特定的目的和充分的必要性** — 不是"可能用到"，而是"必须用到"
2. **取得个人的单独同意** — 不能与一般同意捆绑
3. **处理前进行个人信息保护影响评估** — 评估泄露风险和影响范围
4. **向个人告知处理的必要性及影响** — 明确告知用户为什么必须收集、可能的影响

### App 中常见的敏感信息场景

| 场景 | 涉及的敏感信息 | 合规建议 |
|------|----------------|----------|
| 实名认证 | 身份证号、姓名 | 仅在需要实名认证的功能中收集，用后即删或脱敏存储 |
| Face ID 登录 | 面部生物特征 | 使用 LAContext 本地验证，不上传面部数据到服务器 |
| 位置追踪 | 行踪轨迹 | 提供仅使用时定位选项，避免始终定位 |
| 在线支付 | 银行卡号、支付信息 | 通过第三方支付 SDK 处理，不自行存储 |
| 健康记录 | 健康数据 | 使用 HealthKit 框架，数据存储在用户设备 |
| 儿童内容 | 未成年人信息 | 需取得监护人同意，设置专门的儿童隐私保护规则 |

### SwiftUI 权限请求与说明文案

```swift
import SwiftUI
import CoreLocation

class LocationManager: NSObject, ObservableObject, CLLocationManagerDelegate {
    private let manager = CLLocationManager()
    @Published var authorizationStatus: CLAuthorizationStatus = .notDetermined

    override init() {
        super.init()
        manager.delegate = self
    }

    func requestWhenInUse() {
        manager.requestWhenInUseAuthorization()
    }

    func locationManagerDidChangeAuthorization(_ manager: CLLocationManager) {
        authorizationStatus = manager.authorizationStatus
    }
}

struct SensitivePermissionRequestView: View {
    @StateObject private var locationManager = LocationManager()
    let permissionType: String
    let purpose: String
    let onGrant: () -> Void
    let onDeny: () -> Void

    var body: some View {
        VStack(spacing: 20) {
            Image(systemName: iconForPermission)
                .font(.system(size: 40))
                .foregroundStyle(.blue)

            Text("请求\(permissionType)权限")
                .font(.title3.bold())

            Text(purpose)
                .font(.body)
                .foregroundStyle(.secondary)
                .multilineTextAlignment(.center)
                .padding(.horizontal, 32)

            Text("您可以随时在系统设置中关闭此权限")
                .font(.caption)
                .foregroundStyle(.tertiary)

            HStack(spacing: 16) {
                Button("拒绝") {
                    onDeny()
                }
                .buttonStyle(.bordered)

                Button("允许") {
                    locationManager.requestWhenInUse()
                    onGrant()
                }
                .buttonStyle(.borderedProminent)
            }
        }
        .padding()
    }

    private var iconForPermission: String {
        switch permissionType {
        case "位置": return "location.fill"
        case "相机": return "camera.fill"
        case "麦克风": return "mic.fill"
        case "通讯录": return "person.2.fill"
        default: return "hand.raised.fill"
        }
    }
}
```

### 数据脱敏与最小化处理

在存储和展示敏感信息时，应进行脱敏处理：

```swift
import Foundation

struct DataMasking {
    static func maskPhone(_ phone: String) -> String {
        guard phone.count >= 11 else { return phone }
        let start = phone.index(phone.startIndex, offsetBy: 3)
        let end = phone.index(phone.startIndex, offsetBy: 7)
        return phone.replacingCharacters(in: start..<end, with: "****")
    }

    static func maskIDCard(_ id: String) -> String {
        guard id.count >= 15 else { return id }
        let start = id.index(id.startIndex, offsetBy: 4)
        let end = id.index(id.endIndex, offsetBy: -4)
        return id.replacingCharacters(in: start..<end, with: "**********")
    }

    static func maskBankCard(_ card: String) -> String {
        guard card.count >= 12 else { return card }
        let start = card.index(card.startIndex, offsetBy: 4)
        let end = card.index(card.endIndex, offsetBy: -4)
        return card.replacingCharacters(in: start..<end, with: " **** **** ")
    }

    static func maskEmail(_ email: String) -> String {
        let parts = email.split(separator: "@")
        guard parts.count == 2 else { return email }
        let prefix = String(parts[0])
        let maskedPrefix = prefix.count > 2
            ? String(prefix.prefix(2)) + "***"
            : "***"
        return maskedPrefix + "@" + String(parts[1])
    }
}
```

> 💡 **提示**：脱敏应同时应用于前端展示和后端日志。后端存储时也应考虑是否需要存储完整信息，如果仅需验证，可以存储哈希值而非原文。

---

## 数据跨境传输

### 什么情况涉及数据跨境

数据跨境传输是指将在中国境内收集的个人信息传输至境外。以下场景都可能构成跨境传输：

- 服务器部署在境外（如 AWS 海外区域）
- 使用境外 SaaS 服务处理用户数据（如海外 Analytics 服务）
- 境外团队远程访问境内用户数据
- App 使用 iCloud 等境外云服务同步数据
- 第三方 SDK 将数据发送至境外服务器

> ⚠️ **警告**：很多开发者忽略了第三方 SDK 的数据流向。如果集成的海外 SDK（如 Firebase、Google Analytics）将数据传至境外，你同样需要遵守跨境传输规定。

### 跨境传输的法定条件

PIPL 第三十八条规定，向境外提供个人信息必须满足以下条件之一：

| 条件 | 说明 | 适用场景 |
|------|------|----------|
| **安全评估** | 通过国家网信部门组织的安全评估 | 关键信息基础设施运营者、处理个人信息达到规定数量 |
| **标准合同** | 与境外接收方订立标准合同 | 中小型 App 开发者最常用的方式 |
| **认证** | 经专业机构认证 | 跨国企业内部数据传输 |
| **法律行政法规规定的其他条件** | 如后续出台的新规 | 暂无 |

### Apple 服务器与数据跨境

使用 Apple 平台服务时的数据跨境情况：

| Apple 服务 | 数据存储位置 | 是否涉及跨境 | 注意事项 |
|------------|-------------|-------------|----------|
| App Store 分发 | 主要在美国 | 是 | App 元数据（名称、描述等）存储在境外 |
| CloudKit | 可选择区域 | 视配置而定 | 创建容器时可选择区域，建议选择中国区 |
| iCloud | 全球分布 | 是 | 用户数据可能存储在境外 |
| Apple Push Notification | 全球分布 | 是 | 推送令牌和消息经 Apple 服务器 |
| Game Center | 全球分布 | 是 | 游戏数据经 Apple 服务器 |
| In-App Purchase | 全球分布 | 是 | 交易记录经 Apple 服务器 |

### 使用 iCloud 的合规考量

如果你的 App 使用 CloudKit 或 iCloud 同步用户数据：

1. **创建 CloudKit 容器时选择中国区域**，尽量将数据存储在境内
2. 在隐私政策中明确告知用户数据可能通过 Apple 的全球基础设施传输
3. 对于特别敏感的数据，考虑使用自建服务器而非 iCloud
4. 评估是否需要与 Apple 签署数据处理协议

### 跨境传输合规方案

| 方案 | 适用场景 | 实施难度 | 成本 |
|------|----------|----------|------|
| 数据本地化存储 | 所有 App | 低 | 服务器成本 |
| 签署标准合同（SCC） | 中小型 App | 中 | 法律咨询费 |
| 通过安全评估 | 大型平台 | 高 | 评估费用高 |
| 使用 Apple 中国区服务 | 依赖 Apple 生态的 App | 低 | Apple 开发者费用 |
| 敏感数据不出境 + 非敏感数据跨境 | 混合场景 | 中 | 架构改造成本 |

> 💡 **提示**：对于个人开发者和小团队，最务实的方案是**将用户数据存储在国内云服务器**（如阿里云、腾讯云），避免跨境传输的复杂合规要求。如果必须跨境，签署标准合同是目前门槛最低的合规路径。

---

## 用户权利实现

### 知情权与决定权

用户有权了解其个人信息被如何处理，并有权决定是否同意处理。App 应当：

- 在设置页面提供"隐私管理"入口
- 清晰展示当前已授权的信息类型和用途
- 允许用户随时开启或关闭各项授权

### 查阅权与复制权

用户有权查阅其个人信息，并有权请求复制。App 应当：

- 提供"我的数据"或"个人信息"页面
- 展示所有已收集的用户信息
- 支持数据导出功能（JSON 或 CSV 格式）

### 更正权与补充权

用户发现其个人信息不准确或不完整时，有权请求更正或补充。App 应当：

- 允许用户编辑个人资料中的各项信息
- 对于系统自动生成的信息（如行为标签），提供申诉更正渠道

### 删除权

在以下情形下，用户有权请求删除其个人信息：

- 处理目的已实现、无法实现或不再必要
- 个人信息处理者停止提供产品或服务
- 用户撤回同意
- 处理信息违反法律或约定

### 账号注销功能实现

PIPL 要求 App 提供账号注销功能，且注销流程不应设置不合理条件：

```swift
import SwiftUI

enum DeletionReason: String, CaseIterable, Identifiable {
    case noLongerNeeded = "不再需要此服务"
    case privacyConcern = "担心隐私安全"
    case foundAlternative = "找到了替代产品"
    case accountCompromised = "账号存在安全风险"
    case other = "其他原因"

    var id: String { rawValue }
}

struct AccountDeletionView: View {
    @State private var selectedReason: DeletionReason = .noLongerNeeded
    @State private var customReason = ""
    @State private var confirmedDelete = false
    @State private var showFinalAlert = false
    @State private var isDeleting = false

    var body: some View {
        NavigationStack {
            ScrollView {
                VStack(alignment: .leading, spacing: 20) {
                    Text("注销账号")
                        .font(.title2.bold())

                    Text("注销账号后，您的个人信息将在 30 个工作日内被删除或匿名化处理。此操作不可撤销。")
                        .font(.body)
                        .foregroundStyle(.secondary)

                    GroupBox {
                        VStack(alignment: .leading, spacing: 12) {
                            Label("所有个人资料将被永久删除", systemImage: "trash")
                            Label("订单记录将匿名化处理", systemImage: "doc.text.magnifyingglass")
                            Label("关联的第三方授权将被解除", systemImage: "link.badge.minus")
                            Label("云同步数据将被清除", systemImage: "cloud.slash")
                        }
                        .font(.subheadline)
                    } label: {
                        Text("注销后将发生以下变化")
                            .font(.headline)
                    }

                    Text("请选择注销原因")
                        .font(.headline)

                    Picker("注销原因", selection: $selectedReason) {
                        ForEach(DeletionReason.allCases) { reason in
                            Text(reason.rawValue).tag(reason)
                        }
                    }
                    .pickerStyle(.segmented)

                    if selectedReason == .other {
                        TextField("请说明原因", text: $customReason)
                            .textFieldStyle(.roundedBorder)
                    }

                    Toggle("我已了解注销后果，确认注销", isOn: $confirmedDelete)
                        .font(.subheadline)

                    Button {
                        showFinalAlert = true
                    } label: {
                        Text("确认注销账号")
                            .font(.headline)
                            .frame(maxWidth: .infinity)
                            .padding()
                            .background(confirmedDelete ? Color.red : Color.gray)
                            .foregroundStyle(.white)
                            .clipShape(RoundedRectangle(cornerRadius: 12))
                    }
                    .disabled(!confirmedDelete)
                }
                .padding()
            }
            .alert("最终确认", isPresented: $showFinalAlert) {
                Button("取消", role: .cancel) {}
                Button("确认注销", role: .destructive) {
                    performDeletion()
                }
            } message: {
                Text("账号注销后不可恢复，确定要继续吗？")
            }
        }
    }

    private func performDeletion() {
        isDeleting = true
    }
}
```

### SwiftUI 实现数据管理界面

```swift
import SwiftUI

struct DataManagementView: View {
    @State private var userData: [String: String] = [:]
    @State private var showExportSheet = false
    @State private var exportedData = ""

    var body: some View {
        List {
            Section("我的个人信息") {
                ForEach(Array(userData.sorted(by: { $0.key < $1.key })), id: \.key) { key, value in
                    HStack {
                        Text(key)
                            .foregroundStyle(.secondary)
                        Spacer()
                        Text(DataMasking.maskIfNeeded(key: key, value: value))
                    }
                }
            }

            Section("数据操作") {
                Button {
                    exportData()
                } label: {
                    Label("导出我的数据", systemImage: "square.and.arrow.up")
                }

                NavigationLink {
                    AccountDeletionView()
                } label: {
                    Label("注销账号", systemImage: "person.badge.minus")
                        .foregroundStyle(.red)
                }
            }

            Section("授权管理") {
                NavigationLink {
                    PermissionManagementView()
                } label: {
                    Label("权限与授权", systemImage: "hand.raised")
                }

                NavigationLink {
                    ConsentHistoryView()
                } label: {
                    Label("同意记录", systemImage: "doc.text")
                }
            }
        }
        .navigationTitle("数据管理")
        .onAppear {
            loadUserData()
        }
        .shareSheet(isPresented: $showExportSheet, items: [exportedData])
    }

    private func loadUserData() {
        userData = [
            "手机号": "13800138000",
            "邮箱": "user@example.com",
            "注册时间": "2024-01-15",
            "设备型号": "iPhone 15 Pro"
        ]
    }

    private func exportData() {
        if let jsonData = try? JSONSerialization.data(
            withJSONObject: userData,
            options: [.prettyPrinted, .sortedKeys]
        ) {
            exportedData = String(data: jsonData, encoding: .utf8) ?? ""
            showExportSheet = true
        }
    }
}

extension DataMasking {
    static func maskIfNeeded(key: String, value: String) -> String {
        if key.contains("手机") { return maskPhone(value) }
        if key.contains("邮箱") { return maskEmail(value) }
        return value
    }
}

struct ShareSheet: UIViewControllerRepresentable {
    @Binding var isPresented: Bool
    let items: [Any]

    func makeUIViewController(context: Context) -> UIViewController {
        UIViewController()
    }

    func updateUIViewController(_ uiViewController: UIViewController, context: Context) {
        if isPresented {
            let controller = UIActivityViewController(activityItems: items, applicationActivities: nil)
            controller.completionWithItemsHandler = { _, _, _, _ in
                isPresented = false
            }
            uiViewController.present(controller, animated: true)
        }
    }
}
```

### 响应用户请求的时限要求

| 请求类型 | PIPL 要求时限 | 建议实现 |
|----------|---------------|----------|
| 查阅/复制 | 未明确规定 | 7 个工作日内 |
| 更正/补充 | 未明确规定 | 7 个工作日内 |
| 删除 | 未明确规定 | 15 个工作日内 |
| 账号注销 | 未明确规定 | 15 个工作日内 |
| 撤回同意 | 及时 | 即时生效或 24 小时内 |
| 跨境传输知情 | 未明确规定 | 在隐私政策中持续公示 |

> 💡 **提示**：虽然 PIPL 对部分请求没有明确规定时限，但工信部在《App 违法违规收集使用个人信息行为认定方法》中要求 App 提供便捷的注销功能，注销承诺时限不应超过 15 个工作日。建议在隐私政策中明确各项请求的处理时限。

---

## 合规检查清单

### 上架前合规自查表

| 检查项 | 合规要求 | 是否达标 | 备注 |
|--------|----------|----------|------|
| 隐私政策 | 有可访问的隐私政策 URL | ☐ | 必须在 App Store Connect 填写 |
| 隐私弹窗 | 首次启动前展示隐私弹窗 | ☐ | 不得先收集再弹窗 |
| 同意机制 | 用户主动同意，非默认勾选 | ☐ | 同意按钮不得预设选中 |
| 最小必要 | 仅收集功能必需的信息 | ☐ | 逐项检查每个权限的必要性 |
| 权限说明 | Info.plist 权限描述清晰 | ☐ | 说明具体用途而非笼统描述 |
| 单独同意 | 敏感信息获取单独同意 | ☐ | 人脸、位置等需独立弹窗 |
| 目的限制 | 不超范围使用个人信息 | ☐ | 检查是否存在未告知的用途 |
| 数据安全 | 传输加密 + 存储加密 | ☐ | 全站 HTTPS，敏感数据加密存储 |
| 用户权利 | 提供查阅、更正、删除功能 | ☐ | 设置中需有数据管理入口 |
| 账号注销 | 提供便捷的注销功能 | ☐ | 不得设置不合理条件 |
| 撤回同意 | 提供同意撤回机制 | ☐ | 撤回操作不比同意更复杂 |
| 数据跨境 | 跨境传输满足法定条件 | ☐ | 评估是否涉及跨境 |
| 第三方 SDK | 列明第三方 SDK 信息收集情况 | ☐ | 需在隐私政策中披露 |
| 未成年人保护 | 14 岁以下需监护人同意 | ☐ | 如面向儿童需额外合规 |
| 隐私政策更新 | 更新时通知用户并重新获取同意 | ☐ | 重大变更需弹窗提示 |
| App Privacy 标签 | App Store Connect 准确填写数据类型 | ☐ | 与实际收集行为一致 |
| 数据保留期限 | 明确数据存储期限 | ☐ | 超期数据应删除或匿名化 |
| 隐私清单 | 配置 PrivacyInfo.xcprivacy | ☐ | Apple 强制要求 |

### 常见违规案例与处罚

| 违规类型 | 典型案例 | 处罚结果 |
|----------|----------|----------|
| 超范围收集 | 天气 App 强制要求通讯录权限 | 工信部通报，限期整改 |
| 强制索权 | 不同意隐私政策无法使用任何功能 | App Store 审核被拒 |
| 未提供注销 | 账号注销入口隐藏或无法完成 | 工信部通报，下架整改 |
| 欺骗误导 | 隐私政策与实际收集行为不一致 | 行政处罚，罚款 |
| 未明示第三方共享 | 嵌入 SDK 收集信息未在隐私政策中说明 | 工信部通报 |
| 未成年人保护缺失 | 儿童 App 未取得监护人同意 | 严重处罚，下架 |

### 工信部通报的典型问题

工信部每月发布 App 侵害用户权益整治通报，以下是最常见的问题：

1. **超范围收集个人信息** — 收集与功能无关的权限或数据
2. **App 强制、频繁、过度索取权限** — 拒绝授权后反复弹窗或无法使用
3. **欺骗误导强迫用户** — 隐私政策含糊不清或默认勾选同意
4. **未提供账号注销功能** — 无法注销或注销流程极其复杂
5. **未明示收集使用规则** — 隐私政策缺失或不完整
6. **未告知第三方信息共享** — 集成 SDK 但未在隐私政策中说明

### 持续合规的维护建议

| 维护项 | 频率 | 具体操作 |
|--------|------|----------|
| 隐私政策审查 | 每季度 | 检查政策是否与实际行为一致 |
| 权限使用审计 | 每次版本更新 | 确认新增权限的必要性 |
| 第三方 SDK 审查 | 每季度 | 检查 SDK 更新后的数据收集行为 |
| 合规法规跟踪 | 持续 | 关注工信部、网信办新规 |
| 用户请求响应 | 实时 | 及时处理用户的数据权利请求 |
| 安全漏洞扫描 | 每月 | 检查数据存储和传输安全 |
| App Privacy 标签更新 | 每次版本更新 | 确保标签与实际收集行为一致 |
| 同意记录审计 | 每半年 | 检查同意记录的完整性和准确性 |

---

## 小结

本章系统介绍了《个人信息保护法》的核心要求及 App 开发中的合规实践。以下是核心知识点总结：

| 知识领域 | 核心要点 | 关键行动 |
|----------|----------|----------|
| PIPL 概述 | 五大原则：合法、正当、必要、诚信、公开透明 | 理解法律框架，明确适用范围 |
| 核心要求 | 最小必要、知情同意、目的限制、安全保障 | 逐项检查 App 的数据收集行为 |
| 用户同意 | 首次弹窗、单独同意、撤回机制 | 实现合规的同意流程并留存记录 |
| 敏感信息 | 需单独同意 + 影响评估 + 脱敏处理 | 识别敏感信息场景，加强保护措施 |
| 数据跨境 | 安全评估/标准合同/认证三选一 | 优先数据本地化，必要时签署标准合同 |
| 用户权利 | 查阅、复制、更正、删除、注销 | 提供完整的数据管理界面和注销功能 |
| 合规检查 | 18 项上架前自查 + 持续维护 | 建立合规自查流程，定期审查更新 |

> ⚠️ **警告**：合规不是一次性的工作，而是持续的过程。法律法规在不断完善，App 功能在不断迭代，第三方 SDK 在不断更新——任何变化都可能引入新的合规风险。建议将合规审查纳入每次版本发布的标准流程。

← [隐私政策与用户协议](./隐私政策与用户协议.md) | [App 图标与启动页](./App图标与启动页.md) →