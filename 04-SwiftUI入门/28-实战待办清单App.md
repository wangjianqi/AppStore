# 28-实战①：完成「待办清单」App

> 🎯 **本章目标**：综合运用前面学到的 SwiftUI 知识（@State、@Binding、List、NavigationStack、Sheet），从零开始完成一个完整的「待办清单」App，掌握数据模型设计、列表增删改、数据持久化等核心开发技能，学会用 AI 辅助开发。

---

## 项目概述

这是你学习 SwiftUI 以来的**第一个完整实战项目**！我们将从零开始，一步步构建一个真正能用的待办清单 App。

### 功能清单

| 功能 | 说明 | 涉及知识点 |
|------|------|-----------|
| 添加待办事项 | 通过弹窗输入标题，添加到列表 | Sheet、TextField、@State |
| 标记完成/未完成 | 点击切换完成状态，样式随之变化 | @Binding、条件渲染 |
| 删除待办事项 | 滑动删除，带确认提示 | .onDelete、alert |
| 数据持久化保存 | 关闭 App 后数据不丢失 | UserDefaults、Codable |

### 最终效果预览

完成后的 App 长这样：

```
┌─────────────────────────────┐
│  📝 待办清单          [➕]   │  ← 导航栏，右上角添加按钮
├─────────────────────────────┤
│                             │
│  ✅ 学习 SwiftUI            │  ← 已完成：灰色 + 删除线
│                             │
│  ○ 买菜                     │  ← 未完成：正常显示
│                             │
│  ○ 写周报                   │
│                             │
│  ○ 锻炼身体                 │
│                             │
├─────────────────────────────┤
│  共 4 项，已完成 1 项        │  ← 底部统计栏
└─────────────────────────────┘

点击 ➕ 后弹出添加界面：
┌─────────────────────────────┐
│  添加待办              [✕]  │
├─────────────────────────────┤
│                             │
│  ┌─────────────────────┐   │
│  │ 输入待办事项...       │   │  ← TextField 输入框
│  └─────────────────────┘   │
│                             │
│       [ 保存 ]              │  ← 保存按钮
│                             │
└─────────────────────────────┘
```

---

## 创建项目

### 新建 SwiftUI 项目

1. 打开 Xcode → 点击 **"Create a new Xcode project"**
2. 选择 **iOS → App** → 点击 Next
3. 填写项目信息：

| 字段 | 填写内容 | 说明 |
|------|---------|------|
| Product Name | `MyTodo` | 项目名称 |
| Team | 选择你的开发者账号 | 没有就选 None |
| Organization Identifier | `com.yourname` | 组织标识符 |
| Interface | **SwiftUI** | 必须选 SwiftUI |
| Language | **Swift** | 必须选 Swift |
| Storage | None | 暂时不需要 Core Data |

4. 点击 Next → 选择保存位置 → Create

> 💡 **提示**：项目创建后，Xcode 会自动生成一个 `ContentView.swift` 文件，这就是我们的主视图。我们会在它的基础上进行修改。

---

## 数据模型设计

在开发 App 时，**先设计数据模型**是一个好习惯。数据模型决定了你的 App 如何存储和管理数据。

### TodoItem 结构体

我们需要一个 `TodoItem` 来表示每一条待办事项：

```swift
struct TodoItem: Identifiable, Codable {
    let id: UUID
    var title: String
    var isCompleted: Bool
}
```

逐行解读：

| 代码 | 含义 |
|------|------|
| `struct TodoItem` | 定义一个结构体，表示一条待办事项 |
| `Identifiable` | 协议，要求有唯一的 `id` 属性，让 ForEach 能遍历 |
| `Codable` | 协议，让数据可以编码成 JSON / 从 JSON 解码，用于持久化 |
| `let id: UUID` | 唯一标识符，每条待办事项都有不同的 id |
| `var title: String` | 待办事项的标题，用 var 因为可能需要修改 |
| `var isCompleted: Bool` | 是否已完成，用 var 因为需要切换状态 |

### 为什么需要 Identifiable？

```swift
// ❌ 没有 Identifiable，ForEach 需要手动指定 id
ForEach(items, id: \.id) { item in ... }

// ✅ 有 Identifiable，ForEach 可以省略 id 参数
ForEach(items) { item in ... }
```

### 为什么需要 Codable？

```swift
// 编码：TodoItem → JSON 数据（保存到 UserDefaults）
let data = try JSONEncoder().encode(todoItem)

// 解码：JSON 数据 → TodoItem（从 UserDefaults 读取）
let item = try JSONDecoder().decode(TodoItem.self, from: data)
```

> 💡 **提示**：`UUID()` 会自动生成一个全球唯一的 ID，比如 `550e8400-e29b-41d4-a716-446655440000`。这样即使两条待办事项的标题相同，它们的 id 也不同，SwiftUI 就能正确区分它们。

---

## 主界面开发

### 待办列表

首先实现待办事项的列表展示：

```swift
import SwiftUI

struct ContentView: View {
    @State private var todos: [TodoItem] = [
        TodoItem(id: UUID(), title: "学习 SwiftUI", isCompleted: true),
        TodoItem(id: UUID(), title: "买菜", isCompleted: false),
        TodoItem(id: UUID(), title: "写周报", isCompleted: false),
        TodoItem(id: UUID(), title: "锻炼身体", isCompleted: false)
    ]

    var body: some View {
        NavigationStack {
            List($todos) { $todo in
                TodoRowView(todo: $todo)
            }
            .navigationTitle("待办清单")
        }
    }
}

struct TodoRowView: View {
    @Binding var todo: TodoItem

    var body: some View {
        HStack(spacing: 12) {
            Image(systemName: todo.isCompleted ? "checkmark.circle.fill" : "circle")
                .foregroundColor(todo.isCompleted ? .green : .gray)
                .font(.title2)
                .onTapGesture {
                    withAnimation {
                        todo.isCompleted.toggle()
                    }
                }

            Text(todo.title)
                .strikethrough(todo.isCompleted)
                .foregroundColor(todo.isCompleted ? .gray : .primary)
        }
        .padding(.vertical, 4)
    }
}

struct TodoItem: Identifiable, Codable {
    let id: UUID
    var title: String
    var isCompleted: Bool
}

#Preview {
    ContentView()
}
```

逐行解读关键代码：

| 代码 | 含义 |
|------|------|
| `@State private var todos: [TodoItem]` | 用 @State 管理待办列表数据 |
| `List($todos) { $todo in }` | 用 `$todos` 创建绑定，让每行可以修改数据 |
| `@Binding var todo: TodoItem` | 行视图通过绑定直接修改原始数据 |
| `.onTapGesture { }` | 点击手势，点击图标切换完成状态 |
| `withAnimation { }` | 切换时添加动画效果 |
| `.strikethrough()` | 删除线效果 |

> 💡 **提示**：`List($todos) { $todo in }` 是 SwiftUI 的语法糖——`$todos` 产生 `Binding<[TodoItem]>`，遍历时每个 `$todo` 是 `Binding<TodoItem>`，这样子视图就能直接修改父视图的数据。

### 添加待办

接下来实现"添加待办事项"功能。点击导航栏右上角的 ➕ 按钮，弹出一个 Sheet 来输入新待办：

```swift
import SwiftUI

struct ContentView: View {
    @State private var todos: [TodoItem] = [
        TodoItem(id: UUID(), title: "学习 SwiftUI", isCompleted: true),
        TodoItem(id: UUID(), title: "买菜", isCompleted: false),
        TodoItem(id: UUID(), title: "写周报", isCompleted: false),
        TodoItem(id: UUID(), title: "锻炼身体", isCompleted: false)
    ]

    @State private var showAddSheet = false

    var body: some View {
        NavigationStack {
            List($todos) { $todo in
                TodoRowView(todo: $todo)
            }
            .navigationTitle("待办清单")
            .toolbar {
                ToolbarItem(placement: .topBarTrailing) {
                    Button {
                        showAddSheet = true
                    } label: {
                        Image(systemName: "plus")
                    }
                }
            }
            .sheet(isPresented: $showAddSheet) {
                AddTodoView { newTitle in
                    let newTodo = TodoItem(
                        id: UUID(),
                        title: newTitle,
                        isCompleted: false
                    )
                    todos.append(newTodo)
                }
            }
        }
    }
}

struct AddTodoView: View {
    @State private var newTitle = ""
    let onSave: (String) -> Void
    @Environment(\.dismiss) private var dismiss

    var body: some View {
        NavigationStack {
            VStack(spacing: 20) {
                TextField("输入待办事项...", text: $newTitle)
                    .textFieldStyle(.roundedBorder)
                    .padding(.horizontal)

                Button("保存") {
                    let trimmed = newTitle.trimmingCharacters(in: .whitespaces)
                    guard !trimmed.isEmpty else { return }
                    onSave(trimmed)
                    dismiss()
                }
                .buttonStyle(.borderedProminent)
                .disabled(newTitle.trimmingCharacters(in: .whitespaces).isEmpty)

                Spacer()
            }
            .padding(.top, 20)
            .navigationTitle("添加待办")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .topBarLeading) {
                    Button("取消") {
                        dismiss()
                    }
                }
            }
        }
    }
}

struct TodoRowView: View {
    @Binding var todo: TodoItem

    var body: some View {
        HStack(spacing: 12) {
            Image(systemName: todo.isCompleted ? "checkmark.circle.fill" : "circle")
                .foregroundColor(todo.isCompleted ? .green : .gray)
                .font(.title2)
                .onTapGesture {
                    withAnimation {
                        todo.isCompleted.toggle()
                    }
                }

            Text(todo.title)
                .strikethrough(todo.isCompleted)
                .foregroundColor(todo.isCompleted ? .gray : .primary)
        }
        .padding(.vertical, 4)
    }
}

struct TodoItem: Identifiable, Codable {
    let id: UUID
    var title: String
    var isCompleted: Bool
}

#Preview {
    ContentView()
}
```

添加功能的关键点：

| 关键代码 | 说明 |
|---------|------|
| `@State private var showAddSheet = false` | 控制 Sheet 的显示/隐藏 |
| `.sheet(isPresented: $showAddSheet)` | 绑定布尔值，true 时弹出 Sheet |
| `let onSave: (String) -> Void` | 闭包回调，子视图通过它把数据传回父视图 |
| `@Environment(\.dismiss)` | 环境变量，用于关闭当前 Sheet |
| `.trimmingCharacters(in: .whitespaces)` | 去除输入内容首尾的空格 |
| `guard !trimmed.isEmpty else { return }` | 防止添加空白待办 |

> 💡 **提示**：`AddTodoView` 使用闭包回调而不是 @Binding 来传递数据。这是因为添加操作是"一次性"的——用户输入标题后点保存，数据传回主视图，Sheet 关闭。闭包比 Binding 更适合这种场景。

### 删除待办

使用 `.onDelete` 实现滑动删除：

```swift
List($todos) { $todo in
    TodoRowView(todo: $todo)
}
.onDelete { indexSet in
    todos.remove(atOffsets: indexSet)
}
```

就这么简单！用户在列表行上向左滑动，就会出现红色的"删除"按钮。

⚠️ **警告**：`.onDelete` 必须加在 `List` 或 `ForEach` 上。如果使用 `List($todos)` 的方式，直接加在 List 上即可。

---

## 标记完成功能

### 点击切换完成状态

我们已经在 `TodoRowView` 中实现了点击切换：

```swift
Image(systemName: todo.isCompleted ? "checkmark.circle.fill" : "circle")
    .foregroundColor(todo.isCompleted ? .green : .gray)
    .font(.title2)
    .onTapGesture {
        withAnimation {
            todo.isCompleted.toggle()
        }
    }
```

### 完成样式

当待办事项被标记为完成后，视觉上应该有明显区分：

| 状态 | 图标 | 文字样式 | 颜色 |
|------|------|---------|------|
| 未完成 | ○ 空心圆圈 | 正常文字 | 主色（黑色/白色） |
| 已完成 | ✅ 实心对勾 | 删除线 | 灰色 |

### 动画效果

`withAnimation` 让状态切换有平滑的过渡动画：

```swift
// 有动画：图标和文字平滑过渡
withAnimation {
    todo.isCompleted.toggle()
}

// 无动画：瞬间切换，体验生硬
todo.isCompleted.toggle()
```

> 💡 **提示**：`withAnimation` 默认使用 `.easeInOut` 动画。你也可以指定动画类型：`withAnimation(.spring()) { ... }` 使用弹簧动画，效果更有弹性。

---

## 数据持久化（UserDefaults）

目前我们的待办数据只存在内存中，关闭 App 后数据就丢失了。下面用 **UserDefaults** 实现数据持久化。

### 编码和解码（Codable）

`Codable` 是 Swift 提供的编解码协议，让数据可以在"Swift 对象"和"JSON 数据"之间互相转换：

```swift
// 编码：Swift 对象 → JSON 数据
let todos = [TodoItem(id: UUID(), title: "买菜", isCompleted: false)]
let data = try JSONEncoder().encode(todos)
// data 是 Data 类型，可以保存到 UserDefaults

// 解码：JSON 数据 → Swift 对象
let decodedTodos = try JSONDecoder().decode([TodoItem].self, from: data)
// decodedTodos 是 [TodoItem] 类型，和编码前一模一样
```

数据转换流程：

```
[TodoItem]  ──JSONEncoder──▶  Data  ──UserDefaults──▶  磁盘
   ↑                                              │
   └─────────────JSONDecoder─────── Data ◀─────────┘
```

### 保存数据

创建一个专门管理数据持久化的类：

```swift
class TodoStore {
    private let saveKey = "saved_todos"

    func save(_ todos: [TodoItem]) {
        do {
            let data = try JSONEncoder().encode(todos)
            UserDefaults.standard.set(data, forKey: saveKey)
        } catch {
            print("保存失败：\(error)")
        }
    }

    func load() -> [TodoItem] {
        do {
            if let data = UserDefaults.standard.data(forKey: saveKey) {
                return try JSONDecoder().decode([TodoItem].self, from: data)
            }
        } catch {
            print("加载失败：\(error)")
        }
        return []
    }
}
```

### 加载数据

在主视图中使用 `TodoStore`：

```swift
@State private var todos: [TodoItem] = {
    let store = TodoStore()
    return store.load()
}()
```

> 💡 **提示**：`{ ... }()` 是闭包立即执行语法，相当于先创建 TodoStore，再调用 load()，把返回值作为 @State 的初始值。

### 数据变化时自动保存

使用 `.onChange` 修饰符，在数据变化时自动保存：

```swift
List($todos) { $todo in
    TodoRowView(todo: $todo)
}
.onChange(of: todos) {
    TodoStore().save(todos)
}
```

⚠️ **警告**：`onChange` 在每次数据变化时都会触发，包括添加、删除、标记完成。这确保了所有操作都会被保存。但如果你有非常大量的数据，频繁保存可能影响性能，这时可以考虑使用"防抖"（debounce）策略。

---

## 用 AI 辅助开发实战

在实际开发中，AI 是你的得力助手。下面展示如何用 AI 来辅助开发待办清单 App。

### 让 AI 帮你生成初始代码

当你不知道从何开始时，可以让 AI 帮你生成初始代码框架：

> **Prompt 示例**：
>
> 请用 SwiftUI 帮我创建一个待办清单 App，要求：
> 1. 使用 List 展示待办事项列表
> 2. 每条待办有标题和完成状态
> 3. 点击可以切换完成状态
> 4. 右上角有添加按钮，点击弹出 Sheet 添加新待办
> 5. 支持滑动删除
> 6. 数据用 UserDefaults 持久化
>
> 请给出完整可运行的代码。

AI 会生成一个完整的代码框架，你可以在此基础上修改和完善。

### 让 AI 帮你优化 UI

当你觉得界面不够好看时，可以让 AI 帮你优化：

> **Prompt 示例**：
>
> 我的待办清单 App 界面太朴素了，请帮我优化 UI：
> 1. 每行待办加上圆角卡片样式
> 2. 已完成的待办加上渐变删除线效果
> 3. 底部添加统计信息（共几项、已完成几项）
> 4. 添加空状态提示（没有待办时显示插图和文字）
> 5. 整体配色更现代、更美观
>
> 这是我的当前代码：
> ```swift
> // 粘贴你的代码
> ```

### 让 AI 帮你添加功能

当你想给 App 添加新功能时，可以让 AI 帮你实现：

> **Prompt 示例**：
>
> 请帮我的待办清单 App 添加以下功能：
> 1. 待办事项支持设置优先级（高/中/低），不同优先级显示不同颜色标记
> 2. 支持拖拽排序
> 3. 添加搜索功能，可以按标题搜索待办
> 4. 支持分类（工作/生活/学习），可以按分类筛选
>
> 这是我的当前代码：
> ```swift
> // 粘贴你的代码
> ```

> 💡 **提示**：使用 AI 辅助开发时，**一定要理解 AI 生成的代码**，而不是直接复制粘贴。你可以逐行阅读，不懂的地方追问 AI，这样你才能真正学到知识。

---

## 完整代码

下面是整个待办清单 App 的完整代码，可以直接复制到 Xcode 运行：

```swift
import SwiftUI

struct TodoItem: Identifiable, Codable {
    let id: UUID
    var title: String
    var isCompleted: Bool
}

class TodoStore {
    private let saveKey = "saved_todos"

    func save(_ todos: [TodoItem]) {
        do {
            let data = try JSONEncoder().encode(todos)
            UserDefaults.standard.set(data, forKey: saveKey)
        } catch {
            print("保存失败：\(error)")
        }
    }

    func load() -> [TodoItem] {
        do {
            if let data = UserDefaults.standard.data(forKey: saveKey) {
                return try JSONDecoder().decode([TodoItem].self, from: data)
            }
        } catch {
            print("加载失败：\(error)")
        }
        return []
    }
}

struct ContentView: View {
    @State private var todos: [TodoItem] = TodoStore().load()
    @State private var showAddSheet = false

    private var completedCount: Int {
        todos.filter { $0.isCompleted }.count
    }

    var body: some View {
        NavigationStack {
            ZStack(alignment: .bottom) {
                List {
                    ForEach($todos) { $todo in
                        TodoRowView(todo: $todo)
                    }
                    .onDelete { indexSet in
                        todos.remove(atOffsets: indexSet)
                    }
                }

                if !todos.isEmpty {
                    HStack {
                        Text("共 \(todos.count) 项")
                            .foregroundColor(.secondary)
                        Spacer()
                        Text("已完成 \(completedCount) 项")
                            .foregroundColor(.green)
                    }
                    .font(.subheadline)
                    .padding()
                    .background(
                        RoundedRectangle(cornerRadius: 12)
                            .fill(.ultraThinMaterial)
                    )
                    .padding(.horizontal)
                    .padding(.bottom, 8)
                }
            }
            .navigationTitle("待办清单")
            .toolbar {
                ToolbarItem(placement: .topBarTrailing) {
                    Button {
                        showAddSheet = true
                    } label: {
                        Image(systemName: "plus")
                    }
                }
            }
            .sheet(isPresented: $showAddSheet) {
                AddTodoView { newTitle in
                    let newTodo = TodoItem(
                        id: UUID(),
                        title: newTitle,
                        isCompleted: false
                    )
                    todos.append(newTodo)
                }
            }
            .onChange(of: todos) {
                TodoStore().save(todos)
            }
            .overlay {
                if todos.isEmpty {
                    ContentUnavailableView(
                        "暂无待办",
                        systemImage: "checklist",
                        description: Text("点击右上角 ➕ 添加你的第一条待办事项")
                    )
                }
            }
        }
    }
}

struct TodoRowView: View {
    @Binding var todo: TodoItem

    var body: some View {
        HStack(spacing: 14) {
            Button {
                withAnimation(.spring(response: 0.3)) {
                    todo.isCompleted.toggle()
                }
            } label: {
                Image(systemName: todo.isCompleted ? "checkmark.circle.fill" : "circle")
                    .foregroundColor(todo.isCompleted ? .green : .gray)
                    .font(.title2)
            }

            Text(todo.title)
                .font(.body)
                .strikethrough(todo.isCompleted, color: .gray)
                .foregroundColor(todo.isCompleted ? .gray : .primary)

            Spacer()

            if todo.isCompleted {
                Text("已完成")
                    .font(.caption)
                    .foregroundColor(.green)
                    .padding(.horizontal, 8)
                    .padding(.vertical, 4)
                    .background(
                        Capsule()
                            .fill(Color.green.opacity(0.1))
                    )
            }
        }
        .padding(.vertical, 6)
    }
}

struct AddTodoView: View {
    @State private var newTitle = ""
    @FocusState private var isFocused: Bool
    let onSave: (String) -> Void
    @Environment(\.dismiss) private var dismiss

    var body: some View {
        NavigationStack {
            VStack(spacing: 24) {
                VStack(alignment: .leading, spacing: 8) {
                    Text("待办内容")
                        .font(.headline)

                    TextField("例如：买牛奶", text: $newTitle)
                        .textFieldStyle(.roundedBorder)
                        .focused($isFocused)
                        .onSubmit {
                            saveTodo()
                        }
                }
                .padding(.horizontal)

                Button {
                    saveTodo()
                } label: {
                    HStack {
                        Image(systemName: "plus.circle.fill")
                        Text("添加待办")
                    }
                    .frame(maxWidth: .infinity)
                }
                .buttonStyle(.borderedProminent)
                .disabled(newTitle.trimmingCharacters(in: .whitespaces).isEmpty)
                .padding(.horizontal)

                Spacer()
            }
            .padding(.top, 24)
            .navigationTitle("添加待办")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .topBarLeading) {
                    Button("取消") {
                        dismiss()
                    }
                }
            }
            .onAppear {
                isFocused = true
            }
        }
    }

    private func saveTodo() {
        let trimmed = newTitle.trimmingCharacters(in: .whitespaces)
        guard !trimmed.isEmpty else { return }
        onSave(trimmed)
        dismiss()
    }
}

#Preview {
    ContentView()
}
```

### 代码结构总览

| 组件 | 职责 | 关键知识点 |
|------|------|-----------|
| `TodoItem` | 数据模型 | Identifiable、Codable、UUID |
| `TodoStore` | 数据持久化 | JSONEncoder/Decoder、UserDefaults |
| `ContentView` | 主界面 | @State、List、Sheet、onChange、toolbar |
| `TodoRowView` | 列表行 | @Binding、withAnimation、条件渲染 |
| `AddTodoView` | 添加弹窗 | 闭包回调、@FocusState、@Environment |

### 功能实现对照表

| 功能 | 实现方式 | 代码位置 |
|------|---------|---------|
| 展示列表 | `List + ForEach` | ContentView |
| 添加待办 | `Sheet + TextField + 闭包回调` | AddTodoView |
| 标记完成 | `@Binding + onTapGesture + withAnimation` | TodoRowView |
| 删除待办 | `.onDelete` | ContentView |
| 数据保存 | `JSONEncoder + UserDefaults` | TodoStore |
| 数据加载 | `JSONDecoder + UserDefaults` | TodoStore |
| 自动保存 | `.onChange(of: todos)` | ContentView |
| 空状态 | `ContentUnavailableView` | ContentView |
| 统计信息 | 计算属性 + 底部悬浮栏 | ContentView |
| 自动聚焦 | `@FocusState` | AddTodoView |

---

## 小结

恭喜你！🎉 你已经完成了第一个完整的 SwiftUI App！

本章我们学到的核心知识：

| 知识点 | 核心要点 |
|-------|---------|
| **数据模型设计** | struct + Identifiable + Codable，先设计数据再写界面 |
| **List + ForEach** | 动态列表展示，`$todos` 创建绑定让子视图可修改 |
| **Sheet 弹窗** | `isPresented` 控制显示，闭包回调传递数据 |
| **@Binding** | 父子视图双向通信，子视图直接修改父视图数据 |
| **滑动删除** | `.onDelete` 修饰符，接收 IndexSet |
| **withAnimation** | 状态变化时添加动画，提升用户体验 |
| **UserDefaults** | 轻量级数据持久化，配合 Codable 编解码 |
| **Codable** | JSON 编解码协议，让自定义类型可以序列化 |
| **.onChange** | 监听数据变化，自动触发保存 |
| **ContentUnavailableView** | 空状态展示，系统内置组件 |
| **@FocusState** | 控制输入框焦点，Sheet 打开时自动聚焦 |

**开发流程总结**：

```
1. 设计数据模型（TodoItem）
2. 搭建主界面（List + NavigationStack）
3. 实现核心功能（添加、删除、标记完成）
4. 添加数据持久化（UserDefaults + Codable）
5. 优化体验（动画、空状态、统计信息）
6. 测试和完善
```

**下一步可以尝试的扩展功能**：

- 🔸 添加待办事项的优先级（高/中/低）
- 🔸 支持编辑待办标题
- 🔸 添加分类/标签功能
- 🔸 支持搜索和筛选
- 🔸 添加到期日期提醒

👉 **下一章**：[22-更复杂的状态管理](./22-更复杂的状态管理.md)

**上一步** ← [20-状态管理](./20-状态管理.md)
