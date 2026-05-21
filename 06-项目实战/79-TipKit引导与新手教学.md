# 79-TipKit 引导与新手教学

## 本章目标

- 理解 TipKit 是什么、为什么需要它
- 学会定义 Tip（标题、消息、图片）
- 掌握 Popover（气泡）和 Inline（内嵌）两种展示方式
- 学会设置 Tip 的触发条件和显示规则
- 能够定制 Tip 的样式（颜色、图标、动作按钮）
- 理解如何控制显示逻辑（已读、关闭后不再显示、A/B 展示）
- 完成实战：给天气 App 添加新手引导
- 掌握提示设计最佳实践

---

## 1. TipKit 简介

### 1.1 没有 TipKit 的日子

想象你搬进了一栋新房子，但没有人告诉你：
- 厨房的灯开关在哪儿
- 热水器怎么调温度
- 遥控器哪个键是静音

你只能自己摸索，体验很差。App 也一样——如果用户不知道某个功能怎么用，他们可能永远不会发现它。

在 TipKit 出现之前，开发者通常这样引导用户：

| 方式 | 问题 |
|------|------|
| 自己写气泡视图 | 代码量大，位置计算麻烦 |
| 用第三方库 | 增加依赖，风格不统一 |
| 弹窗（Alert） | 太打断用户，体验差 |
| 教程页面 | 用户懒得看，直接跳过 |

### 1.2 TipKit 登场

**TipKit** 是 Apple 在 iOS 17 / WWDC23 推出的官方提示框架。它的核心理念：

> 💡 **在合适的时机，用合适的方式，优雅地告诉用户他们还不知道的事。**

就像一位贴心的导购员——不会追着你推销，但当你走到一个货架前犹豫时，会适时走过来介绍。

### 1.3 TipKit 的核心优势

| 优势 | 说明 |
|------|------|
| 原生框架 | 无需第三方依赖，风格与系统一致 |
| 声明式 API | 和 SwiftUI 风格统一，上手简单 |
| 智能展示 | 内置频率控制，不会过度打扰用户 |
| 多种展示方式 | Popover 气泡 / Inline 内嵌，灵活选择 |
| 持久化 | 自动记录已读状态，App 重启后不会重复提示 |

### 1.4 最低系统要求

| 平台 | 最低版本 |
|------|----------|
| iOS | 17.0+ |
| iPadOS | 17.0+ |
| macOS | 14.0+ |
| watchOS | 10.0+ |

> ⚠️ 如果你的 App 需要支持 iOS 16 及以下，无法使用 TipKit，需要自行实现或使用第三方库。

---

## 2. 定义 Tip

### 2.1 Tip 协议

定义一个 Tip 就像填一张表格——你只需要告诉系统"提示什么"，展示的事交给框架。

所有 Tip 都必须遵循 `Tip` 协议：

```swift
import TipKit

struct MyFirstTip: Tip {
    var title: Text {
        Text("欢迎使用天气 App")
    }

    var message: Text? {
        Text("点击右上角的 + 号可以添加新城市")
    }
}
```

> 💡 `Tip` 是一个协议（protocol），不是类。所以用 `struct` 来定义即可。

### 2.2 Tip 的三大组成要素

| 要素 | 类型 | 是否必填 | 说明 |
|------|------|----------|------|
| `title` | `Text` | ✅ 必填 | 提示标题，简短有力 |
| `message` | `Text?` | ❌ 可选 | 详细说明，补充标题 |
| `image` | `Image?` | ❌ 可选 | 配图，增强视觉效果 |

一个完整的示例：

```swift
struct AddCityTip: Tip {
    var title: Text {
        Text("添加城市")
    }

    var message: Text? {
        Text("点击 + 号可以添加你关心的城市，随时查看当地天气")
    }

    var image: Image? {
        Image(systemName: "plus.circle.fill")
    }
}
```

### 2.3 标题和消息的样式

`title` 和 `message` 的类型是 `Text`，所以你可以使用 SwiftUI 的 `Text` 修饰符：

```swift
struct StyledTip: Tip {
    var title: Text {
        Text("新功能上线！")
            .foregroundStyle(.blue)
            .bold()
    }

    var message: Text? {
        Text("现在支持 **15天天气预报**，提前规划出行")
    }
}
```

> ⚠️ 注意：`Text` 上的样式修饰符在某些展示方式下可能不生效，具体效果以实际运行为准。核心文案内容比样式更重要。

---

## 3. Tip 展示方式

TipKit 提供两种展示方式，就像老师有两种方式提醒学生：

| 展示方式 | 类比 | 特点 |
|----------|------|------|
| **Popover** | 老师走到你桌前，轻声提醒 | 气泡指向具体元素，醒目 |
| **Inline** | 黑板角落贴了一张便签 | 内嵌在页面中，不打断操作 |

### 3.1 Popover（气泡提示）

Popover 会以气泡的形式出现在目标元素旁边，带有一个小箭头指向它。

```swift
import SwiftUI
import TipKit

struct WeatherListView: View {
    let addCityTip = AddCityTip()

    var body: some View {
        NavigationStack {
            List {
                Text("北京  晴  28°")
                Text("上海  多云  24°")
            }
            .navigationTitle("天气")
            .toolbar {
                ToolbarItem(placement: .topBarTrailing) {
                    Button {
                        // 添加城市
                    } label: {
                        Image(systemName: "plus")
                    }
                    .popoverTip(addCityTip) // 👈 气泡提示附着在 + 按钮上
                }
            }
        }
    }
}
```

效果：一个气泡从 + 按钮旁弹出，箭头指向 + 按钮，显示"添加城市"的提示。

### 3.2 Inline（内嵌提示）

Inline 会像普通视图一样嵌入页面布局中，不会遮挡其他内容。

```swift
struct WeatherListView: View {
    let forecastTip = ForecastTip()

    var body: some View {
        NavigationStack {
            VStack {
                // 内嵌提示显示在列表上方
                TipView(forecastTip, arrowEdge: .bottom)

                List {
                    Text("北京  晴  28°")
                    Text("上海  多云  24°")
                }
            }
            .navigationTitle("天气")
        }
    }
}
```

### 3.3 两种方式对比

| 对比项 | Popover | Inline |
|--------|---------|--------|
| 位置 | 悬浮在目标元素旁 | 嵌入页面布局 |
| 箭头 | 有，指向目标 | 可选 |
| 遮挡 | 可能遮挡部分内容 | 不遮挡 |
| 适用场景 | 指向具体按钮/控件 | 页面级提示、功能介绍 |
| 用户感知 | 较强，像弹窗 | 较弱，像通知 |
| 关闭方式 | 点击外部自动关闭 | 需要关闭按钮 |

> 💡 选择建议：如果提示和某个具体控件相关（如"点这个按钮"），用 Popover；如果是页面级功能介绍，用 Inline。

### 3.4 Popover 的箭头方向

Popover 可以指定箭头方向：

```swift
.popoverTip(addCityTip, arrowEdge: .bottom) // 箭头在底部，气泡在上方
.popoverTip(addCityTip, arrowEdge: .top)    // 箭头在顶部，气泡在下方
.popoverTip(addCityTip, arrowEdge: .leading) // 箭头在左侧，气泡在右方
.popoverTip(addCityTip, arrowEdge: .trailing) // 箭头在右侧，气泡在左方
```

> ⚠️ 箭头方向是建议值，系统会根据可用空间自动调整。如果指定方向空间不够，系统会选择其他方向。

---

## 4. Tip 触发条件

### 4.1 为什么需要触发条件？

如果没有触发条件，所有 Tip 会在页面出现时一起弹出来——就像十个导购员同时冲过来，用户只会想逃跑。

触发条件让 Tip 在**合适的时机**才出现。

### 4.2 基本显示规则

通过 `rules` 属性设置 Tip 的显示条件：

```swift
struct HourlyForecastTip: Tip {
    var title: Text {
        Text("逐小时预报")
    }

    var message: Text? {
        Text("左右滑动可以查看未来24小时的天气变化")
    }

    // 👇 定义显示规则
    var rules: [Rule] {
        // 用户已经查看过天气详情页时才显示
        Rule(#Predicate<TipKit.Tips.Status> {
            $0.donations.contains(where: { $0.id == "viewedDetail" })
        })
    }
}
```

> 💡 `#Predicate` 是 Swift 5.9 引入的宏，用于编写类型安全的谓词。你可以把它理解为"条件判断的 Swift 原生写法"。

### 4.3 记录事件（Donation）

规则需要依赖事件。当用户执行某个操作时，我们"捐赠"一个事件：

```swift
struct WeatherDetailView: View {
    let hourlyForecastTip = HourlyForecastTip()

    var body: some View {
        VStack {
            Text("北京天气详情")
        }
        .onAppear {
            // 👇 记录用户查看过详情页
            hourlyForecastTip.donate(reason: .actionPerformed)
        }
    }
}
```

也可以用自定义 Donation：

```swift
struct WeatherDetailView: View {
    var body: some View {
        VStack {
            Text("北京天气详情")
        }
        .onAppear {
            // 👇 捐赠自定义事件
            Tips.Donate(reason: .actionPerformed, id: "viewedDetail")
        }
    }
}
```

### 4.4 多条件组合

可以用 `&&` 和 `||` 组合多个规则：

```swift
var rules: [Rule] {
    // 方式一：所有条件都满足（AND）
    Rule(#Predicate<TipKit.Tips.Status> {
        $0.donations.contains(where: { $0.id == "viewedDetail" })
    })
    Rule(#Predicate<TipKit.Tips.Status> {
        $0.donations.contains(where: { $0.id == "addedCity" })
    })

    // 方式二：满足任一条件（OR）—— 使用 Rule.or
    // Rule.or([
    //     Rule(#Predicate<...> { ... }),
    //     Rule(#Predicate<...> { ... })
    // ])
}
```

| 组合方式 | 写法 | 含义 |
|----------|------|------|
| AND（全部满足） | `rules` 数组中放多个 `Rule` | 所有规则都满足才显示 |
| OR（任一满足） | `Rule.or([...])` | 任一规则满足就显示 |

### 4.5 最大显示次数

限制 Tip 最多显示几次，避免反复打扰：

```swift
struct SearchTip: Tip {
    var title: Text {
        Text("搜索城市")
    }

    var message: Text? {
        Text("下拉即可搜索全球城市")
    }

    var options: [TipOption] {
        // 最多显示 3 次
        Tips.MaxDisplayCount(3)
    }
}
```

### 4.6 Tips.Group（提示组）

当页面有多个 Tip 时，用 `Tips.Group` 控制同时只显示一个：

```swift
struct WeatherListView: View {
    let addCityTip = AddCityTip()
    let searchTip = SearchTip()
    let forecastTip = ForecastTip()

    var body: some View {
        Tips.Group(maxDisplayCount: 1) {
            TipView(addCityTip)
            TipView(searchTip)
            TipView(forecastTip)
        }
    }
}
```

| 参数 | 说明 |
|------|------|
| `maxDisplayCount` | 同时最多显示几个 Tip |
| 不设置 | 所有满足条件的 Tip 都显示 |

> 💡 `Tips.Group` 就像红绿灯——同一时刻只有一个方向是绿灯，避免信息过载。

---

## 5. Tip 样式定制

### 5.1 默认样式

TipKit 的默认样式已经很好看，遵循系统设计规范。但如果你需要品牌化，可以自定义。

### 5.2 自定义动作按钮

Tip 可以添加动作按钮，让用户直接执行操作：

```swift
struct AddCityTip: Tip {
    var title: Text {
        Text("添加城市")
    }

    var message: Text? {
        Text("点击下方按钮快速添加你的第一个城市")
    }

    var actions: [Action] {
        Action(id: "addNow") {
            Text("立即添加")
        }
        Action(id: "later") {
            Text("稍后")
        }
    }
}
```

处理按钮点击：

```swift
struct WeatherListView: View {
    @State private var showAddSheet = false
    let addCityTip = AddCityTip()

    var body: some View {
        List {
            // ...
        }
        .popoverTip(addCityTip) { action in
            if action.id == "addNow" {
                showAddSheet = true
            }
        }
        .sheet(isPresented: $showAddSheet) {
            Text("添加城市页面")
        }
    }
}
```

### 5.3 自定义 TipView 样式

使用 `tipViewStyle` 修饰符或直接自定义 `TipView`：

```swift
// 使用系统内置样式
TipView(myTip)
    .tipViewStyle(.standard)

// 自定义背景色和圆角
TipView(myTip)
    .tipBackground(.blue.gradient)
    .foregroundStyle(.white)
```

### 5.4 完整样式定制示例

```swift
struct PremiumTip: Tip {
    var title: Text {
        Text("✨ 升级高级版")
    }

    var message: Text? {
        Text("解锁雷达图、空气质量指数等高级功能")
    }

    var image: Image? {
        Image(systemName: "crown.fill")
    }

    var actions: [Action] {
        Action(id: "upgrade") {
            Text("了解详情")
                .bold()
        }
    }
}

// 使用时
TipView(premiumTip)
    .tipBackground(
        LinearGradient(
            colors: [.purple, .blue],
            startPoint: .leading,
            endPoint: .trailing
        )
    )
    .foregroundStyle(.white)
```

### 5.5 样式定制速查表

| 定制项 | 修饰符/属性 | 示例 |
|--------|-------------|------|
| 背景色 | `.tipBackground()` | `.tipBackground(.yellow)` |
| 渐变背景 | `.tipBackground()` | `.tipBackground(.blue.gradient)` |
| 文字颜色 | `.foregroundStyle()` | `.foregroundStyle(.white)` |
| 动作按钮 | `actions` 属性 | `Action(id: "x") { Text("按钮") }` |
| 图片 | `image` 属性 | `Image(systemName: "star")` |

---

## 6. 控制显示逻辑

### 6.1 已读状态（自动管理）

TipKit 会自动持久化 Tip 的显示状态。用户关闭一个 Tip 后，下次启动 App 不会再显示。

```swift
// 在 App 启动时配置 TipKit
import TipKit

@main
struct WeatherApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
                .task {
                    // 初始化 TipKit，自动加载已读状态
                    try? Tips.configure()
                }
        }
    }
}
```

> 💡 `Tips.configure()` 只需调用一次，通常放在 App 的根视图 `.task` 中。

### 6.2 手动关闭 Tip

有时你想在代码中主动关闭某个 Tip：

```swift
struct WeatherListView: View {
    let addCityTip = AddCityTip()

    var body: some View {
        List {
            // ...
        }
        .popoverTip(addCityTip)
        .onChange(of: cityCount) { _, newValue in
            if newValue > 0 {
                // 用户已添加城市，不再需要这个提示
                addCityTip.invalidate(reason: .actionPerformed)
            }
        }
    }
}
```

| 方法 | 说明 |
|------|------|
| `invalidate(reason: .actionPerformed)` | 用户已执行相关操作，标记为已完成 |
| `invalidate(reason: .tipClosed)` | 用户关闭了提示 |
| `invalidate(reason: .clearAll)` | 清除所有提示状态 |

### 6.3 重置所有提示（调试用）

开发阶段，你可能想重置所有提示的已读状态：

```swift
.task {
    try? Tips.configure {
        // 重置所有提示数据（仅调试用！）
        DisplayFrequency(.immediate)
        DatastoreLocation(.applicationDefault)
    }
}
```

也可以在 Xcode 调试时通过菜单操作：

> **Debug → Reset TipKit Data** （仅模拟器/真机调试时可用）

### 6.4 控制显示频率

```swift
try? Tips.configure {
    // 每次都立即显示（调试用）
    DisplayFrequency(.immediate)

    // 每天最多显示一次提示
    DisplayFrequency(.daily)

    // 自定义间隔（单位：秒）
    DisplayFrequency(3600) // 每小时最多一次
}
```

| 频率设置 | 说明 |
|----------|------|
| `.immediate` | 无限制，适合调试 |
| `.daily` | 每天最多显示一个 Tip |
| 自定义秒数 | 自定义间隔 |

> ⚠️ `DisplayFrequency` 控制的是全局频率，不是单个 Tip 的频率。如果设为 `.daily`，那么一天内用户只会看到第一个满足条件的 Tip。

### 6.5 A/B 展示

如果你想让不同用户看到不同的提示内容（A/B 测试），可以这样实现：

```swift
struct OnboardingTip: Tip {
    let variant: String

    var title: Text {
        switch variant {
        case "A":
            return Text("快速上手")
        case "B":
            return Text("3步学会使用")
        default:
            return Text("开始使用")
        }
    }

    var message: Text? {
        switch variant {
        case "A":
            return Text("浏览底部标签栏，探索所有功能")
        case "B":
            return Text("1.添加城市 → 2.查看预报 → 3.设置提醒")
        default:
            return Text("探索 App 的各项功能")
        }
    }
}

// 使用时随机选择
let variant = Bool.random() ? "A" : "B"
let onboardingTip = OnboardingTip(variant: variant)
```

> 💡 真正的 A/B 测试通常需要服务端配合，这里只是本地简单实现。生产环境建议使用远程配置来控制变体分配。

---

## 7. 实战示例：给天气 App 添加新手引导

### 7.1 场景描述

我们有一个天气 App，需要引导用户：

1. 首次打开时，提示"添加你的第一个城市"
2. 添加城市后，提示"左右滑动查看逐小时预报"
3. 查看预报后，提示"长按可以添加天气提醒"

### 7.2 定义所有 Tip

```swift
import TipKit

// Tip 1：添加城市
struct AddFirstCityTip: Tip {
    var title: Text {
        Text("添加你的第一个城市")
    }

    var message: Text? {
        Text("点击右上角的 + 号，搜索并添加你所在的城市")
    }

    var image: Image? {
        Image(systemName: "location.circle.fill")
    }

    var actions: [Action] {
        Action(id: "addNow") {
            Text("立即添加")
        }
    }

    var options: [TipOption] {
        Tips.MaxDisplayCount(3)
    }
}

// Tip 2：逐小时预报
struct HourlyForecastTip: Tip {
    var title: Text {
        Text("逐小时预报")
    }

    var message: Text? {
        Text("左右滑动可以查看未来24小时的温度和天气变化")
    }

    var image: Image? {
        Image(systemName: "clock.fill")
    }

    var rules: [Rule] {
        Rule(#Predicate<Tips.Status> {
            $0.donations.contains(where: { $0.id == "addedFirstCity" })
        })
    }
}

// Tip 3：天气提醒
struct WeatherAlertTip: Tip {
    var title: Text {
        Text("设置天气提醒")
    }

    var message: Text? {
        Text("长按某天的天气预报，可以添加降雨提醒，出门不再忘带伞")
    }

    var image: Image? {
        Image(systemName: "bell.badge.fill")
    }

    var rules: [Rule] {
        Rule(#Predicate<Tips.Status> {
            $0.donations.contains(where: { $0.id == "viewedHourlyForecast" })
        })
    }
}
```

### 7.3 配置 TipKit

```swift
import SwiftUI
import TipKit

@main
struct WeatherApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
                .task {
                    do {
                        try Tips.configure {
                            DisplayFrequency(.immediate)
                            DatastoreLocation(.applicationDefault)
                        }
                    } catch {
                        print("TipKit 配置失败: \(error)")
                    }
                }
        }
    }
}
```

### 7.4 主页面：添加城市提示

```swift
struct WeatherListView: View {
    @State private var cities: [String] = []
    @State private var showAddCity = false

    let addFirstCityTip = AddFirstCityTip()

    var body: some View {
        NavigationStack {
            Group {
                if cities.isEmpty {
                    VStack(spacing: 16) {
                        Image(systemName: "cloud.sun")
                            .font(.system(size: 60))
                            .foregroundStyle(.secondary)
                        Text("还没有添加城市")
                            .font(.title3)
                            .foregroundStyle(.secondary)

                        // 👇 内嵌提示：引导添加第一个城市
                        TipView(addFirstCityTip) { action in
                            if action.id == "addNow" {
                                showAddCity = true
                            }
                        }
                        .padding(.horizontal)
                    }
                } else {
                    List(cities, id: \.self) { city in
                        NavigationLink(value: city) {
                            Text(city)
                        }
                    }
                }
            }
            .navigationTitle("天气")
            .toolbar {
                if !cities.isEmpty {
                    ToolbarItem(placement: .topBarTrailing) {
                        Button {
                            showAddCity = true
                        } label: {
                            Image(systemName: "plus")
                        }
                    }
                }
            }
            .sheet(isPresented: $showAddCity) {
                AddCityView { city in
                    cities.append(city)
                    // 👇 捐赠事件：已添加第一个城市
                    addFirstCityTip.invalidate(reason: .actionPerformed)
                    Tips.Donate(reason: .actionPerformed, id: "addedFirstCity")
                }
            }
        }
    }
}
```

### 7.5 详情页面：逐小时预报提示

```swift
struct WeatherDetailView: View {
    let city: String
    let hourlyForecastTip = HourlyForecastTip()

    var body: some View {
        ScrollView {
            VStack(alignment: .leading, spacing: 20) {
                Text(city)
                    .font(.largeTitle)
                    .bold()

                // 👇 逐小时预报区域 + Popover 提示
                ScrollView(.horizontal, showsIndicators: false) {
                    HStack(spacing: 16) {
                        ForEach(0..<24, id: \.self) { hour in
                            VStack {
                                Text("\(hour):00")
                                    .font(.caption)
                                Image(systemName: "cloud.sun")
                                Text("\(20 + hour % 5)°")
                                    .bold()
                            }
                            .frame(width: 60)
                        }
                    }
                }
                .popoverTip(hourlyForecastTip, arrowEdge: .top) { action in
                    // 用户点击了提示上的按钮
                }
            }
            .padding()
        }
        .onAppear {
            // 👇 捐赠事件：已查看逐小时预报
            Tips.Donate(reason: .actionPerformed, id: "viewedHourlyForecast")
        }
    }
}
```

### 7.6 七日预报页面：天气提醒提示

```swift
struct WeeklyForecastView: View {
    let weatherAlertTip = WeatherAlertTip()

    let days = ["周一", "周二", "周三", "周四", "周五", "周六", "周日"]

    var body: some View {
        VStack(alignment: .leading, spacing: 12) {
            // 👇 内嵌提示：设置天气提醒
            TipView(weatherAlertTip)
                .padding(.horizontal)

            List(days, id: \.self) { day in
                HStack {
                    Text(day)
                    Spacer()
                    Image(systemName: "sun.max")
                    Text("28° / 18°")
                }
            }
        }
        .navigationTitle("七日预报")
    }
}
```

### 7.7 完整流程回顾

| 步骤 | 用户行为 | 触发的 Tip | 触发方式 |
|------|----------|------------|----------|
| 1 | 首次打开 App | "添加你的第一个城市" | Inline（空页面内嵌） |
| 2 | 添加城市后进入详情 | "逐小时预报" | Popover（指向横向滚动区） |
| 3 | 查看逐小时预报后 | "设置天气提醒" | Inline（七日预报页顶部） |

> 💡 整个引导链是串联的：每一步完成后才触发下一步，不会一次性弹出所有提示。

---

## 8. 设计最佳实践

### 8.1 提示时机

| ✅ 好的时机 | ❌ 差的时机 |
|------------|------------|
| 用户首次遇到某个功能时 | App 刚启动就弹出 5 个提示 |
| 用户可能需要帮助时 | 用户正在快速操作时 |
| 新功能上线时 | 用户已经熟悉功能后 |
| 相关操作刚完成后 | 与当前页面无关 |

**核心原则：在用户需要的时候出现，而不是你想的时候。**

### 8.2 文案原则

| 原则 | 好的示例 | 差的示例 |
|------|----------|----------|
| 简短 | "左右滑动查看更多" | "您可以通过在屏幕上进行从左到右或从右到左的滑动手势来查看更多的天气预报信息" |
| 具体可操作 | "点击 + 添加城市" | "本应用支持城市管理功能" |
| 说好处 | "提前知道会不会下雨" | "天气提醒功能已上线" |
| 友好语气 | "试试左右滑动" | "请务必左右滑动" |

### 8.3 不要过度提示

> ⚠️ **Tip 是调味料，不是主菜。** 加一点提味，加太多就毁了整道菜。

具体建议：

1. **每个页面最多 1 个 Tip**——使用 `Tips.Group(maxDisplayCount: 1)`
2. **设置最大显示次数**——`Tips.MaxDisplayCount(3)` 足够了
3. **控制全局频率**——不要让用户每次打开 App 都看到提示
4. **用户关闭后尊重选择**——不要换个 Tip 再说一遍同样的事
5. **已有引导页的不再重复**——如果 App 已有 Onboarding 流程，不要再用 Tip 重复

### 8.4 提示层级设计

```
优先级从高到低：
┌─────────────────────────────┐
│ 1. 关键操作提示（必须知道）    │  如：保存按钮在哪
├─────────────────────────────┤
│ 2. 效率提升提示（知道更好）    │  如：快捷键、手势
├─────────────────────────────┤
│ 3. 新功能介绍（可选了解）      │  如：新上线的雷达图
├─────────────────────────────┤
│ 4. 高级功能（感兴趣再看）      │  如：自定义通知规则
└─────────────────────────────┘
```

优先展示层级 1 的提示，低层级的提示等用户更熟悉 App 后再展示。

### 8.5 常见错误与修正

| 常见错误 | 修正方案 |
|----------|----------|
| 所有 Tip 同时弹出 | 用 `rules` 控制触发条件，用 `Tips.Group` 限制同时显示数量 |
| 提示文案太长 | 标题 ≤ 8 个字，消息 ≤ 2 行 |
| 提示指向错误位置 | 确保 Popover 附着在正确的视图上 |
| 每次打开 App 都显示 | 检查 `Tips.configure()` 是否正确调用，确保持久化生效 |
| 调试时看不到 Tip | 使用 `DisplayFrequency(.immediate)` 并重置数据 |

---

## 小结

| 知识点 | 核心要点 |
|--------|----------|
| TipKit 是什么 | iOS 17+ 官方提示框架，优雅引导用户 |
| 定义 Tip | 遵循 `Tip` 协议，提供 `title` / `message` / `image` |
| Popover 展示 | `.popoverTip()` 附着在目标元素上，带箭头 |
| Inline 展示 | `TipView()` 嵌入页面布局，不遮挡内容 |
| 触发条件 | `rules` + `#Predicate`，配合 Donation 事件 |
| 显示控制 | `MaxDisplayCount`、`Tips.Group`、`DisplayFrequency` |
| 样式定制 | `tipBackground()`、`actions`、`foregroundStyle()` |
| 显示逻辑 | 自动持久化、`invalidate()`、A/B 测试 |
| 最佳实践 | 合适时机、简短文案、不要过度提示 |

> 💡 **记住：好的提示就像好的导航——在你需要的时候出现，在你不需要的时候隐身。** TipKit 让这件事变得简单，但"什么时候该提示"的设计决策，永远需要站在用户角度思考。

下一章我们将学习另一个实用话题，继续加油！🚀
