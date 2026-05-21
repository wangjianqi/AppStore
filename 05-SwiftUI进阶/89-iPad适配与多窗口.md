# 89-iPad 适配与多窗口

## 本章目标

- 理解 iPad 适配的重要性及 App Store 审核要求
- 掌握 SwiftUI 响应式布局核心工具（GeometryReader、size classes、adaptive modifiers）
- 学会使用 NavigationSplitView 实现双栏/三栏布局
- 实现 iPad 多窗口支持与窗口间数据共享
- 适配 Stage Manager 场景下的窗口尺寸变化
- 为 iPad 添加键盘快捷键与鼠标/触控板交互支持
- 实现跨 App 拖放功能（Drag and Drop）
- 掌握 iPad 与 iPhone 适配策略的差异与取舍
- 实战：将 iPhone App 适配为 iPad 通用版本

---

## 1. iPad 适配的重要性

💡 **通俗理解**：如果你的 App 只在 iPhone 上"穿着合身"，到了 iPad 上就像穿了一件小号衣服套在大个子身上——到处露馅。iPad 适配就是给你的 App 做一套"大号定制西装"，让它在大屏幕上也体面好看。

### 1.1 iPad 市场份额与趋势

| 指标 | 数据 |
|------|------|
| iPad 活跃设备数 | 超过 10 亿台 |
| iPad 在平板市场占有率 | 约 40% |
| iPadOS App 下载量占比 | 约占 iOS 生态 20% |
| 支持 iPad 的通用购买比例 | 超过 60% 的付费 App |

### 1.2 Apple 的适配要求

Apple 对 iPad 适配有明确的审核要求：

| 要求 | 说明 |
|------|------|
| **必须支持所有尺寸** | App 必须在所有 iPad 尺寸下正常运行，不能崩溃或布局错乱 |
| **禁止 iPhone 模式拉伸** | iPadOS 16 起，App 不再默认以 iPhone 兼容模式运行 |
| **通用购买** | iPhone 和 iPad 版本必须作为同一个 App 提供，不能分开收费 |
| **键盘与鼠标** | 如果支持外接键盘/鼠标，必须正确处理焦点与交互 |
| **多窗口** | 文档类 App 应支持多窗口 |

> ⚠️ **审核注意**：如果你的 App 在 iPad 上只是 iPhone 版的 2x 放大，很可能被审核拒绝。Apple 要求"在所有提交的设备上提供良好的用户体验"。

### 1.3 不适配的后果

```
不适配的连锁反应：
布局错乱 → 用户体验差 → 差评增加 → 排名下降 → 下载量减少
     ↓
审核被拒 → 反复修改 → 上线延迟 → 错过市场窗口
```

---

## 2. SwiftUI 响应式布局基础

💡 **通俗理解**：响应式布局就像一件"弹力衣"——不管穿的人是胖是瘦，都能自动贴合身形。你不需要为每种身材做一件衣服，一件弹力衣就够了。

### 2.1 GeometryReader 获取可用空间

```swift
struct ResponsiveView: View {
    var body: some View {
        GeometryReader { geometry in
            let columns = geometry.size.width > 700 ? 3 : 2

            LazyVGrid(columns: Array(repeating: GridItem(.flexible()),
                                     count: columns)) {
                ForEach(0..<12) { index in
                    RoundedRectangle(cornerRadius: 12)
                        .fill(Color.blue.opacity(0.2))
                        .frame(height: 120)
                        .overlay(Text("项目 \(index + 1)"))
                }
            }
            .padding()
        }
    }
}
```

| GeometryReader 属性 | 类型 | 说明 |
|---------------------|------|------|
| `size` | CGSize | 容器的可用尺寸 |
| `safeAreaInsets` | EdgeInsets | 安全区域边距 |
| `frame` | CGRect | 在全局坐标中的位置 |

> ⚠️ **警告**：GeometryReader 会贪婪地占据所有可用空间。不要滥用，否则会导致布局膨胀。尽量只在需要判断尺寸的层级使用。

### 2.2 Size Classes 尺寸类别

Size Classes 是 Apple 提供的设备尺寸抽象，将屏幕尺寸归纳为"紧凑"和"常规"两类：

| 尺寸类别 | 水平方向 | 垂直方向 | 典型设备 |
|----------|---------|---------|---------|
| **compact × compact** | 紧凑 | 紧凑 | iPhone 横屏 |
| **compact × regular** | 紧凑 | 常规 | iPhone 竖屏 |
| **regular × compact** | 常规 | 紧凑 | iPad 分屏 1/3 |
| **regular × regular** | 常规 | 常规 | iPad 全屏 |

```swift
struct AdaptiveLayoutView: View {
    @Environment(\.horizontalSizeClass) var horizontalSizeClass
    @Environment(\.verticalSizeClass) var verticalSizeClass

    var body: some View {
        if horizontalSizeClass == .regular {
            iPadLayout()
        } else {
            iPhoneLayout()
        }
    }
}
```

### 2.3 Adaptive Modifiers 自适应修饰符

SwiftUI 提供了一系列根据 size class 自动调整的修饰符：

```swift
struct AdaptiveModifiersView: View {
    var body: some View {
        NavigationStack {
            List(1..<20, id: \.self) { item in
                Text("条目 \(item)")
            }
            .navigationTitle("列表")
        }
        .listStyle(.automatic)
        .formStyle(.automatic)
        .navigationViewStyle(.automatic)
    }
}
```

| 修饰符 | iPad 行为 | iPhone 行为 |
|--------|----------|------------|
| `.listStyle(.automatic)` | `.insetGrouped` 或 `.sidebar` | `.insetGrouped` |
| `.navigationViewStyle(.automatic)` | 双栏/三栏 | 单栏堆叠 |
| `.formStyle(.automatic)` | 分组表单 | 分组表单 |

> 💡 **提示**：优先使用 `.automatic` 样式，让系统根据设备自动选择最佳表现。只在需要精确控制时才指定具体样式。

---

## 3. NavigationSplitView 双栏/三栏布局

💡 **通俗理解**：NavigationSplitView 就像一个"三抽屉文件柜"——左边抽屉放分类目录，中间抽屉放文件列表，右边抽屉放文件详情。在 iPad 上三个抽屉同时可见，在 iPhone 上则变成"一次只开一个抽屉"的堆叠模式。

### 3.1 基本三栏布局

```swift
struct ThreeColumnView: View {
    @State private var selectedCategory: Category?
    @State private var selectedItem: Item?

    var body: some View {
        NavigationSplitView {
            List(Category.allCases, selection: $selectedCategory) { category in
                Label(category.name, systemImage: category.icon)
                    .tag(category)
            }
            .navigationTitle("分类")
        } content: {
            if let category = selectedCategory {
                List(category.items, selection: $selectedItem) { item in
                    Text(item.name)
                        .tag(item)
                }
                .navigationTitle(category.name)
            } else {
                Text("请选择分类")
            }
        } detail: {
            if let item = selectedItem {
                ItemDetailView(item: item)
            } else {
                Text("请选择项目")
            }
        }
        .navigationSplitViewStyle(.balanced)
    }
}
```

### 3.2 NavigationSplitView 样式对比

| 样式 | 侧边栏宽度 | 适用场景 |
|------|-----------|---------|
| `.automatic` | 系统决定 | 默认选择，适配所有设备 |
| `.balanced` | 中等宽度 | 侧边栏与内容区平衡展示 |
| `.prominentDetail` | 侧边栏较窄 | 详情内容更重要时使用 |
| `.detailOnly` | 隐藏侧边栏 | 仅显示详情，侧边栏按需滑出 |

### 3.3 双栏布局（侧边栏 + 详情）

```swift
struct TwoColumnView: View {
    @State private var selectedFolder: Folder?

    var body: some View {
        NavigationSplitView {
            List(Folder.allCases, selection: $selectedFolder) { folder in
                Label(folder.name, systemImage: folder.icon)
                    .tag(folder)
            }
            .navigationTitle("文件夹")
        } detail: {
            if let folder = selectedFolder {
                FolderDetailView(folder: folder)
            } else {
                ContentUnavailableView("未选择",
                                       systemImage: "folder",
                                       description: Text("从侧边栏选择一个文件夹"))
            }
        }
    }
}
```

### 3.4 控制侧边栏显示与隐藏

```swift
struct SidebarControlView: View {
    @State private var columnVisibility: NavigationSplitViewVisibility = .all
    @State private var selectedItem: Item?

    var body: some View {
        NavigationSplitView(columnVisibility: $columnVisibility) {
            List(Item.samples, selection: $selectedItem) { item in
                Text(item.name).tag(item)
            }
            .navigationTitle("项目")
        } detail: {
            if let item = selectedItem {
                Text(item.description)
            }
        }
        .toolbar {
            ToolbarItem(placement: .navigationBarLeading) {
                Button("切换侧边栏") {
                    withAnimation {
                        columnVisibility = columnVisibility == .all ? .detailOnly : .all
                    }
                }
            }
        }
    }
}
```

| Visibility 值 | 效果 |
|---------------|------|
| `.all` | 显示所有列 |
| `.doubleColumn` | 仅显示双栏 |
| `.detailOnly` | 仅显示详情列 |
| `.automatic` | 系统根据空间自动决定 |

> ⚠️ **警告**：在 iPhone 上，NavigationSplitView 会自动退化为 NavigationStack 的堆叠导航模式。不要假设侧边栏始终可见。

---

## 4. iPad 多窗口支持

💡 **通俗理解**：多窗口就像你在桌上同时打开了两本笔记本——一本写日记，一本做计划。两本可以独立翻页，但都属于你。在 iPad 上，用户可以同时打开多个窗口来处理不同内容。

### 4.1 WindowGroup 基础

```swift
@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }

        WindowGroup(for: Document.self) { $document in
            DocumentEditorView(document: $document)
        }
    }
}
```

| Scene 类型 | 说明 | 多窗口 |
|-----------|------|--------|
| `WindowGroup` | 主窗口组 | 默认支持多窗口 |
| `WindowGroup(for:)` | 文档类型窗口 | 每个文档一个窗口 |
| `Window` | 单例窗口 | 只能有一个实例 |

### 4.2 打开新窗口

```swift
struct ContentView: View {
    @Environment(\.openWindow) var openWindow

    var body: some View {
        List {
            Button("打开新窗口") {
                openWindow(id: "main")
            }

            Button("打开文档") {
                let doc = Document(title: "新建文档")
                openWindow(value: doc)
            }
        }
    }
}
```

> 💡 **提示**：`openWindow(id:)` 用于打开普通 WindowGroup，`openWindow(value:)` 用于打开带数据类型的 WindowGroup。

### 4.3 多窗口数据共享

多窗口之间需要共享数据，否则各窗口会变成"信息孤岛"：

```swift
@MainActor
class SharedStore: ObservableObject {
    static let shared = SharedStore()
    @Published var documents: [Document] = []

    private init() {}
}

struct DocumentListView: View {
    @StateObject private var store = SharedStore.shared

    var body: some View {
        List(store.documents) { doc in
            NavigationLink(doc.title, value: doc)
        }
    }
}
```

对于更复杂的多窗口同步，可以使用 `@AppStorage` 或文件系统：

```swift
struct SettingsView: View {
    @AppStorage("fontSize") var fontSize: Double = 16
    @AppStorage("theme") var theme: String = "light"

    var body: some View {
        Form {
            Slider(value: $fontSize, in: 12...24) {
                Text("字体大小: \(Int(fontSize))")
            }
            Picker("主题", selection: $theme) {
                Text("浅色").tag("light")
                Text("深色").tag("dark")
            }
        }
    }
}
```

> ⚠️ **警告**：`@AppStorage` 的变更通知只在同一进程内生效。在 iPadOS 上，同一 App 的多个窗口共享同一进程，因此 `@AppStorage` 可以在窗口间同步。但如果使用 App Group 跨进程通信，需要配合 `UserDefaults` 的 KVO 监听。

---

## 5. Stage Manager 适配

💡 **通俗理解**：Stage Manager 就像一个"可伸缩的办公桌"——桌面空间有限时，你的文件会被缩小放在一边；空间充裕时，文件可以铺开。你的 App 需要像弹性文件夹一样，随时适应桌面给的任何尺寸。

### 5.1 Stage Manager 简介

Stage Manager 是 iPadOS 16 引入的窗口管理方式，允许用户自由调整 App 窗口大小：

| 特性 | 说明 |
|------|------|
| **自由调整** | 用户可以拖拽调整窗口大小 |
| **窗口组** | 多个 App 窗口可以叠放为一组 |
| **外接显示器** | 支持外接显示器扩展桌面 |
| **最小尺寸** | 窗口有最小尺寸限制（约 1/3 屏宽） |

### 5.2 适配窗口尺寸变化

```swift
struct StageManagerAdaptiveView: View {
    @Environment(\.horizontalSizeClass) var hSizeClass
    @State private var columnVisibility: NavigationSplitViewVisibility = .automatic

    var body: some View {
        NavigationSplitView(columnVisibility: $columnVisibility) {
            SidebarView()
        } detail: {
            DetailView()
        }
        .onChange(of: hSizeClass) { _, newValue in
            withAnimation {
                columnVisibility = newValue == .compact ? .detailOnly : .all
            }
        }
    }
}
```

### 5.3 使用 WindowInsets 处理安全区域

```swift
struct SafeAreaAwareView: View {
    var body: some View {
        GeometryReader { geometry in
            ScrollView {
                VStack(spacing: 16) {
                    ForEach(0..<30) { i in
                        Text("内容行 \(i)")
                            .frame(maxWidth: .infinity)
                            .padding()
                            .background(Color.blue.opacity(0.1))
                            .clipShape(RoundedRectangle(cornerRadius: 8))
                    }
                }
                .padding()
            }
            .safeAreaInset(edge: .bottom) {
                BottomToolbar()
            }
        }
    }
}
```

> 💡 **提示**：在 Stage Manager 下，窗口可能比全屏小很多。始终使用 `.safeAreaInset` 和 GeometryReader 来动态调整布局，而不是硬编码尺寸。

---

## 6. 键盘与鼠标/触控板支持

💡 **通俗理解**：iPad 接上键盘和鼠标后，就从"平板"变成了"电脑"。你的 App 需要像一个懂礼貌的管家——当客人用键盘时，用快捷键招呼；当客人用鼠标时，用悬停高亮回应。

### 6.1 键盘快捷键

```swift
struct KeyboardShortcutView: View {
    @State private var showNewDoc = false
    @State private var searchText = ""

    var body: some View {
        NavigationStack {
            List(1..<20, id: \.self) { i in
                Text("文档 \(i)")
            }
            .searchable(text: $searchText)
            .navigationTitle("文档")
        }
        .keyboardShortcut("n", modifiers: .command) {
            showNewDoc = true
        }
        .keyboardShortcut(",", modifiers: .command) {
            // 打开设置
        }
        .sheet(isPresented: $showNewDoc) {
            Text("新建文档")
        }
    }
}
```

常用键盘快捷键约定：

| 快捷键 | 功能 | 说明 |
|--------|------|------|
| ⌘N | 新建 | 新建文档/项目 |
| ⌘S | 保存 | 保存当前文档 |
| ⌘F | 搜索 | 搜索内容 |
| ⌘, | 设置 | 打开设置页面 |
| ⌘W | 关闭窗口 | 关闭当前窗口 |
| ⌘Z | 撤销 | 撤销操作 |

### 6.2 鼠标悬停效果

```swift
struct HoverItemView: View {
    let title: String
    @State private var isHovering = false

    var body: some View {
        HStack {
            Image(systemName: "doc.fill")
            Text(title)
            Spacer()
            Image(systemName: "chevron.right")
                .opacity(isHovering ? 1 : 0)
        }
        .padding(10)
        .background(isHovering ? Color.blue.opacity(0.1) : Color.clear)
        .clipShape(RoundedRectangle(cornerRadius: 8))
        .onHover { hovering in
            withAnimation(.easeInOut(duration: 0.15)) {
                isHovering = hovering
            }
        }
    }
}
```

### 6.3 连续悬停追踪

```swift
struct ContinuousHoverCanvas: View {
    @State private var pointerLocation: CGPoint = .zero

    var body: some View {
        Canvas { context, size in
            let radius: CGFloat = 20
            context.fill(
                Path(ellipseIn: CGRect(x: pointerLocation.x - radius,
                                       y: pointerLocation.y - radius,
                                       width: radius * 2,
                                       height: radius * 2)),
                with: .color(.blue.opacity(0.3))
            )
        }
        .frame(maxWidth: .infinity, maxHeight: .infinity)
        .contentShape(Rectangle())
        .onContinuousHover { phase in
            switch phase {
            case .active(let location):
                pointerLocation = location
            case .ended:
                pointerLocation = .zero
            }
        }
    }
}
```

### 6.4 右键菜单（Context Menu）

```swift
struct ContextMenuView: View {
    @State private var items = ["文档A", "文档B", "文档C"]

    var body: some View {
        List {
            ForEach(items, id: \.self) { item in
                Text(item)
                    .contextMenu {
                        Button {
                            // 打开
                        } label: {
                            Label("打开", systemImage: "doc")
                        }

                        Button {
                            // 复制
                        } label: {
                            Label("复制", systemImage: "doc.on.doc")
                        }

                        Divider()

                        Button(role: .destructive) {
                            items.removeAll { $0 == item }
                        } label: {
                            Label("删除", systemImage: "trash")
                        }
                    }
            }
        }
    }
}
```

> 💡 **提示**：在 iPad 上，右键菜单可以通过鼠标右键或长按触发。确保两种交互方式都能正常工作。

---

## 7. Drag and Drop 跨 App 拖放

💡 **通俗理解**：拖放就像"传递纸条"——你在备忘录里写了一段话，直接拖到邮件 App 里发送。不需要复制粘贴，手指一拖就过去了。跨 App 拖放让 iPad 的 App 之间像同事一样协作。

### 7.1 onDrop 接收拖放

```swift
struct DropTargetView: View {
    @State private var droppedText: String = "将内容拖放到这里"
    @State private var isTargeted = false

    var body: some View {
        VStack(spacing: 20) {
            Image(systemName: "arrow.down.doc.fill")
                .font(.system(size: 48))
                .foregroundStyle(isTargeted ? .blue : .gray)

            Text(droppedText)
                .font(.headline)
        }
        .frame(maxWidth: .infinity, maxHeight: .infinity)
        .background(isTargeted ? Color.blue.opacity(0.1) : Color.gray.opacity(0.05))
        .clipShape(RoundedRectangle(cornerRadius: 16))
        .padding()
        .onDrop(of: [.text, .url], isTargeted: $isTargeted) { providers in
            handleDrop(providers: providers)
            return true
        }
    }

    private func handleDrop(providers: [NSItemProvider]) {
        guard let provider = providers.first else { return }

        if provider.hasItemConformingToTypeIdentifier(UTType.text.identifier) {
            provider.loadItem(forTypeIdentifier: UTType.text.identifier) { data, _ in
                if let textData = data as? Data,
                   let text = String(data: textData, encoding: .utf8) {
                    DispatchQueue.main.async {
                        droppedText = text
                    }
                }
            }
        }
    }
}
```

### 7.2 transferable 拖出数据

```swift
struct DraggableItemView: View {
    let note: Note

    var body: some View {
        HStack {
            Image(systemName: "note.text")
            Text(note.title)
        }
        .padding(8)
        .background(Color.yellow.opacity(0.2))
        .clipShape(RoundedRectangle(cornerRadius: 8))
        .draggable(note.content) {
            Text(note.title)
                .padding()
                .background(Color.yellow.opacity(0.3))
                .clipShape(RoundedRectangle(cornerRadius: 8))
        }
    }
}
```

### 7.3 使用 UTType 支持多种数据类型

```swift
struct AdvancedDropView: View {
    @State private var receivedImage: UIImage?
    @State private var receivedText: String = ""

    var body: some View {
        VStack {
            if let image = receivedImage {
                Image(uiImage: image)
                    .resizable()
                    .scaledToFit()
                    .frame(height: 200)
            }

            Text(receivedText)
        }
        .frame(maxWidth: .infinity, maxHeight: .infinity)
        .onDrop(of: [.image, .text, .pdf, .url]) { providers in
            for provider in providers {
                if provider.hasItemConformingToTypeIdentifier(UTType.image.identifier) {
                    provider.loadItem(forTypeIdentifier: UTType.image.identifier) { data, _ in
                        if let url = data as? URL,
                           let image = UIImage(contentsOfFile: url.path) {
                            DispatchQueue.main.async { receivedImage = image }
                        }
                    }
                }
                if provider.hasItemConformingToTypeIdentifier(UTType.text.identifier) {
                    provider.loadItem(forTypeIdentifier: UTType.text.identifier) { data, _ in
                        if let data = data as? Data,
                           let text = String(data: data, encoding: .utf8) {
                            DispatchQueue.main.async { receivedText = text }
                        }
                    }
                }
            }
            return true
        }
    }
}
```

| UTType 常用类型 | 说明 |
|----------------|------|
| `.text` | 纯文本 |
| `.plainText` | 纯文本（更具体） |
| `.url` | URL 链接 |
| `.image` | 图片 |
| `.png` / `.jpeg` | 特定图片格式 |
| `.pdf` | PDF 文档 |
| `.fileURL` | 文件路径 |

> ⚠️ **警告**：拖放操作是异步的，`loadItem` 的回调不在主线程。更新 UI 时必须切换到主线程。

---

## 8. iPad vs iPhone 适配策略对比

💡 **通俗理解**：iPad 和 iPhone 就像"大房子"和"小公寓"——大房子可以分多个房间（多栏布局），小公寓只能用折叠家具（堆叠导航）。但无论住哪里，核心生活需求是一样的。

| 维度 | iPhone 策略 | iPad 策略 |
|------|------------|----------|
| **布局** | 单栏垂直滚动，紧凑排列 | 多栏并排，充分利用水平空间 |
| **导航** | NavigationStack 堆叠推入 | NavigationSplitView 侧边栏 + 详情 |
| **列表样式** | `.insetGrouped` | `.sidebar` 或 `.inset` |
| **交互** | 触摸为主，手势驱动 | 触摸 + 键盘 + 鼠标/触控板 |
| **输入** | 虚拟键盘 | 虚拟键盘 + 外接键盘 + 快捷键 |
| **多窗口** | 不支持 | WindowGroup 多实例 |
| **拖放** | App 内拖放 | 跨 App 拖放 |
| **上下文菜单** | 长按触发 | 长按 + 右键触发 |
| **悬停** | 不适用 | `onHover` 高亮反馈 |
| **分屏** | 不支持 | Slide Over / Split View / Stage Manager |
| **模态展示** | 全屏覆盖 | Popover 或 Sheet（半屏） |
| **工具栏** | 底部 TabBar | 顶部 Toolbar + 侧边栏 |

### 8.1 使用条件编译处理差异

```swift
struct ConditionalLayoutView: View {
    var body: some View {
        #if os(iOS)
        if UIDevice.current.userInterfaceIdiom == .pad {
            iPadRootView()
        } else {
            iPhoneRootView()
        }
        #endif
    }
}
```

> 💡 **提示**：优先使用 SwiftUI 的自适应组件（如 NavigationSplitView 的 `.automatic` 样式），而不是手动判断设备类型。这样代码更简洁，也能自动适配未来的新设备。

### 8.2 使用 Size Class 而非设备判断

```swift
struct SizeClassLayoutView: View {
    @Environment(\.horizontalSizeClass) var hSizeClass

    var body: some View {
        if hSizeClass == .regular {
            HStack {
                SidebarView()
                DetailView()
            }
        } else {
            NavigationStack {
                ContentView()
            }
        }
    }
}
```

> ⚠️ **警告**：不要用 `UIDevice.current.userInterfaceIdiom` 来决定布局。iPad 在分屏模式下可能变成 compact 尺寸，硬编码设备类型会导致布局错误。

---

## 9. 实战：将 iPhone App 适配为 iPad 通用版本

💡 **通俗理解**：这次实战就像把一间"小公寓"改造成"大房子"——不是简单地放大墙壁，而是重新规划空间：加一个客厅（侧边栏）、开辟书房（详情面板）、装上智能开关（键盘快捷键），让整个空间既宽敞又实用。

### 9.1 原始 iPhone 版本

```swift
struct NotesApp_iPhone: View {
    @State private var notes: [Note] = Note.samples
    @State private var selectedNote: Note?

    var body: some View {
        NavigationStack {
            List(notes, selection: $selectedNote) { note in
                NavigationLink {
                    NoteEditorView(note: note)
                } label: {
                    VStack(alignment: .leading) {
                        Text(note.title).font(.headline)
                        Text(note.preview).font(.subheadline).foregroundStyle(.secondary)
                    }
                }
            }
            .navigationTitle("备忘录")
        }
    }
}
```

### 9.2 适配后的 iPad 通用版本

```swift
struct NotesApp_Universal: View {
    @State private var notes: [Note] = Note.samples
    @State private var selectedNoteID: UUID?
    @State private var columnVisibility: NavigationSplitViewVisibility = .automatic

    var body: some View {
        NavigationSplitView(columnVisibility: $columnVisibility) {
            List(notes, selection: $selectedNoteID) { note in
                VStack(alignment: .leading) {
                    Text(note.title).font(.headline)
                    Text(note.preview)
                        .font(.subheadline)
                        .foregroundStyle(.secondary)
                        .lineLimit(1)
                }
                .tag(note.id)
            }
            .navigationTitle("备忘录")
            .listStyle(.sidebar)
            .toolbar {
                ToolbarItem(placement: .primaryAction) {
                    Button {
                        let newNote = Note(title: "新建备忘录", content: "")
                        notes.append(newNote)
                        selectedNoteID = newNote.id
                    } label: {
                        Label("新建", systemImage: "square.and.pencil")
                    }
                }
            }
        } detail: {
            if let noteID = selectedNoteID,
               let note = notes.first(where: { $0.id == noteID }) {
                NoteEditorView(note: binding(for: note))
                    .toolbar {
                        ToolbarItemGroup(placement: .primaryAction) {
                            Button {
                                // 分享
                            } label: {
                                Label("分享", systemImage: "square.and.arrow.up")
                            }
                            Button {
                                // 删除
                                notes.removeAll { $0.id == noteID }
                                selectedNoteID = nil
                            } label: {
                                Label("删除", systemImage: "trash")
                            }
                        }
                    }
            } else {
                ContentUnavailableView(
                    "选择备忘录",
                    systemImage: "note.text",
                    description: Text("从侧边栏选择一个备忘录开始编辑")
                )
            }
        }
        .navigationSplitViewStyle(.balanced)
    }

    private func binding(for note: Note) -> Binding<Note> {
        guard let index = notes.firstIndex(where: { $0.id == note.id }) else {
            fatalError("Note not found")
        }
        return $notes[index]
    }
}
```

### 9.3 添加键盘快捷键与右键菜单

```swift
struct NoteEditorView: View {
    @Binding var note: Note
    @State private var isHovering = false

    var body: some View {
        VStack(alignment: .leading, spacing: 0) {
            TextField("标题", text: $note.title)
                .font(.title.bold())
                .textFieldStyle(.plain)
                .padding()

            Divider()

            TextEditor(text: $note.content)
                .font(.body)
                .padding()
        }
        .keyboardShortcut("s", modifiers: .command) {
            saveNote()
        }
        .contextMenu {
            Button {
                copyNoteContent()
            } label: {
                Label("复制全部内容", systemImage: "doc.on.doc")
            }
            Button {
                shareNote()
            } label: {
                Label("分享", systemImage: "square.and.arrow.up")
            }
        }
    }

    private func saveNote() { }
    private func copyNoteContent() { }
    private func shareNote() { }
}
```

### 9.4 支持拖放排序

```swift
struct DraggableNotesList: View {
    @State private var notes: [Note] = Note.samples

    var body: some View {
        List {
            ForEach(notes) { note in
                HStack {
                    Image(systemName: "line.3.horizontal")
                        .foregroundStyle(.secondary)
                    VStack(alignment: .leading) {
                        Text(note.title).font(.headline)
                        Text(note.preview).font(.caption).foregroundStyle(.secondary)
                    }
                }
                .draggable(note.title) {
                    Text(note.title)
                        .padding(8)
                        .background(.regularMaterial)
                        .clipShape(RoundedRectangle(cornerRadius: 8))
                }
            }
            .onMove { from, to in
                notes.move(fromOffsets: from, toOffset: to)
            }
        }
    }
}
```

### 9.5 App 入口配置多窗口

```swift
@main
struct NotesApp: App {
    var body: some Scene {
        WindowGroup(for: Note.ID.self) { $noteID in
            NotesApp_Universal(selectedNoteID: noteID)
        }
    }
}
```

### 9.6 适配检查清单

| 检查项 | 状态 | 说明 |
|--------|------|------|
| NavigationSplitView 替代 NavigationStack | ☐ | iPad 使用侧边栏布局 |
| 列表样式改为 `.sidebar` | ☐ | 侧边栏使用 sidebar 样式 |
| 添加 ContentUnavailableView | ☐ | 详情区空状态提示 |
| 键盘快捷键 ⌘N / ⌘S | ☐ | 支持外接键盘操作 |
| 右键菜单 contextMenu | ☐ | 支持鼠标右键 |
| onHover 悬停反馈 | ☐ | 鼠标悬停高亮 |
| 拖放支持 draggable/drop | ☐ | 跨 App 拖放 |
| 多窗口 WindowGroup | ☐ | 文档类 App 支持多窗口 |
| Stage Manager 适配 | ☐ | 窗口缩放时布局正常 |
| Split View / Slide Over | ☐ | 1/3 和 1/2 分屏布局正常 |
| 不硬编码设备类型 | ☐ | 使用 Size Class 判断 |

---

## 本章小结

| 主题 | 核心要点 |
|------|---------|
| **iPad 适配重要性** | 市场份额大、Apple 审核要求、不适配会被拒 |
| **响应式布局** | GeometryReader 获取空间、Size Class 判断尺寸、优先用 `.automatic` |
| **NavigationSplitView** | 三栏（sidebar+content+detail）、控制 columnVisibility、iPhone 自动退化 |
| **多窗口** | WindowGroup 支持多实例、openWindow 打开新窗口、SharedStore 共享数据 |
| **Stage Manager** | iPadOS 16+ 自由调整窗口、用 Size Class 动态适配、注意安全区域 |
| **键盘与鼠标** | keyboardShortcut 快捷键、onHover 悬停、onContinuousHover 追踪、contextMenu 右键 |
| **Drag and Drop** | onDrop 接收、draggable/transferable 拖出、UTType 指定数据类型 |
| **适配策略** | 用 Size Class 而非设备类型判断、iPad 多栏 vs iPhone 堆叠、条件编译兜底 |
| **实战适配** | NavigationStack → NavigationSplitView、添加键盘/鼠标/拖放支持、多窗口配置 |
