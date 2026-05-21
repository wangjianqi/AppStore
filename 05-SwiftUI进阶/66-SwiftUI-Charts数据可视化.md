# 66-SwiftUI Charts 数据可视化

## 本章目标

- 理解 SwiftUI Charts 框架的设计理念与核心架构
- 掌握五种基础图表类型：柱状图、折线图、面积图、散点图、饼图
- 学会构建符合 Identifiable 和 Plottable 协议的数据模型
- 熟练定制图表外观：颜色渐变、图例、标注、标记线
- 实现图表交互：选区、点击响应、动画过渡
- 构建高级图表：堆叠柱状图、范围图、双轴图、可滚动图表
- 完成一个综合数据仪表盘实战项目
- 掌握图表设计的最佳实践与无障碍适配

---

## 1. SwiftUI Charts 概述

想象一下：你走进一家超市，货架上密密麻麻全是数字报表——你肯定头大。但如果把这些数字变成一张张直观的柱状图、折线图，趋势和差异一目了然。**SwiftUI Charts** 就是 Apple 在 iOS 16 给你提供的"数据翻译官"，把枯燥数字变成直观图形。

### 1.1 什么是 SwiftUI Charts

SwiftUI Charts 是 Apple 推出的**原生声明式图表框架**，随 iOS 16 / macOS 13 一同发布。它采用与 SwiftUI 一致的声明式语法，让你用极少的代码就能创建专业级图表。

| 特性 | 说明 |
|------|------|
| 声明式 API | 与 SwiftUI 视图构建方式一致，所见即所得 |
| 原生框架 | 无需第三方依赖，Apple 官方维护 |
| 自动布局 | 图表自动适应容器大小和方向变化 |
| 深色模式 | 自动适配系统外观 |
| 无障碍 | 内置 VoiceOver 支持 |
| 动画支持 | 数据变化时自动产生平滑过渡动画 |
| 最低版本 | iOS 16+ / macOS 13+ / watchOS 9+ |

### 1.2 与第三方图表库对比

| 对比项 | SwiftUI Charts | Charts (DGCharts) | SwiftCharts (swift-charts) |
|--------|---------------|-------------------|---------------------------|
| 维护方 | Apple 官方 | 开源社区 | Apple 开源实验项目 |
| 最低版本 | iOS 16+ | iOS 12+ | iOS 16+ |
| 图表类型 | 柱/线/面/点/饼 | 柱/线/饼/雷达/散点等 30+ | 柱/线/点 |
| 定制程度 | 中等（官方风格） | 高（完全自定义） | 高（底层 API） |
| SwiftUI 集成 | ✅ 原生 | ❌ 需 UIViewRepresentable | ✅ 原生 |
| 学习曲线 | 低 | 中 | 高 |
| 包体积 | 无额外开销 | ~2MB | 无额外开销 |
| 动画 | 内置 | 手动配置 | 手动配置 |

> 💡 **选择建议**：新项目且只需常见图表类型，首选 SwiftUI Charts；需要雷达图、热力图等特殊类型，考虑 DGCharts；追求极致自定义，可研究 swift-charts 开源项目。

### 1.3 适用场景

- 健康与健身 App：展示步数、心率、睡眠趋势
- 金融与理财 App：收支统计、投资收益曲线
- 效率工具 App：任务完成率、时间分配
- 天气 App：温度变化、降水概率
- 电商 App：销售数据、用户增长

---

## 2. 基础图表类型

SwiftUI Charts 的核心思路是：**数据 + 标记（Mark）= 图表**。你提供数据，选择一种 Mark 类型，框架帮你渲染。

### 2.1 柱状图 BarChart

柱状图就像一排高低不同的柱子，适合**比较不同类别之间的数值差异**。

```swift
import SwiftUI
import Charts

struct SalesData: Identifiable {
    let id = UUID()
    let month: String
    let revenue: Double
}

struct BarChartView: View {
    let data: [SalesData] = [
        SalesData(month: "1月", revenue: 4200),
        SalesData(month: "2月", revenue: 3800),
        SalesData(month: "3月", revenue: 5100),
        SalesData(month: "4月", revenue: 4700),
        SalesData(month: "5月", revenue: 6200),
        SalesData(month: "6月", revenue: 5800)
    ]

    var body: some View {
        Chart(data) { item in
            BarMark(
                x: .value("月份", item.month),
                y: .value("营收", item.revenue)
            )
            .foregroundStyle(Color.blue.gradient)
        }
        .frame(height: 300)
        .padding()
    }
}
```

### 2.2 折线图 LineChart

折线图就像用线把散落的点连起来，适合**展示数据随时间的变化趋势**。

```swift
struct LineChartView: View {
    let data: [SalesData] = [
        SalesData(month: "1月", revenue: 4200),
        SalesData(month: "2月", revenue: 3800),
        SalesData(month: "3月", revenue: 5100),
        SalesData(month: "4月", revenue: 4700),
        SalesData(month: "5月", revenue: 6200),
        SalesData(month: "6月", revenue: 5800)
    ]

    var body: some View {
        Chart(data) { item in
            LineMark(
                x: .value("月份", item.month),
                y: .value("营收", item.revenue)
            )
            .foregroundStyle(Color.purple)
            .lineStyle(StrokeStyle(lineWidth: 3))
        }
        .frame(height: 300)
        .padding()
    }
}
```

### 2.3 面积图 AreaChart

面积图是折线图的"填充版"，线以下区域被颜色填满，适合**强调数据的累积感和量级**。

```swift
struct AreaChartView: View {
    let data: [SalesData] = [
        SalesData(month: "1月", revenue: 4200),
        SalesData(month: "2月", revenue: 3800),
        SalesData(month: "3月", revenue: 5100),
        SalesData(month: "4月", revenue: 4700),
        SalesData(month: "5月", revenue: 6200),
        SalesData(month: "6月", revenue: 5800)
    ]

    var body: some View {
        Chart(data) { item in
            AreaMark(
                x: .value("月份", item.month),
                y: .value("营收", item.revenue)
            )
            .foregroundStyle(
                .linearGradient(
                    colors: [.purple.opacity(0.6), .purple.opacity(0.1)],
                    startPoint: .top,
                    endPoint: .bottom
                )
            )
        }
        .frame(height: 300)
        .padding()
    }
}
```

### 2.4 散点图 PointChart

散点图就像在坐标系上撒豆子，适合**展示数据的分布和离散程度**。

```swift
struct ScatterData: Identifiable {
    let id = UUID()
    let studyHours: Double
    let score: Double
}

struct PointChartView: View {
    let data: [ScatterData] = [
        ScatterData(studyHours: 1, score: 45),
        ScatterData(studyHours: 2, score: 55),
        ScatterData(studyHours: 3, score: 62),
        ScatterData(studyHours: 4, score: 71),
        ScatterData(studyHours: 5, score: 78),
        ScatterData(studyHours: 6, score: 85),
        ScatterData(studyHours: 7, score: 90),
        ScatterData(studyHours: 8, score: 94)
    ]

    var body: some View {
        Chart(data) { item in
            PointMark(
                x: .value("学习时长(h)", item.studyHours),
                y: .value("成绩", item.score)
            )
            .foregroundStyle(Color.orange)
            .symbolSize(80)
        }
        .frame(height: 300)
        .padding()
    }
}
```

### 2.5 饼图 PieChart

> ⚠️ **注意**：iOS 17 才正式支持 SectorMark（饼图/扇形图），iOS 16 没有原生饼图。

饼图就像把一个披萨切成大小不同的块，适合**展示各部分占整体的比例**。

```swift
struct CategoryData: Identifiable {
    let id = UUID()
    let category: String
    let amount: Double
}

struct PieChartView: View {
    let data: [CategoryData] = [
        CategoryData(category: "餐饮", amount: 2800),
        CategoryData(category: "交通", amount: 1200),
        CategoryData(category: "娱乐", amount: 1500),
        CategoryData(category: "购物", amount: 2000),
        CategoryData(category: "其他", amount: 800)
    ]

    var body: some View {
        Chart(data) { item in
            SectorMark(
                angle: .value("金额", item.amount),
                innerRadius: .ratio(0.5),
                angularInset: 2
            )
            .foregroundStyle(by: .value("类别", item.category))
            .annotation(position: .overlay) {
                Text("\(Int(item.amount / data.map(\.amount).reduce(0, +) * 100))%")
                    .font(.caption2.bold())
                    .foregroundStyle(.white)
            }
        }
        .frame(height: 300)
        .padding()
    }
}
```

> 💡 **生活类比**：柱状图像比身高，折线图像看走势，面积图像看水库蓄水量，散点图像撒豆子找规律，饼图像切披萨看份额。

---

## 3. 图表数据模型

数据是图表的灵魂。SwiftUI Charts 要求你提供结构化的数据，框架负责把它们"翻译"成图形。

### 3.1 Identifiable 数据

Charts 的 `Chart` 初始化方法要求集合元素遵循 `Identifiable` 协议：

```swift
struct MonthlyRevenue: Identifiable {
    let id = UUID()
    let month: String
    let revenue: Double
    let expense: Double
}
```

### 3.2 Plottable 协议

`x:` 和 `y:` 参数接受 `Plottable` 类型的值。Swift 标准类型已自动遵循：

| 已支持类型 | 说明 |
|-----------|------|
| `String` | 用于分类轴（如月份、城市名） |
| `Int` / `Double` / `Float` | 用于数值轴 |
| `Date` | 用于时间轴，自动格式化 |

使用 `.value()` 包装器提供语义标签：

```swift
BarMark(
    x: .value("月份", "3月"),
    y: .value("营收(元)", 5100.0)
)
```

### 3.3 多数据系列

通过 `foregroundStyle(by:)` 或 `series` 参数区分多组数据：

```swift
struct MultiSeriesData: Identifiable {
    let id = UUID()
    let month: String
    let series: String
    let value: Double
}

struct MultiSeriesChart: View {
    let data: [MultiSeriesData] = [
        MultiSeriesData(month: "1月", series: "收入", value: 4200),
        MultiSeriesData(month: "1月", series: "支出", value: 3100),
        MultiSeriesData(month: "2月", series: "收入", value: 3800),
        MultiSeriesData(month: "2月", series: "支出", value: 2900),
        MultiSeriesData(month: "3月", series: "收入", value: 5100),
        MultiSeriesData(month: "3月", series: "支出", value: 3400)
    ]

    var body: some View {
        Chart(data) { item in
            BarMark(
                x: .value("月份", item.month),
                y: .value("金额", item.value)
            )
            .foregroundStyle(by: .value("类型", item.series))
        }
        .frame(height: 300)
    }
}
```

### 3.4 自定义数据类型

对于 `Date` 类型的 X 轴，建议使用 `Calendar.Component` 进行分组：

```swift
struct DailyStep: Identifiable {
    let id = UUID()
    let date: Date
    let steps: Int
}

struct DateChartView: View {
    let data: [DailyStep]

    var body: some View {
        Chart(data) { item in
            LineMark(
                x: .value("日期", item.date, unit: .day),
                y: .value("步数", item.steps)
            )
        }
        .chartXAxis {
            AxisMarks(values: .stride(by: .day)) { value in
                AxisGridLine()
                AxisValueLabel(format: .dateTime.month(.abbreviated).day())
            }
        }
    }
}
```

---

## 4. 图表定制

默认图表能用，但要让图表"好看又好用"，还需要定制外观。

### 4.1 颜色与渐变

```swift
BarMark(
    x: .value("月份", item.month),
    y: .value("营收", item.revenue)
)
.foregroundStyle(
    .linearGradient(
        colors: [.cyan, .blue],
        startPoint: .bottom,
        endPoint: .top
    )
)
```

也可以按数值动态着色：

```swift
.foregroundStyle(
    by: .value("营收等级", item.revenue > 5000 ? "高" : "低")
)
```

### 4.2 图例 Legend

```swift
Chart(data) { item in
    BarMark(
        x: .value("月份", item.month),
        y: .value("金额", item.value)
    )
    .foregroundStyle(by: .value("类型", item.series))
}
.chartLegend(position: .bottom, alignment: .leading)
.chartLegend {
    HStack {
        RoundedRectangle(cornerRadius: 4)
            .fill(Color.blue)
            .frame(width: 12, height: 12)
        Text("收入")
        RoundedRectangle(cornerRadius: 4)
            .fill(Color.orange)
            .frame(width: 12, height: 12)
        Text("支出")
    }
    .font(.caption)
}
```

### 4.3 标注 Annotation

在数据点上添加文字说明，就像给照片加备注：

```swift
Chart(data) { item in
    BarMark(
        x: .value("月份", item.month),
        y: .value("营收", item.revenue)
    )
    .annotation(position: .top) {
        Text("¥\(Int(item.revenue))")
            .font(.caption2)
            .foregroundStyle(.secondary)
    }
}
```

### 4.4 标记线与参考线

用 `ChartOverlay` 或 `RuleMark` 添加参考线，就像在地图上画一条"及格线"：

```swift
Chart(data) { item in
    BarMark(
        x: .value("月份", item.month),
        y: .value("营收", item.revenue)
    )
    RuleMark(y: .value("目标", 5000))
        .foregroundStyle(Color.red.opacity(0.5))
        .lineStyle(StrokeStyle(lineWidth: 2, dash: [5, 5]))
        .annotation(position: .top, alignment: .trailing) {
            Text("目标线")
                .font(.caption)
                .foregroundStyle(.red)
        }
}
```

### 4.5 自定义标记 Mark

除了标准 Mark，你还可以在同一 Chart 中组合多种 Mark：

```swift
Chart(data) { item in
    BarMark(
        x: .value("月份", item.month),
        y: .value("营收", item.revenue)
    )
    .foregroundStyle(Color.blue.opacity(0.3))

    LineMark(
        x: .value("月份", item.month),
        y: .value("营收", item.revenue)
    )
    .foregroundStyle(Color.blue)
    .lineStyle(StrokeStyle(lineWidth: 2))

    PointMark(
        x: .value("月份", item.month),
        y: .value("营收", item.revenue)
    )
    .foregroundStyle(Color.blue)
}
```

### 4.6 坐标轴定制

```swift
.chartXAxis {
    AxisMarks(position: .bottom) { value in
        AxisGridLine()
        AxisValueLabel()
    }
}
.chartYAxis {
    AxisMarks(position: .leading) { value in
        AxisGridLine(stroke: StrokeStyle(lineWidth: 0.5))
            .foregroundStyle(Color.gray.opacity(0.3))
        AxisValueLabel()
    }
}
.chartYScale(domain: 0...8000)
```

---

## 5. 交互与动画

静态图表是"死的"，加上交互和动画，图表就"活"了。

### 5.1 图表选区 .chartXSelection / .chartYSelection

用户点击或拖拽图表时，可以捕获选中的数据点：

```swift
struct InteractiveLineChart: View {
    let data: [SalesData]
    @State private var selectedMonth: String?

    var body: some View {
        Chart(data) { item in
            LineMark(
                x: .value("月份", item.month),
                y: .value("营收", item.revenue)
            )
            .foregroundStyle(Color.purple)

            if let selected = selectedMonth,
               item.month == selected {
                PointMark(
                    x: .value("月份", item.month),
                    y: .value("营收", item.revenue)
                )
                .foregroundStyle(Color.purple)
                .symbolSize(120)
                .annotation(position: .top) {
                    VStack {
                        Text(item.month)
                            .font(.caption.bold())
                        Text("¥\(Int(item.revenue))")
                            .font(.caption)
                            .foregroundStyle(.secondary)
                    }
                }
            }
        }
        .chartXSelection(value: $selectedMonth)
        .frame(height: 300)
        .padding()
    }
}
```

### 5.2 Y 轴选区

```swift
@State private var selectedRevenue: Double?

Chart(data) { item in
    BarMark(
        x: .value("月份", item.month),
        y: .value("营收", item.revenue)
    )
}
.chartYSelection(value: $selectedRevenue)
```

### 5.3 图表动画

SwiftUI Charts 默认就有动画效果。当数据变化时，图表会自动平滑过渡：

```swift
struct AnimatedChart: View {
    @State private var showFullData = false

    var displayedData: [SalesData] {
        showFullData ? fullData : Array(fullData.prefix(3))
    }

    let fullData: [SalesData] = [
        SalesData(month: "1月", revenue: 4200),
        SalesData(month: "2月", revenue: 3800),
        SalesData(month: "3月", revenue: 5100),
        SalesData(month: "4月", revenue: 4700),
        SalesData(month: "5月", revenue: 6200),
        SalesData(month: "6月", revenue: 5800)
    ]

    var body: some View {
        VStack {
            Chart(displayedData) { item in
                BarMark(
                    x: .value("月份", item.month),
                    y: .value("营收", item.revenue)
                )
                .foregroundStyle(Color.blue.gradient)
            }
            .frame(height: 300)
            .animation(.easeInOut(duration: 0.6), value: displayedData.count)

            Button(showFullData ? "显示部分" : "显示全部") {
                showFullData.toggle()
            }
            .buttonStyle(.borderedProminent)
        }
        .padding()
    }
}
```

### 5.4 数据变化过渡动画

当数据值本身变化时，配合 `withAnimation` 实现平滑过渡：

```swift
struct TransitionChart: View {
    @State private var revenues = [4200.0, 3800, 5100, 4700, 6200, 5800]
    let months = ["1月", "2月", "3月", "4月", "5月", "6月"]

    var body: some View {
        VStack {
            Chart(Array(zip(months, revenues)), id: \.0) { month, revenue in
                BarMark(
                    x: .value("月份", month),
                    y: .value("营收", revenue)
                )
                .foregroundStyle(Color.teal.gradient)
            }
            .frame(height: 300)

            Button("刷新数据") {
                withAnimation(.spring(duration: 0.6, bounce: 0.3)) {
                    revenues = revenues.map { $0 * Double.random(in: 0.7...1.3) }
                }
            }
            .buttonStyle(.borderedProminent)
        }
        .padding()
    }
}
```

---

## 6. 高级图表

### 6.1 堆叠柱状图

堆叠柱状图就像把不同颜色的积木叠在一起，展示各部分的构成：

```swift
struct StackedBarChart: View {
    let data: [MultiSeriesData] = [
        MultiSeriesData(month: "1月", series: "餐饮", value: 1500),
        MultiSeriesData(month: "1月", series: "交通", value: 600),
        MultiSeriesData(month: "1月", series: "娱乐", value: 1000),
        MultiSeriesData(month: "2月", series: "餐饮", value: 1300),
        MultiSeriesData(month: "2月", series: "交通", value: 700),
        MultiSeriesData(month: "2月", series: "娱乐", value: 900),
        MultiSeriesData(month: "3月", series: "餐饮", value: 1600),
        MultiSeriesData(month: "3月", series: "交通", value: 500),
        MultiSeriesData(month: "3月", series: "娱乐", value: 1300)
    ]

    var body: some View {
        Chart(data) { item in
            BarMark(
                x: .value("月份", item.month),
                y: .value("金额", item.value)
            )
            .foregroundStyle(by: .value("类别", item.series))
            .position(by: .value("类别", item.series), stack: .standard)
        }
        .frame(height: 300)
    }
}
```

### 6.2 范围图

范围图用矩形区域表示数据的上下限，就像天气预报中的"最高温/最低温"：

```swift
struct WeatherData: Identifiable {
    let id = UUID()
    let day: String
    let low: Double
    let high: Double
}

struct RangeChartView: View {
    let data: [WeatherData] = [
        WeatherData(day: "周一", low: 12, high: 22),
        WeatherData(day: "周二", low: 14, high: 24),
        WeatherData(day: "周三", low: 11, high: 20),
        WeatherData(day: "周四", low: 13, high: 23),
        WeatherData(day: "周五", low: 15, high: 26),
        WeatherData(day: "周六", low: 16, high: 28),
        WeatherData(day: "周日", low: 14, high: 25)
    ]

    var body: some View {
        Chart(data) { item in
            RectangleMark(
                x: .value("日期", item.day),
                yStart: .value("最低温", item.low),
                yEnd: .value("最高温", item.high)
            )
            .foregroundStyle(Color.orange.opacity(0.3))

            LineMark(
                x: .value("日期", item.day),
                y: .value("最高温", item.high)
            )
            .foregroundStyle(Color.red)

            LineMark(
                x: .value("日期", item.day),
                y: .value("最低温", item.low)
            )
            .foregroundStyle(Color.blue)
        }
        .frame(height: 300)
    }
}
```

### 6.3 双轴图

当两组数据的量级差异很大时，需要左右两条 Y 轴：

```swift
struct DualAxisData: Identifiable {
    let id = UUID()
    let month: String
    let revenue: Double
    let userCount: Double
}

struct DualAxisChart: View {
    let data: [DualAxisData] = [
        DualAxisData(month: "1月", revenue: 42000, userCount: 120),
        DualAxisData(month: "2月", revenue: 38000, userCount: 135),
        DualAxisData(month: "3月", revenue: 51000, userCount: 168),
        DualAxisData(month: "4月", revenue: 47000, userCount: 155),
        DualAxisData(month: "5月", revenue: 62000, userCount: 210),
        DualAxisData(month: "6月", revenue: 58000, userCount: 195)
    ]

    var body: some View {
        Chart(data) { item in
            BarMark(
                x: .value("月份", item.month),
                y: .value("营收", item.revenue)
            )
            .foregroundStyle(Color.blue.opacity(0.4))

            LineMark(
                x: .value("月份", item.month),
                y: .value("用户数", item.userCount)
            )
            .foregroundStyle(Color.orange)
            .lineStyle(StrokeStyle(lineWidth: 3))
        }
        .chartYScale(domain: 0...70000)
        .chartYAxis {
            AxisMarks(position: .leading) { _ in
                AxisGridLine()
                AxisValueLabel()
            }
        }
        .frame(height: 300)
    }
}
```

> ⚠️ **注意**：SwiftUI Charts 目前不原生支持双 Y 轴，上面的方案是简化版。如需精确双轴，可通过 `chartOverlay` 自定义绘制第二轴，或使用 ZStack 叠加两个 Chart。

### 6.4 多图组合

用 `VStack` / `HStack` / `Grid` 组合多个图表，构建仪表盘：

```swift
struct DashboardGrid: View {
    var body: some View {
        ScrollView {
            LazyVGrid(columns: [
                GridItem(.flexible()),
                GridItem(.flexible())
            ], spacing: 16) {
                ChartCard(title: "月度营收") {
                    BarChartContent()
                }
                ChartCard(title: "用户增长") {
                    LineChartContent()
                }
                ChartCard(title: "支出分布") {
                    PieChartContent()
                }
                ChartCard(title: "温度范围") {
                    RangeChartContent()
                }
            }
            .padding()
        }
    }
}

struct ChartCard<Content: View>: View {
    let title: String
    @ViewBuilder let content: () -> Content

    var body: some View {
        VStack(alignment: .leading) {
            Text(title)
                .font(.headline)
            content()
        }
        .padding()
        .background(.ultraThinMaterial, in: RoundedRectangle(cornerRadius: 12))
    }
}
```

### 6.5 可滚动图表

数据量大时，用 `chartScrollableAxes` 让图表可以横向滚动：

```swift
Chart(data) { item in
    LineMark(
        x: .value("日期", item.date, unit: .day),
        y: .value("步数", item.steps)
    )
}
.chartScrollableAxes(.horizontal)
.chartScrollPosition(initialX: data.last?.date)
.frame(height: 300)
```

> 💡 **提示**：`chartScrollableAxes` 需要 iOS 17+。iOS 16 可通过 `ScrollView` + 固定宽度 Chart 实现类似效果。

---

## 7. 实战：数据仪表盘

综合运用前面所学，构建一个包含多种图表的实时数据仪表盘。

### 7.1 数据模型

```swift
import SwiftUI
import Charts

struct DashboardData {
    let monthlyRevenues: [MonthlyRevenue]
    let categoryExpenses: [CategoryExpense]
    let weeklySteps: [DailyStep]
    let temperatureRange: [WeatherRange]

    static let preview = DashboardData(
        monthlyRevenues: [
            MonthlyRevenue(month: "1月", revenue: 4200, expense: 3100),
            MonthlyRevenue(month: "2月", revenue: 3800, expense: 2900),
            MonthlyRevenue(month: "3月", revenue: 5100, expense: 3400),
            MonthlyRevenue(month: "4月", revenue: 4700, expense: 3200),
            MonthlyRevenue(month: "5月", revenue: 6200, expense: 3800),
            MonthlyRevenue(month: "6月", revenue: 5800, expense: 3600)
        ],
        categoryExpenses: [
            CategoryExpense(category: "餐饮", amount: 2800),
            CategoryExpense(category: "交通", amount: 1200),
            CategoryExpense(category: "娱乐", amount: 1500),
            CategoryExpense(category: "购物", amount: 2000),
            CategoryExpense(category: "其他", amount: 800)
        ],
        weeklySteps: [
            DailyStep(day: "周一", steps: 6800),
            DailyStep(day: "周二", steps: 8200),
            DailyStep(day: "周三", steps: 5400),
            DailyStep(day: "周四", steps: 9100),
            DailyStep(day: "周五", steps: 7600),
            DailyStep(day: "周六", steps: 11200),
            DailyStep(day: "周日", steps: 9800)
        ],
        temperatureRange: [
            WeatherRange(day: "周一", low: 12, high: 22),
            WeatherRange(day: "周二", low: 14, high: 24),
            WeatherRange(day: "周三", low: 11, high: 20),
            WeatherRange(day: "周四", low: 13, high: 23),
            WeatherRange(day: "周五", low: 15, high: 26),
            WeatherRange(day: "周六", low: 16, high: 28),
            WeatherRange(day: "周日", low: 14, high: 25)
        ]
    )
}

struct MonthlyRevenue: Identifiable {
    let id = UUID()
    let month: String
    let revenue: Double
    let expense: Double
}

struct CategoryExpense: Identifiable {
    let id = UUID()
    let category: String
    let amount: Double
}

struct DailyStep: Identifiable {
    let id = UUID()
    let day: String
    let steps: Int
}

struct WeatherRange: Identifiable {
    let id = UUID()
    let day: String
    let low: Double
    let high: Double
}
```

### 7.2 仪表盘主视图

```swift
struct DashboardView: View {
    @State private var data = DashboardData.preview
    @State private var selectedMonth: String?

    var body: some View {
        ScrollView {
            VStack(spacing: 20) {
                summaryCards

                revenueChart

                HStack(spacing: 16) {
                    expensePieChart
                    stepsChart
                }

                temperatureChart
            }
            .padding()
        }
        .navigationTitle("数据仪表盘")
        .toolbar {
            Button("刷新") {
                withAnimation(.spring(duration: 0.6, bounce: 0.2)) {
                    data = DashboardData.preview
                }
            }
        }
    }

    private var summaryCards: some View {
        HStack(spacing: 12) {
            SummaryCard(title: "总营收", value: "¥29,800", trend: "+12%", color: .blue)
            SummaryCard(title: "总支出", value: "¥20,000", trend: "-3%", color: .orange)
            SummaryCard(title: "净收入", value: "¥9,800", trend: "+28%", color: .green)
        }
    }

    private var revenueChart: some View {
        VStack(alignment: .leading) {
            Text("月度收支趋势")
                .font(.headline)
            Chart(data.monthlyRevenues) { item in
                BarMark(
                    x: .value("月份", item.month),
                    y: .value("金额", item.revenue)
                )
                .foregroundStyle(Color.blue.gradient)
                .position(by: .value("类型", "收入"), stack: .standard)

                BarMark(
                    x: .value("月份", item.month),
                    y: .value("金额", item.expense)
                )
                .foregroundStyle(Color.orange.gradient)
                .position(by: .value("类型", "支出"), stack: .standard)
            }
            .chartXSelection(value: $selectedMonth)
            .frame(height: 250)
        }
        .padding()
        .background(.ultraThinMaterial, in: RoundedRectangle(cornerRadius: 16))
    }

    private var expensePieChart: some View {
        VStack(alignment: .leading) {
            Text("支出分布")
                .font(.headline)
            Chart(data.categoryExpenses) { item in
                SectorMark(
                    angle: .value("金额", item.amount),
                    innerRadius: .ratio(0.5),
                    angularInset: 2
                )
                .foregroundStyle(by: .value("类别", item.category))
            }
            .frame(height: 200)
        }
        .padding()
        .background(.ultraThinMaterial, in: RoundedRectangle(cornerRadius: 16))
    }

    private var stepsChart: some View {
        VStack(alignment: .leading) {
            Text("本周步数")
                .font(.headline)
            Chart(data.weeklySteps) { item in
                BarMark(
                    x: .value("日期", item.day),
                    y: .value("步数", item.steps)
                )
                .foregroundStyle(Color.green.gradient)
            }
            .frame(height: 200)
        }
        .padding()
        .background(.ultraThinMaterial, in: RoundedRectangle(cornerRadius: 16))
    }

    private var temperatureChart: some View {
        VStack(alignment: .leading) {
            Text("温度范围")
                .font(.headline)
            Chart(data.temperatureRange) { item in
                RectangleMark(
                    x: .value("日期", item.day),
                    yStart: .value("最低温", item.low),
                    yEnd: .value("最高温", item.high)
                )
                .foregroundStyle(Color.orange.opacity(0.3))

                LineMark(
                    x: .value("日期", item.day),
                    y: .value("最高温", item.high)
                )
                .foregroundStyle(Color.red)
                .lineStyle(StrokeStyle(lineWidth: 2))

                LineMark(
                    x: .value("日期", item.day),
                    y: .value("最低温", item.low)
                )
                .foregroundStyle(Color.blue)
                .lineStyle(StrokeStyle(lineWidth: 2))
            }
            .frame(height: 200)
        }
        .padding()
        .background(.ultraThinMaterial, in: RoundedRectangle(cornerRadius: 16))
    }
}

struct SummaryCard: View {
    let title: String
    let value: String
    let trend: String
    let color: Color

    var body: some View {
        VStack(alignment: .leading, spacing: 4) {
            Text(title)
                .font(.caption)
                .foregroundStyle(.secondary)
            Text(value)
                .font(.title3.bold())
            Text(trend)
                .font(.caption.bold())
                .foregroundStyle(trend.hasPrefix("+") ? .green : .red)
        }
        .frame(maxWidth: .infinity, alignment: .leading)
        .padding()
        .background(color.opacity(0.1), in: RoundedRectangle(cornerRadius: 12))
    }
}
```

### 7.3 深色模式适配

SwiftUI Charts 自动适配深色模式，但自定义颜色需要适配：

```swift
.foregroundStyle(
    Color(.label)
)

.background(
    Color(.systemBackground)
)
```

> 💡 **提示**：使用 `.ultraThinMaterial` 作为卡片背景，深色模式下自动变为半透明毛玻璃效果，比固定颜色更优雅。

---

## 8. 图表设计最佳实践

好的图表不仅要"画出来"，更要"看得懂"。

### 8.1 数据墨水比

数据墨水比 = 数据相关墨水 / 总墨水。**墨水应该花在数据上，而不是装饰上**。

| 做法 | 数据墨水比 | 建议 |
|------|-----------|------|
| 粗网格线 + 渐变背景 | 低 ❌ | 减少非数据元素 |
| 细网格线 + 白色背景 | 中 ✅ | 保留必要参考线 |
| 无网格线 + 数据标注 | 高 ✅ | 数据本身说话 |

```swift
Chart(data) { item in
    BarMark(
        x: .value("月份", item.month),
        y: .value("营收", item.revenue)
    )
}
.chartXAxis {
    AxisMarks { _ in
        AxisValueLabel()
    }
}
.chartYAxis {
    AxisMarks { _ in
        AxisGridLine(stroke: StrokeStyle(lineWidth: 0.5))
            .foregroundStyle(.gray.opacity(0.2))
        AxisValueLabel()
    }
}
```

### 8.2 颜色选择策略

| 场景 | 推荐方案 | 示例 |
|------|---------|------|
| 单数据系列 | 单色渐变 | 蓝色从浅到深 |
| 对比数据 | 对比色 | 蓝色 vs 橙色 |
| 多类别数据 | 色相环均匀取色 | 系统默认配色 |
| 正负数据 | 语义色 | 绿色(正) / 红色(负) |
| 深色模式 | 高饱和度 | 避免浅色变不可见 |

```swift
.foregroundStyle(
    by: .value("趋势", item.value >= 0 ? "增长" : "下降")
)
```

### 8.3 标注策略

| 标注方式 | 适用场景 | 注意事项 |
|---------|---------|---------|
| 数据标签 | 数据点少（≤8） | 避免标签重叠 |
| Tooltip/选区 | 数据点多 | 需交互触发 |
| 图例 + 轴标签 | 通用方案 | 确保可读性 |
| 无标注 | 趋势展示 | 仅展示大趋势 |

> ⚠️ **警告**：数据点超过 10 个时，不要在每根柱子上方都加标注——密密麻麻的文字反而降低可读性。改用选区交互。

### 8.4 无障碍适配

图表对视觉障碍用户来说是个挑战。SwiftUI Charts 提供了内置支持：

```swift
Chart(data) { item in
    BarMark(
        x: .value("月份", item.month),
        y: .value("营收", item.revenue)
    )
}
.accessibilityLabel("月度营收图表")
.accessibilityValue("1月到6月的营收数据")
.chartLegend {
    AxisMarks { _ in
        AxisValueLabel()
    }
}
```

无障碍设计要点：

| 要点 | 实现方式 |
|------|---------|
| 图表描述 | `.accessibilityLabel` / `.accessibilityValue` |
| 数据可读 | VoiceOver 自动朗读数据点 |
| 颜色盲友好 | 不只依赖颜色，加上形状/纹理区分 |
| 动态字体 | 标注文字随系统字体缩放 |
| 对比度 | 确保前景/背景对比度 ≥ 4.5:1 |

> 💡 **生活类比**：图表就像演讲——数据是内容，颜色是语气，标注是重点强调，无障碍是确保每个听众都能听懂。

---

## 本章小结

| 主题 | 核心要点 |
|------|---------|
| Charts 概述 | iOS 16+ 原生框架，声明式 API，零依赖 |
| 基础图表 | BarMark / LineMark / AreaMark / PointMark / SectorMark |
| 数据模型 | Identifiable + Plottable，.value() 提供语义标签 |
| 图表定制 | 渐变色、图例、标注、RuleMark 参考线、坐标轴定制 |
| 交互动画 | .chartXSelection / .chartYSelection、withAnimation 过渡 |
| 高级图表 | 堆叠柱状图、范围图、双轴图、多图组合、可滚动图表 |
| 实战仪表盘 | 综合运用多种图表 + 实时刷新 + 深色模式 |
| 最佳实践 | 高数据墨水比、语义化颜色、精简标注、无障碍适配 |

> 💡 **下一步**：结合第 35 章 SwiftData，将图表绑定到持久化数据源，实现真正的"数据驱动可视化"。
