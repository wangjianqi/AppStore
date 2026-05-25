# 36-SwiftUI 初体验：第一个项目

> 🎯 **本章目标**：理解 SwiftUI 的核心思想，学会在 Xcode 中创建 SwiftUI 项目，搞懂项目里每个文件的作用，并掌握实时预览功能。

---

## 声明式 vs 命令式 UI

在正式写代码之前，我们需要先理解一个关键概念：**声明式 UI** 和 **命令式 UI** 的区别。这决定了你用 SwiftUI 和 UIKit 写代码时的思维方式完全不同。

### 命令式：告诉计算机"怎么做"

命令式 UI 就像你在给一个完全不懂做菜的助手写操作手册——每一步都要说清楚：

```swift
// UIKit（命令式）—— 创建一个标签，你需要一步步告诉它怎么做
let label = UILabel()
label.text = "Hello, World!"
label.textColor = .red
label.font = UIFont.systemFont(ofSize: 24)
label.textAlignment = .center
view.addSubview(label)

// 然后还要手动设置约束（布局）
label.translatesAutoresizingMaskIntoConstraints = false
NSLayoutConstraint.activate([
    label.centerXAnchor.constraint(equalTo: view.centerXAnchor),
    label.centerYAnchor.constraint(equalTo: view.centerYAnchor)
])
```

### 声明式：告诉计算机"要什么"

声明式 UI 就像你在餐厅点菜——你只需要说"我要一份红烧肉"，不需要告诉厨师先放油还是先放盐：

```swift
// SwiftUI（声明式）—— 直接描述你要什么
Text("Hello, World!")
    .font(.system(size: 24))
    .foregroundColor(.red)
    .multilineTextAlignment(.center)
```

### 生活类比：点餐

| 场景 | 命令式（UIKit） | 声明式（SwiftUI） |
|------|----------------|------------------|
| 你对服务员说 | "请先拿一个盘子，然后把猪肉切成3厘米的块，锅中放两勺油，加热到七成热，放入冰糖20克炒色……" | "我要一份红烧肉" |
| 你关心的是 | 每一步怎么做 | 最终要什么 |
| 厨师换了做法 | 你的指令全废了，得重写 | 不用管，只要结果对就行 |
| 代码类比 | 手动创建控件 → 设置属性 → 添加到视图 → 设置约束 | 直接描述 UI 长什么样 |

💡 **提示**：SwiftUI 采用声明式，意味着你只需要描述"界面应该长什么样"，系统会自动帮你完成渲染和更新。这大大减少了代码量，也降低了出错概率。

### 对比代码示例

下面用同一个需求——"屏幕中间显示一段红色文字"——来对比两种方式：

```swift
// ❌ UIKit 方式（命令式）—— 约 15 行代码
class ViewController: UIViewController {
    override func viewDidLoad() {
        super.viewDidLoad()
        let label = UILabel()
        label.text = "欢迎使用 SwiftUI"
        label.textColor = .systemRed
        label.font = .preferredFont(forTextStyle: .title1)
        label.sizeToFit()
        label.center = view.center
        view.addSubview(label)
    }
}
```

```swift
// ✅ SwiftUI 方式（声明式）—— 约 5 行代码
struct ContentView: View {
    var body: some View {
        Text("欢迎使用 SwiftUI")
            .font(.title)
            .foregroundColor(.red)
    }
}
```

⚠️ **警告**：本教程全程使用 SwiftUI，不再涉及 UIKit。如果你之前学过 UIKit，请暂时放下"手动控制"的习惯，拥抱声明式思维。

---

## SwiftUI 工作原理

理解 SwiftUI 的工作原理，就像理解一辆汽车怎么跑起来——不需要知道每个零件的细节，但要知道"油门一踩，车就走了"这个基本因果关系。

### View 协议和 body 属性

SwiftUI 中，所有能看到的东西都是 **View**（视图）。`View` 是一个协议（可以理解为一个"模板"或"合同"），任何遵循这个协议的类型都必须提供一个 `body` 属性：

```swift
struct ContentView: View {
    var body: some View {
        Text("Hello, SwiftUI!")
    }
}
```

拆解一下：

| 部分 | 含义 |
|------|------|
| `struct ContentView` | 定义一个名为 ContentView 的结构体 |
| `: View` | 遵循 View 协议（签了合同） |
| `var body: some View` | 必须提供一个 body 属性，返回某种 View |
| `some View` | "某种视图"，Swift 会自动推断具体类型 |
| `Text("Hello, SwiftUI!")` | body 的具体内容——一段文字 |

💡 **提示**：`some View` 中的 `some` 是 Swift 5.1 引入的不透明返回类型。你不需要深究它，只要记住"body 必须返回一个视图"就行。

### 状态驱动 UI 更新

SwiftUI 的核心理念是：**UI 是状态的函数**。

```
状态（数据） → SwiftUI 自动计算 → UI（界面）
```

当状态（数据）发生变化时，SwiftUI 会自动重新计算 `body`，然后更新界面。你不需要手动刷新——就像 Excel 表格，改了数据，图表自动更新。

```swift
struct CounterView: View {
    @State var count = 0  // 状态：记录点击次数

    var body: some View {
        VStack {
            Text("点击次数：\(count)")  // UI 自动读取 count 的值
            Button("点我 +1") {
                count += 1  // 修改状态 → SwiftUI 自动刷新 UI
            }
        }
    }
}
```

上面的代码中：
1. `@State var count = 0` 定义了一个状态变量
2. `Text` 显示 `count` 的值
3. 点击按钮时 `count += 1`，SwiftUI 检测到状态变化，自动重新执行 `body`，界面更新

⚠️ **警告**：`@State` 只能用在结构体内部，且应该用 `private` 修饰。跨视图共享状态需要用 `@Binding`、`@ObservedObject` 等，后面章节会讲。

### 视图树概念

SwiftUI 会把你写的视图代码组织成一棵"视图树"：

```
VStack（垂直布局）
├── Text("标题")
├── HStack（水平布局）
│   ├── Image(systemName: "star.fill")
│   └── Text("5.0")
└── Button("购买") { ... }
```

- **根节点**：`body` 返回的最外层视图
- **子节点**：嵌套在里面的视图
- **叶子节点**：不再包含子视图的基础组件（Text、Image 等）

SwiftUI 通过遍历这棵树来渲染界面，当状态变化时，只更新树中受影响的部分，而不是全部重绘。

---

## 创建 SwiftUI 项目

### 新建项目步骤

跟着下面的步骤，一步一步来：

**第 1 步：打开 Xcode**

启动 Xcode，你会看到欢迎界面：

![](https://docs-assets.developer.apple.com/published/7dbb17089a/rendered2x-1604993744.png)

点击 **"Create a new Xcode project"**（创建新项目）。

💡 **提示**：如果欢迎界面没出现，可以按 `⇧ + ⌘ + N` 快捷键，或者菜单栏选择 File → New → Project。

**第 2 步：选择项目模板**

在模板选择界面：

1. 左侧选择 **iOS** 平台
2. 在 Application 区域选择 **App**
3. 点击 **Next**

**第 3 步：填写项目信息**

| 字段 | 填写内容 | 说明 |
|------|---------|------|
| Product Name | `MyFirstApp` | 项目名称，建议用英文，首字母大写 |
| Team | 选择你的开发者账号 | 没有可以选 None |
| Organization Identifier | `com.yourname` | 组织标识符，通常用反向域名 |
| Bundle Identifier | 自动生成 | `com.yourname.MyFirstApp`，App 的唯一标识 |
| Interface | **SwiftUI** ⚠️ | 一定要选 SwiftUI！ |
| Language | **Swift** ⚠️ | 一定要选 Swift！ |
| Storage | None | 暂时不需要数据存储 |

⚠️ **警告**：Interface 一定要选 **SwiftUI**，不要选 Storyboard！Language 一定要选 **Swift**，不要选 Objective-C！选错了后面代码完全不一样。

**第 4 步：选择保存位置**

选择一个你容易找到的文件夹，点击 **Create**。

💡 **提示**：建议勾选 **Create Git repository on my Mac**，这样 Xcode 会自动帮你做版本管理。

### 项目模板选择

Xcode 提供了多种项目模板：

| 模板 | 适用场景 | 我们是否使用 |
|------|---------|-------------|
| **App** | 普通 App 开发 | ✅ 我们选这个 |
| Document App | 文档编辑类 App（如文本编辑器） | ❌ |
| Game | 游戏开发（配合 SpriteKit/SceneKit） | ❌ |
| Augmented Reality App | AR 增强现实 App | ❌ |

---

## 项目结构详解

创建完项目后，左侧导航栏会出现以下文件结构：

```
MyFirstApp/
├── MyFirstApp.swift        ← App 入口
├── ContentView.swift       ← 主界面视图
├── Assets.xcassets/        ← 资源目录
├── Preview Content/        ← 预览资源
│   └── Preview Assets.xcassets
├── MyFirstApp.entitlements ← 权限配置
└── Info.plist (嵌入项目配置中) ← 项目配置
```

### App 入口

打开 `MyFirstApp.swift`，你会看到：

```swift
import SwiftUI

@main
struct MyFirstApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
    }
}
```

逐行解释：

| 代码 | 含义 |
|------|------|
| `import SwiftUI` | 导入 SwiftUI 框架，就像"引入工具箱" |
| `@main` | 标记这是程序的入口点，App 启动时从这里开始 |
| `struct MyFirstApp: App` | 遵循 App 协议的结构体 |
| `var body: some Scene` | 必须提供的 body 属性，返回一个"场景" |
| `WindowGroup` | 窗口组，管理 App 的主窗口 |
| `ContentView()` | 在窗口中显示 ContentView 视图 |

💡 **提示**：`@main` 是 Swift 5.3 引入的属性，替代了以前的 `@UIApplicationMain`。一个项目只能有一个 `@main` 标记。

⚠️ **警告**：不要删除或重命名这个文件中的 `@main` 标记，否则 App 无法启动。

### View 协议

打开 `ContentView.swift`，这是你的主界面：

```swift
import SwiftUI

struct ContentView: View {
    var body: some View {
        VStack {
            Image(systemName: "globe")
                .imageScale(.large)
                .foregroundStyle(.tint)
            Text("Hello, world!")
        }
        .padding()
    }
}

#Preview {
    ContentView()
}
```

逐行解释：

| 代码 | 含义 |
|------|------|
| `struct ContentView: View` | 定义一个遵循 View 协议的结构体 |
| `var body: some View` | 视图的内容描述（必须实现） |
| `VStack { ... }` | 垂直布局容器，子视图从上到下排列 |
| `Image(systemName: "globe")` | 显示系统图标（地球） |
| `.imageScale(.large)` | 图标放大 |
| `.foregroundStyle(.tint)` | 使用主题色 |
| `Text("Hello, world!")` | 显示文字 |
| `.padding()` | 添加内边距 |
| `#Preview { ContentView() }` | Xcode 预览宏 |

💡 **提示**：`View` 协议只有一个必须实现的要求——`body` 属性。只要你能写出 `var body: some View { ... }`，你就创建了一个自定义视图。

### Asset 目录

`Assets.xcassets` 是资源管理器，用来存放：

| 资源类型 | 说明 | 使用方式 |
|---------|------|---------|
| App Icon | App 图标（桌面显示的那个图标） | 自动识别 |
| 颜色 | 自定义颜色 | `Color("颜色名")` |
| 图片 | 本地图片资源 | `Image("图片名")` |

添加图片的步骤：
1. 在左侧导航栏点击 `Assets.xcassets`
2. 右键空白处 → Import → 选择图片
3. 或直接从 Finder 拖拽图片到右侧区域

💡 **提示**：建议使用 PDF 或 SVG 格式的矢量图，这样在任何屏幕尺寸下都清晰。也可以提供 1x/2x/3x 三种分辨率的 PNG。

### Info.plist

在 Xcode 14 及以后版本中，Info.plist 的内容已经整合到项目设置里了：

1. 点击左侧导航栏最顶部的项目名称
2. 选择 **Info** 标签页
3. 在这里可以配置：

| 配置项 | 说明 |
|-------|------|
| Bundle Identifier | App 的唯一标识 |
| Version | 版本号（如 1.0） |
| Build | 构建号（如 1） |
| Privacy - Camera Usage Description | 相机权限说明 |
| Privacy - Location Usage Description | 定位权限说明 |
| Supported interface orientations | 支持的屏幕方向 |

⚠️ **警告**：如果你的 App 需要访问相机、相册、定位等隐私功能，必须在 Info.plist 中添加对应的权限说明，否则 App 会直接崩溃！

---

## Preview 实时预览

Preview（预览）是 SwiftUI 最强大的开发体验之一——你不需要每次都运行模拟器，就能实时看到界面效果。

### 静态预览

在 `ContentView.swift` 文件底部，你会看到：

```swift
#Preview {
    ContentView()
}
```

这就是预览宏。当你在代码中修改了 `body` 的内容，预览会自动刷新，无需手动操作。

💡 **提示**：如果预览没有自动出现，按 `⌥ + ⌘ + ↵` 打开预览面板，或者点击编辑器右上角的 Canvas 按钮。

### 动态预览

静态预览只能看，不能交互。如果你想点击按钮、滚动列表，需要使用动态预览：

```swift
#Preview {
    ContentView()
}

// 或者更明确地指定为动态预览
#Preview {
    ContentView()
        .environment(\.locale, .init(identifier: "zh-CN"))
}
```

在预览面板中，点击右下角的 **"Play"** 按钮（▶️），预览就会进入可交互模式，你可以点击按钮、输入文字等。

⚠️ **警告**：动态预览的性能不如真机运行，复杂动画可能会卡顿。正式测试还是要在模拟器或真机上进行。

### 预览设备切换

你可以通过代码指定预览的设备、外观等：

```swift
// 指定 iPhone 15 Pro 预览
#Preview("iPhone 15 Pro") {
    ContentView()
        .previewDevice(PreviewDevice(rawValue: "iPhone 15 Pro"))
}

// 指定暗黑模式预览
#Preview("暗黑模式") {
    ContentView()
        .preferredColorScheme(.dark)
}

// 指定中文环境预览
#Preview("中文") {
    ContentView()
        .environment(\.locale, .init(identifier: "zh-Hans"))
}

// 同时预览多种配置
#Preview("浅色") {
    ContentView()
        .preferredColorScheme(.light)
}

#Preview("深色") {
    ContentView()
        .preferredColorScheme(.dark)
}
```

也可以在预览面板中快速切换：

| 操作 | 方法 |
|------|------|
| 切换设备型号 | 预览面板底部点击设备名称 → 选择设备 |
| 切换暗黑模式 | 预览面板底部点击颜色图标 🎨 |
| 切换语言 | 预览面板底部点击语言图标 |
| 缩放预览 | `⌘ +` 放大 / `⌘ -` 缩小 |

💡 **提示**：建议同时创建浅色和深色两个预览，这样写代码时可以同时检查两种模式下的显示效果。

---

## 小结

本章我们学习了：

| 知识点 | 要点 |
|-------|------|
| 声明式 vs 命令式 | 声明式告诉计算机"要什么"，命令式告诉"怎么做" |
| SwiftUI 工作原理 | View 协议 + body 属性 + 状态驱动更新 + 视图树 |
| 创建项目 | Xcode → App 模板 → Interface 选 SwiftUI → Language 选 Swift |
| 项目结构 | App 入口（@main）、视图（ContentView）、资源（Assets）、配置（Info.plist） |
| 实时预览 | #Preview 宏、静态/动态预览、设备/外观切换 |

恭喜你！🎉 你已经创建了自己的第一个 SwiftUI 项目，并且理解了项目的基本结构。下一章，我们将学习 SwiftUI 的基础组件，开始真正地构建界面！

← [-Swift 泛型与类型系统深入](../03-Swift语言基础/32-Swift泛型与类型系统深入.md) | [-基础组件：文本、图片与按钮](./37-基础组件.md) →
