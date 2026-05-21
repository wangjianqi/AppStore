# 28-ARC 与内存管理

> 🎯 **本章目标**：
> - 理解栈与堆的内存模型差异，掌握值类型与引用类型的内存分配方式
> - 深入理解 ARC 引用计数机制，明白 retain/release 的自动插入时机
> - 识别并修复三种典型循环引用场景（类互持、闭包捕获、Delegate 模式）
> - 熟练使用 weak 与 unowned，掌握其区别与安全风险
> - 掌握闭包捕获列表与逃逸闭包的内存管理技巧
> - 能使用 Instruments 进行内存泄漏检测与性能分析
> - 理解 SwiftUI 视图生命周期中的内存管理要点
> - 建立完整的内存管理最佳实践 checklist

---

## 1. 内存管理基础

### 1.1 栈 vs 堆

程序运行时的内存主要分为**栈（Stack）**和**堆（Heap）**两个区域。你可以把它们想象成两种不同的储物方式：

- **栈**就像办公桌上的文件架——空间有限，但存取极快，用完自动收回，不需要你操心
- **堆**就像一个大型仓库——空间充裕，但存取较慢，需要手动管理（或由 ARC 代劳）何时放进去、何时取出来

| 特性 | 栈（Stack） | 堆（Heap） |
|------|-------------|------------|
| 管理方式 | 自动（编译器） | ARC 自动 / 手动 |
| 分配速度 | 极快（移动指针） | 较慢（搜索空闲块） |
| 释放方式 | 作用域结束自动弹出 | 引用计数归零时释放 |
| 空间大小 | 较小（通常 1~8 MB） | 较大（受系统可用内存限制） |
| 线程安全 | 每线程独立栈 | 多线程共享，需同步 |
| 存储内容 | 局部变量、函数参数 | class 实例、闭包 |

### 1.2 值类型 vs 引用类型的内存模型

Swift 中 `struct`、`enum`、`tuple` 是**值类型**，`class`、`closure` 是**引用类型**。它们的内存模型截然不同：

```swift
struct Point {
    var x: Double
    var y: Double
}

class Person {
    var name: String
    init(name: String) { self.name = name }
}

var p1 = Point(x: 1, y: 2)
var p2 = p1
p2.x = 99
print(p1.x)

let person1 = Person(name: "小明")
let person2 = person1
person2.name = "小红"
print(person1.name)
```

> 💡 **类比**：值类型像**复印文件**——你拿到的是副本，修改副本不影响原件；引用类型像**共享文档链接**——大家看的是同一份文档，谁改了所有人都看得到。

| 对比项 | 值类型（struct/enum） | 引用类型（class） |
|--------|----------------------|-------------------|
| 赋值行为 | 拷贝副本 | 共享同一实例 |
| 存储位置 | 通常在栈上 | 在堆上 |
| 内存管理 | 无需 ARC | ARC 引用计数 |
| 修改影响 | 仅影响当前副本 | 影响所有引用 |
| 线程安全 | 天然安全（各自独立） | 需要同步机制 |

### 1.3 Swift 内存分配图解

```
栈（Stack）                          堆（Heap）
┌─────────────────┐               ┌─────────────────────┐
│ p1: Point       │               │                     │
│   x = 1.0       │               │  Person 实例         │
│   y = 2.0       │               │  ┌───────────────┐  │
├─────────────────┤               │  │ refCount = 2  │  │
│ p2: Point       │               │  │ name = "小红"  │  │
│   x = 99.0      │               │  └───────────────┘  │
│   y = 2.0       │               │         ▲           │
├─────────────────┤               │         │           │
│ person1: ───────────────────────┼─────────┘           │
│ (指针 0x1000)   │               │         ▲           │
├─────────────────┤               │         │           │
│ person2: ───────────────────────┼─────────┘           │
│ (指针 0x1000)   │               │                     │
└─────────────────┘               └─────────────────────┘
```

> ⚠️ **注意**：值类型并非永远在栈上。当值类型过大、被闭包捕获、或作为 class 的属性时，也可能在堆上分配。Swift 编译器会通过"写时复制（Copy-on-Write）"优化大值类型的拷贝开销。

---

## 2. ARC 工作原理

### 2.1 引用计数机制

ARC（Automatic Reference Counting，自动引用计数）是 Swift 管理类实例内存的方式。你可以把它想象成**共享办公室的钥匙管理**：

- 每来一个人（新引用），钥匙数 +1（retain）
- 每走一个人（引用离开作用域），钥匙数 -1（release）
- 钥匙数归零，办公室关闭（内存释放）

```swift
class Dog {
    let name: String
    init(name: String) {
        self.name = name
        print("🐕 \(name) 被创建")
    }
    deinit {
        print("🐕 \(name) 被释放")
    }
}

func createDog() {
    let dog = Dog(name: "旺财")
    print("引用计数 = 1")
}

createDog()
```

> 💡 **关键**：ARC 只管理**引用类型**（class、闭包）。值类型不需要 ARC，因为它们通过拷贝传递，不存在共享所有权的问题。

### 2.2 强引用 / 弱引用 / 无主引用

Swift 提供三种引用方式，决定了引用是否影响对象的生命周期：

| 引用类型 | 关键字 | 是否增加引用计数 | 对象释放后 | 生命周期要求 |
|----------|--------|------------------|------------|-------------|
| 强引用 | （默认） | ✅ +1 | 不会发生 | 无 |
| 弱引用 | `weak` | ❌ 不增加 | 自动置为 `nil` | 被引用者可更短 |
| 无主引用 | `unowned` | ❌ 不增加 | 访问会崩溃 | 被引用者 ≥ 引用者 |

```swift
class Owner {
    var name: String
    weak var pet: Pet?
    init(name: String) { self.name = name }
    deinit { print("Owner \(name) 释放") }
}

class Pet {
    var name: String
    unowned var owner: Owner
    init(name: String, owner: Owner) {
        self.name = name
        self.owner = owner
    }
    deinit { print("Pet \(name) 释放") }
}
```

### 2.3 ARC 自动插入 retain/release 的时机

编译器会在以下位置自动插入 `retain` 和 `release` 调用：

```swift
func example() {
    let obj = MyClass()
    // 编译器插入: retain(obj)  —— 赋值给局部变量

    doSomething(obj)
    // 编译器插入: retain(obj)  —— 传入函数参数
    // 函数返回后: release(obj) —— 参数生命周期结束

    globalRef = obj
    // 编译器插入: retain(obj)  —— 赋值给全局/类属性

}
// 编译器插入: release(obj)  —— 局部变量离开作用域
// 如果此时引用计数归零 → 调用 deinit → 释放内存
```

> 💡 **提示**：你不需要手动写 retain/release，编译器会在编译期自动分析并插入。这也是 ARC 与手动引用计数（MRC）的核心区别——ARC 是编译期技术，不是运行时垃圾回收。

---

## 3. 循环引用

### 3.1 什么是循环引用

循环引用就像两个人互相拽着对方的手，谁都不肯先松开——结果谁也走不了。

```
A →→→ B
↑     ↓
←←←←←┘
```

当 A 强引用 B，B 又强引用 A 时，两者的引用计数永远无法归零，内存永远不会被释放，这就是**内存泄漏**。

### 3.2 场景一：类互持

❌ **错误写法**：

```swift
class Apartment {
    var tenant: Person?
    deinit { print("Apartment 释放") }
}

class Person {
    var apartment: Apartment?
    deinit { print("Person 释放") }
}

var john: Person? = Person()
var unit4A: Apartment? = Apartment()

john?.apartment = unit4A
unit4A?.tenant = john

john = nil
unit4A = nil
```

✅ **修复写法**——将其中一方改为 `weak`：

```swift
class Apartment {
    weak var tenant: Person?
    deinit { print("Apartment 释放") }
}

class Person {
    var apartment: Apartment?
    deinit { print("Person 释放") }
}

var john: Person? = Person()
var unit4A: Apartment? = Apartment()

john?.apartment = unit4A
unit4A?.tenant = john

john = nil
unit4A = nil
```

> 💡 **类比**：人（Person）和公寓（Apartment）的关系中，公寓"拥有"租客是弱关系——租客可以随时搬走，公寓不应该因为租客在就不拆除。所以 `tenant` 用 `weak`。

### 3.3 场景二：闭包捕获

❌ **错误写法**：

```swift
class DataLoader {
    var data: String = ""
    var onComplete: (() -> Void)?

    func load() {
        onComplete = {
            self.data = "已加载"
            print("数据：\(self.data)")
        }
    }

    deinit { print("DataLoader 释放") }
}

var loader: DataLoader? = DataLoader()
loader?.load()
loader = nil
```

✅ **修复写法**——使用闭包捕获列表 `[weak self]`：

```swift
class DataLoader {
    var data: String = ""
    var onComplete: (() -> Void)?

    func load() {
        onComplete = { [weak self] in
            guard let self = self else { return }
            self.data = "已加载"
            print("数据：\(self.data)")
        }
    }

    deinit { print("DataLoader 释放") }
}

var loader: DataLoader? = DataLoader()
loader?.load()
loader = nil
```

### 3.4 场景三：Delegate 模式

❌ **错误写法**：

```swift
protocol ListViewControllerDelegate: AnyObject {
    func didSelectItem(_ item: String)
}

class ListViewController: UIViewController {
    var delegate: ListViewControllerDelegate?
    deinit { print("ListViewController 释放") }
}

class DetailViewController: UIViewController {
    var listVC: ListViewController?
    deinit { print("DetailViewController 释放") }
}

extension DetailViewController: ListViewControllerDelegate {
    func didSelectItem(_ item: String) {
        print("选中：\(item)")
    }
}
```

✅ **修复写法**——Delegate 声明为 `weak`：

```swift
protocol ListViewControllerDelegate: AnyObject {
    func didSelectItem(_ item: String)
}

class ListViewController: UIViewController {
    weak var delegate: ListViewControllerDelegate?
    deinit { print("ListViewController 释放") }
}

class DetailViewController: UIViewController {
    var listVC: ListViewController?
    deinit { print("DetailViewController 释放") }
}

extension DetailViewController: ListViewControllerDelegate {
    func didSelectItem(_ item: String) {
        print("选中：\(item)")
    }
}
```

> ⚠️ **注意**：`weak` 只能用于 `AnyObject` 协议（即 class-only 协议），因为值类型不存在引用计数。Swift 4+ 推荐用 `AnyObject` 而非 `class` 来标记类专属协议。

---

## 4. weak 与 unowned 详解

### 4.1 区别对比

| 特性 | `weak` | `unowned` |
|------|--------|-----------|
| 声明方式 | `weak var ref: T?` | `unowned var ref: T` |
| 可选性 | 必须 Optional | 必须 non-Optional |
| 对象释放后 | 自动置 `nil` | 变成悬垂指针 |
| 访问释放后的对象 | 安全（返回 `nil`） | **崩溃** |
| 性能 | 略低（需检查 nil） | 略高（无额外检查） |
| 适用场景 | 被引用者可能先释放 | 被引用者一定同生共死 |

### 4.2 使用场景选择

```swift
class Customer {
    var name: String
    var card: CreditCard?
    init(name: String) { self.name = name }
    deinit { print("Customer 释放") }
}

class CreditCard {
    let number: String
    unowned let owner: Customer
    init(number: String, owner: Customer) {
        self.number = number
        self.owner = owner
    }
    deinit { print("CreditCard 释放") }
}

let customer = Customer(name: "张三")
customer.card = CreditCard(number: "6222****", owner: customer)
```

> 💡 **类比**：信用卡（CreditCard）不可能比持卡人（Customer）活得更久——人没了卡自然失效。所以用 `unowned` 是安全的。而公寓的租客可能比公寓先"消失"，所以用 `weak`。

### 4.3 unowned 的安全风险

```swift
class ViewModel {
    unowned var service: Service

    init(service: Service) {
        self.service = service
    }

    func doWork() {
        service.execute()
    }
}

class Service {
    var viewModel: ViewModel?

    func execute() {
        print("执行任务")
    }

    deinit { print("Service 释放") }
}

var service: Service? = Service()
let vm = ViewModel(service: service!)
service = nil
vm.doWork()
```

> ⚠️ **警告**：`unowned` 是一把双刃剑——当你**确定**被引用对象的生命周期 ≥ 引用者时，用 `unowned` 可以避免 Optional 解包的麻烦；但如果判断失误，访问已释放的对象会导致**确定性崩溃**。在不确定时，永远优先选择 `weak`。

### 4.4 Optional vs non-Optional 选择决策

```
被引用对象可能先释放吗？
    │
    ├── 是 → weak（Optional，安全降级为 nil）
    │
    └── 否 → 被引用对象和引用者生命周期完全一致吗？
                │
                ├── 是 → unowned（non-Optional，性能略优）
                │
                └── 不确定 → weak（安全第一）
```

---

## 5. 闭包与内存管理

### 5.1 闭包捕获列表 `[weak self]`

闭包默认会**强捕获**引用的变量，包括 `self`。捕获列表让你可以显式控制捕获方式：

```swift
class NetworkManager {
    var result: String = ""

    func fetchData() {
        URLSession.shared.dataTask(with: url) { [weak self] data, response, error in
            guard let self = self else { return }
            self.result = "完成"
        }.resume()
    }

    func fetchDataWithBoth() {
        URLSession.shared.dataTask(with: url) { [weak self, weak delegate = self.delegate] in
            guard let self = self else { return }
            self.result = "完成"
            delegate?.didFinish()
        }.resume()
    }
}
```

> 💡 **提示**：`guard let self = self else { return }` 是 Swift 中处理 `weak self` 的惯用模式，在闭包开头安全解包，后续代码无需反复解包。

### 5.2 逃逸闭包 @escaping 的内存影响

```swift
class CacheManager {
    var cache: [String: String] = [:]

    func compute(key: String, completion: @escaping (String) -> Void) {
        DispatchQueue.global().async {
            let result = "cached_\(key)"
            DispatchQueue.main.async {
                completion(result)
            }
        }
    }
}
```

| 闭包类型 | 关键字 | 执行时机 | 捕获 self | 内存风险 |
|----------|--------|----------|-----------|---------|
| 非逃逸闭包 | `@noescape`（默认） | 函数内同步执行 | 安全，函数返回前释放 | 低 |
| 逃逸闭包 | `@escaping` | 函数返回后异步执行 | 强持有 self，需 `[weak self]` | 高 |

> ⚠️ **警告**：`@escaping` 闭包会在函数返回后继续存在，因此它**必须**显式使用 `self`（编译器强制要求），这其实是一个安全提醒——提醒你思考是否需要 `[weak self]`。

### 5.3 lazy 闭包中的 self

```swift
class Document {
    var title: String
    lazy var summary: String = {
        return "摘要：\(self.title)"
    }()

    init(title: String) { self.title = title }
    deinit { print("Document 释放") }
}
```

`lazy` 属性的初始化闭包不会造成循环引用，因为：

1. 闭包只在首次访问时执行一次
2. 闭包执行后，结果被存储为普通属性值，闭包本身被释放
3. 闭包不会作为属性值被长期持有

> ⚠️ **注意**：但如果 `lazy` 闭包内部又引用了自身的其他 `lazy` 属性，且形成环路，仍然可能产生循环引用。此外，`lazy` 属性是线程不安全的。

---

## 6. Instruments 内存分析实战

### 6.1 Allocations 工具

**实操步骤**：

1. Xcode → Product → Profile（⌘I）→ 选择 **Allocations**
2. 运行 App，执行待分析的操作
3. 观察内存增长曲线——正常情况应呈锯齿状（分配后释放）
4. 关注 **Persistent Bytes**（未释放的内存）持续增长的区域
5. 点击增长点 → 查看调用栈 → 定位代码位置

| 指标 | 含义 | 健康表现 |
|------|------|---------|
| All Heap Allocations | 堆上分配总量 | 稳定，不持续增长 |
| Persistent Objects | 未释放的对象数 | 操作结束后回落 |
| Transient Objects | 已释放的对象数 | 数值较大是正常的 |
| Overall Bytes | 总分配量（含已释放） | 参考值，非问题指标 |

### 6.2 Leaks 工具

**实操步骤**：

1. Xcode → Product → Profile → 选择 **Leaks**
2. 运行 App，反复进入/退出可疑页面
3. Leaks 工具会在红色区域显示检测到的泄漏
4. 点击泄漏对象 → 查看 **Extended Detail** 中的循环引用路径
5. 根据引用链定位代码中的强引用环

### 6.3 Memory Graph Debugger

**实操步骤**：

1. 运行 App 后，点击 Xcode 底部调试栏的 🐉 图标（或 Debug → Memory Graph）
2. 左侧导航器切换到 **Debug Navigator**，查看内存概览
3. 筛选显示 **仅泄漏对象**（Filter → Show Only Leaked）
4. 点击对象查看其引用关系图
5. 箭头表示引用方向，红色 ⚠️ 标记循环引用

> 💡 **提示**：Memory Graph 是最直观的内存分析工具，能以图形方式展示对象间的引用关系，快速定位循环引用链。建议在每次页面退出后都检查一次。

### 6.4 常见内存泄漏模式识别

| 泄漏模式 | 特征表现 | 典型原因 |
|----------|---------|---------|
| 阶梯式增长 | 每次操作后内存不回落 | 循环引用 / 未取消订阅 |
| 线性增长 | 内存随时间匀速上升 | Timer 未停止 / 缓存无限增长 |
| 突然飙升 | 某操作后内存大幅跳升 | 大图片未压缩 / 数据未分页 |
| 间歇性泄漏 | 偶发出现泄漏对象 | 多线程竞态 / 条件性强引用 |

---

## 7. SwiftUI 中的内存管理

### 7.1 @StateObject vs @ObservedObject 生命周期

```swift
class TimerModel: ObservableObject {
    @Published var count = 0
    private var timer: Timer?

    init() {
        timer = Timer.scheduledTimer(withTimeInterval: 1, repeats: true) { [weak self] _ in
            self?.count += 1
        }
    }

    deinit {
        timer?.invalidate()
        print("TimerModel 释放")
    }
}

struct ParentView: View {
    @StateObject var model = TimerModel()

    var body: some View {
        VStack {
            Text("计数：\(model.count)")
            ChildView(model: model)
        }
    }
}

struct ChildView: View {
    @ObservedObject var model: TimerModel

    var body: some View {
        Text("子视图：\(model.count)")
    }
}
```

| 属性包装器 | 创建者 | 持有者 | 生命周期 | 适用场景 |
|-----------|--------|--------|---------|---------|
| `@StateObject` | 当前 View | 当前 View | View 重建时保持 | **拥有**该对象 |
| `@ObservedObject` | 外部传入 | 外部 | 随外部生命周期 | **接收**该对象 |

> ⚠️ **警告**：在 View 中用 `@ObservedObject` 初始化对象是常见错误——View 是 struct，每次重建都会重新创建 ObservableObject，导致状态丢失。**拥有者用 `@StateObject`，接收者用 `@ObservedObject`**。

### 7.2 View struct 的值语义优势

SwiftUI 的 View 是 `struct`（值类型），这意味着：

```swift
struct ContentView: View {
    var count: Int = 0

    var body: some View {
        Text("\(count)")
    }
}
```

- View 本身**不存在循环引用风险**——struct 没有引用计数
- View 的 diff 算法依赖值比较，struct 的值语义天然支持
- 状态由属性包装器（`@State`、`@StateObject`）统一管理，而非散落在 View 实例中

> 💡 **提示**：这也是 SwiftUI 比 UIKit 在内存管理上更安全的原因之一——View struct 不参与 ARC，不会产生循环引用。

### 7.3 EnvironmentObject 持有链

```swift
@main
struct MyApp: App {
    @StateObject var appModel = AppModel()

    var body: some Scene {
        WindowGroup {
            ContentView()
                .environmentObject(appModel)
        }
    }
}

struct DeepView: View {
    @EnvironmentObject var appModel: AppModel

    var body: some View {
        Text(appModel.userName)
    }
}
```

`EnvironmentObject` 的持有链：

```
App (@StateObject) → environmentObject() 修饰 → View 层级读取
```

- `@StateObject` 在 `App` 层持有，生命周期 = App 生命周期
- 子 View 通过 `@EnvironmentObject` 读取，**不持有**
- 不会产生循环引用，因为 `@EnvironmentObject` 不增加引用计数

> ⚠️ **注意**：如果 `@EnvironmentObject` 在 View 层级中找不到对应对象，App 会直接崩溃（不是可选的）。确保在正确的层级注入。

### 7.4 Timer / NotificationCenter 订阅泄漏

```swift
class NotificationObserver: ObservableObject {
    @Published var message: String = ""
    private var cancellables = Set<AnyCancellable>()

    init() {
        NotificationCenter.default.publisher(for: .init("MyNotification"))
            .sink { [weak self] notification in
                self?.message = notification.userInfo?["text"] as? String ?? ""
            }
            .store(in: &cancellables)
    }

    deinit {
        print("NotificationObserver 释放")
    }
}
```

| 订阅方式 | 泄漏风险 | 正确做法 |
|----------|---------|---------|
| `NotificationCenter.default.addObserver` | 高（需手动 removeObserver） | 在 `deinit` 中移除，或用 Combine |
| Combine `sink` | 中（cancellable 需存储） | 存入 `Set<AnyCancellable>`，deinit 自动取消 |
| Combine `sink` + `[weak self]` | 低 | 同时使用 weak self + store |
| `Timer.scheduledTimer` | 高（需手动 invalidate） | 在 `deinit` 中 invalidate |

> ⚠️ **警告**：Combine 的 `sink` 闭包会强持有 `self`，必须使用 `[weak self]` 避免循环引用。即使 `cancellable` 会在 `deinit` 时取消，但如果 `sink` 强持有 `self`，`deinit` 永远不会被调用——又是一个循环引用！

---

## 8. 内存管理最佳实践 Checklist

### 8.1 十条核心规则

| # | 规则 | 说明 |
|---|------|------|
| 1 | **Delegate 用 weak** | Delegate 模式中，delegate 属性始终声明为 `weak` |
| 2 | **闭包捕获用 weak self** | 逃逸闭包和存储型闭包中，使用 `[weak self]` |
| 3 | **guard let self 解包** | 使用 `guard let self = self else { return }` 惯用模式 |
| 4 | **unowned 慎用** | 仅在生命周期完全一致时使用，否则用 weak |
| 5 | **协议加 AnyObject** | 需要弱引用的协议声明为 `protocol X: AnyObject` |
| 6 | **Timer 必须 invalidate** | 页面退出时停止所有 Timer，否则 target 被 Timer 强持有 |
| 7 | **NotificationCenter 必须移除** | 在 deinit 或 viewWillDisappear 中移除观察者 |
| 8 | **StateObject 拥有，ObservedObject 接收** | 不要在 View 中用 ObservedObject 初始化 |
| 9 | **定期 Memory Graph 检查** | 每完成一个功能模块，用 Memory Graph 检查泄漏 |
| 10 | **Instruments 定期巡检** | 发版前用 Allocations + Leaks 做完整内存巡检 |

### 8.2 常见面试题

**Q1：ARC 是垃圾回收吗？**

不是。ARC 是**编译期**技术，编译器在编译时自动插入 retain/release 代码；垃圾回收（GC）是**运行时**技术，通过后台线程定期扫描并回收不可达对象。ARC 没有运行时开销，也不会出现"停顿"现象。

**Q2：weak 和 unowned 的本质区别是什么？**

`weak` 引用的对象释放后，引用自动变为 `nil`，因此必须为 Optional；`unowned` 引用的对象释放后，引用变成悬垂指针，访问会崩溃，因此不能为 Optional。底层实现上，`weak` 需要维护一张弱引用表来追踪并置 nil，`unowned` 不需要。

**Q3：struct 会有循环引用吗？**

不会。struct 是值类型，没有引用计数，赋值时是拷贝。只有引用类型（class、closure）才可能产生循环引用。

**Q4：@escaping 闭包为什么更容易泄漏？**

非逃逸闭包在函数返回前一定执行完毕，不会超出函数作用域，因此编译器可以保证它不会造成循环引用。逃逸闭包可能被存储或异步执行，生命周期超出函数范围，如果它强持有 self，就会阻止 self 释放。

**Q5：SwiftUI View 为什么不需要担心循环引用？**

SwiftUI 的 View 是 struct（值类型），没有引用计数，不存在循环引用的可能。状态管理通过 `@State`、`@StateObject` 等属性包装器完成，它们由 SwiftUI 框架统一管理生命周期。

---

## 本章小结

| 主题 | 核心要点 |
|------|---------|
| 内存基础 | 栈自动管理、堆需 ARC；值类型拷贝、引用类型共享 |
| ARC 原理 | 引用计数 +1/-1，归零释放；编译期自动插入 retain/release |
| 循环引用 | 类互持→weak 一方；闭包捕获→[weak self]；Delegate→weak delegate |
| weak vs unowned | weak 安全（变 nil），unowned 高效（可能崩溃）；不确定就用 weak |
| 闭包内存 | 逃逸闭包需 [weak self]；lazy 闭包通常安全；Combine sink 同样需要 weak |
| Instruments | Allocations 看增长、Leaks 找泄漏、Memory Graph 看引用链 |
| SwiftUI | View struct 无循环引用；@StateObject 拥有、@ObservedObject 接收；订阅必须 weak + 取消 |
| 最佳实践 | Delegate weak、闭包 weak self、Timer invalidate、定期 Memory Graph 检查 |
