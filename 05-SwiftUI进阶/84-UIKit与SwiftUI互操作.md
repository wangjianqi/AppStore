# 84-UIKit 与 SwiftUI 互操作

> 🎯 **本章目标**：理解为什么现实项目中几乎必然需要 UIKit 与 SwiftUI 互操作，掌握 UIViewRepresentable 和 UIViewControllerRepresentable 两大桥接协议，学会在 UIKit 中嵌入 SwiftUI 视图，深入理解 Coordinator 模式处理 delegate 回调，掌握跨框架数据双向绑定与手势传递，避开常见坑并建立最佳实践，最终实战将 Sign in with Apple 包装为 SwiftUI 组件。

---

## 1. 为什么需要 UIKit 与 SwiftUI 互操作

💡 **通俗理解**：SwiftUI 和 UIKit 就像两座城市，中间有一条河隔开。你的项目可能住在"SwiftUI 新城"，但有些重要的资源（第三方 SDK、成熟组件）还在"UIKit 老城"。互操作就是在这两座城市之间修桥——让你可以自由穿梭，享受两边的好处。

### 1.1 现实项目中的必然需求

即使你从零开始一个 SwiftUI 项目，也几乎不可避免地会遇到需要 UIKit 的场景：

| 场景 | 说明 | 示例 |
|------|------|------|
| **第三方 SDK** | 大量 SDK 只提供 UIKit 接口 | 微信分享、支付宝支付、Google Maps |
| **成熟 UIKit 组件** | 某些功能 SwiftUI 尚未覆盖 | UITextView 富文本、MKMapView 地图、UITextView 附件键盘 |
| **遗留代码迁移** | 已有 UIKit 项目逐步迁移到 SwiftUI | 大型项目不可能一次性重写 |
| **系统控制器** | 部分系统控制器没有 SwiftUI 原生封装 | UIImagePickerController、ASAuthorizationController |
| **精细控制** | 需要对视图生命周期做精细控制 | 自定义文本输入响应链、焦点管理 |

### 1.2 互操作的两个方向

| 方向 | 协议/工具 | 说明 |
|------|----------|------|
| **UIKit → SwiftUI** | `UIViewRepresentable` / `UIViewControllerRepresentable` | 把 UIKit 视图/控制器包装进 SwiftUI |
| **SwiftUI → UIKit** | `UIHostingController` | 把 SwiftUI 视图嵌入 UIKit 项目 |

> 💡 **提示**：Apple 的设计哲学是"SwiftUI 优先，UIKit 兜底"。SwiftUI 能做的尽量用 SwiftUI，做不到的用互操作桥接 UIKit。

---

## 2. UIViewRepresentable 协议详解

`UIViewRepresentable` 是把 UIKit 的 `UIView` 包装成 SwiftUI 视图的桥接协议。

💡 **通俗理解**：UIViewRepresentable 就像一个"翻译官"——SwiftUI 说"我需要一个视图"，翻译官就把 UIKit 的 UIView 翻译成 SwiftUI 能理解的形式。SwiftUI 不需要知道 UIView 的细节，只需要通过翻译官和它交流。

### 2.1 协议核心方法

```swift
protocol UIViewRepresentable {
    associatedtype UIViewType: UIView

    func makeUIView(context: Context) -> UIViewType
    func updateUIView(_ uiView: UIViewType, context: Context)

    static func dismantleUIView(_ uiView: UIViewType, coordinator: Coordinator)
    func makeCoordinator() -> Coordinator
}
```

| 方法 | 调用时机 | 职责 |
|------|---------|------|
| `makeUIView(context:)` | 视图首次创建时 | 创建并返回 UIKit 视图，做一次性配置 |
| `updateUIView(_:context:)` | SwiftUI 状态变化时 | 将 SwiftUI 的最新状态同步到 UIKit 视图 |
| `dismantleUIView(_:coordinator:)` | 视图被移除时 | 清理资源、移除观察者（可选） |
| `makeCoordinator()` | 在 `makeUIView` 之前调用 | 创建 Coordinator，处理 delegate 等回调 |

### 2.2 生命周期流程

```
SwiftUI 创建视图
    │
    ▼
makeCoordinator()          ← 创建协调器
    │
    ▼
makeUIView(context:)       ← 创建 UIKit 视图
    │
    ▼
updateUIView(_:context:)   ← SwiftUI 状态变化时反复调用
    │
    ▼
dismantleUIView(_:coordinator:)  ← 视图被移除时调用
```

> ⚠️ **警告**：`makeUIView` 只调用一次，`updateUIView` 可能调用多次。不要在 `updateUIView` 中做昂贵的操作（如重新创建视图），应该只更新变化的属性。

### 2.3 实战：包装 UITextField

SwiftUI 的 `TextField` 功能有限，比如不支持自定义光标颜色、键盘上方添加工具栏等。通过包装 `UITextField` 可以获得完整控制：

```swift
import SwiftUI
import UIKit

struct AdvancedTextField: UIViewRepresentable {
    @Binding var text: String
    var placeholder: String
    var font: UIFont = .systemFont(ofSize: 17)
    var textColor: UIColor = .label
    var tintColor: UIColor = .systemBlue
    var keyboardType: UIKeyboardType = .default
    var autocorrectionType: UITextAutocorrectionType = .default
    var onCommit: (() -> Void)?

    func makeUIView(context: Context) -> UITextField {
        let textField = UITextField()
        textField.delegate = context.coordinator
        textField.font = font
        textField.textColor = textColor
        textField.tintColor = tintColor
        textField.keyboardType = keyboardType
        textField.autocorrectionType = autocorrectionType
        textField.placeholder = placeholder
        textField.addTarget(
            context.coordinator,
            action: #selector(Coordinator.textDidChange),
            for: .editingChanged
        )

        let toolbar = UIToolbar()
        toolbar.items = [
            UIBarButtonItem(
                barButtonSystemItem: .flexibleSpace,
                target: nil,
                action: nil
            ),
            UIBarButtonItem(
                title: "完成",
                style: .done,
                target: context.coordinator,
                action: #selector(Coordinator.commit)
            )
        ]
        toolbar.sizeToFit()
        textField.inputAccessoryView = toolbar

        return textField
    }

    func updateUIView(_ uiView: UITextField, context: Context) {
        if uiView.text != text {
            uiView.text = text
        }
        uiView.placeholder = placeholder
    }

    func makeCoordinator() -> Coordinator {
        Coordinator(text: $text, onCommit: onCommit)
    }

    class Coordinator: NSObject, UITextFieldDelegate {
        var text: Binding<String>
        var onCommit: (() -> Void)?

        init(text: Binding<String>, onCommit: (() -> Void)?) {
            self.text = text
            self.onCommit = onCommit
        }

        @objc func textDidChange(_ textField: UITextField) {
            text.wrappedValue = textField.text ?? ""
        }

        @objc func commit() {
            onCommit?()
            UIApplication.shared.sendAction(
                #selector(UIResponder.resignFirstResponder),
                to: nil,
                from: nil,
                for: nil
            )
        }

        func textFieldShouldReturn(_ textField: UITextField) -> Bool {
            commit()
            return true
        }
    }
}
```

使用方式：

```swift
struct AdvancedTextFieldDemo: View {
    @State private var username = ""
    @State private var email = ""

    var body: some View {
        NavigationStack {
            Form {
                Section("用户信息") {
                    AdvancedTextField(
                        text: $username,
                        placeholder: "输入用户名",
                        tintColor: .systemBlue
                    )
                    .frame(height: 44)

                    AdvancedTextField(
                        text: $email,
                        placeholder: "输入邮箱",
                        keyboardType: .emailAddress,
                        autocorrectionType: .no
                    )
                    .frame(height: 44)
                }

                Section("预览") {
                    Text("用户名：\(username)")
                    Text("邮箱：\(email)")
                }
            }
            .navigationTitle("高级文本框")
        }
    }
}
```

### 2.4 实战：包装 UITextView

SwiftUI 的 `TextEditor` 不支持占位符、自定义行高等，包装 `UITextView` 可以解决这些问题：

```swift
import SwiftUI
import UIKit

struct PlaceholderTextView: UIViewRepresentable {
    @Binding var text: String
    var placeholder: String = ""
    var font: UIFont = .systemFont(ofSize: 17)
    var maxHeight: CGFloat = 200

    func makeUIView(context: Context) -> UITextView {
        let textView = UITextView()
        textView.delegate = context.coordinator
        textView.font = font
        textView.isScrollEnabled = true
        textView.isEditable = true
        textView.isUserInteractionEnabled = true
        textView.backgroundColor = .clear
        textView.textContainerInset = UIEdgeInsets(top: 8, left: 4, bottom: 8, right: 4)

        return textView
    }

    func updateUIView(_ uiView: UITextView, context: Context) {
        if uiView.text != text {
            uiView.text = text
        }
        uiView.font = font
        updatePlaceholder(uiView)
    }

    private func updatePlaceholder(_ textView: UITextView) {
        if text.isEmpty {
            textView.text = placeholder
            textView.textColor = .placeholderText
        } else {
            textView.textColor = .label
        }
    }

    func makeCoordinator() -> Coordinator {
        Coordinator(text: $text, placeholder: placeholder)
    }

    class Coordinator: NSObject, UITextViewDelegate {
        var text: Binding<String>
        let placeholder: String

        init(text: Binding<String>, placeholder: String) {
            self.text = text
            self.placeholder = placeholder
        }

        func textViewDidChange(_ textView: UITextView) {
            text.wrappedValue = textView.text ?? ""
        }

        func textViewDidBeginEditing(_ textView: UITextView) {
            if textView.text == placeholder {
                textView.text = ""
                textView.textColor = .label
            }
        }

        func textViewDidEndEditing(_ textView: UITextView) {
            if textView.text.isEmpty {
                textView.text = placeholder
                textView.textColor = .placeholderText
            }
        }
    }
}
```

### 2.5 实战：包装 MKMapView

MapKit 的 `MKMapView` 是典型的"SwiftUI 没有原生替代"的组件：

```swift
import SwiftUI
import MapKit

struct MapView: UIViewRepresentable {
    var latitude: Double
    var longitude: Double
    var title: String?
    var subtitle: String?
    var span: Double = 0.01

    func makeUIView(context: Context) -> MKMapView {
        let mapView = MKMapView()
        mapView.delegate = context.coordinator
        return mapView
    }

    func updateUIView(_ uiView: MKMapView, context: Context) {
        let coordinate = CLLocationCoordinate2D(latitude: latitude, longitude: longitude)
        let region = MKCoordinateRegion(
            center: coordinate,
            span: MKCoordinateSpan(latitudeDelta: span, longitudeDelta: span)
        )
        uiView.setRegion(region, animated: true)

        let annotation = MKPointAnnotation()
        annotation.coordinate = coordinate
        annotation.title = title
        annotation.subtitle = subtitle
        uiView.removeAnnotations(uiView.annotations)
        uiView.addAnnotation(annotation)
    }

    func makeCoordinator() -> Coordinator {
        Coordinator()
    }

    class Coordinator: NSObject, MKMapViewDelegate {
        func mapView(_ mapView: MKMapView, viewFor annotation: MKAnnotation) -> MKAnnotationView? {
            let identifier = "pin"
            var view = mapView.dequeueReusableAnnotationView(withIdentifier: identifier) as? MKMarkerAnnotationView
            if view == nil {
                view = MKMarkerAnnotationView(annotation: annotation, reuseIdentifier: identifier)
                view?.canShowCallout = true
            } else {
                view?.annotation = annotation
            }
            return view
        }
    }
}
```

使用方式：

```swift
struct MapViewDemo: View {
    var body: some View {
        NavigationStack {
            MapView(
                latitude: 39.9042,
                longitude: 116.4074,
                title: "北京天安门",
                subtitle: "中国北京",
                span: 0.05
            )
            .ignoresSafeArea()
            .navigationTitle("地图")
            .navigationBarTitleDisplayMode(.inline)
        }
    }
}
```

---

## 3. UIViewControllerRepresentable 协议详解

`UIViewControllerRepresentable` 用于将 UIKit 的 `UIViewController` 包装进 SwiftUI，适用于需要完整控制器生命周期的场景。

💡 **通俗理解**：如果 UIViewRepresentable 是"翻译一个视图"，那 UIViewControllerRepresentable 就是"翻译一个完整的页面"——包含视图层级、导航逻辑、转场动画等。

### 3.1 协议核心方法

```swift
protocol UIViewControllerRepresentable {
    associatedtype UIViewControllerType: UIViewController

    func makeUIViewController(context: Context) -> UIViewControllerType
    func updateUIViewController(
        _ uiViewController: UIViewControllerType,
        context: Context
    )

    static func dismantleUIViewController(
        _ uiViewController: UIViewControllerType,
        coordinator: Coordinator
    )
    func makeCoordinator() -> Coordinator
}
```

| 方法 | 调用时机 | 职责 |
|------|---------|------|
| `makeUIViewController(context:)` | 首次创建时 | 创建并返回 UIKit 控制器 |
| `updateUIViewController(_:context:)` | 状态变化时 | 将 SwiftUI 状态同步到控制器 |
| `dismantleUIViewController(_:coordinator:)` | 移除时 | 清理资源 |
| `makeCoordinator()` | 创建前 | 创建 Coordinator 处理回调 |

### 3.2 实战：包装 UIImagePickerController

SwiftUI 的 `PhotosPicker` 虽然存在，但 `UIImagePickerController` 提供了更多控制（相机、相册切换等）：

```swift
import SwiftUI
import UIKit

struct ImagePicker: UIViewControllerRepresentable {
    @Binding var selectedImage: UIImage?
    @Environment(\.dismiss) private var dismiss
    var sourceType: UIImagePickerController.SourceType = .photoLibrary

    func makeUIViewController(context: Context) -> UIImagePickerController {
        let picker = UIImagePickerController()
        picker.sourceType = sourceType
        picker.delegate = context.coordinator
        return picker
    }

    func updateUIViewController(
        _ uiViewController: UIImagePickerController,
        context: Context
    ) {}

    func makeCoordinator() -> Coordinator {
        Coordinator(selectedImage: $selectedImage, dismiss: dismiss)
    }

    class Coordinator: NSObject, UIImagePickerControllerDelegate, UINavigationControllerDelegate {
        @Binding var selectedImage: UIImage?
        let dismiss: DismissAction

        init(selectedImage: Binding<UIImage?>, dismiss: DismissAction) {
            self._selectedImage = selectedImage
            self.dismiss = dismiss
        }

        func imagePickerController(
            _ picker: UIImagePickerController,
            didFinishPickingMediaWithInfo info: [UIImagePickerController.InfoKey: Any]
        ) {
            selectedImage = info[.originalImage] as? UIImage
            dismiss()
        }

        func imagePickerControllerDidCancel(_ picker: UIImagePickerController) {
            dismiss()
        }
    }
}
```

使用方式：

```swift
struct ImagePickerDemo: View {
    @State private var selectedImage: UIImage?
    @State private var showPicker = false
    @State private var showCamera = false

    var body: some View {
        NavigationStack {
            VStack(spacing: 24) {
                if let image = selectedImage {
                    Image(uiImage: image)
                        .resizable()
                        .scaledToFit()
                        .frame(maxHeight: 300)
                        .clipShape(RoundedRectangle(cornerRadius: 16))
                } else {
                    RoundedRectangle(cornerRadius: 16)
                        .fill(Color.gray.opacity(0.1))
                        .frame(height: 200)
                        .overlay {
                            VStack(spacing: 8) {
                                Image(systemName: "photo.on.rectangle")
                                    .font(.system(size: 40))
                                    .foregroundStyle(.secondary)
                                Text("选择一张图片")
                                    .foregroundStyle(.secondary)
                            }
                        }
                }

                HStack(spacing: 16) {
                    Button("从相册选择") {
                        showPicker = true
                    }
                    .buttonStyle(.borderedProminent)

                    Button("拍照") {
                        showCamera = true
                    }
                    .buttonStyle(.bordered)
                }
            }
            .padding()
            .navigationTitle("图片选择器")
            .sheet(isPresented: $showPicker) {
                ImagePicker(selectedImage: $selectedImage, sourceType: .photoLibrary)
            }
            .sheet(isPresented: $showCamera) {
                ImagePicker(selectedImage: $selectedImage, sourceType: .camera)
            }
        }
    }
}
```

### 3.3 实战：包装 UITableViewController

```swift
import SwiftUI
import UIKit

struct SettingsTableViewController: UIViewControllerRepresentable {
    var items: [String]
    var onSelect: (Int) -> Void

    func makeUIViewController(context: Context) -> UITableViewController {
        let controller = UITableViewController(style: .insetGrouped)
        controller.tableView.delegate = context.coordinator
        controller.tableView.dataSource = context.coordinator
        controller.tableView.register(
            UITableViewCell.self,
            forCellReuseIdentifier: "cell"
        )
        return controller
    }

    func updateUIViewController(
        _ uiViewController: UITableViewController,
        context: Context
    ) {
        context.coordinator.items = items
        context.coordinator.onSelect = onSelect
        uiViewController.tableView.reloadData()
    }

    func makeCoordinator() -> Coordinator {
        Coordinator(items: items, onSelect: onSelect)
    }

    class Coordinator: NSObject, UITableViewDelegate, UITableViewDataSource {
        var items: [String]
        var onSelect: (Int) -> Void

        init(items: [String], onSelect: @escaping (Int) -> Void) {
            self.items = items
            self.onSelect = onSelect
        }

        func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
            items.count
        }

        func tableView(
            _ tableView: UITableView,
            cellForRowAt indexPath: IndexPath
        ) -> UITableViewCell {
            let cell = tableView.dequeueReusableCell(
                withIdentifier: "cell",
                for: indexPath
            )
            cell.textLabel?.text = items[indexPath.row]
            cell.accessoryType = .disclosureIndicator
            return cell
        }

        func tableView(
            _ tableView: UITableView,
            didSelectRowAt indexPath: IndexPath
        ) {
            tableView.deselectRow(at: indexPath, animated: true)
            onSelect(indexPath.row)
        }
    }
}
```

---

## 4. 反向桥接：在 UIKit 中嵌入 SwiftUI 视图

`UIHostingController` 是反向桥接的核心——它把 SwiftUI 视图包装成一个 UIKit 的 `UIViewController`，让你可以在 UIKit 项目中使用 SwiftUI。

💡 **通俗理解**：如果说 UIViewRepresentable 是"UIKit 人去 SwiftUI 城市工作"，那 UIHostingController 就是"SwiftUI 人来 UIKit 城市工作"——UIHostingController 就是 SwiftUI 人的"工作签证"。

### 4.1 UIHostingController 基础

```swift
let swiftUIView = MySwiftUIView()
let hostingController = UIHostingController(rootView: swiftUIView)
```

`UIHostingController` 是一个完整的 `UIViewController`，可以：
- 被 push 到 `UINavigationController`
- 被 present 出来
- 作为 `UITabBarController` 的 tab
- 作为 `UIWindow` 的 rootViewController

### 4.2 实战：在 UIKit 项目中逐步迁移到 SwiftUI

假设你有一个 UIKit 项目，想逐步引入 SwiftUI：

```swift
import SwiftUI

class SceneDelegate: UIResponder, UIWindowSceneDelegate {
    var window: UIWindow?

    func scene(
        _ scene: UIScene,
        willConnectTo session: UISceneSession,
        options connectionOptions: UIScene.ConnectionOptions
    ) {
        guard let windowScene = (scene as? UIWindowScene) else { return }

        let tabBarCtrl = UITabBarController()

        let homeVC = UIHostingController(rootView: HomeView())
        homeVC.tabBarItem = UITabBarItem(
            title: "首页",
            image: UIImage(systemName: "house"),
            selectedImage: UIImage(systemName: "house.fill")
        )

        let searchVC = UIHostingController(rootView: SearchView())
        searchVC.tabBarItem = UITabBarItem(
            title: "搜索",
            image: UIImage(systemName: "magnifyingglass"),
            selectedImage: UIImage(systemName: "magnifyingglass")
        )

        let profileVC = ProfileViewController()
        profileVC.tabBarItem = UITabBarItem(
            title: "我的",
            image: UIImage(systemName: "person"),
            selectedImage: UIImage(systemName: "person.fill")
        )

        tabBarCtrl.viewControllers = [homeVC, searchVC, profileVC]

        window = UIWindow(windowScene: windowScene)
        window?.rootViewController = tabBarCtrl
        window?.makeKeyAndVisible()
    }
}
```

> 💡 **提示**：迁移策略是"新页面用 SwiftUI，旧页面逐步改"。不需要一次性重写，可以逐个页面迁移。

### 4.3 从 UIKit 跳转到 SwiftUI 页面

```swift
class ProfileViewController: UIViewController {
    override func viewDidLoad() {
        super.viewDidLoad()

        let button = UIButton(type: .system)
        button.setTitle("查看详情（SwiftUI）", for: .normal)
        button.addTarget(
            self,
            action: #selector(showSwiftUIDetail),
            for: .touchUpInside
        )
        button.translatesAutoresizingMaskIntoConstraints = false
        view.addSubview(button)

        NSLayoutConstraint.activate([
            button.centerXAnchor.constraint(equalTo: view.centerXAnchor),
            button.centerYAnchor.constraint(equalTo: view.centerYAnchor)
        ])
    }

    @objc func showSwiftUIDetail() {
        let detailView = ProfileDetailView(userName: "张三")
        let hostingController = UIHostingController(rootView: detailView)
        navigationController?.pushViewController(hostingController, animated: true)
    }
}
```

### 4.4 UIHostingController 传递数据

从 UIKit 向 SwiftUI 传递数据非常简单——直接通过 SwiftUI 视图的初始化参数：

```swift
struct ProfileDetailView: View {
    let userName: String
    @State private var isFollowing = false

    var body: some View {
        VStack(spacing: 20) {
            Circle()
                .fill(Color.blue.gradient)
                .frame(width: 80, height: 80)
                .overlay {
                    Text(String(userName.prefix(1)))
                        .font(.title.bold())
                        .foregroundStyle(.white)
                }

            Text(userName)
                .font(.title2.bold())

            Button(isFollowing ? "已关注" : "关注") {
                isFollowing.toggle()
            }
            .buttonStyle(.borderedProminent)
        }
        .navigationTitle("用户详情")
    }
}
```

| 传递方向 | 方式 | 说明 |
|---------|------|------|
| UIKit → SwiftUI | 初始化参数 | 直接传入 SwiftUI 视图的属性 |
| SwiftUI → UIKit | 回调闭包 / ObservableObject | 通过闭包或共享状态对象回传 |
| 双向 | @Binding + Coordinator | 通过 Binding 实现双向绑定 |

---

## 5. 数据传递：SwiftUI ↔ UIKit 双向绑定

💡 **通俗理解**：数据传递就像两个人打电话——单向绑定是"广播"（一方说，另一方听），双向绑定是"对讲机"（双方都能说能听）。

### 5.1 SwiftUI → UIKit（单向）

通过 `updateUIView` / `updateUIViewController` 将 SwiftUI 状态同步到 UIKit：

```swift
struct ProgressIndicator: UIViewRepresentable {
    var progress: Float

    func makeUIView(context: Context) -> UIProgressView {
        UIProgressView(progressViewStyle: .default)
    }

    func updateUIView(_ uiView: UIProgressView, context: Context) {
        uiView.setProgress(progress, animated: true)
    }
}
```

### 5.2 UIKit → SwiftUI（通过 @Binding）

通过 `@Binding` 和 Coordinator 实现 UIKit 的变化回传给 SwiftUI：

```swift
struct SliderView: UIViewRepresentable {
    @Binding var value: Double
    var range: ClosedRange<Double>

    func makeUIView(context: Context) -> UISlider {
        let slider = UISlider()
        slider.minimumValue = Float(range.lowerBound)
        slider.maximumValue = Float(range.upperBound)
        slider.value = Float(value)
        slider.addTarget(
            context.coordinator,
            action: #selector(Coordinator.valueChanged),
            for: .valueChanged
        )
        return slider
    }

    func updateUIView(_ uiView: UISlider, context: Context) {
        if Float(value) != uiView.value {
            uiView.value = Float(value)
        }
    }

    func makeCoordinator() -> Coordinator {
        Coordinator(value: $value)
    }

    class Coordinator {
        var value: Binding<Double>

        init(value: Binding<Double>) {
            self.value = value
        }

        @objc func valueChanged(_ sender: UISlider) {
            value.wrappedValue = Double(sender.value)
        }
    }
}
```

### 5.3 双向绑定完整示例

```swift
struct BidirectionalBindingDemo: View {
    @State private var sliderValue: Double = 50

    var body: some View {
        NavigationStack {
            VStack(spacing: 30) {
                Text("当前值：\(Int(sliderValue))")
                    .font(.title.bold())

                SliderView(value: $sliderValue, range: 0...100)
                    .padding(.horizontal)

                Slider(value: $sliderValue, in: 0...100)
                    .padding(.horizontal)

                HStack(spacing: 16) {
                    Button("设为 0") { sliderValue = 0 }
                    Button("设为 50") { sliderValue = 50 }
                    Button("设为 100") { sliderValue = 100 }
                }
                .buttonStyle(.bordered)
            }
            .padding()
            .navigationTitle("双向绑定")
        }
    }
}
```

> 💡 **提示**：在 `updateUIView` 中检查 `if Float(value) != uiView.value` 非常重要！如果不检查，可能会造成"循环更新"——SwiftUI 更新 UIKit → UIKit 回调更新 SwiftUI → SwiftUI 又更新 UIKit……

### 5.4 使用 ObservableObject 共享状态

对于复杂场景，推荐使用 `ObservableObject` 作为共享状态：

```swift
class SharedStore: ObservableObject {
    @Published var message: String = ""
    @Published var counter: Int = 0
}

struct SharedStateDemo: View {
    @StateObject private var store = SharedStore()

    var body: some View {
        NavigationStack {
            VStack(spacing: 20) {
                Text("计数器：\(store.counter)")
                    .font(.title.bold())

                Text("消息：\(store.message)")
                    .font(.headline)

                CounterUIKitView(store: store)
                    .frame(height: 120)
                    .padding(.horizontal)

                HStack(spacing: 16) {
                    Button("SwiftUI +1") { store.counter += 1 }
                    Button("SwiftUI 发消息") { store.message = "来自 SwiftUI" }
                }
                .buttonStyle(.bordered)
            }
            .padding()
            .navigationTitle("共享状态")
        }
    }
}

struct CounterUIKitView: UIViewRepresentable {
    @ObservedObject var store: SharedStore

    func makeUIView(context: Context) -> UIStepper {
        let stepper = UIStepper()
        stepper.addTarget(
            context.coordinator,
            action: #selector(Coordinator.step),
            for: .valueChanged
        )
        return stepper
    }

    func updateUIView(_ uiView: UIStepper, context: Context) {
    }

    func makeCoordinator() -> Coordinator {
        Coordinator(store: store)
    }

    class Coordinator {
        var store: SharedStore

        init(store: SharedStore) {
            self.store = store
        }

        @objc func step(_ sender: UIStepper) {
            store.counter += 1
            store.message = "来自 UIKit Stepper"
        }
    }
}
```

---

## 6. Coordinator 模式深入

Coordinator 是 UIViewRepresentable / UIViewControllerRepresentable 中最核心的设计模式，负责处理 UIKit 的 delegate、dataSource、target-action 等回调机制。

💡 **通俗理解**：Coordinator 就像一个"联络员"——UIKit 那边发生了事件（用户点击了、滚动停止了、输入了文字），联络员负责把这些事件"翻译"并传递给 SwiftUI。没有 Coordinator，UIKit 和 SwiftUI 就无法沟通。

### 6.1 Coordinator 的角色

| UIKit 回调机制 | Coordinator 处理方式 | 示例 |
|---------------|---------------------|------|
| **Delegate** | Coordinator 遵循协议并实现方法 | UITextFieldDelegate, MKMapViewDelegate |
| **DataSource** | Coordinator 遵循 DataSource 协议 | UITableViewDataSource, UICollectionViewDataSource |
| **Target-Action** | Coordinator 提供 @objc 方法 | UIButton 点击、UISlider 值变化 |
| **Notification** | Coordinator 注册并响应通知 | 键盘出现/隐藏通知 |

### 6.2 处理 Delegate 回调

```swift
struct WebView: UIViewRepresentable {
    let url: URL
    var onLoadFinish: (() -> Void)?
    var onLoadError: ((Error) -> Void)?

    func makeUIView(context: Context) -> WKWebView {
        let webView = WKWebView()
        webView.navigationDelegate = context.coordinator
        webView.load(URLRequest(url: url))
        return webView
    }

    func updateUIView(_ uiView: WKWebView, context: Context) {}

    func makeCoordinator() -> Coordinator {
        Coordinator(onLoadFinish: onLoadFinish, onLoadError: onLoadError)
    }

    class Coordinator: NSObject, WKNavigationDelegate {
        var onLoadFinish: (() -> Void)?
        var onLoadError: ((Error) -> Void)?

        init(onLoadFinish: (() -> Void)?, onLoadError: ((Error) -> Void)?) {
            self.onLoadFinish = onLoadFinish
            self.onLoadError = onLoadError
        }

        func webView(_ webView: WKWebView, didFinish navigation: WKNavigation!) {
            onLoadFinish?()
        }

        func webView(
            _ webView: WKWebView,
            didFail navigation: WKNavigation!,
            withError error: Error
        ) {
            onLoadError?(error)
        }

        func webView(
            _ webView: WKWebView,
            decidePolicyFor navigationAction: WKNavigationAction,
            decisionHandler: @escaping (WKNavigationActionPolicy) -> Void
        ) {
            decisionHandler(.allow)
        }
    }
}
```

### 6.3 处理 DataSource

```swift
struct PickerView: UIViewRepresentable {
    @Binding var selectedIndex: Int
    var items: [String]

    func makeUIView(context: Context) -> UIPickerView {
        let picker = UIPickerView()
        picker.delegate = context.coordinator
        picker.dataSource = context.coordinator
        return picker
    }

    func updateUIView(_ uiView: UIPickerView, context: Context) {
        context.coordinator.items = items
        if uiView.selectedRow(inComponent: 0) != selectedIndex {
            uiView.selectRow(selectedIndex, inComponent: 0, animated: true)
        }
        uiView.reloadAllComponents()
    }

    func makeCoordinator() -> Coordinator {
        Coordinator(selectedIndex: $selectedIndex, items: items)
    }

    class Coordinator: NSObject, UIPickerViewDelegate, UIPickerViewDataSource {
        @Binding var selectedIndex: Int
        var items: [String]

        init(selectedIndex: Binding<Int>, items: [String]) {
            self._selectedIndex = selectedIndex
            self.items = items
        }

        func numberOfComponents(in pickerView: UIPickerView) -> Int {
            1
        }

        func pickerView(
            _ pickerView: UIPickerView,
            numberOfRowsInComponent component: Int
        ) -> Int {
            items.count
        }

        func pickerView(
            _ pickerView: UIPickerView,
            titleForRow row: Int,
            forComponent component: Int
        ) -> String? {
            items[row]
        }

        func pickerView(
            _ pickerView: UIPickerView,
            didSelectRow row: Int,
            inComponent component: Int
        ) {
            selectedIndex = row
        }
    }
}
```

### 6.4 处理 Target-Action

```swift
struct SwitchView: UIViewRepresentable {
    @Binding var isOn: Bool

    func makeUIView(context: Context) -> UISwitch {
        let toggle = UISwitch()
        toggle.isOn = isOn
        toggle.addTarget(
            context.coordinator,
            action: #selector(Coordinator.valueChanged),
            for: .valueChanged
        )
        return toggle
    }

    func updateUIView(_ uiView: UISwitch, context: Context) {
        if uiView.isOn != isOn {
            uiView.isOn = isOn
        }
    }

    func makeCoordinator() -> Coordinator {
        Coordinator(isOn: $isOn)
    }

    class Coordinator {
        @Binding var isOn: Bool

        init(isOn: Binding<Bool>) {
            self._isOn = isOn
        }

        @objc func valueChanged(_ sender: UISwitch) {
            isOn = sender.isOn
        }
    }
}
```

### 6.5 Coordinator 更新闭包的问题

> ⚠️ **警告**：`updateUIView` / `updateUIViewController` 被调用时，Coordinator 中的闭包属性不会自动更新！你必须在 `updateUIView` 中手动同步闭包：

```swift
func updateUIView(_ uiView: WKWebView, context: Context) {
    context.coordinator.onLoadFinish = onLoadFinish
    context.coordinator.onLoadError = onLoadError
}
```

这是因为 Coordinator 只在 `makeCoordinator()` 时创建一次，后续 SwiftUI 状态变化不会重新创建 Coordinator。如果闭包捕获了旧的 `@State`，可能导致数据不同步。

---

## 7. 手势与焦点管理跨框架传递

### 7.1 手势冲突处理

当 SwiftUI 视图和 UIKit 视图嵌套时，手势可能产生冲突：

| 场景 | 问题 | 解决方案 |
|------|------|---------|
| UIScrollView 嵌套在 SwiftUI ScrollView 中 | 滚动手势冲突 | 使用 `scrollDisabled(true)` 禁用 SwiftUI 滚动 |
| UIKit 手势与 SwiftUI 手势重叠 | 只有一个能响应 | 使用 `.simultaneousGesture()` 或在 UIKit 中设置 `cancelsTouchesInView` |
| UITextField 在 SwiftUI 手势视图中 | 点击无法聚焦 | 使用 `.allowsHitTesting(true)` |

```swift
struct EmbeddedScrollView: UIViewRepresentable {
    var content: String

    func makeUIView(context: Context) -> UITextView {
        let textView = UITextView()
        textView.isEditable = false
        textView.isScrollEnabled = true
        textView.text = content
        return textView
    }

    func updateUIView(_ uiView: UITextView, context: Context) {
        uiView.text = content
    }
}

struct GestureConflictDemo: View {
    let longText = String(repeating: "这是一段很长的文本内容，需要滚动查看。", count: 20)

    var body: some View {
        ScrollView {
            VStack(spacing: 20) {
                Text("SwiftUI ScrollView 内容")
                    .font(.headline)

                EmbeddedScrollView(content: longText)
                    .frame(height: 200)
                    .scrollDisabled(true)
            }
            .padding()
        }
        .navigationTitle("手势冲突处理")
    }
}
```

### 7.2 焦点管理

在 UIKit 中，焦点（first responder）通过 `becomeFirstResponder` / `resignFirstResponder` 管理。在 SwiftUI 中，焦点通过 `@FocusState` 管理。跨框架时需要桥接：

```swift
struct FocusableTextField: UIViewRepresentable {
    @Binding var text: String
    var placeholder: String
    var isFocused: Bool

    func makeUIView(context: Context) -> UITextField {
        let textField = UITextField()
        textField.placeholder = placeholder
        textField.delegate = context.coordinator
        textField.addTarget(
            context.coordinator,
            action: #selector(Coordinator.textDidChange),
            for: .editingChanged
        )
        return textField
    }

    func updateUIView(_ uiView: UITextField, context: Context) {
        if uiView.text != text {
            uiView.text = text
        }

        if isFocused && !uiView.isFirstResponder {
            uiView.becomeFirstResponder()
        } else if !isFocused && uiView.isFirstResponder {
            uiView.resignFirstResponder()
        }
    }

    func makeCoordinator() -> Coordinator {
        Coordinator(text: $text)
    }

    class Coordinator: NSObject, UITextFieldDelegate {
        var text: Binding<String>

        init(text: Binding<String>) {
            self.text = text
        }

        @objc func textDidChange(_ textField: UITextField) {
            text.wrappedValue = textField.text ?? ""
        }

        func textFieldDidBeginEditing(_ textField: UITextField) {
        }

        func textFieldDidEndEditing(_ textField: UITextField) {
        }
    }
}
```

使用 SwiftUI 的 `@FocusState` 控制焦点：

```swift
struct FocusManagementDemo: View {
    @State private var username = ""
    @State private var email = ""
    @FocusState private var focusedField: Field?

    enum Field {
        case username, email
    }

    var body: some View {
        NavigationStack {
            Form {
                Section("注册信息") {
                    FocusableTextField(
                        text: $username,
                        placeholder: "用户名",
                        isFocused: focusedField == .username
                    )
                    .frame(height: 44)

                    FocusableTextField(
                        text: $email,
                        placeholder: "邮箱",
                        isFocused: focusedField == .email
                    )
                    .frame(height: 44)
                }

                Section("操作") {
                    Button("聚焦用户名") {
                        focusedField = .username
                    }
                    Button("聚焦邮箱") {
                        focusedField = .email
                    }
                    Button("收起键盘") {
                        focusedField = nil
                    }
                }

                Section("预览") {
                    Text("用户名：\(username)")
                    Text("邮箱：\(email)")
                }
            }
            .navigationTitle("焦点管理")
        }
    }
}
```

---

## 8. 常见坑与最佳实践

### 8.1 常见坑

| 坑 | 表现 | 解决方案 |
|----|------|---------|
| **循环更新** | `updateUIView` 和回调互相触发，性能骤降 | 在 `updateUIView` 中检查值是否真的变化了再更新 |
| **Coordinator 闭包过期** | 闭包捕获旧值，回调时数据不对 | 在 `updateUIView` 中手动更新 Coordinator 的闭包 |
| **内存泄漏** | Coordinator 持有强引用循环 | Coordinator 中对 UIView 使用 `weak` 引用 |
| **线程安全** | UIKit 回调在主线程，但 SwiftUI 状态可能在后台线程更新 | 确保状态更新在主线程 `DispatchQueue.main.async` |
| **生命周期差异** | SwiftUI 视图是值类型，可能被反复创建销毁；UIView 是引用类型 | 不要在 `makeUIView` 之外创建 UIView |
| **尺寸不匹配** | UIKit 视图不遵循 SwiftUI 布局 | 使用 `.frame()` 明确指定大小，或实现 `sizeThatFits` |

### 8.2 线程安全

UIKit 的所有操作必须在主线程执行。如果 SwiftUI 的状态在后台线程更新，需要确保 UIKit 操作回到主线程：

```swift
func updateUIView(_ uiView: UILabel, context: Context) {
    DispatchQueue.main.async {
        uiView.text = text
    }
}
```

> ⚠️ **警告**：虽然 SwiftUI 的 `@State` / `@Binding` 更新会自动在主线程触发 `updateUIView`，但如果你的 Coordinator 回调中涉及 UIKit 操作，仍需确保在主线程。

### 8.3 生命周期差异

```swift
struct MyView: UIViewRepresentable {
    @State var data: String = ""

    func makeUIView(context: Context) -> MyUIView {
        print("makeUIView 被调用")
        return MyUIView()
    }

    func updateUIView(_ uiView: MyUIView, context: Context) {
        print("updateUIView 被调用")
    }
}
```

SwiftUI 的 `View` 是 **struct（值类型）**，每次状态变化都会重新创建 struct 实例，但 `UIView` 是 **class（引用类型）**，不会被重新创建。这意味着：

- `makeUIView` 只调用一次
- `updateUIView` 可能被频繁调用
- 不要在 struct 中存储 UIKit 状态，应该放在 Coordinator 或 `@State` 中

### 8.4 性能注意事项

| 建议 | 说明 |
|------|------|
| **避免在 updateUIView 中做重计算** | 该方法可能被频繁调用，只做最小更新 |
| **使用 diff 检查** | 比较新旧值，只在真正变化时更新 UIKit 视图 |
| **缓存不变的部分** | 如 delegate、dataSource 数据，避免每次重建 |
| **合理使用 invalidateIntrinsicContentSize** | 只在尺寸真正变化时调用 |
| **避免频繁 reload** | UITableView/UICollectionView 的 reloadData 很昂贵 |

### 8.5 最佳实践清单

```swift
struct BestPracticeView: UIViewRepresentable {
    @Binding var text: String

    func makeUIView(context: Context) -> UITextField {
        let textField = UITextField()
        textField.delegate = context.coordinator
        textField.addTarget(
            context.coordinator,
            action: #selector(Coordinator.textChanged),
            for: .editingChanged
        )
        return textField
    }

    func updateUIView(_ uiView: UITextField, context: Context) {
        if uiView.text != text {
            uiView.text = text
        }
    }

    func makeCoordinator() -> Coordinator {
        Coordinator(text: $text)
    }

    static func dismantleUIView(_ uiView: UITextField, coordinator: Coordinator) {
        uiView.delegate = nil
        uiView.removeTarget(coordinator, action: nil, for: .editingChanged)
    }

    class Coordinator: NSObject, UITextFieldDelegate {
        var text: Binding<String>

        init(text: Binding<String>) {
            self.text = text
        }

        @objc func textChanged(_ sender: UITextField) {
            text.wrappedValue = sender.text ?? ""
        }

        func textFieldShouldReturn(_ textField: UITextField) -> Bool {
            textField.resignFirstResponder()
            return true
        }
    }
}
```

> 💡 **提示**：实现 `dismantleUIView` 清理 delegate 和 target-action 是好习惯，可以避免内存泄漏和野指针崩溃。

---

## 9. 实战：把 ASAuthorizationController 包装成 SwiftUI 组件

Sign in with Apple 是 iOS App 上架的常见需求，但 `ASAuthorizationController` 是纯 UIKit API。我们将把它包装成一个优雅的 SwiftUI 组件。

💡 **通俗理解**：这就像给一台老式收音机装上蓝牙模块——收音机本身只有旋钮和天线（UIKit API），但加上蓝牙模块（SwiftUI 包装）后，你就可以用手机远程控制它了。

### 9.1 完整实现

```swift
import SwiftUI
import AuthenticationServices

struct SignInWithAppleButton: View {
    var onSuccess: (String, String) -> Void
    var onFailure: (Error) -> Void

    var body: some View {
        SignInWithAppleRepresentable(
            onSuccess: onSuccess,
            onFailure: onFailure
        )
        .frame(height: 50)
    }
}

struct SignInWithAppleRepresentable: UIViewRepresentable {
    var onSuccess: (String, String) -> Void
    var onFailure: (Error) -> Void

    func makeUIView(context: Context) -> ASAuthorizationAppleIDButton {
        let button = ASAuthorizationAppleIDButton(
            authorizationButtonType: .signIn,
            authorizationButtonStyle: .black
        )
        button.addTarget(
            context.coordinator,
            action: #selector(Coordinator.signIn),
            for: .touchUpInside
        )
        return button
    }

    func updateUIView(_ uiView: ASAuthorizationAppleIDButton, context: Context) {
        context.coordinator.onSuccess = onSuccess
        context.coordinator.onFailure = onFailure
    }

    func makeCoordinator() -> Coordinator {
        Coordinator(onSuccess: onSuccess, onFailure: onFailure)
    }

    class Coordinator: NSObject, ASAuthorizationControllerDelegate, ASAuthorizationControllerPresentationContextProviding {
        var onSuccess: (String, String) -> Void
        var onFailure: (Error) -> Void

        init(onSuccess: @escaping (String, String) -> Void, onFailure: @escaping (Error) -> Void) {
            self.onSuccess = onSuccess
            self.onFailure = onFailure
        }

        @objc func signIn() {
            let provider = ASAuthorizationAppleIDProvider()
            let request = provider.createRequest()
            request.requestedScopes = [.fullName, .email]

            let controller = ASAuthorizationController(authorizationRequests: [request])
            controller.delegate = self
            controller.presentationContextProvider = self
            controller.performRequests()
        }

        func authorizationController(
            controller: ASAuthorizationController,
            didCompleteWithAuthorization authorization: ASAuthorization
        ) {
            guard let credential = authorization.credential as? ASAuthorizationAppleIDCredential else {
                return
            }

            let userId = credential.user
            let email = credential.email ?? ""
            let fullName = (credential.fullName?.givenName ?? "") + (credential.fullName?.familyName.map { " " + $0 } ?? "")

            onSuccess(userId, email.isEmpty ? fullName : email)
        }

        func authorizationController(
            controller: ASAuthorizationController,
            didCompleteWithError error: Error
        ) {
            onFailure(error)
        }

        func presentationAnchor(for controller: ASAuthorizationController) -> ASPresentationAnchor {
            guard let scene = UIApplication.shared.connectedScenes.first as? UIWindowScene,
                  let window = scene.windows.first else {
                return UIWindow()
            }
            return window
        }
    }
}
```

### 9.2 使用方式

```swift
struct SignInWithAppleDemo: View {
    @State private var message = ""
    @State private var isLoggedIn = false

    var body: some View {
        NavigationStack {
            VStack(spacing: 30) {
                if isLoggedIn {
                    VStack(spacing: 16) {
                        Image(systemName: "checkmark.circle.fill")
                            .font(.system(size: 60))
                            .foregroundStyle(.green)

                        Text("登录成功！")
                            .font(.title2.bold())

                        Text(message)
                            .font(.subheadline)
                            .foregroundStyle(.secondary)
                            .multilineTextAlignment(.center)
                            .padding(.horizontal)

                        Button("退出登录") {
                            isLoggedIn = false
                            message = ""
                        }
                        .buttonStyle(.bordered)
                    }
                } else {
                    VStack(spacing: 20) {
                        Image(systemName: "apple.logo")
                            .font(.system(size: 60))
                            .foregroundStyle(.primary)

                        Text("通过 Apple 登录")
                            .font(.title2.bold())

                        Text("使用你的 Apple ID 快速登录，无需记忆额外密码")
                            .font(.subheadline)
                            .foregroundStyle(.secondary)
                            .multilineTextAlignment(.center)
                            .padding(.horizontal, 40)

                        SignInWithAppleButton(
                            onSuccess: { userId, email in
                                message = "用户ID: \(userId)\n邮箱/姓名: \(email)"
                                isLoggedIn = true
                            },
                            onFailure: { error in
                                message = "登录失败: \(error.localizedDescription)"
                            }
                        )
                        .padding(.horizontal, 40)
                    }
                }
            }
            .padding()
            .navigationTitle("Sign in with Apple")
        }
    }
}
```

### 9.3 关键点解析

| 关键点 | 说明 |
|--------|------|
| **ASAuthorizationAppleIDButton** | Apple 提供的官方按钮样式，必须使用（审核要求） |
| **requestedScopes** | 请求的用户信息范围：`.fullName` 和 `.email` |
| **presentationContextProvider** | 告诉系统在哪个 Window 上弹出登录界面 |
| **credential.user** | 用户的唯一标识符，用于后续验证 |
| **credential.email** | 邮箱只在首次授权时返回，后续为 nil |
| **闭包更新** | 在 `updateUIView` 中更新 Coordinator 的闭包，避免过期 |

> ⚠️ **警告**：Apple 审核指南要求，如果你的 App 支持第三方社交登录（微信、Google 等），就必须同时提供 Sign in with Apple 选项。

---

## 本章小结

| 知识点 | 核心内容 |
|--------|---------|
| **互操作必要性** | 第三方 SDK、成熟 UIKit 组件、遗留代码迁移、系统控制器 |
| **UIViewRepresentable** | makeUIView / updateUIView / dismantleUIView，包装 UIView |
| **UIViewControllerRepresentable** | makeUIViewController / updateUIViewController，包装 UIViewController |
| **反向桥接 UIHostingController** | 在 UIKit 中嵌入 SwiftUI，逐步迁移策略 |
| **数据双向绑定** | @Binding、Coordinator 回调、ObservableObject 共享状态 |
| **Coordinator 模式** | 处理 delegate / dataSource / target-action / notification |
| **手势与焦点** | 手势冲突处理、@FocusState 与 becomeFirstResponder 桥接 |
| **常见坑** | 循环更新、闭包过期、内存泄漏、线程安全、生命周期差异 |
| **实战 Sign in with Apple** | ASAuthorizationController 完整包装，闭包回传 |

🔑 **核心记忆点**：
1. `makeUIView` 只调用一次做初始化，`updateUIView` 每次状态变化都调用做更新
2. Coordinator 是 UIKit 回调与 SwiftUI 之间的桥梁，处理所有 delegate 和 target-action
3. 在 `updateUIView` 中必须检查值是否真正变化，避免循环更新
4. Coordinator 的闭包属性不会自动更新，需要在 `updateUIView` 中手动同步
5. `dismantleUIView` 中清理 delegate 和 target-action，防止内存泄漏
6. UIHostingController 是在 UIKit 项目中引入 SwiftUI 的入口
7. 所有 UIKit 操作必须在主线程执行
8. Sign in with Apple 的 ASAuthorizationController 是互操作的典型实战场景

📖 **上一章**：[83-visionOS入门](../10-visionOS入门/83-visionOS入门.md)
