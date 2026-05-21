# 105-App Thinning 与包大小优化

> 🎯 **本章目标**：理解 App Thinning 的三大技术（Slicing、Bitcode、On-Demand Resources）及其工作原理，掌握包大小分析工具的使用方法，学会从图片、代码、资源、架构等多维度优化包大小，并能根据实战 checklist 将 App 从 100MB 优化到 50MB，同时了解 App Store 对包大小的各档限制与审核注意事项。

---

## 1. App Thinning 概述

### 1.1 为什么需要 App Thinning

打个比方：你去超市买一瓶酱油，超市不会把整个仓库的库存都搬给你，而是只给你你需要的那一瓶。App Thinning 的道理一样——用户下载 App 时，不需要下载所有设备架构和所有资源的"全家桶"，只需要下载适配自己设备的那一部分。

在 App Thinning 出现之前，一个 Universal App 的安装包包含了所有架构（arm64、armv7）和所有分辨率的图片资源，用户实际用到的可能只有其中 60%。这导致了：

| 问题 | 影响 |
|------|------|
| 下载体积过大 | 用户在蜂窝网络下不愿下载 |
| 占用存储空间 | 设备空间紧张时用户优先卸载大 App |
| 下载转化率低 | 包越大，用户中途取消下载的比例越高 |
| Apple 服务器带宽浪费 | 传输了大量用户设备不需要的数据 |

### 1.2 150MB 蜂窝下载限制

Apple 对 App 的蜂窝网络下载有严格的大小限制：

| 限制类型 | 大小 | 说明 |
|----------|------|------|
| 蜂窝网络下载限制 | **150MB** | 超过此大小，用户必须连接 Wi-Fi 才能下载 |
| App Store 推荐限制 | **200MB** | 超过此大小，App Store 可能不会在搜索结果中优先展示 |
| 上传大小上限 | **1GB**（压缩后） | 超过此大小无法上传到 App Store Connect |

> 💡 **提示**：150MB 限制是针对**下载大小**（Download Size），即用户实际从 App Store 下载的数据量，而不是安装大小（Install Size）。App Thinning 的目标就是让下载大小尽可能小。

### 1.3 App Thinning 三大技术

App Thinning 包含三个核心技术，它们从不同维度裁剪 App：

| 技术 | 裁剪维度 | 类比 |
|------|----------|------|
| **Slicing** | 按设备特征裁剪资源 | 买衣服只拿你尺码的那件 |
| **Bitcode** | 按设备架构编译中间码 | 定制裁缝根据你的身材现场做衣服 |
| **On-Demand Resources** | 按需下载资源 | 先买基础款，配件以后按需购买 |

三者协同工作的流程：

```
完整 App 包
    │
    ├── Slicing ──→ 按 设备/分辨率/架构 裁剪 → 适配特定设备的精简包
    │
    ├── Bitcode ──→ 上传中间码 → Apple 服务器为每种架构单独编译
    │
    └── ODR ──────→ 基础包 + 按需资源 → 用户先下载核心，需要时再加载
```

> ⚠️ **警告**：Bitcode 已在 Xcode 14 中被弃用，Apple 从 2023 年 4 月起不再接受包含 Bitcode 的 App 提交。虽然了解 Bitcode 的概念仍有价值，但新项目无需再关注此技术。下文会做简要介绍，重点放在 Slicing 和 ODR 上。

---

## 2. App Slicing

### 2.1 Slicing 工作原理

Slicing 是 App Thinning 中最核心的技术。它的原理很简单：Apple 服务器根据用户设备的特征，从完整的 App 包中"切出"该设备需要的部分，然后只把这部分下发给用户。

**Slicing 考虑的设备特征：**

| 特征维度 | 示例 | 裁剪效果 |
|----------|------|----------|
| CPU 架构 | arm64 | 只保留当前架构的二进制代码 |
| 屏幕分辨率 | @2x / @3x | 只保留匹配的图片资源 |
| 设备类型 | iPhone / iPad | 只保留对应设备的资源 |
| 内存容量 | 2GB / 4GB+ | Metal 纹理按设备能力裁剪 |
| 系统版本 | iOS 17 | 只包含兼容的系统框架 |

**Slicing 工作流程：**

```
开发者上传 Universal App
         │
         ▼
App Store Connect 接收
         │
         ▼
Apple 服务器创建不同设备的变体（Variant）
         │
         ├── iPhone 15 Pro 变体（arm64 + @3x + ...）
         ├── iPad Air 变体（arm64 + @2x + ...）
         └── iPhone SE 变体（arm64 + @2x + ...）
         │
         ▼
用户下载时，服务器自动选择对应变体
```

### 2.2 Asset Catalog 自动切片

Asset Catalog 是 Slicing 最主要的应用场景。当你把图片资源放在 Asset Catalog（`.xcassets`）中时，Xcode 和 Apple 服务器会自动完成切片。

**正确使用 Asset Catalog 的姿势：**

```
Images.xcassets/
├── AppIcon.appiconset/
│   └── Contents.json        ← 自动按设备切片
├── LaunchImage.launchimage/
│   └── Contents.json
├── logo.imageset/
│   ├── logo@2x.png          ← 2x 设备用
│   ├── logo@3x.png          ← 3x 设备用
│   └── Contents.json
└── Contents.json
```

**Asset Catalog vs 直接引用图片对比：**

| 对比项 | Asset Catalog | 直接引用 PNG |
|--------|:------------:|:------------:|
| 自动 Slicing | ✅ 支持 | ❌ 不支持 |
| 按 @2x/@3x 切片 | ✅ 自动 | ❌ 全部打包 |
| 按设备类型切片 | ✅ 自动 | ❌ 全部打包 |
| App Thinning 支持 | ✅ 完整 | ❌ 无 |
| 运行时加载 | `UIImage(named:)` | `UIImage(contentsOfFile:)` |
| 缓存管理 | 系统自动管理 | 需手动管理 |

> 💡 **提示**：所有图片资源都应放入 Asset Catalog，而不是直接拖入项目目录。这是获得 Slicing 收益最简单的方式——你不需要做任何额外配置，Xcode 和 Apple 服务器会自动处理。

### 2.3 Xcode 自动处理

Slicing 对开发者几乎是透明的。在开发和打包阶段，Xcode 会自动完成以下处理：

| 阶段 | 自动处理内容 |
|------|-------------|
| 编译时 | Asset Catalog 中的图片按 @2x/@3x 和设备类型编译为 `.car` 文件 |
| Archive 时 | 生成包含所有设备和架构的 Universal App |
| 上传后 | App Store Connect 自动创建各设备变体 |
| 下载时 | 用户设备自动获取匹配的变体 |

你唯一需要做的是：**确保资源放在 Asset Catalog 中**。

### 2.4 查看 App Thinning Size Report

Archive 之后，你可以查看 App Thinning 的大小报告，了解不同设备变体的大小。

**查看步骤：**

| 步骤 | 操作 |
|:----:|------|
| 1 | Product → Archive |
| 2 | 在 Organizer 中选择归档 |
| 3 | 点击 **Distribute App** |
| 4 | 选择 **Development** 或 **Ad Hoc** |
| 5 | 勾选 **App Thinning: All compatible device variants** |
| 6 | 导出完成后，在导出目录中找到 `App Thinning Size Report.txt` |

**报告内容示例：**

```
App Thinning Size Report for MyApp

Variant: iPhone 15 Pro
    App size: 23.4 MB (compressed)
    On-demand resources size: 0 bytes
    Install size: 48.2 MB

Variant: iPad Air (5th generation)
    App size: 25.1 MB (compressed)
    On-demand resources size: 0 bytes
    Install size: 52.7 MB
```

> 💡 **提示**：关注 **compressed** 大小，这是用户实际下载的大小。Install size 是安装后占用的磁盘空间，通常比下载大小大 1.5-2 倍。

---

## 3. On-Demand Resources

### 3.1 ODR 概念

On-Demand Resources（ODR）是 App Thinning 的另一项重要技术，它允许你将部分资源标记为"按需加载"，这些资源不会随 App 一起下载，而是在用户需要时才从 App Store 下载。

类比：买手机时，手机自带 16GB 存储（基础包），但你可以随时从云端下载电影来观看（按需资源）。看完了还可以删掉腾出空间，想看再下载。

**ODR 适用场景：**

| 场景 | 示例 |
|------|------|
| 游戏关卡 | 前几关随 App 下载，后续关卡按需加载 |
| 教程资源 | 入门教程随 App，高级教程按需 |
| 多语言资源 | 仅下载用户选择的语言包 |
| 高清素材 | 默认标清，用户选择时下载高清 |
| 临时资源 | 一次性使用的引导动画、活动素材 |

### 3.2 资源标签 Tag

ODR 通过"标签"（Tag）来组织资源。每个标签代表一组资源，Apple 会根据标签来管理资源的下载和清理。

**标签类型：**

| 标签类型 | 说明 | 下载时机 | 清理时机 |
|----------|------|----------|----------|
| **Initial Install Tags** | 初始安装标签 | 随 App 一起下载 | 不会自动清理 |
| **Prefetched Tag Order** | 预取标签 | App 安装后自动下载 | 存储不足时清理 |
| **Downloaded On Demand** | 按需下载标签 | 代码请求时下载 | 存储不足时清理 |

**在 Xcode 中配置 Tag：**

```
项目设置 → Target → Resource Tags
    │
    ├── 添加 Tag 名称（如 "level_5"）
    ├── 将资源文件分配给 Tag
    └── 设置 Tag 类型（Initial Install / Prefetch / On Demand）
```

**Tag 配置示例：**

| Tag 名称 | 类型 | 包含资源 | 说明 |
|-----------|------|----------|------|
| `tutorial_basic` | Initial Install | 基础教程图片、视频 | 新用户必看 |
| `level_1_4` | Initial Install | 第 1-4 关资源 | 游戏初始关卡 |
| `level_5_8` | Prefetched | 第 5-8 关资源 | 预取，玩家快到时已就绪 |
| `level_9_12` | On Demand | 第 9-12 关资源 | 玩家到达时才下载 |
| `hd_textures` | On Demand | 高清纹理包 | 用户手动选择下载 |

### 3.3 NSBundleResourceRequest 使用

使用 ODR 需要通过 `NSBundleResourceRequest` API 来请求资源。

**基本用法：**

```swift
import UIKit

class LevelManager {
    private var currentRequest: NSBundleResourceRequest?

    func loadLevel(_ levelNumber: Int, completion: @escaping (Result<Void, Error>) -> Void) {
        let tag = "level_\(levelNumber)"

        let request = NSBundleResourceRequest(tags: Set([tag]))
        currentRequest = request

        request.loadingPriority = NSBundleResourceRequestLoadingPriorityUrgent

        request.beginAccessingResources { error in
            if let error = error {
                completion(.failure(error))
            } else {
                completion(.success(()))
            }
        }
    }

    func unloadLevel(_ levelNumber: Int) {
        let tag = "level_\(levelNumber)"
        let request = NSBundleResourceRequest(tags: Set([tag]))
        request.endAccessingResources()
    }
}
```

**加载资源后使用：**

```swift
let manager = LevelManager()

manager.loadLevel(5) { result in
    switch result {
    case .success:
        let image = UIImage(named: "level5_background")
        let path = Bundle.main.path(forResource: "level5_data", ofType: "json")
    case .failure(let error):
        print("资源加载失败: \(error.localizedDescription)")
    }
}
```

### 3.4 优先级管理

当多个资源需要同时下载时，可以通过设置优先级来控制下载顺序：

```swift
let request = NSBundleResourceRequest(tags: ["level_5", "level_6"])

request.loadingPriority = NSBundleResourceRequestLoadingPriorityUrgent

request.setProgressHandler { progress in
    print("下载进度: \(Int(progress.fractionCompleted * 100))%")
}

request.beginAccessingResources { error in
    if let error = error {
        print("加载失败: \(error)")
    } else {
        print("资源加载完成")
    }
}
```

**优先级常量：**

| 常量 | 值 | 使用场景 |
|------|:--:|----------|
| `LoadingPriorityUrgent` | 1.0 | 用户正在等待的资源 |
| `LoadingPriorityHigh` | 0.5 | 即将需要的资源 |
| `LoadingPriorityLow` | 0.0 | 可以慢慢下载的资源 |

### 3.5 实战：游戏关卡按需加载

下面是一个完整的游戏关卡按需加载方案：

```swift
import UIKit

class GameLevelLoader {
    private var activeRequests: [String: NSBundleResourceRequest] = [:]
    private let tagsPerLevel: [Int: String] = [
        1: "level_1_4",
        2: "level_1_4",
        3: "level_1_4",
        4: "level_1_4",
        5: "level_5_8",
        6: "level_5_8",
        7: "level_5_8",
        8: "level_5_8",
        9: "level_9_12",
        10: "level_9_12",
        11: "level_9_12",
        12: "level_9_12"
    ]

    func preloadNextLevels(currentLevel: Int) {
        let nextGroup = (currentLevel / 4) + 1
        guard let tag = tagsPerLevel[nextGroup * 4 + 1] else { return }

        if activeRequests[tag] != nil { return }

        let request = NSBundleResourceRequest(tags: [tag])
        request.loadingPriority = NSBundleResourceRequestLoadingPriorityHigh
        activeRequests[tag] = request

        request.beginAccessingResources { [weak self] error in
            if error != nil {
                self?.activeRequests.removeValue(forKey: tag)
            }
        }
    }

    func loadLevel(_ level: Int, completion: @escaping (Bool) -> Void) {
        guard let tag = tagsPerLevel[level] else {
            completion(false)
            return
        }

        if let existing = activeRequests[tag] {
            existing.loadingPriority = NSBundleResourceRequestLoadingPriorityUrgent
            completion(true)
            return
        }

        let request = NSBundleResourceRequest(tags: [tag])
        request.loadingPriority = NSBundleResourceRequestLoadingPriorityUrgent
        activeRequests[tag] = request

        request.beginAccessingResources { [weak self] error in
            if error != nil {
                self?.activeRequests.removeValue(forKey: tag)
                completion(false)
            } else {
                completion(true)
            }
        }
    }

    func cleanupOldLevels(currentLevel: Int) {
        let currentTag = tagsPerLevel[currentLevel] ?? ""
        for (tag, request) in activeRequests {
            if tag != currentTag {
                request.endAccessingResources()
                activeRequests.removeValue(forKey: tag)
            }
        }
    }
}
```

> ⚠️ **警告**：ODR 资源在系统存储不足时可能被自动清理。因此每次使用前都应检查资源是否可用，如果不可用需要重新请求下载。不要假设资源一旦下载就永远存在。

---

## 4. 包大小分析工具

### 4.1 Xcode Archive 后的 App Thinning Size Report

这是最直接的包大小分析方式，在上一节已介绍查看步骤。报告中的关键字段：

| 字段 | 含义 | 关注重点 |
|------|------|----------|
| App size (compressed) | 压缩后的下载大小 | 是否超过 150MB |
| Install size | 安装后占用空间 | 用户体验参考 |
| On-demand resources size | ODR 资源大小 | ODR 配置是否合理 |

### 4.2 App Store Connect 大小报告

App 上传到 App Store Connect 后，可以在后台查看更详细的大小信息：

**查看路径：**

```
App Store Connect → 我的 App → 选择 App → 活动选项卡 → 选择构建版本 → 构建版本元数据
```

**报告内容：**

| 数据项 | 说明 |
|--------|------|
| 通用安装大小 | 所有设备上的最大安装大小 |
| 通用下载大小 | 所有设备上的最大下载大小 |
| 各设备安装大小 | 按设备类型分别列出 |
| 各设备下载大小 | 按设备类型分别列出 |

### 4.3 命令行工具分析

对于更精细的分析，可以使用命令行工具：

**1. 查看 .app 包内容：**

```bash
# 查看 App 包结构
ls -la MyApp.app/

# 查看各文件大小
du -sh MyApp.app/*
```

**2. 分析 Asset Catalog：**

```bash
# 使用 assetutil 分析 .car 文件
assetutil --info Assets.car > assets_info.json

# 筛选大于 100KB 的图片
cat assets_info.json | python3 -c "
import json, sys
data = json.load(sys.stdin)
for item in data:
    size = item.get('Render Time', 0)
    name = item.get('Name', 'unknown')
    if size > 100:
        print(f'{name}: {size}KB')
"
```

**3. 分析 Mach-O 二进制大小：**

```bash
# 查看各架构大小
lipo -detailed_info MyApp.app/MyApp

# 查看 Segments 大小
size -m MyApp.app/MyApp

# 查看符号表
nm -gU MyApp.app/MyApp | wc -l
```

**4. 分析链接库：**

```bash
# 查看动态库依赖
otool -L MyApp.app/MyApp

# 查看所有 Framework
ls -la MyApp.app/Frameworks/
du -sh MyApp.app/Frameworks/*
```

**常用分析工具对比：**

| 工具 | 用途 | 优点 | 缺点 |
|------|------|------|------|
| `assetutil` | 分析 Asset Catalog | 官方工具，信息详细 | 输出为 JSON，需二次处理 |
| `size` | 分析二进制段大小 | 快速直观 | 信息较粗略 |
| `otool` | 分析链接库和符号 | 功能强大 | 输出冗长 |
| `du` | 查看目录大小 | 简单直接 | 无法区分压缩前后 |
| Instruments | 内存和磁盘分析 | 可视化 | 需运行 App |

---

## 5. 包大小优化策略

### 5.1 Asset Catalog 优化

**1. 确保所有图片都在 Asset Catalog 中**

```bash
# 查找项目中不在 xcassets 中的图片
find . -name "*.png" -o -name "*.jpg" | grep -v ".xcassets" | grep -v "Pods"
```

**2. 移除未使用的图片：**

```bash
# 使用工具查找未引用的图片
# 推荐工具：LSUnusedResources
# GitHub: https://github.com/tinymind/LSUnusedResources
```

**3. 合理使用 PDF 矢量图：**

| 方案 | 适用场景 | 优点 | 缺点 |
|------|----------|------|------|
| @2x + @3x PNG | 大多数场景 | 渲染快 | 需要两套图 |
| 单个 PDF 矢量图 | 简单图标、形状 | 一份图适配所有分辨率 | 复杂图形渲染慢 |
| SVG（iOS 13+） | 简单矢量图形 | 标准格式 | 低版本不支持 |

> 💡 **提示**：对于简单图标，使用单个 PDF 矢量图替代 @2x/@3x 两套 PNG，可以减少约 40% 的图片资源体积。

### 5.2 图片压缩

图片通常是包大小的最大贡献者，优化图片效果最显著。

**图片压缩方案对比：**

| 工具/格式 | 压缩率 | 无损/有损 | 适用场景 |
|-----------|:------:|:---------:|----------|
| pngquant | 60-80% | 有损 | PNG 照片类图片 |
| ImageOptim | 20-40% | 无损 | 所有 PNG |
| TinyPNG | 50-70% | 有损 | Web 端和移动端通用 |
| WebP | 30-50% | 有损/无损 | 需要解码库支持 |
| HEIC | 40-60% | 有损/无损 | iOS 11+ 原生支持 |

**批量压缩脚本示例：**

```bash
#!/bin/bash
# 使用 ImageOptim-CLI 批量压缩
# 安装：brew install imageoptim-cli

# 压缩 xcassets 中的所有 PNG
find . -name "*.xcassets" -exec imageoptim -a -d {} \;

# 压缩项目中所有 PNG（排除 Pods）
find . -name "*.png" -not -path "*/Pods/*" -not -path "*/.git/*" | xargs imageoptim -a
```

**使用 HEIC 替代 PNG（iOS 11+）：**

```swift
if #available(iOS 11.0, *) {
    if let url = Bundle.main.url(forResource: "hero_image", withExtension: "heic"),
       let data = try? Data(contentsOf: url),
       let image = UIImage(data: data) {
        imageView.image = image
    }
}
```

### 5.3 无用资源清理

**1. 查找未使用的图片资源：**

```bash
# 在源代码中搜索图片引用
# 注意：需要考虑 imageNamed、UIImage(named:) 等多种引用方式
grep -rn "imageNamed" --include="*.swift" --include="*.m" .
grep -rn "UIImage(named:" --include="*.swift" .
```

**2. 查找未使用的文件资源：**

```bash
# 查找大于 500KB 的文件
find . -type f -size +500k -not -path "*/Pods/*" -not -path "*/.git/*" | \
    xargs ls -lhS
```

**3. 清理常见冗余资源：**

| 冗余类型 | 说明 | 清理方法 |
|----------|------|----------|
| @1x 图片 | 早已不需要 | 删除所有 @1x 资源 |
| 未引用的图片 | 代码中未使用 | 工具扫描后删除 |
| 重复图片 | 同一图片多个副本 | 统一引用，删除副本 |
| 调试资源 | 测试用的 mock 数据 | Release 配置中排除 |
| 大型 JSON/XML | 静态数据文件 | 改为服务端下发 |

**4. 在 Build Phase 中排除调试资源：**

```bash
# 在 Build Phases → Run Script 中添加
if [ "${CONFIGURATION}" = "Debug" ]; then
    echo "Debug mode: keeping all resources"
else
    echo "Release mode: removing debug resources"
    rm -rf "${BUILT_RESOURCES_DIR}/debug_data"
fi
```

### 5.4 代码优化

**1. 编译器优化选项：**

| 优化选项 | 位置 | 效果 | 建议 |
|----------|------|------|------|
| Optimization Level → `-O` | Build Settings → Swift Compiler | 减小二进制大小 | Release 使用 `-O` |
| Strip Debug Symbols | Build Settings → Deployment | 移除调试符号 | ✅ 开启 |
| Strip Swift Symbols | Build Settings → Deployment | 移除 Swift 符号 | ✅ 开启 |
| Make Strings Read-Only | Build Settings → Deployment | 字符串常量合并 | ✅ 开启 |
| Dead Code Stripping | Build Settings → Deployment | 移除未引用代码 | ✅ 开启 |
| Deployment Postprocessing | Build Settings → Deployment | 启用后续处理 | ✅ 开启 |

**2. 减少代码体积的实践：**

```swift
// ❌ 避免：大量重复的字符串字面量
let title1 = "设置"
let title2 = "设置"
let title3 = "设置"

// ✅ 推荐：使用常量
enum StringConstants {
    static let settingsTitle = "设置"
}

// ❌ 避免：大段未使用的条件编译代码
#if DEBUG
// 500 行调试代码...
#endif

// ✅ 推荐：将调试代码独立文件，通过 Build Phase 排除
```

**3. 检查未使用的代码：**

```bash
# 使用 Periphery 扫描未使用的代码
# 安装：brew install peripheryapp/periphery/periphery
periphery scan
```

### 5.5 Swift ABI 稳定性影响

Swift 5.0 实现了 ABI 稳定性（ABI Stability），这对包大小有重要影响：

| Swift 版本 | ABI 状态 | 包含 Swift 运行时 | 对包大小影响 |
|:----------:|----------|:-----------------:|:------------:|
| Swift 4.x 及之前 | 不稳定 | ✅ 需要内嵌 | 增加约 5-10MB |
| Swift 5.0+ | 稳定 | ❌ 系统内置（iOS 12.2+） | 不增加包大小 |
| Swift 5.0+ 最低支持 iOS 12.2 以下 | 混合 | ✅ 需要内嵌 | 增加约 5-10MB |

> 💡 **提示**：如果你的 App 最低支持 iOS 12.2 及以上，Swift 运行时库由系统提供，不需要内嵌到 App 中，可以减少约 5-10MB。在 Xcode 中确认：Build Settings → `Always Embed Swift Standard Libraries` 设为 `No`（当 Deployment Target ≥ iOS 12.2 时）。

### 5.6 动态库 vs 静态库

第三方库的链接方式对包大小影响显著：

| 对比项 | 动态库（Dynamic Framework） | 静态库（Static Library） |
|--------|:-------------------------:|:-----------------------:|
| 链接方式 | 运行时动态加载 | 编译时静态链接 |
| 包大小影响 | 每个库独立，无法去重 | 链接器可移除未使用符号 |
| 启动速度 | 较慢（需 dyld 加载） | 较快 |
| 内存占用 | 较高 | 较低 |
| 多 App 共享 | 可共享 | 不可共享 |
| App Extension | 可共享动态库 | 需重复包含 |

**CocoaPods 配置静态链接：**

```ruby
# Podfile 中全局设置
# 所有 Pod 使用静态链接
use_frameworks!:linkage => :static

# 或对单个 Pod 设置
pod 'Alamofire', :linkage => :static
pod 'SnapKit', :linkage => :static
```

**优化效果估算：**

| 场景 | 动态库总大小 | 静态库总大小 | 节省 |
|------|:----------:|:----------:|:----:|
| 10 个常用 Pod | ~15MB | ~8MB | ~47% |
| 5 个常用 Pod | ~8MB | ~5MB | ~38% |

> ⚠️ **警告**：将动态库改为静态链接时，需要注意符号冲突问题。如果多个静态库包含相同的符号，会导致链接错误。建议逐个转换并测试。

---

## 6. 包大小优化实战 Checklist

### 6.1 从 100MB 优化到 50MB 的完整路径

以下是一个真实项目的优化路径，按优先级排序——优先做收益大、成本低的：

| 优先级 | 优化项 | 预估收益 | 实施难度 | 累计大小 |
|:------:|--------|:--------:|:--------:|:--------:|
| P0 | 图片压缩（pngquant + ImageOptim） | -20MB | ⭐ | 80MB |
| P0 | 所有图片移入 Asset Catalog | -8MB | ⭐ | 72MB |
| P0 | 移除 @1x 图片资源 | -3MB | ⭐ | 69MB |
| P1 | 移除未使用的图片和资源 | -5MB | ⭐⭐ | 64MB |
| P1 | 第三方库从动态链接改为静态链接 | -7MB | ⭐⭐ | 57MB |
| P1 | 移除未使用的第三方库 | -4MB | ⭐⭐ | 53MB |
| P2 | 编译器优化选项全部开启 | -2MB | ⭐ | 51MB |
| P2 | 大型静态数据改为服务端下发 | -3MB | ⭐⭐⭐ | 48MB |
| P2 | 使用 ODR 按需加载非核心资源 | -5MB | ⭐⭐⭐ | 43MB |
| P3 | PDF 矢量图替代多分辨率 PNG | -2MB | ⭐⭐ | 41MB |
| P3 | 代码层面精简（Periphery 扫描） | -1MB | ⭐⭐⭐ | 40MB |

### 6.2 优化优先级排序原则

**"二八法则"——80% 的包大小来自 20% 的文件。**

先找到最大的文件，优先优化它们：

```bash
# 按大小排序，找出 Top 20 大文件
find . -type f -not -path "*/Pods/*" -not -path "*/.git/*" | \
    xargs du -sk | sort -rn | head -20
```

**优先级判断矩阵：**

| | 实施难度低 | 实施难度高 |
|--------|:----------:|:----------:|
| **收益大** | ✅ **立即做**（P0） | 📅 **计划做**（P1） |
| **收益小** | 🔄 **顺手做**（P2） | ❌ **暂不做**（P3） |

### 6.3 优化前后的验证清单

| 检查项 | 验证方法 | 通过标准 |
|--------|----------|----------|
| 功能完整性 | 全量回归测试 | 所有功能正常 |
| 图片显示正常 | 视觉走查 | 无模糊、无缺失 |
| 启动速度 | Instruments Time Profiler | 不劣于优化前 |
| 内存占用 | Instruments Allocations | 不劣于优化前 |
| 下载大小 | App Thinning Size Report | < 150MB |
| 各设备兼容性 | 多设备测试 | 全部正常 |

---

## 7. 审核与包大小

### 7.1 App Store 大小限制表

| 限制类型 | 大小限制 | 说明 |
|----------|:--------:|------|
| 蜂窝下载限制 | **150MB** | 超过此大小用户必须用 Wi-Fi 下载 |
| 推荐上传大小 | **200MB** 以内 | 超过可能影响搜索排名和推荐 |
| 上传大小上限 | **1GB**（压缩后） | 超过无法上传到 App Store Connect |
| 安装大小上限 | **2GB**（未压缩） | 安装后占用空间不得超过此值 |
| ODR 单个标签 | **2GB** | 单个 ODR 标签的资源上限 |
| ODR 总大小 | **20GB** | 所有 ODR 资源的总上限 |

### 7.2 各档位详细说明

**150MB 档位——蜂窝下载生死线：**

| 项目 | 说明 |
|------|------|
| 含义 | 用户使用蜂窝网络（4G/5G）下载时的最大限制 |
| 超过后果 | 用户必须连接 Wi-Fi 才能下载 |
| 影响范围 | 所有 iOS 设备，无法绕过 |
| 优化目标 | **这是最重要的目标线** |

类比：150MB 就像快递的"免运费门槛"——超过这个门槛，用户就要"额外付出代价"（找 Wi-Fi），大量用户会因此放弃下载。

**200MB 档位——推荐大小线：**

| 项目 | 说明 |
|------|------|
| 含义 | Apple 建议的 App 下载大小上限 |
| 超过后果 | 可能影响 App Store 搜索排名和推荐位 |
| 影响范围 | 间接影响，非硬性限制 |

**1GB 档位——上传硬性上限：**

| 项目 | 说明 |
|------|------|
| 含义 | 上传到 App Store Connect 的压缩包大小上限 |
| 超过后果 | 无法上传，直接被拒绝 |
| 影响范围 | 硬性限制，无例外 |

### 7.3 大包提审注意事项

如果你的 App 下载大小超过 150MB，提审时需要注意：

**1. 审核备注说明**

在提交审核时，务必在审核备注中说明：

```
审核备注示例：

本 App 下载大小为 180MB，超过 150MB 蜂窝下载限制。
主要原因：
1. 包含高质量教学视频资源（约 80MB）
2. 多语言支持包含 5 种语言资源（约 30MB）

我们已采取以下优化措施：
1. 使用 App Thinning 按设备裁剪
2. 非核心资源使用 On-Demand Resources 按需加载
3. 图片资源全部使用 HEIC 格式压缩

测试账号：test@example.com
测试密码：Test1234
```

**2. 大包常见被拒原因：**

| 被拒原因 | 条款 | 应对方法 |
|----------|------|----------|
| 包含可按需加载的资源却打包在主包 | 2.1 | 使用 ODR 按需加载 |
| 包含未使用的架构代码 | 2.1 | 确保 Slicing 正常工作 |
| 资源未压缩 | 2.1 | 使用图片压缩工具 |
| 安装大小过大影响用户体验 | 2.1 | 提供优化计划 |

**3. 大包优化紧急方案：**

如果提交前发现包大小超标，可以快速采取以下紧急措施：

| 紧急措施 | 预估收益 | 实施时间 | 风险 |
|----------|:--------:|:--------:|:----:|
| 压缩所有 PNG | 20-40% | 1 小时 | 低 |
| 移除非必要语言资源 | 10-20% | 30 分钟 | 低 |
| 大文件改为服务端下载 | 不限 | 2-4 小时 | 中 |
| 启用 ODR | 30-50% | 4-8 小时 | 中 |
| 移除未使用的第三方库 | 5-15% | 2 小时 | 中 |

> ⚠️ **警告**：不要为了减小包大小而牺牲核心功能。如果 App 本身就需要大量资源（如导航离线地图、专业素材库），应在审核备注中充分说明原因，而不是删减核心功能。

---

## 本章小结

| 知识点 | 核心内容 | 关键要点 |
|--------|----------|----------|
| App Thinning 概述 | 三大技术：Slicing、Bitcode、ODR | Bitcode 已弃用，重点掌握 Slicing 和 ODR |
| App Slicing | 按设备特征自动裁剪 | 资源放入 Asset Catalog 即可自动享受 |
| On-Demand Resources | 按需下载非核心资源 | 用 NSBundleResourceRequest 管理，注意资源可能被清理 |
| 包大小分析工具 | Size Report、ASC 报告、命令行工具 | 优先关注 compressed 下载大小 |
| 图片优化 | 压缩、Asset Catalog、矢量图 | 图片是包大小最大贡献者，优化收益最高 |
| 代码优化 | 编译器选项、静态链接、ABI 稳定性 | 静态链接可减少约 40% 第三方库体积 |
| 实战 Checklist | 按优先级排序优化路径 | 先做收益大、成本低的优化 |
| 审核与包大小 | 150MB 蜂窝限制、200MB 推荐线、1GB 上限 | 超过 150MB 必须在审核备注中说明 |

> 💡 **提示**：包大小优化不是一次性工作，而是贯穿整个开发周期的持续任务。建议在 CI/CD 流程中加入包大小监控，每次构建自动记录大小变化，超过阈值时发出警告。
