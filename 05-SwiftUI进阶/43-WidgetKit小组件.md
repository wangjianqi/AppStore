# 43-WidgetKit 小组件

## 本章目标

- 理解 WidgetKit 小组件的概念与工作原理
- 掌握创建小组件的完整流程
- 学会使用 Timeline Provider 控制小组件刷新
- 能够为不同尺寸设计适配的小组件 UI
- 实现 App 与小组件之间的数据共享
- 了解 iOS 17+ 交互式小组件的新特性
- 完成一个天气小组件的实战项目

---

## 1. WidgetKit 简介

### 1.1 什么是小组件？

小组件（Widget）就是你在 iPhone 主屏幕上看到的那些"小卡片"——天气、日历、电池……它们能让用户**不打开 App 就能快速查看关键信息**。

打个比方：App 就像一栋房子，你需要推门进去才能看到里面的东西；而小组件就像房子外面的**橱窗**，路过就能一眼看到最重要的信息。

| 对比项 | App | 小组件 |
|--------|-----|--------|
| 交互方式 | 完整交互（点击、滑动、输入等） | 有限交互（展示为主，iOS 17+ 支持按钮/开关） |
| 运行方式 | 前台活跃运行 | 后台定时刷新，不持续运行 |
| 刷新频率 | 实时 | 由系统控制，最少 15 分钟间隔 |
| 内存限制 | 较宽裕 | 约 30MB，非常严格 |
| 生命周期 | 用户控制 | 系统控制 |

### 1.2 小组件与 App Extension 的关系

小组件本质上是一个 **App Extension**（应用扩展）。它不是独立的 App，而是依附于主 App 的一个"插件"。

```
主 App（Host App）
  ├── 主应用代码
  └── Widget Extension（小组件扩展）
        ├── 小组件 UI
        └── Timeline Provider
```

> 💡 关键理解：小组件和主 App 是**两个独立的进程**，它们不能直接共享内存，必须通过 App Group 等机制来共享数据。

### 1.3 系统要求

| 特性 | 最低系统版本 |
|------|-------------|
| 基础小组件 | iOS 14+ |
| 小组件配置（Configurable Widget） | iOS 14+ |
| 交互式小组件（Button/Toggle） | iOS 17+ |
| App Intent 配置 | iOS 17+ |

---

## 2. 创建第一个小组件

### 2.1 在 Xcode 中添加 Widget Extension

步骤如下：

1. 打开你的 Xcode 项目
2. 菜单栏 → **File → New → Target**
3. 选择 **Widget Extension**，点击 Next
4. 填写 Product Name（如 `MyWidget`）
5. 取消勾选 "Include Live Activity"（暂时不需要）
6. 点击 Finish，Xcode 会自动生成小组件代码

> ⚠️ 如果勾选了 "Include Configuration App Intent"，Xcode 会生成可配置小组件的代码。初学阶段建议先不勾选，从最简单的开始。

### 2.2 项目结构解析

添加 Widget Extension 后，项目中会多出一个文件夹，结构如下：

```
MyWidget/
  ├── MyWidget.swift          ← 小组件入口 & UI
  ├── MyWidgetBundle.swift    ← Bundle，注册所有小组件
  ├── MyWidgetLiveActivity.swift  ← Live Activity（暂不关注）
  └── Info.plist              ← 扩展配置信息
```

Xcode 自动生成的核心文件内容：

```swift
// MyWidgetBundle.swift - 小组件的"注册表"
@main
struct MyWidgetBundle: WidgetBundle {
    var body: some Widget {
        MyWidget()
    }
}
```

```swift
// MyWidget.swift - 小组件主体
struct MyWidget: Widget {
    let kind: String = "MyWidget"

    var body: some WidgetConfiguration {
        StaticConfiguration(kind: kind, provider: SimpleProvider()) { entry in
            MyWidgetEntryView(entry: entry)
        }
        .configurationDisplayName("我的小组件")
        .description("这是一个示例小组件")
        .supportedFamilies([.systemSmall, .systemMedium, .systemLarge])
    }
}
```

用表格理解各部分职责：

| 组成部分 | 职责 | 类比 |
|----------|------|------|
| `WidgetBundle` | 注册所有小组件，标记 `@main` | 书架，摆放所有书 |
| `Widget` | 定义小组件配置（名称、描述、尺寸） | 书的封面和目录 |
| `TimelineProvider` | 提供数据和刷新时间 | 定时送报员 |
| `EntryView` | 小组件的 UI 界面 | 书的内容 |

---

## 3. Timeline Provider 详解

Timeline Provider 是小组件的**数据引擎**——它决定小组件"显示什么"以及"什么时候刷新"。

### 3.1 三个核心方法

```swift
struct SimpleProvider: TimelineProvider {
    // 1. 占位视图 - 当系统无法获取数据时的"兜底"显示
    func placeholder(in context: Context) -> SimpleEntry {
        SimpleEntry(date: Date(), title: "加载中…")
    }

    // 2. 快照 - 在小组件选择库中展示的预览
    func getSnapshot(in context: Context, completion: @escaping (SimpleEntry) -> Void) {
        let entry = SimpleEntry(date: Date(), title: "预览数据")
        completion(entry)
    }

    // 3. 时间线 - 定义一组未来时刻的数据，系统按时间线依次展示
    func getTimeline(in context: Context, completion: @escaping (Timeline<SimpleEntry>) -> Void) {
        var entries: [SimpleEntry] = []

        let currentDate = Date()
        for hourOffset in 0 ..< 5 {
            let entryDate = Calendar.current.date(byAdding: .hour, value: hourOffset, to: currentDate)!
            let entry = SimpleEntry(date: entryDate, title: "第\(hourOffset)小时")
            entries.append(entry)
        }

        let timeline = Timeline(entries: entries, policy: .atEnd)
        completion(timeline)
    }
}
```

三个方法的调用时机：

| 方法 | 调用时机 | 必须实现 | 类比 |
|------|---------|---------|------|
| `placeholder` | 系统首次加载、无数据时 | ✅ | 餐厅的"样品菜单" |
| `getSnapshot` | 用户在小组件选择库中预览时 | ✅ | 餐厅门口的"招牌菜展示" |
| `getTimeline` | 小组件被添加到桌面后，系统请求更新数据时 | ✅ | 厨师按订单依次上菜 |

### 3.2 TimelineEntry

每个 Entry 代表小组件在**某个时刻**要显示的数据：

```swift
struct SimpleEntry: TimelineEntry {
    let date: Date       // 必须有：告诉系统这条数据在什么时间显示
    let title: String    // 自定义：你的业务数据
    let subtitle: String? // 自定义：可选数据
}
```

> 💡 `date` 是 `TimelineEntry` 协议的**唯一必须属性**，其他属性完全由你自定义。

### 3.3 Timeline Reload Policy（刷新策略）

在 `getTimeline` 返回的 `Timeline` 中，你可以指定刷新策略：

```swift
let timeline = Timeline(entries: entries, policy: .atEnd)
```

| 策略 | 含义 | 使用场景 |
|------|------|---------|
| `.atEnd` | 时间线中最后一个 Entry 显示完后，系统重新请求 | 大多数场景（推荐） |
| `.never` | 永远不再请求更新 | 静态内容（如名言警句） |
| `.after(date)` | 在指定日期后请求更新 | 精确控制刷新时间（如倒计时） |

```swift
// 示例：5 分钟后刷新
let nextUpdate = Calendar.current.date(byAdding: .minute, value: 5, to: Date())!
let timeline = Timeline(entries: entries, policy: .after(nextUpdate))
```

> ⚠️ 刷新策略只是**建议**，系统会根据电量、使用频率等因素决定实际刷新时间。你请求 5 分钟刷新，系统可能 15 分钟后才刷新。

---

## 4. 小组件 UI 设计

### 4.1 WidgetFamily（小组件尺寸）

iOS 提供了三种标准尺寸：

| 尺寸 | 枚举值 | 网格占用 | 适合展示 |
|------|--------|---------|---------|
| 小 | `.systemSmall` | 2×2 | 单一核心信息（温度、步数） |
| 中 | `.systemMedium` | 4×2 | 信息 + 简要描述（天气 + 城市） |
| 大 | `.systemLarge` | 4×4 | 多条信息列表（待办事项） |

iOS 16+ 新增：

| 尺寸 | 枚举值 | 网格占用 | 适合展示 |
|------|--------|---------|---------|
| 超大 | `.systemExtraLarge` | 4×4（iPad 上更大） | 详细信息展示 |

### 4.2 SwiftUI 视图适配不同尺寸

```swift
struct MyWidgetEntryView: View {
    var entry: SimpleEntry
    @Environment(\.widgetFamily) var family

    var body: some View {
        switch family {
        case .systemSmall:
            SmallView(entry: entry)
        case .systemMedium:
            MediumView(entry: entry)
        case .systemLarge:
            LargeView(entry: entry)
        default:
            SmallView(entry: entry)
        }
    }
}

struct SmallView: View {
    let entry: SimpleEntry

    var body: some View {
        VStack {
            Text(entry.title)
                .font(.headline)
            Text(entry.date, style: .time)
                .font(.caption)
        }
    }
}

struct MediumView: View {
    let entry: SimpleEntry

    var body: some View {
        HStack {
            VStack(alignment: .leading) {
                Text(entry.title)
                    .font(.headline)
                Text(entry.subtitle ?? "")
                    .font(.subheadline)
            }
            Spacer()
            Text(entry.date, style: .time)
        }
        .padding()
    }
}

struct LargeView: View {
    let entry: SimpleEntry

    var body: some View {
        VStack(alignment: .leading, spacing: 12) {
            Text(entry.title)
                .font(.title2)
                .fontWeight(.bold)
            Divider()
            ForEach(0..<5) { i in
                HStack {
                    Text("项目 \(i + 1)")
                    Spacer()
                    Text("详情")
                        .foregroundColor(.secondary)
                }
            }
        }
        .padding()
    }
}
```

### 4.3 小组件的内边距与安全区

> ⚠️ 小组件有系统自动添加的**内边距**，你不需要手动添加外边距。系统会在你的视图外面套一层圆角矩形。

```swift
// ❌ 错误：不要手动设置外边距，系统已有
VStack {
    Text("Hello")
}
.padding(20)  // 多余！系统已有内边距

// ✅ 正确：只需关注内容布局
VStack {
    Text("Hello")
}
```

### 4.4 常用 SwiftUI 修饰符适配

| 修饰符 | 说明 | 小组件中可用 |
|--------|------|-------------|
| `.containerBackground(for: .widget)` | iOS 17+ 设置小组件背景 | ✅ |
| `.widgetURL(url)` | 点击小组件跳转的 URL（小尺寸仅支持整个点击） | ✅ |
| `.Link(url:, label:)` | 中/大尺寸的不同区域点击跳转 | ✅ |
| `.font()` | 字体设置 | ✅ |
| `.foregroundColor()` | 前景色 | ✅ |

iOS 17+ 设置背景的方式：

```swift
struct MyWidgetEntryView: View {
    var entry: SimpleEntry

    var body: some View {
        VStack {
            Text(entry.title)
                .font(.title)
            Text(entry.date, style: .time)
        }
        .containerBackground(for: .widget) {
            Color.blue.opacity(0.2)
        }
    }
}
```

> 💡 iOS 17 之前，小组件背景是透明的，系统会自动添加黑色/白色背景。iOS 17+ 使用 `.containerBackground` 来明确控制背景。

---

## 5. 数据共享

### 5.1 为什么需要数据共享？

小组件和主 App 运行在**不同的进程**中，它们无法直接访问彼此的内存或文件。就像两个人住在两栋不同的楼里，要传递东西必须通过"快递"——这个快递就是 **App Group**。

```
┌─────────────┐     App Group     ┌─────────────────┐
│   主 App     │ ◄───────────────► │  Widget Extension │
│  (进程 A)    │   共享容器/UserDefaults  │   (进程 B)       │
└─────────────┘                   └─────────────────┘
```

### 5.2 配置 App Group

**步骤一：在 Apple Developer 中创建 App Group**

1. 登录 [Apple Developer](https://developer.apple.com)
2. Certificates, Identifiers & Profiles → Identifiers
3. 选择你的 App ID，编辑 → App Groups → 添加
4. Group Identifier 格式：`group.你的团队ID.应用名`（如 `group.com.example.myapp`）

**步骤二：在 Xcode 中开启 App Group**

1. 选择主 App Target → Signing & Capabilities → + Capability → App Groups
2. 添加刚才创建的 Group ID
3. 对 Widget Extension Target 重复同样操作，添加**同一个** Group ID

> ⚠️ 主 App 和 Widget Extension 必须使用**同一个 App Group ID**，否则无法共享数据！

### 5.3 使用 UserDefaults 共享数据

```swift
// 共享的 UserDefaults 工具类
struct SharedDefaults {
    static let appGroupID = "group.com.example.myapp"

    static var shared: UserDefaults {
        UserDefaults(suiteName: appGroupID)!
    }

    static let temperatureKey = "temperature"
    static let cityKey = "city"
}
```

**主 App 中写入数据：**

```swift
// 在主 App 中保存数据
SharedDefaults.shared.set("北京", forKey: SharedDefaults.cityKey)
SharedDefaults.shared.set(25, forKey: SharedDefaults.temperatureKey)
```

**小组件中读取数据：**

```swift
struct WeatherProvider: TimelineProvider {
    func placeholder(in context: Context) -> WeatherEntry {
        WeatherEntry(date: Date(), city: "北京", temperature: 25)
    }

    func getSnapshot(in context: Context, completion: @escaping (WeatherEntry) -> Void) {
        let city = SharedDefaults.shared.string(forKey: SharedDefaults.cityKey) ?? "未知"
        let temp = SharedDefaults.shared.integer(forKey: SharedDefaults.temperatureKey)
        completion(WeatherEntry(date: Date(), city: city, temperature: temp))
    }

    func getTimeline(in context: Context, completion: @escaping (Timeline<WeatherEntry>) -> Void) {
        let city = SharedDefaults.shared.string(forKey: SharedDefaults.cityKey) ?? "未知"
        let temp = SharedDefaults.shared.integer(forKey: SharedDefaults.temperatureKey)

        let entry = WeatherEntry(date: Date(), city: city, temperature: temp)
        let timeline = Timeline(entries: [entry], policy: .atEnd)
        completion(timeline)
    }
}
```

### 5.4 使用文件共享数据

对于更复杂的数据（如图片、JSON 文件），可以使用共享文件容器：

```swift
struct SharedFileManager {
    static let appGroupID = "group.com.example.myapp"

    static var sharedContainerURL: URL {
        FileManager.default.containerURL(forSecurityApplicationGroupIdentifier: appGroupID)!
    }

    static func saveImage(_ image: UIImage, name: String) {
        guard let data = image.pngData() else { return }
        let fileURL = sharedContainerURL.appendingPathComponent(name)
        try? data.write(to: fileURL)
    }

    static func loadImage(name: String) -> UIImage? {
        let fileURL = sharedContainerURL.appendingPathComponent(name)
        guard let data = try? Data(contentsOf: fileURL) else { return nil }
        return UIImage(data: data)
    }
}
```

| 共享方式 | 适合数据 | 优点 | 缺点 |
|----------|---------|------|------|
| UserDefaults (App Group) | 简单键值对 | 简单易用 | 不适合大量数据 |
| 共享文件容器 | 图片、JSON、数据库 | 灵活，支持大文件 | 需要处理文件读写 |
| Keychain (App Group) | 敏感信息（token） | 安全加密 | API 较复杂 |

---

## 6. 交互式小组件（iOS 17+）

### 6.1 从"只看"到"可操作"

iOS 17 之前，小组件只能**展示信息**，点击只能跳转到 App。iOS 17 引入了交互式小组件，用户可以直接在小组件上**点击按钮、切换开关**。

类比：以前的小组件像一张**海报**（只能看），现在的小组件像一个**遥控器**（可以按）。

### 6.2 Button 交互

```swift
import WidgetKit
import AppIntents

struct ToggleTaskIntent: AppIntent {
    static var title: LocalizedStringResource = "切换任务状态"

    @Parameter(title: "任务ID")
    var taskID: String

    func perform() async throws -> some IntentResult {
        // 切换任务的完成状态
        var tasks = TaskStore.loadTasks()
        if let index = tasks.firstIndex(where: { $0.id == taskID }) {
            tasks[index].isCompleted.toggle()
            TaskStore.saveTasks(tasks)
        }
        // 请求刷新小组件
        WidgetCenter.shared.reloadAllTimelines()
        return .result()
    }
}
```

```swift
struct TaskWidgetEntryView: View {
    var entry: TaskEntry

    var body: some View {
        VStack(alignment: .leading) {
            ForEach(entry.tasks) { task in
                HStack {
                    // 交互式按钮
                    Button(intent: ToggleTaskIntent(taskID: task.id)) {
                        Image(systemName: task.isCompleted ? "checkmark.circle.fill" : "circle")
                    }
                    .buttonStyle(.plain)

                    Text(task.title)
                        .strikethrough(task.isCompleted)
                }
            }
        }
        .containerBackground(for: .widget) {
            Color.white
        }
    }
}
```

### 6.3 Toggle 交互

```swift
struct ToggleReminderIntent: AppIntent {
    static var title: LocalizedStringResource = "切换提醒"

    @Parameter(title: "提醒ID")
    var reminderID: String

    func perform() async throws -> some IntentResult {
        var reminders = ReminderStore.load()
        if let index = reminders.firstIndex(where: { $0.id == reminderID }) {
            reminders[index].isOn.toggle()
            ReminderStore.save(reminders)
        }
        WidgetCenter.shared.reloadAllTimelines()
        return .result()
    }
}
```

```swift
struct ReminderWidgetView: View {
    var entry: ReminderEntry

    var body: some View {
        VStack {
            ForEach(entry.reminders) { reminder in
                Toggle(reminder.title, isOn: true)
                    .toggledIntent(ToggleReminderIntent(reminderID: reminder.id))
            }
        }
        .containerBackground(for: .widget) {
            Color(UIColor.systemBackground)
        }
    }
}
```

### 6.4 AppIntent 核心要素

| 要素 | 说明 | 示例 |
|------|------|------|
| `static var title` | Intent 的显示名称 | `"切换任务状态"` |
| `@Parameter` | 传递给 Intent 的参数 | `taskID: String` |
| `perform()` | 执行具体操作的方法 | 切换状态、保存数据 |
| `WidgetCenter.shared.reloadAllTimelines()` | 操作完成后刷新小组件 | 必须调用，否则 UI 不会更新 |

> 💡 交互式小组件的 Intent 执行完毕后，**必须手动调用** `WidgetCenter.shared.reloadAllTimelines()` 来刷新 UI，系统不会自动刷新。

---

## 7. 实战示例：天气小组件

让我们从零实现一个完整的天气小组件，涵盖数据获取、缓存、UI 展示的完整流程。

### 7.1 数据模型

```swift
struct WeatherData: Codable {
    let city: String
    let temperature: Int
    let condition: String
    let icon: String
    let humidity: Int
    let forecast: [ForecastItem]

    struct ForecastItem: Codable {
        let day: String
        let high: Int
        let low: Int
        let icon: String
    }

    static let placeholder = WeatherData(
        city: "北京",
        temperature: 25,
        condition: "晴",
        icon: "sun.max.fill",
        humidity: 45,
        forecast: [
            ForecastItem(day: "明天", high: 27, low: 18, icon: "cloud.sun.fill"),
            ForecastItem(day: "后天", high: 23, low: 16, icon: "cloud.rain.fill"),
            ForecastItem(day: "大后天", high: 26, low: 17, icon: "sun.max.fill")
        ]
    )
}
```

### 7.2 数据管理器

```swift
struct WeatherManager {
    static let appGroupID = "group.com.example.myapp"
    static let weatherKey = "cachedWeather"

    static var sharedDefaults: UserDefaults {
        UserDefaults(suiteName: appGroupID)!
    }

    static func save(_ data: WeatherData) {
        if let encoded = try? JSONEncoder().encode(data) {
            sharedDefaults.set(encoded, forKey: weatherKey)
        }
    }

    static func load() -> WeatherData {
        guard let data = sharedDefaults.data(forKey: weatherKey),
              let weather = try? JSONDecoder().decode(WeatherData.self, from: data) else {
            return .placeholder
        }
        return weather
    }
}
```

### 7.3 Timeline Entry

```swift
struct WeatherEntry: TimelineEntry {
    let date: Date
    let weather: WeatherData
}
```

### 7.4 Timeline Provider

```swift
struct WeatherTimelineProvider: TimelineProvider {
    func placeholder(in context: Context) -> WeatherEntry {
        WeatherEntry(date: Date(), weather: .placeholder)
    }

    func getSnapshot(in context: Context, completion: @escaping (WeatherEntry) -> Void) {
        let weather = WeatherManager.load()
        completion(WeatherEntry(date: Date(), weather: weather))
    }

    func getTimeline(in context: Context, completion: @escaping (Timeline<WeatherEntry>) -> Void) {
        let weather = WeatherManager.load()
        let entry = WeatherEntry(date: Date(), weather: weather)

        let nextUpdate = Calendar.current.date(byAdding: .hour, value: 1, to: Date())!
        let timeline = Timeline(entries: [entry], policy: .after(nextUpdate))
        completion(timeline)
    }
}
```

### 7.5 小组件 UI

```swift
struct WeatherWidgetEntryView: View {
    var entry: WeatherEntry
    @Environment(\.widgetFamily) var family

    var body: some View {
        switch family {
        case .systemSmall:
            SmallWeatherView(weather: entry.weather)
        case .systemMedium:
            MediumWeatherView(weather: entry.weather)
        case .systemLarge:
            LargeWeatherView(weather: entry.weather)
        default:
            SmallWeatherView(weather: entry.weather)
        }
    }
}

struct SmallWeatherView: View {
    let weather: WeatherData

    var body: some View {
        VStack(spacing: 4) {
            Image(systemName: weather.icon)
                .font(.title)
                .foregroundColor(.orange)
            Text("\(weather.temperature)°")
                .font(.system(size: 36, weight: .bold))
            Text(weather.city)
                .font(.caption)
                .foregroundColor(.secondary)
        }
        .containerBackground(for: .widget) {
            LinearGradient(
                colors: [Color.blue.opacity(0.3), Color.cyan.opacity(0.2)],
                startPoint: .top,
                endPoint: .bottom
            )
        }
    }
}

struct MediumWeatherView: View {
    let weather: WeatherData

    var body: some View {
        HStack {
            VStack(alignment: .leading, spacing: 4) {
                Text(weather.city)
                    .font(.headline)
                Text("\(weather.temperature)°")
                    .font(.system(size: 40, weight: .bold))
                Text(weather.condition)
                    .font(.subheadline)
                    .foregroundColor(.secondary)
                Text("湿度 \(weather.humidity)%")
                    .font(.caption)
                    .foregroundColor(.secondary)
            }
            Spacer()
            Image(systemName: weather.icon)
                .font(.system(size: 50))
                .foregroundColor(.orange)
        }
        .padding()
        .containerBackground(for: .widget) {
            LinearGradient(
                colors: [Color.blue.opacity(0.3), Color.cyan.opacity(0.2)],
                startPoint: .topLeading,
                endPoint: .bottomTrailing
            )
        }
    }
}

struct LargeWeatherView: View {
    let weather: WeatherData

    var body: some View {
        VStack(alignment: .leading, spacing: 8) {
            HStack {
                Image(systemName: weather.icon)
                    .font(.title2)
                    .foregroundColor(.orange)
                Text(weather.city)
                    .font(.headline)
                Spacer()
                Text("\(weather.temperature)°")
                    .font(.system(size: 32, weight: .bold))
            }

            Text("\(weather.condition) · 湿度 \(weather.humidity)%")
                .font(.subheadline)
                .foregroundColor(.secondary)

            Divider()

            Text("未来天气")
                .font(.caption)
                .foregroundColor(.secondary)

            ForEach(weather.forecast, id: \.day) { item in
                HStack {
                    Text(item.day)
                        .frame(width: 60, alignment: .leading)
                    Image(systemName: item.icon)
                        .foregroundColor(.orange)
                        .frame(width: 30)
                    Text("\(item.high)°")
                        .fontWeight(.medium)
                    Text("\(item.low)°")
                        .foregroundColor(.secondary)
                    Spacer()
                }
                .font(.subheadline)
            }
        }
        .padding()
        .containerBackground(for: .widget) {
            LinearGradient(
                colors: [Color.blue.opacity(0.3), Color.cyan.opacity(0.2)],
                startPoint: .topLeading,
                endPoint: .bottomTrailing
            )
        }
    }
}
```

### 7.6 Widget 定义

```swift
struct WeatherWidget: Widget {
    let kind: String = "WeatherWidget"

    var body: some WidgetConfiguration {
        StaticConfiguration(kind: kind, provider: WeatherTimelineProvider()) { entry in
            WeatherWidgetEntryView(entry: entry)
        }
        .configurationDisplayName("天气")
        .description("查看当前天气和未来预报")
        .supportedFamilies([.systemSmall, .systemMedium, .systemLarge])
    }
}
```

### 7.7 主 App 中写入天气数据

```swift
// 主 App 中的天气数据获取
class WeatherViewModel: ObservableObject {
    func fetchAndSaveWeather() {
        // 实际项目中这里调用天气 API
        let weather = WeatherData(
            city: "上海",
            temperature: 28,
            condition: "多云",
            icon: "cloud.sun.fill",
            humidity: 60,
            forecast: [
                WeatherData.ForecastItem(day: "明天", high: 30, low: 22, icon: "sun.max.fill"),
                WeatherData.ForecastItem(day: "后天", high: 25, low: 19, icon: "cloud.rain.fill"),
                WeatherData.ForecastItem(day: "大后天", high: 27, low: 20, icon: "cloud.sun.fill")
            ]
        )

        WeatherManager.save(weather)

        // 通知小组件刷新
        WidgetCenter.shared.reloadAllTimelines()
    }
}
```

### 7.8 注册 Widget Bundle

```swift
@main
struct MyAppWidgetBundle: WidgetBundle {
    var body: some Widget {
        WeatherWidget()
    }
}
```

整体数据流：

```
主 App 获取天气数据
    ↓
保存到 App Group (UserDefaults)
    ↓
调用 WidgetCenter.shared.reloadAllTimelines()
    ↓
系统调用 getTimeline()
    ↓
Provider 从 App Group 读取数据
    ↓
创建 Entry → 渲染 UI
```

---

## 8. 小组件最佳实践

### 8.1 刷新策略

| 策略 | 说明 | 建议 |
|------|------|------|
| 最少 15 分钟间隔 | 系统限制，无法更频繁 | 不要试图绕过此限制 |
| 使用 `.after(date)` | 精确控制下次刷新时间 | 适合有明确更新时间的场景（如倒计时） |
| 主 App 主动刷新 | 数据变化时调用 `reloadAllTimelines()` | 最推荐：数据变了才刷新 |
| 避免频繁刷新 | 每次刷新都消耗电量和流量 | 一天刷新几十次即可，不要每分钟刷新 |

```swift
// ✅ 推荐：主 App 数据变化时主动刷新
func updateData() {
    // ... 更新数据 ...
    WeatherManager.save(newWeather)
    WidgetCenter.shared.reloadAllTimelines()
}

// ❌ 不推荐：在 getTimeline 中请求网络
func getTimeline(in context: Context, completion: @escaping (Timeline<WeatherEntry>) -> Void) {
    // 不要在这里做网络请求！
    // URLSession.shared.dataTask(...)  ← 避免！
}
```

> ⚠️ `getTimeline` 中**不建议做网络请求**。应该在主 App 中获取网络数据，保存到 App Group，然后通知小组件刷新。

### 8.2 性能优化

| 优化项 | 做法 | 原因 |
|--------|------|------|
| 减少视图层级 | 避免嵌套过深的 VStack/HStack | 渲染耗时影响体验 |
| 避免网络请求 | 在主 App 中请求，小组件只读缓存 | 小组件运行时间有限 |
| 控制图片大小 | 使用 `Image` 而非 `AsyncImage` | AsyncImage 在小组件中不可用 |
| 轻量化数据 | 只传递小组件需要的数据 | 减少内存占用 |
| 使用占位图 | `placeholder` 方法返回合理默认值 | 首次加载时不会显示空白 |

### 8.3 设计规范

| 规范 | 说明 |
|------|------|
| 信息密度适中 | 小组件空间有限，只展示最核心的信息 |
| 文字大小合理 | 小尺寸字体不小于 12pt，确保可读性 |
| 避免纯装饰 | 每个元素都应有信息价值 |
| 适配深色模式 | 使用 `Color(UIColor.systemBackground)` 等语义颜色 |
| 圆角自动处理 | 系统自动裁剪圆角，无需手动设置 |
| 不使用动画 | 小组件不支持动画，视图是静态渲染的 |

### 8.4 调试技巧

```swift
// 在 Timeline Provider 中打印调试信息
func getTimeline(in context: Context, completion: @escaping (Timeline<WeatherEntry>) -> Void) {
    let weather = WeatherManager.load()
    print("📍 小组件获取数据: \(weather.city) \(weather.temperature)°")
    // ...
}
```

Xcode 调试步骤：

1. 选择 Widget Extension 的 Scheme 运行
2. Xcode 会弹出选择小组件的界面
3. 选择你的小组件即可调试
4. 在 Console 中查看 `print` 输出

> 💡 也可以在主 App 运行时，通过 **Debug → Attach to Process** 附加到 Widget Extension 进程来调试。

### 8.5 常见问题速查表

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| 小组件不刷新 | 系统限流或未调用 reload | 确保主 App 数据变化时调用 `reloadAllTimelines()` |
| 数据读取为空 | App Group 未正确配置 | 检查两边的 Group ID 是否一致 |
| UI 显示空白 | 视图布局溢出 | 减少内容，确保适配小组件尺寸 |
| 深色模式下看不清 | 使用了硬编码颜色 | 改用语义颜色 |
| 小组件不显示 | Bundle 未注册 | 检查 `@main` 标记的 WidgetBundle |
| iOS 17 背景异常 | 未使用 containerBackground | 添加 `.containerBackground(for: .widget)` |

---

## 小结

本章我们学习了 WidgetKit 小组件开发的完整流程：

1. **WidgetKit 简介**：小组件是 App Extension，运行在独立进程中，用于在桌面展示关键信息
2. **创建小组件**：通过 Xcode 添加 Widget Extension，理解项目结构
3. **Timeline Provider**：三个核心方法（placeholder / getSnapshot / getTimeline）控制数据和时间线
4. **UI 设计**：使用 `@Environment(\.widgetFamily)` 适配小/中/大三种尺寸
5. **数据共享**：通过 App Group + UserDefaults 实现主 App 与小组件的数据传递
6. **交互式小组件**：iOS 17+ 支持 Button/Toggle 交互，通过 AppIntent 处理用户操作
7. **天气实战**：从数据模型到 UI 展示的完整实现
8. **最佳实践**：合理刷新策略、性能优化、设计规范

> 💡 核心原则：小组件是**展示型**组件，不是迷你 App。保持简洁、信息清晰、刷新克制，才能做出好用的小组件。
