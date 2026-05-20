---
name: network-api
description: 涉及网络请求、API 调用、后端接口、HTTP、URLSession、数据解析的任务
---

# 网络 / API 层

## 网络层架构
- 基础层：**URLSession**（禁止引入 Alamofire，除非已存在）
- 封装层：`NetworkService.swift` 统一处理请求、错误、重试
- 接口定义：每个模块有对应 `XxxAPI.swift`，定义 endpoint + 参数

```swift
// 标准请求结构
struct APIRequest<T: Decodable> {
    let path: String
    let method: HTTPMethod
    let parameters: [String: Any]?
    let requiresAuth: Bool
}
```

## 后端：Spring Boot
- Base URL 通过 `Config.swift` 管理（区分 dev / staging / prod）
- 认证：**Bearer Token**，存 Keychain，请求时自动注入 Header
- 通用响应格式：
  ```json
  {
    "code": 0,
    "message": "success",
    "data": { ... }
  }
  ```
- `code != 0` 统一走 `AppError.serverError(code:message:)` 处理

## AI API 集成
### OpenAI / Whisper
- API Key **禁止硬编码在客户端**，必须通过自有后端中转
- 语音转文字：优先用 `whisper-1`，文件限制 25MB，超出需分段
- 流式输出：使用 `URLSessionDataTask` + chunked response 解析

### Anthropic Claude
- 同上，通过后端代理，禁止客户端直接调用
- 长文本处理：注意 token 限制，超出时做客户端截断

### 本地 Whisper（whisper.cpp）
- 通过 C++ 桥接调用，封装为 `WhisperService`
- 音频格式：16kHz, 16-bit, mono WAV
- 滑动窗口去重：使用项目内 `TranscriptDeduplicator`

## 错误处理规范
```swift
enum NetworkError: AppError {
    case noConnection          // 无网络
    case timeout               // 超时（> 30s）
    case serverError(Int)      // HTTP 5xx
    case clientError(Int)      // HTTP 4xx
    case decodingFailed        // JSON 解析失败
    case unauthorized          // 401，触发重新登录
}
```
- 401 错误：自动刷新 Token，失败则跳登录页
- 网络无连接：展示离线提示，不弹 Alert
- 5xx 错误：展示"服务异常，请稍后重试"

## 数据解析
- 全部使用 **Codable**，禁止手动解析 JSON
- 服务端字段命名蛇形（`snake_case`），客户端驼峰（`camelCase`）：
  ```swift
  decoder.keyDecodingStrategy = .convertFromSnakeCase
  ```
- 可选字段用 `?`，避免因缺字段导致整体解析失败

## 缓存策略
- GET 请求：URLCache 默认缓存（根据 Cache-Control）
- 用户数据：写入 CoreData，离线可读
- 图片：Kingfisher 自动磁盘缓存，限制 200MB

## 调试
- Debug 模式打印完整 request / response（用 `#if DEBUG` 包裹）
- 禁止在 Release 包里打印 API Key 或 Token
