# 29-Swift 泛型与类型系统深入

## 本章目标

- 掌握泛型进阶用法：where 子句、多约束组合、泛型下标，写出真正灵活的泛型代码
- 理解关联类型 associatedtype 的设计动机与使用方式，区分泛型协议与关联类型的选择
- 掌握 Swift 5.7+ 主要关联类型 Primary Associated Types 语法，简化泛型表达
- 深入理解不透明类型 some 与存在类型 any 的语义差异、性能影响与使用场景
- 学会条件遵循 Conditional Conformance，让标准库类型自动获得能力
- 理解类型擦除的原理，能手写类型擦除器解决泛型信息泄漏问题
- 将泛型知识应用到 SwiftUI 开发中，理解 View、PreferenceKey、Environment 的泛型设计

---

## 1. 泛型深入

### 1.1 从生活类比理解泛型

泛型就像**快递柜**——柜子的结构是固定的（大小、锁、门），但你可以放任何东西进去：书、零食、手机。你不需要为每种物品建一种柜子，一个通用设计就够了。

```swift
func swapValues<T>(_ a: inout T, _ b: inout T) {
    let temp = a
    a = b
    b = temp
}

var x = 10, y = 20
swapValues(&x, &y)

var a = "Hello", b = "World"
swapValues(&a, &b)
```

### 1.2 泛型类型与泛型函数

泛型不仅用于函数，还可以用于结构体、类、枚举：

```swift
struct Stack<Element> {
    private var items: [Element] = []

    mutating func push(_ item: Element) {
        items.append(item)
    }

    mutating func pop() -> Element? {
        items.popLast()
    }

    var top: Element? {
        items.last
    }
}

var intStack = Stack<Int>()
intStack.push(1)
intStack.push(2)

var stringStack = Stack<String>()
stringStack.push("Swift")
```

枚举也可以是泛型的——标准库的 `Optional` 就是最好的例子：

```swift
enum Optional<Wrapped> {
    case some(Wrapped)
    case none
}

enum Result<Success, Failure: Error> {
    case success(Success)
    case failure(Failure)
}
```

### 1.3 泛型约束

裸泛型 `T` 能做的事很少——编译器不知道它支持哪些操作。约束告诉编译器"T 至少具备什么能力"：

```swift
func findIndex<T: Equatable>(of value: T, in array: [T]) -> Int? {
    for (index, item) in array.enumerated() {
        if item == value {
            return index
        }
    }
    return nil
}
```

| 约束写法 | 含义 |
|----------|------|
| `<T: Equatable>` | T 必须遵循 Equatable |
| `<T: Comparable>` | T 必须遵循 Comparable |
| `<T: Hashable>` | T 必须遵循 Hashable |
| `<T: Codable>` | T 必须遵循 Codable |
| `<T: AnyObject>` | T 必须是引用类型 |
| `<T: UIView>` | T 必须是 UIView 或其子类 |

### 1.4 where 子句

`where` 子句提供了更灵活的约束方式，可以用于泛型函数、扩展、关联类型等：

```swift
func allEqual<T>(_ array: [T]) -> Bool where T: Equatable {
    guard let first = array.first else { return true }
    return array.allSatisfy { $0 == first }
}
```

`where` 子句在扩展中特别强大：

```swift
extension Stack where Element: Equatable {
    func contains(_ item: Element) -> Bool {
        items.contains(item)
    }
}

extension Stack where Element: Numeric {
    func sum() -> Element {
        items.reduce(0, +)
    }
}
```

> 💡 只有当 `Element` 满足约束时，这些方法才会存在。不满足约束的 `Stack` 实例调用这些方法会直接编译报错——这是类型安全的保证。

### 1.5 多约束组合

一个类型参数可以同时满足多个约束，用 `&` 连接：

```swift
func compareAndHash<T>(_ a: T, _ b: T) -> Int where T: Comparable & Hashable {
    if a < b { return -1 }
    if a > b { return 1 }
    return a.hashValue
}
```

多个类型参数之间也可以建立约束关系：

```swift
func merge<C1: Collection, C2: Collection>(
    _ c1: C1, _ c2: C2
) -> [C1.Element] where C1.Element == C2.Element, C1.Element: Hashable {
    var result = Set<C1.Element>()
    for item in c1 { result.insert(item) }
    for item in c2 { result.insert(item) }
    return Array(result)
}
```

> ⚠️ `where` 子句中的约束顺序不影响语义，但建议把 `Equatable` / `Hashable` 等基础约束写在前面，把 `==` 类型等式约束写在后面，提高可读性。

### 1.6 泛型下标

Swift 5.2 开始支持泛型下标，让下标访问也能享受泛型的灵活性：

```swift
struct Matrix {
    let rows: Int, columns: Int
    private var grid: [Double]

    init(rows: Int, columns: Int) {
        self.rows = rows
        self.columns = columns
        grid = Array(repeating: 0, count: rows * columns)
    }

    subscript(row: Int, col: Int) -> Double {
        get { grid[row * columns + col] }
        set { grid[row * columns + col] = newValue }
    }

    subscript<Indices: Sequence>(indices: Indices) -> [Double]
        where Indices.Element == Int {
        indices.map { grid[$0] }
    }
}

let matrix = Matrix(rows: 3, columns: 3)
let values = matrix[indices: [0, 4, 8]]
```

---

## 2. 关联类型 associatedtype

### 2.1 为什么需要关联类型

协议中如果直接用泛型参数，会导致一个类型只能遵循一种"版本"的协议。关联类型让协议成为一个**插槽**，遵循类型自己决定插什么：

```swift
protocol Container {
    associatedtype Item
    mutating func append(_ item: Item)
    var count: Int { get }
    subscript(i: Int) -> Item { get }
}
```

> 💡 生活类比：关联类型就像 USB 接口——协议定义了"有个接口"，至于插 U 盘还是键盘，由具体设备决定。

### 2.2 实现关联类型

```swift
struct IntStack: Container {
    typealias Item = Int

    private var items: [Int] = []

    mutating func append(_ item: Int) {
        items.append(item)
    }

    var count: Int { items.count }

    subscript(i: Int) -> Int {
        items[i]
    }
}
```

Swift 通常可以自动推断 `associatedtype` 的具体类型，`typealias Item = Int` 这行可以省略——编译器根据 `append(_ item: Int)` 自动推断出 `Item == Int`。

### 2.3 Self 关联

协议中的 `Self` 代表遵循协议的具体类型本身。它是 Swift 实现类型安全相等比较的关键：

```swift
protocol Equatable {
    static func == (lhs: Self, rhs: Self) -> Bool
}
```

当 `Int` 遵循 `Equatable` 时，`Self` 就是 `Int`；当 `String` 遵循时，`Self` 就是 `String`。这保证了你不会拿 `Int` 和 `String` 比较。

```swift
protocol Copyable {
    func copy() -> Self
}

class Document: Copyable {
    var title: String

    init(title: String) {
        self.title = title
    }

    func copy() -> Self {
        return type(of: self).init(title: title)
    }

    required init(title: String) {
        self.title = title
    }
}
```

### 2.4 关联类型的约束

关联类型本身也可以有约束，进一步限定遵循类型的行为：

```swift
protocol SortedContainer {
    associatedtype Element: Comparable
    mutating func insert(_ element: Element)
    var sortedElements: [Element] { get }
}

struct SortedArray<Element: Comparable>: SortedContainer {
    private var items: [Element] = []

    mutating func insert(_ element: Element) {
        items.append(element)
        items.sort()
    }

    var sortedElements: [Element] { items }
}
```

### 2.5 泛型协议 vs 关联类型的选择

| 场景 | 选择 | 原因 |
|------|------|------|
| 协议作为类型约束（`<T: Protocol>`） | 关联类型 | 编译器能推断具体类型，类型安全 |
| 需要同一类型多种遵循方式 | 泛型参数 | 泛型参数在声明时确定，可以多次实例化 |
| 协议用作存在类型（`any Protocol`） | 关联类型 | Swift 5.7+ 支持，配合 `some` / `any` 使用 |
| 标准库模式（Collection、View） | 关联类型 | 这是 Swift 的惯用模式 |

> ⚠️ Swift 不支持"泛型协议"（`protocol Protocol<T>`），但 Swift 5.7 的主要关联类型语法在调用侧达到了类似效果。

---

## 3. 主要关联类型 Primary Associated Types

### 3.1 Swift 5.7 的新语法

Swift 5.7 允许在协议声明时用尖括号标注"主要关联类型"，让调用侧可以像泛型一样指定具体类型：

```swift
protocol Container<Element> {
    associatedtype Element
    mutating func append(_ item: Element)
    var count: Int { get }
    subscript(i: Int) -> Element { get }
}
```

标准库中的经典例子：

```swift
protocol Collection<Element> {
    associatedtype Element
    associatedtype Index: Comparable
    var startIndex: Index { get }
    var endIndex: Index { get }
    subscript(position: Index) -> Element { get }
}
```

### 3.2 配合 some 和 any 使用

有了主要关联类型，`some` 和 `any` 可以直接指定具体类型：

```swift
// Swift 5.7 之前
func makeContainer() -> some Container {
    IntStack()
}

// Swift 5.7+：可以明确指定 Element 类型
func makeIntContainer() -> some Container<Int> {
    IntStack()
}

func process(container: any Container<String>) {
    for i in 0..<container.count {
        print(container[i])
    }
}
```

### 3.3 标准库中的主要关联类型

| 协议 | 主要关联类型 |
|------|-------------|
| `Collection<Element>` | Element |
| `Sequence<Element>` | Element |
| `AsyncSequence<Element>` | Element |
| `Publisher<Output, Failure>` | Output, Failure |

> 💡 主要关联类型的设计原则：选择协议中"最核心"的 1-2 个关联类型作为主要关联类型。不要把所有关联类型都标为主要——那会让语法变得臃肿。

---

## 4. 不透明类型 some

### 4.1 some View 原理

SwiftUI 中最常见的不透明类型就是 `some View`：

```swift
struct ContentView: View {
    var body: some View {
        Text("Hello")
    }
}
```

`some View` 的含义是："我返回一个**具体的** View 类型，但调用方不需要知道具体是什么，只需要知道它遵循 View 协议。"

> 💡 生活类比：some 就像餐厅菜单上写"主厨推荐"——你知道会有一道菜，但不需要知道具体是什么菜。重要的是它一定是一道菜（遵循协议），而且每次点的都是同一道菜（具体类型固定）。

### 4.2 不透明类型的两个核心语义

| 语义 | 说明 |
|------|------|
| **具体类型固定** | 同一个返回路径总是返回同一类型，编译器知道但调用方不知道 |
| **调用方只看协议** | 调用方只能使用协议中定义的 API，不能访问具体类型的方法 |

```swift
func makeValue() -> some Numeric {
    return 42
}

let value = makeValue()
// value 的静态类型是 some Numeric
// 可以做算术运算，但编译器知道底层是 Int
```

### 4.3 some 的限制

```swift
// ❌ 错误：不同分支返回不同类型
func makeView(showDetail: Bool) -> some View {
    if showDetail {
        return Text("Detail")
    } else {
        return Image("placeholder")
    }
}

// ✅ 正确：用 Group 包装保证同一类型
func makeView(showDetail: Bool) -> some View {
    Group {
        if showDetail {
            Text("Detail")
        } else {
            Image("placeholder")
        }
    }
}
```

> ⚠️ `some` 要求所有返回路径返回**同一具体类型**。Swift 5.9 对此有所放宽（支持返回不同但遵循同一协议的类型），但大多数场景仍需保持一致。

### 4.4 some vs 泛型对比

| 特性 | 泛型 `<T: Protocol>` | 不透明类型 `some Protocol` |
|------|---------------------|--------------------------|
| 类型由谁决定 | 调用方 | 实现方 |
| 类型可见性 | 调用方可见 | 调用方不可见 |
| 多态性 | 调用方传入不同类型 | 实现方返回固定类型 |
| 典型场景 | 通用算法 | 隐藏实现细节 |
| 性能 | 静态派发 | 静态派发 |

```swift
// 泛型：调用方决定 T
func first<T: Collection>(of collection: T) -> T.Element? {
    collection.first
}

// some：实现方决定具体类型
func makeCollection() -> some Collection<Int> {
    [1, 2, 3]
}
```

---

## 5. 存在类型 any

### 5.1 any Protocol 语法

Swift 5.6 引入 `any` 关键字，Swift 5.7+ 强制要求在用作存在类型时显式标注 `any`：

```swift
protocol Drawable {
    func draw()
}

struct Circle: Drawable {
    func draw() { print("Drawing circle") }
}

struct Square: Drawable {
    func draw() { print("Drawing square") }
}

// any：可以存储任何遵循 Drawable 的类型
let shapes: [any Drawable] = [Circle(), Square()]
for shape in shapes {
    shape.draw()
}
```

> 💡 生活类比：`any` 就像"盲盒"——你知道里面一定是个玩具（遵循协议），但具体是什么玩具在运行时才揭晓，而且每次可能不同。

### 5.2 存在类型的性能开销

| 方面 | `some Protocol` | `any Protocol` |
|------|----------------|----------------|
| 派发方式 | 静态派发（编译期确定） | 动态派发（运行时查表） |
| 内存布局 | 编译期已知大小 | 需要 existential container |
| 性能 | 零开销 | 有间接调用开销 |
| 类型信息 | 编译期保留 | 运行时擦除 |

existential container 的内部结构：

```
┌─────────────────────────┐
│  witness table pointer  │ ← 方法查找表
├─────────────────────────┤
│  value buffer (3 words) │ ← 内联存储小对象
├─────────────────────────┤
│  reference count        │ ← 引用计数（如需要）
└─────────────────────────┘
```

当存储的值超过 3 个字（24 字节 on 64-bit）时，会在堆上分配额外内存，带来额外的分配/释放开销。

### 5.3 any vs some 对比表

| 特性 | `some Protocol` | `any Protocol` |
|------|----------------|----------------|
| 类型确定性 | 编译期固定一种类型 | 运行时可变 |
| 异构集合 | ❌ 不支持 | ✅ 支持 |
| 性能 | 零开销抽象 | 有动态派发开销 |
| 类型转换 | 不需要 | 可能需要 `as?` |
| Swift 版本 | 5.1+ | 5.6+（显式标注） |
| 典型场景 | 函数返回值 | 集合存储多种类型 |

### 5.4 何时用 any

```swift
// ✅ 需要异构集合时用 any
let listeners: [any EventListener] = [
    AnalyticsListener(),
    LoggingListener(),
    CrashReportListener()
]

// ✅ 运行时类型可能变化时用 any
func makeRenderer(apiVersion: Int) -> any Renderer {
    if apiVersion >= 3 {
        return ModernRenderer()
    } else {
        return LegacyRenderer()
    }
}

// ❌ 不需要异构时优先用 some
func makeBackgroundView() -> some View {
    Color.blue
}
```

> ⚠️ Swift 6 中，省略 `any` 会在严格并发模式下产生警告甚至错误。养成习惯：凡是把协议当类型使用（而非约束），就加 `any`。

---

## 6. 条件遵循 Conditional Conformance

### 6.1 核心概念

条件遵循的意思是："当某个条件满足时，类型才遵循某个协议。"最经典的例子就是标准库中 `Array` 的 `Equatable` 遵循：

```swift
extension Array: Equatable where Element: Equatable {
    static func == (lhs: [Element], rhs: [Element]) -> Bool {
        lhs.count == rhs.count && zip(lhs, rhs).allSatisfy(==)
    }
}
```

> 💡 生活类比：条件遵循就像"有驾照才能开车"——你（Array）本身不会自动开车（遵循 Equatable），只有当你有驾照（Element 是 Equatable）时才可以。

### 6.2 标准库中的条件遵循示例

| 类型 | 条件遵循 | 条件 |
|------|---------|------|
| `Array<Element>` | `Equatable` | `Element: Equatable` |
| `Array<Element>` | `Hashable` | `Element: Hashable` |
| `Array<Element>` | `Codable` | `Element: Codable` |
| `Optional<Wrapped>` | `Equatable` | `Wrapped: Equatable` |
| `Optional<Wrapped>` | `Hashable` | `Wrapped: Hashable` |
| `Result<Success, Failure>` | `Equatable` | `Success: Equatable, Failure: Equatable` |
| `Dictionary<Key, Value>` | `Codable` | `Key: Codable, Value: Codable` |

### 6.3 自定义条件遵循

```swift
struct Pair<T> {
    let first: T
    let second: T
}

extension Pair: Equatable where T: Equatable {
    static func == (lhs: Pair<T>, rhs: Pair<T>) -> Bool {
        lhs.first == rhs.first && lhs.second == rhs.second
    }
}

extension Pair: Comparable where T: Comparable {
    static func < (lhs: Pair<T>, rhs: Pair<T>) -> Bool {
        if lhs.first != rhs.first {
            return lhs.first < rhs.first
        }
        return lhs.second < rhs.second
    }
}

extension Pair: Codable where T: Codable {}

let p1 = Pair(first: 1, second: 2)
let p2 = Pair(first: 1, second: 2)
print(p1 == p2) // true

let p3 = Pair(first: "a", second: "b")
let p4 = Pair(first: "b", second: "a")
print(p3 < p4) // true
```

### 6.4 条件遵循的传递性

条件遵循具有传递性：如果 `A` 条件遵循 `P`，而 `B` 的条件遵循依赖 `A: P`，那么条件会自动传递：

```swift
struct Box<T> {
    var value: T
}

extension Box: Equatable where T: Equatable {
    static func == (lhs: Box<T>, rhs: Box<T>) -> Bool {
        lhs.value == rhs.value
    }
}

// Array<Box<Int>> 自动遵循 Equatable
// 因为 Int: Equatable → Box<Int>: Equatable → [Box<Int>]: Equatable
let boxes1 = [Box(value: 1), Box(value: 2)]
let boxes2 = [Box(value: 1), Box(value: 2)]
print(boxes1 == boxes2) // true
```

> ⚠️ 条件遵循不能重复声明。如果已经在某个模块中为类型声明了条件遵循，其他模块不能再为同一类型和协议声明条件遵循，否则会产生"重复遵循"冲突。

---

## 7. 类型擦除

### 7.1 为什么需要类型擦除

泛型类型会"泄漏"具体信息。当你想隐藏这些信息时，就需要类型擦除：

```swift
// 问题：每个返回类型的泛型参数不同，无法统一存储
func makeIntPublisher() -> some Publisher<Int, Never> { ... }
func makeStringPublisher() -> some Publisher<String, Never> { ... }

// 想把它们放在同一个数组里？不行！泛型参数不同
// let publishers: [some Publisher] = [...] // ❌ 编译错误
```

> 💡 生活类比：类型擦除就像把信封上的寄件人地址涂掉——信还在，但你不知道是谁寄的。内容（值）保留，身份（具体类型）隐藏。

### 7.2 标准库中的类型擦除器

| 类型擦除器 | 擦除的协议 | 用途 |
|-----------|-----------|------|
| `AnyView` | `View` | 隐藏具体 View 类型 |
| `AnyPublisher` | `Publisher` | 隐藏具体 Publisher 类型 |
| `AnySequence` | `Sequence` | 隐藏具体 Sequence 类型 |
| `AnyCollection` | `Collection` | 隐藏具体 Collection 类型 |
| `AnyHashable` | `Hashable` | 隐藏具体 Hashable 类型 |

### 7.3 AnyView 的使用与代价

```swift
import SwiftUI

// 使用 AnyView 擦除类型
func makeRow(for item: Item) -> AnyView {
    switch item.type {
    case .text:
        return AnyView(TextRow(item: item))
    case .image:
        return AnyView(ImageRow(item: item))
    case .video:
        return AnyView(VideoRow(item: item))
    }
}
```

> ⚠️ `AnyView` 有性能代价：SwiftUI 无法在编译期推断 View 的具体类型，导致 diff 算法退化为全量比较。在列表中大量使用 `AnyView` 会显著影响性能。优先使用 `@ViewBuilder` 或 `Group` 代替。

```swift
// ✅ 更好的方式：使用 @ViewBuilder
@ViewBuilder
func makeRow(for item: Item) -> some View {
    switch item.type {
    case .text:
        TextRow(item: item)
    case .image:
        ImageRow(item: item)
    case .video:
        VideoRow(item: item)
    }
}
```

### 7.4 手写类型擦除器

理解类型擦除的原理，最好的方式是手写一个。以下是一个简化版 `AnySequence`：

```swift
struct AnySequence<Element>: Sequence {
    private struct _Iterator: IteratorProtocol {
        var _next: () -> Element?

        mutating func next() -> Element? {
            _next()
        }
    }

    private let _makeIterator: () -> _Iterator

    init<S: Sequence>(_ sequence: S) where S.Element == Element {
        var iterator = sequence.makeIterator()
        _makeIterator = {
            _Iterator { iterator.next() }
        }
    }

    func makeIterator() -> _Iterator {
        _makeIterator()
    }
}

// 使用
let numbers = AnySequence([1, 2, 3])
let strings = AnySequence(["a", "b", "c"])

// 现在可以统一存储了
let sequences: [AnySequence<Any>] = [
    AnySequence([1, 2, 3] as [Any]),
    AnySequence(["a", "b", "c"] as [Any])
]
```

手写类型擦除器的核心思路：

| 步骤 | 说明 |
|------|------|
| 1. 定义泛型包装结构体 | `AnySequence<Element>` |
| 2. 在内部定义私有迭代器 | 用闭包存储 `next()` 的实现 |
| 3. 用闭包捕获原始实现 | `init<S: Sequence>` 中把方法调用转发给原始类型 |
| 4. 暴露协议要求的 API | `makeIterator()` 返回内部迭代器 |

---

## 8. 泛型在 SwiftUI 中的应用

### 8.1 View 协议的泛型设计

SwiftUI 的 `View` 协议是泛型设计的典范：

```swift
protocol View {
    associatedtype Body: View
    @ViewBuilder var body: Self.Body { get }
}
```

关键设计决策：

| 设计 | 原因 |
|------|------|
| `associatedtype Body` | 每个 View 的 body 类型不同，用关联类型保留具体类型信息 |
| `Body: View` | 递归约束：body 本身也必须是 View |
| `some View` 返回 | 隐藏具体类型，保留静态派发性能 |

```swift
struct ContentView: View {
    var body: some View {
        VStack {
            Text("Hello")
                .font(.title)
            Button("Tap me") {
                print("tapped")
            }
        }
    }
}
// body 的实际类型是 VStack<TupleView<(Text, Button<Text>)>>
// 但调用方只需要知道它是 some View
```

### 8.2 PreferenceKey 的泛型机制

PreferenceKey 是 SwiftUI 中子视图向父视图传递数据的机制，它利用泛型确保类型安全：

```swift
struct CustomTitlePreferenceKey: PreferenceKey {
    static var defaultValue: String = ""

    static func reduce(value: inout String, nextValue: () -> String) {
        value = nextValue()
    }
}

struct TitleModifier: ViewModifier {
    let title: String

    func body(content: Content) -> some View {
        content
            .preference(key: CustomTitlePreferenceKey.self, value: title)
    }
}

struct ParentView: View {
    @State private var title = ""

    var body: some View {
        VStack {
            ChildView()
                .onPreferenceChange(CustomTitlePreferenceKey.self) { value in
                    title = value
                }
            Text("Current: \(title)")
        }
    }
}
```

PreferenceKey 的泛型设计保证了：
- `preference(key:value:)` 的 `value` 类型与 Key 的 `Value` 类型一致
- `onPreferenceChange` 的回调参数类型自动推断
- 不同 PreferenceKey 的值不会混淆

### 8.3 Environment 的泛型机制

Environment 使用 `EnvironmentKey` 协议和泛型来提供类型安全的环境值注入：

```swift
private struct ThemeColorKey: EnvironmentKey {
    static let defaultValue: Color = .blue
}

extension EnvironmentValues {
    var themeColor: Color {
        get { self[ThemeColorKey.self] }
        set { self[ThemeColorKey.self] = newValue }
    }
}

struct ThemedView: View {
    @Environment(\.themeColor) var themeColor

    var body: some View {
        RoundedRectangle(cornerRadius: 12)
            .fill(themeColor)
            .frame(width: 100, height: 100)
    }
}

struct RootView: View {
    var body: some View {
        VStack {
            ThemedView()
                .environment(\.themeColor, .red)
            ThemedView()
        }
    }
}
```

Environment 的泛型工作原理：

```
EnvironmentValues
  └── subscript<K: EnvironmentKey>(key: K.Type) -> K.Value
        ├── get: 从环境字典中取出 K.Value
        └── set: 写入 K.Value 到环境字典
```

| 机制 | 泛型角色 |
|------|---------|
| `EnvironmentKey` | `associatedtype Value` 定义环境值类型 |
| `EnvironmentValues subscript` | 泛型下标 `<K: EnvironmentKey>` 保证类型安全 |
| `@Environment` | 泛型属性包装器，自动推断 Value 类型 |
| `.environment(_:)` | 泛型修饰符，注入特定 Key 的 Value |

### 8.4 综合实战：泛型配置系统

将本章知识综合运用，构建一个类型安全的配置系统：

```swift
protocol ConfigurationKey {
    associatedtype Value: Sendable
    static var defaultValue: Value { get }
}

struct Configuration {
    private var storage: [ObjectIdentifier: Any] = [:]

    subscript<K: ConfigurationKey>(key: K.Type) -> K.Value {
        get {
            storage[ObjectIdentifier(key)] as? K.Value ?? K.defaultValue
        }
        set {
            storage[ObjectIdentifier(key)] = newValue
        }
    }
}

struct APITimeoutKey: ConfigurationKey {
    static let defaultValue: TimeInterval = 30
}

struct MaxRetryKey: ConfigurationKey {
    static let defaultValue: Int = 3
}

struct BaseURLKey: ConfigurationKey {
    static let defaultValue: String = "https://api.example.com"
}

var config = Configuration()
print(config[APITimeoutKey.self])     // 30.0
config[APITimeoutKey.self] = 60
config[MaxRetryKey.self] = 5
config[BaseURLKey.self] = "https://prod.example.com"
```

这个设计综合运用了：
- **关联类型**：`ConfigurationKey.Value` 定义每个配置项的值类型
- **泛型下标**：`subscript<K: ConfigurationKey>` 保证类型安全存取
- **条件遵循**：如果需要，可以让 `Configuration` 条件遵循 `Equatable`
- **类型擦除**：内部用 `[ObjectIdentifier: Any]` 存储不同类型的值

---

## 本章小结

| 主题 | 核心要点 | 关键语法 |
|------|---------|---------|
| 泛型深入 | where 子句、多约束、泛型下标提供灵活的类型约束 | `where T: Equatable & Hashable` |
| 关联类型 | 协议中的类型占位符，由遵循类型具体化 | `associatedtype Item` |
| 主要关联类型 | Swift 5.7+ 简化泛型协议调用 | `protocol Container<Element>` |
| 不透明类型 some | 实现方决定具体类型，静态派发零开销 | `some View` |
| 存在类型 any | 运行时多态，支持异构集合，有动态派发开销 | `any Drawable` |
| 条件遵循 | 满足条件时自动遵循协议，具有传递性 | `extension Array: Equatable where Element: Equatable` |
| 类型擦除 | 隐藏具体类型信息，统一接口 | `AnyView` / `AnyPublisher` |
| SwiftUI 泛型 | View/PreferenceKey/Environment 的类型安全设计 | `associatedtype Body: View` |

> 💡 **学习建议**：泛型和类型系统是 Swift 最强大也最复杂的特性。建议从 `some View` 和 `any Protocol` 这两个最常用的语法入手，逐步理解背后的类型系统设计。遇到编译器报错时，仔细阅读错误信息——Swift 的泛型错误提示在近年版本中已经大幅改善，通常能直接告诉你缺少什么约束。

← [-ARC 与内存管理](./28-ARC与内存管理.md) | [-SwiftUI 初体验：第一个项目](../04-SwiftUI入门/30-SwiftUI初体验.md) →
