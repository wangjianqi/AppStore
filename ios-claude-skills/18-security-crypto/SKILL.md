---
name: security-crypto
description: 涉及加密、安全、CryptoKit、Keychain 最佳实践、Certificate Pinning、App Attest、数据加密、HTTPS、防抓包、防篡改、安全存储的任务
---

# 安全与加密

## 安全分层

| 层级 | 威胁 | 防护手段 |
|------|------|---------|
| 传输层 | 中间人攻击、流量劫持 | HTTPS + Certificate Pinning |
| 存储层 | 数据泄露、越狱读取 | Keychain + Data Protection |
| 代码层 | 逆向工程、注入 | 混淆 + App Attest |
| 认证层 | Token 窃取、重放攻击 | JWT + 短有效期 + PKCE |
| API 层 | 滥用、爬虫 | 速率限制 + 签名验证 |

---

## CryptoKit

### 哈希

```swift
import CryptoKit

func sha256(_ data: Data) -> String {
    let hash = SHA256.hash(data: data)
    return hash.compactMap { String(format: "%02x", $0) }.joined()
}

func sha256(_ string: String) -> String {
    guard let data = string.data(using: .utf8) else { return "" }
    return sha256(data)
}
```

### HMAC（消息认证码）

```swift
func hmac(key: Data, message: Data) -> String {
    let key = SymmetricKey(data: key)
    let signature = HMAC<SHA256>.authenticationCode(for: message, using: key)
    return signature.compactMap { String(format: "%02x", $0) }.joined()
}
```

### AES 加解密

```swift
func encrypt(plaintext: Data, key: SymmetricKey) throws -> Data {
    let sealedBox = try AES.GCM.seal(plaintext, using: key)
    return sealedBox.combined!
}

func decrypt(ciphertext: Data, key: SymmetricKey) throws -> Data {
    let sealedBox = try AES.GCM.SealedBox(combined: ciphertext)
    return try AES.GCM.open(sealedBox, using: key)
}

func generateAESKey() -> SymmetricKey {
    SymmetricKey(size: .bits256)
}
```

### 签名验证

```swift
func sign(data: Data, privateKey: P256.Signing.PrivateKey) throws -> Data {
    let signature = try privateKey.signature(for: data)
    return signature.rawRepresentation
}

func verify(data: Data, signature: Data, publicKey: P256.Signing.PublicKey) -> Bool {
    guard let sig = try? P256.Signing.ECDSASignature(rawRepresentation: signature) else {
        return false
    }
    return publicKey.isValidSignature(sig, for: data)
}
```

### Key Agreement（密钥协商）

```swift
func sharedSecret(privateKey: P256.KeyAgreement.PrivateKey, publicKey: P256.KeyAgreement.PublicKey) throws -> SharedSecret {
    return try privateKey.sharedSecretFromKeyAgreement(with: publicKey)
}
```

### 规范
- **禁止使用 MD5、SHA1**（已被攻破），统一用 SHA256 或以上
- **禁止自研加密算法**，只用 CryptoKit 或 Security 框架
- AES 使用 GCM 模式（认证加密），禁止 ECB 模式
- 密钥通过 Keychain 存储，**禁止硬编码在代码中**
- 密钥长度：AES 256 位，RSA 2048 位以上，ECDSA P-256 以上

---

## Keychain 最佳实践

### 访问控制

```swift
func saveToKeychain(data: Data, key: String, biometric: Bool = false) throws {
    var query: [String: Any] = [
        kSecClass as String: kSecClassGenericPassword,
        kSecAttrService as String: Bundle.main.bundleIdentifier ?? "com.app",
        kSecAttrAccount as String: key,
        kSecValueData as String: data,
    ]

    if biometric {
        let access = SecAccessControlCreateWithFlags(
            nil,
            kSecAttrAccessibleWhenUnlockedThisDeviceOnly,
            .biometryCurrentSet,
            nil
        )!
        query[kSecAttrAccessControl as String] = access
    } else {
        query[kSecAttrAccessible as String] = kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly
    }

    SecItemDelete(query as CFDictionary)
    let status = SecItemAdd(query as CFDictionary, nil)
    guard status == errSecSuccess else {
        throw SecurityError.keychainSaveFailed(status)
    }
}
```

### 访问级别选择

| 级别 | 说明 | 适用场景 |
|------|------|---------|
| `afterFirstUnlock` | 首次解锁后可访问（含后台） | 后台推送需要读取的 Token |
| `afterFirstUnlockThisDeviceOnly` | 同上，不跨设备迁移 | 设备绑定 Token |
| `whenUnlocked` | 仅解锁状态可访问 | 敏感数据（不后台访问） |
| `whenUnlockedThisDeviceOnly` | 同上，不跨设备迁移 | 最高安全级别 |
| `biometryCurrentSet` | 需要生物识别 | 支付密码、加密密钥 |

### 规范
- **Token 类数据用 `afterFirstUnlock`**（后台推送需要读取）
- **支付/加密密钥用 `whenUnlockedThisDeviceOnly`**（最高安全）
- **需要 Face ID/Touch ID 验证的用 `biometryCurrentSet`**（换生物识别后密钥失效）
- 禁止使用 `always`（已废弃，不安全）
- App Group 共享 Keychain：配置 `kSecAttrAccessGroup`

---

## Certificate Pinning（证书锁定）

### URLSession 实现

```swift
final class PinningDelegate: NSObject, URLSessionDelegate {
    private let pinnedCertificates: [Data]

    init(certNames: [String]) {
        self.pinnedCertificates = certNames.compactMap { name in
            let url = Bundle.main.url(forResource: name, withExtension: "cer")!
            return try? Data(contentsOf: url)
        }
    }

    func urlSession(_ session: URLSession, didReceive challenge: URLAuthenticationChallenge,
        completionHandler: @escaping (URLSession.AuthChallengeDisposition, URLCredential?) -> Void) {

        guard challenge.protectionSpace.authenticationMethod == NSURLAuthenticationMethodServerTrust,
              let serverTrust = challenge.protectionSpace.serverTrust else {
            completionHandler(.cancelAuthenticationChallenge, nil)
            return
        }

        let policy = SecPolicyCreateSSL(true, challenge.protectionSpace.host as CFString)
        SecTrustSetPolicies(serverTrust, policy)

        var trustResult: SecTrustResultType = .invalid
        SecTrustEvaluate(serverTrust, &trustResult)

        guard trustResult == .unspecified || trustResult == .proceed else {
            completionHandler(.cancelAuthenticationChallenge, nil)
            return
        }

        let serverCerts = SecTrustCopyCertificateChain(serverTrust) as! [SecCertificate]
        guard let serverCertData = SecCertificateCopyData(serverCerts[0]) as Data? else {
            completionHandler(.cancelAuthenticationChallenge, nil)
            return
        }

        if pinnedCertificates.contains(serverCertData) {
            completionHandler(.useCredential, URLCredential(trust: serverTrust))
        } else {
            completionHandler(.cancelAuthenticationChallenge, nil)
        }
    }
}
```

### 公钥 Pinning（推荐）
- 比证书 Pinning 更灵活：证书轮换时只要公钥不变就不影响
- 提取公钥：

```swift
func extractPublicKey(from certName: String) -> SecKey? {
    guard let url = Bundle.main.url(forResource: certName, withExtension: "cer"),
          let data = try? Data(contentsOf: url),
          let cert = SecCertificateCreateWithData(nil, data as CFData) else { return nil }
    var trust: SecTrust?
    let policy = SecPolicyCreateSSL(true, "" as CFString)
    SecTrustCreateWithCertificates(cert, policy, &trust)
    return SecTrustCopyKey(trust!)
}
```

### 规范
- **生产环境必须启用 Certificate Pinning**，防止中间人攻击
- 至少 Pin 2 个证书/公钥（主证书 + 备份证书），避免证书更新后 App 无法联网
- Pin 失败时**禁止静默降级为不验证**，应提示用户网络不安全
- 证书文件（`.cer`）放 Bundle，**禁止从服务端动态下载验证证书**
- 测试环境可通过 `Build Configuration` 跳过 Pinning

---

## App Attest（防篡改）

### 用途
- 证明请求来自正版 App（非越狱/重签名版本）
- 防止 API 被伪造客户端调用

### 流程

```swift
import DeviceCheck

func attestKey(keyId: String, challenge: Data) async throws -> String {
    let service = DCAppAttestService.shared
    guard service.isSupported else {
        throw SecurityError.appAttestNotSupported
    }

    let keyId = try await service.generateKey()
    let clientDataHash = SHA256.hash(data: challenge).data
    let attestation = try await service.attestKey(keyId, clientDataHash: clientDataHash)

    try await sendAttestationToServer(keyId: keyId, attestation: attestation)

    return keyId
}

func generateAssertion(keyId: String, request: URLRequest) async throws -> Data {
    let service = DCAppAttestService.shared
    let clientDataHash = SHA256.hash(data: request.httpBody ?? Data()).data
    return try await service.generateAssertion(keyId, clientDataHash: clientDataHash)
}
```

### 服务端验证
1. 验证 `attestation` 对象的签名链（Apple 根证书）
2. 验证 `challenge` 匹配
3. 存储 `keyId` 与用户绑定
4. 后续请求验证 `assertion` 签名

### 规范
- iOS 14+ 支持，低版本需降级处理
- `isSupported` 在模拟器上返回 `false`
- 每个设备每个 App 最多生成 3 个 Key
- **App Attest 不能替代完整的服务端验证**，是辅助手段
- 审核不要求必须实现 App Attest，但金融/支付类 App 强烈建议

---

## HTTPS 与网络安全

### ATS (App Transport Security)
- iOS 9+ 默认强制 HTTPS
- **禁止在 Release 中使用 `NSAllowsArbitraryLoads = true`**
- 特定域名例外用 `NSExceptionDomains`，需说明理由
- Web View 中的内容也受 ATS 限制

### 防抓包
- Certificate Pinning 防止 Charles/Burp 等代理抓包
- 越狱检测（辅助手段，不可靠）：

```swift
func isJailbroken() -> Bool {
    let paths = [
        "/Applications/Cydia.app",
        "/Library/MobileSubstrate/MobileSubstrate.dylib",
        "/bin/bash",
        "/usr/sbin/sshd",
        "/etc/apt"
    ]
    return paths.contains { FileManager.default.fileExists(atPath: $0) }
}
```

- **禁止依赖越狱检测做安全防护**（可被绕过），仅做辅助提示

---

## 数据安全

### 文件保护

```swift
func writeProtected(data: Data, to url: URL) throws {
    try data.write(to: url, options: [.completeFileProtection])
}
```

| 保护级别 | 说明 |
|---------|------|
| `completeFileProtection` | 设备锁定时不可访问 |
| `completeFileProtectionUnlessOpen` | 锁定时已打开的文件可继续访问 |
| `completeFileProtectionUntilFirstUserAuthentication` | 首次解锁后可访问 |
| `noFileProtection` | 无保护（禁止使用） |

### 剪贴板安全
- 敏感数据（密码、Token）**禁止写入剪贴板**
- 读取剪贴板时注意用户隐私

### 日志安全
- **禁止在 Release 日志中输出 Token、密码、密钥**
- Debug 日志用 `#if DEBUG` 包裹
- 第三方 SDK 日志在 Release 中关闭

### URL Scheme 安全
- 自定义 URL Scheme 可被其他 App 仿冒
- 敏感操作改用 Universal Links（关联域名验证）
- URL Scheme 参数不做信任假设，必须验证

---

## 已知陷阱

- **Certificate Pinning 在越狱设备上可被绕过**（SSL Kill Switch 等工具），需配合 App Attest
- **Keychain 数据在设备解锁后可被所有进程访问**，极高安全场景用 Secure Enclave
- **`Data(contentsOf: url)` 读取 Keychain 证书文件可能失败**，用 `SecCertificateCreateWithData`
- **App Attest 在模拟器上不可用**，Debug 构建需跳过验证
- **AES-GCM 的 `combined` 属性在加密失败时为 nil**，必须强制解包前检查
- **HMAC 密钥泄露等于签名失效**，密钥必须存 Keychain
- **`SecAccessControlCreateWithFlags` 在旧设备上可能返回 nil**（不支持生物识别），需降级处理
