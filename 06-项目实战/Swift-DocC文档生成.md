# Swift-DocC 文档生成

> 🎯 **本章目标**：掌握 Swift-DocC 文档生成工具的使用，学会为 Swift 代码编写文档注释，能够生成和发布 API 文档，了解开源项目的文档最佳实践。

---

## Swift-DocC 概述

### 为什么需要 API 文档

在团队协作和开源项目中，API 文档是连接代码与使用者的桥梁。良好的文档能够：

- 降低新成员的学习成本，快速理解接口用途与行为
- 减少反复沟通的时间消耗，让代码自己"说话"
- 提升库或框架的采纳率，开源项目中文档质量直接影响 Star 数
- 在重构时提供契约参考，避免破坏性变更

没有文档的 API 就像一个没有说明书的工具箱——功能再强大也难以被正确使用。

### DocC 是什么

Swift-DocC 是 Apple 于 2021 年 WWDC 推出的官方文档编译器，专为 Swift 生态设计。它将源代码中的文档注释编译为结构化的、可交互的文档网站，并深度集成于 Xcode 和 Swift Package Manager。

DocC 的核心特性包括：

- 从源码注释自动生成 API 参考
- 支持教程（Tutorial）和文章（Article）两种补充内容
- 输出静态站点，可部署到任意 Web 服务器
- 与 Xcode 预览无缝集成，开发时即时查看文档
- 支持跨模块链接和符号解析

### 与其他文档工具对比

| 特性 | Swift-DocC | Jazzy | SourceDocs | Swagger/OpenAPI |
|------|-----------|-------|------------|-----------------|
| 维护方 | Apple | 社区 | 社区 | OpenAPI Initiative |
| 语言支持 | Swift | Swift | Swift | 多语言（REST API） |
| 输出格式 | 静态网站 | 静态网站 | Markdown | JSON/YAML + UI |
| 教程支持 | ✅ | ❌ | ❌ | ❌ |
| Xcode 集成 | ✅ | ❌ | ❌ | ❌ |
| SPM 集成 | ✅ | ❌ | ❌ | ❌ |
| 实时预览 | ✅ | ❌ | ❌ | ❌ |
| 学习曲线 | 低 | 中 | 低 | 高 |

> 💡 **提示**：对于纯 Swift 项目，Swift-DocC 是首选方案。如果需要生成 Markdown 格式文档供 GitHub Wiki 使用，可以配合 SourceDocs 作为补充。

---

## 文档注释编写

### 基本注释语法

Swift-DocC 使用 `///` 或 `/** */` 格式的文档注释，编译器会将其关联到紧随其后的声明：

```swift
/// 表示一个二维平面上的点。
///
/// 使用 `Point` 结构体来表示坐标系统中的位置，
/// 支持基本的几何运算。
struct Point {
    let x: Double
    let y: Double
}
```

### 参数说明

使用 `- Parameter` 标注函数参数：

```swift
/// 计算两点之间的欧几里得距离。
///
/// - Parameter from: 起始点。
/// - Parameter to: 终止点。
/// - Returns: 两点之间的距离值。
func distance(from: Point, to: Point) -> Double {
    sqrt(pow(to.x - from.x, 2) + pow(to.y - from.y, 2))
}
```

当参数较多时，可以使用 `- Parameters` 块语法：

```swift
/// 在指定位置创建圆形区域。
///
/// - Parameters:
///   - center: 圆心坐标。
///   - radius: 半径，单位为米。
///   - identifier: 区域唯一标识符。
/// - Returns: 构建完成的圆形区域对象。
func createRegion(center: Point, radius: Double, identifier: String) -> Region {
    Region(center: center, radius: radius, id: identifier)
}
```

### 返回值说明

使用 `- Returns` 描述返回值：

```swift
/// 尝试解析 JSON 数据为指定类型。
///
/// - Parameter data: 待解析的 JSON 数据。
/// - Parameter type: 目标解码类型。
/// - Returns: 解码成功时返回对应类型的实例。
/// - Throws: 当数据格式不匹配时抛出 `DecodingError`。
func parse<T: Decodable>(_ data: Data, as type: T.Type) throws -> T {
    try JSONDecoder().decode(type, from: data)
}
```

### 代码示例

在文档注释中嵌入代码示例，帮助使用者快速上手：

```swift
/// 一个线程安全的值容器。
///
/// 使用 ``ThreadSafeBox`` 包装需要在多线程间共享的值：
///
/// ```swift
/// let counter = ThreadSafeBox<Int>(0)
/// counter.update { value in
///     value += 1
/// }
/// print(counter.read())
/// ```
final class ThreadSafeBox<T> {
    private var value: T
    private let lock = NSLock()

    init(_ value: T) {
        self.value = value
    }

    func read() -> T {
        lock.lock()
        defer { lock.unlock() }
        return value
    }

    func update(_ transform: (inout T) -> Void) {
        lock.lock()
        defer { lock.unlock() }
        transform(&value)
    }
}
```

### Markdown 支持

DocC 注释中支持丰富的 Markdown 语法：

| 语法 | 用途 | 示例 |
|------|------|------|
| `` `code` `` | 行内代码 | 使用 `` `UIView` `` |
| `**bold**` | 加粗 | **重要提示** |
| `*italic*` | 斜体 | *可选参数* |
| `[link](url)` | 外部链接 | [Swift.org](https://swift.org) |
| `` `Symbol` `` | 符号链接 | 参见 `` `Point` `` |
| `- item` | 无序列表 | 参数列表 |
| `1. item` | 有序列表 | 步骤说明 |
| `> quote` | 引用块 | 注意事项 |

> ⚠️ **警告**：符号链接使用双反引号 `` ``SymbolName`` ``，而不是单反引号。单反引号仅表示行内代码，不会生成可点击的交叉引用链接。

---

## DocC 架构

### Documentation Catalog

Documentation Catalog（文档目录）是 DocC 的组织单元，位于 Swift Package 的 `Sources` 同级目录下。其结构如下：

```
MyPackage/
├── Sources/
│   └── MyLibrary/
│       └── Point.swift
├── Documentation/
│   └── MyLibrary/
│       ├── MyLibrary.md          # 根页面
│       ├── Article-GettingStarted.md
│       ├── Tutorial-Introduction.md
│       └── Resources/
│           └── diagram.png
└── Package.swift
```

根页面 `MyLibrary.md` 定义了文档的入口和导航结构：

```markdown
# ``MyLibrary``

MyLibrary 是一个几何计算库，提供点、线、面的基础运算。

## Overview

![库架构图](diagram.png)

## Topics

### 基础类型

- ``Point``
- ``Line``
- ``Region``

### 运算函数

- ``distance(from:to:)``
- ``area(of:)``
```

### 文章页面

文章（Article）用于补充 API 参考无法覆盖的内容，如设计理念、迁移指南、最佳实践等：

```markdown
# Getting Started with MyLibrary

## Overview

本文将引导你完成 MyLibrary 的安装与基本使用。

## Installation

在 `Package.swift` 中添加依赖：

```swift
dependencies: [
    .package(url: "https://github.com/example/MyLibrary", from: "1.0.0")
]
```

## Quick Start

创建一个点并计算距离：

```swift
let origin = Point(x: 0, y: 0)
let target = Point(x: 3, y: 4)
let dist = distance(from: origin, to: target)
```
```

### 教程页面

教程（Tutorial）提供分步骤的交互式学习体验，包含步骤说明和代码检查点：

```markdown
@Tutorial(time: 20 minutes, projectFiles: "https://example.com/files.zip") {
   @Intro(title: "Creating a Point") {
      Learn how to create and use points in MyLibrary.
   }

   @Section(title: "Create a Point") {
      @ContentAndMedia(layout: vertical) {
         Start by creating a point with x and y coordinates.
      }

      @Steps {
         @Step {
            Import the library.

            @Code(name: "main.swift", file: "step1.swift") {
               import MyLibrary
            }
         }
         @Step {
            Create a point instance.

            @Code(name: "main.swift", file: "step2.swift") {
               let p = Point(x: 1, y: 2)
            }
         }
      }
   }
}
```

### 扩展页面

扩展文件（Extension File）允许为不属于自己的类型添加文档，常用于为系统框架类型补充说明：

```markdown
# ``Swift/Array``

## Extensions

### MyLibrary Extensions

对 `Array` 的扩展方法。

## Topics

### Geometric Operations

- ``Array/centroid()``
- ``Array/boundingRect()``
```

> ⚠️ **警告**：扩展文件中引用的符号必须确实存在，否则 DocC 编译时会报错。不要为尚不存在的 API 编写扩展文档。

---

## 生成与预览

### 命令行生成

使用 `swift package` 命令生成文档：

```bash
swift package generate-documentation
```

指定输出目录和方案名称：

```bash
swift package generate-documentation \
    --output-path ./docs \
    --target MyLibrary \
    --transform-for-static-hosting
```

常用参数说明：

| 参数 | 说明 |
|------|------|
| `--output-path` | 输出目录路径 |
| `--target` | 指定文档化的目标 |
| `--transform-for-static-hosting` | 生成可静态托管的站点 |
| `--hosting-base-path` | 部署子路径（如 GitHub Pages 项目页） |
| `--disable-indexing` | 禁用索引生成 |
| `--analyze` | 仅分析，不生成输出 |

### Xcode 预览

在 Xcode 中预览文档的步骤：

1. 打开包含 Swift Package 的 Xcode 项目
2. 选择菜单 **Product → Build Documentation**（快捷键 `⌃⇧⌘D`）
3. Xcode 自动编译文档并在 Documentation Viewer 中展示
4. 修改注释后重新 Build Documentation 即可刷新

> 💡 **提示**：Xcode 的 Documentation Viewer 支持搜索和跨模块跳转，是日常开发中最便捷的文档查阅方式。

### 静态站点输出

生成可部署的静态站点需要添加 `--transform-for-static-hosting` 参数：

```bash
swift package generate-documentation \
    --transform-for-static-hosting \
    --hosting-base-path MyLibrary \
    --output-path ./docs
```

生成的目录结构：

```
docs/
├── index.html
├── data/
│   └── documentation/
│       └── mylibrary.json
├── css/
├── js/
└── img/
```

该站点可以直接部署到任何静态文件服务器，无需后端支持。

---

## 发布文档

### GitHub Pages 部署

使用 GitHub Actions 自动部署文档到 GitHub Pages：

```yaml
name: Publish Documentation

on:
  push:
    branches: [main]

permissions:
  contents: read
  pages: write
  id-token: write

jobs:
  publish:
    runs-on: macos-14
    steps:
      - uses: actions/checkout@v4
      - uses: swift-actions/setup-swift@v2
        with:
          swift-version: "5.9"
      - run: swift package generate-documentation --transform-for-static-hosting --hosting-base-path MyLibrary --output-path ./docs
      - uses: actions/upload-pages-artifact@v3
        with:
          path: ./docs
      - uses: actions/deploy-pages@v4
```

> ⚠️ **警告**：`--hosting-base-path` 必须与仓库名称一致，否则资源路径会 404。例如仓库名为 `MyLibrary`，则参数值为 `/MyLibrary`。

### SPM 集成

在 `Package.swift` 中声明文档目标：

```swift
let package = Package(
    name: "MyLibrary",
    platforms: [.iOS(.v15), .macOS(.v12)],
    products: [
        .library(name: "MyLibrary", targets: ["MyLibrary"]),
    ],
    targets: [
        .target(
            name: "MyLibrary",
            path: "Sources/MyLibrary"
        ),
        .documentationTarget(
            name: "MyLibrary",
            path: "Documentation/MyLibrary"
        ),
    ]
)
```

`.documentationTarget` 显式声明文档目录的位置，使 SPM 在构建文档时能正确找到资源。

### 开源项目文档最佳实践

| 实践 | 说明 |
|------|------|
| 每个公开 API 都有文档注释 | 确保零文档覆盖率空白 |
| 提供快速入门文章 | 降低新用户门槛 |
| 包含可运行的代码示例 | 代码比文字更直观 |
| 使用教程引导核心流程 | 分步骤教学优于大段说明 |
| CI 自动生成与部署 | 文档与代码同步更新 |
| 版本化文档 | 与语义化版本对应，避免误导 |
| 跨符号链接 | 使用双反引号语法 `` `Symbol` `` 实现跳转 |
| 图片与图表辅助 | 架构图、流程图提升理解效率 |

> 💡 **提示**：在开源项目中，文档覆盖率是衡量项目质量的重要指标。可以结合 Swift-DocC 的 `--analyze` 模式在 CI 中检查文档覆盖率，对未文档化的公开 API 发出警告。

---

## 小结

| 主题 | 关键要点 |
|------|---------|
| DocC 概述 | Apple 官方文档编译器，深度集成 Xcode 和 SPM，支持教程与文章 |
| 文档注释 | 使用 `///` 语法，支持参数、返回值、代码示例和 Markdown |
| DocC 架构 | Documentation Catalog 组织文档，包含根页面、文章、教程、扩展 |
| 生成与预览 | 命令行生成、Xcode 实时预览、静态站点输出 |
| 发布文档 | GitHub Pages 自动部署、SPM 集成、版本化与覆盖率管理 |

Swift-DocC 让文档不再是事后的补充，而是开发流程中自然产出的一部分。通过持续维护文档注释，你的代码将自带清晰的使用说明，大幅降低沟通成本。

← [Swift Package 开发与发布](./Swift-Package开发与发布.md) | [代码签名与证书管理深入](./代码签名与证书管理深入.md) →