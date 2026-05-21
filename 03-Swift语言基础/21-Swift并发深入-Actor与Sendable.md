# 21-Swift 并发深入：Actor、Sendable 与结构化并发

## 本章目标

- 理解为什么需要深入掌握并发：数据竞争的危害与 Swift 6 的严格并发趋势
- 掌握 Actor 模型的核心概念：actor isolation、方法调用规则、与 class 的区别
- 能够编写实战 Actor：计数器、数据缓存、避免数据竞争的完整示例
- 理解 Sendable 协议：自动遵循与不遵循的类型、@Sendable 闭包、nonisolated(nonsending)
- 掌握结构化并发：Task 生命周期管理、TaskGroup 与 withTaskGroup
- 区分非结构化并发：Task.detached 与 Task.init 的使用场景
- 学会使用 AsyncSequence 与 AsyncStream 构建异步数据流
- 掌握并发最佳实践：避免过度创建 Task、MainActor 使用原则、优先级与取消处理

---

## 1. 为什么需要深入理解并发

### 1.1 数据竞争——并发编程的头号敌人

想象一个厨房里，两位厨师同时往同一口锅里加盐：

| | 厨师 A | 厨师 B | 结果 |
|---|---|---|---|
| **时刻 1** | 看到盐罐有 10g | — | — |
| **时刻 2** | — | 看到盐罐有 10g | — |
| **时刻 3** | 取走 5g，剩 5g | — | — |
| **时刻 4** | — | 取走 5g，剩 5g | ❌ 实际剩 0g，但两人都以为剩 5g |

这就是**数据竞争（Data Race）**：多个线程同时访问同一块可变状态，且至少一个线程在写入，导致结果不可预测。

```swift
class BankAccount {
    var balance: Double = 1000

    func withdraw(_ amount: Double) {
        let current = balance      // 线程 A 读到 1000
        Thread.sleep(forTimeInterval: 0.01) // 模拟延迟
        balance = current - amount  // 线程 A 写回 900
    }
}

let account = BankAccount()
// 两个线程同时取 100，期望余额 800，实际可能变成 900
```

> ⚠️ 数据竞争是**最难以调试的 Bug 之一**——它在开发时可能从不出现，却在用户设备上频繁触发。Thread Sanitizer（TSan）可以帮助检测，但治标不治本。

### 1.2 传统线程安全手段的局限

| 手段 | 问题 |
|---|---|
| **锁（NSLock / os_unfair_lock）** | 忘记加锁、锁粒度难控制、死锁风险 |
| **串行队列（DispatchQueue）** | 嵌套回调地狱、难以追踪执行顺序 |
| **@synchronized** | 性能差、粒度粗、Swift 中不推荐 |

### 1.3 Swift 6 的严格并发趋势

Swift 5.5 引入了 async/await，Swift 5.9 引入了 `SE-0393` 和 `SE-0411`，Swift 6 更是**默认开启严格并发检查**：

```swift
// Swift 6 中，以下代码会报错：
class SharedState {
    var count = 0  // ⚠️ Error: shared mutable state is not safe
}

// 必须使用 Actor 或 Sendable 来保证安全
actor SafeState {
    var count = 0  // ✅ Actor 自动保护可变状态
}
```

> 💡 Swift 6 的核心理念：**编译时消灭数据竞争**，而非运行时崩溃后再修复。Actor 和 Sendable 就是实现这一目标的两把利器。

---

## 2. Actor 模型详解

### 2.1 Actor 是什么

Actor 是一种**引用类型**，但与 class 不同的是，它天生保证线程安全。想象 Actor 是一个**有独立办公室的职员**：

- 你不能直接进他的办公室翻文件（不能直接访问属性）
- 你必须通过门口的对讲机跟他说话（通过方法调用）
- 他一次只处理一个请求（串行访问可变状态）
- 你需要等他回复（`await`）

```swift
actor Counter {
    private var value = 0

    func increment() -> Int {
        value += 1
        return value
    }

    func getValue() -> Int {
        return value
    }
}

let counter = Counter()
Task {
    let v1 = await counter.increment()  // 必须用 await
    let v2 = await counter.getValue()   // 必须用 await
    print(v1, v2) // 1 1 —— 不会出现数据竞争
}
```

### 2.2 Actor Isolation（Actor 隔离）

Actor Isolation 是 Actor 的核心机制，它确保：

1. **外部访问必须经过 `await`**——调用方会被挂起，等 Actor 处理完
2. **Actor 内部方法可以直接访问属性**——因为在隔离域内，不存在并发冲突
3. **同一时刻只有一个任务在 Actor 内执行**——自动串行化

```swift
actor UserDataManager {
    private var cache: [String: String] = [:]

    // 在 actor 内部，直接访问 cache，无需 await
    func set(_ value: String, forKey key: String) {
        cache[key] = value
    }

    func get(_ key: String) -> String? {
        return cache[key]
    }

    // nonisolated 方法不在 actor 隔离域内
    nonisolated func description() -> String {
        return "UserDataManager — a thread-safe cache"
        // ⚠️ 不能在这里访问 cache，因为不在隔离域内
    }
}
```

### 2.3 Actor 与 Class 的对比

| 特性 | Class | Actor |
|---|---|---|
| **类型** | 引用类型 | 引用类型 |
| **线程安全** | ❌ 不保证 | ✅ 自动保证 |
| **属性访问** | 任意线程直接访问 | 外部必须 `await` |
| **方法调用** | 直接调用 | 外部调用需 `await` |
| **继承** | 支持单继承 | ❌ 不支持继承 |
| **协议遵循** | `AnyObject` | `Actor` |
| **可变状态** | 需手动加锁 | 自动串行化保护 |
| **适用场景** | UI 模型、ViewController | 共享状态管理、数据缓存 |

### 2.4 Actor 的方法调用规则

```swift
actor ImageCache {
    private var storage: [URL: Data] = [:]

    func store(_ data: Data, for url: URL) {
        storage[url] = data
    }

    func retrieve(for url: URL) -> Data? {
        storage[url]
    }
}

// ✅ 外部调用：必须 await
let cache = ImageCache()
Task {
    await cache.store(imageData, for: url)
    if let data = await cache.retrieve(for: url) {
        process(data)
    }
}

// ❌ 直接访问属性：编译错误
// cache.storage  // Error: Actor-isolated property 'storage' can not be referenced
```

> 💡 规则记忆：**Actor 外 = await，Actor 内 = 直接访问**。就像银行柜台——你在柜台外必须排队等叫号，柜员在柜台内可以直接操作账本。

---

## 3. Actor 实战

### 3.1 计数器 Actor

```swift
actor ThreadSafeCounter {
    private var count = 0

    func increment() -> Int {
        count += 1
        return count
    }

    func decrement() -> Int {
        count -= 1
        return count
    }

    func current() -> Int {
        count
    }

    func reset() {
        count = 0
    }
}

// 并发测试
let counter = ThreadSafeCounter()
await withTaskGroup(of: Void.self) { group in
    for _ in 0..<1000 {
        group.addTask {
            _ = await counter.increment()
        }
    }
}
print(await counter.current()) // 1000 —— 永远正确，不会丢失更新
```

### 3.2 数据缓存 Actor

```swift
actor Cache<Key: Hashable & Sendable, Value: Sendable> {
    private var storage: [Key: Value] = [:]
    private var accessCount: [Key: Int] = [:]

    func get(_ key: Key) -> Value? {
        accessCount[key, default: 0] += 1
        return storage[key]
    }

    func set(_ value: Value, for key: Key) {
        storage[key] = value
        accessCount[key, default: 0] += 1
    }

    func remove(_ key: Key) {
        storage.removeValue(forKey: key)
        accessCount.removeValue(forKey: key)
    }

    func stats(for key: Key) -> (exists: Bool, accessCount: Int) {
        (storage[key] != nil, accessCount[key, default: 0])
    }

    func clear() {
        storage.removeAll()
        accessCount.removeAll()
    }
}
```

### 3.3 避免数据竞争的完整示例

以下是一个典型的"多线程抢票"场景，对比 class 和 actor 的表现：

```swift
// ❌ 不安全：使用 class
class TicketPoolUnsafe {
    private var tickets: Int

    init(tickets: Int) { self.tickets = tickets }

    func buy() -> Bool {
        guard tickets > 0 else { return false }
        tickets -= 1  // 数据竞争可能在这里发生
        return true
    }
}

// ✅ 安全：使用 actor
actor TicketPool {
    private var tickets: Int

    init(tickets: Int) { self.tickets = tickets }

    func buy() -> Bool {
        guard tickets > 0 else { return false }
        tickets -= 1
        return true
    }

    func remaining() -> Int { tickets }
}

// 测试
let pool = TicketPool(tickets: 100)
await withTaskGroup(of: Void.self) { group in
    for _ in 0..<200 {
        group.addTask {
            let success = await pool.buy()
            // 每次只有一个任务能进入 actor，不会超卖
        }
    }
}
print(await pool.remaining()) // 0 —— 永远不会出现负数
```

> ⚠️ Actor 的串行化保证**正确性**，但不保证**公平性**。在高并发场景下，某些任务可能等待较长时间。如果需要更细粒度的控制，考虑将数据拆分到多个 Actor 中。

---

## 4. Sendable 协议

### 4.1 什么是 Sendable

Sendable 是一个**标记协议**（Marker Protocol），表示该类型可以**安全地跨并发域传递**。想象它是一份"安全通行证"——有了它，编译器才允许你把数据从一个并发域送到另一个。

```swift
// Sendable 协议定义（简化）
protocol Sendable {}
```

> 💡 Sendable 本身没有方法要求，它只是一个**编译器指令**，告诉 Swift："这个类型跨并发域传递是安全的"。

### 4.2 哪些类型自动遵循 Sendable

| 类型 | 自动 Sendable | 原因 |
|---|---|---|
| `Int`, `Double`, `Bool` 等 | ✅ | 值类型，不可变 |
| `String` | ✅ | 不可变值类型 |
| `Enum`（关联值也是 Sendable） | ✅ | 值类型，无共享可变状态 |
| `Struct`（所有属性都是 Sendable） | ✅ | 值类型，无共享可变状态 |
| `Array<Sendable>` | ✅ | 元素为 Sendable 的集合 |
| `Dictionary<K: Sendable, V: Sendable>` | ✅ | 键值都为 Sendable |
| `Result<Success: Sendable, Failure: Error>` | ✅ | 两个泛型都是 Sendable |

### 4.3 哪些类型不遵循 Sendable

| 类型 | Sendable | 原因 |
|---|---|---|
| `class` | ❌ | 引用类型，可变状态可被共享 |
| `Actor` | ❌（本身不是 Sendable 值） | 虽然安全，但传递的是引用 |
| 含 `var` 的 struct（属性为 class） | ❌ | 包含非 Sendable 属性 |
| 闭包 | ❌ | 可能捕获可变状态 |
| `NSObject` 子类 | ❌ | 引用类型 |

```swift
// ❌ 不自动遵循 Sendable
class User {
    var name: String
    init(name: String) { self.name = name }
}

// ✅ struct 自动遵循（属性都是 Sendable）
struct UserValue: Sendable {
    let name: String  // String 是 Sendable，let 保证不可变
}

// ✅ 手动声明 Sendable（需要你确保安全）
final class SharedConfig: @unchecked Sendable {
    private let _value: Int
    init(value: Int) { _value = value }
    var value: Int { _value }  // 只读，实际安全
}
```

> ⚠️ `@unchecked Sendable` 绕过了编译器检查，你需要**自己保证线程安全**。只在性能关键路径且你确信安全时使用。

### 4.4 @Sendable 闭包

传递给并发域的闭包必须标记 `@Sendable`，确保闭包不会捕获可变状态：

```swift
func performWork(_ work: @Sendable () async -> Int) async {
    let result = await work()
    print(result)
}

// ✅ 闭包只捕获值类型
let multiplier = 3
await performWork {
    multiplier * 10  // multiplier 是 Int（Sendable），安全
}

// ❌ 捕获可变引用类型
class Counter {
    var count = 0
}
let counter = Counter()
// await performWork {
//     counter.count += 1  // Error: captured 'counter' is not Sendable
// }
```

### 4.5 nonisolated(nonsending)

Swift 6 引入了 `nonisolated(nonsending)` 来标注那些**不会将数据发送到其他并发域**的函数或闭包：

```swift
actor Processor {
    private var data: [String] = []

    // 普通的 nonisolated 方法不能访问 actor 的可变状态
    nonisolated func helper() -> String {
        "processing"  // 不涉及 actor 隔离状态
    }

    // nonisolated(nonsending) 表示参数不会跨域传递
    nonisolated(nonsending) func format(_ input: String) -> String {
        input.uppercased()
    }
}
```

> 💡 `nonisolated(nonsending)` 是 Swift 6 严格并发模型下的新工具，帮助你在不牺牲安全性的前提下，减少不必要的 `await` 开销。

---

## 5. 结构化并发

### 5.1 什么是结构化并发

结构化并发就像**公司的组织架构**：

- 每个任务（员工）都有明确的上级（父任务）
- 员工离职前必须完成手头工作（子任务必须完成，父任务才能结束）
- 部门解散时，所有员工一起离开（取消传播）

| 特性 | 结构化并发 | 非结构化并发 |
|---|---|---|
| **生命周期** | 父任务管理子任务 | 独立生命周期 |
| **错误传播** | 子任务错误自动传播给父 | 需手动处理 |
| **取消传播** | 父取消 → 子自动取消 | 需手动传播 |
| **典型 API** | `withTaskGroup` | `Task.init`, `Task.detached` |

### 5.2 Task 的生命周期管理

```swift
func fetchDashboard() async throws -> Dashboard {
    // async let：并发启动，结构化等待
    async let user = fetchUser()
    async let feed = fetchFeed()
    async let notifications = fetchNotifications()

    // 三个请求并发执行，await 时按需等待
    return try await Dashboard(
        user: user,
        feed: feed,
        notifications: notifications
    )
}
```

> 💡 `async let` 是最轻量的结构化并发方式——适合"并发启动、统一等待"的场景。但如果子任务数量动态变化，需要用 TaskGroup。

### 5.3 TaskGroup 与 throwingTaskGroup

| 类型 | 子任务是否可抛出 | 用法 |
|---|---|---|
| `TaskGroup<Result>` | 子任务不会抛出 | `withTaskGroup` |
| `ThrowingTaskGroup<Result>` | 子任务可能抛出 | `withThrowingTaskGroup` |

### 5.4 withTaskGroup 代码示例

```swift
func fetchAllImages(urls: [URL]) async throws -> [UIImage] {
    try await withThrowingTaskGroup(of: (Int, UIImage).self) { group in
        // 添加子任务
        for (index, url) in urls.enumerated() {
            group.addTask {
                let (data, _) = try await URLSession.shared.data(from: url)
                guard let image = UIImage(data: data) else {
                    throw ImageError.invalidData
                }
                return (index, image)
            }
        }

        // 收集结果，保持顺序
        var results: [(Int, UIImage)] = []
        results.reserveCapacity(urls.count)
        for try await (index, image) in group {
            results.append((index, image))
        }
        return results.sorted { $0.0 < $1.0 }.map { $0.1 }
    }
}
```

> ⚠️ 在 `withTaskGroup` 的闭包中，**必须**消费所有子任务的结果（通过 `for await in group`）。如果提前 return，未完成的子任务会被自动取消。

### 5.5 TaskGroup 的取消传播

```swift
func searchItems(query: String) async throws -> [Item] {
    try await withThrowingTaskGroup(of: Item.self) { group in
        for source in dataSources {
            group.addTask {
                try await source.search(query: query)
            }
        }

        var results: [Item] = []
        for try await item in group {
            results.append(item)
            if results.count >= 20 {
                group.cancelAll()  // 已收集足够结果，取消剩余任务
                break
            }
        }
        return results
    }
}
```

---

## 6. 非结构化并发

### 6.1 Task.init——继承上下文的非结构化任务

```swift
@MainActor
class ViewModel: ObservableObject {
    @Published var items: [Item] = []

    func load() {
        Task {  // 继承当前 Actor 上下文（MainActor）
            let data = await fetchItems()
            self.items = data  // ✅ 在 MainActor 上，可以更新 UI
        }
    }
}
```

### 6.2 Task.detached——完全独立的任务

```swift
func processInBackground() {
    Task.detached(priority: .background) {
        // 不继承任何 Actor 上下文
        let result = await heavyComputation()
        await MainActor.run {
            updateUI(with: result)
        }
    }
}
```

### 6.3 Task.init 与 Task.detached 对比

| 特性 | `Task.init` | `Task.detached` |
|---|---|---|
| **继承 Actor 上下文** | ✅ 继承调用者的 Actor | ❌ 不继承 |
| **继承优先级** | ✅ 继承 | ❌ 需手动指定 |
| **继承任务本地值** | ✅ 继承 | ❌ 不继承 |
| **取消传播** | 父任务取消 → 子任务取消 | ❌ 独立生命周期 |
| **使用频率** | 日常开发首选 | 后台独立任务 |
| **类比** | 员工在公司架构内工作 | 自由职业者独立接单 |

> 💡 **经验法则**：90% 的场景用 `Task.init`，只有当你确实需要一个完全独立于当前上下文的任务时，才用 `Task.detached`。

---

## 7. AsyncSequence 与 AsyncStream

### 7.1 AsyncSequence——异步版的 Sequence

`AsyncSequence` 是异步序列协议，让你可以用 `for await in` 遍历异步产生的值：

```swift
// URLSession 的 lines 方法返回 AsyncSequence
func streamChat(url: URL) async throws {
    let (bytes, _) = try await URLSession.shared.bytes(from: url)
    for try await line in bytes.lines {
        print("收到: \(line)")
    }
}
```

### 7.2 自定义异步序列

```swift
struct Countdown: AsyncSequence {
    typealias Element = Int
    let from: Int

    struct AsyncIterator: AsyncIteratorProtocol {
        var current: Int

        mutating func next() async -> Int? {
            guard current >= 0 else { return nil }
            try? await Task.sleep(nanoseconds: 1_000_000_000)
            let value = current
            current -= 1
            return value
        }
    }

    func makeAsyncIterator() -> AsyncIterator {
        AsyncIterator(current: from)
    }
}

// 使用
for await number in Countdown(from: 5) {
    print(number) // 5, 4, 3, 2, 1, 0
}
```

### 7.3 AsyncStream——桥接回调到异步世界

AsyncStream 是将传统回调式 API 转换为 `AsyncSequence` 的桥梁：

```swift
func locationStream() -> AsyncStream<CLLocation> {
    AsyncStream(bufferingPolicy: .bufferingNewest(1)) { continuation in
        let delegate = LocationDelegate { location in
            continuation.yield(location)
        }

        let manager = CLLocationManager()
        manager.delegate = delegate
        manager.startUpdatingLocation()

        continuation.onTermination = { _ in
            manager.stopUpdatingLocation()
        }
    }
}

// 使用
func trackLocation() async {
    for await location in locationStream() {
        print("当前位置: \(location.coordinate)")
    }
}
```

> 💡 `bufferingPolicy` 决定了消费者跟不上生产者时的策略：`.unbounded`（无限缓冲）、`.bufferingNewest(n)`（保留最新 n 个）、`.bufferingOldest(n)`（保留最早 n 个）。

### 7.4 AsyncStream 与 Combine 对比

| 特性 | AsyncStream | Combine |
|---|---|---|
| **语言层级** | Swift 标准库 | 框架（需 import Combine） |
| **学习曲线** | 低（类似 for 循环） | 高（操作符众多） |
| **背压处理** | bufferingPolicy | 自定义 Publisher |
| **错误处理** | AsyncThrowingStream | Publisher 的 Failure 类型 |
| **组合操作** | 需手动编写 | 丰富的操作符（map, filter, merge...） |
| **适用场景** | 简单回调桥接、单一数据流 | 复杂响应式链、多流组合 |
| **取消** | for 循环退出自动取消 | AnyCancellable 手动管理 |

> ⚠️ 不要在 AsyncStream 的 `yield` 之后继续使用已 yield 的值——值已被消费端取走。同时避免在 `onTermination` 之外清理资源。

---

## 8. 并发最佳实践

### 8.1 避免过度创建 Task

```swift
// ❌ 不要为每个小操作创建 Task
func loadPage() async {
    Task { await loadHeader() }
    Task { await loadContent() }
    Task { await loadFooter() }
    // 三个独立 Task，无法统一管理取消和错误
}

// ✅ 使用 async let 或 TaskGroup
func loadPage() async throws {
    async let header = loadHeader()
    async let content = loadContent()
    async let footer = loadFooter()
    _ = try await (header, content, footer)
}
```

### 8.2 MainActor 使用原则

```swift
// ❌ 不要把整个类标记为 @MainActor，如果只有部分需要
@MainActor
class ViewModel {
    func computeHash() -> String {  // 不需要主线程
        // 纯计算，不需要 MainActor
        return "hash"
    }
}

// ✅ 只在需要的地方使用 MainActor
class ViewModel {
    @MainActor var displayText: String = ""

    func computeHash() -> String {
        // 在当前线程执行，不阻塞主线程
        return "hash"
    }

    @MainActor
    func updateDisplay() {
        displayText = computeHash()
    }
}
```

> 💡 MainActor 的判断标准：**是否涉及 UI 更新或 UIKit API 调用**？如果不是，就不需要 MainActor。

### 8.3 优先级管理

```swift
// 合理设置优先级
Task(priority: .userInitiated) {
    // 用户主动触发的操作：高优先级
    await searchDatabase()
}

Task(priority: .background) {
    // 后台同步、日志上传：低优先级
    await syncAnalytics()
}

Task(priority: .utility) {
    // 数据预加载：中等优先级
    await prefetchData()
}
```

| 优先级 | 适用场景 | 示例 |
|---|---|---|
| `.high` / `.userInitiated` | 用户正在等待的操作 | 搜索、页面加载 |
| `.medium` / `.default` | 默认优先级 | 普通数据请求 |
| `.low` / `.utility` | 非紧急操作 | 预加载、缓存预热 |
| `.background` | 用户无感知的操作 | 日志上传、数据同步 |

### 8.4 取消处理

```swift
func downloadLargeFile(url: URL) async throws -> Data {
    try Task.checkCancellation()  // 入口处检查

    let (bytes, response) = try await URLSession.shared.bytes(from: url)
    var data = Data()
    data.reserveCapacity(Int(response.expectedContentLength))

    for try await byte in bytes {
        try Task.checkCancellation()  // 循环中定期检查
        data.append(byte)
    }
    return data
}

// 使用 cooperative cancellation
func startDownload() {
    let task = Task {
        do {
            let data = try await downloadLargeFile(url: fileURL)
            process(data)
        } catch is CancellationError {
            print("下载已取消")
        } catch {
            print("下载失败: \(error)")
        }
    }

    // 用户点击取消
    cancelButton.addAction(UIAction { _ in
        task.cancel()
    })
}
```

> ⚠️ Swift 并发的取消是**协作式**的——调用 `task.cancel()` 只是设置取消标志，不会强制中断。你需要在长时间运行的任务中定期调用 `Task.checkCancellation()` 或检查 `Task.isCancelled`。

---

## 本章小结

| 主题 | 核心要点 | 关键 API |
|---|---|---|
| **并发必要性** | 数据竞争难调试，Swift 6 严格并发是趋势 | `-strict-concurrency` |
| **Actor** | 引用类型 + 自动串行化 = 线程安全 | `actor`, `nonisolated` |
| **Actor 实战** | 计数器、缓存、抢票——用 Actor 替代手动加锁 | `actor`, `await` |
| **Sendable** | 标记协议，保证跨并发域安全传递 | `Sendable`, `@Sendable`, `@unchecked Sendable` |
| **结构化并发** | 父管理子，取消传播，错误传播 | `withTaskGroup`, `async let` |
| **非结构化并发** | 独立生命周期，需手动管理 | `Task.init`, `Task.detached` |
| **AsyncSequence/Stream** | 异步数据流，桥接回调到 async/await | `AsyncStream`, `AsyncThrowingStream` |
| **最佳实践** | 不过度创建 Task、精用 MainActor、优先级、取消处理 | `Task.checkCancellation()`, `priority:` |

> 💡 **学习路径建议**：先掌握 Actor 和 Sendable（解决数据竞争），再学结构化并发（管理任务生命周期），最后深入 AsyncStream（处理异步数据流）。Swift 6 的严格并发模式是未来，现在打好基础，迁移时事半功倍。
