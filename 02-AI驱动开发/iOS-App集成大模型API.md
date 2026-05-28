# iOS App 集成大模型 API

> 🎯 **本章目标**：
> - 理解为什么要在 iOS App 中集成大模型 API，以及典型应用场景
> - 掌握主流云端 LLM API 的选型方法与对比
> - 学会设计网络层架构，包括协议、请求/响应模型、错误处理
> - 使用 Swift 原生 URLSession + async/await 实现完整的 LLM API 调用
> - 实现流式输出（SSE），让用户看到逐字生成的效果
> - 掌握 API Key 安全管理的多种方案
> - 实现 Token 用量追踪与成本控制
> - 掌握错误重试、降级策略与性能优化技巧

---

## 1. 为什么要在 iOS App 中集成大模型 API

### 从"用 AI 写 App"到"在 App 里用 AI"

前面章节我们一直在讲怎么用 AI 工具来**写** App——用 Claude Code 生成代码、用 Cursor 辅助编程、用 AI 做代码审查。这些是**开发阶段**的 AI 赋能。

但 AI 的价值不止于此。当你的 App **自身**具备 AI 能力时，用户体验会发生质变：

```
传统 App：用户输入 → 固定逻辑处理 → 返回结果
AI App：  用户输入 → AI 理解意图 → 智能生成结果
```

一个笔记 App，如果加了 AI 摘要功能，用户就不需要自己总结长文；一个翻译 App，如果用了大模型，翻译质量会远超传统机器翻译；一个客服 App，如果接入了 LLM，就能 24 小时提供拟人化的服务。

**这就是从"用 AI 写 App"到"在 App 里用 AI"的跨越。**

### 典型应用场景

| 场景 | 说明 | 示例 App |
|------|------|---------|
| **智能客服** | 用 LLM 理解用户问题，生成自然语言回复 | 电商售后、银行客服 |
| **内容生成** | 根据提示生成文章、文案、诗歌等 | 写作助手、营销工具 |
| **代码助手** | 代码补全、解释、重构建议 | 开发者工具 App |
| **智能翻译** | 上下文感知的高质量翻译 | 翻译 App、阅读器 |
| **文本摘要** | 长文自动提取关键信息 | 新闻阅读、论文工具 |
| **对话陪伴** | 拟人化的多轮对话 | 聊天机器人、虚拟角色 |
| **数据分析** | 自然语言查询数据库，生成图表 | BI 工具、报表 App |
| **教育辅导** | 个性化题目讲解、知识问答 | 学习 App、考试助手 |
| **图片理解** | 多模态模型分析图片内容 | 拍照识物、无障碍辅助 |
| **语音交互** | 语音识别 + LLM 理解 + 语音合成 | 语音助手、车载系统 |

> 💡 **提示**：不是所有功能都需要大模型。对于规则明确、逻辑固定的任务（如计算器、格式转换），传统代码更高效、更可靠。大模型适合处理**开放性、创造性、理解性**的任务。

---

## 2. 云端 LLM API 概览

### 主流 API 对比

| 模型 | API 地址 | 上下文长度 | 输入价格（/百万 Token） | 输出价格（/百万 Token） | 特点 |
|------|---------|-----------|----------------------|----------------------|------|
| **GPT-4o** | `api.openai.com` | 128K | $2.50 | $10.00 | 综合能力最强，多模态，生态最完善 |
| **GPT-4o mini** | `api.openai.com` | 128K | $0.15 | $0.60 | 性价比高，适合轻量任务 |
| **Claude 3.5 Sonnet** | `api.anthropic.com` | 200K | $3.00 | $15.00 | 长文本优秀，代码能力强 |
| **Claude 3.5 Haiku** | `api.anthropic.com` | 200K | $0.80 | $4.00 | 速度快，成本低 |
| **通义千问 Qwen-Max** | `dashscope.aliyuncs.com` | 128K | ¥2.00 | ¥6.00 | 中文能力优秀，国内合规 |
| **通义千问 Qwen-Plus** | `dashscope.aliyuncs.com` | 128K | ¥0.40 | ¥1.20 | 性价比高，适合日常任务 |
| **DeepSeek-V3** | `api.deepseek.com` | 128K | ¥1.00 | ¥2.00 | 代码能力强，价格极低 |
| **DeepSeek-R1** | `api.deepseek.com` | 128K | ¥4.00 | ¥16.00 | 推理能力强，适合复杂任务 |
| **文心一言 ERNIE-4.0** | `aip.baidubce.com` | 128K | ¥3.00 | ¥9.00 | 百度生态，中文理解好 |
| **智谱 GLM-4** | `open.bigmodel.cn` | 128K | ¥5.00 | ¥5.00 | 输入输出同价，中文能力强 |

> ⚠️ **警告**：以上价格为参考价格，各厂商会不定期调整。实际使用前请查阅官方最新价格表。国内模型价格通常以人民币计费，海外模型以美元计费。

### 选型建议

#### 海外 vs 国内

| 维度 | 海外模型（OpenAI / Anthropic） | 国内模型（通义 / DeepSeek / 文心 / 智谱） |
|------|------------------------------|----------------------------------------|
| **合规性** | 需关注数据出境合规 | 国内合规，数据不出境 |
| **网络** | 需要代理或中转，延迟较高 | 直连，延迟低 |
| **中文能力** | 优秀但非母语级 | 母语级中文理解 |
| **英文能力** | 最强 | 良好 |
| **价格** | 较高 | 较低 |
| **审核** | App Store 审核无特殊要求 | 国内上线需算法备案 |

#### 成本 vs 质量

| 场景 | 推荐策略 | 原因 |
|------|---------|------|
| 高质量内容生成 | GPT-4o / Claude 3.5 Sonnet | 质量优先，用户付费意愿高 |
| 日常对话 / 简单问答 | DeepSeek-V3 / Qwen-Plus | 性价比高，体验够用 |
| 大量调用 / 后台批处理 | GPT-4o mini / Qwen-Turbo | 成本最低，批量处理 |
| 复杂推理任务 | DeepSeek-R1 / Claude 3.5 Sonnet | 推理能力最强 |

#### 延迟 vs 能力

| 需求 | 推荐策略 |
|------|---------|
| 实时对话（< 1s 首字） | 使用国内模型 + 流式输出 |
| 后台分析（延迟不敏感） | 使用最强模型，等待完整结果 |
| 高并发场景 | 使用轻量模型 + 负载均衡 |

> 💡 **提示**：很多国内模型（如 DeepSeek、通义千问）兼容 OpenAI API 格式，这意味着你可以用同一套代码，只需改 API 地址和 Key 就能切换模型。后文会详细讲解如何设计兼容多模型的架构。

---

## 3. 网络层架构设计

### 整体架构

```
┌─────────────────────────────────────────────────────────┐
│                      SwiftUI View                        │
│                   （展示对话界面）                          │
└──────────────────────────┬──────────────────────────────┘
                           │ 用户输入
                           ▼
┌─────────────────────────────────────────────────────────┐
│                      ViewModel                           │
│              （管理状态、调用服务、更新 UI）                  │
│                                                          │
│   messages: [ChatMessage]                                │
│   isLoading: Bool                                        │
│   tokenUsage: TokenUsage                                 │
└──────────────────────────┬──────────────────────────────┘
                           │ 调用服务
                           ▼
┌─────────────────────────────────────────────────────────┐
│                    LLMService                            │
│            （协议层：定义 AI 调用接口）                      │
│                                                          │
│   send(_:completion:) -> [ChatMessage]                   │
│   stream(_:onToken:) -> AsyncThrowingStream              │
└──────────────────────────┬──────────────────────────────┘
                           │ 具体实现
                           ▼
┌─────────────────────────────────────────────────────────┐
│              OpenAICompatibleService                     │
│        （实现层：构建请求、解析响应、处理错误）                │
│                                                          │
│   baseURL: URL                                           │
│   apiKey: String                                         │
│   model: String                                          │
└──────────────────────────┬──────────────────────────────┘
                           │ 网络请求
                           ▼
┌─────────────────────────────────────────────────────────┐
│                     URLSession                           │
│              （系统网络层：发送 HTTP 请求）                   │
└──────────────────────────┬──────────────────────────────┘
                           │ HTTPS
                           ▼
┌─────────────────────────────────────────────────────────┐
│                   LLM API 服务端                          │
│        （OpenAI / DeepSeek / 通义千问 / ...）              │
└─────────────────────────────────────────────────────────┘
```

### LLMService 协议设计

```swift
import Foundation

protocol LLMServiceProtocol {
    var configuration: LLMConfiguration { get }

    func send(_ request: LLMRequest) async throws -> LLMResponse
    func stream(_ request: LLMRequest) -> AsyncThrowingStream<StreamChunk, Error>
    func cancel()
}

struct LLMConfiguration {
    var baseURL: URL
    var apiKey: String
    var model: String
    var temperature: Double = 0.7
    var maxTokens: Int = 2048
    var topP: Double = 1.0
    var timeoutInterval: TimeInterval = 60
}
```

### 请求模型设计

```swift
struct LLMRequest: Codable {
    let model: String
    let messages: [ChatMessage]
    let temperature: Double?
    let maxTokens: Int?
    let topP: Double?
    let stream: Bool?

    enum CodingKeys: String, CodingKey {
        case model, messages, temperature, topP, stream
        case maxTokens = "max_tokens"
    }
}

struct ChatMessage: Codable, Identifiable {
    let id: UUID
    let role: Role
    var content: String
    let timestamp: Date

    enum Role: String, Codable {
        case system
        case user
        case assistant
    }

    init(role: Role, content: String) {
        self.id = UUID()
        self.role = role
        self.content = content
        self.timestamp = Date()
    }
}
```

### 响应模型设计

```swift
struct LLMResponse: Codable {
    let id: String
    let object: String
    let created: Int
    let model: String
    let choices: [Choice]
    let usage: Usage

    var message: ChatMessage? {
        choices.first?.message
    }

    struct Choice: Codable {
        let index: Int
        let message: ResponseMessage
        let finishReason: String?

        enum CodingKeys: String, CodingKey {
            case index, message
            case finishReason = "finish_reason"
        }
    }

    struct ResponseMessage: Codable {
        let role: String
        let content: String
    }

    struct Usage: Codable {
        let promptTokens: Int
        let completionTokens: Int
        let totalTokens: Int

        enum CodingKeys: String, CodingKey {
            case promptTokens = "prompt_tokens"
            case completionTokens = "completion_tokens"
            case totalTokens = "total_tokens"
        }
    }
}

struct StreamChunk: Codable {
    let id: String
    let object: String
    let created: Int
    let model: String
    let choices: [StreamChoice]

    var deltaContent: String? {
        choices.first?.delta?.content
    }

    var isFinished: Bool {
        choices.first?.finishReason != nil
    }

    struct StreamChoice: Codable {
        let index: Int
        let delta: Delta?
        let finishReason: String?

        enum CodingKeys: String, CodingKey {
            case index, delta
            case finishReason = "finish_reason"
        }
    }

    struct Delta: Codable {
        let role: String?
        let content: String?
    }
}
```

### 错误处理模型

```swift
enum LLMError: LocalizedError {
    case invalidURL
    case invalidAPIKey
    case networkError(Error)
    case rateLimited(retryAfter: TimeInterval?)
    case contextLengthExceeded
    case modelNotFound
    case serverError(Int, String)
    case invalidResponse
    case streamDisconnected
    case cancelled
    case quotaExceeded

    var errorDescription: String? {
        switch self {
        case .invalidURL:
            return "API 地址无效"
        case .invalidAPIKey:
            return "API Key 无效，请检查配置"
        case .networkError(let error):
            return "网络错误：\(error.localizedDescription)"
        case .rateLimited(let retryAfter):
            if let after = retryAfter {
                return "请求过于频繁，请 \(Int(after)) 秒后重试"
            }
            return "请求过于频繁，请稍后重试"
        case .contextLengthExceeded:
            return "对话内容超出模型上下文长度限制"
        case .modelNotFound:
            return "模型不存在或不可用"
        case .serverError(let code, let message):
            return "服务器错误 (\(code))：\(message)"
        case .invalidResponse:
            return "响应格式异常"
        case .streamDisconnected:
            return "流式连接断开"
        case .cancelled:
            return "请求已取消"
        case .quotaExceeded:
            return "API 调用额度已用尽"
        }
    }
}
```

> 💡 **提示**：将错误类型细分，可以让上层根据不同错误采取不同策略——比如 `rateLimited` 时自动重试，`contextLengthExceeded` 时自动截断历史消息，`quotaExceeded` 时提示用户升级套餐。

---

## 4. Swift 原生网络调用

### URLSession + async/await 实现

Swift 5.5 引入的 `async/await` 让异步网络调用变得像同步代码一样清晰。我们不需要第三方库（如 Alamofire），用系统原生的 `URLSession` 就能完成所有工作。

### 完整的 LLMService 实现

以下代码支持 OpenAI 兼容格式，适用于 OpenAI、DeepSeek、通义千问等模型：

```swift
import Foundation

final class OpenAICompatibleService: LLMServiceProtocol {
    let configuration: LLMConfiguration
    private var currentTask: Task<Void, Never>?

    init(configuration: LLMConfiguration) {
        self.configuration = configuration
    }

    func send(_ request: LLMRequest) async throws -> LLMResponse {
        var request = request
        request = request.withStream(false)

        let urlRequest = try buildURLRequest(from: request)

        do {
            let (data, response) = try await URLSession.shared.data(for: urlRequest)
            return try parseResponse(data: data, response: response)
        } catch let error as LLMError {
            throw error
        } catch {
            throw LLMError.networkError(error)
        }
    }

    func stream(_ request: LLMRequest) -> AsyncThrowingStream<StreamChunk, Error> {
        var request = request
        request = request.withStream(true)

        return AsyncThrowingStream { continuation in
            let task = Task {
                do {
                    let urlRequest = try self.buildURLRequest(from: request)
                    let (bytes, response) = try await URLSession.shared.bytes(for: urlRequest)

                    guard let httpResponse = response as? HTTPURLResponse else {
                        throw LLMError.invalidResponse
                    }

                    guard httpResponse.statusCode == 200 else {
                        let body = try? await bytes.lines.reduce("", { $0 + $1 })
                        throw self.parseError(statusCode: httpResponse.statusCode, body: body ?? "")
                    }

                    for try await line in bytes.lines {
                        guard !Task.isCancelled else { break }

                        if line.hasPrefix("data: ") {
                            let jsonString = String(line.dropFirst(6))

                            if jsonString == "[DONE]" {
                                continuation.finish()
                                return
                            }

                            if let jsonData = jsonString.data(using: .utf8),
                               let chunk = try? JSONDecoder().decode(StreamChunk.self, from: jsonData) {
                                continuation.yield(chunk)
                            }
                        }
                    }

                    continuation.finish()
                } catch is CancellationError {
                    continuation.finish(throwing: LLMError.cancelled)
                } catch let error as LLMError {
                    continuation.finish(throwing: error)
                } catch {
                    continuation.finish(throwing: LLMError.networkError(error))
                }
            }

            self.currentTask = task

            continuation.onTermination = { _ in
                task.cancel()
            }
        }
    }

    func cancel() {
        currentTask?.cancel()
        currentTask = nil
    }

    private func buildURLRequest(from request: LLMRequest) throws -> URLRequest {
        guard let url = URL(string: configuration.baseURL.absoluteString + "/chat/completions") else {
            throw LLMError.invalidURL
        }

        var urlRequest = URLRequest(url: url)
        urlRequest.httpMethod = "POST"
        urlRequest.setValue("application/json", forHTTPHeaderField: "Content-Type")
        urlRequest.setValue("Bearer \(configuration.apiKey)", forHTTPHeaderField: "Authorization")
        urlRequest.timeoutInterval = configuration.timeoutInterval

        let encoder = JSONEncoder()
        encoder.keyEncodingStrategy = .convertToSnakeCase
        urlRequest.httpBody = try encoder.encode(request)

        return urlRequest
    }

    private func parseResponse(data: Data, response: URLResponse) throws -> LLMResponse {
        guard let httpResponse = response as? HTTPURLResponse else {
            throw LLMError.invalidResponse
        }

        guard httpResponse.statusCode == 200 else {
            throw parseError(statusCode: httpResponse.statusCode, body: String(data: data, encoding: .utf8) ?? "")
        }

        let decoder = JSONDecoder()
        do {
            return try decoder.decode(LLMResponse.self, from: data)
        } catch {
            throw LLMError.invalidResponse
        }
    }

    private func parseError(statusCode: Int, body: String) -> LLMError {
        switch statusCode {
        case 401:
            return .invalidAPIKey
        case 429:
            let retryAfter = extractRetryAfter(from: body)
            return .rateLimited(retryAfter: retryAfter)
        case 404:
            return .modelNotFound
        case 402, 503:
            return .quotaExceeded
        default:
            return .serverError(statusCode, body)
        }
    }

    private func extractRetryAfter(from body: String) -> TimeInterval? {
        guard let data = body.data(using: .utf8),
              let json = try? JSONSerialization.jsonObject(with: data) as? [String: Any],
              let error = json["error"] as? [String: Any],
              let retryAfter = error["retry_after"] as? Double else {
            return nil
        }
        return retryAfter
    }
}
```

### LLMRequest 辅助方法

```swift
extension LLMRequest {
    init(model: String, messages: [ChatMessage], configuration: LLMConfiguration) {
        self.model = model
        self.messages = messages
        self.temperature = configuration.temperature
        self.maxTokens = configuration.maxTokens
        self.topP = configuration.topP
        self.stream = nil
    }

    func withStream(_ stream: Bool) -> LLMRequest {
        return LLMRequest(
            model: model,
            messages: messages,
            temperature: temperature,
            maxTokens: maxTokens,
            topP: topP,
            stream: stream
        )
    }
}
```

### 使用示例

```swift
let config = LLMConfiguration(
    baseURL: URL(string: "https://api.deepseek.com/v1")!,
    apiKey: "your-api-key",
    model: "deepseek-chat"
)

let service = OpenAICompatibleService(configuration: config)

let messages = [
    ChatMessage(role: .system, content: "你是一个有帮助的助手"),
    ChatMessage(role: .user, content: "用 Swift 写一个冒泡排序")
]

let request = LLMRequest(model: config.model, messages: messages, configuration: config)

do {
    let response = try await service.send(request)
    if let message = response.message {
        print("AI 回复：\(message.content)")
        print("Token 用量：输入 \(response.usage.promptTokens)，输出 \(response.usage.completionTokens)")
    }
} catch let error as LLMError {
    print("LLM 错误：\(error.localizedDescription)")
} catch {
    print("未知错误：\(error.localizedDescription)")
}
```

> 💡 **提示**：DeepSeek、通义千问、智谱 GLM 等国内模型大多兼容 OpenAI API 格式，只需修改 `baseURL` 和 `model` 即可切换。这就是协议层设计的价值——一套代码，多模型复用。

---

## 5. 流式输出（SSE）实现

### SSE 原理简介

SSE（Server-Sent Events）是 LLM API 流式输出的标准协议。与一次性返回完整响应不同，SSE 让服务器逐个 Token 推送数据，实现"打字机"效果。

```
普通请求：客户端 → 请求 → 服务器 → 完整响应 → 客户端
                                              （等待 5-30 秒）

SSE 请求：  客户端 → 请求 → 服务器 → Token1 → Token2 → ... → [DONE]
                                          （首字 0.5-2 秒，逐字输出）
```

SSE 数据格式：

```
data: {"id":"chatcmpl-123","choices":[{"delta":{"content":"你"},"finish_reason":null}]}

data: {"id":"chatcmpl-123","choices":[{"delta":{"content":"好"},"finish_reason":null}]}

data: {"id":"chatcmpl-123","choices":[{"delta":{"content":"！"},"finish_reason":null}]}

data: [DONE]
```

每条消息以 `data: ` 开头，以空行分隔。最后以 `data: [DONE]` 表示结束。

### URLSession bytes 实现流式接收

Swift 的 `URLSession.shared.bytes(for:)` 方法天然支持 SSE 流式读取。它返回一个 `AsyncBytes` 序列，可以逐行读取服务器推送的数据：

```swift
let (bytes, response) = try await URLSession.shared.bytes(for: urlRequest)

for try await line in bytes.lines {
    if line.hasPrefix("data: ") {
        let jsonString = String(line.dropFirst(6))
        if jsonString == "[DONE]" { break }
        if let data = jsonString.data(using: .utf8),
           let chunk = try? JSONDecoder().decode(StreamChunk.self, from: data) {
            onToken(chunk.deltaContent ?? "")
        }
    }
}
```

### 完整的 StreamingLLMService

上一节的 `OpenAICompatibleService` 已经包含了流式输出实现。这里我们再封装一个更易用的 `StreamingService`，专门处理流式场景：

```swift
import Foundation

final class StreamingChatService {
    private let service: OpenAICompatibleService
    private var accumulatedContent = ""
    private var onTokenCallback: ((String) -> Void)?
    private var onCompleteCallback: ((Result<ChatMessage, Error>) -> Void)?
    private var usageCallback: ((LLMResponse.Usage) -> Void)?

    init(service: OpenAICompatibleService) {
        self.service = service
    }

    func onToken(_ callback: @escaping (String) -> Void) -> Self {
        self.onTokenCallback = callback
        return self
    }

    func onComplete(_ callback: @escaping (Result<ChatMessage, Error>) -> Void) -> Self {
        self.onCompleteCallback = callback
        return self
    }

    func onUsage(_ callback: @escaping (LLMResponse.Usage) -> Void) -> Self {
        self.usageCallback = callback
        return self
    }

    func send(messages: [ChatMessage]) {
        accumulatedContent = ""

        let request = LLMRequest(
            model: service.configuration.model,
            messages: messages,
            configuration: service.configuration
        )

        Task {
            do {
                let stream = service.stream(request)

                for try await chunk in stream {
                    if let token = chunk.deltaContent {
                        accumulatedContent += token
                        onTokenCallback?(token)
                    }

                    if chunk.isFinished {
                        let message = ChatMessage(role: .assistant, content: accumulatedContent)
                        onCompleteCallback?(.success(message))
                    }
                }
            } catch {
                onCompleteCallback?(.failure(error))
            }
        }
    }

    func cancel() {
        service.cancel()
    }
}
```

### ViewModel 中使用流式输出

```swift
import Foundation

@MainActor
final class ChatViewModel: ObservableObject {
    @Published var messages: [ChatMessage] = []
    @Published var isStreaming = false
    @Published var currentStreamingContent = ""
    @Published var tokenUsage = TokenUsage()

    private var streamingService: StreamingChatService?

    func sendMessage(_ content: String) {
        let userMessage = ChatMessage(role: .user, content: content)
        messages.append(userMessage)

        currentStreamingContent = ""
        isStreaming = true

        let service = OpenAICompatibleService(configuration: currentConfiguration)

        streamingService = StreamingChatService(service: service)
            .onToken { [weak self] token in
                Task { @MainActor in
                    self?.currentStreamingContent += token
                }
            }
            .onComplete { [weak self] result in
                Task { @MainActor in
                    self?.isStreaming = false
                    switch result {
                    case .success(let message):
                        self?.messages.append(message)
                        self?.currentStreamingContent = ""
                    case .failure(let error):
                        self?.currentStreamingContent = "错误：\(error.localizedDescription)"
                    }
                }
            }

        streamingService?.send(messages: messages)
    }

    func stopStreaming() {
        streamingService?.cancel()
        isStreaming = false
        if !currentStreamingContent.isEmpty {
            let partialMessage = ChatMessage(role: .assistant, content: currentStreamingContent)
            messages.append(partialMessage)
            currentStreamingContent = ""
        }
    }
}
```

> ⚠️ **警告**：流式输出时，UI 更新频率可能很高（每秒数十次 Token）。务必确保 UI 更新在主线程执行，并考虑对 Token 做节流（throttle），避免过于频繁的 UI 刷新导致卡顿。

---

## 6. API Key 安全管理

### ⚠️ 绝不在客户端硬编码 API Key

这是最重要的安全原则。将 API Key 硬编码在 App 中，等于把钥匙放在门口：

```swift
// ❌ 绝对不要这样做！
let apiKey = "sk-xxxxxxxxxxxxxxxxxxxxxxxx"
```

反编译 IPA 文件只需要一行命令：

```bash
strings YourApp.ipa | grep "sk-"
```

一旦 Key 泄露，攻击者可以：
- 用你的额度调用 API，产生高额账单
- 访问你的 API 账户信息
- 滥用你的模型权限

### 方案一：后端代理（推荐）

```
┌──────────┐     ┌──────────────┐     ┌──────────┐
│  iOS App │────▶│  你的后端服务  │────▶│  LLM API │
└──────────┘     └──────────────┘     └──────────┘
   无 API Key        存储 API Key         验证 Key
   只发用户请求      鉴权 + 限流 + 转发     返回结果
```

后端代理架构：

```swift
// iOS 端：只调用你自己的后端，不接触 LLM API Key
let config = LLMConfiguration(
    baseURL: URL(string: "https://your-server.com/api/v1")!,
    apiKey: "",  // 不需要 LLM API Key
    model: "gpt-4o"
)

// 后端使用用户的身份 Token 鉴权
var request = URLRequest(url: url)
request.setValue("Bearer \(userAccessToken)", forHTTPHeaderField: "Authorization")
```

后端实现（Node.js 示例）：

```javascript
// 后端：验证用户身份后，用服务端 Key 调用 LLM API
app.post('/api/v1/chat/completions', authMiddleware, rateLimit, async (req, res) => {
    const response = await fetch('https://api.openai.com/v1/chat/completions', {
        method: 'POST',
        headers: {
            'Authorization': `Bearer ${process.env.OPENAI_API_KEY}`,
            'Content-Type': 'application/json'
        },
        body: JSON.stringify(req.body)
    });

    // 流式转发
    res.setHeader('Content-Type', 'text/event-stream');
    response.body.pipe(res);
});
```

### 方案二：Keychain 存储 + 环境配置

如果暂时没有后端，可以用 Keychain 安全存储 API Key，配合环境配置区分开发/生产：

```swift
import Security

final class APIKeyManager {
    static let shared = APIKeyManager()

    private init() {}

    func saveAPIKey(_ key: String, service: String = "com.yourapp.llm") {
        let data = Data(key.utf8)
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: "api_key"
        ]

        SecItemDelete(query as CFDictionary)

        let attributes: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: "api_key",
            kSecValueData as String: data,
            kSecAttrAccessible as String: kSecAttrAccessibleWhenUnlockedThisDeviceOnly
        ]

        SecItemAdd(attributes as CFDictionary, nil)
    }

    func loadAPIKey(service: String = "com.yourapp.llm") -> String? {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: "api_key",
            kSecReturnData as String: true,
            kSecMatchLimit as String: kSecMatchLimitOne
        ]

        var result: AnyObject?
        SecItemCopyMatching(query as CFDictionary, &result)

        guard let data = result as? Data else { return nil }
        return String(data: data, encoding: .utf8)
    }

    func deleteAPIKey(service: String = "com.yourapp.llm") {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: "api_key"
        ]
        SecItemDelete(query as CFDictionary)
    }
}
```

用户输入 Key 后保存到 Keychain：

```swift
// 首次使用时，让用户输入 API Key
func saveUserAPIKey(_ key: String) {
    APIKeyManager.shared.saveAPIKey(key)
}

// 使用时从 Keychain 读取
func getConfiguration() -> LLMConfiguration? {
    guard let apiKey = APIKeyManager.shared.loadAPIKey() else {
        return nil
    }
    return LLMConfiguration(
        baseURL: URL(string: "https://api.deepseek.com/v1")!,
        apiKey: apiKey,
        model: "deepseek-chat"
    )
}
```

### 方案三：Cloud Function / API Gateway

使用云函数（如阿里云函数计算、腾讯云 SCF、AWS Lambda）作为中间层：

```
┌──────────┐     ┌────────────────┐     ┌──────────┐
│  iOS App │────▶│  Cloud Function │────▶│  LLM API │
└──────────┘     └────────────────┘     └──────────┘
   发送请求         验证 + 注入 Key        处理请求
   无 Key           转发请求              返回结果
```

优势：
- 无需维护服务器，按调用次数计费
- Key 存储在云函数的环境变量中，不会暴露给客户端
- 可以在云函数中添加鉴权、限流、日志等逻辑

### 安全最佳实践

| 实践 | 说明 | 优先级 |
|------|------|--------|
| **后端代理** | API Key 只存在于服务端 | ⭐⭐⭐⭐⭐ |
| **Keychain 存储** | 不明文存储，使用 `WhenUnlockedThisDeviceOnly` | ⭐⭐⭐⭐ |
| **环境隔离** | 开发/测试/生产使用不同 Key | ⭐⭐⭐⭐ |
| **Key 轮换** | 定期更换 API Key | ⭐⭐⭐⭐ |
| **用量监控** | 设置 API 调用上限和告警 | ⭐⭐⭐⭐ |
| **证书锁定** | 防止中间人攻击截获 Key | ⭐⭐⭐ |
| **混淆代码** | 增加逆向工程难度（但不是安全措施） | ⭐⭐ |
| **禁用越狱检测** | 越狱设备 Key 更容易泄露 | ⭐⭐⭐ |

> ⚠️ **警告**：方案二（Keychain 存储）和方案三（Cloud Function）只是缓兵之计。对于面向公众的 App，**必须使用后端代理**。任何存储在客户端的密钥都有被提取的风险。

---

## 7. Token 用量追踪与成本控制

### Token 计算原理

Token 是 LLM 计费的基本单位。大致规则：

| 语言 | 1 Token ≈ |
|------|-----------|
| 英文 | 4 个字符 / 0.75 个单词 |
| 中文 | 1-2 个汉字 |

例如：
- "Hello, how are you?" ≈ 6 Tokens
- "你好，今天天气怎么样？" ≈ 8-10 Tokens

> 💡 **提示**：不同模型的 Tokenizer 不同，同样文本在不同模型中的 Token 数可能有差异。精确计算需要使用对应模型的 Tokenizer。

### 用量追踪实现

```swift
struct TokenUsage {
    var promptTokens: Int = 0
    var completionTokens: Int = 0
    var totalTokens: Int = 0

    var totalCost: Double = 0.0

    mutating func add(_ usage: LLMResponse.Usage, model: PricingModel) {
        promptTokens += usage.promptTokens
        completionTokens += usage.completionTokens
        totalTokens += usage.totalTokens

        let inputCost = Double(usage.promptTokens) * model.inputPricePerToken
        let outputCost = Double(usage.completionTokens) * model.outputPricePerToken
        totalCost += inputCost + outputCost
    }
}

struct PricingModel {
    let name: String
    let inputPricePerMillion: Double
    let outputPricePerMillion: Double

    var inputPricePerToken: Double {
        inputPricePerMillion / 1_000_000
    }

    var outputPricePerToken: Double {
        outputPricePerMillion / 1_000_000
    }

    static let gpt4o = PricingModel(name: "gpt-4o", inputPricePerMillion: 2.5, outputPricePerMillion: 10.0)
    static let gpt4oMini = PricingModel(name: "gpt-4o-mini", inputPricePerMillion: 0.15, outputPricePerMillion: 0.6)
    static let deepseekV3 = PricingModel(name: "deepseek-chat", inputPricePerMillion: 1.0, outputPricePerMillion: 2.0)
    static let qwenPlus = PricingModel(name: "qwen-plus", inputPricePerMillion: 0.4, outputPricePerMillion: 1.2)
}
```

### 用量持久化存储

```swift
import Foundation

final class UsageTracker {
    static let shared = UsageTracker()

    private let defaults = UserDefaults.standard
    private let usageKey = "llm_token_usage"
    private let dateKey = "llm_usage_date"

    private init() {
        resetIfNewDay()
    }

    var todayUsage: TokenUsage {
        guard let data = defaults.data(forKey: usageKey),
              let usage = try? JSONDecoder().decode(TokenUsage.self, from: data) else {
            return TokenUsage()
        }
        return usage
    }

    func record(_ usage: LLMResponse.Usage, model: PricingModel) {
        var current = todayUsage
        current.add(usage, model: model)

        if let data = try? JSONEncoder().encode(current) {
            defaults.set(data, forKey: usageKey)
        }
    }

    func recordStreamUsage(promptTokens: Int, completionTokens: Int, model: PricingModel) {
        let usage = LLMResponse.Usage(
            promptTokens: promptTokens,
            completionTokens: completionTokens,
            totalTokens: promptTokens + completionTokens
        )
        record(usage, model: model)
    }

    private func resetIfNewDay() {
        let today = Calendar.current.startOfDay(for: Date())
        let savedDate = defaults.object(forKey: dateKey) as? Date ?? .distantPast

        if Calendar.current.isDate(today, inSameDayAs: savedDate) == false {
            defaults.removeObject(forKey: usageKey)
            defaults.set(today, forKey: dateKey)
        }
    }
}
```

### 成本估算工具

```swift
struct CostEstimator {
    static func estimate(
        messages: [ChatMessage],
        model: PricingModel,
        expectedOutputTokens: Int = 500
    ) -> CostEstimate {
        let inputTokens = estimateTokenCount(messages: messages)
        let inputCost = Double(inputTokens) * model.inputPricePerToken
        let outputCost = Double(expectedOutputTokens) * model.outputPricePerToken

        return CostEstimate(
            estimatedInputTokens: inputTokens,
            estimatedOutputTokens: expectedOutputTokens,
            estimatedCost: inputCost + outputCost,
            model: model.name
        )
    }

    static func estimateTokenCount(messages: [ChatMessage]) -> Int {
        var total = 0
        for message in messages {
            total += estimateTokenCount(text: message.content)
            total += 4
        }
        total += 2
        return total
    }

    static func estimateTokenCount(text: String) -> Int {
        var count = 0
        for scalar in text.unicodeScalars {
            if scalar.value >= 0x4E00 && scalar.value <= 0x9FFF {
                count += 2
            } else {
                count += 1
            }
        }
        return max(1, count / 4 + count / 2)
    }
}

struct CostEstimate {
    let estimatedInputTokens: Int
    let estimatedOutputTokens: Int
    let estimatedCost: Double
    let model: String

    var formattedCost: String {
        if estimatedCost < 0.01 {
            return String(format: "$%.4f", estimatedCost)
        }
        return String(format: "$%.2f", estimatedCost)
    }
}
```

### 用户配额管理策略

| 策略 | 说明 | 适用场景 |
|------|------|---------|
| **每日限额** | 每天限制 Token 用量 | 免费用户 |
| **每月配额** | 每月分配固定额度 | 订阅用户 |
| **按次计费** | 每次调用扣除对应额度 | 付费用户 |
| **分级配额** | 不同会员等级不同额度 | 会员体系 |
| **动态调整** | 根据服务器负载调整限额 | 高并发场景 |

```swift
struct QuotaManager {
    var dailyLimit: Int
    var monthlyLimit: Int

    func canSend(estimatedTokens: Int) -> Bool {
        let daily = UsageTracker.shared.todayUsage
        return daily.totalTokens + estimatedTokens <= dailyLimit
    }

    func remainingDailyQuota() -> Int {
        let daily = UsageTracker.shared.todayUsage
        return max(0, dailyLimit - daily.totalTokens)
    }

    func quotaStatus() -> QuotaStatus {
        let daily = UsageTracker.shared.todayUsage
        let usageRatio = Double(daily.totalTokens) / Double(dailyLimit)

        if usageRatio >= 1.0 {
            return .exceeded
        } else if usageRatio >= 0.8 {
            return .warning(remaining: dailyLimit - daily.totalTokens)
        } else {
            return .available(remaining: dailyLimit - daily.totalTokens)
        }
    }
}

enum QuotaStatus {
    case available(remaining: Int)
    case warning(remaining: Int)
    case exceeded
}
```

---

## 8. 错误重试与降级策略

### 常见错误类型与处理策略

| 错误类型 | HTTP 状态码 | 原因 | 处理策略 |
|---------|-----------|------|---------|
| **认证失败** | 401 | API Key 无效或过期 | 提示用户检查 Key |
| **频率限制** | 429 | 请求过于频繁 | 指数退避重试 |
| **上下文超长** | 400 | 输入 Token 超限 | 截断历史消息 |
| **模型不可用** | 404 | 模型名错误或下线 | 切换备用模型 |
| **服务器错误** | 500/502/503 | 服务端临时故障 | 重试 + 降级 |
| **网络超时** | - | 网络不稳定 | 重试 + 离线提示 |
| **额度用尽** | 402 | 账户余额不足 | 提示充值 / 降级 |

### 指数退避重试实现

```swift
final class RetryableLLMService: LLMServiceProtocol {
    let configuration: LLMConfiguration
    private let baseService: OpenAICompatibleService
    private let maxRetries: Int
    private let baseDelay: TimeInterval

    init(
        configuration: LLMConfiguration,
        maxRetries: Int = 3,
        baseDelay: TimeInterval = 1.0
    ) {
        self.configuration = configuration
        self.baseService = OpenAICompatibleService(configuration: configuration)
        self.maxRetries = maxRetries
        self.baseDelay = baseDelay
    }

    func send(_ request: LLMRequest) async throws -> LLMResponse {
        var lastError: Error?

        for attempt in 0..<maxRetries {
            do {
                return try await baseService.send(request)
            } catch let error as LLMError {
                lastError = error

                if !isRetriable(error) {
                    throw error
                }

                let delay = retryDelay(for: attempt, error: error)
                try await Task.sleep(nanoseconds: UInt64(delay * 1_000_000_000))
            } catch {
                lastError = error

                let delay = retryDelay(for: attempt, error: nil)
                try await Task.sleep(nanoseconds: UInt64(delay * 1_000_000_000))
            }
        }

        throw lastError ?? LLMError.serverError(0, "重试次数耗尽")
    }

    func stream(_ request: LLMRequest) -> AsyncThrowingStream<StreamChunk, Error> {
        baseService.stream(request)
    }

    func cancel() {
        baseService.cancel()
    }

    private func isRetriable(_ error: LLMError) -> Bool {
        switch error {
        case .rateLimited, .serverError, .networkError:
            return true
        case .invalidAPIKey, .contextLengthExceeded, .modelNotFound, .cancelled, .quotaExceeded:
            return false
        case .invalidURL, .invalidResponse, .streamDisconnected:
            return false
        }
    }

    private func retryDelay(for attempt: Int, error: LLMError?) -> TimeInterval {
        if case .rateLimited(let retryAfter) = error, let after = retryAfter {
            return after
        }

        let jitter = Double.random(in: 0...0.5)
        return baseDelay * pow(2.0, Double(attempt)) + jitter
    }
}
```

### 模型降级

当主模型不可用时，自动切换到备用模型：

```swift
final class FallbackLLMService: LLMServiceProtocol {
    let configuration: LLMConfiguration
    private let services: [OpenAICompatibleService]

    init(configurations: [LLMConfiguration]) {
        self.configuration = configurations.first!
        self.services = configurations.map { OpenAICompatibleService(configuration: $0) }
    }

    func send(_ request: LLMRequest) async throws -> LLMResponse {
        var lastError: Error?

        for (index, service) in services.enumerated() {
            do {
                var request = request
                request = LLMRequest(
                    model: service.configuration.model,
                    messages: request.messages,
                    temperature: request.temperature,
                    maxTokens: request.maxTokens,
                    topP: request.topP,
                    stream: request.stream
                )
                return try await service.send(request)
            } catch {
                lastError = error
                if index < services.count - 1 {
                    print("模型 \(service.configuration.model) 失败，切换到备用模型")
                }
            }
        }

        throw lastError ?? LLMError.serverError(0, "所有模型均不可用")
    }

    func stream(_ request: LLMRequest) -> AsyncThrowingStream<StreamChunk, Error> {
        AsyncThrowingStream { continuation in
            Task {
                var lastError: Error?

                for (index, service) in self.services.enumerated() {
                    do {
                        var request = request
                        request = LLMRequest(
                            model: service.configuration.model,
                            messages: request.messages,
                            temperature: request.temperature,
                            maxTokens: request.maxTokens,
                            topP: request.topP,
                            stream: request.stream
                        )

                        let stream = service.stream(request)
                        for try await chunk in stream {
                            continuation.yield(chunk)
                        }
                        continuation.finish()
                        return
                    } catch {
                        lastError = error
                        if index < self.services.count - 1 {
                            continue
                        }
                    }
                }

                continuation.finish(throwing: lastError ?? LLMError.serverError(0, "所有模型均不可用"))
            }
        }
    }

    func cancel() {
        services.forEach { $0.cancel() }
    }
}
```

降级配置示例：

```swift
let fallbackService = FallbackLLMService(configurations: [
    LLMConfiguration(
        baseURL: URL(string: "https://api.openai.com/v1")!,
        apiKey: "sk-xxx",
        model: "gpt-4o"
    ),
    LLMConfiguration(
        baseURL: URL(string: "https://api.deepseek.com/v1")!,
        apiKey: "sk-yyy",
        model: "deepseek-chat"
    ),
    LLMConfiguration(
        baseURL: URL(string: "https://dashscope.aliyuncs.com/compatible-mode/v1")!,
        apiKey: "sk-zzz",
        model: "qwen-plus"
    )
])
```

### 离线降级提示

```swift
final class OfflineAwareService: LLMServiceProtocol {
    let configuration: LLMConfiguration
    private let wrappedService: RetryableLLMService
    private let reachability: NetworkReachability

    init(configuration: LLMConfiguration, reachability: NetworkReachability) {
        self.configuration = configuration
        self.wrappedService = RetryableLLMService(configuration: configuration)
        self.reachability = reachability
    }

    func send(_ request: LLMRequest) async throws -> LLMResponse {
        guard reachability.isConnected else {
            throw LLMError.networkError(
                NSError(domain: "offline", code: -1009, userInfo: [
                    NSLocalizedDescriptionKey: "网络不可用，请检查网络连接"
                ])
            )
        }
        return try await wrappedService.send(request)
    }

    func stream(_ request: LLMRequest) -> AsyncThrowingStream<StreamChunk, Error> {
        wrappedService.stream(request)
    }

    func cancel() {
        wrappedService.cancel()
    }
}

struct NetworkReachability {
    var isConnected: Bool {
        let monitor = NWPathMonitor()
        let semaphore = DispatchSemaphore(value: 0)
        var connected = false

        monitor.pathUpdateHandler = { path in
            connected = path.status == .satisfied
            semaphore.signal()
        }
        monitor.start(queue: DispatchQueue.global())

        _ = semaphore.wait(timeout: .now() + 2)
        monitor.cancel()
        return connected
    }
}
```

> 💡 **提示**：生产环境中，建议将重试、降级、离线检测组合使用。调用链为：`OfflineAwareService → FallbackLLMService → RetryableLLMService → OpenAICompatibleService`。这种装饰器模式让你可以灵活组合各种策略。

---

## 9. 完整代码示例

### 完整的 LLM API 调用封装

将前面所有模块整合为一个可直接使用的封装：

```swift
import Foundation

// MARK: - 配置

struct LLMConfiguration {
    var baseURL: URL
    var apiKey: String
    var model: String
    var temperature: Double = 0.7
    var maxTokens: Int = 2048
    var topP: Double = 1.0
    var timeoutInterval: TimeInterval = 60

    static func deepSeek(apiKey: String) -> LLMConfiguration {
        LLMConfiguration(
            baseURL: URL(string: "https://api.deepseek.com/v1")!,
            apiKey: apiKey,
            model: "deepseek-chat"
        )
    }

    static func qwenPlus(apiKey: String) -> LLMConfiguration {
        LLMConfiguration(
            baseURL: URL(string: "https://dashscope.aliyuncs.com/compatible-mode/v1")!,
            apiKey: apiKey,
            model: "qwen-plus"
        )
    }

    static func gpt4oMini(apiKey: String) -> LLMConfiguration {
        LLMConfiguration(
            baseURL: URL(string: "https://api.openai.com/v1")!,
            apiKey: apiKey,
            model: "gpt-4o-mini"
        )
    }
}

// MARK: - 数据模型

struct ChatMessage: Codable, Identifiable {
    let id: UUID
    let role: Role
    var content: String
    let timestamp: Date

    enum Role: String, Codable {
        case system, user, assistant
    }

    init(role: Role, content: String) {
        self.id = UUID()
        self.role = role
        self.content = content
        self.timestamp = Date()
    }
}

struct LLMRequest: Codable {
    let model: String
    let messages: [ChatMessage]
    let temperature: Double?
    let maxTokens: Int?
    let topP: Double?
    let stream: Bool?

    enum CodingKeys: String, CodingKey {
        case model, messages, temperature, topP, stream
        case maxTokens = "max_tokens"
    }
}

struct LLMResponse: Codable {
    let id: String
    let model: String
    let choices: [Choice]
    let usage: Usage

    var content: String {
        choices.first?.message.content ?? ""
    }

    struct Choice: Codable {
        let message: ResponseMessage
        let finishReason: String?
        enum CodingKeys: String, CodingKey {
            case message
            case finishReason = "finish_reason"
        }
    }

    struct ResponseMessage: Codable {
        let role: String
        let content: String
    }

    struct Usage: Codable {
        let promptTokens: Int
        let completionTokens: Int
        let totalTokens: Int
        enum CodingKeys: String, CodingKey {
            case promptTokens = "prompt_tokens"
            case completionTokens = "completion_tokens"
            case totalTokens = "total_tokens"
        }
    }
}

struct StreamChunk: Codable {
    let choices: [StreamChoice]

    var deltaContent: String? { choices.first?.delta?.content }
    var isFinished: Bool { choices.first?.finishReason != nil }

    struct StreamChoice: Codable {
        let delta: Delta?
        let finishReason: String?
        enum CodingKeys: String, CodingKey {
            case delta
            case finishReason = "finish_reason"
        }
    }

    struct Delta: Codable {
        let content: String?
    }
}

// MARK: - 错误

enum LLMError: LocalizedError {
    case invalidURL
    case invalidAPIKey
    case networkError(Error)
    case rateLimited(retryAfter: TimeInterval?)
    case contextLengthExceeded
    case modelNotFound
    case serverError(Int, String)
    case invalidResponse
    case cancelled
    case quotaExceeded

    var errorDescription: String? {
        switch self {
        case .invalidURL: return "API 地址无效"
        case .invalidAPIKey: return "API Key 无效"
        case .networkError(let e): return "网络错误：\(e.localizedDescription)"
        case .rateLimited(let t): return t.map { "请求频繁，\($0)秒后重试" } ?? "请求频繁，请稍后重试"
        case .contextLengthExceeded: return "对话超出上下文长度"
        case .modelNotFound: return "模型不可用"
        case .serverError(let c, let m): return "服务器错误(\(c))：\(m)"
        case .invalidResponse: return "响应格式异常"
        case .cancelled: return "已取消"
        case .quotaExceeded: return "额度已用尽"
        }
    }
}

// MARK: - 核心服务

final class LLMService {
    private let configuration: LLMConfiguration
    private var currentTask: Task<Void, Never>?

    init(configuration: LLMConfiguration) {
        self.configuration = configuration
    }

    // 普通请求
    func send(messages: [ChatMessage]) async throws -> LLMResponse {
        let request = LLMRequest(
            model: configuration.model,
            messages: messages,
            temperature: configuration.temperature,
            maxTokens: configuration.maxTokens,
            topP: configuration.topP,
            stream: false
        )

        let urlRequest = try buildURLRequest(from: request)
        let (data, response) = try await URLSession.shared.data(for: urlRequest)
        return try parseResponse(data: data, response: response)
    }

    // 流式请求
    func stream(messages: [ChatMessage]) -> AsyncThrowingStream<StreamChunk, Error> {
        let request = LLMRequest(
            model: configuration.model,
            messages: messages,
            temperature: configuration.temperature,
            maxTokens: configuration.maxTokens,
            topP: configuration.topP,
            stream: true
        )

        return AsyncThrowingStream { [weak self] continuation in
            guard let self else {
                continuation.finish()
                return
            }

            let task = Task {
                do {
                    let urlRequest = try self.buildURLRequest(from: request)
                    let (bytes, response) = try await URLSession.shared.bytes(for: urlRequest)

                    guard let http = response as? HTTPURLResponse, http.statusCode == 200 else {
                        let http = response as? HTTPURLResponse
                        throw LLMError.serverError(http?.statusCode ?? 0, "请求失败")
                    }

                    for try await line in bytes.lines {
                        guard !Task.isCancelled else { break }
                        guard line.hasPrefix("data: ") else { continue }

                        let json = String(line.dropFirst(6))
                        if json == "[DONE]" {
                            continuation.finish()
                            return
                        }

                        if let data = json.data(using: .utf8),
                           let chunk = try? JSONDecoder().decode(StreamChunk.self, from: data) {
                            continuation.yield(chunk)
                        }
                    }
                    continuation.finish()
                } catch is CancellationError {
                    continuation.finish(throwing: LLMError.cancelled)
                } catch let error as LLMError {
                    continuation.finish(throwing: error)
                } catch {
                    continuation.finish(throwing: LLMError.networkError(error))
                }
            }

            self.currentTask = task
            continuation.onTermination = { _ in task.cancel() }
        }
    }

    func cancel() {
        currentTask?.cancel()
        currentTask = nil
    }

    private func buildURLRequest(from request: LLMRequest) throws -> URLRequest {
        guard let url = URL(string: configuration.baseURL.absoluteString + "/chat/completions") else {
            throw LLMError.invalidURL
        }

        var urlRequest = URLRequest(url: url)
        urlRequest.httpMethod = "POST"
        urlRequest.setValue("application/json", forHTTPHeaderField: "Content-Type")
        urlRequest.setValue("Bearer \(configuration.apiKey)", forHTTPHeaderField: "Authorization")
        urlRequest.timeoutInterval = configuration.timeoutInterval
        urlRequest.httpBody = try JSONEncoder().encode(request)
        return urlRequest
    }

    private func parseResponse(data: Data, response: URLResponse) throws -> LLMResponse {
        guard let http = response as? HTTPURLResponse else {
            throw LLMError.invalidResponse
        }

        guard http.statusCode == 200 else {
            let body = String(data: data, encoding: .utf8) ?? ""
            switch http.statusCode {
            case 401: throw LLMError.invalidAPIKey
            case 429: throw LLMError.rateLimited(retryAfter: nil)
            case 404: throw LLMError.modelNotFound
            default: throw LLMError.serverError(http.statusCode, body)
            }
        }

        return try JSONDecoder().decode(LLMResponse.self, from: data)
    }
}

// MARK: - 带重试的服务

final class RetryableLLMService {
    private let service: LLMService
    private let maxRetries: Int

    init(service: LLMService, maxRetries: Int = 3) {
        self.service = service
        self.maxRetries = maxRetries
    }

    func send(messages: [ChatMessage]) async throws -> LLMResponse {
        var lastError: Error?

        for attempt in 0..<maxRetries {
            do {
                return try await service.send(messages: messages)
            } catch let error as LLMError {
                lastError = error
                guard isRetriable(error) else { throw error }

                let delay = pow(2.0, Double(attempt)) + Double.random(in: 0...0.5)
                try await Task.sleep(nanoseconds: UInt64(delay * 1_000_000_000))
            } catch {
                lastError = error
                let delay = pow(2.0, Double(attempt)) + Double.random(in: 0...0.5)
                try await Task.sleep(nanoseconds: UInt64(delay * 1_000_000_000))
            }
        }

        throw lastError ?? LLMError.serverError(0, "重试耗尽")
    }

    func stream(messages: [ChatMessage]) -> AsyncThrowingStream<StreamChunk, Error> {
        service.stream(messages: messages)
    }

    func cancel() { service.cancel() }

    private func isRetriable(_ error: LLMError) -> Bool {
        switch error {
        case .rateLimited, .serverError, .networkError: return true
        default: return false
        }
    }
}
```

### SwiftUI 调用示例

```swift
import SwiftUI

@MainActor
final class ChatViewModel: ObservableObject {
    @Published var messages: [ChatMessage] = []
    @Published var inputText = ""
    @Published var isStreaming = false
    @Published var streamingContent = ""
    @Published var errorMessage: String?

    private var service: RetryableLLMService?
    private var tokenUsage = TokenUsage()

    func configure(apiKey: String) {
        let config = LLMConfiguration.deepSeek(apiKey: apiKey)
        let baseService = LLMService(configuration: config)
        service = RetryableLLMService(service: baseService)
    }

    func sendMessage() {
        let text = inputText.trimmingCharacters(in: .whitespacesAndNewlines)
        guard !text.isEmpty, let service else { return }

        let userMessage = ChatMessage(role: .user, content: text)
        messages.append(userMessage)
        inputText = ""
        isStreaming = true
        streamingContent = ""
        errorMessage = nil

        let allMessages = messages

        Task {
            do {
                let stream = service.stream(messages: allMessages)
                for try await chunk in stream {
                    if let token = chunk.deltaContent {
                        streamingContent += token
                    }
                }

                let assistantMessage = ChatMessage(role: .assistant, content: streamingContent)
                messages.append(assistantMessage)
                streamingContent = ""
            } catch {
                errorMessage = error.localizedDescription
            }
            isStreaming = false
        }
    }

    func stopStreaming() {
        service?.cancel()
        isStreaming = false
        if !streamingContent.isEmpty {
            messages.append(ChatMessage(role: .assistant, content: streamingContent))
            streamingContent = ""
        }
    }
}

struct ChatView: View {
    @StateObject private var viewModel = ChatViewModel()

    var body: some View {
        VStack(spacing: 0) {
            ScrollViewReader { proxy in
                ScrollView {
                    LazyVStack(spacing: 12) {
                        ForEach(viewModel.messages) { message in
                            MessageBubble(message: message)
                                .id(message.id)
                        }

                        if viewModel.isStreaming && !viewModel.streamingContent.isEmpty {
                            MessageBubble(
                                message: ChatMessage(role: .assistant, content: viewModel.streamingContent)
                            )
                            .id("streaming")
                        }
                    }
                    .padding()
                }
                .onChange(of: viewModel.messages.count) { _ in
                    withAnimation {
                        proxy.scrollTo(viewModel.messages.last?.id, anchor: .bottom)
                    }
                }
                .onChange(of: viewModel.streamingContent) { _ in
                    withAnimation {
                        proxy.scrollTo("streaming", anchor: .bottom)
                    }
                }
            }

            if let error = viewModel.errorMessage {
                Text(error)
                    .font(.caption)
                    .foregroundColor(.red)
                    .padding(.horizontal)
            }

            HStack(spacing: 12) {
                TextField("输入消息...", text: $viewModel.inputText, axis: .vertical)
                    .textFieldStyle(.roundedBorder)
                    .lineLimit(1...5)
                    .onSubmit { viewModel.sendMessage() }

                if viewModel.isStreaming {
                    Button("停止") { viewModel.stopStreaming() }
                        .buttonStyle(.bordered)
                        .tint(.red)
                } else {
                    Button("发送") { viewModel.sendMessage() }
                        .buttonStyle(.borderedProminent)
                        .disabled(viewModel.inputText.trimmingCharacters(in: .whitespacesAndNewlines).isEmpty)
                }
            }
            .padding()
        }
        .navigationTitle("AI 助手")
        .onAppear {
            viewModel.configure(apiKey: APIKeyManager.shared.loadAPIKey() ?? "")
        }
    }
}

struct MessageBubble: View {
    let message: ChatMessage

    var body: some View {
        HStack {
            if message.role == .user { Spacer(minLength: 60) }

            VStack(alignment: message.role == .user ? .trailing : .leading, spacing: 4) {
                Text(message.content)
                    .padding(12)
                    .background(
                        message.role == .user
                            ? Color.blue.opacity(0.15)
                            : Color.gray.opacity(0.1)
                    )
                    .cornerRadius(16)
            }

            if message.role == .assistant { Spacer(minLength: 60) }
        }
    }
}
```

> 💡 **提示**：以上代码可以直接复制到 Xcode 项目中使用。只需在 `onAppear` 中配置你的 API Key，或通过设置页面让用户输入。完整项目建议搭配后端代理使用。

---

## 10. 性能优化

### 请求并发控制

多个请求同时发送可能导致 429 错误（频率限制）。使用信号量控制并发数：

```swift
final class ConcurrencyLimiter {
    private let semaphore: DispatchSemaphore
    private let queue = DispatchQueue(label: "llm.concurrency", attributes: .concurrent)

    init(maxConcurrent: Int = 3) {
        self.semaphore = DispatchSemaphore(value: maxConcurrent)
    }

    func execute<T>(_ block: @escaping () async throws -> T) async throws -> T {
        semaphore.wait()
        defer { semaphore.signal() }
        return try await block()
    }
}

// 使用
let limiter = ConcurrencyLimiter(maxConcurrent: 2)

func sendWithLimit(messages: [ChatMessage]) async throws -> LLMResponse {
    try await limiter.execute {
        try await service.send(messages: messages)
    }
}
```

### 响应缓存策略

对相同输入缓存 AI 响应，避免重复调用：

```swift
final class CachedLLMService {
    private let service: LLMService
    private var cache: [String: LLMResponse] = [:]
    private let cacheQueue = DispatchQueue(label: "llm.cache")
    private let maxCacheSize = 100

    init(service: LLMService) {
        self.service = service
    }

    func send(messages: [ChatMessage], useCache: Bool = true) async throws -> LLMResponse {
        if useCache {
            let key = cacheKey(for: messages)
            if let cached = cacheQueue.sync({ cache[key] }) {
                return cached
            }

            let response = try await service.send(messages: messages)

            cacheQueue.async(flags: .barrier) { [weak self] in
                guard let self else { return }
                self.cache[key] = response
                if self.cache.count > self.maxCacheSize {
                    self.cache.removeValue(forKey: self.cache.keys.first ?? "")
                }
            }

            return response
        }

        return try await service.send(messages: messages)
    }

    func clearCache() {
        cacheQueue.async(flags: .barrier) { [weak self] in
            self?.cache.removeAll()
        }
    }

    private func cacheKey(for messages: [ChatMessage]) -> String {
        let content = messages.map { "\($0.role.rawValue):\($0.content)" }.joined(separator: "|")
        return String(content.sha256Hash.prefix(32))
    }
}
```

> ⚠️ **警告**：缓存 AI 响应时要考虑时效性。对于需要实时性的场景（如代码生成、数据分析），应禁用缓存。缓存更适合 FAQ 类的固定问答场景。

### 上下文窗口管理

对话越长，Token 消耗越大。需要策略性地管理上下文窗口：

| 策略 | 说明 | 优点 | 缺点 |
|------|------|------|------|
| **截断历史** | 只保留最近 N 条消息 | 简单有效 | 丢失早期上下文 |
| **摘要压缩** | 用 AI 总结历史对话 | 保留关键信息 | 额外 Token 消耗 |
| **滑动窗口** | 保留系统提示 + 最近 N 条 + 摘要 | 平衡效果与成本 | 实现复杂 |
| **Token 计数** | 精确计算 Token 数，到阈值时截断 | 最精确 | 需要本地 Tokenizer |

```swift
final class ContextWindowManager {
    let maxContextTokens: Int
    let reservedOutputTokens: Int

    init(maxContextTokens: Int = 8000, reservedOutputTokens: Int = 2000) {
        self.maxContextTokens = maxContextTokens
        self.reservedOutputTokens = reservedOutputTokens
    }

    var maxInputTokens: Int {
        maxContextTokens - reservedOutputTokens
    }

    func trimMessages(_ messages: [ChatMessage], systemPrompt: String? = nil) -> [ChatMessage] {
        var result: [ChatMessage] = []
        var totalTokens = 0

        if let prompt = systemPrompt {
            let msg = ChatMessage(role: .system, content: prompt)
            result.append(msg)
            totalTokens += estimateTokens(prompt) + 4
        }

        for message in messages.reversed() {
            let tokens = estimateTokens(message.content) + 4
            if totalTokens + tokens > maxInputTokens { break }
            result.insert(message, at: result.count > 0 ? (systemPrompt != nil ? 1 : 0) : 0)
            totalTokens += tokens
        }

        return result
    }

    private func estimateTokens(_ text: String) -> Int {
        var count = 0
        for scalar in text.unicodeScalars {
            count += (scalar.value >= 0x4E00 && scalar.value <= 0x9FFF) ? 2 : 1
        }
        return max(1, count / 3)
    }
}
```

### 取消请求的处理

用户可能中途取消请求（如切换页面、点击停止）。正确处理取消非常重要：

```swift
final class CancellableChatService {
    private var activeTask: Task<Void, Never>?

    func stream(
        messages: [ChatMessage],
        onToken: @escaping (String) -> Void,
        onComplete: @escaping (Result<String, Error>) -> Void
    ) {
        cancel()

        activeTask = Task { @MainActor in
            do {
                var content = ""
                let stream = service.stream(messages: messages)

                for try await chunk in stream {
                    guard !Task.isCancelled else {
                        onComplete(.success(content))
                        return
                    }
                    if let token = chunk.deltaContent {
                        content += token
                        onToken(token)
                    }
                }
                onComplete(.success(content))
            } catch is CancellationError {
                // 用户主动取消，不视为错误
            } catch {
                onComplete(.failure(error))
            }
        }
    }

    func cancel() {
        activeTask?.cancel()
        activeTask = nil
    }
}
```

取消时的注意事项：

| 场景 | 处理方式 |
|------|---------|
| 用户点击"停止" | 取消 Task，保存已生成的内容 |
| 页面退出 | 取消 Task，不保存部分内容 |
| 网络中断 | 自动取消，提示用户重试 |
| 切换对话 | 取消当前请求，开始新请求 |

> 💡 **提示**：取消请求后，服务端可能仍在处理（并计费）。部分 API 支持 `cancel` 端点来通知服务端停止生成。如果成本敏感，建议实现服务端取消逻辑。

---

## 小结

1. **从开发到产品的跨越**：集成大模型 API 让 App 从"用 AI 写"升级为"在 App 里用 AI"，开启智能化的用户体验
2. **选型是第一步**：根据合规性、成本、延迟、能力四个维度选择合适的 LLM API，国内模型兼容 OpenAI 格式让切换成本极低
3. **架构设计先行**：协议层（`LLMServiceProtocol`）+ 实现层（`OpenAICompatibleService`）的分层设计，让一套代码支持多模型
4. **Swift 原生即可**：`URLSession` + `async/await` 完全够用，无需引入第三方网络库
5. **流式输出是标配**：SSE + `URLSession.bytes` 实现逐 Token 输出，大幅提升用户体验
6. **API Key 安全第一**：后端代理是唯一推荐的生产方案，Keychain 只是开发阶段的过渡方案
7. **Token 就是钱**：追踪用量、估算成本、管理配额，避免账单失控
8. **优雅地处理错误**：指数退避重试 + 模型降级 + 离线检测，让 App 在各种异常情况下都能优雅应对
9. **性能优化不可忽视**：并发控制、响应缓存、上下文窗口管理、请求取消，这些细节决定了 App 的稳定性
10. **组合使用策略**：将重试、降级、缓存、限流等策略通过装饰器模式组合，构建健壮的 LLM 调用链

---

← [AI辅助UI设计工具](./AI辅助UI设计工具.md) | [构建AI对话界面](./构建AI对话界面.md) →