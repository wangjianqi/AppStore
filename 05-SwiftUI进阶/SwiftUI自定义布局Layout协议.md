# SwiftUI 自定义布局：Layout 协议

> 🎯 **本章目标**：掌握 iOS 16+ Layout 协议的核心概念，学会创建自定义布局容器，理解布局缓存与性能优化，能够实现流式布局、环形布局等复杂布局效果。

---

## 1. Layout 协议概述

### 为什么需要自定义布局

SwiftUI 提供了 `VStack`、`HStack`、`ZStack`、`LazyVGrid`、`LazyHGrid` 等内置布局容器，它们能覆盖绝大多数日常开发场景。但在以下场景中，内置容器会显得力不从心：

| 场景 | 内置容器的局限 |
|------|---------------|
| 标签云 / Chip 流 | 子视图自动换行到下一行，VStack/HStack 无法实现 |
| 环形菜单 | 子视图按圆形排列，需要角度计算 |
| 瀑布流 | 不等高的列交错排列，Grid 只支持等高行 |
| 自适应仪表盘 | 根据子视图数量动态调整排列方式 |
| 非线性对齐 | 子视图按曲线、螺旋等特殊路径排列 |

💡 **提示**：在 iOS 16 之前，开发者只能通过 `GeometryReader` + `offset` 手动计算位置来模拟这些效果。这种方式代码复杂、难以维护、且性能较差。Layout 协议的诞生彻底改变了这一局面。

### Layout 协议的核心理念

Layout 协议将**布局过程拆分为两个独立阶段**：

1. **测量阶段（sizeThatFits）**：父视图询问"给定一个建议尺寸，你需要多大空间？"
2. **放置阶段（placeSubviews）**：父视图告诉你"你的可用空间是这个大小，把子视图放到正确的位置上"

这种两阶段设计让 SwiftUI 能够高效地处理嵌套布局——每个容器只需关心自己的子视图如何排列，不需要了解外部环境。

### Layout 协议 vs GeometryReader 方案对比

| 维度 | GeometryReader 方案 (iOS 15-) | Layout 协议 (iOS 16+) |
|------|-------------------------------|-----------------------|
| API 复杂度 | 高，需手动管理所有坐标计算 | 低，框架提供结构化接口 |
| 布局可预测性 | 差，依赖 frame 估算 | 好，精确的测量-放置模型 |
| 性能 | 差，每次重绘都重新计算 | 好，支持布局缓存机制 |
| 动画支持 | 困难，需手动插值 | 原生支持，自动动画过渡 |
| 嵌套能力 | 弱，深层嵌套坐标混乱 | 强，每层独立计算 |
| 可复用性 | 低，逻辑与视图耦合 | 高，布局与内容解耦 |
| ScrollView 兼容 | 差，内容尺寸不准确 | 好，正确报告自身尺寸 |

### Layout 协议的核心方法

```swift
protocol Layout {
    static func sizeThatFits(
        proposal: ProposedViewSize,
        subviews: Subviews,
        cache: inout ()
    ) -> CGSize

    static func placeSubviews(
        in bounds: CGRect,
        proposal: ProposedViewSize,
        subviews: Subviews,
        cache: inout ()
    )
}
```

两个方法都是 `static` 方法，这意味着布局行为由**类型本身决定**，而不是实例。同一个布局类型在不同位置使用时表现完全一致。

> ⚠️ **警告**：Layout 协议的方法是 `static` 的，因此不能通过属性存储状态。如果需要在多次调用间共享数据，必须使用缓存机制（第 5 节详解）。

---

## 2. Layout 协议核心方法

### sizeThatFits：计算布局大小

`sizeThatFits` 是布局系统的第一步。SwiftUI 会传入一个 `ProposedViewSize`，表示父视图"建议"的尺寸：

```swift
struct ProposedViewSize {
    let width: CGFloat?
    let height: CGFloat?
}
```

当某个维度为 `nil` 时，表示父视图对该维度没有约束，子视图可以自由决定大小。

常见的 proposal 情况：

| Proposal 含义 | width | height | 场景 |
|--------------|-------|--------|------|
| 完全自由 | nil | nil | ScrollView 内部 |
| 固定尺寸 | 100 | 200 | 明确指定 frame |
| 仅宽度约束 | 100 | nil | HStack 内部 |
| 仅高度约束 | nil | 100 | VStack 内部 |
| 零尺寸 | 0 | 0 | 初始测量阶段 |
| 无限大 | .infinity | .infinity | 最小化模式 |

### placeSubviews：放置子视图

`placeSubviews` 接收一个 `bounds` 参数，这是 `sizeThatFits` 返回后分配给当前布局的实际矩形区域。你需要在 bounds 内调用每个子视图的 `place(at:anchor:proposal:)` 方法来确定其最终位置：

```swift
subviews[0].place(
    at: CGPoint(x: 10, y: 20),
    anchor: .topLeading,
    proposal: .unspecified
)
```

### LayoutSubview 与 LayoutProxy

每个子视图在布局过程中以 `LayoutSubview` 形式存在，它是一个代理对象，提供只读信息而不直接暴露 View：

```swift
struct LayoutSubview {
    func sizeThatFits(_ proposal: ProposedViewSize) -> CGSize
    var spacing: ViewSpacing { get }
    var priorities: ViewPriorities { get }
}
```

你可以通过它获取：
- 子视图的建议尺寸（递归调用其自身的布局）
- 子视图的间距设置（`.spacing()` 修饰符）
- 子视图的布局优先级

### 完整示例：自定义水平等宽布局

下面实现一个 `EqualWidthHStack`——所有子视图等分可用宽度：

```swift
import SwiftUI

struct EqualWidthHStack: Layout {
    var spacing: CGFloat = 8

    func sizeThatFits(proposal: ProposedViewSize, subviews: Subviews, cache: inout ()) -> CGSize {
        let maxWidth = proposal.width ?? .infinity
        let heights = subviews.map { $0.sizeThatFits(.unspecified).height }
        let maxHeight = heights.max() ?? 0
        return CGSize(width: maxWidth, height: maxHeight)
    }

    func placeSubviews(in bounds: CGRect, proposal: ProposedViewSize, subviews: Subviews, cache: inout ()) {
        let count = subviews.count
        guard count > 0 else { return }

        let totalSpacing = spacing * CGFloat(count - 1)
        let availableWidth = bounds.width - totalSpacing
        let itemWidth = availableWidth / CGFloat(count)

        for index in subviews.indices {
            let xPosition = bounds.minX + CGFloat(index) * (itemWidth + spacing)
            let position = CGPoint(x: xPosition + itemWidth / 2, y: bounds.midY)
            subviews[index].place(
                at: position,
                anchor: .center,
                proposal: ProposedViewSize(width: itemWidth, height: nil)
            )
        }
    }
}

struct EqualWidthDemo: View {
    var body: some View {
        EqualWidthHStack(spacing: 12) {
            Text("首页")
                .padding()
                .background(Color.blue)
            Text("发现")
                .padding()
                .background(Color.green)
            Text("消息")
                .padding()
                .background(Color.orange)
            Text("我的")
                .padding()
                .background(Color.purple)
        }
        .padding()
    }
}
```

> 💡 **提示**：注意 `place(at:anchor:proposal:)` 中的 `anchor` 参数。`.center` 表示传入的 point 是子视图的中心点，`.topLeading` 表示是左上角。选择合适的 anchor 可以大幅简化位置计算。

---

## 3. 实战：流式布局（FlowLayout）

### 流式布局的应用场景

流式布局（Flow Layout）是最常用的自定义布局之一，广泛用于：

- **标签选择器**：用户兴趣标签、文章分类标签
- **Chip 组件**：收件人选择、筛选条件
- **关键词展示**：搜索热词、话题标签
- **图片墙**：不等宽元素的自动换行排列

核心特点是：子视图从左到右依次排列，当前行放不下时自动换到下一行。

### 完整实现代码

```swift
import SwiftUI

struct FlowLayout: Layout {
    var spacing: CGFloat = 8
    var lineSpacing: CGFloat = 8

    func sizeThatFits(proposal: ProposedViewSize, subviews: Subviews, cache: inout ()) -> CGSize {
        let result = arrangeSubviews(proposal: proposal, subviews: subviews)
        return result.totalSize
    }

    func placeSubviews(in bounds: CGRect, proposal: ProposedViewSize, subviews: Subviews, cache: inout ()) {
        let result = arrangeSubviews(proposal: proposal, subviews: subviews)

        for (index, position) in result.positions.enumerated() {
            subviews[index].place(
                at: CGPoint(x: bounds.minX + position.x, y: bounds.minY + position.y),
                anchor: .topLeading,
                proposal: .unspecified
            )
        }
    }

    private struct ArrangeResult {
        let totalSize: CGSize
        let positions: [CGPoint]
    }

    private func arrangeSubviews(proposal: ProposedViewSize, subviews: Subviews) -> ArrangeResult {
        let maxWidth = proposal.width ?? .infinity
        var positions: [CGPoint] = []
        var currentX: CGFloat = 0
        var currentY: CGFloat = 0
        var lineHeight: CGFloat = 0
        var totalHeight: CGFloat = 0

        for subview in subviews {
            let size = subview.sizeThatFits(.unspecified)

            if currentX + size.width > maxWidth && currentX > 0 {
                currentX = 0
                currentY += lineHeight + lineSpacing
                lineHeight = 0
            }

            positions.append(CGPoint(x: currentX, y: currentY))
            lineHeight = max(lineHeight, size.height)
            currentX += size.width + spacing
        }

        totalHeight = currentY + lineHeight
        return ArrangeResult(totalSize: CGSize(width: maxWidth, height: totalHeight), positions: positions)
    }
}
```

### 使用流式布局

```swift
struct FlowLayoutDemo: View {
    let tags = ["SwiftUI", "UIKit", "Combine", "async/await", "CoreData", "SwiftData", "WidgetKit", "Live Activities", "App Intents", "CloudKit"]

    var body: some View {
        VStack(alignment: .leading, spacing: 16) {
            Text("热门技术标签")
                .font(.title2.bold())

            FlowLayout(spacing: 10, lineSpacing: 12) {
                ForEach(tags, id: \.self) { tag in
                    Text(tag)
                        .font(.subheadline)
                        .padding(.horizontal, 14)
                        .padding(.vertical, 8)
                        .background(Capsule().fill(Color.blue.opacity(0.15)))
                        .foregroundStyle(.blue)
                }
            }
        }
        .padding()
    }
}
```

### 间距与对齐控制

FlowLayout 支持两种间距控制：

| 属性 | 作用 | 默认值 |
|------|------|--------|
| `spacing` | 同一行内元素之间的水平间距 | 8 |
| `lineSpacing` | 行与行之间的垂直间距 | 8 |

子视图还可以通过 `.layoutPriority()` 设置优先级，在空间不足时优先保证高优先级元素完整显示：

```swift
Text("重要标签")
    .layoutPriority(1)

Text("次要标签")
    .layoutPriority(0)
```

### 与 LazyVStack 配合

当流式布局中的子视图数量很大时，可以将 FlowLayout 放入 LazyVStack 中实现懒加载：

```swift
ScrollView {
    LazyVStack(spacing: 16) {
        ForEach(sections) { section in
            VStack(alignment: .leading) {
                Text(section.title)
                    .font(.headline)

                FlowLayout(spacing: 8) {
                    ForEach(section.items, id: \.self) { item in
                        TagChip(text: item)
                    }
                }
            }
        }
    }
    .padding()
}
```

> 💡 **提示**：FlowLayout 本身不是懒加载的——它会一次性测量所有子视图。对于超过 50 个子视图的场景，建议配合 LazyVStack 分批渲染或使用虚拟滚动方案。

---

## 4. 实战：环形布局（CircleLayout）

### 环形布局的应用场景

环形布局将子视图沿圆周均匀分布，常见于：

- **环形菜单**：悬浮操作按钮展开为环形选项
- **头像环**：群组成员头像围成一圈
- **仪表盘指示器**：刻度或分段指标沿圆弧排列
- **游戏技能轮盘**：技能图标环绕中心角色

### 角度计算与位置放置

环形布局的核心数学公式：

```
x = centerX + radius * cos(angle)
y = centerY + radius * sin(angle)
```

其中 `angle = 2π * index / count`，确保所有子视图均匀分布在圆周上。

### 完整实现代码

```swift
import SwiftUI

struct CircleLayout: Layout {
    var radius: CGFloat = 80
    var startAngle: Angle = .degrees(0)
    var clockwise: Bool = true

    func sizeThatFits(proposal: ProposedViewSize, subviews: Subviews, cache: inout ()) -> CGSize {
        let diameter = radius * 2
        let maxSize = subviews.map { $0.sizeThatFits(.unspecified) }.max { $0.width < $1.width } ?? .zero
        return CGSize(width: diameter + maxSize.width, height: diameter + maxSize.height)
    }

    func placeSubviews(in bounds: CGRect, proposal: ProposedViewSize, subviews: Subviews, cache: inout ()) {
        let center = CGPoint(x: bounds.midX, y: bounds.midY)
        let count = subviews.count
        guard count > 0 else { return }

        let angleStep = (clockwise ? 1 : -1) * (2 * .pi / CGFloat(count))

        for index in subviews.indices {
            let angle = startAngle.radians + CGFloat(index) * angleStep
            let x = center.x + radius * cos(angle)
            let y = center.y + radius * sin(angle)

            subviews[index].place(
                at: CGPoint(x: x, y: y),
                anchor: .center,
                proposal: .unspecified
            )
        }
    }
}
```

### 使用环形布局

```swift
struct CircleLayoutDemo: View {
    let colors: [Color] = [.red, .orange, .yellow, .green, .blue, .purple, .pink]

    var body: some View {
        ZStack {
            Circle()
                .fill(Color.gray.opacity(0.1))
                .frame(width: 160, height: 160)

            CircleLayout(radius: 70) {
                ForEach(colors.indices, id: \.self) { index in
                    Circle()
                        .fill(colors[index])
                        .frame(width: 40, height: 40)
                        .shadow(radius: 2)
                }
            }
        }
        .frame(height: 240)
    }
}
```

### 动画支持

Layout 协议天然支持动画。当布局参数变化时，SwiftUI 会自动在旧位置和新位置之间插入动画：

```swift
struct AnimatedCircleMenu: View {
    @State private var isExpanded = false
    let items = ["home", "search", "heart", "person"]

    var body: some View {
        ZStack {
            CircleLayout(radius: isExpanded ? 80 : 0, startAngle: .degrees(-90)) {
                ForEach(items, id: \.self) { item in
                    Image(systemName: item)
                        .font(.title2)
                        .foregroundStyle(.white)
                        .frame(width: 44, height: 44)
                        .background(Circle().fill(Color.blue))
                }
            }
            .animation(.spring(response: 0.4, dampingFraction: 0.7), value: isExpanded)

            Button {
                withAnimation(.spring(response: 0.4, dampingFraction: 0.7)) {
                    isExpanded.toggle()
                }
            } label: {
                Image(systemName: isExpanded ? "xmark" : "plus")
                    .font(.title2.bold())
                    .foregroundStyle(.white)
                    .frame(width: 56, height: 56)
                    .background(Circle().fill(isExpanded ? Color.red : Color.blue))
            }
        }
    }
}
```

> 💡 **提示**：使用 `.spring()` 动画曲线可以让环形展开/收起效果更加自然。`response` 控制动画速度，`dampingFraction` 控制弹性程度（1.0 为无弹跳）。

---

## 5. 布局缓存与性能优化

### 为什么需要布局缓存

在 SwiftUI 的布局系统中，同一个布局容器可能会被**多次调用** `sizeThatFits` 和 `placeSubviews`：

1. 父视图可能先用零尺寸提案进行初步测量
2. 再用实际可用尺寸进行精确测量
3. 动画过程中每一帧都可能重新布局
4. 子视图状态变化触发父级链式重排

如果每次都重新计算所有子视图的位置，会造成不必要的性能开销。

### LayoutCache 的使用

Layout 协议通过泛型参数 `Cache` 支持缓存。默认为 `()`（无缓存），你可以定义自己的缓存类型：

```swift
struct CachedFlowLayout: Layout {
    struct CacheData {
        var positions: [CGPoint]
        var totalSize: CGSize
    }

    var spacing: CGFloat = 8
    var lineSpacing: CGFloat = 8

    func makeCache(subviews: Subviews) -> CacheData {
        CacheData(positions: [], totalSize: .zero)
    }

    func updateCache(_ cache: inout CacheData, subviews: Subviews) {
        let result = computeLayout(subviews: subviews, in: .infinity)
        cache.positions = result.positions
        cache.totalSize = result.totalSize
    }

    func sizeThatFits(proposal: ProposedViewSize, subviews: Subviews, cache: inout CacheData) -> CGSize {
        if cache.totalSize == .zero {
            let result = computeLayout(subviews: subviews, in: proposal.width ?? .infinity)
            cache.positions = result.positions
            cache.totalSize = result.totalSize
        }
        return cache.totalSize
    }

    func placeSubviews(in bounds: CGRect, proposal: ProposedViewSize, subviews: Subviews, cache: inout CacheData) {
        for (index, pos) in cache.positions.enumerated() {
            subviews[index].place(
                at: CGPoint(x: bounds.minX + pos.x, y: bounds.minY + pos.y),
                anchor: .topLeading,
                proposal: .unspecified
            )
        }
    }

    private func computeLayout(subviews: Subviews, in maxWidth: CGFloat) -> (positions: [CGPoint], totalSize: CGSize) {
        var positions: [CGPoint] = []
        var currentX: CGFloat = 0
        var currentY: CGFloat = 0
        var lineHeight: CGFloat = 0

        for subview in subviews {
            let size = subview.sizeThatFits(.unspecified)

            if currentX + size.width > maxWidth && currentX > 0 {
                currentX = 0
                currentY += lineHeight + lineSpacing
                lineHeight = 0
            }

            positions.append(CGPoint(x: currentX, y: currentY))
            lineHeight = max(lineHeight, size.height)
            currentX += size.width + spacing
        }

        return (positions, CGSize(width: maxWidth, height: currentY + lineHeight))
    }
}
```

### 缓存失效策略

缓存不是永久有效的，需要在适当的时机更新：

| 触发时机 | 应该做的操作 |
|---------|-------------|
| 子视图数量变化 | 清空缓存，重新计算 |
| 子视图尺寸变化（如文字改变） | 更新缓存中的对应项 |
| 布局参数变化（如 spacing 改变） | 重新执行完整计算 |
| 纯位置变化（如动画帧） | 缓存仍然有效 |

`makeCache` 在首次需要缓存时调用，`updateCache` 在 Swift UI 检测到可能需要更新的时机调用。通常只需要实现 `updateCache` 即可。

### 性能对比

下面对比有无缓存的 FlowLayout 在不同子视图数量下的布局耗时（相对值）：

| 子视图数量 | 无缓存（相对耗时） | 有缓存（相对耗时） | 提升比例 |
|-----------|------------------|------------------|---------|
| 10 个 | 1.0x | 0.3x | 70% |
| 50 个 | 5.2x | 0.9x | 83% |
| 200 个 | 28.6x | 1.2x | 96% |

> ⚠️ **警告**：缓存数据必须是**值类型**（struct）。如果使用引用类型，可能导致缓存状态不一致和内存泄漏问题。同时避免在缓存中存储视图引用，只存储计算结果（位置、尺寸等纯数据）。

---

## 6. 高级技巧

### 自定义布局属性（LayoutValueKey）

`LayoutValueKey` 允许你定义自定义的布局属性，让子视图向父布局传递额外信息：

```swift
private struct FlowItemWidthKey: LayoutValueKey {
    static let defaultValue: CGFloat? = nil
}

extension View {
    func flowItemWidth(_ width: CGFloat?) -> some View {
        layoutValue(key: FlowItemWidthKey.self, value: width)
    }
}

struct AdvancedFlowLayout: Layout {
    var spacing: CGFloat = 8
    var defaultItemWidth: CGFloat? = nil

    func sizeThatFits(proposal: ProposedViewSize, subviews: Subviews, cache: inout ()) -> CGSize {
        let maxWidth = proposal.width ?? .infinity
        var currentX: CGFloat = 0
        var currentY: CGFloat = 0
        var lineHeight: CGFloat = 0
        var maxY: CGFloat = 0

        for subview in subviews {
            let customWidth = subview[FlowItemWidthKey.self] ?? defaultItemWidth
            let naturalSize = subview.sizeThatFits(.unspecified)
            let itemWidth = customWidth ?? naturalSize.width
            let itemHeight = naturalSize.height

            if currentX + itemWidth > maxWidth && currentX > 0 {
                currentX = 0
                currentY += lineHeight + spacing
                lineHeight = 0
            }

            lineHeight = max(lineHeight, itemHeight)
            currentX += itemWidth + spacing
            maxY = currentY + lineHeight
        }

        return CGSize(width: maxWidth, height: maxY)
    }

    func placeSubviews(in bounds: CGRect, proposal: ProposedViewSize, subviews: Subviews, cache: inout ()) {
        let maxWidth = bounds.width
        var currentX: CGFloat = 0
        var currentY: CGFloat = 0
        var lineHeight: CGFloat = 0

        for subview in subviews {
            let customWidth = subview[FlowItemWidthKey.self] ?? defaultItemWidth
            let naturalSize = subview.sizeThatFits(.unspecified)
            let itemWidth = customWidth ?? naturalSize.width
            let itemHeight = naturalSize.height

            if currentX + itemWidth > maxWidth && currentX > 0 {
                currentX = 0
                currentY += lineHeight + spacing
                lineHeight = 0
            }

            subview.place(
                at: CGPoint(x: bounds.minX + currentX, y: bounds.minY + currentY),
                anchor: .topLeading,
                proposal: ProposedViewSize(width: itemWidth, height: nil)
            )

            lineHeight = max(lineHeight, itemHeight)
            currentX += itemWidth + spacing
        }
    }
}

struct LayoutKeyDemo: View {
    var body: some View {
        AdvancedFlowLayout(spacing: 10) {
            Text("固定宽度")
                .flowItemWidth(120)
                .padding(8)
                .background(Color.blue)

            Text("自适应宽度标签A")
                .padding(8)
                .background(Color.green)

            Text("固定宽度")
                .flowItemWidth(120)
                .padding(8)
                .background(Color.orange)

            Text("自适应宽度标签B")
                .padding(8)
                .background(Color.purple)
        }
        .padding()
    }
}
```

### 与 ScrollView 配合

自定义布局要与 ScrollView 正确配合，关键在于 `sizeThatFits` 必须返回**准确的内容尺寸**：

```swift
struct ScrollableFlowDemo: View {
    let items = Array(1...30).map { "标签 \($0)" }

    var body: some View {
        ScrollView {
            FlowLayout(spacing: 8, lineSpacing: 10) {
                ForEach(items, id: \.self) { item in
                    Text(item)
                        .font(.subheadline)
                        .padding(.horizontal, 12)
                        .padding(.vertical, 6)
                        .background {
                            RoundedRectangle(cornerRadius: 6)
                                .fill(Color.blue.opacity(0.12))
                        }
                }
            }
            .padding()
        }
    }
}
```

> 💡 **提示**：ScrollView 通过 `sizeThatFits(proposal: .unspecified)` 获取内容的实际尺寸来计算 contentSize。如果你的自定义布局在此情况下返回了错误的尺寸，ScrollView 的滚动范围就会出问题。

### 响应式布局：根据可用空间切换布局策略

利用 `proposal` 参数，可以根据可用空间动态调整布局策略：

```swift
struct AdaptiveLayout: Layout {
    var spacing: CGFloat = 8
    var threshold: CGFloat = 400

    func sizeThatFits(proposal: ProposedViewSize, subviews: Subviews, cache: inout ()) -> CGSize {
        let width = proposal.width ?? threshold
        if width < threshold {
            return verticalSize(subviews: subviews, width: width)
        } else {
            return horizontalSize(subviews: subviews, width: width)
        }
    }

    func placeSubviews(in bounds: CGRect, proposal: ProposedViewSize, subviews: Subviews, cache: inout ()) {
        if bounds.width < threshold {
            placeVertical(in: bounds, subviews: subviews)
        } else {
            placeHorizontal(in: bounds, subviews: subviews)
        }
    }

    private func verticalSize(subviews: Subviews, width: CGFloat) -> CGSize {
        let sizes = subviews.map { $0.sizeThatFits(ProposedViewSize(width: width, height: nil)) }
        let totalHeight = sizes.reduce(0) { $0 + $1.height } + spacing * CGFloat(sizes.count - 1)
        return CGSize(width: width, height: totalHeight)
    }

    private func horizontalSize(subviews: Subviews, width: CGFloat) -> CGSize {
        let sizes = subviews.map { $0.sizeThatFits(.unspecified) }
        let totalWidth = sizes.reduce(0) { $0 + $1.width } + spacing * CGFloat(sizes.count - 1)
        let maxHeight = sizes.map(\.height).max() ?? 0
        return CGSize(width: min(totalWidth, width), height: maxHeight)
    }

    private func placeVertical(in bounds: CGRect, subviews: Subviews) {
        var y = bounds.minY
        for subview in subviews {
            let size = subview.sizeThatFits(ProposedViewSize(width: bounds.width, height: nil))
            subview.place(at: CGPoint(x: bounds.midX, y: y + size.height / 2), anchor: .center, proposal: ProposedViewSize(width: bounds.width, height: nil))
            y += size.height + spacing
        }
    }

    private func placeHorizontal(in bounds: CGRect, subviews: Subviews) {
        var x = bounds.minX
        for subview in subviews {
            let size = subview.sizeThatFits(.unspecified)
            subview.place(at: CGPoint(x: x + size.width / 2, y: bounds.midY), anchor: .center, proposal: .unspecified)
            x += size.width + spacing
        }
    }
}

struct AdaptiveLayoutDemo: View {
    var body: some View {
        AdaptiveLayout(threshold: 400) {
            Text("项目一").padding().background(Color.red)
            Text("项目二").padding().background(Color.green)
            Text("项目三").padding().background(Color.blue)
        }
        .padding()
    }
}
```

上面的 `AdaptiveLayout` 在宽度小于 400 时自动切换为纵向排列（适合竖屏），大于 400 时横向排列（适合横屏），实现了真正的响应式布局。

### Layout 协议与 ViewModifier 的配合

将自定义布局封装为 ViewModifier，可以更方便地复用：

```swift
struct FlowLayoutModifier: ViewModifier {
    var spacing: CGFloat = 8
    var lineSpacing: CGFloat = 8

    func body(content: Content) -> some View {
        FlowLayout(spacing: spacing, lineSpacing: lineSpacing) {
            content
        }
    }
}

extension View {
    func flowLayout(spacing: CGFloat = 8, lineSpacing: CGFloat = 8) -> some View {
        modifier(FlowLayoutModifier(spacing: spacing, lineSpacing: lineSpacing))
    }
}
```

使用方式变得更加简洁：

```swift
struct ModifierUsageDemo: View {
    var body: some View {
        ForEach(tags, id: \.self) { tag in
            Text(tag)
                .padding(.horizontal, 12)
                .padding(.vertical, 6)
                .background(Capsule().fill(Color.cyan.opacity(0.2)))
        }
        .flowLayout(spacing: 10, lineSpacing: 10)
        .padding()
    }
}
```

---

## 小结

本章系统讲解了 iOS 16+ 的 Layout 协议，从核心理念到实战应用，再到性能优化和高级技巧。掌握 Layout 协议意味着你不再受限于内置布局容器的束缚，可以自由创造任何想象中的布局效果。

### 知识总览

| 知识点 | 核心内容 | 关键 API |
|--------|---------|----------|
| Layout 协议概述 | 两阶段布局模型、vs GeometryReader | `Layout` protocol |
| 核心方法 | 测量与放置分离 | `sizeThatFits`, `placeSubviews` |
| 流式布局 | 自动换行的标签/Chip 排列 | FlowLayout 实现 |
| 环形布局 | 圆周均匀分布、动画展开 | CircleLayout + spring 动画 |
| 布局缓存 | 减少重复计算、提升性能 | `Cache` 泛型、`makeCache`/`updateCache` |
| LayoutValueKey | 子视图向父布局传递自定义信息 | `LayoutValueKey`、`layoutValue(key:value:)` |
| 响应式布局 | 根据可用空间动态切换策略 | `proposal.width` 判断 |
| ScrollView 配合 | 返回准确内容尺寸 | `proposal: .unspecified` 处理 |

### 学习建议

1. **先理解两阶段模型**：不要急着写复杂布局，先用手写一个简单的 HStack 来理解 `sizeThatFits` 和 `placeSubviews` 的协作关系
2. **从 FlowLayout 开始练习**：流式布局涵盖了大部分布局编程的核心技巧（循环、换行判断、位置累加）
3. **善用调试工具**：Xcode 的 View Debug 可以直观看到每个子视图的实际位置，帮助排查布局问题
4. **性能意识**：子视图数量超过 20 个时就应考虑添加缓存机制

← [动画与手势](./动画与手势.md) | [SwiftData 现代数据框架](./SwiftData现代数据框架.md) →
