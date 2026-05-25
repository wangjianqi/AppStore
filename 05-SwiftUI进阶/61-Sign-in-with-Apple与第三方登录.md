# 61-Sign in with Apple 与第三方登录

> 🎯 **本章目标**：
> - 理解登录功能在 App 中的核心价值与必要性
> - 掌握 Sign in with Apple 的完整实现流程
> - 学会 SwiftUI 中封装 Apple 登录按钮的方式
> - 了解 Token 验证与后端对接的 JWT 验证流程
> - 掌握凭据状态检查与撤销处理机制
> - 能够集成微信、Google、GitHub 等第三方登录
> - 建立多登录方式统一管理的架构
> - 使用 Keychain 安全存储登录凭据
> - 规避 App Store 审核中与登录相关的常见被拒原因

---

## 1. 为什么需要登录功能

> 💡 **生活类比**：想象一家酒店——没有身份证登记，你每次入住都要重新填表、选房间偏好、设置叫醒时间。有了会员系统，你的偏好和历史记录都跟着你走。App 登录就是用户的"数字身份证"。

### 1.1 登录的三大核心价值

| 价值维度 | 说明 | 示例 |
|----------|------|------|
| 用户身份 | 唯一标识用户，防止数据混淆 | 多设备登录同一账号 |
| 数据同步 | 跨设备同步用户数据 | 备忘录、阅读进度云端同步 |
| 个性化 | 基于用户行为提供定制体验 | 推荐算法、主题偏好保存 |

### 1.2 何时需要登录

并非所有 App 都需要登录。以下决策表帮你判断：

| 场景 | 是否需要登录 | 原因 |
|------|-------------|------|
| 工具类 App（计算器、手电筒） | ❌ | 无需持久化用户数据 |
| 笔记类 App | ✅ | 需要跨设备同步 |
| 社交类 App | ✅ | 核心功能依赖用户身份 |
| 电商类 App | ✅ | 订单、购物车需关联用户 |
| 单机游戏 | ❌ | 本地存档即可 |
| 多人在线游戏 | ✅ | 需要身份匹配与进度同步 |

> ⚠️ **注意**：如果 App 核心功能不需要登录，就不要强制用户登录。Apple 审核可能因"强制登录才能使用基本功能"而拒审。

---

## 2. Apple 审核强制要求

### 2.1 审核规则解读

自 2019 年起，Apple 在 App Store 审核指南 4.8 节明确规定：

> **如果 App 支持任何第三方社交登录服务（如 Facebook、Google、微信等），则必须同时提供 Sign in with Apple 选项。**

> ⚠️ **严重后果**：不遵守此规则的 App 将被拒绝上架或更新。

### 2.2 规则适用范围

| 情况 | 是否必须提供 Sign in with Apple |
|------|-------------------------------|
| 仅支持 Sign in with Apple | 不适用（已满足） |
| 支持微信 + Google 登录 | ✅ 必须提供 |
| 仅支持手机号验证码登录 | ❌ 不需要（非社交登录） |
| 仅支持邮箱密码登录 | ❌ 不需要（非社交登录） |
| 支持企业内部 SSO 登录 | ❌ 不需要（仅限企业内部分发） |
| 支持第三方登录 + 手机号登录 | ✅ 必须提供 |

### 2.3 合规策略

```swift
// ❌ 错误做法：只放第三方登录，没有 Apple 登录
VStack {
    WeChatLoginButton()
    GoogleLoginButton()
}

// ✅ 正确做法：有第三方登录时，必须同时提供 Apple 登录
VStack {
    SignInWithAppleButton()
    WeChatLoginButton()
    GoogleLoginButton()
    PhoneLoginButton() // 额外的非社交登录方式也可以保留
}
```

---

## 3. Sign in with Apple 完整实现

### 3.1 核心类关系

> 💡 **生活类比**：把登录流程想象成去银行办业务——`ASAuthorizationAppleIDProvider` 是前台接待，负责帮你取号；`ASAuthorizationController` 是业务窗口，负责处理你的请求；`ASAuthorizationAppleIDCredential` 是办好的银行卡，包含你的身份信息。

| 类名 | 职责 | 类比 |
|------|------|------|
| `ASAuthorizationAppleIDProvider` | 创建授权请求 | 银行前台取号 |
| `ASAuthorizationController` | 执行授权流程 | 业务窗口办理 |
| `ASAuthorizationAppleIDCredential` | 持有授权结果 | 办好的银行卡 |

### 3.2 配置 Xcode 项目

在实现代码之前，需要先在 Xcode 中配置：

1. 打开 **Signing & Capabilities** 标签页
2. 点击 **+ Capability**，添加 **Sign in with Apple**
3. 确认 Bundle ID 已正确设置

### 3.3 UIKit 风格完整实现

```swift
import AuthenticationServices

class LoginViewController: UIViewController {

    override func viewDidLoad() {
        super.viewDidLoad()
        setupAppleLoginButton()
    }

    private func setupAppleLoginButton() {
        let button = ASAuthorizationAppleIDButton(
            authorizationButtonType: .signIn,
            authorizationButtonStyle: .black
        )
        button.addTarget(self, action: #selector(handleAppleSignIn), for: .touchUpInside)
        button.translatesAutoresizingMaskIntoConstraints = false
        view.addSubview(button)

        NSLayoutConstraint.activate([
            button.centerXAnchor.constraint(equalTo: view.centerXAnchor),
            button.centerYAnchor.constraint(equalTo: view.centerYAnchor),
            button.widthAnchor.constraint(equalToConstant: 280),
            button.heightAnchor.constraint(equalToConstant: 44)
        ])
    }

    @objc private func handleAppleSignIn() {
        let provider = ASAuthorizationAppleIDProvider()
        let request = provider.createRequest()
        request.requestedScopes = [.fullName, .email]

        let controller = ASAuthorizationController(authorizationRequests: [request])
        controller.delegate = self
        controller.performRequests()
    }
}

extension LoginViewController: ASAuthorizationControllerDelegate {

    func authorizationController(controller: ASAuthorizationController,
                                 didCompleteWithAuthorization authorization: ASAuthorization) {
        guard let credential = authorization.credential as? ASAuthorizationAppleIDCredential else {
            return
        }

        let userIdentifier = credential.user
        let fullName = credential.fullName
        let email = credential.email
        let identityToken = credential.identityToken
        let authorizationCode = credential.authorizationCode

        print("用户ID: \(userIdentifier)")
        print("姓名: \(fullName?.givenName ?? "") \(fullName?.familyName ?? "")")
        print("邮箱: \(email ?? "未提供")")

        if let tokenData = identityToken,
           let tokenString = String(data: tokenData, encoding: .utf8) {
            print("identityToken: \(tokenString)")
        }

        if let codeData = authorizationCode,
           let codeString = String(data: codeData, encoding: .utf8) {
            print("authorizationCode: \(codeString)")
        }
    }

    func authorizationController(controller: ASAuthorizationController,
                                 didCompleteWithError error: Error) {
        print("Apple 登录失败: \(error.localizedDescription)")
    }
}
```

### 3.4 请求的 Scopes 说明

| Scope | 说明 | 仅首次登录提供 |
|-------|------|---------------|
| `.fullName` | 用户全名 | ✅ 是 |
| `.email` | 用户邮箱（可能是代理邮箱） | ✅ 是 |

> ⚠️ **重要**：用户姓名和邮箱**仅在首次授权时返回**，后续登录这些字段为 `nil`。必须在首次获取时立即保存！

---

## 4. SwiftUI 封装 Sign in with Apple

### 4.1 方式一：使用 signInWithAppleButton 修饰符（iOS 16+）

iOS 16 开始，SwiftUI 原生提供了 `signInWithAppleButton` 修饰符，使用非常简洁：

```swift
import SwiftUI
import AuthenticationServices

struct LoginView: View {
    @State private var errorMessage: String?

    var body: some View {
        VStack(spacing: 20) {
            Text("欢迎登录")
                .font(.largeTitle)
                .bold()

            signInWithAppleButton
                .frame(height: 44)
                .padding(.horizontal, 40)

            if let error = errorMessage {
                Text(error)
                    .foregroundStyle(.red)
                    .font(.caption)
            }
        }
    }

    private var signInWithAppleButton: some View {
        SignInWithAppleButton(.signIn) { request in
            request.requestedScopes = [.fullName, .email]
        } onCompletion: { result in
            handleSignInResult(result)
        }
        .signInWithAppleButtonStyle(.black)
    }

    private func handleSignInResult(_ result: Result<ASAuthorization, Error>) {
        switch result {
        case .success(let authorization):
            guard let credential = authorization.credential as? ASAuthorizationAppleIDCredential else {
                return
            }
            processCredential(credential)
        case .failure(let error):
            errorMessage = error.localizedDescription
        }
    }

    private func processCredential(_ credential: ASAuthorizationAppleIDCredential) {
        let userId = credential.user
        let email = credential.email
        let fullName = credential.fullName

        if let tokenData = credential.identityToken,
           let tokenString = String(data: tokenData, encoding: .utf8) {
            sendTokenToBackend(tokenString)
        }
    }

    private func sendTokenToBackend(_ token: String) {
        // 将 token 发送到后端验证
    }
}
```

### 4.2 方式二：使用 UIViewRepresentable 封装（兼容 iOS 14+）

如果需要支持 iOS 14/15，可以通过 `UIViewRepresentable` 封装：

```swift
import SwiftUI
import AuthenticationServices

struct AppleSignInButtonRepresentable: UIViewRepresentable {

    let onSignIn: (Result<ASAuthorizationAppleIDCredential, Error>) -> Void

    func makeUIView(context: Context) -> ASAuthorizationAppleIDButton {
        let button = ASAuthorizationAppleIDButton(
            authorizationButtonType: .signIn,
            authorizationButtonStyle: .black
        )
        button.addTarget(context.coordinator, action: #selector(Coordinator.handleSignIn), for: .touchUpInside)
        return button
    }

    func updateUIView(_ uiView: ASAuthorizationAppleIDButton, context: Context) {}

    func makeCoordinator() -> Coordinator {
        Coordinator(onSignIn: onSignIn)
    }

    class Coordinator: NSObject, ASAuthorizationControllerDelegate {

        let onSignIn: (Result<ASAuthorizationAppleIDCredential, Error>) -> Void

        init(onSignIn: @escaping (Result<ASAuthorizationAppleIDCredential, Error>) -> Void) {
            self.onSignIn = onSignIn
        }

        @objc func handleSignIn() {
            let provider = ASAuthorizationAppleIDProvider()
            let request = provider.createRequest()
            request.requestedScopes = [.fullName, .email]

            let controller = ASAuthorizationController(authorizationRequests: [request])
            controller.delegate = self
            controller.performRequests()
        }

        func authorizationController(controller: ASAuthorizationController,
                                     didCompleteWithAuthorization authorization: ASAuthorization) {
            if let credential = authorization.credential as? ASAuthorizationAppleIDCredential {
                onSignIn(.success(credential))
            }
        }

        func authorizationController(controller: ASAuthorizationController,
                                     didCompleteWithError error: Error) {
            onSignIn(.failure(error))
        }
    }
}
```

使用方式：

```swift
struct LegacyLoginView: View {
    var body: some View {
        AppleSignInButtonRepresentable { result in
            switch result {
            case .success(let credential):
                print("登录成功: \(credential.user)")
            case .failure(let error):
                print("登录失败: \(error)")
            }
        }
        .frame(height: 44)
        .padding(.horizontal, 40)
    }
}
```

### 4.3 两种方式对比

| 对比项 | signInWithAppleButton 修饰符 | UIViewRepresentable 封装 |
|--------|------------------------------|-------------------------|
| 最低系统版本 | iOS 16+ | iOS 14+ |
| 代码量 | 少 | 较多 |
| 样式自定义 | 内置 `.black` / `.white` | 同左 |
| 维护成本 | 低 | 中 |
| 推荐场景 | 新项目，最低支持 iOS 16 | 需兼容旧系统 |

---

## 5. Token 验证与后端对接

### 5.1 登录流程全景

> 💡 **生活类比**：Token 验证就像机场安检——客户端拿到的登机牌（identityToken）不能直接登机，必须经过安检（后端验证）确认是真实有效的，才能放行。

```
┌──────────┐    ① 授权请求     ┌──────────┐    ③ 发送 Token    ┌──────────┐
│  客户端   │ ──────────────→  │  Apple   │                    │  后端     │
│  (App)   │ ←──────────────  │  服务器   │ ──────────────→   │  服务器   │
└──────────┘    ② 返回 Token   └──────────┘                    └──────────┘
                                                                 │
                                                    ④ 向 Apple 验证
                                                                 │
                                                                 ▼
                                                          ┌──────────┐
                                                          │  Apple   │
                                                          │  验证端   │
                                                          └──────────┘
```

### 5.2 关键 Token 字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `identityToken` | JWT (Data) | 用户身份令牌，包含用户信息的 JWT |
| `authorizationCode` | Data | 授权码，后端用于向 Apple 换取 refresh_token |
| `user` | String | 用户唯一标识符，在同一开发者账号下所有 App 中一致 |

### 5.3 JWT 结构解析

identityToken 是一个标准的 JWT，由三部分组成：

```
Header.Payload.Signature
```

Payload 中包含的关键声明：

| 字段 | 说明 |
|------|------|
| `iss` | 签发者，固定为 `https://appleid.apple.com` |
| `aud` | 受众，即你的 Bundle ID |
| `sub` | 用户唯一标识，与 `credential.user` 一致 |
| `email` | 用户邮箱（可能为代理邮箱） |
| `email_verified` | 邮箱是否已验证 |
| `iat` | 签发时间 |
| `exp` | 过期时间 |

### 5.4 后端验证流程（Node.js 示例）

```javascript
const jwt = require("jsonwebtoken");
const jwksClient = require("jwks-rsa");

const client = jwksClient({
    jwksUri: "https://appleid.apple.com/auth/keys"
});

function getApplePublicKey(header, callback) {
    client.getSigningKey(header.kid, (err, key) => {
        const signingKey = key.getPublicKey();
        callback(null, signingKey);
    });
}

function verifyAppleToken(identityToken) {
    return new Promise((resolve, reject) => {
        jwt.verify(identityToken, getApplePublicKey, {
            issuer: "https://appleid.apple.com",
            audience: "com.yourteam.yourapp"
        }, (err, decoded) => {
            if (err) reject(err);
            else resolve(decoded);
        });
    });
}
```

### 5.5 客户端发送 Token 到后端

```swift
import Alamofire

func sendTokenToBackend(credential: ASAuthorizationAppleIDCredential) {
    guard let identityToken = credential.identityToken,
          let tokenString = String(data: identityToken, encoding: .utf8),
          let authCode = credential.authorizationCode,
          let codeString = String(data: authCode, encoding: .utf8) else {
        return
    }

    let parameters: [String: String] = [
        "identity_token": tokenString,
        "authorization_code": codeString,
        "user_id": credential.user
    ]

    AF.request("https://your-api.com/auth/apple",
               method: .post,
               parameters: parameters,
               encoder: JSONParameterEncoder.default)
        .responseDecodable(of: AuthResponse.self) { response in
            switch response.result {
            case .success(let authResponse):
                self.saveSession(authResponse)
            case .failure(let error):
                print("后端验证失败: \(error)")
            }
        }
}
```

> ⚠️ **安全提醒**：永远不要在客户端直接解析 JWT 来做身份验证。JWT 的验证必须在后端完成，客户端只负责传递 Token。

---

## 6. 凭据状态检查与撤销处理

### 6.1 为什么需要检查凭据状态

用户可能在系统设置中撤销了 App 的 Apple ID 登录授权，你的 App 需要感知到这一变化。

> 💡 **生活类比**：就像你挂失了银行卡——银行系统知道卡已失效，但你的钱包里还插着旧卡。App 需要主动去"银行"查询卡的状态。

### 6.2 检查凭据状态

```swift
func checkCredentialState(for userId: String) {
    let provider = ASAuthorizationAppleIDProvider()
    provider.getCredentialState(forUserID: userId) { state, error in
        switch state {
        case .authorized:
            print("凭据有效，用户已授权")
        case .revoked:
            print("凭据已被撤销，需要重新登录")
            handleRevokedCredential()
        case .notFound:
            print("凭据未找到，用户可能已注销")
            handleRevokedCredential()
        case .transferred:
            print("凭据已转移至其他开发者")
            handleTransferredCredential()
        @unknown default:
            break
        }
    }
}
```

### 6.3 监听撤销通知

Apple 会在凭据被撤销时发送系统通知，App 应在启动时注册监听：

```swift
import AuthenticationServices

class AuthManager: ObservableObject {
    @Published var isLoggedIn = false

    private var revokeObserver: Any?

    init() {
        observeCredentialRevocation()
    }

    private func observeCredentialRevocation() {
        revokeObserver = NotificationCenter.default.addObserver(
            forName: ASAuthorizationAppleIDProvider.credentialRevokedNotification,
            object: nil,
            queue: .main
        ) { [weak self] _ in
            self?.handleRevokedCredential()
        }
    }

    private func handleRevokedCredential() {
        DispatchQueue.main.async {
            self.isLoggedIn = false
            self.clearLocalSession()
        }
    }

    private func clearLocalSession() {
        // 清除本地存储的登录状态
    }

    deinit {
        if let observer = revokeObserver {
            NotificationCenter.default.removeObserver(observer)
        }
    }
}
```

### 6.4 凭据状态一览

| 状态 | 含义 | 应对措施 |
|------|------|---------|
| `.authorized` | 用户已授权，凭据有效 | 正常使用 |
| `.revoked` | 用户通过系统设置撤销了授权 | 强制退出登录，清除本地数据 |
| `.notFound` | 凭据不存在 | 视为未登录 |
| `.transferred` | App 已转移至其他开发者账号 | 需重新引导用户登录 |

---

## 7. 第三方登录集成

### 7.1 微信登录

#### 配置步骤

1. 在[微信开放平台](https://open.weixin.qq.com)注册 App，获取 AppID
2. 配置 URL Scheme：`wx{你的AppID}`
3. 在 `Info.plist` 添加 `LSApplicationQueriesSchemes`：`weixin`、`weixinULAPI`

#### 核心代码

```swift
import UIKit

class WeChatAuthService: NSObject, WXApiDelegate {

    static let shared = WeChatAuthService()
    var onAuthComplete: ((String?, Error?) -> Void)?

    func sendAuthRequest() {
        let req = SendAuthReq()
        req.scope = "snsapi_userinfo"
        req.state = "wechat_login_\(UUID().uuidString)"
        WXApi.send(req) { [weak self] success in
            if !success {
                self?.onAuthComplete?(nil, NSError(domain: "WeChat", code: -1,
                    userInfo: [NSLocalizedDescriptionKey: "微信未安装或请求发送失败"]))
            }
        }
    }

    func onReq(_ req: BaseReq) {}

    func onResp(_ resp: BaseResp) {
        guard let authResp = resp as? SendAuthResp else { return }

        if authResp.errCode == 0, let code = authResp.code {
            sendCodeToBackend(code)
        } else {
            onAuthComplete?(nil, NSError(domain: "WeChat", code: Int(authResp.errCode),
                userInfo: [NSLocalizedDescriptionKey: "微信授权失败"]))
        }
    }

    private func sendCodeToBackend(_ code: String) {
        // 将 code 发送到后端，后端用 code 换取 access_token 和 openid
    }
}
```

在 `AppDelegate` 中处理回调：

```swift
func application(_ app: UIApplication, open url: URL,
                 options: [UIApplication.OpenURLOptionsKey: Any] = [:]) -> Bool {
    if WXApi.handleOpen(url, delegate: WeChatAuthService.shared) {
        return true
    }
    return false
}
```

### 7.2 Google Sign-In

#### 配置步骤

1. 在 [Google Cloud Console](https://console.cloud.google.com) 创建 OAuth 客户端
2. 下载 `GoogleService-Info.plist` 加入项目
3. 添加 URL Scheme：反转的 Client ID

#### 核心代码

```swift
import GoogleSignIn

struct GoogleAuthService {

    static func signIn(presenting viewController: UIViewController) {
        guard let clientID = FirebaseApp.app()?.options.clientID else { return }

        let config = GIDConfiguration(clientID: clientID)
        GIDSignIn.sharedInstance.configuration = config

        GIDSignIn.sharedInstance.signIn(withPresenting: viewController) { result, error in
            if let error = error {
                print("Google 登录失败: \(error)")
                return
            }

            guard let user = result?.user,
                  let idToken = user.idToken?.tokenString else {
                return
            }

            let accessToken = user.accessToken.tokenString
            sendGoogleTokenToBackend(idToken: idToken, accessToken: accessToken)
        }
    }

    private static func sendGoogleTokenToBackend(idToken: String, accessToken: String) {
        // 发送到后端验证
    }
}
```

### 7.3 GitHub OAuth

GitHub OAuth 基于 Web 流程，需要通过 `ASWebAuthenticationSession` 实现：

```swift
import AuthenticationServices

class GitHubAuthService: NSObject, ASWebAuthenticationControllerDelegate {

    private let clientId = "your_github_client_id"
    private let clientSecret = "your_github_client_secret"
    private let redirectURI = "yourapp://github-callback"
    private let scope = "read:user user:email"

    func signIn(presentingVC: UIViewController) {
        var components = URLComponents(string: "https://github.com/login/oauth/authorize")!
        components.queryItems = [
            URLQueryItem(name: "client_id", value: clientId),
            URLQueryItem(name: "redirect_uri", value: redirectURI),
            URLQueryItem(name: "scope", value: scope),
            URLQueryItem(name: "state", value: UUID().uuidString)
        ]

        guard let authURL = components.url else { return }

        let session = ASWebAuthenticationSession(
            url: authURL,
            callbackURLScheme: "yourapp"
        ) { [weak self] callbackURL, error in
            if let error = error {
                print("GitHub 登录失败: \(error)")
                return
            }
            guard let callbackURL = callbackURL,
                  let code = URLComponents(url: callbackURL, resolvingAgainstBaseURL: false)?
                    .queryItems?.first(where: { $0.name == "code" })?.value else {
                return
            }
            self?.exchangeCodeForToken(code)
        }

        session.presentationContextProvider = self
        session.start()
    }

    private func exchangeCodeForToken(_ code: String) {
        // 将 code 发送到后端，后端用 code + client_secret 换取 access_token
    }
}

extension GitHubAuthService: ASWebAuthenticationPresentationContextProviding {
    func presentationAnchor(for session: ASWebAuthenticationSession) -> ASPresentationAnchor {
        UIApplication.shared.windows.first { $0.isKeyWindow } ?? ASPresentationAnchor()
    }
}
```

### 7.4 三种第三方登录对比

| 对比项 | 微信登录 | Google Sign-In | GitHub OAuth |
|--------|---------|---------------|--------------|
| 主要用户群 | 中国大陆用户 | 全球用户 | 开发者群体 |
| SDK 集成 | 需集成微信 SDK | 需集成 Google SDK | 无需 SDK，Web 流程 |
| 回调方式 | URL Scheme | URL Scheme | ASWebAuthenticationSession |
| 获取用户信息 | 后端用 code 换 token | 客户端直接获取 | 后端用 code 换 token |
| 审核注意事项 | 需处理未安装微信的情况 | 需处理未安装 Google 的情况 | 无特殊要求 |

---

## 8. 多登录方式统一管理

### 8.1 统一认证模型

> 💡 **生活类比**：就像一个人的身份可以用身份证、护照、驾照来证明——不同证件对应不同场景，但都指向同一个人。多登录方式管理就是建立"证件"与"人"的映射关系。

```swift
enum AuthProvider: String, Codable, CaseIterable {
    case apple
    case wechat
    case google
    case github
    case phone
    case email

    var displayName: String {
        switch self {
        case .apple:   return "Apple"
        case .wechat:  return "微信"
        case .google:  return "Google"
        case .github:  return "GitHub"
        case .phone:   return "手机号"
        case .email:   return "邮箱"
        }
    }

    var iconName: String {
        switch self {
        case .apple:   return "apple.logo"
        case .wechat:  return "message.fill"
        case .google:  return "globe"
        case .github:  return "chevron.left.forwardslash.chevron.right"
        case .phone:   return "phone.fill"
        case .email:   return "envelope.fill"
        }
    }
}

struct AuthCredential: Codable {
    let provider: AuthProvider
    let providerUserId: String
    let accessToken: String?
    let refreshToken: String?
    let linkedAt: Date
}

struct UserProfile: Codable {
    let id: String
    var name: String
    var email: String?
    var avatarURL: String?
    var linkedProviders: [AuthCredential]
}
```

### 8.2 统一认证管理器

```swift
import Combine

class AuthManager: ObservableObject {
    static let shared = AuthManager()

    @Published var currentUser: UserProfile?
    @Published var isAuthenticated: Bool = false

    private var cancellables = Set<AnyCancellable>()

    func login(with provider: AuthProvider, credential: AuthCredential) {
        if let existingUser = currentUser {
            linkProvider(credential, to: existingUser)
        } else {
            fetchOrCreateUser(credential: credential) { [weak self] user in
                DispatchQueue.main.async {
                    self?.currentUser = user
                    self?.isAuthenticated = true
                    self?.saveToKeychain(user)
                }
            }
        }
    }

    func linkProvider(_ credential: AuthCredential, to user: UserProfile) {
        var updatedUser = user
        if !updatedUser.linkedProviders.contains(where: { $0.provider == credential.provider }) {
            updatedUser.linkedProviders.append(credential)
        }
        currentUser = updatedUser
        saveToKeychain(updatedUser)
        sendLinkedProviderToBackend(credential)
    }

    func unlink(provider: AuthProvider) {
        guard var user = currentUser else { return }
        user.linkedProviders.removeAll { $0.provider == provider }
        if user.linkedProviders.isEmpty {
            logout()
        } else {
            currentUser = user
            saveToKeychain(user)
        }
    }

    func logout() {
        currentUser = nil
        isAuthenticated = false
        clearKeychain()
    }

    private func fetchOrCreateUser(credential: AuthCredential, completion: @escaping (UserProfile) -> Void) {
        // 后端接口：根据 provider + providerUserId 查找或创建用户
    }

    private func sendLinkedProviderToBackend(_ credential: AuthCredential) {
        // 通知后端关联新的登录方式
    }
}
```

### 8.3 统一登录界面

```swift
struct UnifiedLoginView: View {
    @StateObject private var authManager = AuthManager.shared
    @State private var showPhoneLogin = false

    var body: some View {
        VStack(spacing: 24) {
            Text("选择登录方式")
                .font(.title2)
                .bold()

            SignInWithAppleButton(.signIn) { request in
                request.requestedScopes = [.fullName, .email]
            } onCompletion: { result in
                handleAppleResult(result)
            }
            .signInWithAppleButtonStyle(.black)
            .frame(height: 50)

            SocialLoginButton(provider: .wechat) {
                WeChatAuthService.shared.sendAuthRequest()
            }

            SocialLoginButton(provider: .google) {
                // Google 登录
            }

            SocialLoginButton(provider: .github) {
                // GitHub 登录
            }

            Divider()

            SocialLoginButton(provider: .phone) {
                showPhoneLogin = true
            }
        }
        .padding(.horizontal, 32)
        .sheet(isPresented: $showPhoneLogin) {
            PhoneLoginView()
        }
    }

    private func handleAppleResult(_ result: Result<ASAuthorization, Error>) {
        // 处理 Apple 登录结果
    }
}

struct SocialLoginButton: View {
    let provider: AuthProvider
    let action: () -> Void

    var body: some View {
        Button(action: action) {
            HStack {
                Image(systemName: provider.iconName)
                Text("通过\(provider.displayName)登录")
                    .font(.headline)
            }
            .frame(maxWidth: .infinity)
            .frame(height: 50)
            .foregroundStyle(.white)
            .background(Color.accentColor)
            .clipShape(RoundedRectangle(cornerRadius: 12))
        }
    }
}
```

---

## 9. Keychain 安全存储登录凭据

### 9.1 为什么用 Keychain

> 💡 **生活类比**：UserDefaults 就像把贵重物品放在抽屉里——谁都能打开；Keychain 就像银行保险柜——有系统级加密保护，只有你的 App 能访问。

| 存储方式 | 安全级别 | 数据持久化 | 卸载后保留 | 适用数据 |
|----------|---------|-----------|-----------|---------|
| UserDefaults | ❌ 低 | ✅ 是 | ❌ 否 | 偏好设置 |
| Keychain | ✅ 高 | ✅ 是 | ✅ 是（iOS） | 密码、Token |
| 文件存储 | ❌ 低 | ✅ 是 | ❌ 否 | 缓存数据 |
| SwiftData | ❌ 低 | ✅ 是 | ❌ 否 | 结构化业务数据 |

### 9.2 Keychain 封装

```swift
import Security

enum KeychainError: LocalizedError {
    case duplicateEntry
    case entryNotFound
    case unexpectedStatus(OSStatus)

    var errorDescription: String? {
        switch self {
        case .duplicateEntry:    return "Keychain 中已存在该条目"
        case .entryNotFound:     return "Keychain 中未找到该条目"
        case .unexpectedStatus(let s): return "Keychain 操作失败，状态码: \(s)"
        }
    }
}

enum KeychainService {
    private static let service = Bundle.main.bundleIdentifier ?? "com.app.auth"

    static func save(data: Data, for key: String) throws {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: key
        ]

        let status = SecItemDelete(query as CFDictionary)
        if status != errSecSuccess && status != errSecItemNotFound {
            throw KeychainError.unexpectedStatus(status)
        }

        var attributes = query
        attributes[kSecValueData as String] = data
        attributes[kSecAttrAccessible as String] = kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly

        let addStatus = SecItemAdd(attributes as CFDictionary, nil)
        if addStatus != errSecSuccess {
            throw KeychainError.unexpectedStatus(addStatus)
        }
    }

    static func load(for key: String) throws -> Data {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: key,
            kSecReturnData as String: true,
            kSecMatchLimit as String: kSecMatchLimitOne
        ]

        var result: AnyObject?
        let status = SecItemCopyMatching(query as CFDictionary, &result)

        if status == errSecItemNotFound {
            throw KeychainError.entryNotFound
        }
        if status != errSecSuccess {
            throw KeychainError.unexpectedStatus(status)
        }

        return result as! Data
    }

    static func delete(for key: String) throws {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: key
        ]

        let status = SecItemDelete(query as CFDictionary)
        if status != errSecSuccess && status != errSecItemNotFound {
            throw KeychainError.unexpectedStatus(status)
        }
    }
}
```

### 9.3 在 AuthManager 中使用 Keychain

```swift
extension AuthManager {

    func saveToKeychain(_ profile: UserProfile) {
        do {
            let data = try JSONEncoder().encode(profile)
            try KeychainService.save(data: data, for: "currentUserProfile")
        } catch {
            print("Keychain 保存失败: \(error)")
        }
    }

    func loadFromKeychain() -> UserProfile? {
        do {
            let data = try KeychainService.load(for: "currentUserProfile")
            return try JSONDecoder().decode(UserProfile.self, from: data)
        } catch {
            return nil
        }
    }

    func clearKeychain() {
        try? KeychainService.delete(for: "currentUserProfile")
    }
}
```

> ⚠️ **注意**：使用 `kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly` 表示数据在设备首次解锁后可访问，且不会通过 iCloud Keychain 同步到其他设备。对于敏感的登录凭据，这是推荐的安全级别。

---

## 10. 常见审核被拒原因与应对

### 10.1 登录相关被拒原因汇总

| 被拒原因 | 审核指南条款 | 应对措施 |
|----------|------------|---------|
| 有第三方登录但未提供 Sign in with Apple | 4.8 | 添加 Sign in with Apple 按钮 |
| 强制登录才能使用基本功能 | 2.1 | 允许未登录状态下使用核心功能 |
| Sign in with Apple 按钮位置不显眼 | 4.8 | Apple 登录按钮应与其他第三方登录按钮同等醒目 |
| 未处理凭据撤销 | 2.1 | 实现 credentialRevoked 通知监听 |
| 收集不必要的用户信息 | 5.1.1 | 仅请求必要的 Scopes（姓名、邮箱） |
| 隐私政策未说明登录数据使用 | 5.1.1 | 在隐私政策中明确说明登录数据收集与使用 |
| 微信/Google 登录在审核员设备上不可用 | 2.1 | 对未安装第三方 App 的情况做优雅降级 |
| 未提供账号删除功能 | 5.1.1(v) | 提供账号注销入口，并在 30 天内完成删除 |

### 10.2 Sign in with Apple 按钮规范

Apple 对 Sign in with Apple 按钮有严格的视觉规范要求：

```swift
// ✅ 正确：使用系统提供的按钮样式
SignInWithAppleButton(.signIn)
    .signInWithAppleButtonStyle(.black) // 或 .white

// ❌ 错误：自定义 Apple 登录按钮外观
// Apple 不允许修改按钮的颜色、圆角、字体等视觉属性
```

> ⚠️ **按钮位置要求**：Sign in with Apple 按钮应放在与其他第三方登录按钮同等显眼的位置，不能隐藏在折叠区域或次要页面中。

### 10.3 账号删除功能

自 2022 年起，Apple 要求所有支持账号创建的 App 必须提供账号删除功能：

```swift
struct AccountDeletionView: View {
    @State private var showDeleteConfirmation = false

    var body: some View {
        VStack(spacing: 16) {
            Text("删除账号")
                .font(.title2)
                .bold()

            Text("删除账号后，所有数据将被永久清除且无法恢复。")
                .foregroundStyle(.secondary)
                .multilineTextAlignment(.center)

            Button("删除我的账号", role: .destructive) {
                showDeleteConfirmation = true
            }
            .buttonStyle(.borderedProminent)
            .tint(.red)
        }
        .padding()
        .alert("确认删除", isPresented: $showDeleteConfirmation) {
            Button("取消", role: .cancel) {}
            Button("确认删除", role: .destructive) {
                requestAccountDeletion()
            }
        } message: {
            Text("此操作不可撤销，所有数据将被永久删除。")
        }
    }

    private func requestAccountDeletion() {
        // 调用后端 API 删除账号
        // 同时撤销 Sign in with Apple 的凭据
    }
}
```

### 10.4 审核自查清单

在提交审核前，对照以下清单逐项检查：

- [ ] 如有第三方社交登录，是否提供了 Sign in with Apple？
- [ ] Sign in with Apple 按钮是否与其他登录按钮同等醒目？
- [ ] 是否处理了凭据撤销（credentialRevoked）通知？
- [ ] 未登录状态下是否可以使用基本功能？
- [ ] 是否提供了账号删除功能？
- [ ] 隐私政策是否说明了登录数据的收集与使用？
- [ ] 第三方登录在未安装对应 App 时是否有降级处理？
- [ ] 首次登录获取的姓名和邮箱是否已保存？
- [ ] Token 是否通过后端验证而非客户端直接解析？

---

## 本章小结

| 主题 | 核心要点 |
|------|---------|
| 登录必要性 | 用户身份、数据同步、个性化三大价值 |
| Apple 审核要求 | 有第三方社交登录就必须提供 Sign in with Apple |
| 核心类 | ASAuthorizationAppleIDProvider → ASAuthorizationController → ASAuthorizationAppleIDCredential |
| SwiftUI 集成 | iOS 16+ 用 `signInWithAppleButton` 修饰符，低版本用 UIViewRepresentable |
| Token 验证 | identityToken 是 JWT，必须由后端验证，客户端不可信 |
| 凭据状态 | 监听 credentialRevoked 通知，启动时调用 getCredentialState |
| 第三方登录 | 微信（SDK）、Google（SDK）、GitHub（ASWebAuthenticationSession） |
| 统一管理 | 建立统一认证模型，支持多登录方式关联同一账号 |
| Keychain 存储 | 使用 Keychain 而非 UserDefaults 存储敏感凭据 |
| 审核避坑 | Apple 登录按钮规范、账号删除功能、凭据撤销处理 |

> 💡 **最佳实践**：登录是用户进入 App 的第一道门，体验好坏直接影响留存率。遵循 Apple 规范、做好安全存储、优雅处理异常状态，才能让用户"进得来、留得住、走得安心"。

← [-UIKit 与 SwiftUI 互操作](./60-UIKit与SwiftUI互操作.md) | [-后台任务与多任务](./62-后台任务与多任务.md) →
