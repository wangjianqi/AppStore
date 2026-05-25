# J-Swift 语法速查表

> 本速查表涵盖 Swift 5.9+ 核心语法，适用于 iOS 开发日常查阅。

---

## 1. 变量和常量声明

| 关键字 | 说明 | 示例 |
|--------|------|------|
| `var` | 可变变量 | `var count = 0` |
| `let` | 不可变常量 | `let pi = 3.14159` |
| 类型标注 | 显式指定类型 | `var name: String = "Hello"` |
| 类型推断 | 编译器自动推断 | `let age = 25` // 推断为 Int |
| 多行声明 | 同类型多变量 | `var x = 0, y = 0, z = 0` |

```swift
var score: Int = 100
let greeting: String = "你好"
var items: [String] = []
var dict: [String: Int] = [:]
var optionalName: String? = nil
```

---

## 2. 基本数据类型

### 2.1 整数与浮点数

| 类型 | 位数 | 范围 | 示例 |
|------|------|------|------|
| `Int` | 平台相关 | 64 位系统 ±9.2×10¹⁸ | `let i: Int = 42` |
| `UInt` | 平台相关 | 0 到 2×Int.max | `let u: UInt = 100` |
| `Int8/16/32/64` | 固定位数 | 对应范围 | `let byte: Int8 = 127` |
| `Double` | 64 位 | 15-17 位精度 | `let d: Double = 3.14` |
| `Float` | 32 位 | 6-9 位精度 | `let f: Float = 3.14` |

### 2.2 字符串

| 操作 | 语法 | 说明 |
|------|------|------|
| 创建 | `String()` / `"..."` | 字面量创建 |
| 拼接 | `+` / `+=` / 插值 | `s1 + s2` |
| 插值 | `\(expr)` | `"值是\(count)"` |
| 多行 | `"""..."""` | 保留换行和缩进 |
| 长度 | `count` | `"你好".count` → 2 |
| 索引 | `index(_:offsetBy:)` | 不能用整数下标 |
| 子串 | `prefix(_:)` / `suffix(_:)` | 返回 Substring |
| 判断 | `isEmpty` / `hasPrefix(_:)` / `hasSuffix(_:)` | 常用判断 |

```swift
let name = "世界"
let greeting = "你好，\(name)！"
let multiLine = """
    第一行
    第二行
    """

let index = name.index(name.startIndex, offsetBy: 1)
let char = name[index]

if greeting.hasPrefix("你好") {
    print("以你好开头")
}
```

### 2.3 集合类型

| 类型 | 语法 | 有序 | 唯一 | 示例 |
|------|------|------|------|------|
| 数组 | `[Element]` | ✅ | ❌ | `var arr = [1, 2, 3]` |
| 字典 | `[Key: Value]` | ❌ | 键唯一 | `var dict = ["a": 1]` |
| 集合 | `Set<Element>` | ❌ | ✅ | `var set: Set = [1, 2, 3]` |

```swift
// 数组
var fruits = ["苹果", "香蕉"]
fruits.append("橙子")
fruits.insert("葡萄", at: 1)
fruits.remove(at: 0)
let first = fruits.first
let last = fruits.last

// 字典
var scores = ["数学": 95, "英语": 88]
scores["语文"] = 90
scores["英语"] = nil
for (key, value) in scores {
    print("\(key): \(value)")
}

// 集合
let a: Set = [1, 2, 3, 4]
let b: Set = [3, 4, 5, 6]
let intersection = a.intersection(b)
let union = a.union(b)
let subtracting = a.subtracting(b)
```

---

## 3. 控制流

### 3.1 条件语句

| 语句 | 语法 | 说明 |
|------|------|------|
| `if-else` | `if condition { } else { }` | 条件不需要括号 |
| `switch` | `switch value { case ...: }` | 必须穷举或用 default |
| `guard` | `guard condition else { return }` | 提前退出，解包后可用 |
| 三元运算 | `condition ? a : b` | 简短条件赋值 |

```swift
let score = 85

if score >= 90 {
    print("优秀")
} else if score >= 60 {
    print("及格")
} else {
    print("不及格")
}

switch score {
case 90...100:
    print("优秀")
case 60..<90:
    print("及格")
case let s where s < 0 || s > 100:
    print("无效分数")
default:
    print("不及格")
}

guard let user = currentUser else {
    return
}
print("当前用户：\(user.name)")
```

### 3.2 循环语句

| 语句 | 语法 | 说明 |
|------|------|------|
| `for-in` | `for item in collection { }` | 遍历集合 |
| `for-in range` | `for i in 0..<10 { }` | 遍历范围 |
| `for-in stride` | `for i in stride(from:0, to:10, by:2) { }` | 按步长遍历 |
| `while` | `while condition { }` | 先判断后执行 |
| `repeat-while` | `repeat { } while condition` | 先执行后判断 |
| `forEach` | `collection.forEach { }` | 闭包遍历，不能 break |

```swift
for i in 0..<5 {
    print(i)
}

for (index, value) in array.enumerated() {
    print("第\(index)个：\(value)")
}

for i in stride(from: 10, through: 0, by: -2) {
    print(i)
}

var count = 5
while count > 0 {
    count -= 1
}
```

---

## 4. 函数和闭包

### 4.1 函数

| 特性 | 语法 | 说明 |
|------|------|------|
| 基本定义 | `func name(params) -> ReturnType { }` | 参数标签 + 参数名 |
| 参数标签 | `func greet(person name: String)` | 外部用 person，内部用 name |
| 省略标签 | `func f(_ value: Int)` | 调用时省略标签 |
| 默认值 | `func f(count: Int = 10)` | 可选参数 |
| 可变参数 | `func f(numbers: Int...)` | 传入 0 或多个值 |
| 输入输出参数 | `func f(value: inout Int)` | 可修改外部变量 |
| 函数重载 | 同名不同参数 | 返回值不同不算重载 |

```swift
func greet(person name: String, from city: String = "北京") -> String {
    return "\(name)来自\(city)"
}
greet(person: "张三", from: "上海")
greet(person: "李四")

func sum(_ numbers: Double...) -> Double {
    return numbers.reduce(0, +)
}
sum(1, 2, 3, 4)

func swapValues(_ a: inout Int, _ b: inout Int) {
    let temp = a
    a = b
    b = temp
}
var x = 1, y = 2
swapValues(&x, &y)
```

### 4.2 闭包

| 形式 | 语法 | 说明 |
|------|------|------|
| 完整闭包 | `{ (params) -> ReturnType in code }` | 最完整写法 |
| 尾随闭包 | `func(arg) { }` | 最后一个闭包参数可外置 |
| 简写参数 | `$0, $1` | 省略参数声明 |
| 单表达式 | `{ $0 + $1 }` | 自动推断返回值 |
| 逃逸闭包 | `@escaping` | 异步回调时使用 |
| 自动闭包 | `@autoclosure` | 延迟求值 |

```swift
let add: (Int, Int) -> Int = { (a, b) -> Int in
    return a + b
}

let multiply: (Int, Int) -> Int = { $0 * $1 }

let numbers = [3, 1, 4, 1, 5]
let sorted = numbers.sorted { $0 < $1 }
let doubled = numbers.map { $0 * 2 }
let evens = numbers.filter { $0 % 2 == 0 }

class ViewModel {
    var onComplete: (() -> Void)?

    func setHandler(_ handler: @escaping () -> Void) {
        onComplete = handler
    }
}
```

---

## 5. 枚举

| 特性 | 语法 | 说明 |
|------|------|------|
| 基本枚举 | `enum Name { case a, b }` | 不隐式赋值 |
| 原始值 | `enum Name: Type { case a = "val" }` | Int/String/Character |
| 关联值 | `case success(Data)` | 每个成员可携带不同类型 |
| 递归枚举 | `indirect enum` | 成员引用自身类型 |

```swift
enum Result<Value> {
    case success(Value)
    case failure(Error)
}

let result: Result<String> = .success("操作成功")
switch result {
case .success(let value):
    print(value)
case .failure(let error):
    print(error)
}

enum Planet: Int {
    case mercury = 1, venus, earth, mars
}
let earth = Planet(rawValue: 3)

indirect enum ArithmeticExpression {
    case number(Int)
    case addition(ArithmeticExpression, ArithmeticExpression)
    case multiplication(ArithmeticExpression, ArithmeticExpression)
}
```

---

## 6. 结构体和类

### 6.1 对比

| 特性 | 结构体 (struct) | 类 (class) |
|------|-----------------|------------|
| 类型 | 值类型 | 引用类型 |
| 继承 | ❌ | ✅ |
| 初始化器 | 自动生成 memberwise | 需手动写（无默认值属性时） |
| deinit | ❌ | ✅ |
| 引用比较 | `==`（需遵循 Equatable） | `===`（同一性） |
| 可变性 | 需 mutating | 直接修改 |
| 适用场景 | 数据模型、轻量对象 | 需要继承、引用语义 |

### 6.2 结构体

```swift
struct Point {
    var x: Double
    var y: Double

    var description: String {
        return "(\(x), \(y))"
    }

    mutating func moveBy(dx: Double, dy: Double) {
        x += dx
        y += dy
    }
}

var p = Point(x: 3, y: 4)
p.moveBy(dx: 1, dy: 2)
```

### 6.3 类

```swift
class Person {
    var name: String
    var age: Int

    init(name: String, age: Int) {
        self.name = name
        self.age = age
    }

    deinit {
        print("\(name) 被释放")
    }
}

class Student: Person {
    var school: String

    init(name: String, age: Int, school: String) {
        self.school = school
        super.init(name: name, age: age)
    }
}
```

### 6.4 属性类型

| 类型 | 语法 | 说明 |
|------|------|------|
| 存储属性 | `var name: Type` | 存储值 |
| 计算属性 | `var name: Type { get set }` | 动态计算 |
| 延迟属性 | `lazy var name = expr` | 首次访问时初始化 |
| 类型属性 | `static var name: Type` | 类级别共享 |
| 属性观察器 | `willSet` / `didSet` | 监听变化 |

```swift
class StepTracker {
    var totalSteps: Int = 0 {
        willSet(newSteps) {
            print("即将从\(totalSteps)变为\(newSteps)")
        }
        didSet {
            if totalSteps > oldValue {
                print("增加了\(totalSteps - oldValue)步")
            }
        }
    }

    static var version = "1.0"
}
```

---

## 7. 协议

### 7.1 定义与遵循

| 特性 | 语法 | 说明 |
|------|------|------|
| 定义 | `protocol Name { }` | 声明要求 |
| 遵循 | `class C: Proto1, Proto2 { }` | 可遵循多个 |
| 属性要求 | `var name: Type { get set }` | 指定可读/可写 |
| 方法要求 | `func method()` | 不含实现 |
| 可选方法 | `@objc optional func method()` | 仅 @objc 协议 |
| 默认实现 | `extension Proto { func method() { } }` | 协议扩展 |

```swift
protocol Drawable {
    var area: Double { get }
    func draw()
}

extension Drawable {
    func draw() {
        print("绘制图形，面积：\(area)")
    }
}

struct Circle: Drawable {
    var radius: Double
    var area: Double {
        return .pi * radius * radius
    }
}
```

### 7.2 常用标准库协议

| 协议 | 用途 | 需实现 |
|------|------|--------|
| `Equatable` | `==` 比较 | `static func ==` |
| `Comparable` | 排序比较 | `<` |
| `Hashable` | 用作 Dictionary/Set 键 | `hash(into:)` |
| `Codable` | JSON 编解码 | 通常自动合成 |
| `CustomStringConvertible` | `description` | `var description` |
| `Identifiable` | 唯一标识 | `var id` |

---

## 8. 泛型

| 特性 | 语法 | 说明 |
|------|------|------|
| 泛型函数 | `func f<T>(_ value: T)` | T 为类型参数 |
| 泛型类型 | `struct Stack<Element> { }` | 类型级别泛型 |
| 类型约束 | `<T: Protocol>` | 限制类型参数 |
| 多约束 | `<T: Proto1 & Proto2>` | 同时满足 |
| 关联类型 | `associatedtype Item` | 协议中的泛型 |
| where 子句 | `where T: Equatable` | 额外约束 |

```swift
func findIndex<T: Equatable>(of value: T, in array: [T]) -> Int? {
    for (index, item) in array.enumerated() {
        if item == value {
            return index
        }
    }
    return nil
}

struct Stack<Element> {
    private var items: [Element] = []

    mutating func push(_ item: Element) {
        items.append(item)
    }

    mutating func pop() -> Element? {
        return items.popLast()
    }

    var top: Element? {
        return items.last
    }
}

protocol Container {
    associatedtype Item
    var count: Int { get }
    subscript(i: Int) -> Item { get }
    mutating func append(_ item: Item)
}
```

---

## 9. 错误处理

| 关键字 | 说明 | 示例 |
|--------|------|------|
| `throw` | 抛出错误 | `throw NetworkError.timeout` |
| `throws` | 标记可抛出函数 | `func fetch() throws -> Data` |
| `try` | 调用可抛出函数 | `let data = try fetch()` |
| `try?` | 返回 Optional | `let data = try? fetch()` |
| `try!` | 强制解包（危险） | `let data = try! fetch()` |
| `do-catch` | 捕获错误 | `do { try ... } catch { }` |
| `rethrows` | 转发闭包抛出 | `func f(_ closure: () throws -> Void) rethrows` |

```swift
enum NetworkError: Error {
    case badURL
    case timeout
    case serverError(statusCode: Int)
}

func fetchData(from url: String) throws -> Data {
    guard !url.isEmpty else {
        throw NetworkError.badURL
    }
    return Data()
}

do {
    let data = try fetchData(from: "https://example.com")
    print("获取成功")
} catch NetworkError.badURL {
    print("URL 无效")
} catch NetworkError.timeout {
    print("请求超时")
} catch NetworkError.serverError(let code) {
    print("服务器错误：\(code)")
} catch {
    print("未知错误：\(error)")
}

let optionalData = try? fetchData(from: "https://example.com")
```

---

## 10. 并发

### 10.1 async/await

| 关键字 | 说明 | 示例 |
|--------|------|------|
| `async` | 标记异步函数 | `func load() async -> Data` |
| `await` | 等待异步结果 | `let data = await load()` |
| `async let` | 并发执行 | `async let a = loadA(); async let b = loadB()` |
| `Task` | 创建异步任务 | `Task { await load() }` |
| `TaskGroup` | 结构化并发 | `withTaskGroup(of:) { group in }` |

```swift
func fetchUser() async throws -> User {
    let (data, _) = try await URLSession.shared.data(from: url)
    return try JSONDecoder().decode(User.self, from: data)
}

Task {
    do {
        let user = try await fetchUser()
        print(user.name)
    } catch {
        print("获取失败：\(error)")
    }
}

func loadAll() async -> (User, Posts) {
    async let user = fetchUser()
    async let posts = fetchPosts()
    return await (user, posts)
}
```

### 10.2 Actor

```swift
actor DataCache {
    private var cache: [String: Data] = [:]

    func get(_ key: String) -> Data? {
        return cache[key]
    }

    func set(_ key: String, data: Data) {
        cache[key] = data
    }
}

let cache = DataCache()
Task {
    await cache.set("profile", data: profileData)
    let data = await cache.get("profile")
}
```

### 10.3 Sendable

| 概念 | 说明 |
|------|------|
| `Sendable` | 可安全跨并发域传递的类型 |
| `@Sendable` | 标记可跨并发域的闭包 |
| 自动 Sendable | 值类型、无可变状态的结构体 |
| 非 Sendable | 类（引用类型默认不安全） |

```swift
struct Config: Sendable {
    let apiKey: String
    let baseURL: String
}

func process(handler: @Sendable @escaping () -> Void) {
    Task {
        handler()
    }
}
```

### 10.4 MainActor

```swift
@MainActor
class ViewModel: ObservableObject {
    @Published var items: [Item] = []

    func load() async {
        let data = await fetchItems()
        self.items = data
    }
}
```

---

## 11. 可选值操作

| 操作 | 语法 | 说明 |
|------|------|------|
| 声明 | `Type?` | `var name: String? = nil` |
| 强制解包 | `!` | 危险，nil 时崩溃 |
| 可选绑定 | `if let` / `guard let` | 安全解包 |
| 空合运算 | `??` | 提供默认值 |
| 可选链 | `?.` | 链式安全访问 |
| 隐式解包 | `Type!` | 自动解包，慎用 |
| compactMap | `.compactMap { $0 }` | 过滤 nil |
| guard let 多值 | `guard let a = x, let b = y` | 同时解包 |

```swift
var nickname: String? = nil

if let name = nickname {
    print("昵称：\(name)")
} else {
    print("无昵称")
}

guard let name = nickname else {
    return
}

let displayName = nickname ?? "匿名用户"

class Person {
    var address: Address?
}
class Address {
    var city: String = "北京"
}

let person: Person? = Person()
let city = person?.address?.city

let mixed: [String?] = ["a", nil, "b", nil, "c"]
let valid = mixed.compactMap { $0 }
```

---

## 12. 扩展和下标

### 12.1 扩展

| 用途 | 语法 | 说明 |
|------|------|------|
| 添加方法 | `extension Type { func ... }` | 不影响原类型 |
| 添加计算属性 | `extension Type { var ... { get } }` | 不能添加存储属性 |
| 遵循协议 | `extension Type: Protocol { }` | 隔离协议实现 |
| 条件扩展 | `extension Type where ... { }` | 有约束的扩展 |

```swift
extension Int {
    var isEven: Bool {
        return self % 2 == 0
    }

    func times(_ action: () -> Void) {
        for _ in 0..<self {
            action()
        }
    }
}

3.isEven
5.times { print("Hello") }

extension Array where Element: Numeric {
    var sum: Element {
        return reduce(0, +)
    }
}
```

### 12.2 下标

```swift
struct Matrix {
    let rows: Int, columns: Int
    private var grid: [Double]

    init(rows: Int, columns: Int) {
        self.rows = rows
        self.columns = columns
        grid = Array(repeating: 0, count: rows * columns)
    }

    subscript(row: Int, column: Int) -> Double {
        get {
            return grid[row * columns + column]
        }
        set {
            grid[row * columns + column] = newValue
        }
    }
}

var matrix = Matrix(rows: 2, columns: 2)
matrix[0, 1] = 1.5
print(matrix[0, 1])
```

---

## 13. 常用标准库函数速查

| 函数 | 用途 | 示例 |
|------|------|------|
| `map` | 转换每个元素 | `[1,2,3].map { $0 * 2 }` |
| `filter` | 过滤元素 | `[1,2,3].filter { $0 > 1 }` |
| `reduce` | 合并为单值 | `[1,2,3].reduce(0, +)` |
| `compactMap` | 转换并过滤 nil | `[1,nil,3].compactMap { $0 }` |
| `flatMap` | 展平嵌套 | `[[1,2],[3]].flatMap { $0 }` |
| `sorted` | 排序 | `[3,1,2].sorted()` |
| `forEach` | 遍历执行 | `[1,2].forEach { print($0) }` |
| `contains` | 是否包含 | `[1,2,3].contains(2)` |
| `first(where:)` | 查找首个 | `[1,2,3].first { $0 > 1 }` |
| `zip` | 合并两个序列 | `zip([1,2], ["a","b"])` |
| `enumerated` | 带索引遍历 | `arr.enumerated()` |

---

## 14. 访问控制

| 关键字 | 范围 | 说明 |
|--------|------|------|
| `private` | 当前作用域 | 最严格 |
| `fileprivate` | 当前文件 | 文件内可见 |
| `internal` | 当前模块 | 默认级别 |
| `public` | 模块内外 | 可访问不可继承 |
| `open` | 模块内外 | 可访问可继承 |

---

> 💡提示：Swift 语法持续演进，建议关注 [Swift Evolution](https://github.com/apple/swift-evolution) 获取最新提案。
