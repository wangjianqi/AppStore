# AI 驱动端到端项目实战

> 🎯 **本章目标**：
> - 理解从零散 AI 辅助到系统化 AI 驱动开发的跨越意义
> - 掌握 AI 驱动开发的核心循环：Spec → 代码 → 测试 → 审查 → 迭代
> - 通过"AI 记账助手"实战项目，体验完整的 5 阶段 AI 驱动开发流程
> - 学会在每个阶段使用精准的 Prompt 模板让 AI 高效产出
> - 掌握 AI 驱动开发的效率分析方法与最佳实践

---

## 1. 为什么要做端到端 AI 驱动实战

### 从零散 AI 辅助到系统化 AI 驱动的跨越

在前面的章节中，我们已经学习了各种 AI 工具的使用方法：Claude Code 写代码、Cursor 辅助编程、AI 生成测试、AI 代码审查……但很多开发者的现状是——**零散地使用 AI，效果时好时坏**。

```
❌ 零散 AI 辅助的模式：
  遇到问题 → 随手问 AI → 得到答案 → 粘贴代码 → 下次遇到问题再问
  问题：没有规划、没有上下文、没有质量保障、没有积累

✅ 系统化 AI 驱动的模式：
  写 Spec → AI 按规范生成 → AI 测试 → AI 审查 → 迭代优化
  优势：有规划、有上下文、有质量保障、有知识积累
```

零散 AI 辅助和系统化 AI 驱动的本质区别：

| 对比维度 | 零散 AI 辅助 | 系统化 AI 驱动 |
|---------|------------|-------------|
| **使用方式** | 遇到问题才用 AI | 每个环节都有 AI 参与 |
| **上下文** | 每次对话都是全新的 | AI 始终了解项目全貌 |
| **质量保障** | 靠人工检查 | AI 生成 + AI 审查 + 人工把关 |
| **效率** | 碎片化提升 10-20% | 系统化提升 50-70% |
| **可复现性** | 同样的问题可能得到不同答案 | 规范化的流程确保输出一致 |
| **知识积累** | 每次从零开始 | Spec 和 Prompt 模板可复用 |

💡 **提示**：零散 AI 辅助就像"有事儿找朋友帮忙"，系统化 AI 驱动就像"有一个了解你项目的全职搭档"。后者效率远高于前者。

### AI 驱动开发的核心循环

AI 驱动开发不是让 AI 一次性写完所有代码，而是遵循一个**核心循环**：

```
┌─────────────────────────────────────────────┐
│                                             │
│   ┌──────┐    ┌──────┐    ┌──────┐        │
│   │ Spec │───→│ 代码 │───→│ 测试 │        │
│   └──────┘    └──────┘    └──────┘        │
│       ↑                       │            │
│       │                       ↓            │
│       │                  ┌──────┐          │
│       └──────────────────│ 审查 │←── 迭代  │
│                          └──────┘          │
│                                             │
└─────────────────────────────────────────────┘
```

每个阶段的含义：

| 阶段 | AI 的角色 | 人的角色 | 产出物 |
|------|----------|---------|-------|
| **Spec** | AI 辅助生成 PRD、技术方案、任务拆解 | 审核和调整 Spec | PRD、技术方案、任务列表 |
| **代码** | AI 按规范生成代码 | 审查代码逻辑 | 可运行的代码 |
| **测试** | AI 生成测试用例 | 验证测试覆盖率 | 测试套件 |
| **审查** | AI 审查代码质量和安全性 | 做最终决策 | 审查报告、修复方案 |
| **迭代** | AI 根据反馈修改代码 | 确认修改方向 | 改进后的代码 |

⚠️ **警告**：核心循环中，**人始终是决策者**。AI 是执行者，你是指挥者。不要把决策权交给 AI，否则项目方向会失控。

### 本章实战项目介绍：AI 记账助手

为了让你真正掌握 AI 驱动开发，我们将从零开始，用 AI 驱动的方式开发一个完整的 iOS App——**AI 记账助手**。

**项目简介**：

| 项目属性 | 说明 |
|---------|------|
| **App 名称** | AI 记账助手 |
| **核心功能** | 智能记账 + AI 对话 + 统计分析 |
| **技术栈** | SwiftUI + SwiftData + LLM API |
| **目标用户** | 想要轻松记账的年轻人 |
| **亮点功能** | 自然语言输入自动识别金额和分类 |

**功能清单**：

```
P0（核心功能）：
├── 记账 CRUD（增删改查）
├── 自然语言智能记账（"午饭花了 35" → 金额:35, 分类:餐饮）
├── AI 对话（消费建议、预算提醒）
└── 分类统计（饼图 + 趋势图）

P1（增强功能）：
├── 月度预算设置与提醒
├── 数据导出（CSV）
└── 深色模式

P2（未来扩展）：
├── iCloud 同步
├── Widget 小组件
└── Siri 快捷指令记账
```

💡 **提示**：选择这个项目有三个原因——(1) 功能完整，覆盖 CRUD、网络、UI、数据层；(2) 包含 AI 功能，可以实践第 23、24 章的知识；(3) 复杂度适中，适合在教程中完整演示。

---

## 2. Phase 1：AI 辅助写 Spec

Spec 是 AI 驱动开发的起点。没有 Spec，AI 就像没有图纸的建筑工人——可能很努力，但建出来的不是你想要的。

### 用 Claude Code 生成 PRD

PRD（Product Requirements Document，产品需求文档）定义了"做什么"。用 AI 生成 PRD，可以快速从模糊的想法变成清晰的需求。

#### Prompt 模板：生成完整的 PRD 文档

```
你是一位资深产品经理，擅长撰写清晰、可执行的 PRD。

【项目背景】
我要开发一个 iOS 记账 App，名称为"AI 记账助手"。
目标用户：18-35 岁的年轻人，希望轻松记账但嫌传统记账 App 太繁琐。
核心差异化：支持自然语言输入记账，AI 自动识别金额和分类。

【任务】
请生成一份完整的 PRD 文档，包含以下部分：
1. 产品概述（一句话描述、目标用户、核心价值）
2. 功能需求列表（按 P0/P1/P2 分级，每个功能包含描述和验收标准）
3. 用户故事（至少 5 个核心用户故事）
4. 非功能需求（性能、安全、可用性）
5. 数据模型（核心实体和关系）
6. 信息架构（页面层级和导航关系）

【约束】
- 技术栈：SwiftUI + SwiftData + LLM API
- 最低支持 iOS 17
- App 需要离线可用，AI 功能需要网络
- 面向中国大陆市场，需要考虑合规性

【输出格式】
使用 Markdown 格式，表格用于功能列表，用户故事使用"As a... I want... So that..."格式。
```

AI 生成的 PRD 示例（节选）：

```markdown
## 功能需求列表

| 优先级 | 功能 | 描述 | 验收标准 |
|-------|------|------|---------|
| P0 | 快速记账 | 支持自然语言输入，AI 自动提取金额和分类 | 输入"午饭35"自动识别为金额35元、分类餐饮 |
| P0 | 记账列表 | 按时间倒序展示所有记录 | 支持按日/周/月筛选，下拉刷新 |
| P0 | 分类统计 | 饼图展示各分类占比 | 支持切换时间范围，点击分类查看明细 |
| P0 | AI 对话 | 与 AI 助手对话获取消费建议 | 支持多轮对话，基于用户数据给出建议 |
| P1 | 预算管理 | 设置月度预算，超支提醒 | 预算使用进度条，达到 80% 和 100% 时提醒 |
| P1 | 数据导出 | 导出 CSV 格式账单 | 支持选择时间范围导出 |
| P2 | iCloud 同步 | 多设备数据同步 | 自动同步，冲突处理 |
```

### 用 AI 生成技术选型方案

技术选型决定了"怎么做"。让 AI 帮你分析各方案的优劣，比自己在网上搜索更高效。

#### Prompt 模板：技术选型决策

```
你是一位 iOS 架构师，需要为以下项目做技术选型。

【项目信息】
- 项目：AI 记账助手（iOS App）
- 功能：记账 CRUD + AI 对话 + 统计图表
- 团队：1 人独立开发，熟悉 SwiftUI，不熟悉 UIKit
- 约束：最低支持 iOS 17，需要离线可用

【任务】
请为以下技术决策提供选型分析和推荐：

1. 数据持久化方案：SwiftData vs Core Data vs Realm
2. 网络层方案：URLSession vs Alamofire vs Moya
3. 图表库：Swift Charts vs DGCharts vs Charts框架
4. 架构模式：MVVM vs TCA vs MVC
5. LLM API 选型：OpenAI vs 通义千问 vs DeepSeek

【输出格式】
每个决策项使用表格对比，包含维度：学习成本、功能满足度、社区生态、维护状态。
最后给出推荐方案和理由。
```

AI 生成的技术选型示例（节选）：

| 决策项 | 推荐方案 | 理由 |
|-------|---------|------|
| 数据持久化 | SwiftData | iOS 17 原生，SwiftUI 集成好，学习成本低 |
| 网络层 | URLSession + async/await | 原生方案，无第三方依赖，够用 |
| 图表 | Swift Charts | iOS 16+ 原生，与 SwiftUI 无缝集成 |
| 架构 | MVVM | SwiftUI 天然适配，1 人开发足够 |
| LLM API | DeepSeek | 代码能力强，价格低，国内合规 |

### 用 AI 拆解任务

任务拆解把大目标变成可执行的小步骤。AI 擅长结构化拆解，但需要你把控优先级。

#### Prompt 模板：任务拆解与优先级排序

```
你是一位敏捷开发教练，擅长任务拆解和优先级排序。

【项目信息】
- 项目：AI 记账助手
- PRD：（粘贴上面生成的 PRD）
- 技术方案：SwiftUI + SwiftData + MVVM + DeepSeek API

【任务】
请将项目拆解为可执行的开发任务，要求：
1. 每个任务粒度控制在 1-4 小时内
2. 标注任务之间的依赖关系
3. 按开发顺序排列（先基础设施，后业务功能）
4. 标注每个任务的预估时间和优先级

【输出格式】
使用表格，列：序号、任务名称、描述、依赖、预估时间、优先级
按阶段分组：基础设施 → 数据层 → 网络层 → UI 层 → AI 功能 → 测试 → 上架
```

AI 生成的任务拆解示例（节选）：

| 序号 | 任务 | 描述 | 依赖 | 预估 | 优先级 |
|-----|------|------|-----|------|-------|
| 1 | 创建 Xcode 项目 | SwiftUI + SwiftData 模板 | 无 | 0.5h | P0 |
| 2 | 配置 CLAUDE.md | 项目规范文件 | 1 | 0.5h | P0 |
| 3 | 定义数据模型 | Expense / Category 模型 | 1 | 1h | P0 |
| 4 | 实现 SwiftData CRUD | 增删改查操作 | 3 | 2h | P0 |
| 5 | 搭建网络层 | LLMService 封装 | 1 | 2h | P0 |
| 6 | 记账列表 UI | 列表视图 + 筛选 | 4 | 3h | P0 |
| 7 | 智能记账 UI | 自然语言输入界面 | 5,6 | 2h | P0 |
| 8 | AI 对话界面 | 聊天 UI + 流式输出 | 5 | 4h | P0 |
| 9 | 统计图表 | 饼图 + 趋势图 | 4 | 3h | P0 |
| 10 | 单元测试 | 核心逻辑测试 | 4,5 | 2h | P1 |

### 用 AI 定义验收标准

验收标准是"怎么算做完了"。没有验收标准，你无法判断 AI 生成的代码是否合格。

```
你是一位 QA 工程师，需要为以下功能定义验收标准。

【功能】智能记账——用户输入自然语言，AI 自动提取金额和分类

【任务】
请定义详细的验收标准，包含：
1. 正常路径（Happy Path）
2. 边界情况（金额为 0、负数、极大值等）
3. 异常路径（网络错误、AI 返回异常等）
4. 性能要求（响应时间、内存占用）
5. UI/UX 要求（加载状态、错误提示、空状态）

【输出格式】
使用 Given-When-Then 格式描述每个验收条件。
```

验收标签示例：

```markdown
### AC-001: 正常记账
Given 用户在记账输入框中
When 输入"午饭花了35"并提交
Then 系统自动创建一条记录：金额 35.00，分类 餐饮，备注 午饭

### AC-002: 无金额输入
Given 用户在记账输入框中
When 输入"买了一杯咖啡"（无金额）并提交
Then 系统提示"未识别到金额，请补充金额信息"

### AC-003: 网络错误
Given 用户设备无网络连接
When 输入"打车回家28"并提交
Then 系统使用本地规则进行基础分类，显示"离线模式：分类可能不准确"
```

### Phase 1 产出物清单

| 产出物 | 格式 | 用途 |
|-------|------|------|
| PRD 文档 | Markdown | 定义产品功能需求 |
| 技术选型方案 | Markdown | 确定技术栈和架构 |
| 任务拆解表 | Markdown / 表格 | 指导开发顺序 |
| 验收标准 | Markdown | 判断功能是否完成 |
| 项目规范（CLAUDE.md） | Markdown | 让 AI 理解项目上下文 |

💡 **提示**：Phase 1 看起来"只是写文档"，但它是整个项目的基础。投入 2-3 小时写好 Spec，可以节省 10+ 小时的返工时间。这就是"磨刀不误砍柴工"。

---

## 3. Phase 2：AI 辅助搭建项目

有了 Spec 之后，下一步是搭建项目基础架构。这个阶段的目标是：**让项目"能跑起来"，为后续功能开发打好地基**。

### 用 AI 生成项目架构

项目架构决定了代码的组织方式。让 AI 帮你设计架构，可以避免"写到一半发现结构不合理"的尴尬。

#### Prompt 模板：项目架构设计

```
你是一位 iOS 架构师，需要设计一个 SwiftUI + SwiftData + MVVM 项目的目录结构。

【项目信息】
- 名称：AI 记账助手
- 技术栈：SwiftUI + SwiftData + MVVM + URLSession
- 功能模块：记账 CRUD、AI 对话、统计图表、设置

【任务】
请设计项目的目录结构，要求：
1. 按功能模块分组（不是按文件类型分组）
2. 每个模块包含 Views / ViewModels / Models 子目录
3. 公共组件放在 Shared 目录
4. 网络层放在 Services 目录
5. 列出每个目录下需要创建的文件名

【输出格式】
使用树形结构展示目录，每个文件标注用途。
```

AI 生成的项目架构：

```
AIExpenseTracker/
├── App/
│   ├── AIExpenseTrackerApp.swift
│   └── ContentView.swift
├── Features/
│   ├── Expense/
│   │   ├── Views/
│   │   │   ├── ExpenseListView.swift
│   │   │   ├── ExpenseAddView.swift
│   │   │   └── ExpenseDetailView.swift
│   │   ├── ViewModels/
│   │   │   └── ExpenseViewModel.swift
│   │   └── Models/
│   │       └── Expense.swift
│   ├── Chat/
│   │   ├── Views/
│   │   │   ├── ChatView.swift
│   │   │   └── MessageBubbleView.swift
│   │   ├── ViewModels/
│   │   │   └── ChatViewModel.swift
│   │   └── Models/
│   │       └── Message.swift
│   ├── Statistics/
│   │   ├── Views/
│   │   │   └── StatisticsView.swift
│   │   └── ViewModels/
│   │       └── StatisticsViewModel.swift
│   └── Settings/
│       ├── Views/
│       │   └── SettingsView.swift
│       └── ViewModels/
│           └── SettingsViewModel.swift
├── Services/
│   ├── LLMService.swift
│   └── SmartParseService.swift
├── Shared/
│   ├── Components/
│   │   ├── LoadingView.swift
│   │   └── EmptyStateView.swift
│   ├── Extensions/
│   │   └── Color+Extensions.swift
│   └── Constants/
│       └── AppConstants.swift
└── Resources/
    ├── Assets.xcassets
    └── Info.plist
```

### 用 AI 搭建网络层

结合第 23 章的 LLMService，让 AI 生成网络层代码。这一层是 AI 对话功能的基础。

```
你是一位 iOS 开发工程师，需要实现 LLM API 的网络层。

【技术方案】
- 使用 URLSession + async/await
- 支持 DeepSeek API（兼容 OpenAI 格式）
- 支持流式输出（SSE）
- API Key 存储在 Keychain 中

【任务】
请实现以下文件：
1. LLMService.swift - 核心网络请求服务
2. LLMRequest.swift - 请求模型
3. LLMResponse.swift - 响应模型
4. APIKeyManager.swift - API Key 安全存储

【约束】
- Swift 5.9+，iOS 17+
- 使用 Swift Concurrency（async/await）
- 错误处理使用自定义 LLMError 枚举
- 流式输出使用 AsyncThrowingStream
```

AI 生成的网络层核心代码：

```swift
import Foundation

enum LLMError: LocalizedError {
    case invalidAPIKey
    case networkError(Error)
    case invalidResponse
    case streamInterrupted
    case rateLimited

    var errorDescription: String? {
        switch self {
        case .invalidAPIKey: "API Key 无效，请在设置中重新配置"
        case .networkError(let error): "网络错误：\(error.localizedDescription)"
        case .invalidResponse: "服务器返回了无效的响应"
        case .streamInterrupted: "AI 响应中断，请重试"
        case .rateLimited: "请求过于频繁，请稍后再试"
        }
    }
}

struct LLMMessage: Codable {
    let role: String
    let content: String
}

struct LLMRequest: Codable {
    let model: String
    let messages: [LLMMessage]
    let stream: Bool
    let temperature: Double
    let maxTokens: Int

    init(messages: [LLMMessage], stream: Bool = false, temperature: Double = 0.7, maxTokens: Int = 2048) {
        self.model = "deepseek-chat"
        self.messages = messages
        self.stream = stream
        self.temperature = temperature
        self.maxTokens = maxTokens
    }
}

struct LLMResponse: Codable {
    let id: String
    let choices: [Choice]

    struct Choice: Codable {
        let message: LLMMessage
        let finishReason: String?

        enum CodingKeys: String, CodingKey {
            case message
            case finishReason = "finish_reason"
        }
    }
}

actor LLMService {
    private let baseURL = "https://api.deepseek.com/v1/chat/completions"
    private let session: URLSession

    init() {
        let config = URLSessionConfiguration.default
        config.timeoutIntervalForRequest = 30
        config.timeoutIntervalForResource = 60
        self.session = URLSession(configuration: config)
    }

    func chat(messages: [LLMMessage]) async throws -> String {
        guard let apiKey = APIKeyManager.shared.getKey() else {
            throw LLMError.invalidAPIKey
        }

        let request = LLMRequest(messages: messages)
        var urlRequest = try createURLRequest(from: request, apiKey: apiKey)

        do {
            let (data, response) = try await session.data(for: urlRequest)
            try checkHTTPStatus(response: response)
            let llmResponse = try JSONDecoder().decode(LLMResponse.self, from: data)
            guard let content = llmResponse.choices.first?.message.content else {
                throw LLMError.invalidResponse
            }
            return content
        } catch let error as LLMError {
            throw error
        } catch {
            throw LLMError.networkError(error)
        }
    }

    func chatStream(messages: [LLMMessage]) -> AsyncThrowingStream<String, Error> {
        AsyncThrowingStream { continuation in
            Task {
                guard let apiKey = APIKeyManager.shared.getKey() else {
                    continuation.finish(throwing: LLMError.invalidAPIKey)
                    return
                }

                let request = LLMRequest(messages: messages, stream: true)
                var urlRequest = try createURLRequest(from: request, apiKey: apiKey)

                do {
                    let (bytes, response) = try await session.bytes(for: urlRequest)
                    try checkHTTPStatus(response: response)

                    for try await line in bytes.lines {
                        guard line.hasPrefix("data: ") else { continue }
                        let jsonString = String(line.dropFirst(6))
                        if jsonString == "[DONE]" { break }

                        if let data = jsonString.data(using: .utf8),
                           let delta = try? JSONDecoder().decode(StreamDelta.self, from: data),
                           let content = delta.choices.first?.delta.content {
                            continuation.yield(content)
                        }
                    }
                    continuation.finish()
                } catch {
                    continuation.finish(throwing: error)
                }
            }
        }
    }

    private func createURLRequest(from request: LLMRequest, apiKey: String) throws -> URLRequest {
        guard let url = URL(string: baseURL) else {
            throw LLMError.invalidResponse
        }
        var urlRequest = URLRequest(url: url)
        urlRequest.httpMethod = "POST"
        urlRequest.setValue("Bearer \(apiKey)", forHTTPHeaderField: "Authorization")
        urlRequest.setValue("application/json", forHTTPHeaderField: "Content-Type")
        urlRequest.httpBody = try JSONEncoder().encode(request)
        return urlRequest
    }

    private func checkHTTPStatus(response: URLResponse) throws {
        guard let httpResponse = response as? HTTPURLResponse else {
            throw LLMError.invalidResponse
        }
        if httpResponse.statusCode == 429 {
            throw LLMError.rateLimited
        }
        guard (200...299).contains(httpResponse.statusCode) else {
            throw LLMError.invalidResponse
        }
    }
}

struct StreamDelta: Codable {
    let choices: [DeltaChoice]

    struct DeltaChoice: Codable {
        let delta: DeltaContent

        struct DeltaContent: Codable {
            let content: String?
        }
    }
}
```

### 用 AI 搭建数据层

SwiftData 是 iOS 17+ 的现代数据持久化方案，与 SwiftUI 天然集成。

```swift
import SwiftData
import Foundation

@Model
final class Expense {
    @Attribute(.unique) var id: UUID
    var amount: Double
    var category: String
    var note: String
    var date: Date
    var isSmartParsed: Bool
    var createdAt: Date

    init(
        amount: Double,
        category: String,
        note: String = "",
        date: Date = .now,
        isSmartParsed: Bool = false
    ) {
        self.id = UUID()
        self.amount = amount
        self.category = category
        self.note = note
        self.date = date
        self.isSmartParsed = isSmartParsed
        self.createdAt = .now
    }
}

@Model
final class CategoryDefinition {
    @Attribute(.unique) var id: UUID
    var name: String
    var icon: String
    var colorHex: String
    var isDefault: Bool
    var sortOrder: Int

    init(name: String, icon: String, colorHex: String, isDefault: Bool = false, sortOrder: Int = 0) {
        self.id = UUID()
        self.name = name
        self.icon = icon
        self.colorHex = colorHex
        self.isDefault = isDefault
        self.sortOrder = sortOrder
    }
}

@Model
final class ChatMessage {
    @Attribute(.unique) var id: UUID
    var role: String
    var content: String
    var timestamp: Date

    init(role: String, content: String) {
        self.id = UUID()
        self.role = role
        self.content = content
        self.timestamp = .now
    }
}
```

### 用 AI 配置项目规范文件

项目规范文件让 AI 始终了解你的项目上下文，是 AI 驱动开发的关键基础设施。

**CLAUDE.md 配置**：

```markdown
# AI 记账助手 - 项目规范

## 技术栈
- Swift 5.9+, iOS 17+
- SwiftUI + SwiftData + MVVM
- URLSession + async/await（网络层）
- DeepSeek API（LLM 服务）
- Swift Charts（图表）

## 架构
- 按功能模块分组：Expense / Chat / Statistics / Settings
- 每个模块：Views / ViewModels / Models
- 公共组件在 Shared 目录
- 网络服务在 Services 目录

## 编码规范
- 使用 Swift Concurrency（async/await），不使用 Combine
- ViewModel 使用 @Observable 宏（iOS 17+）
- SwiftData 模型使用 @Model 宏
- 错误处理使用自定义枚举，遵循 LocalizedError 协议
- 命名：变量和函数用 camelCase，类型用 PascalCase
- 视图拆分：单个视图不超过 100 行

## 关键文件
- LLMService.swift：LLM API 调用服务
- SmartParseService.swift：自然语言解析服务
- Expense.swift：记账数据模型
- ChatMessage.swift：对话消息模型

## 注意事项
- AI 功能需要网络，离线时使用本地规则降级
- API Key 存储在 Keychain，不硬编码
- 中国大陆市场，注意合规性
```

**.cursorrules 配置**：

```markdown
This is an iOS expense tracking app called "AI 记账助手".
Tech stack: SwiftUI + SwiftData + MVVM + DeepSeek API.
Follow MVVM pattern strictly.
Use @Observable for ViewModels, @Model for SwiftData models.
All UI text must be in Chinese.
```

### Phase 2 产出物清单

| 产出物 | 格式 | 用途 |
|-------|------|------|
| Xcode 项目 | .xcodeproj | 项目工程文件 |
| 目录结构 | 文件夹 | 代码组织方式 |
| 网络层代码 | Swift | LLM API 调用能力 |
| 数据模型代码 | Swift | SwiftData 模型定义 |
| 项目规范文件 | CLAUDE.md / .cursorrules | AI 上下文配置 |

💡 **提示**：Phase 2 的关键是"基础设施先行"。网络层和数据层搭好了，后续的功能开发就像在高速公路上开车——顺畅高效。如果跳过这步直接写 UI，后续一定会反复修改。

---

## 4. Phase 3：AI 辅助核心功能开发

这是最核心的阶段——用 AI 开发实际的功能代码。我们将按照任务拆解的顺序，逐个功能用 AI 驱动开发。

### 用 AI 开发记账 UI

#### Prompt 模板：SwiftUI 视图生成

```
你是一位 SwiftUI 高级开发工程师。

【项目背景】
项目：AI 记账助手
技术栈：SwiftUI + SwiftData + MVVM + @Observable
架构：按功能模块分组，ViewModel 使用 @Observable

【任务】
实现记账列表视图 ExpenseListView，要求：
1. 使用 @Query 从 SwiftData 获取记账记录
2. 按日期倒序排列
3. 每行显示：分类图标 + 备注 + 金额 + 日期
4. 支持左滑删除
5. 下拉刷新
6. 空状态提示"还没有记账记录，点击 + 开始记账"
7. 导航栏标题"账单"，右侧添加按钮

【约束】
- 使用 NavigationStack
- 金额显示两位小数，负数用红色
- 日期显示"今天"/"昨天"/具体日期
- 单个视图不超过 80 行，超出则拆分子视图
- 所有 UI 文本使用中文

【参考】
项目已有 Expense 模型（SwiftData @Model），包含 amount/category/note/date 字段。
```

AI 生成的记账列表视图：

```swift
import SwiftUI
import SwiftData

struct ExpenseListView: View {
    @Environment(\.modelContext) private var modelContext
    @Query(sort: \Expense.date, order: .reverse) private var expenses: [Expense]
    @State private var showAddSheet = false

    var body: some View {
        NavigationStack {
            Group {
                if expenses.isEmpty {
                    emptyState
                } else {
                    expenseList
                }
            }
            .navigationTitle("账单")
            .toolbar {
                ToolbarItem(placement: .primaryAction) {
                    Button {
                        showAddSheet = true
                    } label: {
                        Image(systemName: "plus")
                    }
                }
            }
            .sheet(isPresented: $showAddSheet) {
                ExpenseAddView()
            }
        }
    }

    private var emptyState: some View {
        ContentUnavailableView(
            "暂无记录",
            systemImage: "doc.text.magnifyingglass",
            description: Text("还没有记账记录，点击 + 开始记账")
        )
    }

    private var expenseList: some View {
        List {
            ForEach(expenses) { expense in
                ExpenseRowView(expense: expense)
            }
            .onDelete(perform: deleteExpenses)
        }
        .refreshable {
            try? await Task.sleep(for: .milliseconds(300))
        }
    }

    private func deleteExpenses(at offsets: IndexSet) {
        for index in offsets {
            modelContext.delete(expenses[index])
        }
    }
}

struct ExpenseRowView: View {
    let expense: Expense

    var body: some View {
        HStack(spacing: 12) {
            categoryIcon
            VStack(alignment: .leading, spacing: 4) {
                Text(expense.note.isEmpty ? expense.category : expense.note)
                    .font(.subheadline)
                Text(formatDate(expense.date))
                    .font(.caption)
                    .foregroundStyle(.secondary)
            }
            Spacer()
            Text(String(format: "¥%.2f", expense.amount))
                .font(.subheadline.bold())
                .foregroundStyle(expense.amount < 0 ? .red : .primary)
        }
        .padding(.vertical, 4)
    }

    private var categoryIcon: some View {
        Text(categoryEmoji(expense.category))
            .font(.title2)
            .frame(width: 40, height: 40)
            .background(Color(.systemGray6))
            .clipShape(Circle())
    }

    private func formatDate(_ date: Date) -> String {
        let calendar = Calendar.current
        if calendar.isDateInToday(date) { return "今天" }
        if calendar.isDateInYesterday(date) { return "昨天" }
        let formatter = DateFormatter()
        formatter.dateFormat = "MM月dd日"
        return formatter.string(from: date)
    }

    private func categoryEmoji(_ category: String) -> String {
        switch category {
        case "餐饮": "🍽️"
        case "交通": "🚗"
        case "购物": "🛍️"
        case "娱乐": "🎮"
        case "居住": "🏠"
        case "医疗": "💊"
        case "教育": "📚"
        default: "💰"
        }
    }
}
```

### 用 AI 开发业务逻辑

记账 CRUD 和分类统计是 App 的核心业务逻辑。让 AI 生成 ViewModel，把业务逻辑从视图中分离出来。

```swift
import SwiftUI
import SwiftData

@Observable
final class ExpenseViewModel {
    var expenses: [Expense] = []
    var monthlyTotal: Double = 0
    var categoryBreakdown: [CategoryStat] = []
    var isLoading = false
    var errorMessage: String?

    private var modelContext: ModelContext?

    func configure(modelContext: ModelContext) {
        self.modelContext = modelContext
        fetchExpenses()
    }

    func fetchExpenses() {
        guard let modelContext else { return }
        let descriptor = FetchDescriptor<Expense>(
            sortBy: [SortDescriptor(\.date, order: .reverse)]
        )
        expenses = (try? modelContext.fetch(descriptor)) ?? []
        calculateStatistics()
    }

    func addExpense(amount: Double, category: String, note: String, date: Date = .now, isSmartParsed: Bool = false) {
        guard let modelContext else { return }
        let expense = Expense(
            amount: amount,
            category: category,
            note: note,
            date: date,
            isSmartParsed: isSmartParsed
        )
        modelContext.insert(expense)
        try? modelContext.save()
        fetchExpenses()
    }

    func deleteExpense(_ expense: Expense) {
        guard let modelContext else { return }
        modelContext.delete(expense)
        try? modelContext.save()
        fetchExpenses()
    }

    func expensesForMonth(_ date: Date) -> [Expense] {
        let calendar = Calendar.current
        let startOfMonth = calendar.date(from: calendar.dateComponents([.year, .month], from: date))!
        let endOfMonth = calendar.date(byAdding: .month, value: 1, to: startOfMonth)!

        return expenses.filter { expense in
            expense.date >= startOfMonth && expense.date < endOfMonth
        }
    }

    private func calculateStatistics() {
        let calendar = Calendar.current
        let currentMonthExpenses = expensesForMonth(.now)

        monthlyTotal = currentMonthExpenses.reduce(0) { $0 + $1.amount }

        var categoryMap: [String: Double] = [:]
        for expense in currentMonthExpenses {
            categoryMap[expense.category, default: 0] += expense.amount
        }

        categoryBreakdown = categoryMap.map { CategoryStat(category: $0.key, amount: $0.value) }
            .sorted { $0.amount > $1.amount }
    }
}

struct CategoryStat: Identifiable {
    let id = UUID()
    let category: String
    let amount: Double
}
```

### 用 AI 集成 AI 对话功能

结合第 24 章的知识，集成 AI 对话功能。这是本项目的亮点功能。

#### Prompt 模板：AI 功能集成

```
你是一位 iOS 开发工程师，需要实现 AI 对话功能。

【项目背景】
项目：AI 记账助手
已有代码：
- LLMService.swift：封装了 DeepSeek API 调用，支持普通请求和流式输出
- ChatMessage.swift：SwiftData 模型，包含 role/content/timestamp
- 项目使用 @Observable 宏和 Swift Concurrency

【任务】
实现 ChatViewModel，要求：
1. 管理对话消息列表
2. 发送用户消息并获取 AI 回复
3. 支持流式输出，逐字显示 AI 回复
4. 将对话历史持久化到 SwiftData
5. 构建系统提示词，让 AI 了解用户的消费数据
6. 处理网络错误和超时

【约束】
- 使用 @Observable 宏
- 流式输出使用 AsyncThrowingStream
- 系统提示词需要包含用户当月消费摘要
- 错误时显示友好提示，不暴露技术细节
- 对话历史最多保留最近 20 条（控制 Token 用量）
```

AI 生成的 ChatViewModel：

```swift
import SwiftUI
import SwiftData

@Observable
final class ChatViewModel {
    var messages: [ChatMessage] = []
    var isLoading = false
    var streamingContent = ""
    var errorMessage: String?

    private let llmService = LLMService()
    private var modelContext: ModelContext?
    private var currentStreamTask: Task<Void, Never>?

    func configure(modelContext: ModelContext) {
        self.modelContext = modelContext
        loadMessages()
    }

    func sendMessage(_ text: String) {
        let userMessage = ChatMessage(role: "user", content: text)
        messages.append(userMessage)
        persistMessage(userMessage)

        isLoading = true
        streamingContent = ""
        errorMessage = nil

        currentStreamTask = Task { [weak self] in
            guard let self else { return }
            do {
                let systemPrompt = self.buildSystemPrompt()
                let llmMessages = self.buildLLMMessages(systemPrompt: systemPrompt)

                var fullContent = ""
                let stream = self.llmService.chatStream(messages: llmMessages)

                for try await chunk in stream {
                    guard !Task.isCancelled else { return }
                    fullContent += chunk
                    self.streamingContent = fullContent
                }

                let assistantMessage = ChatMessage(role: "assistant", content: fullContent)
                self.messages.append(assistantMessage)
                self.persistMessage(assistantMessage)

            } catch {
                self.errorMessage = "AI 助手暂时无法回复，请稍后再试"
            }

            self.isLoading = false
            self.streamingContent = ""
        }
    }

    func cancelStream() {
        currentStreamTask?.cancel()
        currentStreamTask = nil
        isLoading = false
        if !streamingContent.isEmpty {
            let partialMessage = ChatMessage(role: "assistant", content: streamingContent)
            messages.append(partialMessage)
            persistMessage(partialMessage)
        }
        streamingContent = ""
    }

    func clearHistory() {
        guard let modelContext else { return }
        for message in messages {
            modelContext.delete(message)
        }
        try? modelContext.save()
        messages.removeAll()
    }

    private func loadMessages() {
        guard let modelContext else { return }
        let descriptor = FetchDescriptor<ChatMessage>(
            sortBy: [SortDescriptor(\.timestamp)]
        )
        messages = (try? modelContext.fetch(descriptor)) ?? []
    }

    private func persistMessage(_ message: ChatMessage) {
        guard let modelContext else { return }
        modelContext.insert(message)
        try? modelContext.save()
    }

    private func buildSystemPrompt() -> String {
        var prompt = """
        你是一个专业的记账助手。你的职责是：
        1. 帮助用户分析消费习惯
        2. 提供合理的理财建议
        3. 回答关于消费和预算的问题
        4. 用轻松友好的语气交流

        回答要求：
        - 简洁明了，不超过 200 字
        - 给出具体可执行的建议
        - 适当使用 emoji 让回复更生动
        """

        let recentExpenses = fetchRecentExpenseSummary()
        if !recentExpenses.isEmpty {
            prompt += "\n\n用户本月消费摘要：\n\(recentExpenses)"
        }

        return prompt
    }

    private func fetchRecentExpenseSummary() -> String {
        guard let modelContext else { return "" }
        let calendar = Calendar.current
        let startOfMonth = calendar.date(from: calendar.dateComponents([.year, .month], from: .now))!

        var descriptor = FetchDescriptor<Expense>(
            predicate: #Predicate { $0.date >= startOfMonth },
            sortBy: [SortDescriptor(\.amount, order: .reverse)]
        )
        descriptor.fetchLimit = 20

        guard let expenses = try? modelContext.fetch(descriptor), !expenses.isEmpty else {
            return ""
        }

        let total = expenses.reduce(0) { $0 + $1.amount }
        var categorySummary: [String: Double] = [:]
        for expense in expenses {
            categorySummary[expense.category, default: 0] += expense.amount
        }

        var summary = "本月总支出：¥\(String(format: "%.2f", total))\n"
        for (category, amount) in categorySummary.sorted(by: { $0.value > $1.value }) {
            summary += "- \(category)：¥\(String(format: "%.2f", amount))\n"
        }

        return summary
    }

    private func buildLLMMessages(systemPrompt: String) -> [LLMMessage] {
        var result: [LLMMessage] = [
            LLMMessage(role: "system", content: systemPrompt)
        ]

        let recentMessages = Array(messages.suffix(20))
        for message in recentMessages {
            result.append(LLMMessage(role: message.role, content: message.content))
        }

        return result
    }
}
```

### 用 AI 实现智能记账

智能记账是本项目的核心亮点——用户输入自然语言，AI 自动提取金额和分类。这个功能有两种实现方式：

| 方案 | 优点 | 缺点 |
|------|------|------|
| **LLM 解析** | 准确率高，能理解复杂表达 | 需要网络，有延迟和成本 |
| **本地规则解析** | 离线可用，零延迟 | 准确率有限，复杂表达无法识别 |

最佳策略是**本地优先 + LLM 增强**：先用本地规则快速解析，有网络时再用 LLM 校验。

```swift
import Foundation

@Observable
final class SmartParseService {
    var isParsing = false

    private let llmService = LLMService()

    struct ParseResult {
        let amount: Double
        let category: String
        let note: String
        let confidence: Double
        let source: ParseSource
    }

    enum ParseSource {
        case local
        case llm
    }

    func parse(_ input: String) async -> ParseResult? {
        let localResult = localParse(input)

        if await Reachability.isConnected() {
            isParsing = true
            let llmResult = await llmParse(input)
            isParsing = false

            if let llmResult {
                return llmResult
            }
        }

        return localResult
    }

    private func localParse(_ input: String) -> ParseResult? {
        let amount = extractAmount(from: input)
        let category = classifyCategory(from: input)

        guard let amount else { return nil }

        return ParseResult(
            amount: amount,
            category: category,
            note: input,
            confidence: 0.7,
            source: .local
        )
    }

    private func extractAmount(from input: String) -> Double? {
        let patterns = [
            "[0-9]+\\.?[0-9]*元",
            "[0-9]+\\.?[0-9]*块",
            "¥[0-9]+\\.?[0-9]*",
            "[0-9]+\\.?[0-9]*"
        ]

        for pattern in patterns {
            if let regex = try? NSRegularExpression(pattern: pattern),
               let match = regex.firstMatch(in: input, range: NSRange(input.startIndex..., in: input)),
               let range = Range(match.range, in: input) {
                let matched = String(input[range])
                let cleaned = matched
                    .replacingOccurrences(of: "元", with: "")
                    .replacingOccurrences(of: "块", with: "")
                    .replacingOccurrences(of: "¥", with: "")
                return Double(cleaned)
            }
        }

        return nil
    }

    private func classifyCategory(from input: String) -> String {
        let rules: [(keywords: [String], category: String)] = [
            (["早餐", "午餐", "晚餐", "外卖", "饭", "奶茶", "咖啡", "零食", "水果"], "餐饮"),
            (["打车", "地铁", "公交", "加油", "停车", "高铁", "机票", "滴滴"], "交通"),
            (["衣服", "鞋", "包", "化妆品", "护肤", "淘宝", "京东"], "购物"),
            (["电影", "游戏", "KTV", "旅游", "门票", "会员"], "娱乐"),
            (["房租", "水电", "物业", "网费", "燃气"], "居住"),
            (["医院", "药", "体检", "挂号"], "医疗"),
            (["课程", "书", "培训", "考试"], "教育"),
        ]

        for rule in rules {
            for keyword in rule.keywords {
                if input.contains(keyword) {
                    return rule.category
                }
            }
        }

        return "其他"
    }

    private func llmParse(_ input: String) async -> ParseResult? {
        let prompt = """
        请从以下记账文本中提取信息，以 JSON 格式返回：
        {"amount": 数字, "category": "分类", "note": "备注"}

        分类只能是：餐饮、交通、购物、娱乐、居住、医疗、教育、其他

        记账文本：\(input)

        只返回 JSON，不要其他内容。
        """

        let messages = [LLMMessage(role: "user", content: prompt)]

        do {
            let response = try await llmService.chat(messages: messages)
            return parseLLMResponse(response, originalInput: input)
        } catch {
            return nil
        }
    }

    private func parseLLMResponse(_ response: String, originalInput: String) -> ParseResult? {
        guard let data = response.data(using: .utf8),
              let json = try? JSONSerialization.jsonObject(with: data) as? [String: Any],
              let amount = json["amount"] as? Double,
              let category = json["category"] as? String else {
            return nil
        }

        let note = json["note"] as? String ?? originalInput

        return ParseResult(
            amount: amount,
            category: category,
            note: note,
            confidence: 0.95,
            source: .llm
        )
    }
}
```

智能记账输入视图：

```swift
import SwiftUI

struct SmartExpenseInputView: View {
    @Environment(\.modelContext) private var modelContext
    @Environment(\.dismiss) private var dismiss

    @State private var inputText = ""
    @State private var parsedAmount = ""
    @State private var parsedCategory = ""
    @State private var isParsing = false

    private let parseService = SmartParseService()

    var body: some View {
        NavigationStack {
            VStack(spacing: 20) {
                inputSection
                parsedResultSection
                Spacer()
            }
            .padding()
            .navigationTitle("智能记账")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .cancellationAction) {
                    Button("取消") { dismiss() }
                }
                ToolbarItem(placement: .confirmationAction) {
                    Button("保存") { saveExpense() }
                        .disabled(parsedAmount.isEmpty)
                }
            }
        }
    }

    private var inputSection: some View {
        VStack(alignment: .leading, spacing: 8) {
            Text("描述你的消费")
                .font(.headline)

            TextField("例如：午饭花了35", text: $inputText)
                .textFieldStyle(.roundedBorder)
                .onSubmit { parseInput() }

            Button {
                parseInput()
            } label: {
                HStack {
                    if isParsing {
                        ProgressView()
                            .controlSize(.small)
                    }
                    Text(isParsing ? "AI 识别中..." : "智能识别")
                }
                .frame(maxWidth: .infinity)
            }
            .buttonStyle(.borderedProminent)
            .disabled(inputText.isEmpty || isParsing)
        }
    }

    private var parsedResultSection: some View {
        Group {
            if !parsedAmount.isEmpty {
                VStack(alignment: .leading, spacing: 12) {
                    Text("识别结果")
                        .font(.headline)

                    HStack {
                        Label("金额", systemImage: "yensign.circle")
                        Spacer()
                        Text("¥\(parsedAmount)")
                            .bold()
                    }

                    HStack {
                        Label("分类", systemImage: "tag.circle")
                        Spacer()
                        Text(parsedCategory)
                    }
                }
                .padding()
                .background(Color(.systemGray6))
                .clipShape(RoundedRectangle(cornerRadius: 12))
            }
        }
    }

    private func parseInput() {
        guard !inputText.isEmpty else { return }
        isParsing = true

        Task {
            if let result = await parseService.parse(inputText) {
                parsedAmount = String(format: "%.2f", result.amount)
                parsedCategory = result.category
            }
            isParsing = false
        }
    }

    private func saveExpense() {
        guard let amount = Double(parsedAmount) else { return }
        let expense = Expense(
            amount: amount,
            category: parsedCategory,
            note: inputText,
            isSmartParsed: true
        )
        modelContext.insert(expense)
        try? modelContext.save()
        dismiss()
    }
}
```

### Phase 3 产出物清单

| 产出物 | 格式 | 用途 |
|-------|------|------|
| 记账列表视图 | SwiftUI | 展示和管理记账记录 |
| 记账 ViewModel | Swift | 记账业务逻辑 |
| AI 对话 ViewModel | Swift | 对话功能逻辑 |
| 智能解析服务 | Swift | 自然语言→金额+分类 |
| 智能记账视图 | SwiftUI | 自然语言输入界面 |

⚠️ **警告**：Phase 3 是最容易"失控"的阶段。AI 生成代码很快，但如果不逐个审查，技术债会快速积累。建议每完成一个功能就运行一次，确保能正常工作再进入下一个。

---

## 5. Phase 4：AI 辅助测试与审查

代码写完了，但还没有经过验证。Phase 4 的目标是：**确保代码质量，发现并修复问题**。

### 用 AI 生成单元测试

#### Prompt 模板：测试用例生成

```
你是一位 iOS 测试工程师，擅长编写高质量的单元测试。

【项目背景】
项目：AI 记账助手
技术栈：SwiftUI + SwiftData + MVVM
测试框架：XCTest

【待测试代码】
（粘贴 SmartParseService 的代码）

【任务】
请为 SmartParseService 生成完整的单元测试，要求：
1. 覆盖所有公开方法
2. 测试正常路径和边界情况
3. 测试金额提取的各种输入格式
4. 测试分类识别的各个类别
5. 测试 LLM 解析的 Mock 方案
6. 每个测试方法名清晰表达测试意图

【约束】
- 使用 XCTest 框架
- LLM 调用使用 Mock，不发送真实请求
- 测试方法命名：test_方法名_场景_预期结果
- 使用 async/await 测试异步方法
```

AI 生成的测试代码：

```swift
import XCTest
@testable import AIExpenseTracker

final class SmartParseServiceTests: XCTestCase {

    var service: SmartParseService!

    override func setUp() {
        super.setUp()
        service = SmartParseService()
    }

    override func tearDown() {
        service = nil
        super.tearDown()
    }

    // MARK: - 金额提取

    func test_localParse_带元字_正确提取金额() {
        let result = service.localParse("午饭花了35元")
        XCTAssertEqual(result?.amount, 35.0)
    }

    func test_localParse_带小数_正确提取金额() {
        let result = service.localParse("打车12.5块")
        XCTAssertEqual(result?.amount, 12.5)
    }

    func test_localParse_带符号_正确提取金额() {
        let result = service.localParse("咖啡¥28")
        XCTAssertEqual(result?.amount, 28.0)
    }

    func test_localParse_无金额_返回nil() {
        let result = service.localParse("买了一杯咖啡")
        XCTAssertNil(result)
    }

    func test_localParse_纯数字_正确提取金额() {
        let result = service.localParse("午饭35")
        XCTAssertEqual(result?.amount, 35.0)
    }

    // MARK: - 分类识别

    func test_classifyCategory_餐饮关键词_返回餐饮() {
        let cases = ["午饭花了35", "外卖28", "奶茶15", "咖啡28"]
        for input in cases {
            let result = service.localParse(input)
            XCTAssertEqual(result?.category, "餐饮", "Failed for: \(input)")
        }
    }

    func test_classifyCategory_交通关键词_返回交通() {
        let cases = ["打车回家28", "地铁5块", "加油300元"]
        for input in cases {
            let result = service.localParse(input)
            XCTAssertEqual(result?.category, "交通", "Failed for: \(input)")
        }
    }

    func test_classifyCategory_无匹配关键词_返回其他() {
        let result = service.localParse("花了50")
        XCTAssertEqual(result?.category, "其他")
    }

    // MARK: - 置信度

    func test_localParse_本地解析_置信度为0点7() {
        let result = service.localParse("午饭35")
        XCTAssertEqual(result?.confidence, 0.7)
    }

    func test_localParse_本地解析_来源为local() {
        let result = service.localParse("午饭35")
        XCTAssertEqual(result?.source, .local)
    }
}
```

### 用 AI 进行代码审查

#### Prompt 模板：代码审查

```
你是一位资深 iOS 代码审查专家，请对以下代码进行全面审查。

【审查维度】
1. 正确性：逻辑是否正确，是否有潜在 Bug
2. 安全性：是否有安全隐患（API Key 泄露、SQL 注入等）
3. 性能：是否有性能问题（主线程阻塞、内存泄漏等）
4. 可维护性：代码是否清晰、是否易于修改
5. 最佳实践：是否遵循 Swift/iOS 开发最佳实践
6. SwiftUI 特定：视图是否过于复杂、状态管理是否正确

【待审查代码】
（粘贴代码）

【输出格式】
按严重程度分级：
🔴 严重问题：必须修复
🟡 建议改进：推荐修复
🟢 良好实践：值得保留

每个问题包含：位置、描述、修复建议。
最后给出总体评分（1-10）和总结。
```

AI 审查结果示例：

```markdown
## 代码审查报告

### 🔴 严重问题

1. **LLMService 中 API Key 可能泄露到日志**
   - 位置：LLMService.swift, createURLRequest 方法
   - 描述：URLRequest 的 header 可能被网络调试工具捕获
   - 建议：确保不在 console 输出完整的 URLRequest

2. **ChatViewModel 的 Task 未处理取消**
   - 位置：ChatViewModel.swift, sendMessage 方法
   - 描述：Task 创建后如果 ViewModel 被释放，Task 仍在运行
   - 建议：在 deinit 中取消 Task，使用 [weak self] 避免循环引用

### 🟡 建议改进

1. **SmartParseService 的正则表达式可以优化**
   - 位置：SmartParseService.swift, extractAmount 方法
   - 描述：多个正则逐个匹配效率低
   - 建议：合并为一个正则，使用捕获组

2. **ExpenseListView 缺少加载状态**
   - 位置：ExpenseListView.swift
   - 描述：SwiftData 查询可能需要时间，没有 loading 状态
   - 建议：添加 @Query 的 animation 参数

3. **日期格式化器应缓存**
   - 位置：ExpenseRowView.swift, formatDate 方法
   - 描述：每次渲染都创建新的 DateFormatter，性能差
   - 建议：使用静态属性缓存 DateFormatter

### 🟢 良好实践

1. 使用 @Observable 而非 ObservableObject，符合 iOS 17+ 最佳实践
2. 错误处理使用自定义枚举 + LocalizedError，用户友好
3. 流式输出使用 AsyncThrowingStream，Swift Concurrency 最佳实践

### 总体评分：7/10
代码整体质量不错，架构清晰，命名规范。主要问题在内存管理和性能优化方面，修复后可达 9/10。
```

### 用 AI 修复发现的问题

根据审查结果，让 AI 逐个修复问题：

```
请修复以下代码审查中发现的问题：

【问题 1】ChatViewModel 的 Task 未处理取消
- 在 deinit 中取消 currentStreamTask
- 使用 [weak self] 避免循环引用

【问题 2】DateFormatter 应缓存
- 将 DateFormatter 改为静态属性

【问题 3】SmartParseService 正则优化
- 合并多个正则为一个

请直接给出修复后的代码，标注修改位置。
```

修复后的关键代码片段：

```swift
@Observable
final class ChatViewModel {
    var messages: [ChatMessage] = []
    var isLoading = false
    var streamingContent = ""
    var errorMessage: String?

    private let llmService = LLMService()
    private var modelContext: ModelContext?
    private var currentStreamTask: Task<Void, Never>?

    deinit {
        currentStreamTask?.cancel()
    }

    func sendMessage(_ text: String) {
        let userMessage = ChatMessage(role: "user", content: text)
        messages.append(userMessage)
        persistMessage(userMessage)

        isLoading = true
        streamingContent = ""
        errorMessage = nil

        currentStreamTask = Task { [weak self] in
            guard let self else { return }
            do {
                let systemPrompt = self.buildSystemPrompt()
                let llmMessages = self.buildLLMMessages(systemPrompt: systemPrompt)

                var fullContent = ""
                let stream = self.llmService.chatStream(messages: llmMessages)

                for try await chunk in stream {
                    guard !Task.isCancelled else { return }
                    fullContent += chunk
                    self.streamingContent = fullContent
                }

                let assistantMessage = ChatMessage(role: "assistant", content: fullContent)
                self.messages.append(assistantMessage)
                self.persistMessage(assistantMessage)
            } catch {
                self.errorMessage = "AI 助手暂时无法回复，请稍后再试"
            }

            self.isLoading = false
            self.streamingContent = ""
        }
    }
}
```

```swift
struct ExpenseRowView: View {
    let expense: Expense

    private static let dateFormatter: DateFormatter = {
        let formatter = DateFormatter()
        formatter.dateFormat = "MM月dd日"
        return formatter
    }()

    private func formatDate(_ date: Date) -> String {
        let calendar = Calendar.current
        if calendar.isDateInToday(date) { return "今天" }
        if calendar.isDateInYesterday(date) { return "昨天" }
        return Self.dateFormatter.string(from: date)
    }
}
```

### Phase 4 产出物清单

| 产出物 | 格式 | 用途 |
|-------|------|------|
| 单元测试套件 | Swift (XCTest) | 验证核心逻辑正确性 |
| 代码审查报告 | Markdown | 记录发现的问题和改进建议 |
| 修复后的代码 | Swift | 修复审查发现的问题 |
| 测试覆盖率报告 | Xcode Report | 量化测试覆盖程度 |

💡 **提示**：AI 代码审查不是一次性的事情。建议每完成一个功能模块就做一次审查，而不是等所有代码写完再审查。早期发现问题，修复成本远低于后期。

---

## 6. Phase 5：AI 辅助上架

代码开发和测试完成后，最后一步是准备上架材料。这个阶段 AI 特别擅长——因为 App Store 的文案、关键词、隐私政策等都是文本工作，正是 AI 的强项。

### 用 AI 生成 App Store 元数据

#### Prompt 模板：App 描述与关键词

```
你是一位 App Store 优化（ASO）专家，擅长撰写高转化率的 App 描述。

【App 信息】
- 名称：AI 记账助手
- 核心功能：智能记账、AI 对话、统计分析
- 差异化：自然语言输入，AI 自动识别金额和分类
- 目标用户：18-35 岁年轻人
- 价格：免费（基础功能），高级版 ¥12/月（无限 AI 对话）

【任务】
请生成以下 App Store 元数据：
1. 副标题（30 字以内）
2. 描述（前 3 行最关键，需要吸引眼球）
3. 关键词（100 字以内，逗号分隔）
4. 新功能描述（1.0 版本）
5. 预览文案（3-5 张截图的文案）

【约束】
- 面向中国大陆市场，使用简体中文
- 描述要突出"智能"和"简单"两大卖点
- 关键词要覆盖用户搜索习惯
- 符合 App Store 审核规范
```

AI 生成的 App Store 元数据：

```markdown
## 副标题
说句话就能记账，AI 帮你自动分类

## 描述
还在手动输入金额和分类？太麻烦了！

AI 记账助手让记账变得像聊天一样简单——只需说"午饭花了35"，AI 自动识别金额和分类，1秒完成记账。

✨ 核心功能：
• 智能记账：自然语言输入，AI 自动提取金额和分类
• AI 助手：随时对话，获取消费分析和理财建议
• 统计图表：饼图+趋势图，消费一目了然
• 预算管理：设置月度预算，超支及时提醒

🎯 为什么选择 AI 记账助手？
- 记账速度提升 10 倍：从打开 App 到完成记账，只需 5 秒
- 分类准确率 95%+：AI 智能识别，无需手动选择分类
- 消费洞察：AI 分析你的消费习惯，给出个性化建议

💡 基础功能完全免费，升级高级版解锁无限 AI 对话和高级分析。

## 关键词
记账,智能记账,AI记账,记账本,记账app,理财,消费记录,账单,预算,支出管理

## 新功能（1.0 版本）
🎉 首次发布！
- 智能记账：自然语言输入，AI 自动识别
- AI 助手：消费分析和理财建议
- 统计图表：消费数据可视化
- 预算管理：月度预算设置与提醒
```

### 用 AI 辅助截图文案

App Store 截图是用户决定是否下载的关键因素。让 AI 帮你设计截图文案：

```markdown
## 截图文案设计

| 截图序号 | 主标题 | 副标题 | 展示内容 |
|---------|--------|--------|---------|
| 1 | 说句话就能记账 | AI 自动识别金额和分类 | 智能记账输入界面 |
| 2 | 你的 AI 理财助手 | 消费分析，智能建议 | AI 对话界面 |
| 3 | 消费一目了然 | 饼图+趋势图，清晰直观 | 统计图表界面 |
| 4 | 预算超支提醒 | 合理消费，不再月光 | 预算管理界面 |
| 5 | 5秒完成记账 | 从打开到完成，超快体验 | 完整记账流程 |
```

### 用 AI 生成隐私政策

隐私政策是 App Store 上架的必要文件。AI 可以根据你的 App 实际数据使用情况生成合规的隐私政策。

```
你是一位法律顾问，擅长撰写 App 隐私政策。

【App 信息】
- 名称：AI 记账助手
- 收集的数据：记账记录（本地存储）、对话历史（本地存储）
- 网络请求：DeepSeek API（发送对话内容用于 AI 回复）
- 第三方服务：无
- 数据存储：全部本地存储（SwiftData），不上传服务器
- 位置信息：不收集
- 广告：无

【任务】
请生成符合中国《个人信息保护法》和 App Store 审核要求的隐私政策，包含：
1. 信息收集与使用
2. 信息存储与安全
3. 信息共享与披露
4. 用户权利
5. 未成年人保护
6. 隐私政策更新
7. 联系方式

【约束】
- 使用简体中文
- 符合中国大陆法律法规
- 语言通俗易懂，非法律专业人士也能理解
```

### 用 AI 准备审核说明

App Store 审核时，如果 App 包含 AI 功能，需要提供额外说明：

```markdown
## 审核说明

Dear App Store Review Team,

感谢审核我们的 App。以下是关于 AI 功能的说明：

1. **AI 功能说明**
   - 本 App 使用 DeepSeek API 提供 AI 对话功能
   - AI 功能用于：消费分析建议、自然语言记账解析
   - 用户可以在设置中配置自己的 API Key，也可以使用内置的共享 Key

2. **内容审核机制**
   - AI 输出包含系统提示词约束，限制回复范围在记账和理财领域
   - 不生成任何违法、色情、暴力内容
   - 用户可以举报不当回复

3. **数据使用**
   - 对话内容仅发送至 DeepSeek API 用于生成回复
   - 不存储用户对话到我们自己的服务器
   - 用户可以随时清除对话历史

4. **测试账号**
   - 提供测试用的 API Key：（如有）
   - 或说明：用户需要自行配置 API Key

5. **隐私政策**
   - 已在 App 内和 App Store 提供隐私政策链接
```

### Phase 5 产出物清单

| 产出物 | 格式 | 用途 |
|-------|------|------|
| App 描述与关键词 | 文本 | App Store 展示信息 |
| 截图文案 | 文本 | 截图设计参考 |
| 隐私政策 | HTML / Markdown | 合规文件 |
| 审核说明 | 文本 | 提交审核时的说明 |
| App 分类与年龄分级 | 文本 | App Store 分类信息 |

💡 **提示**：AI 生成的隐私政策和审核说明需要人工审核。特别是隐私政策，涉及法律合规，建议让有法律背景的人审阅，或使用专业法律模板作为基础。

---

## 7. AI 驱动开发的效率分析

### 传统开发 vs AI 驱动开发的时间对比表

以"AI 记账助手"项目为例，对比传统开发和 AI 驱动开发的时间消耗：

| 开发阶段 | 传统开发 | AI 驱动开发 | 节省时间 | 节省比例 |
|---------|---------|-----------|---------|---------|
| 需求分析 & PRD | 4h | 1h | 3h | 75% |
| 技术选型 | 2h | 0.5h | 1.5h | 75% |
| 任务拆解 | 2h | 0.5h | 1.5h | 75% |
| 项目搭建 | 2h | 1h | 1h | 50% |
| 数据层开发 | 4h | 1.5h | 2.5h | 63% |
| 网络层开发 | 6h | 2h | 4h | 67% |
| UI 开发 | 12h | 5h | 7h | 58% |
| 业务逻辑开发 | 8h | 3h | 5h | 63% |
| AI 功能集成 | 8h | 3h | 5h | 63% |
| 单元测试 | 6h | 2h | 4h | 67% |
| 代码审查 | 3h | 1h | 2h | 67% |
| Bug 修复 | 4h | 2h | 2h | 50% |
| 上架材料准备 | 4h | 1h | 3h | 75% |
| **合计** | **65h** | **24h** | **41h** | **63%** |

### 各阶段 AI 节省的时间比例

```
节省比例可视化：

需求分析  ████████████████████ 75%
技术选型  ████████████████████ 75%
任务拆解  ████████████████████ 75%
项目搭建  ██████████████       50%
数据层    █████████████████    63%
网络层    ██████████████████   67%
UI 开发   ████████████████     58%
业务逻辑  █████████████████    63%
AI 集成   █████████████████    63%
单元测试  ██████████████████   67%
代码审查  ██████████████████   67%
Bug 修复  ██████████████       50%
上架材料  ████████████████████ 75%
```

### AI 驱动开发的质量评估

效率提升不能以牺牲质量为代价。以下是 AI 驱动开发的质量评估：

| 质量维度 | 传统开发 | AI 驱动开发 | 评估 |
|---------|---------|-----------|------|
| **代码正确性** | 依赖开发者水平 | AI 生成 + 人工审查 | 略低，需加强审查 |
| **代码规范性** | 因人而异 | AI 遵循规范更一致 | 更好 |
| **测试覆盖率** | 通常 30-50% | AI 可达 70-80% | 显著更好 |
| **安全审计** | 容易遗漏 | AI 系统性检查 | 更好 |
| **文档完整性** | 经常缺失 | AI 可自动生成 | 显著更好 |
| **Bug 密度** | 中等 | 略高（AI 幻觉导致） | 需要更多测试 |
| **可维护性** | 依赖架构设计 | AI 遵循架构更一致 | 相当 |

⚠️ **警告**：AI 驱动开发最大的质量风险是"看起来对但实际有 Bug"。AI 生成的代码往往表面逻辑正确，但边界情况处理不当。**测试覆盖率是 AI 驱动开发的生命线**。

### 适合 AI 辅助 vs 不适合 AI 辅助的任务分类表

| 适合 AI 辅助 ✅ | 不适合 AI 辅助 ❌ |
|----------------|------------------|
| CRUD 操作代码 | 核心业务规则定义 |
| UI 视图代码（SwiftUI） | 架构设计决策 |
| 单元测试生成 | 用户体验设计 |
| API 调用封装 | 性能关键路径优化 |
| 数据模型定义 | 复杂动画实现 |
| 文档和注释 | 安全敏感的加密逻辑 |
| 正则表达式 | 多线程竞态条件处理 |
| 格式转换代码 | 底层硬件交互 |
| 配置文件编写 | 算法核心逻辑 |
| App Store 文案 | 产品方向决策 |

💡 **提示**：一个简单的判断标准——**如果这个任务有明确的输入和输出，适合 AI；如果需要创造性判断或深度业务理解，不适合 AI**。

---

## 8. 常见问题与解决方案

### AI 生成的代码不符合预期怎么办

这是最常见的问题。AI 生成的代码"看起来对但用起来不对"，通常有以下原因和解决方案：

| 原因 | 症状 | 解决方案 |
|------|------|---------|
| **上下文不足** | AI 不知道项目用了什么库、什么架构 | 提供完整的 CLAUDE.md / .cursorrules |
| **需求不清晰** | AI 生成的功能和你想的不一样 | 使用 5 要素法重写 Prompt |
| **一次要求太多** | AI 生成的代码又长又乱 | 拆成小任务，每次只做一件事 |
| **缺少约束** | AI 使用了不存在的 API 或库 | 在 Prompt 中明确指定可用 API |
| **缺少示例** | AI 的输出格式不符合预期 | 提供期望的输出示例 |

**实操步骤**：

```
1. 不要直接说"你写错了"——AI 不理解"错"在哪里
2. 明确指出哪里不符合预期
3. 给出正确的期望行为
4. 让 AI 重新生成

示例：
❌ "这个代码不对，重写"
✅ "这个视图缺少空状态处理。当列表为空时，应该显示 ContentUnavailableView，请补充这个逻辑"
```

### AI 幻觉导致的技术方案错误

AI 幻觉是指 AI 编造不存在的信息。在技术领域，常见的幻觉类型：

| 幻觉类型 | 示例 | 危害等级 | 检测方法 |
|---------|------|---------|---------|
| **虚构 API** | 调用不存在的 `SwiftUI.LazyVGrid.columns` | 🔴 高 | 编译验证 |
| **虚构参数** | `@Query(filter:)` 使用了不存在的参数 | 🔴 高 | 查阅官方文档 |
| **虚构库** | 推荐使用 `SwiftUIX` 但项目中未集成 | 🟡 中 | 检查依赖 |
| **错误版本信息** | 声称某 API 在 iOS 16 可用但实际需要 17 | 🟡 中 | 查阅 Release Notes |
| **错误用法** | SwiftData 的 `#Predicate` 语法错误 | 🔴 高 | 编译验证 |

**应对策略**：

```
1. 编译验证：AI 生成的代码必须编译通过
2. 文档对照：关键 API 使用前查阅官方文档
3. 渐进式开发：每次只让 AI 生成一小段，立即验证
4. 交叉验证：用不同的 AI 工具验证同一个方案
5. 版本确认：涉及 iOS 版本兼容性时，明确指定目标版本
```

### 多个 AI 工具的协作策略

不同的 AI 工具各有所长，合理组合可以发挥最大价值：

| 工具 | 最擅长 | 适合的场景 |
|------|-------|----------|
| **Claude Code** | 长上下文理解、代码生成、文件操作 | 项目搭建、批量代码生成、重构 |
| **Cursor** | 实时代码补全、上下文感知 | 日常编码、快速修改 |
| **GitHub Copilot** | 行级补全、模式匹配 | 重复性代码、样板代码 |
| **Trae** | 中文理解、国内生态 | 中文文案、国内 API 集成 |
| **OpenAI Codex** | 异步任务、CI/CD 集成 | 自动化测试、批量操作 |

**推荐协作模式**：

```
项目规划阶段：Claude Code（生成 Spec、架构设计）
    ↓
日常编码阶段：Cursor（实时补全）+ Copilot（行级建议）
    ↓
复杂功能开发：Claude Code（完整代码生成）
    ↓
代码审查阶段：Claude Code（全面审查）+ Trae（中文文案审查）
    ↓
上架准备阶段：Claude Code + Trae（文案生成）
```

### 何时该自己写代码 vs 让 AI 写

| 场景 | 建议 | 理由 |
|------|------|------|
| 标准的 CRUD 操作 | 🤖 让 AI 写 | 模式固定，AI 写得又快又好 |
| 复杂的业务逻辑 | 👤 自己写 | 需要深度理解业务，AI 容易出错 |
| UI 视图代码 | 🤖 让 AI 写 + 👤 调整 | AI 可以快速生成框架，细节需要人工调整 |
| 单元测试 | 🤖 让 AI 写 | AI 覆盖边界情况更全面 |
| 核心算法 | 👤 自己写 | 算法正确性至关重要，AI 幻觉风险高 |
| 配置文件 | 🤖 让 AI 写 | 格式固定，AI 不容易出错 |
| 性能优化 | 👤 自己写 + 🤖 辅助 | 需要理解瓶颈，AI 可以提供优化建议 |
| 文档和文案 | 🤖 让 AI 写 | 文本生成是 AI 的强项 |

💡 **提示**：一个实用的判断方法——**如果你能在 30 秒内清楚描述要做什么，让 AI 写；如果你需要思考 5 分钟才能理清逻辑，自己写**。

---

## 9. AI 驱动开发的最佳实践总结

### Spec 先行的原则

Spec 先行是 AI 驱动开发的第一原则。没有 Spec，AI 就没有方向。

```
Spec 先行的执行要点：

1. 每个 Feature 开发前，先写一个简短的 Spec
   - 这个功能做什么？
   - 输入是什么？输出是什么？
   - 有什么约束和边界条件？

2. Spec 不需要很长，但必须清晰
   - 好的 Spec：3-5 行，清晰明确
   - 差的 Spec：20 行，模糊不清

3. Spec 写好后，让 AI 基于 Spec 生成代码
   - AI 按 Spec 生成 → 你按 Spec 验收
   - 形成闭环，确保结果符合预期

4. Spec 可以迭代
   - 第一次 Spec 不完美没关系
   - 根据 AI 的输出调整 Spec
   - Spec 越写越好，Prompt 也越来越精准
```

### 增量式开发策略

不要让 AI 一次性生成大量代码。增量式开发是 AI 驱动开发的核心策略：

| 策略 | 说明 | 示例 |
|------|------|------|
| **功能增量** | 每次只做一个功能 | 先做记账列表，再做 AI 对话 |
| **层次增量** | 先数据层，再逻辑层，最后 UI 层 | 先写 Model，再写 ViewModel，最后写 View |
| **质量增量** | 先让它跑起来，再优化质量 | 先实现基本功能，再做代码审查和优化 |
| **复杂度增量** | 先做简单版本，再增加复杂度 | 先做本地规则解析，再集成 LLM 解析 |

```
增量式开发的节奏：

Step 1: 让 AI 生成最简版本（能跑就行）
Step 2: 运行验证，确认基本功能正常
Step 3: 让 AI 补充边界处理和错误处理
Step 4: 让 AI 生成测试
Step 5: 让 AI 审查代码
Step 6: 根据审查结果修复问题
Step 7: 进入下一个功能

每个 Step 都是一个完整的"生成→验证"循环。
```

### 持续审查与验证

AI 生成的代码必须持续审查和验证，不能"生成完就完事"：

```
审查与验证的检查清单：

□ 编译通过：代码能成功编译，没有语法错误
□ 功能正确：核心功能按预期工作
□ 边界处理：空值、零值、极大值等边界情况已处理
□ 错误处理：网络错误、数据异常等错误情况已处理
□ 安全检查：没有硬编码密钥、没有注入风险
□ 性能检查：没有主线程阻塞、没有内存泄漏
□ 代码规范：命名、格式、注释符合项目规范
□ 测试覆盖：关键逻辑有对应的单元测试
```

### 人机协作的黄金比例

AI 驱动开发不是让 AI 做所有事，而是找到人和 AI 的最佳分工：

```
理想的分工比例：

规划阶段：人 70% + AI 30%
  人：定义产品方向、业务规则、优先级
  AI：辅助生成文档、分析竞品、格式化输出

开发阶段：人 30% + AI 70%
  人：审查代码、处理复杂逻辑、做架构决策
  AI：生成代码、编写测试、处理重复性工作

测试阶段：人 40% + AI 60%
  人：设计测试策略、验证关键路径、评估用户体验
  AI：生成测试用例、执行代码审查、分析覆盖率

上架阶段：人 20% + AI 80%
  人：审核合规性、确认品牌调性
  AI：生成文案、设计截图文案、准备审核材料
```

### 从 AI 辅助到 AI 驱动的进阶路径

```
Level 1：AI 辅助（入门）
├── 用 AI 回答编程问题
├── 用 AI 生成代码片段
└── 用 AI 解释报错信息

Level 2：AI 协作（进阶）
├── 用 AI 生成完整功能
├── 用 AI 编写测试
├── 用 AI 审查代码
└── 配置项目规范文件

Level 3：AI 驱动（高级）
├── Spec 先行的开发流程
├── AI 生成 + 人工审查的闭环
├── 多 AI 工具协作
└── Prompt 模板库积累

Level 4：AI 原生（专家）
├── AI Agent 自主编程
├── 自定义 Skill 和工作流
├── MCP 协议集成
└── AI 驱动的 CI/CD
```

每个 Level 的关键能力：

| Level | 核心能力 | 时间投入 | 效率提升 |
|-------|---------|---------|---------|
| Level 1 | 会用 AI 工具 | 1-2 周 | 10-20% |
| Level 2 | 会与 AI 协作 | 1-2 月 | 30-50% |
| Level 3 | 会驱动 AI | 2-3 月 | 50-70% |
| Level 4 | 会设计 AI 工作流 | 3-6 月 | 70-90% |

💡 **提示**：不要急于跳级。Level 2 是最关键的阶段——在这个阶段，你建立了"AI 生成 + 人工审查"的习惯，这是 AI 驱动开发的基础。跳过这个阶段直接用 AI Agent，很容易产生大量低质量代码。

---

## 小结

本章通过"AI 记账助手"的完整实战，展示了 AI 驱动端到端开发的 5 个阶段：

| 阶段 | 核心任务 | AI 的价值 | 关键产出 |
|------|---------|----------|---------|
| **Phase 1** | 写 Spec | 快速生成 PRD、技术方案、任务拆解 | 规范文档 |
| **Phase 2** | 搭建项目 | 生成架构、网络层、数据层 | 可运行的项目骨架 |
| **Phase 3** | 核心开发 | 生成 UI、业务逻辑、AI 功能 | 功能完整的 App |
| **Phase 4** | 测试审查 | 生成测试、审查代码、修复问题 | 质量达标的代码 |
| **Phase 5** | 上架准备 | 生成文案、隐私政策、审核说明 | 可提交的上架材料 |

**核心要点回顾**：

1. **Spec 先行**：先写清楚要什么，再让 AI 去做。Spec 是 AI 驱动开发的方向盘。
2. **增量开发**：每次只让 AI 做一件事，做完验证再做下一件。不要一次性生成大量代码。
3. **持续审查**：AI 生成的代码必须审查。编译验证 + 功能验证 + 代码审查，三重保障。
4. **人机协作**：人做决策，AI 做执行。找到人和 AI 的最佳分工，效率最大化。
5. **Prompt 模板**：积累和使用 Prompt 模板，让 AI 的输出越来越精准。

AI 驱动开发不是让 AI 替代你，而是让 AI 成为你的超级搭档。你的判断力和决策力依然是项目成功的关键——AI 只是让你从重复劳动中解放出来，把精力集中在真正需要创造力的地方。

---

← [国内大模型与AI生态](./国内大模型与AI生态.md) | [初识Swift](../03-Swift语言基础/初识Swift.md) →
