# 16-AI Skill 与工作流优化

## 本章目标

- 理解 AI Skill 的概念，知道它和普通对话的区别
- 掌握 Trae Skills、Cursor Rules、Claude Code Custom Instructions 的使用方法
- 学会自定义 Skill / Rule，打造专属 AI 助手
- 建立完整的 AI 驱动 iOS 开发工作流
- 避开 AI 辅助开发中的常见陷阱

---

## 1. AI Skill 是什么

### 1.1 Skill 的概念

想象你在玩一款 RPG 游戏，角色可以学习各种"技能"——火球术、治愈术、隐身术……每个技能都有明确的触发方式、执行流程和预期效果。

AI Skill 也是一样：**它是一个预定义的专业能力模块**，告诉 AI "在什么条件下、按照什么流程、完成什么任务"。

| 类比 | RPG 技能 | AI Skill |
|------|----------|----------|
| 本质 | 角色学会的一种能力 | AI 学会的一种工作模式 |
| 触发 | 按下快捷键或选择菜单 | 输入 @skill 或自动推荐 |
| 执行 | 按固定动画和逻辑释放 | 按预设流程和模板执行 |
| 效果 | 造成伤害 / 治疗 / 增益 | 生成代码 / 审查代码 / 构建项目 |
| 升级 | 技能点加点 | 修改 Skill 配置文件 |

没有 Skill 的 AI，就像一个只会普通攻击的角色——能聊天、能写代码，但每次都要从头指导，效率低下。有了 Skill，AI 就像学会了"大招"，一键触发，专业输出。

### 1.2 Skill vs 普通对话

| 对比维度 | 普通对话 | AI Skill |
|----------|----------|----------|
| 结构化 | ❌ 自由散漫，每次可能不同 | ✅ 有固定模板和流程 |
| 可复用 | ❌ 每次都要重新描述需求 | ✅ 一次定义，反复使用 |
| 专业化 | ❌ 通用回答，可能不够深入 | ✅ 针对特定领域优化 |
| 一致性 | ❌ 不同对话结果差异大 | ✅ 输出格式和质量稳定 |
| 学习成本 | 低（直接说就行） | 中（需要了解 Skill 用法） |
| 效率 | 低（反复沟通） | 高（一键触发） |

> 💡 **一句话总结**：普通对话是"临时起意"，Skill 是"标准作业流程（SOP）"。当你发现自己反复对 AI 说同样的话时，就该用 Skill 了。

### 1.3 各平台 Skill 生态

目前主流 AI 编程工具都提供了类似 Skill 的机制，只是名字和形式不同：

| 平台 | 机制名称 | 文件格式 | 核心思路 |
|------|----------|----------|----------|
| **Trae** | Skills | `skill.md` | 结构化的能力模块，支持自动推荐和手动触发 |
| **Cursor** | Rules | `.cursorrules` / `.cursor/rules/` | 项目级规则文件，自动加载 |
| **Claude Code** | Custom Instructions | `CLAUDE.md` | 项目级指令文件，自动读取 |
| **GitHub Copilot** | Custom Instructions | `.github/copilot-instructions.md` | 仓库级指令，自动应用 |
| **Windsurf** | Rules | `.windsurfrules` | 项目级规则，类似 Cursor |

> ⚠️ 虽然各平台叫法不同，但核心思想一致：**把你的偏好和专业知识"写下来"，让 AI 每次都按规矩办事。**

---

## 2. Trae Skills 深度使用

### 2.1 内置 Skill 列表与功能介绍

Trae 内置了多个专业 Skill，覆盖开发全流程：

| Skill 名称 | 功能 | 适用场景 |
|------------|------|----------|
| `web-dev` | 快速生成生产级网页 | 需要搭建 Landing Page、原型页面 |
| `TRAE-code-review` | 代码审查 | 审查 PR、检查代码质量 |
| `asc-xcode-build` | Xcode 构建与归档 | 打包 IPA、管理版本号 |
| `asc-metadata-sync` | App Store 元数据同步 | 更新应用描述、关键词、本地化 |
| `asc-screenshot-resize` | 截图尺寸适配 | 生成各设备尺寸的 App Store 截图 |
| `find-skills` | 发现和安装新 Skill | 想找某个功能但不知道有没有 Skill |
| `skill-creator` | 创建自定义 Skill | 想自己开发一个 Skill |

### 2.2 Skill 调用方式

**方式一：@skill 触发**

在对话框中输入 `@` 然后选择 Skill：

```
@web-dev 帮我创建一个 iOS App 的 Landing Page
```

**方式二：自动推荐**

当你描述的任务匹配某个 Skill 时，Trae 会自动推荐：

```
你：帮我审查一下最近的代码改动
Trae：检测到代码审查任务，已自动加载 TRAE-code-review Skill
```

> 💡 自动推荐基于任务语义匹配，不需要你手动选择。但如果你发现 AI 没有自动推荐，可以手动用 `@skill` 触发。

### 2.3 常用 Skill 实战

#### 2.3.1 web-dev：快速生成网页

假设你的 iOS App 需要一个官网或隐私政策页面：

```
@web-dev 为我的习惯追踪 App "HabitFlow" 创建一个 Landing Page，
风格简洁现代，包含：Hero 区域、功能介绍、下载按钮、页脚
```

Trae 会自动：
1. 选择技术栈（HTML + Tailwind CSS）
2. 生成完整的页面代码
3. 启动本地预览服务器
4. 提供预览链接

#### 2.3.2 TRAE-code-review：代码审查

```
@TRAE-code-review 审查我最近的改动，重点关注性能和安全性
```

审查报告示例：

```markdown
## 代码审查报告

### 🔴 严重问题
- **内存泄漏**：`ProfileViewModel` 中的 `timer` 未在 `deinit` 中释放

### 🟡 建议改进
- **性能**：`fetchUsers()` 返回全量数据，建议加分页
- **安全**：API Key 硬编码在代码中，应移到环境变量

### 🟢 做得好的
- SwiftUI 视图拆分合理，职责清晰
- 错误处理使用了 Result 类型
```

#### 2.3.3 asc-xcode-build：Xcode 构建与归档

```
@asc-xcode-build 帮我构建 HabitFlow 项目并导出 IPA
```

Skill 会执行以下流程：

```
1. 读取项目配置 → 找到 .xcodeproj / .xcworkspace
2. 自动递增 Build Number → 42 → 43
3. 执行 xcodebuild archive → 生成 .xcarchive
4. 执行 xcodebuild -exportArchive → 生成 .ipa
5. 输出 IPA 路径 → /build/HabitFlow.ipa
```

#### 2.3.4 asc-metadata-sync：App Store 元数据同步

```
@asc-metadata-sync 同步 App Store Connect 的元数据到本地
```

同步的内容包括：

| 数据类型 | 说明 |
|----------|------|
| App 名称 | 各语言的应用名称 |
| 副标题 | App Store 搜索关键词 |
| 描述 | 应用详细描述 |
| 关键词 | ASO 搜索关键词 |
| What's New | 版本更新说明 |
| 截图信息 | 截图顺序和说明 |

#### 2.3.5 asc-screenshot-resize：截图尺寸适配

```
@asc-screenshot-resize 将这组截图适配所有设备尺寸
```

App Store 要求的设备截图尺寸：

| 设备 | 分辨率 | 比例 |
|------|--------|------|
| 6.9" (iPhone 16 Pro Max) | 1320 × 2868 | 需提供 |
| 6.7" (iPhone 15 Plus) | 1290 × 2796 | 需提供 |
| 6.5" (iPhone 14 Plus) | 1284 × 2778 | 可选 |
| 5.5" (iPhone 8 Plus) | 1242 × 2208 | 可选 |

> ⚠️ 6.9" 和 6.7" 的截图是必须提供的，缺少会导致提交被拒。asc-screenshot-resize Skill 会用 macOS 自带的 `sips` 工具自动缩放。

### 2.4 自定义 Skill 开发

当内置 Skill 满足不了你的需求时，可以自己开发。

#### Skill 的文件结构

一个 Skill 本质上就是一个 `skill.md` 文件，放在 `.trae/skills/` 目录下：

```
.trae/
└── skills/
    ├── my-custom-skill/
    │   └── skill.md        ← Skill 定义文件
    └── another-skill/
        └── skill.md
```

#### skill.md 的基本结构

```markdown
---
name: swiftui-component-generator
description: 根据 SwiftUI 设计规范，生成标准化的组件代码
triggers:
  - 生成 SwiftUI 组件
  - 创建 SwiftUI 视图
  - 新建 UI 组件
---

# SwiftUI 组件生成器

## 角色定义
你是一位资深的 SwiftUI 开发者，擅长编写可复用、可测试的 UI 组件。

## 执行流程

1. **确认需求**：询问组件名称、功能描述、设计风格
2. **生成代码**：按照以下模板生成组件
3. **生成预览**：提供 SwiftUI Preview 代码
4. **生成测试**：提供基本的单元测试

## 代码模板

\`\`\`swift
import SwiftUI

struct {{ComponentName}}: View {
    // MARK: - Properties
    
    // MARK: - Body
    var body: some View {
        // 实现代码
    }
}

// MARK: - Preview
#Preview {
    {{ComponentName}}()
}
\`\`\`

## 编码规范

- 使用 `// MARK: -` 分区
- 属性在前，body 在中，方法在后
- Preview 使用 #Preview 宏
- 颜色使用 Asset Catalog 引用
```

#### 关键字段说明

| 字段 | 必填 | 说明 |
|------|------|------|
| `name` | ✅ | Skill 的唯一标识，小写 + 短横线 |
| `description` | ✅ | 功能描述，用于自动推荐匹配 |
| `triggers` | ✅ | 触发关键词列表，用户输入匹配时自动推荐 |
| 角色定义 | ✅ | 告诉 AI 以什么身份工作 |
| 执行流程 | ✅ | 定义 AI 的工作步骤 |
| 代码模板 | ❌ | 提供输出模板，保证格式一致 |
| 编码规范 | ❌ | 约束 AI 的输出风格 |

> 💡 **开发 Skill 的核心思路**：把你每次都要重复告诉 AI 的话，写成一个文件。这样 AI 每次都会自动遵守，不用你反复提醒。

---

## 3. Cursor Rules 实战

### 3.1 .cursorrules 文件配置

Cursor Rules 是 Cursor 编辑器的"家规"——告诉 AI 在这个项目里必须遵守的规则。

**文件位置**：

```
项目根目录/
├── .cursorrules              ← 旧版单文件（仍然支持）
├── .cursor/
│   └── rules/                ← 新版多文件（推荐）
│       ├── swift-style.mdc
│       ├── architecture.mdc
│       └── naming.mdc
└── ...
```

> 💡 推荐使用 `.cursor/rules/` 目录方式，可以按主题拆分规则，管理更清晰。

**最简单的 .cursorrules 示例**：

```
本项目使用 SwiftUI 开发 iOS App。
请遵循 Swift 编码规范，使用 MVVM 架构。
所有新文件使用 Swift 6 语法。
```

### 3.2 iOS 项目常用 Rules

#### Rule 1：SwiftUI 编码规范

```markdown
---
description: SwiftUI 编码规范，适用于所有 SwiftUI 视图文件
globs: ["**/*.swift"]
---

# SwiftUI 编码规范

## 视图结构
- 每个视图文件只包含一个主视图结构体
- 视图拆分原则：单个 body 不超过 30 行
- 提取子视图时使用 `private func` 或独立结构体

## 属性顺序
1. 环境变量（@Environment）
2. 状态变量（@State, @StateObject）
3. 绑定变量（@Binding, @ObservedObject）
4. 传入参数（普通属性）
5. 常量

## 代码示例

```swift
struct ProfileView: View {
    // 1. Environment
    @Environment(\.dismiss) private var dismiss
    @EnvironmentObject private var appState: AppState
    
    // 2. State
    @State private var isEditing = false
    @StateObject private var viewModel = ProfileViewModel()
    
    // 3. Binding
    @Binding var selectedTab: Tab
    
    // 4. 参数
    let userName: String
    
    // 5. 常量
    private let avatarSize: CGFloat = 80
    
    var body: some View {
        // ...
    }
}
```

## 样式规范
- 间距使用 `Spacing` 常量，不硬编码数字
- 颜色使用 Asset Catalog，不硬编码色值
- 字体使用自定义 Typography 系统
- 圆角统一使用 `CornerRadius` 常量
```

#### Rule 2：项目架构约束

```markdown
---
description: 项目架构约束，确保代码分层清晰
globs: ["**/*.swift"]
---

# 项目架构约束

## 分层结构
```
App/
├── App/           ← 应用入口、路由
├── Features/      ← 功能模块（每个模块独立文件夹）
│   ├── Auth/
│   ├── Home/
│   └── Profile/
├── Core/          ← 核心服务（网络、存储、定位等）
├── Shared/        ← 共享组件（UI 组件、工具类）
└── Resources/     ← 资源文件（图片、颜色、字体）
```

## 模块规则
- Features 之间禁止直接依赖，通过 Router 解耦
- ViewModel 只能依赖 Core 层服务，不能依赖其他 ViewModel
- View 只能引用自己的 ViewModel，不能跨模块引用
- 网络请求统一通过 Repository 模式，不直接调用 URLSession

## 禁止事项
- ❌ 禁止在 View 中直接写网络请求
- ❌ 禁止在 ViewModel 中导入 SwiftUI
- ❌ 禁止使用单例模式（除 Core 层服务外）
- ❌ 禁止使用 NotificationCenter（使用 Combine 替代）
```

#### Rule 3：命名约定

```markdown
---
description: 命名约定，确保代码风格统一
globs: ["**/*.swift"]
---

# 命名约定

## 文件命名
- 视图：`XXXView.swift`（如 `LoginView.swift`）
- ViewModel：`XXXViewModel.swift`（如 `LoginViewModel.swift`）
- 模型：`XXX.swift` 或 `XXXModel.swift`（如 `User.swift`）
- 协议：`XXXProtocol.swift` 或 `XXXable.swift`
- 扩展：`Type+Extension.swift`（如 `View+Modifiers.swift`）

## 变量命名
- 布尔值用 `is`/`has`/`should` 前缀：`isLoading`、`hasError`
- 闭包用 `on` 前缀：`onTap`、`onDismiss`
- 私有属性加 `_` 前缀（仅 backing storage）：`_isLoading`
- 常量用 `static let`：`static let cornerRadius: CGFloat = 12`

## 函数命名
- 动词开头：`fetchUsers()`、`updateProfile()`、`deleteItem()`
- 配置函数用 `configure` 前缀：`configureWith(_:)`
- 工厂方法用 `make` 前缀：`makeURLRequest()`
```

### 3.3 Rules 最佳实践

| 实践 | 说明 | 示例 |
|------|------|------|
| **按主题拆分** | 不要把所有规则塞进一个文件 | swift-style.mdc、architecture.mdc、naming.mdc |
| **用 globs 限定范围** | 只在相关文件上生效 | `"globs": ["**/Features/**/*.swift"]` |
| **提供正反例** | 告诉 AI 什么该做、什么不该做 | ✅ 用 `@EnvironmentObject` / ❌ 用 `Singleton` |
| **保持简洁** | 规则太长 AI 会忽略 | 每个文件控制在 100 行以内 |
| **定期更新** | 随项目演进更新规则 | 新增模块时补充架构规则 |
| **团队统一** | 规则文件纳入 Git 版本控制 | 团队成员共享同一套规则 |

> ⚠️ **规则不是越多越好**。规则太多会导致 AI "注意力分散"，反而降低输出质量。建议核心规则不超过 5 个文件。

---

## 4. Claude Code Custom Instructions

### 4.1 CLAUDE.md 文件详解

Claude Code 是 Anthropic 官方的命令行 AI 编程工具。它通过 `CLAUDE.md` 文件来接收项目级指令——相当于给 Claude 的"入职手册"。

**文件查找顺序**：

```
1. ~/.claude/CLAUDE.md          ← 全局配置（所有项目生效）
2. 项目根目录/CLAUDE.md         ← 项目级配置（仅当前项目）
3. 子目录/CLAUDE.md             ← 目录级配置（仅该目录）
```

Claude Code 启动时会自动读取这些文件，你不需要手动指定。

### 4.2 iOS 项目 CLAUDE.md 模板

```markdown
# HabitFlow 项目指南

## 项目概述
- 类型：iOS 原生应用
- 最低版本：iOS 17.0
- 语言：Swift 6.0
- 框架：SwiftUI + Combine
- 架构：MVVM + Repository Pattern

## 技术栈
- UI：SwiftUI（不使用 UIKit）
- 网络：async/await + URLSession
- 存储：SwiftData
- 依赖管理：Swift Package Manager
- CI/CD：Xcode Cloud

## 项目结构
```
HabitFlow/
├── App/                    ← @main 入口、App 结构体
├── Features/               ← 功能模块
│   ├── Dashboard/          ← 首页仪表盘
│   ├── HabitList/          ← 习惯列表
│   ├── Statistics/         ← 数据统计
│   └── Settings/           ← 设置页面
├── Core/                   ← 核心服务
│   ├── Network/            ← 网络层
│   ├── Storage/            ← 数据持久化
│   └── Analytics/          ← 埋点统计
├── Shared/                 ← 共享组件
│   ├── Components/         ← UI 组件
│   ├── Extensions/         ← 扩展方法
│   └── Utils/              ← 工具类
└── Resources/              ← 资源文件
    ├── Assets.xcassets/    ← 图片和颜色
    └── Localization/       ← 多语言文件
```

## 编码规范
- 使用 Swift 6 严格并发模式
- 优先使用 async/await，不使用 Combine（除非跨组件通信）
- View 文件不超过 200 行，超过则拆分子视图
- 使用 #Preview 宏提供预览
- 注释使用中文，代码使用英文
- 错误处理使用自定义 Error 枚举

## 测试规范
- 单元测试覆盖 ViewModel 的所有公开方法
- UI 测试覆盖核心用户流程
- 测试文件与源文件同目录，命名为 `XXXTests.swift`

## Git 规范
- 分支命名：feature/xxx、fix/xxx、refactor/xxx
- Commit 格式：`type(scope): description`
- type 可选：feat、fix、refactor、test、docs、chore

## 注意事项
- 不要修改 Core/Network/ 下的文件，这是公共网络库
- 多语言文本必须使用 String Catalog，不要硬编码
- API 地址统一在 Core/Network/APIConfig.swift 中配置
```

### 4.3 项目级 vs 全局配置

| 配置类型 | 文件位置 | 适用场景 | 示例 |
|----------|----------|----------|------|
| **全局** | `~/.claude/CLAUDE.md` | 所有项目通用的偏好 | "回复使用中文"、"代码风格偏好" |
| **项目级** | `项目根/CLAUDE.md` | 项目特定的规则 | "本项目使用 MVVM"、"目录结构说明" |
| **目录级** | `子目录/CLAUDE.md` | 模块特定的约束 | "此模块禁止外部依赖" |

**全局配置示例**（`~/.claude/CLAUDE.md`）：

```markdown
# 全局偏好

- 回复使用中文
- 代码注释使用中文
- 优先使用 Swift 新特性（async/await、#Preview 等）
- 不使用第三方库，除非我明确要求
- 生成代码时同时生成对应的单元测试
```

> 💡 **分层原则**：全局配置放"个人偏好"，项目配置放"项目规则"，目录配置放"模块约束"。越具体的配置优先级越高。

---

## 5. AI 工作流优化

### 5.1 完整开发工作流

一个完整的 iOS App 开发流程，AI 可以在每个环节发挥作用：

```
📋 需求分析 → 🎨 UI 设计 → 💻 编码开发 → 🧪 测试验证 → 🔍 代码审查 → 📦 打包上架
     ↑                                                              ↓
     └──────────── 问题反馈 ←────────────────────────────────────────┘
```

### 5.2 每个环节的 AI 工具选择矩阵

| 开发环节 | 核心任务 | 推荐 AI 工具 | 推荐 Skill/Rule |
|----------|----------|-------------|-----------------|
| 需求分析 | 梳理功能点、写 PRD | Claude（长文能力强） | — |
| UI 设计 | 生成设计稿、配色方案 | ChatGPT + Figma AI | — |
| 编码开发 | 写代码、重构 | Cursor / Trae / Claude Code | SwiftUI 编码规范 Rule |
| 测试验证 | 写测试、找 Bug | Trae / Cursor | code-review Skill |
| 代码审查 | 检查质量、安全 | Trae | TRAE-code-review Skill |
| 打包上架 | 构建、截图、元数据 | Trae | asc-xcode-build / asc-metadata-sync / asc-screenshot-resize |

### 5.3 多工具协作：组合策略

没有哪个 AI 工具是万能的，组合使用效果最佳：

| 策略 | 工具组合 | 适用场景 | 工作方式 |
|------|----------|----------|----------|
| **主力 + 辅助** | Cursor（主力编码）+ Claude Code（架构设计） | 日常开发 | Cursor 写代码，Claude Code 做技术方案 |
| **双编辑器** | Trae（iOS 专项）+ Cursor（通用） | iOS 项目 | Trae 处理 Xcode 相关，Cursor 处理其他 |
| **终端 + 编辑器** | Claude Code（终端操作）+ Cursor（编辑器操作） | 复杂项目 | Claude Code 管理构建和脚本，Cursor 写业务代码 |
| **三件套** | Claude Code + Cursor + Copilot | 大型项目 | Claude Code 做架构，Cursor 做功能，Copilot 做补全 |

**实际协作流程示例**：

```
1. Claude Code  → 分析需求，输出技术方案（Spec）
2. Cursor       → 根据 Spec 编写核心代码
3. Copilot      → 在编码过程中提供行级补全
4. Trae         → 用 code-review Skill 审查代码
5. Trae         → 用 asc-xcode-build Skill 打包
6. Trae         → 用 asc-screenshot-resize + asc-metadata-sync 上架
```

### 5.4 效率提升量化

以一个真实的 iOS 小功能开发为例——"添加习惯标签功能"：

| 指标 | 传统开发 | AI 辅助开发 | 提升幅度 |
|------|----------|-------------|----------|
| 需求文档 | 2 小时 | 20 分钟 | **6x** |
| UI 实现 | 4 小时 | 1 小时 | **4x** |
| 网络层对接 | 3 小时 | 45 分钟 | **4x** |
| 单元测试 | 2 小时 | 30 分钟 | **4x** |
| 代码审查 | 1 小时 | 15 分钟 | **4x** |
| **总计** | **12 小时** | **2.5 小时** | **~5x** |

> ⚠️ 以上数据仅供参考，实际效率提升取决于：你对 AI 工具的熟练度、项目的复杂度、需求描述的清晰度。AI 不是万能的，但用好了确实能大幅提效。

---

## 6. iOS 开发 AI 工作流实战

### 6.1 需求阶段：AI 辅助需求分析

**目标**：把模糊的想法变成清晰的需求文档。

**Prompt 模板**：

```
我正在开发一个习惯追踪 App "HabitFlow"，现在需要添加一个"习惯标签"功能。
请帮我：
1. 梳理功能点，列出所有需要实现的子功能
2. 识别潜在的边界情况和异常场景
3. 输出一份结构化的需求文档（PRD）
```

**AI 输出示例**：

```markdown
## 习惯标签功能 PRD

### 功能概述
用户可以为习惯添加自定义标签，用于分类和筛选。

### 子功能列表
1. 标签创建：用户可创建自定义标签（名称 + 颜色）
2. 标签关联：创建/编辑习惯时可选择标签（支持多选）
3. 标签筛选：习惯列表页支持按标签筛选
4. 标签管理：设置页可查看、编辑、删除标签
5. 标签统计：统计页按标签聚合展示数据

### 边界情况
- 标签名称重复 → 提示"标签已存在"
- 标签被删除 → 关联的习惯自动解除标签
- 标签数量上限 → 限制最多 20 个标签
- 习惯无标签 → 显示"未分类"分组
```

### 6.2 设计阶段：Figma MCP → SwiftUI 代码

> 💡 MCP（Model Context Protocol）是让 AI 工具与外部服务交互的协议。Figma MCP 可以让 AI 直接读取 Figma 设计稿。

**工作流**：

```
Figma 设计稿 → AI 读取设计信息 → 生成 SwiftUI 代码
```

**实际操作**：

```
1. 在 Figma 中完成 UI 设计
2. 配置 Figma MCP Server（在 AI 工具中添加 MCP 配置）
3. 告诉 AI："根据 Figma 链接生成 SwiftUI 代码"
4. AI 读取设计稿的布局、颜色、字体信息
5. 输出对应的 SwiftUI 视图代码
```

**MCP 配置示例**（Trae / Claude Code）：

```json
{
  "mcpServers": {
    "figma": {
      "command": "npx",
      "args": ["-y", "@anthropic/figma-mcp-server"],
      "env": {
        "FIGMA_ACCESS_TOKEN": "your-figma-token"
      }
    }
  }
}
```

### 6.3 开发阶段：Spec 驱动 + AI 编码

**Spec 驱动开发**的核心思路：先写规格说明（Spec），再让 AI 根据规格写代码。

**Step 1：编写 Spec**

```markdown
## TagListView Spec

### 功能描述
展示习惯标签的水平滚动列表，支持选中/取消选中。

### 数据模型
- Input: `[Tag]`（标签数组）
- Output: `Set<Tag.ID>`（选中的标签 ID 集合）

### UI 规格
- 水平滚动，左右留 16pt padding
- 标签胶囊：圆角 16pt，内边距 8x16
- 未选中：浅灰底 + 深灰字
- 选中：主题色底 + 白字
- 超出屏幕时可横向滚动

### 交互
- 点击标签 → 切换选中状态
- 支持多选
```

**Step 2：让 AI 根据 Spec 编码**

```
根据上面的 TagListView Spec，用 SwiftUI 实现这个组件。
遵循项目的编码规范（见 CLAUDE.md）。
```

**Step 3：AI 输出代码**

```swift
import SwiftUI

struct TagListView: View {
    let tags: [Tag]
    @Binding var selectedTagIDs: Set<Tag.ID>

    var body: some View {
        ScrollView(.horizontal, showsIndicators: false) {
            HStack(spacing: 8) {
                ForEach(tags) { tag in
                    tagCapsule(tag)
                }
            }
            .padding(.horizontal, 16)
        }
    }

    private func tagCapsule(_ tag: Tag) -> some View {
        let isSelected = selectedTagIDs.contains(tag.id)

        return Button {
            toggleSelection(tag.id)
        } label: {
            Text(tag.name)
                .font(.subheadline)
                .fontWeight(.medium)
                .padding(.horizontal, 16)
                .padding(.vertical, 8)
                .background(isSelected ? Color.accentColor : Color(.systemGray5))
                .foregroundStyle(isSelected ? .white : Color(.systemGray))
                .clipShape(Capsule())
        }
    }

    private func toggleSelection(_ tagID: Tag.ID) {
        if selectedTagIDs.contains(tagID) {
            selectedTagIDs.remove(tagID)
        } else {
            selectedTagIDs.insert(tagID)
        }
    }
}

#Preview {
    struct PreviewWrapper: View {
        @State private var selectedIDs: Set<Tag.ID> = []

        var body: some View {
            TagListView(
                tags: Tag.samples,
                selectedTagIDs: $selectedIDs
            )
        }
    }

    return PreviewWrapper()
}
```

### 6.4 测试阶段：AI 生成测试 + 自动修复

**生成测试**：

```
为 TagListView 的 ViewModel（TagListViewModel）生成单元测试，
覆盖以下场景：
1. 初始状态没有选中标签
2. 选中一个标签后，selectedTagIDs 包含该标签
3. 取消选中后，selectedTagIDs 不包含该标签
4. 选中多个标签
```

**AI 输出测试代码**：

```swift
import Testing
@testable import HabitFlow

@Suite("TagListViewModel Tests")
struct TagListViewModelTests {

    @Test("初始状态没有选中标签")
    func initialNoSelection() {
        let viewModel = TagListViewModel()
        #expect(viewModel.selectedTagIDs.isEmpty)
    }

    @Test("选中一个标签")
    func selectOneTag() {
        let viewModel = TagListViewModel()
        let tagID = UUID()

        viewModel.toggleSelection(tagID)

        #expect(viewModel.selectedTagIDs.contains(tagID))
    }

    @Test("取消选中标签")
    func deselectTag() {
        let viewModel = TagListViewModel()
        let tagID = UUID()

        viewModel.toggleSelection(tagID)
        viewModel.toggleSelection(tagID)

        #expect(!viewModel.selectedTagIDs.contains(tagID))
    }

    @Test("选中多个标签")
    func selectMultipleTags() {
        let viewModel = TagListViewModel()
        let id1 = UUID()
        let id2 = UUID()

        viewModel.toggleSelection(id1)
        viewModel.toggleSelection(id2)

        #expect(viewModel.selectedTagIDs.count == 2)
        #expect(viewModel.selectedTagIDs.contains(id1))
        #expect(viewModel.selectedTagIDs.contains(id2))
    }
}
```

**自动修复**：如果测试失败，把错误信息发给 AI：

```
测试 "取消选中标签" 失败，错误信息：
  Expectation failed: !viewModel.selectedTagIDs.contains(tagID)
  实际值：selectedTagIDs 仍然包含 tagID

请修复 TagListViewModel 中的 toggleSelection 方法。
```

### 6.5 上架阶段：AI 辅助截图、描述、ASO

**截图生成**：

```
@asc-screenshot-resize 我有一组 6.9" iPhone 的截图，
请帮我生成 6.7" 和 5.5" 的版本。
截图路径：/Users/xxx/Screenshots/
```

**App Store 描述优化**：

```
请帮我优化 HabitFlow 的 App Store 描述，
目标用户：25-35 岁的职场人士，
核心卖点：简单易用、数据可视化、AI 智能提醒，
关键词：习惯追踪、自律、目标管理、打卡
```

**ASO 关键词优化**：

```
请为 HabitFlow 生成 App Store 关键词（100 字符限制），
参考竞品：Streaks、Habitica、Productive
```

| 优化项 | 优化前 | 优化后 |
|--------|--------|--------|
| 关键词 | 习惯,追踪,打卡 | 习惯追踪,自律打卡,目标管理,每日习惯,习惯养成,效率工具 |
| 副标题 | 习惯追踪 App | 养成好习惯，遇见更好的自己 |
| 描述开头 | HabitFlow 是一个习惯追踪应用 | 每天进步一点点，让好习惯成为你的超能力 |

---

## 7. 常见陷阱与解决方案

### 7.1 AI 幻觉问题

**什么是幻觉**：AI 自信地给出错误的信息，比如编造不存在的 API、捏造方法签名。

| 幻觉类型 | 示例 | 危害等级 |
|----------|------|----------|
| 编造 API | `ScrollView(.auto)` （不存在的参数） | 🔴 高 |
| 过时语法 | 使用 `NavigationView` 而非 `NavigationStack` | 🟡 中 |
| 捏造框架 | 引用不存在的 `SwiftUICharts` 模块名 | 🔴 高 |
| 错误参数 | `@State var items: [Item] = nil`（数组不能为 nil） | 🟡 中 |

**解决方案**：

| 策略 | 具体做法 |
|------|----------|
| **验证 API** | 让 AI 生成代码后，在 Xcode 中编译验证 |
| **指定版本** | 在 Rule 中明确："使用 iOS 17+ API，使用 NavigationStack 而非 NavigationView" |
| **要求引用** | 让 AI 标注 API 来源："请标注每个 API 的 Apple 文档链接" |
| **交叉验证** | 用不同 AI 工具验证同一份代码 |

### 7.2 上下文窗口限制

**问题**：AI 能"记住"的内容是有限的，对话太长会丢失早期信息。

| 模型 | 上下文窗口 | 大约相当于 |
|------|-----------|-----------|
| Claude 3.5 Sonnet | 200K tokens | ~150 页文档 |
| GPT-4o | 128K tokens | ~100 页文档 |
| Gemini 1.5 Pro | 1M tokens | ~750 页文档 |

**解决方案**：

| 策略 | 具体做法 |
|------|----------|
| **拆分对话** | 不同功能模块在不同对话中开发 |
| **使用 Rule/CLAUDE.md** | 把关键信息写在配置文件中，每次自动加载 |
| **定期总结** | 长对话中让 AI 总结当前进度，然后开新对话继续 |
| **精简输入** | 只提供相关代码，不要粘贴整个文件 |

### 7.3 代码一致性维护

**问题**：AI 在不同对话中可能生成风格不一致的代码。

| 不一致表现 | 示例 |
|------------|------|
| 命名风格 | 对话1用 `fetchData()`，对话2用 `loadData()` |
| 架构模式 | 对话1用 MVVM，对话2用 MVC |
| 错误处理 | 对话1用 `throws`，对话2用 `Result` |
| UI 风格 | 对话1用系统默认色，对话2用自定义色 |

**解决方案**：

```markdown
在 .cursorrules 或 CLAUDE.md 中明确：

## 命名约定
- 网络请求统一用 fetch 前缀：fetchUsers()、fetchProfile()
- 数据加载统一用 load 前缀：loadCache()、loadFromDisk()

## 架构模式
- 严格使用 MVVM，ViewModel 通过 @Observable 实现

## 错误处理
- 网络层使用 Result<T, Error>
- 业务层使用 throws + try await

## UI 风格
- 颜色统一使用 Asset Catalog 中定义的语义颜色
- 间距使用 Spacing 枚举中定义的常量
```

### 7.4 安全与隐私

**问题**：AI 可能无意中暴露敏感信息。

| 风险类型 | 示例 | 防护措施 |
|----------|------|----------|
| API Key 泄露 | 代码中硬编码 `sk-xxx` | 使用 `.env` 文件 + `.gitignore` |
| 用户数据泄露 | 把真实用户数据发给 AI | 只使用模拟数据（Mock Data） |
| 代码泄露 | 把公司私有代码发给云端 AI | 使用本地部署模型或企业版 |
| 依赖安全 | AI 推荐的第三方库有漏洞 | 使用前检查库的维护状态和 Star 数 |

**.gitignore 配置示例**：

```gitignore
# 敏感信息
.env
.env.local
*.secret
Config/Secrets.swift

# AI 工具配置（可能包含 Token）
.mcp.json
.claude/mcp.json
```

**安全 Rule 示例**：

```markdown
---
description: 安全规范，防止敏感信息泄露
globs: ["**/*.swift"]
---

# 安全规范

## 禁止事项
- ❌ 禁止在代码中硬编码 API Key、Secret、Token
- ❌ 禁止使用真实的用户数据做测试
- ❌ 禁止将 .env 文件提交到 Git
- ❌ 禁止使用 HTTP 协议，必须使用 HTTPS

## 必须遵守
- ✅ API Key 通过环境变量或 Config 文件读取
- ✅ 测试使用 Mock 数据
- ✅ 敏感信息存储在 Keychain 中
- ✅ 网络请求启用 Certificate Pinning
```

> ⚠️ **最重要的原则**：永远不要把真实的密钥、Token、密码发给 AI。AI 的对话可能被用于训练数据，敏感信息一旦发出就无法撤回。

---

## 小结

本章我们学习了：

| 主题 | 核心要点 |
|------|----------|
| **AI Skill 概念** | Skill 是预定义的专业能力模块，比普通对话更结构化、可复用、专业化 |
| **Trae Skills** | 内置多种实用 Skill（web-dev、code-review、asc-*），支持自定义开发 |
| **Cursor Rules** | 通过 .cursorrules 或 .cursor/rules/ 定义项目规则，自动约束 AI 行为 |
| **Claude Code CLAUDE.md** | 项目级指令文件，支持全局/项目/目录三级配置 |
| **AI 工作流** | Spec 驱动开发 + 多工具协作，效率可提升 3-5 倍 |
| **iOS 实战** | 需求→设计→编码→测试→上架，每个环节都有 AI 最佳实践 |
| **常见陷阱** | 幻觉、上下文限制、一致性、安全——都有对应的解决方案 |

**记住这个公式**：

```
高效 AI 开发 = 清晰的 Spec + 合适的 Skill/Rule + 多工具协作 + 人工验证
```

AI 是你的超级助手，但你才是那个做决策的人。用好 Skill 和 Rule，让 AI 按你的规矩办事，才能真正释放 AI 的生产力。

← [-AI CLI 工具与自动化](./15-AI-CLI工具与自动化.md) | [-AI Agent 自主编程代理](./17-AI-Agent自主编程代理.md) →
