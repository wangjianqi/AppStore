# 网络安全与 ATS 配置

## 本章目标

- 理解 iOS 网络安全的核心威胁与 HTTPS 的重要性
- 深入掌握 App Transport Security (ATS) 的配置与豁免机制
- 学会证书固定（Certificate Pinning）的多种实现方式
- 掌握 TLS 最佳实践与自签名证书处理
- 了解网络请求安全：签名、防重放、加密、Key 安全存储
- 熟练使用网络安全诊断与审计工具
- 了解国内等保合规对网络安全的硬性要求

---

## 1. iOS 网络安全概述

### 1.1 为什么 HTTPS 如此重要

想象你在一间开放式办公室里打电话，所有同事都能听到你说的每一句话——这就是 HTTP 传输的样子。而 HTTPS 就像在一个隔音会议室里通话，外人只能看到你们在交流，却听不到具体内容。

HTTP 的三大致命问题：

| 问题 | 说明 | 类比 |
|------|------|------|
| **窃听** | 数据明文传输，任何人可截获 | 明信片：邮递员、邻居都能看到内容 |
| **篡改** | 中间人可修改传输中的数据 | 快递被掉包：收到的不是发件人寄的东西 |
| **冒充** | 无法验证服务器真实身份 | 假客服：你以为在和银行通话，其实是骗子 |

HTTPS 通过 **TLS（Transport Layer Security）** 协议解决了这三个问题：

- **加密**：对称加密保护数据机密性
- **完整性**：消息认证码（MAC）防止数据被篡改
- **身份验证**：数字证书验证服务器身份

### 1.2 中间人攻击（MITM）原理

中间人攻击是网络安全的头号威胁，攻击者夹在客户端与服务器之间，双方都以为在直接通信：

```
┌──────────┐      ┌──────────┐      ┌──────────┐
│  客户端    │ ◄──► │  攻击者    │ ◄──► │  服务器    │
│  (App)    │      │  (代理)    │      │  (API)   │
└──────────┘      └──────────┘      └──────────┘
   以为在和服务器通信    解密→查看→重加密     以为在和客户端通信
```

攻击流程：
1. 攻击者通过公共 Wi-Fi 或 DNS 劫持，将客户端请求导向自己的代理
2. 攻击者向客户端出示伪造证书，建立"安全"连接
3. 攻击者再与真实服务器建立连接，充当中间人
4. 双方的所有通信都经过攻击者，可被窃听或篡改

> ⚠️ **真实案例**：2015 年中国多家运营商和公共 Wi-Fi 被发现进行 HTTP 注入，在网页中插入广告和追踪脚本。这本质上就是一种中间人攻击。

### 1.3 Apple 的安全策略演进

Apple 一直在推动 iOS 生态走向更安全的网络通信：

| 时间节点 | 事件 | 影响 |
|---------|------|------|
| **2015 年** | iOS 9 引入 ATS | 默认强制 HTTPS，HTTP 请求直接失败 |
| **2016 年** | Apple 宣布 2017 年起 ATS 不允许豁免 | 后因开发者反馈推迟执行 |
| **2017 年** | iOS 11 新增 `NSAllowsArbitraryLoadsInWebContent` | WebView 内容可单独豁免 |
| **2020 年** | iOS 14 加强隐私报告 | 网络请求透明度提升 |
| **2024 年** | iOS 18 强化隐私清单要求 | 需声明网络使用原因 |

> 💡 **趋势**：Apple 的方向非常明确——全面 HTTPS 化。新项目务必从第一天就使用 HTTPS，不要心存侥幸。

---

## 2. App Transport Security (ATS) 详解

### 2.1 ATS 默认规则

ATS 是 iOS 9 引入的安全功能，**默认要求所有网络连接满足以下条件**：

| 要求 | 最低标准 | 说明 |
|------|---------|------|
| **协议** | HTTPS | HTTP 连接直接被拒绝 |
| **TLS 版本** | TLS 1.2+ | TLS 1.0/1.1 不被接受 |
| **证书签名** | SHA-256+ | SHA-1 签名的证书被拒绝 |
| **密钥交换** | ECDHE / RSA 2048+ | 低于 2048 位的 RSA 密钥被拒绝 |
| **加密套件** | AES-128-GCM / AES-256-GCM | CBC 模式套件被拒绝 |

当 ATS 拦截请求时，控制台会输出类似日志：

```
App Transport Security has blocked a cleartext HTTP (http://) resource load
since it is insecure. Temporary exceptions can be configured via your app's
Info.plist file.
```

### 2.2 NSAppTransportSecurity 配置

ATS 的所有配置都在 Info.plist 的 `NSAppTransportSecurity` 字典中：

```
NSAppTransportSecurity（字典）
├── NSAllowsArbitraryLoads（布尔）—— 全局允许 HTTP
├── NSAllowsArbitraryLoadsInWebContent（布尔）—— 仅 WebView 允许 HTTP
├── NSAllowsLocalNetworking（布尔）—— 允许本地网络 HTTP
└── NSExceptionDomains（字典）—— 按域名配置豁免
    ├── example.com（字典）
    │   ├── NSExceptionAllowsInsecureHTTPLoads
    │   ├── NSExceptionMinimumTLSVersion
    │   ├── NSExceptionRequiresForwardSecrecy
    │   ├── NSIncludesSubdomains
    │   └── NSRequiresCertificateTransparency
    └── another.com（字典）
        └── ...
```

### 2.3 NSAllowsArbitraryLoads：全局豁免

最粗暴的方式——允许所有 HTTP 连接：

```xml
<key>NSAppTransportSecurity</key>
<dict>
    <key>NSAllowsArbitraryLoads</key>
    <true/>
</dict>
```

> ⚠️ **强烈不建议**：这等于完全关闭 ATS 保护。App Store 审核时必须提供充分理由，否则会被拒。

### 2.4 NSExceptionDomains：按域名豁免

更精细的方式——只对特定域名放宽限制：

```xml
<key>NSAppTransportSecurity</key>
<dict>
    <key>NSExceptionDomains</key>
    <dict>
        <key>legacy-api.example.com</key>
        <dict>
            <key>NSExceptionAllowsInsecureHTTPLoads</key>
            <true/>
            <key>NSIncludesSubdomains</key>
            <true/>
        </dict>
        <key>tls10-server.example.com</key>
        <dict>
            <key>NSExceptionMinimumTLSVersion</key>
            <string>TLSv1.0</string>
            <key>NSExceptionRequiresForwardSecrecy</key>
            <false/>
        </dict>
    </dict>
</dict>
```

各字段含义：

| 字段 | 类型 | 说明 |
|------|------|------|
| `NSExceptionAllowsInsecureHTTPLoads` | Boolean | 允许该域名使用 HTTP |
| `NSExceptionMinimumTLSVersion` | String | 最低 TLS 版本（如 `TLSv1.0`） |
| `NSExceptionRequiresForwardSecrecy` | Boolean | 是否要求前向保密（PFS） |
| `NSIncludesSubdomains` | Boolean | 规则是否包含子域名 |
| `NSRequiresCertificateTransparency` | Boolean | 是否要求证书透明度 |

### 2.5 Info.plist 完整配置示例

一个实际项目中的推荐配置：

```xml
<key>NSAppTransportSecurity</key>
<dict>
    <key>NSAllowsArbitraryLoads</key>
    <false/>
    <key>NSAllowsLocalNetworking</key>
    <true/>
    <key>NSExceptionDomains</key>
    <dict>
        <key>localhost</key>
        <dict>
            <key>NSExceptionAllowsInsecureHTTPLoads</key>
            <true/>
        </dict>
        <key>192.168.1.100</key>
        <dict>
            <key>NSExceptionAllowsInsecureHTTPLoads</key>
            <true/>
        </dict>
        <key>cdn.legacy-partner.cn</key>
        <dict>
            <key>NSExceptionAllowsInsecureHTTPLoads</key>
            <true/>
            <key>NSIncludesSubdomains</key>
            <true/>
        </dict>
    </dict>
</dict>
```

> 💡 **最佳实践**：`NSAllowsLocalNetworking` 设为 `true` 是安全的，开发阶段经常需要访问本地服务器。但生产环境的远程接口务必使用 HTTPS。

---

## 3. ATS 豁免与审核

### 3.1 何时可以申请豁免

Apple 允许在以下合理场景中豁免 ATS：

| 豁免场景 | 说明 | 审核通过率 |
|---------|------|:---------:|
| **第三方 CDN** | 无法控制第三方服务的 HTTPS 支持 | ⭐⭐⭐ |
| **设备局域网通信** | 与局域网内设备（打印机、IoT）通信 | ⭐⭐⭐⭐ |
| **媒体流** | 某些流媒体协议不支持 HTTPS | ⭐⭐⭐ |
| **已有服务端限制** | 遗留系统短期无法升级 | ⭐⭐ |
| **WebView 内容** | 加载第三方网页内容 | ⭐⭐⭐⭐ |

### 3.2 审核说明信撰写

当使用了 ATS 豁免时，必须在 App Store Connect 的审核备注中说明原因。模板如下：

```
【ATS 豁免说明】

1. 豁免域名：cdn.legacy-partner.cn
2. 豁免原因：该 CDN 由合作方提供，目前仅支持 HTTP 协议，
   我方已要求合作方在 2026 年 Q3 前完成 HTTPS 升级。
3. 替代方案评估：已评估迁移至支持 HTTPS 的 CDN 服务商，
   但涉及合同约束和数据迁移，短期内无法完成。
4. 数据保护措施：该域名仅用于下载公开的图片资源，
   不涉及任何用户隐私数据传输。
5. 过渡计划：预计 2026 年 9 月前完成 HTTPS 迁移，
   届时将移除此 ATS 豁免配置。
```

> ⚠️ **关键要点**：说明信必须包含"为什么不能 HTTPS"和"何时能解决"两个核心信息。没有明确过渡计划的豁免很可能被拒。

### 3.3 国内 HTTP 接口处理方案

国内开发中经常遇到 HTTP 接口问题，推荐处理策略：

| 方案 | 适用场景 | 优缺点 |
|------|---------|--------|
| **推动后端升级 HTTPS** | 自有后端服务 | ✅ 根本解决 ❌ 需要后端配合 |
| **使用 NSExceptionDomains** | 第三方 HTTP 接口 | ✅ 精准豁免 ❌ 需审核说明 |
| **自建 HTTPS 代理** | 无法修改的第三方接口 | ✅ 不改 ATS 配置 ❌ 增加运维成本 |
| **WebView 加载** | 网页内容展示 | ✅ 用 `NSAllowsArbitraryLoadsInWebContent` ❌ 仅限 WebView |

自建 HTTPS 代理的架构：

```
┌──────────┐    HTTPS    ┌──────────┐    HTTP    ┌──────────┐
│  iOS App  │ ─────────► │  代理服务器 │ ────────► │  第三方 API │
│           │ ◄───────── │  (Nginx)  │ ◄──────── │  (仅HTTP)  │
└──────────┘    HTTPS    └──────────┘    HTTP    └──────────┘
```

---

## 4. 证书固定 / Certificate Pinning

### 4.1 什么是证书固定

证书固定就像"认脸不认证件"——即使别人拿着合法的身份证（CA 签发的证书），只要不是你认识的那张脸（App 内置的证书/公钥），就不让通过。

传统 HTTPS 验证信任任何 CA 签发的证书，这意味着如果某个 CA 被攻破或恶意签发证书，中间人攻击就能成功。证书固定通过在 App 内置期望的证书或公钥，将信任链缩短到最小范围。

| 对比项 | 普通 HTTPS | 证书固定 |
|-------|-----------|---------|
| 信任范围 | 数百个 CA | 仅 App 内置的证书/公钥 |
| CA 被攻破的影响 | 全部受影响 | 不受影响 |
| 证书轮换 | 无影响 | 需要更新 App |
| 实现复杂度 | 低 | 中高 |

### 4.2 公钥固定 vs 证书固定

| 对比项 | 证书固定 | 公钥固定 |
|-------|---------|---------|
| **固定内容** | 完整的 X.509 证书 | 证书中的公钥（SPKI SHA-256 哈希） |
| **证书续期** | ❌ 必须更新 App | ✅ 只要公钥不变即可 |
| **安全性** | 高 | 高 |
| **灵活性** | 低 | 较高 |
| **推荐度** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |

> 💡 **推荐公钥固定**：大多数 CA 续期证书时会保持同一公钥对，因此公钥固定比证书固定更灵活，减少了强制更新 App 的频率。

### 4.3 URLSessionDelegate 实现

使用原生 URLSession 实现证书固定：

```swift
import Foundation

class PinningDelegate: NSObject, URLSessionDelegate {
    private let pinnedPublicKeys: [SecKey] = []

    init(pinnedCertificateNames: [String]) {
        super.init()
        for name in pinnedCertificateNames {
            guard let certURL = Bundle.main.url(forResource: name, withExtension: "cer"),
                  let certData = try? Data(contentsOf: certURL),
                  let cert = SecCertificateCreateWithData(nil, certData as CFData),
                  let publicKey = SecCertificateCopyKey(cert) else {
                continue
            }
            pinnedPublicKeys.append(publicKey)
        }
    }

    func urlSession(
        _ session: URLSession,
        didReceive challenge: URLAuthenticationChallenge,
        completionHandler: @escaping (URLSession.AuthChallengeDisposition, URLCredential?) -> Void
    ) {
        guard challenge.protectionSpace.authenticationMethod == NSURLAuthenticationMethodServerTrust,
              let serverTrust = challenge.protectionSpace.serverTrust else {
            completionHandler(.cancelAuthenticationChallenge, nil)
            return
        }

        let host = challenge.protectionSpace.host
        SecTrustSetPolicies(serverTrust, SecPolicyCreateSSL(true, host as CFString))

        var error: CFError?
        let isValid = SecTrustEvaluateWithError(serverTrust, &error)

        guard isValid else {
            completionHandler(.cancelAuthenticationChallenge, nil)
            return
        }

        let certificateCount = SecTrustGetCertificateCount(serverTrust)
        var publicKeyFound = false

        for i in 0..<certificateCount {
            guard let certificate = SecTrustGetCertificateAtIndex(serverTrust, i),
                  let publicKey = SecCertificateCopyKey(certificate) else {
                continue
            }

            for pinnedKey in pinnedPublicKeys {
                if SecKeyIsEqualTo(publicKey, pinnedKey) {
                    publicKeyFound = true
                    break
                }
            }
            if publicKeyFound { break }
        }

        if publicKeyFound {
            let credential = URLCredential(trust: serverTrust)
            completionHandler(.useCredential, credential)
        } else {
            completionHandler(.cancelAuthenticationChallenge, nil)
        }
    }
}
```

使用方式：

```swift
let delegate = PinningDelegate(pinnedCertificateNames: ["myserver"])
let session = URLSession(configuration: .default, delegate: delegate, delegateQueue: nil)

session.dataTask(with: URL(string: "https://api.example.com/user")!) { data, response, error in
    if let error = error {
        print("请求失败: \(error.localizedDescription)")
        return
    }
    print("请求成功，数据已安全接收")
}.resume()
```

### 4.4 Alamofire Pinning

Alamofire 内置了证书固定支持，使用更加简洁：

```swift
import Alamofire

let serverTrustManager = ServerTrustManager(
    evaluators: [
        "api.example.com": PublicKeysTrustEvaluator(
            performDefaultValidation: true,
            validateHost: true
        ),
        "cdn.example.com": CertificatesTrustEvaluator(
            performDefaultValidation: true,
            validateHost: true
        )
    ]
)

let session = Session(
    configuration: .default,
    serverTrustManager: serverTrustManager
)

session.request("https://api.example.com/user").responseDecodable(of: User.self) { response in
    switch response.result {
    case .success(let user):
        print("用户: \(user.name)")
    case .failure(let error):
        print("请求失败: \(error)")
    }
}
```

Alamofire 支持的评估器类型：

| 评估器 | 说明 | 适用场景 |
|-------|------|---------|
| `PublicKeysTrustEvaluator` | 公钥固定 | ✅ 推荐，灵活性高 |
| `CertificatesTrustEvaluator` | 证书固定 | 需要严格固定完整证书 |
| `CompositeTrustEvaluator` | 组合评估 | 同时要求公钥和证书 |
| `DisabledTrustEvaluator` | 禁用验证 | ⚠️ 仅用于调试 |
| `DefaultTrustEvaluator` | 系统默认验证 | 不需要额外固定 |

> ⚠️ **重要**：将 `.cer` 证书文件放入项目 Bundle 时，确保勾选了 Target Membership，否则运行时找不到证书文件。

---

## 5. SSL/TLS 最佳实践

### 5.1 TLS 版本对比

| 版本 | 发布年份 | 安全性 | iOS 支持 | 状态 |
|------|---------|--------|---------|------|
| **TLS 1.0** | 1999 | ❌ 不安全 | iOS 5+ | 已废弃 |
| **TLS 1.1** | 2006 | ❌ 不安全 | iOS 5+ | 已废弃 |
| **TLS 1.2** | 2008 | ✅ 安全 | iOS 5+ | ATS 最低要求 |
| **TLS 1.3** | 2018 | ✅✅ 最安全 | iOS 13+ | 推荐使用 |

TLS 1.3 的关键改进：

- **1-RTT 握手**：连接建立速度提升 50%
- **0-RTT 恢复**：重连几乎零延迟
- **移除弱算法**：删除了 RSA 密钥交换、CBC 模式等不安全特性
- **前向保密默认开启**：每次会话使用不同的临时密钥

> 💡 **生活类比**：TLS 1.2 像是打电话先确认对方身份再聊正事（2-RTT），TLS 1.3 像是老朋友见面直接聊，身份确认和正事同时进行（1-RTT）。

### 5.2 证书链验证

一个完整的证书链由三层构成：

```
┌──────────────────────┐
│   根证书 (Root CA)     │  ← 操作系统内置信任
│   DigiCert Global Root │
└──────────┬───────────┘
           │ 签发
┌──────────▼───────────┐
│   中间证书 (Intermediate)│  ← 服务器通常提供
│   DigiCert SHA2        │
└──────────┬───────────┘
           │ 签发
┌──────────▼───────────┐
│   服务器证书 (Leaf)     │  ← 绑定域名
│   api.example.com      │
└──────────────────────┘
```

验证逻辑：从叶子证书开始，逐级向上验证签名，直到到达系统信任的根证书。

### 5.3 自签名证书处理

开发环境中经常使用自签名证书，需要在 URLSessionDelegate 中手动信任：

```swift
func urlSession(
    _ session: URLSession,
    didReceive challenge: URLAuthenticationChallenge,
    completionHandler: @escaping (URLSession.AuthChallengeDisposition, URLCredential?) -> Void
) {
    guard challenge.protectionSpace.host == "dev-api.example.com",
          let serverTrust = challenge.protectionSpace.serverTrust else {
        completionHandler(.performDefaultHandling, nil)
        return
    }

    guard let localCertURL = Bundle.main.url(forResource: "dev-cert", withExtension: "cer"),
          let localCertData = try? Data(contentsOf: localCertURL),
          let localCert = SecCertificateCreateWithData(nil, localCertData as CFData) else {
        completionHandler(.cancelAuthenticationChallenge, nil)
        return
    }

    SecTrustSetAnchorCertificates(serverTrust, [localCert] as CFArray)
    SecTrustSetAnchorCertificatesOnly(serverTrust, true)

    var error: CFError?
    if SecTrustEvaluateWithError(serverTrust, &error) {
        completionHandler(.useCredential, URLCredential(trust: serverTrust))
    } else {
        completionHandler(.cancelAuthenticationChallenge, nil)
    }
}
```

> ⚠️ **安全警告**：自签名证书的信任逻辑**绝不能**出现在生产环境构建中。建议使用编译条件隔离：

```swift
#if DEBUG
    // 仅在 Debug 模式下信任自签名证书
    SecTrustSetAnchorCertificates(serverTrust, [localCert] as CFArray)
#else
    completionHandler(.cancelAuthenticationChallenge, nil)
#endif
```

---

## 6. 网络请求安全

### 6.1 请求签名

请求签名确保请求未被篡改，就像快递包裹上的封条——一旦签名不匹配，说明内容被改动过。

```swift
import CryptoKit

struct RequestSigner {
    private let secretKey: String

    init(secretKey: String) {
        self.secretKey = secretKey
    }

    func sign(method: String, path: String, params: [String: String], timestamp: String) -> String {
        let sortedParams = params.sorted { $0.key < $1.key }
        let paramString = sortedParams.map { "\($0.key)=\($0.value)" }.joined(separator: "&")
        let stringToSign = "\(method)\n\(path)\n\(paramString)\n\(timestamp)"

        let key = SymmetricKey(data: Data(secretKey.utf8))
        let signature = HMAC<SHA256>.authenticationCode(
            for: Data(stringToSign.utf8),
            using: key
        )
        return Data(signature).map { String(format: "%02x", $0) }.joined()
    }
}
```

### 6.2 时间戳防重放

重放攻击是指攻击者截获合法请求后重新发送。时间戳 + 随机数（nonce）可以有效防御：

```swift
struct SecureRequest {
    let timestamp: String
    let nonce: String
    let signature: String

    static func create(
        method: String,
        path: String,
        params: [String: String],
        signer: RequestSigner
    ) -> SecureRequest {
        let timestamp = String(Int(Date().timeIntervalSince1970))
        let nonce = UUID().uuidString.replacingOccurrences(of: "-", with: "").prefix(16).lowercased()

        var signedParams = params
        signedParams["timestamp"] = timestamp
        signedParams["nonce"] = nonce

        let signature = signer.sign(
            method: method,
            path: path,
            params: signedParams,
            timestamp: timestamp
        )

        return SecureRequest(
            timestamp: timestamp,
            nonce: nonce,
            signature: signature
        )
    }
}
```

> 💡 **服务端配合**：服务端需要校验时间戳与当前时间的差值（通常允许 ±5 分钟），并记录已使用的 nonce 防止重复请求。

### 6.3 敏感参数加密

对于密码、身份证号等敏感字段，即使 HTTPS 传输也建议在应用层额外加密：

```swift
import CryptoKit

struct SensitiveEncryptor {
    private let publicKey: SecKey

    init(publicKey: SecKey) {
        self.publicKey = publicKey
    }

    func encrypt(_ plaintext: String) -> String? {
        guard let plaintextData = plaintext.data(using: .utf8) else { return nil }

        var error: Unmanaged<CFError>?
        guard let encryptedData = SecKeyCreateEncryptedData(
            publicKey,
            .rsaEncryptionOAEPSHA256,
            plaintextData as CFData,
            &error
        ) else {
            return nil
        }

        return (encryptedData as Data).base64EncodedString()
    }
}
```

### 6.4 API Key 安全存储

API Key 绝不能硬编码在代码中，应使用 Keychain 安全存储：

```swift
import Security

enum APIKeyManager {
    private static let service = "com.example.app.apikeys"

    static func save(key: String, identifier: String) -> Bool {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: identifier,
            kSecValueData as String: Data(key.utf8),
            kSecAttrAccessible as String: kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly
        ]
        SecItemDelete(query as CFDictionary)
        return SecItemAdd(query as CFDictionary, nil) == errSecSuccess
    }

    static func load(identifier: String) -> String? {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: identifier,
            kSecReturnData as String: true,
            kSecMatchLimit as String: kSecMatchLimitOne
        ]
        var result: AnyObject?
        guard SecItemCopyMatching(query as CFDictionary, &result) == errSecSuccess,
              let data = result as? Data,
              let key = String(data: data, encoding: .utf8) else {
            return nil
        }
        return key
    }
}
```

| 存储方式 | 安全性 | 可逆向 | 适用场景 |
|---------|:-----:|:-----:|---------|
| 硬编码在代码中 | ❌ 极低 | ✅ 轻松提取 | ❌ 绝不使用 |
| Info.plist / xcconfig | ❌ 低 | ✅ 可提取 | 仅存非敏感配置 |
| Keychain | ✅ 高 | ❌ 加密保护 | ✅ 推荐 |
| 服务端动态下发 | ✅✅ 最高 | ❌ 不在客户端 | ✅ 最推荐 |

---

## 7. 网络安全审计工具

### 7.1 nscurl 诊断

macOS 自带的 `nscurl` 工具可以检测服务器是否满足 ATS 要求：

```bash
# 基础 ATS 检测
nscurl --ats-diagnostics https://api.example.com

# 输出示例
# ATS Default Connection
# ATS Direct Connection
#   TLSv1.2 ✅
#   TLSv1.3 ✅
# ATS Exception Allows Insecure HTTP Loads ❌
```

常用诊断命令：

| 命令 | 用途 |
|------|------|
| `nscurl --ats-diagnostics <URL>` | 完整 ATS 诊断 |
| `nscurl --verbose <URL>` | 详细连接信息 |
| `nscurl -I <URL>` | 查看响应头 |

### 7.2 Charles 代理抓包

Charles 是 iOS 开发中最常用的网络调试工具，可以抓取和分析 HTTPS 请求：

**配置步骤：**

1. Mac 端：`Proxy → SSL Proxying Settings → 添加 *.example.com:443`
2. iOS 端：`设置 → Wi-Fi → 配置代理 → 手动`，填入 Mac 的 IP 和端口（默认 8888）
3. 安装 Charles 根证书：`Help → SSL Proxying → Install Charles Root Certificate on Mobile Device`

> ⚠️ **安全提示**：Charles 抓包本质上就是一种中间人攻击。生产环境的 App 应通过 Certificate Pinning 阻止 Charles 抓包，保护用户数据安全。

**Debug 环境允许抓包的配置：**

```swift
#if DEBUG
let session = URLSession(configuration: .default)
#else
let session = URLSession(configuration: .default, delegate: PinningDelegate(...), delegateQueue: nil)
#endif
```

### 7.3 SSL Labs 在线测试

[SSL Labs](https://www.ssllabs.com/ssltest/) 是最权威的 TLS 配置在线检测工具：

| 评级 | 含义 | 建议 |
|------|------|------|
| **A+** | 最佳配置 | ✅ 目标等级 |
| **A** | 优秀配置 | ✅ 可接受 |
| **B** | 有轻微问题 | ⚠️ 建议优化 |
| **C** | 有明显问题 | ❌ 必须修复 |
| **F** | 严重安全问题 | ❌ 紧急修复 |
| **T** | 证书不被信任 | ❌ 不可用于生产 |

检测要点：
- TLS 版本支持情况
- 加密套件强度
- 证书链完整性
- 已知漏洞（如 Heartbleed、POODLE）
- HSTS 配置

---

## 8. 国内等保合规要求

### 8.1 等保等级与网络安全要求

信息安全等级保护（简称"等保"）是中国网络安全法规定的合规要求，App 上架前可能需要通过等保测评：

| 等保等级 | 适用场景 | 网络安全要求 | 与 App 的关系 |
|---------|---------|-------------|-------------|
| **一级** | 一般信息系统 | 基本安全防护 | 大多数 App 最低要求 |
| **二级** | 重要信息系统 | 系统性安全防护 | 涉及用户信息的 App |
| **三级** | 核心信息系统 | 专业化安全防护 | 金融、医疗、政务类 App |

### 8.2 数据传输加密要求

等保对数据传输的核心要求：

| 要求项 | 等保二级 | 等保三级 | 实现方式 |
|-------|---------|---------|---------|
| 传输加密 | 通信过程加密 | 采用密码技术加密 | TLS 1.2+ / 应用层加密 |
| 完整性校验 | 检测篡改 | 密码技术保证完整性 | HMAC / 数字签名 |
| 身份鉴别 | 单因素认证 | 双因素认证 | Token + 短信/生物识别 |
| 通信可信 | 基本验证 | 强身份认证 | Certificate Pinning |

### 8.3 日志审计

等保要求对网络访问行为进行记录和审计：

```swift
import OSLog

enum NetworkAuditLogger {
    private static let logger = Logger(subsystem: "com.example.app.network", category: "audit")

    static func logRequest(url: String, method: String, statusCode: Int, duration: TimeInterval) {
        logger.info("""
        [网络审计] 请求完成
        URL: \(url, privacy: .private)
        方法: \(method)
        状态码: \(statusCode)
        耗时: \(String(format: "%.2f", duration))s
        时间: \(Date().ISO8601Format())
        """)
    }

    static func logSecurityEvent(event: String, detail: String) {
        logger.warning("""
        [安全事件] \(event)
        详情: \(detail, privacy: .private)
        时间: \(Date().ISO8601Format())
        """)
    }
}
```

> 💡 **隐私合规**：日志中不应记录完整的请求参数和响应内容，特别是密码、Token 等敏感信息。使用 `privacy: .private` 标记敏感字段，系统会在日志导出时自动脱敏。

### 8.4 合规检查清单

| 检查项 | 要求 | 状态 |
|-------|------|:----:|
| 所有远程接口使用 HTTPS | TLS 1.2+ | ☐ |
| 敏感数据应用层加密 | RSA/AES | ☐ |
| 请求签名防篡改 | HMAC-SHA256 | ☐ |
| 时间戳 + nonce 防重放 | 服务端 5 分钟窗口 | ☐ |
| Certificate Pinning | 公钥固定 | ☐ |
| API Key 安全存储 | Keychain | ☐ |
| 网络行为日志审计 | OSLog + 脱敏 | ☐ |
| 隐私政策声明网络使用 | 信息收集说明 | ☐ |
| 无硬编码密钥/密码 | 代码扫描确认 | ☐ |

---

## 本章小结

| 主题 | 核心要点 |
|------|---------|
| **网络安全概述** | HTTPS 解决窃听、篡改、冒充三大问题，中间人攻击是主要威胁 |
| **ATS 配置** | 默认强制 HTTPS + TLS 1.2，用 `NSExceptionDomains` 精准豁免 |
| **ATS 豁免与审核** | 必须说明豁免原因和过渡计划，国内 HTTP 接口可自建 HTTPS 代理 |
| **证书固定** | 公钥固定优于证书固定，URLSessionDelegate 和 Alamofire 均可实现 |
| **TLS 最佳实践** | 优先 TLS 1.3，自签名证书仅限 Debug 环境 |
| **请求安全** | 签名防篡改、时间戳防重放、敏感参数加密、Key 存 Keychain |
| **审计工具** | nscurl 诊断 ATS、Charles 抓包调试、SSL Labs 在线检测 |
| **等保合规** | 二级需通信加密+完整性校验，三级需双因素认证+强身份验证 |

> 💡 **一句话总结**：网络安全不是可选项，而是 App 上架的必答题。从 ATS 配置到证书固定，从请求签名到等保合规，每一层防护都在为用户数据筑起一道墙。安全投入越早，修复成本越低。

⬅️ [深度链接与Universal-Links](./深度链接与Universal-Links.md) ｜

← [深度链接与 Universal Links](./深度链接与Universal-Links.md) | [Web 视图与 Safari Services](./Web视图与Safari-Services.md) →
