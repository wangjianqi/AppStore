# Instruments 性能分析实战

> 🎯 **本章目标**：掌握 Xcode Instruments 工具套件的核心使用方法，能够使用 Time Profiler 定位性能瓶颈，使用 Allocations 和 Leaks 检测内存问题，学会 SwiftUI 视图性能调试，建立系统化的性能优化工作流。

---

## 1. Instruments 概述

### 1.1 Instruments 是什么

Instruments 是 Xcode 自带的性能分析工具套件，它可以帮助开发者从 CPU、内存、网络、磁盘 I/O 等多个维度对 App 进行深入分析。与普通调试不同，Instruments 关注的不是"代码逻辑对不对"，而是"代码跑得快不快、省不省资源"。

打个生活类比：Instruments 就像 App 的**"体检中心"**，每个模板是一项检查——Time Profiler 是心电图，Allocations 是血常规，Leaks 是肿瘤筛查，Network 是血管造影。你需要根据症状选择对应的检查项目，然后由 Instruments 给出详细的诊断报告。

### 1.2 如何启动 Instruments

启动方式有三种：

1. **从 Xcode 启动**：菜单栏 Product → Profile（快捷键 `Cmd + I`），Xcode 会先编译项目，然后自动打开 Instruments 并弹出模板选择窗口。
2. **直接打开 Instruments**：在 Xcode 菜单栏 Xcode → Open Developer Tool → Instruments，然后手动选择要分析的 App 进程。
3. **从命令行启动**：使用 `instruments` 命令行工具，适合自动化场景。

> 💡 **提示**：使用 `Cmd + I` 启动时，Xcode 会使用 Release 配置编译，这比 Debug 配置更接近真实性能表现。如果直接用 Debug 模式分析，结果可能不准确。

### 1.3 常用模板一览

| 模板名称 | 主要用途 | 适用场景 |
|---------|---------|---------|
| Time Profiler | CPU 时间采样分析 | 界面卡顿、操作延迟、方法耗时过长 |
| Allocations | 内存分配追踪 | 内存持续增长、大对象排查 |
| Leaks | 内存泄漏检测 | 循环引用、未释放对象 |
| App Launch | 启动耗时分析 | 冷启动慢、首屏渲染慢 |
| Network | 网络请求分析 | 请求耗时、重复请求、数据量过大 |
| Core Data | 数据库操作分析 | 持久化查询慢、写入瓶颈 |
| SwiftUI | SwiftUI 视图调试 | 视图重绘过多、Body 计算频繁 |
| Metal System Trace | GPU 渲染分析 | 动画掉帧、渲染瓶颈 |
| File Activity | 文件 I/O 分析 | 磁盘读写频繁、文件操作慢 |
| Energy Log | 电量消耗分析 | 后台耗电、CPU 唤醒频繁 |

### 1.4 Instruments 界面介绍

Instruments 的界面主要由以下区域组成：

- **时间线面板（Timeline）**：位于界面顶部，以时间轴形式展示各项指标的实时数据曲线。每条轨道对应一个分析维度，可以拖动和缩放时间线来聚焦特定时段。
- **详情面板（Detail Panel）**：位于界面下方，展示选中时间段内的详细数据。不同模板的详情面板内容不同，例如 Time Profiler 显示调用树，Allocations 显示对象列表。
- **扩展详情面板（Extended Detail）**：位于界面右侧，展示选中条目的补充信息，如具体代码位置、调用栈等。
- **检查器面板（Inspector）**：左下角区域，可配置显示选项和运行设置。

> 💡 **提示**：在时间线上拖选一段区域，可以聚焦分析该时段的数据，这在定位偶发性问题时非常有效。

---

## 2. Time Profiler：CPU 性能分析

### 2.1 Time Profiler 的工作原理

Time Profiler 采用**采样法（Sampling）**工作：它每隔一小段时间（默认 1ms）暂停所有线程，记录当前每个线程的调用栈。经过大量采样后，统计每个函数出现在调用栈中的次数，以此估算各函数的 CPU 时间占比。

这就像统计一个公司各部门的工作时间——你不需要每秒盯着每个人，只需每隔一段时间拍一张照片，最后看哪个人出现在工位上的次数最多，就知道谁最忙。

采样法的优点是**性能开销低**，不会显著影响被分析的 App 运行；缺点是**精度有限**，非常短暂的操作可能被遗漏。

### 2.2 启动 Time Profiler 分析

1. 在 Xcode 中打开项目，按 `Cmd + I`
2. 在模板选择窗口中选择 **Time Profiler**
3. 点击 Choose，Instruments 会启动 App 并开始录制
4. 在 App 中执行你要分析的操作（如快速滚动列表）
5. 点击停止按钮结束录制

### 2.3 读懂调用树（Call Tree）

录制结束后，详情面板会显示调用树。每一行代表一个函数调用，包含以下关键信息：

- **Weight**：该函数消耗的 CPU 时间占总采样时间的百分比
- **Self**：仅该函数自身消耗的时间（不含子函数）
- **Running Time**：实际运行时间（毫秒）

调用树默认从主线程入口开始展开，层级很深。为了快速定位问题，需要掌握以下过滤技巧。

### 2.4 常用过滤选项

在详情面板底部的 Call Tree 区域，勾选以下选项可以大幅提升分析效率：

| 选项 | 作用 | 使用建议 |
|------|------|---------|
| Separate by Thread | 按线程分组显示 | 多线程问题时勾选 |
| Invert Call Tree | 反转调用树，从叶子节点开始 | 快速找到最耗时的叶子方法 |
| Hide System Libraries | 隐藏系统库调用 | **必勾**，只看自己的代码 |
| Flatten Recursion | 合并递归调用 | 递归函数时勾选 |
| Top Functions | 按总耗时排序 | 快速找到最耗时的函数 |

> ⚠️ **警告**：**Hide System Libraries** 是最常用的选项，但要注意，有时候性能问题恰恰出在系统库的调用方式上（如频繁调用系统 API），完全隐藏系统库可能遗漏线索。建议先勾选定位大致方向，必要时取消勾选深入查看。

### 2.5 定位耗时方法的完整步骤

1. 勾选 **Hide System Libraries** 和 **Invert Call Tree**
2. 按 Weight 列降序排列，找到占比最高的方法
3. 双击该方法，进入源码视图，查看具体哪一行耗时
4. 如果是调用子函数导致的耗时，回到调用树展开该节点继续追踪
5. 在右侧扩展详情面板查看完整调用栈，确认调用来源

### 2.6 实战案例：列表滚动卡顿的性能排查

假设一个 SwiftUI 列表在快速滚动时明显卡顿，使用 Time Profiler 排查：

```swift
struct ProductListView: View {
    let products: [Product]

    var body: some View {
        List(products) { product in
            ProductRowView(product: product)
        }
    }
}

struct ProductRowView: View {
    let product: Product

    var body: some View {
        HStack {
            AsyncImage(url: URL(string: product.imageURL))
                .frame(width: 80, height: 80)
            VStack(alignment: .leading) {
                Text(product.name)
                Text(product.formattedPrice)
                Text(product.description)
            }
        }
    }
}
```

Time Profiler 分析后发现 `ProductRowView.body` 的计算时间占比极高。进一步查看源码，发现 `product.formattedPrice` 每次都在做复杂的字符串格式化：

```swift
var formattedPrice: String {
    let formatter = NumberFormatter()
    formatter.numberStyle = .currency
    formatter.locale = Locale(identifier: "zh_CN")
    return formatter.string(from: NSNumber(value: price)) ?? ""
}
```

每次创建 `NumberFormatter` 实例开销很大。优化方案是缓存 formatter：

```swift
private static let priceFormatter: NumberFormatter = {
    let formatter = NumberFormatter()
    formatter.numberStyle = .currency
    formatter.locale = Locale(identifier: "zh_CN")
    return formatter
}()

var formattedPrice: String {
    Self.priceFormatter.string(from: NSNumber(value: price)) ?? ""
}
```

### 2.7 Self Time vs Total Time

这是 Time Profiler 中最容易混淆的两个概念：

- **Self Time**：函数自身代码执行的时间，不包括它调用的子函数。如果 Self Time 很高，说明这个函数本身有计算密集的操作。
- **Total Time**：函数及其所有子函数的总执行时间。如果 Total Time 很高但 Self Time 很低，说明耗时来自子函数调用。

| 指标 | 含义 | 高值暗示 |
|------|------|---------|
| Self Time 高 | 函数本身耗时 | 该函数内有计算密集操作 |
| Total Time 高、Self Time 低 | 子函数耗时 | 需要展开查看子函数 |
| Self Time = Total Time | 没有子函数调用 | 叶子节点，优化目标明确 |

### 2.8 优化建议与验证

优化完成后，务必重新用 Time Profiler 跑一遍，对比优化前后的数据。步骤：

1. 记录优化前的 Weight 和 Running Time
2. 实施优化
3. 同样操作下重新录制
4. 对比数据，确认改善幅度

> 💡 **提示**：每次只改一处，然后验证效果。同时改多处会导致无法判断哪个改动有效，甚至可能引入新问题。

---

## 3. Allocations 与 Leaks：内存分析

### 3.1 Allocations 工具概述

Allocations 工具追踪 App 运行期间所有对象的内存分配和释放。它可以回答以下问题：

- App 当前占用了多少内存？
- 哪种类型的对象占用最多？
- 内存是否在持续增长（不释放）？
- 某个时间点新分配了哪些对象？

### 3.2 查看内存增长趋势

启动 Allocations 模板后，时间线上会显示两条曲线：

- **All Heap Allocations**：堆上分配的对象内存
- **All Anonymous VM**：虚拟内存映射（如 mmap、图片解码缓冲区）

关注 **Overall Bytes**（总分配量）和 **Persistent Bytes**（当前仍存活的分配量）。如果 Persistent Bytes 持续上升不回落，说明存在内存增长问题。

### 3.3 识别大对象和异常分配

在详情面板中，按 **Persistent Bytes** 降序排列，可以快速找到当前占用内存最多的对象类型。点击某个类型可以展开查看该类型的所有实例，再点击实例可以在右侧查看分配时的调用栈。

常见的异常分配模式：

- 某个类型的实例数量异常多（可能有缓存未清理）
- 单个对象占用内存过大（如加载了原始分辨率的图片）
- 短时间内大量临时对象分配（如循环中创建大量字符串）

### 3.4 Mark Heap 标记法

Mark Heap 是 Allocations 中非常实用的功能，用于对比两个时间点的内存差异。操作步骤：

1. 在 App 进入某个页面之前，点击 **Mark Heap** 按钮（或按快捷键 `Cmd + M`），标记为 Heap 1
2. 操作 App（如进入页面再返回）
3. 再次点击 **Mark Heap**，标记为 Heap 2
4. 点击 Heap 2 旁边的箭头，查看 **Growth** 列

Growth 列显示两个标记之间新增且未释放的对象。如果进入页面再返回后仍有大量 Growth，说明该页面存在内存泄漏。

### 3.5 Leaks 工具：检测循环引用

Leaks 工具自动检测 App 中的内存泄漏。当它发现一个对象没有任何引用指向它、但系统又无法回收它时，就会标记为泄漏。

在 Instruments 中，Leaks 以红色柱状条显示在时间线上。点击红色区域，详情面板会列出所有泄漏的对象。选中一个泄漏对象，右侧扩展详情会显示**循环引用链**——这是修复泄漏的关键线索。

### 3.6 常见内存泄漏模式

**闭包循环引用**是最常见的泄漏模式：

```swift
class ViewModel: ObservableObject {
    var onDataLoaded: (([Item]) -> Void)?

    func loadData() {
        APIClient.fetch { [weak self] items in
            self?.process(items)
            self?.onDataLoaded?(items)
        }
    }

    func process(_ items: [Item]) {
    }
}
```

如果闭包中不使用 `[weak self]`，闭包会强引用 self，而 self 又持有闭包属性，形成循环引用。

**Delegate 强引用**：

```swift
protocol ListPresenterDelegate: AnyObject {
    func didUpdateItems()
}

class ListPresenter {
    weak var delegate: ListPresenterDelegate?

    func refresh() {
        delegate?.didUpdateItems()
    }
}
```

Delegate 必须声明为 `weak`，否则 Presenter 和 Delegate 互相持有，形成循环引用。

**Timer 未释放**：

```swift
class PollingManager {
    private var timer: Timer?

    func startPolling() {
        timer = Timer.scheduledTimer(
            withTimeInterval: 5.0,
            repeats: true
        ) { [weak self] _ in
            self?.fetchUpdates()
        }
    }

    func stopPolling() {
        timer?.invalidate()
        timer = nil
    }

    deinit {
        stopPolling()
    }
}
```

Timer 的 target 会被强引用，如果不使用 `[weak self]` 且忘记 invalidate，Timer 和 target 互相持有，永远不会释放。

### 3.7 SwiftUI 中常见的内存泄漏场景

SwiftUI 的声明式语法容易隐藏引用关系，导致不易察觉的泄漏：

```swift
struct ProfileView: View {
    @StateObject var viewModel = ProfileViewModel()

    var body: some View {
        ScrollView {
            VStack {
                ForEach(viewModel.posts) { post in
                    PostCellView(post: post, onTap: {
                        viewModel.selectPost(post)
                    })
                }
            }
        }
        .task {
            await viewModel.loadProfile()
        }
    }
}
```

如果 `viewModel.selectPost` 在闭包中强引用了外部对象，或者 `.task` 中的异步任务没有正确取消，都可能造成泄漏。建议在视图销毁时确保取消所有异步任务：

```swift
struct ProfileView: View {
    @StateObject var viewModel = ProfileViewModel()

    var body: some View {
        ScrollView {
            VStack {
                ForEach(viewModel.posts) { post in
                    PostCellView(post: post, onTap: {
                        viewModel.selectPost(post)
                    })
                }
            }
        }
        .task(id: viewModel.userId) {
            await viewModel.loadProfile()
        }
    }
}
```

### 3.8 修复内存泄漏的步骤

1. 使用 Leaks 工具或 Mark Heap 确认存在泄漏
2. 在详情面板找到泄漏的对象类型
3. 在扩展详情面板查看循环引用链
4. 找到引用链中可以断开的一环（通常是闭包捕获或属性引用）
5. 将强引用改为 `weak` 或 `unowned`
6. 重新运行 Leaks 工具验证修复效果

> ⚠️ **警告**：修复泄漏时不要盲目添加 `weak`。有些引用关系本就该是强引用（如父持有子），错误地使用 `weak` 可能导致对象提前释放而崩溃。务必先理解引用链，再决定在哪里断开。

---

## 4. SwiftUI 视图性能调试

### 4.1 SwiftUI Instruments 模板

Xcode 提供了专门的 SwiftUI Instruments 模板，可以追踪 SwiftUI 视图的生命周期事件，包括 Body 计算、属性更新、视图挂载和卸载等。

启动方式：`Cmd + I` → 选择 **SwiftUI** 模板。

### 4.2 视图 Body 计算次数分析

SwiftUI 的核心机制是：当 `@State`、`@ObservedObject` 等属性变化时，依赖它的视图会重新计算 Body。如果 Body 计算过于频繁或耗时过长，就会导致界面卡顿。

在 SwiftUI 模板的时间线上，可以查看 **View Body** 轨道，它会标记每次 Body 计算的时间点。点击某个标记，详情面板会显示是哪个视图的 Body 被计算，以及触发计算的属性变更。

### 4.3 识别不必要的视图重绘

一个常见问题是：某个深层子视图的 Body 被频繁计算，但它依赖的数据其实并没有变化。例如：

```swift
struct ContentView: View {
    @State var counter = 0
    @State var items: [Item] = []

    var body: some View {
        VStack {
            Text("Count: \(counter)")
            ItemListView(items: items)
        }
    }
}
```

当 `counter` 变化时，`ItemListView` 的 Body 也会被重新计算，即使 `items` 没有任何变化。这就是不必要重绘。

### 4.4 @State 变更导致的级联刷新

SwiftUI 的视图刷新是自上而下的：父视图的 Body 重新计算，所有子视图也会跟着重新计算（除非子视图是独立的）。这种级联效应在小项目中不明显，但在大型项目中会严重影响性能。

识别级联刷新的方法：在 SwiftUI 模板中，按时间顺序查看 Body 计算事件，如果发现一个属性变更后大量不相关视图的 Body 被计算，就说明存在级联刷新问题。

### 4.5 优化策略

**策略一：拆分视图**

将大视图拆分为小组件，让每个组件只依赖它需要的数据：

```swift
struct ContentView: View {
    @State var counter = 0
    @State var items: [Item] = []

    var body: some View {
        VStack {
            CounterView(counter: counter)
            ItemListView(items: items)
        }
    }
}

struct CounterView: View {
    let counter: Int

    var body: some View {
        Text("Count: \(counter)")
    }
}
```

这样 `counter` 变化时，只有 `CounterView` 会重新计算，`ItemListView` 不受影响。

**策略二：使用 @Observable 精准刷新**

iOS 17 引入的 `@Observable` 宏可以实现属性级别的精准刷新：

```swift
@Observable
class ContentViewModel {
    var counter = 0
    var items: [Item] = []
}

struct ContentView: View {
    @State var viewModel = ContentViewModel()

    var body: some View {
        VStack {
            Text("Count: \(viewModel.counter)")
            ItemListView(items: viewModel.items)
        }
    }
}
```

使用 `@Observable` 后，SwiftUI 会自动追踪每个视图实际访问了哪些属性，只有被访问的属性变化时才会触发对应视图的 Body 重算。`counter` 变化不会导致 `ItemListView` 重绘。

**策略三：equatable() 修饰符**

对于接收值类型参数的视图，可以使用 `.equatable()` 修饰符，让 SwiftUI 在参数未变化时跳过 Body 计算：

```swift
struct ProductRowView: View, Equatable {
    let product: Product

    static func == (lhs: Self, rhs: Self) -> Bool {
        lhs.product.id == rhs.product.id &&
        lhs.product.name == rhs.product.name &&
        lhs.product.price == rhs.product.price
    }

    var body: some View {
        HStack {
            Text(product.name)
            Text(product.formattedPrice)
        }
    }
}

ProductRowView(product: product)
    .equatable()
```

### 4.6 实战案例：优化一个卡顿的列表

一个包含 1000 条数据的列表在滚动时明显卡顿，使用 SwiftUI 模板分析后发现 `RowView` 的 Body 计算次数远超预期。

原始代码：

```swift
struct ListView: View {
    @StateObject var viewModel = ListViewModel()

    var body: some View {
        List(viewModel.items) { item in
            RowView(item: item, selectedId: viewModel.selectedId)
        }
    }
}

struct RowView: View {
    let item: Item
    let selectedId: UUID?

    var body: some View {
        HStack {
            Text(item.title)
            Spacer()
            if item.id == selectedId {
                Image(systemName: "checkmark")
            }
        }
    }
}
```

问题在于 `selectedId` 变化时，所有 `RowView` 都会重算 Body。优化方案是将选中状态拆分到独立视图：

```swift
struct RowView: View {
    let item: Item
    let isSelected: Bool

    var body: some View {
        HStack {
            Text(item.title)
            Spacer()
            if isSelected {
                Image(systemName: "checkmark")
            }
        }
    }
}
```

并配合 `@Observable` 的精准刷新，确保只有实际被选中的行才会重绘。优化后，滚动帧率从 30fps 提升到 60fps。

---

## 5. 网络性能分析

### 5.1 Network 模板的使用

Network 模板可以追踪 App 的所有网络活动，包括 HTTP 请求、TCP 连接、DNS 解析等。启动方式：`Cmd + I` → 选择 **Network** 模板。

录制后，时间线上会显示网络活动曲线，详情面板列出所有请求。

### 5.2 分析 HTTP 请求耗时

在详情面板中，每个请求显示以下信息：

- **Start Time**：请求开始时间
- **Duration**：请求总耗时
- **Request Size / Response Size**：请求和响应的数据量
- **Status Code**：HTTP 状态码

按 Duration 降序排列，可以快速找到最慢的请求。点击某个请求，扩展详情会展示 DNS 解析、TCP 握手、TLS 握手、请求发送、等待响应、数据接收各阶段的耗时。

### 5.3 识别慢请求和重复请求

**慢请求**的常见原因：

| 原因 | 特征 | 优化方向 |
|------|------|---------|
| 服务器响应慢 | Waiting 阶段耗时长 | 后端优化、增加 CDN |
| 数据量过大 | Receiving 阶段耗时长 | 压缩响应、分页加载 |
| DNS 解析慢 | DNS 阶段耗时长 | 预解析域名、使用 HTTPDNS |
| TLS 握手慢 | TLS 阶段耗时长 | 复用连接、启用 TLS 1.3 |

**重复请求**的识别方法：在详情面板中按 URL 排序，如果同一个 URL 在短时间内出现多次，就可能是重复请求。常见场景包括：列表刷新时未取消旧请求、多个页面同时请求相同接口。

### 5.4 数据加载优化建议

```swift
class APIClient {
    private let session: URLSession

    init() {
        let config = URLSessionConfiguration.default
        config.timeoutIntervalForRequest = 15
        config.waitsForConnectivity = true
        config.requestCachePolicy = .returnCacheDataElseLoad
        session = URLSession(configuration: config)
    }

    func fetch<T: Decodable>(_ url: URL) async throws -> T {
        let (data, response) = try await session.data(from: url)
        guard let httpResponse = response as? HTTPURLResponse,
              (200...299).contains(httpResponse.statusCode) else {
            throw APIError.invalidResponse
        }
        return try JSONDecoder().decode(T.self, from: data)
    }
}
```

关键优化点：

- 设置合理的超时时间，避免用户长时间等待
- 启用缓存策略，减少重复请求
- 使用 async/await 管理并发，避免回调地狱
- 对大列表使用分页加载，减少单次数据量

> 💡 **提示**：网络优化不仅要关注请求速度，还要关注用户体验。即使请求本身无法更快，也可以通过骨架屏、渐进式加载、离线缓存等手段让用户感觉更快。

---

## 6. 性能优化工作流

### 6.1 建立性能基线（Baseline）

性能优化的第一步不是改代码，而是**建立基线**——记录当前的关键性能指标，作为后续对比的参照。

基线应包含以下指标：

| 指标 | 测量方式 | 目标参考值 |
|------|---------|-----------|
| 启动时间 | App Launch 模板 | 冷启动 < 400ms |
| 帧率 | Time Profiler / Metal | 滚动时 ≥ 60fps |
| 内存峰值 | Allocations | 不超过设备物理内存的 50% |
| 网络首屏时间 | Network 模板 | < 2s（WiFi 环境） |
| 内存泄漏数 | Leaks | 0 |

### 6.2 对比优化前后的数据

每次优化后，在相同设备和相同操作下重新录制，对比数据变化。建议记录一个优化日志：

```
优化项：缓存 NumberFormatter 实例
优化前：ProductRowView.body 平均耗时 12ms
优化后：ProductRowView.body 平均耗时 2ms
改善幅度：83%
副作用：无
```

### 6.3 自动化性能测试（XCTest 性能测试）

XCTest 提供了性能测试 API，可以将性能指标纳入自动化测试：

```swift
import XCTest

final class PerformanceTests: XCTestCase {
    func testProductListRendering() throws {
        let products = (0..<1000).map { Product(id: UUID(), name: "Item \($0)", price: Double($0)) }

        measure {
            let view = ProductListView(products: products)
            _ = view.body
        }
    }

    func testJSONParsingPerformance() throws {
        let data = try Data(contentsOf: Bundle.main.url(forResource: "products", withExtension: "json")!)

        measure {
            _ = try? JSONDecoder().decode([Product].self, from: data)
        }
    }
}
```

`measure` 闭包会执行 10 次，取平均值作为基线。如果后续运行的结果偏离基线超过标准差范围，测试会失败，提醒你性能出现了回归。

### 6.4 持续监控：Xcode Cloud 集成

将性能测试集成到 Xcode Cloud 的 CI/CD 流程中，可以在每次提交代码时自动运行性能测试，及时发现性能回归：

1. 在 Xcode Cloud 的 workflow 配置中添加性能测试 Action
2. 设置基线文件，提交到仓库
3. 每次 PR 自动运行性能测试，结果与基线对比
4. 如果性能回归超过阈值，PR 会被标记为需要检查

### 6.5 性能优化的优先级策略

不是所有性能问题都值得优化。遵循以下优先级：

1. **崩溃和内存泄漏**：必须立即修复，影响 App 稳定性
2. **启动时间**：用户第一印象，优先级高
3. **主线程卡顿**：直接影响交互体验，优先级高
4. **内存占用过高**：可能导致系统杀进程，优先级中
5. **网络请求慢**：可通过缓存和骨架屏缓解，优先级中
6. **包体积过大**：影响下载转化率，优先级低

> 💡 **提示**：性能优化要遵循"二八定律"——80% 的性能问题来自 20% 的代码。先用 Instruments 定位到真正的瓶颈，再集中精力优化，不要凭感觉猜测。

---

## 小结

| 知识点 | 核心内容 | 关键操作 |
|--------|---------|---------|
| Instruments 概述 | Xcode 自带性能分析工具套件 | Cmd+I 启动，选择对应模板 |
| 常用模板 | Time Profiler、Allocations、Leaks、SwiftUI、Network 等 | 根据症状选择模板 |
| Time Profiler | CPU 采样分析，定位耗时方法 | 勾选 Hide System Libraries + Invert Call Tree |
| Self vs Total Time | Self 是函数自身耗时，Total 包含子函数 | Self 高优化函数本身，Total 高查找子函数 |
| Allocations | 追踪内存分配和释放 | 关注 Persistent Bytes 持续增长 |
| Mark Heap | 对比两个时间点的内存差异 | Cmd+M 标记，查看 Growth 列 |
| Leaks | 检测循环引用和内存泄漏 | 查看红色标记和循环引用链 |
| 常见泄漏模式 | 闭包循环引用、Delegate 强引用、Timer 未释放 | 使用 weak/unowned 断开引用链 |
| SwiftUI 调试 | Body 计算次数、不必要重绘 | 使用 SwiftUI 模板追踪视图事件 |
| 视图优化策略 | 拆分视图、@Observable、equatable() | 减少级联刷新范围 |
| 网络分析 | 请求耗时、慢请求、重复请求 | Network 模板 + Duration 排序 |
| 性能基线 | 记录关键指标的当前值 | 每次优化前后对比数据 |
| XCTest 性能测试 | measure 闭包自动检测性能回归 | 集成到 CI/CD 持续监控 |
| 优化优先级 | 崩溃 > 启动 > 卡顿 > 内存 > 网络 > 包体积 | 先定位瓶颈再优化 |

← [调试与性能优化](./调试与性能优化.md) | [Xcode Cloud CI/CD](./Xcode-Cloud-CI-CD.md) →