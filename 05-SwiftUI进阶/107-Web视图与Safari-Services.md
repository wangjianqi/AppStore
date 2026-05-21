# 107-Web 视图与 Safari Services

> 🎯 **本章目标**：
> - 理解 iOS 中展示网页内容的三种主要方式及其适用场景
> - 掌握 SFSafariViewController 的使用与 SwiftUI 封装
> - 深入理解 WKWebView 的配置、导航与 UI 代理
> - 学会 WKWebView 与 JavaScript 的双向交互及 JS Bridge 设计
> - 掌握 Cookie 管理与持久化策略
> - 能够使用 ASWebAuthenticationSession 完成 OAuth 授权流程
> - 实战封装 SwiftUI WebView 组件（含进度条、导航控制、URL 拦截）
> - 规避 App Store 审核中与 Web 视图相关的常见被拒原因

---

## 1. iOS 中展示网页的三种方式对比

> 💡 **生活类比**：在手机上看网页就像"出门办事"——SFSafariViewController 是坐"公司班车"（安全、方便、但路线固定），WKWebView 是开"私家车"（灵活、可控、但需要自己驾驶），ASWebAuthenticationSession 是坐"出租车"（专门跑一趟授权、跑完就走）。

### 1.1 三种方式总览

| 特性 | SFSafariViewController | WKWebView | ASWebAuthenticationSession |
|------|----------------------|-----------|---------------------------|
| **所属框架** | SafariServices | WebKit | AuthenticationServices |
| **核心用途** | 展示网页内容 | 嵌入式自定义浏览器 | OAuth/第三方授权登录 |
| **Cookie 共享** | 与 Safari 共享 | 独立进程 | 与 Safari 共享 |
| **自定义程度** | 极低（只改色调） | 完全自定义 | 极低（系统 UI） |
| **JS 交互** | 不支持 | 完整支持 | 不支持 |
| **自动填充** | 支持（Safari 密码） | 不支持 | 不支持 |
| **Content Blocker** | 支持 | 需手动实现 | 不适用 |
| **隐私浏览** | 不支持 | 可自行实现 | 不适用 |
| **最低系统版本** | iOS 9+ | iOS 8+ | iOS 12+ |
| **App Store 审核** | 风险低 | 需注意 4.2 条款 | 风险低 |

### 1.2 如何选择

```
需要展示网页？
    │
    ├─ 仅展示内容，不需要 JS 交互
    │   └─ ✅ SFSafariViewController
    │
    ├─ 需要与网页 JS 交互 / 自定义浏览器
    │   └─ ✅ WKWebView
    │
    └─ 需要 OAuth 授权 / 第三方登录
        └─ ✅ ASWebAuthenticationSession
```

> ⚠️ **警告**：不要用 WKWebView 做简单的网页展示——SFSafariViewController 自带 Safari 的安全特性（防钓鱼、密码自动填充等），是更安全的选择。

---

## 2. SFSafariViewController 详解

> 💡 **生活类比**：SFSafariViewController 就像在商场里的"品牌专柜"——虽然开在商场（你的 App）里，但店员和运营都是品牌方（Safari）的人，你只能决定专柜的装修色调，不能换店员。

### 2.1 基本使用

```swift
import SafariServices

class ViewController: UIViewController {
    func openWebPage() {
        let url = URL(string: "https://developer.apple.com")!
        let config = SFSafariViewController.Configuration()
        config.entersReaderIfAvailable = true
        config.barCollapsingEnabled = true

        let safariVC = SFSafariViewController(url: url, configuration: config)
        safariVC.preferredControlTintColor = .systemBlue
        safariVC.preferredBarTintColor = .systemBackground
        safariVC.dismissButtonStyle = .close
        safariVC.delegate = self

        present(safariVC, animated: true)
    }
}

extension ViewController: SFSafariViewControllerDelegate {
    func safariViewControllerDidFinish(_ controller: SFSafariViewController) {
        print("用户点击了关闭按钮")
    }

    func safariViewController(
        _ controller: SFSafariViewController,
        didCompleteInitialLoad didLoadSuccessfully: Bool
    ) {
        print("初始加载\(didLoadSuccessfully ? "成功" : "失败")")
    }
}
```

### 2.2 SwiftUI 封装（UIViewControllerRepresentable）

```swift
import SwiftUI
import SafariServices

struct SafariView: UIViewControllerRepresentable {
    let url: URL
    var entersReaderIfAvailable: Bool = false
    var dismissButtonStyle: SFSafariViewController.DismissButtonStyle = .done

    func makeUIViewController(
        context: Context
    ) -> SFSafariViewController {
        let config = SFSafariViewController.Configuration()
        config.entersReaderIfAvailable = entersReaderIfAvailable
        config.barCollapsingEnabled = true

        let safariVC = SFSafariViewController(url: url, configuration: config)
        safariVC.dismissButtonStyle = dismissButtonStyle
        safariVC.delegate = context.coordinator
        return safariVC
    }

    func updateUIViewController(
        _ uiViewController: SFSafariViewController,
        context: Context
    ) {}

    func makeCoordinator() -> Coordinator {
        Coordinator()
    }

    class Coordinator: NSObject, SFSafariViewControllerDelegate {
        func safariViewControllerDidFinish(
            _ controller: SFSafariViewController
        ) {
            print("Safari 关闭")
        }
    }
}
```

在 SwiftUI 中使用：

```swift
struct ContentView: View {
    @State private var showSafari = false

    var body: some View {
        Button("打开网页") {
            showSafari = true
        }
        .sheet(isPresented: $showSafari) {
            SafariView(
                url: URL(string: "https://developer.apple.com")!,
                dismissButtonStyle: .close
            )
        }
    }
}
```

### 2.3 SFSafariViewController 与 Safari 的区别

| 特性 | Safari App | SFSafariViewController |
|------|-----------|----------------------|
| **地址栏编辑** | ✅ 可输入 URL | ❌ 只读展示 |
| **标签页管理** | ✅ 多标签 | ❌ 单页面 |
| **下载文件** | ✅ 支持 | ❌ 不支持 |
| **添加书签** | ✅ 支持 | ❌ 不支持 |
| **分享功能** | ✅ 完整分享菜单 | ✅ 有限分享 |
| **密码自动填充** | ✅ 支持 | ✅ 支持 |
| **Cookie** | 与 Safari 共享 | 与 Safari 共享 |
| **关闭方式** | 切换 App | 关闭按钮返回 App |

> 💡 **提示**：SFSafariViewController 的 Cookie 与 Safari 共享，这意味着用户在 Safari 中登录过的网站，在 SFSafariViewController 中也是登录状态——这对用户体验非常友好。

---

## 3. WKWebView 详解

> 💡 **生活类比**：WKWebView 就像一块"空白画布"——你可以自由决定画什么、怎么画，但也意味着你需要自己处理所有细节（安全、导航、弹窗等）。

### 3.1 创建与配置

```swift
import WebKit

class WebViewController: UIViewController {
    var webView: WKWebView!

    override func viewDidLoad() {
        super.viewDidLoad()

        let config = WKWebViewConfiguration()
        config.preferences.javaScriptEnabled = true
        config.preferences.javaScriptCanOpenWindowsAutomatically = false
        config.defaultWebpagePreferences.allowsContentJavaScript = true

        webView = WKWebView(frame: view.bounds, configuration: config)
        webView.autoresizingMask = [.flexibleWidth, .flexibleHeight]
        webView.navigationDelegate = self
        webView.uiDelegate = self
        webView.allowsBackForwardNavigationGestures = true

        view.addSubview(webView)

        if let url = URL(string: "https://www.example.com") {
            webView.load(URLRequest(url: url))
        }
    }
}
```

### 3.2 WKWebViewConfiguration 详解

```swift
let config = WKWebViewConfiguration()

config.processPool = WKProcessPool()
config.websiteDataStore = WKWebsiteDataStore.default()

config.preferences.javaScriptEnabled = true
config.preferences.javaScriptCanOpenWindowsAutomatically = false

if #available(iOS 16.4, *) {
    config.preferences.elementFullscreenEnabled = true
}

config.allowsInlineMediaPlayback = true
config.mediaTypesRequiringUserActionForPlayback = .all
config.suppressesIncrementalRendering = false

config.applicationNameForUserAgent = "MyApp/1.0"
```

| 配置项 | 说明 | 推荐值 |
|--------|------|--------|
| `javaScriptEnabled` | 是否启用 JS | `true` |
| `javaScriptCanOpenWindowsAutomatically` | JS 是否可自动开窗口 | `false` |
| `allowsInlineMediaPlayback` | 内联播放视频 | `true` |
| `mediaTypesRequiringUserActionForPlayback` | 需要用户操作才能播放 | `.all` |
| `suppressesIncrementalRendering` | 抑制增量渲染 | `false` |
| `websiteDataStore` | 数据存储 | `.default()` |

### 3.3 WKNavigationDelegate

```swift
extension WebViewController: WKNavigationDelegate {
    func webView(
        _ webView: WKWebView,
        decidePolicyFor navigationAction: WKNavigationAction,
        decisionHandler: @escaping (WKNavigationActionPolicy) -> Void
    ) {
        guard let url = navigationAction.request.url else {
            decisionHandler(.allow)
            return
        }

        if url.scheme == "tel" || url.scheme == "mailto" {
            UIApplication.shared.open(url)
            decisionHandler(.cancel)
            return
        }

        decisionHandler(.allow)
    }

    func webView(
        _ webView: WKWebView,
        decidePolicyFor navigationResponse: WKNavigationResponse,
        decisionHandler: @escaping (WKNavigationResponsePolicy) -> Void
    ) {
        decisionHandler(.allow)
    }

    func webView(_ webView: WKWebView, didStartProvisionalNavigation navigation: WKNavigation!) {
        print("开始加载")
    }

    func webView(_ webView: WKWebView, didFinish navigation: WKNavigation!) {
        print("加载完成")
    }

    func webView(
        _ webView: WKWebView,
        didFail navigation: WKNavigation!,
        withError error: Error
    ) {
        print("加载失败: \(error.localizedDescription)")
    }

    func webView(
        _ webView: WKWebView,
        didReceive challenge: URLAuthenticationChallenge,
        completionHandler: @escaping (URLSession.AuthChallengeDisposition, URLCredential?) -> Void
    ) {
        completionHandler(.performDefaultHandling, nil)
    }
}
```

### 3.4 WKUIDelegate

```swift
extension WebViewController: WKUIDelegate {
    func webView(
        _ webView: WKWebView,
        runJavaScriptAlertPanelWithMessage message: String,
        initiatedByFrame frame: WKFrameInfo,
        completionHandler: @escaping () -> Void
    ) {
        let alert = UIAlertController(
            title: nil, message: message, preferredStyle: .alert
        )
        alert.addAction(UIAlertAction(title: "确定", style: .default) { _ in
            completionHandler()
        })
        present(alert, animated: true)
    }

    func webView(
        _ webView: WKWebView,
        runJavaScriptConfirmPanelWithMessage message: String,
        initiatedByFrame frame: WKFrameInfo,
        completionHandler: @escaping (Bool) -> Void
    ) {
        let alert = UIAlertController(
            title: nil, message: message, preferredStyle: .alert
        )
        alert.addAction(UIAlertAction(title: "取消", style: .cancel) { _ in
            completionHandler(false)
        })
        alert.addAction(UIAlertAction(title: "确定", style: .default) { _ in
            completionHandler(true)
        })
        present(alert, animated: true)
    }

    func webView(
        _ webView: WKWebView,
        createWebViewWith configuration: WKWebViewConfiguration,
        for navigationAction: WKNavigationAction,
        windowFeatures: WKWindowFeatures
    ) -> WKWebView? {
        if navigationAction.targetFrame == nil {
            webView.load(navigationAction.request)
        }
        return nil
    }
}
```

> ⚠️ **警告**：WKWebView 中的 `alert()` / `confirm()` / `prompt()` 不会自动弹出，必须通过 WKUIDelegate 手动实现，否则这些调用会被静默忽略。

---

## 4. WKWebView 与 JavaScript 交互

> 💡 **生活类比**：Swift 和 JavaScript 就像两个说不同语言的人——`evaluateJavaScript` 是 Swift 给 JS"递纸条"，`WKScriptMessageHandler` 是 JS 给 Swift"递纸条"，JS Bridge 就是两个人约定好的"翻译规则"。

### 4.1 Swift 调用 JavaScript（evaluateJavaScript）

```swift
webView.evaluateJavaScript("document.title") { result, error in
    if let title = result as? String {
        print("页面标题: \(title)")
    }
}

webView.evaluateJavaScript("document.body.style.backgroundColor = '#f0f0f0'")

let jsCode = """
function getMetaContent(name) {
    var meta = document.querySelector('meta[name="' + name + '"]');
    return meta ? meta.content : null;
}
getMetaContent('description');
"""
webView.evaluateJavaScript(jsCode) { result, error in
    if let description = result as? String {
        print("页面描述: \(description)")
    }
}
```

### 4.2 JavaScript 调用 Swift（WKScriptMessageHandler）

```swift
class WebViewController: UIViewController {
    var webView: WKWebView!

    override func viewDidLoad() {
        super.viewDidLoad()

        let config = WKWebViewConfiguration()
        let contentController = config.userContentController
        contentController.add(self, name: "nativeBridge")

        let userScript = WKUserScript(
            source: "window.nativeBridge = window.webkit.messageHandlers.nativeBridge;",
            injectionTime: .atDocumentStart,
            forMainFrameOnly: true
        )
        contentController.addUserScript(userScript)

        webView = WKWebView(frame: view.bounds, configuration: config)
        view.addSubview(webView)
    }
}

extension WebViewController: WKScriptMessageHandler {
    func userContentController(
        _ userContentController: WKUserContentController,
        didReceive message: WKScriptMessage
    ) {
        guard message.name == "nativeBridge" else { return }

        if let body = message.body as? [String: Any] {
            let action = body["action"] as? String
            let data = body["data"]
            handleJSMessage(action: action, data: data)
        }
    }

    func handleJSMessage(action: String?, data: Any?) {
        switch action {
        case "share":
            handleShare(data: data)
        case "openCamera":
            handleOpenCamera()
        case "getToken":
            handleGetToken()
        default:
            print("未知 action: \(action ?? "nil")")
        }
    }

    func handleGetToken() {
        let token = "user_token_123"
        let js = "window.onTokenReceived('\(token)')"
        webView.evaluateJavaScript(js)
    }
}
```

### 4.3 JS Bridge 设计模式

> 💡 **生活类比**：JS Bridge 就像"快递站"——JS 端寄件（发消息），Swift 端收件（处理消息），处理完还可以回寄（回调结果）。

```javascript
// 前端 JS 侧的 Bridge 封装
class NativeBridge {
    static call(action, data = {}, callback = null) {
        const messageId = Date.now().toString();
        if (callback) {
            window._bridgeCallbacks = window._bridgeCallbacks || {};
            window._bridgeCallbacks[messageId] = callback;
        }
        window.webkit.messageHandlers.nativeBridge.postMessage({
            action: action,
            data: data,
            messageId: messageId
        });
    }

    static callback(messageId, result) {
        if (window._bridgeCallbacks && window._bridgeCallbacks[messageId]) {
            window._bridgeCallbacks[messageId](result);
            delete window._bridgeCallbacks[messageId];
        }
    }
}

// 使用示例
NativeBridge.call('getToken', {}, (token) => {
    console.log('收到 Token:', token);
});

NativeBridge.call('share', { title: '分享标题', url: 'https://example.com' });
```

Swift 侧回调处理：

```swift
func handleJSMessage(action: String?, data: Any?, messageId: String?) {
    switch action {
    case "getToken":
        let token = KeychainHelper.getToken() ?? ""
        callbackToJS(messageId: messageId, result: ["token": token])
    case "share":
        handleShare(data: data)
        callbackToJS(messageId: messageId, result: ["success": true])
    default:
        break
    }
}

func callbackToJS(messageId: String?, result: [String: Any]) {
    guard let messageId = messageId,
          let jsonData = try? JSONSerialization.data(withJSONObject: result),
          let jsonString = String(data: jsonData, encoding: .utf8)
    else { return }

    let js = "NativeBridge.callback('\(messageId)', \(jsonString))"
    webView.evaluateJavaScript(js)
}
```

### 4.4 WKUserContentController 注入脚本

```swift
let contentController = WKUserContentController()

let darkModeScript = WKUserScript(
    source: """
    var style = document.createElement('style');
    style.textContent = `
        body { background-color: #1a1a1a !important; color: #e0e0e0 !important; }
        a { color: #64b5f6 !important; }
    `;
    document.head.appendChild(style);
    """,
    injectionTime: .atDocumentEnd,
    forMainFrameOnly: true
)
contentController.addUserScript(darkModeScript)
```

| 注入时机 | 说明 | 适用场景 |
|----------|------|---------|
| `atDocumentStart` | 文档开始加载时 | 注入全局变量、Bridge 定义 |
| `atDocumentEnd` | 文档加载完成后 | 操作 DOM、修改样式 |
| `atDocumentIdle` | 文档空闲时 | 低优先级脚本（iOS 16+） |

> ⚠️ **警告**：`addUserScript` 添加的脚本在页面导航后不会自动移除，但脚本内容不会重新注入。如果需要每次导航都注入，应在 `didFinish` 回调中手动执行。

---

## 5. Cookie 管理与持久化

> 💡 **生活类比**：Cookie 就像商场的"会员卡"——你第一次办卡后，每次来商场刷卡就能识别身份。但不同商场（WKWebView vs URLSession）的卡不通用，需要想办法"互认"。

### 5.1 WKHTTPCookieStore

```swift
let cookieStore = WKWebsiteDataStore.default().httpCookieStore

cookieStore.getAllCookies { cookies in
    for cookie in cookies {
        print("Cookie: \(cookie.name) = \(cookie.value)")
    }
}

let cookie = HTTPCookie(properties: [
    .domain: ".example.com",
    .path: "/",
    .name: "session_id",
    .value: "abc123",
    .secure: "TRUE",
    .expires: Date().addingTimeInterval(3600 * 24 * 30)
])!

cookieStore.setCookie(cookie) {
    print("Cookie 设置完成")
}
```

### 5.2 与 URLSession Cookie 共享

WKWebView 和 URLSession 使用不同的 Cookie 存储，需要手动同步：

```swift
class CookieSyncManager {
    static let shared = CookieSyncManager()

    func syncCookiesFromURLSession(to webView: WKWebView) {
        let urlSessionCookies = HTTPCookieStorage.shared.cookies ?? []
        let cookieStore = webView.configuration.websiteDataStore.httpCookieStore

        for cookie in urlSessionCookies {
            cookieStore.setCookie(cookie)
        }
    }

    func syncCookiesFromWKWebView(
        _ webView: WKWebView,
        to urlSession: URLSession
    ) {
        let cookieStore = webView.configuration.websiteDataStore.httpCookieStore
        cookieStore.getAllCookies { cookies in
            for cookie in cookies {
                HTTPCookieStorage.shared.setCookie(cookie)
            }
        }
    }

    func syncCookiesBeforeRequest(
        url: URL,
        webView: WKWebView,
        completion: @escaping () -> Void
    ) {
        let cookies = HTTPCookieStorage.shared.cookies(for: url) ?? []
        let cookieStore = webView.configuration.websiteDataStore.httpCookieStore
        let group = DispatchGroup()

        for cookie in cookies {
            group.enter()
            cookieStore.setCookie(cookie) { group.leave() }
        }

        group.notify(queue: .main) {
            completion()
        }
    }
}
```

### 5.3 Cookie 持久化策略

| 策略 | 说明 | 适用场景 |
|------|------|---------|
| `WKWebsiteDataStore.default()` | 系统自动持久化到磁盘 | 大多数场景 |
| `WKWebsiteDataStore.nonPersistent()` | 不持久化，退出即清除 | 隐私浏览模式 |
| 手动序列化到 Keychain | 完全控制 Cookie 生命周期 | 需要跨安装保留登录态 |

> ⚠️ **警告**：使用 `nonPersistent()` 数据存储时，Cookie 在 WebView 关闭后丢失。如果需要保留登录状态，务必使用 `default()` 数据存储或手动持久化。

---

## 6. ASWebAuthenticationSession

> 💡 **生活类比**：ASWebAuthenticationSession 就像"银行柜台验证身份"——你到柜台（系统浏览器），出示证件（OAuth 授权），银行确认后给你一个回执（授权码），你拿着回执回到原来的地方（App）继续办理业务。

### 6.1 OAuth 授权码流程

```
┌──────┐         ┌──────────┐         ┌──────────┐
│  App │         │ 授权服务器 │         │  资源服务器 │
└──┬───┘         └────┬─────┘         └────┬─────┘
   │                  │                    │
   │ 1. 打开授权页面    │                    │
   │─────────────────>│                    │
   │                  │                    │
   │ 2. 用户登录并授权  │                    │
   │<─────────────────│                    │
   │                  │                    │
   │ 3. 回调 URL + 授权码│                   │
   │<─────────────────│                    │
   │                  │                    │
   │ 4. 用授权码换 Token │                   │
   │─────────────────>│                    │
   │                  │                    │
   │ 5. 返回 Access Token                  │
   │<─────────────────│                    │
   │                  │                    │
   │ 6. 用 Token 访问资源                    │
   │─────────────────────────────────────>│
```

### 6.2 ASWebAuthenticationSession 使用

```swift
import AuthenticationServices

class AuthManager: NSObject, ASWebAuthenticationPresentationContextProviding {
    var session: ASWebAuthenticationSession?

    func startOAuthFlow() {
        let authURL = URL(string: "https://auth.example.com/authorize?client_id=MY_APP&redirect_uri=myapp://oauth&response_type=code&scope=profile+email")!
        let callbackURLScheme = "myapp"

        session = ASWebAuthenticationSession(
            url: authURL,
            callbackURLScheme: callbackURLScheme
        ) { callbackURL, error in
            if let error = error {
                print("授权失败: \(error.localizedDescription)")
                return
            }

            guard let callbackURL = callbackURL,
                  let code = URLComponents(url: callbackURL, resolvingAgainstBaseURL: false)?
                      .queryItems?
                      .first(where: { $0.name == "code" })?
                      .value
            else { return }

            self.exchangeCodeForToken(code: code)
        }

        session?.presentationContextProvider = self
        session?.prefersEphemeralWebBrowserSession = false
        session?.start()
    }

    func presentationAnchor(for session: ASWebAuthenticationSession) -> ASPresentationAnchor {
        UIApplication.shared.windows.first { $0.isKeyWindow } ?? ASPresentationAnchor()
    }

    func exchangeCodeForToken(code: String) {
        let tokenURL = URL(string: "https://auth.example.com/token")!
        var request = URLRequest(url: tokenURL)
        request.httpMethod = "POST"
        request.setValue("application/x-www-form-urlencoded", forHTTPHeaderField: "Content-Type")
        request.httpBody = "grant_type=authorization_code&code=\(code)&client_id=MY_APP&redirect_uri=myapp://oauth"
            .data(using: .utf8)

        URLSession.shared.dataTask(with: request) { data, response, error in
            guard let data = data,
                  let json = try? JSONSerialization.jsonObject(with: data) as? [String: Any],
                  let accessToken = json["access_token"] as? String
            else { return }

            DispatchQueue.main.async {
                self.saveToken(accessToken)
            }
        }.resume()
    }

    func saveToken(_ token: String) {
        KeychainHelper.save(key: "access_token", value: token)
    }
}
```

### 6.3 回调 URL Scheme 配置

在 Xcode 中注册自定义 URL Scheme：

1. 打开 `Info.plist`
2. 添加 `CFBundleURLTypes` 数组
3. 配置 URL Scheme：

```xml
<key>CFBundleURLTypes</key>
<array>
    <dict>
        <key>CFBundleURLName</key>
        <string>com.myapp.auth</string>
        <key>CFBundleURLSchemes</key>
        <array>
            <string>myapp</string>
        </array>
    </dict>
</array>
```

### 6.4 第三方登录集成对比

| 第三方 | 授权方式 | 回调 Scheme | 注意事项 |
|--------|---------|------------|---------|
| **微信** | SDK 原生 | `wx{appId}` | 必须用微信 SDK，不能用 ASWebAuthenticationSession |
| **Google** | ASWebAuthenticationSession | 自定义 Scheme | 推荐使用官方 SDK |
| **GitHub** | ASWebAuthenticationSession | 自定义 Scheme | 纯 OAuth，最简单 |
| **Facebook** | SDK 原生 | `fb{appId}` | 必须用 Facebook SDK |
| **Apple** | ASAuthorizationAppleIDProvider | 无需 | 系统原生，无需网页 |

> 💡 **提示**：`prefersEphemeralWebBrowserSession = true` 时，系统会优先使用隐私浏览模式，不会共享 Safari Cookie。适合"登录后即走"的场景。

---

## 7. SwiftUI 集成实战

> 💡 **生活类比**：封装 WebView 组件就像打造一辆"定制房车"——底盘是 WKWebView，你加装了导航仪（进度条）、方向盘（前进后退）、行车记录仪（URL 拦截），让它既好用又安全。

### 7.1 完整 WebView 组件封装

```swift
import SwiftUI
import WebKit

struct WebView: UIViewRepresentable {
    let url: URL
    @Binding var isLoading: Bool
    @Binding var estimatedProgress: Double
    @Binding var canGoBack: Bool
    @Binding var canGoForward: Bool
    var onURLChange: ((URL) -> Void)?
    var shouldIntercept: ((URL) -> Bool)?

    func makeUIView(context: Context) -> WKWebView {
        let config = WKWebViewConfiguration()
        config.preferences.javaScriptEnabled = true

        let webView = WKWebView(frame: .zero, configuration: config)
        webView.navigationDelegate = context.coordinator
        webView.uiDelegate = context.coordinator
        webView.allowsBackForwardNavigationGestures = true
        webView.isOpaque = false
        webView.backgroundColor = .systemBackground

        context.coordinator.webView = webView
        context.coordinator.startObservation()

        let request = URLRequest(url: url)
        webView.load(request)

        return webView
    }

    func updateUIView(_ webView: WKWebView, context: Context) {}

    func makeCoordinator() -> Coordinator {
        Coordinator(parent: self)
    }

    class Coordinator: NSObject, WKNavigationDelegate, WKUIDelegate {
        let parent: WebView
        weak var webView: WKWebView?
        private var progressObservation: NSKeyValueObservation?

        init(parent: WebView) {
            self.parent = parent
        }

        func startObservation() {
            guard let webView = webView else { return }

            progressObservation = webView.observe(
                \.estimatedProgress,
                options: .new
            ) { [weak self] _, change in
                guard let progress = change.newValue else { return }
                DispatchQueue.main.async {
                    self?.parent.estimatedProgress = progress
                }
            }
        }

        func webView(
            _ webView: WKWebView,
            decidePolicyFor navigationAction: WKNavigationAction,
            decisionHandler: @escaping (WKNavigationActionPolicy) -> Void
        ) {
            guard let url = navigationAction.request.url else {
                decisionHandler(.allow)
                return
            }

            DispatchQueue.main.async {
                self.parent.onURLChange?(url)
                self.parent.canGoBack = webView.canGoBack
                self.parent.canGoForward = webView.canGoForward
            }

            if let shouldIntercept = parent.shouldIntercept,
               shouldIntercept(url) {
                decisionHandler(.cancel)
                return
            }

            if url.scheme == "tel" {
                UIApplication.shared.open(url)
                decisionHandler(.cancel)
                return
            }

            decisionHandler(.allow)
        }

        func webView(_ webView: WKWebView, didStartProvisionalNavigation navigation: WKNavigation!) {
            DispatchQueue.main.async {
                self.parent.isLoading = true
            }
        }

        func webView(_ webView: WKWebView, didFinish navigation: WKNavigation!) {
            DispatchQueue.main.async {
                self.parent.isLoading = false
                self.parent.canGoBack = webView.canGoBack
                self.parent.canGoForward = webView.canGoForward
            }
        }

        func webView(
            _ webView: WKWebView,
            runJavaScriptAlertPanelWithMessage message: String,
            initiatedByFrame frame: WKFrameInfo,
            completionHandler: @escaping () -> Void
        ) {
            DispatchQueue.main.async {
                let alert = UIAlertController(
                    title: nil, message: message, preferredStyle: .alert
                )
                alert.addAction(UIAlertAction(title: "确定", style: .default) { _ in
                    completionHandler()
                })
                UIApplication.shared.windows.first?.rootViewController?.present(
                    alert, animated: true
                )
            }
        }

        deinit {
            progressObservation?.invalidate()
        }
    }
}
```

### 7.2 带进度条和导航的完整浏览器

```swift
struct BrowserView: View {
    @State private var isLoading = false
    @State private var estimatedProgress: Double = 0
    @State private var canGoBack = false
    @State private var canGoForward = false
    @State private var currentURL: URL?
    @State private var showShareSheet = false

    private let initialURL = URL(string: "https://www.apple.com")!

    var body: some View {
        VStack(spacing: 0) {
            ProgressBar(progress: estimatedProgress, isLoading: isLoading)

            WebView(
                url: initialURL,
                isLoading: $isLoading,
                estimatedProgress: $estimatedProgress,
                canGoBack: $canGoBack,
                canGoForward: $canGoForward,
                onURLChange: { url in
                    currentURL = url
                },
                shouldIntercept: { url in
                    if url.host?.contains("malware-site.com") == true {
                        print("已拦截恶意网站: \(url)")
                        return true
                    }
                    return false
                }
            )

            Divider()

            HStack(spacing: 0) {
                Button {
                    WebViewHelper.goBack()
                } label: {
                    Image(systemName: "chevron.left")
                        .font(.body)
                        .frame(maxWidth: .infinity)
                        .foregroundStyle(canGoBack ? .primary : .gray)
                }
                .disabled(!canGoBack)

                Button {
                    WebViewHelper.goForward()
                } label: {
                    Image(systemName: "chevron.right")
                        .font(.body)
                        .frame(maxWidth: .infinity)
                        .foregroundStyle(canGoForward ? .primary : .gray)
                }
                .disabled(!canGoForward)

                Button {
                    WebViewHelper.reload()
                } label: {
                    Image(systemName: "arrow.clockwise")
                        .font(.body)
                        .frame(maxWidth: .infinity)
                }

                Button {
                    showShareSheet = true
                } label: {
                    Image(systemName: "square.and.arrow.up")
                        .font(.body)
                        .frame(maxWidth: .infinity)
                }
            }
            .padding(.vertical, 8)
            .background(Color(.systemBackground))
        }
        .navigationTitle(currentURL?.host ?? "浏览器")
        .navigationBarTitleDisplayMode(.inline)
        .sheet(isPresented: $showShareSheet) {
            if let url = currentURL {
                ShareSheet(items: [url])
            }
        }
    }
}
```

### 7.3 进度条组件

```swift
struct ProgressBar: View {
    let progress: Double
    let isLoading: Bool

    var body: some View {
        GeometryReader { geometry in
            if isLoading {
                ZStack(alignment: .leading) {
                    Rectangle()
                        .fill(Color.gray.opacity(0.2))
                        .frame(height: 2)

                    Rectangle()
                        .fill(Color.blue)
                        .frame(
                            width: geometry.size.width * min(progress, 1.0),
                            height: 2
                        )
                        .animation(.linear(duration: 0.2), value: progress)
                }
            }
        }
        .frame(height: 2)
        .animation(.easeInOut, value: isLoading)
    }
}
```

### 7.4 ShareSheet 封装

```swift
struct ShareSheet: UIViewControllerRepresentable {
    let items: [Any]

    func makeUIViewController(context: Context) -> UIActivityViewController {
        let controller = UIActivityViewController(
            activityItems: items,
            applicationActivities: nil
        )
        return controller
    }

    func updateUIViewController(
        _ uiViewController: UIActivityViewController,
        context: Context
    ) {}
}
```

---

## 8. 审核注意事项

> 💡 **生活类比**：App Store 审核就像"小区物业验收"——你的 App 可以有 Web 内容，但不能是个"空壳子"（只有网页没有原生功能），否则物业（Apple）不会让你入住。

### 8.1 常见审核被拒原因

| 审核条款 | 被拒原因 | 解决方案 |
|----------|---------|---------|
| **4.2 最小功能** | App 仅为网站包装，缺乏原生功能 | 添加原生交互、推送、离线功能等 |
| **4.2 Web Clip** | 行为像 Web Clip（桌面快捷方式） | 确保有明显的原生功能模块 |
| **2.5.6** | WKWebView 中引导用户绕过 App 内购买 | 不得在 Web 视图中提供应用内购买替代方案 |
| **5.1.1 隐私** | 未说明数据收集用途 | 在隐私政策中明确 Cookie 使用目的 |
| **2.1 性能** | Web 视图加载过慢或崩溃 | 优化加载策略、处理超时 |

### 8.2 4.2 条款深度解读

Apple 审核指南 4.2 节要求 App 具有足够的原生功能，不能是简单的"网站打包"：

```
❌ 容易被拒的情况：
├── App 只有一个全屏 WebView，加载公司官网
├── 所有功能都通过网页实现，没有原生交互
├── 没有利用任何 iOS 原生能力（推送、相机、定位等）
└── 用户体验与 Safari 打开网站无异

✅ 容易通过的情况：
├── 混合架构：核心功能原生实现，部分内容用 WebView 展示
├── WebView 中集成了 JS Bridge，与原生深度交互
├── 利用了推送通知、离线缓存、生物识别等原生能力
└── 提供了比 Safari 更好的用户体验（如内容过滤、阅读模式）
```

### 8.3 Cookie 隐私合规

```swift
import AppTrackingTransparency

func requestTrackingPermission() {
    ATTrackingManager.requestTrackingAuthorization { status in
        switch status {
        case .authorized:
            print("用户允许跟踪")
        case .denied, .restricted, .notDetermined:
            print("用户拒绝跟踪")
            useNonPersistentStorage()
        @unknown default:
            break
        }
    }
}

func useNonPersistentStorage() {
    let config = WKWebViewConfiguration()
    config.websiteDataStore = .nonPersistent()
}
```

### 8.4 Web 视图审核自查清单

| 检查项 | 是否通过 | 备注 |
|--------|---------|------|
| App 是否有原生功能模块？ | ☐ | 至少 2-3 个原生功能 |
| WebView 是否仅用于内容展示？ | ☐ | 核心功能不应完全依赖 Web |
| 是否在 WebView 中绕过 IAP？ | ☐ | 严禁在 Web 中提供付费替代 |
| 隐私政策是否说明 Cookie 使用？ | ☐ | 需明确数据收集与用途 |
| 是否请求了 ATT 权限？ | ☐ | 涉及跨应用跟踪时必须 |
| 是否处理了网络异常？ | ☐ | 避免白屏或崩溃 |
| 是否拦截了恶意 URL？ | ☐ | 安全防护措施 |

> ⚠️ **警告**：如果你的 App 本质上是一个"网站打包"，请认真评估是否应该做成 PWA（渐进式 Web 应用）而非原生 App。Apple 对纯 Web 包装 App 的审核越来越严格。

---

## 本章小结

| 主题 | 核心要点 | 关键 API |
|------|---------|---------|
| **三种方式对比** | SFSafariViewController 安全简单、WKWebView 灵活可控、ASWebAuthenticationSession 专用于授权 | — |
| **SFSafariViewController** | 共享 Safari Cookie 和密码自动填充，自定义程度低 | `SFSafariViewController` |
| **WKWebView 配置** | 通过 Configuration 控制行为，两个 Delegate 处理导航和 UI | `WKWebViewConfiguration`、`WKNavigationDelegate`、`WKUIDelegate` |
| **JS 交互** | evaluateJavaScript 调用 JS，WKScriptMessageHandler 接收 JS 消息，JS Bridge 统一管理 | `evaluateJavaScript`、`WKUserContentController`、`WKScriptMessageHandler` |
| **Cookie 管理** | WKWebView 与 URLSession Cookie 不共享，需手动同步 | `WKHTTPCookieStore`、`HTTPCookieStorage` |
| **ASWebAuthenticationSession** | 系统级 OAuth 授权，共享 Safari Cookie，需配置回调 Scheme | `ASWebAuthenticationSession` |
| **SwiftUI 集成** | UIViewRepresentable 封装，KVO 监听进度，Coordinator 处理回调 | `UIViewRepresentable`、`NSKeyValueObservation` |
| **审核合规** | 避免纯 Web 包装、不在 WebView 中绕过 IAP、Cookie 隐私合规 | ATT、隐私政策 |

> 💡 **最佳实践总结**：能用 SFSafariViewController 就不要用 WKWebView；必须用 WKWebView 时，务必实现 WKUIDelegate 处理弹窗；涉及 OAuth 授权优先使用 ASWebAuthenticationSession；始终注意 Cookie 同步和隐私合规。
