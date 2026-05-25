# 76-SwiftUI性能优化专题

> 🎯 **本章目标**：深入理解 SwiftUI 的渲染机制，掌握识别和解决性能问题的方法，学会使用 Instruments 分析 SwiftUI 应用性能。

---

## SwiftUI 渲染原理

理解 SwiftUI 的渲染机制是性能优化的基础。SwiftUI 采用声明式 + 差异比较的渲染模型，与 UIKit 的命令式渲染有本质区别。

### 视图差异比较（Diffing）

SwiftUI 的渲染流程可以概括为三步：

1. **声明视图**：开发者用代码描述视图的状态和外观
2. **差异比较**：当状态变化时，SwiftUI 对比新旧视图树，找出差异
3. **更新 UI**：只更新有差异的部分

```swift
struct DiffingDemoView: View {
    @State private var count = 0

    var body: some View {
        VStack {
            Text("计数：\(count)")
                .font(.title)

            Text("这段文字不会变")
                .font(.body)

            Button("增加") {
                count += 1
            }
        }
    }
}
```

当 `count` 变化时，SwiftUI 的处理流程：

| 步骤 | 操作 | 说明 |
|------|------|------|
| 1 | 重新计算 `body` | 因为 `count` 是 `@State`，变化触发重算 |
| 2 | Diffing 对比 | 发现第一个 `Text` 的内容变了，其他没变 |
| 3 | 最小更新 | 只更新第一个 `Text`，其他视图保持不变 |

💡 **提示**：SwiftUI 的 diffing 是增量的，不是全量替换。它只会更新真正发生变化的部分，这是 SwiftUI 性能的基础保障。

### 视图身份（Identity）

SwiftUI 通过三种方式识别视图的身份：

#### 1. 结构化身份（Structural Identity）

由视图在视图树中的位置决定：

```swift
struct StructuralIdentityView: View {
    @State private var showDetail = false

    var body: some View {
        VStack {
            if showDetail {
                Text("详情内容")
            } else {
                Text("概览内容")
            }
        }
    }
}
```

⚠️ **警告**：`if-else` 会创建两个不同的视图身份。当 `showDetail` 切换时，SwiftUI 会销毁一个视图、创建另一个视图，而不是修改现有视图。

#### 2. 显式身份（Explicit Identity）

通过 `id()` 修饰符显式指定：

```swift
struct ExplicitIdentityView: View {
    @State private var items = ["A", "B", "C"]

    var body: some View {
        List {
            ForEach(items, id: \.self) { item in
                Text(item)
            }
        }
    }
}
```

当 `id` 变化时，SwiftUI 认为这是一个全新的视图，会销毁旧的、创建新的。

#### 3. 数据身份（Data Identity）

通过 `Identifiable` 协议提供：

```swift
struct TodoItem: Identifiable {
    let id: UUID
    var title: String
    var isDone: Bool
}

struct DataIdentityView: View {
    @State private var todos = [
        TodoItem(id: UUID(), title: "学习 SwiftUI", isDone: false),
        TodoItem(id: UUID(), title: "写代码", isDone: true)
    ]

    var body: some View {
        List(todos) { todo in
            Text(todo.title)
        }
    }
}
```

💡 **提示**：`Identifiable` 的 `id` 应该是稳定不变的。如果 `id` 随每次渲染变化（如使用 `UUID()` 在 `body` 中），SwiftUI 会认为这是新视图，导致不必要的销毁和重建。

### 身份对性能的影响

```swift
struct IdentityImpactView: View {
    @State private var useStableID = true
    @State private var items: [Item] = (1...100).map { Item(value: $0) }

    struct Item: Identifiable {
        let id: UUID
        let value: Int

        init(value: Int) {
            self.id = UUID()
            self.value = value
        }
    }

    var body: some View {
        VStack {
            Toggle("稳定 ID", isOn: $useStableID)

            List {
                ForEach(items) { item in
                    Text("项目 \(item.value)")
                }
            }
        }
    }
}
```

⚠️ **警告**：上面的代码中，每次 `items` 数组重新赋值时，每个 `Item` 的 `id` 都会变化（因为 `UUID()` 在 `init` 中生成），导致 SwiftUI 认为所有项都是新的，触发全量重建。应该使用稳定的 `id`（如数据库主键）。

---

## 视图重绘控制

控制视图的重绘范围是 SwiftUI 性能优化的核心。

### @State 的重绘范围

`@State` 变化只会重绘当前视图的 `body`：

```swift
struct StateRedrawView: View {
    @State private var counter = 0

    var body: some View {
        VStack {
            Text("计数：\(counter)")

            ExpensiveView()

            Button("增加") {
                counter += 1
            }
        }
    }
}

struct ExpensiveView: View {
    var body: some View {
        let _ = print("ExpensiveView 重绘了！")
        return Text("昂贵的视图")
    }
}
```

💡 **提示**：`ExpensiveView` 是一个独立的视图结构体，即使父视图重绘，如果 `ExpensiveView` 的输入没有变化，它不会重绘。这是 SwiftUI 的优化之一——子视图的 body 只在自己的输入变化时才重新计算。

### @ObservedObject vs @StateObject

```swift
class DataStore: ObservableObject {
    @Published var items: [String] = []
    @Published var selectedTab = 0
}

struct ObservedObjectDemoView: View {
    @StateObject private var store = DataStore()

    var body: some View {
        TabView(selection: $store.selectedTab) {
            TabOneView(store: store)
                .tabItem { Label("列表", systemImage: "list.bullet") }
                .tag(0)

            TabTwoView(store: store)
                .tabItem { Label("设置", systemImage: "gearshape") }
                .tag(1)
        }
    }
}

struct TabOneView: View {
    @ObservedObject var store: DataStore

    var body: some View {
        List(store.items, id: \.self) { item in
            Text(item)
        }
    }
}

struct TabTwoView: View {
    @ObservedObject var store: DataStore

    var body: some View {
        Text("设置页面")
    }
}
```

⚠️ **警告**：当 `store.selectedTab` 变化时，`TabTwoView` 也会重绘，即使它不使用 `selectedTab`！因为 `@ObservedObject` 会在任何 `@Published` 属性变化时通知所有订阅者。

### @StateObject vs @ObservedObject 性能差异

| 特性 | @StateObject | @ObservedObject |
|------|-------------|-----------------|
| 创建时机 | 视图首次创建时 | 由外部传入 |
| 生命周期 | 跟随视图 | 跟随传入者 |
| 重绘触发 | `@Published` 变化 | `@Published` 变化 |
| 推荐用法 | 拥有者创建 | 子视图接收 |

💡 **提示**：`@StateObject` 和 `@ObservedObject` 在重绘行为上没有区别。区别在于对象的所有权和生命周期。选择哪个取决于"谁创建这个对象"。

### @Observable（iOS 17+）的性能优势

iOS 17 引入的 `@Observable` 宏提供了更精细的重绘控制：

```swift
import Observation

@Observable
class ModernDataStore {
    var items: [String] = []
    var selectedTab = 0
    var isLoading = false
}

struct ObservableDemoView: View {
    @State private var store = ModernDataStore()

    var body: some View {
        TabView(selection: $store.selectedTab) {
            ModernTabOneView(store: store)
                .tabItem { Label("列表", systemImage: "list.bullet") }
                .tag(0)

            ModernTabTwoView(store: store)
                .tabItem { Label("设置", systemImage: "gearshape") }
                .tag(1)
        }
    }
}

struct ModernTabOneView: View {
    var store: ModernDataStore

    var body: some View {
        List(store.items, id: \.self) { item in
            Text(item)
        }
    }
}

struct ModernTabTwoView: View {
    var store: ModernDataStore

    var body: some View {
        Text("设置页面")
    }
}
```

| 特性 | @ObservedObject | @Observable |
|------|----------------|-------------|
| 重绘粒度 | 任何 `@Published` 变化都触发 | 只在视图实际使用的属性变化时触发 |
| 性能 | 粗粒度 | 细粒度 |
| iOS 版本 | iOS 14+ | iOS 17+ |
| 协议 | `ObservableObject` | 无需协议 |

💡 **提示**：`@Observable` 的最大性能优势是**自动追踪属性依赖**。`ModernTabTwoView` 只使用了 `store` 但没有读取任何属性，所以不会因为 `store` 的属性变化而重绘。即使它读取了某个属性，也只有那个属性变化时才重绘。

### 拆分视图减少重绘

将频繁变化的部分拆分为独立子视图：

```swift
// ❌ 不好：整个视图都会重绘
struct BadCounterView: View {
    @State private var count = 0
    let items = Array(1...50).map { "项目 \($0)" }

    var body: some View {
        VStack {
            Text("计数：\(count)")
            Button("增加") { count += 1 }

            List(items, id: \.self) { item in
                Text(item)
            }
        }
    }
}

// ✅ 好：只有 CounterSubView 重绘
struct GoodCounterView: View {
    let items = Array(1...50).map { "项目 \($0)" }

    var body: some View {
        VStack {
            CounterSubView()

            List(items, id: \.self) { item in
                Text(item)
            }
        }
    }
}

struct CounterSubView: View {
    @State private var count = 0

    var body: some View {
        VStack {
            Text("计数：\(count)")
            Button("增加") { count += 1 }
        }
    }
}
```

---

## Equatable 优化

### .equatable() 修饰符

当视图的输入参数相同时，使用 `.equatable()` 可以跳过重绘：

```swift
struct ColorBox: View, Equatable {
    let color: Color
    let size: CGFloat

    static func == (lhs: ColorBox, rhs: ColorBox) -> Bool {
        lhs.color == rhs.color && lhs.size == rhs.size
    }

    var body: some View {
        let _ = print("ColorBox 重绘")
        return RoundedRectangle(cornerRadius: 8)
            .fill(color)
            .frame(width: size, height: size)
    }
}

struct EquatableDemoView: View {
    @State private var counter = 0

    var body: some View {
        VStack(spacing: 20) {
            Text("计数：\(counter)")

            ColorBox(color: .blue, size: 80)
                .equatable()

            Button("增加") {
                counter += 1
            }
        }
    }
}
```

💡 **提示**：`.equatable()` 的工作原理是：当父视图重绘时，SwiftUI 会先比较新旧的 `ColorBox` 参数。如果 `==` 返回 `true`，就跳过 `body` 的重新计算。

### 何时使用 Equatable

| 场景 | 是否使用 | 原因 |
|------|---------|------|
| 视图参数很少变化 | ✅ 使用 | 避免不必要的重绘 |
| 视图 body 计算很昂贵 | ✅ 使用 | 节省计算时间 |
| 视图参数经常变化 | ❌ 不使用 | `==` 比较本身也有开销 |
| 简单视图 | ❌ 不使用 | 重绘成本很低，Equatable 反而增加开销 |

### 使用 @Observable 代替 Equatable

iOS 17+ 中，`@Observable` 已经提供了更细粒度的重绘控制，通常不再需要 `.equatable()`：

```swift
@Observable
class Theme {
    var primaryColor: Color = .blue
    var fontSize: CGFloat = 16
}

struct ThemeableView: View {
    var theme: Theme

    var body: some View {
        Text("主题文字")
            .foregroundStyle(theme.primaryColor)
            .font(.system(size: theme.fontSize))
    }
}
```

只有 `primaryColor` 或 `fontSize` 变化时，`ThemeableView` 才会重绘。其他属性变化不会触发。

---

## Lazy 容器性能

### LazyVStack / LazyHStack

`LazyVStack` 和 `LazyHStack` 只渲染可见区域的视图，适合大数据量场景：

```swift
struct LazyStackDemoView: View {
    let items = Array(1...10000).map { "项目 \($0)" }

    var body: some View {
        ScrollView {
            LazyVStack(spacing: 12) {
                ForEach(items, id: \.self) { item in
                    ItemRow(title: item)
                }
            }
            .padding()
        }
    }
}

struct ItemRow: View {
    let title: String

    var body: some View {
        HStack {
            Image(systemName: "star.fill")
                .foregroundStyle(.yellow)
            Text(title)
            Spacer()
        }
        .padding()
        .background(Color(.systemGray6))
        .clipShape(RoundedRectangle(cornerRadius: 8))
    }
}
```

### VStack vs LazyVStack

| 特性 | VStack | LazyVStack |
|------|--------|------------|
| 渲染策略 | 一次性渲染所有子视图 | 只渲染可见的子视图 |
| 内存占用 | 高（全部在内存） | 低（按需加载） |
| 适用数据量 | 少量（< 50） | 大量（> 50） |
| 滚动性能 | 数据量大时卡顿 | 流畅 |
| 间距计算 | 精确 | 动态计算 |

⚠️ **警告**：`LazyVStack` 在滚动时动态创建视图，可能导致快速滚动时出现闪烁。对于复杂行视图，建议使用 `List` 代替，它有更好的复用机制。

### List 的复用机制

`List` 底层使用 `UICollectionView` 的复用机制，性能优于 `LazyVStack`：

```swift
struct ListPerformanceView: View {
    let items = Array(1...10000).map { "项目 \($0)" }

    var body: some View {
        List(items, id: \.self) { item in
            ItemRow(title: item)
        }
    }
}
```

| 容器 | 底层实现 | 复用机制 | 推荐场景 |
|------|---------|---------|---------|
| `List` | `UICollectionView` | ✅ 有复用 | 大数据列表 |
| `LazyVStack` | 自定义实现 | ❌ 无复用 | 自定义布局 |
| `VStack` | 一次性渲染 | ❌ 无复用 | 少量内容 |

💡 **提示**：对于标准列表场景，优先使用 `List`。只有需要自定义布局（如瀑布流、不规则间距）时才用 `LazyVStack`。

### Lazy 容器的常见陷阱

#### 陷阱 1：Lazy 容器中的 @State

```swift
// ❌ 不好：LazyVStack 中的 @State 会在视图被回收时丢失
struct LazyStateProblemView: View {
    let items = Array(1...100).map { "项目 \($0)" }

    var body: some View {
        ScrollView {
            LazyVStack {
                ForEach(items, id: \.self) { item in
                    RowWithState(title: item)
                }
            }
        }
    }
}

struct RowWithState: View {
    let title: String
    @State private var isExpanded = false

    var body: some View {
        VStack(alignment: .leading) {
            Text(title)
            if isExpanded {
                Text("展开的内容")
            }
        }
        .onTapGesture { isExpanded.toggle() }
    }
}
```

⚠️ **警告**：当 `RowWithState` 滚出可见区域后被回收，再滚回来时 `@State` 会被重置。解决方案是将状态提升到父视图或使用 `@Observable` 对象管理。

#### 陷阱 2：Lazy 容器中动态高度计算

```swift
// ✅ 好：使用固定高度避免动态计算
struct LazyFixedHeightView: View {
    let items = Array(1...1000).map { "项目 \($0)" }

    var body: some View {
        ScrollView {
            LazyVStack(spacing: 8) {
                ForEach(items, id: \.self) { item in
                    Text(item)
                        .frame(height: 44)
                        .frame(maxWidth: .infinity, alignment: .leading)
                        .padding(.horizontal)
                }
            }
        }
    }
}
```

---

## @ViewBuilder 与条件渲染的性能影响

### if-else vs ternary vs switch

```swift
struct ConditionalRenderView: View {
    @State private var mode = 0

    // 方式 1：if-else
    var ifElseBody: some View {
        VStack {
            if mode == 0 {
                Text("模式 A")
            } else if mode == 1 {
                Text("模式 B")
            } else {
                Text("模式 C")
            }
        }
    }

    // 方式 2：ternary
    var ternaryBody: some View {
        VStack {
            Text(mode == 0 ? "模式 A" : mode == 1 ? "模式 B" : "模式 C")
        }
    }

    // 方式 3：switch
    var switchBody: some View {
        VStack {
            switch mode {
            case 0:
                Text("模式 A")
            case 1:
                Text("模式 B")
            default:
                Text("模式 C")
            }
        }
    }

    var body: some View {
        switchBody
    }
}
```

### 性能差异

| 方式 | 视图身份 | 切换时行为 | 性能影响 |
|------|---------|----------|---------|
| `if-else` | 不同身份 | 销毁+重建 | 高 |
| ternary | 同一身份 | 修改属性 | 低 |
| `switch` | 不同身份 | 销毁+重建 | 高 |
| `.transition()` | 不同身份+动画 | 动画过渡 | 中 |

💡 **提示**：当只是修改视图的属性（如颜色、文字内容）时，用 ternary 操作符比 `if-else` 性能更好，因为 ternary 保持了视图身份不变。当需要切换完全不同的视图结构时，`if-else` 和 `switch` 是正确的选择。

### 避免在 ViewBuilder 中做计算

```swift
// ❌ 不好：在 body 中做大量计算
struct BadCalculationView: View {
    @State private var data: [Double] = []

    var body: some View {
        VStack {
            let average = data.reduce(0, +) / Double(data.count)
            let max = data.max() ?? 0
            let min = data.min() ?? 0

            Text("平均值：\(average)")
            Text("最大值：\(max)")
            Text("最小值：\(min)")
        }
    }
}

// ✅ 好：使用计算属性或缓存
struct GoodCalculationView: View {
    @State private var data: [Double] = []

    var average: Double {
        guard !data.isEmpty else { return 0 }
        return data.reduce(0, +) / Double(data.count)
    }

    var maxValue: Double { data.max() ?? 0 }
    var minValue: Double { data.min() ?? 0 }

    var body: some View {
        VStack {
            Text("平均值：\(average)")
            Text("最大值：\(maxValue)")
            Text("最小值：\(minValue)")
        }
    }
}
```

⚠️ **警告**：`body` 中的 `let` 绑定会在每次重绘时重新计算。如果计算量很大，应该提取为计算属性或使用缓存。

---

## 图片优化

### AsyncImage 缓存

`AsyncImage` 默认不缓存图片，每次重绘都会重新下载：

```swift
// ❌ 不好：没有缓存，每次重绘都重新下载
struct BadAsyncImageView: View {
    let url: URL

    var body: some View {
        AsyncImage(url: url)
    }
}
```

#### 自定义图片缓存

```swift
import Foundation

class ImageCache {
    static let shared = ImageCache()
    private var cache = NSCache<NSString, UIImage>()

    func get(_ key: String) -> UIImage? {
        cache.object(forKey: key as NSString)
    }

    func set(_ image: UIImage, for key: String) {
        cache.setObject(image, forKey: key as NSString)
    }

    func remove(_ key: String) {
        cache.removeObject(forKey: key as NSString)
    }

    func clear() {
        cache.removeAllObjects()
    }
}

struct CachedAsyncImage: View {
    let url: URL
    @State private var image: UIImage?

    var body: some View {
        Group {
            if let image {
                Image(uiImage: image)
                    .resizable()
                    .aspectRatio(contentMode: .fill)
            } else {
                ProgressView()
            }
        }
        .task {
            let key = url.absoluteString
            if let cached = ImageCache.shared.get(key) {
                image = cached
                return
            }
            do {
                let (data, _) = try await URLSession.shared.data(from: url)
                if let uiImage = UIImage(data: data) {
                    ImageCache.shared.set(uiImage, for: key)
                    image = uiImage
                }
            } catch {
                print("图片加载失败：\(error)")
            }
        }
    }
}
```

### 图片降采样

加载大图时，降采样可以显著减少内存占用：

```swift
import ImageIO

struct DownsamplingImage: View {
    let url: URL
    let targetSize: CGSize
    @State private var image: UIImage?

    var body: some View {
        Group {
            if let image {
                Image(uiImage: image)
                    .resizable()
                    .aspectRatio(contentMode: .fill)
            } else {
                Color.gray
            }
        }
        .task {
            image = downsample(imageAt: url, to: targetSize)
        }
    }

    func downsample(imageAt url: URL, to size: CGSize) -> UIImage? {
        let options: [CFString: Any] = [
            kCGImageSourceThumbnailMaxPixelSize: max(size.width, size.height),
            kCGImageSourceCreateThumbnailFromImageAlways: true
        ]

        guard let source = CGImageSourceCreateWithURL(url as CFURL, nil),
              let cgImage = CGImageSourceCreateThumbnailAtIndex(source, 0, options as CFDictionary) else {
            return nil
        }

        return UIImage(cgImage: cgImage)
    }
}
```

| 方式 | 内存占用 | 加载速度 | 适用场景 |
|------|---------|---------|---------|
| 原图加载 | 高 | 快 | 需要完整分辨率 |
| 降采样 | 低 | 快 | 列表缩略图 |
| AsyncImage | 高 | 慢 | 简单场景 |
| 自定义缓存+降采样 | 低 | 快（缓存命中时） | 生产环境 |

💡 **提示**：在列表中显示图片时，务必使用降采样。一张 4000x3000 的照片原图占用约 48MB 内存，降采样到 200x150 只需约 120KB，内存减少 400 倍！

### 图片视图优化

```swift
struct OptimizedImageRow: View {
    let imageName: String

    var body: some View {
        HStack(spacing: 12) {
            Image(imageName)
                .resizable()
                .aspectRatio(contentMode: .fill)
                .frame(width: 60, height: 60)
                .clipShape(RoundedRectangle(cornerRadius: 8))

            VStack(alignment: .leading, spacing: 4) {
                Text(imageName)
                    .font(.headline)
                Text("图片描述")
                    .font(.caption)
                    .foregroundStyle(.secondary)
            }
        }
    }
}
```

⚠️ **警告**：`.resizable()` 是必须的！不加 `.resizable()` 的 `Image` 会以原始尺寸渲染，大图会撑爆布局并占用大量内存。

---

## 列表性能优化

### ForEach vs List

```swift
// ForEach + ScrollView：无复用
struct ForEachDemoView: View {
    let items = Array(1...1000).map { "项目 \($0)" }

    var body: some View {
        ScrollView {
            LazyVStack {
                ForEach(items, id: \.self) { item in
                    Text(item)
                        .padding()
                }
            }
        }
    }
}

// List：有复用
struct ListDemoView: View {
    let items = Array(1...1000).map { "项目 \($0)" }

    var body: some View {
        List(items, id: \.self) { item in
            Text(item)
        }
    }
}
```

| 方式 | 复用 | 性能 | 灵活性 |
|------|------|------|--------|
| `List` + `ForEach` | ✅ | 优 | 中 |
| `ScrollView` + `LazyVStack` + `ForEach` | ❌ | 中 | 高 |
| `ScrollView` + `VStack` + `ForEach` | ❌ | 差 | 高 |

### id 修饰符的影响

```swift
// ❌ 不好：使用不稳定 id，导致全量重建
struct UnstableIDView: View {
    @State private var items: [String] = ["A", "B", "C"]

    var body: some View {
        List {
            ForEach(items, id: \.self) { item in
                Text(item)
                    .id(UUID())
            }
        }
    }
}

// ✅ 好：使用稳定 id
struct StableIDView: View {
    struct Item: Identifiable {
        let id: Int
        let name: String
    }

    @State private var items: [Item] = [
        Item(id: 1, name: "A"),
        Item(id: 2, name: "B"),
        Item(id: 3, name: "C")
    ]

    var body: some View {
        List(items) { item in
            Text(item.name)
        }
    }
}
```

⚠️ **警告**：`.id(UUID())` 是最常见的性能反模式！它让每个视图在每次重绘时都有新的身份，导致 SwiftUI 销毁旧视图、创建新视图，完全丧失了 diffing 的优势。

### 列表项优化

```swift
struct OptimizedListRow: View {
    let item: ListItem

    struct ListItem: Identifiable {
        let id: UUID
        let title: String
        let subtitle: String
        let imageName: String
    }

    var body: some View {
        HStack(spacing: 12) {
            Image(systemName: item.imageName)
                .font(.title3)
                .foregroundStyle(.blue)
                .frame(width: 32)

            VStack(alignment: .leading, spacing: 2) {
                Text(item.title)
                    .font(.headline)
                    .lineLimit(1)
                Text(item.subtitle)
                    .font(.caption)
                    .foregroundStyle(.secondary)
                    .lineLimit(2)
            }

            Spacer()

            Image(systemName: "chevron.right")
                .font(.caption)
                .foregroundStyle(.tertiary)
        }
        .padding(.vertical, 4)
    }
}
```

💡 **提示**：列表项中避免使用 `.frame(maxHeight: .infinity)` 或不确定的高度，这会导致 List 无法正确计算布局，影响滚动性能。

---

## Instruments 使用

### SwiftUI 工具

Xcode 14+ 提供了专门的 SwiftUI Instruments 工具，可以分析视图的重绘情况。

#### 使用步骤

1. 在 Xcode 中，选择 Product → Profile（⌘I）
2. 选择 **SwiftUI** 模板
3. 运行应用，操作界面
4. 查看 SwiftUI 视图更新的时间线

#### SwiftUI Instruments 能看到什么

| 信息 | 说明 |
|------|------|
| View Body | 哪些视图的 body 被重新计算了 |
| View Updates | 视图更新的频率和耗时 |
| State Changes | 哪些状态变化触发了更新 |
| Property Changes | 具体哪个属性变了 |

```swift
struct ProfileDemoView: View {
    @State private var count = 0
    @State private var text = ""

    var body: some View {
        VStack {
            Text("计数：\(count)")
            TextField("输入", text: $text)
            Button("增加") { count += 1 }
        }
    }
}
```

在 Instruments 中，点击"增加"按钮时，可以看到只有 `Text("计数：\(count)")` 被重新计算，`TextField` 不会。

### Time Profiler

Time Profiler 用于找出 CPU 热点，定位耗时操作：

#### 使用步骤

1. Product → Profile（⌘I）
2. 选择 **Time Profiler**
3. 操作应用到卡顿场景
4. 查看调用栈，找到耗时最多的函数

#### 常见热点

| 热点 | 原因 | 解决方案 |
|------|------|---------|
| `body` 计算耗时 | 视图过于复杂 | 拆分子视图 |
| `NSAttributedString` | 富文本计算 | 缓存计算结果 |
| 图片解码 | 大图加载 | 降采样 |
| JSON 解析 | 数据量大 | 后台线程解析 |
| 布局计算 | 嵌套层级深 | 简化布局 |

### Hangs 工具

Hangs 工具检测主线程卡顿：

1. Product → Profile（⌘I）
2. 选择 **Hangs**
3. 操作应用
4. 查看 Hang 报告

💡 **提示**：Hang 定义为主线程阻塞超过 250ms，用户会感知到明显卡顿。所有 UI 更新和事件处理必须在主线程完成，耗时操作应该放到后台线程。

### 自定义性能标记

使用 `os_signpost` 标记关键代码段：

```swift
import os.signpost

let log = OSLog(subsystem: "com.example.app", category: "Performance")

struct ProfiledView: View {
    @State private var items: [String] = []

    var body: some View {
        List(items, id: \.self) { item in
            Text(item)
        }
        .task {
            let signpostID = OSSignpostID(log: log)
            os_signpost(.begin, log: log, name: "LoadData", signpostID: signpostID)

            await loadData()

            os_signpost(.end, log: log, name: "LoadData", signpostID: signpostID)
        }
    }

    func loadData() async {
        os_signpost(.event, log: log, name: "DataCount", "%d", items.count)
        items = Array(1...100).map { "项目 \($0)" }
    }
}
```

---

## 常见性能反模式及修复

### 反模式 1：巨型 body

```swift
// ❌ 不好：一个视图的 body 有几百行
struct GiantView: View {
    @State private var text = ""

    var body: some View {
        VStack {
            // 几百行代码...
            Text("标题")
            // 更多代码...
            TextField("输入", text: $text)
            // 还有更多代码...
        }
    }
}

// ✅ 好：拆分为多个子视图
struct RefactoredView: View {
    @State private var text = ""

    var body: some View {
        VStack {
            HeaderSection()
            InputSection(text: $text)
            ContentSection()
        }
    }
}

struct HeaderSection: View {
    var body: some View {
        Text("标题")
    }
}

struct InputSection: View {
    @Binding var text: String

    var body: some View {
        TextField("输入", text: $text)
    }
}

struct ContentSection: View {
    var body: some View {
        Text("内容")
    }
}
```

### 反模式 2：在 body 中创建对象

```swift
// ❌ 不好：每次重绘都创建新对象
struct ObjectCreationView: View {
    @State private var count = 0

    var body: some View {
        VStack {
            Text("计数：\(count)")

            let formatter = DateFormatter()
            formatter.dateStyle = .long
            Text(formatter.string(from: Date()))

            Button("增加") { count += 1 }
        }
    }
}

// ✅ 好：使用静态或缓存的格式化器
struct FixedObjectCreationView: View {
    @State private var count = 0

    private static let formatter: DateFormatter = {
        let f = DateFormatter()
        f.dateStyle = .long
        return f
    }()

    var body: some View {
        VStack {
            Text("计数：\(count)")
            Text(Self.formatter.string(from: Date()))
            Button("增加") { count += 1 }
        }
    }
}
```

### 反模式 3：不必要的 @Published

```swift
// ❌ 不好：所有属性都是 @Published
class BadViewModel: ObservableObject {
    @Published var items: [String] = []
    @Published var filteredItems: [String] = []
    @Published var searchText = ""
    @Published var isLoading = false
    @Published var error: String?
}

// ✅ 好：只发布需要驱动 UI 的属性，派生值用计算属性
class GoodViewModel: ObservableObject {
    @Published var items: [String] = []
    @Published var searchText = ""
    @Published var isLoading = false
    @Published var error: String?

    var filteredItems: [String] {
        if searchText.isEmpty {
            return items
        }
        return items.filter { $0.localizedCaseInsensitiveContains(searchText) }
    }
}
```

💡 **提示**：`filteredItems` 作为计算属性，不会额外触发 `objectWillChange`。当 `items` 或 `searchText` 变化时，视图会重绘并自动获取最新的 `filteredItems`。

### 反模式 4：过度使用 @State

```swift
// ❌ 不好：派生状态用 @State
struct BadDerivedStateView: View {
    @State private var items: [String] = []
    @State private var itemCount: Int = 0

    var body: some View {
        VStack {
            Text("共 \(itemCount) 项")
            List(items, id: \.self) { item in
                Text(item)
            }
        }
        .onAppear {
            items = loadItems()
            itemCount = items.count
        }
    }
}

// ✅ 好：派生状态用计算属性
struct GoodDerivedStateView: View {
    @State private var items: [String] = []

    var itemCount: Int { items.count }

    var body: some View {
        VStack {
            Text("共 \(itemCount) 项")
            List(items, id: \.self) { item in
                Text(item)
            }
        }
        .onAppear {
            items = loadItems()
        }
    }

    func loadItems() -> [String] {
        Array(1...20).map { "项目 \($0)" }
    }
}
```

### 反模式 5：在 onChange 中做重计算

```swift
// ❌ 不好：onChange 中做大量计算
struct BadOnChangeView: View {
    @State private var searchText = ""
    @State private var allItems: [String] = []
    @State private var filteredItems: [String] = []

    var body: some View {
        List(filteredItems, id: \.self) { item in
            Text(item)
        }
        .searchable(text: $searchText)
        .onChange(of: searchText) { _, newValue in
            filteredItems = allItems.filter { $0.localizedCaseInsensitiveContains(newValue) }
        }
    }
}

// ✅ 好：使用计算属性自动过滤
struct GoodOnChangeView: View {
    @State private var searchText = ""
    let allItems: [String]

    var filteredItems: [String] {
        if searchText.isEmpty {
            return allItems
        }
        return allItems.filter { $0.localizedCaseInsensitiveContains(searchText) }
    }

    var body: some View {
        List(filteredItems, id: \.self) { item in
            Text(item)
        }
        .searchable(text: $searchText)
    }
}
```

### 反模式汇总

| 反模式 | 问题 | 修复方案 |
|--------|------|---------|
| 巨型 body | 重绘范围大 | 拆分子视图 |
| body 中创建对象 | 每次重绘都创建 | 静态属性/缓存 |
| 不必要的 @Published | 触发多余重绘 | 计算属性替代 |
| 派生状态用 @State | 状态同步问题 | 计算属性 |
| onChange 重计算 | 可能触发循环 | 计算属性 |
| 不稳定 id | 全量重建 | 稳定唯一 id |
| 大图不降采样 | 内存爆炸 | ImageIO 降采样 |

---

## 大型数据集的虚拟化

当数据量非常大（10,000+）时，即使使用 `LazyVStack` 也可能出现性能问题。需要更高级的虚拟化策略。

### 分页加载

```swift
struct PaginationListView: View {
    @State private var items: [String] = []
    @State private var isLoading = false
    @State private var currentPage = 1
    @State private var hasMore = true

    let pageSize = 50

    var body: some View {
        List {
            ForEach(items, id: \.self) { item in
                Text(item)
                    .onAppear {
                        if item == items.last && hasMore {
                            loadMore()
                        }
                    }
            }

            if isLoading {
                HStack {
                    Spacer()
                    ProgressView()
                    Spacer()
                }
            }
        }
        .task {
            loadMore()
        }
    }

    func loadMore() {
        guard !isLoading && hasMore else { return }
        isLoading = true

        DispatchQueue.main.asyncAfter(deadline: .now() + 0.5) {
            let newItems = ((currentPage - 1) * pageSize..<(currentPage * pageSize)).map { "项目 \($0)" }
            items.append(contentsOf: newItems)
            currentPage += 1
            hasMore = items.count < 10000
            isLoading = false
        }
    }
}
```

### 搜索防抖

```swift
struct DebouncedSearchView: View {
    @State private var searchText = ""
    @State private var debouncedText = ""
    @State private var searchTask: Task<Void, Never>?

    let allItems = Array(1...10000).map { "项目 \($0)" }

    var results: [String] {
        if debouncedText.isEmpty {
            return Array(allItems.prefix(50))
        }
        return allItems.filter { $0.localizedCaseInsensitiveContains(debouncedText) }
    }

    var body: some View {
        List(results, id: \.self) { item in
            Text(item)
        }
        .searchable(text: $searchText)
        .onChange(of: searchText) { _, newValue in
            searchTask?.cancel()
            searchTask = Task {
                try? await Task.sleep(nanoseconds: 300_000_000)
                guard !Task.isCancelled else { return }
                debouncedText = newValue
            }
        }
    }
}
```

💡 **提示**：搜索防抖的延迟一般设为 300ms，这是用户停止输入后等待的合理时间。太短（100ms）可能还是会在快速输入时频繁搜索，太长（1s）会让用户觉得响应慢。

### 虚拟化策略对比

| 策略 | 适用场景 | 实现复杂度 | 效果 |
|------|---------|-----------|------|
| 分页加载 | 列表无限滚动 | 低 | 好 |
| 搜索防抖 | 搜索过滤 | 低 | 好 |
| 数据分片 | 超大数据集 | 中 | 优 |
| 后台线程处理 | 复杂计算 | 中 | 优 |
| 预加载 | 需要流畅体验 | 高 | 优 |

### 后台线程处理

```swift
struct BackgroundProcessingView: View {
    @State private var items: [String] = []
    @State private var filteredItems: [String] = []
    @State private var searchText = ""

    var body: some View {
        List(filteredItems, id: \.self) { item in
            Text(item)
        }
        .searchable(text: $searchText)
        .onChange(of: searchText) { _, newValue in
            Task.detached(priority: .userInitiated) {
                let result = performFilter(items: items, query: newValue)
                await MainActor.run {
                    filteredItems = result
                }
            }
        }
        .task {
            items = Array(1...50000).map { "项目 \($0)" }
            filteredItems = Array(items.prefix(50))
        }
    }

    func performFilter(items: [String], query: String) -> [String] {
        if query.isEmpty {
            return Array(items.prefix(50))
        }
        return items.filter { $0.localizedCaseInsensitiveContains(query) }
    }
}
```

⚠️ **警告**：SwiftUI 视图的 `body` 必须在主线程执行。所有耗时计算都应该在后台线程完成，然后将结果传回主线程更新 UI。

---

## AI 辅助性能优化

AI 工具可以在性能优化中发挥重要作用，帮助识别问题和生成优化方案。

### 使用 Claude Code 分析性能问题

向 AI 提供代码和性能描述，获取优化建议：

```
提示词示例：

我的 SwiftUI 列表在滚动时很卡，以下是相关代码：

[粘贴代码]

列表有 1000+ 条数据，每行包含图片和文字。请帮我分析性能瓶颈并给出优化方案。
```

AI 可以帮你：
- 识别性能反模式
- 建议视图拆分方案
- 推荐合适的缓存策略
- 生成优化后的代码

### 常见 AI 优化提示词

| 场景 | 提示词 |
|------|--------|
| 视图重绘过多 | "分析这段 SwiftUI 代码的重绘范围，找出不必要的重绘并优化" |
| 列表卡顿 | "优化这个列表的性能，数据量 10000+，每行有图片和复杂布局" |
| 内存过高 | "分析这段图片加载代码的内存问题，图片来自网络，尺寸较大" |
| 启动慢 | "优化这个视图的初始化性能，启动时加载大量数据" |
| 动画卡顿 | "这个动画在低端设备上卡顿，请优化动画性能" |

### AI 辅助代码审查

```swift
// 提交给 AI 审查的代码
struct ProductListView: View {
    @ObservedObject var store: ProductStore
    @State private var searchText = ""

    var body: some View {
        VStack {
            TextField("搜索", text: $searchText)
                .textFieldStyle(.roundedBorder)
                .padding()

            ScrollView {
                VStack(spacing: 12) {
                    ForEach(store.products) { product in
                        ProductRow(product: product)
                    }
                }
                .padding()
            }
        }
    }
}
```

AI 可能指出的问题：

1. ❌ `VStack` 而非 `LazyVStack`——所有行一次性渲染
2. ❌ `ScrollView + VStack` 而非 `List`——没有复用机制
3. ❌ `@ObservedObject` 可能导致不必要的重绘
4. ❌ 没有搜索防抖
5. ❌ 没有分页加载

### AI 生成的优化方案

```swift
struct OptimizedProductListView: View {
    @State private var store = ProductStore()
    @State private var searchText = ""
    @State private var debouncedSearch = ""
    @State private var searchTask: Task<Void, Never>?

    var filteredProducts: [Product] {
        if debouncedSearch.isEmpty {
            return store.products
        }
        return store.products.filter {
            $0.name.localizedCaseInsensitiveContains(debouncedSearch)
        }
    }

    var body: some View {
        List(filteredProducts) { product in
            ProductRow(product: product)
                .equatable()
        }
        .searchable(text: $searchText)
        .onChange(of: searchText) { _, newValue in
            searchTask?.cancel()
            searchTask = Task {
                try? await Task.sleep(nanoseconds: 300_000_000)
                guard !Task.isCancelled else { return }
                debouncedSearch = newValue
            }
        }
    }
}
```

💡 **提示**：AI 工具擅长识别常见的性能反模式，但最终的优化决策还是需要结合实际场景。建议先用 AI 获取优化方向，再用 Instruments 验证效果。

---

## 性能优化检查清单

在提交代码前，对照这个清单检查：

### 视图结构

- [ ] 单个视图的 `body` 不超过 50 行
- [ ] 频繁变化的部分已拆分为独立子视图
- [ ] 没有在 `body` 中创建对象（DateFormatter 等）
- [ ] 条件渲染使用正确的方式（ternary vs if-else）

### 状态管理

- [ ] 使用 `@Observable`（iOS 17+）替代 `@ObservedObject`
- [ ] 派生状态使用计算属性，不使用额外的 `@State`
- [ ] `@Published` 只用于需要驱动 UI 的属性
- [ ] 没有不必要的 `@State` 声明

### 列表与数据

- [ ] 大数据列表使用 `List` 而非 `ScrollView + VStack`
- [ ] `ForEach` 使用稳定的 `id`
- [ ] 没有使用 `.id(UUID())`
- [ ] 搜索有防抖处理
- [ ] 大数据集有分页加载

### 图片

- [ ] 列表中的图片使用降采样
- [ ] 网络图片有缓存
- [ ] `Image` 使用了 `.resizable()`
- [ ] 图片有合理的 `frame` 约束

### 内存

- [ ] 没有循环引用
- [ ] 大对象及时释放
- [ ] `Task` 在视图消失时取消
- [ ] 定时器在视图消失时失效

---

## 小结

本章我们深入学习了 SwiftUI 性能优化的完整知识：

| 知识点 | 核心内容 |
|--------|---------|
| **渲染原理** | Diffing 机制、视图身份（结构化/显式/数据） |
| **重绘控制** | @State/@ObservedObject/@StateObject/@Observable 的性能差异 |
| **Equatable** | `.equatable()` 修饰符、适用场景 |
| **Lazy 容器** | LazyVStack/LazyHStack vs List、常见陷阱 |
| **条件渲染** | if-else vs ternary、ViewBuilder 性能影响 |
| **图片优化** | AsyncImage 缓存、降采样、内存优化 |
| **列表优化** | ForEach vs List、id 修饰符、行视图优化 |
| **Instruments** | SwiftUI 工具、Time Profiler、Hangs |
| **性能反模式** | 巨型 body、对象创建、不稳定 id 等 |
| **大型数据集** | 分页加载、搜索防抖、后台线程 |
| **AI 辅助** | 提示词技巧、代码审查、优化方案生成 |

🔑 **核心记忆点**：
1. 理解 SwiftUI 的 diffing 机制是优化的基础
2. 拆分视图是最有效的优化手段——让重绘范围最小化
3. iOS 17+ 优先使用 `@Observable`，它提供最精细的重绘控制
4. 大数据列表用 `List`，自定义布局用 `LazyVStack`
5. 列表中的图片必须降采样，内存差异可达数百倍
6. 永远不要使用 `.id(UUID())`，这是最严重的性能反模式
7. 用 Instruments 验证优化效果，不要凭感觉优化
8. AI 工具可以快速识别反模式，但最终需要实测验证

← [-Combine与响应式编程](./75-Combine与响应式编程.md) | [-附录](../附录/) →
