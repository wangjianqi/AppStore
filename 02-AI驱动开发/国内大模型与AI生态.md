# 国内大模型与 AI 生态

> 🎯 **本章目标**：
> - 了解 2023-2025 年国内大模型的发展历程与"百模大战"格局
> - 掌握国内主流大模型 API 的详细对比与选型方法
> - 实战接入通义千问和 DeepSeek API，完成 Swift 调用
> - 熟悉国内 AI 编程工具的功能与使用体验
> - 理解国内 AI 合规法规，掌握数据出境合规方案
> - 获取国内 AI 开发者资源与学习路径

---

## 1. 国内大模型发展现状

### 1.1 2023-2025 国内大模型发展时间线

2023 年被称为"中国大模型元年"。从 ChatGPT 引爆全球到国内厂商密集跟进，短短两年间，国内大模型从"追赶者"逐渐成长为部分领域的"并跑者"。

| 时间 | 里程碑事件 | 意义 |
|------|-----------|------|
| 2023 年 2 月 | 复旦大学发布 MOSS | 国内首个类 ChatGPT 对话模型 |
| 2023 年 3 月 | 百度文心一言发布 | 国内大厂首个大模型产品 |
| 2023 年 4 月 | 阿里通义千问发布 | 阿里入局，开源生态起步 |
| 2023 年 6 月 | 智谱 ChatGLM-6B 开源 | 国内首个可商用的开源大模型 |
| 2023 年 8 月 | 《生成式人工智能服务管理暂行办法》施行 | 国内首部 AI 专项法规 |
| 2023 年 10 月 | 百模大战白热化，超过 200 个大模型 | 行业进入"百模大战"阶段 |
| 2024 年 1 月 | DeepSeek 发布 DeepSeek-MoE | 深度求索以极低成本训练出高性能模型 |
| 2024 年 3 月 | 月之暗面 Kimi 支持 200 万字上下文 | 国产长上下文能力突破 |
| 2024 年 5 月 | 腾讯混元大模型全面开放 | 腾讯正式加入大模型竞赛 |
| 2024 年 9 月 | 通义千问 Qwen2.5 系列发布 | 开源模型多项基准超越 Llama |
| 2025 年 1 月 | DeepSeek-R1 发布 | 推理模型对标 OpenAI o1，全球轰动 |
| 2025 年 3 月 | 通义千问 Qwen3 发布 | 混合推理架构，开源生态进一步壮大 |
| 2025 年 5 月 | 国内大模型 API 价格战白热化 | 多家厂商推出免费额度，开发者受益 |

### 1.2 百模大战格局

截至 2025 年，国内大模型市场已经从"百模大战"的混战阶段，逐步走向"头部集中"的格局。按照厂商背景和能力定位，可以这样划分：

**第一梯队：互联网大厂**

| 厂商 | 模型名称 | 核心优势 | 代表产品 |
|------|---------|---------|---------|
| 阿里巴巴 | 通义千问（Qwen） | 开源生态最强，模型系列最全 | 通义千问 App、通义灵码 |
| 百度 | 文心一言（ERNIE） | 搜索数据积累深，中文理解强 | 文心一言 App、百度 Comate |
| 字节跳动 | 豆包（Doubao） | 内容生态丰富，用户量大 | 豆包 App、MarsCode |
| 腾讯 | 混元（Hunyuan） | 社交数据优势，微信生态整合 | 腾讯元宝 |
| 华为 | 盘古（Pangu） | 行业模型深耕，政企市场强 | 盘古大模型 |

**第二梯队：AI 独角兽**

| 厂商 | 模型名称 | 核心优势 | 代表产品 |
|------|---------|---------|---------|
| 智谱 AI | ChatGLM | 开源先行者，学术基因强 | 智谱清言、CodeGeeX |
| 月之暗面 | Kimi | 超长上下文，文档处理强 | Kimi 智能助手 |
| 深度求索 | DeepSeek | 极致性价比，推理能力强 | DeepSeek App |
| MiniMax | abab | 多模态能力强，语音出色 | 星野、海螺 AI |
| 百川智能 | Baichuan | 开源社区活跃，中文优化 | 百小应 |

**第三梯队：垂直领域与学术机构**

| 类型 | 代表 | 方向 |
|------|------|------|
| 科研机构 | 复旦 MOSS、中科院紫东太初 | 学术研究、开源贡献 |
| 金融领域 | 蚂蚁蚁盾、度小满轩辕 | 金融风控、合规 |
| 医疗领域 | 百度灵医、腾讯觅影 | 医疗诊断辅助 |
| 教育领域 | 好未来 MathGPT、科大讯飞星火 | 教育场景优化 |

### 1.3 国内大模型 vs 海外大模型的核心差异

了解差异才能做出正确选择。国内大模型和海外大模型各有优势，不能简单地说"谁更好"：

| 维度 | 国内大模型 | 海外大模型 |
|------|-----------|-----------|
| **中文理解** | ✅ 明显更强，理解成语、网络用语、文化语境 | ❌ 中文理解较弱，翻译腔明显 |
| **英文能力** | ⚠️ 日常够用，学术写作偏弱 | ✅ 原生英文，学术写作强 |
| **编程能力** | ⚠️ 通用编程尚可，Swift/Rust 等偏弱 | ✅ 编程能力强，覆盖语言广 |
| **合规性** | ✅ 已完成算法备案，国内使用合规 | ❌ 未备案，国内商用有合规风险 |
| **访问便利** | ✅ 国内直连，无需梯子 | ❌ 需要特殊网络环境 |
| **价格** | ✅ 大幅更低，多家提供免费额度 | ❌ 价格较高，免费额度少 |
| **上下文长度** | ✅ Kimi 200 万字，行业领先 | ⚠️ Gemini 100 万 token，其他较短 |
| **推理能力** | ⚠️ DeepSeek-R1 对标 o1，其他偏弱 | ✅ o1/o3 推理能力强 |
| **多模态** | ⚠️ 正在追赶，部分模型已支持 | ✅ GPT-4o、Claude 多模态成熟 |
| **生态工具** | ⚠️ 正在建设，API 生态不如海外 | ✅ 插件、Agent 生态成熟 |

💡 **提示**：对于国内 iOS 开发者来说，最实际的选择是"国内模型为主、海外模型为辅"——日常中文需求用国产模型，编程和英文需求用海外模型。

⚠️ **警告**：如果你的 App 面向国内用户且包含 AI 功能，使用未备案的海外模型 API 存在合规风险，详见本章第 6-7 节。

---

## 2. 国内大模型 API 对比

### 2.1 主流模型详细对比

以下是国内开发者最常使用的 6 个大模型 API 的详细对比：

| 维度 | 通义千问 Qwen | 文心一言 ERNIE | 智谱 GLM | 月之暗面 Kimi | DeepSeek | 讯飞星火 Spark |
|------|-------------|---------------|---------|-------------|----------|--------------|
| **出品方** | 阿里云 | 百度 | 智谱 AI | 月之暗面 | 深度求索 | 科大讯飞 |
| **API 地址** | dashscope.aliyuncs.com | aip.baidubce.com | open.bigmodel.cn | api.moonshot.cn | api.deepseek.com | spark-api.xf-yun.com |
| **旗舰模型** | qwen-max | ernie-4.0-8k | glm-4 | moonshot-v1 | deepseek-chat | spark-4.0-ultra |
| **推理模型** | qwen-plus-thinking | — | glm-4-thinking | — | deepseek-reasoner | — |
| **上下文长度** | 128K | 128K | 128K | 200 万字 | 128K | 128K |
| **输入价格** | ¥0.02/千token | ¥0.03/千token | ¥0.05/千token | ¥0.012/千token | ¥0.001/千token | ¥0.03/千token |
| **输出价格** | ¥0.06/千token | ¥0.09/千token | ¥0.05/千token | ¥0.012/千token | ¥0.002/千token | ¥0.06/千token |
| **免费额度** | 100 万 token | 50 万 token | 100 万 token | 100 万 token | 500 万 token | 200 万 token |
| **OpenAI 兼容** | ✅ | ⚠️ 部分兼容 | ✅ | ✅ | ✅ | ❌ |
| **流式输出** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Function Call** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **算法备案** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |

⚠️ **警告**：以上价格为 2025 年 5 月参考价格，各厂商价格调整频繁，请以官网最新价格为准。国内大模型 API 价格整体呈下降趋势。

### 2.2 编程能力对比

对于 iOS 开发者来说，编程能力是最关键的指标之一。以下是基于实际测试的编程能力对比：

| 测试维度 | 通义千问 | 文心一言 | 智谱 GLM | Kimi | DeepSeek | 讯飞星火 |
|---------|---------|---------|---------|------|----------|---------|
| **Swift 基础** | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| **SwiftUI** | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ |
| **UIKit** | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ |
| **iOS API 调用** | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ |
| **StoreKit** | ⭐⭐⭐ | ⭐⭐ | ⭐⭐ | ⭐⭐ | ⭐⭐⭐ | ⭐⭐ |
| **Python** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **算法题** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| **Debug 能力** | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| **代码解释** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |

💡 **提示**：DeepSeek 在编程能力上表现最为突出，尤其是算法和 Debug 方面，性价比极高。通义千问在 Swift/SwiftUI 方面表现较好，且开源生态完善。

### 2.3 中文能力对比

| 测试维度 | 通义千问 | 文心一言 | 智谱 GLM | Kimi | DeepSeek | 讯飞星火 |
|---------|---------|---------|---------|------|----------|---------|
| **日常对话** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **成语/典故** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ |
| **公文写作** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **长文档理解** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ |
| **中文代码注释** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| **网络用语** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |

### 2.4 选型建议

不同场景推荐不同的模型，没有"万能模型"：

| 使用场景 | 推荐模型 | 理由 |
|---------|---------|------|
| **iOS/Swift 编程** | DeepSeek + 通义千问 | DeepSeek 编程能力最强，通义千问 Swift 支持好 |
| **长文档处理** | Kimi | 200 万字上下文，文档处理能力最强 |
| **中文内容创作** | 文心一言 + 讯飞星火 | 中文写作能力突出，公文场景强 |
| **低成本批量调用** | DeepSeek | 价格最低，输出 ¥0.002/千token |
| **开源/私有化部署** | 通义千问 Qwen | 开源模型系列最全，社区最活跃 |
| **企业级应用** | 通义千问 + 文心一言 | 阿里云/百度云生态完善，企业支持好 |
| **学术研究** | 智谱 GLM | 学术基因强，开源版本丰富 |
| **语音交互** | 讯飞星火 | 语音技术积累深厚，语音合成质量高 |
| **推理/数学** | DeepSeek-R1 | 推理能力对标 o1，数学能力强 |
| **App 内嵌 AI** | DeepSeek + 通义千问 | API 价格低，OpenAI 兼容格式易接入 |

💡 **提示**：推荐采用"双模型策略"——主力用 DeepSeek 处理编程和推理任务，辅助用通义千问处理中文和通用任务。两个模型都兼容 OpenAI API 格式，切换成本低。

---

## 3. API 接入实战：通义千问

### 3.1 注册与 API Key 获取

通义千问的 API 通过阿里云百炼平台提供，注册流程如下：

1. 访问 [阿里云百炼平台](https://bailian.console.aliyun.com/)
2. 使用阿里云账号登录（没有则注册）
3. 进入控制台，点击左侧菜单「API-KEY 管理」
4. 点击「创建 API Key」，系统自动生成
5. 复制并保存 API Key（⚠️ 只显示一次）

```bash
# 设置环境变量（推荐写入 ~/.zshrc）
export DASHSCOPE_API_KEY="sk-xxxxxxxxxxxxxxxxxxxxxxxx"
```

💡 **提示**：新注册用户赠送 100 万 token 免费额度，足够完成本章所有实验。

### 3.2 API 格式说明

通义千问 API 兼容 OpenAI 格式，这意味着你可以用任何支持 OpenAI API 的 SDK 直接对接，只需修改 base_url 和 api_key：

```
基础地址：https://dashscope.aliyuncs.com/compatible-mode/v1
聊天接口：POST https://dashscope.aliyuncs.com/compatible-mode/v1/chat/completions
模型名称：qwen-max（旗舰）、qwen-plus（均衡）、qwen-turbo（快速）
```

请求格式示例：

```bash
curl -X POST "https://dashscope.aliyuncs.com/compatible-mode/v1/chat/completions" \
  -H "Authorization: Bearer $DASHSCOPE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen-max",
    "messages": [
      {"role": "system", "content": "你是一个 iOS 开发助手"},
      {"role": "user", "content": "用 Swift 写一个单例模式"}
    ]
  }'
```

### 3.3 Swift 调用代码示例

以下是一个完整的 Swift 网络请求封装，用于调用通义千问 API：

```swift
import Foundation

struct QwenMessage: Codable {
    let role: String
    let content: String
}

struct QwenRequest: Codable {
    let model: String
    let messages: [QwenMessage]
    let temperature: Double?
    let max_tokens: Int?
}

struct QwenChoice: Codable {
    let message: QwenMessage
    let finish_reason: String?
}

struct QwenResponse: Codable {
    let id: String?
    let choices: [QwenChoice]
    let usage: Usage?

    struct Usage: Codable {
        let prompt_tokens: Int?
        let completion_tokens: Int?
        let total_tokens: Int?
    }
}

class QwenClient {
    private let apiKey: String
    private let baseURL = "https://dashscope.aliyuncs.com/compatible-mode/v1"
    private let session = URLSession.shared

    init(apiKey: String) {
        self.apiKey = apiKey
    }

    func chat(
        messages: [QwenMessage],
        model: String = "qwen-max",
        temperature: Double = 0.7,
        maxTokens: Int = 2048
    ) async throws -> QwenResponse {
        let url = URL(string: "\(baseURL)/chat/completions")!
        var request = URLRequest(url: url)
        request.httpMethod = "POST"
        request.setValue("Bearer \(apiKey)", forHTTPHeaderField: "Authorization")
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")

        let body = QwenRequest(
            model: model,
            messages: messages,
            temperature: temperature,
            max_tokens: maxTokens
        )
        request.httpBody = try JSONEncoder().encode(body)

        let (data, response) = try await session.data(for: request)

        guard let httpResponse = response as? HTTPURLResponse,
              (200...299).contains(httpResponse.statusCode) else {
            let statusCode = (response as? HTTPURLResponse)?.statusCode ?? -1
            throw QwenError.httpError(statusCode: statusCode, body: String(data: data, encoding: .utf8) ?? "")
        }

        return try JSONDecoder().decode(QwenResponse.self, from: data)
    }
}

enum QwenError: Error {
    case httpError(statusCode: Int, body: String)
}
```

调用示例：

```swift
Task {
    let client = QwenClient(apiKey: ProcessInfo.processInfo.environment["DASHSCOPE_API_KEY"] ?? "")

    let messages = [
        QwenMessage(role: "system", content: "你是一个 iOS 开发专家"),
        QwenMessage(role: "user", content: "用 Swift 写一个线程安全的缓存管理器")
    ]

    do {
        let response = try await client.chat(messages: messages)
        if let choice = response.choices.first {
            print(choice.message.content)
        }
        if let usage = response.usage {
            print("Token 用量：输入 \(usage.prompt_tokens ?? 0)，输出 \(usage.completion_tokens ?? 0)")
        }
    } catch {
        print("请求失败：\(error)")
    }
}
```

### 3.4 流式输出接入

流式输出（Streaming）对于聊天类 App 至关重要——用户不需要等几十秒才看到完整回复，而是逐字显示。通义千问支持 SSE（Server-Sent Events）格式的流式输出：

```swift
class QwenStreamClient {
    private let apiKey: String
    private let baseURL = "https://dashscope.aliyuncs.com/compatible-mode/v1"

    init(apiKey: String) {
        self.apiKey = apiKey
    }

    func chatStream(
        messages: [QwenMessage],
        model: String = "qwen-max",
        onChunk: @escaping (String) -> Void,
        onComplete: @escaping (Error?) -> Void
    ) {
        let url = URL(string: "\(baseURL)/chat/completions")!
        var request = URLRequest(url: url)
        request.httpMethod = "POST"
        request.setValue("Bearer \(apiKey)", forHTTPHeaderField: "Authorization")
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")

        let body: [String: Any] = [
            "model": model,
            "messages": messages.map { ["role": $0.role, "content": $0.content] },
            "stream": true
        ]
        request.httpBody = try? JSONSerialization.data(withJSONObject: body)

        let task = URLSession.shared.dataTask(with: request) { data, response, error in
            if let error = error {
                onComplete(error)
                return
            }

            guard let data = data,
                  let text = String(data: data, encoding: .utf8) else {
                onComplete(nil)
                return
            }

            let lines = text.components(separatedBy: "\n")
            for line in lines {
                guard line.hasPrefix("data: ") else { continue }
                let jsonString = String(line.dropFirst(6))
                if jsonString == "[DONE]" { break }

                if let jsonData = jsonString.data(using: .utf8),
                   let json = try? JSONSerialization.jsonObject(with: jsonData) as? [String: Any],
                   let choices = json["choices"] as? [[String: Any]],
                   let delta = choices.first?["delta"] as? [String: Any],
                   let content = delta["content"] as? String {
                    onChunk(content)
                }
            }
            onComplete(nil)
        }
        task.resume()
    }
}
```

使用流式输出：

```swift
let streamClient = QwenStreamClient(apiKey: "your-api-key")

streamClient.chatStream(
    messages: [
        QwenMessage(role: "user", content: "解释 Swift 中的 async/await")
    ],
    onChunk: { chunk in
        print(chunk, terminator: "")
    },
    onComplete: { error in
        if let error = error {
            print("\n流式输出错误：\(error)")
        } else {
            print("\n--- 输出完成 ---")
        }
    }
)
```

⚠️ **警告**：上述流式输出代码使用了简单的 `dataTask`，在生产环境中建议使用 `URLSessionStreamTask` 或第三方 SSE 库来处理更可靠的流式连接。

💡 **提示**：通义千问的 OpenAI 兼容接口意味着你也可以直接使用 OpenAI 的官方 Swift SDK（如 MacPaw/OpenAI），只需修改 `baseURL` 即可。

---

## 4. API 接入实战：DeepSeek

### 4.1 注册与 API Key 获取

DeepSeek 的 API 注册非常简单，且免费额度慷慨：

1. 访问 [DeepSeek 开放平台](https://platform.deepseek.com/)
2. 注册账号（支持手机号/邮箱）
3. 进入「API Keys」页面
4. 点击「创建 API Key」
5. 复制并保存 Key

```bash
# 设置环境变量
export DEEPSEEK_API_KEY="sk-xxxxxxxxxxxxxxxxxxxxxxxx"
```

💡 **提示**：DeepSeek 新用户赠送 500 万 token 免费额度，是所有国内大模型中最慷慨的。加上极低的定价（输出 ¥0.002/千token），非常适合开发阶段大量测试。

### 4.2 API 格式说明

DeepSeek API 完全兼容 OpenAI 格式，迁移成本几乎为零：

```
基础地址：https://api.deepseek.com
聊天接口：POST https://api.deepseek.com/chat/completions
模型名称：
  - deepseek-chat（通用对话，V3 模型）
  - deepseek-reasoner（推理模型，R1 模型）
```

快速测试：

```bash
curl -X POST "https://api.deepseek.com/chat/completions" \
  -H "Authorization: Bearer $DEEPSEEK_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "deepseek-chat",
    "messages": [
      {"role": "user", "content": "用 Swift 实现一个 LRU 缓存"}
    ]
  }'
```

### 4.3 Swift 调用代码示例

由于 DeepSeek 完全兼容 OpenAI 格式，我们可以复用通义千问的代码，只需修改 base URL：

```swift
class DeepSeekClient {
    private let apiKey: String
    private let baseURL = "https://api.deepseek.com"
    private let session = URLSession.shared

    init(apiKey: String) {
        self.apiKey = apiKey
    }

    func chat(
        messages: [QwenMessage],
        model: String = "deepseek-chat",
        temperature: Double = 0.7,
        maxTokens: Int = 4096
    ) async throws -> QwenResponse {
        let url = URL(string: "\(baseURL)/chat/completions")!
        var request = URLRequest(url: url)
        request.httpMethod = "POST"
        request.setValue("Bearer \(apiKey)", forHTTPHeaderField: "Authorization")
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")

        let body = QwenRequest(
            model: model,
            messages: messages,
            temperature: temperature,
            max_tokens: maxTokens
        )
        request.httpBody = try JSONEncoder().encode(body)

        let (data, response) = try await session.data(for: request)

        guard let httpResponse = response as? HTTPURLResponse,
              (200...299).contains(httpResponse.statusCode) else {
            let statusCode = (response as? HTTPURLResponse)?.statusCode ?? -1
            throw QwenError.httpError(statusCode: statusCode, body: String(data: data, encoding: .utf8) ?? "")
        }

        return try JSONDecoder().decode(QwenResponse.self, from: data)
    }
}
```

💡 **提示**：由于通义千问和 DeepSeek 都兼容 OpenAI 格式，建议封装一个统一的 `LLMClient`，通过配置切换不同模型：

```swift
struct LLMConfig {
    let baseURL: String
    let apiKey: String
    let defaultModel: String

    static let qwen = LLMConfig(
        baseURL: "https://dashscope.aliyuncs.com/compatible-mode/v1",
        apiKey: ProcessInfo.processInfo.environment["DASHSCOPE_API_KEY"] ?? "",
        defaultModel: "qwen-max"
    )

    static let deepseek = LLMConfig(
        baseURL: "https://api.deepseek.com",
        apiKey: ProcessInfo.processInfo.environment["DEEPSEEK_API_KEY"] ?? "",
        defaultModel: "deepseek-chat"
    )
}

class LLMClient {
    private let config: LLMConfig
    private let session = URLSession.shared

    init(config: LLMConfig) {
        self.config = config
    }

    func chat(messages: [QwenMessage], model: String? = nil) async throws -> QwenResponse {
        let url = URL(string: "\(config.baseURL)/chat/completions")!
        var request = URLRequest(url: url)
        request.httpMethod = "POST"
        request.setValue("Bearer \(config.apiKey)", forHTTPHeaderField: "Authorization")
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")

        let body = QwenRequest(
            model: model ?? config.defaultModel,
            messages: messages,
            temperature: 0.7,
            max_tokens: 4096
        )
        request.httpBody = try JSONEncoder().encode(body)

        let (data, response) = try await session.data(for: request)

        guard let httpResponse = response as? HTTPURLResponse,
              (200...299).contains(httpResponse.statusCode) else {
            let statusCode = (response as? HTTPURLResponse)?.statusCode ?? -1
            throw QwenError.httpError(statusCode: statusCode, body: String(data: data, encoding: .utf8) ?? "")
        }

        return try JSONDecoder().decode(QwenResponse.self, from: data)
    }
}
```

使用统一客户端切换模型：

```swift
Task {
    let qwenClient = LLMClient(config: .qwen)
    let deepseekClient = LLMClient(config: .deepseek)

    let messages = [
        QwenMessage(role: "user", content: "用 Swift 写一个观察者模式")
    ]

    let qwenResult = try await qwenClient.chat(messages: messages)
    print("通义千问：\(qwenResult.choices.first?.message.content ?? "")")

    let deepseekResult = try await deepseekClient.chat(messages: messages)
    print("DeepSeek：\(deepseekResult.choices.first?.message.content ?? "")")
}
```

### 4.4 DeepSeek 特有功能：DeepThink 推理

DeepSeek-R1（对应 API 模型名 `deepseek-reasoner`）是 DeepSeek 最具特色的功能——推理模型。它会在给出最终回答前，先进行一段"思考过程"（reasoning_content），类似于 OpenAI 的 o1 模型。

推理模型的响应结构有所不同，增加了 `reasoning_content` 字段：

```swift
struct DeepSeekReasoningMessage: Codable {
    let role: String
    let content: String?
    let reasoningContent: String?

    enum CodingKeys: String, CodingKey {
        case role, content
        case reasoningContent = "reasoning_content"
    }
}

struct DeepSeekReasoningChoice: Codable {
    let message: DeepSeekReasoningMessage
    let finishReason: String?

    enum CodingKeys: String, CodingKey {
        case message
        case finishReason = "finish_reason"
    }
}

struct DeepSeekReasoningResponse: Codable {
    let id: String?
    let choices: [DeepSeekReasoningChoice]
}
```

调用推理模型：

```swift
func reason(messages: [QwenMessage]) async throws -> DeepSeekReasoningResponse {
    let url = URL(string: "\(config.baseURL)/chat/completions")!
    var request = URLRequest(url: url)
    request.httpMethod = "POST"
    request.setValue("Bearer \(config.apiKey)", forHTTPHeaderField: "Authorization")
    request.setValue("application/json", forHTTPHeaderField: "Content-Type")

    let body = QwenRequest(
        model: "deepseek-reasoner",
        messages: messages,
        temperature: nil,
        max_tokens: nil
    )
    request.httpBody = try JSONEncoder().encode(body)

    let (data, response) = try await session.data(for: request)

    guard let httpResponse = response as? HTTPURLResponse,
          (200...299).contains(httpResponse.statusCode) else {
        let statusCode = (response as? HTTPURLResponse)?.statusCode ?? -1
        throw QwenError.httpError(statusCode: statusCode, body: String(data: data, encoding: .utf8) ?? "")
    }

    return try JSONDecoder().decode(DeepSeekReasoningResponse.self, from: data)
}
```

使用推理模型：

```swift
Task {
    let client = LLMClient(config: .deepseek)

    let messages = [
        QwenMessage(role: "user", content: "一个 App 有 100 个用户，每天新增 10%，多少天后用户超过 10000？")
    ]

    let result = try await client.reason(messages: messages)
    if let choice = result.choices.first {
        print("🧠 思考过程：\(choice.message.reasoningContent ?? "")")
        print("📝 最终回答：\(choice.message.content ?? "")")
    }
}
```

⚠️ **警告**：推理模型（deepseek-reasoner）不支持设置 `temperature`、`top_p`、`presence_penalty` 等参数，也不支持流式输出中的 `system` 消息。调用时请勿传入这些参数，否则会报错。

💡 **提示**：推理模型适合处理复杂逻辑问题、数学题、代码 Debug 等需要"深度思考"的场景。日常对话和简单编程任务用 `deepseek-chat` 即可，速度更快、成本更低。

---

## 5. 国内 AI 编程工具

国内 AI 编程工具正在快速发展，部分工具已经可以替代或补充 GitHub Copilot。以下是目前最值得关注的四款工具。

### 5.1 通义灵码（阿里云）

**简介**：通义灵码是阿里云推出的 AI 编程助手，基于通义千问大模型，支持代码补全、对话、代码解释等功能。

**安装方式**：
- VS Code 插件：搜索「通义灵码」安装
- JetBrains 插件：在插件市场搜索「通义灵码」安装

**核心功能**：

| 功能 | 说明 |
|------|------|
| 行级代码补全 | 根据上下文自动补全代码，支持多行补全 |
| 行间对话 | 在编辑器内直接与 AI 对话，无需切换窗口 |
| 代码解释 | 选中代码，一键获取解释 |
| 单元测试生成 | 自动为函数生成测试用例 |
| 代码注释 | 自动生成中文注释 |
| 提交信息生成 | 分析代码变更，自动生成 commit message |

**使用体验**：

- ✅ 中文注释和文档生成质量高
- ✅ 与阿里云生态深度集成
- ✅ 免费使用，无调用次数限制
- ⚠️ Swift/Objective-C 支持一般
- ⚠️ 复杂逻辑补全偶有"幻觉"

### 5.2 CodeGeeX（智谱 AI）

**简介**：CodeGeeX 是智谱 AI 推出的 AI 编程助手，基于 ChatGLM 模型，是国内最早的一批 AI 编程工具之一。

**安装方式**：
- VS Code 插件：搜索「CodeGeeX」安装
- JetBrains 插件：在插件市场搜索「CodeGeeX」安装

**核心功能**：

| 功能 | 说明 |
|------|------|
| 代码补全 | 支持多语言代码补全 |
| 代码翻译 | 将代码从一种语言翻译为另一种语言 |
| 对话模式 | 侧边栏对话，支持代码问答 |
| 代码解释 | 选中代码获取解释 |
| 模板代码 | 提供常用代码模板 |

**使用体验**：

- ✅ 代码翻译功能独特，支持 100+ 语言互译
- ✅ 完全免费，无需注册即可使用
- ✅ 支持离线模式（轻量模型）
- ⚠️ 补全速度有时较慢
- ⚠️ iOS 开发相关能力偏弱

### 5.3 Baidu Comate（百度）

**简介**：Baidu Comate 是百度推出的 AI 编程助手，基于文心大模型，主打企业级开发场景。

**安装方式**：
- VS Code 插件：搜索「Baidu Comate」安装
- JetBrains 插件：在插件市场搜索「Baidu Comate」安装

**核心功能**：

| 功能 | 说明 |
|------|------|
| 代码补全 | 上下文感知的代码补全 |
| 对话助手 | 自然语言对话编程 |
| 代码解释 | 选中代码获取详细解释 |
| 单测生成 | 自动生成单元测试 |
| 代码优化 | 分析代码并提供优化建议 |
| 私有化部署 | 支持企业私有化部署 |

**使用体验**：

- ✅ 企业级功能完善，支持私有化部署
- ✅ 中文理解能力强，文档生成质量高
- ✅ 与百度云生态集成
- ⚠️ 免费版功能有限制
- ⚠️ Swift 支持较弱

### 5.4 豆包 MarsCode（字节跳动）

**简介**：MarsCode 是字节跳动推出的 AI 编程助手，基于豆包大模型，同时提供云端 IDE 和本地插件两种形态。

**安装方式**：
- VS Code 插件：搜索「MarsCode」安装
- JetBrains 插件：在插件市场搜索「MarsCode」安装
- 云端 IDE：访问 [marscode.com](https://www.marscode.com/)

**核心功能**：

| 功能 | 说明 |
|------|------|
| 代码补全 | 智能代码补全，支持多行 |
| AI 对话 | 编辑器内对话编程 |
| 代码解释 | 选中代码获取解释 |
| Bug 修复 | 分析代码并提供修复建议 |
| 云端 IDE | 浏览器内完整开发环境 |
| 项目模板 | 提供多种项目模板快速起步 |

**使用体验**：

- ✅ 云端 IDE 免配置，开箱即用
- ✅ 免费使用，功能较完整
- ✅ 对前端开发支持较好
- ⚠️ iOS/Swift 支持一般
- ⚠️ 云端 IDE 性能受网络影响

### 5.5 国内 AI 编程工具对比

| 维度 | 通义灵码 | CodeGeeX | Baidu Comate | MarsCode |
|------|---------|----------|-------------|----------|
| **出品方** | 阿里云 | 智谱 AI | 百度 | 字节跳动 |
| **底层模型** | 通义千问 | ChatGLM | 文心一言 | 豆包 |
| **VS Code** | ✅ | ✅ | ✅ | ✅ |
| **JetBrains** | ✅ | ✅ | ✅ | ✅ |
| **云端 IDE** | ❌ | ❌ | ❌ | ✅ |
| **代码补全** | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **对话质量** | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **Swift 支持** | ⭐⭐⭐ | ⭐⭐ | ⭐⭐ | ⭐⭐ |
| **Python 支持** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **Java 支持** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **中文注释** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **私有化部署** | ✅ | ❌ | ✅ | ❌ |
| **价格** | 免费 | 免费 | 免费+付费版 | 免费 |
| **iOS 开发推荐度** | ⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ |

💡 **提示**：对于 iOS 开发者，国内 AI 编程工具在 Swift 支持上整体不如 GitHub Copilot 和 Cursor。建议将国内工具作为辅助，主力编程仍使用 Trae/Cursor + Copilot 组合，国内工具用于中文文档生成、注释编写等场景。

---

## 6. 国内 AI 合规实操

### 6.1 《生成式人工智能服务管理暂行办法》逐条解读

2023 年 8 月 15 日，《生成式人工智能服务管理暂行办法》（以下简称《暂行办法》）正式施行，这是中国首部专门针对生成式 AI 的行政法规。对于开发者来说，这是必须了解的法规。

**核心条款解读**：

| 条款 | 内容摘要 | 对开发者的要求 |
|------|---------|-------------|
| 第五条 | 提供者应当依法开展预训练、优化训练等数据处理活动 | 训练数据需合法合规，不得侵犯他人知识产权 |
| 第六条 | 提供者应当采取有效措施防范未成年人过度依赖 | App 如面向未成年人，需添加使用时长限制 |
| 第七条 | 提供者应当对生成内容进行标识 | AI 生成内容必须标注"由 AI 生成" |
| 第八条 | 提供者应当承担网络信息内容生产者法定责任 | 你对 AI 生成的内容负法律责任 |
| 第九条 | 提供者应当防范生成虚假信息 | 需建立内容审核机制 |
| 第十条 | 提供者应当保护用户个人信息 | 不得将用户输入用于训练（未经同意） |
| 第十一条 | 提供者应当提供投诉举报渠道 | App 内需提供反馈渠道 |
| 第十二条 | 提供者应当开展安全评估 | 上线前需完成安全评估 |
| 第十三条 | 提供者应当进行算法备案 | 使用 AI 算法需向网信部门备案 |
| 第十四条 | 提供者应当建立用户权益保护机制 | 用户有权删除 AI 生成记录 |

### 6.2 关键要点详解

**算法备案**

算法备案是《暂行办法》中最重要的合规要求之一。如果你的 App 包含 AI 功能，你需要：

1. 登录 [互联网信息服务算法备案系统](https://beian.cac.gov.cn/)
2. 填报算法基本信息（算法类型、应用领域等）
3. 提交算法安全评估报告
4. 等待审核通过（通常需要 1-3 个月）

⚠️ **警告**：未完成算法备案的 AI 服务不得上线运营。如果你的 App 使用了第三方已备案的 API（如通义千问、DeepSeek），通常不需要单独备案，但需确认 API 提供方的备案状态。

**安全评估**

安全评估是上线前的必经环节，主要评估以下方面：

| 评估维度 | 评估内容 | 常见问题 |
|---------|---------|---------|
| 内容安全 | 是否可能生成违法有害信息 | 政治敏感、暴力色情、虚假信息 |
| 数据安全 | 用户数据是否得到保护 | 数据泄露、未授权使用 |
| 算法安全 | 算法是否存在偏见或歧视 | 性别歧视、地域歧视 |
| 系统安全 | 系统是否可能被攻击 | 提示注入、越狱攻击 |
| 权益保护 | 用户权益是否得到保障 | 知情权、删除权、退出权 |

**内容标识**

AI 生成内容必须进行标识，具体要求：

| 标识类型 | 适用场景 | 实现方式 |
|---------|---------|---------|
| 显式标识 | 用户可见的 AI 生成内容 | 在内容旁标注"AI 生成"或"由 AI 辅助创作" |
| 隐式标识 | AI 生成内容的元数据 | 在文件元数据中嵌入 AI 生成标记 |
| 水印标识 | AI 生成的图片/视频 | 添加不可见水印 |

**用户权益保护**

| 权益 | 要求 | 实现方式 |
|------|------|---------|
| 知情权 | 用户需知晓正在与 AI 交互 | 首次使用时弹窗提示 |
| 删除权 | 用户可删除 AI 交互记录 | 设置中提供"清除对话记录"功能 |
| 退出权 | 用户可选择不使用 AI 功能 | 提供 AI 功能开关 |
| 投诉权 | 用户可投诉 AI 生成内容 | App 内提供投诉入口 |

### 6.3 含 AI 功能的 App ICP 备案注意事项

如果你的 App 包含 AI 功能，ICP 备案时需要额外注意：

| 注意事项 | 说明 | 操作建议 |
|---------|------|---------|
| 服务类型选择 | 需选择"含 AI 生成内容"类型 | 在 ICP 备案系统中如实勾选 |
| 算法备案号 | 需填写算法备案号 | 先完成算法备案，再进行 ICP 备案 |
| 安全评估报告 | 需上传安全评估报告 | 提前完成安全评估 |
| 内容审核机制 | 需说明内容审核方案 | 提供审核流程文档 |
| 用户协议更新 | 需在隐私政策中说明 AI 功能 | 更新隐私政策，增加 AI 相关条款 |

### 6.4 AI 生成内容标识要求

在 App 内展示 AI 生成内容时，需要遵循以下标识规范：

**文本内容标识**：

```swift
import UIKit

class AIGeneratedLabel: UIView {
    private let contentLabel = UILabel()
    private let badgeLabel = UILabel()

    init(text: String) {
        super.init(frame: .zero)
        setupViews(text: text)
    }

    required init?(coder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }

    private func setupViews(text: String) {
        badgeLabel.text = "AI 生成"
        badgeLabel.font = .systemFont(ofSize: 10, weight: .medium)
        badgeLabel.textColor = .systemBackground
        badgeLabel.backgroundColor = .systemGray
        badgeLabel.layer.cornerRadius = 4
        badgeLabel.clipsToBounds = true
        badgeLabel.textAlignment = .center
        badgeLabel.translatesAutoresizingMaskIntoConstraints = false

        contentLabel.text = text
        contentLabel.font = .systemFont(ofSize: 15)
        contentLabel.numberOfLines = 0
        contentLabel.translatesAutoresizingMaskIntoConstraints = false

        addSubview(badgeLabel)
        addSubview(contentLabel)

        NSLayoutConstraint.activate([
            badgeLabel.topAnchor.constraint(equalTo: topAnchor, constant: 8),
            badgeLabel.leadingAnchor.constraint(equalTo: leadingAnchor, constant: 8),
            badgeLabel.widthAnchor.constraint(equalToConstant: 50),
            badgeLabel.heightAnchor.constraint(equalToConstant: 18),

            contentLabel.topAnchor.constraint(equalTo: badgeLabel.bottomAnchor, constant: 6),
            contentLabel.leadingAnchor.constraint(equalTo: leadingAnchor, constant: 8),
            contentLabel.trailingAnchor.constraint(equalTo: trailingAnchor, constant: -8),
            contentLabel.bottomAnchor.constraint(equalTo: bottomAnchor, constant: -8)
        ])
    }
}
```

**图片水印标识**：

```swift
import UIKit

extension UIImage {
    func addAIWatermark(text: String = "AI 生成") -> UIImage? {
        let renderer = UIGraphicsImageRenderer(size: size)
        return renderer.image { context in
            draw(in: CGRect(origin: .zero, size: size))

            let paragraphStyle = NSMutableParagraphStyle()
            paragraphStyle.alignment = .right

            let attrs: [NSAttributedString.Key: Any] = [
                .font: UIFont.systemFont(ofSize: 14, weight: .medium),
                .foregroundColor: UIColor.white.withAlphaComponent(0.7),
                .paragraphStyle: paragraphStyle
            ]

            let textRect = CGRect(
                x: size.width - 120,
                y: size.height - 30,
                width: 110,
                height: 25
            )
            text.draw(in: textRect, withAttributes: attrs)
        }
    }
}
```

### 6.5 合规检查清单

在提交含 AI 功能的 App 前，请逐项检查以下清单：

| 序号 | 检查项 | 是否必须 | 状态 |
|------|-------|---------|------|
| 1 | 算法备案已完成（或使用已备案的第三方 API） | ✅ 必须 | ☐ |
| 2 | 安全评估报告已出具 | ✅ 必须 | ☐ |
| 3 | AI 生成内容已添加显式标识 | ✅ 必须 | ☐ |
| 4 | AI 生成图片已添加水印 | ✅ 必须 | ☐ |
| 5 | 首次使用 AI 功能时有提示弹窗 | ✅ 必须 | ☐ |
| 6 | 隐私政策已更新 AI 相关条款 | ✅ 必须 | ☐ |
| 7 | 用户可删除 AI 交互记录 | ✅ 必须 | ☐ |
| 8 | App 内有投诉/反馈入口 | ✅ 必须 | ☐ |
| 9 | AI 功能可关闭/退出 | ⚠️ 建议 | ☐ |
| 10 | 内容审核机制已建立 | ✅ 必须 | ☐ |
| 11 | ICP 备案已包含 AI 服务类型 | ✅ 必须 | ☐ |
| 12 | 用户协议已说明 AI 生成内容免责 | ⚠️ 建议 | ☐ |
| 13 | 未成年人使用限制已设置 | ⚠️ 视场景 | ☐ |
| 14 | AI 生成内容元数据已添加标识 | ⚠️ 建议 | ☐ |
| 15 | 数据出境合规已评估 | ✅ 必须 | ☐ |

💡 **提示**：如果使用国内已备案的 API（如通义千问、DeepSeek），算法备案和安全评估通常由 API 提供方完成，你只需确保在 App 内做好内容标识和用户权益保护即可。

---

## 7. 数据出境合规

### 7.1 调用海外 API 的法律风险

很多开发者习惯直接调用 OpenAI、Anthropic 等海外大模型 API，但这在中国法律框架下存在明确的风险：

| 风险类型 | 具体风险 | 法律依据 |
|---------|---------|---------|
| **数据出境违规** | 用户数据传输至境外服务器 | 《数据出境安全评估办法》 |
| **算法未备案** | 海外 AI 服务未在中国备案 | 《生成式人工智能服务管理暂行办法》 |
| **内容安全** | 无法控制海外模型生成的内容 | 《网络信息内容生态治理规定》 |
| **用户隐私** | 境外服务器不受中国法律管辖 | 《个人信息保护法》 |
| **服务不稳定** | 海外 API 可能被阻断 | 网络安全审查 |

⚠️ **警告**：2023 年以来，多家使用海外 AI API 的国内 App 被要求下架整改。如果你的 App 面向国内用户，强烈建议使用已备案的国内大模型 API。

### 7.2 《数据出境安全评估办法》要点

2022 年 9 月 1 日施行的《数据出境安全评估办法》规定，以下情况必须向国家网信部门申报数据出境安全评估：

| 触发条件 | 具体标准 | 对开发者的意义 |
|---------|---------|-------------|
| 重要数据出境 | 关键信息基础设施运营者处理的重要数据 | App 如涉及重要数据，调用海外 API 需申报 |
| 大规模个人信息出境 | 处理 100 万人以上个人信息的数据处理者 | 用户量大的 App 需特别注意 |
| 累计出境个人信息量大 | 累计向境外提供 10 万人以上个人信息 | 长期调用海外 API 可能触发 |
| 累计出境敏感个人信息量大 | 累计向境外提供 1 万人以上敏感个人信息 | 用户输入可能包含敏感信息 |
| 其他情形 | 国家网信部门规定的其他情形 | 兜底条款，需关注政策动态 |

### 7.3 合规方案

针对数据出境问题，有以下三种合规方案：

**方案一：数据本地化（推荐）**

使用国内大模型 API 替代海外 API，数据不出境：

| 维度 | 说明 |
|------|------|
| 方案 | 将 OpenAI/Claude API 替换为通义千问/DeepSeek API |
| 优点 | 完全合规，无需申报，数据安全 |
| 缺点 | 部分场景能力可能不如海外模型 |
| 适用 | 面向国内用户的 App |
| 成本 | 低（国内 API 价格更低） |

**方案二：API 中转**

通过国内服务器中转海外 API 请求，在中间层做数据处理：

| 维度 | 说明 |
|------|------|
| 方案 | 搭建国内中转服务器，过滤敏感信息后再调用海外 API |
| 优点 | 可使用海外模型，部分降低风险 |
| 缺点 | 增加延迟和成本，仍需评估是否触发数据出境 |
| 适用 | 对模型能力有强需求的场景 |
| 成本 | 中（需额外服务器和中转逻辑） |

中转服务架构示例：

```swift
struct ProxyConfig {
    static let proxyBaseURL = "https://your-proxy-server.cn/api"
}

class ProxiedLLMClient {
    private let session = URLSession.shared

    func chat(messages: [QwenMessage]) async throws -> QwenResponse {
        let url = URL(string: "\(ProxyConfig.proxyBaseURL)/chat/completions")!
        var request = URLRequest(url: url)
        request.httpMethod = "POST"
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")

        let body = QwenRequest(
            model: "gpt-4o",
            messages: messages,
            temperature: 0.7,
            max_tokens: 4096
        )
        request.httpBody = try JSONEncoder().encode(body)

        let (data, response) = try await session.data(for: request)

        guard let httpResponse = response as? HTTPURLResponse,
              (200...299).contains(httpResponse.statusCode) else {
            let statusCode = (response as? HTTPURLResponse)?.statusCode ?? -1
            throw QwenError.httpError(statusCode: statusCode, body: String(data: data, encoding: .utf8) ?? "")
        }

        return try JSONDecoder().decode(QwenResponse.self, from: data)
    }
}
```

⚠️ **警告**：API 中转方案并不能完全消除数据出境风险。中转服务器如果最终将数据转发到境外，仍属于数据出境行为。此方案仅适用于过滤敏感信息后、数据出境量较小的场景。

**方案三：国产替代**

使用国产开源模型私有化部署，数据完全不出境：

| 维度 | 说明 |
|------|------|
| 方案 | 部署 Qwen/ChatGLM 等开源模型到自有服务器 |
| 优点 | 数据完全自主可控，无出境风险 |
| 缺点 | 需要 GPU 服务器，运维成本高 |
| 适用 | 对数据安全要求极高的场景 |
| 成本 | 高（GPU 服务器 + 运维） |

私有化部署快速参考：

```bash
# 使用 vLLM 部署 Qwen2.5-7B-Instruct
pip install vllm

python -m vllm.entrypoints.openai.api_server \
    --model Qwen/Qwen2.5-7B-Instruct \
    --served-model-name qwen-local \
    --host 0.0.0.0 \
    --port 8000

# 测试
curl http://localhost:8000/v1/chat/completions \
    -H "Content-Type: application/json" \
    -d '{
        "model": "qwen-local",
        "messages": [{"role": "user", "content": "你好"}]
    }'
```

### 7.4 风险评估表格

根据你的具体情况，评估数据出境风险等级：

| 场景 | 用户规模 | 数据类型 | 使用方式 | 风险等级 | 建议方案 |
|------|---------|---------|---------|---------|---------|
| 个人学习项目 | < 100 人 | 非敏感 | 直接调用海外 API | 🟢 低 | 无需特别处理 |
| 小型工具 App | < 1 万人 | 非敏感 | 直接调用海外 API | 🟡 中 | 考虑切换国内 API |
| 中型 App | 1-10 万人 | 可能含敏感 | 直接调用海外 API | 🟠 较高 | 必须切换国内 API |
| 大型 App | > 10 万人 | 含个人信息 | 直接调用海外 API | 🔴 高 | 必须使用国产替代 |
| 企业级 App | 不限 | 含重要数据 | 任何方式调用海外 | 🔴 极高 | 私有化部署 |
| 任何规模 | 不限 | 非敏感 | 使用国内已备案 API | 🟢 低 | 正常使用 |
| 任何规模 | 不限 | 非敏感 | 私有化部署国产模型 | 🟢 极低 | 最佳方案 |

💡 **提示**：对于大多数独立开发者来说，直接使用国内已备案的 API（如通义千问、DeepSeek）是最简单、最合规的方案。只有在模型能力确实无法满足需求时，才考虑其他方案。

---

## 8. 国内 AI 开发者资源

### 8.1 国内 AI 开发者社区

| 社区 | 地址 | 特点 |
|------|------|------|
| 魔搭社区（ModelScope） | modelscope.cn | 阿里云旗下，国内最大的 AI 模型社区 |
| 飞桨 AI Studio | aistudio.baidu.com | 百度旗下，提供免费 GPU 算力 |
| 智谱 AI 开放平台 | open.bigmodel.cn | 智谱官方平台，GLM 模型体验 |
| Hugging Face 镜像 | hf-mirror.com | HF 国内镜像，加速模型下载 |
| AI 论文解读社区 | papers.cool | 国内 AI 论文解读平台 |
| 机器之心 | jiqizhixin.com | AI 行业资讯与深度分析 |
| 量子位 | qbitai.com | AI 行业新闻与趋势 |
| CSDN AI 板块 | ai.csdn.net | 技术博客与教程 |

### 8.2 国内 AI 开源模型

以下是最值得关注的国内开源模型，按用途分类：

**通用对话模型**

| 模型 | 出品方 | 参数量 | 许可证 | 特点 |
|------|-------|-------|-------|------|
| Qwen2.5-72B-Instruct | 阿里 | 720 亿 | Apache 2.0 | 综合能力最强，编程优秀 |
| Qwen2.5-7B-Instruct | 阿里 | 70 亿 | Apache 2.0 | 轻量级，适合私有化部署 |
| ChatGLM4-9B | 智谱 | 90 亿 | Apache 2.0 | 中英双语，推理效率高 |
| Baichuan2-13B | 百川 | 130 亿 | Apache 2.0 | 中文优化，社区活跃 |
| Yi-1.5-34B | 零一万物 | 340 亿 | Apache 2.0 | 李开复团队，综合能力强 |

**代码模型**

| 模型 | 出品方 | 参数量 | 特点 |
|------|-------|-------|------|
| DeepSeek-Coder-V2 | 深度求索 | 2360 亿（MoE） | 编程能力最强，支持 338 种语言 |
| Qwen2.5-Coder-7B | 阿里 | 70 亿 | 轻量代码模型，支持 92 种语言 |
| CodeGeeX4-9B | 智谱 | 90 亿 | 代码补全与生成 |

**推理模型**

| 模型 | 出品方 | 参数量 | 特点 |
|------|-------|-------|------|
| DeepSeek-R1-Distill-Qwen-7B | 深度求索 | 70 亿 | 推理能力蒸馏版，可本地部署 |
| DeepSeek-R1-Distill-Qwen-32B | 深度求索 | 320 亿 | 推理能力更强，需较大 GPU |
| QwQ-32B | 阿里 | 320 亿 | 通义推理模型，开源可用 |

**多模态模型**

| 模型 | 出品方 | 能力 | 特点 |
|------|-------|------|------|
| Qwen2-VL-7B | 阿里 | 图像理解 | 图文理解能力强 |
| InternVL2-8B | 上海 AI Lab | 图像理解 | 学术开源，多模态领先 |
| CogVLM2-19B | 智谱 | 图像理解 | 视觉问答能力强 |

💡 **提示**：对于 iOS 开发者，如果需要私有化部署，推荐 Qwen2.5-7B-Instruct（轻量通用）或 DeepSeek-R1-Distill-Qwen-7B（轻量推理），单张 RTX 4090 即可运行。

### 8.3 国内 AI 开发工具链

| 工具 | 用途 | 说明 |
|------|------|------|
| vLLM | 模型推理加速 | 高吞吐推理引擎，支持 Qwen/GLM 等 |
| LMDeploy | 模型部署 | 商汤出品，支持量化推理 |
| MindIE | 推理引擎 | 华为昇腾生态推理引擎 |
| XTuner | 模型微调 | 魔搭社区出品，支持 LoRA 微调 |
| SWIFT | 模型训练 | 魔搭社区出品，支持全参数微调 |
| FunASR | 语音识别 | 阿里达摩院开源语音识别 |
| CosyVoice | 语音合成 | 阿里开源语音合成 |
| FunClip | 视频剪辑 | AI 视频剪辑工具 |

### 8.4 学习资源推荐

**在线课程**

| 资源 | 平台 | 说明 |
|------|------|------|
| 吴恩达 AI 课程 | Coursera | 入门首选，中文字幕 |
| 李宏毅机器学习 | YouTube/B站 | 中文讲解，深入浅出 |
| 魔搭社区教程 | ModelScope | 实战导向，免费 GPU |
| 飞桨 AI Studio 课程 | Baidu | 体系完整，中文友好 |
| Hugging Face 课程 | Hugging Face | NLP 经典教程 |

**技术博客与公众号**

| 资源 | 类型 | 说明 |
|------|------|------|
| 机器之心 | 公众号/网站 | AI 行业动态，论文解读 |
| 量子位 | 公众号/网站 | AI 新闻与趋势 |
| PaperWeekly | 公众号 | AI 论文精选与解读 |
| 魔搭社区 | 公众号 | 开源模型与工具更新 |
| 阿里技术 | 公众号 | 阿里 AI 技术实践 |
| 美团技术团队 | 博客 | 工程实践经验 |

**开源项目与示例**

| 项目 | 地址 | 说明 |
|------|------|------|
| Qwen 官方示例 | github.com/QwenLM | 通义千问使用示例 |
| ChatGLM 官方示例 | github.com/THUDM | ChatGLM 使用示例 |
| LobeChat | github.com/lobehub/lobe-chat | 开源 AI 聊天界面 |
| Open WebUI | github.com/open-webui | 开源 LLM Web 界面 |
| Dify | github.com/langgenius/dify | 开源 LLM 应用开发平台 |
| FastGPT | github.com/labring/FastGPT | 开源 AI 知识库平台 |

**API 速查表**

| 模型 | API 文档 | SDK |
|------|---------|-----|
| 通义千问 | help.aliyun.com/zh/dashscope | dashscope (Python/Java) |
| DeepSeek | platform.deepseek.com/api-docs | openai (Python，兼容模式) |
| 智谱 GLM | open.bigmodel.cn/dev/api | zhipuai (Python) |
| Kimi | platform.moonshot.cn/docs | openai (Python，兼容模式) |
| 文心一言 | cloud.baidu.com/doc/WENXINWORKSHOP | qianfan (Python) |
| 讯飞星火 | xinghuo.xfyun.cn/sparkapi | websocket 接入 |

💡 **提示**：通义千问、DeepSeek、Kimi 都兼容 OpenAI API 格式，你可以直接使用 OpenAI 的 Python SDK 或 Swift SDK，只需修改 `base_url` 和 `api_key`，学习成本极低。

---

## 小结

本章我们全面了解了国内大模型与 AI 生态：

1. **国内大模型已从"百模大战"走向"头部集中"**，阿里通义千问、百度文心一言、DeepSeek、智谱 GLM、Kimi 构成第一梯队
2. **API 选型没有万能答案**——编程场景推荐 DeepSeek，中文场景推荐文心一言/讯飞星火，长文档推荐 Kimi，开源/私有化推荐通义千问
3. **通义千问和 DeepSeek 都兼容 OpenAI 格式**，迁移成本极低，Swift 接入只需修改 base URL
4. **DeepSeek-R1 推理模型**是国产 AI 的亮点，推理能力对标 OpenAI o1，且价格极低
5. **国内 AI 编程工具正在快速追赶**，通义灵码在中文场景表现突出，但 Swift 支持仍需提升
6. **AI 合规是必须面对的现实**——《暂行办法》要求算法备案、内容标识、安全评估，使用国内已备案 API 是最简单的合规路径
7. **数据出境风险不容忽视**——面向国内用户的 App 应优先使用国产 API，避免直接调用海外 API
8. **国产开源模型生态蓬勃发展**——Qwen 系列开源模型在多项基准上已超越 Llama，私有化部署门槛持续降低

> 📖 **下一步**：了解了国内大模型生态后，让我们进入实战环节——用 AI 驱动的方式，从零开始完成一个完整的 iOS 项目。👉 [AI驱动端到端项目实战](./AI驱动端到端项目实战.md)

← [RAG与知识库问答](./RAG与知识库问答.md) | [AI驱动端到端项目实战](./AI驱动端到端项目实战.md) →
