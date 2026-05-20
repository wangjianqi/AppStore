# H-SwiftUI 组件视觉手册

> 本手册面向零基础 iOS 开发者，以表格 + 代码片段的形式，快速查阅 SwiftUI 常用组件的用途、写法与修饰符。

---

## 1. 文本组件

### Text

| 项目 | 说明 |
|------|------|
| 用途 | 显示静态文本，是最基础的展示组件 |
| 核心代码 | `Text("Hello, SwiftUI!")` |

```swift
Text("Hello, SwiftUI!")
    .font(.title)
    .foregroundColor(.blue)
    .bold()
    .italic()
    .underline()
    .lineLimit(2)
    .multilineTextAlignment(.center)
```

**常用修饰符**

| 修饰符 | 作用 | 示例 |
|--------|------|------|
| `.font(_:)` | 设置字体 | `.font(.title2)` |
| `.foregroundColor(_:)` | 文字颜色 | `.foregroundColor(.red)` |
| `.bold()` | 加粗 | `.bold()` |
| `.italic()` | 斜体 | `.italic()` |
| `.underline(_:)` | 下划线 | `.underline(true, color: .blue)` |
| `.strikethrough(_:)` | 删除线 | `.strikethrough(true)` |
| `.lineLimit(_:)` | 行数限制 | `.lineLimit(3)` |
| `.multilineTextAlignment(_:)` | 多行对齐 | `.multilineTextAlignment(.leading)` |
| `.tracking(_:)` | 字间距 | `.tracking(2)` |
| `.kerning(_:)` | 字距调整 | `.kerning(1.5)` |

---

### TextField

| 项目 | 说明 |
|------|------|
| 用途 | 单行文本输入框，用于接收用户输入 |
| 核心代码 | `TextField("请输入", text: $input)` |

```swift
@State private var username = ""

TextField("请输入用户名", text: $username)
    .textFieldStyle(.roundedBorder)
    .padding()
```

**常用修饰符**

| 修饰符 | 作用 | 示例 |
|--------|------|------|
| `.textFieldStyle(_:)` | 输入框样式 | `.textFieldStyle(.roundedBorder)` |
| `.keyboardType(_:)` | 键盘类型 | `.keyboardType(.emailAddress)` |
| `.autocapitalization(_:)` | 自动大写 | `.autocapitalization(.none)` |
| `.disableAutocorrection(_:)` | 禁用自动纠错 | `.disableAutocorrection(true)` |
| `.onSubmit(_:)` | 提交回调 | `.onSubmit { print("提交") }` |
| `.prompt(_:)` | 占位提示 | `.prompt(Text("提示文字"))` |

---

### SecureField

| 项目 | 说明 |
|------|------|
| 用途 | 密码输入框，输入内容以圆点遮盖显示 |
| 核心代码 | `SecureField("密码", text: $password)` |

```swift
@State private var password = ""

SecureField("请输入密码", text: $password)
    .textFieldStyle(.roundedBorder)
    .padding()
```

**常用修饰符**

| 修饰符 | 作用 | 示例 |
|--------|------|------|
| `.textFieldStyle(_:)` | 输入框样式 | `.textFieldStyle(.roundedBorder)` |
| `.onSubmit(_:)` | 提交回调 | `.onSubmit { login() }` |
| `.keyboardType(_:)` | 键盘类型 | `.keyboardType(.asciiCapable)` |

---

### TextEditor

| 项目 | 说明 |
|------|------|
| 用途 | 多行文本编辑器，适合长文本输入 |
| 核心代码 | `TextEditor(text: $content)` |

```swift
@State private var content = "在此输入..."

TextEditor(text: $content)
    .font(.body)
    .scrollContentBackground(.hidden)
    .padding()
    .frame(height: 200)
```

**常用修饰符**

| 修饰符 | 作用 | 示例 |
|--------|------|------|
| `.font(_:)` | 字体 | `.font(.body)` |
| `.scrollContentBackground(_:)` | 隐藏默认背景 | `.scrollContentBackground(.hidden)` |
| `.lineSpacing(_:)` | 行间距 | `.lineSpacing(8)` |
| `.frame(height:)` | 固定高度 | `.frame(height: 200)` |

---

### Label

| 项目 | 说明 |
|------|------|
| 用途 | 图标 + 文字的组合组件 |
| 核心代码 | `Label("设置", systemImage: "gear")` |

```swift
Label("设置", systemImage: "gearshape")
    .font(.title3)

Label {
    Text("自定义标题").foregroundColor(.blue)
} icon: {
    Image("custom-icon")
}
```

**常用修饰符**

| 修饰符 | 作用 | 示例 |
|--------|------|------|
| `.labelStyle(_:)` | 标签样式 | `.labelStyle(.titleOnly)` / `.labelStyle(.iconOnly)` |
| `.font(_:)` | 字体 | `.font(.headline)` |
| `.foregroundColor(_:)` | 颜色 | `.foregroundColor(.primary)` |

---

## 2. 按钮组件

### Button

| 项目 | 说明 |
|------|------|
| 用途 | 可点击的按钮，触发操作 |
| 核心代码 | `Button("点击") { action }` |

```swift
Button("确定") {
    print("按钮被点击")
}
.buttonStyle(.borderedProminent)
.controlSize(.large)

Button(role: .destructive) {
    deleteItem()
} label: {
    Label("删除", systemImage: "trash")
}
```

**常用修饰符**

| 修饰符 | 作用 | 示例 |
|--------|------|------|
| `.buttonStyle(_:)` | 按钮样式 | `.buttonStyle(.bordered)` / `.borderedProminent` / `.borderless` |
| `.controlSize(_:)` | 尺寸 | `.controlSize(.mini)` / `.small` / `.regular` / `.large` |
| `.tint(_:)` | 主题色 | `.tint(.blue)` |
| `.disabled(_:)` | 禁用 | `.disabled(true)` |
| `.keyboardShortcut(_:)` | 键盘快捷键 | `.keyboardShortcut("s", modifiers: .command)` |

---

### NavigationLink

| 项目 | 说明 |
|------|------|
| 用途 | 导航跳转链接，用于在 NavigationStack 中跳转页面 |
| 核心代码 | `NavigationLink("下一页", value: item)` |

```swift
NavigationStack {
    List {
        NavigationLink("详情页") {
            DetailView()
        }
        NavigationLink(value: "settings") {
            Label("设置", systemImage: "gear")
        }
    }
    .navigationDestination(for: String.self) { value in
        SettingsView()
    }
}
```

**常用修饰符**

| 修饰符 | 作用 | 示例 |
|--------|------|------|
| `.navigationDestination(for:)` | 定义目标页面 | `.navigationDestination(for: Item.self) { ... }` |
| `.listRowBackground(_:)` | 行背景 | `.listRowBackground(Color.clear)` |

---

### Menu

| 项目 | 说明 |
|------|------|
| 用途 | 下拉菜单，点击展开一组选项 |
| 核心代码 | `Menu { options } label: { label }` |

```swift
Menu {
    Button("复制", systemImage: "doc.on.doc") { copy() }
    Button("粘贴", systemImage: "doc.on.clipboard") { paste() }
    Divider()
    Button("删除", systemImage: "trash", role: .destructive) { delete() }
} label: {
    Label("操作", systemImage: "ellipsis.circle")
}
```

**常用修饰符**

| 修饰符 | 作用 | 示例 |
|--------|------|------|
| `.menuStyle(_:)` | 菜单样式 | `.menuStyle(.borderlessButton)` |
| `.menuOrder(_:)` | 选项排序 | `.menuOrder(.priority)` |

---

### Link

| 项目 | 说明 |
|------|------|
| 用途 | 打开外部 URL 链接，跳转到 Safari |
| 核心代码 | `Link("访问", destination: url)` |

```swift
Link("访问 Apple", destination: URL(string: "https://www.apple.com")!)
    .tint(.blue)
```

**常用修饰符**

| 修饰符 | 作用 | 示例 |
|--------|------|------|
| `.tint(_:)` | 链接颜色 | `.tint(.green)` |
| `.font(_:)` | 字体 | `.font(.headline)` |

---

## 3. 选择组件

### Toggle

| 项目 | 说明 |
|------|------|
| 用途 | 开关切换，布尔值选择 |
| 核心代码 | `Toggle("飞行模式", isOn: $isOn)` |

```swift
@State private var isEnabled = true

Toggle("通知", isOn: $isEnabled)
    .tint(.green)
    .padding()
```

**常用修饰符**

| 修饰符 | 作用 | 示例 |
|--------|------|------|
| `.tint(_:)` | 开关颜色 | `.tint(.green)` |
| `.toggleStyle(_:)` | 开关样式 | `.toggleStyle(.switch)` / `.button` |
| `.labelsHidden()` | 隐藏标签 | `.labelsHidden()` |

---

### Picker

| 项目 | 说明 |
|------|------|
| 用途 | 从多个选项中选择一个值 |
| 核心代码 | `Picker("选择", selection: $selected) { options }` |

```swift
@State private var color = "红"

Picker("颜色", selection: $color) {
    Text("红").tag("红")
    Text("绿").tag("绿")
    Text("蓝").tag("蓝")
}
.pickerStyle(.segmented)
```

**常用修饰符**

| 修饰符 | 作用 | 示例 |
|--------|------|------|
| `.pickerStyle(_:)` | 选择器样式 | `.segmented` / `.wheel` / `.menu` / `.navigationLink` |
| `.labelsHidden()` | 隐藏标签 | `.labelsHidden()` |

---

### DatePicker

| 项目 | 说明 |
|------|------|
| 用途 | 日期/时间选择器 |
| 核心代码 | `DatePicker("日期", selection: $date)` |

```swift
@State private var date = Date()

DatePicker("选择日期", selection: $date, in: Date()..., displayedComponents: .date)
    .datePickerStyle(.graphical)
    .padding()
```

**常用修饰符**

| 修饰符 | 作用 | 示例 |
|--------|------|------|
| `.datePickerStyle(_:)` | 样式 | `.graphical` / `.wheel` / `.compact` / `.graphical` |
| `.environment(\.locale,)` | 地区 | `.environment(\.locale, Locale(identifier: "zh_CN"))` |
| `.displayedComponents(_:)` | 显示组件 | `.date` / `.hourAndMinute` |

---

### Stepper

| 项目 | 说明 |
|------|------|
| 用途 | 步进器，按固定步长增减数值 |
| 核心代码 | `Stepper("数量: \(count)", value: $count)` |

```swift
@State private var count = 0

Stepper("数量: \(count)", value: $count, in: 0...10, step: 1)
    .padding()
```

**常用修饰符**

| 修饰符 | 作用 | 示例 |
|--------|------|------|
| `.in` 范围 | 限定范围 | `value: $count, in: 0...100` |
| `step` | 步长 | `step: 5` |
| `.labelsHidden()` | 隐藏标签 | `.labelsHidden()` |

---

### Slider

| 项目 | 说明 |
|------|------|
| 用途 | 滑块，在范围内连续选择数值 |
| 核心代码 | `Slider(value: $volume, in: 0...1)` |

```swift
@State private var volume: Double = 0.5

Slider(value: $volume, in: 0...1, step: 0.01) {
    Text("音量")
} minimumValueLabel: {
    Image(systemName: "speaker.fill")
} maximumValueLabel: {
    Image(systemName: "speaker.wave.3.fill")
}
.tint(.blue)
.padding()
```

**常用修饰符**

| 修饰符 | 作用 | 示例 |
|--------|------|------|
| `.tint(_:)` | 滑块颜色 | `.tint(.orange)` |
| `step` | 步长 | `step: 0.1` |
| `.labelsHidden()` | 隐藏标签 | `.labelsHidden()` |

---

## 4. 容器组件

### VStack

| 项目 | 说明 |
|------|------|
| 用途 | 垂直方向排列子视图 |
| 核心代码 | `VStack { children }` |

```swift
VStack(alignment: .leading, spacing: 12) {
    Text("标题").font(.headline)
    Text("描述文字").font(.subheadline)
    Button("操作") { }
}
.padding()
```

**常用修饰符**

| 修饰符 | 作用 | 示例 |
|--------|------|------|
| `alignment` | 对齐方式 | `.leading` / `.center` / `.trailing` |
| `spacing` | 子视图间距 | `spacing: 16` |
| `.padding(_:)` | 内边距 | `.padding()` / `.padding(.horizontal, 16)` |

---

### HStack

| 项目 | 说明 |
|------|------|
| 用途 | 水平方向排列子视图 |
| 核心代码 | `HStack { children }` |

```swift
HStack(spacing: 8) {
    Image(systemName: "star.fill")
        .foregroundColor(.yellow)
    Text("评分")
    Spacer()
    Text("4.8")
}
.padding()
```

**常用修饰符**

| 修饰符 | 作用 | 示例 |
|--------|------|------|
| `alignment` | 垂直对齐 | `.top` / `.center` / `.bottom` |
| `spacing` | 子视图间距 | `spacing: 10` |
| `.padding(_:)` | 内边距 | `.padding(.horizontal)` |

---

### ZStack

| 项目 | 说明 |
|------|------|
| 用途 | 层叠排列子视图，后写的视图在上层 |
| 核心代码 | `ZStack { children }` |

```swift
ZStack {
    Color.blue
    Text("前景文字")
        .font(.title)
        .foregroundColor(.white)
}
.frame(width: 200, height: 100)
.cornerRadius(12)
```

**常用修饰符**

| 修饰符 | 作用 | 示例 |
|--------|------|------|
| `alignment` | 对齐方式 | `.topLeading` / `.center` / `.bottomTrailing` |
| `.frame()` | 固定尺寸 | `.frame(width: 200, height: 200)` |

---

### GroupBox

| 项目 | 说明 |
|------|------|
| 用途 | 分组容器，带标题和视觉边框 |
| 核心代码 | `GroupBox("标题") { content }` |

```swift
GroupBox("个人信息") {
    VStack(alignment: .leading) {
        Text("姓名：张三")
        Text("年龄：25")
        Text("城市：北京")
    }
}
.padding()
```

**常用修饰符**

| 修饰符 | 作用 | 示例 |
|--------|------|------|
| `.groupBoxStyle(_:)` | 分组样式 | `.groupBoxStyle(.automatic)` |
| `.padding(_:)` | 内边距 | `.padding()` |

---

### Section

| 项目 | 说明 |
|------|------|
| 用途 | 在 List 或 Form 中创建分组段落 |
| 核心代码 | `Section("标题") { content }` |

```swift
List {
    Section("基本信息") {
        Text("姓名")
        Text("邮箱")
    }
    Section("偏好设置") {
        Toggle("深色模式", isOn: $darkMode)
        Toggle("通知", isOn: $notification)
    }
}
```

**常用修饰符**

| 修饰符 | 作用 | 示例 |
|--------|------|------|
| `header` | 头部 | `Section(header: Text("标题"))` |
| `footer` | 底部 | `Section(footer: Text("说明"))` |
| `.collapsible()` | 可折叠 (iOS 17+) | `.collapsible(true)` |

---

## 5. 列表组件

### List

| 项目 | 说明 |
|------|------|
| 用途 | 可滚动的列表视图，支持分组、滑动操作 |
| 核心代码 | `List { items }` |

```swift
@State private var fruits = ["苹果", "香蕉", "橘子"]

List {
    ForEach(fruits, id: \.self) { fruit in
        Text(fruit)
    }
    .onDelete { indexSet in
        fruits.remove(atOffsets: indexSet)
    }
    .onMove { from, to in
        fruits.move(fromOffsets: from, toOffset: to)
    }
}
.listStyle(.insetGrouped)
```

**常用修饰符**

| 修饰符 | 作用 | 示例 |
|--------|------|------|
| `.listStyle(_:)` | 列表样式 | `.insetGrouped` / `.plain` / `.sidebar` |
| `.onDelete(_:)` | 滑动删除 | `.onDelete { offsets in ... }` |
| `.onMove(_:)` | 拖动排序 | `.onMove { from, to in ... }` |
| `.listRowSeparator(_:)` | 分割线 | `.listRowSeparator(.hidden)` |
| `.swipeActions(_:)` | 滑动操作 | `.swipeActions { Button("删除", role: .destructive) { } }` |
| `.refreshable(_:)` | 下拉刷新 | `.refreshable { await loadData() }` |

---

### ForEach

| 项目 | 说明 |
|------|------|
| 用途 | 遍历数据集合生成视图，常与 List 搭配 |
| 核心代码 | `ForEach(items) { item in view }` |

```swift
struct Item: Identifiable {
    let id = UUID()
    let name: String
}

@State private var items = [Item(name: "A"), Item(name: "B")]

ForEach(items) { item in
    Text(item.name)
}
```

**常用修饰符**

| 修饰符 | 作用 | 示例 |
|--------|------|------|
| `id:` | 指定标识 | `ForEach(fruits, id: \.self)` |
| 配合 `.onDelete` | 删除 | `.onDelete { ... }` |
| 配合 `.onMove` | 移动 | `.onMove { ... }` |

---

### LazyVStack / LazyHStack

| 项目 | 说明 |
|------|------|
| 用途 | 懒加载的垂直/水平堆叠，仅渲染可见区域，适合大量数据 |
| 核心代码 | `LazyVStack { items }` / `LazyHStack { items }` |

```swift
ScrollView {
    LazyVStack(spacing: 12) {
        ForEach(0..<1000, id: \.self) { index in
            Text("第 \(index) 项")
                .padding()
                .frame(maxWidth: .infinity)
                .background(Color(.systemGray6))
                .cornerRadius(8)
        }
    }
    .padding()
}

ScrollView(.horizontal) {
    LazyHStack(spacing: 16) {
        ForEach(0..<50, id: \.self) { index in
            RoundedRectangle(cornerRadius: 12)
                .fill(Color.blue.opacity(0.3))
                .frame(width: 120, height: 120)
                .overlay(Text("\(index)"))
        }
    }
    .padding()
}
```

**常用修饰符**

| 修饰符 | 作用 | 示例 |
|--------|------|------|
| `alignment` | 对齐 | `alignment: .leading` |
| `spacing` | 间距 | `spacing: 10` |
| `pinnedViews` | 固定视图 | `pinnedViews: [.sectionHeaders]` |

---

## 6. 导航组件

### NavigationStack

| 项目 | 说明 |
|------|------|
| 用途 | 导航容器，管理页面栈的推入与弹出 |
| 核心代码 | `NavigationStack { content }` |

```swift
NavigationStack {
    List {
        NavigationLink("详情") {
            DetailView()
        }
    }
    .navigationTitle("首页")
    .navigationBarTitleDisplayMode(.large)
    .toolbar {
        ToolbarItem(placement: .navigationBarTrailing) {
            Button("编辑") { }
        }
    }
}
```

**常用修饰符**

| 修饰符 | 作用 | 示例 |
|--------|------|------|
| `.navigationTitle(_:)` | 导航栏标题 | `.navigationTitle("首页")` |
| `.navigationBarTitleDisplayMode(_:)` | 标题模式 | `.large` / `.inline` |
| `.toolbar(_:)` | 工具栏 | `.toolbar { ToolbarItem { ... } }` |
| `.navigationBarBackButtonHidden(_:)` | 隐藏返回按钮 | `.navigationBarBackButtonHidden(true)` |
| `.navigationDestination(for:)` | 路由目标 | `.navigationDestination(for: Route.self) { ... }` |

---

### NavigationSplitView

| 项目 | 说明 |
|------|------|
| 用途 | 分栏导航，iPad/Mac 上显示侧边栏 + 详情 |
| 核心代码 | `NavigationSplitView { sidebar } detail: { detail }` |

```swift
NavigationSplitView {
    List(categories, selection: $selectedCategory) { category in
        Text(category.name).tag(category)
    }
    .navigationTitle("分类")
} detail: {
    if let category = selectedCategory {
        CategoryDetailView(category: category)
    } else {
        Text("请选择分类")
    }
}
```

**常用修饰符**

| 修饰符 | 作用 | 示例 |
|--------|------|------|
| `.navigationSplitViewStyle(_:)` | 分栏样式 | `.balanced` / `.prominentDetail` / `.automatic` |
| `.navigationSplitViewColumnWidth(_:)` | 列宽 | `.navigationSplitViewColumnWidth(250)` |

---

### TabView

| 项目 | 说明 |
|------|------|
| 用途 | 底部标签栏导航，切换不同页面 |
| 核心代码 | `TabView { tabs }` |

```swift
TabView(selection: $selectedTab) {
    HomeView()
        .tabItem {
            Label("首页", systemImage: "house")
        }
        .tag(0)

    SearchView()
        .tabItem {
            Label("搜索", systemImage: "magnifyingglass")
        }
        .tag(1)

    ProfileView()
        .tabItem {
            Label("我的", systemImage: "person")
        }
        .tag(2)
}
.tint(.blue)
```

**常用修饰符**

| 修饰符 | 作用 | 示例 |
|--------|------|------|
| `.tabItem(_:)` | 标签项 | `.tabItem { Label("首页", systemImage: "house") }` |
| `.tint(_:)` | 选中颜色 | `.tint(.orange)` |
| `.tag(_:)` | 标识值 | `.tag(0)` |

---

### Sheet

| 项目 | 说明 |
|------|------|
| 用途 | 模态弹出页面，从底部滑出 |
| 核心代码 | `.sheet(isPresented:) { content }` |

```swift
@State private var showSheet = false

Button("打开") {
    showSheet = true
}
.sheet(isPresented: $showSheet) {
    VStack {
        Text("这是一个 Sheet")
            .font(.title)
        Button("关闭") {
            showSheet = false
        }
        .buttonStyle(.borderedProminent)
    }
    .presentationDetents([.medium, .large])
    .presentationDragIndicator(.visible)
}
```

**常用修饰符**

| 修饰符 | 作用 | 示例 |
|--------|------|------|
| `.presentationDetents(_:)` | 弹出高度 | `.presentationDetents([.medium, .large])` |
| `.presentationDragIndicator(_:)` | 拖拽指示器 | `.presentationDragIndicator(.visible)` |
| `.presentationBackgroundInteraction(_:)` | 背景交互 | `.presentationBackgroundInteraction(.enabled)` |
| `.interactiveDismissDisabled(_:)` | 禁止下拉关闭 | `.interactiveDismissDisabled()` |

---

## 7. 提示组件

### Alert

| 项目 | 说明 |
|------|------|
| 用途 | 弹出警告对话框，提示重要信息 |
| 核心代码 | `.alert("标题", isPresented:) { buttons }` |

```swift
@State private var showAlert = false

Button("显示 Alert") {
    showAlert = true
}
.alert("确认删除？", isPresented: $showAlert) {
    Button("取消", role: .cancel) { }
    Button("删除", role: .destructive) {
        deleteItem()
    }
} message: {
    Text("此操作不可撤销")
}
```

**常用修饰符**

| 修饰符 | 作用 | 示例 |
|--------|------|------|
| `role: .cancel` | 取消按钮 | `Button("取消", role: .cancel) { }` |
| `role: .destructive` | 危险操作按钮 | `Button("删除", role: .destructive) { }` |
| `message:` | 副标题 | `message: { Text("说明文字") }` |

---

### ConfirmationDialog

| 项目 | 说明 |
|------|------|
| 用途 | 确认对话框，从底部弹出操作列表 |
| 核心代码 | `.confirmationDialog("标题", isPresented:) { actions }` |

```swift
@State private var showConfirm = false

Button("操作") {
    showConfirm = true
}
.confirmationDialog("选择操作", isPresented: $showConfirm) {
    Button("复制") { copy() }
    Button("移动") { move() }
    Button("删除", role: .destructive) { delete() }
    Button("取消", role: .cancel) { }
} message: {
    Text("请选择要执行的操作")
}
```

**常用修饰符**

| 修饰符 | 作用 | 示例 |
|--------|------|------|
| `titleVisibility:` | 标题可见性 | `titleVisibility: .hidden` |
| `role: .destructive` | 危险操作 | `Button("删除", role: .destructive)` |
| `role: .cancel` | 取消 | `Button("取消", role: .cancel)` |

---

### ActionSheet

| 项目 | 说明 |
|------|------|
| 用途 | 底部操作表（iOS 15 已废弃，请使用 ConfirmationDialog） |
| 核心代码 | `.actionSheet("标题", isPresented:) { buttons }` |

```swift
// 已废弃，推荐使用 confirmationDialog
.confirmationDialog("操作", isPresented: $showSheet) {
    Button("选项1") { }
    Button("取消", role: .cancel) { }
}
```

> ⚠️ `ActionSheet` 在 iOS 15+ 已废弃，请统一使用 `ConfirmationDialog`。

---

### ProgressView

| 项目 | 说明 |
|------|------|
| 用途 | 显示加载进度（确定/不确定） |
| 核心代码 | `ProgressView()` / `ProgressView(value: progress)` |

```swift
// 不确定进度（转圈）
ProgressView("加载中...")

// 确定进度（进度条）
@State private var progress: Double = 0.6

ProgressView("下载中", value: progress, total: 1.0)
    .tint(.blue)
    .padding()
```

**常用修饰符**

| 修饰符 | 作用 | 示例 |
|--------|------|------|
| `.tint(_:)` | 进度条颜色 | `.tint(.green)` |
| `.progressViewStyle(_:)` | 进度样式 | `.circular` / `.linear` |
| `.controlSize(_:)` | 尺寸 | `.controlSize(.large)` |

---

## 8. 图片组件

### Image

| 项目 | 说明 |
|------|------|
| 用途 | 显示本地图片或系统图标 |
| 核心代码 | `Image("photo")` / `Image(systemName: "star")` |

```swift
// 资源目录图片
Image("cover")
    .resizable()
    .aspectRatio(contentMode: .fit)
    .frame(width: 200, height: 150)
    .clipShape(RoundedRectangle(cornerRadius: 12))

// SF Symbols 系统图标
Image(systemName: "heart.fill")
    .font(.title)
    .foregroundColor(.red)
```

**常用修饰符**

| 修饰符 | 作用 | 示例 |
|--------|------|------|
| `.resizable()` | 允许缩放 | `.resizable()` |
| `.aspectRatio(_:, contentMode:)` | 宽高比 | `.aspectRatio(contentMode: .fit)` / `.fill` |
| `.frame()` | 固定尺寸 | `.frame(width: 100, height: 100)` |
| `.clipShape(_:)` | 裁剪形状 | `.clipShape(Circle())` |
| `.cornerRadius(_:)` | 圆角 | `.cornerRadius(12)` |
| `.shadow(_:)` | 阴影 | `.shadow(color: .gray, radius: 4, x: 0, y: 2)` |
| `.overlay(_:)` | 叠加层 | `.overlay(Circle().stroke(Color.white, lineWidth: 2))` |
| `.renderingMode(_:)` | 渲染模式 | `.renderingMode(.template)` |
| `.interpolation(_:)` | 插值质量 | `.interpolation(.high)` |

---

### AsyncImage

| 项目 | 说明 |
|------|------|
| 用途 | 异步加载网络图片 |
| 核心代码 | `AsyncImage(url: url)` |

```swift
// 基础用法
AsyncImage(url: URL(string: "https://example.com/photo.jpg"))

// 带占位图
AsyncImage(url: URL(string: "https://example.com/photo.jpg")) { phase in
    switch phase {
    case .empty:
        ProgressView()
    case .success(let image):
        image
            .resizable()
            .aspectRatio(contentMode: .fit)
    case .failure:
        Image(systemName: "photo")
            .foregroundColor(.gray)
    @unknown default:
        EmptyView()
    }
}
.frame(width: 200, height: 150)
```

**常用修饰符**

| 修饰符 | 作用 | 示例 |
|--------|------|------|
| `.frame()` | 固定尺寸 | `.frame(width: 200, height: 200)` |
| `.clipShape(_:)` | 裁剪 | `.clipShape(RoundedRectangle(cornerRadius: 12))` |
| `phase` 回调 | 加载状态 | `case .success(let image): image.resizable()` |

---

## 9. 形状组件

### Rectangle

| 项目 | 说明 |
|------|------|
| 用途 | 矩形形状 |
| 核心代码 | `Rectangle()` |

```swift
Rectangle()
    .fill(Color.blue.gradient)
    .frame(width: 200, height: 100)
    .cornerRadius(8)

Rectangle()
    .stroke(Color.red, lineWidth: 2)
    .frame(width: 200, height: 100)
```

**常用修饰符**

| 修饰符 | 作用 | 示例 |
|--------|------|------|
| `.fill(_:)` | 填充颜色 | `.fill(Color.blue)` |
| `.stroke(_:, lineWidth:)` | 描边 | `.stroke(Color.red, lineWidth: 2)` |
| `.frame()` | 尺寸 | `.frame(width: 200, height: 100)` |
| `.cornerRadius(_:)` | 圆角 | `.cornerRadius(12)` |

---

### Circle

| 项目 | 说明 |
|------|------|
| 用途 | 圆形形状 |
| 核心代码 | `Circle()` |

```swift
Circle()
    .fill(Color.orange.gradient)
    .frame(width: 100, height: 100)

Circle()
    .strokeBorder(Color.purple, lineWidth: 3)
    .frame(width: 100, height: 100)
```

**常用修饰符**

| 修饰符 | 作用 | 示例 |
|--------|------|------|
| `.fill(_:)` | 填充 | `.fill(Color.orange)` |
| `.stroke(_:, lineWidth:)` | 描边 | `.stroke(Color.red, lineWidth: 2)` |
| `.strokeBorder(_:, lineWidth:)` | 内描边 | `.strokeBorder(Color.blue, lineWidth: 4)` |
| `.frame()` | 尺寸 | `.frame(width: 80, height: 80)` |

---

### RoundedRectangle

| 项目 | 说明 |
|------|------|
| 用途 | 圆角矩形形状 |
| 核心代码 | `RoundedRectangle(cornerRadius: 12)` |

```swift
RoundedRectangle(cornerRadius: 16)
    .fill(Color.teal.gradient)
    .frame(width: 200, height: 100)

RoundedRectangle(cornerRadius: 16)
    .stroke(Color.teal, style: StrokeStyle(lineWidth: 2, dash: [8]))
    .frame(width: 200, height: 100)
```

**常用修饰符**

| 修饰符 | 作用 | 示例 |
|--------|------|------|
| `cornerRadius` | 圆角半径 | `cornerRadius: 16` |
| `.fill(_:)` | 填充 | `.fill(Color.teal)` |
| `.stroke(_:, style:)` | 描边 | `.stroke(Color.teal, lineWidth: 2)` |
| `.frame()` | 尺寸 | `.frame(width: 200, height: 100)` |

---

### Capsule

| 项目 | 说明 |
|------|------|
| 用途 | 胶囊形状（两端半圆的矩形） |
| 核心代码 | `Capsule()` |

```swift
Capsule()
    .fill(Color.pink.gradient)
    .frame(width: 120, height: 40)
    .overlay(
        Text("标签")
            .font(.caption)
            .foregroundColor(.white)
    )
```

**常用修饰符**

| 修饰符 | 作用 | 示例 |
|--------|------|------|
| `.fill(_:)` | 填充 | `.fill(Color.pink)` |
| `.stroke(_:, lineWidth:)` | 描边 | `.stroke(Color.pink, lineWidth: 2)` |
| `.frame()` | 尺寸 | `.frame(width: 120, height: 40)` |
| `.overlay(_:)` | 叠加内容 | `.overlay(Text("标签"))` |

---

## 10. 图表组件（iOS 16+）

> 使用前需导入：`import Charts`

### Chart

| 项目 | 说明 |
|------|------|
| 用途 | 图表容器，承载各类 Mark 数据标记 |
| 核心代码 | `Chart { marks }` |

```swift
import Charts

struct SalesData: Identifiable {
    let id = UUID()
    let month: String
    let revenue: Double
}

@State private var data = [
    SalesData(month: "1月", revenue: 120),
    SalesData(month: "2月", revenue: 180),
    SalesData(month: "3月", revenue: 150),
    SalesData(month: "4月", revenue: 220)
]

Chart(data) { item in
    BarMark(
        x: .value("月份", item.month),
        y: .value("收入", item.revenue)
    )
    .foregroundStyle(Color.blue.gradient)
}
.frame(height: 250)
.padding()
```

**常用修饰符**

| 修饰符 | 作用 | 示例 |
|--------|------|------|
| `.chartXScale(_:)` | X 轴范围 | `.chartXScale(domain: 0...100)` |
| `.chartYScale(_:)` | Y 轴范围 | `.chartYScale(domain: 0...300)` |
| `.chartXAxis(_:)` | X 轴样式 | `.chartXAxis { AxisMarks() }` |
| `.chartYAxis(_:)` | Y 轴样式 | `.chartYAxis { AxisMarks() }` |
| `.chartLegend(_:)` | 图例 | `.chartLegend(position: .bottom)` |

---

### LineMark

| 项目 | 说明 |
|------|------|
| 用途 | 折线图标记 |
| 核心代码 | `LineMark(x:, y:)` |

```swift
Chart(data) { item in
    LineMark(
        x: .value("月份", item.month),
        y: .value("收入", item.revenue)
    )
    .foregroundStyle(Color.blue)
    .lineStyle(StrokeStyle(lineWidth: 2))

    AreaMark(
        x: .value("月份", item.month),
        y: .value("收入", item.revenue)
    )
    .foregroundStyle(Color.blue.opacity(0.1))
}
.frame(height: 250)
```

**常用修饰符**

| 修饰符 | 作用 | 示例 |
|--------|------|------|
| `.foregroundStyle(_:)` | 线条颜色 | `.foregroundStyle(.blue)` |
| `.lineStyle(_:)` | 线条样式 | `.lineStyle(StrokeStyle(lineWidth: 3))` |
| `.interpolationMethod(_:)` | 插值方式 | `.interpolationMethod(.catmullRom)` |
| `.symbol(_:)` | 数据点形状 | `.symbol(Circle())` |

---

### BarMark

| 项目 | 说明 |
|------|------|
| 用途 | 柱状图标记 |
| 核心代码 | `BarMark(x:, y:)` |

```swift
Chart(data) { item in
    BarMark(
        x: .value("月份", item.month),
        y: .value("收入", item.revenue)
    )
    .foregroundStyle(Color.orange.gradient)
    .cornerRadius(4)
}
.chartYAxis {
    AxisMarks(position: .leading)
}
.frame(height: 250)
```

**常用修饰符**

| 修饰符 | 作用 | 示例 |
|--------|------|------|
| `.foregroundStyle(_:)` | 柱体颜色 | `.foregroundStyle(.orange.gradient)` |
| `.cornerRadius(_:)` | 圆角 | `.cornerRadius(6)` |
| `.position(by:)` | 分组 | `.position(by: .value("类别", item.category))` |
| `.stack(by:)` | 堆叠 | `.stack(by: .value("类型", item.type))` |

---

### PieMark

| 项目 | 说明 |
|------|------|
| 用途 | 饼图标记（iOS 17+） |
| 核心代码 | `SectorMark(angle:, innerRadius:)` |

```swift
struct PieData: Identifiable {
    let id = UUID()
    let category: String
    let value: Double
}

@State private var pieData = [
    PieData(category: "食品", value: 40),
    PieData(category: "交通", value: 25),
    PieData(category: "娱乐", value: 20),
    PieData(category: "其他", value: 15)
]

Chart(pieData) { item in
    SectorMark(
        angle: .value("占比", item.value),
        innerRadius: .ratio(0.5),
        angularInset: 2
    )
    .foregroundStyle(by: .value("类别", item.category))
    .cornerRadius(4)
    .annotation(position: .overlay) {
        Text("\(Int(item.value))%")
            .font(.caption2)
            .foregroundColor(.white)
            .bold()
    }
}
.frame(height: 250)
```

**常用修饰符**

| 修饰符 | 作用 | 示例 |
|--------|------|------|
| `innerRadius:` | 内圆半径（环形图） | `.ratio(0.5)` / `.fixed(size: 40)` |
| `angularInset:` | 扇形间距 | `angularInset: 2` |
| `.foregroundStyle(by:)` | 按数据着色 | `.foregroundStyle(by: .value("类别", item.category))` |
| `.cornerRadius(_:)` | 圆角 | `.cornerRadius(4)` |
| `.annotation(position:)` | 标注 | `.annotation(position: .overlay) { Text(...) }` |

---

## 通用修饰符速查

以下修饰符几乎适用于所有 SwiftUI 组件：

| 修饰符 | 作用 | 示例 |
|--------|------|------|
| `.padding(_:)` | 内边距 | `.padding()` / `.padding(.horizontal, 16)` |
| `.frame()` | 尺寸约束 | `.frame(maxWidth: .infinity, alignment: .leading)` |
| `.background(_:)` | 背景 | `.background(Color(.systemGray6))` |
| `.foregroundColor(_:)` | 前景色 | `.foregroundColor(.primary)` |
| `.font(_:)` | 字体 | `.font(.headline)` |
| `.opacity(_:)` | 透明度 | `.opacity(0.5)` |
| `.hidden()` | 隐藏 | `.hidden()` |
| `.disabled(_:)` | 禁用交互 | `.disabled(true)` |
| `.onAppear(_:)` | 出现回调 | `.onAppear { loadData() }` |
| `.onChange(of:)` | 值变化监听 | `.onChange(of: value) { newValue in ... }` |
| `.accessibilityLabel(_:)` | 无障碍标签 | `.accessibilityLabel("关闭按钮")` |
| `.environment(\.colorScheme,)` | 颜色模式 | `.environment(\.colorScheme, .dark)` |

---

> 💡 **提示**：SwiftUI 组件通过修饰符链式调用实现样式定制，修饰符顺序有时会影响最终效果。建议多在 Xcode Preview 中实时调试。
