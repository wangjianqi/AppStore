# 122-watchOS 快速入门

## 本章目标

- 理解 Apple Watch 的产品定位与适合的 App 类型
- 掌握 watchOS 项目创建与 Target 配置方法
- 学会使用 SwiftUI 开发 watchOS 界面，包括布局、手势与 Digital Crown
- 理解 Watch 与 iPhone 之间的通信机制（WatchConnectivity）
- 了解 Complication 表盘复杂功能的开发
- 掌握 HealthKit 在 Watch 上的健康数据读取
- 理解通知与后台刷新机制
- 熟悉 watchOS 设计规范（HIG for Watch）
- 完成一个简单的计时器 Watch App 实战

---

## 1. watchOS 开发概述

### 1.1 Apple Watch 的定位

Apple Watch 是一款**手腕上的智能设备**，它的核心使用场景是**短暂、频繁、轻量**的交互——用户抬起手腕，看一眼，点一下，放下手腕，整个过程通常不超过 **10 秒**。

> 💡 **生活类比**：iPhone 像你的书桌，你可以坐下来慢慢处理事情；Apple Watch 像你手腕上的便利贴，只写最重要的信息，扫一眼就够了。

### 1.2 适合的 App 类型

| 适合的 App 类型 | 不适合的 App 类型 |
|----------------|------------------|
| 运动与健身追踪 | 长文阅读 |
| 快速消息回复 | 复杂文档编辑 |
| 心率/健康监测 | 大型游戏 |
| 计时器/秒表 | 视频播放 |
| 快捷控制（家居/音乐） | 数据密集型表格 |
| 支付与出行码 | 社交媒体浏览 |
| 导航与提醒 | 图片编辑 |

> ⚠️ **核心原则**：watchOS App 应该是 **"Glanceable"（可一瞥）** 的——用户在 2~3 秒内就能获取关键信息。

### 1.3 Watch App 与 iPhone App 的关系

| 架构模式 | 说明 | 适用场景 |
|----------|------|----------|
| **Watch-only App** | 独立运行，不依赖 iPhone | watchOS 6+，纯手表功能 |
| **Companion App** | 与 iPhone App 配合使用 | 需要手机端数据或配置 |
| **独立 + 配套** | 可独立运行，但与 iPhone 联动更强大 | 大多数推荐模式 |

从 watchOS 6 开始，Apple Watch 支持**独立 App Store**，用户可以直接在手表上下载和安装 App，无需 iPhone。

```
┌─────────────┐         WatchConnectivity        ┌─────────────┐
│  iPhone App  │◄──────────────────────────────►│  Watch App   │
│             │         消息 / 数据 / 文件          │             │
│  主数据处理  │                                   │  轻量展示    │
│  复杂逻辑    │                                   │  快速交互    │
└─────────────┘                                   └─────────────┘
```

---

## 2. 项目创建与 Target 配置

### 2.1 创建包含 Watch App 的项目

1. 打开 Xcode → **File → New → Project…**
2. 选择 **watchOS** 标签页 → **App**
3. 填写项目信息：

| 配置项 | 说明 |
|--------|------|
| Product Name | 项目名称 |
| Team | 开发团队 |
| Organization Identifier | 组织标识符 |
| Interface | **SwiftUI**（推荐） |
| Language | Swift |
| Include Complication | 是否包含表盘复杂功能 |

### 2.2 项目 Target 结构

创建完成后，项目会包含以下 Target：

| Target | 说明 |
|--------|------|
| **iOS App** | iPhone 端应用 |
| **Watch App** | Apple Watch 端应用（watchOS 7+ 简化为单一 Target） |
| **Watch Extension** | Watch 端代码逻辑（watchOS 6 及更早） |

> 💡 **watchOS 7+ 变化**：从 watchOS 7 开始，Watch App 和 Watch Extension 合并为一个 Target，结构更简洁。

### 2.3 Info.plist 关键配置

```xml
<!-- Watch App 的 Info.plist -->
<key>WKApplicationMode</key>
<string>WKApplicationModeStandard</string>

<!-- 如果需要后台运行 -->
<key>UIBackgroundModes</key>
<array>
    <string>remote-notification</string>
</array>
```

### 2.4 Capabilities 配置

| Capability | 用途 |
|------------|------|
| HealthKit | 读取健康数据 |
| Push Notifications | 接收远程通知 |
| App Groups | Watch 与 iPhone 共享数据 |
| WatchConnectivity | 与 iPhone 通信 |

---

## 3. SwiftUI for watchOS

### 3.1 布局限制与适配

Apple Watch 屏幕尺寸非常有限，开发时需要特别注意：

| 设备 | 屏幕尺寸（pt） | 实际分辨率 |
|------|---------------|-----------|
| Apple Watch SE (40mm) | 174 × 174 | 324 × 396 |
| Apple Watch SE (44mm) | 198 × 198 | 368 × 448 |
| Apple Watch Series 9 (41mm) | 178 × 178 | 352 × 430 |
| Apple Watch Series 9 (45mm) | 198 × 198 | 396 × 484 |
| Apple Watch Ultra 2 (49mm) | 208 × 208 | 410 × 502 |

> ⚠️ **注意**：watchOS 没有横屏模式，所有界面都是竖屏显示。圆形屏幕意味着四角区域会被裁切。

### 3.2 基础布局示例

```swift
import SwiftUI

struct ContentView: View {
    var body: some View {
        VStack(spacing: 8) {
            Text("当前心率")
                .font(.caption2)
                .foregroundStyle(.secondary)
            Text("72 BPM")
                .font(.title2)
                .fontWeight(.bold)
            HStack(spacing: 16) {
                Label("步行", systemImage: "figure.walk")
                Label("跑步", systemImage: "figure.run")
            }
            .font(.caption)
        }
        .padding()
    }
}
```

### 3.3 手势交互

watchOS 支持的手势较为有限，但足以应对常见场景：

| 手势 | SwiftUI 修饰符 | 说明 |
|------|---------------|------|
| 点击 | `.onTapGesture` | 最常用的交互方式 |
| 长按 | `.onLongPressGesture` | 触发次要操作 |
| 拖拽 | `.gesture(DragGesture)` | 滑动操作 |
| 捏合 | `.gesture(MagnifyGesture)` | 缩放（较少使用） |

```swift
struct GestureDemo: View {
    @State private var scale: CGFloat = 1.0

    var body: some View {
        Circle()
            .fill(.blue)
            .frame(width: 80, height: 80)
            .scaleEffect(scale)
            .onTapGesture {
                withAnimation { scale = 1.5 }
            }
            .onLongPressGesture {
                withAnimation { scale = 1.0 }
            }
    }
}
```

### 3.4 Digital Crown 数字表冠

Digital Crown 是 Apple Watch 最独特的输入方式，类似于"旋转+按压"的旋钮：

> 💡 **生活类比**：Digital Crown 就像老式收音机的调频旋钮——转动它来浏览列表或调节数值，按下去来确认选择。

```swift
struct DigitalCrownDemo: View {
    @State private var selectedItem = 0
    let items = ["心率", "步数", "卡路里", "距离", "血氧"]

    var body: some View {
        List(items, id: \.self) { item in
            Text(item)
        }
        .focusable()
        .digitalCrownRotation(
            $selectedItem,
            from: 0,
            through: Double(items.count - 1),
            by: 1,
            sensitivity: .medium,
            isContinuous: false,
            isHapticFeedbackEnabled: true
        )
    }
}
```

### 3.5 列表优化

watchOS 列表性能至关重要，以下是优化要点：

| 优化策略 | 说明 |
|----------|------|
| 使用 `List` 而非 `ScrollView` | List 有懒加载机制 |
| 保持 Cell 简单 | 避免复杂视图层级 |
| 使用 `LazyVStack` | 当必须用 ScrollView 时 |
| 避免大图加载 | 使用缩略图或 SF Symbols |
| 减少动画 | watchOS 动画消耗较大 |

```swift
struct OptimizedList: View {
    let workouts: [Workout] = []

    var body: some View {
        List(workouts) { workout in
            HStack {
                Image(systemName: workout.icon)
                    .foregroundStyle(.green)
                VStack(alignment: .leading) {
                    Text(workout.name)
                        .font(.headline)
                    Text(workout.duration)
                        .font(.caption2)
                        .foregroundStyle(.secondary)
                }
            }
        }
    }
}
```

---

## 4. Watch 与 iPhone 通信

### 4.1 WatchConnectivity 框架概述

WatchConnectivity 是 Watch 与 iPhone 之间通信的核心框架，提供了三种通信方式：

| 通信方式 | 方法 | 特点 | 有序保证 |
|----------|------|------|----------|
| **Interactive Message** | `sendMessage(_:replyHandler:)` | 实时双向，要求对方可达 | ✅ |
| **Application Context** | `updateApplicationContext(_:)` | 最新状态同步，只保留最新 | ❌ 覆盖式 |
| **User Info Transfer** | `transferUserInfo(_:)` | 队列式传输，后台也可传输 | ✅ |
| **File Transfer** | `transferFile(_:metadata:)` | 传输文件，后台也可传输 | ✅ |

> 💡 **生活类比**：Message 像打电话——对方必须在线；Context 像更新白板——只看最新内容；UserInfo 像寄信——排队送达；File Transfer 像寄快递——可以寄包裹。

### 4.2 WCSession 基础配置

```swift
import WatchConnectivity

class PhoneSessionManager: NSObject, WCSessionDelegate {
    static let shared = PhoneSessionManager()
    let session: WCSession

    override init() {
        self.session = WCSession.default
        super.init()
        session.delegate = self
        session.activate()
    }

    func session(
        _ session: WCSession,
        activationDidCompleteWith activationState: WCSessionActivationState,
        error: Error?
    ) {
        if let error = error {
            print("Session 激活失败: \(error.localizedDescription)")
        }
    }

    func sessionDidBecomeInactive(_ session: WCSession) {}
    func sessionDidDeactivate(_ session: WCSession) {
        session.activate()
    }
}
```

### 4.3 发送与接收消息

```swift
// iPhone 端发送消息
func sendToWatch(message: [String: Any]) {
    guard session.isReachable else {
        print("Watch 不可达，无法发送消息")
        return
    }
    session.sendMessage(message, replyHandler: { reply in
        print("Watch 回复: \(reply)")
    })
}

// Watch 端接收消息
func session(
    _ session: WCSession,
    didReceiveMessage message: [String: Any],
    replyHandler: @escaping ([String: Any]) -> Void
) {
    DispatchQueue.main.async {
        if let command = message["command"] as? String {
            switch command {
            case "start":
                self.handleStart()
            case "stop":
                self.handleStop()
            default:
                break
            }
        }
        replyHandler(["status": "received"])
    }
}
```

### 4.4 后台数据传输

```swift
// 发送 Application Context（只保留最新）
func updateContext(data: [String: Any]) {
    do {
        try session.updateApplicationContext(data)
    } catch {
        print("更新 Context 失败: \(error)")
    }
}

// 传输 UserInfo（队列式，保证送达）
func transferUserInfo(info: [String: Any]) {
    session.transferUserInfo(info)
}

// 传输文件
func transferFile(url: URL, metadata: [String: Any]? = nil) {
    session.transferFile(url, metadata: metadata)
}
```

> ⚠️ **注意**：`sendMessage` 要求对方 App 处于前台且可达，如果需要后台传输，请使用 `transferUserInfo` 或 `transferFile`。

---

## 5. Complication 表盘复杂功能

### 5.1 什么是 Complication

Complication 是 Apple Watch 表盘上的**小组件**，可以在不打开 App 的情况下展示关键信息。

> 💡 **生活类比**：Complication 就像你手表表盘上的小窗口——日期窗口、星期窗口、计时码表，每个小窗口显示一种信息，你不用打开任何功能就能直接看到。

### 5.2 Complication 家族与模板

| Complication 家族 | 尺寸 | 适合内容 |
|-------------------|------|----------|
| `circularSmall` | 小圆形 | 单个数据点 |
| `extraLarge` | 大矩形 | 大字体展示 |
| `graphicCorner` | 角落图形 | 角落弧线+文字 |
| `graphicCircular` | 圆形图形 | 圆形进度/图像 |
| `graphicRectangular` | 矩形图形 | 标题+正文+图标 |
| `modularSmall` | 小模块 | 简短文字 |
| `modularLarge` | 大模块 | 多行文字 |
| `utilitarianSmall` | 小实用 | 单行文字 |
| `utilitarianLarge` | 大实用 | 宽行文字 |

### 5.3 使用 SwiftUI 创建 Complication（watchOS 9+）

从 watchOS 9 开始，可以使用 SwiftUI 直接创建 Complication：

```swift
import WidgetKit
import SwiftUI

struct ComplicationEntry: TimelineEntry {
    let date: Date
    let heartRate: Int
}

struct ComplicationProvider: TimelineProvider {
    func placeholder(in context: Context) -> ComplicationEntry {
        ComplicationEntry(date: Date(), heartRate: 72)
    }

    func getSnapshot(
        in context: Context,
        completion: @escaping (ComplicationEntry) -> Void
    ) {
        completion(ComplicationEntry(date: Date(), heartRate: 72))
    }

    func getTimeline(
        in context: Context,
        completion: @escaping (Timeline<ComplicationEntry>) -> Void
    ) {
        let now = Date()
        let entry = ComplicationEntry(date: now, heartRate: fetchCurrentHeartRate())
        let timeline = Timeline(entries: [entry], policy: .atEnd)
        completion(timeline)
    }

    private func fetchCurrentHeartRate() -> Int {
        return 72
    }
}

@main
struct WatchComplication: Widget {
    var body: some WidgetConfiguration {
        StaticConfiguration(
            kind: "HeartRateComplication",
            provider: ComplicationProvider()
        ) { entry in
            ComplicationView(entry: entry)
        }
        .supportedFamilies([
            .graphicCorner,
            .graphicCircular,
            .graphicRectangular
        ])
    }
}

struct ComplicationView: View {
    let entry: ComplicationEntry

    var body: some View {
        switch widgetFamily {
        case .graphicCorner:
            Text("\(entry.heartRate) ❤️")
        case .graphicCircular:
            ZStack {
                AccessoryWidgetBackground()
                VStack(spacing: 2) {
                    Text("\(entry.heartRate)")
                        .font(.headline)
                    Text("BPM")
                        .font(.caption2)
                }
            }
        case .graphicRectangular:
            HStack {
                Image(systemName: "heart.fill")
                    .foregroundStyle(.red)
                VStack(alignment: .leading) {
                    Text("心率")
                        .font(.caption)
                    Text("\(entry.heartRate) BPM")
                        .font(.headline)
                }
            }
        default:
            Text("\(entry.heartRate)")
        }
    }
}
```

### 5.4 Timeline 策略

| Timeline 策略 | 说明 | 适用场景 |
|---------------|------|----------|
| `.atEnd` | 最后一条 Entry 过期后重新请求 | 周期性数据 |
| `.after(Date)` | 指定时间后重新请求 | 定时更新 |
| `.never` | 不再更新 | 静态数据 |

> ⚠️ **注意**：Complication 的刷新由系统控制，你只能提供"建议"，系统会根据电量和使用情况决定实际刷新频率。不要依赖 Complication 做实时数据展示。

---

## 6. 健康数据读取

### 6.1 HealthKit on Watch

Apple Watch 是 Apple 生态中最核心的健康数据采集设备。HealthKit 在 Watch 上可以直接读取心率、步数、锻炼等数据。

| 数据类型 | 读取方式 | 是否需要配对 iPhone |
|----------|---------|-------------------|
| 心率 | HKQuantityTypeIdentifier.heartRate | ❌ 可独立 |
| 步数 | HKQuantityTypeIdentifier.stepCount | ❌ 可独立 |
| 锻炼 | HKWorkout | ❌ 可独立 |
| 血氧 | HKQuantityTypeIdentifier.oxygenSaturation | ❌ 可独立 |
| 心电图 | HKElectrocardiogram | ✅ 需要配对 |
| 睡眠 | HKCategoryTypeIdentifier.sleepAnalysis | ❌ 可独立 |

### 6.2 请求权限

```swift
import HealthKit

class HealthManager: ObservableObject {
    let healthStore = HKHealthStore()

    func requestAuthorization() async {
        guard HKHealthStore.isHealthDataAvailable() else {
            print("HealthKit 不可用")
            return
        }

        let typesToRead: Set<HKObjectType> = [
            HKQuantityType(.heartRate),
            HKQuantityType(.stepCount),
            HKQuantityType(.activeEnergyBurned),
            HKObjectType.workoutType()
        ]

        do {
            try await healthStore.requestAuthorization(toShare: [], read: typesToRead)
        } catch {
            print("授权失败: \(error)")
        }
    }
}
```

### 6.3 读取心率数据

```swift
func fetchLatestHeartRate() async -> Double? {
    let heartRateType = HKQuantityType(.heartRate)
    let sortDescriptor = NSSortDescriptor(
        key: HKSampleSortIdentifierStartDate,
        ascending: false
    )

    let query = HKSampleQuery(
        sampleType: heartRateType,
        predicate: nil,
        limit: 1,
        sortDescriptors: [sortDescriptor]
    ) { _, samples, error in
        // 处理结果
    }

    healthStore.execute(query)

    let samples = try? await withCheckedThrowingContinuation {
        (continuation: CheckedContinuation<[HKSample]?, Error>) in
        let query = HKSampleQuery(
            sampleType: heartRateType,
            predicate: nil,
            limit: 1,
            sortDescriptors: [sortDescriptor]
        ) { _, samples, error in
            if let error = error {
                continuation.resume(throwing: error)
            } else {
                continuation.resume(returning: samples)
            }
        }
        healthStore.execute(query)
    }

    guard let quantitySample = samples?.first as? HKQuantitySample else {
        return nil
    }

    let heartRateUnit = HKUnit.count().unitDivided(by: .minute())
    return quantitySample.quantity.doubleValue(for: heartRateUnit)
}
```

### 6.4 查询步数

```swift
func fetchTodaySteps() async -> Double {
    let stepType = HKQuantityType(.stepCount)
    let now = Date()
    let startOfDay = Calendar.current.startOfDay(for: now)

    let predicate = HKQuery.predicateForSamples(
        withStart: startOfDay,
        end: now,
        options: .strictStartDate
    )

    let query = HKStatisticsQuery(
        quantityType: stepType,
        quantitySamplePredicate: predicate,
        options: .cumulativeSum
    ) { _, statistics, _ in
        let sum = statistics?.sumQuantity()
        let steps = sum?.doubleValue(for: .count()) ?? 0
    }

    healthStore.execute(query)
    return 0
}
```

> ⚠️ **注意**：在 Watch App 的 Info.plist 中需要添加 `NSHealthShareUsageDescription`（读取权限描述）和 `NSHealthUpdateUsageDescription`（写入权限描述），并在 Capabilities 中开启 HealthKit。

---

## 7. 通知与后台刷新

### 7.1 远程通知

watchOS 支持接收远程推送通知，用户可以直接在手表上查看和回复：

```json
{
    "aps": {
        "alert": {
            "title": "运动提醒",
            "body": "你已经坐了1小时，起来活动一下吧！"
        },
        "category": "ACTIVITY_REMINDER",
        "sound": "default"
    }
}
```

### 7.2 通知界面自定义

```swift
import SwiftUI

struct NotificationView: View {
    let title: String
    let message: String

    var body: some View {
        VStack(spacing: 8) {
            Image(systemName: "figure.walk")
                .font(.title2)
                .foregroundStyle(.green)
            Text(title)
                .font(.headline)
            Text(message)
                .font(.caption)
                .multilineTextAlignment(.center)
        }
        .padding()
    }
}
```

### 7.3 快照（Snapshot）

快照是 App 在 Dock 中显示的静态预览图。系统会在以下时机请求快照：

| 触发时机 | 说明 |
|----------|------|
| App 进入后台 | 系统自动请求 |
| 用户打开 Dock | 展示最近快照 |
| 时间变化 | 可能刷新快照 |

```swift
class SnapshotDelegate: NSObject, WKSnapshotRefreshBackgroundTaskDelegate {
    func handle(_ task: WKSnapshotRefreshBackgroundTask) {
        let snapshotDate = Date()
        let configuration = WKSnapshotConfiguration()
        configuration.rect = CGRect(x: 0, y: 0, width: 208, height: 208)

        task.setTaskCompleted(
            restoredDefaultState: true,
            estimatedSnapshotExpiration: Date(timeIntervalSinceNow: 3600),
            userInfo: nil
        )
    }
}
```

### 7.4 后台刷新 API

watchOS 提供了**计划后台刷新**的能力，让 App 在后台也能更新数据：

```swift
import WatchKit

func scheduleBackgroundRefresh() {
    let targetDate = Date(timeIntervalSinceNow: 30 * 60)

    WKExtension.shared().scheduleBackgroundRefresh(
        withPreferredDate: targetDate,
        userInfo: nil
    ) { error in
        if let error = error {
            print("调度后台刷新失败: \(error)")
        }
    }
}

func handle(_ backgroundTasks: Set<WKRefreshBackgroundTask>) {
    for task in backgroundTasks {
        switch task {
        case let backgroundTask as WKApplicationRefreshBackgroundTask:
            updateDataInBackground()
            scheduleBackgroundRefresh()
            backgroundTask.setTaskCompletedWithSnapshot(false)
        case let snapshotTask as WKSnapshotRefreshBackgroundTask:
            snapshotTask.setTaskCompleted(
                restoredDefaultState: true,
                estimatedSnapshotExpiration: Date(timeIntervalSinceNow: 3600),
                userInfo: nil
            )
        default:
            task.setTaskCompletedWithSnapshot(false)
        }
    }
}
```

> ⚠️ **注意**：后台刷新的调度时间只是"建议"，系统会根据电量、使用频率等因素决定实际执行时间。不要依赖精确的后台刷新时间。

---

## 8. watchOS 设计规范

### 8.1 字体规范

| 字体样式 | 用途 | 大小参考 |
|----------|------|----------|
| `.title` | 页面主标题 | 20pt |
| `.title2` | 次级标题 | 18pt |
| `.title3` | 三级标题 | 16pt |
| `.headline` | 列表项标题 | 15pt |
| `.body` | 正文 | 15pt |
| `.callout` | 强调文字 | 14pt |
| `.caption` | 辅助说明 | 12pt |
| `.caption2` | 最小文字 | 11pt |

> 💡 **建议**：watchOS 上尽量使用 `.title2` 或更大的字体，过小的文字在手腕上难以阅读。

### 8.2 间距与布局规范

| 规范项 | 建议值 |
|--------|--------|
| 页面左右内边距 | 16pt |
| 元素间距 | 8~12pt |
| 可点击区域最小尺寸 | 44×44pt |
| 列表项高度 | 建议 44pt 以上 |
| 圆形屏幕内容安全区 | 距边缘 8pt 以上 |

### 8.3 交互规范（HIG for Watch 要点）

| 要点 | 说明 |
|------|------|
| **Glanceable** | 信息一目了然，2~3 秒内获取关键内容 |
| **Actionable** | 减少步骤，一键完成核心操作 |
| **Responsive** | 即时反馈，避免等待 |
| **Minimal Text** | 精简文字，用图标和颜色传达信息 |
| **Use Color Purposefully** | 颜色用于传达含义，而非装饰 |
| **Avoid Scrolling** | 尽量让内容在一屏内展示完整 |
| **Prefer Lists** | 列表是 watchOS 最自然的交互方式 |
| **Use Haptics** | 触觉反馈增强交互感知 |

### 8.4 颜色使用

```swift
// watchOS 推荐使用系统语义颜色
Text("正常文字").foregroundStyle(.primary)
Text("次要文字").foregroundStyle(.secondary)
Text("强调内容").foregroundStyle(.tint)

// 状态颜色
Text("成功").foregroundStyle(.green)
Text("警告").foregroundStyle(.yellow)
Text("错误").foregroundStyle(.red)
```

> 💡 **生活类比**：设计 Watch App 就像设计一张高速公路路牌——信息必须极度精简，字体必须够大，颜色必须醒目，司机（用户）以 100km/h 的速度（抬腕一瞥）也能看懂。

---

## 9. 实战：创建一个简单的计时器 Watch App

### 9.1 项目创建

1. Xcode → **File → New → Project…**
2. 选择 **watchOS → App**
3. Product Name: **WatchTimer**
4. Interface: **SwiftUI**
5. Language: **Swift**

### 9.2 数据模型

```swift
import Foundation

enum TimerState {
    case stopped
    case running
    case paused
}

class TimerManager: ObservableObject {
    @Published var remainingSeconds: Int = 0
    @Published var totalSeconds: Int = 300
    @Published var state: TimerState = .stopped

    private var timer: Timer?

    var progress: Double {
        guard totalSeconds > 0 else { return 0 }
        return Double(remainingSeconds) / Double(totalSeconds)
    }

    var displayTime: String {
        let minutes = remainingSeconds / 60
        let seconds = remainingSeconds % 60
        return String(format: "%d:%02d", minutes, seconds)
    }

    func start() {
        state = .running
        remainingSeconds = totalSeconds
        timer = Timer.scheduledTimer(
            withTimeInterval: 1.0,
            repeats: true
        ) { [weak self] _ in
            self?.tick()
        }
    }

    func pause() {
        state = .paused
        timer?.invalidate()
        timer = nil
    }

    func resume() {
        state = .running
        timer = Timer.scheduledTimer(
            withTimeInterval: 1.0,
            repeats: true
        ) { [weak self] _ in
            self?.tick()
        }
    }

    func stop() {
        state = .stopped
        timer?.invalidate()
        timer = nil
        remainingSeconds = totalSeconds
    }

    func setDuration(minutes: Int) {
        totalSeconds = minutes * 60
        remainingSeconds = totalSeconds
    }

    private func tick() {
        if remainingSeconds > 0 {
            remainingSeconds -= 1
        } else {
            stop()
            WKInterfaceDevice.current().play(.notification)
        }
    }
}
```

### 9.3 主界面

```swift
import SwiftUI

struct ContentView: View {
    @StateObject private var timerManager = TimerManager()

    var body: some View {
        VStack(spacing: 12) {
            ProgressView(
                value: timerManager.progress
            )
            .tint(timerManager.state == .running ? .green : .orange)

            Text(timerManager.displayTime)
                .font(.title2)
                .fontWeight(.bold)
                .monospacedDigit()

            HStack(spacing: 20) {
                switch timerManager.state {
                case .stopped:
                    Button(action: { timerManager.start() }) {
                        Image(systemName: "play.fill")
                            .font(.title3)
                    }
                    .buttonStyle(.borderedProminent)
                    .tint(.green)

                case .running:
                    Button(action: { timerManager.pause() }) {
                        Image(systemName: "pause.fill")
                            .font(.title3)
                    }
                    .buttonStyle(.borderedProminent)
                    .tint(.orange)

                case .paused:
                    Button(action: { timerManager.resume() }) {
                        Image(systemName: "play.fill")
                            .font(.title3)
                    }
                    .buttonStyle(.borderedProminent)
                    .tint(.green)
                }

                if timerManager.state != .stopped {
                    Button(action: { timerManager.stop() }) {
                        Image(systemName: "stop.fill")
                            .font(.title3)
                    }
                    .buttonStyle(.bordered)
                    .tint(.red)
                }
            }

            if timerManager.state == .stopped {
                DurationPicker(
                    totalSeconds: $timerManager.totalSeconds
                )
                .labelsHidden()
            }
        }
        .padding()
    }
}
```

### 9.4 Complication 支持

为计时器 App 添加一个图形矩形 Complication，让用户在表盘上直接看到倒计时：

```swift
import WidgetKit
import SwiftUI

struct TimerComplicationEntry: TimelineEntry {
    let date: Date
    let remainingSeconds: Int
    let isRunning: Bool
}

struct TimerComplicationProvider: TimelineProvider {
    func placeholder(in context: Context) -> TimerComplicationEntry {
        TimerComplicationEntry(date: Date(), remainingSeconds: 300, isRunning: true)
    }

    func getSnapshot(
        in context: Context,
        completion: @escaping (TimerComplicationEntry) -> Void
    ) {
        completion(TimerComplicationEntry(date: Date(), remainingSeconds: 300, isRunning: true))
    }

    func getTimeline(
        in context: Context,
        completion: @escaping (Timeline<TimerComplicationEntry>) -> Void
    ) {
        let now = Date()
        let entry = TimerComplicationEntry(
            date: now,
            remainingSeconds: 300,
            isRunning: true
        )
        let timeline = Timeline(entries: [entry], policy: .after(now.addingTimeInterval(60)))
        completion(timeline)
    }
}

struct TimerComplicationView: View {
    let entry: TimerComplicationEntry

    var body: some View {
        HStack {
            Image(systemName: entry.isRunning ? "timer" : "timer")
                .foregroundStyle(entry.isRunning ? .green : .orange)
            VStack(alignment: .leading) {
                Text("计时器")
                    .font(.caption2)
                Text(formatTime(entry.remainingSeconds))
                    .font(.headline)
            }
        }
    }

    private func formatTime(_ seconds: Int) -> String {
        let m = seconds / 60
        let s = seconds % 60
        return String(format: "%d:%02d", m, s)
    }
}
```

### 9.5 运行与调试

| 步骤 | 操作 |
|------|------|
| 1 | 选择 Watch App Target |
| 2 | 选择 Apple Watch 模拟器或真机 |
| 3 | 点击 Run（⌘R）运行 |
| 4 | 在模拟器中测试计时器功能 |
| 5 | 测试 Complication：长按表盘 → 添加 Complication → 选择 WatchTimer |

> 💡 **调试技巧**：在 Xcode 中可以选择直接运行 Watch App，而不需要先运行 iPhone App。watchOS 6+ 的独立 App 使得调试更加方便。

---

## 本章小结

| 主题 | 核心要点 |
|------|----------|
| **watchOS 概述** | 短暂、频繁、轻量的交互；Glanceable 原则 |
| **项目配置** | watchOS 7+ 单一 Target；注意 Capabilities 配置 |
| **SwiftUI for watchOS** | 小屏幕布局；Digital Crown；列表优化 |
| **Watch-iPhone 通信** | WatchConnectivity 四种方式；Message 实时、UserInfo 可靠 |
| **Complication** | 表盘小组件；Timeline 策略；SwiftUI Widget（watchOS 9+） |
| **健康数据** | HealthKit 直接在 Watch 上读取；心率/步数/锻炼 |
| **通知与后台** | 远程通知；快照预览；计划后台刷新 |
| **设计规范** | 大字体、精简文字、最少滚动、触觉反馈 |
| **实战计时器** | SwiftUI + Timer + Complication 完整流程 |
