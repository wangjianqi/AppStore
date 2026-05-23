---
name: network-api
description: 涉及网络请求、API 调用、后端接口、HTTP、URLSession、数据解析、Token 刷新、重试、AI API 集成的任务
---

# 网络 / API 层

## 网络层架构
- 基础层：**URLSession**（禁止引入 Alamofire，除非已存在）
- 封装层：`NetworkService.swift` 统一处理请求、错误、重试
- 接口定义：每个模块有对应 `XxxAPI.swift`，定义 endpoint + 参数

---

## NetworkService 完整封装

```swift
protocol NetworkServiceProtocol {
    func request<T: Decodable>(_ endpoint: APIEndpoint) async throws -> T
    func upload<T: Decodable>(_ endpoint: APIEndpoint, data: Data) async throws -> T
}

final class NetworkService: NetworkServiceProtocol {
    private let session: URLSession
    private let keychain: KeychainStorage
    private let config: APIConfig

    init(session: URLSession = .shared, keychain: KeychainStorage = .shared, config: APIConfig = .current) {
        self.session = session
        self.keychain = keychain
        self.config = config
    }

    func request<T: Decodable>(_ endpoint: APIEndpoint) async throws -> T {
        let urlRequest = try buildRequest(for: endpoint)
        let response: T = try await executeWithRetry(urlRequest, maxRetries: endpoint.retryCount)
        return response
    }

    func upload<T: Decodable>(_ endpoint: APIEndpoint, data: Data) async throws -> T {
        var urlRequest = try buildRequest(for: endpoint)
        urlRequest.httpBody = data
        urlRequest.setValue("application/octet-stream", forHTTPHeaderField: "Content-Type")
        return try await execute(urlRequest)
    }

    private func buildRequest(for endpoint: APIEndpoint) throws -> URLRequest {
        var components = URLComponents(string: config.baseURL + endpoint.path)
        if let parameters = endpoint.queryParameters {
            components?.queryItems = parameters.map { URLQueryItem(name: $0.key, value: "\($0.value)") }
        }
        guard let url = components?.url else { throw NetworkError.invalidURL }

        var request = URLRequest(url: url)
        request.httpMethod = endpoint.method.rawValue
        request.timeoutInterval = endpoint.timeout

        if endpoint.requiresAuth {
            guard let token = keychain.loadToken() else { throw NetworkError.unauthorized }
            request.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")
        }

        if let body = endpoint.body {
            request.httpBody = try JSONSerialization.data(withJSONObject: body)
            request.setValue("application/json", forHTTPHeaderField: "Content-Type")
        }

        request.setValue("application/json", forHTTPHeaderField: "Accept")
        request.setValue(Bundle.main.appVersion, forHTTPHeaderField: "X-App-Version")
        request.setValue(UIDevice.current.systemVersion, forHTTPHeaderField: "X-iOS-Version")

        return request
    }

    private func execute<T: Decodable>(_ request: URLRequest) async throws -> T {
        #if DEBUG
        logRequest(request)
        #endif

        let (data, urlResponse) = try await session.data(for: request)

        #if DEBUG
        logResponse(data, response: urlResponse)
        #endif

        guard let httpResponse = urlResponse as? HTTPURLResponse else {
            throw NetworkError.invalidResponse
        }

        switch httpResponse.statusCode {
        case 200...299:
            do {
                let decoder = JSONDecoder()
                decoder.keyDecodingStrategy = .convertFromSnakeCase
                decoder.dateDecodingStrategy = .iso8601
                return try decoder.decode(T.self, from: data)
            } catch {
                throw NetworkError.decodingFailed
            }
        case 401:
            throw NetworkError.unauthorized
        case 400...499:
            if let errorBody = try? JSONDecoder().decode(ErrorResponse.self, from: data) {
                throw NetworkError.clientError(httpResponse.statusCode, errorBody.message)
            }
            throw NetworkError.clientError(httpResponse.statusCode, "客户端错误")
        case 500...599:
            throw NetworkError.serverError(httpResponse.statusCode)
        default:
            throw NetworkError.unknown
        }
    }
}
```

---

## API Endpoint 定义

```swift
enum HTTPMethod: String {
    case get = "GET"
    case post = "POST"
    case put = "PUT"
    case patch = "PATCH"
    case delete = "DELETE"
}

struct APIEndpoint {
    let path: String
    let method: HTTPMethod
    let queryParameters: [String: Any]?
    let body: [String: Any]?
    let requiresAuth: Bool
    let timeout: TimeInterval
    let retryCount: Int

    init(path: String, method: HTTPMethod = .get, queryParameters: [String: Any]? = nil,
         body: [String: Any]? = nil, requiresAuth: Bool = true, timeout: TimeInterval = 30, retryCount: Int = 0) {
        self.path = path
        self.method = method
        self.queryParameters = queryParameters
        self.body = body
        self.requiresAuth = requiresAuth
        self.timeout = timeout
        self.retryCount = retryCount
    }
}

enum AuthAPI {
    static let login = APIEndpoint(path: "/auth/login", method: .post, requiresAuth: false)
    static func login(email: String, password: String) -> APIEndpoint {
        APIEndpoint(path: "/auth/login", method: .post, body: ["email": email, "password": password], requiresAuth: false)
    }
    static let refreshToken = APIEndpoint(path: "/auth/refresh", method: .post, retryCount: 0)
}

enum ItemAPI {
    static let list = APIEndpoint(path: "/items", method: .get)
    static func create(_ item: Item) -> APIEndpoint {
        APIEndpoint(path: "/items", method: .post, body: ["title": item.title])
    }
    static func delete(id: String) -> APIEndpoint {
        APIEndpoint(path: "/items/\(id)", method: .delete)
    }
}
```

---

## Token 刷新流程

```swift
final class TokenRefreshHandler {
    private let network: NetworkServiceProtocol
    private let keychain: KeychainStorage
    private var isRefreshing = false
    private var pendingRequests: [(String) -> Void] = []

    init(network: NetworkServiceProtocol, keychain: KeychainStorage) {
        self.network = network
        self.keychain = keychain
    }

    func refreshIfNeeded() async throws -> String {
        if let token = keychain.loadToken(), !token.isExpired {
            return token
        }

        return try await withCheckedThrowingContinuation { continuation in
            pendingRequests.append { newToken in
                continuation.resume(returning: newToken)
            }

            guard !isRefreshing else { return }
            isRefreshing = true

            Task {
                do {
                    let response: TokenResponse = try await network.request(AuthAPI.refreshToken)
                    try keychain.saveToken(response.token)
                    let newToken = response.token
                    pendingRequests.forEach { $0(newToken) }
                    pendingRequests.removeAll()
                } catch {
                    keychain.deleteToken()
                    pendingRequests.removeAll()
                    throw error
                }
                isRefreshing = false
            }
        }
    }
}
```

### 自动刷新拦截

```swift
private func executeWithRetry<T: Decodable>(_ request: URLRequest, maxRetries: Int) async throws -> T {
    do {
        return try await execute(request)
    } catch NetworkError.unauthorized {
        guard maxRetries > 0 else { throw NetworkError.unauthorized }
        let newToken = try await tokenRefresh.refreshIfNeeded()
        var retryRequest = request
        retryRequest.setValue("Bearer \(newToken)", forHTTPHeaderField: "Authorization")
        return try await execute(retryRequest)
    } catch NetworkError.serverError, NetworkError.noConnection where maxRetries > 0 {
        try await Task.sleep(nanoseconds: UInt64(pow(2.0, Double(maxRetries))) * 1_000_000_000)
        return try await executeWithRetry(request, maxRetries: maxRetries - 1)
    }
}
```

---

## 后端配置

### 环境切换

```swift
enum APIConfig {
    case dev
    case staging
    case production

    static var current: APIConfig {
        #if DEBUG
        return .dev
        #else
        return .production
        #endif
    }

    var baseURL: String {
        switch self {
        case .dev: return "http://localhost:8080/api/v1"
        case .staging: return "https://staging-api.example.com/api/v1"
        case .production: return "https://api.example.com/api/v1"
        }
    }
}
```

### 通用响应格式

```json
{
    "code": 0,
    "message": "success",
    "data": { ... }
}
```

- `code != 0` 统一走 `AppError.serverError(code:message:)` 处理

---

## AI API 集成

### OpenAI / Whisper
- API Key **禁止硬编码在客户端**，必须通过自有后端中转
- 语音转文字：优先用 `whisper-1`，文件限制 25MB，超出需分段
- 流式输出：使用 `URLSessionDataTask` + chunked response 解析

```swift
func streamChat(messages: [ChatMessage]) -> AsyncThrowingStream<String, Error> {
    AsyncThrowingStream { continuation in
        var request = URLRequest(url: URL(string: config.baseURL + "/chat/completions")!)
        request.httpMethod = "POST"
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")
        request.setValue("Bearer \(apiKey)", forHTTPHeaderField: "Authorization")
        request.httpBody = try? JSONEncoder().encode(StreamRequest(messages: messages, stream: true))

        let task = session.bytesTask(with: request)
        Task {
            for try await line in task {
                guard let line = String(bytes: line, encoding: .utf8),
                      line.hasPrefix("data: "),
                      let data = line.dropFirst(6).data(using: .utf8),
                      let chunk = try? JSONDecoder().decode(StreamChunk.self, from: data),
                      let content = chunk.choices.first?.delta.content else { continue }
                continuation.yield(content)
            }
            continuation.finish()
        }
    }
}
```

### Anthropic Claude
- 同上，通过后端代理，禁止客户端直接调用
- 长文本处理：注意 token 限制，超出时做客户端截断

### 本地 Whisper（whisper.cpp）
- 通过 C++ 桥接调用，封装为 `WhisperService`
- 音频格式：16kHz, 16-bit, mono WAV
- 滑动窗口去重：使用项目内 `TranscriptDeduplicator`

---

## 错误处理规范

```swift
enum NetworkError: LocalizedError {
    case invalidURL
    case invalidResponse
    case noConnection
    case timeout
    case serverError(Int)
    case clientError(Int, String)
    case decodingFailed
    case unauthorized

    var errorDescription: String? {
        switch self {
        case .invalidURL: return "请求地址错误"
        case .invalidResponse: return "服务器响应异常"
        case .noConnection: return "网络连接失败，请检查网络后重试"
        case .timeout: return "请求超时，请稍后重试"
        case .serverError: return "服务异常，请稍后重试"
        case .clientError(_, let message): return message
        case .decodingFailed: return "数据解析失败"
        case .unauthorized: return "登录已过期，请重新登录"
        }
    }
}
```

- 401 错误：自动刷新 Token，失败则跳登录页
- 网络无连接：展示离线提示，不弹 Alert
- 5xx 错误：展示"服务异常，请稍后重试"

---

## 数据解析
- 全部使用 **Codable**，禁止手动解析 JSON
- 服务端字段命名蛇形（`snake_case`），客户端驼峰（`camelCase`）：
  ```swift
  decoder.keyDecodingStrategy = .convertFromSnakeCase
  ```
- 可选字段用 `?`，避免因缺字段导致整体解析失败
- 日期统一 ISO 8601：`decoder.dateDecodingStrategy = .iso8601`

---

## 缓存策略
- GET 请求：URLCache 默认缓存（根据 Cache-Control）
- 用户数据：写入 CoreData，离线可读
- 图片：Kingfisher 自动磁盘缓存，限制 200MB

---

## 调试
- Debug 模式打印完整 request / response（用 `#if DEBUG` 包裹）
- 禁止在 Release 包里打印 API Key 或 Token

```swift
#if DEBUG
private func logRequest(_ request: URLRequest) {
    print("→ \(request.httpMethod ?? "") \(request.url?.absoluteString ?? "")")
    if let body = request.httpBody, let json = try? JSONSerialization.jsonObject(with: body) {
        print("  Body: \(json)")
    }
}

private func logResponse(_ data: Data, response: URLResponse) {
    if let http = response as? HTTPURLResponse {
        print("← \(http.statusCode) \(http.url?.absoluteString ?? "")")
    }
    if let json = try? JSONSerialization.jsonObject(with: data) {
        print("  Data: \(json)")
    }
}
#endif
```

---

## 已知陷阱

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| Token 刷新竞态 | 多个请求同时 401 | 用 `isRefreshing` + 队列合并刷新请求 |
| 请求超时无回调 | URLSession 默认超时过长 | 设置 `timeoutIntervalForResource = 60` |
| 后台请求失败 | App 挂起后网络中断 | 使用 `URLSessionConfiguration.background` |
| JSON 解析崩溃 | 服务端返回非 JSON | 先检查 `Content-Type`，再解析 |
| 内存泄漏 | URLSession delegate 强引用 | 使用 `[weak self]` 或 `session.finishTasksAndInvalidate()` |
| DNS 劫持 | HTTP DNS 解析被篡改 | 关键请求启用 Certificate Pinning |
| 请求重复发送 | 用户快速点击 | 按钮点击后禁用，请求完成再启用 |
| Codable 解码失败 | 可选字段未标记 `?` | 所有非必填字段用可选类型 |
