# tvOS 快速入门

## 本章目标

- 理解 Apple TV 的产品定位与适合的 App 类型
- 掌握 tvOS 项目创建与 Target 配置方法
- 学会使用 SwiftUI 开发 tvOS 界面，包括布局适配与焦点管理
- 深入理解 Focus Engine 焦点导航机制
- 掌握 tvOS 媒体播放（AVPlayer 全屏播放、字幕、画中画）
- 了解 tvOS 特有功能（Top Shelf、Siri 搜索、TVServices 框架）
- 熟悉 tvOS 设计规范（10-foot UI、安全区域、交互模式）
- 完成一个视频流媒体 tvOS App 实战

---

## 1. tvOS 开发概述

### 1.1 Apple TV 的定位

Apple TV 是一款**客厅大屏设备**，用户通常坐在沙发上，距离电视屏幕约 **3 米（10 英尺）**，通过遥控器进行操作。交互方式以**焦点导航**为核心，没有触摸屏，没有多点触控。

> 💡 **生活类比**：iPhone 像你的私人笔记本，贴在手边、指尖操控；Apple TV 像客厅里的画框，远距离观赏、遥控器指挥——内容要大、文字要粗、操作要简。

### 1.2 适合的 App 类型

| 适合的 App 类型 | 不适合的 App 类型 |
|----------------|------------------|
| 视频流媒体（Netflix、B站） | 文字密集型阅读 |
| 音乐与播客 | 复杂表单输入 |
| 照片展示与相册 | 精细绘图工具 |
| 健身与运动（Apple Fitness+） | 实时策略游戏 |
| 休闲游戏 | 需要精确触控的游戏 |
| 智能家居控制 | 代码编辑器 |
| 教育与儿童内容 | 文档处理 |

> ⚠️ **核心原则**：tvOS App 应该是 **"Lean-back"（后仰式）** 的——用户靠在沙发上，用最少的操作获取最大的内容享受。

### 1.3 tvOS 与 iOS 开发的异同

| 维度 | iOS | tvOS |
|------|-----|------|
| 交互方式 | 触摸屏 | 遥控器焦点导航 |
| 屏幕尺寸 | 3.5~13 英寸 | 40~80 英寸 |
| 观看距离 | 0.3~0.5 米 | 2.5~4 米 |
| 输入方式 | 触摸、键盘、语音 | 遥控器、语音、键盘（辅助） |
| 焦点系统 | 无（直接触摸） | Focus Engine（核心机制） |
| 多点触控 | 支持 | 不支持 |
| 框架 | UIKit / SwiftUI | UIKit（部分）/ SwiftUI |
| 不可用框架 | — | WebKit、MapKit、MailKit 等 |
| 资源限制 | 较充裕 | 内存受限（需注意优化） |

> 💡 **好消息**：tvOS 基于 iOS，大部分 Swift/SwiftUI 知识可以直接复用。最大的差异在于**焦点导航**和**大屏适配**。

---

## 2. 项目创建与配置

### 2.1 创建 tvOS 项目

1. 打开 Xcode → **File → New → Project…**
2. 选择 **tvOS** 标签页 → **App**
3. 填写项目信息：

| 配置项 | 说明 |
|--------|------|
| Product Name | 项目名称 |
| Team | 开发团队 |
| Organization Identifier | 组织标识符 |
| Interface | **SwiftUI**（推荐） |
| Language | Swift |

### 2.2 项目 Target 结构

tvOS 项目结构比 watchOS 更简单，通常只有一个 Target：

| Target | 说明 |
|--------|------|
| **tvOS App** | Apple TV 端应用（包含 UI 和逻辑） |
| **TV Services Extension** | Top Shelf 动态内容扩展（可选） |

### 2.3 遥控器交互基础

Apple TV 遥控器（Siri Remote）是 tvOS 的核心输入设备：

| 交互方式 | 对应操作 | SwiftUI 响应 |
|----------|---------|-------------|
| 点击触控板 | 选择/确认 | `.onTapGesture` / Button |
| 上/下/左/右滑动 | 焦点移动 | Focus Engine 自动处理 |
| 菜单键 | 返回/退出 | 自动处理 / 自定义 |
| Siri 键 | 语音搜索 | `NSSearchableItem` |
| 播放/暂停 | 媒体控制 | `onPlayPauseCommand` |
| 长按 | 上下文菜单 | `.contextMenu` |

> 💡 **关键理解**：在 tvOS 上，用户**不能直接触摸屏幕**，所有交互都通过遥控器间接完成。Focus Engine 负责管理"当前选中哪个 UI 元素"。

### 2.4 Info.plist 关键配置

```xml
<key>UIUserInterfaceStyle</key>
<string>Automatic</string>

<key>NSAppTransportSecurity</key>
<dict>
    <key>NSAllowsArbitraryLoads</key>
    <true/>
</dict>
```

> ⚠️ **注意**：tvOS App 通常需要加载网络视频资源，建议在 Info.plist 中配置 ATS 例外。上架时需在审核说明中解释原因。

---

## 3. SwiftUI for tvOS

### 3.1 布局适配要点

tvOS 的布局核心原则是**大而清晰**——所有内容必须在 3 米外可读：

| 设计参数 | 推荐值 |
|----------|--------|
| 最小可点击区域 | 192 × 88 pt |
| 正文文字大小 | ≥ 29 pt |
| 标题文字大小 | ≥ 38 pt |
| 卡片间距 | 40~60 pt |
| 边距 | ≥ 90 pt（安全区域内） |

### 3.2 基础布局示例

```swift
import SwiftUI

struct ContentView: View {
    var body: some View {
        NavigationStack {
            ScrollView {
                VStack(alignment: .leading, spacing: 40) {
                    FeaturedSection()
                    CategorySection(title: "热门推荐", items: sampleItems)
                    CategorySection(title: "最新上线", items: recentItems)
                }
                .padding(.horizontal, 90)
                .padding(.vertical, 60)
            }
        }
    }
}
```

### 3.3 卡片式 UI 设计

tvOS 最常见的 UI 模式是**横向滚动的卡片列表**：

```swift
struct CategorySection: View {
    let title: String
    let items: [MediaItem]

    var body: some View {
        VStack(alignment: .leading, spacing: 20) {
            Text(title)
                .font(.title3)
                .fontWeight(.semibold)

            ScrollView(.horizontal, showsIndicators: false) {
                HStack(spacing: 40) {
                    ForEach(items) { item in
                        MediaCard(item: item)
                    }
                }
            }
        }
    }
}

struct MediaCard: View {
    let item: MediaItem

    var body: some View {
        VStack(spacing: 12) {
            AsyncImage(url: item.imageURL) { image in
                image
                    .resizable()
                    .aspectRatio(16/9, contentMode: .fit)
            } placeholder: {
                Rectangle()
                    .fill(.gray.opacity(0.3))
                    .aspectRatio(16/9, contentMode: .fit)
            }
            .frame(width: 400, height: 225)
            .cornerRadius(12)

            Text(item.title)
                .font(.body)
                .lineLimit(1)
        }
    }
}
```

### 3.4 TVLayoutGuide

tvOS 17 引入了 `TVLayoutGuide`，帮助自动适配安全区域：

```swift
struct SafeAreaDemo: View {
    var body: some View {
        GeometryReader { geometry in
            let safeArea = geometry.safeAreaInsets

            VStack {
                Text("顶部安全区域: \(safeArea.top)")
                Spacer()
                Text("内容区域")
                Spacer()
                Text("底部安全区域: \(safeArea.bottom)")
            }
            .padding(.horizontal, safeArea.leading + 40)
            .padding(.horizontal, safeArea.trailing + 40)
        }
    }
}
```

> 💡 **提示**：tvOS 的安全区域比 iOS 更大，因为电视屏幕存在**过扫描（Overscan）** 问题。始终使用 `safeAreaInsets` 或 `.padding()` 确保内容不被裁切。

---

## 4. Focus Engine 详解

### 4.1 焦点导航原理

Focus Engine 是 tvOS 的核心交互机制。它就像一个**看不见的光标**，在 UI 元素之间自动移动：

```
┌──────────────────────────────────────────────┐
│                                              │
│   ┌──────┐   ┌──────┐   ┌──────┐           │
│   │  A   │   │  B◄──│───│  C   │  ← 焦点在 B│
│   └──────┘   └──────┘   └──────┘           │
│                                              │
│   ┌──────┐   ┌──────┐   ┌──────┐           │
│   │  D   │   │  E   │   │  F   │           │
│   └──────┘   └──────┘   └──────┘           │
│                                              │
│   按遥控器 ↓ → 焦点从 B 移到 E              │
└──────────────────────────────────────────────┘
```

> 💡 **生活类比**：Focus Engine 就像聚光灯——舞台上只有一个人被照亮，遥控器的方向键控制聚光灯移向谁。

### 4.2 SwiftUI 焦点管理

```swift
struct FocusDemo: View {
    @FocusState private var focusedItem: ItemID?

    var body: some View {
        HStack(spacing: 40) {
            ForEach(items) { item in
                MediaCard(item: item)
                    .focused($focusedItem, equals: item.id)
                    .scaleEffect(focusedItem == item.id ? 1.05 : 1.0)
                    .shadow(radius: focusedItem == item.id ? 20 : 0)
                    .animation(.easeInOut(duration: 0.2), value: focusedItem)
            }
        }
    }
}
```

### 4.3 focusSection 分组

SwiftUI 的 `focusSection` 修饰符可以将元素分组，控制焦点在组间的跳转行为：

```swift
struct FocusSectionDemo: View {
    var body: some View {
        VStack(spacing: 60) {
            HStack(spacing: 40) {
                ForEach(featuredItems) { item in
                    MediaCard(item: item)
                }
            }
            .focusSection()

            HStack(spacing: 40) {
                ForEach(recentItems) { item in
                    MediaCard(item: item)
                }
            }
            .focusSection()
        }
    }
}
```

> 💡 **`focusSection` 的作用**：当焦点在第一行时，按遥控器下键，焦点会跳到第二行的第一个元素，而不是第一行正下方的元素。

### 4.4 UIFocusGuide（UIKit 方式）

在 UIKit 中，可以使用 `UIFocusGuide` 在不可聚焦的区域创建"焦点桥梁"：

```swift
class FocusGuideViewController: UIViewController {
    let focusGuide = UIFocusGuide()

    override func viewDidLoad() {
        super.viewDidLoad()

        view.addLayoutGuide(focusGuide)
        focusGuide.leadingAnchor.constraint(equalTo: leftButton.trailingAnchor).isActive = true
        focusGuide.trailingAnchor.constraint(equalTo: rightButton.leadingAnchor).isActive = true
        focusGuide.topAnchor.constraint(equalTo: leftButton.topAnchor).isActive = true
        focusGuide.heightAnchor.constraint(equalTo: leftButton.heightAnchor).isActive = true

        focusGuide.preferredFocusEnvironments = [rightButton]
    }
}
```

> ⚠️ **注意**：在 SwiftUI 中，`focusSection` 已经替代了大部分 `UIFocusGuide` 的使用场景。仅在 UIKit 混合开发时才需要手动使用 `UIFocusGuide`。

### 4.5 焦点动画最佳实践

| 效果 | 实现方式 | 说明 |
|------|---------|------|
| 放大 | `.scaleEffect(1.05)` | 最常用的焦点反馈 |
| 阴影 | `.shadow(radius: 20)` | 增强层次感 |
| 边框 | `.overlay(RoundedRectangle...stroke)` | 清晰标识焦点 |
| 抬起 | `.offset(y: -10)` + shadow | 模拟"浮起"效果 |
| 颜色变化 | `.foregroundStyle` | 辅助焦点反馈 |

```swift
struct FocusedCard: View {
    let item: MediaItem
    @FocusState private var isFocused: Bool

    var body: some View {
        VStack(spacing: 12) {
            AsyncImage(url: item.imageURL) { image in
                image.resizable().aspectRatio(16/9, contentMode: .fit)
            } placeholder: {
                Rectangle().fill(.gray.opacity(0.3))
            }
            .frame(width: 400, height: 225)
            .cornerRadius(12)
            .overlay(
                RoundedRectangle(cornerRadius: 12)
                    .stroke(Color.white, lineWidth: isFocused ? 4 : 0)
            )

            Text(item.title)
                .font(.body)
        }
        .focused($isFocused)
        .scaleEffect(isFocused ? 1.08 : 1.0)
        .shadow(radius: isFocused ? 24 : 0)
        .animation(.easeInOut(duration: 0.15), value: isFocused)
    }
}
```

---

## 5. 媒体播放

### 5.1 AVPlayer on tvOS

tvOS 上最核心的功能就是视频播放。使用 `AVPlayer` + `AVPlayerViewController` 实现：

```swift
import AVKit
import SwiftUI

struct VideoPlayerView: View {
    let videoURL: URL

    var body: some View {
        VideoPlayer(player: AVPlayer(url: videoURL))
    }
}
```

### 5.2 自定义播放器

对于更复杂的播放需求，可以使用 `AVPlayerViewController` 进行自定义：

```swift
import AVKit
import SwiftUI

struct CustomPlayerView: UIViewControllerRepresentable {
    let url: URL

    func makeUIViewController(context: Context) -> AVPlayerViewController {
        let controller = AVPlayerViewController()
        let player = AVPlayer(url: url)
        controller.player = player
        controller.showsPlaybackControls = true
        controller.allowsPictureInPicturePlayback = true
        return controller
    }

    func updateUIViewController(_ uiViewController: AVPlayerViewController, context: Context) {}
}
```

### 5.3 字幕与多音轨

```swift
func setupSubtitles(for player: AVPlayer) {
    guard let currentItem = player.currentItem else { return }

    let subtitleURL = URL(string: "https://example.com/subtitles.vtt")!
    let subtitleAsset = AVURLAsset(url: subtitleURL)
    let subtitleItem = AVAssetImportSession(asset: subtitleAsset, presetName: AVAssetExportPresetPassthrough)

    let composition = AVMutableComposition()
    let compositionTrack = composition.addMutableTrack(
        withMediaType: .subtitle,
        preferredTrackID: kCMPersistentTrackID_Invalid
    )

    if let subtitleTrack = subtitleAsset.tracks(withMediaType: .subtitle).first {
        try? compositionTrack?.insertTimeRange(
            CMTimeRange(start: .zero, duration: subtitleAsset.duration),
            of: subtitleTrack,
            at: .zero
        )
    }
}
```

### 5.4 画中画（PiP）

tvOS 支持画中画播放，用户可以在浏览其他内容时继续观看视频：

```swift
class PlayerViewModel: ObservableObject {
    let player = AVPlayer()
    var pipController: AVPlayerViewController?

    func enablePiP() {
        pipController?.allowsPictureInPicturePlayback = true
        pipController?.startPictureInPicture()
    }

    func disablePiP() {
        pipController?.stopPictureInPicture()
    }
}
```

> ⚠️ **注意**：tvOS 的画中画与 iOS 不同——在 tvOS 上，PiP 窗口出现在屏幕右下角，用户可以继续浏览 App 的其他内容。

---

## 6. tvOS 特有功能

### 6.1 Top Shelf

Top Shelf 是 Apple TV 主界面顶部的**大横幅区域**，当用户选中 App 图标时展示：

| Top Shelf 类型 | 说明 | 更新方式 |
|---------------|------|---------|
| **静态** | 使用 App 图标和默认内容 | Info.plist 配置 |
| **动态** | 展示个性化内容（推荐视频等） | TVServices Extension |

**静态 Top Shelf 配置：**

```xml
<key>UIBackgroundModes</key>
<array>
    <string>fetch</string>
</array>
```

**动态 Top Shelf 实现：**

```swift
import TVServices

class Provider: TVTopShelfProvider {
    var topShelfStyle: TVTopShelfContentStyle {
        return .sectioned
    }

    var topShelfItems: [TVContentItem] {
        let sectionItem = TVContentItem(identifier: TVContentIdentifier(identifier: "featured", container: nil)!)
        sectionItem.title = "热门推荐"

        var items: [TVContentItem] = []
        for video in featuredVideos {
            let item = TVContentItem(identifier: TVContentIdentifier(identifier: video.id, container: nil)!)
            item.title = video.title
            item.imageURL = video.thumbnailURL
            item.displayURL = URL(string: "myapp://video/\(video.id)")
            items.append(item)
        }

        sectionItem.topShelfItems = items
        return [sectionItem]
    }
}
```

### 6.2 Siri 搜索与 Universal Links

tvOS 用户经常使用 Siri 搜索内容。通过 `NSUserActivity` 让你的内容可被搜索：

```swift
func makeActivity(for item: MediaItem) -> NSUserActivity {
    let activity = NSUserActivity(activityType: "com.example.app.search")
    activity.title = item.title
    activity.userInfo = ["id": item.id]
    activity.isEligibleForSearch = true
    activity.isEligibleForPublicIndexing = true
    activity.keywords = Set(item.tags)
    activity.contentAttributeSet = CSSearchableItemAttributeSet()
    activity.contentAttributeSet?.title = item.title
    activity.contentAttributeSet?.contentDescription = item.description
    activity.contentAttributeSet?.thumbnailURL = item.thumbnailURL
    return activity
}
```

**Universal Links 深度链接：**

```swift
struct DeepLinkHandler {
    func handle(url: URL) -> MediaItem? {
        guard url.host == "video" else { return nil }
        let id = url.lastPathComponent
        return MediaStore.find(by: id)
    }
}

@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
                .onOpenURL { url in
                    if let item = DeepLinkHandler().handle(url: url) {
                        router.navigate(to: .player(item))
                    }
                }
        }
    }
}
```

### 6.3 TVServices 框架

TVServices 框架提供 tvOS 特有的服务能力：

| 功能 | 类/协议 | 说明 |
|------|--------|------|
| Top Shelf 动态内容 | `TVTopShelfProvider` | 提供主界面顶部内容 |
| 内容目录 | `TVContentItem` | 描述可搜索的内容条目 |
| 内容标识 | `TVContentIdentifier` | 唯一标识内容 |

> 💡 **提示**：要使用动态 Top Shelf，需要在项目中添加 **TV Services Extension** Target，系统会定期调用 `TVTopShelfProvider` 获取最新内容。

---

## 7. tvOS 设计规范

### 7.1 10-foot UI 原则

tvOS 设计的核心是 **"10-foot UI"**——假设用户坐在距屏幕 10 英尺（约 3 米）的位置：

| 设计原则 | 具体要求 |
|----------|---------|
| **大字体** | 正文 ≥ 29pt，标题 ≥ 38pt |
| **大触控区域** | 最小 192 × 88 pt |
| **高对比度** | 避免浅色文字在浅色背景上 |
| **简洁布局** | 每屏展示核心内容，避免信息过载 |
| **清晰反馈** | 焦点状态必须明显可辨 |

### 7.2 字体大小对照

| 用途 | tvOS 推荐大小 | iOS 对应大小 |
|------|-------------|-------------|
| 大标题 | 76 pt | 34 pt |
| 标题 | 48 pt | 20 pt |
| 正文 | 29 pt | 17 pt |
| 辅助文字 | 23 pt | 14 pt |
| 脚注 | 19 pt | 12 pt |

> ⚠️ **注意**：tvOS 上的字体大小约为 iOS 的 **1.5~2 倍**。不要直接把 iOS 的布局搬到 tvOS 上。

### 7.3 安全区域

电视屏幕存在**过扫描**问题，不同品牌和型号的电视裁切程度不同：

```
┌─────────────────────────────────────┐
│          可能被裁切的区域             │
│   ┌─────────────────────────────┐   │
│   │                             │   │
│   │       安全内容区域           │   │
│   │    （放置所有重要内容）       │   │
│   │                             │   │
│   └─────────────────────────────┘   │
│          可能被裁切的区域             │
└─────────────────────────────────────┘
```

| 区域 | 最小边距 |
|------|---------|
| 左/右边距 | 90 pt（6% 屏幕宽度） |
| 上边距 | 60 pt |
| 下边距 | 60 pt |

### 7.4 交互模式对比

| 交互模式 | iOS | tvOS |
|----------|-----|------|
| 导航 | 直接点击目标 | 焦点移动 → 选择 |
| 滚动 | 手指滑动 | 遥控器滑动（自动分页） |
| 返回 | 左滑/返回按钮 | 菜单键 |
| 搜索 | 键盘输入 | Siri / 遥控器输入 |
| 上下文菜单 | 长按 | 长按遥控器 |
| 多任务 | App 切换器 | TV 按钮 → App 切换器 |

---

## 8. 实战：创建一个视频流媒体 tvOS App

### 8.1 项目结构

```
StreamTV/
├── StreamTVApp.swift
├── Models/
│   └── Video.swift
├── Views/
│   ├── HomeView.swift
│   ├── CategoryRow.swift
│   ├── VideoCard.swift
│   └── PlayerView.swift
├── ViewModels/
│   └── HomeViewModel.swift
└── Services/
    └── VideoService.swift
```

### 8.2 数据模型

```swift
import Foundation

struct Video: Identifiable, Hashable {
    let id: String
    let title: String
    let subtitle: String
    let thumbnailURL: URL
    let streamURL: URL
    let duration: TimeInterval
    let category: String
}
```

### 8.3 主界面

```swift
import SwiftUI

struct HomeView: View {
    @StateObject private var viewModel = HomeViewModel()

    var body: some View {
        NavigationStack {
            ScrollView {
                VStack(alignment: .leading, spacing: 50) {
                    FeaturedBanner(video: viewModel.featuredVideo)

                    ForEach(viewModel.categories, id: \.self) { category in
                        CategoryRow(
                            title: category,
                            videos: viewModel.videos(for: category)
                        )
                    }
                }
                .padding(.horizontal, 90)
                .padding(.top, 60)
                .padding(.bottom, 80)
            }
            .navigationTitle("StreamTV")
        }
    }
}
```

### 8.4 分类行与卡片

```swift
struct CategoryRow: View {
    let title: String
    let videos: [Video]

    var body: some View {
        VStack(alignment: .leading, spacing: 20) {
            Text(title)
                .font(.title3)
                .fontWeight(.bold)

            ScrollView(.horizontal, showsIndicators: false) {
                HStack(spacing: 40) {
                    ForEach(videos) { video in
                        NavigationLink(value: video) {
                            VideoCard(video: video)
                        }
                        .buttonStyle(.card)
                    }
                }
            }
        }
        .focusSection()
    }
}
```

### 8.5 视频卡片（含焦点效果）

```swift
struct VideoCard: View {
    let video: Video
    @FocusState private var isFocused: Bool

    var body: some View {
        VStack(alignment: .leading, spacing: 10) {
            AsyncImage(url: video.thumbnailURL) { image in
                image
                    .resizable()
                    .aspectRatio(16/9, contentMode: .fit)
            } placeholder: {
                Rectangle()
                    .fill(Color.gray.opacity(0.3))
                    .aspectRatio(16/9, contentMode: .fit)
            }
            .cornerRadius(12)

            Text(video.title)
                .font(.callout)
                .fontWeight(.medium)
                .lineLimit(1)

            Text(video.subtitle)
                .font(.caption)
                .foregroundStyle(.secondary)
                .lineLimit(1)
        }
        .frame(width: 360)
        .focused($isFocused)
        .scaleEffect(isFocused ? 1.1 : 1.0)
        .shadow(radius: isFocused ? 20 : 0)
        .animation(.easeInOut(duration: 0.2), value: isFocused)
    }
}
```

### 8.6 播放器视图

```swift
import AVKit
import SwiftUI

struct PlayerView: UIViewControllerRepresentable {
    let video: Video

    func makeUIViewController(context: Context) -> AVPlayerViewController {
        let controller = AVPlayerViewController()
        let player = AVPlayer(url: video.streamURL)
        controller.player = player
        controller.showsPlaybackControls = true
        controller.allowsPictureInPicturePlayback = true
        player.play()
        return controller
    }

    func updateUIViewController(_ uiViewController: AVPlayerViewController, context: Context) {}
}
```

### 8.7 导航与路由

```swift
import SwiftUI

@main
struct StreamTVApp: App {
    var body: some Scene {
        WindowGroup {
            HomeView()
                .navigationDestination(for: Video.self) { video in
                    PlayerView(video: video)
                }
        }
    }
}
```

### 8.8 ViewModel

```swift
import SwiftUI

class HomeViewModel: ObservableObject {
    @Published var featuredVideo: Video?
    @Published var categories: [String] = []

    private var allVideos: [Video] = []

    init() {
        loadVideos()
    }

    func videos(for category: String) -> [Video] {
        allVideos.filter { $0.category == category }
    }

    private func loadVideos() {
        allVideos = VideoService.shared.fetchVideos()
        categories = Array(Set(allVideos.map(\.category))).sorted()
        featuredVideo = allVideos.first
    }
}
```

### 8.9 tvOS 按钮样式

tvOS 提供了专用的按钮样式，最常用的是 `.card` 样式：

```swift
NavigationLink(value: video) {
    VideoCard(video: video)
}
.buttonStyle(.card)
```

| 按钮样式 | 说明 |
|----------|------|
| `.card` | 卡片样式，焦点时自动放大+阴影 |
| `.plain` | 无样式，适合自定义 |
| `.bordered` | 带边框的按钮 |

> 💡 **提示**：`.card` 样式会自动处理焦点动画（放大 + 阴影），非常适合视频卡片。如果你需要完全自定义焦点效果，使用 `.plain` 样式。

---

## 本章小结

| 主题 | 核心要点 |
|------|---------|
| tvOS 概述 | 客厅大屏设备，"Lean-back" 后仰式体验，遥控器交互 |
| 项目创建 | tvOS Target，Siri Remote 交互，Info.plist 配置 |
| SwiftUI for tvOS | 大字体、大间距、卡片式 UI、TVLayoutGuide |
| Focus Engine | 焦点导航核心机制，`@FocusState`、`focusSection`、焦点动画 |
| 媒体播放 | AVPlayer + AVPlayerViewController，字幕、画中画 |
| 特有功能 | Top Shelf 动态内容、Siri 搜索、TVServices 框架 |
| 设计规范 | 10-foot UI、字体 ≥ 29pt、安全区域边距 90pt、高对比度 |
| 实战项目 | 视频流媒体 App：数据模型 → 主界面 → 卡片 → 播放器 → 导航 |

> 💡 **学习建议**：tvOS 开发的核心难点在于 **Focus Engine** 和 **大屏适配**。建议先用模拟器熟悉焦点导航行为，再连接真机测试遥控器交互。如果你已经熟悉 iOS 开发，tvOS 的学习曲线相对平缓——最大的思维转变是从"触摸"到"遥控"。

← [macCatalyst 与 macOS 移植](./macCatalyst与macOS移植.md)

← [macCatalyst 与 macOS 移植](./macCatalyst与macOS移植.md)
