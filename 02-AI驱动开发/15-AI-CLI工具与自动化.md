# 15-AI CLI 工具与自动化

## 本章目标

- 理解什么是 CLI，以及 AI CLI 工具为什么比传统 CLI 更强大
- 掌握主流 AI CLI 工具的特点与适用场景，能根据需求选择合适工具
- 深度实战 Claude Code，学会在 iOS 项目中高效使用
- 了解 Codex CLI 和 Aider 的基本用法
- 学会用 AI CLI 驱动 xcodebuild，实现构建、测试、修复的自动化循环
- 掌握实用自动化脚本的编写，提升日常开发效率
- 建立 AI CLI 最佳实践意识：何时用、怎么写 Prompt、如何控制成本与安全

---

## 1. AI CLI 工具全景

### 1.1 什么是 CLI？

CLI 全称 **Command Line Interface**，即命令行界面。简单来说，就是你在一个黑底白字的终端窗口里，通过输入文字命令来操作电脑，而不是用鼠标点来点去。

> 💡 **生活类比**：GUI（图形界面）就像去餐厅看菜单点菜，有图片有分类，一目了然；CLI 就像直接跟厨师说你要什么，虽然需要你知道菜名，但说得快、说得准，效率更高。

传统 CLI 的典型例子：

```bash
# 列出当前目录的文件
ls -la

# 创建一个新文件夹
mkdir MyProject

# 用 Xcode 构建项目
xcodebuild -project MyApp.xcodeproj -scheme MyApp build
```

传统 CLI 的问题是：**你必须精确知道命令的语法和参数**。记不住？那就得查文档，效率大打折扣。

### 1.2 为什么 AI CLI 更强大？

AI CLI = 传统 CLI + 大语言模型。它让你可以用**自然语言**告诉电脑你想做什么，AI 会帮你翻译成具体的命令并执行。

| 对比维度 | 传统 CLI | AI CLI |
|---------|---------|--------|
| 输入方式 | 精确的命令语法 | 自然语言描述 |
| 学习成本 | 高，需记大量命令 | 低，会说人话就行 |
| 出错处理 | 报错后自己查原因 | AI 自动分析并修复 |
| 上下文理解 | 无 | 能理解项目结构和代码上下文 |
| 批量操作 | 需写复杂脚本 | 一句话描述即可 |
| 适用人群 | 老手 | 新手也能上手 |

> 💡 **一句话总结**：传统 CLI 是"你告诉电脑怎么做"，AI CLI 是"你告诉电脑你想要什么"。

### 1.3 主流 AI CLI 工具对比表

| 工具 | 开发者 | 核心模型 | 开源 | 代码编辑 | 项目理解 | iOS 适配 | 价格 |
|------|--------|---------|------|---------|---------|---------|------|
| **Claude Code** | Anthropic | Claude 4 Sonnet | 否 | ✅ 直接编辑 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | API 计费 |
| **Codex CLI** | OpenAI | o4-mini / GPT-4.1 | ✅ | ✅ 沙盒编辑 | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | API 计费 |
| **Aider** | 开源社区 | 多模型 | ✅ | ✅ Git 集成 | ⭐⭐⭐⭐ | ⭐⭐⭐ | 按模型计费 |
| **GitHub Copilot CLI** | GitHub | GPT-4 | 否 | ❌ 仅建议 | ⭐⭐ | ⭐⭐ | 订阅制 |
| **Cursor CLI** | Cursor | 多模型 | 否 | ✅ 直接编辑 | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | 订阅制 |

### 1.4 各工具适用场景

| 场景 | 推荐工具 | 原因 |
|------|---------|------|
| iOS 项目日常开发 | Claude Code | 项目理解最深，SwiftUI 支持最好 |
| 全自动批量任务 | Codex CLI | `--full-auto` 模式适合无人值守 |
| 需要多模型切换 | Aider | 支持 GPT-4o、Claude 等多种模型 |
| 只想知道命令怎么写 | GitHub Copilot CLI | 生成命令建议，不直接执行 |
| 已有 Cursor 订阅 | Cursor CLI | 无需额外付费，体验一致 |

> ⚠️ **重要提醒**：AI CLI 工具更新非常快，以上信息基于 2025 年中的情况，请以各工具官网最新信息为准。

---

## 2. Claude Code 深度实战（iOS 开发视角）

Claude Code 是目前对 iOS 开发支持最好的 AI CLI 工具。它能够理解 Xcode 项目结构、Swift/SwiftUI 语法，甚至能直接帮你修改代码文件。

### 2.1 安装 Claude Code

```bash
# 前提：已安装 Node.js 18+
npm install -g @anthropic-ai/claude-code

# 验证安装
claude --version

# 首次运行，会引导你登录 Anthropic 账号
claude
```

### 2.2 常用命令速查

在 Claude Code 交互界面中，你可以使用以下斜杠命令：

| 命令 | 作用 | 使用场景 |
|------|------|---------|
| `/init` | 初始化项目配置 | 在新项目中首次使用时 |
| `/add <文件>` | 添加文件到上下文 | 让 AI 关注特定文件 |
| `/compact` | 压缩对话历史 | 对话太长导致变慢时 |
| `/cost` | 查看当前会话费用 | 随时关注 API 消耗 |
| `/permissions` | 管理权限规则 | 控制哪些操作需要确认 |
| `/clear` | 清空对话 | 开始新任务时 |
| `/help` | 查看帮助 | 忘了命令时 |
| `/quit` | 退出 | 结束使用时 |

> 💡 **小技巧**：对话太长时，AI 的响应会变慢且容易"遗忘"早期内容。及时用 `/compact` 压缩，或用 `/clear` 重新开始。

### 2.3 iOS 项目实战命令示例

#### 示例 1：创建一个 SwiftUI 登录页面

```
> 帮我创建一个 SwiftUI 登录页面，包含用户名和密码输入框，以及一个登录按钮。按钮点击时打印输入内容。使用现代 iOS 设计风格。
```

Claude Code 会自动：
1. 创建 `LoginView.swift` 文件
2. 编写完整的 SwiftUI 代码
3. 将文件添加到 Xcode 项目中（如果项目使用文件系统引用）

生成的代码大致如下：

```swift
import SwiftUI

struct LoginView: View {
    @State private var username = ""
    @State private var password = ""

    var body: some View {
        VStack(spacing: 20) {
            Text("欢迎登录")
                .font(.largeTitle)
                .bold()

            TextField("用户名", text: $username)
                .textFieldStyle(.roundedBorder)
                .autocapitalization(.none)

            SecureField("密码", text: $password)
                .textFieldStyle(.roundedBorder)

            Button(action: {
                print("用户名: \(username), 密码: \(password)")
            }) {
                Text("登录")
                    .frame(maxWidth: .infinity)
                    .padding()
                    .background(Color.blue)
                    .foregroundColor(.white)
                    .cornerRadius(10)
            }
            .disabled(username.isEmpty || password.isEmpty)

            Spacer()
        }
        .padding()
    }
}
```

#### 示例 2：给 View 添加暗黑模式支持

```
> 给 LoginView 添加暗黑模式支持，在浅色和深色模式下都要有良好的视觉效果
```

Claude Code 会修改代码，使用自适应颜色和系统语义颜色：

```swift
VStack(spacing: 20) {
    Text("欢迎登录")
        .font(.largeTitle)
        .bold()
        .foregroundColor(Color(.label))

    TextField("用户名", text: $username)
        .textFieldStyle(.roundedBorder)

    SecureField("密码", text: $password)
        .textFieldStyle(.roundedBorder)

    Button(action: {
        print("用户名: \(username), 密码: \(password)")
    }) {
        Text("登录")
            .frame(maxWidth: .infinity)
            .padding()
            .background(Color(.systemBlue))
            .foregroundColor(.white)
            .cornerRadius(10)
    }
    .disabled(username.isEmpty || password.isEmpty)
}
.padding()
.background(Color(.systemBackground))
```

> 💡 **关键点**：`Color(.label)`、`Color(.systemBackground)`、`Color(.systemBlue)` 这些语义颜色会自动适配浅色/深色模式，不需要手动判断。

#### 示例 3：修复编译错误

```
> 修复这个编译错误：'ObservableObject' is only available in iOS 14.0 or newer
```

Claude Code 会分析错误并给出修复方案：

```swift
// 修复前（可能缺少可用性声明）
class ViewModel: ObservableObject {
    @Published var items: [Item] = []
}

// 修复后（添加可用性标注）
@available(iOS 14.0, *)
class ViewModel: ObservableObject {
    @Published var items: [Item] = []
}
```

或者更推荐的方式——直接在项目部署目标中设置最低版本：

```
> 把项目的部署目标改为 iOS 16.0
```

#### 示例 4：为网络请求添加错误处理

```
> 为这个网络请求添加错误处理，包括网络超时、服务器错误、数据解析失败等情况
```

Claude Code 会帮你完善错误处理：

```swift
enum NetworkError: LocalizedError {
    case invalidURL
    case timeout
    case serverError(Int)
    case decodingFailed
    case unknown(Error)

    var errorDescription: String? {
        switch self {
        case .invalidURL:
            return "无效的网址"
        case .timeout:
            return "请求超时，请检查网络连接"
        case .serverError(let code):
            return "服务器错误（\(code)）"
        case .decodingFailed:
            return "数据解析失败"
        case .unknown(let error):
            return "未知错误：\(error.localizedDescription)"
        }
    }
}

func fetchItems() async throws -> [Item] {
    guard let url = URL(string: "https://api.example.com/items") else {
        throw NetworkError.invalidURL
    }

    var request = URLRequest(url: url)
    request.timeoutInterval = 30

    let (data, response) = try await URLSession.shared.data(for: request)

    guard let httpResponse = response as? HTTPURLResponse,
          (200...299).contains(httpResponse.statusCode) else {
        let statusCode = (response as? HTTPURLResponse)?.statusCode ?? 0
        throw NetworkError.serverError(statusCode)
    }

    do {
        return try JSONDecoder().decode([Item].self, from: data)
    } catch {
        throw NetworkError.decodingFailed
    }
}
```

### 2.4 CLAUDE.md 项目配置文件详解

`CLAUDE.md` 是 Claude Code 的项目配置文件，放在项目根目录下。它就像给 AI 写的一份"项目说明书"，让 AI 更好地理解你的项目。

```
MyApp/
├── CLAUDE.md          ← Claude Code 的项目配置
├── MyApp.xcodeproj/
├── MyApp/
│   ├── Views/
│   ├── Models/
│   └── ViewModels/
└── MyAppTests/
```

一个 iOS 项目的 `CLAUDE.md` 示例：

```markdown
# MyApp 项目说明

## 项目概况
- 类型：iOS 原生应用
- 最低部署目标：iOS 16.0
- 主要语言：Swift + SwiftUI
- 架构模式：MVVM

## 代码规范
- 使用 Swift 5.9+ 特性
- 视图文件放在 Views/ 目录
- ViewModel 文件放在 ViewModels/ 目录
- 模型文件放在 Models/ 目录
- 网络请求统一使用 async/await
- 注释使用中文

## 重要约定
- 所有网络请求必须包含错误处理
- 颜色使用 Asset Catalog 中定义的语义颜色
- 字体使用系统字体，不引入第三方字体库
- 图片使用 SF Symbols 优先

## 测试
- 单元测试放在 MyAppTests/
- UI 测试放在 MyAppUITests/
- 运行测试：xcodebuild test -project MyApp.xcodeproj -scheme MyApp -destination 'platform=iOS Simulator,name=iPhone 16'
```

> 💡 **为什么要写 CLAUDE.md？** 想象你新招了一个实习生，你不告诉他项目规范，他写出来的代码肯定不符合要求。CLAUDE.md 就是给 AI 这个"超级实习生"的项目规范手册。

### 2.5 权限管理：allow/deny 规则

Claude Code 默认会在执行某些操作前询问你确认。你可以通过权限规则来跳过确认，提高效率。

在 `CLAUDE.md` 中添加权限规则：

```markdown
## 权限规则

### 允许的操作
- 读取任何 .swift 文件
- 修改 Views/ 目录下的文件
- 运行 xcodebuild build
- 运行 swift test

### 禁止的操作
- 修改 .xcodeproj 文件
- 删除任何文件
- 执行 git push
- 修改 Package.swift
```

也可以在对话中用 `/permissions` 命令动态调整：

```
> /permissions
当前权限设置：
  读取文件：允许
  写入文件：需确认
  执行命令：需确认

> 允许写入 .swift 文件
已更新：写入 .swift 文件 → 允许
```

> ⚠️ **安全提醒**：不要轻易允许"执行任意命令"或"删除文件"等高风险操作。AI 虽然聪明，但也会犯错，保留关键操作的确认机制是明智的。

---

## 3. Codex CLI 实战

Codex CLI 是 OpenAI 推出的开源命令行 AI 工具，最大的特色是支持 `--full-auto` 全自动模式。

### 3.1 安装与配置

```bash
# 前提：已安装 Node.js 22+
npm install -g @openai/codex

# 设置 OpenAI API Key
export OPENAI_API_KEY="sk-your-api-key-here"

# 验证安装
codex --version
```

> 💡 **建议**：把 API Key 写进 `~/.zshrc`，这样每次打开终端都自动生效：
> ```bash
> echo 'export OPENAI_API_KEY="sk-your-key"' >> ~/.zshrc
> source ~/.zshrc
> ```

### 3.2 codex --full-auto 模式

Codex CLI 有两种主要模式：

| 模式 | 命令 | 特点 |
|------|------|------|
| 交互模式 | `codex` | 每步操作需确认，安全 |
| 全自动模式 | `codex --full-auto` | 自动执行所有操作，无需确认 |

> ⚠️ **警告**：`--full-auto` 模式下 AI 会自动执行命令、修改文件，不经过确认。务必在受控环境（如 Git 仓库）中使用，方便随时回滚！

全自动模式的使用示例：

```bash
# 全自动修复所有编译错误
codex --full-auto "修复这个项目中的所有编译错误"

# 全自动添加单元测试
codex --full-auto "为 Models/ 目录下的所有模型添加单元测试"

# 全自动重构
codex --full-auto "把所有 UIKit 代码迁移到 SwiftUI"
```

### 3.3 iOS 项目中的使用场景

```bash
# 场景 1：自动修复 SwiftLint 警告
codex --full-auto "运行 SwiftLint 并修复所有可以自动修复的警告"

# 场景 2：批量添加访问控制修饰符
codex "给所有未标注访问级别的类型和函数添加合适的访问控制修饰符（private/fileprivate/internal/public）"

# 场景 3：生成模型文件
codex "根据 API 文档（docs/api.md）生成对应的 Swift 模型文件，使用 Codable 协议"
```

> 💡 **Codex CLI vs Claude Code**：Codex CLI 的全自动模式适合"跑起来就不用管"的批量任务；Claude Code 更适合需要精细控制的交互式开发。

---

## 4. Aider 实战

Aider 是一个开源的 AI 编程助手，最大的特色是**深度集成 Git** 和**支持多种大模型**。

### 4.1 安装与配置

```bash
# 使用 pip 安装
pip install aider-chat

# 或使用 pipx（推荐，避免依赖冲突）
pipx install aider-chat

# 验证安装
aider --version
```

配置 API Key：

```bash
# 使用 OpenAI
export OPENAI_API_KEY="sk-your-key"

# 使用 Anthropic Claude
export ANTHROPIC_API_KEY="sk-ant-your-key"

# 使用 OpenRouter（可访问多种模型）
export OPENROUTER_API_KEY="sk-or-your-key"
```

### 4.2 多模型支持

Aider 的强大之处在于可以灵活切换模型：

```bash
# 使用 GPT-4o（默认）
aider --model gpt-4o

# 使用 Claude 3.5 Sonnet
aider --model claude-3.5-sonnet

# 使用 DeepSeek Coder（性价比高）
aider --model deepseek/deepseek-coder

# 使用 OpenRouter 的模型
aider --model openrouter/anthropic/claude-3.5-sonnet
```

| 模型 | 编程能力 | 速度 | 成本 | 推荐场景 |
|------|---------|------|------|---------|
| GPT-4o | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | 💰💰💰 | 复杂逻辑、架构设计 |
| Claude 3.5 Sonnet | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | 💰💰 | 代码重构、长上下文 |
| DeepSeek Coder | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | 💰 | 日常编码、批量任务 |

### 4.3 iOS 项目中的使用场景

```bash
# 进入 iOS 项目目录
cd ~/Projects/MyApp

# 启动 Aider，添加需要编辑的文件
aider MyApp/Views/ContentView.swift MyApp/ViewModels/ContentViewModel.swift

# 在 Aider 交互界面中输入指令
> 把 ContentView 中的列表改为使用 LazyVStack 实现，并添加下拉刷新功能
```

Aider 的 Git 集成特别好用：

```bash
# Aider 每次修改都会自动创建 Git commit
# 查看修改历史
git log --oneline

# 如果 AI 改坏了，一键回滚
git revert HEAD

# 查看 AI 做了什么修改
git diff HEAD~1
```

> 💡 **Aider 的 Git 工作流**：Aider 就像一个自带版本控制的搭档——每次修改都自动提交，不满意随时回退。这给了你"随便试"的底气。

---

## 5. xcodebuild + AI 自动化

`xcodebuild` 是 Apple 提供的命令行构建工具，可以让你脱离 Xcode 图形界面来构建、测试、打包 iOS 项目。结合 AI CLI，就能实现真正的自动化开发流程。

### 5.1 xcodebuild 基础命令

```bash
# 构建项目
xcodebuild -project MyApp.xcodeproj \
  -scheme MyApp \
  -destination 'platform=iOS Simulator,name=iPhone 16' \
  build

# 运行测试
xcodebuild -project MyApp.xcodeproj \
  -scheme MyApp \
  -destination 'platform=iOS Simulator,name=iPhone 16' \
  test

# 清理构建缓存
xcodebuild -project MyApp.xcodeproj \
  -scheme MyApp \
  clean

# 导出 Archive
xcodebuild -project MyApp.xcodeproj \
  -scheme MyApp \
  -archivePath build/MyApp.xcarchive \
  archive
```

> 💡 **生活类比**：xcodebuild 就像 Xcode 的"遥控器"——你在终端输入命令，它帮你做和 Xcode 一样的事情，但可以自动化、可以批量执行。

### 5.2 用 AI CLI 驱动 xcodebuild 构建

```bash
# 用 Claude Code 驱动构建
claude "运行 xcodebuild 构建项目，如果有编译错误就修复它们"

# 用 Codex CLI 全自动构建+修复
codex --full-auto "构建这个 iOS 项目，自动修复所有编译错误，直到构建成功为止"
```

AI 会自动执行类似这样的流程：

```
1. 运行 xcodebuild build
2. 解析编译错误
3. 修改源代码修复错误
4. 再次运行 xcodebuild build
5. 重复直到构建成功
```

### 5.3 自动化测试与错误修复循环

这是 AI CLI 最强大的用法之一——**自动测试修复循环**：

```bash
# Claude Code 版本
claude "运行所有单元测试，如果测试失败，分析失败原因并修复代码，然后重新运行测试，直到所有测试通过"
```

AI 的执行过程可能如下：

```
第 1 轮：
  运行测试 → 3 个失败
  修复 TestA：数组越界问题
  修复 TestB：异步等待超时
  修复 TestC：JSON 解码错误

第 2 轮：
  运行测试 → 1 个失败
  修复 TestB：异步逻辑仍有问题

第 3 轮：
  运行测试 → 全部通过 ✅
```

> ⚠️ **注意**：自动修复循环可能消耗大量 API 额度。建议设置最大重试次数，避免无限循环。

### 5.4 CI/CD 集成思路

将 AI CLI 集成到 CI/CD 流水线中，可以实现更高级的自动化：

```yaml
# GitHub Actions 示例
name: AI Auto Fix

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  auto-fix:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build and Test
        run: |
          xcodebuild test \
            -project MyApp.xcodeproj \
            -scheme MyApp \
            -destination 'platform=iOS Simulator,name=iPhone 16'

      - name: AI Auto Fix (if tests fail)
        if: failure()
        run: |
          npm install -g @anthropic-ai/claude-code
          claude --full-auto "测试失败了，请查看测试输出，修复代码，确保测试通过"

      - name: Commit Fix
        if: failure()
        run: |
          git config user.name "AI Bot"
          git config user.email "bot@example.com"
          git add -A
          git commit -m "fix: AI 自动修复测试失败"
          git push
```

> 💡 **CI/CD 中的 AI**：目前 AI CLI 在 CI/CD 中的使用还处于早期阶段，建议先在本地验证效果，再考虑集成到自动化流水线中。

---

## 6. 实用自动化脚本

这一节我们编写一些实用的 Shell 脚本，结合 AI CLI 使用，可以大幅提升日常开发效率。

### 6.1 自动生成 Swift 文件模板

每次新建 Swift 文件都要写一堆模板代码？让脚本帮你搞定：

```bash
#!/bin/bash
# 文件名：swift-template.sh
# 用法：./swift-template.sh View LoginView

TYPE=$1
NAME=$2
DIR=$3

if [ -z "$NAME" ]; then
    echo "用法: ./swift-template.sh <类型> <名称> [目录]"
    echo "类型: View, ViewModel, Model, Service"
    echo "示例: ./swift-template.sh View LoginView Views"
    exit 1
fi

TARGET_DIR="${DIR:-.}"
mkdir -p "$TARGET_DIR"
FILE_PATH="$TARGET_DIR/${NAME}.swift"

case $TYPE in
    View)
        cat > "$FILE_PATH" << EOF
import SwiftUI

struct ${NAME}: View {
    var body: some View {
        Text("${NAME}")
    }
}

#Preview {
    ${NAME}()
}
EOF
        ;;
    ViewModel)
        cat > "$FILE_PATH" << EOF
import Foundation

@MainActor
class ${NAME}: ObservableObject {
    @Published var isLoading = false

    func loadData() async {
        isLoading = true
        defer { isLoading = false }
    }
}
EOF
        ;;
    Model)
        cat > "$FILE_PATH" << EOF
import Foundation

struct ${NAME}: Codable, Identifiable {
    let id: String
}
EOF
        ;;
    Service)
        cat > "$FILE_PATH" << EOF
import Foundation

actor ${NAME} {
    static let shared = ${NAME}()

    private init() {}
}
EOF
        ;;
    *)
        echo "未知类型: $TYPE"
        echo "支持的类型: View, ViewModel, Model, Service"
        exit 1
        ;;
esac

echo "✅ 已创建: $FILE_PATH"
```

使用方式：

```bash
# 创建一个 View 文件
./swift-template.sh View LoginView MyApp/Views

# 创建一个 ViewModel 文件
./swift-template.sh ViewModel LoginViewModel MyApp/ViewModels

# 创建一个 Model 文件
./swift-template.sh Model User MyApp/Models
```

### 6.2 自动化截图生成

App Store 需要各种尺寸的截图，手动截图非常繁琐。以下脚本结合 Simulator 自动化截图：

```bash
#!/bin/bash
# 文件名：auto-screenshots.sh
# 用法：./auto-screenshots.sh

SCHEME="MyApp"
PROJECT="MyApp.xcodeproj"
SCREENSHOT_DIR="./screenshots"
DEVICES=("iPhone 16" "iPhone 16 Pro Max" "iPad Pro 13-inch (M4)")

mkdir -p "$SCREENSHOT_DIR"

echo "🏗️  构建项目..."
xcodebuild -project "$PROJECT" -scheme "$SCHEME" \
  -destination "platform=iOS Simulator,name=${DEVICES[0]}" \
  build > /dev/null 2>&1

for DEVICE in "${DEVICES[@]}"; do
    echo "📱 在 $DEVICE 上截图..."

    BOOTED=$(xcrun simctl list devices | grep "$DEVICE" | grep "Booted" | head -1)
    if [ -z "$BOOTED" ]; then
        xcrun simctl boot "$DEVICE" 2>/dev/null || true
    fi

    DEVICE_DIR="$SCREENSHOT_DIR/$DEVICE"
    mkdir -p "$DEVICE_DIR"

    xcrun simctl io "$DEVICE" screenshot "$DEVICE_DIR/01_home.png"

    echo "  ✅ 截图已保存到 $DEVICE_DIR"
done

echo "🎉 所有截图完成！"
```

> 💡 **进阶用法**：结合 AI CLI，可以让 AI 自动生成截图所需的 UI 状态代码，截完图后再恢复原状。

### 6.3 自动化版本号更新

每次发版都要手动改版本号？这个脚本一键搞定：

```bash
#!/bin/bash
# 文件名：bump-version.sh
# 用法：./bump-version.sh [major|minor|patch]

BUMP_TYPE=${1:-patch}
PLIST_PATH="MyApp/Info.plist"

CURRENT_VERSION=$(/usr/libexec/PlistBuddy -c "Print CFBundleShortVersionString" "$PLIST_PATH")
CURRENT_BUILD=$(/usr/libexec/PlistBuddy -c "Print CFBundleVersion" "$PLIST_PATH")

IFS='.' read -r MAJOR MINOR PATCH <<< "$CURRENT_VERSION"

case $BUMP_TYPE in
    major)
        MAJOR=$((MAJOR + 1))
        MINOR=0
        PATCH=0
        ;;
    minor)
        MINOR=$((MINOR + 1))
        PATCH=0
        ;;
    patch)
        PATCH=$((PATCH + 1))
        ;;
    *)
        echo "用法: ./bump-version.sh [major|minor|patch]"
        exit 1
        ;;
esac

NEW_VERSION="$MAJOR.$MINOR.$PATCH"
NEW_BUILD=$((CURRENT_BUILD + 1))

/usr/libexec/PlistBuddy -c "Set CFBundleShortVersionString $NEW_VERSION" "$PLIST_PATH"
/usr/libexec/PlistBuddy -c "Set CFBundleVersion $NEW_BUILD" "$PLIST_PATH"

echo "📌 版本号: $CURRENT_VERSION → $NEW_VERSION"
echo "📌 构建号: $CURRENT_BUILD → $NEW_BUILD"
```

使用方式：

```bash
# 修订版 +1（1.0.0 → 1.0.1）
./bump-version.sh patch

# 次版本 +1（1.0.1 → 1.1.0）
./bump-version.sh minor

# 主版本 +1（1.1.0 → 2.0.0）
./bump-version.sh major
```

### 6.4 自动化 App Store 提交

将构建、导出、上传整合为一个脚本：

```bash
#!/bin/bash
# 文件名：submit-appstore.sh
# 用法：./submit-appstore.sh

SCHEME="MyApp"
PROJECT="MyApp.xcodeproj"
ARCHIVE_PATH="./build/MyApp.xcarchive"
EXPORT_PATH="./build/export"
EXPORT_OPTIONS="./ExportOptions.plist"

echo "🧹 清理旧构建..."
rm -rf ./build
mkdir -p ./build

echo "🏗️  Archive 项目..."
xcodebuild archive \
  -project "$PROJECT" \
  -scheme "$SCHEME" \
  -archivePath "$ARCHIVE_PATH" \
  -destination "generic/platform=iOS" \
  | tail -5

if [ ${PIPESTATUS[0]} -ne 0 ]; then
    echo "❌ Archive 失败！"
    exit 1
fi

echo "📦 导出 IPA..."
xcodebuild -exportArchive \
  -archivePath "$ARCHIVE_PATH" \
  -exportPath "$EXPORT_PATH" \
  -exportOptionsPlist "$EXPORT_OPTIONS" \
  | tail -5

if [ ${PIPESTATUS[0]} -ne 0 ]; then
    echo "❌ 导出失败！"
    exit 1
fi

echo "☁️  上传到 App Store Connect..."
xcrun altool --upload-app \
  --type ios \
  --file "$EXPORT_PATH"/*.ipa \
  --apiKey YOUR_API_KEY \
  --apiIssuer YOUR_ISSUER_ID

if [ $? -eq 0 ]; then
    echo "🎉 上传成功！请在 App Store Connect 中提交审核。"
else
    echo "❌ 上传失败！请检查错误信息。"
    exit 1
fi
```

> ⚠️ **注意**：`ExportOptions.plist` 需要提前配置好，包含你的团队 ID、打包方式等信息。建议先在 Xcode 中手动导出一次，Xcode 会自动生成这个文件。

`ExportOptions.plist` 示例：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>method</key>
    <string>app-store</string>
    <key>teamID</key>
    <string>YOUR_TEAM_ID</string>
    <key>uploadSymbols</key>
    <true/>
</dict>
</plist>
```

---

## 7. AI CLI 最佳实践

### 7.1 何时用 CLI vs IDE

| 场景 | 推荐 CLI | 推荐 IDE | 原因 |
|------|---------|---------|------|
| 快速修复编译错误 | ✅ Claude Code | | CLI 可以自动循环修复 |
| 探索性 UI 开发 | | ✅ Xcode + SwiftUI Preview | 需要实时预览 |
| 批量重构 | ✅ Codex CLI | | 全自动模式效率高 |
| 调试复杂布局 | | ✅ Xcode View Debugger | 需要可视化工具 |
| 编写单元测试 | ✅ Aider | | AI 可以自动生成测试 |
| 精细 UI 调整 | | ✅ Cursor / Xcode | 需要即时反馈 |
| CI/CD 流水线 | ✅ 任何 CLI | | 服务器上没有 IDE |
| 学习新框架 | | ✅ IDE + 文档 | 需要代码补全和文档 |

> 💡 **经验法则**：需要"看效果"的用 IDE，需要"批量做"的用 CLI。两者结合才是最高效的工作方式。

### 7.2 Prompt 编写技巧

写好 Prompt 是用好 AI CLI 的关键。以下是几条实用技巧：

#### 技巧 1：明确上下文

```
❌ 差："修复这个 bug"
✅ 好："修复 ContentView.swift 第 42 行的 'Cannot find type User in scope' 编译错误，User 模型定义在 Models/User.swift 中"
```

#### 技巧 2：指定约束条件

```
❌ 差："创建一个列表页面"
✅ 好："创建一个 SwiftUI 列表页面，使用 LazyVStack，支持下拉刷新，数据从 ViewModel 获取，最低支持 iOS 16.0"
```

#### 技巧 3：分步骤描述复杂任务

```
❌ 差："把整个项目从 UIKit 迁移到 SwiftUI"
✅ 好："分三步完成迁移：
1. 先把 LoginViewController 迁移为 LoginView
2. 然后把 HomeViewController 迁移为 HomeView
3. 最后更新 AppDelegate 和 SceneDelegate 使用新的 SwiftUI 生命周期"
```

#### 技巧 4：提供示例

```
❌ 差："写一个网络请求函数"
✅ 好："写一个网络请求函数，参考现有的 fetchUsers() 函数的风格，使用 async/await，包含错误处理"
```

#### 技巧 5：要求 AI 先分析再动手

```
✅ "在修改代码之前，先分析当前的项目结构和相关文件，然后告诉我你的修改计划，等我确认后再执行"
```

### 7.3 安全与成本控制

#### 安全注意事项

| 风险 | 说明 | 应对措施 |
|------|------|---------|
| 代码泄露 | AI 可能将代码发送到云端 | 使用企业版 API，确认数据处理协议 |
| 误操作 | AI 可能删除重要文件 | 始终在 Git 仓库中使用，定期提交 |
| 依赖注入 | AI 可能引入有漏洞的依赖 | 审查 AI 添加的所有第三方依赖 |
| 敏感信息 | AI 可能读取配置文件中的密钥 | 在 CLAUDE.md 中排除敏感文件 |

在 `CLAUDE.md` 中添加安全规则：

```markdown
## 安全规则
- 不要读取或修改以下文件：Config.secret.swift、APIKeys.plist
- 不要执行 git push 或 git push --force
- 不要修改 .xcodeproj 或 .xcworkspace 文件
- 不要安装新的第三方依赖，除非我明确要求
```

#### 成本控制

AI CLI 按 API 调用计费，不注意的话费用可能很高：

| 控制手段 | 说明 |
|---------|------|
| 使用 `/cost` 命令 | 随时查看当前会话费用 |
| 及时 `/compact` | 压缩对话，减少 Token 消耗 |
| 选择合适的模型 | 简单任务用便宜模型，复杂任务用贵模型 |
| 精准添加文件 | 只 `/add` 必要的文件，不要添加整个项目 |
| 设置预算上限 | 在 API 平台设置月度消费上限 |

```bash
# 成本估算参考（2025 年中价格）
# Claude 4 Sonnet: ~$3 / 百万输入 Token, ~$15 / 百万输出 Token
# GPT-4o: ~$2.5 / 百万输入 Token, ~$10 / 百万输出 Token
# GPT-4.1 mini: ~$0.4 / 百万输入 Token, ~$1.6 / 百万输出 Token

# 一次典型的代码修改会话大约消耗：
# 输入：~10K Token，输出：~2K Token
# 使用 Claude 4 Sonnet 约 $0.06
# 使用 GPT-4.1 mini 约 $0.004
```

> 💡 **省钱秘诀**：简单任务（改个变量名、加个注释）用便宜模型；复杂任务（架构重构、跨文件修改）用贵模型。就像打车——近距离骑共享单车，远距离才叫专车。

---

## 小结

本章我们学习了 AI CLI 工具与自动化的核心知识：

| 知识点 | 关键内容 |
|-------|---------|
| AI CLI 全景 | CLI 是命令行界面，AI CLI 让你用自然语言操作命令行 |
| 工具选择 | Claude Code 最适合 iOS 开发，Codex CLI 适合全自动任务，Aider 支持多模型 |
| Claude Code | 掌握常用命令、CLAUDE.md 配置、权限管理 |
| Codex CLI | `--full-auto` 模式适合批量无人值守任务 |
| Aider | 深度 Git 集成，多模型灵活切换 |
| xcodebuild + AI | AI 驱动构建、测试、修复的自动化循环 |
| 自动化脚本 | 文件模板、截图、版本号、App Store 提交 |
| 最佳实践 | CLI vs IDE 按场景选择，Prompt 要明确具体，注意安全与成本 |

**核心心法**：

1. **AI CLI 是放大器，不是替代品**——它放大你的效率，但方向还是你来定
2. **Git 是你的安全网**——在 Git 仓库中使用 AI CLI，随时可以回滚
3. **从简单任务开始**——先用 AI CLI 做小任务，熟练后再挑战复杂自动化
4. **成本意识**——每次使用前想想：这个任务值得花多少 API 费用？

下一章，我们将探索 AI 辅助调试与性能优化，看看 AI 如何帮你更快地找到 Bug 和性能瓶颈。

← [-MCP 协议与 AI 工具集成](./14-MCP协议与AI工具集成.md) | [-AI Skill 与工作流优化](./16-AI-Skill与工作流优化.md) →
