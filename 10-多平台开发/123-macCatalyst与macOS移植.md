# 123-macCatalyst 与 macOS 移植

## 本章目标

- 理解 Mac Catalyst 的定位与能力边界，明确何时该用 Catalyst、何时该走原生
- 掌握在 Xcode 中启用 Mac Catalyst 的完整流程与优化级别选择
- 学会适配键盘快捷键、菜单栏、鼠标/触控板、窗口大小等 Mac 特有交互
- 熟练使用条件编译与 UIKit→AppKit 桥接，在 Catalyst 环境中调用 macOS 专属 API
- 了解原生 macOS 开发（SwiftUI）的核心模式，为未来深度移植做准备
- 掌握 Mac App Store 上架的沙盒、公证、Hardened Runtime 等合规要求
- 通过实战将一个 iOS App 完整移植到 Mac

---

## 1. Mac Catalyst 概述

### 1.1 一套代码上 Mac

Mac Catalyst 是 Apple 在 2019 年随 macOS 10.15 Catalina 推出的技术，它允许开发者将 iPadOS App **几乎不加修改**地运行在 macOS 上。其本质是：macOS 内置了一套 UIKit 兼容层，将 UIKit 控件映射为对应的 AppKit 控件。

> 💡 生活类比：Catalyst 就像一位"同声传译员"——你的 App 仍然在说 UIKit 的语言，但这位翻译员会实时把它翻译成 AppKit 能听懂的话，让 Mac 用户也能理解。

核心工作原理：

```
iPadOS App (UIKit)
       │
       ▼
  ┌─────────────┐
  │  Catalyst   │  ← macOS 内置的 UIKit 兼容层
  │  兼容层      │     将 UIKit 映射为 AppKit
  └─────────────┘
       │
       ▼
  macOS (AppKit 渲染)
```

### 1.2 与原生 macOS App 的区别

| 维度 | Mac Catalyst App | 原生 macOS App |
|------|------------------|----------------|
| UI 框架 | UIKit（通过兼容层） | AppKit / SwiftUI |
| 控件外观 | iOS 风格，部分自动适配 | 原生 Mac 风格 |
| API 覆盖 | 仅 UIKit 子集，部分 API 不可用 | 全部 macOS API |
| 输入方式 | 触控优先，键盘/鼠标为辅 | 键盘/鼠标优先 |
| 窗口管理 | 有限支持 | 完整支持 |
| 性能 | 多一层映射，略有损耗 | 原生性能 |
| 开发成本 | 低（复用 iOS 代码） | 高（需重写或大量适配） |

### 1.3 适用场景

**适合使用 Catalyst 的场景：**

- 工具类、效率类 App（如计算器、笔记、待办事项）
- 内容消费类 App（如阅读器、媒体播放器）
- 快速验证 Mac 市场需求
- 团队资源有限，无法维护独立 macOS 版本

**不适合使用 Catalyst 的场景：**

- 需要深度系统集成（如 Finder 扩展、系统偏好面板）
- 对性能极度敏感（如专业音视频编辑）
- 需要原生 Mac 交互范式（如多窗口拖拽、复杂菜单系统）
- 需要使用 AppKit 独有 API（如 NSCollectionView、NSOutlineView）

> ⚠️ Catalyst 不是万能药。如果你的 App 需要深度融入 Mac 生态，原生开发才是正确选择。Catalyst 的价值在于"快速上桌"，而非"精致大餐"。

---

## 2. 启用 Mac Catalyst

### 2.1 Xcode 配置步骤

1. 打开 Xcode 项目，选择 iOS App Target
2. 进入 **General** → **Supported Destinations**
3. 点击 **+** 添加 **Mac (Designed for iPad)** 或 **Mac (Catalyst)**
4. 在 **Deployment Info** 中确认 macOS 最低部署版本

```
项目导航器
  └── Target: MyiPadApp
        └── General
              └── Supported Destinations
                    ├── iPhone
                    ├── iPad
                    └── Mac (Catalyst)  ← 添加此项
```

### 2.2 优化级别选择

启用 Catalyst 后，Xcode 提供两种 UI 优化级别：

| 优化级别 | 说明 | 适用场景 |
|----------|------|----------|
| **Scaled to Fit** | 直接将 iPad UI 等比缩放到 Mac 窗口 | 快速验证，几乎零适配 |
| **Optimized for Mac** | 使用 Mac 原生控件替换部分 UIKit 控件 | 正式发布，体验更原生 |

在 Target 的 **Build Settings** 中设置：

```
Target → Build Settings → Mac Catalyst
  └── Optimize for Mac: Yes/No
```

> 💡 选择 "Optimized for Mac" 后，系统会自动将部分 UIKit 控件替换为 Mac 原生外观：如 `UISwitch` 变为 Mac 风格开关、`UITextView` 支持系统拼写检查等。

### 2.3 Info.plist 配置

Catalyst App 可能需要额外的 Info.plist 键值：

```xml
<!-- 支持多窗口（macOS 11+） -->
<key>UIApplicationSceneManifest</key>
<dict>
    <key>UIApplicationSupportsMultipleScenes</key>
    <true/>
</dict>

<!-- 声明 Mac 上的隐私权限 -->
<key>NSCameraUsageDescription</key>
<string>需要访问摄像头以进行视频通话</string>
```

> ⚠️ macOS 的隐私权限弹窗与 iOS 不同，必须在 Info.plist 中显式声明，否则 App 会直接崩溃而非弹出授权对话框。

---

## 3. 适配要点

### 3.1 键盘快捷键

Mac 用户高度依赖键盘快捷键。Catalyst 默认只提供少量快捷键（如 Cmd+C/V），你需要手动添加更多。

```swift
import UIKit

class DocumentViewController: UIViewController {

    override func viewDidLoad() {
        super.viewDidLoad()
        setupKeyCommands()
    }

    func setupKeyCommands() {
        let saveCommand = UIKeyCommand(
            input: "s",
            modifierFlags: .command,
            action: #selector(saveDocument(_:)),
            discoverabilityTitle: "保存文档"
        )

        let newCommand = UIKeyCommand(
            input: "n",
            modifierFlags: [.command, .shift],
            action: #selector(newDocument(_:)),
            discoverabilityTitle: "新建窗口"
        )

        let findCommand = UIKeyCommand(
            input: "f",
            modifierFlags: .command,
            action: #selector(findText(_:)),
            discoverabilityTitle: "查找"
        )

        addKeyCommands([saveCommand, newCommand, findCommand])
    }

    @objc func saveDocument(_ sender: UIKeyCommand) {
        print("保存文档")
    }

    @objc func newDocument(_ sender: UIKeyCommand) {
        print("新建窗口")
    }

    @objc func findText(_ sender: UIKeyCommand) {
        print("查找文本")
    }
}
```

> 💡 `discoverabilityTitle` 会让快捷键出现在 Mac 的"帮助"菜单搜索结果中，这是 Mac 用户发现快捷键的重要途径。

### 3.2 菜单栏

Catalyst 支持通过 `UIMenuBuilder` 自定义菜单栏：

```swift
override func buildMenu(with builder: UIMenuBuilder) {
    super.buildMenu(with: builder)

    guard builder.system == .main else { return }

    let exportAction = UICommand(
        title: "导出为 PDF",
        image: nil,
        action: #selector(exportToPDF(_:)),
        propertyList: nil
    )

    let exportMenu = UIMenu(
        title: "导出",
        image: nil,
        identifier: UIMenu.Identifier("com.app.exportMenu"),
        options: .displayInline,
        children: [exportAction]
    )

    builder.insertChild(exportMenu, atStartOfMenu: .file)
}

@objc func exportToPDF(_ sender: Any) {
    print("导出为 PDF")
}
```

### 3.3 鼠标与触控板

Catalyst 自动将触摸事件映射为鼠标点击，但悬停（Hover）效果需要手动实现：

```swift
class HoverButton: UIButton {

    private var isHovering = false

    override func awakeFromNib() {
        super.awakeFromNib()
        if traitCollection.userInterfaceIdiom == .mac {
            addHoverGesture()
        }
    }

    private func addHoverGesture() {
        let hover = UIHoverGestureRecognizer(
            target: self,
            action: #selector(handleHover(_:))
        )
        addGestureRecognizer(hover)
    }

    @objc private func handleHover(_ gesture: UIHoverGestureRecognizer) {
        switch gesture.state {
        case .began, .changed:
            if !isHovering {
                isHovering = true
                UIView.animate(withDuration: 0.2) {
                    self.backgroundColor = .systemBlue
                    self.layer.cornerRadius = 8
                    self.transform = CGAffineTransform(scaleX: 1.05, y: 1.05)
                }
            }
        case .ended, .cancelled:
            isHovering = false
            UIView.animate(withDuration: 0.2) {
                self.backgroundColor = .systemGray5
                self.transform = .identity
            }
        default:
            break
        }
    }
}
```

### 3.4 窗口大小适配

Mac 窗口可以自由调整大小，需要使用 Auto Layout 和 Size Classes 适配：

```swift
override func viewDidLoad() {
    super.viewDidLoad()

    let contentView = UIView()
    contentView.translatesAutoresizingMaskIntoConstraints = false
    view.addSubview(contentView)

    NSLayoutConstraint.activate([
        contentView.topAnchor.constraint(equalTo: view.topAnchor),
        contentView.leadingAnchor.constraint(equalTo: view.leadingAnchor),
        contentView.trailingAnchor.constraint(equalTo: view.trailingAnchor),
        contentView.bottomAnchor.constraint(equalTo: view.bottomAnchor),
    ])

    registerForTraitChanges([UITraitPreferredContentSizeCategory.self]) { (self: Self, _) in
        self.updateLayoutForCurrentSize()
    }
}

private func updateLayoutForCurrentSize() {
    let isWide = view.frame.width > 768
    stackView.axis = isWide ? .horizontal : .vertical
}
```

### 3.5 工具栏

Catalyst 支持将 `UIToolbar` 自动映射为 Mac 原生窗口工具栏：

```swift
override func viewDidLoad() {
    super.viewDidLoad()

    let flexibleSpace = UIBarButtonItem(barButtonSystemItem: .flexibleSpace, target: nil, action: nil)

    let composeItem = UIBarButtonItem(
        barButtonSystemItem: .compose,
        target: self,
        action: #selector(composeTapped)
    )

    let shareItem = UIBarButtonItem(
        barButtonSystemItem: .action,
        target: self,
        action: #selector(shareTapped)
    )

    let trashItem = UIBarButtonItem(
        barButtonSystemItem: .trash,
        target: self,
        action: #selector(trashTapped)
    )

    toolbarItems = [composeItem, flexibleSpace, shareItem, flexibleSpace, trashItem]
    navigationController?.isToolbarHidden = false
}
```

---

## 4. UIKit → AppKit 桥接

### 4.1 条件编译

使用 `#if targetEnvironment(macCatalyst)` 区分 Catalyst 和 iOS 环境：

```swift
#if targetEnvironment(macCatalyst)
import AppKit
#endif

class PlatformAdapter {

    static var isMacCatalyst: Bool {
        #if targetEnvironment(macCatalyst)
        return true
        #else
        return false
        #endif
    }

    static func configureForPlatform() {
        #if targetEnvironment(macCatalyst)
        enableMacSpecificFeatures()
        #else
        enableiOSSpecificFeatures()
        #endif
    }

    #if targetEnvironment(macCatalyst)
    private static func enableMacSpecificFeatures() {
        if let windowScene = UIApplication.shared.connectedScenes.first as? UIWindowScene {
            windowScene.titlebar?.titleVisibility = .visible
            windowScene.titlebar?.toolbar = nil
        }
    }
    #endif

    private static func enableiOSSpecificFeatures() {
        // iOS 专属配置
    }
}
```

### 4.2 NSTouchBar 支持

Catalyst App 可以通过 `NSTouchBar` 提供 Touch Bar 支持：

```swift
#if targetEnvironment(macCatalyst)
import AppKit

extension DocumentViewController {

    override func makeTouchBar() -> NSTouchBar? {
        let touchBar = NSTouchBar()
        touchBar.delegate = self
        touchBar.defaultItemIdentifiers = [.emojiPicker, .formatText]
        return touchBar
    }
}

extension DocumentViewController: NSTouchBarDelegate {

    func touchBar(_ touchBar: NSTouchBar, makeItemForIdentifier identifier: NSTouchBarItem.Identifier) -> NSTouchBarItem? {
        switch identifier {
        case .emojiPicker:
            let item = NSCustomTouchBarItem(identifier: identifier)
            item.view = NSButton(title: "😀", target: nil, action: nil)
            return item
        case .formatText:
            let item = NSCustomTouchBarItem(identifier: identifier)
            let segmented = NSSegmentedControl(
                labels: ["B", "I", "U"],
                trackingMode: .momentary,
                target: self,
                action: #selector(formatTapped(_:))
            )
            item.view = segmented
            return item
        default:
            return nil
        }
    }

    @objc func formatTapped(_ sender: NSSegmentedControl) {
        print("格式化选项: \(sender.selectedSegment)")
    }
}
#endif
```

### 4.3 NSMenu 桥接

在 Catalyst 中可以通过 `NSMenu` 创建上下文菜单（右键菜单）：

```swift
#if targetEnvironment(macCatalyst)
import AppKit

extension NoteListViewController {

    func showContextMenu(for note: Note, at point: CGPoint) {
        let menu = NSMenu()

        menu.addItem(NSMenuItem(title: "编辑", action: #selector(editNote(_:)), keyEquivalent: "e"))
        menu.addItem(NSMenuItem(title: "复制", action: #selector(copyNote(_:)), keyEquivalent: "c"))
        menu.addItem(NSMenuItem.separator())
        menu.addItem(NSMenuItem(title: "删除", action: #selector(deleteNote(_:)), keyEquivalent: "⌫"))

        if let view = view.hitTest(point) {
            let nsView = view as NSView
            menu.popUp(positioning: nil, at: point, in: nsView)
        }
    }
}
#endif
```

### 4.4 不可用 API 处理

部分 iOS API 在 Catalyst 中不可用，需要做降级处理：

```swift
class FeatureManager {

    func requestReview() {
        #if targetEnvironment(macCatalyst)
        // StoreKit 的 requestReview 在 Catalyst 中行为不同
        // Mac 上使用 SKStoreReviewController 仍然可用
        // 但部分 API 如 SKPaymentQueue 的 canMakePayments() 需要检查
        #else
        // iOS 正常流程
        #endif
    }

    func shareContent() {
        let activityVC = UIActivityViewController(
            activityItems: ["分享内容"],
            applicationActivities: nil
        )

        #if targetEnvironment(macCatalyst)
        // Catalyst 中 UIActivityViewController 会以 Popover 形式展示
        // 需要设置 popoverPresentationController
        activityVC.popoverPresentationController?.sourceView = view
        activityVC.popoverPresentationController?.sourceRect = CGRect(
            x: view.bounds.midX,
            y: view.bounds.midY,
            width: 0,
            height: 0
        )
        #endif

        present(activityVC, animated: true)
    }
}
```

> ⚠️ 以下 API 在 Catalyst 中不可用或行为受限：`UIDevice` 的部分属性、`UIApplication.shared.openURL()` 的某些选项、`UIWebView`（已废弃但仍有老项目使用）、部分 `UIKit` 动画 API。

---

## 5. 原生 macOS 开发

### 5.1 SwiftUI for macOS

如果 Catalyst 无法满足需求，原生 macOS 开发是更好的选择。SwiftUI 提供了跨平台能力：

```swift
import SwiftUI

@main
struct MacNotesApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .commands {
            NoteCommands()
        }

        Settings {
            SettingsView()
        }
    }
}

struct ContentView: View {
    @State private var notes: [Note] = []
    @State private var selectedNote: Note?

    var body: some View {
        NavigationSplitView {
            List(notes, selection: $selectedNote) { note in
                Text(note.title)
                    .tag(note)
            }
            .listStyle(.sidebar)
            .frame(minWidth: 200)
        } detail: {
            if let note = selectedNote {
                NoteEditorView(note: note)
            } else {
                Text("选择一篇笔记")
                    .frame(maxWidth: .infinity, maxHeight: .infinity)
            }
        }
    }
}
```

### 5.2 WindowGroup 与多窗口

macOS 原生支持多窗口，SwiftUI 通过 `WindowGroup` 实现：

```swift
@main
struct MacNotesApp: App {
    var body: some Scene {
        WindowGroup(for: Note.ID.self) { $noteID in
            if let noteID = noteID {
                NoteEditorView(noteID: noteID)
            } else {
                NoteListView()
            }
        }

        Window("快速记录", id: "quick-note") {
            QuickNoteView()
        }
        .defaultSize(width: 300, height: 200)
    }
}
```

> 💡 `WindowGroup(for:)` 允许每个窗口关联不同的数据，双击列表项即可在新窗口中打开。

### 5.3 菜单命令

SwiftUI 的 `.commands()` 修饰器用于构建菜单栏：

```swift
struct NoteCommands: Commands {
    var body: some Commands {
        CommandGroup(replacing: .newItem) {
            Button("新建笔记") {
                NotificationCenter.default.post(name: .newNote, object: nil)
            }
            .keyboardShortcut("n", modifiers: .command)

            Button("新建文件夹") {
                NotificationCenter.default.post(name: .newFolder, object: nil)
            }
            .keyboardShortcut("n", modifiers: [.command, .shift])
        }

        CommandMenu("格式") {
            Button("加粗") {
                NotificationCenter.default.post(name: .toggleBold, object: nil)
            }
            .keyboardShortcut("b", modifiers: .command)

            Button("斜体") {
                NotificationCenter.default.post(name: .toggleItalic, object: nil)
            }
            .keyboardShortcut("i", modifiers: .command)
        }
    }
}
```

### 5.4 macOS 特有组件

```swift
struct SettingsView: View {
    @AppStorage("fontSize") private var fontSize: Double = 14
    @AppStorage("showLineNumbers") private var showLineNumbers = true
    @AppStorage("theme") private var theme = "auto"

    var body: some View {
        Form {
            TextField("作者名称", text: .constant(""))

            Picker("主题", selection: $theme) {
                Text("跟随系统").tag("auto")
                Text("浅色").tag("light")
                Text("深色").tag("dark")
            }

            Toggle("显示行号", isOn: $showLineNumbers)

            Slider(value: $fontSize, in: 10...24, step: 1) {
                Text("字体大小: \(Int(fontSize))pt")
            }
        }
        .frame(width: 400, height: 300)
    }
}
```

> 💡 macOS 的 `Settings` 场景会自动出现在应用菜单的"偏好设置…"中（Cmd+,），无需手动添加入口。

---

## 6. Mac App Store 上架

### 6.1 沙盒要求

所有 Mac App Store 应用**必须**启用 App Sandbox：

| 权限 | Entitlement 键 | 说明 |
|------|----------------|------|
| 文件读取 | `com.apple.security.files.user-selected.read-only` | 只读访问用户选择的文件 |
| 文件读写 | `com.apple.security.files.user-selected.read-write` | 读写用户选择的文件 |
| 网络 | `com.apple.security.network.client` | 出站网络连接 |
| 网络（入站） | `com.apple.security.network.server` | 入站网络连接 |
| 摄像头 | `com.apple.security.device.camera` | 访问摄像头 |
| 麦克风 | `com.apple.security.device.audio-input` | 访问麦克风 |

在 Target → **Signing & Capabilities** → **+ Capability** → **App Sandbox** 中启用。

### 6.2 Hardened Runtime

Hardened Runtime 是 macOS 的安全机制，限制动态代码加载和内存注入：

```
Target → Signing & Capabilities
  └── Hardened Runtime: ✅ 勾选
        ├── Disable Library Validation（如需加载第三方插件）
        ├── Allow Dyld Environment Variables（调试用）
        └── Allow Unsigned Executable Memory（JIT 编译需要）
```

> ⚠️ 如果你的 App 使用了网络、摄像头等权限，必须同时启用 Hardened Runtime 和对应的 Entitlement，否则 App 会崩溃。

### 6.3 公证（Notarization）

从 macOS 10.15 起，所有分发到 Mac 上的 App（包括 App Store 外的分发）都需要通过 Apple 公证：

```bash
# 1. 归档
xcodebuild archive \
    -scheme MyMacApp \
    -archivePath build/MyMacApp.xcarchive

# 2. 导出
xcodebuild -exportArchive \
    -archivePath build/MyMacApp.xcarchive \
    -exportOptionsPlist ExportOptions.plist \
    -exportPath build/export

# 3. 提交公证（App Store 分发时 Xcode 自动处理）
xcrun notarytool submit build/export/MyMacApp.pkg \
    --apple-id "dev@example.com" \
    --team-id "TEAMID" \
    --password "app-specific-password" \
    --wait
```

> 💡 通过 App Store 分发的 App 会自动完成公证，无需手动操作。公证主要针对 Mac App Store 之外的分发方式。

### 6.4 App Review 差异

Mac App Store 审核与 iOS 有以下关键差异：

| 审核要点 | iOS | macOS |
|----------|-----|-------|
| 沙盒 | 推荐 | **强制** |
| 文件访问 | 自由 | 必须通过 Open/Save Panel |
| 后台运行 | 严格限制 | 相对宽松 |
| 退出方式 | 无需处理 | 必须支持 Cmd+Q |
| 窗口行为 | 无 | 必须支持缩放、最小化 |
| 帮助文档 | 可选 | 强烈建议提供 |

---

## 7. Catalyst vs 原生 macOS 对比

| 维度 | Mac Catalyst | 原生 macOS (SwiftUI/AppKit) |
|------|-------------|---------------------------|
| **性能** | 多一层 UIKit→AppKit 映射，有少量损耗 | 原生性能，无额外开销 |
| **用户体验** | iOS 风格为主，部分 Mac 化 | 完全原生 Mac 体验 |
| **API 覆盖** | UIKit 子集 + 部分 AppKit 桥接 | 全部 macOS API |
| **开发成本** | 低，复用 90%+ iOS 代码 | 高，需大量重写 |
| **维护成本** | 一套代码维护 | 需维护独立 macOS 版本 |
| **多窗口** | 有限支持（macOS 13+ 改善） | 完整支持 |
| **菜单系统** | 通过 UIMenuBuilder 定制 | 完全自由定制 |
| **系统集成** | 受限 | 深度集成（Finder 扩展、系统服务等） |
| **Touch Bar** | 需手动桥接 NSTouchBar | 原生支持 |
| **拖拽** | 基础支持 | 完整的跨 App 拖拽 |
| **辅助功能** | 自动映射，部分缺失 | 完整 VoiceOver 支持 |
| **分发** | Mac App Store + Mac App Store 外 | 同左 |

> 💡 选择建议：如果你的 App 是工具类且 UI 简单，Catalyst 是性价比最高的选择；如果需要深度 Mac 体验或复杂系统集成，直接走原生路线。

---

## 8. 实战：将 iOS App 通过 Catalyst 移植到 Mac

### 8.1 项目准备

假设我们有一个 iOS 笔记 App "QuickNote"，现在要移植到 Mac。

**步骤 1：启用 Catalyst**

在 Xcode 中：
1. 选择 QuickNote Target → General → Supported Destinations
2. 添加 **Mac (Designed for iPad)**
3. 选择优化级别为 **Optimized for Mac**

**步骤 2：解决编译错误**

```swift
// 修复 1：UIDevice 不可用的属性
extension UIDevice {
    static var isMac: Bool {
        #if targetEnvironment(macCatalyst)
        return true
        #else
        return false
        #endif
    }
}

// 修复 2：替代不可用的 API
class DeviceInfo {
    var deviceModel: String {
        #if targetEnvironment(macCatalyst)
        return "Mac (Catalyst)"
        #else
        return UIDevice.current.model
        #endif
    }
}
```

### 8.2 适配键盘与菜单

```swift
class NoteEditorViewController: UIViewController {

    override func viewDidLoad() {
        super.viewDidLoad()
        configureForMac()
    }

    private func configureForMac() {
        #if targetEnvironment(macCatalyst)
        setupKeyCommands()
        #endif
    }

    #if targetEnvironment(macCatalyst)
    private func setupKeyCommands() {
        let commands: [UIKeyCommand] = [
            UIKeyCommand(input: "s", modifierFlags: .command,
                         action: #selector(saveNote),
                         discoverabilityTitle: "保存"),
            UIKeyCommand(input: "b", modifierFlags: .command,
                         action: #selector(toggleBold),
                         discoverabilityTitle: "加粗"),
            UIKeyCommand(input: "i", modifierFlags: .command,
                         action: #selector(toggleItalic),
                         discoverabilityTitle: "斜体"),
            UIKeyCommand(input: "f", modifierFlags: [.command, .option],
                         action: #selector(findAndReplace),
                         discoverabilityTitle: "查找和替换"),
        ]
        addKeyCommands(commands)
    }
    #endif

    @objc private func saveNote() { /* 保存逻辑 */ }
    @objc private func toggleBold() { /* 加粗逻辑 */ }
    @objc private func toggleItalic() { /* 斜体逻辑 */ }
    @objc private func findAndReplace() { /* 查找替换逻辑 */ }

    override func buildMenu(with builder: UIMenuBuilder) {
        super.buildMenu(with: builder)
        guard builder.system == .main else { return }

        #if targetEnvironment(macCatalyst)
        let exportPDF = UICommand(title: "导出为 PDF",
                                  action: #selector(exportPDF))
        let exportMarkdown = UICommand(title: "导出为 Markdown",
                                       action: #selector(exportMarkdown))

        let exportMenu = UIMenu(title: "导出",
                                identifier: UIMenu.Identifier("com.quicknote.export"),
                                children: [exportPDF, exportMarkdown])

        builder.insertChild(exportMenu, atStartOfMenu: .file)
        #endif
    }

    @objc private func exportPDF() { /* PDF 导出 */ }
    @objc private func exportMarkdown() { /* Markdown 导出 */ }
}
```

### 8.3 适配窗口大小

```swift
class NoteListViewController: UIViewController {

    private var notesCollectionView: UICollectionView!
    private var dataSource: UICollectionViewDiffableDataSource<Section, Note>!

    override func viewDidLoad() {
        super.viewDidLoad()
        setupCollectionView()
        observeWindowSize()
    }

    private func observeWindowSize() {
        registerForTraitChanges([UITraitHorizontalSizeClass.self]) { (self: Self, _) in
            self.updateLayout()
        }
    }

    private func updateLayout() {
        let isWide = traitCollection.horizontalSizeClass == .regular
        let columnCount = isWide ? 3 : 1

        let layout = UICollectionViewCompositionalLayout { _, _ in
            let itemSize = NSCollectionLayoutSize(
                widthDimension: .fractionalWidth(1.0 / CGFloat(columnCount)),
                heightDimension: .estimated(120)
            )
            let item = NSCollectionLayoutItem(layoutSize: itemSize)
            item.contentInsets = NSDirectionalEdgeInsets(top: 4, leading: 4, bottom: 4, trailing: 4)

            let groupSize = NSCollectionLayoutSize(
                widthDimension: .fractionalWidth(1.0),
                heightDimension: .estimated(120)
            )
            let group = NSCollectionLayoutItem(layoutSize: groupSize)
            let horizontalGroup = NSCollectionLayoutGroup.horizontal(
                layoutSize: groupSize,
                subitem: item,
                count: columnCount
            )

            let section = NSCollectionLayoutSection(group: horizontalGroup)
            section.contentInsets = NSDirectionalEdgeInsets(top: 8, leading: 8, bottom: 8, trailing: 8)
            return section
        }

        notesCollectionView.collectionViewLayout = layout
    }
}
```

### 8.4 添加沙盒权限

在 **Signing & Capabilities** 中添加：

```
App Sandbox
  ├── Outgoing Connections (Client) ✅  ← 网络同步
  ├── File Read/Write (User Selected) ✅  ← 导出文件
  └── Camera ✅  ← 拍照插入笔记
```

对应的 `.entitlements` 文件：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>com.apple.security.app-sandbox</key>
    <true/>
    <key>com.apple.security.network.client</key>
    <true/>
    <key>com.apple.security.files.user-selected.read-write</key>
    <true/>
    <key>com.apple.security.device.camera</key>
    <true/>
</dict>
</plist>
```

### 8.5 最终检查清单

| 检查项 | 状态 |
|--------|------|
| Target 已添加 Mac (Catalyst) 目标 | ☐ |
| 优化级别已选择 Optimized for Mac | ☐ |
| 所有编译错误已修复 | ☐ |
| 键盘快捷键已添加（Cmd+S 等） | ☐ |
| 菜单栏已自定义 | ☐ |
| 鼠标悬停效果已实现 | ☐ |
| 窗口大小适配已测试 | ☐ |
| 隐私权限已在 Info.plist 声明 | ☐ |
| App Sandbox 已启用并配置权限 | ☐ |
| Hardened Runtime 已启用 | ☐ |
| Cmd+Q 退出行为正常 | ☐ |
| Mac App Store 截图已准备 | ☐ |

---

## 本章小结

| 主题 | 核心要点 |
|------|----------|
| Mac Catalyst 概述 | UIKit 兼容层让 iPad App 运行在 Mac 上，适合工具类和内容消费类 App |
| 启用 Catalyst | Xcode 添加 Mac 目标，选择 Scaled 或 Optimized 优化级别 |
| 适配要点 | 键盘快捷键（UIKeyCommand）、菜单栏（UIMenuBuilder）、鼠标悬停（UIHoverGestureRecognizer）、窗口大小、工具栏 |
| UIKit→AppKit 桥接 | `#if targetEnvironment(macCatalyst)` 条件编译、NSTouchBar、NSMenu、不可用 API 降级 |
| 原生 macOS 开发 | SwiftUI WindowGroup、菜单命令（.commands()）、macOS 特有组件（Settings、Form） |
| Mac App Store 上架 | 强制沙盒、Hardened Runtime、公证、App Review 差异（文件访问、窗口行为） |
| Catalyst vs 原生 | Catalyst 开发成本低但体验受限；原生开发成本高但体验完整 |
| 实战移植 | 启用目标→修复编译→适配交互→配置权限→测试上架 |

> 💡 总体策略：先用 Catalyst 快速验证 Mac 市场，如果用户反馈良好且需求深入，再考虑投入资源做原生 macOS 版本。技术选型的核心不是"哪个更好"，而是"哪个更适合当前阶段"。

← [-watchOS 快速入门](./122-watchOS快速入门.md) | [-tvOS 快速入门](./124-tvOS快速入门.md) →
