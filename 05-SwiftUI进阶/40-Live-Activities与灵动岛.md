# 60-Live Activities 与灵动岛

## 本章目标

- 理解 Live Activities 的概念与使用场景
- 掌握灵动岛的三种展示形态及设计规范
- 学会使用 ActivityKit 创建、更新、结束 Live Activity
- 能够实现灵动岛的 WidgetKit 小组件扩展 UI
- 完成一个外卖配送进度追踪的实战项目
- 了解 Live Activity 的设计最佳实践与审核注意事项

---

## 1. Live Activities 简介

### 1.1 什么是 Live Activities？

Live Activities 是 Apple 在 iOS 16.1 中引入的一项功能，允许 App 在**锁屏界面**和**灵动岛**上展示实时、动态的信息。

> 💡 生活类比：想象你在等外卖，以前你得反复解锁手机、打开 App 查看进度。有了 Live Activities，锁屏上直接显示"骑手距你 500 米"，就像楼下的电子屏实时显示快递状态一样方便。

### 1.2 Live Activities vs 传统通知

| 特性 | 传统通知 | Live Activities |
|------|---------|----------------|
| 展示位置 | 锁屏/通知中心 | 锁屏 + 灵动岛 |
| 信息更新 | 每次发新通知 | 原地更新，不堆积 |
| 交互性 | 点击跳转 | 可含按钮交互 |
| 持续时间 | 永久保留 | 最长 12 小时 |
| 用户感知 | 容易忽略 | 始终可见 |
| 主动关闭 | 手动删除 | 用户可滑掉 |

### 1.3 典型使用场景

| 场景 | 展示内容 | 为什么适合 |
|------|---------|-----------|
| 🍔 外卖配送 | 骑手位置、预计送达时间 | 用户高频查看，信息实时变化 |
| ⚽ 运动比分 | 实时比分、比赛时间 | 比分持续更新，用户不想错过 |
| 🚗 打车出行 | 司机位置、预计到达 | 等待焦虑，需要实时安抚 |
| ⏱️ 倒计时 | 剩余时间 | 时间流逝是天然动态内容 |
| 🏋️ 健身训练 | 运动时长、心率 | 运动中不方便操作手机 |
| 🎫 登机牌 | 登机口、登机时间 | 出行场景需要快速查看 |

> ⚠️ Live Activities 不是万能的！它适合**有明确开始和结束**的实时任务。如果你的信息不需要频繁更新，或者没有明确的结束点，传统通知可能更合适。

---

## 2. 灵动岛（Dynamic Island）设计规范

### 2.1 什么是灵动岛？

灵动岛是 iPhone 14 Pro 及后续机型顶部的交互区域，它将前置摄像头和传感器融合为一个动态的交互元素。Live Activities 可以在灵动岛上展示信息。

> 💡 生活类比：灵动岛就像一个"变形金刚"——平时安安静静待在顶部，需要时可以膨胀展开显示更多信息，用完又缩回去。

### 2.2 三种展示形态

灵动岛有三种展示形态，每种形态适用于不同的场景：

| 形态 | 尺寸 | 何时显示 | 用途 |
|------|------|---------|------|
| **紧凑（Compact）** | 两侧小药丸 | 前台运行时默认 | 展示最核心信息 |
| **最小（Minimal）** | 单侧小圆 | 有其他 App 的 Live Activity 在前台 | 仅展示图标或极简信息 |
| **扩展（Expanded）** | 大面积弹窗 | 长按灵动岛 | 展示详细信息与交互 |

#### 形态示意

```
┌─────────────────────────────────────┐
│  紧凑（Compact）                      │
│  ┌──────┐          ┌──────┐         │
│  │ 🍔 5min │          │ 🏠 2km │         │
│  └──────┘          └──────┘         │
├─────────────────────────────────────┤
│  最小（Minimal）                      │
│  ┌──┐                               │
│  │🍔│                               │
│  └──┘                               │
├─────────────────────────────────────┤
│  扩展（Expanded）                     │
│  ┌────────────────────────────┐     │
│  │  🍔 外卖配送中               │     │
│  │  骑手距你 500m              │     │
│  │  预计 5 分钟送达             │     │
│  │  [联系骑手]  [查看地图]      │     │
│  └────────────────────────────┘     │
└─────────────────────────────────────┘
```

### 2.3 设计要点

| 要点 | 说明 |
|------|------|
| 信息精简 | 紧凑形态空间极小，只放最关键信息 |
| 层级分明 | 紧凑→核心数据，扩展→详细内容+交互 |
| 颜色克制 | 使用 App 主色调，避免花哨 |
| 动画流畅 | 状态变化时使用平滑过渡动画 |
| 无文字滚动 | 灵动岛不支持滚动文字 |

> ⚠️ 灵动岛仅在 iPhone 14 Pro 及更新机型上显示。在旧机型上，Live Activities 只会出现在锁屏上。

---

## 3. ActivityKit 框架

### 3.1 ActivityKit 是什么？

ActivityKit 是 Apple 提供的框架，用于创建和管理 Live Activities。它的核心概念非常简单：

> 💡 生活类比：ActivityKit 就像一个"公告栏管理员"——你告诉它要贴什么公告（创建）、改什么内容（更新）、什么时候撤掉（结束）。

### 3.2 核心类型：ActivityAttributes

`ActivityAttributes` 是 Live Activities 的数据模型，定义了哪些数据是**静态的**（创建后不变），哪些是**动态的**（可以随时更新）。

```swift
import ActivityKit

struct FoodDeliveryAttributes: ActivityAttributes {
    let orderName: String
    let orderNumber: String

    struct ContentState: Codable, Hashable {
        var driverName: String
        var estimatedDeliveryTime: Date
        var distance: Double
        var status: DeliveryStatus
    }
}

enum DeliveryStatus: String, Codable {
    case confirmed = "已接单"
    case preparing = "制作中"
    case pickedUp = "骑手取餐"
    case onTheWay = "配送中"
    case arrived = "已送达"
}
```

#### 数据分类

| 类型 | 定义位置 | 特点 | 示例 |
|------|---------|------|------|
| 静态数据 | `ActivityAttributes` 属性 | 创建后不变 | 订单号、餐品名 |
| 动态数据 | `ContentState` 属性 | 可随时更新 | 骑手位置、预计时间 |

> 💡 `ContentState` 必须嵌套在 `ActivityAttributes` 内部，且必须遵循 `Codable` 和 `Hashable` 协议。

### 3.3 ActivityKit 核心流程

```
创建 Activity → 更新 ContentState → 结束 Activity
     ↓                  ↓                  ↓
  锁屏/灵动岛        实时刷新内容        移除或保留在锁屏
  开始展示
```

---

## 4. 创建 Live Activity

### 4.1 请求权限

在启动 Live Activity 之前，需要先检查用户是否允许：

```swift
import ActivityKit

func checkLiveActivityAvailability() -> Bool {
    return ActivityAuthorizationInfo().areActivitiesEnabled
}
```

> ⚠️ Live Activities 不需要像推送通知那样显式请求权限。用户可以在**设置 → 你的 App → Live Activities** 中关闭，所以每次启动前都要检查。

### 4.2 启动 Activity

```swift
func startDeliveryActivity(order: Order) async throws {
    let attributes = FoodDeliveryAttributes(
        orderName: order.name,
        orderNumber: order.number
    )

    let initialState = FoodDeliveryAttributes.ContentState(
        driverName: "张师傅",
        estimatedDeliveryTime: Date().addingTimeInterval(30 * 60),
        distance: 3.2,
        status: .confirmed
    )

    let activity = try Activity.request(
        attributes: attributes,
        content: .init(state: initialState, staleDate: nil),
        pushType: nil
    )

    print("Live Activity 已启动，ID: \(activity.id)")
}
```

#### Activity.request 参数说明

| 参数 | 类型 | 说明 |
|------|------|------|
| `attributes` | `ActivityAttributes` | 静态数据 |
| `content` | `ActivityContent` | 动态数据 + 过期时间 |
| `pushType` | `PushType?` | 推送更新类型，`nil` 表示仅本地更新 |

### 4.3 监听 Activity 状态

```swift
func observeActivity(activity: Activity<FoodDeliveryAttributes>) {
    Task {
        for await state in activity.activityStateUpdates {
            switch state {
            case .active:
                print("Activity 活跃中")
            case .dismissed:
                print("用户已关闭 Activity")
            case .ended:
                print("Activity 已结束")
            case .stale:
                print("Activity 数据已过期")
            @unknown default:
                break
            }
        }
    }
}
```

---

## 5. 灵动岛 UI 实现

### 5.1 添加 Widget Extension

Live Activities 的 UI 是通过 **WidgetKit 小组件扩展**来实现的。你需要：

1. 在 Xcode 中 **File → New → Target → Widget Extension**
2. 在 Widget 中实现 `LiveActivity` 协议

> 💡 生活类比：Widget Extension 就像给 App 开了一个"橱窗"——App 本身在店里忙活，橱窗负责向路人展示最新信息。

### 5.2 项目结构

```
YourApp/
├── YourApp.swift              ← 主 App
├── Models/
│   └── FoodDeliveryAttributes.swift  ← 共享的数据模型
└── YourWidget/                ← Widget Extension
    ├── YourWidget.swift       ← Widget + Live Activity UI
    └── Info.plist
```

> ⚠️ `ActivityAttributes` 模型必须在主 App 和 Widget Extension 之间共享。推荐做法是将模型放在共享的 Framework 或使用 App Group。

### 5.3 实现 LiveActivity 协议

```swift
import WidgetKit
import ActivityKit

@main
struct FoodDeliveryWidget: Widget {
    var body: some WidgetConfiguration {
        ActivityConfiguration(for: FoodDeliveryAttributes.self) { context in
            // 锁屏 Live Activity UI
            LockScreenLiveActivityView(context: context)
        } dynamicIsland: { context in
            DynamicIsland {
                // 扩展形态
                DynamicIslandExpandedRegion(.leading) {
                    Label("配送中", systemImage: "bicycle")
                        .font(.caption2)
                        .foregroundColor(.orange)
                }
                DynamicIslandExpandedRegion(.trailing) {
                    Text(context.state.estimatedDeliveryTime, style: .timer)
                        .font(.caption2)
                        .foregroundColor(.secondary)
                }
                DynamicIslandExpandedRegion(.center) {
                    Text(context.state.status.rawValue)
                        .font(.title3)
                        .bold()
                }
                DynamicIslandExpandedRegion(.bottom) {
                    HStack {
                        Text("骑手: \(context.state.driverName)")
                            .font(.caption)
                        Spacer()
                        Text("\(String(format: "%.1f", context.state.distance))km")
                            .font(.caption)
                            .foregroundColor(.blue)
                    }
                }
            } compactLeading: {
                // 紧凑形态 - 左侧
                Label {
                    Text(context.attributes.orderName)
                        .font(.caption2)
                } icon: {
                    Image(systemName: "takeoutbag.and.cup.and.straw")
                        .foregroundColor(.orange)
                }
            } compactTrailing: {
                // 紧凑形态 - 右侧
                Text(context.state.estimatedDeliveryTime, style: .timer)
                    .font(.caption2)
                    .foregroundColor(.secondary)
            } minimal: {
                // 最小形态
                Image(systemName: "bicycle")
                    .foregroundColor(.orange)
            }
        }
    }
}
```

### 5.4 锁屏 Live Activity 视图

```swift
struct LockScreenLiveActivityView: View {
    let context: ActivityViewContext<FoodDeliveryAttributes>

    var body: some View {
        HStack(spacing: 12) {
            Image(systemName: "bicycle")
                .font(.title2)
                .foregroundColor(.orange)

            VStack(alignment: .leading, spacing: 4) {
                Text(context.attributes.orderName)
                    .font(.headline)
                Text(context.state.status.rawValue)
                    .font(.subheadline)
                    .foregroundColor(.secondary)
            }

            Spacer()

            VStack(alignment: .trailing, spacing: 4) {
                Text(context.state.estimatedDeliveryTime, style: .timer)
                    .font(.title3)
                    .monospacedDigit()
                Text("\(String(format: "%.1f", context.state.distance))km")
                    .font(.caption)
                    .foregroundColor(.blue)
            }
        }
        .padding()
    }
}
```

### 5.5 灵动岛区域布局

扩展形态的灵动岛分为四个区域：

```
┌─────────────────────────────────────┐
│  .leading        │       .trailing  │
│                  │                  │
├──────────────────┴──────────────────┤
│              .center                │
│                                     │
├─────────────────────────────────────┤
│              .bottom                │
└─────────────────────────────────────┘
```

| 区域 | 位置 | 适合放什么 |
|------|------|-----------|
| `.leading` | 左上 | 图标、标签 |
| `.trailing` | 右上 | 倒计时、数字 |
| `.center` | 中间 | 标题、状态文字 |
| `.bottom` | 底部 | 详细信息、进度条 |

---

## 6. 更新与结束 Live Activity

### 6.1 本地更新

```swift
func updateDeliveryStatus(activity: Activity<FoodDeliveryAttributes>) async {
    let newState = FoodDeliveryAttributes.ContentState(
        driverName: "张师傅",
        estimatedDeliveryTime: Date().addingTimeInterval(15 * 60),
        distance: 1.5,
        status: .onTheWay
    )

    await activity.update(
        ActivityContent(
            state: newState,
            staleDate: Date().addingTimeInterval(5 * 60)
        )
    )
}
```

#### staleDate 说明

| staleDate | 效果 |
|-----------|------|
| `nil` | 内容永不过期 |
| 具体时间 | 超过该时间后，系统可能降低更新频率或标记为过期 |

> 💡 建议始终设置 `staleDate`，这样即使 App 崩溃或网络断开，用户也能知道信息可能已过时。

### 6.2 结束 Activity

```swift
func endDeliveryActivity(activity: Activity<FoodDeliveryAttributes>) async {
    let finalState = FoodDeliveryAttributes.ContentState(
        driverName: "张师傅",
        estimatedDeliveryTime: Date(),
        distance: 0,
        status: .arrived
    )

    await activity.end(
        ActivityContent(state: finalState, staleDate: nil),
        dismissalPolicy: .after(.now + 10 * 60)
    )
}
```

#### dismissalPolicy 选项

| 策略 | 说明 |
|------|------|
| `.immediate` | 立即从锁屏移除 |
| `.after(date)` | 在指定时间后自动移除 |
| `.default` | 系统默认行为（约 4 小时后移除） |

### 6.3 推送更新（远程）

当 App 不在前台时，可以通过**推送通知**来更新 Live Activity：

```swift
// 启动时指定 pushType
let activity = try Activity.request(
    attributes: attributes,
    content: .init(state: initialState, staleDate: nil),
    pushType: .token
)

// 获取推送 token
let pushToken = await activity.pushToken
let tokenString = pushToken.map { String(format: "%02x", $0) }.joined()
print("Push Token: \(tokenString)")
```

#### 推送 payload 格式

```json
{
    "aps": {
        "timestamp": 1700000000,
        "event": "update",
        "content-state": {
            "driverName": "李师傅",
            "estimatedDeliveryTime": 7100000000,
            "distance": 0.5,
            "status": "arrived"
        },
        "alert": {
            "title": "外卖已送达",
            "body": "请及时取餐"
        }
    }
}
```

#### 推送 event 类型

| event | 说明 |
|-------|------|
| `"update"` | 更新 ContentState |
| `"end"` | 结束 Live Activity |

> ⚠️ 推送更新需要你的服务器实现，且推送 token 可能会变化，需要监听 `activity.pushTokenUpdates` 并及时上报服务器。

### 6.4 完整的更新流程图

```
App 前台运行？
    │
    ├── 是 → 本地更新（activity.update）
    │
    └── 否 → 服务器发送推送更新
              │
              ├── 推送到达 → 系统更新 Live Activity
              │
              └── 推送未到达 → 内容标记为 stale
```

---

## 7. 实战示例：外卖配送进度追踪

### 7.1 数据模型

在主 App 和 Widget Extension 共享的文件中定义：

```swift
// FoodDeliveryAttributes.swift
import ActivityKit
import Foundation

struct FoodDeliveryAttributes: ActivityAttributes {
    let orderName: String
    let orderNumber: String
    let restaurantName: String

    struct ContentState: Codable, Hashable {
        var driverName: String
        var estimatedDeliveryTime: Date
        var distance: Double
        var status: DeliveryStatus
        var progress: Double
    }
}

enum DeliveryStatus: String, Codable {
    case confirmed = "已接单"
    case preparing = "制作中"
    case pickedUp = "骑手取餐"
    case onTheWay = "配送中"
    case arrived = "已送达"
}
```

### 7.2 Live Activity 管理器

```swift
// DeliveryActivityManager.swift
import ActivityKit
import Foundation

@MainActor
class DeliveryActivityManager: ObservableObject {
    private var currentActivity: Activity<FoodDeliveryAttributes>?

    func start(orderName: String, orderNumber: String, restaurantName: String) throws {
        guard ActivityAuthorizationInfo().areActivitiesEnabled else {
            print("Live Activities 未启用")
            return
        }

        let attributes = FoodDeliveryAttributes(
            orderName: orderName,
            orderNumber: orderNumber,
            restaurantName: restaurantName
        )

        let initialState = FoodDeliveryAttributes.ContentState(
            driverName: "等待分配",
            estimatedDeliveryTime: Date().addingTimeInterval(35 * 60),
            distance: 5.0,
            status: .confirmed,
            progress: 0.1
        )

        let activity = try Activity.request(
            attributes: attributes,
            content: .init(state: initialState, staleDate: Date().addingTimeInterval(10 * 60)),
            pushType: nil
        )

        self.currentActivity = activity
        observeActivityState()
    }

    func update(status: DeliveryStatus, driverName: String, distance: Double, progress: Double) async {
        guard let activity = currentActivity else { return }

        let newState = FoodDeliveryAttributes.ContentState(
            driverName: driverName,
            estimatedDeliveryTime: Date().addingTimeInterval(Double(distance) * 3 * 60),
            distance: distance,
            status: status,
            progress: progress
        )

        await activity.update(
            .init(state: newState, staleDate: Date().addingTimeInterval(5 * 60))
        )
    }

    func end() async {
        guard let activity = currentActivity else { return }

        let finalState = FoodDeliveryAttributes.ContentState(
            driverName: "",
            estimatedDeliveryTime: Date(),
            distance: 0,
            status: .arrived,
            progress: 1.0
        )

        await activity.end(
            .init(state: finalState, staleDate: nil),
            dismissalPolicy: .after(.now + 10 * 60)
        )

        self.currentActivity = nil
    }

    private func observeActivityState() {
        guard let activity = currentActivity else { return }
        Task {
            for await state in activity.activityStateUpdates {
                if state == .dismissed || state == .ended {
                    self.currentActivity = nil
                }
            }
        }
    }
}
```

### 7.3 Widget Extension 完整实现

```swift
// FoodDeliveryWidget.swift
import WidgetKit
import ActivityKit
import SwiftUI

@main
struct FoodDeliveryWidgets: WidgetBundle {
    var body: some Widget {
        FoodDeliveryLiveActivity()
    }
}

struct FoodDeliveryLiveActivity: Widget {
    var body: some WidgetConfiguration {
        ActivityConfiguration(for: FoodDeliveryAttributes.self) { context in
            lockScreenView(context: context)
        } dynamicIsland: { context in
            DynamicIsland {
                DynamicIslandExpandedRegion(.leading) {
                    VStack {
                        Image(systemName: statusIcon(for: context.state.status))
                            .font(.title3)
                            .foregroundColor(.orange)
                        Text(context.state.status.rawValue)
                            .font(.caption2)
                            .foregroundColor(.secondary)
                    }
                }
                DynamicIslandExpandedRegion(.trailing) {
                    VStack {
                        Text(context.state.estimatedDeliveryTime, style: .timer)
                            .font(.title3)
                            .monospacedDigit()
                        Text("预计送达")
                            .font(.caption2)
                            .foregroundColor(.secondary)
                    }
                }
                DynamicIslandExpandedRegion(.center) {
                    Text(context.attributes.orderName)
                        .font(.headline)
                }
                DynamicIslandExpandedRegion(.bottom) {
                    VStack(spacing: 6) {
                        ProgressView(value: context.state.progress)
                            .tint(.orange)

                        HStack {
                            Text("骑手: \(context.state.driverName)")
                                .font(.caption)
                            Spacer()
                            Text("\(String(format: "%.1f", context.state.distance))km")
                                .font(.caption)
                                .foregroundColor(.blue)
                        }
                    }
                    .padding(.horizontal, 4)
                }
            } compactLeading: {
                HStack(spacing: 2) {
                    Image(systemName: "takeoutbag.and.cup.and.straw")
                        .font(.caption2)
                        .foregroundColor(.orange)
                    Text(context.state.status.rawValue)
                        .font(.caption2)
                }
            } compactTrailing: {
                Text(context.state.estimatedDeliveryTime, style: .timer)
                    .font(.caption2)
                    .monospacedDigit()
                    .foregroundColor(.secondary)
            } minimal: {
                Image(systemName: "bicycle")
                    .foregroundColor(.orange)
            }
        }
    }

    private func lockScreenView(context: ActivityViewContext<FoodDeliveryAttributes>) -> some View {
        HStack(spacing: 14) {
            Image(systemName: statusIcon(for: context.state.status))
                .font(.title)
                .foregroundColor(.orange)

            VStack(alignment: .leading, spacing: 4) {
                Text(context.attributes.orderName)
                    .font(.headline)
                Text("\(context.attributes.restaurantName) · \(context.state.status.rawValue)")
                    .font(.subheadline)
                    .foregroundColor(.secondary)
                ProgressView(value: context.state.progress)
                    .tint(.orange)
            }

            Spacer()

            VStack(alignment: .trailing, spacing: 4) {
                Text(context.state.estimatedDeliveryTime, style: .timer)
                    .font(.title3)
                    .monospacedDigit()
                Text("骑手: \(context.state.driverName)")
                    .font(.caption2)
                    .foregroundColor(.secondary)
                Text("\(String(format: "%.1f", context.state.distance))km")
                    .font(.caption)
                    .foregroundColor(.blue)
            }
        }
        .padding(16)
    }
}

private func statusIcon(for status: DeliveryStatus) -> String {
    switch status {
    case .confirmed: return "checkmark.circle"
    case .preparing: return "flame"
    case .pickedUp: return "bag"
    case .onTheWay: return "bicycle"
    case .arrived: return "house"
    }
}
```

### 7.4 在主 App 中使用

```swift
// ContentView.swift
import SwiftUI

struct ContentView: View {
    @StateObject private var manager = DeliveryActivityManager()
    @State private var orderName = "黄焖鸡米饭"
    @State private var orderNumber = "MT20260520001"
    @State private var restaurantName = "老王家常菜"

    var body: some View {
        VStack(spacing: 20) {
            Text("外卖配送追踪")
                .font(.largeTitle)
                .bold()

            VStack(alignment: .leading, spacing: 8) {
                Label("订单: \(orderNumber)", systemImage: "number")
                Label("餐品: \(orderName)", systemImage: "takeoutbag.and.cup.and.straw")
                Label("商家: \(restaurantName)", systemImage: "storefront")
            }
            .font(.subheadline)

            Divider()

            Button("开始配送追踪") {
                try? manager.start(
                    orderName: orderName,
                    orderNumber: orderNumber,
                    restaurantName: restaurantName
                )
            }
            .buttonStyle(.borderedProminent)

            Button("更新: 制作中") {
                Task {
                    await manager.update(
                        status: .preparing,
                        driverName: "张师傅",
                        distance: 5.0,
                        progress: 0.3
                    )
                }
            }
            .buttonStyle(.bordered)

            Button("更新: 配送中") {
                Task {
                    await manager.update(
                        status: .onTheWay,
                        driverName: "张师傅",
                        distance: 1.5,
                        progress: 0.7
                    )
                }
            }
            .buttonStyle(.bordered)

            Button("更新: 即将送达") {
                Task {
                    await manager.update(
                        status: .onTheWay,
                        driverName: "张师傅",
                        distance: 0.3,
                        progress: 0.9
                    )
                }
            }
            .buttonStyle(.bordered)

            Button("已送达，结束追踪") {
                Task {
                    await manager.end()
                }
            }
            .buttonStyle(.bordered)
            .tint(.green)
        }
        .padding()
    }
}
```

### 7.5 Info.plist 配置

在主 App 的 `Info.plist` 中添加：

```xml
<key>NSSupportsLiveActivities</key>
<true/>
```

> ⚠️ 如果不添加此配置，`Activity.request` 会抛出异常！

---

## 8. 设计最佳实践

### 8.1 信息层级设计

| 层级 | 紧凑形态 | 扩展形态 | 锁屏 |
|------|---------|---------|------|
| 第一优先级 | 状态图标 + 核心数字 | 状态 + 标题 | 餐品名 + 状态 |
| 第二优先级 | 倒计时 | 倒计时 + 详细信息 | 骑手 + 距离 |
| 第三优先级 | — | 进度条 + 操作 | 进度条 |

### 8.2 动画建议

```swift
// 使用 contentTransition 实现平滑更新
Text(context.state.status.rawValue)
    .font(.headline)
    .contentTransition(.numericText())
```

| 动画类型 | 适用场景 | 代码 |
|---------|---------|------|
| `.numericText()` | 数字变化（倒计时、距离） | `Text(value).contentTransition(.numericText())` |
| `.identity` | 图标切换 | `Image(icon).contentTransition(.identity)` |
| 默认 | 文字替换 | 系统自动处理 |

### 8.3 颜色使用

| 原则 | 说明 | 示例 |
|------|------|------|
| 主色调统一 | 使用 App 品牌色作为强调色 | 外卖 App 用橙色 |
| 背景透明 | 灵动岛视图使用透明背景 | 系统自动处理 |
| 对比度足够 | 确保在深色背景下可读 | 使用 `.foregroundColor(.white)` |
| 语义化颜色 | 不同状态用不同颜色 | 绿色=完成，橙色=进行中 |

### 8.4 常见错误

| 错误 | 正确做法 |
|------|---------|
| 紧凑形态放太多文字 | 最多 2-3 个字符 + 图标 |
| 扩展形态放可滚动列表 | 扩展形态不支持滚动，内容要精简 |
| 忘记设置 staleDate | 始终设置 staleDate，防止过时信息 |
| 频繁更新（< 1秒/次） | 更新间隔至少 1-2 秒 |
| 不处理 dismissed 状态 | 监听状态变化，及时清理资源 |

---

## 9. 审核注意事项

### 9.1 Apple 审核要求

| 要求 | 说明 |
|------|------|
| 必须有明确结束 | Live Activity 必须在合理时间内结束（最长 12 小时） |
| 信息必须真实 | 展示的信息必须反映 App 的实际功能 |
| 不能做广告 | 不能用 Live Activity 展示广告或促销信息 |
| 用户可关闭 | 用户必须能手动关闭 Live Activity |
| 需要声明 | Info.plist 中 `NSSupportsLiveActivities` 必须为 `true` |

### 9.2 常见被拒原因

| 被拒原因 | 解决方案 |
|---------|---------|
| Live Activity 无法启动 | 检查 Info.plist 配置、权限检查 |
| 内容不更新 | 确认 `activity.update` 被正确调用 |
| 灵动岛显示异常 | 检查四种形态是否全部实现 |
| 推送更新不工作 | 检查 pushToken 上报和服务器推送格式 |
| 内存超限 | 灵动岛视图内存限制约 30MB，避免加载大图 |

### 9.3 测试清单

- [ ] 锁屏 Live Activity 正常显示
- [ ] 灵动岛三种形态（紧凑/最小/扩展）正常显示
- [ ] 更新内容后 UI 实时刷新
- [ ] 结束 Activity 后正确移除
- [ ] 用户手动关闭后不再出现
- [ ] App 杀死后推送更新仍能工作
- [ ] 在不支持灵动岛的设备上锁屏正常显示
- [ ] staleDate 过期后显示正确

---

## 小结

| 知识点 | 核心内容 |
|--------|---------|
| Live Activities | 锁屏 + 灵动岛上的实时信息展示 |
| 灵动岛形态 | 紧凑（双药丸）、最小（单圆）、扩展（大面积） |
| ActivityKit | `ActivityAttributes` 定义数据，`Activity.request` 创建，`activity.update` 更新，`activity.end` 结束 |
| Widget Extension | 通过 `ActivityConfiguration` 实现灵动岛 UI |
| 推送更新 | App 不在前台时，通过推送通知更新 Live Activity |
| 设计原则 | 信息精简、层级分明、颜色克制、动画流畅 |
| 审核要求 | 必须有明确结束、信息真实、不能做广告 |

> 💡 Live Activities 是提升用户体验的利器，但要用在"刀刃"上——只在用户真正需要实时信息的场景下使用。一个好的 Live Activity 应该让用户**少打开 App，多享受生活**。
