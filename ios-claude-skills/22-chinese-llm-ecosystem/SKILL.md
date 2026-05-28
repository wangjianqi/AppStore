---
name: chinese-llm-ecosystem
description: 涉及国产大模型 API、通义千问、DeepSeek、文心一言、智谱 GLM、国内 AI 合规、算法备案、数据出境、AI 生成内容标识的任务
---

# 国内大模型生态

## API 接入

### OpenAI 兼容格式
大多数国产大模型支持 OpenAI 兼容 API 格式，只需修改 baseURL 和 model 参数：

| 提供商 | baseURL | 模型名 |
|--------|---------|--------|
| 通义千问 | https://dashscope.aliyuncs.com/compatible-mode/v1 | qwen-plus / qwen-turbo / qwen-max |
| DeepSeek | https://api.deepseek.com/v1 | deepseek-chat / deepseek-reasoner |
| 智谱 GLM | https://open.bigmodel.cn/api/paas/v4 | glm-4 / glm-4-flash |
| 月之暗面 | https://api.moonshot.cn/v1 | moonshot-v1-8k / moonshot-v1-32k |
| 讯飞星火 | https://spark-api-open.xf-yun.com/v1 | generalv3.5 / 4.0Ultra |

### Swift 调用示例
```swift
struct QwenService: LLMServiceProtocol {
    private let baseURL = "https://dashscope.aliyuncs.com/compatible-mode/v1"
    private let apiKey: String

    func chat(messages: [ChatMessage]) async throws -> ChatResponse {
        var request = URLRequest(url: URL(string: "\(baseURL)/chat/completions")!)
        request.httpMethod = "POST"
        request.setValue("Bearer \(apiKey)", forHTTPHeaderField: "Authorization")
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")

        let body: [String: Any] = [
            "model": "qwen-plus",
            "messages": messages.map { ["role": $0.role.rawValue, "content": $0.content] }
        ]
        request.httpBody = try JSONSerialization.data(withJSONObject: body)

        let (data, _) = try await URLSession.shared.data(for: request)
        return try JSONDecoder().decode(ChatResponse.self, from: data)
    }
}
```

---

## 国内 AI 合规

### 算法备案判断
- App 内提供 AI 对话功能 → 通常需要备案
- 仅调用 API 展示结果 → 模型提供方已备案，App 方视情况
- 本地模型推理 → 通常不需要
- 仅内部使用 AI 辅助开发 → 不需要

### AI 生成内容标识
- 显式标识：在 AI 生成内容旁显示"AI 生成"标签
- 隐式标识：在 AI 生成图片中嵌入数字水印
- 元数据标识：在内容元数据中标注 AI 生成

### 隐私政策 AI 条款
- 说明 AI 功能的数据处理方式
- 说明对话内容是否存储及存储位置
- 提供关闭 AI 功能的选项
- 声明 AI 生成内容仅供参考

---

## 数据出境合规

### 风险评估
- 调用海外 API（OpenAI/Claude）→ 用户数据出境 → 需评估
- 调用国内 API → 数据不出境 → 合规
- 后端代理 + 脱敏 → 部分合规

### 推荐方案
面向国内用户的 App，优先使用国内大模型 API，避免数据出境合规风险。

### 合规检查清单
- [ ] AI 功能已在隐私政策中说明
- [ ] AI 生成内容已添加标识
- [ ] 用户可选择关闭 AI 功能
- [ ] 已评估是否需要算法备案
- [ ] 已评估数据出境合规性
- [ ] AI 生成内容已过滤违法信息
- [ ] 已添加 AI 生成内容免责声明
