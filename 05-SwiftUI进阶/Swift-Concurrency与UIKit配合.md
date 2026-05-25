# Swift Concurrency 与 UIKit 配合

> 🎯 **本章目标**：掌握在 UIKit 项目中使用 Swift Concurrency 的方法，学会 @MainActor 与 UIKit 的配合，理解 ViewController 生命周期与 Task 的协调，能够将 Delegate/闭包模式迁移为 async 接口。

---

## UIKit 项目中的 Concurrency 挑战

### UIKit 的线程模型：主线程更新 UI

UIKit 从诞生之初就遵循一个核心规则：所有 UI 操作必须在主线程执行。这一约定源于 UIKit 内部的非线程安全设计——UIView、UIViewController 等核心类的属性和方法都没有加锁保护，跨线程访问会导致不可预测的崩溃或渲染异常。

在传统开发中，我们通过 GCD 将耗时操作放到后台队列，再切回主线程更新 UI：

```swift
DispatchQueue.global().async {
    let data = fetchData()
    DispatchQueue.main.async {
        self.label.text = "完成"
    }
}
```

这种嵌套回调的方式虽然可行，但随着业务复杂度增加，回调层级越来越深，错误处理分散，代码可读性急剧下降。

### 传统 GCD/闭包模式的问题

| 问题 | 说明 |
|------|------|
| 回调地狱 | 多层嵌套闭包，代码呈金字塔形缩进 |
| 错误处理分散 | 每层闭包都需要单独处理错误，容易遗漏 |
| 取消支持弱 | GCD 没有原生取消机制，需手动维护标志位 |
| 上下文捕获风险 | 闭包隐式捕获 self，容易造成循环引用 |
| 执行顺序不直观 | 多个异步操作的串行/并行编排代码难以阅读 |

### Swift Concurrency 在 UIKit 中的优势

Swift Concurrency 通过编译器和运行时的协作，从根本上解决了上述问题：

- **结构化并发**：Task 的生命周期有明确的作用域，子 Task 随父 Task 取消而取消
- **async/await 直线逻辑**：异步代码可以像同步代码一样顺序编写
- **Actor 隔离**：编译器强制数据访问安全，消除数据竞争
- **协作式取消**：TaskCancellationCheck 让取消逻辑融入业务流程
- **@MainActor 自动调度**：标注后编译器保证主线程执行，无需手动切线程

### SwiftUI vs UIKit 中 Concurrency 的差异

| 维度 | SwiftUI | UIKit |
|------|---------|-------|
| UI 线程保证 | 视图体自动在 MainActor 执行 | 需手动标注 @MainActor |
| Task 启动位置 | `.task` 修饰符自动绑定生命周期 | 需在 viewDidLoad 等方法中手动创建 |
| 取消时机 | 视图消失自动取消 | 需在 viewWillDisappear 中手动取消 |
| 状态更新 | @State/@Observable 自动刷新 | 需手动更新 UI 属性 |
| 动画异步化 | withAnimation 配合 async | 需封装 UIView.animate |
| Delegate 处理 | 较少使用 Delegate | 大量 Delegate 需桥接为 async |

> 💡 **提示**：SwiftUI 的 Concurrency 体验更流畅，但 UIKit 项目无需迁移到 SwiftUI 也能充分享受 Swift Concurrency 的优势。关键在于理解 @MainActor 和 Task 生命周期的管理。

---

## @MainActor 与 UIKit

### @MainActor 标注 ViewController

UIViewController 的所有属性和方法都应该在主线程上访问。使用 `@MainActor` 标注整个类是最直接的方式：

```swift
@MainActor
class ProfileViewController: UIViewController {
    let nameLabel = UILabel()
    let avatarImageView = UIImageView()

    override func viewDidLoad() {
        super.viewDidLoad()
        setupUI()
        loadProfile()
    }

    func setupUI() {
        view.addSubview(nameLabel)
        view.addSubview(avatarImageView)
    }

    func loadProfile() {
        Task {
            let profile = await fetchProfile()
            nameLabel.text = profile.name
            avatarImageView.image = profile.avatar
        }
    }
}
```

标注 `@MainActor` 后，类的所有方法和属性默认在主线程执行。在 `loadProfile` 中，`await fetchProfile()` 会挂起当前 Task 并让出主线程，等数据返回后自动回到主线程继续执行后续 UI 更新。

### @MainActor 标注 UIView 方法

对于自定义 UIView 中的配置方法，同样需要 `@MainActor` 保护：

```swift
@MainActor
class CardView: UIView {
    let titleLabel = UILabel()
    let subtitleLabel = UILabel()

    func configure(with model: CardModel) {
        titleLabel.text = model.title
        subtitleLabel.text = model.subtitle
        backgroundColor = model.backgroundColor
    }
}
```

如果从非 @MainActor 上下文调用 `configure(with:)`，编译器会强制要求使用 `await`，确保调用切到主线程：

```swift
Task.detached {
    let model = CardModel(title: "标题", subtitle: "副标题", backgroundColor: .white)
    await cardView.configure(with: model)
}
```

### @MainActor 与 IBOutlet 的关系

IBOutlet 连接的视图属性天然需要在主线程访问。将 ViewController 标注为 `@MainActor` 后，IBOutlet 的访问自动受到保护：

```swift
@MainActor
class SettingsViewController: UIViewController {
    @IBOutlet weak var tableView: UITableView!
    @IBOutlet weak var switchControl: UISwitch!

    override func viewDidLoad() {
        super.viewDidLoad()
        tableView.dataSource = self
        switchControl.isOn = UserDefaults.standard.bool(forKey: "enabled")
    }
}
```

> ⚠️ **警告**：如果 IBOutlet 所在的 ViewController 未标注 `@MainActor`，在后台 Task 中访问 IBOutlet 属性不会产生编译错误，但会在运行时导致不可预测的行为甚至崩溃。务必养成标注 `@MainActor` 的习惯。

### MainActor.run 的使用场景

当你在非 @MainActor 上下文中需要临时执行主线程操作时，使用 `MainActor.run`：

```swift
actor ImageCache {
    private var storage: [String: UIImage] = [:]

    func loadImage(url: URL) async -> UIImage {
        if let cached = storage[url.absoluteString] {
            return cached
        }
        let (data, _) = try! await URLSession.shared.data(from: url)
        let image = await MainActor.run {
            UIImage(data: data)!
        }
        storage[url.absoluteString] = image
        return image
    }
}
```

`UIImage(data:)` 的初始化在主线程更安全，因为 UIImage 内部可能触发渲染资源分配。通过 `MainActor.run` 将这一步切到主线程，执行完毕后自动回到 Actor 上下文。

---

## ViewController 生命周期与 Task

### viewDidLoad 中启动 Task

`viewDidLoad` 是启动异步数据加载的最佳位置。由于 ViewController 已标注 `@MainActor`，在 `viewDidLoad` 中创建的 Task 默认继承 MainActor 上下文：

```swift
@MainActor
class ArticleViewController: UIViewController {
    let textView = UITextView()
    var loadTask: Task<Void, Never>?

    override func viewDidLoad() {
        super.viewDidLoad()
        view.addSubview(textView)
        loadTask = Task {
            await loadArticle()
        }
    }

    func loadArticle() async {
        do {
            let article = try await fetchArticle()
            textView.text = article.content
        } catch {
            textView.text = "加载失败：\(error.localizedDescription)"
        }
    }
}
```

### Task 与 ViewController 生命周期的协调

Task 不会随 ViewController 的释放自动取消。如果 ViewController 被 pop 或 dismiss，但 Task 仍在运行，它可能尝试更新已经不存在或已释放的 UI，导致崩溃或异常。

因此，必须将 Task 的生命周期与 ViewController 绑定：

```swift
@MainActor
class OrderViewController: UIViewController {
    var activeTasks: [Task<Void, Never>] = []

    func startOrderSync() {
        let task = Task {
            await syncOrders()
        }
        activeTasks.append(task)
    }

    deinit {
        activeTasks.forEach { $0.cancel() }
    }
}
```

> 💡 **提示**：`deinit` 中取消 Task 是最后一道防线，更好的做法是在 `viewWillDisappear` 中主动取消，避免不必要的后台计算。

### 视图消失时取消 Task

在 `viewWillDisappear` 中取消不再需要的 Task，可以节省系统资源并避免对不可见 UI 的更新：

```swift
@MainActor
class FeedViewController: UIViewController {
    var refreshTask: Task<Void, Never>?

    override func viewDidLoad() {
        super.viewDidLoad()
        refreshTask = Task {
            await refreshFeed()
        }
    }

    override func viewWillDisappear(_ animated: Bool) {
        super.viewWillDisappear(animated)
        refreshTask?.cancel()
        refreshTask = nil
    }

    override func viewDidAppear(_ animated: Bool) {
        super.viewDidAppear(animated)
        if refreshTask == nil {
            refreshTask = Task {
                await refreshFeed()
            }
        }
    }
}
```

### viewDidAppear / viewWillDisappear 中的异步操作

有些操作需要在视图可见时执行，例如开始定位、启动动画轮播等。利用生命周期方法配合 Task 进行管理：

```swift
@MainActor
class NearbyViewController: UIViewController {
    var locationTask: Task<Void, Never>?
    let collectionView = UICollectionView(frame: .zero, collectionViewLayout: UICollectionViewFlowLayout())

    override func viewDidAppear(_ animated: Bool) {
        super.viewDidAppear(animated)
        locationTask = Task {
            let locations = await fetchNearbyLocations()
            guard !Task.isCancelled else { return }
            updateCollectionView(with: locations)
        }
    }

    override func viewWillDisappear(_ animated: Bool) {
        super.viewWillDisappear(animated)
        locationTask?.cancel()
        locationTask = nil
    }

    func updateCollectionView(with locations: [Location]) {
        collectionView.reloadData()
    }
}
```

在 Task 内部通过 `Task.isCancelled` 检查取消状态，避免在视图已消失后执行无意义的 UI 更新。

---

## Delegate → async 接口迁移

### 传统 Delegate 模式的异步问题

UIKit 大量使用 Delegate 模式处理异步结果，例如图片选择、定位更新、网络下载等。Delegate 将一个完整的异步流程拆散到多个回调方法中，导致：

- 状态分散在多个方法中，难以追踪完整流程
- 多个异步操作串行编排需要层层嵌套
- 取消逻辑需要手动维护标志位
- 代码可读性差，流程不直观

### withCheckedContinuation 桥接 Delegate

`withCheckedContinuation` 是将 Delegate 回调桥接为 async 接口的核心工具。它将一个 continuation 对象注入闭包中，在 Delegate 回调中调用 `resume` 即可将结果返回给 await 调用方：

```swift
func pickImage() async -> UIImage? {
    await withCheckedContinuation { continuation in
        let picker = UIImagePickerController()
        picker.delegate = SomeDelegateHandler(continuation: continuation)
        present(picker, animated: true)
    }
}
```

> ⚠️ **警告**：每个 continuation 必须且只能 resume 一次。重复 resume 会导致运行时崩溃，不 resume 则会泄漏 Task。务必确保在所有代码路径（包括错误路径）上恰好调用一次 resume。

### 常见 UIKit Delegate 的 async 封装

封装的核心思路是创建一个中间对象，让它作为 Delegate 接收回调，再通过 continuation 将结果传回 async 上下文。

### 代码示例：UIImagePickerController async 封装

```swift
@MainActor
class ImagePickerDelegate: NSObject, UIImagePickerControllerDelegate, UINavigationControllerDelegate {
    private var continuation: CheckedContinuation<UIImage?, Never>?

    func setContinuation(_ continuation: CheckedContinuation<UIImage?, Never>) {
        self.continuation = continuation
    }

    func imagePickerController(_ picker: UIImagePickerController, didFinishPickingMediaWithInfo info: [UIImagePickerController.InfoKey: Any]) {
        let image = info[.originalImage] as? UIImage
        continuation?.resume(returning: image)
        continuation = nil
    }

    func imagePickerControllerDidCancel(_ picker: UIImagePickerController) {
        continuation?.resume(returning: nil)
        continuation = nil
    }
}

@MainActor
extension UIViewController {
    func pickImage(sourceType: UIImagePickerController.SourceType = .photoLibrary) async -> UIImage? {
        let picker = UIImagePickerController()
        picker.sourceType = sourceType

        return await withCheckedContinuation { continuation in
            let delegate = ImagePickerDelegate()
            delegate.setContinuation(continuation)
            picker.delegate = delegate
            objc_setAssociatedObject(picker, &AssociatedKey.delegate, delegate, .OBJC_ASSOCIATION_RETAIN)
            present(picker, animated: true)
        }
    }
}

private struct AssociatedKey {
    nonisolated(unsafe) static var delegate = "imagePickerDelegate"
}
```

这里使用 `objc_setAssociatedObject` 保持 delegate 的强引用，避免 ARC 释放导致回调丢失。

### 代码示例：CLLocationManager async 封装

```swift
@MainActor
class LocationDelegate: NSObject, CLLocationManagerDelegate {
    private var continuation: CheckedContinuation<CLLocation, Error>?

    func setContinuation(_ continuation: CheckedContinuation<CLLocation, Error>) {
        self.continuation = continuation
    }

    func locationManager(_ manager: CLLocationManager, didUpdateLocations locations: [CLLocation]) {
        if let location = locations.first {
            continuation?.resume(returning: location)
            continuation = nil
        }
    }

    func locationManager(_ manager: CLLocationManager, didFailWithError error: Error) {
        continuation?.resume(throwing: error)
        continuation = nil
    }
}

@MainActor
func requestCurrentLocation() async throws -> CLLocation {
    let manager = CLLocationManager()
    return try await withCheckedThrowingContinuation { continuation in
        let delegate = LocationDelegate()
        delegate.setContinuation(continuation)
        manager.delegate = delegate
        objc_setAssociatedObject(manager, &LocationAssociatedKey.delegate, delegate, .OBJC_ASSOCIATION_RETAIN)
        manager.requestLocation()
    }
}

private struct LocationAssociatedKey {
    nonisolated(unsafe) static var delegate = "locationDelegate"
}
```

使用 `withCheckedThrowingContinuation` 可以在 Delegate 回调中抛出错误，适用于定位失败等需要错误处理的场景。

---

## 闭包回调 → async/await 迁移

### URLSession async API

Apple 从 iOS 15 开始为 URLSession 提供了原生 async API，无需再用闭包回调：

```swift
func fetchUser() async throws -> User {
    let url = URL(string: "https://api.example.com/user")!
    let (data, response) = try await URLSession.shared.data(from: url)
    guard let httpResponse = response as? HTTPURLResponse,
          (200...299).contains(httpResponse.statusCode) else {
        throw APIError.badResponse
    }
    return try JSONDecoder().decode(User.self, from: data)
}
```

对比传统闭包写法：

| 维度 | 闭包回调 | async/await |
|------|----------|-------------|
| 代码结构 | 嵌套闭包 | 直线顺序 |
| 错误处理 | 回调中判断 | try/catch 统一处理 |
| 多请求串行 | 嵌套加深 | 顺序 await |
| 多请求并行 | DispatchGroup | async let / TaskGroup |
| 可读性 | 低 | 高 |

### 第三方库闭包 → async 封装

许多第三方库仍使用闭包回调。可以通过 `withCheckedThrowingContinuation` 将其封装为 async 接口：

```swift
func loadImage(url: URL) async throws -> UIImage {
    try await withCheckedThrowingContinuation { continuation in
        ImageDownloader.shared.download(url) { result in
            switch result {
            case .success(let image):
                continuation.resume(returning: image)
            case .failure(let error):
                continuation.resume(throwing: error)
            }
        }
    }
}
```

> 💡 **提示**：封装第三方库的闭包为 async 接口时，建议将封装函数放在 extension 中，与原始 API 保持清晰的层次关系。这样库更新后只需调整封装层，业务代码无需变动。

### UIKit 动画 completion → async

UIView.animate 的 completion 闭包也可以封装为 async 形式，让动画链的编排更加清晰：

```swift
@MainActor
func animate(duration: TimeInterval, animations: @escaping () -> Void) async {
    await withCheckedContinuation { continuation in
        UIView.animate(
            withDuration: duration,
            animations: animations
        ) { _ in
            continuation.resume()
        }
    }
}
```

使用示例——顺序执行多段动画：

```swift
@MainActor
class OnboardingViewController: UIViewController {
    let logoView = UIImageView()
    let label = UILabel()

    func playAnimationSequence() async {
        logoView.alpha = 0
        label.alpha = 0

        await animate(duration: 0.5) {
            self.logoView.alpha = 1
        }

        await animate(duration: 0.3) {
            self.label.alpha = 1
        }
    }
}
```

对比传统闭包嵌套，async 写法的动画链逻辑一目了然，无需层层缩进。

---

## 常见陷阱与最佳实践

### 主线程阻塞风险

`await` 虽然不会阻塞线程，但 `@MainActor` 上的长计算仍会阻塞主线程。以下代码看似合理，实际上会造成卡顿：

```swift
@MainActor
class DataViewController: UIViewController {
    func processLargeDataset() async {
        let result = performHeavyCalculation()
        updateUI(with: result)
    }
}
```

`performHeavyCalculation()` 是同步函数，在 @MainActor 上下文中直接执行会占据主线程。正确做法是将计算移到非 MainActor 上下文：

```swift
@MainActor
class DataViewController: UIViewController {
    func processLargeDataset() async {
        let result = await Task.detached {
            return self.performHeavyCalculation()
        }.value
        updateUI(with: result)
    }
}
```

### Task 泄漏与内存问题

未持有引用的 Task 会脱离结构化并发体系，成为"孤儿 Task"。它不会随 ViewController 释放而取消，可能持有对 ViewController 的强引用导致内存泄漏：

```swift
@MainActor
class ListViewController: UIViewController {
    func loadData() {
        Task {
            let items = await fetchItems()
            self.tableView.reloadData()
        }
    }
}
```

这个 Task 强引用了 `self`，即使 ViewController 被 pop，Task 仍会继续运行直到完成。解决方案：

```swift
@MainActor
class ListViewController: UIViewController {
    var dataTask: Task<Void, Never>?

    func loadData() {
        dataTask = Task { [weak self] in
            let items = await fetchItems()
            guard let self, !Task.isCancelled else { return }
            self.tableView.reloadData()
        }
    }

    override func viewWillDisappear(_ animated: Bool) {
        super.viewWillDisappear(animated)
        dataTask?.cancel()
        dataTask = nil
    }
}
```

### 数据竞争与 Actor 隔离

UIKit 对象是 @MainActor 隔离的，但后台 Task 可能意外访问 UI 属性。Swift 6 严格并发模式下，编译器会捕获这类错误；但在 Swift 5 模式下，需要开发者自行注意：

```swift
@MainActor
class CounterViewController: UIViewController {
    let countLabel = UILabel()
    var count = 0

    func startCounting() {
        Task {
            while !Task.isCancelled {
                try? await Task.sleep(for: .seconds(1))
                count += 1
                countLabel.text = "\(count)"
            }
        }
    }
}
```

如果 `count` 属性被非 @MainActor 的 Task 访问，就会产生数据竞争。确保所有访问 UI 相关状态的代码都在 @MainActor 上下文中执行。

### UIKit + Concurrency Checklist

| 检查项 | 说明 |
|--------|------|
| ViewController 标注 @MainActor | 确保所有 UI 操作在主线程 |
| Task 持有引用 | 将需要管理的 Task 存为属性 |
| viewWillDisappear 取消 Task | 视图消失时取消不再需要的异步操作 |
| deinit 取消 Task | 作为最后一道防线 |
| continuation 只 resume 一次 | 所有路径（含错误）恰好调用一次 |
| 同步长计算移出 MainActor | 使用 Task.detached 或非 @MainActor 函数 |
| weak self 防循环引用 | Task 闭包中捕获 self 时注意引用关系 |
| 检查 Task.isCancelled | 长时间运行的 Task 中定期检查取消状态 |

---

## 小结

| 主题 | 核心要点 |
|------|----------|
| UIKit Concurrency 挑战 | UIKit 要求主线程更新 UI，传统 GCD 闭包模式存在回调地狱、取消困难等问题 |
| @MainActor 与 UIKit | 标注 ViewController/View 为 @MainActor，编译器自动保证主线程执行；MainActor.run 用于临时切回主线程 |
| 生命周期与 Task | Task 不随 ViewController 释放自动取消，需在 viewWillDisappear 中手动取消，deinit 作为兜底 |
| Delegate → async | 使用 withCheckedContinuation 桥接 Delegate 回调，注意 continuation 只能 resume 一次 |
| 闭包 → async/await | URLSession 已有原生 async API；第三方闭包用 continuation 封装；动画 completion 也可 async 化 |
| 陷阱与最佳实践 | 避免主线程长计算、防止 Task 泄漏、注意数据竞争、遵循 Checklist |

Swift Concurrency 不是 SwiftUI 的专属特性。在 UIKit 项目中合理运用 @MainActor、结构化 Task 管理和 async 接口封装，同样可以大幅提升代码的可读性和可维护性。关键在于理解 MainActor 的调度机制、Task 生命周期的手动管理，以及 continuation 桥接的正确使用方式。

← [UIKit 与 SwiftUI 互操作](./UIKit与SwiftUI互操作.md) | [Sign in with Apple 与第三方登录](./Sign-in-with-Apple与第三方登录.md) →