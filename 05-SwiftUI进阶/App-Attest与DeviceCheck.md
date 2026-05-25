# App Attest 与 DeviceCheck

> 🎯 **本章目标**：理解 Apple 反欺诈与设备验证服务，掌握 DeviceCheck 设备唯一标识方案，学会 App Attest 完整性验证流程，了解越狱检测与防篡改策略。

---

## 为什么需要设备验证与完整性保护

### 常见安全威胁

移动应用面临的安全威胁远比想象中严重，尤其在国内市场环境下，以下问题尤为突出：

**刷单与薅羊毛**：攻击者使用自动化脚本或模拟器批量注册账号，利用新用户奖励、签到积分等运营活动获取不当利益。这类攻击不仅造成直接经济损失，还会扭曲运营数据，影响决策判断。

**篡改 App**：通过逆向工程修改 App 二进制文件，去除广告、解锁付费功能或注入恶意代码。重打包后的 App 通过第三方渠道分发，用户难以辨别真伪。

**重放攻击**：截获合法请求并重复发送，例如重复提交积分兑换请求或重复使用一次性优惠码，绕过服务端的业务逻辑校验。

**设备伪造**：伪造设备标识符，绕过基于设备的限制策略，例如突破每日领取次数限制或绕过设备绑定验证。

### Apple 提供的解决方案

Apple 针对上述威胁提供了两层防护体系：

- **DeviceCheck**：轻量级设备验证方案，通过硬件绑定的设备 Token 实现设备唯一标识，并提供两位比特位的持久化存储，适合简单的防刷场景
- **App Attest**：完整的 App 完整性验证方案，基于硬件密钥和 Apple 服务器证明，确保请求来自未篡改的正版 App，适合高安全等级场景

### 适用场景分析

| 场景 | 推荐方案 | 原因 |
|------|----------|------|
| 新用户注册防刷 | DeviceCheck | 只需识别设备是否已注册，实现简单 |
| 每日签到防重复 | DeviceCheck | 利用比特位记录签到状态，无需自建存储 |
| 积分兑换验证 | App Attest | 涉及资产转移，需要确保请求来源可信 |
| 付费功能校验 | App Attest | 防止篡改 App 绕过付费验证 |
| API 请求防篡改 | App Attest | 需要验证请求完整性和 App 身份 |
| 内容版权保护 | App Attest | 防止被篡改的 App 提取受保护内容 |
| 投票活动防刷 | DeviceCheck + App Attest | 双重验证，DeviceCheck 限频，App Attest 防伪造 |
| 临时活动防刷 | DeviceCheck | 快速接入，两位比特位足够标记参与状态 |

---

## DeviceCheck 详解

### DeviceCheck 是什么

DeviceCheck 是 Apple 在 iOS 11 中引入的轻量级设备验证框架。它的核心能力有两个：

1. **设备唯一标识**：生成与设备硬件绑定的 Token，同一设备同一开发者账号下的所有 App 获得相同 Token，不同开发者账号则不同
2. **两位比特位存储**：每个设备提供两个比特位（共四种状态组合），由开发者服务端写入，Apple 服务器持久化保存，即使 App 卸载重装也不会丢失

DeviceCheck 的设计哲学是"最小化信息泄露"——它不提供设备序列号、广告标识符等可追踪信息，而是只提供一个不可逆的 Token 和极有限的存储空间。

### DCDevice API 使用

DeviceCheck 的客户端 API 非常简洁，核心只有一个类 `DCDevice`：

```swift
import DeviceCheck

func generateDeviceToken() {
    let device = DCDevice.current

    guard device.isSupported else {
        print("DeviceCheck 不支持此设备")
        return
    }

    device.generateToken { token, error in
        if let error = error {
            print("Token 生成失败: \(error.localizedDescription)")
            return
        }

        guard let token = token else { return }

        let tokenString = token.base64EncodedString()

        sendTokenToServer(tokenString)
    }
}

func sendTokenToServer(_ token: String) {
    guard let url = URL(string: "https://api.example.com/devicecheck/verify") else { return }

    var request = URLRequest(url: url)
    request.httpMethod = "POST"
    request.setValue("application/json", forHTTPHeaderField: "Content-Type")

    let body: [String: String] = ["device_token": token]
    request.httpBody = try? JSONSerialization.data(withJSONObject: body)

    URLSession.shared.dataTask(with: request) { data, response, error in
        if let error = error {
            print("请求失败: \(error.localizedDescription)")
            return
        }

        if let httpResponse = response as? HTTPURLResponse,
           httpResponse.statusCode == 200 {
            print("设备验证成功")
        }
    }.resume()
}
```

### 两位比特位的用途

两位比特位是 DeviceCheck 最独特的设计，虽然只有 2 bit 的存储空间，但在防刷场景中非常实用：

| 比特位组合 | 二进制 | 典型用途 |
|-----------|--------|----------|
| 0, 0 | 00 | 设备首次出现，未参与任何活动 |
| 0, 1 | 01 | 已参与活动 A（如新用户注册） |
| 1, 0 | 10 | 已参与活动 B（如每日签到） |
| 1, 1 | 11 | 同时参与活动 A 和 B |

> 💡 **提示**：比特位的读写完全由服务端控制，客户端无法读取或修改。即使 App 被卸载重装，比特位状态也不会重置，这是 DeviceCheck 防刷的核心优势。

### 服务端验证流程

服务端验证是 DeviceCheck 的关键环节，需要与 Apple 服务器交互：

1. 接收客户端发送的设备 Token
2. 使用开发者私钥生成 JWT（JSON Web Token）
3. 向 Apple 的 DeviceCheck API 发送查询或更新请求
4. Apple 服务器验证 JWT 签名后返回设备信息或更新比特位

**查询设备状态**：

```python
import jwt
import time
import requests

TEAM_ID = "YOUR_TEAM_ID"
KEY_ID = "YOUR_KEY_ID"
PRIVATE_KEY = """-----BEGIN PRIVATE KEY-----
YOUR_P8_KEY_CONTENT
-----END PRIVATE KEY-----"""

def create_jwt():
    now = int(time.time())
    payload = {
        "iss": TEAM_ID,
        "iat": now,
        "exp": now + 3600
    }
    headers = {
        "alg": "ES256",
        "kid": KEY_ID
    }
    return jwt.encode(payload, PRIVATE_KEY, algorithm="ES256", headers=headers)

def query_device(token_string):
    jwt_token = create_jwt()
    url = "https://api.development.devicecheck.apple.com/v1/query_two_bits"

    headers = {
        "Authorization": f"Bearer {jwt_token}",
        "Content-Type": "application/json"
    }

    payload = {
        "device_token": token_string,
        "transaction_id": str(uuid.uuid4()),
        "timestamp": int(time.time() * 1000)
    }

    response = requests.post(url, json=payload, headers=headers)

    if response.status_code == 200:
        data = response.json()
        return {
            "bit0": data.get("bit0", False),
            "bit1": data.get("bit1", False),
            "last_update_time": data.get("last_update_time", "")
        }
    elif response.status_code == 404:
        return {"bit0": False, "bit1": False, "last_update_time": ""}
    else:
        raise Exception(f"查询失败: {response.status_code}")

def update_device(token_string, bit0: bool, bit1: bool):
    jwt_token = create_jwt()
    url = "https://api.development.devicecheck.apple.com/v1/update_two_bits"

    headers = {
        "Authorization": f"Bearer {jwt_token}",
        "Content-Type": "application/json"
    }

    payload = {
        "device_token": token_string,
        "transaction_id": str(uuid.uuid4()),
        "timestamp": int(time.time() * 1000),
        "bit0": bit0,
        "bit1": bit1
    }

    response = requests.post(url, json=payload, headers=headers)

    if response.status_code == 200:
        return True
    else:
        raise Exception(f"更新失败: {response.status_code}")
```

> ⚠️ **警告**：开发环境使用 `api.development.devicecheck.apple.com`，生产环境使用 `api.devicecheck.apple.com`。上线前务必切换 API 地址。每个 Token 每天有查询次数限制，服务端应做好缓存。

---

## App Attest 详解

### App Attest 是什么

App Attest 是 Apple 在 iOS 14 中引入的 App 完整性验证服务。它基于设备安全隔区（Secure Enclave）中的硬件密钥，通过 Apple 服务器的证明链确保：

1. 请求来自正版 App（未被篡改）
2. 请求来自合法的 Apple 设备（非模拟器）
3. 请求由 App 自身发出（非第三方伪造）

与 DeviceCheck 的"识别设备"不同，App Attest 的核心目标是"验证 App 完整性"。

### 与 DeviceCheck 的区别

| 特性 | DeviceCheck | App Attest |
|------|-------------|------------|
| 最低系统版本 | iOS 11 | iOS 14 |
| 核心功能 | 设备标识 + 比特位存储 | App 完整性验证 |
| 安全等级 | 中等 | 高 |
| 硬件依赖 | 无特殊要求 | 需要 Secure Enclave |
| 密钥管理 | 无 | 自动生成与管理 |
| 服务端验证 | Apple DeviceCheck API | Apple App Attest API |
| 数据持久化 | 两位比特位 | 无持久化 |
| 防篡改能力 | 无 | 强 |
| 防重放攻击 | 无 | 内置支持 |
| 实现复杂度 | 低 | 中高 |
| 适用场景 | 防刷、限频 | 高价值操作验证 |
| 模拟器支持 | 部分支持 | 不支持 |

### 完整验证流程

App Attest 的验证流程分为三个阶段：

**阶段一：生成密钥对**

App 向安全隔区请求生成一对非对称密钥，私钥永远不离开安全隔区，公钥由 App 发送给服务端存储。

**阶段二：证明密钥（Attestation）**

App 请求 Apple 服务器对密钥进行证明，Apple 会验证 App 的签名证书和设备完整性，返回一个包含证明链的 Attestation 对象。服务端验证此对象即可确认密钥确实由正版 App 在合法设备上生成。

**阶段三：验证断言（Assertion）**

每次需要验证请求时，App 使用私钥对请求数据签名生成 Assertion，服务端验证签名即可确认请求未被篡改且来自持有私钥的 App。

### DCAppAttestService API 使用

```swift
import DeviceCheck

class AppAttestManager {
    private let service = DCAppAttestService.shared
    private var keyId: String?

    func isSupported() -> Bool {
        return DCAppAttestService.shared.isSupported
    }

    func setupAttestation() async throws {
        guard service.isSupported else {
            throw AttestationError.notSupported
        }

        let keyId = try await service.generateKey()
        self.keyId = keyId

        let clientDataHash = Data("attestation-\(keyId)".utf8).sha256()

        let attestationObject = try await service.attestKey(keyId, clientDataHash: clientDataHash)

        try await sendAttestationToServer(keyId: keyId, attestationObject: attestationObject.base64EncodedString(), clientDataHash: clientDataHash.base64EncodedString())
    }

    func generateAssertion(for request: URLRequest) async throws -> String {
        guard let keyId = keyId else {
            throw AttestationError.keyNotReady
        }

        let requestHash = computeRequestHash(request)

        let assertion = try await service.generateAssertion(keyId, clientDataHash: requestHash)

        return assertion.base64EncodedString()
    }

    private func computeRequestHash(_ request: URLRequest) -> Data {
        var components: [String] = []

        if let method = request.httpMethod {
            components.append(method)
        }

        if let url = request.url?.absoluteString {
            components.append(url)
        }

        if let body = request.httpBody {
            components.append(body.base64EncodedString())
        }

        let combined = components.joined(separator: "|")
        return Data(combined.utf8).sha256()
    }

    private func sendAttestationToServer(keyId: String, attestationObject: String, clientDataHash: String) async throws {
        guard let url = URL(string: "https://api.example.com/appattest/attest") else { return }

        var request = URLRequest(url: url)
        request.httpMethod = "POST"
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")

        let body: [String: String] = [
            "key_id": keyId,
            "attestation_object": attestationObject,
            "client_data_hash": clientDataHash
        ]
        request.httpBody = try JSONSerialization.data(withJSONObject: body)

        let (_, response) = try await URLSession.shared.data(for: request)

        guard let httpResponse = response as? HTTPURLResponse,
              httpResponse.statusCode == 200 else {
            throw AttestationError.attestationFailed
        }
    }
}

enum AttestationError: Error {
    case notSupported
    case keyNotReady
    case attestationFailed
    case assertionFailed
}
```

> 💡 **提示**：`generateKey`、`attestKey`、`generateAssertion` 都是异步方法，建议使用 Swift Concurrency（async/await）来管理调用流程。密钥 ID 应持久化存储在 Keychain 中，避免 App 重启后丢失。

### 服务端验证对接

服务端验证 App Attest 需要完成两个关键步骤：验证 Attestation 和验证 Assertion。

**验证 Attestation**：

```python
import base64
import hashlib
import json
import requests
from cryptography.x509 import load_der_x509_certificate

APPLE_ATTESTATION_URL = "https://appattest.apple.com/v1/attestation"

def verify_attestation(key_id, attestation_object_b64, client_data_hash_b64, bundle_id, team_id):
    attestation_data = base64.b64decode(attestation_object_b64)
    client_data_hash = base64.b64decode(client_data_hash_b64)

    apple_response = requests.post(
        APPLE_ATTESTATION_URL,
        headers={"Content-Type": "application/json"},
        json={
            "attestation": attestation_object_b64,
            "key_id": key_id,
            "client_data_hash": client_data_hash_b64
        }
    )

    if apple_response.status_code != 200:
        raise Exception("Apple Attestation 验证失败")

    cert_data = extract_certificate_from_attestation(attestation_data)
    cert = load_der_x509_certificate(cert_data)

    credential_id = extract_credential_id(attestation_data)
    public_key = cert.public_key()

    store_key_in_database(key_id, credential_id, public_key, bundle_id, team_id)

    return True

def verify_assertion(key_id, assertion_b64, request_hash, bundle_id, team_id):
    assertion_data = base64.b64decode(assertion_b64)

    stored_key = get_key_from_database(key_id)
    if stored_key is None:
        raise Exception("未找到已证明的密钥")

    public_key = stored_key["public_key"]

    authenticator_data = assertion_data[:37]
    signature = assertion_data[37:]

    verify_signature(public_key, authenticator_data, request_hash, signature)

    rp_id_hash = authenticator_data[:32]
    expected_rp_id = hashlib.sha256(bundle_id.encode()).digest()
    if rp_id_hash != expected_rp_id:
        raise Exception("RP ID 不匹配")

    return True
```

> ⚠️ **警告**：App Attest 的服务端验证涉及 CBOR 解码、X.509 证书验证、ECDSA 签名验证等复杂操作。建议使用 Apple 官方提供的验证库或社区成熟方案，不要自行实现加密验证逻辑。生产环境必须验证证书链的根证书是否为 Apple 的 CA 证书。

---

## 越狱检测实践

### 越狱对 App 安全的影响

越狱设备对 App 安全构成严重威胁，主要体现在：

1. **沙盒突破**：越狱后 App 沙盒限制失效，任何进程都可以读取其他 App 的数据文件，包括 Keychain、UserDefaults 和数据库
2. **代码注入**：通过 Cydia Substrate 或 Substitute 框架，攻击者可以在运行时修改 App 的方法实现，绕过验证逻辑
3. **重打包**：解密 App 二进制文件后修改代码并重新签名分发，用户可能安装了被篡改的版本
4. **调试附加**：越狱设备上可以轻松附加调试器，实时分析 App 运行时行为
5. **证书绕过**：越狱后可以安装自签名证书，实施中间人攻击截获 HTTPS 通信

### 常见越狱检测方法

越狱检测通常从以下几个维度进行：

**文件系统检测**：检查越狱工具的典型文件路径是否存在

**URL Scheme 检测**：尝试打开越狱商店（Cydia）的 URL Scheme

**沙盒完整性检测**：尝试写入沙盒外的路径

**动态库检测**：检查是否加载了异常的动态库

### 检测代码示例

```swift
import Foundation
import UIKit
import MachO

class JailbreakDetector {

    static func isJailbroken() -> Bool {
        return checkFileExistence()
            || checkCydiaURLScheme()
            || checkSandboxIntegrity()
            || checkSuspiciousDylibs()
            || checkForkAbility()
    }

    private static func checkFileExistence() -> Bool {
        let suspiciousPaths = [
            "/Applications/Cydia.app",
            "/Library/MobileSubstrate/MobileSubstrate.dylib",
            "/bin/bash",
            "/usr/sbin/sshd",
            "/etc/apt",
            "/private/var/lib/apt",
            "/usr/bin/ssh",
            "/var/cache/apt",
            "/var/lib/cydia",
            "/Applications/Sileo.app",
            "/Applications/Zebra.app",
            "/usr/lib/libsubstitute.dylib",
            "/usr/lib/substrate",
            "/bootstrap/Sileo.app"
        ]

        for path in suspiciousPaths {
            if FileManager.default.fileExists(atPath: path) {
                return true
            }
        }

        if canOpen(path: "/Applications/Cydia.app") {
            return true
        }

        if canOpen(path: "/private/jailbreak.txt") {
            return true
        }

        return false
    }

    private static func canOpen(path: String) -> Bool {
        let fd = open(path, O_RDONLY)
        if fd >= 0 {
            close(fd)
            return true
        }
        return false
    }

    private static func checkCydiaURLScheme() -> Bool {
        guard let url = URL(string: "cydia://package/com.example.package") else {
            return false
        }
        return UIApplication.shared.canOpenURL(url)
    }

    private static func checkSandboxIntegrity() -> Bool {
        let testPath = "/private/jailbreak_test_\(Int.random(in: 1000...9999))"
        do {
            try "test".write(toFile: testPath, atomically: true, encoding: .utf8)
            try? FileManager.default.removeItem(atPath: testPath)
            return true
        } catch {
            return false
        }
    }

    private static func checkSuspiciousDylibs() -> Bool {
        let suspiciousDylibs = [
            "MobileSubstrate",
            "Substitute",
            "CydiaSubstrate",
            "libhooker",
            "SubstrateLoader"
        ]

        for index in 0..<_dyld_image_count() {
            let imageName = String(cString: _dyld_get_image_name(index))
            for dylib in suspiciousDylibs {
                if imageName.contains(dylib) {
                    return true
                }
            }
        }
        return false
    }

    private static func checkForkAbility() -> Bool {
        let pid = fork()
        if pid == 0 {
            exit(0)
        }
        return pid >= 0
    }
}
```

### 越狱检测的局限性与对抗

越狱检测并非万无一失，攻击者可以通过多种方式绕过：

**越狱隐藏工具**：如 Liberty Lite、Shadow、FlyJB 等越狱插件可以逐个 Hook 检测函数，使其返回"未越狱"的结果

**运行时 Hook**：使用 Frida 或 Cycript 在运行时替换检测方法的实现

**二进制补丁**：直接修改 App 二进制文件中的检测逻辑

因此越狱检测应遵循以下原则：

1. **多层检测**：不要依赖单一检测手段，组合使用多种方法
2. **分散检测时机**：不要在 App 启动时集中检测，分散在不同业务流程中
3. **服务端协同**：将检测结果上报服务端，由服务端决定是否限制功能
4. **混淆检测代码**：增加逆向分析难度

### Apple 对越狱检测的态度

Apple 官方并未提供越狱检测 API，也未明确鼓励或禁止越狱检测。App Store Review Guidelines 中没有专门针对越狱检测的条款，但以下情况可能导致审核被拒：

- 检测到越狱后直接崩溃或退出（被视为"破坏用户体验"）
- 在 App 描述中声称"防越狱"或"越狱检测"（可能被视为不当宣传）

> 💡 **提示**：推荐的做法是检测到越狱后"降级功能"而非"拒绝运行"。例如，越狱设备上禁用支付功能但仍允许浏览内容，这样既保护了安全又不会影响审核。

---

## 防篡改与安全加固

### 代码混淆策略

代码混淆通过增加逆向工程难度来保护 App 逻辑：

**命名混淆**：将关键类名、方法名替换为无意义字符串

```swift
protocol SecurityProtocol {
    func validate() -> Bool
    func process(_ data: Data) -> Data
}

class X8K2M: SecurityProtocol {
    private let k = "a3f2b1c4"

    func validate() -> Bool {
        return performInternalCheck()
    }

    func process(_ data: Data) -> Data {
        return transform(data)
    }

    private func performInternalCheck() -> Bool {
        return true
    }

    private func transform(_ input: Data) -> Data {
        var result = input
        for i in 0..<result.count {
            result[i] ^= UInt8(k.hashValue & 0xFF)
        }
        return result
    }
}
```

**控制流扁平化**：将线性的代码逻辑转换为 switch-case 状态机，增加静态分析难度

**垃圾代码注入**：在关键逻辑中插入不会执行的冗余代码

### 字符串加密

硬编码的字符串是逆向分析的重要线索，攻击者可以通过字符串搜索快速定位关键逻辑：

```swift
import CryptoKit
import Foundation

class StringProtection {
    static func decrypt(_ encrypted: [UInt8], key: UInt8) -> String {
        let decrypted = encrypted.map { $0 ^ key }
        return String(bytes: decrypted, encoding: .utf8) ?? ""
    }

    static func obfuscate(_ string: String, salt: String) -> String {
        let bytes = Array(string.utf8)
        let saltBytes = Array(salt.utf8)
        var result = [UInt8]()

        for (index, byte) in bytes.enumerated() {
            let saltByte = saltBytes[index % saltBytes.count]
            result.append(byte ^ saltByte)
        }

        return Data(result).base64EncodedString()
    }
}

let apiKey = StringProtection.decrypt(
    [0x4a, 0x5b, 0x4c, 0x5d, 0x4e, 0x3f, 0x50, 0x41],
    key: 0x2a
)
```

> 💡 **提示**：Xcode 自带的 Swift 编译器会在编译时对字符串常量进行一定程度的优化，但不会加密。建议使用构建脚本在编译前对敏感字符串进行预处理加密，运行时再解密使用。

### class-dump 防护

class-dump 是 Objective-C 逆向的经典工具，它可以导出 App 中所有 Objective-C 类的接口定义。防护措施包括：

1. **减少 Objective-C 暴露面**：关键逻辑使用 Swift 编写，Swift 类默认不暴露 Objective-C 运行时信息
2. **移除不必要的 @objc 标注**：只在必须与 Objective-C 交互时才使用
3. **使用 final 和 private**：阻止方法被重写和外部访问

```swift
final class SecurePaymentHandler {
    private let encryptionKey: Data

    private init() {
        self.encryptionKey = Self.generateKey()
    }

    private static func generateKey() -> Data {
        let key = SymmetricKey(size: .bits256)
        return key.withUnsafeBytes { Data($0) }
    }

    private func processPayment(_ amount: Decimal) -> Bool {
        return true
    }

    private func encryptPayload(_ payload: Data) -> Data? {
        let key = SymmetricKey(data: encryptionKey)
        let sealed = try? AES.GCM.seal(payload, using: key)
        return sealed?.combined
    }
}
```

### 调试器检测

检测调试器附加是防止运行时分析的重要手段：

```swift
import Foundation

class AntiDebugProtection {
    static func isDebuggerAttached() -> Bool {
        var info = kinfo_proc()
        var size = MemoryLayout<kinfo_proc>.size
        var mib: [Int32] = [CTL_KERN, KERN_PROC, KERN_PROC_PID, getpid()]

        let result = sysctl(&mib, 4, &info, &size, nil, 0)
        guard result == 0 else { return false }

        return (info.kp_proc.p_flag & P_TRACED) != 0
    }

    static func enableAntiDebug() {
        let ptr = dlsym(dlopen(nil, RTLD_GLOBAL), "ptrace")
        typealias PtraceType = @convention(c) (Int32, pid_t, UnsafeMutableRawPointer?, UnsafeMutableRawPointer?) -> Int32
        let ptrace = unsafeBitCast(ptr, to: PtraceType.self)
        ptrace(PT_DENY_ATTACH, 0, nil, nil)
    }

    static func startDebugMonitoring() {
        Timer.scheduledTimer(withTimeInterval: 1.0, repeats: true) { _ in
            if isDebuggerAttached() {
                exit(0)
            }
        }
    }
}
```

> ⚠️ **警告**：`PT_DENY_ATTACH` 在 App Store 审核中可能被视为"私有 API 使用"而被拒。生产环境建议仅使用 `sysctl` 方式检测，并将结果上报服务端而非直接退出。直接调用 `exit` 也可能导致审核被拒。

### 安全加固 Checklist

| 检查项 | 优先级 | 实施方式 |
|--------|--------|----------|
| 敏感数据使用 Keychain 存储 | 高 | 替代 UserDefaults 和文件存储 |
| 网络通信强制 HTTPS | 高 | 配置 ATS 策略 |
| 证书绑定（Certificate Pinning） | 高 | URLSessionDelegate 实现 |
| App Attest 验证高价值操作 | 高 | DCAppAttestService |
| DeviceCheck 防刷限频 | 中 | DCDevice + 服务端比特位 |
| 越狱检测与功能降级 | 中 | 多层检测 + 服务端协同 |
| 字符串加密 | 中 | 构建脚本预处理 |
| 代码混淆 | 中 | 命名混淆 + 控制流变换 |
| 调试器检测 | 中 | sysctl 检测 |
| class-dump 防护 | 低 | Swift 化 + 减少运行时暴露 |
| 截屏防护 | 低 | UITextField.isSecureTextEntry |
| 粘贴板防护 | 低 | 禁用敏感内容复制 |

---

## 小结

本章介绍了 Apple 提供的设备验证与完整性保护方案，以及越狱检测和安全加固的实践方法：

| 知识点 | 核心内容 | 关键 API / 技术 |
|--------|----------|----------------|
| 安全威胁 | 刷单、薅羊毛、篡改、重放攻击 | — |
| DeviceCheck | 设备唯一标识 + 两位比特位 | DCDevice |
| DeviceCheck 服务端 | JWT 认证 + Apple API 查询/更新 | ES256 签名 |
| App Attest | App 完整性验证 | DCAppAttestService |
| Attestation 流程 | 生成密钥 → 证明密钥 → 服务端验证 | generateKey / attestKey |
| Assertion 流程 | 私钥签名 → 服务端验证签名 | generateAssertion |
| 越狱检测 | 文件检测 / URL Scheme / 沙盒 / 动态库 | FileManager / sysctl / dyld |
| 越狱检测局限 | 可被 Hook 绕过，需多层防御 | — |
| 代码混淆 | 命名混淆、控制流扁平化 | Swift final / private |
| 字符串加密 | 运行时解密敏感字符串 | XOR / AES |
| class-dump 防护 | 减少 ObjC 运行时暴露 | Swift 化 / 移除 @objc |
| 调试器检测 | sysctl 检测 + PT_DENY_ATTACH | sysctl / ptrace |
| 安全加固 | Checklist 驱动的系统化加固 | Keychain / ATS / Pinning |

DeviceCheck 适合轻量级防刷场景，App Attest 适合高安全等级的完整性验证。两者可以组合使用，形成"设备识别 + App 验证"的双重防护体系。越狱检测和安全加固则是纵深防御的重要补充，但需要注意 Apple 审核政策的限制，采用"功能降级"而非"拒绝运行"的策略。

← [Keychain 与数据安全](./Keychain与数据安全.md) | [iPad 适配与多窗口](./iPad适配与多窗口.md) →