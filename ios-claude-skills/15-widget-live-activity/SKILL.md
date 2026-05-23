---
name: widget-live-activity
description: 涉及 Widget、WidgetKit、Live Activity、ActivityKit、Dynamic Island、灵动岛、桌面小组件、锁屏小组件的开发任务
---

# Widget / Live Activity / 动态岛

## 架构概览

| 特性 | 最低版本 | 框架 | 运行环境 |
|------|---------|------|---------|
| 主屏幕 Widget | iOS 14+ | WidgetKit | 独立进程 |
| 锁屏 Widget | iOS 16+ | WidgetKit | 独立进程 |
| Live Activity | iOS 16.1+ | ActivityKit | 主 App 进程 |
| Dynamic Island | iOS 16.1+ | ActivityKit | 主 App 进程 |

**核心区别：**
- Widget 是独立 Extension 进程，与主 App 不共享内存
- Live Activity / Dynamic Island 运行在主 App 进程内
- Widget **必须用 SwiftUI**，不支持 UIKit
- Live Activity 的 UI 也必须用 SwiftUI

---

## Widget 开发

### 创建 Widget Extension
- Xcode → File → New → Target → Widget Extension
- 勾选 "Include Live Activity" 可同时生成 Live Activity 模板
- Widget Extension 有独立 Bundle ID：`com.app.WidgetExtension`

### 目录结构
```
WidgetExtension/
├── WidgetExtension.swift        # 入口 + Provider + Widget View
├── WidgetExtensionBundle.swift  # Bundle（多个 Widget 时）
├── Assets.xcassets
└── Info.plist
```

### 基本结构

```swift
import WidgetKit
import SwiftUI

struct Provider: TimelineProvider {
    func placeholder(in context: Context) -> SimpleEntry {
        SimpleEntry(date: Date(), title: "示例标题", value: "示例值")
    }

    func getSnapshot(in context: Context, completion: @escaping (SimpleEntry) -> Void) {
        completion(SimpleEntry(date: Date(), title: "快照标题", value: "快照值"))
    }

    func getTimeline(in context: Context, completion: @escaping (Timeline<SimpleEntry>) -> Void) {
        let currentDate = Date()
        let entry = SimpleEntry(date: currentDate, title: "实时标题", value: "实时值")

        let nextUpdate = Calendar.current.date(byAdding: .minute, value: 30, to: currentDate)!
        let timeline = Timeline(entries: [entry], policy: .after(nextUpdate))
        completion(timeline)
    }
}

struct SimpleEntry: TimelineEntry {
    let date: Date
    let title: String
    let value: String
}

struct WidgetEntryView: View {
    let entry: SimpleEntry

    var body: some View {
        VStack(alignment: .leading, spacing: 4) {
            Text(entry.title)
                .font(.headline)
            Text(entry.value)
                .font(.subheadline)
                .foregroundColor(.secondary)
        }
        .padding()
    }
}

struct AppWidget: Widget {
    let kind: String = "AppWidget"

    var body: some WidgetConfiguration {
        StaticConfiguration(kind: kind, provider: Provider()) { entry in
            WidgetEntryView(entry: entry)
        }
        .configurationDisplayName("我的小组件")
        .description("显示最新数据")
        .supportedFamilies([.systemSmall, .systemMedium])
    }
}
```

### Widget 尺寸

| Family | 尺寸（pt） | 适用场景 |
|--------|-----------|---------|
| systemSmall | 158×158 | 单一数据展示 |
| systemMedium | 348×158 | 数据+图表 |
| systemLarge | 348×338 | 多数据+图表 |
| systemExtraLarge | 348×400 | iPad 专属 |
| accessoryCircular | 直径 56 | 锁屏圆形 |
| accessoryRectangular | 148×56 | 锁屏矩形 |
| accessoryInline | 全宽一行 | 锁屏单行文字 |

### Timeline 策略

```swift
enum TimelinePolicy {
    case atEnd                    // 最后一个 entry 时间到后刷新
    case after(Date)              // 指定时间后刷新
    case never                    // 不自动刷新
}
```

- **禁止频繁刷新**：系统限制每小时约 40-70 次刷新预算
- 推荐刷新间隔：15-60 分钟
- 紧急更新用 `WidgetCenter.shared.reloadAllTimelines()` 主动触发（消耗预算）
- 多个 entry 可实现动画过渡效果

### 数据共享

#### App Group + UserDefaults
```swift
let sharedDefaults = UserDefaults(suiteName: "group.com.app.shared")

// 主 App 写入
sharedDefaults?.set(data, forKey: "widget_data")

// Widget 读取
let data = sharedDefaults?.object(forKey: "widget_data")
```

#### App Group + FileManager
```swift
guard let groupURL = FileManager.default.containerURL(
    forSecurityApplicationGroupIdentifier: "group.com.app.shared") else { return }
let fileURL = groupURL.appendingPathComponent("widget_data.json")
```

#### CoreData 共享
- Widget 和主 App 使用同一个 App Group 容器中的 CoreData 数据库
- `NSPersistentStoreDescription` 的 URL 指向 App Group 容器

### Widget 交互（Deep Link）

```swift
Link(destination: URL(string: "myapp://detail/123")!) {
    Text("查看详情")
}
```

- 点击 Widget 跳转主 App，通过 `onOpenURL` 处理
- 主 App 的 SceneDelegate 中处理：

```swift
func scene(_ scene: UIScene, openURLContexts URLContexts: Set<UIOpenURLContext>) {
    guard let url = URLContexts.first?.url else { return }
    handleDeepLink(url)
}
```

---

## Live Activity

### 创建 Live Activity

```swift
import ActivityKit

struct ActivityAttributes: ActivityAttributes {
    struct ContentState: Codable, Hashable {
        var status: String
        var progress: Double
        var estimatedEndTime: Date
    }
    var title: String
    var orderId: String
}
```

### 启动 Live Activity

```swift
func startLiveActivity(title: String, orderId: String) throws -> Activity<ActivityAttributes> {
    let attributes = ActivityAttributes(title: title, orderId: orderId)
    let state = ActivityAttributes.ContentState(
        status: "进行中",
        progress: 0.0,
        estimatedEndTime: Date().addingTimeInterval(1800)
    )

    let activity = try Activity.request(
        attributes: attributes,
        content: .init(state: state, staleDate: nil),
        pushType: nil
    )
    return activity
}
```

### 更新 Live Activity

```swift
func updateLiveActivity(_ activity: Activity<ActivityAttributes>, progress: Double) async {
    let state = ActivityAttributes.ContentState(
        status: progress >= 1.0 ? "已完成" : "进行中",
        progress: progress,
        estimatedEndTime: Date()
    )
    await activity.update(
        .init(state: state, staleDate: nil)
    )
}
```

### 结束 Live Activity

```swift
func endLiveActivity(_ activity: Activity<ActivityAttributes>) async {
    let finalState = ActivityAttributes.ContentState(
        status: "已完成",
        progress: 1.0,
        estimatedEndTime: Date()
    )
    await activity.end(
        .init(state: finalState, staleDate: nil),
        dismissalPolicy: .after(.now + 3600)
    )
}
```

### Live Activity UI

```swift
struct LiveActivityView: Widget {
    var body: some WidgetConfiguration {
        ActivityConfiguration(for: ActivityAttributes.self) { context in
            LockScreenLiveActivityView(context: context)
        } dynamicIsland: { context in
            DynamicIsland {
                DynamicIslandExpandedRegion(.leading) {
                    Text(context.state.status)
                }
                DynamicIslandExpandedRegion(.trailing) {
                    Text("\(Int(context.state.progress * 100))%")
                }
                DynamicIslandExpandedRegion(.center) {
                    ProgressView(value: context.state.progress)
                }
                DynamicIslandExpandedRegion(.bottom) {
                    Text(context.attributes.title)
                        .font(.caption)
                }
            } compactLeading: {
                Image(systemName: "progress.indicator")
            } compactTrailing: {
                Text("\(Int(context.state.progress * 100))%")
            } minimal: {
                Image(systemName: "progress.indicator")
            }
        }
    }
}
```

### 远程推送更新 Live Activity
- 设置 `pushType: .token` 启用推送更新
- 获取推送 Token：

```swift
for await pushToken in activity.pushTokenStream {
    let token = pushToken.map { String(format: "%02x", $0) }.joined()
    await sendTokenToBackend(token)
}
```

- 后端通过 APNs 向 Live Activity 推送更新，Payload 格式：
```json
{
    "aps": {
        "timestamp": 1234567890,
        "event": "update",
        "content-state": {
            "status": "配送中",
            "progress": 0.6,
            "estimatedEndTime": 762349200
        }
    }
}
```

---

## 规范

### Widget 规范
- **禁止在 Widget 中做网络请求**，数据通过 App Group 共享或 Timeline Provider 获取
- Widget 内存限制约 30MB，**禁止加载大图**
- Widget 渲染时间限制约 30 秒，超时系统显示占位图
- **禁止在 Widget 中播放动画或视频**
- 图片资源用 Asset Catalog，通过 `Image("name")` 引用

### Live Activity 规范
- 同时最多存在 8 个 Live Activity
- Live Activity 最长存活 12 小时（iOS 16.1-16.2 无限制，16.2+ 限制 8 小时显示 + 4 小时结束态）
- **禁止在 Live Activity 中显示广告或推广内容**
- 结束时必须调用 `activity.end()`，禁止让 Live Activity 自然过期
- 网络状态更新优先用推送，轮询会消耗大量电量

### Dynamic Island 规范
- compactLeading + compactTrailing 宽度各约 28pt
- minimal 模式只显示一个圆形图标
- expanded 模式高度约 160pt
- **禁止在 Dynamic Island 中放置可滚动内容**

### 审核
- Widget 必须提供实际功能，**禁止纯装饰性 Widget**
- Live Activity 必须有时效性（外卖配送、运动比分、航班状态等），**禁止用于常驻通知**
- 锁屏 Widget 必须使用 `accessoryCircular` / `accessoryRectangular` / `accessoryInline`

---

## 已知陷阱

- **Widget Extension 不共享主 App 的 Keychain**，敏感数据通过 App Group + 加密传递
- **Widget 在后台被系统杀死后不会自动重启**，需要用户点击或 App 主动调用 `reloadAllTimelines`
- **Live Activity 在 App 被杀后仍可存活**，但无法更新（除非用推送更新）
- **Widget 预览（Xcode Preview）和真机表现可能不同**，必须真机测试
- **`TimelineProvider.getTimeline` 中禁止同步网络请求**，必须用 async/await 或 completion
- **Live Activity 的 `ContentState` 属性必须是 `Codable`**，不支持非 Codable 类型
- **Dynamic Island 在 iPhone 14 及以下机型不可用**，需检查 `ActivityAuthorizationInfo().areActivitiesEnabled`
