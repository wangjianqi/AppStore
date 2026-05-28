---
name: app-intents-apple-intelligence
description: 涉及 App Intents、Apple Intelligence、Controls API、Spotlight 索引、Siri 快捷指令、SiriKit、App Shortcuts 的任务
---

# App Intents / Apple Intelligence

## App Intents 框架

### 基本结构
```swift
struct SearchExpenseIntent: AppIntent {
    static var title: LocalizedStringResource = "搜索记账记录"
    static var description = IntentDescription("按关键词搜索记账记录")

    @Parameter(title: "关键词")
    var keyword: String

    func perform() async throws -> some IntentResult {
        let results = try await ExpenseService.search(keyword: keyword)
        return .result(dialog: "找到 \(results.count) 条记录")
    }
}
```

### Intent 类型
- AppIntent：基本意图，执行操作
- AppShortcutsProvider：注册 App 快捷指令
- WidgetConfigurationIntent：Widget 配置意图
- AudioPlaybackIntent：音频播放意图
- ForegroundContinuableIntent：需要前台继续的意图

### 参数设计
- 使用 @Parameter 标注参数
- 支持 String、Int、Double、Bool、Date、URL 等基本类型
- 支持自定义 Enum（需实现 AppEnum 协议）
- 支持 Entity 查询（需实现 EntityQuery 协议）

---

## App Shortcuts

### 注册快捷指令
```swift
struct ExpenseAppShortcuts: AppShortcutsProvider {
    static var appShortcuts: [AppShortcut] {
        AppShortcut(
            intent: SearchExpenseIntent(),
            phrases: [
                "在\(.applicationName)中搜索\(\.$keyword)",
                "用\(.applicationName)查账"
            ],
            shortTitle: "搜索记账",
            systemImageName: "magnifyingglass"
        )
    }
}
```

### 短语设计原则
- 提供多种自然语言表达方式
- 包含 App 名称占位符
- 短语应简洁自然
- 避免过于技术化的表达

---

## Controls API（iOS 18+）

### 控件类型
- Widget：信息展示型控件
- Toggle：开关型控件
- Button：操作型控件

### 实现示例
```swift
struct QuickExpenseControl: ControlWidget {
    var body: some ControlWidgetConfiguration {
        StaticControlConfiguration(
            kind: "com.example.expense.quick"
        ) {
            ControlWidgetButton(action: QuickExpenseIntent()) {
                Label("快速记账", systemImage: "plus.circle.fill")
            }
        }
    }
}
```

### 设计要点
- 控件应在控制中心和锁屏可用
- 操作应一步完成，不需要额外输入
- 提供清晰的视觉反馈

---

## Spotlight 索引

### CSSearchableItem
```swift
import CoreSpotlight

func indexExpense(_ expense: Expense) {
    let item = CSSearchableItem(
        uniqueIdentifier: expense.id.uuidString,
        domainIdentifier: "expenses",
        attributeSet: {
            let set = CSSearchableItemAttributeSet(contentType: .text)
            set.title = expense.category
            set.contentDescription = "¥\(expense.amount) - \(expense.note)"
            set.keywords = [expense.category, expense.note]
            return set
        }()
    )
    CSSearchableIndex.default().indexSearchableItems([item])
}
```

### 索引策略
- 在数据创建/更新时同步索引
- 在数据删除时移除索引
- 批量索引使用 indexSearchableItems
- 避免索引过多数据影响性能

---

## Apple Intelligence 集成

### 适配要点
- 确保所有 App Intents 正确实现
- 提供丰富的 Siri 短语
- 使用 Semantic Attributes 帮助系统理解内容
- 遵循隐私框架：只在必要时请求数据访问

### 隐私要求
- 在 PrivacyInfo.xcprivacy 中声明数据使用
- App Intents 涉及的数据类型需明确声明
- 用户可通过系统设置关闭特定 Intent
