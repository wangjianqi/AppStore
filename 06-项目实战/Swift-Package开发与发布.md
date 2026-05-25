# Swift Package 开发与发布

## 本章目标

- 理解开发 Swift Package 的核心动机：代码复用、模块化、开源贡献与团队共享
- 掌握从零创建 Swift Package 的完整流程，包括目录结构与 Package.swift 配置
- 学会 Package 设计原则：API 设计、访问控制、语义化版本与向后兼容策略
- 掌握资源文件处理：.process() / .copy()、本地化资源与 Asset Catalog
- 能够为 Package 编写单元测试并生成 DocC 文档
- 掌握发布流程：Git Tag 管理、GitHub Release、Swift Package Index 注册
- 了解 SwiftPM Plugin 开发：Command Plugin 与 Build Tool Plugin
- 掌握 Binary Framework 的创建与闭源分发方式

---

## 1. 为什么开发自己的 Package

### 1.1 四大核心动机

想象你是一个厨师——如果每次做菜都要从种菜开始，一天只能做一顿饭。把常用调料提前配好、装瓶贴标签，随时取用，效率就会大幅提升。Swift Package 就是你的"调料瓶"——把通用代码封装好，随取随用。

| 动机 | 说明 | 生活类比 |
|------|------|---------|
| 代码复用 | 多个项目共享同一套工具代码，避免复制粘贴 | 常用调料瓶，哪个厨房都能用 |
| 模块化 | 将大项目拆分为独立模块，职责清晰、编译更快 | 衣柜分格，上衣裤子各归其位 |
| 开源贡献 | 将优秀方案分享给社区，建立技术影响力 | 拿手菜谱公开，收获食客好评 |
| 团队共享 | 企业内部统一基础组件，保证一致性与质量 | 连锁餐厅统一供应链，口味一致 |

### 1.2 何时应该抽取 Package

并非所有代码都适合抽取为独立 Package。判断标准如下：

| 适合抽取 | 不适合抽取 |
|---------|-----------|
| 被多个项目使用的工具类 | 仅当前项目使用的业务逻辑 |
| 通用 UI 组件（按钮、弹窗） | 与特定后端 API 强耦合的模型 |
| 网络请求封装、日志框架 | 包含硬编码配置的代码 |
| 加密算法、数据格式转换 | 依赖大量项目特定资源文件的代码 |

> 💡 一个好的判断标准：如果这段代码脱离当前项目，是否还能独立工作？如果能，它就适合抽取为 Package。

---

## 2. 创建 Swift Package

### 2.1 使用命令行创建

创建 Swift Package 就像开一家新店——先办营业执照（init），再装修店面（目录结构），最后上架商品（编写代码）。

```bash
# 创建一个名为 NetworkKit 的库类型 Package
mkdir NetworkKit && cd NetworkKit
swift package init --name NetworkKit --type library

# 创建一个可执行类型的 Package
swift package init --name MyCLI --type executable
```

`--type` 参数选项：

| 类型 | 说明 | 适用场景 |
|------|------|---------|
| `library` | 生成库项目（默认） | 供其他项目引用的代码 |
| `executable` | 生成可执行项目 | 命令行工具 |
| `empty` | 生成空项目 | 完全自定义结构 |

### 2.2 目录结构详解

创建后的标准目录结构：

```
NetworkKit/
├── Package.swift              # 核心配置文件
├── Sources/
│   └── NetworkKit/            # 源码目录（与 Package 名一致）
│       └── NetworkKit.swift   # 默认生成的入口文件
├── Tests/
│   └── NetworkKitTests/       # 测试目录
│       └── NetworkKitTests.swift
└── README.md
```

更完善的 Package 目录结构：

```
NetworkKit/
├── Package.swift
├── Sources/
│   └── NetworkKit/
│       ├── NetworkKit.swift           # 公开 API 入口
│       ├── HTTPClient.swift           # 核心网络客户端
│       ├── Request/
│       │   ├── APIRequest.swift
│       │   └── RequestInterceptor.swift
│       ├── Response/
│       │   ├── APIResponse.swift
│       │   └── APIError.swift
│       └── Resources/                 # 资源文件目录
│           ├── PrivacyInfo.xcprivacy
│           └── Localization/
├── Tests/
│   └── NetworkKitTests/
│       ├── HTTPClientTests.swift
│       └── RequestInterceptorTests.swift
├── Documentation/                     # DocC 文档目录
│   └── NetworkKit.docc/
├── .spi.yml                           # Swift Package Index 配置
└── README.md
```

### 2.3 Package.swift 完整配置

Package.swift 是整个 Package 的"身份证"——名称、版本、依赖、产物，一目了然。

```swift
// swift-tools-version:5.9
import PackageDescription

let package = Package(
    name: "NetworkKit",
    platforms: [
        .iOS(.v15),
        .macOS(.v12),
        .tvOS(.v15),
        .watchOS(.v8),
        .visionOS(.v1)
    ],
    products: [
        .library(
            name: "NetworkKit",
            targets: ["NetworkKit"]
        ),
        .library(
            name: "NetworkKit-Dynamic",
            type: .dynamic,
            targets: ["NetworkKit"]
        )
    ],
    dependencies: [
        .package(url: "https://github.com/apple/swift-log.git", from: "1.5.0"),
        .package(url: "https://github.com/apple/swift-async-algorithms.git", from: "1.0.0")
    ],
    targets: [
        .target(
            name: "NetworkKit",
            dependencies: [
                .product(name: "Logging", package: "swift-log"),
                .product(name: "AsyncAlgorithms", package: "swift-async-algorithms")
            ],
            resources: [
                .process("Resources/Localization"),
                .copy("Resources/PrivacyInfo.xcprivacy")
            ],
            swiftSettings: [
                .unsafeFlags(["-warnings-as-errors"], .when(configuration: .release))
            ]
        ),
        .testTarget(
            name: "NetworkKitTests",
            dependencies: ["NetworkKit"],
            resources: [
                .copy("Fixtures/mock_response.json")
            ]
        )
    ],
    swiftLanguageVersions: [.v5, .v6]
)
```

关键字段解读：

| 字段 | 作用 | 必填 |
|------|------|------|
| `name` | Package 名称，也是默认的模块名 | ✅ |
| `platforms` | 最低支持平台版本 | ❌（但强烈建议填写） |
| `products` | 对外暴露的产物（静态库/动态库） | ✅ |
| `dependencies` | 外部依赖声明 | ❌ |
| `targets` | 编译目标（源码/测试/二进制） | ✅ |
| `swiftLanguageVersions` | 支持的 Swift 语言版本 | ❌ |

### 2.4 Library Target 配置

SPM 支持两种库类型：

```swift
products: [
    .library(
        name: "NetworkKit",
        targets: ["NetworkKit"]
    ),
    .library(
        name: "NetworkKitDynamic",
        type: .dynamic,
        targets: ["NetworkKit"]
    )
]
```

| 类型 | 说明 | 适用场景 |
|------|------|---------|
| 静态库（默认） | 编译时链接到可执行文件，体积略大但启动快 | 大多数场景首选 |
| 动态库 | 运行时加载，多 Target 共享同一份二进制 | 插件化架构、多 Extension 共享 |

> ⚠️ 不指定 `type` 时默认为静态库。App Store 对动态库数量有限制，且动态库会增加启动时间，除非有明确需求，否则优先使用静态库。

---

## 3. Package 设计原则

### 3.1 API 设计

好的 API 设计就像好的遥控器——按键少、直觉强，不用看说明书就能用。

**核心原则：**

| 原则 | 说明 | 示例 |
|------|------|------|
| 最小接口 | 只暴露必要的 API，隐藏实现细节 | 只公开 `send()` 而非内部的 `buildRequest()` |
| 命名清晰 | 名称应自解释，避免缩写 | `fetchUserProfile()` 而非 `fup()` |
| 合理默认值 | 常见场景零配置，高级场景可定制 | `timeout: TimeInterval = 30` |
| 类型安全 | 用类型系统防止错误 | 用 `enum HTTPMethod` 而非 `String` |
| 单一职责 | 一个类型只做一件事 | `RequestBuilder` 只负责构建请求 |

```swift
public final class HTTPClient {
    private let session: URLSession
    private let interceptors: [RequestInterceptor]

    public init(
        session: URLSession = .shared,
        interceptors: [RequestInterceptor] = []
    ) {
        self.session = session
        self.interceptors = interceptors
    }

    public func send<T: Decodable>(
        _ request: APIRequest<T>,
        timeout: TimeInterval = 30
    ) async throws -> T {
        var urlRequest = try request.buildURLRequest()
        urlRequest.timeoutInterval = timeout

        for interceptor in interceptors {
            urlRequest = try interceptor.intercept(urlRequest)
        }

        let (data, response) = try await session.data(for: urlRequest)

        guard let httpResponse = response as? HTTPURLResponse else {
            throw APIError.invalidResponse
        }

        guard (200...299).contains(httpResponse.statusCode) else {
            throw APIError.statusCode(httpResponse.statusCode, data)
        }

        return try request.decode(data)
    }
}
```

### 3.2 访问控制

访问控制是 Package 的"门禁系统"——决定谁能进哪个房间。

| 修饰符 | 作用域 | Package 中的使用场景 |
|--------|--------|---------------------|
| `open` | 同模块 + 子类可继承/重写 | 框架需要被扩展的类 |
| `public` | 同模块可见，不可继承/重写 | 对外暴露的 API |
| `internal` | 同模块可见（默认） | 模块内部实现 |
| `fileprivate` | 同文件可见 | 辅助类型、私有嵌套 |
| `private` | 同作用域可见 | 实现细节 |

```swift
public final class NetworkKit {
    public static let shared = NetworkKit()

    public var baseURL: URL

    internal let session: URLSession
    internal var interceptors: [RequestInterceptor]

    private var retryCount: Int = 0

    public init(baseURL: URL) {
        self.baseURL = baseURL
        self.session = URLSession(configuration: .default)
        self.interceptors = []
    }

    public func addInterceptor(_ interceptor: RequestInterceptor) {
        interceptors.append(interceptor)
    }
}
```

> 💡 经验法则：默认使用 `internal`（不写修饰符），只有确定需要对外暴露的才标记 `public`，需要被继承的才用 `open`。暴露越少，未来修改的自由度越大。

### 3.3 语义化版本（SemVer）

语义化版本就像产品的保质期标签——一眼看出兼容性。

格式：`MAJOR.MINOR.PATCH`（如 `2.4.1`）

| 版本号 | 变更含义 | 对用户的影响 |
|--------|---------|-------------|
| MAJOR | 不兼容的 API 变更 | 必须修改代码才能升级 |
| MINOR | 向后兼容的功能新增 | 可选升级，无需改代码 |
| PATCH | 向后兼容的 Bug 修复 | 建议升级，无破坏性 |

```swift
// Package.swift 中的版本规则与 SemVer 对应
.package(url: "...", from: "1.0.0")   // 允许 1.x.x，不升到 2.0.0
.package(url: "...", from: "1.2.0")   // 允许 1.2.x ~ 1.x.x
.package(url: "...", .exact("1.2.3")) // 仅 1.2.3
```

### 3.4 向后兼容策略

修改 API 就像翻修公路——不能封路，要保证旧车还能跑。

| 变更类型 | 兼容性 | 做法 |
|---------|--------|------|
| 新增方法 | ✅ 兼容 | 直接添加 |
| 新增参数 | ✅ 兼容 | 给默认值 |
| 重命名方法 | ❌ 不兼容 | 旧方法标记 `@available(*, deprecated)` |
| 删除方法 | ❌ 不兼容 | 先废弃一个版本，再在下一个 MAJOR 删除 |
| 修改返回类型 | ❌ 不兼容 | 提供新方法，旧方法保留 |

```swift
public struct APIRequest<T: Decodable> {
    public let path: String
    public let method: HTTPMethod
    public let parameters: [String: Any]?

    @available(*, deprecated, renamed: "init(path:method:parameters:headers:)")
    public init(
        path: String,
        method: HTTPMethod,
        parameters: [String: Any]? = nil
    ) {
        self.init(path: path, method: method, parameters: parameters, headers: nil)
    }

    public init(
        path: String,
        method: HTTPMethod,
        parameters: [String: Any]? = nil,
        headers: [String: String]? = nil
    ) {
        self.path = path
        self.method = method
        self.parameters = parameters
        self.headers = headers
    }
}
```

> ⚠️ 永远不要在 MINOR 或 PATCH 版本中引入破坏性变更。如果必须破坏兼容性，请递增 MAJOR 版本号，并在 CHANGELOG 中明确说明迁移路径。

---

## 4. 资源文件处理

### 4.1 .process() 与 .copy()

SPM 5.3+ 支持在 Package 中包含资源文件。两种处理方式就像"精加工"和"原样发货"——前者会优化处理，后者原封不动。

```swift
.target(
    name: "UIComponents",
    dependencies: [],
    resources: [
        .process("Assets.xcassets"),
        .process("Localization"),
        .copy("Config.json"),
        .copy("PrivacyInfo.xcprivacy")
    ]
)
```

| 处理方式 | `.process()` | `.copy()` |
|---------|-------------|-----------|
| 行为 | SPM 自动优化（编译 Asset Catalog、处理 lproj） | 原样拷贝到 bundle |
| 适用资源 | 图片、Asset Catalog、本地化字符串 | JSON、XML、配置文件、隐私清单 |
| 目录结构 | 可能重组 | 保留原始目录结构 |
| 推荐度 | 优先使用 | 仅在需要保留原样时使用 |

在代码中访问资源：

```swift
import Foundation

public enum ThemeConfig {
    public static func loadConfig() throws -> [String: String] {
        guard let url = Bundle.module.url(forResource: "Config", withExtension: "json") else {
            throw ConfigError.fileNotFound
        }
        let data = try Data(contentsOf: url)
        return try JSONDecoder().decode([String: String].self, from: data)
    }
}
```

### 4.2 本地化资源

Package 中的本地化就像多语言菜单——同一道菜，不同语言各有描述。

目录结构：

```
Sources/UIComponents/
└── Resources/
    └── Localization/
        ├── en.lproj/
        │   └── Localizable.strings
        ├── zh-Hans.lproj/
        │   └── Localizable.strings
        └── ja.lproj/
            └── Localizable.strings
```

Package.swift 配置：

```swift
.target(
    name: "UIComponents",
    dependencies: [],
    resources: [
        .process("Resources/Localization")
    ]
)
```

代码中使用：

```swift
public extension String {
    static func localized(_ key: String, bundle: Bundle = .module) -> String {
        return NSLocalizedString(key, bundle: bundle, comment: "")
    }
}

// 使用
let title = String.localized("ui_components.save_button")
```

### 4.3 Asset Catalog 在 Package 中使用

```swift
.target(
    name: "UIComponents",
    dependencies: [],
    resources: [
        .process("Resources/Assets.xcassets")
    ]
)
```

```swift
import UIKit

public extension UIImage {
    static func packageImage(named name: String) -> UIImage? {
        return UIImage(named: name, in: .module, compatibleWith: nil)
    }
}

// 使用
let icon = UIImage.packageImage(named: "arrow_right")
```

> ⚠️ 在 Package 中访问图片和颜色资源，必须使用 `Bundle.module` 而非 `Bundle.main`。SPM 会为每个 target 生成独立的资源 bundle，运行时与主 App 的 bundle 是分离的。

---

## 5. 测试与文档

### 5.1 Unit Test Target 配置

测试就像出厂质检——每个功能都要经过检验才能交付。

```swift
targets: [
    .target(
        name: "NetworkKit",
        dependencies: [
            .product(name: "Logging", package: "swift-log")
        ]
    ),
    .testTarget(
        name: "NetworkKitTests",
        dependencies: ["NetworkKit"],
        resources: [
            .copy("Fixtures/mock_users.json"),
            .copy("Fixtures/mock_error.json")
        ]
    )
]
```

编写测试：

```swift
import XCTest
@testable import NetworkKit

final class HTTPClientTests: XCTestCase {

    var client: HTTPClient!
    var mockSession: MockURLSession!

    override func setUp() {
        mockSession = MockURLSession()
        client = HTTPClient(session: mockSession)
    }

    override func tearDown() {
        client = nil
        mockSession = nil
    }

    func testSendSuccess() async throws {
        let mockData = """
        {"id": 1, "name": "Test User"}
        """.data(using: .utf8)!

        mockSession.mockData = mockData
        mockSession.mockResponse = HTTPURLResponse(
            url: URL(string: "https://api.example.com/users/1")!,
            statusCode: 200,
            httpVersion: nil,
            headerFields: nil
        )!

        let request = APIRequest<User>(path: "/users/1", method: .get)
        let result = try await client.send(request)

        XCTAssertEqual(result.id, 1)
        XCTAssertEqual(result.name, "Test User")
    }

    func testSendFailureStatusCode() async {
        mockSession.mockResponse = HTTPURLResponse(
            url: URL(string: "https://api.example.com/users/1")!,
            statusCode: 404,
            httpVersion: nil,
            headerFields: nil
        )!

        let request = APIRequest<User>(path: "/users/1", method: .get)

        do {
            _ = try await client.send(request)
            XCTFail("Should throw error")
        } catch APIError.statusCode(let code, _) {
            XCTAssertEqual(code, 404)
        } catch {
            XCTFail("Unexpected error: \(error)")
        }
    }
}
```

### 5.2 swift test 命令

```bash
# 运行所有测试
swift test

# 运行特定测试
swift test --filter HTTPClientTests

# 使用 verbose 输出
swift test -v

# 指定 Swift 版本
swift test --swift-version 6

# 并行执行测试（加速）
swift test --parallel

# 生成代码覆盖率报告
swift test --enable-code-coverage
```

| 命令 | 用途 |
|------|------|
| `swift test` | 运行全部测试 |
| `swift test --filter` | 运行匹配的测试 |
| `swift test --parallel` | 并行运行测试 |
| `swift test --enable-code-coverage` | 生成覆盖率 |
| `swift build` | 仅编译不测试 |
| `swift build --build-tests` | 编译测试代码但不运行 |

### 5.3 DocC 文档生成与发布

DocC 是 Apple 的文档编译器，就像出版社的排版系统——把你的注释变成精美的技术文档。

目录结构：

```
Documentation/
└── NetworkKit.docc/
    ├── NetworkKit.md          # 根目录页
    ├── HTTPClient.md          # 类型文档
    ├── APIRequest.md          # 类型文档
    └── Resources/
        └── tutorial-hero.png  # 教程插图
```

源码中的文档注释：

```swift
/// A type that represents an API request with typed response.
///
/// Use ``APIRequest`` to define type-safe network requests. Each request
/// specifies a path, HTTP method, and expected response type.
///
/// ```swift
/// let request = APIRequest<User>(path: "/users/1", method: .get)
/// let user = try await client.send(request)
/// ```
///
/// - Note: The generic type `T` must conform to ``Decodable``.
public struct APIRequest<T: Decodable> {

    /// The endpoint path relative to the base URL.
    public let path: String

    /// The HTTP method for the request.
    public let method: HTTPMethod

    /// Creates a new API request.
    /// - Parameters:
    ///   - path: The endpoint path (e.g., "/users/1").
    ///   - method: The HTTP method. Defaults to `.get`.
    ///   - parameters: Optional query or body parameters.
    ///   - headers: Optional custom HTTP headers.
    public init(
        path: String,
        method: HTTPMethod = .get,
        parameters: [String: Any]? = nil,
        headers: [String: String]? = nil
    ) {
        self.path = path
        self.method = method
        self.parameters = parameters
        self.headers = headers
    }
}
```

构建与预览文档：

```bash
# 构建 DocC 文档
swift package generate-documentation

# 启动本地预览服务器
swift package --disable-sandbox preview-documentation --target NetworkKit
```

发布到 GitHub Pages：

```bash
# 生成静态站点
$(xcrun --find docc) process-archive \
    transform-for-static-hosting \
    ~/Library/Developer/Xcode/DerivedData/.../NetworkKit.doccarchive \
    --output-path ./docs \
    --hosting-base-path NetworkKit
```

> 💡 在 GitHub Actions 中可以自动化文档发布：每次推送到 main 分支时自动构建并部署到 GitHub Pages。

---

## 6. 发布流程

### 6.1 Git Tag 版本管理

Git Tag 就像产品的出厂编号——每个版本都有唯一标识，可追溯、可回滚。

```bash
# 创建带注释的版本标签
git tag -a 1.0.0 -m "Release 1.0.0: Initial stable release"

# 推送标签到远程
git push origin 1.0.0

# 推送所有标签
git push origin --tags

# 查看所有标签
git tag -l

# 删除错误标签
git tag -d 1.0.0
git push origin :refs/tags/1.0.0
```

版本发布流程：

```
1. 确保 main 分支所有测试通过
2. 更新 CHANGELOG.md
3. 提交变更：git commit -m "chore: prepare for 1.0.0 release"
4. 打标签：git tag -a 1.0.0 -m "Release 1.0.0"
5. 推送：git push origin main --tags
6. 在 GitHub 创建 Release
```

### 6.2 GitHub Release

GitHub Release 是发布 Package 的标准方式，就像商品上架——附上说明、标明版本、提供下载。

```bash
# 使用 GitHub CLI 创建 Release
gh release create 1.0.0 \
    --title "NetworkKit 1.0.0" \
    --notes "## What's New

- Initial stable release
- HTTPClient with async/await support
- Request interceptor pipeline
- Built-in retry mechanism

### Breaking Changes
None (first release)

### Migration Guide
N/A"
```

Release Notes 最佳实践：

| 内容 | 说明 |
|------|------|
| 新功能列表 | 本次新增了什么 |
| Bug 修复列表 | 修复了哪些问题 |
| 破坏性变更 | 升级需要改什么代码 |
| 迁移指南 | 如何从上一版本升级 |
| 贡献者 | 感谢本次版本的贡献者 |

### 6.3 Swift Package Index 注册

[Swift Package Index](https://swiftpackageindex.com/) 是 Swift 社区的包搜索引擎，就像 Swift 世界的"淘宝"——用户在这里搜索、比较、选择 Package。

注册步骤：

1. 确保仓库是公开的 GitHub 仓库
2. 确保仓库根目录有有效的 `Package.swift`
3. 访问 [swiftpackageindex.com/add](https://swiftpackageindex.com/add) 提交仓库 URL
4. 等待自动索引（通常几分钟内完成）

创建 `.spi.yml` 配置文件：

```yaml
version: 1
builder:
  configs:
    - documentation_targets:
        - NetworkKit
      platform: ios
```

SPI 会自动检测：

| 信息 | 来源 |
|------|------|
| 支持平台 | Package.swift 中的 `platforms` |
| Swift 版本 | `swiftLanguageVersions` |
| 依赖关系 | `dependencies` |
| 许可证 | 仓库根目录的 LICENSE 文件 |
| 文档 | DocC 构建 |

> 💡 Swift Package Index 是开发者发现 Package 的主要渠道。注册后，你的 Package 会出现在搜索结果中，并自动构建兼容性矩阵，帮助用户判断是否适合他们的项目。

### 6.4 Xcode Package Registry

Xcode 16+ 支持 Package Registry，这是 Apple 官方的包注册中心，就像 Apple 自己开的"App Store for Code"。

```bash
# 配置默认 Registry
swift package-registry set github://token

# 发布到 Registry
swift package-registry publish NetworkKit 1.0.0

# 从 Registry 添加依赖
.package(id: "github.com.your-org.NetworkKit", from: "1.0.0")
```

| 特性 | Git 依赖（传统方式） | Package Registry |
|------|---------------------|-----------------|
| 解析速度 | 需要克隆仓库元数据 | 直接查询注册中心，更快 |
| 版本发现 | 遍历 Git Tag | 注册中心索引，精确 |
| 安全性 | 依赖 Git 平台 | 支持签名验证 |
| 可用性 | 所有 Git 仓库 | 需要发布到 Registry |

---

## 7. SwiftPM Plugin 开发

### 7.1 Plugin 类型概述

Plugin 就像给 Xcode 装上的"外挂工具"——不用改项目配置，就能在构建流程中插入自定义操作。

| 类型 | 执行时机 | 典型用途 |
|------|---------|---------|
| Command Plugin | 用户手动触发 | 代码生成、格式化、Lint |
| Build Tool Plugin | 每次构建自动执行 | 代码生成、资源处理、编译检查 |

### 7.2 Command Plugin 开发

Command Plugin 就像厨房里的"按需工具"——厨师需要时才拿出来用。

Package.swift：

```swift
// swift-tools-version:5.9
import PackageDescription

let package = Package(
    name: "SwiftFormatPlugin",
    products: [
        .plugin(
            name: "SwiftFormatPlugin",
            targets: ["SwiftFormatPlugin"]
        )
    ],
    dependencies: [
        .package(url: "https://github.com/nicklockwood/SwiftFormat.git", from: "0.52.0")
    ],
    targets: [
        .plugin(
            name: "SwiftFormatPlugin",
            capability: .command(
                intent: .sourceCodeFormatting(),
                permissions: [
                    .writeToPackageDirectory(reason: "SwiftFormat needs to write formatted source files")
                ]
            ),
            dependencies: [
                .product(name: "swiftformat", package: "SwiftFormat")
            ]
        )
    ]
)
```

Plugin 实现：

```swift
import PackagePlugin
import Foundation

@main
struct SwiftFormatPlugin: CommandPlugin {
    func performCommand(
        context: PluginContext,
        arguments: [String]
    ) async throws {
        let swiftFormatTool = try context.tool(named: "swiftformat")
        let swiftFormatPath = swiftFormatTool.path.string

        let sourceFiles = context.package.targets
            .filter { $0.kind == .source }
            .flatMap { target in
                target.sourceFiles.map { $0.path.string }
            }

        guard !sourceFiles.isEmpty else {
            print("No source files found to format.")
            return
        }

        let process = Process()
        process.executableURL = URL(fileURLWithPath: swiftFormatPath)
        process.arguments = sourceFiles + arguments

        try process.run()
        process.waitUntilExit()

        if process.terminationStatus != 0 {
            throw PluginError.formattingFailed
        }
    }
}

enum PluginError: Error {
    case formattingFailed
}
```

### 7.3 Build Tool Plugin 开发

Build Tool Plugin 就像工厂流水线上的"自动检测仪"——每次生产都自动运行，无需人工干预。

```swift
.target(
    name: "MyApp",
    dependencies: [],
    plugins: [
        "SwiftLintPlugin"
    ]
)
```

Plugin 实现：

```swift
import PackagePlugin
import Foundation

@main
struct SwiftLintPlugin: BuildToolPlugin {
    func createBuildCommands(
        context: PluginContext,
        target: Target
    ) async throws -> [Command] {
        let swiftLintPath = try context.tool(named: "swiftlint").path

        let sourceFiles = target.sourceFiles
            .filter { $0.path.extension == "swift" }
            .map { $0.path.string }

        guard !sourceFiles.isEmpty else { return [] }

        return [
            .prebuildCommand(
                displayName: "Running SwiftLint on \(target.name)",
                executable: swiftLintPath,
                arguments: ["lint", "--path", target.directory.string] + sourceFiles,
                outputFilesDirectory: context.pluginWorkDirectory
            )
        ]
    }
}
```

### 7.4 Plugin 与 Target 交互

| 交互方式 | Command Plugin | Build Tool Plugin |
|---------|---------------|------------------|
| 读取源文件 | ✅ `context.package.targets` | ✅ `target.sourceFiles` |
| 写入文件 | ✅ 需声明权限 | ✅ 输出到 `pluginWorkDirectory` |
| 访问外部工具 | ✅ `context.tool(named:)` | ✅ `context.tool(named:)` |
| 访问网络 | ❌ 沙箱限制 | ❌ 沙箱限制 |
| 修改源码 | ✅ 需声明权限 | ❌ 仅生成新文件 |
| 执行时机 | 用户手动 | 构建时自动 |

> ⚠️ Plugin 运行在沙箱环境中，无法访问网络和大部分文件系统。如果需要网络访问，必须使用 `--disable-sandbox` 参数，但这会降低安全性。

---

## 8. Binary Framework

### 8.1 为什么需要 Binary Framework

并非所有代码都适合开源——就像可口可乐的配方，核心秘密不能公开，但产品还是要卖。

| 场景 | 说明 |
|------|------|
| 闭源 SDK | 付费 SDK、商业机密，不希望暴露源码 |
| 遗留代码 | Objective-C 或 C++ 代码，不适合以源码形式分发 |
| 构建加速 | 预编译二进制，使用者无需编译，加快集成速度 |
| 跨语言 | 包含 C/C++ 代码，Swift Package 无法直接管理 |

### 8.2 XCFramework 创建

XCFramework 就像"万能插头"——一个包装里包含所有平台的版本，系统自动选择合适的。

```bash
# 第一步：分别编译各平台的 framework

# iOS 设备（arm64）
xcodebuild archive \
    -scheme NetworkKit \
    -destination "generic/platform=iOS" \
    -archivePath ./build/ios.xcarchive \
    SKIP_INSTALL=NO \
    BUILD_LIBRARY_FOR_DISTRIBUTION=YES

# iOS 模拟器（arm64 + x86_64）
xcodebuild archive \
    -scheme NetworkKit \
    -destination "generic/platform=iOS Simulator" \
    -archivePath ./build/ios-simulator.xcarchive \
    SKIP_INSTALL=NO \
    BUILD_LIBRARY_FOR_DISTRIBUTION=YES

# 第二步：合并为 XCFramework
xcodebuild -create-xcframework \
    -framework ./build/ios.xcarchive/Products/Library/Frameworks/NetworkKit.framework \
    -framework ./build/ios-simulator.xcarchive/Products/Library/Frameworks/NetworkKit.framework \
    -output ./build/NetworkKit.xcframework
```

> ⚠️ `BUILD_LIBRARY_FOR_DISTRIBUTION=YES` 是关键参数，它会生成 `.swiftinterface` 和 `.swiftdoc` 文件，确保不同 Swift 版本的用户都能使用该 framework。

### 8.3 闭源分发

方式一：通过 URL 分发

```swift
targets: [
    .binaryTarget(
        name: "NetworkKit",
        url: "https://github.com/your-org/NetworkKit/releases/download/1.0.0/NetworkKit.xcframework.zip",
        checksum: "a1b2c3d4e5f6789012345678901234567890abcdef1234567890abcdef123456"
    )
]
```

计算 checksum：

```bash
swift package compute-checksum ./build/NetworkKit.xcframework.zip
```

方式二：通过本地路径分发

```swift
targets: [
    .binaryTarget(
        name: "PaidSDK",
        path: "Frameworks/PaidSDK.xcframework"
    )
]
```

| 分发方式 | 适用场景 | 优缺点 |
|---------|---------|--------|
| URL + checksum | 公开分发、版本管理 | ✅ 版本精确 ❌ 需要托管 |
| 本地路径 | 私有 SDK、内部使用 | ✅ 简单 ❌ 不适合远程协作 |
| 私有 Git 仓库 | 企业内部 | ✅ 版本管理 ✅ 访问控制 ❌ 需配置认证 |

### 8.4 多平台 XCFramework 合并

完整的 XCFramework 可以包含所有 Apple 平台：

```bash
xcodebuild -create-xcframework \
    -framework ./build/ios.xcarchive/Products/Library/Frameworks/NetworkKit.framework \
    -framework ./build/ios-simulator.xcarchive/Products/Library/Frameworks/NetworkKit.framework \
    -framework ./build/macos.xcarchive/Products/Library/Frameworks/NetworkKit.framework \
    -framework ./build/tvos.xcarchive/Products/Library/Frameworks/NetworkKit.framework \
    -framework ./build/visionos.xcarchive/Products/Library/Frameworks/NetworkKit.framework \
    -output ./build/NetworkKit.xcframework
```

生成的 XCFramework 内部结构：

```
NetworkKit.xcframework/
├── Info.plist
├── ios-arm64/
│   └── NetworkKit.framework/
├── ios-arm64_x86_64-simulator/
│   └── NetworkKit.framework/
├── macos-arm64_x86_64/
│   └── NetworkKit.framework/
├── tvos-arm64/
│   └── NetworkKit.framework/
└── visionos-arm64/
    └── NetworkKit.framework/
```

> 💡 使用 GitHub Actions 可以自动化 XCFramework 的构建和发布：每次打 Tag 时，自动编译所有平台、合并 XCFramework、打包 zip、计算 checksum 并创建 Release。

---

## 本章小结

| 主题 | 核心要点 |
|------|---------|
| 开发动机 | 代码复用、模块化、开源贡献、团队共享——被多项目使用的通用代码才适合抽取 |
| 创建 Package | `swift package init` 创建，Package.swift 是核心配置，products/targets/dependencies 三大板块 |
| 设计原则 | API 最小暴露、默认 internal、SemVer 版本号、废弃标记代替直接删除 |
| 资源处理 | `.process()` 自动优化、`.copy()` 原样拷贝、始终用 `Bundle.module` 访问 |
| 测试与文档 | testTarget 配置、`swift test` 命令、DocC 注释生成精美文档 |
| 发布流程 | Git Tag 标记版本、GitHub Release 附带说明、Swift Package Index 注册增加曝光 |
| Plugin 开发 | Command Plugin 手动触发、Build Tool Plugin 自动执行、沙箱限制注意网络访问 |
| Binary Framework | XCFramework 多平台合并、`BUILD_LIBRARY_FOR_DISTRIBUTION=YES`、checksum 校验完整性 |

← [架构模式对比与选型](./架构模式对比与选型.md) | [国内上架：ICP 备案全流程](../07-上架准备/国内上架ICP备案全流程.md) →
