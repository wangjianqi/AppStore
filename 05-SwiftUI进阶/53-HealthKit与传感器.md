# 53-HealthKit 与传感器

## 本章目标

- ✅ 理解 HealthKit 框架的定位、适用 App 类型与 Apple 审核要求
- ✅ 掌握权限请求流程：Info.plist 配置、HKHealthStore、读写权限分开请求、Entitlements 配置
- ✅ 学会使用 HKSampleQuery 与 HKStatisticsQuery 读取步数、心率、睡眠、锻炼等健康数据
- ✅ 掌握 HKQuantitySample 的写入、批量保存与数据删除操作
- ✅ 了解 CoreMotion 框架：CMPedometer 计步器、CMMotionActivity 活动识别、CMAccelerometer 加速度计
- ✅ 实现 HKObserverQuery 实时数据监听与后台交付
- ✅ 通过 SwiftUI 构建健康数据仪表盘，集成图表展示与异步数据加载
- ✅ 掌握健康类 App 的审核指南 5.2.1、隐私政策要求与数据用途声明

---

## 1. HealthKit 概述

### 1.1 什么是 HealthKit

HealthKit 是 Apple 提供的健康数据中央管理平台，它就像一个"健康数据银行"——各个 App 可以向这个银行存入或读取健康数据，而用户则是这个银行的唯一管理员，决定谁能访问哪些数据。

> 💡 **生活类比**：HealthKit 就像一家医院的总病历系统——心内科记录心率、呼吸科记录血氧、运动医学科记录步数，各科室共享同一份病历，但每个科室只能看到患者授权的那部分数据。

```
┌──────────────────────────────────────────────────┐
│                  HealthKit 架构                   │
├────────────┬────────────┬────────────────────────┤
│  数据来源    │  健康数据存储  │      数据消费方        │
│            │            │                      │
│ Apple Watch│            │  健康管理 App          │
│ iPhone 传感器│ HKHealth  │  运动追踪 App          │
│ 第三方设备   │ Store     │  医疗研究 App          │
│ 手动录入    │            │  保险/福利 App         │
├────────────┴────────────┴────────────────────────┤
│              用户授权与隐私保护层                     │
│         （每个 App 独立授权，可随时撤销）              │
└──────────────────────────────────────────────────┘
```

### 1.2 适用 App 类型

| App 类型 | 典型场景 | 可用数据类型 |
|----------|----------|-------------|
| **健身运动** | 跑步追踪、瑜伽教练 | 步数、距离、心率、锻炼记录、能量消耗 |
| **健康监测** | 血压记录、血糖管理 | 血压、血糖、体温、血氧 |
| **睡眠分析** | 睡眠质量追踪 | 睡眠阶段、入睡时间、起床时间 |
| **营养饮食** | 卡路里计算、饮食日记 | 膳食能量、营养素摄入、水分摄入 |
| **医疗研究** | 临床数据采集 | 几乎所有数据类型（需额外资质） |
| **心理健康** | 正念冥想 | 正念时长、心率变异性 |

### 1.3 Apple 审核基本要求

> ⚠️ **重要**：HealthKit 是 Apple 严格管控的框架，不是所有 App 都能使用。审核团队会逐一审查你的 App 是否真正需要健康数据。

- App 必须提供与健康/健身直接相关的核心功能
- 纯粹为了营销或广告目的收集健康数据将被直接拒绝
- 必须在 App 内提供隐私政策
- 必须向用户清晰说明数据用途
- 使用 HealthKit 数据的 App 不得将数据出售给第三方

---

## 2. 权限请求与配置

### 2.1 Entitlements 配置

使用 HealthKit 的第一步，是在 Xcode 中启用 HealthKit 能力：

1. 打开 Xcode → Target → Signing & Capabilities
2. 点击 "+ Capability" → 添加 **HealthKit**
3. Xcode 会自动在 `.entitlements` 文件中生成配置

```xml
<!-- YourApp.entitlements -->
<key>com.apple.developer.healthkit</key>
<true/>
<key>com.apple.developer.healthkit.background-delivery</key>
<true/>
```

> 💡 **提示**：`background-delivery` 选项允许 App 在后台接收健康数据变化通知。如果你的 App 只在前台读取数据，可以不勾选此项。

### 2.2 Info.plist 隐私描述

HealthKit 需要两个隐私描述键，缺一不可：

```xml
<key>NSHealthShareUsageDescription</key>
<string>我们需要读取您的步数和心率数据，以便展示每日运动统计</string>
<key>NSHealthUpdateUsageDescription</key>
<string>我们需要将您的运动记录写入健康 App，以便统一管理健康数据</string>
```

| Info.plist 键 | 用途 | 示例描述 |
|---------------|------|----------|
| `NSHealthShareUsageDescription` | 请求读取健康数据的理由 | "读取步数和心率以展示运动统计" |
| `NSHealthUpdateUsageDescription` | 请求写入健康数据的理由 | "将运动记录写入健康 App 统一管理" |

> ⚠️ **警告**：描述文字必须具体说明用途，不能写"需要访问健康数据"这种笼统描述，否则审核会被拒。

### 2.3 HKHealthStore 与权限请求

`HKHealthStore` 是 HealthKit 的核心入口，所有数据操作都通过它完成。权限请求有一个重要原则：**读写权限必须分开请求**。

> 💡 **生活类比**：请求健康数据权限就像去医院——你不能一次性要求看所有科室的病历，而是先看心内科（读心率），再开处方（写用药记录），逐步获取授权。

```swift
import HealthKit

@MainActor
class HealthKitManager: ObservableObject {
    private let healthStore = HKHealthStore()
    @Published var isAuthorized = false
    @Published var authorizationStatus: HKAuthorizationStatus = .notDetermined

    // 定义需要读取的数据类型
    private let readTypes: Set<HKObjectType> = [
        HKObjectType.quantityType(forIdentifier: .stepCount)!,
        HKObjectType.quantityType(forIdentifier: .heartRate)!,
        HKObjectType.categoryType(forIdentifier: .sleepAnalysis)!,
        HKObjectType.workoutType()
    ]

    // 定义需要写入的数据类型
    private let shareTypes: Set<HKSampleType> = [
        HKSampleType.quantityType(forIdentifier: .stepCount)!,
        HKSampleType.workoutType()
    ]

    // 第一步：请求读取权限
    func requestReadPermission() async throws {
        guard HKHealthStore.isHealthDataAvailable() else {
            throw HealthKitError.notAvailable
        }

        try await healthStore.requestAuthorization(toShare: [], read: readTypes)
    }

    // 第二步：请求写入权限
    func requestWritePermission() async throws {
        try await healthStore.requestAuthorization(toShare: shareTypes, read: [])
    }

    // 一次性请求所有权限（不推荐，仅作演示）
    func requestAllPermissions() async throws {
        guard HKHealthStore.isHealthDataAvailable() else {
            throw HealthKitError.notAvailable
        }
        try await healthStore.requestAuthorization(toShare: shareTypes, read: readTypes)
        isAuthorized = true
    }

    func checkAuthorizationStatus(for type: HKObjectType) -> HKAuthorizationStatus {
        healthStore.authorizationStatus(for: type)
    }
}

enum HealthKitError: LocalizedError {
    case notAvailable
    case authorizationDenied
    case queryFailed(Error)

    var errorDescription: String? {
        switch self {
        case .notAvailable: return "此设备不支持 HealthKit"
        case .authorizationDenied: return "用户拒绝了健康数据访问权限"
        case .queryFailed(let error): return "查询失败: \(error.localizedDescription)"
        }
    }
}
```

### 2.4 权限请求最佳实践

| 实践 | 说明 | 原因 |
|------|------|------|
| **分开请求读写** | 先请求读取，用户使用后再请求写入 | 减少首次授权的心理负担 |
| **按需请求** | 只在用户进入相关功能页面时请求 | 避免一启动就弹一堆权限弹窗 |
| **解释后请求** | 先展示说明页，再触发系统授权弹窗 | 提高用户授权率 |
| **优雅降级** | 权限被拒后仍提供基础功能 | 避免用户因拒绝权限而无法使用 App |
| **不要反复弹** | 用户拒绝后不要立即再次请求 | 尊重用户选择，引导去设置页手动开启 |

```swift
struct HealthPermissionOnboardingView: View {
    @State private var showSystemPrompt = false
    @StateObject private var healthKit = HealthKitManager()

    var body: some View {
        VStack(spacing: 24) {
            Image(systemName: "heart.text.square.fill")
                .font(.system(size: 64))
                .foregroundStyle(.pink)

            Text("连接健康数据")
                .font(.title.bold())

            Text("读取您的步数和心率，帮助您更好地了解每日运动情况。您的数据仅存储在设备上，不会上传至任何服务器。")
                .multilineTextAlignment(.center)
                .foregroundStyle(.secondary)

            Button("授权读取健康数据") {
                Task {
                    try? await healthKit.requestReadPermission()
                }
            }
            .buttonStyle(.borderedProminent)
            .tint(.pink)

            NavigationLink("暂不开启，稍后设置") {
                MainDashboardView()
            }
            .foregroundStyle(.secondary)
        }
        .padding(32)
    }
}
```

---

## 3. 读取健康数据

### 3.1 查询类型概览

HealthKit 提供多种查询方式，适用于不同场景：

| 查询类型 | 核心类 | 适用场景 |
|----------|--------|----------|
| **样本查询** | `HKSampleQuery` | 获取原始数据点（如最近 10 条心率记录） |
| **统计查询** | `HKStatisticsQuery` | 聚合计算（如今日总步数、平均心率） |
| **观察者查询** | `HKObserverQuery` | 监听数据变化（如新步数写入时触发） |
| **锚点查询** | `HKAnchoredObjectQuery` | 增量获取新增/删除的数据 |

### 3.2 HKSampleQuery — 查询步数

```swift
extension HealthKitManager {
    func fetchStepCount(for date: Date = Date()) async throws -> [HKQuantitySample] {
        let stepType = HKQuantityType.quantityType(forIdentifier: .stepCount)!
        let (start, end) = dateRange(for: date)

        let predicate = HKQuery.predicateForSamples(
            withStart: start,
            end: end,
            options: .strictStartDate
        )

        let sortDescriptor = NSSortDescriptor(
            key: HKSampleSortIdentifierStartDate,
            ascending: false
        )

        let query = HKSampleQuery(
            sampleType: stepType,
            predicate: predicate,
            limit: HKObjectQueryNoLimit,
            sortDescriptors: [sortDescriptor],
            resultsHandler: { _, samples, error in
            }
        )

        return try await withCheckedThrowingContinuation { continuation in
            let query = HKSampleQuery(
                sampleType: stepType,
                predicate: predicate,
                limit: HKObjectQueryNoLimit,
                sortDescriptors: [sortDescriptor]
            ) { _, samples, error in
                if let error {
                    continuation.resume(throwing: HealthKitError.queryFailed(error))
                } else {
                    continuation.resume(returning: (samples as? [HKQuantitySample]) ?? [])
                }
            }
            healthStore.execute(query)
        }
    }

    private func dateRange(for date: Date) -> (Date, Date) {
        let calendar = Calendar.current
        let start = calendar.startOfDay(for: date)
        let end = calendar.date(byAdding: .day, value: 1, to: start)!
        return (start, end)
    }
}
```

### 3.3 HKStatisticsQuery — 统计查询

统计查询就像"数据汇总表"——不需要每条原始记录，只需要一个汇总数字：

```swift
extension HealthKitManager {
    func fetchTodayStepStats() async throws -> Double {
        let stepType = HKQuantityType.quantityType(forIdentifier: .stepCount)!
        let (start, end) = dateRange(for: Date())

        let predicate = HKQuery.predicateForSamples(
            withStart: start,
            end: end,
            options: .strictStartDate
        )

        return try await withCheckedThrowingContinuation { continuation in
            let query = HKStatisticsQuery(
                quantityType: stepType,
                quantitySamplePredicate: predicate,
                options: .cumulativeSum
            ) { _, statistics, error in
                if let error {
                    continuation.resume(throwing: HealthKitError.queryFailed(error))
                } else {
                    let steps = statistics?.sumQuantity()?.doubleValue(for: .count()) ?? 0
                    continuation.resume(returning: steps)
                }
            }
            healthStore.execute(query)
        }
    }

    func fetchAverageHeartRate(for date: Date = Date()) async throws -> Double {
        let heartRateType = HKQuantityType.quantityType(forIdentifier: .heartRate)!
        let (start, end) = dateRange(for: date)

        let predicate = HKQuery.predicateForSamples(
            withStart: start,
            end: end,
            options: .strictStartDate
        )

        return try await withCheckedThrowingContinuation { continuation in
            let query = HKStatisticsQuery(
                quantityType: heartRateType,
                quantitySamplePredicate: predicate,
                options: .discreteAverage
            ) { _, statistics, error in
                if let error {
                    continuation.resume(throwing: HealthKitError.queryFailed(error))
                } else {
                    let bpm = statistics?.averageQuantity()?.doubleValue(
                        for: HKUnit.count().unitDivided(by: .minute())
                    ) ?? 0
                    continuation.resume(returning: bpm)
                }
            }
            healthStore.execute(query)
        }
    }
}
```

### 3.4 查询睡眠数据

```swift
extension HealthKitManager {
    func fetchSleepAnalysis(for date: Date = Date()) async throws -> [HKCategorySample] {
        let sleepType = HKCategoryType.categoryType(forIdentifier: .sleepAnalysis)!
        let (start, end) = dateRange(for: date)

        let predicate = HKQuery.predicateForSamples(
            withStart: start,
            end: end,
            options: .strictStartDate
        )

        let sortDescriptor = NSSortDescriptor(
            key: HKSampleSortIdentifierStartDate,
            ascending: false
        )

        return try await withCheckedThrowingContinuation { continuation in
            let query = HKSampleQuery(
                sampleType: sleepType,
                predicate: predicate,
                limit: HKObjectQueryNoLimit,
                sortDescriptors: [sortDescriptor]
            ) { _, samples, error in
                if let error {
                    continuation.resume(throwing: HealthKitError.queryFailed(error))
                } else {
                    continuation.resume(returning: (samples as? [HKCategorySample]) ?? [])
                }
            }
            healthStore.execute(query)
        }
    }

    func calculateTotalSleep(from samples: [HKCategorySample]) -> TimeInterval {
        var totalSeconds: TimeInterval = 0
        for sample in samples {
            if sample.value == HKCategoryValueSleepAnalysis.asleepDeep.rawValue
                || sample.value == HKCategoryValueSleepAnalysis.asleepCore.rawValue
                || sample.value == HKCategoryValueSleepAnalysis.asleepREM.rawValue
                || sample.value == HKCategoryValueSleepAnalysis.asleepUnspecified.rawValue {
                totalSeconds += sample.endDate.timeIntervalSince(sample.startDate)
            }
        }
        return totalSeconds
    }
}
```

### 3.5 查询锻炼数据

```swift
extension HealthKitManager {
    func fetchWorkouts(for date: Date = Date()) async throws -> [HKWorkout] {
        let workoutType = HKObjectType.workoutType()
        let (start, end) = dateRange(for: date)

        let predicate = HKQuery.predicateForSamples(
            withStart: start,
            end: end,
            options: .strictStartDate
        )

        let sortDescriptor = NSSortDescriptor(
            key: HKSampleSortIdentifierStartDate,
            ascending: false
        )

        return try await withCheckedThrowingContinuation { continuation in
            let query = HKSampleQuery(
                sampleType: workoutType,
                predicate: predicate,
                limit: HKObjectQueryNoLimit,
                sortDescriptors: [sortDescriptor]
            ) { _, samples, error in
                if let error {
                    continuation.resume(throwing: HealthKitError.queryFailed(error))
                } else {
                    continuation.resume(returning: (samples as? [HKWorkout]) ?? [])
                }
            }
            healthStore.execute(query)
        }
    }
}
```

### 3.6 常用 HKUnit 速查表

| 数据类型 | HKQuantityTypeIdentifier | 单位 (HKUnit) |
|----------|--------------------------|---------------|
| 步数 | `.stepCount` | `.count()` |
| 心率 | `.heartRate` | `count().unitDivided(by: .minute())` |
| 距离 | `.distanceWalkingRunning` | `.meter()` / `.meterUnit(with: .kilo)` |
| 能量消耗 | `.activeEnergyBurned` | `.kilocalorie()` |
| 体重 | `.bodyMass` | `.gramUnit(with: .kilo)` |
| 身高 | `.height` | `.meterUnit(with: .centi)` |
| 血压（收缩压） | `.bloodPressureSystolic` | `.millimeterOfMercury()` |
| 血氧 | `.oxygenSaturation` | `.percent()` |

---

## 4. 写入健康数据

### 4.1 保存单条数据

写入健康数据就像"往病历本上记一笔"——必须指定数据类型、数值、起止时间：

```swift
extension HealthKitManager {
    func saveStepCount(steps: Double, start: Date, end: Date) async throws {
        let stepType = HKQuantityType.quantityType(forIdentifier: .stepCount)!
        let quantity = HKQuantity(unit: .count(), doubleValue: steps)

        let sample = HKQuantitySample(
            type: stepType,
            quantity: quantity,
            start: start,
            end: end
        )

        try await healthStore.save(sample)
    }

    func saveHeartRate(bpm: Double, timestamp: Date = Date()) async throws {
        let heartRateType = HKQuantityType.quantityType(forIdentifier: .heartRate)!
        let quantity = HKQuantity(
            unit: HKUnit.count().unitDivided(by: .minute()),
            doubleValue: bpm
        )

        let sample = HKQuantitySample(
            type: heartRateType,
            quantity: quantity,
            start: timestamp,
            end: timestamp
        )

        try await healthStore.save(sample)
    }

    func saveWorkout(type: HKWorkoutActivityType,
                     start: Date,
                     end: Date,
                     calories: Double,
                     distance: Double?) async throws {
        let energyType = HKQuantityType.quantityType(forIdentifier: .activeEnergyBurned)!
        let energyQuantity = HKQuantity(unit: .kilocalorie(), doubleValue: calories)

        var distanceQuantity: HKQuantity?
        if let distance {
            distanceQuantity = HKQuantity(unit: .meterUnit(with: .kilo), doubleValue: distance)
        }

        let workout = HKWorkout(
            activityType: type,
            start: start,
            end: end,
            duration: end.timeIntervalSince(start),
            totalEnergyBurned: energyQuantity,
            totalDistance: distanceQuantity,
            metadata: [HKMetadataKeyWorkoutBrandName: "MyApp"]
        )

        try await healthStore.save(workout)
    }
}
```

### 4.2 批量保存数据

当需要一次性写入多条数据时（如从外部设备同步），批量保存比逐条保存高效得多：

```swift
extension HealthKitManager {
    func saveBatchHeartRates(records: [(bpm: Double, time: Date)]) async throws {
        let heartRateType = HKQuantityType.quantityType(forIdentifier: .heartRate)!

        let samples = records.map { record in
            HKQuantitySample(
                type: heartRateType,
                quantity: HKQuantity(
                    unit: HKUnit.count().unitDivided(by: .minute()),
                    doubleValue: record.bpm
                ),
                start: record.time,
                end: record.time
            )
        }

        try await healthStore.save(samples)
    }
}
```

### 4.3 删除数据

只能删除由你的 App 创建的数据，不能删除其他来源的数据：

```swift
extension HealthKitManager {
    func deleteSteps(from start: Date, to end: Date) async throws {
        let stepType = HKQuantityType.quantityType(forIdentifier: .stepCount)!
        let predicate = HKQuery.predicateForSamples(
            withStart: start,
            end: end,
            options: .strictStartDate
        )

        let objects = try await withCheckedThrowingContinuation { continuation in
            let query = HKSampleQuery(
                sampleType: stepType,
                predicate: predicate,
                limit: HKObjectQueryNoLimit,
                sortDescriptors: nil
            ) { _, samples, error in
                if let error {
                    continuation.resume(throwing: error)
                } else {
                    continuation.resume(returning: samples ?? [])
                }
            }
            healthStore.execute(query)
        }

        for object in objects {
            try await healthStore.delete(object)
        }
    }

    func deleteAllMySteps() async throws {
        let stepType = HKQuantityType.quantityType(forIdentifier: .stepCount)!
        let predicate = HKQuery.predicateForObjects(from: HKSource.default())
        try await healthStore.deleteObjects(of: stepType, predicate: predicate)
    }
}
```

> ⚠️ **警告**：删除操作不可逆！务必在 UI 中提供确认弹窗，避免误删。同时注意，`deleteObjects(of:predicate:)` 方法只能删除当前 App 创建的数据。

---

## 5. CoreMotion 框架

### 5.1 CoreMotion vs HealthKit 对比

CoreMotion 提供设备传感器的实时数据，而 HealthKit 提供持久化的健康数据存储。两者互补而非替代：

| 维度 | CoreMotion | HealthKit |
|------|-----------|-----------|
| **数据时效** | 实时传感器数据 | 历史持久化数据 |
| **数据来源** | 设备内置传感器 | Apple Watch + 第三方设备 + 手动录入 |
| **需要配对** | 不需要 | 需要（部分数据需 Apple Watch） |
| **后台运行** | 需配置 Background Modes | 支持 Background Delivery |
| **典型场景** | 实时计步、动作识别、摇一摇 | 每日步数统计、心率趋势、睡眠分析 |
| **权限** | `NSMotionUsageDescription` | `NSHealthShareUsageDescription` 等 |

### 5.2 CMPedometer — 计步器

CMPedometer 就像一个"随身计步器"——它能告诉你走了多少步、走了多远，甚至上下楼层数：

```swift
import CoreMotion

@MainActor
class PedometerManager: ObservableObject {
    private let pedometer = CMPedometer()
    @Published var stepCount: Int = 0
    @Published var distance: Double = 0
    @Published var floorsAscended: Int = 0
    @Published var isPedometerAvailable = false
    @Published var isTracking = false

    init() {
        isPedometerAvailable = CMPedometer.isStepCountingAvailable()
    }

    func startRealtimeUpdates() {
        guard CMPedometer.isStepCountingAvailable() else { return }

        isTracking = true
        pedometer.startUpdates(from: Calendar.current.startOfDay(for: Date())) { [weak self] data, error in
            guard let data, error == nil else { return }
            Task { @MainActor in
                self?.stepCount = data.numberOfSteps.intValue
                self?.distance = data.distance?.doubleValue ?? 0
                self?.floorsAscended = data.floorsAscended?.intValue ?? 0
            }
        }
    }

    func stopUpdates() {
        pedometer.stopUpdates()
        isTracking = false
    }

    func fetchHistoricalSteps(for date: Date) async throws -> (steps: Int, distance: Double) {
        try await withCheckedThrowingContinuation { continuation in
            let start = Calendar.current.startOfDay(for: date)
            let end = Calendar.current.date(byAdding: .day, value: 1, to: start)!

            pedometer.queryPedometerData(from: start, to: end) { data, error in
                if let error {
                    continuation.resume(throwing: error)
                } else {
                    let steps = data?.numberOfSteps.intValue ?? 0
                    let dist = data?.distance?.doubleValue ?? 0
                    continuation.resume(returning: (steps, dist))
                }
            }
        }
    }
}
```

### 5.3 CMMotionActivity — 活动识别

CMMotionActivity 能识别用户当前的运动状态——是在走路、跑步、骑车还是开车：

```swift
@MainActor
class ActivityManager: ObservableObject {
    private let motionActivityManager = CMMotionActivityManager()
    @Published var currentActivity: String = "未知"
    @Published var confidence: String = "低"

    func startActivityUpdates() {
        guard CMMotionActivityManager.isActivityAvailable() else { return }

        motionActivityManager.startActivityUpdates(to: .main) { [weak self] activity in
            guard let activity else { return }
            Task { @MainActor in
                if activity.walking {
                    self?.currentActivity = "步行"
                } else if activity.running {
                    self?.currentActivity = "跑步"
                } else if activity.cycling {
                    self?.currentActivity = "骑行"
                } else if activity.automotive {
                    self?.currentActivity = "驾车"
                } else if activity.stationary {
                    self?.currentActivity = "静止"
                } else {
                    self?.currentActivity = "未知"
                }

                switch activity.confidence {
                case .high: self?.confidence = "高"
                case .medium: self?.confidence = "中"
                default: self?.confidence = "低"
                }
            }
        }
    }

    func stopActivityUpdates() {
        motionActivityManager.stopActivityUpdates()
    }
}
```

> 💡 **提示**：活动识别在模拟器上不可用，必须在真机上测试。且需要用户授权运动与健身权限（`NSMotionUsageDescription`）。

### 5.4 CMAccelerometer — 加速度计

加速度计能感知设备的运动和倾斜，常用于摇一摇、体感游戏等场景：

```swift
@MainActor
class AccelerometerManager: ObservableObject {
    private let motionManager = CMMotionManager()
    @Published var accelerationX: Double = 0
    @Published var accelerationY: Double = 0
    @Published var accelerationZ: Double = 0
    @Published var isShaking = false

    func startAccelerometerUpdates() {
        guard motionManager.isAccelerometerAvailable else { return }

        motionManager.accelerometerUpdateInterval = 1.0 / 30.0
        motionManager.startAccelerometerUpdates(to: .main) { [weak self] data, error in
            guard let data, error == nil else { return }
            let acc = data.acceleration
            Task { @MainActor in
                self?.accelerationX = acc.x
                self?.accelerationY = acc.y
                self?.accelerationZ = acc.z

                let magnitude = sqrt(acc.x * acc.x + acc.y * acc.y + acc.z * acc.z)
                self?.isShaking = magnitude > 2.5
            }
        }
    }

    func stopAccelerometerUpdates() {
        motionManager.stopAccelerometerUpdates()
    }
}
```

---

## 6. 实时数据监听

### 6.1 HKObserverQuery — 观察者查询

HKObserverQuery 就像给健康数据装了一个"门铃"——每当有新数据写入时，系统就会"按铃"通知你的 App：

```swift
extension HealthKitManager {
    func observeStepCountChanges() {
        let stepType = HKObjectType.quantityType(forIdentifier: .stepCount)!

        let query = HKObserverQuery(sampleType: stepType, predicate: nil) { [weak self] _, completionHandler, error in
            if error == nil {
                Task { @MainActor in
                    self?.handleStepCountUpdate()
                }
            }
            completionHandler()
        }

        healthStore.execute(query)
    }

    private func handleStepCountUpdate() {
        Task {
            do {
                let steps = try await fetchTodayStepStats()
                print("步数更新: \(steps)")
            } catch {
                print("获取更新步数失败: \(error)")
            }
        }
    }

    func stopObserving(stepType: HKSampleType) {
        let query = HKObserverQuery(sampleType: stepType, predicate: nil) { _, _, _ in }
        healthStore.stop(query)
    }
}
```

### 6.2 启用后台交付

后台交付让你的 App 即使不在前台，也能在健康数据变化时被唤醒：

```swift
extension HealthKitManager {
    func enableBackgroundDelivery() async throws {
        let stepType = HKObjectType.quantityType(forIdentifier: .stepCount)!
        let heartRateType = HKObjectType.quantityType(forIdentifier: .heartRate)!

        try await healthStore.enableBackgroundDelivery(
            for: stepType,
            frequency: .immediate
        )

        try await healthStore.enableBackgroundDelivery(
            for: heartRateType,
            frequency: .hourly
        )
    }

    func disableBackgroundDelivery() async throws {
        let stepType = HKObjectType.quantityType(forIdentifier: .stepCount)!
        let heartRateType = HKObjectType.quantityType(forIdentifier: .heartRate)!

        try await healthStore.disableBackgroundDelivery(for: stepType)
        try await healthStore.disableBackgroundDelivery(for: heartRateType)
    }
}
```

### 6.3 后台交付频率选项

| 频率选项 | 说明 | 适用场景 |
|----------|------|----------|
| `.immediate` | 数据变化后立即通知 | 紧急健康指标（心率异常） |
| `.hourly` | 每小时最多通知一次 | 常规运动数据（步数、锻炼） |
| `.daily` | 每天最多通知一次 | 长期趋势数据（体重、睡眠） |
| `.weekly` | 每周最多通知一次 | 周报汇总类数据 |

> ⚠️ **警告**：后台交付需要 `.entitlements` 中配置 `com.apple.developer.healthkit.background-delivery`，且 Apple 审核时会严格审查你是否真正需要后台更新。过度使用 `.immediate` 频率可能导致审核被拒。

---

## 7. SwiftUI 集成实战

### 7.1 健康数据 ViewModel

将前面学到的所有查询整合到一个 ViewModel 中，为 SwiftUI 视图提供数据：

```swift
import SwiftUI
import HealthKit

@MainActor
class HealthDashboardViewModel: ObservableObject {
    private let healthStore = HKHealthStore()
    @Published var todaySteps: Double = 0
    @Published var todayCalories: Double = 0
    @Published var averageHeartRate: Double = 0
    @Published var sleepHours: Double = 0
    @Published var weeklySteps: [Double] = Array(repeating: 0, count: 7)
    @Published var isLoading = false
    @Published var errorMessage: String?

    func requestPermissionAndLoad() async {
        guard HKHealthStore.isHealthDataAvailable() else {
            errorMessage = "此设备不支持 HealthKit"
            return
        }

        do {
            let readTypes: Set<HKObjectType> = [
                HKObjectType.quantityType(forIdentifier: .stepCount)!,
                HKObjectType.quantityType(forIdentifier: .heartRate)!,
                HKObjectType.quantityType(forIdentifier: .activeEnergyBurned)!,
                HKObjectType.categoryType(forIdentifier: .sleepAnalysis)!
            ]
            try await healthStore.requestAuthorization(toShare: [], read: readTypes)
            await loadAllData()
        } catch {
            errorMessage = error.localizedDescription
        }
    }

    func loadAllData() async {
        isLoading = true
        defer { isLoading = false }

        async let steps = fetchTodaySteps()
        async let calories = fetchTodayCalories()
        async let heartRate = fetchAverageHeartRate()
        async let sleep = fetchTodaySleep()
        async let weekly = fetchWeeklySteps()

        do {
            let (s, c, h, sl, w) = try await (steps, calories, heartRate, sleep, weekly)
            todaySteps = s
            todayCalories = c
            averageHeartRate = h
            sleepHours = sl
            weeklySteps = w
        } catch {
            errorMessage = error.localizedDescription
        }
    }

    private func fetchTodaySteps() async throws -> Double {
        let stepType = HKQuantityType.quantityType(forIdentifier: .stepCount)!
        let (start, end) = dayRange(for: Date())
        return try await fetchStatistics(for: stepType, start: start, end: end, option: .cumulativeSum) {
            $0.sumQuantity()?.doubleValue(for: .count()) ?? 0
        }
    }

    private func fetchTodayCalories() async throws -> Double {
        let calorieType = HKQuantityType.quantityType(forIdentifier: .activeEnergyBurned)!
        let (start, end) = dayRange(for: Date())
        return try await fetchStatistics(for: calorieType, start: start, end: end, option: .cumulativeSum) {
            $0.sumQuantity()?.doubleValue(for: .kilocalorie()) ?? 0
        }
    }

    private func fetchAverageHeartRate() async throws -> Double {
        let heartRateType = HKQuantityType.quantityType(forIdentifier: .heartRate)!
        let (start, end) = dayRange(for: Date())
        return try await fetchStatistics(for: heartRateType, start: start, end: end, option: .discreteAverage) {
            $0.averageQuantity()?.doubleValue(for: HKUnit.count().unitDivided(by: .minute())) ?? 0
        }
    }

    private func fetchStatistics<T>(for type: HKQuantityType,
                                     start: Date,
                                     end: Date,
                                     option: HKStatisticsOptions,
                                     transform: (HKStatistics) -> T) async throws -> T {
        let predicate = HKQuery.predicateForSamples(withStart: start, end: end, options: .strictStartDate)
        return try await withCheckedThrowingContinuation { continuation in
            let query = HKStatisticsQuery(
                quantityType: type,
                quantitySamplePredicate: predicate,
                options: option
            ) { _, statistics, error in
                if let error {
                    continuation.resume(throwing: error)
                } else if let statistics {
                    continuation.resume(returning: transform(statistics))
                } else {
                    continuation.resume(throwing: HealthKitError.queryFailed(NSError(domain: "HealthKit", code: -1)))
                }
            }
            healthStore.execute(query)
        }
    }

    private func fetchTodaySleep() async throws -> Double {
        let sleepType = HKCategoryType.categoryType(forIdentifier: .sleepAnalysis)!
        let (start, end) = dayRange(for: Date())
        let predicate = HKQuery.predicateForSamples(withStart: start, end: end, options: .strictStartDate)

        return try await withCheckedThrowingContinuation { continuation in
            let query = HKSampleQuery(
                sampleType: sleepType,
                predicate: predicate,
                limit: HKObjectQueryNoLimit,
                sortDescriptors: nil
            ) { _, samples, error in
                if let error {
                    continuation.resume(throwing: error)
                } else {
                    let totalSeconds = (samples as? [HKCategorySample])?.reduce(0.0) { result, sample in
                        let isAsleep = sample.value != HKCategoryValueSleepAnalysis.inBed.rawValue
                        return isAsleep ? result + sample.endDate.timeIntervalSince(sample.startDate) : result
                    } ?? 0
                    continuation.resume(returning: totalSeconds / 3600)
                }
            }
            healthStore.execute(query)
        }
    }

    private func fetchWeeklySteps() async throws -> [Double] {
        let calendar = Calendar.current
        let today = calendar.startOfDay(for: Date())
        var result: [Double] = []

        for i in (0..<7).reversed() {
            guard let date = calendar.date(byAdding: .day, value: -i, to: today) else { continue }
            let (start, end) = dayRange(for: date)
            let steps = try await fetchTodayStepsRange(start: start, end: end)
            result.append(steps)
        }
        return result
    }

    private func fetchTodayStepsRange(start: Date, end: Date) async throws -> Double {
        let stepType = HKQuantityType.quantityType(forIdentifier: .stepCount)!
        return try await fetchStatistics(for: stepType, start: start, end: end, option: .cumulativeSum) {
            $0.sumQuantity()?.doubleValue(for: .count()) ?? 0
        }
    }

    private func dayRange(for date: Date) -> (Date, Date) {
        let start = Calendar.current.startOfDay(for: date)
        let end = Calendar.current.date(byAdding: .day, value: 1, to: start)!
        return (start, end)
    }
}
```

### 7.2 健康数据仪表盘视图

```swift
struct HealthDashboardView: View {
    @StateObject private var viewModel = HealthDashboardViewModel()

    var body: some View {
        ScrollView {
            VStack(spacing: 20) {
                if viewModel.isLoading {
                    ProgressView("加载健康数据...")
                        .padding(.top, 60)
                } else {
                    todaySummaryGrid
                    weeklyStepChart
                    sleepAndHeartRateRow
                }
            }
            .padding()
        }
        .navigationTitle("健康仪表盘")
        .task {
            await viewModel.requestPermissionAndLoad()
        }
        .refreshable {
            await viewModel.loadAllData()
        }
    }

    private var todaySummaryGrid: some View {
        LazyVGrid(columns: [
            GridItem(.flexible()),
            GridItem(.flexible())
        ], spacing: 16) {
            HealthMetricCard(
                icon: "figure.walk",
                title: "今日步数",
                value: "\(Int(viewModel.todaySteps))",
                unit: "步",
                color: .green
            )
            HealthMetricCard(
                icon: "flame.fill",
                title: "消耗热量",
                value: String(format: "%.0f", viewModel.todayCalories),
                unit: "千卡",
                color: .orange
            )
            HealthMetricCard(
                icon: "heart.fill",
                title: "平均心率",
                value: String(format: "%.0f", viewModel.averageHeartRate),
                unit: "bpm",
                color: .red
            )
            HealthMetricCard(
                icon: "bed.double.fill",
                title: "睡眠时长",
                value: String(format: "%.1f", viewModel.sleepHours),
                unit: "小时",
                color: .indigo
            )
        }
    }

    private var weeklyStepChart: some View {
        VStack(alignment: .leading, spacing: 12) {
            Text("本周步数趋势")
                .font(.headline)

            ChartView(data: viewModel.weeklySteps)
                .frame(height: 180)
        }
        .padding()
        .background(Color(.systemBackground))
        .clipShape(RoundedRectangle(cornerRadius: 16))
        .shadow(color: .black.opacity(0.05), radius: 8, y: 4)
    }

    private var sleepAndHeartRateRow: some View {
        HStack(spacing: 16) {
            MiniMetricView(
                icon: "heart.fill",
                value: String(format: "%.0f", viewModel.averageHeartRate),
                unit: "bpm",
                color: .red
            )
            MiniMetricView(
                icon: "bed.double.fill",
                value: String(format: "%.1f", viewModel.sleepHours),
                unit: "h",
                color: .indigo
            )
        }
    }
}
```

### 7.3 健康指标卡片组件

```swift
struct HealthMetricCard: View {
    let icon: String
    let title: String
    let value: String
    let unit: String
    let color: Color

    var body: some View {
        VStack(alignment: .leading, spacing: 8) {
            HStack {
                Image(systemName: icon)
                    .foregroundStyle(color)
                    .font(.title3)
                Spacer()
            }

            Text(title)
                .font(.caption)
                .foregroundStyle(.secondary)

            HStack(alignment: .lastTextBaseline, spacing: 4) {
                Text(value)
                    .font(.title2.bold())
                Text(unit)
                    .font(.caption)
                    .foregroundStyle(.secondary)
            }
        }
        .padding()
        .background(Color(.systemBackground))
        .clipShape(RoundedRectangle(cornerRadius: 16))
        .shadow(color: .black.opacity(0.05), radius: 8, y: 4)
    }
}

struct MiniMetricView: View {
    let icon: String
    let value: String
    let unit: String
    let color: Color

    var body: some View {
        HStack(spacing: 12) {
            Image(systemName: icon)
                .font(.title2)
                .foregroundStyle(color)
                .frame(width: 40, height: 40)
                .background(color.opacity(0.15))
                .clipShape(Circle())

            VStack(alignment: .leading) {
                Text(value)
                    .font(.title3.bold())
                Text(unit)
                    .font(.caption2)
                    .foregroundStyle(.secondary)
            }
            Spacer()
        }
        .padding()
        .background(Color(.systemBackground))
        .clipShape(RoundedRectangle(cornerRadius: 12))
        .shadow(color: .black.opacity(0.05), radius: 4, y: 2)
    }
}
```

### 7.4 简易柱状图组件

```swift
struct ChartView: View {
    let data: [Double]
    private let days = ["一", "二", "三", "四", "五", "六", "日"]

    var body: some View {
        let maxVal = data.max() ?? 1

        HStack(alignment: .bottom, spacing: 8) {
            ForEach(0..<data.count, id: \.self) { index in
                VStack(spacing: 4) {
                    Text("\(Int(data[index]))")
                        .font(.system(size: 9))
                        .foregroundStyle(.secondary)

                    RoundedRectangle(cornerRadius: 4)
                        .fill(
                            index == data.count - 1
                            ? Color.green
                            : Color.green.opacity(0.4)
                        )
                        .frame(height: max(4, CGFloat(data[index] / maxVal) * 120))

                    Text(days[index])
                        .font(.system(size: 10))
                        .foregroundStyle(.secondary)
                }
                .frame(maxWidth: .infinity)
            }
        }
    }
}
```

---

## 8. 审核注意事项

### 8.1 App Store 审核指南 5.2.1

Apple 对健康类 App 的审核极其严格，审核指南 5.2.1 条款专门针对健康数据：

| 要求 | 说明 | 违规后果 |
|------|------|----------|
| **数据用途必须明确** | App 必须在界面中清晰说明健康数据的使用目的 | 直接拒绝 |
| **核心功能依赖健康数据** | App 的主要功能必须与健康数据直接相关 | 直接拒绝 |
| **不得出售数据** | 禁止将健康数据出售给广告商或第三方 | 直接拒绝 + 可能封号 |
| **不得仅用于研究** | 纯研究类 App 应使用 ResearchKit / CareKit | 引导更换框架 |
| **数据最小化原则** | 只请求完成功能所需的最少数据类型 | 请求过多类型可能被拒 |
| **必须提供隐私政策** | App 内必须有可访问的隐私政策页面 | 直接拒绝 |

### 8.2 隐私政策要求

隐私政策必须包含以下内容：

```markdown
# 隐私政策模板（健康数据部分）

## 健康数据收集说明

1. **收集的数据类型**：步数、心率、睡眠数据
2. **收集目的**：用于展示每日运动统计和健康趋势分析
3. **数据存储**：所有健康数据仅存储在您的设备本地，不会上传至任何服务器
4. **数据共享**：我们不会将您的健康数据与任何第三方共享
5. **数据控制**：您可以随时在系统设置中撤销健康数据访问权限
6. **数据保留**：卸载 App 后，我们写入的健康数据仍保留在健康 App 中，
   您可以手动删除
```

### 8.3 数据用途声明最佳实践

在 App 内提供一个专门的"数据使用说明"页面，比在隐私政策中笼统描述更有效：

```swift
struct DataUsageExplanationView: View {
    var body: some View {
        List {
            Section("我们如何使用您的健康数据") {
                DataUsageRow(
                    icon: "figure.walk",
                    color: .green,
                    dataType: "步数",
                    purpose: "展示每日运动目标完成进度"
                )
                DataUsageRow(
                    icon: "heart.fill",
                    color: .red,
                    dataType: "心率",
                    purpose: "分析运动强度，提供个性化建议"
                )
                DataUsageRow(
                    icon: "bed.double.fill",
                    color: .indigo,
                    dataType: "睡眠",
                    purpose: "追踪睡眠质量，提醒作息规律"
                )
            }

            Section {
                VStack(alignment: .leading, spacing: 8) {
                    Label("所有数据仅存储在您的设备上", systemImage: "lock.shield")
                    Label("我们不会将数据上传至任何服务器", systemImage: "icloud.slash")
                    Label("我们不会将数据出售给第三方", systemImage: "hand.raised")
                }
                .font(.subheadline)
                .foregroundStyle(.secondary)
            }
        }
        .navigationTitle("数据使用说明")
    }
}

struct DataUsageRow: View {
    let icon: String
    let color: Color
    let dataType: String
    let purpose: String

    var body: some View {
        HStack(spacing: 12) {
            Image(systemName: icon)
                .font(.title3)
                .foregroundStyle(color)
                .frame(width: 32)

            VStack(alignment: .leading, spacing: 2) {
                Text(dataType)
                    .font(.subheadline.bold())
                Text(purpose)
                    .font(.caption)
                    .foregroundStyle(.secondary)
            }
        }
        .padding(.vertical, 4)
    }
}
```

### 8.4 审核常见被拒原因与对策

| 被拒原因 | 具体表现 | 解决方案 |
|----------|----------|----------|
| **权限描述不具体** | "需要访问健康数据" | 改为"读取步数以展示运动目标进度" |
| **数据类型过多** | 请求了 20+ 种数据类型但只用了 3 种 | 只请求实际使用的数据类型 |
| **缺少隐私政策** | App 内没有隐私政策入口 | 在设置页添加隐私政策链接 |
| **核心功能无关** | 天气 App 请求心率数据 | 移除无关的健康数据请求 |
| **后台交付滥用** | 所有类型都用 `.immediate` 频率 | 根据实际需求选择合理频率 |
| **数据上传未说明** | 将健康数据发送到服务器但未告知用户 | 在隐私政策中明确说明并获取用户同意 |
| **缺少数据删除选项** | 用户无法删除 App 写入的数据 | 提供数据管理页面，支持查看和删除 |

> ⚠️ **关键提醒**：Apple 在 2024 年后加强了对健康数据 App 的审核力度。如果你的 App 不是以健康/健身为核心定位，建议不要集成 HealthKit，改用 CoreMotion 获取设备传感器数据即可。

---

## 本章小结

| 主题 | 核心要点 | 关键 API |
|------|----------|----------|
| **HealthKit 概述** | 健康数据中央管理平台，需审核合规，仅适用于健康/健身类 App | `HKHealthStore` |
| **权限请求与配置** | Entitlements + Info.plist + 分开请求读写权限，描述必须具体 | `requestAuthorization(toShare:read:)` |
| **读取健康数据** | HKSampleQuery 获取原始数据，HKStatisticsQuery 聚合统计，支持步数/心率/睡眠/锻炼 | `HKSampleQuery`, `HKStatisticsQuery` |
| **写入健康数据** | HKQuantitySample 创建样本，批量保存高效，只能删除自己写入的数据 | `HKQuantitySample`, `HKWorkout`, `save()`, `delete()` |
| **CoreMotion** | 实时传感器数据，CMPedometer 计步、CMMotionActivity 识别运动状态、CMAccelerometer 感知加速度 | `CMPedometer`, `CMMotionActivityManager`, `CMMotionManager` |
| **实时数据监听** | HKObserverQuery 监听变化，Background Delivery 后台唤醒，频率选项按需选择 | `HKObserverQuery`, `enableBackgroundDelivery()` |
| **SwiftUI 集成** | ViewModel 封装异步查询，仪表盘卡片布局，下拉刷新，并发加载优化 | `@StateObject`, `async/await`, `Task` |
| **审核注意事项** | 指南 5.2.1 严格审查，隐私政策必备，数据最小化，禁止出售数据 | 审核指南 5.2.1, `NSHealthShareUsageDescription` |

> 🎯 **一句话总结**：HealthKit 是 Apple 健康生态的核心枢纽——理解其权限模型（读写分离、用户可控），掌握查询体系（Sample/Statistics/Observer），结合 CoreMotion 获取实时传感器数据，并在 SwiftUI 中以异步方式优雅地呈现，同时严格遵循审核指南 5.2.1，才能构建出既功能强大又合规的健康类应用。
