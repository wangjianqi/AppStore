# Xcode Previews 与实时开发

> 🎯 **本章目标**：理解 Xcode Previews 的核心价值，掌握 #Preview 宏的使用方法，能够利用实时预览加速 SwiftUI 开发，学会多设备/多外观/多语言预览，以及 Previews 常见问题的排查。

---

## Xcode Previews 是什么

想象一下，你站在一面神奇的镜子前——你换一件衣服，镜子里的你立刻就穿上了新衣服，不需要走开再回来。Xcode Previews 就是这面"实时镜子"：你修改代码，界面立刻跟着变化，无需编译、运行、等待模拟器启动。

在传统的 iOS 开发流程中，验证一个 UI 修改需要经历"改代码 → 编译 → 启动模拟器 → 导航到对应页面 → 查看效果"这样漫长的循环。而 Previews 把这个循环缩短到了"改代码 → 看效果"，几乎零等待。

### Previews vs 传统模拟器运行

| 对比维度 | Xcode Previews | 传统模拟器运行 |
|---------|---------------|--------------|
| 启动速度 | 秒级，几乎即时 | 需要完整编译+启动，数十秒到数分钟 |
| 交互方式 | 可交互，但有限制 | 完整交互，接近真机 |
| 适用场景 | UI 开发、布局调试、多配置预览 | 完整功能测试、性能测试、复杂交互 |
| 资源占用 | 较低，仅渲染当前视图 | 较高，需要运行完整 App |
| 多配置预览 | 原生支持，同时显示多个配置 | 需要逐个切换模拟器 |
| 状态保持 | 每次代码修改可能重置状态 | 状态持续保持 |
| 网络请求 | 需要模拟数据 | 可发起真实请求 |

### 为什么 Previews 是 SwiftUI 开发的核心工作流

SwiftUI 采用声明式语法，视图的本质是"状态的函数"——给定输入状态，输出确定的界面。这种纯函数特性使得 SwiftUI 视图天然适合预览：只要提供一组输入，就能渲染出对应的界面。

Previews 让开发者可以：
- **逐个组件开发**：不需要运行整个 App，单独预览某个按钮、某个卡片
- **快速试错**：调整颜色、间距、字体，实时看到效果
- **多场景覆盖**：同时预览浅色/深色模式、不同设备尺寸、不同语言
- **文档化组件**：预览代码本身就是组件的使用示例

> 💡 **提示**：Previews 不是模拟器的替代品，而是互补工具。用 Previews 做 UI 开发和调试，用模拟器/真机做完整功能验证，两者结合才是最高效的工作流。

---

## #Preview 宏的使用

### 从 PreviewProvider 到 #Preview

在 Xcode 15 之前，SwiftUI 使用 `PreviewProvider` 协议来定义预览：

```swift
struct ContentView_Previews: PreviewProvider {
    static var previews: some View {
        ContentView()
    }
}
```

这种方式需要为每个预览创建一个独立的结构体，遵循 `PreviewProvider` 协议，代码较为冗余。从 Xcode 15 开始，Apple 引入了 `#Preview` 宏，大幅简化了预览的写法：

```swift
#Preview {
    ContentView()
}
```

两者对比：

| 特性 | PreviewProvider | #Preview 宏 |
|-----|----------------|------------|
| 语法复杂度 | 需要定义结构体+协议 | 一行宏即可 |
| 命名预览 | 通过组名间接实现 | 直接传入字符串 |
| 多预览 | 需要在 previews 中返回 Group | 多个 #Preview 并列 |
| 最低版本 | iOS 13+ | iOS 17+ / Xcode 15+ |
| 类型安全 | 需要手动匹配返回类型 | 宏自动推导 |

> ⚠️ **警告**：如果你的项目需要支持 iOS 16 及以下版本，仍然需要使用 `PreviewProvider`。`#Preview` 宏要求最低部署目标为 iOS 17。但预览代码不会被编译进最终产品，所以即使项目部署目标较低，也可以在开发时使用 `#Preview`（Xcode 15+ 会正确处理）。

### 基本用法

最简单的预览，直接包裹目标视图：

```swift
#Preview {
    ContentView()
}
```

### 命名预览

当有多个预览时，给每个预览起名字可以方便在 Xcode 的预览选择器中快速定位：

```swift
#Preview("默认状态") {
    TodoListView()
}

#Preview("空列表") {
    TodoListView(todos: [])
}

#Preview("深色模式") {
    TodoListView()
        .preferredColorScheme(.dark)
}
```

### 多个预览同时显示

在同一个文件中写多个 `#Preview`，Xcode 会在预览画布中同时显示它们。你可以横向滚动查看所有预览，也可以点击某个预览将其放大到全屏：

```swift
struct UserCardView: View {
    let name: String
    let role: String

    var body: some View {
        HStack {
            Circle()
                .fill(Color.blue)
                .frame(width: 50, height: 50)
            VStack(alignment: .leading) {
                Text(name)
                    .font(.headline)
                Text(role)
                    .font(.subheadline)
                    .foregroundStyle(.secondary)
            }
            Spacer()
        }
        .padding()
        .background(Color(.systemBackground))
        .cornerRadius(12)
        .shadow(radius: 2)
    }
}

#Preview("管理员") {
    UserCardView(name: "张三", role: "管理员")
}

#Preview("普通用户") {
    UserCardView(name: "李四", role: "普通用户")
}

#Preview("深色") {
    UserCardView(name: "王五", role: "审核员")
        .preferredColorScheme(.dark)
}
```

### 传递参数的预览

`#Preview` 宏支持直接传递参数，这在预览需要特定初始化参数的视图时特别有用：

```swift
#Preview("详情页") {
    NavigationStack {
        ItemDetailView(item: Item(
            title: "示例项目",
            description: "这是一个预览用的示例数据",
            isCompleted: false
        ))
    }
}
```

对于使用环境对象或环境值的视图，也可以在预览中注入：

```swift
#Preview {
    ContentView()
        .environmentObject(AppState())
        .environment(\.locale, .init(identifier: "zh-Hans"))
}
```

> 💡 **提示**：预览中传递的参数应该尽量简单、直观。如果构造一个真实对象需要很多步骤，可以创建专门的 Mock 工厂方法来简化预览代码。

---

## 实时预览与动态预览

### 静态预览 vs 实时预览

Xcode Previews 有两种模式：

| 模式 | 特点 | 适用场景 |
|-----|------|---------|
| 静态预览 | 只渲染界面，不支持交互 | 查看布局、颜色、字体 |
| 实时预览（Live Preview） | 支持点击、滑动、输入等交互 | 调试交互逻辑、动画效果 |

默认情况下，Previews 以静态模式启动。点击预览画布下方的 ▶️（Play）按钮，即可切换到实时预览模式。

### 如何启用实时预览

1. 在 Xcode 编辑器中打开带有 `#Preview` 的文件
2. 确保右侧的 Canvas（画布）已打开（快捷键 `⌥⌘↵`）
3. 点击画布左下角的 ▶️ 按钮，预览进入实时模式
4. 现在可以在预览中点击按钮、输入文本、滑动列表

### 在预览中交互

实时预览支持以下交互：

```swift
struct CounterView: View {
    @State private var count = 0

    var body: some View {
        VStack(spacing: 20) {
            Text("计数：\(count)")
                .font(.largeTitle)

            HStack(spacing: 20) {
                Button("减少") {
                    count -= 1
                }
                .buttonStyle(.bordered)

                Button("增加") {
                    count += 1
                }
                .buttonStyle(.borderedProminent)
            }
        }
        .padding()
    }
}

#Preview {
    CounterView()
}
```

在实时预览模式下，你可以点击"增加"和"减少"按钮，看到计数实时变化。这种即时反馈让调试交互逻辑变得非常高效。

### 旋转预览

使用 `.previewInterfaceOrientation()` 可以在预览中模拟设备旋转，无需手动旋转模拟器：

```swift
#Preview("竖屏") {
    LandscapeCardView()
}

#Preview("横屏") {
    LandscapeCardView()
        .previewInterfaceOrientation(.landscapeLeft)
}
```

可用的方向值包括：
- `.portrait` — 竖屏（默认）
- `.portraitUpsideDown` — 倒置竖屏
- `.landscapeLeft` — 横屏左
- `.landscapeRight` — 横屏右

### 预览中的动画效果

实时预览模式下，SwiftUI 的动画效果可以正常播放：

```swift
struct AnimationPreview: View {
    @State private var isExpanded = false

    var body: some View {
        Circle()
            .fill(isExpanded ? Color.orange : Color.blue)
            .frame(width: isExpanded ? 200 : 100, height: isExpanded ? 200 : 100)
            .animation(.easeInOut(duration: 0.5), value: isExpanded)
            .onTapGesture {
                isExpanded.toggle()
            }
    }
}

#Preview {
    AnimationPreview()
}
```

点击圆形即可在实时预览中看到颜色和大小的动画过渡效果。

> ⚠️ **警告**：实时预览中的动画播放速度可能与真机不同，不要依赖预览中的动画时长来做性能优化。动画的流畅度测试应该在真机上进行。

---

## 多设备、多外观、多语言预览

这是 Previews 最强大的能力之一——无需切换模拟器，就能同时看到同一个视图在不同配置下的表现。

### 指定预览设备

使用 `.previewDevice()` 可以指定预览渲染的目标设备：

```swift
#Preview("iPhone 15") {
    ContentView()
        .previewDevice("iPhone 15")
}

#Preview("iPhone 15 Pro Max") {
    ContentView()
        .previewDevice("iPhone 15 Pro Max")
}

#Preview("iPad") {
    ContentView()
        .previewDevice("iPad Pro (12.9-inch) (6th generation)")
}
```

设备名称需要与 Xcode 模拟器列表中的名称完全匹配。你可以在终端运行以下命令查看所有可用设备：

```bash
xcrun simctl list devices available | grep iPhone
```

### 深色模式预览

使用 `.preferredColorScheme()` 在预览中切换浅色/深色模式：

```swift
#Preview("浅色模式") {
    SettingsView()
        .preferredColorScheme(.light)
}

#Preview("深色模式") {
    SettingsView()
        .preferredColorScheme(.dark)
}
```

> 💡 **提示**：养成同时预览浅色和深色模式的习惯。很多 UI 问题只在深色模式下才会暴露，比如浅色文字在浅色背景上不可见。

### 多语言预览

使用 `.environment(\.locale, ...)` 可以预览不同语言下的界面表现，这对于检查本地化布局非常重要：

```swift
#Preview("中文") {
    ProductListView()
        .environment(\.locale, .init(identifier: "zh-Hans"))
}

#Preview("英文") {
    ProductListView()
        .environment(\.locale, .init(identifier: "en"))
}

#Preview("阿拉伯语（RTL）") {
    ProductListView()
        .environment(\.locale, .init(identifier: "ar"))
}
```

阿拉伯语等 RTL（从右到左）语言的预览特别重要——SwiftUI 会自动翻转布局方向，但你需要确认翻转后的效果是否符合预期。

### 动态字体大小预览

使用 `.dynamicTypeSize()` 预览不同辅助功能字体大小下的界面表现：

```swift
#Preview("默认大小") {
    ArticleView()
}

#Preview("超大字体") {
    ArticleView()
        .dynamicTypeSize(.xxxLarge)
}

#Preview("极小字体") {
    ArticleView()
        .dynamicTypeSize(.xSmall)
}
```

### 批量生成多配置预览的技巧

当需要同时预览多种配置时，可以创建辅助函数来批量生成预览：

```swift
struct PreviewHelper {
    static func multiPreview(_ view: some View) -> some View {
        view
    }
}

#Preview("iPhone 浅色") {
    ContentView()
        .previewDevice("iPhone 15")
        .preferredColorScheme(.light)
}

#Preview("iPhone 深色") {
    ContentView()
        .previewDevice("iPhone 15")
        .preferredColorScheme(.dark)
}

#Preview("iPad 浅色") {
    ContentView()
        .previewDevice("iPad Pro (12.9-inch) (6th generation)")
        .preferredColorScheme(.light)
}
```

对于更复杂的批量预览需求，可以使用 `PreviewProvider` 的 `Group` 方式（即使主要使用 `#Preview`，这种旧方式在批量场景下仍然有用）：

```swift
struct ContentView_AllPreviews: PreviewProvider {
    static var previews: some View {
        Group {
            ContentView()
                .previewDevice("iPhone 15")
                .previewDisplayName("iPhone 15")

            ContentView()
                .previewDevice("iPhone 15")
                .preferredColorScheme(.dark)
                .previewDisplayName("iPhone 15 深色")

            ContentView()
                .previewDevice("iPad Pro (12.9-inch) (6th generation)")
                .previewDisplayName("iPad Pro")
        }
    }
}
```

> 💡 **提示**：批量预览虽然方便，但过多的预览会拖慢 Xcode 性能。建议日常开发只保留 2-3 个最常用的预览配置，提交代码前再临时添加完整的多配置预览进行检查。

---

## 预览中注入数据与依赖

SwiftUI 视图通常依赖外部数据（环境对象、环境值、绑定等）。在预览中，这些依赖需要手动注入，否则视图可能无法正常渲染甚至崩溃。

### 注入环境对象

使用 `.environmentObject()` 在预览中注入 `@EnvironmentObject` 依赖：

```swift
class CartViewModel: ObservableObject {
    @Published var items: [String] = ["商品A", "商品B", "商品C"]
    @Published var totalPrice: Double = 299.0
}

struct CartView: View {
    @EnvironmentObject var cart: CartViewModel

    var body: some View {
        List(cart.items, id: \.self) { item in
            Text(item)
        }
        .navigationTitle("购物车（¥\(cart.totalPrice)）")
    }
}

#Preview {
    NavigationStack {
        CartView()
            .environmentObject(CartViewModel())
    }
}
```

### 注入环境值

使用 `.environment()` 注入 `@Environment` 依赖：

```swift
struct ThemeableView: View {
    @Environment(\.colorScheme) var colorScheme
    @Environment(\.locale) var locale

    var body: some View {
        Text("当前语言：\(locale.identifier)")
            .foregroundStyle(colorScheme == .dark ? .white : .black)
            .padding()
            .background(colorScheme == .dark ? Color.black : Color.white)
            .cornerRadius(8)
    }
}

#Preview("中文浅色") {
    ThemeableView()
        .environment(\.locale, .init(identifier: "zh-Hans"))
}

#Preview("英文深色") {
    ThemeableView()
        .preferredColorScheme(.dark)
        .environment(\.locale, .init(identifier: "en"))
}
```

### 模拟网络数据

预览中不应发起真实网络请求。正确做法是创建 Mock 数据，让视图以为数据来自网络：

```swift
struct Article: Identifiable {
    let id: UUID
    let title: String
    let author: String
    let content: String
}

struct ArticleListView: View {
    let articles: [Article]

    var body: some View {
        List(articles) { article in
            VStack(alignment: .leading) {
                Text(article.title)
                    .font(.headline)
                Text(article.author)
                    .font(.subheadline)
                    .foregroundStyle(.secondary)
            }
        }
        .listStyle(.plain)
    }
}

extension Article {
    static let mockData: [Article] = [
        Article(id: UUID(), title: "SwiftUI 入门指南", author: "张三", content: "内容..."),
        Article(id: UUID(), title: "Xcode Previews 详解", author: "李四", content: "内容..."),
        Article(id: UUID(), title: "iOS 18 新特性", author: "王五", content: "内容...")
    ]
}

#Preview("有数据") {
    NavigationStack {
        ArticleListView(articles: Article.mockData)
    }
}

#Preview("空列表") {
    NavigationStack {
        ArticleListView(articles: [])
    }
}
```

> 💡 **提示**：为每个模型创建 `mockData` 静态属性是一个好习惯。这不仅方便预览，也方便单元测试。建议将 Mock 数据放在单独的文件中，使用 `#if DEBUG` 编译标记包裹，确保不会进入生产代码。

### 预览与 @Bindable、@Observable 的配合

iOS 17 引入的 `@Observable` 宏和 `@Bindable` 属性包装器在预览中使用也很方便：

```swift
@Observable
class FormState {
    var username: String = ""
    var email: String = ""
    var isAgreed: Bool = false
}

struct RegistrationFormView: View {
    @Bindable var state: FormState

    var body: some View {
        Form {
            TextField("用户名", text: $state.username)
            TextField("邮箱", text: $state.email)
            Toggle("同意条款", isOn: $state.isAgreed)
        }
    }
}

#Preview {
    RegistrationFormView(state: FormState())
}
```

注意 `@Bindable` 需要配合 `@Observable` 使用，在预览中直接创建 `@Observable` 类的实例即可。

---

## Previews 常见问题与排查

### 预览崩溃的常见原因与解决

| 崩溃原因 | 表现 | 解决方案 |
|---------|------|---------|
| 缺少 EnvironmentObject | 红色错误屏幕，提示找不到环境对象 | 在预览中添加 `.environmentObject()` |
| 强制解包为 nil | 预览白屏或崩溃 | 避免强制解包，使用 `if let` 或提供默认值 |
| 无限循环渲染 | 预览卡死，CPU 100% | 检查 `body` 中是否修改了触发重渲染的 `@State` |
| 资源文件缺失 | 图片/颜色不显示 | 确认资源在 Asset Catalog 中且 Target Membership 正确 |
| 第三方库初始化失败 | 预览启动时崩溃 | 在预览中跳过或 Mock 第三方库的初始化 |

### 预览不更新的排查步骤

当你修改了代码但预览没有更新时，按以下步骤排查：

1. **检查预览是否暂停** — 画布右上角是否有暂停图标，点击恢复
2. **手动刷新** — 按 `⌥⌘P` 强制刷新预览
3. **清理构建缓存** — `Product → Clean Build Folder`（`⇧⌘K`），然后重新构建
4. **重启预览进程** — 在画布中右键点击，选择"Restart"
5. **删除 Derived Data** — `Xcode → Settings → Locations → Derived Data`，删除对应项目的文件夹
6. **重启 Xcode** — 如果以上都不行，重启 Xcode 通常能解决问题

### 预览与真机行为不一致的情况

预览并非完美的模拟器，以下情况可能出现行为差异：

- **动画时序**：预览中的动画速度可能与真机不同
- **安全区域**：预览可能不会正确显示状态栏和底部安全区域
- **键盘行为**：预览中的键盘弹出/收起行为与真机不同
- **手势识别**：复杂手势在预览中可能无法正确触发
- **导航转场**：`NavigationStack` 的转场动画在预览中可能不显示
- **系统字体**：预览使用 Mac 字体渲染，与 iOS 设备字体可能有细微差异

> ⚠️ **警告**：永远不要只依赖预览来验证 UI。在提交代码前，务必在模拟器或真机上做最终确认。预览是加速开发的工具，不是替代测试的工具。

### 大型项目预览加速技巧

在大型项目中，预览可能会变慢。以下技巧可以帮助加速：

1. **预览最小视图单元**：不要预览整个页面，只预览你正在开发的组件
2. **减少预览数量**：日常开发只保留 1-2 个预览，检查时再添加更多
3. **简化依赖注入**：预览中使用轻量级 Mock 对象，而非完整的真实对象
4. **使用 `#if DEBUG`**：将预览代码包裹在 `#if DEBUG` 中，减少 Release 构建时间
5. **关闭不需要的预览**：在画布中点击预览的关闭按钮，减少同时渲染的视图数量
6. **排除无关文件**：在预览文件中只 import 必要的模块

```swift
#if DEBUG
#Preview {
    MyComponent()
        .environmentObject(MockAppState())
}
#endif
```

---

## AI 辅助 + Previews 工作流

### 用 AI 生成代码后立即在 Preview 中验证

AI 编程工具（如 Claude Code、Cursor、Trae）生成的 SwiftUI 代码，最理想的验证方式就是通过 Previews。工作流如下：

1. 向 AI 描述你想要的 UI 组件
2. AI 生成 SwiftUI 视图代码和对应的 `#Preview`
3. 保存文件，Xcode 自动渲染预览
4. 在预览中检查视觉效果是否符合预期
5. 如果不满意，向 AI 提出修改要求，重复步骤 2-4

这种方式的优势在于：**你不需要花时间手动编写 UI 代码，也不需要等待模拟器启动来验证结果**。AI 负责生成代码，Previews 负责即时验证，形成高效的闭环。

### Spec 驱动开发中的 Preview 使用

在 Spec 驱动开发流程中，Previews 扮演着"可视化验收"的角色：

1. **编写 Spec**：用自然语言描述组件的功能和外观要求
2. **AI 根据 Spec 生成代码**：包括视图实现和预览代码
3. **Preview 验证**：通过预览快速检查是否满足 Spec 要求
4. **迭代修正**：不满足要求的部分，更新 Spec 或直接要求 AI 修正

示例 Spec 片段：

```
组件：UserCardView
要求：
- 显示用户头像（圆形，50x50）、用户名（headline 字体）、角色标签（subheadline，灰色）
- 浅色模式：白色背景，深色文字
- 深色模式：深灰背景，浅色文字
- 圆角 12pt，阴影 2pt
- 需要同时预览浅色和深色模式
```

AI 根据这个 Spec 生成的代码会包含两个 `#Preview`，分别展示浅色和深色模式，你可以立即在 Xcode 中验证。

### 快速迭代循环：AI 生成 → Preview 验证 → AI 修正

这是 AI 时代最高效的 SwiftUI 开发循环：

```
┌──────────────────────────────────────────┐
│                                          │
│  ┌─────────┐    ┌──────────┐    ┌─────┐ │
│  │ AI 生成  │───→│ Preview  │───→│ 检查 │ │
│  │ 代码     │    │ 即时渲染  │    │ 结果 │ │
│  └─────────┘    └──────────┘    └─────┘ │
│       ↑                              │   │
│       │         ┌──────────┐         │   │
│       └─────────│ AI 修正   │←────────┘   │
│                 │ 代码     │              │
│                 └──────────┘              │
│                                          │
└──────────────────────────────────────────┘
```

每一轮迭代可能只需要几秒到几十秒，相比传统的"改代码 → 编译 → 运行 → 检查"循环（可能需要几分钟），效率提升了一个数量级。

> 💡 **提示**：在使用 AI 辅助开发时，建议在 Prompt 中明确要求 AI 同时生成 `#Preview` 代码。这样你可以直接保存文件就看到效果，而不需要自己手动添加预览代码。例如："请生成 UserCardView 的 SwiftUI 代码，并包含浅色和深色模式的 #Preview。"

---

## 小结

本章详细介绍了 Xcode Previews 的核心概念和使用方法。以下是知识点总结：

| 知识点 | 关键内容 |
|-------|---------|
| Previews 核心概念 | 实时镜子，改代码即看效果，SwiftUI 声明式语法的天然搭档 |
| PreviewProvider vs #Preview | 旧方式需定义结构体，新 `#Preview` 宏一行搞定（需 Xcode 15+） |
| 命名预览 | `#Preview("名称") { ... }`，方便在多个预览中快速定位 |
| 多预览同时显示 | 同一文件写多个 `#Preview`，Xcode 自动并排显示 |
| 实时预览 | 点击 ▶️ 启用，支持点击、滑动、输入等交互 |
| 旋转预览 | `.previewInterfaceOrientation(.landscapeLeft)` |
| 动画预览 | 实时预览模式下可播放动画，但速度可能与真机不同 |
| 多设备预览 | `.previewDevice("iPhone 15")`，设备名需与模拟器列表匹配 |
| 深色模式预览 | `.preferredColorScheme(.dark)` |
| 多语言预览 | `.environment(\.locale, .init(identifier: "zh-Hans"))` |
| 动态字体预览 | `.dynamicTypeSize(.xxxLarge)` |
| 注入环境对象 | `.environmentObject()` |
| 注入环境值 | `.environment()` |
| Mock 数据 | 创建静态 `mockData` 属性，用 `#if DEBUG` 包裹 |
| @Observable 配合 | 预览中直接创建 `@Observable` 类实例，配合 `@Bindable` |
| 预览崩溃排查 | 检查环境对象、强制解包、无限循环、资源文件 |
| 预览不更新 | 刷新（⌥⌘P）→ 清理缓存 → 删除 Derived Data → 重启 Xcode |
| 真机差异 | 动画时序、安全区域、键盘、手势、转场动画可能不一致 |
| 预览加速 | 预览最小组件、减少数量、轻量 Mock、`#if DEBUG` |
| AI + Previews | AI 生成代码 → Preview 验证 → AI 修正，形成高效闭环 |
| Spec 驱动开发 | Spec → AI 生成 → Preview 验收 → 迭代修正 |

← [状态管理](./状态管理.md) | [实战①：完成「待办清单」App](./实战待办清单App.md) →
