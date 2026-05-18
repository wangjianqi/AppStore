# 附录 C：SwiftUI 速查表

> SwiftUI 组件和 API 太多记不住？本附录将最常用的组件、修饰符、状态管理和布局容器整理成速查表，开发时随时翻阅。

---

## 常用组件

### Text — 文本展示

**基本语法**：

```swift
Text("Hello, World!")
Text("价格：¥\(price)")
Text(attributedString)
```

**常用修饰符**：

| 修饰符 | 功能 | 示例 |
|--------|------|------|
| `.font()` | 设置字体 | `.font(.title)` / `.font(.system(size: 16, weight: .bold))` |
| `.foregroundColor()` | 设置文字颜色 | `.foregroundColor(.blue)` |
| `.multilineTextAlignment()` | 多行对齐 | `.multilineTextAlignment(.center)` |
| `.lineLimit()` | 限制行数 | `.lineLimit(2)` |
| `.truncationMode()` | 截断模式 | `.truncationMode(.tail)` |
| `.kerning()` | 字间距 | `.kerning(1.5)` |
| `.tracking()` | 字符追踪 | `.tracking(2)` |
| `.strikethrough()` | 删除线 | `.strikethrough(true, color: .red)` |
| `.underline()` | 下划线 | `.underline(true, color: .blue)` |
| `.baselineOffset()` | 基线偏移 | `.baselineOffset(5)` |

---

### Image — 图片展示

**基本语法**：

```swift
Image("photo")                // 加载 Asset Catalog 中的图片
Image(systemName: "star.fill") // 加载 SF Symbols 图标
Image(uiImage: image)          // 从 UIImage 创建
```

**常用修饰符**：

| 修饰符 | 功能 | 示例 |
|--------|------|------|
| `.resizable()` | 允许调整大小 | `.resizable()` |
| `.aspectRatio()` | 保持宽高比 | `.aspectRatio(contentMode: .fit)` |
| `.frame()` | 设置尺寸 | `.frame(width: 100, height: 100)` |
| `.clipShape()` | 裁剪形状 | `.clipShape(Circle())` / `.clipShape(RoundedRectangle(cornerRadius: 10))` |
| `.scaledToFit()` | 等比缩放适应 | `.scaledToFit()` |
| `.scaledToFill()` | 等比缩放填充 | `.scaledToFill()` |
| `.renderingMode()` | 渲染模式 | `.renderingMode(.template)` 配合 `.foregroundColor()` 改变图标颜色 |
| `.interpolation()` | 插值质量 | `.interpolation(.high)` |
| `.symbolVariant()` | SF Symbol 变体 | `.symbolVariant(.fill)` |
| `.symbolRenderingMode()` | SF Symbol 渲染 | `.symbolRenderingMode(.multicolor)` |

---

### Button — 按钮

**基本语法**：

```swift
Button("点击我") {
    print("按钮被点击")
}

Button {
    print("按钮被点击")
} label: {
    HStack {
        Image(systemName: "plus")
        Text("添加")
    }
}
```

**常用修饰符**：

| 修饰符 | 功能 | 示例 |
|--------|------|------|
| `.buttonStyle()` | 按钮样式 | `.buttonStyle(.borderedProminent)` / `.buttonStyle(.bordered)` / `.buttonStyle(.plain)` |
| `.tint()` | 按钮颜色 | `.tint(.blue)` |
| `.disabled()` | 禁用按钮 | `.disabled(true)` |
| `.controlSize()` | 控件大小 | `.controlSize(.large)` / `.controlSize(.small)` / `.controlSize(.mini)` |

**按钮样式对比**：

| 样式 | 效果 | 适用场景 |
|------|------|---------|
| `.borderedProminent` | 填充背景色 | 主要操作 |
| `.bordered` | 仅边框 | 次要操作 |
| `.plain` | 无装饰 | 文字链接 |
| `.borderless` | 无边框 | 列表中的按钮 |

---

### TextField — 文本输入

**基本语法**：

```swift
TextField("请输入用户名", text: $username)
SecureField("请输入密码", text: $password)  // 密码输入框
```

**常用修饰符**：

| 修饰符 | 功能 | 示例 |
|--------|------|------|
| `.textFieldStyle()` | 输入框样式 | `.textFieldStyle(.roundedBorder)` / `.textFieldStyle(.plain)` |
| `.keyboardType()` | 键盘类型 | `.keyboardType(.numberPad)` / `.keyboardType(.emailAddress)` |
| `.autocapitalization()` | 自动大写 | `.autocapitalization(.none)` |
| `.disableAutocorrection()` | 禁用自动纠错 | `.disableAutocorrection(true)` |
| `.onSubmit()` | 提交回调 | `.onSubmit { search() }` |
| `.focused()` | 焦点控制 | `.focused($isFocused)` |
| `.submitLabel()` | 回车键文字 | `.submitLabel(.search)` / `.submitLabel(.done)` |

**键盘类型**：

| 类型 | 说明 |
|------|------|
| `.default` | 默认键盘 |
| `.numberPad` | 数字键盘 |
| `.decimalPad` | 小数键盘 |
| `.emailAddress` | 邮箱键盘 |
| `.phonePad` | 电话键盘 |
| `.URL` | URL 键盘 |

---

### Toggle — 开关

**基本语法**：

```swift
Toggle("飞行模式", isOn: $isAirplaneMode)
```

**常用修饰符**：

| 修饰符 | 功能 | 示例 |
|--------|------|------|
| `.tint()` | 开关颜色 | `.tint(.green)` |
| `.toggleStyle()` | 开关样式 | `.toggleStyle(.switch)` / `.toggleStyle(.button)` |
| `.labelsHidden()` | 隐藏标签 | `.labelsHidden()` |

---

### Slider — 滑块

**基本语法**：

```swift
Slider(value: $volume, in: 0...100, step: 1) {
    Text("音量")
} minimumValueLabel: {
    Image(systemName: "speaker.fill")
} maximumValueLabel: {
    Image(systemName: "speaker.wave.3.fill")
}
```

**常用修饰符**：

| 修饰符 | 功能 | 示例 |
|--------|------|------|
| `.tint()` | 滑块颜色 | `.tint(.blue)` |
| `.onEditingChanged()` | 拖动状态回调 | `.onEditingChanged { isEditing in }` |

---

### Picker — 选择器

**基本语法**：

```swift
Picker("选择颜色", selection: $selectedColor) {
    Text("红色").tag("red")
    Text("蓝色").tag("blue")
    Text("绿色").tag("green")
}
```

**常用样式**：

| 样式 | 效果 | 示例 |
|------|------|------|
| `.pickerStyle(.wheel)` | 滚轮选择器 | 日期选择等 |
| `.pickerStyle(.segmented)` | 分段选择器 | 少量选项切换 |
| `.pickerStyle(.menu)` | 下拉菜单 | 中等数量选项 |
| `.pickerStyle(.navigationLink)` | 导航选择 | 大量选项 |

**枚举 + Picker 最佳实践**：

```swift
enum SortOrder: String, CaseIterable {
    case date = "按日期"
    case amount = "按金额"
    case name = "按名称"
}

Picker("排序", selection: $sortOrder) {
    ForEach(SortOrder.allCases, id: \.self) { order in
        Text(order.rawValue).tag(order)
    }
}
.pickerStyle(.segmented)
```

---

### List — 列表

**基本语法**：

```swift
List {
    Text("第一行")
    Text("第二行")
    Text("第三行")
}

List(items) { item in
    HStack {
        Text(item.name)
        Spacer()
        Text("¥\(item.amount)")
    }
}
```

**常用修饰符**：

| 修饰符 | 功能 | 示例 |
|--------|------|------|
| `.listStyle()` | 列表样式 | `.listStyle(.insetGrouped)` |
| `.searchable()` | 搜索功能 | `.searchable(text: $searchText)` |
| `.refreshable()` | 下拉刷新 | `.refreshable { await loadData() }` |
| `.swipeActions()` | 滑动操作 | `.swipeActions { Button("删除", role: .destructive) { } }` |
| `.onDelete()` | 删除操作 | `.onDelete { indexSet in }` |
| `.onMove()` | 移动操作 | `.onMove { from, to in }` |

**列表样式对比**：

| 样式 | 效果 | 适用场景 |
|------|------|---------|
| `.insetGrouped` | 圆角分组 | iOS 默认设置页风格 |
| `.grouped` | 直角分组 | 无圆角的分组列表 |
| `.plain` | 无分组 | 简单列表 |
| `.sidebar` | 侧边栏 | iPad 多栏布局 |

---

### ScrollView — 滚动视图

**基本语法**：

```swift
ScrollView {
    VStack {
        ForEach(0..<50) { i in
            Text("第 \(i) 行")
        }
    }
}

ScrollView(.horizontal, showsIndicators: false) {
    HStack {
        ForEach(items) { item in
            ItemCard(item: item)
        }
    }
}
```

**常用修饰符**：

| 修饰符 | 功能 | 示例 |
|--------|------|------|
| `.scrollDirection()` | 滚动方向 | `.scrollDirection(.horizontal)` (iOS 17+) |
| `.scrollIndicators()` | 滚动条显示 | `.scrollIndicators(.hidden)` |
| `.scrollPosition()` | 滚动位置控制 | `.scrollPosition(id: $scrollToId)` (iOS 17+) |
| `.scrollTargetBehavior()` | 滚动吸附 | `.scrollTargetBehavior(.paging)` (iOS 17+) |
| `.scrollClipDisabled()` | 允许溢出 | `.scrollClipDisabled()` (iOS 17+) |

---

### NavigationStack — 导航

**基本语法**：

```swift
NavigationStack {
    List {
        NavigationLink("详情页") {
            DetailView()
        }
    }
    .navigationTitle("首页")
}
```

**常用修饰符**：

| 修饰符 | 功能 | 示例 |
|--------|------|------|
| `.navigationTitle()` | 页面标题 | `.navigationTitle("首页")` |
| `.navigationBarTitleDisplayMode()` | 标题显示模式 | `.navigationBarTitleDisplayMode(.inline)` / `.navigationBarTitleDisplayMode(.large)` |
| `.toolbar()` | 工具栏 | `.toolbar { Button("编辑") { } }` |
| `.toolbarBackground()` | 导航栏背景 | `.toolbarBackground(.visible, for: .navigationBar)` |
| `.navigationBarBackButtonHidden()` | 隐藏返回按钮 | `.navigationBarBackButtonHidden(true)` |
| `.navigationDestination()` | 导航目标 | `.navigationDestination(for: Route.self) { route in }` |

---

## 常用 Modifier

### 字体与颜色

| 修饰符 | 功能 | 示例 |
|--------|------|------|
| `.font()` | 字体 | `.font(.title)` / `.font(.system(size: 16, weight: .medium, design: .rounded))` |
| `.foregroundColor()` | 前景色 | `.foregroundColor(.blue)` |
| `.background()` | 背景色 | `.background(Color.red)` |
| `.tint()` | 主题色 | `.tint(.blue)` |
| `.opacity()` | 透明度 | `.opacity(0.5)` |
| `.preferredColorScheme()` | 颜色方案 | `.preferredColorScheme(.dark)` |

**系统字体预设**：

| 预设 | 大小 | 用途 |
|------|------|------|
| `.largeTitle` | 34pt | 页面大标题 |
| `.title` | 28pt | 区块标题 |
| `.title2` | 22pt | 二级标题 |
| `.title3` | 20pt | 三级标题 |
| `.headline` | 17pt semibold | 列表项标题 |
| `.body` | 17pt | 正文 |
| `.callout` | 16pt | 强调文字 |
| `.subheadline` | 15pt | 副标题 |
| `.footnote` | 13pt | 脚注 |
| `.caption` | 12pt | 说明文字 |
| `.caption2` | 11pt | 小号说明 |

---

### 间距与尺寸

| 修饰符 | 功能 | 示例 |
|--------|------|------|
| `.padding()` | 内边距 | `.padding()` / `.padding(16)` / `.padding(.horizontal, 16)` |
| `.frame()` | 尺寸 | `.frame(width: 200, height: 100)` / `.frame(maxWidth: .infinity)` |
| `.fixedSize()` | 固定原始尺寸 | `.fixedSize()` / `.fixedSize(horizontal: true, vertical: false)` |
| `.layoutPriority()` | 布局优先级 | `.layoutPriority(1)` |
| `.offset()` | 偏移 | `.offset(x: 10, y: 20)` |
| `.position()` | 绝对定位 | `.position(x: 100, y: 200)` |

**padding 方向**：

| 方向 | 说明 |
|------|------|
| `.all` | 四周（默认） |
| `.horizontal` | 左右 |
| `.vertical` | 上下 |
| `.top` / `.bottom` / `.leading` / `.trailing` | 单方向 |

---

### 背景与边框

| 修饰符 | 功能 | 示例 |
|--------|------|------|
| `.background()` | 背景 | `.background(Color.blue)` / `.background(Rectangle().fill(.blue))` |
| `.overlay()` | 覆盖层 | `.overlay(RoundedRectangle(cornerRadius: 8).stroke(.gray, lineWidth: 1))` |
| `.clipShape()` | 裁剪形状 | `.clipShape(RoundedRectangle(cornerRadius: 10))` |
| `.shadow()` | 阴影 | `.shadow(color: .black.opacity(0.2), radius: 5, x: 0, y: 2)` |
| `.border()` | 边框 | `.border(.gray, width: 1)` |
| `.cornerRadius()` | 圆角 | `.cornerRadius(10)` |
| `.containerRelativeFrame()` | 相对容器尺寸 | `.containerRelativeFrame(.horizontal, count: 2, span: 1, spacing: 10)` (iOS 17+) |

**圆角矩形背景常用写法**：

```swift
Text("标签")
    .padding(.horizontal, 12)
    .padding(.vertical, 6)
    .background(.blue)
    .foregroundColor(.white)
    .clipShape(RoundedRectangle(cornerRadius: 8))
```

---

### 动画

| 修饰符 | 功能 | 示例 |
|--------|------|------|
| `.animation()` | 隐式动画 | `.animation(.easeInOut, value: isExpanded)` |
| `.withAnimation()` | 显式动画 | `withAnimation(.spring()) { isExpanded.toggle() }` |
| `.transition()` | 过渡效果 | `.transition(.slide)` / `.transition(.opacity)` |
| `.matchedGeometryEffect()` | 共享元素动画 | `.matchedGeometryEffect(id: "img", in: namespace)` |

**常用动画曲线**：

| 曲线 | 效果 | 适用场景 |
|------|------|---------|
| `.easeInOut` | 先慢后快再慢 | 通用 |
| `.easeIn` | 先慢后快 | 开始时缓慢 |
| `.easeOut` | 先快后慢 | 结束时减速 |
| `.linear` | 匀速 | 持续动画 |
| `.spring()` | 弹簧效果 | 交互反馈 |
| `.spring(duration: 0.3, bounce: 0.5)` | 自定义弹簧 | 精细控制 |
| `.interactiveSpring()` | 交互弹簧 | 跟随手势 |

**动画示例**：

```swift
@State var isExpanded = false

VStack {
    Text("内容")
        .frame(maxHeight: isExpanded ? .none : 0)
        .opacity(isExpanded ? 1 : 0)
}
.animation(.spring(duration: 0.3), value: isExpanded)
```

---

### 手势

| 手势 | 功能 | 示例 |
|------|------|------|
| `.onTapGesture()` | 点击 | `.onTapGesture { print("点击") }` |
| `.onTapGesture(count: 2)` | 双击 | `.onTapGesture(count: 2) { }` |
| `.onLongPressGesture()` | 长按 | `.onLongPressGesture { print("长按") }` |
| `.dragGesture()` | 拖拽 | `.gesture(DragGesture().onChanged { value in })` |
| `.magnificationGesture()` | 缩放 | `.gesture(MagnificationGesture().onChanged { scale in })` |
| `.rotationGesture()` | 旋转 | `.gesture(RotationGesture().onChanged { angle in })` |

**手势组合**：

```swift
.simultaneously(with: TapGesture().onEnded { })
.exclusively(before: LongPressGesture())
```

---

### 条件显示

| 方式 | 功能 | 示例 |
|------|------|------|
| `if` 条件 | 条件渲染 | `if isLoggedIn { HomeView() } else { LoginView() }` |
| `.opacity()` | 条件透明 | `.opacity(isHidden ? 0 : 1)` |
| `.hidden()` | 隐藏视图 | `.hidden(isLoading)` (iOS 16+) |
| `.disabled()` | 禁用交互 | `.disabled(!isEnabled)` |
| `.allowsHitTesting()` | 禁止点击 | `.allowsHitTesting(false)` |
| `@ViewBuilder` | 条件返回视图 | 用于函数/闭包中返回不同视图 |

---

## 状态属性包装器

### @State

| 项目 | 说明 |
|------|------|
| **用途** | 在视图中持有和管理简单的值类型状态 |
| **适用场景** | 简单的本地状态，如开关、输入框文本、选中索引 |
| **数据类型** | 值类型（Int、String、Bool、结构体等） |
| **数据所有权** | 视图拥有数据 |
| **线程** | 必须在主线程访问 |

```swift
struct CounterView: View {
    @State private var count = 0

    var body: some View {
        VStack {
            Text("计数：\(count)")
            Button("+1") { count += 1 }
        }
    }
}
```

---

### @Binding

| 项目 | 说明 |
|------|------|
| **用途** | 在子视图中引用和修改父视图的状态 |
| **适用场景** | 需要将状态传递给子视图修改 |
| **数据类型** | 任何 @State 或 @Binding 的值 |
| **数据所有权** | 不拥有数据，只是引用 |

```swift
struct ToggleView: View {
    @Binding var isOn: Bool

    var body: some View {
        Toggle("开关", isOn: $isOn)
    }
}

struct ParentView: View {
    @State var isOn = false

    var body: some View {
        ToggleView(isOn: $isOn)
    }
}
```

---

### @StateObject

| 项目 | 说明 |
|------|------|
| **用途** | 在视图中创建并持有 ObservableObject |
| **适用场景** | 视图自己创建 ViewModel |
| **数据所有权** | 视图拥有对象 |
| **生命周期** | 视图重建时不会重新创建 |

```swift
class MyViewModel: ObservableObject {
    @Published var items: [String] = []
}

struct MyView: View {
    @StateObject private var viewModel = MyViewModel()

    var body: some View {
        List(viewModel.items, id: \.self) { item in
            Text(item)
        }
    }
}
```

---

### @ObservedObject

| 项目 | 说明 |
|------|------|
| **用途** | 在视图中引用外部传入的 ObservableObject |
| **适用场景** | ViewModel 由父视图创建并传入 |
| **数据所有权** | 不拥有对象 |
| **生命周期** | 视图重建时不会保留 |

```swift
struct ChildView: View {
    @ObservedObject var viewModel: MyViewModel

    var body: some View {
        List(viewModel.items, id: \.self) { item in
            Text(item)
        }
    }
}
```

> 💡 **@StateObject vs @ObservedObject**：自己创建用 `@StateObject`，外部传入用 `@ObservedObject`。

---

### @EnvironmentObject

| 项目 | 说明 |
|------|------|
| **用途** | 从环境中获取共享的 ObservableObject |
| **适用场景** | 全局共享的数据（用户信息、主题设置等） |
| **数据所有权** | 不拥有，从祖先视图获取 |
| **注入方式** | 在祖先视图使用 `.environmentObject()` |

```swift
class AppState: ObservableObject {
    @Published var isLoggedIn = false
    @Published var userName = ""
}

@main
struct MyApp: App {
    @StateObject var appState = AppState()

    var body: some Scene {
        WindowGroup {
            ContentView()
                .environmentObject(appState)
        }
    }
}

struct ProfileView: View {
    @EnvironmentObject var appState: AppState

    var body: some View {
        Text("用户：\(appState.userName)")
    }
}
```

---

### @Observable

| 项目 | 说明 |
|------|------|
| **用途** | iOS 17+ 新宏，替代 ObservableObject + @Published |
| **适用场景** | 新项目推荐使用 |
| **数据所有权** | 配合 @State 使用时由视图拥有 |
| **优势** | 代码更少，性能更好（自动追踪依赖） |

```swift
@Observable
class MyViewModel {
    var items: [String] = []
    var isLoading = false

    func loadData() async {
        isLoading = true
        // 加载数据...
        isLoading = false
    }
}

struct MyView: View {
    @State private var viewModel = MyViewModel()

    var body: some View {
        List(viewModel.items, id: \.self) { item in
            Text(item)
        }
        .overlay {
            if viewModel.isLoading {
                ProgressView()
            }
        }
    }
}
```

> 💡 **iOS 17+ 推荐方案**：`@Observable` + `@State` 替代 `ObservableObject` + `@StateObject`。

---

### @AppStorage

| 项目 | 说明 |
|------|------|
| **用途** | 读写 UserDefaults 的属性包装器 |
| **适用场景** | 简单的持久化设置（主题、语言、首次启动等） |
| **数据类型** | Int、Double、String、Bool、URL、Data |
| **存储位置** | UserDefaults |

```swift
struct SettingsView: View {
    @AppStorage("isDarkMode") var isDarkMode = false
    @AppStorage("userName") var userName = ""
    @AppStorage("launchCount") var launchCount = 0

    var body: some View {
        Form {
            Toggle("深色模式", isOn: $isDarkMode)
            TextField("用户名", text: $userName)
            Text("已启动 \(launchCount) 次")
        }
    }
}
```

---

### @Environment

| 项目 | 说明 |
|------|------|
| **用途** | 读取 SwiftUI 环境值 |
| **适用场景** | 获取系统级信息（颜色方案、字体大小、时区等） |
| **数据所有权** | 只读，由系统提供 |

```swift
struct MyView: View {
    @Environment(\.colorScheme) var colorScheme
    @Environment(\.dismiss) var dismiss
    @Environment(\.locale) var locale
    @Environment(\.horizontalSizeClass) var sizeClass

    var body: some View {
        VStack {
            Text(colorScheme == .dark ? "深色模式" : "浅色模式")
            Button("关闭") { dismiss() }
        }
    }
}
```

**常用环境值**：

| 环境值 | 类型 | 说明 |
|--------|------|------|
| `\.colorScheme` | ColorScheme | 当前颜色方案 |
| `\.dismiss` | DismissAction | 关闭当前视图 |
| `\.locale` | Locale | 当前语言区域 |
| `\.horizontalSizeClass` | UserInterfaceSizeClass | 水平尺寸类别 |
| `\.verticalSizeClass` | UserInterfaceSizeClass | 垂直尺寸类别 |
| `\.scenePhase` | ScenePhase | 场景状态 |
| `\.networkPath` | NWPath | 网络状态 |
| `\.openURL` | OpenURLAction | 打开 URL |
| `\.timeZone` | TimeZone | 时区 |
| `\.calendar` | Calendar | 日历 |

---

### @FocusState

| 项目 | 说明 |
|------|------|
| **用途** | 控制输入框的焦点状态 |
| **适用场景** | 自动弹出键盘、切换输入框焦点 |
| **数据类型** | Bool 或枚举 |

```swift
struct LoginView: View {
    @FocusState private var isUsernameFocused: Bool
    @FocusState private var focusedField: Field?

    enum Field {
        case username, password
    }

    var body: some View {
        Form {
            TextField("用户名", text: $username)
                .focused($isUsernameFocused)

            SecureField("密码", text: $password)
                .focused($focusedField, equals: .password)
        }
        .onAppear {
            isUsernameFocused = true
        }
    }
}
```

---

### 状态属性包装器速查对比

| 包装器 | 用途 | 拥有数据 | iOS 版本 | 推荐场景 |
|--------|------|---------|---------|---------|
| `@State` | 简单本地状态 | ✅ | iOS 13+ | 开关、计数器 |
| `@Binding` | 传递状态给子视图 | ❌ | iOS 13+ | 子视图修改父状态 |
| `@StateObject` | 创建 ObservableObject | ✅ | iOS 14+ | 旧方式创建 VM |
| `@ObservedObject` | 引用外部 ObservableObject | ❌ | iOS 13+ | 旧方式传入 VM |
| `@EnvironmentObject` | 从环境获取共享对象 | ❌ | iOS 13+ | 全局共享数据 |
| `@Observable` + `@State` | 新方式创建 VM | ✅ | iOS 17+ | **新项目推荐** |
| `@AppStorage` | UserDefaults 读写 | ✅ | iOS 14+ | 简单设置存储 |
| `@Environment` | 读取环境值 | ❌ | iOS 13+ | 系统级信息 |
| `@FocusState` | 控制焦点 | ✅ | iOS 15+ | 输入框焦点 |

---

## 布局容器

### VStack

垂直方向排列子视图。

```swift
VStack(alignment: .leading, spacing: 10) {
    Text("第一行")
    Text("第二行")
    Text("第三行")
}
```

| 参数 | 说明 | 可选值 |
|------|------|--------|
| `alignment` | 水平对齐 | `.leading` / `.center` / `.trailing` |
| `spacing` | 子视图间距 | 数值或 `nil`（默认间距） |

---

### HStack

水平方向排列子视图。

```swift
HStack(alignment: .center, spacing: 16) {
    Image(systemName: "star.fill")
    Text("收藏")
    Spacer()
    Text("128")
}
```

| 参数 | 说明 | 可选值 |
|------|------|--------|
| `alignment` | 垂直对齐 | `.top` / `.center` / `.bottom` / `.firstTextBaseline` |
| `spacing` | 子视图间距 | 数值或 `nil`（默认间距） |

---

### ZStack

沿 Z 轴（深度方向）叠加子视图，后面的视图覆盖前面的。

```swift
ZStack(alignment: .bottomTrailing) {
    Image("background")
        .resizable()
        .ignoresSafeArea()

    VStack {
        Text("标题")
        Text("副标题")
    }
    .padding()
    .background(.ultraThinMaterial)
}
```

| 参数 | 说明 | 可选值 |
|------|------|--------|
| `alignment` | 对齐方式 | `.center` / `.topLeading` / `.bottomTrailing` 等 |

---

### ScrollView

可滚动容器，支持垂直和水平滚动。

```swift
ScrollView([.vertical], showsIndicators: true) {
    LazyVStack(spacing: 16) {
        ForEach(items) { item in
            ItemRow(item: item)
        }
    }
}
```

| 参数 | 说明 | 可选值 |
|------|------|--------|
| `axes` | 滚动方向 | `.vertical` / `.horizontal` / `[.vertical, .horizontal]` |
| `showsIndicators` | 显示滚动条 | `true` / `false` |

---

### LazyVStack / LazyHStack

懒加载容器，只渲染可见区域的子视图，适合大量数据。

```swift
ScrollView {
    LazyVStack(spacing: 12) {
        ForEach(0..<1000) { i in
            Text("第 \(i) 行")
                .padding()
        }
    }
}
```

| 对比 | VStack/HStack | LazyVStack/LazyHStack |
|------|---------------|----------------------|
| 渲染方式 | 一次性渲染所有子视图 | 只渲染可见区域的子视图 |
| 性能 | 子视图多时性能差 | 子视图多时性能好 |
| 适用场景 | 少量子视图 | 大量数据列表 |
| 注意事项 | 无 | 必须放在 ScrollView 中 |

---

### Grid

iOS 16+ 新增的网格布局容器。

```swift
Grid {
    GridRow {
        Text("姓名")
        Text("年龄")
        Text("城市")
    }
    GridRow {
        Text("张三")
        Text("25")
        Text("北京")
    }
}
```

**自适应网格**：

```swift
LazyVGrid(columns: [
    GridItem(.flexible()),
    GridItem(.flexible()),
    GridItem(.flexible()),
]) {
    ForEach(items) { item in
        ItemCell(item: item)
    }
}
```

**GridItem 类型**：

| 类型 | 说明 | 示例 |
|------|------|------|
| `.flexible()` | 弹性宽度，平分空间 | `GridItem(.flexible())` |
| `.adaptive(minimum:)` | 自适应列数 | `GridItem(.adaptive(minimum: 100))` |
| `.fixed(_)` | 固定宽度 | `GridItem(.fixed(80))` |

---

### 布局容器对比

| 容器 | 方向 | 懒加载 | 适用场景 |
|------|------|--------|---------|
| `VStack` | 垂直 | ❌ | 少量子视图垂直排列 |
| `HStack` | 水平 | ❌ | 少量子视图水平排列 |
| `ZStack` | 叠加 | ❌ | 视图叠加（背景+内容） |
| `ScrollView` | 可滚动 | ❌ | 可滚动内容 |
| `LazyVStack` | 垂直 | ✅ | 大量数据垂直列表 |
| `LazyHStack` | 水平 | ✅ | 大量数据水平列表 |
| `Grid` | 网格 | ❌ | 表格/对齐网格 |
| `LazyVGrid` | 垂直网格 | ✅ | 瀑布流/相册 |
| `LazyHGrid` | 水平网格 | ✅ | 水平滚动网格 |

---

## 导航

### NavigationStack

iOS 16+ 推荐的导航容器，替代 NavigationView。

```swift
NavigationStack {
    List {
        NavigationLink("详情") {
            DetailView()
        }
    }
    .navigationTitle("首页")
    .navigationBarTitleDisplayMode(.large)
}
```

**基于路径的导航**（适合复杂导航场景）：

```swift
NavigationStack(path: $path) {
    HomeView()
        .navigationDestination(for: Route.self) { route in
            switch route {
            case .detail(let id):
                DetailView(id: id)
            case .settings:
                SettingsView()
            }
        }
}
```

---

### NavigationLink

导航跳转按钮。

```swift
// 简单跳转（iOS 16+）
NavigationLink("跳转到详情") {
    DetailView()
}

// 带 value 的跳转（配合 navigationDestination 使用）
NavigationLink(value: Route.detail(id: 1)) {
    HStack {
        Image(systemName: "arrow.right")
        Text("查看详情")
    }
}
```

| 写法 | iOS 版本 | 说明 |
|------|---------|------|
| `NavigationLink("标题") { 目标视图 }` | iOS 16+ | 直接指定目标视图 |
| `NavigationLink(value: 路由值) { 标签 }` | iOS 16+ | 配合 navigationDestination |
| `NavigationLink("标题", destination: 目标视图)` | iOS 13+ | 旧写法（已废弃） |

---

### Sheet

模态弹出页面（从底部滑出）。

```swift
.sheet(isPresented: $showSheet) {
    DetailView()
}

.sheet(item: $selectedItem) { item in
    ItemDetailView(item: item)
}
```

| 参数 | 说明 |
|------|------|
| `isPresented` | 绑定 Bool 控制显示/隐藏 |
| `item` | 绑定可选值，非 nil 时显示 |
| `onDismiss` | 关闭时的回调 |
| `detents` | 控制弹出高度（iOS 16+） |

**自定义弹出高度**（iOS 16+）：

```swift
.sheet(isPresented: $showSheet) {
    DetailView()
        .presentationDetents([.medium, .large])
        .presentationDragIndicator(.visible)
}
```

---

### FullScreenCover

全屏模态弹出页面。

```swift
.fullScreenCover(isPresented: $showFullScreen) {
    VideoPlayerView()
}
```

| 对比 | Sheet | FullScreenCover |
|------|-------|-----------------|
| 显示方式 | 从底部滑出，可自定义高度 | 全屏覆盖 |
| 可否下拉关闭 | 默认可以 | 默认不可以 |
| 适用场景 | 表单、详情、分享 | 视频、登录、引导页 |
| 背景遮罩 | 半透明 | 完全遮挡 |

---

### TabView

底部标签栏导航。

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
```

**iOS 18+ 新写法**：

```swift
TabView {
    Tab("首页", systemImage: "house") {
        HomeView()
    }
    Tab("搜索", systemImage: "magnifyingglass") {
        SearchView()
    }
    Tab("我的", systemImage: "person") {
        ProfileView()
    }
}
```

**常用修饰符**：

| 修饰符 | 功能 | 示例 |
|--------|------|------|
| `.tabItem` | 标签项 | `.tabItem { Label("首页", systemImage: "house") }` |
| `.tag` | 标签标识 | `.tag(0)` |
| `.toolbarBackground()` | 标签栏背景 | `.toolbarBackground(.visible, for: .tabBar)` |
| `.tint()` | 选中颜色 | `.tint(.blue)` |

---

### 导航方式对比

| 导航方式 | 显示效果 | 关闭方式 | 适用场景 |
|----------|---------|---------|---------|
| NavigationStack | 推入堆栈 | 返回按钮/滑动手势 | 层级导航 |
| Sheet | 底部弹出 | 下拉/按钮关闭 | 临时任务 |
| FullScreenCover | 全屏覆盖 | 按钮关闭 | 沉浸式体验 |
| TabView | 底部标签切换 | 切换标签 | 主功能分区 |
| NavigationSplitView | 分栏导航 | 选择不同栏目 | iPad/macOS |
