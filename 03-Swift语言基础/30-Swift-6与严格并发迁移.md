# 30-Swift 6 与严格并发迁移

## 本章目标

- 理解 Swift 5 → Swift 6 的核心变化，明白严格并发检查的意义
- 掌握 Swift 6 语言模式与 Swift 5 模式的区别
- 学会识别和修复常见并发编译错误（Sendable、actor isolation、@MainActor 等）
- 掌握渐进式迁移策略，能按模块将项目从 Swift 5 迁移到 Swift 6
- 学会将全局变量、单例、回调、Delegate 等模式改造为并发安全的写法
- 了解如何处理第三方库的 Sendable 兼容性问题
- 了解 Swift 6 的其他新特性：Typed throws、noncopyable 类型、Ownership 机制
- 能够按照 checklist 完成完整的 Swift 5 → Swift 6 迁移

---

## 1. Swift 6 概述

### 1.1 从 Swift 5 到 Swift 6 的重大变化

Swift 6 是 Swift 语言自 2014 年发布以来最重要的版本之一。如果说 Swift 5 带来了 ABI 稳定性，那么 Swift 6 带来的就是**并发安全性**。

| 版本 | 核心主题 | 关键特性 |
|---|---|---|
| Swift 5.0 | ABI 稳定性 | 模块稳定性、标准库稳定 |
| Swift 5.5 | 并发引入 | async/await、Actor、Sendable |
| Swift 5.9 | 并发完善 | 宏（Macro）、noncopyable 类型 |
| Swift 6.0 | 严格并发 | 默认严格并发检查、Typed throws、Ownership |

> 💡 Swift 6 不是一个"推倒重来"的版本，而是把 Swift 5.5 以来引入的并发特性"拧紧螺丝"——从"建议你安全"变成"强制你安全"。

### 1.2 严格并发检查是什么

**严格并发检查（Strict Concurrency Checking）** 是 Swift 编译器对数据竞争（Data Race）进行静态分析的一套规则。它要求：

- 跨并发域传递的数据必须是 **Sendable** 的（即可以安全地跨线程传递）
- 可变状态必须受到 **Actor 隔离**保护
- 对共享数据的访问必须经过明确的同步机制

### 1.3 为什么严格并发如此重要

想象一个生活场景：

| | 没有并发安全 | 有并发安全 |
|---|---|---|
| **类比** | 多人同时修改同一份 Excel，互相覆盖 | 用在线协作文档，每次只允许一人编辑 |
| **编程** | 两个线程同时修改一个变量，结果不可预测 | Actor 保证同一时刻只有一个任务访问状态 |
| **后果** | 数据竞争 → 崩溃、数据损坏、难以复现的 Bug | 编译时就能发现问题，杜绝数据竞争 |

> ⚠️ 数据竞争是并发编程中最难调试的 Bug 之一。它不会每次都出现，可能在开发时正常、上线后偶发崩溃。Swift 6 的严格并发检查让你在**编译时**就能发现这些问题，而不是在用户设备上崩溃后才发现。

---

## 2. Swift 6 语言模式

### 2.1 Strict Concurrency 模式

Swift 6 默认启用严格并发检查。这意味着所有之前只是"警告"的并发问题，现在都会变成**编译错误**。

### 2.2 默认开启的检查项

| 检查项 | 说明 |
|---|---|
| **Sendable 一致性** | 跨并发域传递的类型必须遵循 Sendable 协议 |
| **Actor 隔离** | 可变状态必须在 Actor 内部访问 |
| **@MainActor 隔离** | 标记了 @MainActor 的类型/方法必须在主线程访问 |
| **nonisolated 访问** | 从非隔离上下文访问隔离状态会被禁止 |
| **全局变量的隔离** | 全局变量必须有明确的隔离域 |
| **Sendable 闭包** | 闭包捕获的值必须满足 Sendable |

### 2.3 Swift 5 模式 vs Swift 6 模式对比

| 特性 | Swift 5 模式 | Swift 6 模式 |
|---|---|---|
| 并发检查级别 | 可配置（minimal → complete） | 默认 complete，不可降低 |
| Sendable 违规 | 警告 | 编译错误 |
| Actor isolation 违规 | 警告 | 编译错误 |
| 全局变量并发安全 | 不检查 | 必须有明确隔离 |
| nonisolated 访问 | 警告 | 编译错误 |
| @preconcurrency 导入 | 不需要 | 对不合规的第三方库必需 |

> 💡 即使你暂时不升级到 Swift 6，也强烈建议在 Swift 5 模式下把 `SWIFT_STRICT_CONCURRENCY` 设为 `complete`，提前发现并修复问题。

---

## 3. 常见并发编译错误与修复

### 3.1 Sendable 不合规

**问题**：跨并发域传递了一个不满足 Sendable 的类型。

```swift
class UserSession {
    var token: String
    var userId: String

    init(token: String, userId: String) {
        self.token = token
        self.userId = userId
    }
}

func saveSession(_ session: UserSession) async {
    // ❌ 错误：UserSession 不是 Sendable，不能跨并发域传递
}
```

**修复方式一**：让类型遵循 Sendable（需要类型本身是不可变的或线程安全的）

```swift
// ✅ 修复：用 struct + let 属性，自动满足 Sendable
struct UserSession: Sendable {
    let token: String
    let userId: String
}
```

**修复方式二**：使用 @unchecked Sendable（当你确认类型是线程安全的，但编译器无法证明时）

```swift
// ✅ 修复：@unchecked Sendable——你向编译器保证线程安全
final class UserSession: @unchecked Sendable {
    private let lock = NSLock()
    private var _token: String

    var token: String {
        lock.lock()
        defer { lock.unlock() }
        return _token
    }

    init(token: String) {
        self._token = token
    }
}
```

> ⚠️ `@unchecked Sendable` 是一把双刃剑。你绕过了编译器检查，但你需要自己保证线程安全。除非你非常确定，否则优先用 struct + let 属性的方式。

### 3.2 Actor Isolation 违规

**问题**：从外部直接访问 Actor 的可变状态。

```swift
actor Counter {
    var count = 0

    func increment() -> Int {
        count += 1
        return count
    }
}

let counter = Counter()
// ❌ 错误：不能直接访问 Actor 的隔离属性
// print(counter.count)

// ✅ 修复：通过 Actor 的方法间接访问
let value = await counter.increment()
```

> 💡 Actor 就像一个银行金库——你不能直接伸手进去拿钱，必须通过柜台（Actor 的方法）来操作。`await` 就是你"排队等候"的过程。

### 3.3 Nonisolated 访问

**问题**：从非隔离上下文访问隔离状态。

```swift
@MainActor
class DataStore {
    var items: [String] = []

    func addItem(_ item: String) {
        items.append(item)
    }
}

let store = DataStore()

Task.detached {
    // ❌ 错误：Task.detached 不继承 @MainActor 隔离
    // store.addItem("hello")
}

// ✅ 修复：使用 Task（继承调用者的隔离域）或显式 await
Task.detached {
    await store.addItem("hello")
}
```

### 3.4 @MainActor 标注

**问题**：UI 相关的类型没有标注 @MainActor，导致在后台线程访问 UI。

```swift
// ❌ 错误：ViewModel 没有 @MainActor，可能在后台线程更新 @Published
class HomeViewModel: ObservableObject {
    @Published var title: String = ""

    func loadTitle() async {
        let result = await fetchTitle()
        title = result // ⚠️ 可能在后台线程修改，导致数据竞争
    }
}

// ✅ 修复：给 ViewModel 加 @MainActor
@MainActor
class HomeViewModel: ObservableObject {
    @Published var title: String = ""

    func loadTitle() async {
        let result = await fetchTitle()
        title = result // 安全：在主线程更新
    }
}
```

### 3.5 Data Race 安全检查

**问题**：多个任务同时修改同一个可变状态。

```swift
// ❌ 错误：多个 Task 同时修改共享数组
class ImageCache {
    var cache: [String: UIImage] = [:]

    func loadImage(url: String) async -> UIImage {
        if let image = cache[url] { return image }
        let image = await downloadImage(url: url)
        cache[url] = image // 数据竞争！
        return image
    }
}

// ✅ 修复：用 Actor 保护可变状态
actor ImageCache {
    private var cache: [String: UIImage] = [:]

    func loadImage(url: String) async -> UIImage {
        if let image = cache[url] { return image }
        let image = await downloadImage(url: url)
        cache[url] = image
        return image
    }
}
```

---

## 4. 迁移策略

### 4.1 渐进式迁移——不要一步到位

Swift 6 的迁移不是"开关一拨"就完成的。推荐的做法是**逐步提升并发检查级别**，每修一批问题再提升一级。

### 4.2 编译选项 SWIFT_STRICT_CONCURRENCY

在 Xcode 中，你可以通过 Build Settings 控制并发检查的严格程度：

| 级别 | 说明 | 适用阶段 |
|---|---|---|
| `minimal` | 只检查最明显的并发问题 | 刚开始迁移 |
| `targeted` | 检查标记了 Sendable/Actor 的代码 | 中期过渡 |
| `complete` | 全面严格检查，等同于 Swift 6 模式 | 迁移完成 |

```swift
// 在 Package.swift 中设置
targets: [
    .target(
        name: "MyApp",
        swiftSettings: [.unsafeFlags(["-strict-concurrency=complete"])]
    )
]
```

也可以在 Xcode Build Settings 中搜索 `Strict Concurrency Checking`，手动设置。

### 4.3 按模块迁移

| 步骤 | 操作 | 说明 |
|---|---|---|
| 1 | 选择一个模块 | 从最独立的模块开始（如工具类模块） |
| 2 | 开启 `complete` 检查 | 只对该模块开启严格检查 |
| 3 | 修复所有错误 | 逐个修复编译错误 |
| 4 | 运行测试 | 确保功能正常 |
| 5 | 重复步骤 1-4 | 逐个模块推进 |
| 6 | 全项目开启 | 所有模块完成后，全项目开启 Swift 6 模式 |

> 💡 迁移顺序建议：工具类 → 数据模型 → 网络层 → ViewModel → UI 层。越底层的代码越容易迁移，因为它们的状态通常更简单。

### 4.4 Xcode 迁移工具

Xcode 16+ 提供了内置的并发迁移辅助：

1. 打开项目，选择菜单 **Edit → Convert → To Swift 6**
2. Xcode 会自动分析项目，列出需要修改的文件
3. 你可以逐个文件审查和确认修改建议
4. 工具会自动添加 `@MainActor`、`Sendable` 等标注

> ⚠️ 自动迁移工具只是辅助，不能完全依赖。它可能会过度添加 `@MainActor` 或 `nonisolated(unsafe)`，你需要理解每处修改的含义，确保逻辑正确。

---

## 5. 常见模式改造

### 5.1 全局变量 → Actor

```swift
// ❌ 旧写法：全局可变变量，线程不安全
var appConfig: [String: String] = [:]

func updateConfig(_ key: String, _ value: String) {
    appConfig[key] = value
}

// ✅ 新写法：用 Actor 保护全局状态
actor AppConfigStore {
    static let shared = AppConfigStore()
    private var config: [String: String] = [:]

    func get(_ key: String) -> String? {
        config[key]
    }

    func set(_ key: String, _ value: String) {
        config[key] = value
    }
}

// 使用
let value = await AppConfigStore.shared.get("theme")
await AppConfigStore.shared.set("theme", "dark")
```

### 5.2 单例 → Actor

```swift
// ❌ 旧写法：class 单例，线程不安全
class UserManager {
    static let shared = UserManager()
    var currentUser: User?
    var isLoggedIn: Bool { currentUser != nil }

    private init() {}
}

// ✅ 新写法：Actor 单例，天然线程安全
actor UserManager {
    static let shared = UserManager()
    private var currentUser: User?

    var isLoggedIn: Bool { currentUser != nil }

    func setUser(_ user: User) {
        currentUser = user
    }

    func getUser() -> User? {
        currentUser
    }

    func logout() {
        currentUser = nil
    }
}
```

> 💡 Actor 单例和 class 单例的使用方式几乎一样，只是调用方法时需要 `await`。这就像从"自助餐"变成了"点餐"——你需要等服务员（Actor）帮你操作，但保证了不会有人和你抢同一盘菜。

### 5.3 回调 → async/await

```swift
// ❌ 旧写法：回调地狱
func fetchUser(id: String, completion: @escaping (Result<User, Error>) -> Void) {
    let url = URL(string: "https://api.example.com/users/\(id)")!
    URLSession.shared.dataTask(with: url) { data, response, error in
        if let error = error {
            completion(.failure(error))
            return
        }
        guard let data = data else {
            completion(.failure(NSError(domain: "NoData", code: -1)))
            return
        }
        do {
            let user = try JSONDecoder().decode(User.self, from: data)
            completion(.success(user))
        } catch {
            completion(.failure(error))
        }
    }.resume()
}

// ✅ 新写法：async/await，清晰简洁
func fetchUser(id: String) async throws -> User {
    let url = URL(string: "https://api.example.com/users/\(id)")!
    let (data, _) = try await URLSession.shared.data(from: url)
    return try JSONDecoder().decode(User.self, from: data)
}
```

### 5.4 Delegate → Actor 隔离

```swift
// ❌ 旧写法：Delegate 回调在不确定的线程上执行
protocol DataProcessorDelegate: AnyObject {
    func processorDidFinish(result: String)
}

class DataProcessor {
    weak var delegate: DataProcessorDelegate?

    func process() {
        DispatchQueue.global().async {
            let result = self.heavyComputation()
            self.delegate?.processorDidFinish(result: result) // 哪个线程？
        }
    }
}

// ✅ 新写法：Actor 隔离，明确线程安全
@MainActor
protocol DataProcessorDelegate: AnyObject {
    func processorDidFinish(result: String)
}

actor DataProcessor {
    weak var delegate: (any DataProcessorDelegate)?

    func process() async {
        let result = heavyComputation()
        await delegate?.processorDidFinish(result: result)
    }
}
```

---

## 6. 第三方库兼容性

### 6.1 Sendable 不合规的库

很多第三方库还没有适配 Swift 6 的严格并发检查。当你把它们传递到跨并发域时，会触发编译错误。

```swift
// 假设 SomeLibResponse 是第三方库的类型，不满足 Sendable
func processResponse(_ response: SomeLibResponse) async {
    // ❌ 错误：SomeLibResponse 不是 Sendable
}
```

### 6.2 @preconcurrency 桥接

`@preconcurrency` 是 Swift 提供的"过渡桥梁"，告诉编译器"我知道这个模块还没完全适配并发安全，请暂时放宽检查"。

```swift
// ✅ 使用 @preconcurrency 导入不合规的第三方库
@preconcurrency import SomeThirdPartyLib

func processResponse(_ response: SomeLibResponse) async {
    // 编译器会降低对此模块的 Sendable 检查
}
```

也可以在协议或类型上使用 `@preconcurrency`：

```swift
@preconcurrency
protocol LegacyDelegate: AnyObject {
    func didReceiveData(_ data: [String: Any])
}
```

### 6.3 封装隔离层

当第三方库既不合规又无法修改时，最佳实践是**封装一个隔离层**：

```swift
// 封装层：将不合规的库限制在 Actor 内部
actor LegacyServiceWrapper {
    private let service = LegacyService() // 不合规的第三方服务

    func fetchData() async throws -> [String: String] {
        // 在 Actor 内部调用不合规的 API
        let rawResult = service.fetch() // 返回不 Sendable 的类型

        // 转换为 Sendable 类型后返回
        return rawResult.toDictionary()
    }
}
```

> 💡 封装隔离层就像给一个没有安全认证的食品套上密封袋——你不知道里面是否卫生，但至少保证它不会"污染"其他食物。把不合规的代码限制在一个可控的范围内，是迁移过程中的务实策略。

| 兼容性策略 | 适用场景 | 风险等级 |
|---|---|---|
| `@preconcurrency import` | 库的 API 基本安全，只是缺少 Sendable 标注 | 低 |
| 封装隔离层 | 库的线程安全性不确定 | 中 |
| `nonisolated(unsafe)` | 临时绕过检查，必须确保安全 | 高 |
| 替换库 | 库已停止维护，问题严重 | 无（但迁移成本高） |

---

## 7. Swift 6 其他新特性

### 7.1 Typed throws

Swift 6 允许函数声明具体的错误类型，而不仅仅是 `throws`：

```swift
// Swift 5：不知道具体会抛出什么错误
func fetchUser() throws -> User { ... }

// Swift 6：明确声明错误类型
enum UserError: Error {
    case notFound
    case networkFailure
}

func fetchUser() throws(UserError) -> User {
    guard exists else {
        throw .notFound
    }
    guard connected else {
        throw .networkFailure
    }
    return User()
}

// 调用时可以精确处理
do {
    let user = try fetchUser()
} catch .notFound {
    print("用户不存在")
} catch .networkFailure {
    print("网络故障")
}
```

> 💡 Typed throws 就像快递包裹上写的"易碎品"标签——你提前知道可能会出什么问题，可以针对性地做准备。普通的 `throws` 则像"可能有问题，但不知道什么问题"。

### 7.2 Noncopyable 类型

Swift 6 正式引入了 noncopyable 类型（用 `~Copyable` 标记），表示该类型的值**不能被复制，只能被移动**：

```swift
struct FileHandle: ~Copyable {
    private var descriptor: Int32

    init(path: String) throws {
        descriptor = open(path, O_RDONLY)
        guard descriptor != -1 else { throw FileError.openFailed }
    }

    deinit {
        if descriptor != -1 {
            close(descriptor)
        }
    }

    consuming func read() -> Data {
        defer { descriptor = -1 }
        // 读取文件...
        return Data()
    }
}

// 使用
func processFile() throws {
    let handle = try FileHandle(path: "/tmp/data.bin")
    let data = handle.read()
    // handle 在这里已经不可用了，因为 read() 是 consuming
}
```

> 💡 Noncopyable 类型就像一张演唱会门票——你把票给了朋友，你自己就没有了。这保证了资源的唯一所有权，避免了"两个人同时操作同一个文件句柄"的问题。

### 7.3 Consuming 参数

`consuming` 关键字表示函数会"消费"掉传入的值，调用者之后不能再使用它：

```swift
func processAndClose(_ handle: consuming FileHandle) {
    // handle 的所有权转移到了这里
    // 函数结束后 handle 会被销毁
}

let file = FileHandle(path: "/tmp/data")
processAndClose(file)
// file 在这里已经不可用
```

### 7.4 Ownership 机制简介

Swift 6 引入了更细粒度的所有权控制，让你明确值的"借用"和"消费"：

| 关键字 | 所有权语义 | 类比 |
|---|---|---|
| `borrowing` | 借来用一下，用完还回去 | 借朋友的书，看完还他 |
| `consuming` | 拿过来，归我了 | 朋友把书送给你了 |
| 无标注 | 默认 copy 语义（copyable 类型） | 复印了一份，两人各有一份 |

```swift
// borrowing：只读借用，不消耗
func printLength(of text: borrowing String) {
    print(text.count)
}

// consuming：消费所有权
func send(message: consuming String) {
    // message 的所有权转移到这里
    // 调用者之后不能再使用 message
}
```

---

## 8. 迁移实战 Checklist

以下是从 Swift 5 迁移到 Swift 6 的完整步骤清单：

### 阶段一：准备工作

- [ ] 确认 Xcode 版本 ≥ 16.0
- [ ] 确认所有依赖库已更新到最新版本
- [ ] 创建迁移分支：`git checkout -b migrate/swift-6`
- [ ] 运行现有测试，确保全部通过（作为基准）
- [ ] 记录当前项目的编译警告数量

### 阶段二：开启严格并发检查

- [ ] 在 Build Settings 中将 `SWIFT_STRICT_CONCURRENCY` 设为 `complete`
- [ ] 编译项目，记录所有并发相关错误和警告
- [ ] 按模块统计错误数量，确定迁移优先级

### 阶段三：逐模块修复

- [ ] 修复 Sendable 不合规的类型（优先改为 struct + let）
- [ ] 修复 Actor isolation 违规（添加 `await` 或调整隔离域）
- [ ] 为 ViewModel / UI 相关类型添加 `@MainActor`
- [ ] 将全局可变变量改为 Actor 或 `@MainActor` 隔离
- [ ] 将线程不安全的单例改为 Actor
- [ ] 修复闭包的 Sendable 问题
- [ ] 对不合规的第三方库使用 `@preconcurrency import`
- [ ] 必要时封装隔离层处理不合规的第三方 API

### 阶段四：切换到 Swift 6 模式

- [ ] 在项目设置中将 Swift Language Version 改为 `6`
- [ ] 编译项目，修复剩余错误
- [ ] 运行全部测试，确保功能正常
- [ ] 在真机上测试，特别关注并发相关场景

### 阶段五：清理与优化

- [ ] 移除不再需要的 `@preconcurrency` 标注（如果库已更新）
- [ ] 移除临时的 `nonisolated(unsafe)` 标注
- [ ] 审查所有 `@unchecked Sendable`，确认线程安全
- [ ] 更新 CI/CD 配置，使用 Swift 6 编译
- [ ] 合并迁移分支

> ⚠️ 迁移过程中，每完成一个模块的修复，都要运行测试确认功能正常。不要攒到最后一起测试——那样出了问题很难定位原因。

---

## 本章小结

| 知识点 | 核心要点 |
|---|---|
| Swift 6 概述 | 核心变化是默认严格并发检查，杜绝数据竞争 |
| 语言模式 | Swift 6 默认 complete 检查，Sendable/Actor 违规从警告变为错误 |
| 常见错误修复 | Sendable 不合规→用 struct+let；Actor 违规→用 await；@MainActor→标注 UI 类型 |
| 迁移策略 | 渐进式迁移：按模块开启 complete 检查，从底层到上层逐步推进 |
| 模式改造 | 全局变量→Actor、单例→Actor、回调→async/await、Delegate→Actor 隔离 |
| 第三方库兼容 | @preconcurrency 桥接、封装隔离层、必要时替换库 |
| 其他新特性 | Typed throws 精确错误类型、Noncopyable 类型唯一所有权、Ownership 借用/消费机制 |
| 迁移 Checklist | 准备→开启检查→逐模块修复→切换 Swift 6→清理优化 |

> 💡 Swift 6 的严格并发检查看似"严格"，实则是"保护"。就像系安全带——刚开始觉得麻烦，但关键时刻能救命。花时间做好迁移，你的 App 将拥有更可靠的并发安全性，减少那些难以复现的线上崩溃。

← [-Swift 并发深入：Actor、Sendable 与结构化并发](./29-Swift并发深入-Actor与Sendable.md) | [-ARC 与内存管理](./31-ARC与内存管理.md) →
