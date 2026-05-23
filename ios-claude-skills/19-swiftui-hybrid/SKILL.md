---
name: swiftui-hybrid
description: 涉及 SwiftUI 混合开发、UIKit 与 SwiftUI 互操作、UIHostingController、UIViewRepresentable、UIViewControllerRepresentable、SwiftUI 在 UIKit 项目中嵌入的任务
---

# SwiftUI 混合开发

## 架构原则

本项目主框架为 **UIKit**，SwiftUI 仅在以下场景使用：
1. **Widget / Live Activity**（必须 SwiftUI）
2. **新页面快速原型**（SwiftUI 开发效率更高）
3. **复杂动画/声明式 UI**（SwiftUI 更简洁）
4. **系统新 API 仅提供 SwiftUI 接口**

**禁止：**
- 禁止将已有 UIKit 页面重写为 SwiftUI（投入产出比低）
- 禁止在 SwiftUI View 中嵌入大量 UIKit 组件（失去 SwiftUI 优势）
- 禁止混合使用 UIKit 和 SwiftUI 的导航（选一种，页面内统一）

---

## UIKit 嵌入 SwiftUI

### UIHostingController（SwiftUI 页面嵌入 UIKit 导航）

```swift
let swiftUIView = SettingsSwiftUIView(viewModel: settingsViewModel)
let hostingController = UIHostingController(rootView: swiftUIView)
hostingController.title = "设置"
navigationController?.pushViewController(hostingController, animated: true)
```

### present 方式

```swift
let swiftUIView = PaywallSwiftUIView()
let hostingController = UIHostingController(rootView: swiftUIView)
hostingController.modalPresentationStyle = .pageSheet
if let sheet = hostingController.sheetPresentationController {
    sheet.detents = [.medium(), .large()]
    sheet.prefersGrabberVisible = true
}
present(hostingController, animated: true)
```

### 规范
- 每个 SwiftUI 页面用独立的 `UIHostingController` 包装
- **禁止在 UIHostingController 中覆盖 `viewWillLayoutSubviews` 等 UIKit 生命周期**，用 SwiftUI 的 `onAppear` / `onDisappear`
- UIHostingController 的 `sizingOptions`（iOS 16+）可控制尺寸更新行为
- 传递 ViewModel 时注意生命周期：SwiftUI View 是 struct，每次重建，ViewModel 必须由外部持有

---

## SwiftUI 嵌入 UIKit 组件

### UIViewRepresentable（UIKit View 嵌入 SwiftUI）

```swift
struct MapViewRepresentable: UIViewRepresentable {
    let coordinate: CLLocationCoordinate2D

    func makeUIView(context: Context) -> MKMapView {
        let mapView = MKMapView()
        mapView.delegate = context.coordinator
        mapView.showsUserLocation = true
        return mapView
    }

    func updateUIView(_ mapView: MKMapView, context: Context) {
        let region = MKCoordinateRegion(
            center: coordinate,
            span: MKCoordinateSpan(latitudeDelta: 0.01, longitudeDelta: 0.01)
        )
        mapView.setRegion(region, animated: true)
    }

    func makeCoordinator() -> Coordinator {
        Coordinator()
    }

    class Coordinator: NSObject, MKMapViewDelegate {
        func mapView(_ mapView: MKMapView, didUpdate userLocation: MKUserLocation) {
            // 处理位置更新
        }
    }
}
```

### UIViewControllerRepresentable（UIKit VC 嵌入 SwiftUI）

```swift
struct CameraViewControllerRepresentable: UIViewControllerRepresentable {
    @Binding var capturedImage: UIImage?

    func makeUIViewController(context: Context) -> CameraVC {
        let vc = CameraVC()
        vc.delegate = context.coordinator
        return vc
    }

    func updateUIViewController(_ uiViewController: CameraVC, context: Context) {
    }

    func makeCoordinator() -> Coordinator {
        Coordinator(parent: self)
    }

    class Coordinator: NSObject, CameraVCDelegate {
        let parent: CameraViewControllerRepresentable

        init(parent: CameraViewControllerRepresentable) {
            self.parent = parent
        }

        func cameraVC(_ vc: CameraVC, didCapture image: UIImage) {
            parent.capturedImage = image
        }
    }
}
```

### 规范
- `makeUIView` / `makeUIViewController` 只做初始化，**禁止在这里做数据绑定**
- `updateUIView` / `updateUIViewController` 处理数据更新，**必须判断数据是否变化再更新**（避免无限循环）
- Delegate 模式通过 `Coordinator` 桥接，**禁止在 Representable 中直接实现 Delegate**
- `Coordinator` 持有 parent 引用时注意循环引用

---

## 数据通信

### UIKit → SwiftUI（单向数据流）

```swift
class SettingsViewModel: ObservableObject {
    @Published var darkMode: Bool = false
    @Published var language: String = "zh-Hans"
}

// UIKit 端持有 ViewModel
let viewModel = SettingsViewModel()
let swiftUIView = SettingsView(viewModel: viewModel)
let hostingVC = UIHostingController(rootView: swiftUIView)

// UIKit 修改数据，SwiftUI 自动响应
viewModel.darkMode = true
```

### SwiftUI → UIKit（回调/通知）

```swift
struct SettingsView: View {
    @ObservedObject var viewModel: SettingsViewModel
    var onLogout: (() -> Void)?

    var body: some View {
        Button("退出登录") {
            onLogout?()
        }
    }
}

// UIKit 端
let swiftUIView = SettingsView(viewModel: viewModel) {
    self.handleLogout()
}
```

### 通知方式（跨层级通信）

```swift
// SwiftUI 端发送
NotificationCenter.default.post(name: .userDidLogout, object: nil)

// UIKit 端监听
NotificationCenter.default.addObserver(self, selector: #selector(handleLogout), name: .userDidLogout, object: nil)
```

### 规范
- **优先用 `@ObservedObject` / `@StateObject` 传递数据**，避免通知中心（调试困难）
- SwiftUI → UIKit 的操作用 closure 回调，**禁止在 SwiftUI 中直接操作 UIKit 对象**
- ViewModel 由 UIKit 端创建和持有，SwiftUI 端用 `@ObservedObject`（不是 `@StateObject`）
- **禁止在 `updateUIView` 中修改 `@Binding` 值**（会触发无限更新循环）

---

## 导航混合

### 方案一：UIKit 导航为主（推荐）

```
UINavigationController
  ├── UIKitVC → push UIHostingController(SwiftUIView)
  └── UIKitVC → present UIHostingController(SwiftUIView)
```

- SwiftUI 页面内用 `NavigationLink` 会导致**嵌套导航栏**
- 解决：SwiftUI 页面禁用自身导航，统一由 UIKit 控制

```swift
struct ProfileView: View {
    var onNavigateToEdit: (() -> Void)?

    var body: some View {
        VStack {
            Text("个人资料")
            Button("编辑") {
                onNavigateToEdit?()
            }
        }
        .navigationBarHidden(true)
    }
}
```

### 方案二：SwiftUI 导航为主

```
NavigationStack
  ├── SwiftUI View
  └── UIViewControllerRepresentable(UIKitVC)
```

- 适用于新页面全部用 SwiftUI 写的场景
- UIKit 页面嵌入后，导航栏由 SwiftUI 控制

### 规范
- **一个页面内只用一种导航方式**，禁止混用
- UIKit 导航栏和 SwiftUI 导航栏**禁止同时显示**
- `UIHostingController` push 进 UINavigationController 时，SwiftUI 的 `NavigationStack` 不会生效

---

## 主题与样式统一

### 颜色桥接

```swift
extension Color {
    static let appPrimary = Color(uiColor: AppColors.primary)
    static let appSecondary = Color(uiColor: AppColors.secondary)
    static let appBackground = Color(uiColor: AppColors.background)
}
```

### 字体桥接

```swift
extension Font {
    static let appTitle = Font(UIFont.appTitle)
    static let appBody = Font(UIFont.appBody)
    static let appCaption = Font(UIFont.appCaption)
}
```

### 规范
- SwiftUI 和 UIKit 共用同一套设计系统（`AppColors` / `AppFonts`）
- **禁止在 SwiftUI 中硬编码颜色值**，统一从 `AppColors` 桥接
- 暗色模式：UIKit 用 `UIColor { traitCollection in }`，SwiftUI 用 `Color(light:dark:)` 桥接

---

## 性能注意事项

### UIHostingController 性能
- SwiftUI View 的 `body` 每次状态变化都会重新计算
- **禁止在 `body` 中做耗时计算**，用 `@StateObject` 缓存
- 大列表用 `LazyVStack` / `LazyHStack`，禁止 `VStack` 包裹大量子视图

### UIViewRepresentable 性能
- `updateUIView` 调用频率可能很高（每次 SwiftUI 重绘都调用）
- **必须做 diff 判断**，避免不必要的 UIKit 更新

```swift
func updateUIView(_ label: UILabel, context: Context) {
    if label.text != text {
        label.text = text
    }
}
```

### 规范
- SwiftUI 页面中嵌入 UIKit 组件数量不超过 3 个
- UIKit 页面中嵌入 SwiftUI 页面不超过 1 层
- **禁止 SwiftUI ↔ UIKit ↔ SwiftUI 三层嵌套**

---

## 已知陷阱

- **`UIHostingController` 的 `view` 不是 `UIView`，是 `_UIHostingView`**，部分 UIKit API 对其无效
- **SwiftUI 的 `@Environment` 在 `UIHostingController` 中默认为空**，需要手动注入
- **`UIViewRepresentable` 的 `makeUIView` 只调用一次**，配置变更在 `updateUIView` 处理
- **SwiftUI View 的 `onAppear` 在 `UIHostingController` push 时可能延迟调用**
- **`UIViewControllerRepresentable` 嵌入的 VC 不会收到 `viewWillAppear` 等标准生命周期**，需要手动桥接
- **`@Binding` 在 `updateUIView` 中修改会触发无限循环**，必须加条件判断
- **SwiftUI 的 `NavigationLink` 在 `UIHostingController` 内 push 会创建嵌套导航栏**，必须禁用
