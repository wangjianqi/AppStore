# watchOS Complications表盘开发

> 🎯 **本章目标**：掌握 watchOS Complications（表盘小组件）的开发方法，学会使用 WidgetKit 为 Apple Watch 创建表盘组件。

---

## Complications 概述

### 什么是 Complications

Complications（表盘复杂功能/表盘小组件）是 Apple Watch 表盘上的小型信息展示区域，可以显示来自各种 App 的实时数据。用户可以在表盘上一眼看到天气、活动环、日历事件等信息，无需打开 App。

**Complications 在表盘上的位置：**

```text
┌─────────────────────────────────┐
│                                 │
│         10:30                   │  ← 时间
│         周一                     │
│                                 │
│    ┌───────────────────┐        │
│    │   🌤 25°C         │        │  ← Complication（天气）
│    └───────────────────┘        │
│                                 │
│  ┌──────┐          ┌──────┐    │
│  │ 🔴   │          │ 📅   │    │  ← Complication（活动/日历）
│  │活动环 │          │会议   │    │
│  └──────┘          └──────┘    │
│                                 │
└─────────────────────────────────┘
```

### Complications 的价值

| 价值 | 说明 |
|------|------|
| **即时信息** | 用户抬腕即可查看关键信息 |
| **高频触达** | 每次看时间都会看到你的 Complication |
| **App 入口** | 点击 Complication 可直接打开 App |
| **品牌曝光** | 表盘上持续展示你的 App 品牌 |
| **用户粘性** | 有 Complication 的 App 用户留存率更高 |

**Complications 对用户留存的影响：**

| 指标 | 有 Complication | 无 Complication |
|------|:---------------:|:---------------:|
| 7日留存率 | 约 65% | 约 35% |
| 30日留存率 | 约 45% | 约 15% |
| 日活跃率 | 约 55% | 约 20% |
| App 打开频率 | 5-10 次/天 | 1-2 次/天 |

> 💡 **提示**：Complications 是 watchOS App 最重要的功能之一。如果你的 App 有适合在表盘上展示的数据，强烈建议开发 Complication。它不仅能提升用户体验，还能显著提高 App 的留存率。

### Complications 的演进

| 时间 | 技术 | 说明 |
|------|------|------|
| watchOS 2 | ClockKit | 最早的 Complications API，基于协议 |
| watchOS 9 | WidgetKit | 引入 WidgetKit 支持 Complications |
| watchOS 10 | WidgetKit 增强 | 支持 Smart Stack、更多样式 |
| watchOS 11 | WidgetKit 进一步增强 | 改进交互和数据刷新 |

> ⚠️ **警告**：从 watchOS 9 开始，Apple 推荐使用 WidgetKit 开发 Complications。ClockKit 虽然仍然可用，但不再获得新功能更新。新项目应该使用 WidgetKit。

---

## WidgetKit 在 watchOS 上的应用

### WidgetKit vs ClockKit

| 对比维度 | ClockKit | WidgetKit |
|----------|:--------:|:---------:|
| **引入时间** | watchOS 2 | watchOS 9 |
| **UI 技术** | 模板文本 | SwiftUI |
| **开发方式** | 实现协议方法 | Timeline Provider + View |
| **自定义程度** | 有限 | 高度自定义 |
| **跨平台** | ❌ 仅 watchOS | ✅ iOS + watchOS |
| **Smart Stack** | ❌ | ✅ |
| **Apple 推荐** | ❌ 旧技术 | ✅ 推荐方式 |
| **未来支持** | 维护模式 | 持续更新 |

### WidgetKit Complication 架构

```text
┌──────────────────────────────────────────────────────┐
│              WidgetKit Complication 架构               │
│                                                      │
│  ┌──────────────────┐                                │
│  │  Timeline Provider │  提供数据和刷新策略             │
│  │  (TimelineProvider)│                              │
│  └────────┬─────────┘                                │
│           │                                          │
│           ▼                                          │
│  ┌──────────────────┐                                │
│  │  Timeline Entry   │  每个时间点的数据快照            │
│  │  (TimelineEntry)  │                              │
│  └────────┬─────────┘                                │
│           │                                          │
│           ▼                                          │
│  ┌──────────────────┐                                │
│  │  Complication View│  SwiftUI 渲染的表盘视图         │
│  │  (WidgetEntryView)│                              │
│  └──────────────────┘                                │
│                                                      │
│  ┌──────────────────┐                                │
│  │  Widget Configuration │  定义支持的样式和尺寸        │
│  │  (StaticConfiguration) │                         │
│  └──────────────────┘                                │
└──────────────────────────────────────────────────────┘
```

### 创建 WidgetKit Complication 项目

| 步骤 | 操作 | 说明 |
|:----:|------|------|
| 1 | 在 Xcode 中打开 watchOS 项目 | 或创建新项目 |
| 2 | File → New → Target | 添加 Widget Extension |
| 3 | 选择 Widget Extension | 选择 watchOS 平台 |
| 4 | 配置 Widget | 名称、Bundle ID 等 |
| 5 | 勾选 "Include Live Activity" | 如果需要（可选） |
| 6 | 实现 Timeline Provider | 提供数据 |
| 7 | 实现 Complication View | SwiftUI 视图 |

---

## Complication 家族

### Complication 样式类型

watchOS 支持多种 Complication 样式，每种样式对应不同的表盘位置和尺寸：

| 样式 | 尺寸 | 适用表盘 | 说明 |
|------|------|----------|------|
| **circularSmall** | 小圆形 | 多种表盘 | 最小尺寸，仅显示简短文本 |
| **extraLarge** | 超大 | X-Large 表盘 | 最大尺寸，信息丰富 |
| **graphicBezel** | 圆形边框 | Infograph 等 | 圆形区域 + 底部文字 |
| **graphicCircular** | 圆形 | Infograph 等 | 完整圆形区域 |
| **graphicCorner** | 角落 | Infograph 等 | 表盘角落的弧形区域 |
| **modularSmall** | 小模块 | Modular 表盘 | 方形小区域 |
| **modularLarge** | 大模块 | Modular 表盘 | 方形大区域 |
| **utilitarianSmall** | 小实用 | Utility 等 | 窄条小区域 |
| **utilitarianLarge** | 大实用 | Utility 等 | 宽条大区域 |

### WidgetKit 中的 Complication 样式映射

在 WidgetKit 中，Complication 样式通过 `WidgetFamily` 枚举来表示：

| WidgetFamily | 对应 Complication 样式 | 尺寸 |
|--------------|----------------------|------|
| `.accessoryCircular` | graphicCircular | 圆形 |
| `.accessoryCorner` | graphicCorner | 角落 |
| `.accessoryRectangular` | graphicBezel / modularLarge | 矩形 |

> 💡 **提示**：watchOS 9+ 的 WidgetKit Complication 主要使用三种样式：`accessoryCircular`（圆形）、`accessoryCorner`（角落）和 `accessoryRectangular`（矩形）。这三种样式覆盖了绝大多数表盘位置。

### 各样式展示效果

```text
accessoryCircular（圆形）:        accessoryCorner（角落）:       accessoryRectangular（矩形）:

     ┌──────┐                    ┌─────────────┐              ┌──────────────────┐
    │  🌤    │                   │  🌤 25°C    │              │  🌤 北京          │
    │ 25°C   │                   │  晴天        │              │  25°C 晴天        │
     └──────┘                    └─────────────┘              │  湿度 45%         │
                                                               └──────────────────┘
```

---

## Timeline Provider：数据刷新策略

### TimelineProvider 协议

Timeline Provider 是 WidgetKit 的核心，负责提供数据和定义刷新策略：

```swift
import WidgetKit
import SwiftUI

struct WeatherTimelineProvider: TimelineProvider {
    func placeholder(in context: Context) -> WeatherEntry {
        WeatherEntry(
            date: Date(),
            temperature: 25,
            condition: .sunny,
            location: "北京",
            humidity: 45
        )
    }

    func getSnapshot(in context: Context, completion: @escaping (WeatherEntry) -> Void) {
        let entry = WeatherEntry(
            date: Date(),
            temperature: 25,
            condition: .sunny,
            location: "北京",
            humidity: 45
        )
        completion(entry)
    }

    func getTimeline(in context: Context, completion: @escaping (Timeline<WeatherEntry>) -> Void) {
        Task {
            do {
                let weather = try await WeatherService.shared.fetchWeather(
                    latitude: 39.9,
                    longitude: 116.4
                )

                let entry = WeatherEntry(
                    date: Date(),
                    temperature: weather.temperature,
                    condition: weather.condition,
                    location: weather.location,
                    humidity: weather.humidity
                )

                let nextUpdate = Calendar.current.date(byAdding: .hour, value: 1, to: Date())!
                let timeline = Timeline(entries: [entry], policy: .after(nextUpdate))
                completion(timeline)
            } catch {
                let entry = WeatherEntry(
                    date: Date(),
                    temperature: 0,
                    condition: .cloudy,
                    location: "--",
                    humidity: 0
                )
                let nextRetry = Calendar.current.date(byAdding: .minute, value: 15, to: Date())!
                let timeline = Timeline(entries: [entry], policy: .after(nextRetry))
                completion(timeline)
            }
        }
    }
}
```

### Timeline Entry 定义

```swift
struct WeatherEntry: TimelineEntry {
    let date: Date
    let temperature: Double
    let condition: WeatherCondition
    let location: String
    let humidity: Int
}
```

### 刷新策略

Timeline Provider 通过 `ReloadPolicy` 控制刷新频率：

| 策略 | 说明 | 适用场景 |
|------|------|----------|
| `.atEnd` | Timeline 中最后一个 Entry 过期后刷新 | 有明确过期时间的场景 |
| `.after(date)` | 在指定时间后刷新 | 定时刷新（如每小时） |
| `.never` | 不自动刷新 | 静态内容 |

**刷新策略选择：**

| 数据类型 | 建议刷新频率 | 策略 |
|----------|:----------:|------|
| 天气 | 30-60 分钟 | `.after(date)` |
| 股票 | 1-5 分钟（交易时间） | `.after(date)` |
| 日历事件 | 事件变化时 | `.atEnd` |
| 活动数据 | 15-30 分钟 | `.after(date)` |
| 任务列表 | 变化时 | `.never` + 手动刷新 |

> ⚠️ **警告**：Apple 对 Complication 的刷新频率有严格限制。你请求的刷新间隔只是"建议"，系统会根据电量、使用频率等因素决定实际刷新时间。频繁请求刷新可能导致系统降低你的刷新频率。

### 多 Entry Timeline

你可以提供一个包含多个 Entry 的 Timeline，系统会在对应时间点自动切换：

```swift
func getTimeline(in context: Context, completion: @escaping (Timeline<WeatherEntry>) -> Void) {
    Task {
        do {
            let currentWeather = try await WeatherService.shared.fetchWeather(
                latitude: 39.9,
                longitude: 116.4
            )

            var entries: [WeatherEntry] = []

            let now = Date()
            for hourOffset in 0..<6 {
                let entryDate = Calendar.current.date(byAdding: .hour, value: hourOffset, to: now)!

                let entry: WeatherEntry
                if hourOffset == 0 {
                    entry = WeatherEntry(
                        date: entryDate,
                        temperature: currentWeather.temperature,
                        condition: currentWeather.condition,
                        location: currentWeather.location,
                        humidity: currentWeather.humidity
                    )
                } else {
                    let forecastIndex = min(hourOffset - 1, currentWeather.forecast.count - 1)
                    let forecast = currentWeather.forecast[safe: forecastIndex]
                    entry = WeatherEntry(
                        date: entryDate,
                        temperature: forecast?.highTemp ?? currentWeather.temperature,
                        condition: forecast?.condition ?? currentWeather.condition,
                        location: currentWeather.location,
                        humidity: currentWeather.humidity
                    )
                }
                entries.append(entry)
            }

            let nextUpdate = Calendar.current.date(byAdding: .hour, value: 1, to: now)!
            let timeline = Timeline(entries: entries, policy: .after(nextUpdate))
            completion(timeline)
        } catch {
            let entry = WeatherEntry(
                date: Date(),
                temperature: 0,
                condition: .cloudy,
                location: "--",
                humidity: 0
            )
            let timeline = Timeline(entries: [entry], policy: .after(Date().addingTimeInterval(900)))
            completion(timeline)
        }
    }
}

extension Collection {
    subscript(safe index: Index) -> Element? {
        indices.contains(index) ? self[index] : nil
    }
}
```

---

## Complication 数据更新和刷新

### 手动刷新 Complication

除了 Timeline Provider 的自动刷新，你还可以在 App 中手动触发刷新：

```swift
import WidgetKit

final class ComplicationRefreshManager {
    static let shared = ComplicationRefreshManager()

    func reloadAllComplications() {
        WidgetCenter.shared.reloadAllTimelines()
    }

    func reloadComplications(ofKind kind: String) {
        WidgetCenter.shared.reloadTimelines(ofKind: kind)
    }

    func getCurrentComplicationInfo() {
        WidgetCenter.shared.getCurrentConfigurations { result in
            switch result {
            case .success(let widgets):
                for widget in widgets {
                    print("Widget: \(widget.kind), Family: \(widget.family)")
                }
            case .failure(let error):
                print("Failed to get widget info: \(error)")
            }
        }
    }
}
```

### 在 App 中触发刷新

```swift
import SwiftUI
import WidgetKit

struct WeatherAppView: View {
    @State private var weather: WeatherData?

    var body: some View {
        VStack {
            if let weather = weather {
                WeatherDetailView(weather: weather)
            } else {
                LoadingView()
            }
        }
        .task {
            await refreshWeather()
        }
        .refreshable {
            await refreshWeather()
        }
    }

    private func refreshWeather() async {
        do {
            let newWeather = try await WeatherService.shared.fetchWeather(
                latitude: 39.9,
                longitude: 116.4
            )
            weather = newWeather

            ComplicationRefreshManager.shared.reloadAllComplications()
        } catch {
            print("Refresh failed: \(error)")
        }
    }
}
```

### 后台刷新

watchOS 支持后台刷新 Complication 数据，但有严格限制：

| 限制 | 说明 |
|------|------|
| 每日预算 | 系统分配有限的后台刷新次数 |
| 刷新间隔 | 最少 15 分钟 |
| 电量优先 | 低电量时可能延迟或跳过 |
| 使用频率 | 不常用的 Complication 刷新频率更低 |

**后台刷新最佳实践：**

| 实践 | 说明 |
|------|------|
| 合理设置刷新间隔 | 不要请求过于频繁的刷新 |
| 使用多 Entry Timeline | 一次提供多个时间点的数据 |
| 错误时快速重试 | 网络失败时设置较短的重试间隔 |
| 优先使用 App 内刷新 | 用户打开 App 时主动刷新 Complication |
| 避免不必要刷新 | 数据没变化时不要刷新 |

---

## 设计指南：信息密度、可读性

### Complication 设计原则

| 原则 | 说明 | 示例 |
|------|------|------|
| **一瞥可读** | 用户 1-2 秒内获取关键信息 | "25°C" 比 "当前温度：25摄氏度" 好 |
| **信息优先** | 优先展示最重要的数据 | 天气 Complication 优先显示温度 |
| **简洁明了** | 避免过多文字和装饰 | 不需要标题，直接显示数值 |
| **色彩克制** | 少量色彩突出重点 | 用颜色区分状态，不要五彩斑斓 |
| **一致性** | 与 App 的视觉风格一致 | 使用相同的图标和配色 |

### 各样式的设计建议

**accessoryCircular（圆形）：**

| 建议 | 说明 |
|------|------|
| 最多 2 行信息 | 如温度 + 图标 |
| 使用 SF Symbols | 图标比文字更清晰 |
| 避免长文本 | 圆形空间有限 |
| 使用 Gauge | 进度环非常适合圆形区域 |

```swift
struct CircularComplicationView: View {
    let entry: WeatherEntry

    var body: some View {
        ZStack {
            AccessoryWidgetBackground()
            VStack(spacing: 2) {
                Image(systemName: entry.condition.iconName)
                    .font(.caption2)
                Text(entry.formattedTemperature)
                    .font(.caption)
                    .bold()
            }
        }
    }
}
```

**accessoryCorner（角落）：**

| 建议 | 说明 |
|------|------|
| 最多 3 行信息 | 如图标 + 温度 + 位置 |
| 使用弧形布局 | 顺应角落的弧形空间 |
| 突出核心数据 | 最重要的信息最大 |
| 底部放次要信息 | 位置名称放底部 |

```swift
struct CornerComplicationView: View {
    let entry: WeatherEntry

    var body: some View {
        VStack(spacing: 1) {
            Image(systemName: entry.condition.iconName)
                .font(.caption2)
            Text(entry.formattedTemperature)
                .font(.caption2)
                .bold()
            Text(entry.location)
                .font(.system(size: 8))
                .foregroundStyle(.secondary)
        }
    }
}
```

**accessoryRectangular（矩形）：**

| 建议 | 说明 |
|------|------|
| 最多 3-4 行信息 | 如位置 + 温度 + 状态 + 湿度 |
| 第一行最重要 | 用户最先看到 |
| 使用图标辅助 | 图标 + 文字组合更清晰 |
| 可以显示更多细节 | 矩形空间最充裕 |

```swift
struct RectangularComplicationView: View {
    let entry: WeatherEntry

    var body: some View {
        VStack(alignment: .leading, spacing: 2) {
            HStack {
                Image(systemName: entry.condition.iconName)
                Text(entry.location)
            }
            .font(.caption2)

            Text(entry.formattedTemperature)
                .font(.title3)
                .bold()

            HStack(spacing: 8) {
                Label("\(entry.humidity)%", systemImage: "drop.fill")
                Label(entry.condition.displayName, systemImage: "wind")
            }
            .font(.system(size: 9))
            .foregroundStyle(.secondary)
        }
    }
}
```

### 配色和可访问性

| 建议 | 说明 |
|------|------|
| 使用系统颜色 | `Color.white`、`Color.secondary` 等 |
| 避免纯黑背景 | 系统会自动处理背景 |
| 考虑深色表盘 | 确保在深色和浅色表盘上都可读 |
| 使用高对比度 | 文字和背景要有足够对比度 |
| 测试不同表盘 | 不同表盘颜色可能影响可读性 |

> ⚠️ **警告**：Complication 的背景由系统控制，你不应该设置自定义背景色。使用 `AccessoryWidgetBackground()` 可以获得系统推荐的背景效果。

---

## 从 ClockKit 迁移到 WidgetKit

### 为什么要迁移

| 原因 | 说明 |
|------|------|
| WidgetKit 是未来 | Apple 的战略方向，持续获得新功能 |
| 跨平台代码复用 | 同一套代码可用于 iOS Widget 和 watchOS Complication |
| SwiftUI 渲染 | 更灵活的 UI 定制能力 |
| Smart Stack 支持 | watchOS 10+ 的新功能 |
| 维护成本 | ClockKit 不再获得新功能 |

### 迁移步骤

| 步骤 | 操作 | 说明 |
|:----:|------|------|
| 1 | 创建 Widget Extension | 添加新的 WidgetKit Target |
| 2 | 实现 TimelineProvider | 替换 CLKComplicationDataSource |
| 3 | 实现 SwiftUI View | 替换 ClockKit 模板 |
| 4 | 配置 Widget | 定义支持的样式 |
| 5 | 测试所有样式 | 确保各样式正常显示 |
| 6 | 移除 ClockKit 代码 | 删除旧的 Complication Controller |
| 7 | 更新 Info.plist | 移除 ClockKit 相关配置 |

### ClockKit → WidgetKit 对照表

| ClockKit | WidgetKit | 说明 |
|----------|-----------|------|
| `CLKComplicationDataSource` | `TimelineProvider` | 数据源协议 |
| `CLKComplicationTemplate` | SwiftUI View | UI 渲染 |
| `CLKComplicationTimelineEntry` | `TimelineEntry` | 时间线条目 |
| `getCurrentTimelineEntry` | `getTimeline` | 获取当前数据 |
| `getTimelineEntriesForComplication` | `getTimeline` (多 Entry) | 获取多条目 |
| `getPlaceholderTemplate` | `placeholder` | 占位数据 |
| `requestedUpdateDidBegin` | Timeline Policy | 刷新触发 |
| `CLKComplicationFamily` | `WidgetFamily` | 样式类型 |

### ClockKit 代码示例（旧）

```swift
import ClockKit

class ComplicationController: NSObject, CLKComplicationDataSource {
    func getCurrentTimelineEntry(
        for complication: CLKComplication,
        withHandler handler: @escaping (CLKComplicationTimelineEntry?) -> Void
    ) {
        guard complication.family == .modularSmall else {
            handler(nil)
            return
        }

        let template = CLKComplicationTemplateModularSmallStackText()
        template.line1TextProvider = CLKSimpleTextProvider(text: "25°")
        template.line2TextProvider = CLKSimpleTextProvider(text: "晴天")

        let entry = CLKComplicationTimelineEntry(
            date: Date(),
            complicationTemplate: template
        )
        handler(entry)
    }

    func getPlaceholderTemplate(
        for complication: CLKComplication,
        withHandler handler: @escaping (CLKComplicationTemplate?) -> Void
    ) {
        let template = CLKComplicationTemplateModularSmallStackText()
        template.line1TextProvider = CLKSimpleTextProvider(text: "--°")
        template.line2TextProvider = CLKSimpleTextProvider(text: "--")
        handler(template)
    }
}
```

### WidgetKit 等效代码（新）

```swift
import WidgetKit
import SwiftUI

struct WeatherComplication: Widget {
    let kind: String = "WeatherComplication"

    var body: some WidgetConfiguration {
        StaticConfiguration(kind: kind, provider: WeatherTimelineProvider()) { entry in
            WeatherComplicationEntryView(entry: entry)
        }
        .configurationDisplayName("天气")
        .description("查看当前天气信息")
        .supportedFamilies([
            .accessoryCircular,
            .accessoryCorner,
            .accessoryRectangular
        ])
    }
}
```

### 迁移注意事项

| 注意事项 | 说明 |
|----------|------|
| 保留 ClockKit 一段时间 | 迁移期间两种方式可以共存 |
| 测试所有表盘 | 确保每种样式都正确显示 |
| 数据共享 | 使用 App Group 共享数据 |
| 刷新策略调整 | WidgetKit 的刷新机制不同于 ClockKit |
| 用户需要重新添加 | 迁移后用户可能需要重新配置 Complication |

---

## 实战：为天气 App 创建表盘组件

### 完整项目结构

```text
WeatherWatch/
├── WeatherWatchApp.swift          # Watch App 入口
├── Views/
│   ├── WatchMainView.swift        # 主界面
│   └── WatchDetailView.swift      # 详情界面
│
├── WeatherComplication/           # Complication Extension
│   ├── WeatherComplication.swift  # Widget 定义
│   ├── WeatherTimelineProvider.swift  # Timeline Provider
│   ├── WeatherEntry.swift         # Entry 数据模型
│   └── Views/
│       ├── CircularView.swift     # 圆形样式
│       ├── CornerView.swift       # 角落样式
│       └── RectangularView.swift  # 矩形样式
│
├── Shared/                        # 共享代码
│   ├── Models/
│   │   └── WeatherData.swift
│   ├── Services/
│   │   └── WeatherService.swift
│   └── Extensions/
│       └── Double+Formatting.swift
│
└── Info.plist
```

### Entry 数据模型

```swift
import WidgetKit

struct WeatherEntry: TimelineEntry {
    let date: Date
    let temperature: Double
    let condition: WeatherCondition
    let location: String
    let humidity: Int
    let windSpeed: Double
    let highTemp: Double
    let lowTemp: Double

    var formattedTemperature: String {
        String(format: "%.0f°", temperature)
    }

    var formattedHighLow: String {
        String(format: "%.0f°/%.0f°", highTemp, lowTemp)
    }

    static let placeholder = WeatherEntry(
        date: Date(),
        temperature: 25,
        condition: .sunny,
        location: "北京",
        humidity: 45,
        windSpeed: 12,
        highTemp: 28,
        lowTemp: 18
    )

    static let error = WeatherEntry(
        date: Date(),
        temperature: 0,
        condition: .cloudy,
        location: "--",
        humidity: 0,
        windSpeed: 0,
        highTemp: 0,
        lowTemp: 0
    )
}
```

### Timeline Provider 完整实现

```swift
import WidgetKit
import SwiftUI

struct WeatherTimelineProvider: TimelineProvider {
    func placeholder(in context: Context) -> WeatherEntry {
        .placeholder
    }

    func getSnapshot(in context: Context, completion: @escaping (WeatherEntry) -> Void) {
        if context.isPreview {
            completion(.placeholder)
        } else {
            Task {
                let entry = await fetchWeatherEntry()
                completion(entry)
            }
        }
    }

    func getTimeline(in context: Context, completion: @escaping (Timeline<WeatherEntry>) -> Void) {
        Task {
            let currentEntry = await fetchWeatherEntry()
            var entries: [WeatherEntry] = [currentEntry]

            let now = Date()
            let calendar = Calendar.current

            for hourOffset in 1...5 {
                guard let futureDate = calendar.date(byAdding: .hour, value: hourOffset, to: now) else {
                    continue
                }

                let tempVariation = Double.random(in: -2...2)
                let futureEntry = WeatherEntry(
                    date: futureDate,
                    temperature: currentEntry.temperature + tempVariation,
                    condition: currentEntry.condition,
                    location: currentEntry.location,
                    humidity: currentEntry.humidity,
                    windSpeed: currentEntry.windSpeed,
                    highTemp: currentEntry.highTemp,
                    lowTemp: currentEntry.lowTemp
                )
                entries.append(futureEntry)
            }

            guard let nextUpdate = calendar.date(byAdding: .hour, value: 1, to: now) else {
                completion(Timeline(entries: entries, policy: .atEnd))
                return
            }

            let timeline = Timeline(entries: entries, policy: .after(nextUpdate))
            completion(timeline)
        }
    }

    private func fetchWeatherEntry() async -> WeatherEntry {
        do {
            let weather = try await WeatherService.shared.fetchWeather(
                latitude: 39.9,
                longitude: 116.4
            )
            return WeatherEntry(
                date: Date(),
                temperature: weather.temperature,
                condition: weather.condition,
                location: weather.location,
                humidity: weather.humidity,
                windSpeed: weather.windSpeed,
                highTemp: weather.forecast.first?.highTemp ?? weather.temperature,
                lowTemp: weather.forecast.first?.lowTemp ?? weather.temperature
            )
        } catch {
            return .error
        }
    }
}
```

### Complication View 实现

**圆形样式：**

```swift
struct CircularComplicationView: View {
    let entry: WeatherEntry

    var body: some View {
        ZStack {
            AccessoryWidgetBackground()
            VStack(spacing: 1) {
                Image(systemName: entry.condition.iconName)
                    .font(.system(size: 12))
                    .foregroundStyle(conditionColor)
                Text(entry.formattedTemperature)
                    .font(.system(size: 14, weight: .bold, design: .rounded))
            }
        }
    }

    private var conditionColor: Color {
        switch entry.condition {
        case .sunny: return .yellow
        case .cloudy: return .gray
        case .rainy: return .blue
        case .snowy: return .cyan
        case .stormy: return .purple
        }
    }
}
```

**角落样式：**

```swift
struct CornerComplicationView: View {
    let entry: WeatherEntry

    var body: some View {
        VStack(spacing: 1) {
            HStack(spacing: 2) {
                Image(systemName: entry.condition.iconName)
                    .font(.system(size: 10))
                Text(entry.formattedTemperature)
                    .font(.system(size: 12, weight: .bold, design: .rounded))
            }
            Text(entry.location)
                .font(.system(size: 8))
                .foregroundStyle(.secondary)
        }
    }
}
```

**矩形样式：**

```swift
struct RectangularComplicationView: View {
    let entry: WeatherEntry

    var body: some View {
        VStack(alignment: .leading, spacing: 2) {
            HStack(spacing: 4) {
                Image(systemName: entry.condition.iconName)
                    .foregroundStyle(conditionColor)
                Text(entry.location)
            }
            .font(.system(size: 11))

            HStack(alignment: .firstTextBaseline, spacing: 4) {
                Text(entry.formattedTemperature)
                    .font(.system(size: 20, weight: .bold, design: .rounded))
                Text(entry.condition.displayName)
                    .font(.system(size: 11))
                    .foregroundStyle(.secondary)
            }

            HStack(spacing: 8) {
                Label("\(entry.humidity)%", systemImage: "drop.fill")
                Label(String(format: "%.0f km/h", entry.windSpeed), systemImage: "wind")
                Text(entry.formattedHighLow)
            }
            .font(.system(size: 9))
            .foregroundStyle(.secondary)
        }
    }

    private var conditionColor: Color {
        switch entry.condition {
        case .sunny: return .yellow
        case .cloudy: return .gray
        case .rainy: return .blue
        case .snowy: return .cyan
        case .stormy: return .purple
        }
    }
}
```

### 统一 Entry View

```swift
struct WeatherComplicationEntryView: View {
    let entry: WeatherEntry

    @Environment(\.widgetFamily) var family

    var body: some View {
        switch family {
        case .accessoryCircular:
            CircularComplicationView(entry: entry)
        case .accessoryCorner:
            CornerComplicationView(entry: entry)
        case .accessoryRectangular:
            RectangularComplicationView(entry: entry)
        default:
            CircularComplicationView(entry: entry)
        }
    }
}
```

### Widget 定义

```swift
@main
struct WeatherComplicationBundle: WidgetBundle {
    var body: some Widget {
        WeatherComplication()
    }
}

struct WeatherComplication: Widget {
    let kind: String = "com.weatherapp.complication"

    var body: some WidgetConfiguration {
        StaticConfiguration(kind: kind, provider: WeatherTimelineProvider()) { entry in
            WeatherComplicationEntryView(entry: entry)
        }
        .configurationDisplayName("天气")
        .description("在表盘上查看当前天气信息")
        .supportedFamilies([
            .accessoryCircular,
            .accessoryCorner,
            .accessoryRectangular
        ])
    }
}
```

### App Group 数据共享

Complication Extension 和主 App 运行在不同的进程中，需要通过 App Group 共享数据：

**配置 App Group：**

| 步骤 | 操作 |
|:----:|------|
| 1 | 在 Apple Developer 中创建 App Group |
| 2 | 在主 App 的 Entitlements 中添加 App Group |
| 3 | 在 Widget Extension 的 Entitlements 中添加相同的 App Group |
| 4 | 使用相同的 Group ID 读写数据 |

**共享数据管理：**

```swift
import Foundation

final class SharedDataManager {
    static let shared = SharedDataManager()

    private let appGroupID = "group.com.weatherapp.shared"
    private let userDefaults: UserDefaults

    private init() {
        self.userDefaults = UserDefaults(suiteName: appGroupID) ?? .standard
    }

    func saveWeatherData(_ data: WeatherData) {
        if let encoded = try? JSONEncoder().encode(data) {
            userDefaults.set(encoded, forKey: "cached_weather")
            userDefaults.set(Date(), forKey: "weather_last_update")
        }
    }

    func loadWeatherData() -> WeatherData? {
        guard let data = userDefaults.data(forKey: "cached_weather") else { return nil }
        return try? JSONDecoder().decode(WeatherData.self, from: data)
    }

    var lastUpdateTime: Date? {
        userDefaults.object(forKey: "weather_last_update") as? Date
    }

    var isDataStale: Bool {
        guard let lastUpdate = lastUpdateTime else { return true }
        return Date().timeIntervalSince(lastUpdate) > 3600
    }
}
```

**在主 App 中保存数据：**

```swift
struct WatchMainView: View {
    @State private var weather: WeatherData?

    var body: some View {
        List {
            if let weather = weather {
                WatchWeatherRow(weather: weather)
            }
        }
        .task {
            await loadWeather()
        }
    }

    private func loadWeather() async {
        do {
            let data = try await WeatherService.shared.fetchWeather(
                latitude: 39.9,
                longitude: 116.4
            )
            weather = data
            SharedDataManager.shared.saveWeatherData(data)
            WidgetCenter.shared.reloadAllTimelines()
        } catch {
            weather = SharedDataManager.shared.loadWeatherData()
        }
    }
}
```

**在 Timeline Provider 中读取共享数据：**

```swift
struct WeatherTimelineProvider: TimelineProvider {
    func getTimeline(in context: Context, completion: @escaping (Timeline<WeatherEntry>) -> Void) {
        let sharedData = SharedDataManager.shared.loadWeatherData()

        if let data = sharedData {
            let entry = WeatherEntry(
                date: Date(),
                temperature: data.temperature,
                condition: data.condition,
                location: data.location,
                humidity: data.humidity,
                windSpeed: data.windSpeed,
                highTemp: data.forecast.first?.highTemp ?? data.temperature,
                lowTemp: data.forecast.first?.lowTemp ?? data.temperature
            )

            let nextUpdate = Calendar.current.date(byAdding: .hour, value: 1, to: Date())!
            let timeline = Timeline(entries: [entry], policy: .after(nextUpdate))
            completion(timeline)
        } else {
            let entry = WeatherEntry.error
            let retryDate = Calendar.current.date(byAdding: .minute, value: 15, to: Date())!
            let timeline = Timeline(entries: [entry], policy: .after(retryDate))
            completion(timeline)
        }
    }
}
```

---

## 调试和测试 Complications

### Xcode 调试 Complication

| 步骤 | 操作 | 说明 |
|:----:|------|------|
| 1 | 选择 Widget Extension Scheme | 在 Xcode 顶部的 Scheme 选择器中 |
| 2 | 选择目标设备 | 模拟器或真机 |
| 3 | Run | Xcode 会提示选择要预览的样式 |
| 4 | 选择样式 | circular / corner / rectangular |

### Complication 预览

在 Xcode 中使用 Preview 预览 Complication：

```swift
#Preview(as: .accessoryCircular) {
    WeatherComplication()
} timeline: {
    WeatherEntry.placeholder
    WeatherEntry(
        date: Date().addingTimeInterval(3600),
        temperature: 30,
        condition: .sunny,
        location: "上海",
        humidity: 60,
        windSpeed: 8,
        highTemp: 32,
        lowTemp: 22
    )
}

#Preview(as: .accessoryCorner) {
    WeatherComplication()
} timeline: {
    WeatherEntry.placeholder
}

#Preview(as: .accessoryRectangular) {
    WeatherComplication()
} timeline: {
    WeatherEntry.placeholder
    WeatherEntry(
        date: Date().addingTimeInterval(3600),
        temperature: 15,
        condition: .rainy,
        location: "深圳",
        humidity: 85,
        windSpeed: 20,
        highTemp: 18,
        lowTemp: 12
    )
}
```

### 模拟器测试

在模拟器中测试 Complication 的完整流程：

| 步骤 | 操作 |
|:----:|------|
| 1 | 运行 Widget Extension 到模拟器 |
| 2 | 长按表盘进入编辑模式 |
| 3 | 向左滑动到"Complications"页面 |
| 4 | 选择一个 Complication 位置 |
| 5 | 在列表中找到你的 App |
| 6 | 选择 Complication 样式 |
| 7 | 按下 Digital Crown 完成编辑 |

### 真机调试注意事项

| 注意事项 | 说明 |
|----------|------|
| 需要配对 | Apple Watch 需要与 iPhone 配对 |
| 安装方式 | 通过 Xcode 直接安装到 Watch |
| 刷新延迟 | 真机上的刷新可能比模拟器慢 |
| 电量影响 | 观察 Complication 对电量的影响 |
| 多表盘测试 | 在不同表盘上测试显示效果 |

### 常见调试问题

| 问题 | 原因 | 解决方法 |
|------|------|----------|
| Complication 不显示 | 未正确配置 Widget Family | 检查 `supportedFamilies` |
| 数据不更新 | Timeline Provider 未返回新数据 | 检查 `getTimeline` 实现 |
| 显示占位数据 | `placeholder` 方法返回的数据 | 检查 `getSnapshot` 实现 |
| 刷新太慢 | 系统限制了刷新频率 | 使用 App 内手动刷新 |
| 崩溃 | Extension 内存超限 | 优化数据处理逻辑 |
| 样式错乱 | 不同 Family 的布局问题 | 分别测试每种样式 |

---

## 性能优化和电池消耗

### Complication 性能限制

Apple 对 Complication 有严格的性能限制，以确保手表的电池续航：

| 限制 | 值 | 说明 |
|------|:--:|------|
| 内存限制 | 约 30MB | Extension 运行时内存 |
| CPU 时间 | 约 30 秒 | getTimeline 的执行时间 |
| 网络请求 | 允许但受限 | 建议使用缓存数据 |
| 刷新预算 | 系统动态分配 | 通常每天 40-50 次 |
| 渲染时间 | 约 5 秒 | View 渲染时间 |

### 性能优化策略

**1. 减少 Timeline Provider 的计算量：**

```swift
struct OptimizedTimelineProvider: TimelineProvider {
    func getTimeline(in context: Context, completion: @escaping (Timeline<WeatherEntry>) -> Void) {
        let cachedData = SharedDataManager.shared.loadWeatherData()

        if cachedData != nil && !SharedDataManager.shared.isDataStale {
            let entry = createEntry(from: cachedData!)
            let timeline = Timeline(entries: [entry], policy: .atEnd)
            completion(timeline)
            return
        }

        Task {
            do {
                let weather = try await WeatherService.shared.fetchWeather(
                    latitude: 39.9,
                    longitude: 116.4
                )
                SharedDataManager.shared.saveWeatherData(weather)
                let entry = createEntry(from: weather)
                let nextUpdate = Calendar.current.date(byAdding: .hour, value: 1, to: Date())!
                let timeline = Timeline(entries: [entry], policy: .after(nextUpdate))
                completion(timeline)
            } catch {
                let entry = cachedData.map { createEntry(from: $0) } ?? .error
                let retryDate = Calendar.current.date(byAdding: .minute, value: 30, to: Date())!
                let timeline = Timeline(entries: [entry], policy: .after(retryDate))
                completion(timeline)
            }
        }
    }

    private func createEntry(from data: WeatherData) -> WeatherEntry {
        WeatherEntry(
            date: Date(),
            temperature: data.temperature,
            condition: data.condition,
            location: data.location,
            humidity: data.humidity,
            windSpeed: data.windSpeed,
            highTemp: data.forecast.first?.highTemp ?? data.temperature,
            lowTemp: data.forecast.first?.lowTemp ?? data.temperature
        )
    }
}
```

**2. 优化 SwiftUI View：**

```swift
struct OptimizedCircularView: View {
    let entry: WeatherEntry

    var body: some View {
        ZStack {
            AccessoryWidgetBackground()
            VStack(spacing: 1) {
                Image(systemName: entry.condition.iconName)
                    .font(.system(size: 12))
                Text(entry.formattedTemperature)
                    .font(.system(size: 14, weight: .bold, design: .rounded).monospacedDigit())
            }
        }
    }
}
```

**3. 使用 monospacedDigit 修饰符：**

```swift
// 数字变化时不会导致布局抖动
Text(entry.formattedTemperature)
    .font(.system(.body, design: .rounded).monospacedDigit())
```

### 电池消耗优化

| 优化策略 | 说明 | 效果 |
|----------|------|:----:|
| 使用缓存数据 | 优先使用 App Group 共享的缓存 | ⭐⭐⭐⭐⭐ |
| 减少网络请求 | 只在必要时发起网络请求 | ⭐⭐⭐⭐⭐ |
| 合理刷新间隔 | 1 小时而非 15 分钟 | ⭐⭐⭐⭐ |
| 简化 View | 减少视图层级和动画 | ⭐⭐⭐ |
| 避免图片加载 | 使用 SF Symbols 代替图片 | ⭐⭐⭐⭐ |
| 批量更新 | 一次提供多个 Entry | ⭐⭐⭐ |

### 刷新预算管理

```swift
struct RefreshBudgetManager {
    private let userDefaults = UserDefaults(suiteName: "group.com.weatherapp.shared")
    private let refreshCountKey = "complication_refresh_count"
    private let refreshDateKey = "complication_refresh_date"

    func recordRefresh() {
        let today = Calendar.current.startOfDay(for: Date())
        let lastDate = userDefaults?.object(forKey: refreshDateKey) as? Date

        if lastDate == nil || !Calendar.current.isDate(today, inSameDayAs: lastDate!) {
            userDefaults?.set(1, forKey: refreshCountKey)
            userDefaults?.set(today, forKey: refreshDateKey)
        } else {
            let count = userDefaults?.integer(forKey: refreshCountKey) ?? 0
            userDefaults?.set(count + 1, forKey: refreshCountKey)
        }
    }

    var shouldRefresh: Bool {
        let today = Calendar.current.startOfDay(for: Date())
        let lastDate = userDefaults?.object(forKey: refreshDateKey) as? Date

        guard let lastDate = lastDate,
              Calendar.current.isDate(today, inSameDayAs: lastDate) else {
            return true
        }

        let count = userDefaults?.integer(forKey: refreshCountKey) ?? 0
        return count < 40
    }

    var optimalRefreshInterval: TimeInterval {
        shouldRefresh ? 3600 : 7200
    }
}
```

---

## Complication 开发常见问题

### FAQ

| 问题 | 回答 |
|------|------|
| Complication 支持交互吗？ | watchOS 10+ 支持点击打开 App，不支持复杂交互 |
| 可以显示实时数据吗？ | 不支持真正的实时，依赖 Timeline 刷新 |
| 最多支持几种样式？ | 建议支持 3 种：circular、corner、rectangular |
| Complication 可以显示图片吗？ | 可以使用 SF Symbols，不建议加载网络图片 |
| 多个 Complication 可以共存吗？ | 可以，用户可以在不同位置添加不同 Complication |
| Complication 支持动画吗？ | 有限支持，建议避免动画以节省电量 |
| 如何测试刷新策略？ | 使用 Xcode 的 Widget Simulator 或真机测试 |
| Complication 数据和 App 数据如何同步？ | 通过 App Group 共享 UserDefaults 或文件 |

### 常见错误和避免方法

| 错误 | 后果 | 避免方法 |
|------|------|----------|
| 不提供 placeholder | 表盘编辑时无预览 | 始终实现 placeholder |
| 过于频繁刷新 | 刷新被系统限制 | 合理设置刷新间隔 |
| View 过于复杂 | 渲染超时或内存超限 | 保持 View 简洁 |
| 不处理错误状态 | 显示异常数据 | 提供错误状态的 Entry |
| 不使用 App Group | 数据无法共享 | 配置 App Group |
| 忽略可访问性 | 部分用户无法使用 | 添加 accessibility 标签 |
| 不测试所有表盘 | 某些表盘显示异常 | 在多种表盘上测试 |

### 可访问性

```swift
struct AccessibleCircularView: View {
    let entry: WeatherEntry

    var body: some View {
        ZStack {
            AccessoryWidgetBackground()
            VStack(spacing: 1) {
                Image(systemName: entry.condition.iconName)
                    .font(.system(size: 12))
                Text(entry.formattedTemperature)
                    .font(.system(size: 14, weight: .bold, design: .rounded))
            }
        }
        .accessibilityElement(children: .ignore)
        .accessibilityLabel("\(entry.location)天气：\(entry.condition.displayName)，温度\(entry.formattedTemperature)")
    }
}
```

---

## 本章小结

本章详细介绍了 watchOS Complications 的开发方法：

| 知识点 | 要点 |
|--------|------|
| Complications 概述 | 表盘小组件，提供即时信息和 App 入口 |
| WidgetKit | Apple 推荐的 Complication 开发框架 |
| Complication 家族 | circular、corner、rectangular 三种主要样式 |
| Timeline Provider | 提供数据和刷新策略的核心机制 |
| 数据更新 | 自动刷新 + 手动刷新，注意刷新预算 |
| 设计指南 | 一瞥可读、信息优先、简洁明了 |
| ClockKit 迁移 | 从旧 API 迁移到 WidgetKit |
| 实战 | 天气 App 表盘组件完整实现 |
| 调试测试 | Xcode Preview + 模拟器 + 真机 |
| 性能优化 | 缓存数据、简化 View、合理刷新 |

**核心原则：Complication 是用户与你的 App 最高频的接触点。简洁、准确、省电是三大设计目标。**

> 💡 **提示**：开发 Complication 时，始终以"用户抬腕 1 秒内能获取什么信息"为设计出发点。如果你的 Complication 需要用户花 3 秒以上才能理解，那就太复杂了。

---

**上一章**：[跨平台代码共享策略](跨平台代码共享策略.md)

**下一章**：[macCatalyst与macOS移植](macCatalyst与macOS移植.md)
