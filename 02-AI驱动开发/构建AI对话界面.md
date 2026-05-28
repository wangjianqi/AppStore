# 构建 AI 对话界面

> 🎯 **本章目标**：
> - 理解 AI 对话界面的设计模式与核心元素
> - 掌握消息、对话的数据模型设计
> - 使用 SwiftUI 实现完整的聊天界面（消息列表、输入框、自动滚动）
> - 实现流式输出渲染与打字指示器
> - 管理多轮对话上下文与 Token 预算
> - 在 SwiftUI 中渲染 Markdown 内容与代码高亮
> - 支持多模态输入（图片上传）
> - 掌握体验优化技巧（乐观更新、持久化、深色模式）
> - 实现无障碍与国际化支持

---

## 1. AI 对话界面的设计模式

### 1.1 聊天 UI 的核心元素

一个完整的 AI 对话界面由以下核心元素组成：

| 元素 | 作用 | 设计要点 |
|------|------|---------|
| **消息气泡** | 承载对话内容 | 用户与 AI 需视觉区分，AI 气泡需支持 Markdown |
| **输入框** | 接收用户输入 | 支持多行、自动增高、placeholder 提示 |
| **发送按钮** | 触发消息发送 | 发送中禁用，支持长按附加功能 |
| **打字指示器** | 表示 AI 正在生成 | 三点跳动动画，提升等待体验 |
| **滚动区域** | 展示历史消息 | 自动滚动到底部，支持手动上滑 |
| **工具栏** | 附加操作入口 | 新对话、图片上传、语音输入等 |

这些元素之间的关系：

```
┌─────────────────────────────────┐
│  导航栏（对话标题 / 新建对话）     │
├─────────────────────────────────┤
│                                 │
│  ┌──────────────┐               │
│  │ AI 消息气泡   │  ← Markdown  │
│  └──────────────┘               │
│                                 │
│         ┌──────────────┐        │
│         │ 用户消息气泡  │ ← 纯文本│
│         └──────────────┘        │
│                                 │
│  ┌──────────────┐               │
│  │ ● ● ●       │  ← 打字指示器  │
│  └──────────────┘               │
│                                 │
├─────────────────────────────────┤
│  [附件] [输入框......] [发送]    │
└─────────────────────────────────┘
```

### 1.2 主流 AI App 的 UI 模式分析

不同 AI 应用的界面设计各有侧重，下表对比了主流产品的设计选择：

| 设计维度 | ChatGPT App | Claude App | 豆包 App | Kimi App |
|---------|-------------|-----------|---------|---------|
| **消息对齐** | AI 居左，用户居右 | AI 居左，用户居右 | AI 居左，用户居右 | AI 居左，用户居右 |
| **气泡样式** | 圆角卡片，无背景色区分 | 圆角气泡，淡色区分 | 圆角气泡，颜色区分 | 圆角气泡，颜色区分 |
| **AI 头像** | OpenAI Logo | Claude Logo | 豆包形象 | Kimi Logo |
| **输入框位置** | 底部固定 | 底部固定 | 底部固定 | 底部固定 |
| **流式动画** | 逐字显示 + 光标 | 逐字显示 + 光标 | 逐字显示 | 逐字显示 |
| **代码块** | 深色背景 + 复制按钮 | 深色背景 + 复制 | 深色背景 + 复制 | 深色背景 + 复制 |
| **多模态入口** | + 按钮展开 | 附件图标 | + 按钮展开 | + 按钮展开 |
| **侧边栏** | 左滑抽屉 | 左滑抽屉 | 底部 Tab | 左滑抽屉 |

> 💡 **提示**：虽然各产品风格不同，但核心交互模式高度一致——AI 消息居左、用户消息居右、底部输入框、逐字流式输出。这已经成为用户的心理模型，设计时应遵循这一惯例。

### 1.3 iOS HIG 中对话界面的设计建议

Apple 的 Human Interface Guidelines 对聊天类界面有以下建议：

| HIG 原则 | 对话界面的应用 |
|----------|--------------|
| **一致性** | 消息气泡样式、间距、字号保持统一 |
| **直接操作** | 长按消息可复制/删除，下拉刷新历史 |
| **隐喻** | 气泡隐喻真实对话气泡，输入框隐喻文本框 |
| **用户控制** | 允许中断 AI 生成、重新生成、编辑已发消息 |
| **反馈** | 发送状态、流式输出、错误提示均需明确反馈 |
| **容错** | 网络错误时保留消息，支持重试 |

⚠️ **警告**：不要将聊天界面做成普通的列表视图。AI 对话有独特的交互需求——流式输出、长文本渲染、代码块交互等，这些都需要专门的组件来处理。

### 1.4 消息气泡样式对比

用户消息与 AI 消息的视觉区分是对话界面最基本的设计要求：

| 属性 | 用户消息 | AI 消息 |
|------|---------|--------|
| **对齐方式** | 右对齐 | 左对齐 |
| **背景色（浅色模式）** | 系统蓝色 / 品牌色 | 浅灰色 / 白色 |
| **背景色（深色模式）** | 深蓝色 | 深灰色 |
| **文字颜色** | 白色 | 黑色 / 白色 |
| **最大宽度** | 屏幕 75% | 屏幕 85%（AI 回复通常更长） |
| **圆角** | 右下角直角 | 左下角直角 |
| **头像** | 可选 | 建议显示 |
| **内容格式** | 纯文本 | Markdown（代码块、列表、表格等） |
| **长按操作** | 复制、删除 | 复制、重新生成 |

纯文本消息与 Markdown 消息的渲染差异：

| 内容类型 | 纯文本渲染 | Markdown 渲染 |
|---------|-----------|-------------|
| **普通段落** | Text 视图 | AttributedString / Markdown 库 |
| **代码块** | 等宽字体 | 深色背景 + 语法高亮 + 复制按钮 |
| **列表** | 无 | 缩进 + 项目符号 |
| **表格** | 无 | 表格视图 |
| **粗体/斜体** | 无 | 字体样式变化 |
| **链接** | 纯文本 | 可点击链接 |

---

## 2. 数据模型设计

### 2.1 Message 模型

消息是 AI 对话的核心数据单元：

```swift
import Foundation

struct Message: Identifiable, Equatable {
    let id: UUID
    let role: MessageRole
    var content: String
    let timestamp: Date
    var status: MessageStatus
    var images: [MessageImage]?

    init(
        id: UUID = UUID(),
        role: MessageRole,
        content: String,
        timestamp: Date = Date(),
        status: MessageStatus = .sent,
        images: [MessageImage]? = nil
    ) {
        self.id = id
        self.role = role
        self.content = content
        self.timestamp = timestamp
        self.status = status
        self.images = images
    }

    static func == (lhs: Message, rhs: Message) -> Bool {
        lhs.id == rhs.id
    }
}
```

### 2.2 角色枚举

```swift
enum MessageRole: String, Codable, CaseIterable {
    case user
    case assistant
    case system

    var isUser: Bool { self == .user }
}
```

角色枚举对应大模型 API 的三种消息角色：

| 角色 | 用途 | 是否在 UI 中显示 |
|------|------|----------------|
| `user` | 用户发送的消息 | ✅ 显示在右侧 |
| `assistant` | AI 的回复 | ✅ 显示在左侧 |
| `system` | 系统提示词 | ❌ 不显示 |

### 2.3 消息状态枚举

```swift
enum MessageStatus: Equatable {
    case sending
    case sent
    case streaming
    case error(String)

    var isError: Bool {
        if case .error = self { return true }
        return false
    }

    var isStreaming: Bool { self == .streaming }

    static func == (lhs: MessageStatus, rhs: MessageStatus) -> Bool {
        switch (lhs, rhs) {
        case (.sending, .sending): return true
        case (.sent, .sent): return true
        case (.streaming, .streaming): return true
        case (.error, .error): return true
        default: return false
        }
    }
}
```

消息状态的生命周期：

```
用户点击发送 → sending → sent（成功）/ error（失败）
AI 开始回复 → streaming → sent（完成）/ error（失败）
```

| 状态 | 含义 | UI 表现 |
|------|------|--------|
| `sending` | 消息正在发送中 | 发送按钮转圈 |
| `sent` | 消息已送达 | 正常显示 |
| `streaming` | AI 正在流式输出 | 逐字显示 + 光标闪烁 |
| `error` | 发送/接收失败 | 红色提示 + 重试按钮 |

### 2.4 Conversation 模型

```swift
struct Conversation: Identifiable {
    let id: UUID
    var title: String
    var messages: [Message]
    var systemPrompt: String?
    var createdAt: Date
    var updatedAt: Date

    init(
        id: UUID = UUID(),
        title: String = "新对话",
        messages: [Message] = [],
        systemPrompt: String? = nil,
        createdAt: Date = Date(),
        updatedAt: Date = Date()
    ) {
        self.id = id
        self.title = title
        self.messages = messages
        self.systemPrompt = systemPrompt
        self.createdAt = createdAt
        self.updatedAt = updatedAt
    }

    var lastMessage: Message? {
        messages.last
    }

    var messageCount: Int {
        messages.filter { $0.role != .system }.count
    }
}
```

### 2.5 多模态消息扩展

```swift
struct MessageImage: Identifiable, Codable {
    let id: UUID
    let base64Data: String
    let mimeType: String
    let width: CGFloat
    let height: CGFloat

    var imageURL: URL? {
        guard let data = Data(base64Encoded: base64Data) else { return nil }
        let tempDir = FileManager.default.temporaryDirectory
        let fileURL = tempDir.appendingPathComponent("\(id.uuidString).\(fileExtension)")
        try? data.write(to: fileURL)
        return fileURL
    }

    private var fileExtension: String {
        switch mimeType {
        case "image/png": return "png"
        case "image/jpeg": return "jpg"
        case "image/webp": return "webp"
        default: return "jpg"
        }
    }
}
```

---

## 3. SwiftUI 聊天界面实现

### 3.1 ChatView 主视图结构

```swift
import SwiftUI

struct ChatView: View {
    @StateObject private var viewModel = ChatViewModel()
    @State private var inputText = ""
    @State private var isInputFocused = false

    var body: some View {
        VStack(spacing: 0) {
            messageListView
            Divider()
            inputBarView
        }
        .navigationTitle(viewModel.conversation.title)
        .navigationBarTitleDisplayMode(.inline)
        .toolbar {
            ToolbarItem(placement: .topBarTrailing) {
                Button {
                    viewModel.startNewConversation()
                } label: {
                    Image(systemName: "plus.bubble")
                }
            }
        }
    }
}
```

### 3.2 消息列表

使用 `LazyVStack` + `ScrollViewReader` 实现高性能消息列表：

```swift
extension ChatView {
    var messageListView: some View {
        ScrollViewReader { proxy in
            ScrollView {
                LazyVStack(spacing: 12) {
                    ForEach(viewModel.messages) { message in
                        MessageBubbleView(message: message)
                            .id(message.id)
                    }
                }
                .padding()
            }
            .onChange(of: viewModel.messages.count) { _ in
                scrollToBottom(proxy: proxy)
            }
            .onChange(of: viewModel.lastMessageContent) { _ in
                scrollToBottom(proxy: proxy)
            }
        }
    }

    private func scrollToBottom(proxy: ScrollViewProxy) {
        guard let lastMessage = viewModel.messages.last else { return }
        withAnimation(.easeOut(duration: 0.2)) {
            proxy.scrollTo(lastMessage.id, anchor: .bottom)
        }
    }
}
```

> 💡 **提示**：使用 `LazyVStack` 而非 `VStack`，因为聊天消息可能很多，懒加载可以显著提升性能。`ScrollViewReader` 配合 `scrollTo` 实现自动滚动到底部。

### 3.3 消息气泡组件

```swift
struct MessageBubbleView: View {
    let message: Message

    var body: some View {
        HStack(alignment: .top, spacing: 8) {
            if message.role.isUser {
                Spacer(minLength: 60)
            }

            if !message.role.isUser {
                avatarView
            }

            bubbleContent

            if !message.role.isUser {
                Spacer(minLength: 60)
            }

            if message.role.isUser {
                bubbleContent
            }
        }
    }

    private var avatarView: some View {
        Image(systemName: "sparkles")
            .font(.title3)
            .foregroundStyle(.white)
            .frame(width: 32, height: 32)
            .background(Color.purple.gradient)
            .clipShape(Circle())
    }

    private var bubbleContent: some View {
        VStack(alignment: .leading, spacing: 4) {
            if message.status.isStreaming && message.content.isEmpty {
                TypingIndicatorView()
            } else {
                MessageContentView(message: message)
            }

            if case .error(let errorMessage) = message.status {
                HStack(spacing: 4) {
                    Image(systemName: "exclamationmark.circle")
                    Text(errorMessage)
                }
                .font(.caption)
                .foregroundStyle(.red)
            }
        }
        .padding(.horizontal, 14)
        .padding(.vertical, 10)
        .background(bubbleBackground)
        .clipShape(bubbleShape)
    }

    private var bubbleBackground: Color {
        message.role.isUser
            ? Color.accentColor
            : Color(.systemGray6)
    }

    private var bubbleShape: some Shape {
        RoundedRectangle(cornerRadius: 18, style: .continuous)
    }
}
```

⚠️ **警告**：上面的代码有一个常见错误——`bubbleContent` 在 `HStack` 中被引用了两次（用户和 AI 各一次），这会导致编译错误。正确的做法是将 `bubbleContent` 提取为独立变量，只在 body 中使用一次。下面是修正版：

```swift
struct MessageBubbleView: View {
    let message: Message

    var body: some View {
        HStack(alignment: .top, spacing: 8) {
            if !message.role.isUser {
                avatarView
                bubbleContent
                Spacer(minLength: 60)
            } else {
                Spacer(minLength: 60)
                bubbleContent
            }
        }
    }

    private var avatarView: some View {
        Image(systemName: "sparkles")
            .font(.title3)
            .foregroundStyle(.white)
            .frame(width: 32, height: 32)
            .background(Color.purple.gradient)
            .clipShape(Circle())
    }

    private var bubbleContent: some View {
        VStack(alignment: .leading, spacing: 4) {
            if message.status.isStreaming && message.content.isEmpty {
                TypingIndicatorView()
            } else {
                MessageContentView(message: message)
            }

            if case .error(let errorMessage) = message.status {
                HStack(spacing: 4) {
                    Image(systemName: "exclamationmark.circle")
                    Text(errorMessage)
                }
                .font(.caption)
                .foregroundStyle(.red)
            }
        }
        .padding(.horizontal, 14)
        .padding(.vertical, 10)
        .background(message.role.isUser ? Color.accentColor : Color(.systemGray6))
        .clipShape(RoundedRectangle(cornerRadius: 18, style: .continuous))
    }
}
```

### 3.4 消息内容视图

```swift
struct MessageContentView: View {
    let message: Message

    var body: some View {
        if message.role.isUser {
            Text(message.content)
                .foregroundStyle(.white)
                .font(.body)
                .textSelection(.enabled)
        } else {
            MarkdownContentView(text: message.content)
        }
    }
}
```

### 3.5 输入区域

```swift
extension ChatView {
    var inputBarView: some View {
        HStack(alignment: .bottom, spacing: 8) {
            attachmentButton

            TextField("输入消息...", text: $inputText, axis: .vertical)
                .textFieldStyle(.plain)
                .lineLimit(1...6)
                .padding(.horizontal, 12)
                .padding(.vertical, 8)
                .background(Color(.systemGray6))
                .clipShape(RoundedRectangle(cornerRadius: 20))
                .onSubmit {
                    sendMessage()
                }

            sendButton
        }
        .padding(.horizontal, 12)
        .padding(.vertical, 8)
        .background(.bar)
    }

    private var attachmentButton: some View {
        Button {
            viewModel.showImagePicker = true
        } label: {
            Image(systemName: "plus.circle.fill")
                .font(.title2)
                .foregroundStyle(.secondary)
        }
    }

    private var sendButton: some View {
        Button {
            sendMessage()
        } label: {
            Image(systemName: "arrow.up.circle.fill")
                .font(.title2)
                .foregroundStyle(
                    canSend ? Color.accentColor : Color(.systemGray3)
                )
        }
        .disabled(!canSend)
    }

    private var canSend: Bool {
        !inputText.trimmingCharacters(in: .whitespacesAndNewlines).isEmpty
            && !viewModel.isGenerating
    }

    private func sendMessage() {
        guard canSend else { return }
        let text = inputText.trimmingCharacters(in: .whitespacesAndNewlines)
        inputText = ""
        viewModel.send(text)
    }
}
```

### 3.6 键盘适配

iOS 17+ 提供了 `scrollDismissesKeyboard` 和键盘避让的内置支持：

```swift
ScrollView {
    LazyVStack(spacing: 12) {
        ForEach(viewModel.messages) { message in
            MessageBubbleView(message: message)
                .id(message.id)
        }
    }
    .padding()
}
.scrollDismissesKeyboard(.interactively)
```

对于需要更精细控制的场景，可以使用键盘高度观察器：

```swift
struct KeyboardObserver: ObservableObject {
    @Published var keyboardHeight: CGFloat = 0

    init() {
        NotificationCenter.default.addObserver(
            forName: UIResponder.keyboardWillShowNotification,
            object: nil,
            queue: .main
        ) { notification in
            let frame = notification.userInfo?[UIResponder.keyboardFrameEndUserInfoKey] as? CGRect ?? .zero
            withAnimation(.easeOut(duration: 0.25)) {
                self.keyboardHeight = frame.height
            }
        }

        NotificationCenter.default.addObserver(
            forName: UIResponder.keyboardWillHideNotification,
            object: nil,
            queue: .main
        ) { _ in
            withAnimation(.easeOut(duration: 0.25)) {
                self.keyboardHeight = 0
            }
        }
    }
}
```

---

## 4. 流式输出渲染

### 4.1 逐字显示 AI 回复的原理

流式输出（Streaming）是 AI 对话应用的核心体验。其工作原理如下：

```
传统请求：  用户发送 → [等待...] → 完整回复一次性返回
流式请求：  用户发送 → token₁ → token₂ → token₃ → ... → [DONE]
```

流式输出基于 Server-Sent Events (SSE) 协议，服务端逐个推送 token：

```
data: {"choices":[{"delta":{"content":"你"},"index":0}]}
data: {"choices":[{"delta":{"content":"好"},"index":0}]}
data: {"choices":[{"delta":{"content":"！"},"index":0}]}
data: [DONE]
```

### 4.2 结合 StreamingLLMService

假设上一章已实现 `StreamingLLMService`，我们将其集成到 ViewModel 中：

```swift
@MainActor
class ChatViewModel: ObservableObject {
    @Published var messages: [Message] = []
    @Published var isGenerating = false
    @Published var showImagePicker = false

    private let streamingService: StreamingLLMService
    private var currentStreamingTask: Task<Void, Never>?

    var lastMessageContent: String {
        messages.last?.content ?? ""
    }

    init(streamingService: StreamingLLMService = StreamingLLMService()) {
        self.streamingService = streamingService
    }

    func send(_ text: String, images: [MessageImage]? = nil) {
        let userMessage = Message(
            role: .user,
            content: text,
            images: images
        )
        messages.append(userMessage)

        let assistantMessage = Message(
            role: .assistant,
            content: "",
            status: .streaming
        )
        messages.append(assistantMessage)

        let assistantIndex = messages.count - 1
        isGenerating = true

        currentStreamingTask = Task {
            do {
                let contextMessages = buildContextMessages()
                for try await chunk in streamingService.stream(messages: contextMessages) {
                    guard !Task.isCancelled else { break }
                    messages[assistantIndex].content += chunk
                }
                messages[assistantIndex].status = .sent
            } catch {
                messages[assistantIndex].status = .error(error.localizedDescription)
            }
            isGenerating = false
        }
    }

    func stopGenerating() {
        currentStreamingTask?.cancel()
        currentStreamingTask = nil
        if let lastIndex = messages.indices.last,
           messages[lastIndex].status.isStreaming {
            messages[lastIndex].status = .sent
        }
        isGenerating = false
    }
}
```

### 4.3 流式消息的状态管理

流式消息的状态流转：

```
用户发送消息
    ↓
创建 assistant 消息（content = "", status = .streaming）
    ↓
UI 显示打字指示器（● ● ●）
    ↓
收到第一个 token → content 逐步增长
    ↓
UI 逐字渲染 content
    ↓
流式结束 → status = .sent
    或
流式出错 → status = .error(...)
```

> 💡 **提示**：流式输出时，每次 `content += chunk` 都会触发 SwiftUI 的视图更新。由于 SwiftUI 的 diff 机制，只有变化的部分会重新渲染，性能通常可以接受。但如果 AI 回复非常长（超过 5000 字），可能需要考虑节流更新。

### 4.4 打字指示器

```swift
struct TypingIndicatorView: View {
    @State private var isAnimating = false

    var body: some View {
        HStack(spacing: 4) {
            ForEach(0..<3, id: \.self) { index in
                Circle()
                    .fill(Color.secondary)
                    .frame(width: 8, height: 8)
                    .offset(y: isAnimating ? -4 : 4)
                    .animation(
                        .easeInOut(duration: 0.4)
                            .repeatForever(autoreverses: true)
                            .delay(Double(index) * 0.15),
                        value: isAnimating
                    )
            }
        }
        .frame(height: 20)
        .onAppear { isAnimating = true }
    }
}
```

### 4.5 流式输出中断处理

用户可能在中途停止 AI 生成，需要优雅地处理中断：

```swift
extension ChatViewModel {
    func stopGenerating() {
        currentStreamingTask?.cancel()
        currentStreamingTask = nil

        if let lastIndex = messages.indices.last,
           messages[lastIndex].status.isStreaming {
            if messages[lastIndex].content.isEmpty {
                messages[lastIndex].content = "（生成已中断）"
            }
            messages[lastIndex].status = .sent
        }
        isGenerating = false
    }

    func retryLastMessage() {
        guard let lastIndex = messages.indices.last,
              messages[lastIndex].status.isError else { return }

        messages[lastIndex].content = ""
        messages[lastIndex].status = .streaming
        isGenerating = true

        currentStreamingTask = Task {
            do {
                let contextMessages = buildContextMessages()
                for try await chunk in streamingService.stream(messages: contextMessages) {
                    guard !Task.isCancelled else { break }
                    messages[lastIndex].content += chunk
                }
                messages[lastIndex].status = .sent
            } catch {
                messages[lastIndex].status = .error(error.localizedDescription)
            }
            isGenerating = false
        }
    }
}
```

⚠️ **警告**：中断流式输出时，务必将消息状态从 `.streaming` 改为 `.sent`，否则 UI 会一直显示打字指示器或光标。同时要检查 `content` 是否为空，避免显示空白气泡。

---

## 5. 对话上下文管理

### 5.1 上下文窗口的概念

大模型有**上下文窗口**限制，即一次请求能处理的最大 token 数：

| 模型 | 上下文窗口 | 约等于中文字数 |
|------|-----------|-------------|
| GPT-4o | 128K tokens | ~6 万字 |
| GPT-4o-mini | 128K tokens | ~6 万字 |
| Claude 3.5 Sonnet | 200K tokens | ~10 万字 |
| DeepSeek-V3 | 128K tokens | ~6 万字 |
| Qwen-Max | 32K tokens | ~1.5 万字 |

上下文窗口 = System Prompt + 历史消息 + 用户最新消息 + AI 回复

当对话越来越长，历史消息可能超出上下文窗口，需要截断。

### 5.2 历史消息截断策略

```swift
extension ChatViewModel {
    func buildContextMessages() -> [ContextMessage] {
        var context: [ContextMessage] = []

        if let systemPrompt = conversation.systemPrompt {
            context.append(ContextMessage(role: .system, content: systemPrompt))
        }

        let nonSystemMessages = messages.filter { $0.role != .system }

        let recentMessages = Array(nonSystemMessages.suffix(maxContextMessages))

        for message in recentMessages {
            context.append(ContextMessage(
                role: message.role,
                content: message.content
            ))
        }

        return context
    }

    private var maxContextMessages: Int {
        20
    }
}
```

截断策略对比：

| 策略 | 实现方式 | 优点 | 缺点 |
|------|---------|------|------|
| **保留最近 N 条** | `suffix(N)` | 简单可靠 | 可能丢失重要上下文 |
| **Token 预算** | 计算总 token 数，不超过阈值 | 精确控制 | 需要本地 token 计数器 |
| **摘要压缩** | 用 AI 生成历史摘要 | 保留语义 | 额外 API 调用成本 |
| **滑动窗口 + 摘要** | 近 N 条原文 + 更早的摘要 | 平衡效果与成本 | 实现复杂 |

### 5.3 Token 预算截断

更精确的截断方式是基于 Token 预算：

```swift
extension ChatViewModel {
    func buildContextMessages(tokenBudget: Int = 4000) -> [ContextMessage] {
        var context: [ContextMessage] = []
        var usedTokens = 0

        if let systemPrompt = conversation.systemPrompt {
            let tokens = estimateTokenCount(systemPrompt)
            context.append(ContextMessage(role: .system, content: systemPrompt))
            usedTokens += tokens
        }

        let nonSystemMessages = messages.filter { $0.role != .system }

        var selectedMessages: [Message] = []
        for message in nonSystemMessages.reversed() {
            let tokens = estimateTokenCount(message.content)
            if usedTokens + tokens > tokenBudget { break }
            selectedMessages.insert(message, at: 0)
            usedTokens += tokens
        }

        for message in selectedMessages {
            context.append(ContextMessage(
                role: message.role,
                content: message.content
            ))
        }

        return context
    }

    private func estimateTokenCount(_ text: String) -> Int {
        max(1, text.count / 2)
    }
}
```

> 💡 **提示**：`text.count / 2` 是一个粗略的中文 token 估算。更精确的做法是使用 `tiktoken` 的 Swift 移植版，或者调用 API 时由服务端返回 token 用量。对于大多数场景，粗略估算已经足够。

### 5.4 System Prompt 管理

```swift
struct SystemPromptTemplate {
    static let defaultPrompt = """
    你是一个有用的 AI 助手。请用简洁、准确的方式回答用户的问题。
    """

    static let codingAssistant = """
    你是一个 iOS 开发专家，精通 Swift、SwiftUI、UIKit。
    回答时请提供完整的代码示例，并解释关键实现思路。
    代码使用 markdown 代码块包裹，并标注语言类型。
    """

    static let creativeWriter = """
    你是一个创意写作助手。请用生动、有趣的语言进行创作。
    善用比喻和修辞，让文字更有感染力。
    """
}

extension Conversation {
    mutating func setSystemPrompt(_ prompt: String) {
        systemPrompt = prompt
    }
}
```

### 5.5 上下文压缩技巧

当对话过长时，可以使用 AI 自身来压缩上下文：

```swift
extension ChatViewModel {
    func compressHistory() async -> String? {
        let oldMessages = messages.filter { $0.role != .system }
        guard oldMessages.count > 10 else { return nil }

        let summaryPrompt = """
        请将以下对话历史压缩为一段简洁的摘要，保留关键信息和上下文：

        \(oldMessages.dropLast(4).map { "\($0.role.rawValue): \($0.content)" }.joined(separator: "\n"))
        """

        do {
            let summary = try await streamingService.complete(prompt: summaryPrompt)
            return summary
        } catch {
            return nil
        }
    }
}
```

### 5.6 多轮对话的 Token 计算

```swift
struct TokenCalculator {
    struct TokenUsage {
        let promptTokens: Int
        let completionTokens: Int
        let totalTokens: Int
    }

    static func estimateUsage(
        messages: [Message],
        systemPrompt: String?
    ) -> TokenUsage {
        var promptTokens = 0

        if let systemPrompt {
            promptTokens += estimateTokenCount(systemPrompt) + 4
        }

        for message in messages {
            promptTokens += estimateTokenCount(message.content) + 4
        }

        promptTokens += 2

        return TokenUsage(
            promptTokens: promptTokens,
            completionTokens: 0,
            totalTokens: promptTokens
        )
    }

    static func estimateTokenCount(_ text: String) -> Int {
        var count = 0
        var charBuffer = ""

        for char in text {
            charBuffer.append(char)
            if charBuffer.count >= 2 {
                count += 1
                charBuffer = ""
            }
        }

        if !charBuffer.isEmpty {
            count += 1
        }

        return max(1, count)
    }
}
```

---

## 6. Markdown 渲染

### 6.1 AI 回复中的 Markdown 格式

AI 回复通常包含丰富的 Markdown 格式：

| Markdown 元素 | 示例 | 渲染要求 |
|-------------|------|---------|
| 代码块 | ` ```swift ... ``` ` | 深色背景 + 语法高亮 + 复制按钮 |
| 行内代码 | `` `code` `` | 灰色背景 + 等宽字体 |
| 粗体 | `**粗体**` | 加粗字体 |
| 斜体 | `*斜体*` | 斜体字体 |
| 列表 | `- 项目` | 缩进 + 符号 |
| 有序列表 | `1. 项目` | 缩进 + 数字 |
| 标题 | `## 标题` | 大号字体 |
| 链接 | `[文字](url)` | 可点击蓝色文字 |
| 表格 | `\| 列1 \| 列2 \|` | 表格视图 |
| 引用 | `> 引用` | 左侧竖线 + 缩进 |

### 6.2 SwiftUI Markdown 渲染方案对比

| 方案 | 优点 | 缺点 | 推荐度 |
|------|------|------|-------|
| **AttributedString** | 系统原生，无依赖 | 仅支持基础格式，不支持代码块 | ⭐⭐ |
| **MarkdownUI 库** | 功能全面，社区活跃 | 第三方依赖 | ⭐⭐⭐⭐⭐ |
| **自定义解析** | 完全可控 | 开发成本高，容易出 bug | ⭐⭐ |
| **WKWebView** | 渲染效果最好 | 内存占用高，交互复杂 | ⭐⭐⭐ |

> 💡 **提示**：推荐使用 [MarkdownUI](https://github.com/gonzalezreal/swift-markdown-ui) 库，它是 SwiftUI 生态中最成熟的 Markdown 渲染方案，支持代码高亮、自定义主题等。

### 6.3 使用 MarkdownUI 库

首先在 Package.swift 中添加依赖：

```swift
.package(url: "https://github.com/gonzalezreal/swift-markdown-ui", from: "2.0.0")
```

基础使用：

```swift
import MarkdownUI

struct MarkdownContentView: View {
    let text: String

    var body: some View {
        Markdown(text)
            .markdownTheme(.chatTheme)
            .textSelection(.enabled)
    }
}

extension Theme {
    static let chatTheme = Theme()
        .code {
            FontFamilyVariant(.monospaced)
            FontSize(.em(0.85))
            BackgroundColor(Color(.systemGray6))
        }
        .link {
            ForegroundColor(.accentColor)
        }
        .heading1 { configuration in
            configuration.label
                .markdownMargin(top: 16, bottom: 8)
                .markdownFontSize(.em(1.4))
                .fontWeight(.bold)
        }
        .heading2 { configuration in
            configuration.label
                .markdownMargin(top: 12, bottom: 6)
                .markdownFontSize(.em(1.2))
                .fontWeight(.semibold)
        }
}
```

### 6.4 代码高亮实现

MarkdownUI 支持通过自定义代码块主题实现语法高亮：

```swift
import MarkdownUI
import Splash

extension Theme {
    static let chatTheme = Theme()
        .code {
            FontFamilyVariant(.monospaced)
            FontSize(.em(0.85))
        }
        .codeBlock { configuration in
            CodeBlockView(configuration: configuration)
        }
}

struct CodeBlockView: View {
    let configuration: CodeBlockConfiguration

    var body: some View {
        VStack(alignment: .trailing, spacing: 0) {
            HStack {
                Text(configuration.language ?? "code")
                    .font(.caption2)
                    .foregroundStyle(.secondary)
                Spacer()
                CopyButton(text: configuration.content)
            }
            .padding(.horizontal, 12)
            .padding(.vertical, 6)
            .background(Color(.systemGray5))

            ScrollView(.horizontal, showsIndicators: false) {
                highlightedCode
                    .padding(12)
            }
        }
        .background(Color(.systemGray6))
        .clipShape(RoundedRectangle(cornerRadius: 8))
    }

    @ViewBuilder
    private var highlightedCode: some View {
        if let language = configuration.language,
           let highlighted = try? AttributedString(
               markdown: configuration.content,
               options: .init(interpretedSyntax: .inlineOnlyPreservingWhitespace)
           ) {
            Text(highlighted)
                .font(.system(.caption, design: .monospaced))
        } else {
            Text(configuration.content)
                .font(.system(.caption, design: .monospaced))
        }
    }
}
```

### 6.5 代码块复制功能

```swift
struct CopyButton: View {
    let text: String
    @State private var showCopied = false

    var body: some View {
        Button {
            UIPasteboard.general.string = text
            withAnimation {
                showCopied = true
            }
            DispatchQueue.main.asyncAfter(deadline: .now() + 2) {
                withAnimation {
                    showCopied = false
                }
            }
        } label: {
            HStack(spacing: 4) {
                Image(systemName: showCopied ? "checkmark" : "doc.on.doc")
                Text(showCopied ? "已复制" : "复制")
            }
            .font(.caption2)
            .foregroundStyle(showCopied ? .green : .secondary)
        }
    }
}
```

---

## 7. 多模态输入

### 7.1 图片上传 + Vision API

多模态大模型（如 GPT-4o、Claude 3.5）支持图片输入，让 AI "看懂"图片内容：

```swift
struct VisionRequest {
    let text: String
    let images: [MessageImage]

    func toAPIFormat() -> [[String: Any]] {
        var content: [[String: Any]] = []

        content.append(["type": "text", "text": text])

        for image in images {
            content.append([
                "type": "image_url",
                "image_url": [
                    "url": "data:\(image.mimeType);base64,\(image.base64Data)",
                    "detail": "auto"
                ]
            ])
        }

        return content
    }
}
```

### 7.2 相机/相册选择器

```swift
struct ImagePickerView: UIViewControllerRepresentable {
    @Binding var selectedImages: [MessageImage]
    @Environment(\.dismiss) private var dismiss

    func makeUIViewController(context: Context) -> PHPickerViewController {
        var config = PHPickerConfiguration()
        config.selectionLimit = 3
        config.filter = .images

        let picker = PHPickerViewController(configuration: config)
        picker.delegate = context.coordinator
        return picker
    }

    func updateUIViewController(
        _ uiViewController: PHPickerViewController,
        context: Context
    ) {}

    func makeCoordinator() -> Coordinator {
        Coordinator(selectedImages: $selectedImages, dismiss: dismiss)
    }

    class Coordinator: NSObject, PHPickerViewControllerDelegate {
        @Binding var selectedImages: [MessageImage]
        let dismiss: DismissAction

        init(selectedImages: Binding<[MessageImage]>, dismiss: DismissAction) {
            self._selectedImages = selectedImages
            self.dismiss = dismiss
        }

        func picker(
            _ picker: PHPickerViewController,
            didFinishPicking results: [PHPickerResult]
        ) {
            dismiss()

            for result in results {
                let provider = result.itemProvider
                if provider.canLoadObject(ofClass: UIImage.self) {
                    provider.loadObject(ofClass: UIImage.self) { image, error in
                        guard let uiImage = image as? UIImage,
                              let data = uiImage.jpegData(compressionQuality: 0.7) else { return }

                        let messageImage = MessageImage(
                            base64Data: data.base64EncodedString(),
                            mimeType: "image/jpeg",
                            width: uiImage.size.width,
                            height: uiImage.size.height
                        )

                        DispatchQueue.main.async {
                            self.selectedImages.append(messageImage)
                        }
                    }
                }
            }
        }
    }
}
```

### 7.3 图片压缩与 Base64 编码

```swift
struct ImageCompressor {
    static func compress(
        _ image: UIImage,
        maxWidth: CGFloat = 1024,
        quality: CGFloat = 0.7
    ) -> Data? {
        let scale = min(1.0, maxWidth / max(image.size.width, image.size.height))
        let newSize = CGSize(
            width: image.size.width * scale,
            height: image.size.height * scale
        )

        UIGraphicsBeginImageContextWithOptions(newSize, false, 1.0)
        image.draw(in: CGRect(origin: .zero, size: newSize))
        let resizedImage = UIGraphicsGetImageFromCurrentImageContext()
        UIGraphicsEndImageContext()

        return resizedImage?.jpegData(compressionQuality: quality)
    }

    static func toBase64(_ data: Data) -> String {
        data.base64EncodedString()
    }

    static func estimateImageTokens(width: CGFloat, height: CGFloat, detail: String = "auto") -> Int {
        switch detail {
        case "low":
            return 85
        case "high":
            let tiles = ceil(width / 512) * ceil(height / 512)
            return Int(tiles) * 170 + 85
        default:
            return width < 512 && height < 512 ? 85 : Int(ceil(width / 512) * ceil(height / 512)) * 170 + 85
        }
    }
}
```

⚠️ **警告**：Base64 编码会使图片体积增大约 33%。一张 1MB 的图片编码后约 1.33MB，会显著增加 API 请求体大小。务必在上传前压缩图片，建议最大宽度 1024px，JPEG 质量 0.7。

### 7.4 多模态消息的数据模型扩展

在 ChatViewModel 中支持图片发送：

```swift
extension ChatViewModel {
    func send(_ text: String, images: [MessageImage]? = nil) {
        let userMessage = Message(
            role: .user,
            content: text,
            images: images
        )
        messages.append(userMessage)

        let assistantMessage = Message(
            role: .assistant,
            content: "",
            status: .streaming
        )
        messages.append(assistantMessage)

        let assistantIndex = messages.count - 1
        isGenerating = true

        currentStreamingTask = Task {
            do {
                let contextMessages = buildContextMessages()
                let stream: AsyncThrowingStream<String, Error>

                if let images, !images.isEmpty {
                    stream = streamingService.streamWithImages(
                        messages: contextMessages,
                        images: images
                    )
                } else {
                    stream = streamingService.stream(messages: contextMessages)
                }

                for try await chunk in stream {
                    guard !Task.isCancelled else { break }
                    messages[assistantIndex].content += chunk
                }
                messages[assistantIndex].status = .sent
            } catch {
                messages[assistantIndex].status = .error(error.localizedDescription)
            }
            isGenerating = false
        }
    }
}
```

在输入区域显示已选择的图片预览：

```swift
struct ImagePreviewBar: View {
    let images: [MessageImage]
    let onRemove: (Int) -> Void

    var body: some View {
        ScrollView(.horizontal, showsIndicators: false) {
            HStack(spacing: 8) {
                ForEach(Array(images.enumerated()), id: \.offset) { index, image in
                    if let data = Data(base64Encoded: image.base64Data),
                       let uiImage = UIImage(data: data) {
                        ZStack(alignment: .topTrailing) {
                            Image(uiImage: uiImage)
                                .resizable()
                                .aspectRatio(contentMode: .fill)
                                .frame(width: 60, height: 60)
                                .clipShape(RoundedRectangle(cornerRadius: 8))

                            Button {
                                onRemove(index)
                            } label: {
                                Image(systemName: "xmark.circle.fill")
                                    .font(.caption)
                                    .foregroundStyle(.white, .red)
                            }
                            .offset(x: 4, y: -4)
                        }
                    }
                }
            }
            .padding(.horizontal, 4)
        }
    }
}
```

---

## 8. 完整代码示例

### 8.1 ViewModel 完整实现

```swift
import SwiftUI

struct ContextMessage {
    let role: MessageRole
    let content: String
}

@MainActor
class ChatViewModel: ObservableObject {
    @Published var conversation = Conversation()
    @Published var isGenerating = false
    @Published var showImagePicker = false
    @Published var selectedImages: [MessageImage] = []

    private let streamingService: StreamingLLMService
    private var currentStreamingTask: Task<Void, Never>?

    var messages: [Message] {
        conversation.messages
    }

    var lastMessageContent: String {
        conversation.messages.last?.content ?? ""
    }

    init(streamingService: StreamingLLMService = StreamingLLMService()) {
        self.streamingService = streamingService
    }

    func send(_ text: String, images: [MessageImage]? = nil) {
        let userMessage = Message(
            role: .user,
            content: text,
            images: images
        )
        conversation.messages.append(userMessage)
        conversation.updatedAt = Date()

        let assistantMessage = Message(
            role: .assistant,
            content: "",
            status: .streaming
        )
        conversation.messages.append(assistantMessage)

        let assistantIndex = conversation.messages.count - 1
        isGenerating = true

        currentStreamingTask = Task {
            do {
                let contextMessages = buildContextMessages()

                let stream: AsyncThrowingStream<String, Error>
                if let images, !images.isEmpty {
                    stream = streamingService.streamWithImages(
                        messages: contextMessages,
                        images: images
                    )
                } else {
                    stream = streamingService.stream(messages: contextMessages)
                }

                for try await chunk in stream {
                    guard !Task.isCancelled else { break }
                    conversation.messages[assistantIndex].content += chunk
                }
                conversation.messages[assistantIndex].status = .sent
                updateConversationTitle()
            } catch {
                conversation.messages[assistantIndex].status = .error(
                    error.localizedDescription
                )
            }
            isGenerating = false
        }
    }

    func stopGenerating() {
        currentStreamingTask?.cancel()
        currentStreamingTask = nil

        if let lastIndex = conversation.messages.indices.last,
           conversation.messages[lastIndex].status.isStreaming {
            if conversation.messages[lastIndex].content.isEmpty {
                conversation.messages[lastIndex].content = "（生成已中断）"
            }
            conversation.messages[lastIndex].status = .sent
        }
        isGenerating = false
    }

    func retryLastMessage() {
        guard let lastIndex = conversation.messages.indices.last,
              conversation.messages[lastIndex].status.isError else { return }

        conversation.messages[lastIndex].content = ""
        conversation.messages[lastIndex].status = .streaming
        isGenerating = true

        let retryIndex = lastIndex

        currentStreamingTask = Task {
            do {
                let contextMessages = buildContextMessages()
                for try await chunk in streamingService.stream(messages: contextMessages) {
                    guard !Task.isCancelled else { break }
                    conversation.messages[retryIndex].content += chunk
                }
                conversation.messages[retryIndex].status = .sent
            } catch {
                conversation.messages[retryIndex].status = .error(
                    error.localizedDescription
                )
            }
            isGenerating = false
        }
    }

    func startNewConversation() {
        stopGenerating()
        conversation = Conversation(systemPrompt: SystemPromptTemplate.defaultPrompt)
        selectedImages = []
    }

    func removeImage(at index: Int) {
        guard selectedImages.indices.contains(index) else { return }
        selectedImages.remove(at: index)
    }

    private func buildContextMessages(tokenBudget: Int = 4000) -> [ContextMessage] {
        var context: [ContextMessage] = []
        var usedTokens = 0

        if let systemPrompt = conversation.systemPrompt {
            let tokens = estimateTokenCount(systemPrompt)
            context.append(ContextMessage(role: .system, content: systemPrompt))
            usedTokens += tokens
        }

        let nonSystemMessages = conversation.messages.filter { $0.role != .system }

        var selectedMessages: [Message] = []
        for message in nonSystemMessages.reversed() {
            let tokens = estimateTokenCount(message.content)
            if usedTokens + tokens > tokenBudget { break }
            selectedMessages.insert(message, at: 0)
            usedTokens += tokens
        }

        for message in selectedMessages {
            context.append(ContextMessage(
                role: message.role,
                content: message.content
            ))
        }

        return context
    }

    private func updateConversationTitle() {
        guard conversation.title == "新对话",
              let firstUserMessage = conversation.messages.first(where: { $0.role == .user }) else { return }
        let title = String(firstUserMessage.content.prefix(20))
        conversation.title = title
    }

    private func estimateTokenCount(_ text: String) -> Int {
        max(1, text.count / 2)
    }
}
```

### 8.2 完整 ChatView

```swift
struct ChatView: View {
    @StateObject private var viewModel = ChatViewModel()
    @State private var inputText = ""

    var body: some View {
        VStack(spacing: 0) {
            messageListView
            Divider()
            imagePreviewBar
            inputBarView
        }
        .navigationTitle(viewModel.conversation.title)
        .navigationBarTitleDisplayMode(.inline)
        .toolbar {
            ToolbarItem(placement: .topBarTrailing) {
                Button {
                    viewModel.startNewConversation()
                } label: {
                    Image(systemName: "plus.bubble")
                }
            }
        }
        .sheet(isPresented: $viewModel.showImagePicker) {
            ImagePickerView(selectedImages: $viewModel.selectedImages)
        }
    }

    private var messageListView: some View {
        ScrollViewReader { proxy in
            ScrollView {
                LazyVStack(spacing: 12) {
                    ForEach(viewModel.messages.filter { $0.role != .system }) { message in
                        MessageBubbleView(message: message)
                            .id(message.id)
                    }
                }
                .padding()
            }
            .scrollDismissesKeyboard(.interactively)
            .onChange(of: viewModel.messages.count) { _ in
                scrollToBottom(proxy: proxy)
            }
            .onChange(of: viewModel.lastMessageContent) { _ in
                scrollToBottom(proxy: proxy)
            }
        }
    }

    private func scrollToBottom(proxy: ScrollViewProxy) {
        guard let lastMessage = viewModel.messages.last else { return }
        withAnimation(.easeOut(duration: 0.2)) {
            proxy.scrollTo(lastMessage.id, anchor: .bottom)
        }
    }

    private var imagePreviewBar: some View {
        Group {
            if !viewModel.selectedImages.isEmpty {
                ImagePreviewBar(
                    images: viewModel.selectedImages,
                    onRemove: { index in
                        viewModel.removeImage(at: index)
                    }
                )
                .padding(.horizontal, 12)
                .padding(.vertical, 4)
            }
        }
    }

    private var inputBarView: some View {
        HStack(alignment: .bottom, spacing: 8) {
            Button {
                viewModel.showImagePicker = true
            } label: {
                Image(systemName: "plus.circle.fill")
                    .font(.title2)
                    .foregroundStyle(.secondary)
            }

            TextField("输入消息...", text: $inputText, axis: .vertical)
                .textFieldStyle(.plain)
                .lineLimit(1...6)
                .padding(.horizontal, 12)
                .padding(.vertical, 8)
                .background(Color(.systemGray6))
                .clipShape(RoundedRectangle(cornerRadius: 20))
                .onSubmit { sendMessage() }

            if viewModel.isGenerating {
                Button {
                    viewModel.stopGenerating()
                } label: {
                    Image(systemName: "stop.circle.fill")
                        .font(.title2)
                        .foregroundStyle(.red)
                }
            } else {
                Button {
                    sendMessage()
                } label: {
                    Image(systemName: "arrow.up.circle.fill")
                        .font(.title2)
                        .foregroundStyle(canSend ? Color.accentColor : Color(.systemGray3))
                }
                .disabled(!canSend)
            }
        }
        .padding(.horizontal, 12)
        .padding(.vertical, 8)
        .background(.bar)
    }

    private var canSend: Bool {
        !inputText.trimmingCharacters(in: .whitespacesAndNewlines).isEmpty
            || !viewModel.selectedImages.isEmpty
    }

    private func sendMessage() {
        guard canSend else { return }
        let text = inputText.trimmingCharacters(in: .whitespacesAndNewlines)
        let images = viewModel.selectedImages.isEmpty ? nil : viewModel.selectedImages
        inputText = ""
        viewModel.selectedImages = []
        viewModel.send(text.isEmpty ? "请描述这张图片" : text, images: images)
    }
}
```

### 8.3 错误重试视图

```swift
struct RetryButton: View {
    let action: () -> Void

    var body: some View {
        Button(action: action) {
            HStack(spacing: 4) {
                Image(systemName: "arrow.clockwise")
                Text("重试")
            }
            .font(.caption)
            .padding(.horizontal, 12)
            .padding(.vertical, 6)
            .background(Color.accentColor.opacity(0.1))
            .foregroundStyle(.accentColor)
            .clipShape(Capsule())
        }
    }
}
```

---

## 9. 体验优化

### 9.1 消息发送的乐观更新

乐观更新（Optimistic Update）是指用户点击发送后，消息立即出现在列表中，不等 API 响应：

```swift
func send(_ text: String, images: [MessageImage]? = nil) {
    let userMessage = Message(
        role: .user,
        content: text,
        status: .sent,
        images: images
    )
    conversation.messages.append(userMessage)

    let assistantMessage = Message(
        role: .assistant,
        content: "",
        status: .streaming
    )
    conversation.messages.append(assistantMessage)

    isGenerating = true

    currentStreamingTask = Task {
        do {
            let contextMessages = buildContextMessages()
            let assistantIndex = conversation.messages.count - 1

            for try await chunk in streamingService.stream(messages: contextMessages) {
                guard !Task.isCancelled else { break }
                conversation.messages[assistantIndex].content += chunk
            }
            conversation.messages[assistantIndex].status = .sent
        } catch {
            let assistantIndex = conversation.messages.count - 1
            conversation.messages[assistantIndex].status = .error(
                error.localizedDescription
            )
        }
        isGenerating = false
    }
}
```

> 💡 **提示**：乐观更新的关键在于——用户消息的 `status` 直接设为 `.sent`，而不是 `.sending`。因为从用户视角来看，消息已经"发出"了，即使网络请求还没完成。如果发送失败，再回退状态。

### 9.2 长消息的懒加载

对于非常长的 AI 回复（如代码生成），可以使用懒加载来优化性能：

```swift
struct LazyMessageContent: View {
    let content: String
    let role: MessageRole

    var body: some View {
        if content.count > 5000 {
            LongMessageView(content: content, role: role)
        } else {
            MessageContentView(message: Message(role: role, content: content))
        }
    }
}

struct LongMessageView: View {
    let content: String
    let role: MessageRole
    @State private var isExpanded = false

    var body: some View {
        VStack(alignment: .leading, spacing: 8) {
            if isExpanded {
                MessageContentView(message: Message(role: role, content: content))
            } else {
                MessageContentView(
                    message: Message(role: role, content: String(content.prefix(2000)))
                )
                Button("展开全文") {
                    withAnimation { isExpanded = true }
                }
                .font(.caption)
                .foregroundStyle(.accentColor)
            }
        }
    }
}
```

### 9.3 消息搜索功能

```swift
extension ChatViewModel {
    func searchMessages(query: String) -> [Message] {
        guard !query.isEmpty else { return conversation.messages }
        return conversation.messages.filter { message in
            message.content.localizedCaseInsensitiveContains(query)
        }
    }
}

struct MessageSearchView: View {
    let messages: [Message]
    let query: String
    @State private var results: [Message] = []

    var body: some View {
        List(results) { message in
            VStack(alignment: .leading) {
                Text(message.role.rawValue)
                    .font(.caption)
                    .foregroundStyle(.secondary)
                Text(message.content.prefix(100))
                    .font(.subheadline)
            }
        }
        .searchable(text: .constant(query))
        .onChange(of: query) { newValue in
            results = messages.filter {
                $0.content.localizedCaseInsensitiveContains(newValue)
            }
        }
    }
}
```

### 9.4 对话历史持久化（SwiftData）

使用 SwiftData 将对话持久化到本地：

```swift
import SwiftData

@Model
class PersistentMessage {
    @Attribute(.unique) var id: UUID
    var role: String
    var content: String
    var timestamp: Date
    var status: String

    init(from message: Message) {
        self.id = message.id
        self.role = message.role.rawValue
        self.content = message.content
        self.timestamp = message.timestamp
        self.status = "sent"
    }

    func toMessage() -> Message {
        Message(
            id: id,
            role: MessageRole(rawValue: role) ?? .user,
            content: content,
            timestamp: timestamp,
            status: .sent
        )
    }
}

@Model
class PersistentConversation {
    @Attribute(.unique) var id: UUID
    var title: String
    var systemPrompt: String?
    var createdAt: Date
    var updatedAt: Date
    @Relationship(deleteRule: .cascade) var messages: [PersistentMessage] = []

    init(from conversation: Conversation) {
        self.id = conversation.id
        self.title = conversation.title
        self.systemPrompt = conversation.systemPrompt
        self.createdAt = conversation.createdAt
        self.updatedAt = conversation.updatedAt
        self.messages = conversation.messages.map { PersistentMessage(from: $0) }
    }

    func toConversation() -> Conversation {
        Conversation(
            id: id,
            title: title,
            messages: messages.map { $0.toMessage() },
            systemPrompt: systemPrompt,
            createdAt: createdAt,
            updatedAt: updatedAt
        )
    }
}
```

在 App 入口配置 SwiftData：

```swift
@main
struct AIChatApp: App {
    var body: some Scene {
        WindowGroup {
            ChatListView()
        }
        .modelContainer(for: PersistentConversation.self)
    }
}
```

对话列表视图：

```swift
struct ChatListView: View {
    @Query(sort: \PersistentConversation.updatedAt, order: .reverse)
    private var conversations: [PersistentConversation]
    @Environment(\.modelContext) private var modelContext

    var body: some View {
        NavigationStack {
            List(conversations) { conversation in
                NavigationLink {
                    ChatView(
                        viewModel: ChatViewModel(
                            conversation: conversation.toConversation()
                        )
                    )
                } label: {
                    VStack(alignment: .leading) {
                        Text(conversation.title)
                            .font(.headline)
                        Text(conversation.messages.last?.content.prefix(50) ?? "")
                            .font(.subheadline)
                            .foregroundStyle(.secondary)
                            .lineLimit(1)
                    }
                }
            }
            .navigationTitle("对话")
        }
    }
}
```

### 9.5 深色模式适配

SwiftUI 默认支持深色模式，但 AI 对话界面有一些需要特别注意的地方：

| 元素 | 浅色模式 | 深色模式 | 适配方式 |
|------|---------|---------|---------|
| 用户气泡 | 系统蓝色 | 深蓝色 | 使用 `Color.accentColor` |
| AI 气泡 | `systemGray6` | 自动变深 | 使用语义颜色 |
| AI 文字 | 黑色 | 白色 | 使用 `Color.primary` |
| 代码块背景 | `systemGray6` | 更深灰色 | 使用语义颜色 |
| 打字指示器 | `secondary` | 自动反转 | 使用语义颜色 |

```swift
extension Color {
    static let bubbleUser = Color.accentColor
    static let bubbleAI = Color(.systemGray6)
    static let textPrimary = Color.primary
    static let textOnAccent = Color.white
    static let codeBackground = Color(.systemGray6)
}
```

> 💡 **提示**：使用语义颜色（`systemGray6`、`primary`、`secondary`）而非硬编码颜色值，SwiftUI 会自动处理深色模式切换。避免使用 `.white` 和 `.black`，除非是在固定背景色上。

---

## 10. 无障碍与国际化

### 10.1 VoiceOver 支持

AI 对话界面的 VoiceOver 支持至关重要，因为视障用户同样需要使用 AI 助手：

```swift
struct MessageBubbleView: View {
    let message: Message

    var body: some View {
        HStack(alignment: .top, spacing: 8) {
            if !message.role.isUser {
                avatarView
                bubbleContent
                Spacer(minLength: 60)
            } else {
                Spacer(minLength: 60)
                bubbleContent
            }
        }
        .accessibilityElement(children: .combine)
        .accessibilityLabel(accessibilityLabel)
        .accessibilityHint(accessibilityHint)
        .accessibilityAddTraits(.isStaticText)
    }

    private var accessibilityLabel: String {
        let sender = message.role.isUser ? "你" : "AI 助手"
        return "\(sender)：\(message.content)"
    }

    private var accessibilityHint: String {
        if message.status.isStreaming {
            return "AI 正在回复中"
        }
        if case .error = message.status {
            return "发送失败，双击重试"
        }
        return nil
    }
}
```

打字指示器的无障碍支持：

```swift
struct TypingIndicatorView: View {
    @State private var isAnimating = false

    var body: some View {
        HStack(spacing: 4) {
            ForEach(0..<3, id: \.self) { index in
                Circle()
                    .fill(Color.secondary)
                    .frame(width: 8, height: 8)
                    .offset(y: isAnimating ? -4 : 4)
                    .animation(
                        .easeInOut(duration: 0.4)
                            .repeatForever(autoreverses: true)
                            .delay(Double(index) * 0.15),
                        value: isAnimating
                    )
            }
        }
        .frame(height: 20)
        .onAppear { isAnimating = true }
        .accessibilityLabel("AI 正在输入")
        .accessibilityHidden(false)
    }
}
```

### 10.2 动态字体支持

SwiftUI 默认支持动态字体，但需要确保布局能适应不同字号：

```swift
struct MessageBubbleView: View {
    let message: Message

    var body: some View {
        HStack(alignment: .top, spacing: 8) {
            if !message.role.isUser {
                avatarView
                bubbleContent
                Spacer(minLength: 60)
            } else {
                Spacer(minLength: 60)
                bubbleContent
            }
        }
    }

    private var bubbleContent: some View {
        VStack(alignment: .leading, spacing: 4) {
            if message.status.isStreaming && message.content.isEmpty {
                TypingIndicatorView()
            } else {
                MessageContentView(message: message)
            }
        }
        .padding(.horizontal, 14)
        .padding(.vertical, 10)
        .background(
            message.role.isUser ? Color.accentColor : Color(.systemGray6)
        )
        .clipShape(RoundedRectangle(cornerRadius: 18, style: .continuous))
        .frame(maxWidth: UIScreen.main.bounds.width * 0.8, alignment: .leading)
    }
}
```

动态字体适配要点：

| 要点 | 说明 |
|------|------|
| **使用系统字体** | `.font(.body)` 而非 `.font(.system(size: 16))` |
| **弹性间距** | 使用 `minLength` 而非固定间距 |
| **最大宽度限制** | 气泡最大宽度按比例设置，非固定像素 |
| **多行输入框** | `lineLimit(1...6)` 自动增高 |
| **测试** | 在设置中调整字体大小，验证布局 |

### 10.3 多语言消息处理

AI 对话天然是多语言的——用户可能用中文提问，AI 用英文回答。界面需要正确处理：

```swift
struct MessageContentView: View {
    let message: Message

    var body: some View {
        if message.role.isUser {
            Text(message.content)
                .foregroundStyle(.white)
                .font(.body)
                .multilineTextAlignment(.leading)
                .textSelection(.enabled)
        } else {
            MarkdownContentView(text: message.content)
        }
    }
}
```

多语言处理要点：

| 场景 | 处理方式 |
|------|---------|
| **中英混排** | SwiftUI 默认支持，无需额外处理 |
| **RTL 语言** | 使用 `.environment(\.layoutDirection, .rightToLeft)` |
| **日期格式** | 使用 `DateFormatter` 的 `locale` 属性 |
| **占位符** | 使用 `String(localized:)` 而非硬编码 |
| **输入法** | `TextField` 自动适配系统输入法 |

```swift
extension String {
    static let inputPlaceholder = String(localized: "输入消息...", bundle: .main)
    static let newConversation = String(localized: "新对话", bundle: .main)
    static let copyCode = String(localized: "复制", bundle: .main)
    static let copied = String(localized: "已复制", bundle: .main)
    static let retry = String(localized: "重试", bundle: .main)
    static let generatingInterrupted = String(localized: "（生成已中断）", bundle: .main)
    static let aiTyping = String(localized: "AI 正在输入", bundle: .main)
}
```

---

## 小结

本章详细介绍了如何在 iOS App 中构建一个完整的 AI 对话界面，核心要点如下：

| 主题 | 关键要点 |
|------|---------|
| **设计模式** | AI 居左、用户居右、底部输入框、流式输出——这是用户的心理模型 |
| **数据模型** | Message + Conversation + 状态枚举，清晰的状态流转是关键 |
| **SwiftUI 实现** | LazyVStack + ScrollViewReader 实现高性能消息列表 |
| **流式输出** | SSE 逐 token 推送，SwiftUI 自动响应 content 变化 |
| **上下文管理** | Token 预算截断是最实用的策略，保留最近 N 条 + System Prompt |
| **Markdown 渲染** | 推荐使用 MarkdownUI 库，代码块需要复制按钮 |
| **多模态输入** | 图片压缩 → Base64 编码 → Vision API，注意体积控制 |
| **体验优化** | 乐观更新、SwiftData 持久化、语义颜色深色模式适配 |
| **无障碍** | VoiceOver 标签、动态字体、多语言支持 |

构建 AI 对话界面看似简单，但要做到体验流畅、功能完善，需要关注大量细节。建议先实现核心功能（消息列表 + 输入框 + 流式输出），再逐步添加 Markdown 渲染、多模态输入、持久化等高级功能。

下一章我们将深入 RAG（检索增强生成）与知识库问答，让 AI 能够基于你的私有数据来回答问题。

---

← [iOS-App集成大模型API](./iOS-App集成大模型API.md) | [RAG与知识库问答](./RAG与知识库问答.md) →