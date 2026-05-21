# 121-visionOS 入门

## 本章目标

- 理解 visionOS 是什么，以及 Apple Vision Pro 的定位
- 掌握空间计算的核心概念：空间 UI、Volume、沉浸式空间
- 学会搭建 visionOS 开发环境并创建第一个项目
- 了解空间 UI 基础组件：WindowGroup、Volumetric 窗口、Ornament
- 初步掌握 3D 内容的展示方式（SceneKit / RealityKit / Model3D）
- 了解 visionOS 的手势与交互模式
- 知道如何将 iOS App 适配到 visionOS
- 了解发布到 visionOS App Store 的流程

---

## 1. visionOS 是什么

### 1.1 Apple 的空间计算平台

2023 年 6 月，Apple 在 WWDC 上发布了 **Apple Vision Pro**——一款革命性的空间计算设备，同时推出了全新的操作系统 **visionOS**。

> 💡 **生活类比**：如果说 iPhone 把互联网装进了口袋，那 Vision Pro就是把整个数字世界"搬"到了你眼前。你不再低头看屏幕，而是抬头看空间——App 悬浮在你的房间里，就像把虚拟的物品摆在了真实世界中。

### 1.2 visionOS vs 其他 Apple 平台

| 特性 | iOS | iPadOS | macOS | visionOS |
|------|-----|--------|-------|----------|
| 设备 | iPhone | iPad | Mac | Apple Vision Pro |
| 交互方式 | 触摸 | 触摸 + Apple Pencil | 键鼠 / 触控板 | 注视 + 手势 |
| 显示方式 | 二维屏幕 | 二维屏幕 | 二维屏幕 | 三维空间 |
| 输入方式 | 触控 | 触控 | 键盘鼠标 | 眼睛 + 手指 |
| 沉浸感 | 无 | 无 | 无 | 完全沉浸 |

### 1.3 Apple Vision Pro 核心硬件

| 硬件组件 | 说明 |
|----------|------|
| Micro-OLED 显示屏 | 双眼合计 2300 万像素，单眼超过 4K |
| M2 芯片 | 负责运行 App 和系统 |
| R1 芯片 | 专门处理传感器数据，12 毫秒光子到光子延迟 |
| 12 个摄像头 | 环境感知、手势追踪、眼动追踪 |
| 5 个传感器 | LiDAR、深度等 |
| 6 个麦克风 | 语音输入、空间音频 |

> ⚠️ **注意**：开发 visionOS App **不需要**购买 Vision Pro 设备！你可以使用 Mac + Xcode 中的 Vision Pro 模拟器进行开发和调试。

---

## 2. 空间计算概念

### 2.1 什么是空间计算

**空间计算**（Spatial Computing）是一种让数字内容与物理世界融合的技术。在 visionOS 中，你的房间就是画布，App 可以像真实物品一样存在于你的空间中。

> 💡 **生活类比**：想象你在客厅里摆了一台虚拟电视、一面虚拟白板、一个虚拟的地球仪——它们就像真实物品一样"放在"你的房间里，你可以绕着地球仪走一圈从不同角度观察它。

### 2.2 三种空间呈现方式

visionOS 提供了三种呈现内容的方式：

| 类型 | 说明 | 类比 |
|------|------|------|
| **Windows** | 平面窗口，类似 iPad 窗口 | 像在空中挂了一块 iPad 屏幕 |
| **Volumes** | 3D 体积窗口，有深度 | 像在桌上放了一个透明的展示柜 |
| **Full Spaces** | 完全沉浸式空间 | 像整个房间变成了另一个世界 |

```
┌─────────────────────────────────────────────┐
│              你的真实房间                      │
│                                             │
│    ┌──────────┐                             │
│    │  Window  │  ← 平面窗口（2D 内容）        │
│    │  (Safari)│                             │
│    └──────────┘                             │
│                                             │
│         ┌─────┐                             │
│        /  3D  \                             │
│       │ Earth │  ← Volume（3D 内容）          │
│        \     /                              │
│         └─────┘                             │
│                                             │
│  ╔═══════════════════════════════════╗       │
│  ║     Full Space (沉浸式空间)       ║       │
│  ║     整个视野被替换                ║       │
│  ╚═══════════════════════════════════╝       │
└─────────────────────────────────────────────┘
```

### 2.3 空间中的坐标系统

visionOS 使用右手坐标系：

| 轴 | 方向 | 说明 |
|----|------|------|
| X 轴 | 左右 | 右为正 |
| Y 轴 | 上下 | 上为正 |
| Z 轴 | 前后 | 朝向用户为正 |

> 💡 **记忆技巧**：伸出右手——拇指朝右（X），食指朝上（Y），中指朝自己（Z），这就是 visionOS 的坐标系。

---

## 3. 开发环境准备

### 3.1 硬件要求

| 要求 | 最低配置 |
|------|----------|
| Mac 电脑 | Apple Silicon（M1 及以上）或 Intel Mac（部分功能受限） |
| 内存 | 建议 16GB 及以上 |
| 存储空间 | Xcode 约 12GB + 模拟器约 5GB |
| 操作系统 | macOS Sonoma 14.0 及以上 |

> ⚠️ **重要**：Intel Mac 可以开发 visionOS App，但无法运行 Vision Pro 模拟器。强烈推荐使用 Apple Silicon Mac。

### 3.2 安装 Xcode 和 visionOS SDK

1. 从 Mac App Store 安装 **Xcode 15.2** 或更高版本
2. 打开 Xcode → 菜单栏 → **Xcode → Settings…（设置）→ Platforms**
3. 点击 **"+"** → 选择 **visionOS** → 下载安装

```bash
# 验证 visionOS SDK 是否安装成功
xcodebuild -showsdks | grep visionOS
# 输出应包含：visionOS x.x
```

### 3.3 Vision Pro 模拟器

安装完 visionOS SDK 后，你可以在 Xcode 中启动 Vision Pro 模拟器：

1. Xcode 菜单栏 → **Xcode → Settings… → Platforms**
2. 下载 **visionOS Simulator Runtime**
3. 运行项目时选择 **Vision Pro** 模拟器即可

| 模拟器功能 | 支持情况 |
|------------|----------|
| 运行 visionOS App | ✅ 支持 |
| 2D 窗口展示 | ✅ 支持 |
| 3D Volume 展示 | ✅ 支持 |
| 手势模拟（注视 + 捏合） | ✅ 支持（鼠标模拟） |
| 眼动追踪模拟 | ✅ 支持（鼠标模拟注视方向） |
| 真实环境透视 | ❌ 不支持 |
| ARKit 功能 | ⚠️ 部分支持 |

> 💡 **模拟器操作提示**：
> - **鼠标移动** = 眼睛注视方向
> - **鼠标左键点击** = 捏合手势（tap）
> - **鼠标拖拽** = 拖拽手势
> - **鼠标右键拖拽** = 旋转视角

---

## 4. 创建第一个 visionOS 项目

### 4.1 在 Xcode 中创建项目

1. 打开 Xcode → **File → New → Project…**
2. 选择 **visionOS** 标签页
3. 选择 **App** 模板 → 点击 Next

| 配置项 | 建议值 | 说明 |
|--------|--------|------|
| Product Name | HelloVisionOS | 项目名称 |
| Team | 你的开发者账号 | 没有可以先选 None |
| Organization Identifier | com.yourname | 反向域名 |
| Bundle Identifier | com.yourname.HelloVisionOS | 自动生成 |
| Initial Scene | Window | 初始场景类型 |
| Interface | SwiftUI | visionOS 推荐使用 SwiftUI |
| Language | Swift | 开发语言 |
| Storage | None | 暂不需要数据持久化 |

> 💡 **Initial Scene 选择**：
> - **Window**：创建一个平面窗口 App（类似 iPad App）
> - **Volume**：创建一个 3D 体积窗口 App
> - 首次学习建议选 **Window**，后面再尝试 Volume

### 4.2 项目结构

创建完成后，项目结构如下：

```
HelloVisionOS/
├── HelloVisionOSApp.swift    ← App 入口
├── ContentView.swift          ← 主视图
├── Assets.xcassets/           ← 资源文件
└── Info.plist                 ← 配置信息（部分在 Target Settings 中）
```

### 4.3 App 入口文件

```swift
import SwiftUI

@main
struct HelloVisionOSApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
    }
}
```

> 💡 **关键理解**：`WindowGroup` 是 visionOS App 的默认场景容器。它会在空间中创建一个平面窗口来显示你的 SwiftUI 视图。这就像在空中打开了一块 iPad 屏幕。

### 4.4 主视图文件

```swift
import SwiftUI

struct ContentView: View {
    var body: some View {
        VStack {
            Text("Hello, visionOS!")
                .font(.title)
            Text("欢迎来到空间计算的世界 🌍")
                .font(.subheadline)
                .foregroundStyle(.secondary)
        }
        .padding()
    }
}
```

### 4.5 运行项目

1. 在 Xcode 顶部选择 **Vision Pro** 模拟器
2. 点击 **Run（▶️）** 或按 **Cmd + R**
3. 等待模拟器启动，你的第一个 visionOS App 就会在空间中显示！

> ⚠️ **首次启动模拟器可能较慢**，需要等待 1-2 分钟。后续启动会快很多。

---

## 5. 空间 UI 基础

### 5.1 WindowGroup —— 平面窗口

`WindowGroup` 是最基础的场景类型，创建一个悬浮在空间中的平面窗口：

```swift
@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
    }
}
```

| WindowGroup 特性 | 说明 |
|------------------|------|
| 形状 | 矩形平面 |
| 深度 | 无（2D） |
| 可移动 | ✅ 用户可以拖动位置 |
| 可缩放 | ✅ 用户可以调整大小 |
| 默认大小 | 由内容决定 |

### 5.2 Volumetric 窗口 —— 3D 体积

当你需要展示 3D 内容时，使用 **Volume** 类型的窗口：

```swift
@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .defaultSize(width: 0.5, height: 0.5, depth: 0.5, in: .meters)

        VolumeGroup {
            Model3DView()
        }
    }
}
```

或者使用修饰符将 WindowGroup 标记为 Volume：

```swift
WindowGroup {
    EarthView()
}
.volumetricWindowRotationBehavior(.enabled)
.defaultSize(width: 0.6, height: 0.6, depth: 0.6, in: .meters)
```

> 💡 **生活类比**：Window 像一幅画挂在墙上，Volume 像一个展示柜放在桌上。展示柜有长宽高，你可以绕着它走一圈从各个角度观看里面的 3D 物品。

| Window vs Volume | Window | Volume |
|------------------|--------|--------|
| 维度 | 2D | 3D |
| 深度 | 无 | 有 |
| 可绕行查看 | ❌ | ✅ |
| 适用内容 | 文字、列表、表单 | 3D 模型、场景 |
| 默认修饰符 | `.windowStyle(.automatic)` | `.volumetricWindowRotationBehavior(.enabled)` |

### 5.3 Ornament —— 装饰性控件

**Ornament** 是 visionOS 特有的 UI 组件，它悬浮在窗口的边缘或下方，用于放置工具栏、按钮等辅助控件：

```swift
struct ContentView: View {
    var body: some View {
        Text("主内容区域")
            .font(.title)
            .ornament(
                visibility: .visible,
                attachmentAnchor: .scene(.bottom)
            ) {
                HStack {
                    Button("功能 A") { }
                    Button("功能 B") { }
                    Button("功能 C") { }
                }
                .buttonStyle(.borderless)
                .padding(.horizontal, 20)
                .padding(.vertical, 10)
            }
    }
}
```

| Ornament 特性 | 说明 |
|---------------|------|
| 位置 | 悬浮在窗口边缘（上下左右均可） |
| 外观 | 半透明玻璃质感 |
| 用途 | 工具栏、导航按钮、状态信息 |
| 与窗口关系 | 跟随窗口移动 |

> 💡 **生活类比**：Ornament 就像挂在画框下方的名牌——它不属于画的内容，但和画是一体的，画移动时名牌也跟着移动。

### 5.4 visionOS 特有 UI 组件

| 组件 | 说明 | 示例用途 |
|------|------|----------|
| `GlassEffect` | 玻璃质感背景 | 窗口默认效果 |
| `Ornament` | 悬浮装饰控件 | 工具栏 |
| `TabView`（visionOS 风格） | 侧边标签栏 | App 导航 |
| `NavigationSplitView` | 分栏导航 | 设置页面 |
| `Suspension` | 暂停状态视图 | App 切到后台时显示 |

---

## 6. 3D 内容

### 6.1 RealityKit 简介

**RealityKit** 是 Apple 专为 AR/VR 开发设计的 3D 框架，在 visionOS 中是展示 3D 内容的首选方案。

| 框架 | 特点 | 适用场景 |
|------|------|----------|
| **RealityKit** | 高级 API，Swift 原生，易上手 | 大多数 visionOS 3D 场景 |
| **SceneKit** | 传统 3D 框架，Objective-C 时代 | 简单 3D 展示、游戏 |
| **ARKit** | 增强现实框架 | 需要环境感知的场景 |

> 💡 **建议**：visionOS 开发优先使用 RealityKit，它是 Apple 主推的方向，与 SwiftUI 集成最好。

### 6.2 使用 Model3D 加载 3D 模型

`Model3D` 是 SwiftUI 中加载 3D 模型最简单的方式：

```swift
import SwiftUI
import RealityKit

struct ModelView: View {
    var body: some View {
        Model3D(named: "Earth") { model in
            model
                .resizable()
                .frame(width: 300, height: 300)
        } placeholder: {
            ProgressView()
        }
    }
}
```

| Model3D 参数 | 说明 |
|--------------|------|
| `named` | 模型文件名（不带扩展名） |
| `bundle` | 模型所在 bundle，默认 `.main` |
| `placeholder` | 加载时显示的占位视图 |

### 6.3 支持的 3D 模型格式

| 格式 | 说明 | 推荐度 |
|------|------|--------|
| **USDZ** | Apple 推荐格式，支持 PBR 材质、动画 | ⭐⭐⭐⭐⭐ |
| **reality** | Reality Composer 项目格式 | ⭐⭐⭐⭐ |
| **OBJ** | 传统 3D 格式 | ⭐⭐ |
| **GLTF** | 开源 3D 格式（需转换） | ⭐⭐⭐（转换后使用） |

> ⚠️ **重要**：visionOS 首选 **USDZ** 格式。你可以使用 Apple 的 **Reality Composer Pro**（随 Xcode 安装）来创建和编辑 3D 场景。

### 6.4 使用 RealityView 创建 3D 场景

`RealityView` 提供了更强大的 3D 场景控制能力：

```swift
import SwiftUI
import RealityKit

struct SolarSystemView: View {
    var body: some View {
        RealityView { content in
            let earth = ModelEntity(
                mesh: .generateSphere(radius: 0.1),
                materials: [SimpleMaterial(color: .blue, isMetallic: false)]
            )
            earth.position = [0, 0, 0]
            content.add(earth)

            let moon = ModelEntity(
                mesh: .generateSphere(radius: 0.03),
                materials: [SimpleMaterial(color: .gray, isMetallic: false)]
            )
            moon.position = [0.2, 0, 0]
            content.add(moon)
        }
    }
}
```

### 6.5 Reality Composer Pro

**Reality Composer Pro** 是 Xcode 内置的 3D 场景编辑器：

| 功能 | 说明 |
|------|------|
| 创建 3D 场景 | 拖拽式编辑 |
| 导入 USDZ 模型 | 支持多种 3D 格式转换 |
| 添加材质和动画 | 可视化编辑 |
| 添加行为和交互 | 无代码交互设置 |
| 预览 visionOS 效果 | 实时预览 |

打开方式：**Xcode → File → New → Reality Composer Pro Scene**

---

## 7. 手势与交互

### 7.1 visionOS 三大交互方式

visionOS 的交互核心是 **"看 + 点"**：

| 交互方式 | 说明 | 类比 |
|----------|------|------|
| **注视（Gaze）** | 看向某个元素，它会高亮 | 像用手指指向某个东西 |
| **捏合（Tap/Pinch）** | 拇指和食指捏合 = 点击 | 像按按钮 |
| **拖拽（Drag）** | 捏合后移动手 = 拖动 | 像抓住东西移动 |

> 💡 **生活类比**：想象你有一根魔法手指——你用眼睛"指"向目标，然后用拇指和食指"捏"一下来确认操作。整个过程不需要触碰任何东西。

### 7.2 SwiftUI 手势支持

visionOS 中的 SwiftUI 手势与 iOS 基本一致，因为系统会自动将注视+捏合映射为点击：

```swift
struct GestureDemoView: View {
    @State private var scale: CGFloat = 1.0
    @State private var rotation: Angle = .zero
    @State private var isPressed = false

    var body: some View {
        VStack(spacing: 30) {
            Circle()
                .fill(isPressed ? .green : .blue)
                .frame(width: 100, height: 100)
                .scaleEffect(scale)
                .rotationEffect(rotation)
                .gesture(
                    TapGesture()
                        .onEnded { _ in
                            isPressed.toggle()
                        }
                )
                .gesture(
                    LongPressGesture()
                        .onEnded { _ in
                            scale = scale == 1.0 ? 1.5 : 1.0
                        }
                )
                .gesture(
                    DragGesture()
                        .onChanged { value in
                            rotation = .degrees(Double(value.translation.width))
                        }
                        .onEnded { _ in
                            rotation = .zero
                        }
                )

            Text("注视圆圈，捏合点击切换颜色")
            Text("长按放大，拖拽旋转")
        }
    }
}
```

### 7.3 空间点击（Spatial TapGesture）

visionOS 新增了 `SpatialTapGesture`，可以获取 3D 空间中的点击位置：

```swift
RealityView { content in
    let entity = ModelEntity(
        mesh: .generateSphere(radius: 0.1),
        materials: [SimpleMaterial(color: .orange, isMetallic: true)]
    )
    entity.name = "sphere"
    entity.components.set(HoverEffectComponent())
    content.add(entity)
}
.gesture(
    SpatialTapGesture()
        .targetedToAnyEntity()
        .onEnded { value in
            let point3D = value.location3D
            print("点击了 3D 位置: \(point3D)")
        }
)
```

### 7.4 悬停效果（Hover Effect）

在 visionOS 中，当用户注视某个可交互元素时，系统会自动显示悬停高亮效果：

```swift
Button("点击我") {
    print("按钮被点击")
}
.hoverEffect()
```

| HoverEffect 类型 | 说明 |
|------------------|------|
| `.automatic` | 系统自动选择（默认） |
| `.highlight` | 高亮效果 |
| `.lift` | 轻微抬起效果 |

> 💡 **最佳实践**：所有可交互元素都应该添加 `.hoverEffect()`，这样用户注视时会有视觉反馈，知道这个元素可以交互。

---

## 8. 从 iOS App 适配 visionOS

### 8.1 两种适配方式

| 方式 | 说明 | 工作量 | 体验 |
|------|------|--------|------|
| **兼容模式（Compatible）** | iPad App 直接在 visionOS 上运行 | 几乎为零 | ⭐⭐ 一般 |
| **原生适配（Native）** | 为 visionOS 重新设计 UI | 较大 | ⭐⭐⭐⭐⭐ 优秀 |

### 8.2 兼容模式

如果你的 App 是 iPad App，它可以在 visionOS 上以兼容模式运行——就像在 Vision Pro 里打开了一台虚拟 iPad：

```bash
# 在 Xcode 中，打开你的 iPad App 项目
# Target → General → Supported Destinations
# 添加 visionOS (Designed for iPad)
```

| 兼容模式特点 | 说明 |
|-------------|------|
| 外观 | 显示为 iPad 窗口 |
| 交互 | 注视+捏合模拟触摸 |
| 3D 功能 | ❌ 不支持 |
| 空间功能 | ❌ 不支持 |
| Ornament | ❌ 不支持 |
| 用户评价 | 通常较差（"这只是个 iPad App"） |

> ⚠️ **注意**：兼容模式虽然零成本，但用户体验较差。Apple 审核时也会更倾向于原生 visionOS App。如果条件允许，建议做原生适配。

### 8.3 原生适配

将 iOS App 原生适配到 visionOS 的关键步骤：

**第一步：添加 visionOS Target**

```
Xcode → File → New → Target → visionOS → App
```

**第二步：适配 UI**

| iOS 组件 | visionOS 适配建议 |
|----------|-------------------|
| `UITabBar` | 使用 `TabView`（visionOS 侧边栏风格） |
| `UINavigationController` | 使用 `NavigationSplitView` |
| `UIAlertController` | 使用 `.alert()` 修饰符 |
| `UICollectionView` | 使用 `LazyVGrid` / `LazyHGrid` |
| `UIScrollView` | 使用 `ScrollView` |
| 触摸交互 | 注视+捏合自动映射，基本无需修改 |

**第三步：利用 visionOS 特有功能**

```swift
struct AdaptedView: View {
    var body: some View {
        TabView {
            Tab("首页", systemImage: "house") {
                HomeView()
            }
            Tab("3D 展示", systemImage: "cube") {
                Model3D(named: "Product") { model in
                    model.resizable()
                } placeholder: {
                    Text("加载中...")
                }
            }
        }
        .ornament(attachmentAnchor: .scene(.bottom)) {
            Button("分享") { }
        }
    }
}
```

### 8.4 条件编译

当你的代码需要同时支持 iOS 和 visionOS 时，使用条件编译：

```swift
#if os(visionOS)
Text("visionOS 专属内容")
    .ornament(attachmentAnchor: .scene(.bottom)) {
        Button("3D 查看") { open3DView() }
    }
#else
Text("iOS 内容")
#endif
```

| 条件编译宏 | 说明 |
|-----------|------|
| `#if os(visionOS)` | visionOS 平台 |
| `#if os(iOS)` | iOS 平台 |
| `#if os(macOS)` | macOS 平台 |
| `#if canImport(RealityKit)` | 检查是否可用 RealityKit |

---

## 9. 发布到 visionOS App Store

### 9.1 发布流程概览

```
开发完成 → 测试 → App Store Connect 提交 → 审核 → 上架
```

### 9.2 详细步骤

| 步骤 | 操作 | 说明 |
|------|------|------|
| 1 | 注册 Apple Developer Program | 年费 $99，[developer.apple.com](https://developer.apple.com) |
| 2 | 创建 App Record | 在 App Store Connect 中创建新 App |
| 3 | 配置 App 信息 | 名称、描述、截图、分类等 |
| 4 | 准备截图 | 需要 visionOS 截图（模拟器即可） |
| 5 | Archive 打包 | Xcode → Product → Archive |
| 6 | 上传构建版本 | 通过 Xcode 或 Transporter 上传 |
| 7 | 提交审核 | 在 App Store Connect 中提交 |
| 8 | 等待审核 | 通常 1-3 天 |
| 9 | 上架 | 审核通过后自动发布 |

### 9.3 visionOS 截图要求

| 要求 | 说明 |
|------|------|
| 设备 | Apple Vision Pro |
| 尺寸 | 2064 × 2220 像素（必需） |
| 格式 | PNG 或 JPEG |
| 数量 | 至少 3 张，最多 10 张 |
| 内容 | 展示 App 核心功能 |

> 💡 **截图技巧**：在模拟器中运行 App，使用 **File → Save Screen** 或快捷键保存截图。也可以使用 `xcrun simctl io` 命令行截图。

### 9.4 审核注意事项

| 常见被拒原因 | 说明 |
|-------------|------|
| 仅为兼容模式 iPad App | 没有利用 visionOS 特有功能 |
| 交互体验差 | 未适配注视+捏合交互 |
| 缺少悬停效果 | 可交互元素没有 hoverEffect |
| 3D 内容性能差 | 帧率低于 90fps |
| 隐私问题 | 使用了相机/传感器但未说明 |

---

## 10. 资源与学习路径

### 10.1 官方资源

| 资源 | 链接 | 说明 |
|------|------|------|
| visionOS 开发文档 | developer.apple.com/visionos | 官方文档首页 |
| SwiftUI 文档 | developer.apple.com/documentation/swiftui | SwiftUI 参考 |
| RealityKit 文档 | developer.apple.com/documentation/realitykit | 3D 框架参考 |
| WWDC 视频 | developer.apple.com/videos | 每年 WWDC session |
| Human Interface Guidelines | developer.apple.com/design/human-interface-guidelines/visionos | 设计规范 |

### 10.2 推荐 WWDC Sessions

| Session | 主题 | 编号 |
|---------|------|------|
| Meet visionOS | visionOS 概览 | WWDC23 |
| Get started with visionOS | 入门指南 | WWDC23 |
| Build spatial experiences with RealityKit | RealityKit 3D 开发 | WWDC23 |
| Design for spatial input | 空间交互设计 | WWDC23 |
| Elevate your visionOS app | 提升体验 | WWDC24 |
| Create custom spatial templates | 自定义空间模板 | WWDC24 |

### 10.3 学习路径建议

```
第 1 周：SwiftUI 基础（如果还不熟悉）
    ↓
第 2 周：visionOS 项目创建 + WindowGroup + 基础 UI
    ↓
第 3 周：Volume + 3D 内容（RealityKit / Model3D）
    ↓
第 4 周：手势交互 + Ornament + 空间设计
    ↓
第 5 周：完整项目实战 + 适配 + 发布准备
```

### 10.4 实战项目建议

| 项目 | 难度 | 涉及知识点 |
|------|------|-----------|
| 空间计算器 | ⭐ | WindowGroup、SwiftUI 基础 |
| 3D 图片浏览器 | ⭐⭐ | Volume、Model3D、手势 |
| 空间天气 App | ⭐⭐ | Window + Volume、网络请求 |
| 3D 太阳系 | ⭐⭐⭐ | RealityView、动画、手势 |
| 空间白板 | ⭐⭐⭐⭐ | Full Space、手势绘制、Entity |

---

## 小结

| 本章知识点 | 一句话总结 |
|-----------|-----------|
| visionOS 是什么 | Apple 的空间计算平台，运行在 Vision Pro 上 |
| 空间计算概念 | 数字内容与真实空间融合，三种呈现：Window / Volume / Full Space |
| 开发环境 | Xcode + visionOS SDK + Vision Pro 模拟器 |
| 第一个项目 | Xcode 选择 visionOS App 模板，SwiftUI 开发 |
| 空间 UI | WindowGroup（2D）、Volume（3D）、Ornament（装饰控件） |
| 3D 内容 | Model3D 加载模型、RealityView 创建场景、USDZ 格式 |
| 手势交互 | 注视 + 捏合 + 拖拽，SwiftUI 手势自动映射 |
| iOS 适配 | 兼容模式零成本但体验差，原生适配体验好 |
| 发布流程 | Developer Program → App Store Connect → Archive → 审核 → 上架 |
| 学习路径 | SwiftUI → 基础 UI → 3D → 交互 → 实战项目 |

> 💡 **写在最后**：visionOS 是一个全新的平台，目前还处于早期阶段。这意味着机会很多——越早学习，越能抢占先机。不要被"3D"、"空间计算"这些概念吓到，本质上你还是在用 SwiftUI 写代码，只是画布从平面变成了空间。从 WindowGroup 开始，一步步探索，你会发现空间计算开发并没有想象中那么难！
