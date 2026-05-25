# 75-Combine 与响应式编程

## 本章目标

- 理解响应式编程的核心思想与事件流思维
- 掌握 Combine 框架的 Publisher / Subscriber / Operator 三大支柱
- 熟练使用常用 Publisher（Just / Future / Deferred / Subject）
- 能够运用 Transforming / Filtering / Combining / Timing 四类操作符
- 理解订阅生命周期与内存管理
- 学会在 SwiftUI 中集成 Combine（.onReceive / @Published / ObservableObject）
- 通过实战构建一个防抖搜索框
- 明确 Combine 与 async/await 的选择策略

---

## 1. 响应式编程概述

### 1.1 什么是响应式编程

响应式编程（Reactive Programming）是一种面向**数据流**和**变化传播**的编程范式。简单来说：当数据发生变化时，依赖它的逻辑会**自动响应**，而不需要你手动去通知。

> 💡 **生活类比**：想象你订阅了一份报纸——报社出新报纸（数据变化），邮递员自动送到你家（自动响应）。你不需要每天打电话问"今天有报纸吗？"，也不需要自己去取。

### 1.2 事件流思维

传统编程是"一步一指令"的**命令式思维**，而响应式编程是"数据像水流一样流动"的**事件流思维**：

```
用户输入 → [防抖] → [过滤空值] → [网络请求] → [解析JSON] → [更新UI]
   ↑                                                              ↓
  源头                                                          终点
```

整条链路就像一条**水管**：数据从源头流入，经过一个个"阀门"（操作符）处理，最终流到终点（订阅者）。

### 1.3 Pull vs Push 模型

| 模型 | 谁主动 | 类比 | 典型代表 |
|------|--------|------|----------|
| **Pull（拉取）** | 消费者主动请求数据 | 你去超市买东西 | `for` 循环、`Iterator` |
| **Push（推送）** | 生产者主动推送数据 | 快递送到家门口 | `Combine`、`NotificationCenter` |

Combine 采用的是 **Push 模型**——数据准备好了就推送给你，你不需要反复询问。

### 1.4 响应式 vs 命令式对比

| 对比项 | 命令式编程 | 响应式编程 |
|--------|-----------|-----------|
| 核心思维 | "做什么"的步骤 | "发生什么"的响应 |
| 状态管理 | 手动维护变量 | 数据流自动传播 |
| 异步处理 | 回调 / 闭包嵌套 | 声明式操作符链 |
| 代码风格 | 分散的事件处理 | 统一的流水线 |
| 典型痛点 | 回调地狱 | 学习曲线陡峭 |

> ⚠️ **注意**：响应式编程不是银弹。对于简单的同步逻辑，命令式代码更直观。响应式的优势在于处理**复杂的异步事件流**。

---

## 2. Combine 核心概念

### 2.1 Publisher / Subscriber / Subscription 三者关系

Combine 的核心是三个角色的协作：

| 角色 | 职责 | 类比 |
|------|------|------|
| **Publisher** | 产生并发送数据 | 报社——生产报纸 |
| **Subscriber** | 接收并处理数据 | 读者——阅读报纸 |
| **Subscription** | 管理订阅关系 | 邮局——连接报社和读者 |

```swift
// Publisher 发出三种事件：
// 1. value  —— 正常数据（可以发多次）
// 2. completion —— 成功结束（只发一次）
// 3. failure —— 出错终止（只发一次）
```

### 2.2 生命周期

一个完整的订阅生命周期如下：

```
创建 Publisher → 订阅(Subscriber 请求) → 收到 value(可多次) → 收到 completion 或 failure
```

> 💡 **关键规则**：一旦发出 `completion` 或 `failure`，Publisher 就不会再发出任何值。这就像电视剧大结局——播完就结束了。

### 2.3 Operator 链式调用

Operator 是连接 Publisher 和 Subscriber 的"中间处理器"。它们接收上游数据，处理后传给下游，形成一条**链式流水线**：

```swift
publisher
    .map { $0.uppercased() }      // 转换
    .filter { $0.count > 3 }      // 过滤
    .sink { value in              // 订阅
        print(value)
    }
```

每个 Operator 本质上既是上游的 Subscriber，又是下游的 Publisher——**承上启下**。

---

## 3. Publisher 详解

### 3.1 Just——立即发送一个值

`Just` 是最简单的 Publisher：创建时给定一个值，订阅后**立即发送**，然后完成。

```swift
import Combine

Just("Hello, Combine!")
    .sink { value in
        print(value)  // Hello, Combine!
    }
```

| 特性 | 说明 |
|------|------|
| 发送值数量 | 恰好 1 个 |
| 错误类型 | `Never`（不会失败） |
| 适用场景 | 将常量/同步值包装成 Publisher |

> 💡 **使用场景**：当你需要一个"总是成功"的 Publisher 来衔接操作符链时，`Just` 是最简洁的选择。

### 3.2 Future——异步产生一个值

`Future` 用于包装一个**异步操作**，最终产生一个值或一个错误。

```swift
func fetchUserID() -> Future<Int, Error> {
    Future { promise in
        URLSession.shared.dataTask(with: URL(string: "https://api.example.com/user")!) { data, _, error in
            if let error = error {
                promise(.failure(error))
            } else if let data = data {
                let id = Int(data: data) ?? 0
                promise(.success(id))
            }
        }
        .resume()
    }
}
```

| 特性 | 说明 |
|------|------|
| 发送值数量 | 0 或 1 个 |
| 错误类型 | 自定义 |
| 执行时机 | 创建时即开始（热 Publisher） |
| 适用场景 | 包装单次异步操作（网络请求、文件读取） |

> ⚠️ **注意**：`Future` 在**创建时**就会执行闭包，即使还没有订阅者。如果不希望提前执行，请使用 `Deferred`。

### 3.3 Deferred——延迟创建 Publisher

`Deferred` 会等到**有订阅者时**才创建真正的 Publisher，解决了 `Future` 提前执行的问题。

```swift
let deferredFetch = Deferred {
    Future<Int, Error> { promise in
        print("开始请求...")  // 只在订阅时才打印
        let id = Int.random(in: 1...100)
        promise(.success(id))
    }
}

// 此时不会执行任何操作
// 只有订阅后才会创建 Future 并执行
deferredFetch.sink { completion in
    // 处理完成
} receiveValue: { id in
    print("收到 ID: \(id)")
}
```

| 特性 | 说明 |
|------|------|
| 执行时机 | 每次订阅时才创建 |
| 多次订阅 | 每次创建新的 Publisher |
| 适用场景 | 避免提前执行、需要每次订阅重新计算 |

### 3.4 PassthroughSubject——手动发送值

`Subject` 是一种可以**手动向下游推送值**的 Publisher，类似于"广播站"。

```swift
let searchSubject = PassthroughSubject<String, Never>()

searchSubject
    .filter { !$0.isEmpty }
    .sink { keyword in
        print("搜索: \(keyword)")
    }

searchSubject.send("Swift")     // 搜索: Swift
searchSubject.send("")          // 被过滤
searchSubject.send("Combine")   // 搜索: Combine
searchSubject.send(completion: .finished)  // 结束
```

| 特性 | 说明 |
|------|------|
| 初始值 | 无（不保存当前值） |
| 新订阅者 | 只能收到订阅后的值 |
| 适用场景 | 事件广播、UI 交互信号 |

### 3.5 CurrentValueSubject——带初始值的 Subject

`CurrentValueSubject` 在 `PassthroughSubject` 的基础上增加了一个**当前值**，新订阅者会立即收到当前值。

```swift
let temperature = CurrentValueSubject<Double, Never>(36.5)

temperature
    .sink { value in
        print("温度: \(value)")
    }
// 立即打印: 温度: 36.5

temperature.send(37.2)   // 温度: 37.2
temperature.send(38.0)   // 温度: 38.0

print(temperature.value)  // 38.0（可以随时读取当前值）
```

| 特性 | 说明 |
|------|------|
| 初始值 | 必须提供 |
| 新订阅者 | 立即收到当前值 |
| 读取当前值 | 通过 `.value` 属性 |
| 适用场景 | 状态管理、设置项、ViewModel 属性 |

### 3.6 Publisher 选择指南

| Publisher | 值数量 | 错误 | 执行时机 | 典型用途 |
|-----------|--------|------|----------|----------|
| `Just` | 1 | Never | 立即 | 同步值包装 |
| `Future` | 0 或 1 | 自定义 | 创建时 | 单次异步操作 |
| `Deferred` | 取决于内部 | 自定义 | 订阅时 | 延迟执行 |
| `PassthroughSubject` | 多个 | 自定义 | 手动 send | 事件广播 |
| `CurrentValueSubject` | 多个 | 自定义 | 手动 send | 状态管理 |

---

## 4. Operator 操作符

操作符是 Combine 最强大的部分。它们分为四大类：

### 4.1 Transforming 操作符

| 操作符 | 作用 | 类比 |
|--------|------|------|
| `map` | 转换每个值 | 翻译官——把中文翻成英文 |
| `flatMap` | 转换为新的 Publisher 并展平 | 快递分拣——一个包裹拆成多个 |
| `scan` | 累积计算 | 计算器——逐次累加 |

```swift
// map: 将摄氏度转为华氏度
let celsius = PassthroughSubject<Double, Never>()

celsius
    .map { c in c * 9 / 5 + 32 }
    .sink { f in print("华氏: \(f)") }

celsius.send(0)    // 华氏: 32.0
celsius.send(100)  // 华氏: 212.0
```

```swift
// flatMap: 将每个搜索词转为网络请求 Publisher
let searchQuery = PassthroughSubject<String, Never>()

searchQuery
    .flatMap { query in
        URLSession.shared.dataTaskPublisher(for: URL(string: "https://api.example.com/search?q=\(query)")!)
            .map(\.data)
            .replaceError(with: Data())
    }
    .sink { data in
        print("收到数据: \(data.count) bytes")
    }
```

```swift
// scan: 累计求和
let scores = [10, 20, 30, 40].publisher

scores
    .scan(0, +)
    .sink { print($0) }
// 10, 30, 60, 100
```

### 4.2 Filtering 操作符

| 操作符 | 作用 | 类比 |
|--------|------|------|
| `filter` | 只保留满足条件的值 | 保安——只让有通行证的人进 |
| `removeDuplicates` | 去除连续重复值 | 漏斗——过滤掉重复的沙子 |
| `compactMap` | 过滤 nil 并解包 | 邮局——扔掉空信封 |

```swift
let input = PassthroughSubject<Int, Never>()

input
    .filter { $0 % 2 == 0 }
    .sink { print("偶数: \($0)") }

input.send(1)  // 被过滤
input.send(2)  // 偶数: 2
input.send(3)  // 被过滤
input.send(4)  // 偶数: 4
```

```swift
let text = PassthroughSubject<String, Never>()

text
    .removeDuplicates()
    .sink { print($0) }

text.send("Swift")    // Swift
text.send("Swift")    // 被去除（与上一个相同）
text.send("Combine")  // Combine
```

### 4.3 Combining 操作符

| 操作符 | 作用 | 类比 |
|--------|------|------|
| `zip` | 配对合并（等最慢的） | 拉链——两边对齐才合上 |
| `combineLatest` | 取各自最新值组合 | 新闻联播——取最新消息 |
| `merge` | 合并成一个流 | 两条河汇成一条 |

```swift
let name = PassthroughSubject<String, Never>()
let age = PassthroughSubject<Int, Never>()

// zip: 等双方都有新值才组合
name.zip(age)
    .sink { print("zip: \($0), \($1)") }

name.send("张三")   // 等待 age
age.send(25)        // zip: 张三, 25
name.send("李四")   // 等待 age
age.send(30)        // zip: 李四, 30
```

```swift
// combineLatest: 任一变化就用最新值组合
let width = CurrentValueSubject<Int, Never>(100)
let height = CurrentValueSubject<Int, Never>(200)

width.combineLatest(height)
    .map { w, h in w * h }
    .sink { print("面积: \($0)") }
// 面积: 20000

width.send(150)    // 面积: 30000
height.send(300)   // 面积: 45000
```

### 4.4 Timing 操作符

| 操作符 | 作用 | 类比 |
|--------|------|------|
| `debounce` | 停顿一段时间后才发送 | 电梯等人——没人进来才关门 |
| `throttle` | 固定间隔内只发第一个 | 水龙头限流——每秒最多流一滴 |
| `delay` | 延迟发送 | 定时炸弹——倒计时后爆炸 |

```swift
// debounce: 搜索框防抖——用户停止输入 0.5 秒后才搜索
let searchText = PassthroughSubject<String, Never>()

searchText
    .debounce(for: .milliseconds(500), scheduler: RunLoop.main)
    .sink { print("搜索: \($0)") }

// 用户快速输入 "S", "Sw", "Swi", "Swif", "Swift"
// 只有 "Swift" 会触发搜索
```

```swift
// throttle: 限制按钮点击频率——每 2 秒最多响应一次
let buttonTap = PassthroughSubject<Void, Never>()

buttonTap
    .throttle(for: .seconds(2), scheduler: RunLoop.main, latest: false)
    .sink { print("执行操作") }
```

```swift
// delay: 延迟 3 秒显示欢迎消息
Just("欢迎回来！")
    .delay(for: .seconds(3), scheduler: RunLoop.main)
    .sink { print($0) }
```

---

## 5. Subscriber 与 Cancellable

### 5.1 sink——最灵活的订阅方式

`sink` 提供两个闭包：一个处理值，一个处理完成/错误。

```swift
let publisher = [1, 2, 3].publisher

let subscription = publisher
    .sink { completion in
        switch completion {
        case .finished:
            print("完成")
        case .failure(let error):
            print("错误: \(error)")
        }
    } receiveValue: { value in
        print("值: \(value)")
    }
// 值: 1
// 值: 2
// 值: 3
// 完成
```

### 5.2 assign——将值绑定到属性

`assign` 将 Publisher 的值自动赋给某个对象的属性（通过 KeyPath）。

```swift
class ViewModel: ObservableObject {
    @Published var username: String = ""
    @Published var isLoading: Bool = false
}

let vm = ViewModel()
let namePublisher = Just("张三")

namePublisher
    .assign(to: \.username, on: vm)
```

> ⚠️ **注意**：`assign` 要求 Publisher 的错误类型为 `Never`。如果有错误可能，先用 `replaceError` 或 `catch` 处理。

### 5.3 AnyCancellable 与内存管理

每次调用 `sink` 或 `assign` 都会返回一个 `AnyCancellable`。当它被销毁（`deinit`）时，订阅会**自动取消**。

```swift
class SearchViewController: UIViewController {
    var cancellables = Set<AnyCancellable>()
    let searchText = PassthroughSubject<String, Never>()

    func bind() {
        searchText
            .debounce(for: .milliseconds(300), scheduler: RunLoop.main)
            .sink { [weak self] query in
                self?.performSearch(query)
            }
            .store(in: &cancellables)  // 存入集合，统一管理
    }

    deinit {
        // cancellables 被销毁，所有订阅自动取消
    }
}
```

> 💡 **最佳实践**：在 ViewModel / ViewController 中声明 `var cancellables = Set<AnyCancellable>()`，用 `.store(in:)` 统一管理订阅。对象销毁时，所有订阅自动清理，不会内存泄漏。

### 5.4 订阅生命周期总结

| 阶段 | 说明 | 对应方法 |
|------|------|----------|
| 创建 | Subscriber 订阅 Publisher | `receive(subscription:)` |
| 请求 | Subscriber 告诉 Publisher 需要多少值 | `Subscribers.Demand` |
| 接收值 | Publisher 推送值给 Subscriber | `receive(_:)` |
| 完成 | Publisher 发送 completion 或 failure | `receive(completion:)` |
| 取消 | AnyCancellable 被销毁 | 自动调用 `cancel()` |

---

## 6. SwiftUI 与 Combine 集成

### 6.1 .onReceive 修饰符

`.onReceive` 让 SwiftUI 视图直接订阅 Publisher，每当收到新值时刷新视图。

```swift
struct TimerView: View {
    @State private var currentTime = ""

    var body: some View {
        Text(currentTime)
            .font(.title)
            .onReceive(Timer.publish(every: 1, on: .main, in: .common).autoconnect()) { time in
                currentTime = DateFormatter.localizedString(from: time, dateStyle: .none, timeStyle: .medium)
            }
    }
}
```

### 6.2 @Published 属性包装器

`@Published` 是 Combine 与 SwiftUI 之间的桥梁——它将属性变化包装成一个 Publisher。

```swift
class AppState: ObservableObject {
    @Published var isLoggedIn: Bool = false
    @Published var username: String = ""

    var publishedPublisher: Published<Bool>.Publisher {
        $isLoggedIn  // $ 前缀访问 Publisher
    }
}
```

> 💡 **关键点**：`@Published var value` 中的 `$value` 是一个 `Publisher`，可以在 Combine 链中使用。这是 Combine 监听 `ObservableObject` 变化的底层机制。

### 6.3 ObservableObject 内部机制

`ObservableObject` 的 `objectWillChange` 属性本身就是一个 `Publisher`：

```swift
class Store: ObservableObject {
    @Published var items: [String] = []
    @Published var filter: String = ""

    // objectWillChange 是 ObservableObject 协议提供的 Publisher
    // 任何 @Published 属性变化时，objectWillChange 都会发送通知
}

struct StoreView: View {
    @StateObject private var store = Store()

    var body: some View {
        List(store.items, id: \.self) { item in
            Text(item)
        }
        .onReceive(store.$filter) { newFilter in
            print("筛选条件变为: \(newFilter)")
        }
    }
}
```

### 6.4 自定义 Bindings

利用 Combine，我们可以创建带有验证逻辑的自定义 Binding：

```swift
struct FormView: View {
    @State private var email = ""
    @State private var isValid = false

    var body: some View {
        Form {
            TextField("邮箱", text: $email)
                .onReceive(Just(email)) { _ in
                    isValid = email.contains("@") && email.contains(".")
                }

            Button("提交") {
                print("提交: \(email)")
            }
            .disabled(!isValid)
        }
    }
}
```

更优雅的方式是使用 `onChange` + Combine 操作符链：

```swift
struct SearchableView: View {
    @State private var query = ""
    @State private var results: [String] = []
    @StateObject private var vm = SearchViewModel()

    var body: some View {
        VStack {
            TextField("搜索", text: $query)
                .textFieldStyle(.roundedBorder)
                .onReceive(vm.$searchResults) { results in
                    self.results = results
                }
        }
        .onChange(of: query) { newValue in
            vm.searchSubject.send(newValue)
        }
    }
}
```

---

## 7. 实战：用 Combine 构建搜索框

这是 Combine 最经典的应用场景：用户输入 → 防抖 → 过滤 → 网络请求 → 更新 UI。

### 7.1 ViewModel 设计

```swift
import Combine
import Foundation

class SearchViewModel: ObservableObject {
    @Published var searchText: String = ""
    @Published var results: [SearchResult] = []
    @Published var isLoading: Bool = false
    @Published var errorMessage: String?

    let searchSubject = PassthroughSubject<String, Never>()
    private var cancellables = Set<AnyCancellable>()

    struct SearchResult: Identifiable {
        let id = UUID()
        let title: String
    }

    init() {
        bindSearch()
    }

    private func bindSearch() {
        searchSubject
            .debounce(for: .milliseconds(500), scheduler: RunLoop.main)
            .removeDuplicates()
            .filter { !$0.trimmingCharacters(in: .whitespaces).isEmpty }
            .map { query -> String in
                self.isLoading = true
                self.errorMessage = nil
                return query
            }
            .flatMap { query in
                self.searchAPI(query: query)
                    .catch { error -> Just<[SearchResult]> in
                        DispatchQueue.main.async {
                            self.errorMessage = error.localizedDescription
                        }
                        return Just([])
                    }
            }
            .receive(on: RunLoop.main)
            .sink { results in
                self.isLoading = false
                self.results = results
            }
            .store(in: &cancellables)
    }

    private func searchAPI(query: String) -> Future<[SearchResult], Error> {
        Future { promise in
            guard let url = URL(string: "https://api.example.com/search?q=\(query.addingPercentEncoding(withAllowedCharacters: .urlQueryAllowed) ?? "")") else {
                promise(.failure(URLError(.badURL)))
                return
            }

            URLSession.shared.dataTask(with: url) { data, response, error in
                if let error = error {
                    promise(.failure(error))
                    return
                }

                guard let data = data else {
                    promise(.success([]))
                    return
                }

                do {
                    let decoded = try JSONDecoder().decode([SearchResult].self, from: data)
                    promise(.success(decoded))
                } catch {
                    promise(.failure(error))
                }
            }
            .resume()
        }
    }
}
```

### 7.2 SwiftUI 视图

```swift
import SwiftUI

struct SearchView: View {
    @StateObject private var vm = SearchViewModel()

    var body: some View {
        NavigationStack {
            VStack {
                HStack {
                    Image(systemName: "magnifyingglass")
                        .foregroundStyle(.gray)

                    TextField("搜索...", text: $vm.searchText)
                        .textFieldStyle(.plain)
                        .onChange(of: vm.searchText) { newValue in
                            vm.searchSubject.send(newValue)
                        }

                    if vm.isLoading {
                        ProgressView()
                    }

                    if !vm.searchText.isEmpty {
                        Button(action: { vm.searchText = "" }) {
                            Image(systemName: "xmark.circle.fill")
                                .foregroundStyle(.gray)
                        }
                    }
                }
                .padding(8)
                .background(Color(.systemGray6))
                .cornerRadius(10)
                .padding(.horizontal)

                if let error = vm.errorMessage {
                    Text(error)
                        .foregroundStyle(.red)
                        .font(.caption)
                        .padding(.horizontal)
                }

                List(vm.results) { result in
                    Text(result.title)
                }

                Spacer()
            }
            .navigationTitle("搜索")
        }
    }
}
```

### 7.3 数据流全链路

```
用户输入 "S" → "Sw" → "Swi" → "Swif" → "Swift"
    ↓ (searchSubject.send)
[debounce 500ms] —— 只有 "Swift" 通过
    ↓
[removeDuplicates] —— 重复输入被过滤
    ↓
[filter 空值] —— 空字符串被过滤
    ↓
[flatMap → searchAPI] —— 发起网络请求
    ↓
[catch 错误] —— 网络失败时显示错误
    ↓
[receive(on: main)] —— 切回主线程
    ↓
[sink] —— 更新 results / isLoading
```

> 💡 **关键设计**：`debounce` 防止每敲一个字就发请求，`removeDuplicates` 防止重复搜索，`catch` 保证一个搜索失败不会中断整个链路。

---

## 8. Combine vs async/await 对比

Swift 5.5 引入的 `async/await` 让异步代码更直观，但它和 Combine 并非互斥，而是各有侧重：

### 8.1 核心对比表

| 对比项 | Combine | async/await |
|--------|---------|-------------|
| 编程范式 | 声明式/响应式 | 命令式/顺序式 |
| 核心抽象 | 数据流（Publisher） | 任务（Task） |
| 多值处理 | ✅ 天然支持流 | 需要 AsyncStream 包装 |
| 操作符链 | ✅ 丰富内置 | ❌ 需手动组合 |
| 防抖/节流 | ✅ debounce/throttle | ❌ 需自己实现 |
| 错误处理 | 操作符链中处理 | try/catch |
| 取消机制 | AnyCancellable | Task.cancel() |
| 学习曲线 | 陡峭 | 平缓 |
| 代码可读性 | 链式调用，紧凑 | 顺序执行，直观 |
| SwiftUI 集成 | .onReceive / @Published | .task / async 修饰符 |

### 8.2 何时用 Combine

- 需要处理**持续的事件流**（搜索框、传感器数据、WebSocket）
- 需要**防抖/节流**等时间操作
- 需要**组合多个数据源**（combineLatest / zip）
- 需要在**多个组件间广播**事件（Subject）
- 与 `@Published` / `ObservableObject` 深度集成

### 8.3 何时用 async/await

- **单次异步操作**（一次性网络请求、文件读取）
- 逻辑是**顺序执行**的（先 A 再 B 再 C）
- 团队对响应式编程**不熟悉**
- 代码**可读性**优先

### 8.4 混合使用策略

两者可以完美配合，各取所长：

```swift
class HybridViewModel: ObservableObject {
    @Published var user: User?
    @Published var posts: [Post] = []

    let refreshTrigger = PassthroughSubject<Void, Never>()
    private var cancellables = Set<AnyCancellable>()

    struct User: Codable { let name: String }
    struct Post: Codable { let title: String }

    init() {
        refreshTrigger
            .flatMap { _ in
                self.refreshAll()  // 返回 Future 或 AnyPublisher
            }
            .sink { completion in
                // 处理完成
            } receiveValue: { _ in
                // 数据已通过 @Published 更新
            }
            .store(in: &cancellables)
    }

    private func refreshAll() -> AnyPublisher<Void, Error> {
        Future { promise in
            Task {
                do {
                    async let user = self.fetchUser()
                    async let posts = self.fetchPosts()
                    let (fetchedUser, fetchedPosts) = try await (user, posts)
                    await MainActor.run {
                        self.user = fetchedUser
                        self.posts = fetchedPosts
                    }
                    promise(.success(()))
                } catch {
                    promise(.failure(error))
                }
            }
        }
        .eraseToAnyPublisher()
    }

    private func fetchUser() async throws -> User {
        let (data, _) = try await URLSession.shared.data(from: URL(string: "https://api.example.com/user")!)
        return try JSONDecoder().decode(User.self, from: data)
    }

    private func fetchPosts() async throws -> [Post] {
        let (data, _) = try await URLSession.shared.data(from: URL(string: "https://api.example.com/posts")!)
        return try JSONDecoder().decode([Post].self, from: data)
    }
}
```

> 💡 **混合策略总结**：用 Combine 管理事件流和触发时机（防抖、广播），用 async/await 处理具体的异步操作（网络请求、数据解析）。两者结合，既享受响应式的流式处理，又保持异步代码的直观可读。

---

## 本章小结

| 主题 | 核心要点 |
|------|----------|
| 响应式编程 | 面向数据流和变化传播，Push 模型，事件流思维 |
| 三大角色 | Publisher 产生数据、Subscriber 消费数据、Subscription 管理关系 |
| Publisher | Just（同步单值）、Future（异步单值）、Deferred（延迟创建）、Subject（手动推送） |
| 操作符 | Transforming（map/flatMap/scan）、Filtering（filter/removeDuplicates）、Combining（zip/combineLatest/merge）、Timing（debounce/throttle/delay） |
| 订阅管理 | sink 订阅、assign 绑定、AnyCancellable 自动取消、Set 统一存储 |
| SwiftUI 集成 | .onReceive 监听、@Published 桥接、ObservableObject 内部用 Combine 驱动 |
| 实战搜索框 | debounce 防抖 → removeDuplicates 去重 → filter 过滤 → flatMap 请求 → catch 容错 |
| Combine vs async/await | Combine 擅长事件流与组合，async/await 擅长顺序异步；混合使用最佳 |

← [-SwiftUI Charts 数据可视化](./74-SwiftUI-Charts数据可视化.md) | [-需求分析与产品设计](../06-项目实战/77-需求分析与产品设计.md) →
