---
name: performance-debug
description: 涉及性能优化、Instruments、启动优化、内存泄漏、包体积瘦身、卡顿排查、卡顿监控、耗电优化的任务
---

# 性能优化与调试

## 性能指标基准

| 指标 | 目标值 | 超标影响 |
|------|--------|---------|
| 冷启动时间 | < 400ms（首屏可交互） | 用户流失 |
| 热启动时间 | < 200ms | 体验差 |
| 帧率 | 稳定 60fps（Pro 120fps） | 卡顿感 |
| 内存占用 | < 200MB（常规 App） | 系统杀进程 |
| 包体积 | < 50MB（下载大小） | 下载转化率低 |
| ANR（主线程卡顿） | > 700ms 即需优化 | 审核风险 |

---

## Instruments 使用

### 常用模板

| 模板 | 用途 | 何时使用 |
|------|------|---------|
| Time Profiler | CPU 热点分析 | 启动慢、操作卡顿 |
| Allocations | 内存分配追踪 | 内存持续增长 |
| Leaks | 内存泄漏检测 | 退出页面后内存不降 |
| Network | 网络请求分析 | 请求慢、流量大 |
| Core Data | CoreData 查询性能 | 数据库操作慢 |
| Hangs | 主线程卡顿检测 | UI 卡顿 |

### Time Profiler 使用要点
- 使用 Release 配置（Debug 模式优化被禁用，数据不准）
- 关注 **Self Time**（自身耗时），而非 Total Time（含子调用）
- 系统库调用（UIKit 等）通常无法优化，聚焦业务代码
- Call Tree 选项：勾选 "Invert Call Tree" + "Hide System Libraries"

### Leaks 检测
- 退出页面后等 10 秒再检查，部分释放是延迟的
- 常见泄漏源：closure 未用 `[weak self]`、delegate 声明为 strong、Timer 未 invalidate
- Instruments Leaks 只能检测**循环引用**，单边泄漏用 Allocations 的 Mark Generation 对比

---

## 启动优化

### 启动阶段分析

```
pre-main 阶段（系统加载）
  → dylib loading（动态库加载）
  → rebase/binding（地址修正）
  → ObjC setup（运行时初始化）
  → initializer（+load 和 C++ 构造函数）

post-main 阶段（App 代码）
  → AppDelegate.didFinishLaunching
  → SceneDelegate.sceneDidBecomeActive
  → 首屏渲染完成
```

### pre-main 优化
- **减少动态库数量**：合并小库，目标 < 6 个非系统动态库
- **移除 `+load` 方法**：改用 `+initialize` 或 dispatch_once
- **减少 `__attribute__((constructor))`**：延迟到使用时初始化
- 检查命令：`DYLD_PRINT_STATISTICS=1` 打印 pre-main 各阶段耗时

### post-main 优化
- **`didFinishLaunching` 只做必须的初始化**（SDK 配置、权限检查）
- 非首屏功能延迟初始化：登录模块在进入登录页时才初始化
- 首屏数据预加载：在 `willFinishLaunching` 阶段发起网络请求
- 首屏渲染优化：减少 VC 层级，避免嵌套滚动视图

### 启动耗时测量

```swift
func application(_ application: UIApplication,
    didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
    let start = ProcessInfo.processInfo.systemUptime
    DispatchQueue.main.async {
        let elapsed = ProcessInfo.processInfo.systemUptime - start
        print("首帧渲染耗时: \(elapsed * 1000)ms")
    }
    return true
}
```

---

## 内存优化

### 常见内存问题

| 问题 | 症状 | 排查方式 |
|------|------|---------|
| 循环引用 | 退出页面内存不降 | Instruments Leaks |
| 缓存无上限 | 内存持续增长 | Allocations Mark Generation |
| 大图片未压缩 | 峰值内存飙升 | Allocations 按大小排序 |
| 定时器未释放 | 后台持续占用 | Leaks + Call Tree |
| 单例持有数据 | 退出登录内存不降 | Allocations 对比 |

### 图片内存优化
- **禁止加载原始大图到内存**：1 张 4000×3000 图片 ≈ 48MB 内存
- 缩放策略：

```swift
func downsample(imageAt url: URL, to pointSize: CGSize, scale: CGFloat = UIScreen.main.scale) -> UIImage? {
    let imageSourceOptions = [kCGImageSourceShouldCache: false] as CFDictionary
    guard let imageSource = CGImageSourceCreateWithURL(url as CFURL, imageSourceOptions) else { return nil }

    let maxDimension = max(pointSize.width, pointSize.height) * scale
    let downsampleOptions = [
        kCGImageSourceCreateThumbnailFromImageAlways: true,
        kCGImageSourceShouldCacheImmediately: true,
        kCGImageSourceCreateThumbnailWithTransform: true,
        kCGImageSourceThumbnailMaxPixelSize: maxDimension
    ] as CFDictionary

    guard let cgImage = CGImageSourceCreateThumbnailAtIndex(imageSource, 0, downsampleOptions) else { return nil }
    return UIImage(cgImage: cgImage)
}
```

### 缓存策略
- NSCache 设置 `countLimit` 和 `totalCostLimit`
- Kingfisher 设置缓存大小上限（默认 200MB）
- 收到内存警告时主动清理：

```swift
NotificationCenter.default.addObserver(forName: UIApplication.didReceiveMemoryWarningNotification, queue: .main) { _ in
    ImageCache.default.clearMemoryCache()
}
```

### autoreleasepool
- 循环中创建大量临时对象时使用：

```swift
for item in largeArray {
    autoreleasepool {
        process(item)
    }
}
```

---

## 包体积瘦身

### 分析命令

```bash
# 查看 App 大小分布
xcrun --sdk iphoneos size -l App.app

# 查看 Asset Catalog 大小
assetutil --info Assets.car > assets.json

# 查看各架构大小
lipo -info App.app/App
```

### 优化手段

| 手段 | 预期收益 | 风险 |
|------|---------|------|
| 移除无用代码 | 5-15% | 低 |
| 压缩图片资源 | 10-30% | 低 |
| App Thinning | 30-50% | 无（自动） |
| 移除未使用的架构 | 40-50%（模拟器） | 仅影响开发 |
| 代码混淆/压缩 | 5-10% | 调试困难 |

### 图片优化
- 使用 Asset Catalog 而非 Bundle 直接放图片（自动 App Thinning）
- 矢量图用 PDF（Single Scale），位图用 HEIF 或 WebP
- **禁止在 Asset Catalog 中放 3x 以上分辨率图片**
- 检查未使用图片：`LSUnusedResources` 工具

### 代码优化
- 编译选项：`STRIP_INSTALLED_PRODUCT = YES`、`DEAD_CODE_STRIPPING = YES`
- `SWIFT_OPTIMIZATION_LEVEL = -O`（Release 模式）
- 移除未使用的 SPM 依赖
- **禁止在 Release 包中嵌入 Debug 符号**

---

## 卡顿排查

### 主线程卡顿检测

```swift
final class LagMonitor {
    private var timer: DispatchSourceTimer?
    private let threshold: TimeInterval = 0.4

    func start() {
        let timer = DispatchSource.makeTimerSource(queue: .global())
        timer.schedule(deadline: .now(), repeating: threshold)
        timer.setEventHandler { [weak self] in
            guard let self else { return }
            if let mainThread = Thread.main, !mainThread.isExecuting {
                return
            }
            let semaphore = DispatchSemaphore(value: 0)
            DispatchQueue.main.async {
                semaphore.signal()
            }
            let timeout = semaphore.wait(timeout: .now() + self.threshold)
            if timeout == .timedOut {
                Logger.warning("主线程卡顿超过 \(self.threshold * 1000)ms")
            }
        }
        timer.resume()
        self.timer = timer
    }

    func stop() {
        timer?.cancel()
        timer = nil
    }
}
```

### 常见卡顿原因
- 主线程 I/O 操作（文件读写、数据库查询）
- 主线程网络请求
- 复杂布局计算（深层嵌套 AutoLayout）
- 大量图片同时解码
- 主线程 JSON 解析大数据
- `layoutSubviews` 中做耗时操作

### 优化手段
- I/O 操作移到后台线程
- 图片预解码 + 异步渲染
- 列表分页加载，避免一次性加载大量数据
- AutoLayout 简化：减少嵌套层级，使用 frame 布局复杂 Cell
- JSON 解析大响应时用 `JSONDecoder` 流式解析

---

## 耗电优化

### 常见耗电源
- 高频定位（`desiredAccuracy = kCLLocationAccuracyBest` 持续使用）
- 后台网络轮询
- 持续 GPS 追踪
- 视频播放未暂停
- 蓝牙持续扫描

### 优化手段
- 定位：用 `kCLLocationAccuracyHundredMeters` 替代 `Best`，用显著位置变化替代持续追踪
- 网络：批量请求，减少请求次数，使用缓存
- 后台任务：用 Background Tasks 替代后台常驻
- 蓝牙：扫描到设备后立即停止扫描

---

## 调试技巧

### LLDB 常用命令
```
po self.view.frame               # 打印视图 frame
po [self.view recursiveDescription]  # 打印视图层级
expression self.backgroundColor = .red  # 运行时修改
breakpoint set -r "viewDidLoad"  # 正则断点
memory graph                     # 内存图（Xcode Debug Navigator）
```

### 线程问题排查
- Main Thread Checker：Xcode → Diagnostics → Main Thread Checker ✅
- 常见问题：UI 更新不在主线程、后台线程操作 CoreData viewContext
- Swift 6 严格并发检查：`Build Settings → Strict Concurrency Checking → Complete`

### 网络调试
- Charles / Proxyman 抓包
- `URLSession` 自定义 `URLProtocol` 记录请求日志
- ATS 例外仅限 Debug：`NSAllowsArbitraryLoads` 在 Release 中必须移除

---

## 已知陷阱

- **Instruments 在模拟器上数据不准**，性能测试必须用真机
- **Debug 模式下 Swift 运行时检查会拖慢性能**，性能测试必须用 Release
- **`UIImage(named:)` 会缓存图片**，大量图片场景用 `UIImage(contentsOfFile:)`
- **CoreData 的 `NSFetchedResultsController` 在大数据量下性能差**，设置 `fetchBatchSize`
- **AutoLayout 复杂度是 O(2^n)**，嵌套超过 10 层时考虑手写 frame
- **`UITableView` 滚动卡顿优先检查 `heightForRowAt`**，确保不在滚动时做复杂计算
