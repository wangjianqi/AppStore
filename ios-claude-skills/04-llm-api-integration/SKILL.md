---
name: llm-api-integration
description: 涉及 iOS App 集成大模型 API、LLM 调用、流式输出 SSE、AI 对话界面、Token 管理、API Key 安全、OpenAI 兼容格式、国产大模型 API 的任务
---

# LLM API 集成

## 网络层架构

### 服务协议
```swift
protocol LLMServiceProtocol {
    func chat(messages: [ChatMessage]) async throws -> ChatResponse
    func chatStream(messages: [ChatMessage]) -> AsyncThrowingStream<StreamChunk, Error>
    func cancelCurrentRequest()
}
```

### 请求模型
- 使用 OpenAI 兼容格式（大多数国内外 API 支持）
- 请求体包含 model、messages、temperature、max_tokens、stream
- messages 数组中每条消息有 role（system/user/assistant）和 content

### 响应模型
- 非流式：完整响应包含 choices 数组，每个 choice 有 message 和 usage
- 流式：SSE 格式，每个 data 行是一个 delta 片段
- usage 包含 prompt_tokens 和 completion_tokens

---

## 流式输出（SSE）

### 实现要点
- 使用 URLSession.bytes(byLine:) 逐行读取 SSE 数据
- 解析 "data: " 前缀的行
- 处理 "data: [DONE]" 终止信号
- 在 ViewModel 中使用 @Published 属性更新 UI
- 支持取消流式请求（Task 取消）

### 错误处理
- 网络超时：设置合理的 URLSessionConfiguration.timeoutInterval
- API 限流：实现指数退避重试
- Token 超限：截断上下文或提示用户
- JSON 解析失败：跳过无效行继续处理

---

## API Key 安全

### 安全策略
- ⚠️ 绝不在客户端代码中硬编码 API Key
- 推荐方案：通过自有后端代理转发 API 请求
- 备选方案：Keychain 存储 + 环境配置（Debug/Release 不同 Key）
- 最差方案：Info.plist 存储（仍可被提取，仅用于开发阶段）

### 后端代理架构
```
iOS App → 自有后端（验证用户身份）→ LLM API
```

---

## Token 管理

### 用量追踪
- 每次请求记录 prompt_tokens 和 completion_tokens
- 本地累计存储（UserDefaults 或 SwiftData）
- 提供用量统计 UI

### 成本控制
- 设置单次请求 max_tokens 上限
- 设置用户每日/每月配额
- 上下文截断策略：保留最近 N 条消息或 Token 预算内消息
- 优先使用较便宜的模型处理简单请求

---

## 国产大模型 API 适配

### OpenAI 兼容格式
大多数国产大模型支持 OpenAI 兼容格式，只需修改 baseURL：

| 模型 | baseURL |
|------|---------|
| 通义千问 | https://dashscope.aliyuncs.com/compatible-mode/v1 |
| DeepSeek | https://api.deepseek.com/v1 |
| 智谱 GLM | https://open.bigmodel.cn/api/paas/v4 |

### 差异注意
- 部分 API 的错误码格式不同
- 部分模型不支持 function calling
- 流式输出的 SSE 格式可能有细微差异

---

## SwiftUI 集成

### ViewModel 模式
- 使用 @Observable 或 ObservableObject
- messages 数组驱动 UI 刷新
- 流式输出时逐字更新最后一条 assistant 消息
- isLoading 状态控制发送按钮和打字指示器

### UI 组件
- LazyVStack + ScrollViewReader 实现消息列表
- 自动滚动到底部
- 打字指示器动画
- Markdown 渲染（MarkdownUI 库或 AttributedString）
