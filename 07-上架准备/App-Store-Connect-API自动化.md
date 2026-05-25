# App Store Connect API自动化

> 🎯 **本章目标**：学会使用 App Store Connect API 实现元数据管理、构建版本管理的自动化，结合 CI/CD 流程提升发布效率。

---

## 1. App Store Connect API 概述

### 1.1 什么是 App Store Connect API

App Store Connect API 是 Apple 提供的 RESTful API，允许开发者通过编程方式管理 App Store Connect 中的各种资源，包括 App 元数据、构建版本、TestFlight 测试、审核提交等。

> 💡 生活类比：App Store Connect API 就像银行的"企业网银"——
> - 不用亲自跑银行（登录网页），在线就能操作
> - 可以批量处理业务（自动化脚本）
> - 有严格的身份验证（JWT 令牌）
> - 操作有权限控制（角色和权限）
> - 每笔操作都有记录（审计日志）

### 1.2 API 的能力范围

| 功能域 | API 能力 | 说明 |
|--------|----------|------|
| App 管理 | 查询/修改 App 信息 | 名称、Bundle ID、价格等 |
| 构建版本 | 查询/修改构建 | 上传、分发、过期处理 |
| TestFlight | 管理 Beta 测试 | 测试组、测试员、构建分发 |
| 审核 | 提交/管理审核 | 提交审核、回复审核信息 |
| 元数据 | 管理本地化信息 | 描述、关键词、截图 |
| 用户 | 管理团队用户 | 邀请、角色、权限 |
| 报告 | 下载销售和下载报告 | 销售、下载、订阅数据 |
| In-App Events | 管理活动 | 创建、修改、提交活动 |
| 自定义产品页 | 管理产品页变体 | 创建、修改自定义页 |
| 订阅 | 管理订阅状态 | 退款、续期、状态变更 |

### 1.3 API 版本演进

| 版本 | 发布时间 | 主要变更 |
|------|----------|----------|
| v1 | 2018 | 基础 API，用户和角色管理 |
| v2 | 2021 | 大幅扩展，元数据、构建、审核 |
| v2.1+ | 2022 | In-App Events、自定义产品页 |
| v2.3+ | 2023 | 订阅管理、报告增强 |
| v3 | 2024+ | 更多自动化能力 |

💡 **提示**：Apple 持续扩展 API 能力，建议关注 [官方文档](https://developer.apple.com/documentation/appstoreconnectapi) 的更新日志。

### 1.4 为什么需要 API 自动化

| 手动操作 | 自动化 |
|----------|--------|
| 登录网页 → 手动修改元数据 | 脚本自动更新 |
| 手动上传截图 → 逐个语言 | 批量上传所有语言 |
| 手动提交审核 → 填写表单 | CI/CD 自动提交 |
| 手动管理 TestFlight | 自动分发新构建 |
| 手动下载报告 | 定时自动下载和处理 |

---

## 2. API Key 创建和权限配置

### 2.1 生成 API Key

**步骤：**

1. 登录 [App Store Connect](https://appstoreconnect.apple.com)
2. 进入「用户和访问」→「集成」→「App Store Connect API」
3. 点击「生成 API 密钥」
4. 输入名称（如 "CI/CD Pipeline"）
5. 选择访问权限（角色）
6. 点击「生成」
7. **立即下载 .p8 文件**（只能下载一次！）
8. 记录 Key ID 和 Issuer ID

⚠️ **警告**：.p8 文件只能下载一次！如果丢失，必须撤销旧 Key 重新创建。请将 .p8 文件存储在安全的密钥管理系统中。

### 2.2 Key 信息说明

| 信息 | 说明 | 示例 |
|------|------|------|
| Issuer ID | 发行者 ID，所有 Key 共享 | 69a6de7e-XXXX-XXXX-XXXX-XXXXXXXXXXXX |
| Key ID | 单个 Key 的标识 | 2X9R4HXF34 |
| .p8 文件 | 私钥文件 | AuthKey_2X9R4HXF34.p8 |

### 2.3 API Key 角色和权限

| 角色 | 权限范围 | 适合场景 |
|------|----------|----------|
| Admin | 所有权限 | 不推荐用于 CI/CD |
| App Manager | App 管理、构建、TestFlight | 推荐 CI/CD 使用 |
| Developer | 构建上传、TestFlight | 适合仅上传构建 |
| Marketing | 元数据、截图、定价 | 适合仅更新元数据 |
| Sales | 报告下载 | 适合仅下载报告 |
| Finance | 财务报告 | 适合财务系统 |

**最小权限原则：**

```
场景 1：CI/CD 仅上传构建
    → 选择 Developer 角色

场景 2：CI/CD 上传构建 + 提交审核
    → 选择 App Manager 角色

场景 3：仅更新元数据和截图
    → 选择 Marketing 角色

场景 4：仅下载销售报告
    → 选择 Sales 角色
```

### 2.4 多 Key 策略

建议为不同用途创建不同的 API Key：

| Key 名称 | 角色 | 用途 |
|----------|------|------|
| CI Build Upload | Developer | CI/CD 构建上传 |
| CI Release | App Manager | CI/CD 发布流程 |
| Metadata Update | Marketing | 元数据自动更新 |
| Report Download | Sales | 报告自动下载 |

💡 **提示**：不同 Key 用于不同场景，可以精确控制权限，降低安全风险。

---

## 3. JWT 令牌生成

### 3.1 JWT 认证原理

App Store Connect API 使用 JWT（JSON Web Token）进行认证：

```
JWT 结构：
Header.Payload.Signature

Header:
{
    "alg": "ES256",
    "kid": "2X9R4HXF34",      // Key ID
    "typ": "JWT"
}

Payload:
{
    "iss": "69a6de7e-...",     // Issuer ID
    "iat": 1700000000,         // 签发时间
    "exp": 1700000600,         // 过期时间（最长 20 分钟）
    "aud": "appstoreconnect-v1",
    "scope": [...]             // 可选，限定权限范围
}

Signature:
ES256(Header.Base64 + "." + Payload.Base64, PrivateKey)
```

**JWT 关键参数：**

| 参数 | 说明 | 限制 |
|------|------|------|
| alg | 签名算法 | 必须为 ES256 |
| kid | Key ID | 必须与 .p8 文件对应 |
| iss | Issuer ID | 必须正确 |
| iat | 签发时间 | Unix 时间戳 |
| exp | 过期时间 | 最长 20 分钟 |
| aud | 受众 | 必须为 "appstoreconnect-v1" |
| scope | 权限范围 | 可选 |

### 3.2 Python 生成 JWT

```python
import jwt
import time

def generate_token(key_id, issuer_id, private_key_path):
    with open(private_key_path, 'r') as f:
        private_key = f.read()

    now = int(time.time())
    payload = {
        'iss': issuer_id,
        'iat': now,
        'exp': now + 1200,
        'aud': 'appstoreconnect-v1',
    }

    headers = {
        'alg': 'ES256',
        'kid': key_id,
        'typ': 'JWT'
    }

    token = jwt.encode(
        payload,
        private_key,
        algorithm='ES256',
        headers=headers
    )

    return token

if __name__ == '__main__':
    KEY_ID = '2X9R4HXF34'
    ISSUER_ID = '69a6de7e-XXXX-XXXX-XXXX-XXXXXXXXXXXX'
    PRIVATE_KEY_PATH = './AuthKey_2X9R4HXF34.p8'

    token = generate_token(KEY_ID, ISSUER_ID, PRIVATE_KEY_PATH)
    print(token)
```

**安装依赖：**

```bash
pip install PyJWT
```

### 3.3 Shell 脚本生成 JWT

```bash
#!/bin/bash
# generate_jwt.sh

KEY_ID="2X9R4HXF34"
ISSUER_ID="69a6de7e-XXXX-XXXX-XXXX-XXXXXXXXXXXX"
PRIVATE_KEY_PATH="./AuthKey_2X9R4HXF34.p8"

# 获取当前时间
IAT=$(date +%s)
EXP=$((IAT + 1200))

# 生成 Header
HEADER=$(echo -n '{"alg":"ES256","kid":"'"$KEY_ID"'","typ":"JWT"}' | base64 | tr '+/' '-_' | tr -d '=')

# 生成 Payload
PAYLOAD=$(echo -n '{"iss":"'"$ISSUER_ID"'","iat":'"$IAT"',"exp":'"$EXP"',"aud":"appstoreconnect-v1"}' | base64 | tr '+/' '-_' | tr -d '=')

# 生成 Signature
SIGNATURE=$(echo -n "$HEADER.$PAYLOAD" | openssl dgst -sha256 -sign "$PRIVATE_KEY_PATH" | openssl base64 -e -A | tr '+/' '-_' | tr -d '=' | tr -d '\n')

# 组合 JWT
JWT="$HEADER.$PAYLOAD.$SIGNATURE"

echo "$JWT"
```

### 3.4 Ruby 生成 JWT（Fastlane 使用）

```ruby
require 'jwt'

def generate_token(key_id, issuer_id, private_key_path)
  private_key = OpenSSL::PKey::EC.new(File.read(private_key_path))

  now = Time.now.to_i
  payload = {
    iss: issuer_id,
    iat: now,
    exp: now + 1200,
    aud: 'appstoreconnect-v1'
  }

  headers = {
    kid: key_id
  }

  JWT.encode(payload, private_key, 'ES256', headers)
end

KEY_ID = '2X9R4HXF34'
ISSUER_ID = '69a6de7e-XXXX-XXXX-XXXX-XXXXXXXXXXXX'
PRIVATE_KEY_PATH = './AuthKey_2X9R4HXF34.p8'

token = generate_token(KEY_ID, ISSUER_ID, PRIVATE_KEY_PATH)
puts token
```

### 3.5 Swift 生成 JWT

```swift
import CryptoKit
import Foundation

struct JWTGenerator {
    let keyID: String
    let issuerID: String
    let privateKey: P256.Signing.PrivateKey

    init(keyID: String, issuerID: String, privateKeyP8: String) throws {
        self.keyID = keyID
        self.issuerID = issuerID
        let cleanedKey = privateKeyP8
            .replacingOccurrences(of: "-----BEGIN PRIVATE KEY-----", with: "")
            .replacingOccurrences(of: "-----END PRIVATE KEY-----", with: "")
            .replacingOccurrences(of: "\n", with: "")
        let keyData = Data(base64Encoded: cleanedKey)!
        self.privateKey = try P256.Signing.PrivateKey(rawRepresentation: keyData)
    }

    func generateToken() throws -> String {
        let header = Base64URL.encode(
            try JSONEncoder().encode(JWTHeader(alg: "ES256", kid: keyID, typ: "JWT"))
        )

        let now = Int(Date().timeIntervalSince1970)
        let payload = Base64URL.encode(
            try JSONEncoder().encode(JWTPayload(
                iss: issuerID,
                iat: now,
                exp: now + 1200,
                aud: "appstoreconnect-v1"
            ))
        )

        let signingInput = "\(header).\(payload)"
        let signatureData = Data(signingInput.utf8)
        let signature = try privateKey.signature(for: signatureData)
        let signatureEncoded = Base64URL.encode(signature.rawRepresentation)

        return "\(header).\(payload).\(signatureEncoded)"
    }
}

struct JWTHeader: Encodable {
    let alg: String
    let kid: String
    let typ: String
}

struct JWTPayload: Encodable {
    let iss: String
    let iat: Int
    let exp: Int
    let aud: String
}

enum Base64URL {
    static func encode(_ data: Data) -> String {
        data.base64EncodedString()
            .replacingOccurrences(of: "+", with: "-")
            .replacingOccurrences(of: "/", with: "_")
            .replacingOccurrences(of: "=", with: "")
    }
}
```

### 3.6 JWT 缓存策略

JWT 最长有效期 20 分钟，频繁生成会增加延迟。建议缓存 JWT：

```python
import time

class JWTManager:
    def __init__(self, key_id, issuer_id, private_key_path):
        self.key_id = key_id
        self.issuer_id = issuer_id
        self.private_key_path = private_key_path
        self._token = None
        self._token_expiry = 0

    def get_token(self):
        now = int(time.time())
        if self._token and now < self._token_expiry - 60:
            return self._token

        self._token = generate_token(
            self.key_id,
            self.issuer_id,
            self.private_key_path
        )
        self._token_expiry = now + 1200
        return self._token
```

---

## 4. 核心 API 端点

### 4.1 API 基础信息

| 项目 | 值 |
|------|-----|
| Base URL | `https://api.appstoreconnect.apple.com/v1/` |
| 认证方式 | Bearer Token (JWT) |
| 内容类型 | `application/json` |
| API 版本 | v1 |

### 4.2 App 相关端点

| 端点 | 方法 | 说明 |
|------|------|------|
| `/v1/apps` | GET | 获取 App 列表 |
| `/v1/apps/{id}` | GET | 获取 App 详情 |
| `/v1/apps/{id}/appInfoLocalizations` | GET | 获取本地化信息 |
| `/v1/apps/{id}/appStoreVersions` | GET | 获取版本列表 |
| `/v1/apps/{id}/betaGroups` | GET | 获取 Beta 测试组 |
| `/v1/apps/{id}/appEvents` | GET | 获取活动列表 |
| `/v1/apps/{id}/appCustomProductPages` | GET | 获取自定义产品页 |

**获取 App 列表：**

```bash
curl -H "Authorization: Bearer $JWT_TOKEN" \
    "https://api.appstoreconnect.apple.com/v1/apps"
```

**响应示例：**

```json
{
    "data": [
        {
            "type": "apps",
            "id": "1234567890",
            "attributes": {
                "name": "MyApp",
                "bundleId": "com.example.myapp",
                "sku": "myapp2024",
                "primaryLocale": "zh-Hans"
            },
            "links": {
                "self": "https://api.appstoreconnect.apple.com/v1/apps/1234567890"
            }
        }
    ]
}
```

### 4.3 Build 相关端点

| 端点 | 方法 | 说明 |
|------|------|------|
| `/v1/builds` | GET | 获取构建列表 |
| `/v1/builds/{id}` | GET | 获取构建详情 |
| `/v1/builds/{id}` | PATCH | 更新构建信息 |
| `/v1/builds/{id}/appStoreVersion` | GET | 获取关联版本 |
| `/v1/builds/{id}/betaBuildMetrics` | GET | 获取 Beta 指标 |
| `/v1/builds/{id}/buildBetaDetail` | GET | 获取 Beta 详情 |

**获取构建列表：**

```bash
curl -H "Authorization: Bearer $JWT_TOKEN" \
    "https://api.appstoreconnect.apple.com/v1/builds?filter[app]=1234567890"
```

**更新构建信息（添加 What to Test）：**

```bash
curl -X PATCH \
    -H "Authorization: Bearer $JWT_TOKEN" \
    -H "Content-Type: application/json" \
    -d '{
        "data": {
            "type": "builds",
            "id": "build-id-here",
            "attributes": {
                "usesNonExemptEncryption": false
            }
        }
    }' \
    "https://api.appstoreconnect.apple.com/v1/builds/build-id-here"
```

### 4.4 BetaGroup 相关端点

| 端点 | 方法 | 说明 |
|------|------|------|
| `/v1/betaGroups` | GET | 获取测试组列表 |
| `/v1/betaGroups` | POST | 创建测试组 |
| `/v1/betaGroups/{id}` | PATCH | 更新测试组 |
| `/v1/betaGroups/{id}` | DELETE | 删除测试组 |
| `/v1/betaGroups/{id}/builds` | POST | 添加构建到测试组 |
| `/v1/betaGroups/{id}/betaTesters` | POST | 添加测试员 |

**创建 Beta 测试组：**

```bash
curl -X POST \
    -H "Authorization: Bearer $JWT_TOKEN" \
    -H "Content-Type: application/json" \
    -d '{
        "data": {
            "type": "betaGroups",
            "attributes": {
                "name": "Internal QA",
                "isInternalGroup": false,
                "publicLinkEnabled": false
            },
            "relationships": {
                "app": {
                    "data": {
                        "type": "apps",
                        "id": "1234567890"
                    }
                }
            }
        }
    }' \
    "https://api.appstoreconnect.apple.com/v1/betaGroups"
```

**将构建添加到测试组：**

```bash
curl -X POST \
    -H "Authorization: Bearer $JWT_TOKEN" \
    -H "Content-Type: application/json" \
    -d '{
        "data": [{
            "type": "builds",
            "id": "build-id-here"
        }]
    }' \
    "https://api.appstoreconnect.apple.com/v1/betaGroups/{group-id}/builds"
```

### 4.5 ReviewSubmission 相关端点

| 端点 | 方法 | 说明 |
|------|------|------|
| `/v1/appStoreVersions/{id}/appStoreVersionSubmissions` | POST | 提交审核 |
| `/v1/appStoreVersionSubmissions/{id}` | GET | 获取提交状态 |
| `/v1/appStoreVersionSubmissions/{id}` | DELETE | 取消提交 |

**提交审核：**

```bash
curl -X POST \
    -H "Authorization: Bearer $JWT_TOKEN" \
    -H "Content-Type: application/json" \
    -d '{
        "data": {
            "type": "appStoreVersionSubmissions",
            "relationships": {
                "appStoreVersion": {
                    "data": {
                        "type": "appStoreVersions",
                        "id": "version-id-here"
                    }
                }
            }
        }
    }' \
    "https://api.appstoreconnect.apple.com/v1/appStoreVersionSubmissions"
```

### 4.6 元数据相关端点

| 端点 | 方法 | 说明 |
|------|------|------|
| `/v1/appInfoLocalizations` | GET | 获取本地化信息 |
| `/v1/appInfoLocalizations/{id}` | PATCH | 更新本地化信息 |
| `/v1/appStoreVersionLocalizations` | GET | 获取版本本地化 |
| `/v1/appStoreVersionLocalizations/{id}` | PATCH | 更新版本本地化 |
| `/v1/appScreenshotSets` | GET | 获取截图集 |
| `/v1/appScreenshotSets/{id}/appScreenshots` | POST | 上传截图 |

**更新 App 描述：**

```bash
curl -X PATCH \
    -H "Authorization: Bearer $JWT_TOKEN" \
    -H "Content-Type: application/json" \
    -d '{
        "data": {
            "type": "appStoreVersionLocalizations",
            "id": "localization-id-here",
            "attributes": {
                "description": "全新版本上线！我们带来了AI智能助手功能...",
                "keywords": "效率,工具,AI,助手",
                "whatsNew": "新增AI智能助手，帮你更高效地管理日程"
            }
        }
    }' \
    "https://api.appstoreconnect.apple.com/v1/appStoreVersionLocalizations/localization-id-here"
```

### 4.7 截图上传流程

截图上传是一个多步骤过程：

```
Step 1: 创建截图记录
    POST /v1/appScreenshotSets/{id}/appScreenshots

Step 2: 获取上传参数
    从 Step 1 的响应中获取 uploadAttributes

Step 3: 上传图片文件
    PUT 到 Apple 提供的上传 URL

Step 4: 确认上传完成
    PATCH /v1/appScreenshots/{id}
    设置 uploaded = true

Step 5: 等待处理
    Apple 处理图片（自动裁剪不同尺寸）
```

**Step 1: 创建截图记录：**

```bash
curl -X POST \
    -H "Authorization: Bearer $JWT_TOKEN" \
    -H "Content-Type: application/json" \
    -d '{
        "data": {
            "type": "appScreenshots",
            "attributes": {
                "fileName": "screenshot_01.png",
                "fileSize": 1234567
            },
            "relationships": {
                "appScreenshotSet": {
                    "data": {
                        "type": "appScreenshotSets",
                        "id": "screenshot-set-id"
                    }
                }
            }
        }
    }' \
    "https://api.appstoreconnect.apple.com/v1/appScreenshots"
```

**Step 3: 上传图片：**

```bash
curl -X PUT \
    -H "Content-Type: image/png" \
    -T screenshot_01.png \
    "https://upload-url-from-step-1"
```

---

## 5. 自动化场景

### 5.1 元数据自动更新

**场景：** 多语言 App 的元数据需要同步更新。

```python
import requests
import json
import time

class AppStoreConnectAPI:
    def __init__(self, jwt_manager):
        self.jwt_manager = jwt_manager
        self.base_url = "https://api.appstoreconnect.apple.com/v1"

    def _headers(self):
        return {
            "Authorization": f"Bearer {self.jwt_manager.get_token()}",
            "Content-Type": "application/json"
        }

    def get_app(self, app_id):
        url = f"{self.base_url}/apps/{app_id}"
        response = requests.get(url, headers=self._headers())
        response.raise_for_status()
        return response.json()

    def get_app_store_version(self, app_id, version_string):
        url = f"{self.base_url}/apps/{app_id}/appStoreVersions"
        params = {"filter[versionString]": version_string}
        response = requests.get(url, headers=self._headers(), params=params)
        response.raise_for_status()
        data = response.json()
        if data["data"]:
            return data["data"][0]
        return None

    def get_version_localizations(self, version_id):
        url = f"{self.base_url}/appStoreVersions/{version_id}/appStoreVersionLocalizations"
        response = requests.get(url, headers=self._headers())
        response.raise_for_status()
        return response.json()["data"]

    def update_localization(self, localization_id, attributes):
        url = f"{self.base_url}/appStoreVersionLocalizations/{localization_id}"
        payload = {
            "data": {
                "type": "appStoreVersionLocalizations",
                "id": localization_id,
                "attributes": attributes
            }
        }
        response = requests.patch(url, headers=self._headers(), json=payload)
        response.raise_for_status()
        return response.json()
```

**批量更新多语言描述：**

```python
def update_metadata_for_all_locales(api, app_id, version_string, metadata):
    version = api.get_app_store_version(app_id, version_string)
    if not version:
        print(f"版本 {version_string} 不存在")
        return

    localizations = api.get_version_localizations(version["id"])

    for loc in localizations:
        locale = loc["attributes"]["locale"]
        if locale in metadata:
            api.update_localization(loc["id"], metadata[locale])
            print(f"✅ 已更新 {locale} 的元数据")
        else:
            print(f"⏭ 跳过 {locale}（无数据）")

metadata = {
    "zh-Hans": {
        "description": "全新版本上线！AI 智能助手帮你高效管理日程。",
        "keywords": "效率,工具,AI,助手,日程",
        "whatsNew": "新增 AI 智能助手功能"
    },
    "en-US": {
        "description": "New version released! AI assistant helps you manage your schedule efficiently.",
        "keywords": "productivity,tool,AI,assistant,schedule",
        "whatsNew": "Added AI assistant feature"
    },
    "ja": {
        "description": "新バージョンリリース！AIアシスタントがスケジュール管理をサポート。",
        "keywords": "生産性,ツール,AI,アシスタント,スケジュール",
        "whatsNew": "AIアシスタント機能を追加"
    }
}

api = AppStoreConnectAPI(jwt_manager)
update_metadata_for_all_locales(api, "1234567890", "2.0.0", metadata)
```

### 5.2 构建自动分发

**场景：** 构建上传后自动分发到 TestFlight 测试组。

```python
def auto_distribute_build(api, app_id, build_id, group_names):
    beta_groups = get_beta_groups(api, app_id)

    for group in beta_groups:
        group_name = group["attributes"]["name"]
        if group_name in group_names:
            add_build_to_group(api, group["id"], build_id)
            print(f"✅ 已将构建分发到 {group_name}")

def get_beta_groups(api, app_id):
    url = f"{api.base_url}/apps/{app_id}/betaGroups"
    response = requests.get(url, headers=api._headers())
    response.raise_for_status()
    return response.json()["data"]

def add_build_to_group(api, group_id, build_id):
    url = f"{api.base_url}/betaGroups/{group_id}/builds"
    payload = {
        "data": [{"type": "builds", "id": build_id}]
    }
    response = requests.post(url, headers=api._headers(), json=payload)
    response.raise_for_status()
```

### 5.3 TestFlight 自动管理

**场景：** 自动添加测试员、管理测试组。

```python
def add_beta_tester(api, group_id, email, first_name, last_name):
    url = f"{api.base_url}/betaTesters"
    payload = {
        "data": {
            "type": "betaTesters",
            "attributes": {
                "email": email,
                "firstName": first_name,
                "lastName": last_name
            },
            "relationships": {
                "betaGroups": {
                    "data": [{"type": "betaGroups", "id": group_id}]
                }
            }
        }
    }
    response = requests.post(url, headers=api._headers(), json=payload)
    response.raise_for_status()
    print(f"✅ 已添加测试员: {email}")

def remove_beta_tester(api, group_id, tester_id):
    url = f"{api.base_url}/betaGroups/{group_id}/relationships/betaTesters"
    payload = {
        "data": [{"type": "betaTesters", "id": tester_id}]
    }
    response = requests.delete(url, headers=api._headers(), json=payload)
    response.raise_for_status()
    print(f"✅ 已移除测试员: {tester_id}")
```

### 5.4 自动提交审核

**场景：** 构建准备好后自动提交审核。

```python
def auto_submit_for_review(api, app_id, version_string):
    version = api.get_app_store_version(app_id, version_string)
    if not version:
        print(f"版本 {version_string} 不存在")
        return

    version_id = version["id"]

    url = f"{api.base_url}/appStoreVersionSubmissions"
    payload = {
        "data": {
            "type": "appStoreVersionSubmissions",
            "relationships": {
                "appStoreVersion": {
                    "data": {
                        "type": "appStoreVersions",
                        "id": version_id
                    }
                }
            }
        }
    }

    response = requests.post(url, headers=api._headers(), json=payload)

    if response.status_code == 201:
        print("✅ 已成功提交审核")
    elif response.status_code == 409:
        print("⚠️ 提交冲突：可能已有版本在审核中")
    else:
        print(f"❌ 提交失败: {response.status_code}")
        print(response.json())
```

### 5.5 审核状态监控

```python
def check_review_status(api, app_id, version_string):
    version = api.get_app_store_version(app_id, version_string)
    if not version:
        return None

    version_id = version["id"]
    url = f"{api.base_url}/appStoreVersions/{version_id}/appStoreVersionSubmissions"
    response = requests.get(url, headers=api._headers())
    response.raise_for_status()

    submissions = response.json()["data"]
    if submissions:
        return submissions[0]["attributes"]["state"]
    return None

def poll_review_status(api, app_id, version_string, interval=300):
    while True:
        status = check_review_status(api, app_id, version_string)
        print(f"📋 审核状态: {status}")

        if status in ["ACCEPTED", "REJECTED", "REMOVED"]:
            return status

        time.sleep(interval)
```

**审核状态说明：**

| 状态 | 说明 |
|------|------|
| READY_FOR_SUBMISSION | 准备提交 |
| WAITING_FOR_REVIEW | 等待审核 |
| IN_REVIEW | 审核中 |
| PENDING_APP_RELEASE | 审核通过，等待发布 |
| PROCESSING_FOR_APP_STORE | 正在处理 |
| READY_FOR_SALE | 已上架 |
| REJECTED | 被拒 |
| METADATA_REJECTED | 元数据被拒 |
| REMOVED_FROM_SALE | 已下架 |

---

## 6. 结合 Fastlane 的自动化

### 6.1 Fastlane 与 App Store Connect API

Fastlane 从 2.158 版本开始支持 App Store Connect API Key 认证，无需 Apple ID 和密码。

**配置 API Key：**

```ruby
# Appfile
app_identifier("com.example.myapp")

# 方式 1：使用 .p8 文件
api_key_path("./fastlane/AuthKey_2X9R4HXF34.p8")

# 方式 2：使用环境变量（推荐 CI 环境）
api_key(
  key_id: ENV["APP_STORE_CONNECT_API_KEY_ID"],
  issuer_id: ENV["APP_STORE_CONNECT_ISSUER_ID"],
  key_filepath: "./fastlane/AuthKey_2X9R4HXF34.p8"
)

# 方式 3：直接使用 key 内容
api_key(
  key_id: ENV["APP_STORE_CONNECT_API_KEY_ID"],
  issuer_id: ENV["APP_STORE_CONNECT_ISSUER_ID"],
  key: ENV["APP_STORE_CONNECT_API_KEY"]
)
```

### 6.2 deliver — 元数据和截图上传

`deliver`（也叫 `upload_to_appstore`）是 Fastlane 中管理 App Store 元数据的 Action。

**初始化 deliver：**

```bash
fastlane deliver init
```

**目录结构：**

```
fastlane/
└── metadata/
    ├── zh-Hans/
    │   ├── description.txt
    │   ├── keywords.txt
    │   ├── release_notes.txt
    │   ├── name.txt
    │   ├── privacy_url.txt
    │   ├── support_url.txt
    │   ├── subtitle.txt
    │   └── screenshots/
    │       ├── 01_iphone6_1.png
    │       ├── 02_iphone6_2.png
    │       └── ...
    ├── en-US/
    │   ├── description.txt
    │   ├── keywords.txt
    │   ├── release_notes.txt
    │   └── screenshots/
    │       └── ...
    └── ja/
        ├── description.txt
        └── ...
```

**上传元数据：**

```ruby
lane :upload_metadata do
  deliver(
    api_key: api_key,
    skip_binary_upload: true,
    skip_screenshots: false,
    force: true,
    metadata_path: "./fastlane/metadata",
    screenshots_path: "./fastlane/metadata",
    submit_for_review: false,
    automatic_release: false,
    phased_release: true
  )
end
```

**仅上传截图：**

```ruby
lane :upload_screenshots do
  deliver(
    api_key: api_key,
    skip_binary_upload: true,
    skip_metadata: true,
    screenshots_path: "./fastlane/metadata",
    force: true
  )
end
```

### 6.3 upload_to_appstore — 构建上传

```ruby
lane :upload_build do
  build_path = Dir.glob("build/MyApp.ipa").first

  upload_to_appstore(
    api_key: api_key,
    ipa: build_path,
    skip_metadata: true,
    skip_screenshots: true,
    submit_for_review: false
  )
end
```

### 6.4 完整发布流水线

```ruby
def api_key
  @api_key ||= app_store_connect_api_key(
    key_id: ENV["APP_STORE_CONNECT_API_KEY_ID"],
    issuer_id: ENV["APP_STORE_CONNECT_ISSUER_ID"],
    key_filepath: "./fastlane/AuthKey.p8",
    duration: 1200,
    in_house: false
  )
end

lane :release do
  ensure_git_branch(branch: "main")
  ensure_git_status_clean

  version = get_version_number(target: "MyApp")
  build_number = increment_build_number

  match(
    type: "appstore",
    readonly: true,
    api_key: api_key
  )

  gym(
    scheme: "MyApp",
    export_method: "app-store",
    archive_path: "build/MyApp.xcarchive",
    output_directory: "build",
    output_name: "MyApp.ipa"
  )

  upload_to_appstore(
    api_key: api_key,
    ipa: "build/MyApp.ipa",
    metadata_path: "./fastlane/metadata",
    screenshots_path: "./fastlane/metadata",
    submit_for_review: true,
    automatic_release: false,
    phased_release: true,
    precheck_include_in_app_purchases: false
  )

  add_git_tag(
    tag: "v#{version}-#{build_number}",
    message: "Release v#{version}"
  )
  push_git_tags
end
```

### 6.5 TestFlight 自动分发

```ruby
lane :beta do
  match(
    type: "appstore",
    readonly: true,
    api_key: api_key
  )

  gym(
    scheme: "MyApp",
    export_method: "app-store"
  )

  pilot(
    api_key: api_key,
    distribute_external: true,
    groups: ["External QA", "Public Beta"],
    changelog: "Bug fixes and performance improvements",
    notify_external_testers: true
  )
end
```

---

## 7. 结合 GitHub Actions 的自动化发布

### 7.1 GitHub Actions 工作流配置

```yaml
name: Release to App Store

on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Version string (e.g., 2.0.0)'
        required: true
      submit_for_review:
        description: 'Submit for review after upload'
        type: boolean
        default: false

jobs:
  release:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.2'
          bundler-cache: true

      - name: Install dependencies
        run: |
          bundle install
          pod install

      - name: Setup API Key
        env:
          API_KEY_BASE64: ${{ secrets.APP_STORE_CONNECT_API_KEY_BASE64 }}
        run: |
          mkdir -p fastlane
          echo "$API_KEY_BASE64" | base64 --decode > fastlane/AuthKey.p8

      - name: Build and Upload
        env:
          APP_STORE_CONNECT_API_KEY_ID: ${{ secrets.APP_STORE_CONNECT_API_KEY_ID }}
          APP_STORE_CONNECT_ISSUER_ID: ${{ secrets.APP_STORE_CONNECT_ISSUER_ID }}
          MATCH_PASSWORD: ${{ secrets.MATCH_PASSWORD }}
          MATCH_GIT_URL: ${{ secrets.MATCH_GIT_URL }}
          MATCH_GIT_BASIC_AUTHORIZATION: ${{ secrets.MATCH_GIT_BASIC_AUTHORIZATION }}
          SUBMIT_FOR_REVIEW: ${{ github.event.inputs.submit_for_review }}
        run: |
          bundle exec fastlane release

      - name: Cleanup
        if: always()
        run: rm -f fastlane/AuthKey.p8
```

### 7.2 TestFlight Beta 分发工作流

```yaml
name: Beta Distribution

on:
  push:
    branches: [develop]

jobs:
  beta:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.2'
          bundler-cache: true

      - name: Install dependencies
        run: |
          bundle install
          pod install

      - name: Setup API Key
        env:
          API_KEY_BASE64: ${{ secrets.APP_STORE_CONNECT_API_KEY_BASE64 }}
        run: |
          mkdir -p fastlane
          echo "$API_KEY_BASE64" | base64 --decode > fastlane/AuthKey.p8

      - name: Build and Distribute to TestFlight
        env:
          APP_STORE_CONNECT_API_KEY_ID: ${{ secrets.APP_STORE_CONNECT_API_KEY_ID }}
          APP_STORE_CONNECT_ISSUER_ID: ${{ secrets.APP_STORE_CONNECT_ISSUER_ID }}
          MATCH_PASSWORD: ${{ secrets.MATCH_PASSWORD }}
          MATCH_GIT_URL: ${{ secrets.MATCH_GIT_URL }}
          MATCH_GIT_BASIC_AUTHORIZATION: ${{ secrets.MATCH_GIT_BASIC_AUTHORIZATION }}
        run: |
          bundle exec fastlane beta

      - name: Cleanup
        if: always()
        run: rm -f fastlane/AuthKey.p8
```

### 7.3 元数据自动更新工作流

```yaml
name: Update App Store Metadata

on:
  push:
    paths:
      - 'fastlane/metadata/**'
    branches: [main]

jobs:
  update-metadata:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.2'
          bundler-cache: true

      - name: Install dependencies
        run: bundle install

      - name: Setup API Key
        env:
          API_KEY_BASE64: ${{ secrets.APP_STORE_CONNECT_API_KEY_BASE64 }}
        run: |
          mkdir -p fastlane
          echo "$API_KEY_BASE64" | base64 --decode > fastlane/AuthKey.p8

      - name: Upload Metadata
        env:
          APP_STORE_CONNECT_API_KEY_ID: ${{ secrets.APP_STORE_CONNECT_API_KEY_ID }}
          APP_STORE_CONNECT_ISSUER_ID: ${{ secrets.APP_STORE_CONNECT_ISSUER_ID }}
        run: |
          bundle exec fastlane upload_metadata

      - name: Cleanup
        if: always()
        run: rm -f fastlane/AuthKey.p8
```

### 7.4 Secrets 配置

在 GitHub 仓库的 Settings → Secrets and variables → Actions 中配置：

| Secret 名称 | 说明 | 获取方式 |
|--------------|------|----------|
| APP_STORE_CONNECT_API_KEY_ID | API Key ID | App Store Connect → 集成 → API |
| APP_STORE_CONNECT_ISSUER_ID | Issuer ID | App Store Connect → 集成 → API |
| APP_STORE_CONNECT_API_KEY_BASE64 | .p8 文件 Base64 | `base64 -i AuthKey.p8` |
| MATCH_PASSWORD | match 加密密码 | 自定义 |
| MATCH_GIT_URL | match 仓库地址 | Git 仓库 URL |
| MATCH_GIT_BASIC_AUTHORIZATION | Git 认证 | `echo -n "user:token" \| base64` |

**生成 Base64 编码的 .p8 文件：**

```bash
base64 -i AuthKey_2X9R4HXF34.p8 | pbcopy
```

---

## 8. API 速率限制和错误处理

### 8.1 速率限制

App Store Connect API 有速率限制：

| 限制类型 | 值 |
|----------|-----|
| 每小时请求数 | 根据角色不同，通常 200-1000 次/小时 |
| 并发请求数 | 5 个并发请求 |
| JWT 有效期 | 最长 20 分钟 |

**速率限制响应头：**

| Header | 说明 |
|--------|------|
| X-RateLimit-Limit | 每小时允许的请求总数 |
| X-RateLimit-Remaining | 剩余请求数 |
| X-RateLimit-Reset | 限制重置时间（Unix 时间戳） |

### 8.2 常见错误码

| HTTP 状态码 | 错误码 | 说明 | 处理方式 |
|-------------|--------|------|----------|
| 400 | INVALID_REQUEST | 请求格式错误 | 检查请求体 |
| 401 | NOT_AUTHORIZED | JWT 无效或过期 | 重新生成 JWT |
| 403 | FORBIDDEN | 权限不足 | 检查 API Key 角色 |
| 404 | NOT_FOUND | 资源不存在 | 检查资源 ID |
| 409 | ENTITY_ERROR | 资源冲突 | 检查是否重复操作 |
| 429 | RATE_LIMIT_EXCEEDED | 超出速率限制 | 等待后重试 |
| 500 | SERVER_ERROR | 服务器错误 | 重试 |

### 8.3 错误处理最佳实践

```python
import requests
import time

class APIError(Exception):
    def __init__(self, status_code, error_code, message):
        self.status_code = status_code
        self.error_code = error_code
        self.message = message
        super().__init__(f"[{status_code}] {error_code}: {message}")

def api_request(method, url, headers, json=None, max_retries=3):
    for attempt in range(max_retries):
        try:
            response = requests.request(
                method, url, headers=headers, json=json, timeout=30
            )

            if response.status_code == 429:
                reset_time = int(response.headers.get("X-RateLimit-Reset", 0))
                wait_time = max(reset_time - int(time.time()), 60)
                print(f"⚠️ 速率限制，等待 {wait_time} 秒...")
                time.sleep(wait_time)
                continue

            if response.status_code >= 500:
                wait_time = 2 ** attempt
                print(f"⚠️ 服务器错误，{wait_time} 秒后重试...")
                time.sleep(wait_time)
                continue

            if response.status_code >= 400:
                error_data = response.json().get("errors", [{}])[0]
                raise APIError(
                    status_code=response.status_code,
                    error_code=error_data.get("code", "UNKNOWN"),
                    message=error_data.get("detail", response.text)
                )

            return response.json()

        except requests.exceptions.Timeout:
            print(f"⚠️ 请求超时，重试 {attempt + 1}/{max_retries}")
            time.sleep(2 ** attempt)

    raise APIError(0, "MAX_RETRIES", "达到最大重试次数")
```

### 8.4 分页处理

API 返回大量数据时使用分页：

```python
def get_all_pages(url, headers):
    all_data = []

    while url:
        response = requests.get(url, headers=headers)
        response.raise_for_status()
        data = response.json()

        all_data.extend(data.get("data", []))

        next_url = data.get("links", {}).get("next")
        url = next_url

    return all_data
```

**分页参数：**

| 参数 | 说明 | 默认值 |
|------|------|--------|
| limit | 每页数量 | 50 |
| sort | 排序字段 | - |
| cursor | 分页游标 | - |

```bash
# 使用分页参数
curl -H "Authorization: Bearer $JWT_TOKEN" \
    "https://api.appstoreconnect.apple.com/v1/apps?limit=10&cursor=xxx"
```

### 8.5 请求优化

| 优化策略 | 说明 |
|----------|------|
| 包含关联数据 | 使用 `include` 参数减少请求次数 |
| 字段过滤 | 使用 `fields` 参数只获取需要的字段 |
| 批量操作 | 尽量使用批量 API |
| 缓存结果 | 缓存不常变化的数据 |
| JWT 缓存 | 缓存 JWT 令牌，避免频繁生成 |

```bash
# 包含关联数据
curl -H "Authorization: Bearer $JWT_TOKEN" \
    "https://api.appstoreconnect.apple.com/v1/apps/1234567890?include=appStoreVersions"

# 字段过滤
curl -H "Authorization: Bearer $JWT_TOKEN" \
    "https://api.appstoreconnect.apple.com/v1/apps?fields[apps]=name,bundleId"
```

---

## 9. App Store Connect API 与 Xcode Cloud 的配合

### 9.1 Xcode Cloud 的 API 集成

Xcode Cloud 是 Apple 的 CI/CD 服务，与 App Store Connect 深度集成。

**Xcode Cloud 工作流：**

```
代码提交 → Xcode Cloud 自动构建 → 自动测试 → 自动归档
    → 自动上传到 App Store Connect → 自动分发到 TestFlight
```

### 9.2 Xcode Cloud 自定义脚本

Xcode Cloud 支持在构建过程中运行自定义脚本：

**ci_scripts/ci_post_xcodebuild.sh：**

```bash
#!/bin/bash
# 构建完成后执行

if [ "$CI_XCODEBUILD_RESULT" = "success" ]; then
    echo "✅ 构建成功"

    # 获取构建信息
    BUILD_ID=$(xcrun notarytool info --apple-id "$APPLE_ID" --password "$APP_PASSWORD" --team-id "$TEAM_ID" 2>/dev/null || echo "")

    # 通知团队
    curl -X POST "$SLACK_WEBHOOK_URL" \
        -H "Content-Type: application/json" \
        -d "{\"text\": \"✅ 构建成功: $CI_PRODUCT_NAME $CI_BUILD_NUMBER\"}"
else
    echo "❌ 构建失败"

    curl -X POST "$SLACK_WEBHOOK_URL" \
        -H "Content-Type: application/json" \
        -d "{\"text\": \"❌ 构建失败: $CI_PRODUCT_NAME $CI_BUILD_NUMBER\"}"
fi
```

### 9.3 Xcode Cloud 环境变量

| 变量 | 说明 |
|------|------|
| CI | 是否在 CI 环境 |
| CI_PRODUCT_NAME | 产品名称 |
| CI_BUILD_NUMBER | 构建号 |
| CI_BRANCH | 当前分支 |
| CI_COMMIT | 当前提交哈希 |
| CI_XCODEBUILD_RESULT | 构建结果 |
| CI_PRIMARY_REPO_URL | 仓库地址 |

### 9.4 Xcode Cloud vs Fastlane + GitHub Actions

| 特性 | Xcode Cloud | Fastlane + GitHub Actions |
|------|-------------|---------------------------|
| 配置方式 | App Store Connect 界面 | 代码配置 |
| 灵活度 | ⭐⭐ 中等 | ⭐⭐⭐ 最高 |
| 证书管理 | 全自动 | 需配置 match |
| macOS Runner | Apple 托管 | GitHub 托管或自建 |
| 费用 | 按用量计费 | 免费（公开仓库） |
| 自定义脚本 | 有限 | 完全自由 |
| API 集成 | 内置 | 需手动集成 |
| 适合团队 | 小团队 / iOS 新手 | 需要深度定制的团队 |

---

## 10. 安全最佳实践

### 10.1 密钥管理

| 实践 | 说明 |
|------|------|
| 不提交 .p8 到 Git | .p8 文件是敏感信息，绝不提交到代码仓库 |
| 使用 CI Secrets | 通过 CI 系统的 Secrets 功能注入密钥 |
| Base64 编码 | 将 .p8 文件 Base64 编码后存储在 Secrets 中 |
| 定期轮换 | 定期更换 API Key，降低泄露风险 |
| 最小权限 | 为不同用途创建不同角色的 Key |
| 审计日志 | 定期检查 API 使用记录 |

### 10.2 .gitignore 配置

```gitignore
# App Store Connect API Key
*.p8
AuthKey_*

# Fastlane
fastlane/AuthKey_*.p8
fastlane/README.md

# 环境变量
.env
.env.*
```

### 10.3 JWT 安全

| 实践 | 说明 |
|------|------|
| 短有效期 | JWT 有效期设为最短必要时间 |
| 不复用 | 每次请求使用新 JWT（或缓存的未过期 JWT） |
| 安全传输 | 始终使用 HTTPS |
| 不记录日志 | 不要在日志中输出 JWT |

### 10.4 CI/CD 安全清单

| 检查项 | 说明 |
|--------|------|
| Secrets 不出现在日志中 | CI 系统应自动屏蔽 Secrets |
| 构建后清理 | 删除临时文件和 Keychain |
| 最小权限 | CI 使用的 API Key 只有必要权限 |
| 分支保护 | 限制触发发布的分支 |
| 审批流程 | 发布操作需要人工审批 |
| 审计追踪 | 记录谁在什么时候触发了发布 |

### 10.5 API Key 泄露应急处理

```
1. 立即在 App Store Connect 中撤销泄露的 API Key
2. 创建新的 API Key
3. 更新所有 CI/CD 系统中的 Secrets
4. 检查 API 使用记录，确认无异常操作
5. 通知团队成员
6. 记录事件并改进安全流程
```

---

## 11. 完整自动化脚本示例

### 11.1 Python 自动化脚本

```python
#!/usr/bin/env python3
"""
App Store Connect API 自动化脚本
功能：元数据更新 + 构建分发 + 审核提交
"""

import requests
import json
import time
import sys
import argparse

class AppStoreConnectClient:
    BASE_URL = "https://api.appstoreconnect.apple.com/v1"

    def __init__(self, key_id, issuer_id, private_key_path):
        self.key_id = key_id
        self.issuer_id = issuer_id
        self.private_key_path = private_key_path
        self._token = None
        self._token_expiry = 0

    def _generate_token(self):
        import jwt
        with open(self.private_key_path, 'r') as f:
            private_key = f.read()

        now = int(time.time())
        payload = {
            'iss': self.issuer_id,
            'iat': now,
            'exp': now + 1200,
            'aud': 'appstoreconnect-v1'
        }
        headers = {
            'alg': 'ES256',
            'kid': self.key_id,
            'typ': 'JWT'
        }

        self._token = jwt.encode(payload, private_key, algorithm='ES256', headers=headers)
        self._token_expiry = now + 1200
        return self._token

    def _get_token(self):
        now = int(time.time())
        if not self._token or now >= self._token_expiry - 60:
            self._generate_token()
        return self._token

    def _headers(self):
        return {
            "Authorization": f"Bearer {self._get_token()}",
            "Content-Type": "application/json"
        }

    def get_app_id(self, bundle_id):
        url = f"{self.BASE_URL}/apps"
        params = {"filter[bundleId]": bundle_id}
        resp = requests.get(url, headers=self._headers(), params=params)
        resp.raise_for_status()
        apps = resp.json().get("data", [])
        if apps:
            return apps[0]["id"]
        return None

    def get_latest_version(self, app_id):
        url = f"{self.BASE_URL}/apps/{app_id}/appStoreVersions"
        params = {"limit": 1, "sort": "-versionString"}
        resp = requests.get(url, headers=self._headers(), params=params)
        resp.raise_for_status()
        versions = resp.json().get("data", [])
        if versions:
            return versions[0]
        return None

    def update_whats_new(self, version_localization_id, whats_new):
        url = f"{self.BASE_URL}/appStoreVersionLocalizations/{version_localization_id}"
        payload = {
            "data": {
                "type": "appStoreVersionLocalizations",
                "id": version_localization_id,
                "attributes": {
                    "whatsNew": whats_new
                }
            }
        }
        resp = requests.patch(url, headers=self._headers(), json=payload)
        resp.raise_for_status()
        return resp.json()

    def submit_for_review(self, version_id):
        url = f"{self.BASE_URL}/appStoreVersionSubmissions"
        payload = {
            "data": {
                "type": "appStoreVersionSubmissions",
                "relationships": {
                    "appStoreVersion": {
                        "data": {
                            "type": "appStoreVersions",
                            "id": version_id
                        }
                    }
                }
            }
        }
        resp = requests.post(url, headers=self._headers(), json=payload)
        if resp.status_code == 201:
            print("✅ 已成功提交审核")
            return True
        else:
            print(f"❌ 提交审核失败: {resp.status_code}")
            print(resp.json())
            return False


def main():
    parser = argparse.ArgumentParser(description="App Store Connect 自动化工具")
    parser.add_argument("--key-id", required=True, help="API Key ID")
    parser.add_argument("--issuer-id", required=True, help="Issuer ID")
    parser.add_argument("--private-key", required=True, help=".p8 文件路径")
    parser.add_argument("--bundle-id", required=True, help="App Bundle ID")
    parser.add_argument("--action", choices=["update-whats-new", "submit-review", "check-status"],
                        required=True, help="执行的操作")
    parser.add_argument("--whats-new", help="更新日志内容")
    args = parser.parse_args()

    client = AppStoreConnectClient(args.key_id, args.issuer_id, args.private_key)
    app_id = client.get_app_id(args.bundle_id)

    if not app_id:
        print(f"❌ 找不到 Bundle ID 为 {args.bundle_id} 的 App")
        sys.exit(1)

    print(f"📱 App ID: {app_id}")

    if args.action == "update-whats-new":
        if not args.whats_new:
            print("❌ 请提供 --whats-new 参数")
            sys.exit(1)

        version = client.get_latest_version(app_id)
        if not version:
            print("❌ 找不到版本信息")
            sys.exit(1)

        print(f"📦 版本: {version['attributes']['versionString']}")

        localizations_url = version["relationships"]["appStoreVersionLocalizations"]["links"]["related"]
        resp = requests.get(localizations_url, headers=client._headers())
        resp.raise_for_status()
        localizations = resp.json().get("data", [])

        for loc in localizations:
            locale = loc["attributes"]["locale"]
            client.update_whats_new(loc["id"], args.whats_new)
            print(f"✅ 已更新 {locale} 的更新日志")

    elif args.action == "submit-review":
        version = client.get_latest_version(app_id)
        if not version:
            print("❌ 找不到版本信息")
            sys.exit(1)

        client.submit_for_review(version["id"])

    elif args.action == "check-status":
        version = client.get_latest_version(app_id)
        if not version:
            print("❌ 找不到版本信息")
            sys.exit(1)

        state = version["attributes"].get("appStoreState", "UNKNOWN")
        print(f"📋 当前状态: {state}")


if __name__ == "__main__":
    main()
```

### 11.2 使用方式

```bash
# 更新更新日志
python3 asc_automation.py \
    --key-id "2X9R4HXF34" \
    --issuer-id "69a6de7e-XXXX-XXXX-XXXX-XXXXXXXXXXXX" \
    --private-key "./AuthKey_2X9R4HXF34.p8" \
    --bundle-id "com.example.myapp" \
    --action update-whats-new \
    --whats-new "新增 AI 智能助手功能，优化性能体验"

# 提交审核
python3 asc_automation.py \
    --key-id "2X9R4HXF34" \
    --issuer-id "69a6de7e-XXXX-XXXX-XXXX-XXXXXXXXXXXX" \
    --private-key "./AuthKey_2X9R4HXF34.p8" \
    --bundle-id "com.example.myapp" \
    --action submit-review

# 检查审核状态
python3 asc_automation.py \
    --key-id "2X9R4HXF34" \
    --issuer-id "69a6de7e-XXXX-XXXX-XXXX-XXXXXXXXXXXX" \
    --private-key "./AuthKey_2X9R4HXF34.p8" \
    --bundle-id "com.example.myapp" \
    --action check-status
```

---

## 12. 常见问题与排错

### 12.1 认证问题

| 问题 | 原因 | 解决方法 |
|------|------|----------|
| 401 NOT_AUTHORIZED | JWT 过期 | 重新生成 JWT |
| 401 NOT_AUTHORIZED | Key ID 错误 | 检查 Key ID 是否正确 |
| 401 NOT_AUTHORIZED | .p8 文件不匹配 | 确认 .p8 与 Key ID 对应 |
| 401 NOT_AUTHORIZED | 时间偏差 | 检查服务器时间是否准确 |

### 12.2 权限问题

| 问题 | 原因 | 解决方法 |
|------|------|----------|
| 403 FORBIDDEN | API Key 角色权限不足 | 使用更高权限的角色 |
| 403 FORBIDDEN | 操作超出 Key 权限范围 | 创建对应权限的 Key |

### 12.3 数据问题

| 问题 | 原因 | 解决方法 |
|------|------|----------|
| 404 NOT_FOUND | 资源 ID 不存在 | 检查 ID 是否正确 |
| 409 ENTITY_ERROR | 资源冲突 | 检查是否重复操作 |
| 422 ENTITY_ERROR | 数据验证失败 | 检查请求体格式 |

### 12.4 速率限制问题

| 问题 | 原因 | 解决方法 |
|------|------|----------|
| 429 RATE_LIMIT | 请求过于频繁 | 实现退避重试 |
| 429 RATE_LIMIT | 并发请求过多 | 限制并发数为 5 |

### 12.5 调试技巧

```bash
# 验证 JWT 是否有效
echo $JWT_TOKEN | cut -d. -f2 | base64 -d 2>/dev/null | python3 -m json.tool

# 查看速率限制
curl -I -H "Authorization: Bearer $JWT_TOKEN" \
    "https://api.appstoreconnect.apple.com/v1/apps" | \
    grep -i "x-ratelimit"

# 查看 API 版本
curl -H "Authorization: Bearer $JWT_TOKEN" \
    "https://api.appstoreconnect.apple.com/v1/apps?limit=1" | \
    python3 -m json.tool | head -5
```

---

## 本章小结

| 主题 | 核心要点 |
|------|----------|
| API 概述 | RESTful API，JWT 认证，覆盖 App 管理、构建、TestFlight、审核等 |
| API Key | 在 App Store Connect 创建，选择最小权限角色，.p8 文件安全存储 |
| JWT 生成 | ES256 签名，最长 20 分钟有效期，支持 Python/Shell/Ruby/Swift |
| 核心端点 | App、Build、BetaGroup、ReviewSubmission、Localizations |
| 自动化场景 | 元数据更新、构建分发、TestFlight 管理、审核提交 |
| Fastlane 集成 | deliver 上传元数据、pilot 管理 TestFlight、API Key 认证 |
| GitHub Actions | Secrets 管理 API Key，工作流触发构建和发布 |
| 速率限制 | 注意 429 错误，实现退避重试，使用分页和字段过滤优化请求 |
| Xcode Cloud | Apple 官方 CI/CD，与 API 深度集成，适合小团队 |
| 安全实践 | .p8 不提交 Git、CI Secrets 注入、最小权限、定期轮换 |

> 💡 一句话总结：App Store Connect API 是 iOS 发布自动化的"钥匙"——掌握它，让每次发布从手动操作变成自动化流水线。

← [App内活动与自定义产品页](./App内活动与自定义产品页.md) | [目录](../README.md) →
