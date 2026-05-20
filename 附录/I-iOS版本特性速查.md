# I-iOS 版本特性速查

> 本附录按 iOS 版本分类，汇总各版本关键新 API / 框架及开发者需关注的变化，方便快速查阅。

---

## 一、iOS 18 新特性

| 类别 | 新 API / 框架 | 说明 | 开发者关注点 |
|------|--------------|------|-------------|
| 测试 | Swift Testing | 全新测试框架，用 `@Test` 宏替代 `XCTestCase`，用 `#expect` 替代 `XCTAssert` | 可与 XCTest 共存；新项目推荐使用，语法更简洁 |
| 测试 | `@Test` 宏 | 标记测试函数，替代 `func testXxx()` 命名约定 | 无需继承 `XCTestCase`，支持结构体 / 枚举中定义测试 |
| 测试 | `#expect()` | 断言宏，替代各类 `XCTAssert*` | 失败时自动捕获表达式，错误信息更友好 |
| 测试 | `@Suite` 宏 | 组织测试用例为测试套件 | 替代 `XCTestCase` 子类作为分组方式 |
| 控件定制 | `Button` 新增 `buttonStyle` 定制 | 支持更灵活的按钮外观定制 | 可自定义形状、颜色、动画 |
| 控件定制 | `Toggle` 新增样式定制 | 支持自定义开关外观 | 适配不同设计风格 |
| 控件定制 | `TabBar` 全面可定制 | 可自定义标签栏位置、样式、滚动行为 | 替代旧 `TabView` 默认样式 |
| 控件定制 | `Sidebar` 增强定制 | 侧边栏支持折叠、自定义分区 | 适配 iPad 和 Mac |
| App Intents | `AppIntent` 协议增强 | 新增更多参数类型和交互方式 | 可构建更复杂的快捷指令 |
| App Intents | `AppEntity` 查询增强 | 支持模糊搜索和分页查询 | 大数据量场景性能更优 |
| App Intents | `AppShortcutsProvider` 增强 | 可动态提供快捷指令 | 减少用户手动配置 |
| App Intents | Controls API | 可在控制中心添加自定义控件 | 需实现 `ControlWidget` 和 `ControlWidgetConfiguration` |
| 其他 | `ScrollView` 增强 | 新增 `scrollPosition` 修饰符 | 可程序化控制滚动位置 |
| 其他 | `MeshGradient` | 全新网格渐变效果 | 丰富 UI 视觉表现 |
| 其他 | `VolumeWindow` | visionOS 上的空间体积窗口 | 为 visionOS 应用提供新容器 |

---

## 二、iOS 17 新特性

| 类别 | 新 API / 框架 | 说明 | 开发者关注点 |
|------|--------------|------|-------------|
| 数据观察 | `@Observable` 宏 | Swift Observation 框架核心宏，自动生成属性观察代码 | 替代 `ObservableObject`，无需逐属性标记 `@Published` |
| 数据观察 | `Observation` 框架 | 全新响应式数据观察框架 | 与 SwiftUI 深度集成，性能优于 Combine + `ObservableObject` |
| 数据观察 | `withObservationTracking()` | 精确追踪属性访问 | 可在非 SwiftUI 场景使用观察机制 |
| 数据持久化 | SwiftData | 声明式数据持久化框架，用 `@Model` 宏定义模型 | 替代 Core Data，无需 `.xcdatamodeld` 文件 |
| 数据持久化 | `@Model` 宏 | 标记数据模型类 | 自动生成持久化代码，支持关系、迁移 |
| 数据持久化 | `@Query` 属性包装器 | 在 SwiftUI 中查询 SwiftData 数据 | 替代 `FetchRequest`，语法更简洁 |
| 数据持久化 | `ModelContainer` | 配置数据存储容器 | 可配置内存存储、云端同步等 |
| 提示引导 | TipKit | 全新用户引导提示框架 | 统一管理新手引导、功能提示 |
| 提示引导 | `Tip` 协议 | 定义提示内容 | 需实现 `title`、`message` 等属性 |
| 提示引导 | `Tips.configure()` | 全局配置提示显示规则 | 可设置显示频率、条件等 |
| 提示引导 | `PopoverTip` | 气泡式提示 | 指向特定 UI 元素 |
| Widget | Widget 交互增强 | 支持按钮、切换开关等交互控件 | Widget 不再仅展示，可执行操作 |
| Widget | `AppIntentConfiguration` | 支持交互式 Widget 配置 | 用户可在 Widget 上直接操作 |
| Live Activity | Live Activity 增强 | 支持更多动态内容更新 | 可显示赛事比分、外卖进度等 |
| Live Activity | `ActivityKit` 增强 | 新增 `Activity.update()` 更多选项 | 支持警报类型 Live Activity |
| 其他 | `ScrollView` 增强 | 新增 `scrollTargetBehavior` | 可实现分页滚动、对齐滚动 |
| 其他 | `PhaseAnimator` | 按阶段驱动动画 | 替代复杂 `TimelineView` 场景 |
| 其他 | `SensoryFeedback` | 触觉反馈修饰符 | 替代 `UIImpactFeedbackGenerator` |
| 其他 | `ToolbarItem` 位置增强 | 新增 `topBarLeading` 等位置 | 更灵活的工具栏布局 |
| 其他 | `#Preview` 宏 | 替代 `PreviewProvider` | 可在结构体中直接定义预览 |

---

## 三、iOS 16 新特性

| 类别 | 新 API / 框架 | 说明 | 开发者关注点 |
|------|--------------|------|-------------|
| 实时活动 | Live Activities | 锁屏 / 灵动岛实时展示应用动态 | 需使用 `ActivityKit` 框架 |
| 实时活动 | `ActivityKit` | 管理 Live Activity 生命周期 | 通过 `Activity.request()` 创建，`Activity.update()` 更新 |
| 实时活动 | `ActivityAttributes` | 定义 Live Activity 数据模型 | 静态数据 + 动态状态分离 |
| Widget | WidgetKit 增强 | 锁屏 Widget 支持 | 新增 `widgetFamily` 尺寸：`.accessoryRectangular`、`.accessoryCircular`、`.accessoryInline` |
| Widget | `AccessoryWidgetGroup` | 锁屏 Widget 布局容器 | 适配锁屏圆形 / 矩形样式 |
| 导航 | `NavigationStack` | 替代 `NavigationView` | 支持程序化导航路径管理 |
| 导航 | `NavigationPath` | 类型安全的导航路径 | 可动态 push / pop，支持深链接 |
| 导航 | `navigationDestination()` | 声明式导航目标绑定 | 替代 `NavigationLink` 的值绑定方式 |
| 图表 | Charts 框架 | 全新声明式图表框架 | 支持折线图、柱状图、面积图、点图等 |
| 图表 | `Chart {}` | 图表容器 | 内部放置 `LineMark`、`BarMark` 等 |
| 图表 | `LineMark` / `BarMark` / `PointMark` / `AreaMark` | 各类图表标记 | 声明式定义图表元素 |
| 图表 | `.chartXAxis` / `.chartYAxis` | 自定义坐标轴 | 可自定义刻度、标签样式 |
| 其他 | `AnyLayout` | 运行时切换布局 | 可在 `VStack` / `HStack` 间动态切换 |
| 其他 | `Grid` | 全新网格布局 | 替代 `LazyVGrid` / `LazyHGrid` 的部分场景 |
| 其他 | `ShareLink` | 系统分享按钮 | 替代 `UIActivityViewController` |
| 其他 | `ImageRenderer` | SwiftUI 视图转图片 | 替代 `UIGraphicsImageRenderer` |
| 其他 | `Layout` 协议 | 自定义布局协议 | 可实现完全自定义的布局算法 |
| 其他 | `ViewThatFits` | 自动选择合适尺寸的视图 | 适配不同屏幕尺寸 |

---

## 四、iOS 15 新特性

| 类别 | 新 API / 框架 | 说明 | 开发者关注点 |
|------|--------------|------|-------------|
| 并发 | `async/await` | Swift 原生异步编程模型 | 替代回调 / 闭包嵌套，代码线性可读 |
| 并发 | `Task` | 创建异步任务 | 可在非异步上下文中启动异步操作 |
| 并发 | `TaskGroup` | 结构化并发任务组 | 并行执行多个子任务并收集结果 |
| 并发 | `async let` | 并行绑定异步结果 | 多个异步操作自动并行执行 |
| 并发 | `Actor` | 数据隔离的并发类型 | 保护共享可变状态，替代锁机制 |
| 并发 | `Sendable` 协议 | 标记可安全跨并发域传递的类型 | 编译器检查并发安全性 |
| 并发 | `@MainActor` | 标记必须在主线程执行的代码 | 替代 `DispatchQueue.main.async` |
| 并发 | `withCheckedContinuation()` | 桥接回调式 API 到 async/await | 将 `completionHandler` 包装为 `async` |
| 并发 | Concurrent Queue | 并发队列增强 | `DispatchQueue` 与 Swift Concurrency 互操作 |
| UI | Sheet 变化 | `sheet()` 支持自定义尺寸和交互 | 新增 `presentationDetents()` 修饰符 |
| UI | `presentationDetents()` | 控制 Sheet 高度 | 支持 `.medium`、`.large`、自定义高度 |
| UI | `presentationDragIndicator()` | 显示 Sheet 拖拽指示器 | 提升可发现性 |
| UI | `refreshable()` 修饰符 | 下拉刷新 | 与 `async/await` 配合使用 |
| UI | `searchable()` 修饰符 | 搜索栏集成 | 替代 `UISearchController` |
| UI | `FocusState` | 焦点管理 | 替代 `@FocusScope`，更简洁 |
| 其他 | `AttributedString` | 类型安全的富文本 | 替代 `NSAttributedString`，支持 Markdown 解析 |
| 其他 | `Canvas` | 高性能自定义绘制视图 | 替代 `GeometryReader` + `Path` 绘制场景 |
| 其他 | `TimelineView` | 时间驱动视图更新 | 适用于时钟、倒计时等场景 |
| 其他 | `URLSession` async API | 网络请求原生 async 支持 | `data(for:)` 替代 `dataTask(with:completionHandler:)` |

---

## 五、最低支持版本选择建议

### 5.1 各版本市场占有率（参考值，持续变化）

| iOS 版本 | 大致市场占有率 | 发布年份 | 备注 |
|----------|--------------|---------|------|
| iOS 18 | ~30%（持续增长中） | 2024 | 最新版本，新设备预装 |
| iOS 17 | ~40% | 2023 | 当前主流版本 |
| iOS 16 | ~20% | 2022 | 仍有较多用户 |
| iOS 15 | ~8% | 2021 | 老设备最终支持版本 |
| iOS 14 及更早 | ~2% | 2020 及更早 | 极少用户 |

> ⚠️ 以上数据为估算值，建议参考 Apple 官方分布数据或 App Analytics 实际数据。

### 5.2 如何决策最低支持版本

| 决策因素 | 建议 | 说明 |
|----------|------|------|
| 目标用户群 | 面向年轻 / 技术用户 → 支持最新 2 个版本 | 新版本用户占比高 |
| 目标用户群 | 面向全年龄段 → 支持最新 3 个版本 | 覆盖更多老设备 |
| 核心功能依赖 | 依赖 SwiftData / @Observable → 最低 iOS 17 | 无法降级到旧版本 |
| 核心功能依赖 | 依赖 Live Activities → 最低 iOS 16 | 需灵动岛 / 锁屏功能 |
| 核心功能依赖 | 仅需 async/await → 最低 iOS 15 | 基础并发特性 |
| 开发成本 | 支持版本越多，兼容代码越多 | 权衡开发投入与用户覆盖 |
| 维护周期 | 新项目建议最低 iOS 16+ | 减少旧 API 兼容负担 |
| App Store 审核 | 无强制最低版本要求 | 但需确保在支持版本上功能正常 |

### 5.3 推荐策略

| 场景 | 推荐最低版本 | 覆盖率 |
|------|------------|--------|
| 全新项目，追求新技术 | iOS 17 | ~90% |
| 全新项目，平衡覆盖 | iOS 16 | ~95% |
| 现有项目，保守升级 | iOS 15 | ~98% |
| 企业内部应用 | iOS 17+ | 设备可控 |

---

## 六、API 可用性标注

### 6.1 `@available` 标注

用于标记类型、方法的可用平台和版本：

```swift
@available(iOS 17, *)
struct MyView: View {
    var body: some View {
        ContentUnavailableView("无数据", systemImage: "tray")
    }
}
```

| 标注形式 | 含义 |
|----------|------|
| `@available(iOS 17, *)` | iOS 17 及以上可用，`*` 表示其他平台不限 |
| `@available(iOS 16, macOS 13, *)` | iOS 16+ 且 macOS 13+ 可用 |
| `@available(iOS, introduced: 16, deprecated: 18)` | iOS 16 引入，iOS 18 废弃 |
| `@available(iOS, obsoleted: 18)` | iOS 18 起移除，不可再使用 |
| `@available(iOS 17, macOS 14, tvOS 17, watchOS 10, *)` | 多平台同时标注 |
| `@available(*, unavailable)` | 所有平台不可用（标记已移除 API） |

### 6.2 `if #available()` 运行时检查

用于在运行时根据系统版本执行不同代码：

```swift
if #available(iOS 17, *) {
    let container = try ModelContainer(for: Item.self)
} else {
    let container = NSPersistentContainer(name: "Model")
}
```

| 用法 | 说明 |
|------|------|
| `if #available(iOS 17, *)` | iOS 17+ 走 if 分支，否则走 else |
| `guard #available(iOS 17, *) else { return }` | 守卫模式，不满足则提前退出 |
| `#available(iOS 16, *)` | 可与 `@available` 配合使用 |

### 6.3 两者对比

| 对比项 | `@available` | `if #available()` |
|--------|-------------|-------------------|
| 作用时机 | 编译期 | 运行时 |
| 作用对象 | 类型、函数、属性 | 代码块 |
| 用途 | 声明 API 可用范围 | 运行时分支判断 |
| 缺失后果 | 编译警告 / 错误 | 旧版本崩溃 |

---

## 七、向后兼容技巧

### 7.1 条件编译

```swift
#if canImport(SwiftData)
import SwiftData
#endif
```

| 指令 | 说明 |
|------|------|
| `#if canImport(Framework)` | 检查框架是否可导入 |
| `#if os(iOS)` | 按平台条件编译 |
| `#if swift(>=5.9)` | 按 Swift 版本条件编译 |

### 7.2 View 修饰符降级方案

为高版本 API 提供低版本替代实现：

```swift
extension View {
    @ViewBuilder
    func scrollTargetBehaviorIfAvailable() -> some View {
        if #available(iOS 17, *) {
            scrollTargetBehavior(.paging)
        }
    }
}
```

| 技巧 | 说明 | 示例 |
|------|------|------|
| `@ViewBuilder` | 条件返回不同视图 | `if #available` 分支返回不同 View |
| 空实现降级 | 旧版本不执行高版本修饰符 | 旧版本直接 `return self` |
| 替代实现 | 旧版本用旧 API 实现 | `NavigationStack` 降级为 `NavigationView` |

### 7.3 常见降级对照表

| iOS 17+ API | iOS 16 降级方案 | iOS 15 降级方案 |
|-------------|----------------|----------------|
| `@Observable` | `ObservableObject` + `@Published` | `ObservableObject` + `@Published` |
| SwiftData `@Model` | Core Data `NSManagedObject` | Core Data `NSManagedObject` |
| SwiftData `@Query` | `@FetchRequest` | `@FetchRequest` |
| `#Preview` 宏 | `PreviewProvider` 协议 | `PreviewProvider` 协议 |
| `ScrollView` + `.scrollTargetBehavior` | `ScrollView` + `GeometryReader` 手动计算 | `ScrollView` + `GeometryReader` 手动计算 |
| `NavigationStack` | `NavigationView` + `NavigationLink` | `NavigationView` + `NavigationLink` |
| `ContentUnavailableView` | 自定义空状态视图 | 自定义空状态视图 |
| `.presentationDetents()` | 自定义 Sheet 高度（UIKit） | 自定义 Sheet 高度（UIKit） |

### 7.4 封装兼容层建议

```swift
struct CompatNavigationStack<Content: View>: View {
    let content: Content

    init(@ViewBuilder content: () -> Content) {
        self.content = content()
    }

    var body: some View {
        if #available(iOS 16, *) {
            NavigationStack {
                content
            }
        } else {
            NavigationView {
                content
            }
        }
    }
}
```

| 封装原则 | 说明 |
|----------|------|
| 统一入口 | 业务代码只调用兼容层，不直接使用版本敏感 API |
| 最小化 `#available` | 将版本判断收敛在兼容层内部 |
| 渐进移除 | 当最低版本提升时，直接删除兼容层，替换为原生 API |
| 单元测试 | 对兼容层两个分支分别测试 |

### 7.5 Xcode 配置

| 配置项 | 路径 | 说明 |
|--------|------|------|
| Minimum Deployments | Target → General → Minimum Deployments | 设置最低支持版本 |
| Swift Language Version | Build Settings → Swift Language Version | 设置 Swift 版本 |
| SDK 版本 | Xcode → Settings → Locations | 选择 Xcode / SDK 版本 |

---

> 💡 **提示**：选择最低支持版本时，优先考虑核心功能依赖的 API 版本，再结合目标用户覆盖率做取舍。新项目建议从 iOS 16+ 起步，可覆盖 95% 以上用户，同时获得 `NavigationStack`、Charts、Live Activities 等关键特性。
