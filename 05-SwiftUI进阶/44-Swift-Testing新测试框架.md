# 64-Swift Testing 新测试框架

## 本章目标

- 理解 Swift Testing 是什么，为什么要用它替代 XCTest
- 掌握 `@Test`、`#expect`、`@Suite` 等核心 API
- 学会参数化测试、异步测试、Setup/Teardown
- 了解如何与 XCTest 共存并渐进式迁移
- 能够用 Swift Testing 重写现有单元测试

---

## 1. Swift Testing 简介

### 什么是 Swift Testing？

Swift Testing 是 Apple 在 WWDC24 上推出的全新测试框架，随 Xcode 16 和 Swift 6 一起发布，**最低支持 iOS 18 / macOS 15**。

你可以把它理解成 XCTest 的"下一代升级版"——就像从燃油车换到电动车，核心功能（代步/测试）没变，但操作更简单、体验更好。

### 为什么要推出新框架？

XCTest 诞生于 2013 年的 Objective-C 时代，虽然一直在用，但有几个痛点：

| 痛点 | 具体表现 |
|------|---------|
| 命名冗长 | `XCTAssertEqual`、`XCTAssertTrue`……又长又难记 |
| 必须继承 | 所有测试类必须继承 `XCTestCase` |
| setUp/tearDown 笨重 | 依赖继承和方法重写，不够灵活 |
| 参数化测试麻烦 | 需要手动写循环或用第三方库 |
| 错误信息不友好 | 断言失败时只告诉你"不等于"，不告诉你为什么 |

Swift Testing 的设计哲学：**让写测试像写普通函数一样简单**。

> 💡 **生活类比**：XCTest 就像老式翻盖手机——能打电话，但按键多、屏幕小；Swift Testing 就像智能手机——核心功能一样，但操作更直观、更现代。

---

## 2. Swift Testing vs XCTest 对比表

| 特性 | XCTest | Swift Testing |
|------|--------|---------------|
| 定义测试 | 继承 `XCTestCase`，方法名以 `test` 开头 | `@Test` 宏，函数名随意 |
| 断言 | `XCTAssertEqual`、`XCTAssertTrue` 等 20+ 个 | `#expect` 一个搞定 |
| 测试分组 | 用类继承组织 | `@Suite` 宏，支持嵌套 |
| 参数化测试 | 手动循环或第三方库 | `@Test(arguments:)` 原生支持 |
| Setup/Teardown | 重写 `setUp()` / `tearDown()` | `init()` / `deinit()` |
| 异步测试 | `await` + `XCTestExpectation` | `await` 直接用 + `Confirmation` |
| 错误断言 | `XCTAssertThrowsError` | `#expect(throws:)` 或 `#expect { } throws { }` |
| 最低系统 | iOS 2+ | iOS 18+ / macOS 15+ |
| 框架类型 | 基于类（面向对象） | 基于宏（声明式） |
| 测试发现 | 自动发现 `test` 前缀方法 | 自动发现 `@Test` 标记的函数 |

> ⚠️ **注意**：Swift Testing 要求 iOS 18+，如果你的 App 还需要支持更低版本，可以继续用 XCTest，或者两者混用（后面会讲）。

---

## 3. 定义测试：@Test 宏

### 最简单的测试

在 XCTest 中，定义一个测试需要：

```swift
// XCTest 风格
import XCTest

class MyTests: XCTestCase {
    func testAddition() {
        XCTAssertEqual(1 + 1, 2)
    }
}
```

在 Swift Testing 中，只需要：

```swift
// Swift Testing 风格
import Testing

@Test func addition() {
    #expect(1 + 1 == 2)
}
```

就这么简单！**函数 + `@Test` 宏 = 测试**。

### 关键区别

| 项目 | XCTest | Swift Testing |
|------|--------|---------------|
| 是否需要继承 | ✅ 必须继承 `XCTestCase` | ❌ 不需要 |
| 方法名前缀 | ✅ 必须以 `test` 开头 | ❌ 随意命名 |
| 所在位置 | 必须在类里 | 可以是自由函数、结构体方法、类方法 |
| 导入框架 | `import XCTest` | `import Testing` |

### @Test 的常用参数

```swift
@Test("用户登录成功") 
func loginSuccess() { ... }

@Test(.disabled("等待修复 bug #123"))
func brokenFeature() { ... }

@Test(.tags(.critical))
func importantTest() { ... }
```

| 参数 | 说明 | 示例 |
|------|------|------|
| `"描述"` | 给测试加一个可读的描述名 | `@Test("加法运算")` |
| `.disabled("原因")` | 禁用测试 | `@Test(.disabled("bug #42"))` |
| `.tags()` | 给测试打标签，方便筛选 | `@Test(.tags(.smoke))` |

> 💡 **提示**：`@Test` 标记的函数不需要 `test` 前缀，但为了可读性，建议函数名本身就能表达测试意图。

---

## 4. 断言：#expect 宏

`#expect` 是 Swift Testing 的核心断言宏，**一个顶 XCTest 的二十多个断言函数**。

### 基本用法

```swift
import Testing

@Test func basicExpectations() {
    // XCTest 的 XCTAssertEqual → Swift Testing
    #expect(1 + 1 == 2)

    // XCTest 的 XCTAssertTrue
    #expect("hello".isEmpty == false)

    // XCTest 的 XCTAssertNotNil
    let name: String? = "Swift"
    #expect(name != nil)
}
```

### 为什么 #expect 更好？

**XCTest 的问题**：你需要记住不同的断言函数名。

```swift
// XCTest：20+ 个断言函数，名字各不相同
XCTAssertEqual(a, b)
XCTAssertNotEqual(a, b)
XCTAssertTrue(condition)
XCTAssertFalse(condition)
XCTAssertNil(value)
XCTAssertNotNil(value)
XCTAssertGreaterThan(a, b)
XCTAssertLessThanOrEqual(a, b)
// ...还有更多
```

**Swift Testing 的方案**：只需 `#expect`，写自然表达式就行。

```swift
// Swift Testing：一个 #expect 搞定所有
#expect(a == b)
#expect(a != b)
#expect(condition == true)
#expect(condition == false)
#expect(value == nil)
#expect(value != nil)
#expect(a > b)
#expect(a <= b)
```

> 💡 **生活类比**：XCTest 像一个工具箱，里面有 20 把不同形状的螺丝刀；Swift Testing 像一把万能螺丝刀，一把搞定所有螺丝。

### 表达式断言的魔法

`#expect` 不只是简单地判断真假，它还能**拆解表达式**，给出详细的失败信息：

```swift
@Test func expressionDetail() {
    let a = 1
    let b = 2
    let c = 3
    #expect(a + b == c)
    // 如果失败，输出：Expectation failed: (a + b) == c
    //                → 1 + 2 ≠ 3（假设 c = 4）
}
```

XCTest 的 `XCTAssertEqual(a + b, c)` 失败时只会说"1 不等于 4"，而 `#expect` 会告诉你完整的表达式和每个变量的值。

### 错误断言

#### 断言抛出错误

```swift
// XCTest 风格
func testThrowing() {
    XCTAssertThrowsError(try riskyFunction())
}

// Swift Testing 风格
@Test func throwing() {
    #expect(throws: NetworkError.self) {
        try riskyFunction()
    }
}
```

#### 断言不抛出错误

```swift
@Test func noThrow() {
    #expect(throws: Never.self) {
        try safeFunction()
    }
}
```

#### 检查具体错误类型

```swift
@Test func specificError() {
    #expect {
        try login(password: "wrong")
    } throws: { error in
        return error is AuthError
    }
}
```

### require：断言失败立即终止

`#require` 和 `#expect` 类似，但**断言失败时会立即终止当前测试**，不再继续执行后面的代码。

```swift
@Test func requireExample() {
    let user: User? = fetchUser()
    // 如果 user 为 nil，直接终止，不会继续
    let name = #require(user != nil)
    #expect(name == "Alice")
}
```

> ⚠️ **注意**：`#require` 返回的是解包后的值，相当于同时做了断言和解包。如果条件不满足，测试会标记为失败并立即停止。

---

## 5. 测试组织：@Suite 分组

### 为什么需要分组？

当测试越来越多，所有函数散落一地就像衣服堆满房间——找起来费劲。`@Suite` 就是"衣柜"，帮你把测试分门别类。

### 基本用法

```swift
import Testing

@Suite("用户模块测试")
struct UserTests {

    @Test("注册成功")
    func registrationSuccess() {
        let user = User(name: "Alice")
        #expect(user.isRegistered == true)
    }

    @Test("注册失败 - 用户名为空")
    func registrationFailure() {
        let user = User(name: "")
        #expect(user.isRegistered == false)
    }
}
```

### 嵌套套件

```swift
@Suite("购物车模块")
struct CartTests {

    @Suite("添加商品")
    struct AddItemTests {

        @Test("添加单个商品")
        func addSingleItem() {
            let cart = Cart()
            cart.add(item: Item(name: "Book", price: 29.9))
            #expect(cart.count == 1)
        }

        @Test("添加重复商品增加数量")
        func addDuplicateItem() {
            let cart = Cart()
            cart.add(item: Item(name: "Book", price: 29.9))
            cart.add(item: Item(name: "Book", price: 29.9))
            #expect(cart.count == 2)
        }
    }

    @Suite("删除商品")
    struct RemoveItemTests {

        @Test("删除存在的商品")
        func removeExistingItem() {
            let cart = Cart()
            let item = Item(name: "Book", price: 29.9)
            cart.add(item: item)
            cart.remove(item: item)
            #expect(cart.count == 0)
        }
    }
}
```

### @Suite vs XCTestCase 对比

| 特性 | XCTestCase | @Suite |
|------|-----------|--------|
| 组织方式 | 类继承 | 结构体/类 + 宏 |
| 嵌套 | ❌ 不支持 | ✅ 支持任意嵌套 |
| 描述名 | 类名即描述 | 可自定义描述字符串 |
| 共享状态 | 通过属性共享 | 通过 init 共享（更安全） |

> 💡 **提示**：推荐用 `struct` 而不是 `class` 来定义 Suite，因为 struct 是值类型，每个测试方法都会获得独立的副本，避免测试之间的状态污染。

---

## 6. 参数化测试：@Test(arguments:)

### 什么是参数化测试？

假设你要测试一个加法函数，需要验证多组数据：

```swift
// ❌ 笨办法：写多个测试函数
@Test func add1() { #expect(add(1, 2) == 3) }
@Test func add2() { #expect(add(10, 20) == 30) }
@Test func add3() { #expect(add(-1, 1) == 0) }
```

参数化测试让你**一个测试函数，多组数据**：

```swift
// ✅ 好办法：参数化测试
@Test(arguments: [
    (1, 2, 3),
    (10, 20, 30),
    (-1, 1, 0),
    (0, 0, 0),
    (100, 200, 300),
])
func add(a: Int, b: Int, expected: Int) {
    #expect(add(a, b) == expected)
}
```

> 💡 **生活类比**：普通测试像"一题一卷"——每道题单独一张卷子；参数化测试像"一张卷子多道题"——同一套规则，不同数据。

### 单参数

```swift
@Test(arguments: ["hello", "world", "swift"])
func stringIsNotEmpty(word: String) {
    #expect(word.isEmpty == false)
}
```

### 多参数（元组）

```swift
@Test(arguments: [
    (input: "alice@example.com", isValid: true),
    (input: "invalid-email", isValid: false),
    (input: "", isValid: false),
    (input: "bob@test.org", isValid: true),
])
func emailValidation(input: String, isValid: Bool) {
    #expect(EmailValidator.isValid(input) == isValid)
}
```

### 使用 Collection

```swift
let testCases: [(String, Bool)] = [
    ("alice@example.com", true),
    ("invalid", false),
]

@Test(arguments: testCases)
func emailValidation(input: String, isValid: Bool) {
    #expect(EmailValidator.isValid(input) == isValid)
}
```

### 参数化测试在 Xcode 中的表现

每组参数会作为**独立的测试用例**出现在 Xcode 的测试导航器中，失败时能精确定位是哪组数据出了问题：

```
✅ emailValidation(input: "alice@example.com", isValid: true)
❌ emailValidation(input: "invalid-email", isValid: false) ← 失败
✅ emailValidation(input: "", isValid: false)
```

---

## 7. Setup/Teardown：init/deinit 替代

### XCTest 的方式

```swift
class MyTests: XCTestCase {
    var database: Database!

    override func setUp() {
        super.setUp()
        database = Database()
    }

    override func tearDown() {
        database.close()
        super.tearDown()
    }
}
```

### Swift Testing 的方式

```swift
@Suite("数据库测试")
struct DatabaseTests {
    let database: Database

    init() {
        database = Database()
    }

    deinit {
        database.close()
    }

    @Test func queryReturnsData() {
        #expect(database.query("SELECT 1") != nil)
    }
}
```

### 对比

| 操作 | XCTest | Swift Testing |
|------|--------|---------------|
| 每次测试前 | `override func setUp()` | `init()` |
| 每次测试后 | `override func tearDown()` | `deinit` |
| 所有测试前 | `override class func setUp()` | 结构体的静态属性 / `init()` |
| 所有测试后 | `override class func tearDown()` | 结构体的静态属性 `deinit` |
| 共享状态 | 实例属性 | 结构体属性（每个测试独立副本） |

> 💡 **关键区别**：在 Swift Testing 中，**每个 `@Test` 方法都会创建一个新的 struct 实例**，所以 `init()` 和 `deinit()` 对每个测试方法都会执行一次。这比 XCTest 的 `setUp`/`tearDown` 更安全，因为测试之间不会共享可变状态。

### 如果需要一次性 Setup

```swift
@Suite("需要共享资源的测试")
struct SharedResourceTests {
    static let sharedServer = MockServer()

    init() {
        // 每个测试前的准备
    }

    @Test func testA() { ... }
    @Test func testB() { ... }

    static let cleanup: Void = {
        // 利用懒加载模拟一次性 teardown
    }()
}
```

> ⚠️ **注意**：Swift Testing 没有直接等价于 `setUpWithError` 的 API。如果初始化可能抛出错误，可以在 `init` 中使用 `try!` 或者在测试函数内部处理。

---

## 8. 异步测试

### 直接使用 await

在 XCTest 中测试异步代码，需要用 `XCTestExpectation`，写起来很啰嗦：

```swift
// XCTest 风格 - 繁琐
func testFetchData() async {
    let expectation = XCTestExpectation(description: "数据加载完成")
    
    api.fetchData { result in
        XCTAssertEqual(result.count, 10)
        expectation.fulfill()
    }
    
    await fulfillment(of: [expectation], timeout: 5)
}
```

Swift Testing 中，**直接 `await` 就行**：

```swift
// Swift Testing 风格 - 简洁
@Test func fetchData() async {
    let result = await api.fetchData()
    #expect(result.count == 10)
}
```

就这么简单！因为 Swift Testing 原生支持 `async/await`，不需要任何额外的包装。

### Confirmation：验证异步事件发生次数

有时候你不仅要验证结果，还要验证某个事件发生了几次。这时用 `Confirmation`：

```swift
@Test func delegateCalledOnSuccess() async {
    await confirmation(expected: 1) { didReceive in
        let manager = DataManager()
        manager.onComplete = {
            didReceive()
        }
        await manager.startLoading()
    }
}
```

| 参数 | 说明 |
|------|------|
| `expected` | 期望事件发生的次数，默认 1 |
| 闭包中的 `didReceive` | 每次事件发生时调用 |

### 多次 Confirmation 示例

```swift
@Test func multipleItemsLoaded() async {
    await confirmation(expected: 3) { itemLoaded in
        let loader = ItemLoader()
        loader.onItemLoaded = { _ in
            itemLoaded()
        }
        await loader.loadAll()
    }
}
```

### Confirmation vs XCTestExpectation

| 特性 | XCTestExpectation | Confirmation |
|------|-------------------|-------------|
| 用途 | 等待异步操作完成 | 验证事件发生次数 |
| 超时设置 | 手动设置 `timeout` | 自动管理 |
| 计数验证 | `expectedFulfillmentCount` | `expected` 参数 |
| 代码量 | 较多 | 较少 |

---

## 9. 与 XCTest 共存

### 为什么要共存？

Swift Testing 要求 iOS 18+，但你的项目可能：
- 还需要支持 iOS 17 甚至更低版本
- 已有大量 XCTest 测试，不可能一次性全改
- 某些功能（如 `XCTestCase.perform()`）Swift Testing 暂无替代

所以 Apple 的态度是：**两者可以共存，渐进式迁移**。

### 共存方式

#### 同一文件混用

```swift
import XCTest
import Testing

// XCTest 测试
class LegacyTests: XCTestCase {
    func testOldWay() {
        XCTAssertEqual(1 + 1, 2)
    }
}

// Swift Testing 测试
@Test func newWay() {
    #expect(1 + 1 == 2)
}
```

#### 同一 Target 混用

在 Xcode 的测试 Target 中，两种框架的测试都会被自动发现和执行。

### 迁移策略

推荐按以下步骤渐进式迁移：

| 步骤 | 操作 | 风险 |
|------|------|------|
| 1 | 新测试用 Swift Testing 写 | 🟢 无风险 |
| 2 | 简单的断言测试先迁移 | 🟢 低风险 |
| 3 | 参数化测试迁移（收益最大） | 🟡 中风险 |
| 4 | 异步测试迁移 | 🟡 中风险 |
| 5 | 复杂的 Setup/Teardown 迁移 | 🔴 高风险，最后做 |

### 迁移示例：逐步替换

**第一步**：简单的断言测试

```swift
// 迁移前（XCTest）
class MathTests: XCTestCase {
    func testAdd() {
        XCTAssertEqual(add(1, 2), 3)
    }
    func testSubtract() {
        XCTAssertEqual(subtract(5, 3), 2)
    }
}

// 迁移后（Swift Testing）
@Suite("数学运算")
struct MathTests {
    @Test func add() {
        #expect(add(1, 2) == 3)
    }
    @Test func subtract() {
        #expect(subtract(5, 3) == 2)
    }
}
```

**第二步**：参数化测试（合并重复代码）

```swift
// 迁移前：5 个测试函数
func testAdd1() { XCTAssertEqual(add(1, 2), 3) }
func testAdd2() { XCTAssertEqual(add(0, 0), 0) }
func testAdd3() { XCTAssertEqual(add(-1, 1), 0) }
func testAdd4() { XCTAssertEqual(add(100, 200), 300) }
func testAdd5() { XCTAssertEqual(add(-5, -3), -8) }

// 迁移后：1 个参数化测试
@Test(arguments: [
    (1, 2, 3),
    (0, 0, 0),
    (-1, 1, 0),
    (100, 200, 300),
    (-5, -3, -8),
])
func add(a: Int, b: Int, expected: Int) {
    #expect(add(a, b) == expected)
}
```

> ⚠️ **注意**：XCTest 的 `XCTestCase.perform(_:)`、`addTeardownBlock(_:)` 等高级 API 在 Swift Testing 中没有直接替代，这类测试暂时不要迁移。

---

## 10. 实战示例：用 Swift Testing 重写单元测试

假设我们有一个 `PasswordValidator` 类：

```swift
struct PasswordValidator {
    enum Error: Swift.Error {
        case tooShort
        case noUppercase
        case noNumber
        case noSpecialChar
    }

    static func validate(_ password: String) throws -> Bool {
        if password.count < 8 { throw Error.tooShort }
        if !password.contains(where: { $0.isUppercase }) { throw Error.noUppercase }
        if !password.contains(where: { $0.isNumber }) { throw Error.noNumber }
        if !password.contains(where: { "!@#$%^&*".contains($0) }) { throw Error.noSpecialChar }
        return true
    }
}
```

### XCTest 版本

```swift
import XCTest

class PasswordValidatorTests: XCTestCase {
    var validator: PasswordValidator.Type!

    override func setUp() {
        super.setUp()
        validator = PasswordValidator.self
    }

    override func tearDown() {
        validator = nil
        super.tearDown()
    }

    func testValidPassword() {
        XCTAssertNoThrow(try validator.validate("Abc123!@"))
    }

    func testTooShort() {
        XCTAssertThrowsError(try validator.validate("Ab1!")) { error in
            XCTAssertEqual(error as? PasswordValidator.Error, .tooShort)
        }
    }

    func testNoUppercase() {
        XCTAssertThrowsError(try validator.validate("abc123!@")) { error in
            XCTAssertEqual(error as? PasswordValidator.Error, .noUppercase)
        }
    }

    func testNoNumber() {
        XCTAssertThrowsError(try validator.validate("Abcdef!@")) { error in
            XCTAssertEqual(error as? PasswordValidator.Error, .noNumber)
        }
    }

    func testNoSpecialChar() {
        XCTAssertThrowsError(try validator.validate("Abc12345")) { error in
            XCTAssertEqual(error as? PasswordValidator.Error, .noSpecialChar)
        }
    }
}
```

### Swift Testing 版本

```swift
import Testing

@Suite("密码验证器")
struct PasswordValidatorTests {

    @Test("有效密码通过验证")
    func validPassword() {
        #expect(try: PasswordValidator.validate("Abc123!@"))
    }

    @Test(arguments: [
        ("Ab1!", PasswordValidator.Error.tooShort),
        ("abc123!@", PasswordValidator.Error.noUppercase),
        ("Abcdef!@", PasswordValidator.Error.noNumber),
        ("Abc12345", PasswordValidator.Error.noSpecialChar),
    ])
    func invalidPassword(password: String, expectedError: PasswordValidator.Error) {
        #expect(throws: expectedError) {
            try PasswordValidator.validate(password)
        }
    }

    @Test("边界长度 - 恰好8位")
    func boundaryLength() {
        #expect(try: PasswordValidator.validate("Ab1!5678"))
    }

    @Test(arguments: [
        "", "a", "ab", "abc", "abcd", "abcde", "abcdef", "abcdefg",
    ])
    func shortPasswords(password: String) {
        #expect(throws: PasswordValidator.Error.tooShort) {
            try PasswordValidator.validate(password)
        }
    }
}
```

### 对比总结

| 维度 | XCTest 版本 | Swift Testing 版本 |
|------|------------|-------------------|
| 代码行数 | ~40 行 | ~25 行 |
| 测试函数数 | 5 个 | 3 个（参数化合并） |
| 覆盖场景 | 5 个 | 12+ 个（参数化扩展） |
| 断言可读性 | `XCTAssertThrowsError` + 闭包 | `#expect(throws:)` 一行搞定 |
| 新增测试成本 | 加一个函数 | 加一行参数 |

> 💡 **关键收益**：参数化测试让测试覆盖面大幅提升，同时代码量反而减少。新增测试场景只需添加一行参数数据，而不是复制整个测试函数。

---

## 小结

| 知识点 | 核心内容 |
|--------|---------|
| Swift Testing 简介 | iOS 18+ 新测试框架，替代 XCTest，更简洁 |
| 对比 XCTest | 更少的代码、更友好的错误信息、原生参数化支持 |
| @Test 宏 | 函数 + `@Test` = 测试，无需继承 |
| #expect 宏 | 一个宏替代 20+ 个 XCTAssert 函数 |
| @Suite | 用结构体嵌套组织测试，比类继承更灵活 |
| 参数化测试 | `@Test(arguments:)` 数据驱动，一个函数测多组数据 |
| Setup/Teardown | `init()`/`deinit()` 替代，每个测试独立实例 |
| 异步测试 | 直接 `await`，`Confirmation` 验证事件次数 |
| 共存与迁移 | 新测试用新框架，旧测试渐进式迁移，两者可混用 |
| 实战重写 | 参数化测试 + `#expect(throws:)` 大幅减少代码量 |

**一句话总结**：Swift Testing 让测试代码从"样板工程"变成"自然表达"——写什么就测什么，不用再被框架的条条框框束缚。
