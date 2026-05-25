# 63-Keychain 与数据安全

## 本章目标

- 理解 iOS 数据安全的分层体系，掌握不同敏感度数据的存储策略
- 深入理解 Keychain Services 的原理与适用场景
- 掌握 Keychain 的增删改查四大操作
- 封装一个泛型 KeychainHelper 工具类，支持 Codable 对象存取
- 了解 Keychain 访问控制、Biometry 保护与 Secure Enclave
- 掌握 Keychain Sharing 实现多 App 凭据共享
- 学会使用 CommonCrypto 和 CryptoKit 进行数据加密
- 掌握网络安全配置：ATS、Certificate Pinning、URLSession 安全设置
- 建立安全最佳实践意识，避免常见安全漏洞
- 了解隐私清单中与数据安全相关的声明

---

## 1. iOS 数据安全概述

### 1.1 数据分类

在 iOS 开发中，不是所有数据都需要同样的保护力度。就像生活中，你不会把零钱和存折放在同一个地方——零钱放口袋，存折放保险柜。

根据敏感程度，数据通常分为四个等级：

| 分类 | 说明 | 典型数据 | 存储建议 |
|------|------|---------|---------|
| **公开** | 泄露后无任何影响 | App 主题色、默认设置 | UserDefaults |
| **内部** | 泄露后有轻微影响 | 用户偏好、缓存数据 | UserDefaults / 文件 |
| **敏感** | 泄露后有较大影响 | Token、用户 ID、邮箱 | Keychain / 加密存储 |
| **机密** | 泄露后有严重影响 | 密码、私钥、支付信息 | Keychain + Secure Enclave |

> 💡 **生活类比**：数据分类就像家里的收纳——客厅放杂志（公开），书房放工作文件（内部），卧室放日记（敏感），保险柜放房产证（机密）。

### 1.2 存储位置选择

iOS 提供了多种存储位置，每种都有不同的安全特性：

| 存储方式 | 加密保护 | App 卸载后 | 跨 App 共享 | 适合数据等级 |
|---------|:-------:|:---------:|:----------:|:----------:|
| **UserDefaults** | ❌ 无 | 清除 | ❌ | 公开 / 内部 |
| **文件系统** | ❌ 无 | 清除 | ❌ | 公开 / 内部 |
| **SwiftData / Core Data** | ❌ 无 | 清除 | ❌ | 内部 / 敏感（需加密） |
| **Keychain** | ✅ 硬件加密 | 可保留 | ✅ | 敏感 / 机密 |

> ⚠️ **重要提醒**：UserDefaults 和文件系统存储的数据是**明文**的，越狱设备上可以直接读取。绝不要把密码、Token 等敏感信息存在 UserDefaults 中！

---

## 2. Keychain Services 详解

### 2.1 什么是 Keychain

Keychain 是 iOS 提供的一个**加密数据库**，专门用于安全存储小型敏感数据，如密码、证书、Token 等。它由系统的 **securityd** 进程管理，底层使用 AES 加密，密钥由设备硬件保护。

```
┌─────────────────────────────────────────────┐
│                 App 进程                      │
│         通过 Keychain API 访问               │
└──────────────────┬──────────────────────────┘
                   │ SecItemAdd / SecItemCopyMatching ...
                   ▼
┌─────────────────────────────────────────────┐
│            securityd 进程                     │
│     管理访问权限，执行加密/解密操作            │
└──────────────────┬──────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────┐
│          Keychain 加密数据库                  │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐     │
│  │ 密码条目  │ │ Token条目│ │ 证书条目  │     │
│  └──────────┘ └──────────┘ └──────────┘     │
│         AES-256 加密存储                      │
└─────────────────────────────────────────────┘
```

### 2.2 Keychain 与 UserDefaults 的区别

| 对比项 | UserDefaults | Keychain |
|-------|:-----------:|:--------:|
| 加密 | ❌ 明文存储 | ✅ AES-256 加密 |
| 数据类型 | 基本类型（String、Int 等） | 主要是 Data / String |
| App 卸载 | 数据清除 | 可保留 |
| 跨 App 共享 | ❌ | ✅（Keychain Sharing） |
| 生物识别保护 | ❌ | ✅ |
| 存储大小 | 适合小量数据 | 适合小量敏感数据 |
| API 复杂度 | 简单 | 较复杂（C API） |
| 性能 | 快 | 稍慢（涉及加密） |

> 💡 **生活类比**：UserDefaults 像你桌上的便签本，谁路过都能看到；Keychain 像银行保险柜，需要身份验证才能打开，而且内容是加密的。

### 2.3 Keychain 适用场景

- 用户密码和登录凭据
- OAuth Token / JWT Token
- API 密钥
- 加密密钥（对称密钥、私钥）
- 身份证书
- 需要跨 App 共享的安全数据

---

## 3. Keychain 基本操作

Keychain 使用 C 风格的 API，核心是四个函数，对应增删改查：

| 操作 | 函数 | 说明 |
|------|------|------|
| **增** | `SecItemAdd` | 添加一条 Keychain 条目 |
| **删** | `SecItemDelete` | 删除匹配的条目 |
| **改** | `SecItemUpdate` | 更新匹配的条目 |
| **查** | `SecItemCopyMatching` | 查询匹配的条目 |

### 3.1 Keychain 条目的核心属性

每个 Keychain 条目由一组键值对（Attribute）描述：

| 属性键 | 说明 | 示例 |
|-------|------|------|
| `kSecClass` | 条目类型 | `kSecClassGenericPassword` |
| `kSecAttrAccount` | 账号标识（key） | `"userToken"` |
| `kSecAttrService` | 服务标识 | `"com.myapp.service"` |
| `kSecValueData` | 存储的数据值 | `token.data(using: .utf8)` |
| `kSecAttrAccessible` | 访问控制 | `kSecAttrAccessibleWhenUnlocked` |

### 3.2 添加条目（SecItemAdd）

```swift
func saveToken(_ token: String) -> Bool {
    let query: [String: Any] = [
        kSecClass as String: kSecClassGenericPassword,
        kSecAttrAccount as String: "userToken",
        kSecAttrService as String: Bundle.main.bundleIdentifier ?? "com.myapp",
        kSecValueData as String: token.data(using: .utf8)!,
        kSecAttrAccessible as String: kSecAttrAccessibleWhenUnlocked
    ]

    let status = SecItemAdd(query as CFDictionary, nil)
    return status == errSecSuccess
}
```

### 3.3 查询条目（SecItemCopyMatching）

```swift
func loadToken() -> String? {
    let query: [String: Any] = [
        kSecClass as String: kSecClassGenericPassword,
        kSecAttrAccount as String: "userToken",
        kSecAttrService as String: Bundle.main.bundleIdentifier ?? "com.myapp",
        kSecReturnData as String: true,
        kSecMatchLimit as String: kSecMatchLimitOne
    ]

    var result: AnyObject?
    let status = SecItemCopyMatching(query as CFDictionary, &result)

    guard status == errSecSuccess, let data = result as? Data else { return nil }
    return String(data: data, encoding: .utf8)
}
```

### 3.4 更新条目（SecItemUpdate）

```swift
func updateToken(_ newToken: String) -> Bool {
    let query: [String: Any] = [
        kSecClass as String: kSecClassGenericPassword,
        kSecAttrAccount as String: "userToken",
        kSecAttrService as String: Bundle.main.bundleIdentifier ?? "com.myapp"
    ]

    let attributes: [String: Any] = [
        kSecValueData as String: newToken.data(using: .utf8)!
    ]

    let status = SecItemUpdate(query as CFDictionary, attributes as CFDictionary)
    return status == errSecSuccess
}
```

### 3.5 删除条目（SecItemDelete）

```swift
func deleteToken() -> Bool {
    let query: [String: Any] = [
        kSecClass as String: kSecClassGenericPassword,
        kSecAttrAccount as String: "userToken",
        kSecAttrService as String: Bundle.main.bundleIdentifier ?? "com.myapp"
    ]

    let status = SecItemDelete(query as CFDictionary)
    return status == errSecSuccess
}
```

> ⚠️ **注意**：Keychain API 的返回值是 `OSStatus`，务必检查是否为 `errSecSuccess`。常见错误码：`errSecItemNotFound`（条目不存在）、`errSecDuplicateItem`（条目已存在）。

---

## 4. 封装 KeychainHelper 工具类

原生 Keychain API 是 C 风格的，使用繁琐。我们封装一个 Swift 风格的工具类，支持泛型和 Codable。

### 4.1 基础封装

```swift
import Security
import Foundation

enum KeychainHelper {

    static let service = Bundle.main.bundleIdentifier ?? "com.app.default"

    enum KeychainError: Error {
        case duplicateEntry
        case entryNotFound
        case invalidData
        case unexpectedStatus(OSStatus)
    }

    private static func baseQuery(for key: String) -> [String: Any] {
        [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key,
            kSecAttrService as String: service
        ]
    }

    static func save(data: Data, for key: String) throws {
        var query = baseQuery(for: key)
        query[kSecValueData as String] = data
        query[kSecAttrAccessible as String] = kSecAttrAccessibleWhenUnlocked

        let status = SecItemAdd(query as CFDictionary, nil)

        if status == errSecDuplicateItem {
            try update(data: data, for: key)
        } else if status != errSecSuccess {
            throw KeychainError.unexpectedStatus(status)
        }
    }

    static func load(for key: String) throws -> Data {
        var query = baseQuery(for: key)
        query[kSecReturnData as String] = true
        query[kSecMatchLimit as String] = kSecMatchLimitOne

        var result: AnyObject?
        let status = SecItemCopyMatching(query as CFDictionary, &result)

        guard status == errSecSuccess else {
            if status == errSecItemNotFound {
                throw KeychainError.entryNotFound
            }
            throw KeychainError.unexpectedStatus(status)
        }
        guard let data = result as? Data else {
            throw KeychainError.invalidData
        }
        return data
    }

    static func update(data: Data, for key: String) throws {
        let query = baseQuery(for: key)
        let attributes: [String: Any] = [
            kSecValueData as String: data
        ]

        let status = SecItemUpdate(query as CFDictionary, attributes as CFDictionary)
        guard status == errSecSuccess else {
            if status == errSecItemNotFound {
                throw KeychainError.entryNotFound
            }
            throw KeychainError.unexpectedStatus(status)
        }
    }

    static func delete(for key: String) throws {
        let query = baseQuery(for: key)
        let status = SecItemDelete(query as CFDictionary)
        guard status == errSecSuccess || status == errSecItemNotFound else {
            throw KeychainError.unexpectedStatus(status)
        }
    }
}
```

### 4.2 泛型 + Codable 扩展

```swift
extension KeychainHelper {

    static func save<T: Codable>(_ item: T, for key: String) throws {
        let data = try JSONEncoder().encode(item)
        try save(data: data, for: key)
    }

    static func load<T: Codable>(_ type: T.Type, for key: String) throws -> T {
        let data = try load(for: key)
        return try JSONDecoder().decode(type, from: data)
    }
}
```

### 4.3 使用示例

```swift
struct AuthToken: Codable {
    let accessToken: String
    let refreshToken: String
    let expiresAt: Date
}

// 保存
let token = AuthToken(
    accessToken: "eyJhbGciOi...",
    refreshToken: "dGhpcyBpcy...",
    expiresAt: Date().addingTimeInterval(3600)
)
try KeychainHelper.save(token, for: "authToken")

// 读取
let savedToken = try KeychainHelper.load(AuthToken.self, for: "authToken")
print(savedToken.accessToken)

// 删除
try KeychainHelper.delete(for: "authToken")
```

> 💡 **设计思路**：`save` 方法内部自动处理"已存在则更新"的逻辑，避免调用方需要先查询再决定添加还是更新，简化使用流程。

---

## 5. Keychain 访问控制

### 5.1 kSecAttrAccessible 选项

`kSecAttrAccessible` 决定了 Keychain 条目**何时可以被访问**：

| 选项 | 设备锁定时可访问 | 说明 |
|------|:---:|------|
| `kSecAttrAccessibleWhenUnlocked` | ❌ | **默认值**，仅设备解锁后可访问 |
| `kSecAttrAccessibleWhenUnlockedThisDeviceOnly` | ❌ | 同上，但不迁移到新设备 |
| `kSecAttrAccessibleAfterFirstUnlock` | ✅ | 首次解锁后，即使再锁定也可访问 |
| `kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly` | ✅ | 同上，但不迁移到新设备 |
| `kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly` | ❌ | 仅当设备设置了密码时可用 |

> ⚠️ **推荐**：大多数场景使用 `kSecAttrAccessibleWhenUnlockedThisDeviceOnly`。它既保证安全（锁定时不可访问），又防止数据通过备份迁移到其他设备。

### 5.2 Biometry 保护

从 iOS 9 开始，Keychain 支持通过生物识别（Face ID / Touch ID）保护条目访问：

```swift
import LocalAuthentication

func saveWithBiometry(data: Data, for key: String) throws {
    let access = SecAccessControlCreateWithFlags(
        nil,
        kSecAttrAccessibleWhenUnlockedThisDeviceOnly,
        .biometryCurrentSet,
        nil
    )!

    var query = baseQuery(for: key)
    query[kSecValueData as String] = data
    query[kSecAttrAccessControl as String] = access

    let context = LAContext()
    query[kSecUseAuthenticationContext as String] = context

    let status = SecItemAdd(query as CFDictionary, nil)
    guard status == errSecSuccess else {
        throw KeychainError.unexpectedStatus(status)
    }
}

func loadWithBiometry(for key: String) throws -> Data {
    var query = baseQuery(for: key)
    query[kSecReturnData as String] = true
    query[kSecMatchLimit as String] = kSecMatchLimitOne

    let context = LAContext()
    context.localizedReason = "需要验证身份以访问安全数据"
    query[kSecUseAuthenticationContext as String] = context

    var result: AnyObject?
    let status = SecItemCopyMatching(query as CFDictionary, &result)
    guard status == errSecSuccess, let data = result as? Data else {
        throw KeychainError.entryNotFound
    }
    return data
}
```

> 💡 **`.biometryCurrentSet`** 的含义：只有当前录入的生物特征才能解锁。如果用户重新录入了 Face ID / Touch ID，之前受保护的条目将**无法再访问**。这提供了更强的安全保障。

### 5.3 Secure Enclave

Secure Enclave 是 Apple 芯片中独立的安全子系统，拥有自己的微处理器和内存，即使内核被攻破也无法读取其中的密钥。

```swift
func saveToSecureEnclave(key: String, data: Data) throws {
    let access = SecAccessControlCreateWithFlags(
        nil,
        kSecAttrAccessibleWhenUnlockedThisDeviceOnly,
        [.biometryCurrentSet, .privateKeyUsage],
        nil
    )!

    var query = baseQuery(for: key)
    query[kSecValueData as String] = data
    query[kSecAttrAccessControl as String] = access
    query[kSecAttrTokenID as String] = kSecAttrTokenIDSecureEnclave

    let status = SecItemAdd(query as CFDictionary, nil)
    guard status == errSecSuccess else {
        throw KeychainError.unexpectedStatus(status)
    }
}
```

| 保护级别 | 安全性 | 适用场景 |
|---------|:-----:|---------|
| `kSecAttrAccessibleWhenUnlocked` | ⭐⭐ | 普通敏感数据 |
| Biometry 保护 | ⭐⭐⭐ | 支付密码、重要凭据 |
| Secure Enclave + Biometry | ⭐⭐⭐⭐ | 加密私钥、支付密钥 |

---

## 6. Keychain Sharing

### 6.1 为什么需要共享 Keychain

实际开发中，经常需要在多个 App 之间共享凭据：

- 主 App 和 Extension（Widget、Share Extension）共享登录状态
- 同一公司的多个 App 单点登录（SSO）
- App 换新 Bundle ID 后迁移用户数据

### 6.2 App Group 共享 Keychain

**第一步**：在 Apple Developer 中创建 App Group，并在所有需要共享的 App 的 Entitlements 中启用。

**第二步**：在 Xcode 中开启 Keychain Sharing capability，设置相同的 Keychain Group。

**第三步**：代码中使用共享的 `kSecAttrAccessGroup`：

```swift
extension KeychainHelper {

    static let sharedAccessGroup = "group.com.mycompany.apps"

    private static func sharedQuery(for key: String) -> [String: Any] {
        var query = baseQuery(for: key)
        query[kSecAttrAccessGroup as String] = sharedAccessGroup
        return query
    }

    static func saveShared(data: Data, for key: String) throws {
        var query = sharedQuery(for: key)
        query[kSecValueData as String] = data
        query[kSecAttrAccessible as String] = kSecAttrAccessibleWhenUnlocked

        let status = SecItemAdd(query as CFDictionary, nil)
        if status == errSecDuplicateItem {
            try updateShared(data: data, for: key)
        } else if status != errSecSuccess {
            throw KeychainError.unexpectedStatus(status)
        }
    }

    static func loadShared(for key: String) throws -> Data {
        var query = sharedQuery(for: key)
        query[kSecReturnData as String] = true
        query[kSecMatchLimit as String] = kSecMatchLimitOne

        var result: AnyObject?
        let status = SecItemCopyMatching(query as CFDictionary, &result)
        guard status == errSecSuccess, let data = result as? Data else {
            throw KeychainError.entryNotFound
        }
        return data
    }

    static func updateShared(data: Data, for key: String) throws {
        let query = sharedQuery(for: key)
        let attributes: [String: Any] = [kSecValueData as String: data]
        let status = SecItemUpdate(query as CFDictionary, attributes as CFDictionary)
        guard status == errSecSuccess else {
            throw KeychainError.unexpectedStatus(status)
        }
    }
}
```

### 6.3 多 App 共享凭据流程

```
┌─────────────┐     ┌──────────────────────┐     ┌─────────────┐
│   App A     │     │   共享 Keychain       │     │   App B     │
│  (主应用)    │────▶│  Access Group:       │◀────│  (扩展应用)  │
│  登录并保存  │     │  group.com.mycompany │     │  读取凭据    │
│  凭据       │     │                      │     │  自动登录    │
└─────────────┘     └──────────────────────┘     └─────────────┘
```

> ⚠️ **注意**：共享 Keychain 的 App 必须来自**同一开发者账号**，且在 Entitlements 中配置了相同的 App Group 和 Keychain Group。不同开发者的 App 无法共享 Keychain。

---

## 7. 数据加密

### 7.1 CommonCrypto 对称加密（AES）

CommonCrypto 是 C 语言库，提供 AES 等对称加密算法。适合需要兼容旧系统的场景。

```swift
import CommonCrypto

enum AESCryptor {

    static func encrypt(data: Data, key: Data, iv: Data) throws -> Data {
        var encryptedData = Data(count: data.count + kCCBlockSizeAES128)
        var numBytesEncrypted = 0

        let status = key.withUnsafeBytes { keyBytes in
            iv.withUnsafeBytes { ivBytes in
                data.withUnsafeBytes { dataBytes in
                    encryptedData.withUnsafeMutableBytes { encryptedBytes in
                        CCCrypt(
                            CCOperation(kCCEncrypt),
                            CCAlgorithm(kCCAlgorithmAES),
                            CCOptions(kCCOptionPKCS7Padding),
                            keyBytes.baseAddress,
                            kCCKeySizeAES256,
                            ivBytes.baseAddress,
                            dataBytes.baseAddress,
                            data.count,
                            encryptedBytes.baseAddress,
                            encryptedData.count,
                            &numBytesEncrypted
                        )
                    }
                }
            }
        }

        guard status == kCCSuccess else {
            throw NSError(domain: "AESCryptor", code: Int(status))
        }
        encryptedData.count = numBytesEncrypted
        return encryptedData
    }

    static func decrypt(data: Data, key: Data, iv: Data) throws -> Data {
        var decryptedData = Data(count: data.count + kCCBlockSizeAES128)
        var numBytesDecrypted = 0

        let status = key.withUnsafeBytes { keyBytes in
            iv.withUnsafeBytes { ivBytes in
                data.withUnsafeBytes { dataBytes in
                    decryptedData.withUnsafeMutableBytes { decryptedBytes in
                        CCCrypt(
                            CCOperation(kCCDecrypt),
                            CCAlgorithm(kCCAlgorithmAES),
                            CCOptions(kCCOptionPKCS7Padding),
                            keyBytes.baseAddress,
                            kCCKeySizeAES256,
                            ivBytes.baseAddress,
                            dataBytes.baseAddress,
                            data.count,
                            decryptedBytes.baseAddress,
                            decryptedData.count,
                            &numBytesDecrypted
                        )
                    }
                }
            }
        }

        guard status == kCCSuccess else {
            throw NSError(domain: "AESCryptor", code: Int(status))
        }
        decryptedData.count = numBytesDecrypted
        return decryptedData
    }
}
```

### 7.2 CryptoKit 现代加密

CryptoKit 是 Apple 在 iOS 13 推出的现代加密框架，API 更加 Swift 友好：

```swift
import CryptoKit

enum ModernCryptor {

    static func encrypt(data: Data, key: SymmetricKey) -> Data {
        let sealedBox = try! AES.GCM.seal(data, using: key)
        return sealedBox.combined!
    }

    static func decrypt(combined: Data, key: SymmetricKey) throws -> Data {
        let sealedBox = try AES.GCM.SealedBox(combined: combined)
        let decrypted = try AES.GCM.open(sealedBox, using: key)
        return decrypted
    }

    static func generateKey() -> SymmetricKey {
        SymmetricKey(size: .bits256)
    }
}
```

### 7.3 CommonCrypto vs CryptoKit 对比

| 对比项 | CommonCrypto | CryptoKit |
|-------|:-----------:|:---------:|
| 最低系统版本 | iOS 4+ | iOS 13+ |
| 语言风格 | C API，需手动管理内存 | Swift 原生，类型安全 |
| 加密模式 | CBC（需手动处理 IV） | GCM（自动认证加密） |
| 密钥管理 | 手动 | 内置类型 `SymmetricKey` |
| 完整性校验 | 需额外 HMAC | GCM 自带认证标签 |
| 推荐程度 | ⭐⭐ | ⭐⭐⭐⭐⭐ |

> 💡 **推荐**：新项目优先使用 CryptoKit。AES-GCM 模式自带认证加密（AEAD），既能保密又能防篡改，比 CBC + HMAC 的组合更安全、更简洁。

### 7.4 密钥的安全存储

加密的密钥本身也需要安全存储，最佳实践是**用 Keychain 存储加密密钥**：

```swift
func saveEncryptionKey(_ key: SymmetricKey) throws {
    try KeychainHelper.save(data: key.rawKey, for: "encryptionKey")
}

func loadEncryptionKey() throws -> SymmetricKey {
    let data = try KeychainHelper.load(for: "encryptionKey")
    return SymmetricKey(data: data)
}

extension SymmetricKey {
    var rawKey: Data {
        withUnsafeBytes { Data(Array($0)) }
    }
}
```

---

## 8. 网络安全

### 8.1 App Transport Security（ATS）

ATS 是 Apple 从 iOS 9 开始强制执行的安全策略，要求所有网络请求使用 HTTPS。

| ATS 要求 | 说明 |
|---------|------|
| 必须使用 HTTPS | 禁止明文 HTTP |
| TLS 最低版本 1.2 | 不支持 TLS 1.0/1.1 |
| 证书必须合法 | 自签名证书默认不被信任 |
| 前向保密 | 必须支持 PFS 密码套件 |

如果需要临时豁免（不推荐），在 Info.plist 中配置：

```xml
<key>NSAppTransportSecurity</key>
<dict>
    <key>NSAllowsArbitraryLoads</key>
    <false/>
    <key>NSExceptionDomains</key>
    <dict>
        <key>legacy-api.example.com</key>
        <dict>
            <key>NSExceptionAllowsInsecureHTTPLoads</key>
            <true/>
            <key>NSExceptionMinimumTLSVersion</key>
            <string>TLSv1.1</string>
        </dict>
    </dict>
</dict>
```

> ⚠️ **警告**：`NSAllowsArbitraryLoads = true` 会绕过所有 ATS 保护，App Store 审核时需要提供充分理由。新项目应全面使用 HTTPS。

### 8.2 Certificate Pinning

Certificate Pinning（证书锁定）是一种防止中间人攻击的技术，App 内置预期的证书或公钥，只信任匹配的证书。

**生活类比**：就像你只认快递员的脸，不认工牌——即使有人伪造了工牌，你也不会把包裹交给他。

#### 方式一：URLSession Delegate

```swift
class PinningDelegate: NSObject, URLSessionDelegate {

    func urlSession(
        _ session: URLSession,
        didReceive challenge: URLAuthenticationChallenge,
        completionHandler: @escaping (URLSession.AuthChallengeDisposition, URLCredential?) -> Void
    ) {
        guard let serverTrust = challenge.protectionSpace.serverTrust,
              let certificate = SecTrustCopyCertificateChain(serverTrust) as? [SecCertificate],
              let serverCert = certificate.first else {
            completionHandler(.cancelAuthenticationChallenge, nil)
            return
        }

        let serverCertData = SecCertificateCopyData(serverCert) as Data

        guard let localCertURL = Bundle.main.url(forResource: "server", withExtension: "cer"),
              let localCertData = try? Data(contentsOf: localCertURL) else {
            completionHandler(.cancelAuthenticationChallenge, nil)
            return
        }

        if serverCertData == localCertData {
            completionHandler(.useCredential, URLCredential(trust: serverTrust))
        } else {
            completionHandler(.cancelAuthenticationChallenge, nil)
        }
    }
}
```

#### 方式二：公钥锁定（更灵活）

```swift
func verifyPublicKey(serverTrust: SecTrust, expectedPublicKeyHashes: [String]) -> Bool {
    let policy = SecPolicyCreateSSL(true, nil)
    SecTrustSetPolicies(serverTrust, policy)

    guard let certificateChain = SecTrustCopyCertificateChain(serverTrust) as? [SecCertificate] else {
        return false
    }

    for certificate in certificateChain {
        let publicKey = SecCertificateCopyKey(certificate)
        let publicKeyData = SecKeyCopyExternalRepresentation(publicKey!, nil) as Data
        let hash = SHA256.hash(data: publicKeyData)
        let hashString = hash.compactMap { String(format: "%02x", $0) }.joined()

        if expectedPublicKeyHashes.contains(hashString) {
            return true
        }
    }
    return false
}
```

> 💡 **公钥锁定 vs 证书锁定**：公钥锁定更灵活——证书续期后公钥通常不变，不需要更新 App。证书锁定在证书续期后需要发新版。

### 8.3 URLSession 安全配置

```swift
func createSecureSession() -> URLSession {
    let configuration = URLSessionConfiguration.default

    configuration.timeoutIntervalForRequest = 30
    configuration.timeoutIntervalForResource = 60
    configuration.waitsForConnectivity = true
    configuration.requestCachePolicy = .reloadIgnoringLocalCacheData

    configuration.httpShouldSetCookies = false
    configuration.httpCookieAcceptPolicy = .never

    return URLSession(
        configuration: configuration,
        delegate: PinningDelegate(),
        delegateQueue: nil
    )
}
```

---

## 9. 安全最佳实践清单

### 9.1 核心原则

| 原则 | 说明 | 错误示例 | 正确做法 |
|------|------|---------|---------|
| 不硬编码密钥 | 密钥不应出现在源码中 | `let apiKey = "sk-xxx"` | 从 Keychain 或服务端获取 |
| 最小权限 | 只请求必要的权限 | 请求全部相册权限 | 只请求选中的照片 |
| 纵深防御 | 多层安全保护 | 只靠 HTTPS | HTTPS + Pinning + 加密存储 |
| 安全默认 | 默认配置应是最安全的 | `NSAllowsArbitraryLoads = true` | 逐域名豁免 |
| 数据最小化 | 只收集必要的数据 | 收集完整通讯录 | 只获取选中的联系人 |

### 9.2 常见安全漏洞与修复

#### 漏洞 1：硬编码敏感信息

```swift
// ❌ 危险：密钥直接写在代码中
let apiKey = "sk-proj-abc123def456"
let secret = "my-super-secret-key"

// ✅ 安全：从 Keychain 读取
let apiKey = try KeychainHelper.load(String.self, for: "apiKey")
```

#### 漏洞 2：日志泄露敏感数据

```swift
// ❌ 危险：打印 Token
print("User token: \(token)")

// ✅ 安全：Debug 模式下脱敏
#if DEBUG
print("User token: \(token.prefix(6))...")
#endif
```

#### 漏洞 3：使用不安全的随机数

```swift
// ❌ 危险：可预测的随机数
let token = arc4random()

// ✅ 安全：密码学安全的随机数
let token = SystemRandomNumberGenerator().next()
// 或
let secureBytes = (0..<32).map { _ in UInt8.random(in: 0...255) }
```

### 9.3 混淆敏感字符串

即使不硬编码，字符串常量仍可能被反编译。可以使用简单的混淆：

```swift
enum StringObfuscator {

    static func obfuscate(_ string: String, with key: UInt8) -> [UInt8] {
        string.utf8.map { $0 ^ key }
    }

    static func deobfuscate(_ bytes: [UInt8], with key: UInt8) -> String {
        String(bytes: bytes.map { $0 ^ key }, encoding: .utf8)!
    }
}

// 混淆后存储
let obfuscated = StringObfuscator.obfuscate("my-secret", with: 0x5A)

// 运行时还原
let original = StringObfuscator.deobfuscate(obfuscated, with: 0x5A)
```

### 9.4 越狱检测

越狱设备上 Keychain 保护可能被绕过，对高安全场景应进行检测：

```swift
enum JailbreakDetector {

    static var isJailbroken: Bool {
        checkPaths() || checkCanWriteSystemDir()
    }

    private static func checkPaths() -> Bool {
        let jailbreakPaths = [
            "/Applications/Cydia.app",
            "/Library/MobileSubstrate/MobileSubstrate.dylib",
            "/bin/bash",
            "/usr/sbin/sshd",
            "/etc/apt"
        ]
        return jailbreakPaths.contains { FileManager.default.fileExists(atPath: $0) }
    }

    private static func checkCanWriteSystemDir() -> Bool {
        let testPath = "/private/jailbreak_test"
        defer { try? FileManager.default.removeItem(atPath: testPath) }
        return (try? "test".write(toFile: testPath, atomically: true, encoding: .utf8)) != nil
    }
}
```

> ⚠️ **注意**：越狱检测不是万能的，高级越狱可以绕过检测。对于金融类 App，建议结合服务端风控策略。

---

## 10. 隐私清单中的数据安全声明

### 10.1 与数据安全相关的声明项

在 `PrivacyInfo.xcprivacy` 中，以下声明与数据安全直接相关：

| 声明类型 | 键名 | 说明 |
|---------|------|------|
| 追踪声明 | `NSPrivacyTracking` | App 是否追踪用户 |
| 追踪域名 | `NSPrivacyTrackingDomains` | 追踪使用的域名 |
| 收集的数据 | `NSPrivacyCollectedDataTypes` | 收集了哪些用户数据 |
| 使用的 API | `NSPrivacyAccessedAPITypes` | 使用了哪些隐私相关 API |

### 10.2 数据安全相关 API 声明

如果你的 App 使用了 Keychain 或文件存储，需要在隐私清单中声明：

```xml
<key>NSPrivacyAccessedAPITypes</key>
<array>
    <dict>
        <key>NSPrivacyAccessedAPIType</key>
        <string>NSPrivacyAccessedAPICategoryFileTimestamp</string>
        <key>NSPrivacyAccessedAPITypeReasons</key>
        <array>
            <string>C617.1</string>
        </array>
    </dict>
    <dict>
        <key>NSPrivacyAccessedAPIType</key>
        <string>NSPrivacyAccessedAPICategoryUserDefaults</string>
        <key>NSPrivacyAccessedAPITypeReasons</key>
        <array>
            <string>CA92.1</string>
        </array>
    </dict>
</array>
```

### 10.3 收集数据类型声明

当你的 App 收集了敏感数据，需要声明收集目的：

```xml
<key>NSPrivacyCollectedDataTypes</key>
<array>
    <dict>
        <key>NSPrivacyCollectedDataType</key>
        <string>NSPrivacyCollectedDataTypePassword</string>
        <key>NSPrivacyCollectedDataTypeLinked</key>
        <true/>
        <key>NSPrivacyCollectedDataTypeTracking</key>
        <false/>
        <key>NSPrivacyCollectedDataTypePurposes</key>
        <array>
            <string>NSPrivacyCollectedDataTypePurposeAppFunctionality</string>
        </array>
    </dict>
</array>
```

> 💡 **原则**：只声明你实际收集和使用的数据，不要过度声明。如果 Keychain 中只存储了本地 Token，不需要声明为"收集密码"——因为数据没有离开设备。

---

## 本章小结

| 主题 | 核心要点 |
|------|---------|
| 数据安全概述 | 数据分四级：公开/内部/敏感/机密，不同等级选不同存储方式 |
| Keychain Services | iOS 加密数据库，适合存储密码、Token 等敏感数据 |
| Keychain 基本操作 | SecItemAdd / SecItemDelete / SecItemUpdate / SecItemCopyMatching |
| KeychainHelper 封装 | 泛型 + Codable 支持，自动处理"存在则更新"逻辑 |
| 访问控制 | kSecAttrAccessible 控制访问时机，Biometry 和 Secure Enclave 提供硬件级保护 |
| Keychain Sharing | 通过 Access Group 实现同开发者 App 间凭据共享 |
| 数据加密 | CommonCrypto（AES-CBC）兼容旧系统，CryptoKit（AES-GCM）推荐新项目 |
| 网络安全 | ATS 强制 HTTPS，Certificate Pinning 防中间人攻击 |
| 安全最佳实践 | 不硬编码密钥、不日志泄露、用安全随机数、越狱检测 |
| 隐私清单声明 | 声明使用的隐私 API 和收集的数据类型，确保审核合规 |

> 💡 **一句话总结**：安全不是单一技术，而是从数据分类、存储加密、网络传输到代码习惯的**全链路防护**。Keychain 是 iOS 安全体系的基石，掌握它就掌握了数据安全的核心。

← [-后台任务与多任务](./62-后台任务与多任务.md) | [-iPad 适配与多窗口](./64-iPad适配与多窗口.md) →
