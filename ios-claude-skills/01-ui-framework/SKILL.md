---
name: ui-framework
description: 任何涉及界面、视图、布局、动画、ViewController、导航、UICollectionView、SnapKit、暗色模式、键盘适配的任务
---

# UI 框架约定

## 框架选择
- **主框架：UIKit**（非特殊说明不使用 SwiftUI）
- 布局：**SnapKit** 自动布局，禁止 frame 硬编码
- 入口：**SceneDelegate**，无 Storyboard，无 .xib
- 导航：**UINavigationController**，禁止使用 NavigationStack

---

## 设计系统

### 颜色
- 统一从 `AppColors.swift` 引用，禁止直接写 `UIColor(hex:)`
- 必须同时定义 Light / Dark 模式色值：

```swift
enum AppColors {
    static let primary = UIColor { trait in
        switch trait.userInterfaceStyle {
        case .dark: return UIColor(hex: "#0A84FF")
        default: return UIColor(hex: "#007AFF")
        }
    }
    static let background = UIColor { trait in
        switch trait.userInterfaceStyle {
        case .dark: return UIColor(hex: "#1C1C1E")
        default: return UIColor(hex: "#FFFFFF")
        }
    }
    static let secondaryBackground = UIColor { trait in
        switch trait.userInterfaceStyle {
        case .dark: return UIColor(hex: "#2C2C2E")
        default: return UIColor(hex: "#F2F2F7")
        }
    }
}
```

### 字体
- 统一从 `AppFonts.swift` 引用，使用 SF Pro 系列
- 支持 Dynamic Type：

```swift
enum AppFonts {
    static let title1 = UIFont.preferredFont(forTextStyle: .title1)
    static let headline = UIFont.preferredFont(forTextStyle: .headline)
    static let body = UIFont.preferredFont(forTextStyle: .body)
    static let caption = UIFont.preferredFont(forTextStyle: .caption1)

    static func custom(weight: UIFont.Weight, size: CGFloat) -> UIFont {
        let font = UIFont.systemFont(ofSize: size, weight: weight)
        return UIFontMetrics.default.scaledFont(for: font)
    }
}
```

### 间距与圆角
- 间距：基于 **8pt 基础网格**（8 / 16 / 24 / 32）
- 圆角：统一使用 `CornerRadius` 枚举

```swift
enum Layout {
    static let padding8: CGFloat = 8
    static let padding16: CGFloat = 16
    static let padding24: CGFloat = 24
    static let padding32: CGFloat = 32
}

enum CornerRadius: CGFloat {
    case small = 8
    case medium = 12
    case large = 20
}
```

---

## ViewController 规范

### 基本结构

```swift
final class HomeVC: UIViewController {
    private let viewModel: HomeViewModel

    private lazy var collectionView: UICollectionView = {
        let cv = UICollectionView(frame: .zero, collectionViewLayout: createLayout())
        cv.register(Cell.self, forCellWithReuseIdentifier: Cell.reuseID)
        cv.dataSource = self
        cv.delegate = self
        return cv
    }()

    init(viewModel: HomeViewModel) {
        self.viewModel = viewModel
        super.init(nibName: nil, bundle: nil)
    }

    required init?(coder: NSCoder) { fatalError() }

    override func viewDidLoad() {
        super.viewDidLoad()
        setupUI()
        bindViewModel()
    }

    override func viewWillAppear(_ animated: Bool) {
        super.viewWillAppear(animated)
        viewModel.loadData()
    }
}
```

### 命名与职责
- 命名后缀：`XxxVC`（不是 XxxViewController）
- 遵循 **MVVM**，VC 只负责绑定和响应，禁止在 VC 里写业务逻辑
- 生命周期：viewDidLoad 只做初始化，数据请求放 viewWillAppear 或 ViewModel
- 释放：注意 delegate/closure 循环引用，必要时用 `[weak self]`

---

## SnapKit 布局规范

### 标准布局模式

```swift
private func setupUI() {
    view.addSubview(collectionView)

    collectionView.snp.makeConstraints { make in
        make.top.equalTo(view.safeAreaLayoutGuide.snp.top)
        make.leading.trailing.equalToSuperview()
        make.bottom.equalTo(view.safeAreaLayoutGuide.snp.bottom)
    }
}
```

### 常见布局场景

```swift
// 垂直堆叠
stackView.snp.makeConstraints { make in
    make.top.equalTo(view.safeAreaLayoutGuide).offset(Layout.padding16)
    make.leading.trailing.equalToSuperview().inset(Layout.padding16)
}

// 固定高度 + 宽高比
avatarView.snp.makeConstraints { make in
    make.size.equalTo(48)
}

// 底部安全区域
bottomButton.snp.makeConstraints { make in
    make.bottom.equalTo(view.safeAreaLayoutGuide).offset(-Layout.padding16)
    make.leading.trailing.equalToSuperview().inset(Layout.padding16)
    make.height.equalTo(50)
}
```

### 已知陷阱
- `snp.makeConstraints` 只调用一次，动态更新用 `snp.updateConstraints`
- `equalToSuperview()` 前确保 view 已添加到父视图
- ScrollView 子视图约束：内容视图的 leading/trailing/top/bottom 必须锚定到 ScrollView 的 contentLayoutGuide，宽/高锚定到 frameLayoutGuide

---

## UICollectionView + Compositional Layout

### 标准布局创建

```swift
private func createLayout() -> UICollectionViewCompositionalLayout {
    UICollectionViewCompositionalLayout { sectionIndex, _ in
        switch Section(rawValue: sectionIndex) {
        case .banner: return Self.bannerSection()
        case .grid: return Self.gridSection()
        case .list: return Self.listSection()
        default: return nil
        }
    }
}

private static func gridSection() -> NSCollectionLayoutSection {
    let itemSize = NSCollectionLayoutSize(
        widthDimension: .fractionalWidth(0.5),
        heightDimension: .estimated(200)
    )
    let item = NSCollectionLayoutItem(layoutSize: itemSize)
    item.contentInsets = NSDirectionalEdgeInsets(top: 8, leading: 8, bottom: 8, trailing: 8)

    let groupSize = NSCollectionLayoutSize(
        widthDimension: .fractionalWidth(1.0),
        heightDimension: .estimated(200)
    )
    let group = NSCollectionLayoutGroup.horizontal(layoutSize: groupSize, subitems: [item])

    let section = NSCollectionLayoutSection(group: group)
    section.contentInsets = NSDirectionalEdgeInsets(top: 16, leading: 8, bottom: 16, trailing: 8)
    return section
}

private static func listSection() -> NSCollectionLayoutSection {
    let itemSize = NSCollectionLayoutSize(
        widthDimension: .fractionalWidth(1.0),
        heightDimension: .estimated(72)
    )
    let item = NSCollectionLayoutItem(layoutSize: itemSize)
    let group = NSCollectionLayoutGroup.vertical(layoutSize: itemSize, subitems: [item])
    return NSCollectionLayoutSection(group: group)
}
```

### Cell 注册与复用

```swift
// 统一复用标识
protocol ReusableView {
    static var reuseID: String { get }
}
extension ReusableView {
    static var reuseID: String { String(describing: self) }
}
extension UICollectionViewCell: ReusableView {}

// 注册
collectionView.register(Cell.self, forCellWithReuseIdentifier: Cell.reuseID)

// 出列
let cell = collectionView.dequeueReusableCell(withReuseIdentifier: Cell.reuseID, for: indexPath) as! Cell
```

---

## MVVM 绑定模式

### Closure 绑定（推荐，轻量无依赖）

```swift
final class HomeViewModel {
    var onItemsUpdated: (() -> Void)?
    var onError: ((AppError) -> Void)?
    var onLoadingStateChanged: ((Bool) -> Void)?

    private(set) var items: [Item] = [] {
        didSet { onItemsUpdated?() }
    }

    func loadData() {
        onLoadingStateChanged?(true)
        service.fetchItems { [weak self] result in
            self?.onLoadingStateChanged?(false)
            switch result {
            case .success(let items): self?.items = items
            case .failure(let error): self?.onError?(error)
            }
        }
    }
}

// VC 中绑定
private func bindViewModel() {
    viewModel.onItemsUpdated = { [weak self] in
        self?.collectionView.reloadData()
    }
    viewModel.onError = { [weak self] error in
        self?.showError(error)
    }
    viewModel.onLoadingStateChanged = { [weak self] isLoading in
        self?.loadingView.isHidden = !isLoading
    }
}
```

### Combine 绑定（项目已引入 Combine 时）

```swift
final class HomeViewModel {
    @Published var items: [Item] = []
    @Published var isLoading: Bool = false

    private var cancellables = Set<AnyCancellable>()
}

// VC 中绑定
private func bindViewModel() {
    viewModel.$items
        .receive(on: DispatchQueue.main)
        .sink { [weak self] _ in self?.collectionView.reloadData() }
        .store(in: &cancellables)
}
```

---

## 导航规范

### 路由集中管理

```swift
enum Route {
    case settings
    case paywall
    case camera
    case detail(itemID: String)
}

final class Router {
    private weak var navigationController: UINavigationController?

    init(navigationController: UINavigationController) {
        self.navigationController = navigationController
    }

    func navigate(to route: Route) {
        switch route {
        case .settings:
            let vc = SettingsVC(viewModel: SettingsViewModel())
            navigationController?.pushViewController(vc, animated: true)
        case .paywall:
            let vc = PaywallVC(viewModel: PaywallViewModel())
            vc.modalPresentationStyle = .fullScreen
            navigationController?.present(vc, animated: true)
        case .camera:
            let vc = CameraVC(viewModel: CameraViewModel())
            navigationController?.pushViewController(vc, animated: true)
        case .detail(let itemID):
            let vm = DetailViewModel(itemID: itemID)
            let vc = DetailVC(viewModel: vm)
            navigationController?.pushViewController(vc, animated: true)
        }
    }
}
```

- **禁止在 VC 中硬编码跳转目标**，统一走 Router
- Push 前检查 `navigationController` 是否存在
- Modal 弹出统一用 `modalPresentationStyle = .fullScreen`（非 sheet，除非设计明确要求）

---

## 组件规范

### 列表
- **UICollectionView + Compositional Layout**，禁止新建 UITableView
- Cell 必须实现 `configure(with:)` 方法，禁止在 VC 中配置 Cell 内容

```swift
final class ItemCell: UICollectionViewCell {
    private let titleLabel = UILabel()
    private let subtitleLabel = UILabel()

    func configure(with item: Item) {
        titleLabel.text = item.title
        subtitleLabel.text = item.subtitle
    }
}
```

### 弹窗
- 自定义 ViewController + present，禁止使用第三方弹窗库
- 统一封装 `AlertHelper`：

```swift
enum AlertHelper {
    static func show(on vc: UIViewController, title: String, message: String, actions: [UIAlertAction] = []) {
        let alert = UIAlertController(title: title, message: message, preferredStyle: .alert)
        if actions.isEmpty {
            alert.addAction(UIAlertAction(title: "确定", style: .default))
        } else {
            actions.forEach { alert.addAction($0) }
        }
        vc.present(alert, animated: true)
    }
}
```

### 图片加载
- **Kingfisher**（网络图）/ 直接 UIImage（本地资源）
- 占位图统一使用项目内 `placeholder` 资源

### 加载状态
- 使用项目内 `LoadingView` 组件，禁止使用第三方 HUD

---

## 键盘适配

```swift
// 注册键盘通知
NotificationCenter.default.addObserver(self, selector: #selector(keyboardWillShow),
                                       name: UIResponder.keyboardWillShowNotification, object: nil)
NotificationCenter.default.addObserver(self, selector: #selector(keyboardWillHide),
                                       name: UIResponder.keyboardWillHideNotification, object: nil)

@objc private func keyboardWillShow(_ notification: Notification) {
    guard let frame = notification.keyboardFrameEnd else { return }
    let inset = frame.height - view.safeAreaInsets.bottom
    scrollView.contentInset.bottom = inset
    scrollView.verticalScrollIndicatorInsets.bottom = inset
}

@objc private func keyboardWillHide(_ notification: Notification) {
    scrollView.contentInset.bottom = 0
    scrollView.verticalScrollIndicatorInsets.bottom = 0
}
```

- 点击空白收起键盘：`view.addGestureRecognizer(UITapGestureRecognizer(target: view, action: #selector(UIView.endEditing)))`
- 禁止在 `keyboardWillShow` 中做动画（系统已自带），只调整 inset

---

## 暗色模式适配

- 所有颜色必须通过 `AppColors` 定义，支持 `UITraitCollection` 动态切换
- 图片资源：Asset Catalog 中同时提供 Light / Dark 变体
- 禁止用 `overrideUserInterfaceStyle = .light` 强制亮色（除非设计明确要求）
- 测试：Xcode → Environment Overrides → Dark Appearance

---

## 已知陷阱

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| Safe Area 底部留白不足 | iPhone X+ 底部 Home Indicator | 约束锚定 `safeAreaLayoutGuide`，不用 `bottomLayoutGuide` |
| ScrollView 内容不滚动 | 内容高度未超过 ScrollView | 确保内容约束完整，底部约束锚定到 contentLayoutGuide.bottom |
| Cell 复用闪烁 | 异步加载图片后复用 | 图片加载前先取消前一次请求（Kingfisher 自动处理） |
| 约束冲突日志 | 临时约束和正式约束冲突 | `snp.makeConstraints` 只调用一次，更新用 `updateConstraints` |
| Dark Mode 颜色异常 | 硬编码 `UIColor.white/black` | 全部替换为 `AppColors` 动态颜色 |
| 键盘遮挡输入框 | 未调整 ScrollView inset | 注册键盘通知，动态调整 `contentInset` |
| Navigation Bar 滚动隐藏 | `largeTitleDisplayMode` 设置时机 | 在 `viewWillAppear` 中设置，不在 `init` 中 |
| CollectionView 闪烁 | `reloadData()` 导致全量刷新 | 优先使用 `performBatchUpdates` / `reloadSections` |

---

## 禁止事项
- 禁止在 VC 里直接操作数据库或网络
- 禁止使用 `.xib` 或 Storyboard 创建新组件
- 禁止 `DispatchQueue.main.async` 嵌套超过一层
- 禁止在 UI 初始化之外的地方修改 UI（必须在主线程）
- 禁止硬编码 `UIColor(hex:)` 到业务代码
- 禁止使用 `UITableView` 创建新列表（统一 UICollectionView）
- 禁止使用第三方弹窗库（MBProgressHUD / SVProgressHUD 等）
