# 附录 B：AI 工具速查表

> AI 编程工具是现代开发者的"外挂"，本附录汇总了主流 AI 工具的安装、配置和常用操作，方便随时查阅。

---

## Claude Code 速查

Claude Code 是 Anthropic 推出的命令行 AI 编程助手，可以直接在终端中与 Claude 对话，让它帮你写代码、调试、重构。

### 安装命令

| 步骤 | 命令 | 说明 |
|------|------|------|
| 1 | `node -v` | 确认 Node.js 18+ 已安装 |
| 2 | `npm install -g @anthropic-ai/claude-code` | 全局安装 Claude Code |
| 3 | `claude --version` | 验证安装成功 |
| 4 | `claude login` | 登录 Anthropic 账号 |

**如果 npm 安装慢**：

```bash
# 使用镜像源
npm install -g @anthropic-ai/claude-code --registry=https://registry.npmmirror.com
```

---

### 常用命令

| 命令 | 功能 | 示例 |
|------|------|------|
| `claude` | 启动交互式对话 | `claude` |
| `claude "问题"` | 直接提问 | `claude "如何在 SwiftUI 中创建列表"` |
| `claude -p "指令"` | 执行单条指令后退出 | `claude -p "修复 main.swift 中的错误"` |
| `/help` | 查看帮助信息 | 在对话中输入 |
| `/clear` | 清除对话历史 | 上下文太长时使用 |
| `/compact` | 压缩上下文 | 保留关键信息，减少 token 消耗 |
| `/cost` | 查看当前会话花费 | 查看 token 使用量 |
| `/model` | 切换模型 | 切换 Claude Sonnet / Opus |
| `/quit` | 退出 Claude Code | 结束当前会话 |
| `/init` | 初始化 CLAUDE.md | 在项目根目录生成配置文件 |
| `/review` | 代码审查 | 让 Claude 审查当前代码变更 |
| `/bug` | 报告 Bug | 向 Anthropic 反馈问题 |

---

### 快捷键

| 快捷键 | 功能 |
|--------|------|
| `Enter` | 发送消息 |
| `Shift + Enter` | 换行（不发送） |
| `↑` / `↓` | 浏览历史消息 |
| `Tab` | 自动补全 |
| `Ctrl + C` | 取消当前操作 |
| `Ctrl + D` | 退出 Claude Code |
| `Ctrl + L` | 清屏 |

---

### CLAUDE.md 模板

在项目根目录创建 `CLAUDE.md` 文件，Claude Code 每次启动时会自动读取：

```markdown
# 项目概述
这是一个 iOS 记账 App，使用 SwiftUI + SwiftData 开发。

# 技术栈
- 语言：Swift 6
- UI 框架：SwiftUI（iOS 17+）
- 数据持久化：SwiftData
- 网络请求：URLSession + async/await
- 架构模式：MVVM

# 代码规范
- 所有视图必须是 struct
- ViewModel 使用 @Observable 宏
- 网络请求必须使用 async/await，不要使用回调
- 错误处理使用 do-catch，不要使用 try!
- 注释使用 /// 格式

# 文件结构
- Views/ — 所有视图文件
- ViewModels/ — 所有 ViewModel 文件
- Models/ — 数据模型
- Services/ — 网络和数据服务
- Utilities/ — 工具类和扩展

# 命名规范
- 视图文件以 View 结尾：HomeView.swift
- ViewModel 文件以 ViewModel 结尾：HomeViewModel.swift
- 模型文件使用名词：Transaction.swift

# 禁止事项
- 不要使用 UIKit 组件（除非 SwiftUI 无法实现）
- 不要引入第三方依赖（除非我明确要求）
- 不要使用 print() 调试，使用 os.Logger
- 不要使用强制解包（!）
```

---

## OpenAI Codex 速查

OpenAI Codex 是 OpenAI 推出的命令行 AI 编程工具，基于 GPT 系列模型。

### 安装命令

| 步骤 | 命令 | 说明 |
|------|------|------|
| 1 | `node -v` | 确认 Node.js 22+ 已安装 |
| 2 | `npm install -g @openai/codex` | 全局安装 Codex CLI |
| 3 | `codex --version` | 验证安装成功 |
| 4 | 设置 API Key | `export OPENAI_API_KEY="sk-xxx"` |

**配置 API Key**：

```bash
# 临时设置（当前终端有效）
export OPENAI_API_KEY="sk-xxxxxxxxxxxxxxxx"

# 永久设置（写入配置文件）
echo 'export OPENAI_API_KEY="sk-xxxxxxxxxxxxxxxx"' >> ~/.zshrc
source ~/.zshrc
```

---

### 常用命令

| 命令 | 功能 | 示例 |
|------|------|------|
| `codex` | 启动交互模式 | `codex` |
| `codex "指令"` | 直接执行指令 | `codex "创建一个 SwiftUI 列表视图"` |
| `codex -q "问题"` | 快速问答模式 | `codex -q "SwiftUI 中 @State 的用法"` |
| `codex -a auto` | 自动批准模式 | AI 自动执行命令（慎用） |
| `codex -a suggest` | 建议模式（默认） | AI 提出建议，需确认后执行 |

---

### 最佳实践

1. **明确指定技术栈**：在指令中说明使用的框架和版本，如"使用 SwiftUI + iOS 17+ 开发"
2. **分步骤请求**：将复杂任务拆分为多个小步骤，逐步完成
3. **提供上下文**：引用相关文件让 AI 了解项目结构，如"参考 Models/Transaction.swift 的风格"
4. **验证输出**：始终在 Xcode 中编译验证 AI 生成的代码
5. **使用 CODEX.md**：类似 CLAUDE.md，在项目根目录创建 `CODEX.md` 提供项目上下文

---

## Cursor 速查

Cursor 是一款基于 VS Code 的 AI 编程 IDE，集成了 AI 代码补全、对话和编辑功能。

### 安装与配置

| 步骤 | 操作 | 说明 |
|------|------|------|
| 1 | 下载 Cursor | https://cursor.com |
| 2 | 安装并打开 | 拖入应用程序文件夹 |
| 3 | 导入 VS Code 配置 | 首次启动时可选择导入 | 
| 4 | 登录账号 | 注册/登录 Cursor 账号 |
| 5 | 配置模型 | Settings → Models → 选择默认模型 |

**推荐配置**：

| 配置项 | 推荐值 | 说明 |
|--------|--------|------|
| 默认模型 | Claude Sonnet 4 | 性价比最高 |
| 自动补全模型 | Cursor Tab | 代码补全专用 |
| Composer 模型 | Claude Sonnet 4 | 多文件编辑 |

---

### 常用快捷键

| 快捷键 | 功能 | 使用场景 |
|--------|------|---------|
| `⌘ + K` | 行内编辑 | 选中代码后，AI 帮你修改 |
| `⌘ + L` | 打开 Chat 面板 | 与 AI 对话 |
| `⌘ + I` | 打开 Composer | 多文件 AI 编辑 |
| `⌘ + Shift + K` | 生成代码 | 在光标处生成代码 |
| `Tab` | 接受补全建议 | AI 自动补全时按 Tab 接受 |
| `Esc` | 拒绝补全建议 | 不接受 AI 的建议 |
| `⌘ + .` | 触发 AI 建议 | 在代码中获取 AI 建议 |
| `⌘ + Shift + L` | 清除 Chat 历史 | 清空对话记录 |
| `⌘ + Enter` | 发送消息 | 在 Chat 中发送 |
| `⌘ + Shift + Enter` | 在 Chat 中换行 | 不发送消息，换行继续输入 |
| `@files` | 引用文件 | 在 Chat 中引用项目文件 |
| `@folders` | 引用文件夹 | 在 Chat 中引用整个文件夹 |
| `@web` | 搜索网页 | 在 Chat 中搜索网络信息 |
| `@docs` | 引用文档 | 引用外部文档内容 |
| `@git` | 引用 Git 信息 | 查看 Git 历史、差异等 |
| `Ctrl + ⌘ + →` | 下一个 AI 编辑点 | 在 Composer 中跳转 |
| `Ctrl + ⌘ + ←` | 上一个 AI 编辑点 | 在 Composer 中跳转 |

---

### Composer 模式

Composer 是 Cursor 最强大的功能，可以同时编辑多个文件。

| 模式 | 快捷键 | 说明 |
|------|--------|------|
| Normal | `⌘ + I` | AI 提出修改建议，你确认后应用 |
| Agent | `⌘ + I` 后切换 | AI 自动执行操作（读取文件、运行命令等） |

**Composer 使用技巧**：

1. **明确描述需求**：如"在 HomeView.swift 中添加一个搜索功能，同时在 HomeViewModel.swift 中添加搜索逻辑"
2. **引用文件**：使用 `@文件名` 引用相关文件
3. **逐步验证**：每次 Composer 修改后，先编译验证再继续
4. **使用 Agent 模式**：复杂任务让 AI 自动执行，但要注意审查结果

---

### .cursorrules 模板

在项目根目录创建 `.cursorrules` 文件：

```
# 项目：iOS 记账 App

## 技术栈
- Swift 6 + SwiftUI (iOS 17+)
- SwiftData 数据持久化
- URLSession + async/await 网络请求
- MVVM 架构

## 代码风格
- 缩进：4 空格
- 行宽：120 字符
- 视图必须是 struct
- ViewModel 使用 @Observable 宏
- 错误处理使用 Result 类型
- 注释使用 /// 格式

## 命名规范
- 视图：XxxView.swift
- ViewModel：XxxViewModel.swift
- 模型：名词.swift（如 Transaction.swift）
- 协议：XxxProtocol.swift
- 枚举：XxxType.swift 或 Xxx.swift

## 禁止事项
- 不要使用 UIKit（除非 SwiftUI 无法实现）
- 不要使用第三方库（除非我明确要求）
- 不要使用强制解包
- 不要使用 print()，使用 Logger
- 不要硬编码字符串，使用 Localizable.strings

## 测试要求
- 每个 ViewModel 必须有单元测试
- 测试文件放在 Tests/ 目录
- 测试命名：test_方法名_场景_预期结果
```

---

## Trae 速查

Trae 是字节跳动推出的 AI 编程 IDE，同样基于 VS Code，支持多种 AI 模型。

### 安装与配置

| 步骤 | 操作 | 说明 |
|------|------|------|
| 1 | 下载 Trae | https://trae.ai |
| 2 | 安装并打开 | 拖入应用程序文件夹 |
| 3 | 登录账号 | 注册/登录 Trae 账号 |
| 4 | 导入 VS Code 配置 | 首次启动时可选择导入 |
| 5 | 选择 AI 模型 | 设置中选择默认模型 |

---

### 核心功能

| 功能 | 说明 | 使用场景 |
|------|------|---------|
| AI Chat | 与 AI 对话 | 提问、讨论方案 |
| AI 编辑 | AI 修改代码 | 重构、修复 Bug |
| 代码补全 | AI 自动补全 | 写代码时实时建议 |
| Builder 模式 | 多文件编辑 | 复杂功能开发 |
| 上下文引用 | 引用文件/代码 | 让 AI 了解项目 |
| 终端集成 | AI 操作终端 | 运行命令、安装依赖 |

---

### 快捷键

| 快捷键 | 功能 |
|--------|------|
| `⌘ + J` | 打开/关闭 AI Chat 面板 |
| `⌘ + K` | 行内 AI 编辑 |
| `⌘ + I` | 打开 Builder 模式 |
| `Tab` | 接受补全建议 |
| `Esc` | 拒绝补全建议 |
| `⌘ + Shift + P` | 命令面板 |
| `⌘ + P` | 快速打开文件 |
| `@` | 在 Chat 中引用文件 |

---

## Prompt 模板速查

好的 Prompt 是获得高质量 AI 输出的关键。以下模板可以直接复制使用。

### 代码生成 Prompt

```
请使用 SwiftUI + iOS 17+ 创建一个 [功能描述] 页面。

要求：
- 使用 MVVM 架构
- ViewModel 使用 @Observable 宏
- 视图必须是 struct
- 支持深色模式
- 添加适当的动画效果
- 错误处理使用 do-catch

参考风格：[描述你想要的 UI 风格，如"iOS 设置页面的风格"]
```

**示例**：

```
请使用 SwiftUI + iOS 17+ 创建一个记账列表页面。

要求：
- 使用 MVVM 架构
- ViewModel 使用 @Observable 宏
- 列表显示每笔记录的名称、金额、日期
- 支持按日期排序
- 支持左滑删除
- 空列表时显示占位图
- 支持深色模式
```

---

### 代码审查 Prompt

```
请审查以下代码，从以下维度给出改进建议：

1. **正确性**：是否有逻辑错误或潜在 Bug？
2. **性能**：是否有性能问题？
3. **安全性**：是否有安全隐患？
4. **可读性**：代码是否清晰易懂？
5. **SwiftUI 最佳实践**：是否符合 SwiftUI 的推荐写法？

代码：
[粘贴代码]
```

---

### 调试 Prompt

```
以下代码运行时出现了问题：

**错误信息**：
[粘贴错误信息]

**相关代码**：
[粘贴代码]

**期望行为**：
[描述你期望的效果]

**实际行为**：
[描述实际发生的情况]

请帮我分析原因并提供修复方案。
```

---

### 重构 Prompt

```
请重构以下代码，要求：

1. 提取重复逻辑为独立方法
2. 简化嵌套层级
3. 改善命名（使用更有意义的变量名）
4. 添加错误处理
5. 保持功能不变

代码：
[粘贴代码]
```

---

### 写文档 Prompt

```
请为以下代码编写文档：

要求：
- 使用 /// 格式的文档注释
- 包含功能描述
- 包含参数说明
- 包含返回值说明
- 包含使用示例

代码：
[粘贴代码]
```

---

### 写测试 Prompt

```
请为以下代码编写单元测试：

要求：
- 使用 Swift Testing 框架
- 覆盖正常情况
- 覆盖边界情况
- 覆盖错误情况
- 测试命名清晰：test_方法名_场景_预期结果

代码：
[粘贴代码]
```

---

## AI 工具对比速查

| 特性 | Claude Code | OpenAI Codex | Cursor | Trae |
|------|-------------|-------------|--------|------|
| 类型 | 命令行工具 | 命令行工具 | IDE | IDE |
| 运行环境 | 终端 | 终端 | 独立应用 | 独立应用 |
| 核心模型 | Claude Sonnet/Opus | GPT-4o/o3 | 多模型可选 | 多模型可选 |
| 多文件编辑 | ✅ | ✅ | ✅ Composer | ✅ Builder |
| 代码补全 | ❌ | ❌ | ✅ | ✅ |
| 项目上下文 | CLAUDE.md | CODEX.md | .cursorrules | 项目规则 |
| 免费额度 | 有 | 有 | 有 | 有 |
| 适合场景 | 终端用户、自动化 | 终端用户 | 全流程开发 | 全流程开发 |
| 学习难度 | ⭐⭐ | ⭐⭐ | ⭐ | ⭐ |

**选择建议**：

- 🖥️ **喜欢命令行** → Claude Code 或 OpenAI Codex
- 🎨 **喜欢图形界面** → Cursor 或 Trae
- 🆓 **预算有限** → Trae（免费额度较多）
- 💪 **追求最强** → Cursor + Claude Code 组合使用

---

## 国内 AI 编程工具速查

> 国内 AI 编程工具发展迅速，以下是目前主流的国产 AI 编程工具，适合国内开发者使用。

| 工具 | 出品方 | 类型 | IDE 支持 | Swift 支持 | 价格 | 官网 |
|------|--------|------|---------|-----------|------|------|
| 通义灵码 | 阿里云 | 插件 | VS Code / JetBrains | ⭐⭐⭐ | 免费 | tongyi.aliyun.com/lingma |
| CodeGeeX | 智谱 AI | 插件 | VS Code / JetBrains / Vim | ⭐⭐ | 免费 | codegeex.cn |
| Baidu Comate | 百度 | 插件 | VS Code / JetBrains | ⭐⭐ | 免费 | comate.baidu.com |
| 豆包 MarsCode | 字节跳动 | IDE + 插件 | 独立 IDE / VS Code | ⭐⭐⭐ | 免费 | marscode.com |

**选择建议**：

- 🔌 **已有 VS Code / JetBrains** → 通义灵码或 CodeGeeX（插件形式，无缝集成）
- 🆓 **预算有限** → 全部免费，按 Swift 支持度优先选通义灵码或豆包 MarsCode
- 🖥️ **想要独立 IDE** → 豆包 MarsCode（提供独立 IDE 体验）
- 🇨🇳 **国内网络环境** → 以上工具均无需翻墙，访问速度快

---

## 国产大模型 API 速查

> 在 iOS App 中集成 AI 功能时，面向国内用户应优先使用国产大模型 API，避免数据出境合规风险。以下 API 均兼容 OpenAI 接口格式，迁移成本低。

| 模型 | 提供商 | API 格式 | 上下文长度 | 价格（输入/输出） | 免费额度 |
|------|--------|---------|-----------|-----------------|---------|
| 通义千问 qwen-plus | 阿里云 | OpenAI 兼容 | 128K | ¥0.8/百万token / ¥2/百万token | 100万token |
| DeepSeek Chat | DeepSeek | OpenAI 兼容 | 64K | ¥1/百万token / ¥2/百万token | 500万token |
| GLM-4 | 智谱 AI | OpenAI 兼容 | 128K | ¥15/百万token / ¥15/百万token | 有限 |
| Moonshot v1 | 月之暗面 | OpenAI 兼容 | 8K-32K | ¥12/百万token / ¥12/百万token | 有限 |
| 星火 4.0 Ultra | 讯飞 | OpenAI 兼容 | 8K | 按量计费 | 有限 |

**接入要点**：

1. **OpenAI 兼容格式**：以上模型均支持 OpenAI Chat Completions API 格式，只需更换 `base_url` 和 `api_key` 即可切换
2. **Swift 接入示例**：

```swift
// 通用 OpenAI 兼容请求封装
struct LLMConfig {
    var baseURL: String
    var apiKey: String
    var model: String

    static let qwen = LLMConfig(
        baseURL: "https://dashscope.aliyuncs.com/compatible-mode/v1",
        apiKey: "your-api-key",
        model: "qwen-plus"
    )

    static let deepseek = LLMConfig(
        baseURL: "https://api.deepseek.com/v1",
        apiKey: "your-api-key",
        model: "deepseek-chat"
    )
}
```

3. **成本优化**：简单任务用 DeepSeek / 通义千问（便宜），复杂推理用 GLM-4 / Moonshot

---

## iOS App 集成 AI 功能 Prompt 模板

> 以下 Prompt 模板专门针对 iOS App 集成 AI 功能的场景，可直接复制使用。

### 生成 LLMService 网络封装的 Prompt

```
请使用 Swift 5.9+ 为 iOS App 创建一个 LLMService 网络请求封装类。

要求：
- 使用 async/await 异步模式
- 支持 OpenAI 兼容 API 格式（可切换不同国产模型）
- 支持流式输出（SSE），使用 URLSession.bytes 逐行读取
- 支持非流式输出
- API Key 从 Keychain 读取，不硬编码
- 请求超时设置 60 秒
- 错误处理使用自定义 LLMError 枚举
- 使用 MVVM 架构，Service 层独立于 ViewModel
- 代码添加 /// 格式文档注释

参考接口格式：
POST /v1/chat/completions
Headers: Authorization: Bearer <api_key>
Body: { "model": "...", "messages": [...], "stream": true/false }
```

---

### 生成聊天界面的 Prompt

```
请使用 SwiftUI + iOS 17+ 创建一个 AI 聊天界面。

要求：
- 使用 MVVM 架构，ViewModel 使用 @Observable 宏
- 消息列表使用 LazyVStack + ScrollView + ScrollViewReader
- 支持用户消息和 AI 消息两种气泡样式
- AI 消息支持逐字打字机效果（流式输出时逐步显示）
- 底部输入框固定，支持多行输入（最大 4 行）
- 发送按钮在输入为空时禁用
- AI 正在回复时显示加载指示器
- 支持复制 AI 回复内容（长按菜单）
- 支持深色模式
- 消息模型包含 id、role（user/assistant）、content、timestamp
```

---

### 生成 AI 功能合规检查清单的 Prompt

```
请为 iOS App 集成 AI 功能生成一份合规检查清单，涵盖以下维度：

1. 数据安全：API Key 存储、用户数据传输加密、本地数据保护
2. 隐私合规：隐私政策更新、用户数据收集声明、数据出境评估
3. 内容安全：AI 生成内容标识、敏感词过滤、内容审核机制
4. 算法备案：是否需要算法备案、备案流程、备案信息展示
5. App Store 审核：AI 功能描述、内容审核机制说明、年龄分级
6. 用户体验：AI 回复免责声明、错误提示、离线降级方案

每个维度列出具体的检查项，用 ✅ / ❌ 标记格式，方便逐项核对。
```
