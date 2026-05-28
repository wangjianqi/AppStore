# 附录 A：常见问题 FAQ

> 开发过程中遇到问题？先来这里找答案！本附录汇总了从环境搭建到上架全流程中最常见的问题和解决方案。

---

## 开发环境相关

### Xcode 相关问题

#### Q: Xcode 下载太慢怎么办？

Apple 的服务器在国内访问速度较慢，可以尝试以下方法：

| 方法 | 说明 | 推荐度 |
|------|------|--------|
| 使用迅雷下载 | 从 Apple 开发者网站下载 `.xip` 文件，用迅雷加速 | ⭐⭐⭐⭐⭐ |
| 使用代理 | 配置终端代理后通过 `xcode-select --install` 安装 | ⭐⭐⭐⭐ |
| 夜间下载 | 凌晨时段网络通常更快 | ⭐⭐⭐ |
| 使用 Xcode Cloud | 如果只是需要编译环境，可考虑云端方案 | ⭐⭐ |

> 💡 **小贴士**：直接下载地址：https://developer.apple.com/download/all/ ，搜索 "Xcode" 即可找到 `.xip` 安装包。

---

#### Q: Xcode 占用空间太大怎么办？

Xcode 安装后通常会占用 30GB+ 空间，以下是清理方法：

| 清理项目 | 路径 | 操作方式 | 可释放空间 |
|----------|------|----------|-----------|
| 旧版模拟器 | `~/Library/Developer/CoreSimulator` | Xcode → Window → Devices → 删除不需要的模拟器 | 5-20GB |
| 派生数据 | `~/Library/Developer/Xcode/DerivedData` | Xcode → Preferences → Locations → 点击箭头 → 删除 | 1-10GB |
| 旧版 iOS 支持包 | `/Library/Developer/CoreSimulator/Profiles/Runtimes` | 删除不需要的 iOS 版本运行时 | 2-5GB/个 |
| 文档缓存 | `~/Library/Developer/Xcode/DerivedData` | Xcode → Preferences → Downloads | 1-3GB |
| 归档文件 | `~/Library/Developer/Xcode/Archives` | 手动删除旧的归档 | 视情况而定 |

**一键清理命令**（谨慎使用）：

```bash
# 清理 DerivedData
rm -rf ~/Library/Developer/Xcode/DerivedData

# 清理旧模拟器运行时
xcrun simctl delete unavailable
```

---

#### Q: Xcode 经常卡顿怎么办？

1. **关闭不必要的功能**：
   - 关闭 Source Control（Git）：Preferences → Source Control → 取消勾选
   - 关闭 Live Issues：Preferences → Text Editing → 取消勾选 Live Issues

2. **减少索引范围**：
   - 在 Build Settings 中排除不需要索引的文件夹

3. **清理缓存**：
   ```bash
   rm -rf ~/Library/Developer/Xcode/DerivedData
   ```

4. **硬件建议**：

| 配置项 | 最低要求 | 推荐配置 |
|--------|---------|---------|
| 内存 | 8GB | 16GB+ |
| 硬盘 | 256GB SSD | 512GB+ SSD |
| CPU | M1 | M2 及以上 |

---

#### Q: 如何安装多个版本的 Xcode？

1. 从 [Apple 开发者下载页](https://developer.apple.com/download/all/) 下载不同版本的 `.xip` 文件
2. 解压后重命名（如 `Xcode_15.xip` → `Xcode-15.app`）
3. 移动到 `/Applications` 目录
4. 使用命令行切换默认版本：

```bash
# 查看当前版本
xcode-select -p

# 切换到指定版本
sudo xcode-select -s /Applications/Xcode-15.app/Contents/Developer
```

> ⚠️ **注意**：同一时间只能有一个 Xcode 作为默认命令行工具，但可以同时打开多个 Xcode 应用。

---

#### Q: Xcode 更新后项目打不开？

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| 提示需要更高版本 iOS | Xcode 更新后最低支持版本变了 | 降低 Deployment Target |
| 编译报错 | Swift 版本不兼容 | Build Settings → Swift Language Version 选择对应版本 |
| 界面布局错乱 | SwiftUI API 变更 | 查看更新日志，修改废弃的 API |
| 依赖包报错 | SPM 缓存问题 | File → Packages → Reset Package Caches |

---

#### Q: 模拟器启动很慢？

1. **选择轻量模拟器**：iPhone SE 模拟器比 iPhone 15 Pro Max 启动更快
2. **关闭不需要的模拟器**：只保留当前开发需要的
3. **增加 Mac 内存**：模拟器非常吃内存
4. **使用真机调试**：有条件的话直接用真机，体验更好

---

### 模拟器相关问题

#### Q: 模拟器键盘不弹出？

模拟器默认使用 Mac 键盘输入，需要手动开启软键盘：

```
I/O → Keyboard → 取消勾选 "Connect Hardware Keyboard"
```

或者使用快捷键：**⌘ + K** 切换软键盘显示。

---

#### Q: 模拟器没有声音？

1. 确认 Mac 本身没有静音
2. 模拟器菜单：`I/O → Audio Output` 确认选择了正确的输出设备
3. 重启模拟器试试

> ⚠️ **注意**：部分音频功能（如实时音频处理）在模拟器上可能不支持，建议用真机测试。

---

#### Q: 如何重置模拟器？

```
Device → Erase All Content and Settings...
```

或者通过命令行：

```bash
# 重置指定模拟器
xcrun simctl erase <设备ID>

# 重置所有模拟器
xcrun simctl erase all

# 查看所有模拟器列表
xcrun simctl list devices
```

---

#### Q: 模拟器占用空间太大？

```bash
# 查看模拟器占用空间
du -sh ~/Library/Developer/CoreSimulator

# 删除不可用的模拟器
xcrun simctl delete unavailable

# 删除所有模拟器数据（慎用！）
rm -rf ~/Library/Developer/CoreSimulator
```

---

### 证书与签名问题

#### Q: "No signing certificate found" 怎么办？

这是最常见的签名问题，按以下步骤排查：

| 步骤 | 操作 | 说明 |
|------|------|------|
| 1 | 检查 Apple ID 登录 | Xcode → Preferences → Accounts → 确认已登录 |
| 2 | 查看证书列表 | Accounts → 选中 Apple ID → Manage Certificates |
| 3 | 添加开发证书 | 点击 "+" → Apple Development |
| 4 | 检查 Team 选择 | 项目 → Signing & Capabilities → 选择正确的 Team |
| 5 | 勾选自动签名 | 勾选 "Automatically manage signing" |

如果以上步骤无效，尝试：

```bash
# 删除钥匙串中的旧证书
# 打开"钥匙串访问" → 搜索 "Apple Development" → 删除过期证书
# 然后重新在 Xcode 中添加证书
```

---

#### Q: 证书过期了怎么办？

1. 打开 **钥匙串访问**（Keychain Access）
2. 搜索 "Apple Development" 或 "Apple Distribution"
3. 找到过期证书，右键删除
4. 回到 Xcode → Preferences → Accounts → Manage Certificates
5. 点击 "+" 重新创建证书

> 💡 **小贴士**：证书有效期通常为 1 年，建议在到期前 30 天续期，避免影响开发。

---

#### Q: 如何撤销和重新创建证书？

1. 登录 [Apple 开发者证书页面](https://developer.apple.com/account/resources/certificates/list)
2. 找到需要撤销的证书 → 点击 "Revoke"
3. 回到 Xcode → Preferences → Accounts → Manage Certificates
4. 点击 "+" 创建新证书

> ⚠️ **注意**：撤销证书后，使用该证书签名的 App 将无法运行，请确保团队内其他人不受影响。

---

#### Q: 描述文件（Provisioning Profile）失效怎么办？

| 现象 | 原因 | 解决方案 |
|------|------|---------|
| 无法安装到真机 | 描述文件过期 | 删除旧描述文件，重新生成 |
| 编译报错 | 设备 UDID 未添加 | 在开发者中心添加设备 UDID |
| 证书不匹配 | 证书被重新创建 | 删除旧描述文件，Xcode 会自动重新生成 |

**手动删除描述文件**：

```bash
# 查看已安装的描述文件
ls ~/Library/MobileDevice/Provisioning\ Profiles/

# 删除所有描述文件（Xcode 会重新生成）
rm ~/Library/MobileDevice/Provisioning\ Profiles/*
```

---

## 开发相关

### SwiftUI 常见问题

#### Q: Preview 不显示/崩溃？

这是 SwiftUI 开发中最常见的问题之一：

| 原因 | 解决方案 |
|------|---------|
| Preview 代码有语法错误 | 检查 Preview 宏下方的代码 |
| 使用了不支持 Preview 的类型 | 避免在 Preview 中使用 Core Data 等需要上下文的类型 |
| 缓存问题 | Editor → Canvas → 点击 "Try Again" 或重启 Preview |
| 内存不足 | 关闭其他应用释放内存 |
| Xcode Bug | 清理 DerivedData 后重启 Xcode |

**强制重启 Preview**：

```
Editor → Canvas → 点击刷新按钮（或 ⌥⌘P）
```

---

#### Q: 视图不刷新？

SwiftUI 视图不刷新通常是因为状态管理出了问题：

| 场景 | 原因 | 解决方案 |
|------|------|---------|
| 修改了数组元素 | 直接修改数组元素不会触发刷新 | 使用 `$` 绑定或替换整个数组 |
| 修改了结构体属性 | 结构体是值类型 | 确保属性标记了正确的属性包装器 |
| 异步更新 | 在非主线程更新 UI | 使用 `@MainActor` 或 `DispatchQueue.main.async` |
| Observable 对象不更新 | 没有使用 `@Observable` 宏 | 使用 `@Observable` 宏或手动调用 `objectWillChange` |

**常见错误示例**：

```swift
// ❌ 错误：直接修改不会触发刷新
var items = [1, 2, 3]
items[0] = 99  // 视图不会刷新

// ✅ 正确：替换整个数组触发刷新
@State var items = [1, 2, 3]
items[0] = 99  // @State 会检测到变化
```

---

#### Q: @State 和 @Binding 的区别？

| 特性 | @State | @Binding |
|------|--------|----------|
| 用途 | 在当前视图中持有和管理工作状态 | 在子视图中引用和修改父视图的状态 |
| 数据所有权 | 拥有数据 | 不拥有数据，只是引用 |
| 使用场景 | 简单的本地状态 | 需要传递给子视图修改的状态 |
| 声明方式 | `@State var count = 0` | `@Binding var count: Int` |
| 传递方式 | 直接使用 | 通过 `$` 传递绑定 |

**示例**：

```swift
struct ParentView: View {
    @State var isOn = false

    var body: some View {
        ChildView(isOn: $isOn)  // 用 $ 传递绑定
    }
}

struct ChildView: View {
    @Binding var isOn: Bool    // 子视图用 @Binding 接收

    var body: some View {
        Toggle("开关", isOn: $isOn)
    }
}
```

---

#### Q: List 性能差怎么办？

| 优化方法 | 说明 | 示例 |
|----------|------|------|
| 使用 `LazyVStack` | 只渲染可见区域的行 | `LazyVStack { ForEach(items) { ... } }` |
| 避免复杂行视图 | 简化每行的 UI 结构 | 减少嵌套层级 |
| 使用 `id` 参数 | 帮助 SwiftUI 识别元素 | `ForEach(items, id: \.id) { ... }` |
| 避免在行视图中计算 | 预先计算好数据 | 在 ViewModel 中处理数据 |
| 分页加载 | 不要一次加载所有数据 | 实现上拉加载更多 |

---

#### Q: NavigationLink 跳转无效？

常见原因及解决方案：

| 原因 | 解决方案 |
|------|---------|
| NavigationLink 不在 NavigationStack 内 | 确保外层有 `NavigationStack` 包裹 |
| 使用了 `value` 参数但没加 `navigationDestination` | 添加 `.navigationDestination(for:)` 修饰符 |
| 在 Sheet 中使用 NavigationLink | Sheet 内也需要 `NavigationStack` |
| iOS 版本不兼容 | 检查 API 的最低支持版本 |

**正确用法**：

```swift
NavigationStack {
    List {
        NavigationLink("跳转") {
            DetailView()  // iOS 16+ 简化写法
        }
    }
}
```

---

### 网络请求常见问题

#### Q: 请求超时怎么办？

```swift
let config = URLSessionConfiguration.default
config.timeoutIntervalForRequest = 30    // 请求超时 30 秒
config.timeoutIntervalForResource = 60   // 资源超时 60 秒
let session = URLSession(configuration: config)
```

| 排查步骤 | 操作 |
|----------|------|
| 1 | 检查网络连接是否正常 |
| 2 | 确认服务器地址是否正确 |
| 3 | 尝试增加超时时间 |
| 4 | 添加重试机制 |
| 5 | 使用抓包工具查看请求是否发出 |

---

#### Q: App Transport Security (ATS) 错误？

当请求 HTTP（非 HTTPS）接口时会报 ATS 错误。

**临时解决方案**（仅用于开发环境）：

在 `Info.plist` 中添加：

```xml
<key>NSAppTransportSecurity</key>
<dict>
    <key>NSAllowsArbitraryLoads</key>
    <true/>
</dict>
```

**生产环境推荐做法**：

| 方案 | 说明 |
|------|------|
| 使用 HTTPS | 让服务器配置 SSL 证书（推荐） |
| 配置例外域名 | 只对特定域名允许 HTTP |
| 使用 ATS 例外 | 在审核时说明原因 |

> ⚠️ **注意**：`NSAllowsArbitraryLoads` 在 App Store 审核时可能被拒，生产环境务必使用 HTTPS。

---

#### Q: 如何调试网络请求？

| 工具 | 用途 | 使用方式 |
|------|------|---------|
| Charles | 抓包分析 | 配置代理，查看请求和响应 |
| Proxyman | macOS 抓包 | 比 Charles 更轻量 |
| Xcode Network Debug | 查看网络请求 | Debug Navigator → Network |
| URLSession 日志 | 代码层面打印 | 自定义 `URLSessionDelegate` |
| Postman | 测试 API | 先在 Postman 中验证接口 |

**代码调试技巧**：

```swift
func printResponse(_ response: URLResponse?, data: Data?) {
    if let httpResponse = response as? HTTPURLResponse {
        print("状态码: \(httpResponse.statusCode)")
        print("头部: \(httpResponse.allHeaderFields)")
    }
    if let data = data,
       let json = try? JSONSerialization.jsonObject(with: data) {
        print("响应体: \(json)")
    }
}
```

---

#### Q: JSON 解析失败？

| 常见错误 | 原因 | 解决方案 |
|----------|------|---------|
| KeyNotFound | JSON 字段名和模型属性名不一致 | 使用 `CodingKeys` 映射 |
| TypeMismatch | 类型不匹配（如期望 Int 收到 String） | 使用可选类型或自定义解码 |
| ValueNotFound | 字段不存在且不是可选类型 | 将属性声明为可选 `String?` |
| DataCorrupted | 数据格式完全错误 | 先打印原始 JSON 检查格式 |

**处理技巧**：

```swift
struct User: Codable {
    var name: String
    var age: Int?
    var avatar: String?

    // 字段名映射
    enum CodingKeys: String, CodingKey {
        case name = "user_name"   // JSON 中是 user_name
        case age = "user_age"
        case avatar
    }
}
```

---

### 数据持久化常见问题

#### Q: UserDefaults 数据丢失？

| 原因 | 解决方案 |
|------|---------|
| App 被系统杀死后未及时同步 | 调用 `UserDefaults.standard.synchronize()` |
| 存储的数据量太大 | UserDefaults 只适合存小量数据，大数据用文件或数据库 |
| Key 拼写错误 | 定义常量避免硬编码 |
| 重新安装 App | 重新安装会清除数据，这是正常行为 |
| 沙盒路径变化 | 不要存储绝对路径，存储文件名即可 |

**推荐用法**：

```swift
// 使用 @AppStorage 更方便
@AppStorage("hasSeenOnboarding") var hasSeenOnboarding = false
@AppStorage("userName") var userName: String = ""
```

---

#### Q: SwiftData 和 Core Data 怎么选？

| 对比项 | SwiftData | Core Data |
|--------|-----------|-----------|
| 最低系统版本 | iOS 17+ | iOS 10+ |
| 学习难度 | ⭐⭐ | ⭐⭐⭐⭐ |
| 代码量 | 少（宏驱动） | 多（需配置 xcdatamodeld） |
| 稳定性 | 较新，可能有 Bug | 非常成熟稳定 |
| 功能完整度 | 基础功能完善 | 功能非常丰富 |
| 推荐场景 | 新项目、iOS 17+ | 需要兼容旧系统、复杂查询 |

**选择建议**：

- 🆕 **新项目 + 只支持 iOS 17+** → 选 SwiftData
- 📱 **需要兼容 iOS 16 及以下** → 选 Core Data
- 🔧 **需要复杂查询和迁移** → 选 Core Data
- 🎯 **简单数据存储** → 选 SwiftData 或 UserDefaults

---

#### Q: 数据迁移怎么做？

**SwiftData 迁移**：

```swift
// 定义迁移计划
let schema = Schema([Item.self])
let migrationPlan = SchemaMigrationPlan([
    MigrateV1toV2.self
])

// 版本 1 的模型
@Model
class ItemV1 {
    var name: String
}

// 版本 2 的模型（新增字段）
@Model
class ItemV2 {
    var name: String
    var createdAt: Date = .now  // 新增字段
}
```

**Core Data 迁移**：

1. 在 xcdatamodeld 中创建新版本模型
2. Editor → Add Model Version
3. 设置新版本为 Current Model
4. 配置轻量级迁移选项：

```swift
let options = [
    NSMigratePersistentStoresAutomaticallyOption: true,
    NSInferMappingModelAutomaticallyOption: true
]
```

> 💡 **小贴士**：每次修改数据模型前，先创建新版本，避免数据丢失。

---

## 上架相关

### App Store Connect 问题

#### Q: 构建版本处理失败？

| 失败原因 | 解决方案 |
|----------|---------|
| 缺少隐私描述 | 在 Info.plist 中添加对应的 Usage Description |
| 使用了非公开 API | 检查代码，移除非公开 API 调用 |
| 架构不匹配 | 确保使用 Release 配置打包 |
| 证书/描述文件问题 | 重新生成证书和描述文件 |
| 版本号冲突 | 确保版本号和构建号递增 |

**排查步骤**：

1. 登录 App Store Connect → 活动 → 查看构建版本的详细错误信息
2. 检查邮件，Apple 会发送详细的错误说明
3. 常见缺失的隐私描述：

| 权限 | Info.plist Key |
|------|---------------|
| 相机 | `NSCameraUsageDescription` |
| 相册 | `NSPhotoLibraryUsageDescription` |
| 定位 | `NSLocationWhenInUseUsageDescription` |
| 麦克风 | `NSMicrophoneUsageDescription` |
| 通知 | 在 Capabilities 中开启 Push Notifications |

---

#### Q: 截图尺寸不对？

App Store 对截图尺寸有严格要求，以下是常用设备的截图规格：

| 设备 | 分辨率 | 截图尺寸（像素） |
|------|--------|-----------------|
| iPhone 6.9" (15 Pro Max) | 1290×2796 | 1290×2796 |
| iPhone 6.7" (14 Plus) | 1284×2778 | 1284×2778 |
| iPhone 6.5" (11 Pro Max) | 1242×2688 | 1242×2688 |
| iPhone 5.5" (8 Plus) | 1242×2208 | 1242×2208 |
| iPad 12.9" (6th gen) | 2048×2732 | 2048×2732 |
| iPad 11" (4th gen) | 1668×2388 | 1668×2388 |

> 💡 **小贴士**：使用模拟器截图时，选择 `File → Save Screen` 保存的图片尺寸就是正确的。不要使用截图工具，因为会包含状态栏或分辨率不对。

---

#### Q: 如何修改已发布 App 的信息？

| 可修改内容 | 修改方式 | 是否需要审核 |
|-----------|---------|-------------|
| App 名称 | App Store Connect → App 信息 | 需要 |
| 描述 | App Store Connect → App 信息 | 需要 |
| 截图 | App Store Connect → App 信息 | 需要 |
| 价格 | App Store Connect → 定价和销售范围 | 不需要 |
| 分类 | App Store Connect → App 信息 | 需要 |
| 隐私政策 URL | App Store Connect → App 信息 | 不需要 |
| 支持网址 | App Store Connect → App 信息 | 不需要 |

> ⚠️ **注意**：修改名称、描述、截图等信息后，需要提交新版本审核才会生效。

---

### 审核相关问题

#### Q: 审核一般需要多久？

| 审核类型 | 平均时长 | 说明 |
|----------|---------|------|
| 首次提交 | 2-7 天 | 新 App 首次审核较慢 |
| 版本更新 | 1-3 天 | 更新审核通常更快 |
| 加急审核 | 1 天内 | 需要申请，有次数限制 |
| 节假日期间 | 可能延长 | 圣诞/春节等假期可能更慢 |

---

#### Q: 被拒后多久可以重新提交？

- 修改后可以**立即重新提交**
- 如果对拒审有异议，可以通过 **Resolution Center** 与审核团队沟通
- 申诉通常在 **1-2 个工作日**内回复

**被拒后的正确处理流程**：

1. 仔细阅读拒审原因
2. 在 Resolution Center 中查看详细说明
3. 修改代码或调整 App 内容
4. 重新提交审核

---

#### Q: 如何申请加急审核？

1. 登录 [App Store Connect](https://appstoreconnect.apple.com)
2. 进入对应的 App → 活动
3. 点击"加急审核"按钮
4. 填写加急原因

| 注意事项 | 说明 |
|----------|------|
| 申请次数 | 每年限额，不要滥用 |
| 合理理由 | 安全修复、时效性事件等 |
| 审批结果 | 不保证一定通过 |
| 处理时间 | 通过后通常 1 天内完成审核 |

---

#### Q: 多次被拒会有影响吗？

| 担忧 | 实际情况 |
|------|---------|
| 会被封号吗？ | 不会，除非存在欺诈或恶意行为 |
| 审核会更严格吗？ | 不会，审核标准一致 |
| 会影响排名吗？ | 不会，审核过程和排名无关 |
| 需要换账号吗？ | 不需要，正常修改重新提交即可 |

> 💡 **小贴士**：每次被拒都是改进 App 的机会，认真对待审核反馈，逐步完善 App 质量。

---

### ICP 备案问题

#### Q: 个人可以备案吗？

| 主体类型 | 是否可以备案 | 说明 |
|----------|-------------|------|
| 个人 | ✅ 可以 | 需要身份证和手机号 |
| 个体工商户 | ✅ 可以 | 需要营业执照 |
| 企业 | ✅ 可以 | 需要营业执照和公章 |
| 外籍人士 | ❌ 不可以 | 需要中国身份证 |

---

#### Q: 备案需要多久？

| 阶段 | 时长 | 说明 |
|------|------|------|
| 准备材料 | 1-3 天 | 填写信息、准备证件 |
| 提交审核 | 1-2 天 | 平台初审 |
| 管局审核 | 5-20 个工作日 | 各地管局速度不同 |
| 备案完成 | 即时 | 收到备案号 |

---

#### Q: 备案被拒怎么办？

| 常见拒因 | 解决方案 |
|----------|---------|
| 网站名称不规范 | 使用与备案主体相关的名称 |
| 证件信息不匹配 | 核对身份证号、姓名等 |
| 网站内容不符 | 确保 App 内容与备案描述一致 |
| 联系方式不通 | 确保手机号可接听验证电话 |

> 💡 **小贴士**：被拒后根据退回原因修改，可以重新提交，不限制次数。

---

#### Q: 备案号怎么展示？

根据规定，App 必须在显著位置展示 ICP 备案号：

| 展示位置 | 推荐方式 |
|----------|---------|
| 关于页面 | 在"关于"页面底部展示 |
| 设置页面 | 在设置页面底部展示 |
| 启动页 | 可选，不强制 |

**展示要求**：

- 备案号必须可点击，链接到工信部查询页面
- 链接地址：`https://beian.miit.gov.cn/`
- 文字格式示例：`京ICP备XXXXXXXX号`

---

## AI 工具相关

### Claude Code 常见问题

#### Q: 安装失败怎么办？

| 错误信息 | 原因 | 解决方案 |
|----------|------|---------|
| `npm: command not found` | 未安装 Node.js | 先安装 Node.js 18+ |
| `permission denied` | 权限不足 | 使用 `sudo npm install -g @anthropic-ai/claude-code` |
| `network timeout` | 网络问题 | 配置 npm 代理或使用镜像源 |
| `EACCES error` | npm 全局目录权限问题 | 修改 npm 全局安装路径 |

**安装步骤**：

```bash
# 1. 确认 Node.js 版本
node -v   # 需要 18+

# 2. 安装 Claude Code
npm install -g @anthropic-ai/claude-code

# 3. 验证安装
claude --version

# 4. 登录
claude login
```

---

#### Q: API Key 无效？

| 排查步骤 | 操作 |
|----------|------|
| 1 | 确认 Key 是否正确复制（没有多余空格） |
| 2 | 检查 Key 是否已过期或被撤销 |
| 3 | 确认账户余额是否充足 |
| 4 | 检查是否选择了正确的 API 区域 |
| 5 | 重新生成 Key 试试 |

---

#### Q: 上下文太长怎么办？

| 方法 | 说明 |
|------|------|
| 分模块开发 | 将大任务拆分为小任务 |
| 使用 CLAUDE.md | 在项目根目录创建 CLAUDE.md，让 AI 了解项目结构 |
| 清除对话 | 使用 `/clear` 命令清除历史上下文 |
| 精简提问 | 只提供必要的代码片段，不要粘贴整个文件 |
| 使用 compact | 使用 `/compact` 命令压缩上下文 |

---

#### Q: 生成的代码有 Bug？

| 应对策略 | 说明 |
|----------|------|
| 逐段验证 | 不要一次性生成大量代码，逐段验证 |
| 要求加注释 | 让 AI 在关键逻辑处添加注释 |
| 编写测试 | 让 AI 同时生成单元测试 |
| 代码审查 | 让 AI 自我审查生成的代码 |
| 人工检查 | 始终人工审查 AI 生成的代码 |

---

### Cursor 常见问题

#### Q: Composer 模式不工作？

| 问题 | 解决方案 |
|------|---------|
| Composer 面板不显示 | 按 `⌘I` 打开 Composer |
| AI 不响应 | 检查网络连接和 API 配额 |
| 生成结果不应用 | 确认点击了 "Accept" 按钮 |
| 修改范围不对 | 在 Composer 中明确指定要修改的文件 |

---

#### Q: 如何配置 .cursorrules？

在项目根目录创建 `.cursorrules` 文件：

```
# 项目规则示例

## 技术栈
- SwiftUI + iOS 17+
- 使用 SwiftData 进行数据持久化
- 网络请求使用 URLSession

## 代码规范
- 使用 Swift 6 严格并发模式
- 所有视图必须是 struct
- ViewModel 使用 @Observable 宏
- 函数注释使用 /// 格式

## 命名规范
- 视图文件以 View 结尾
- ViewModel 文件以 ViewModel 结尾
- 常量使用 camelCase

## 禁止事项
- 不要使用 UIKit 组件
- 不要使用第三方网络库
- 不要使用 print() 调试，使用 os.Logger
```

---

#### Q: Cursor 和 VS Code 扩展冲突？

| 问题 | 解决方案 |
|------|---------|
| 快捷键冲突 | 在 Cursor 设置中重新绑定快捷键 |
| 扩展不兼容 | 禁用不兼容的扩展 |
| 设置被覆盖 | 使用 Cursor 专属设置文件 |
| 性能问题 | 禁用不必要的扩展 |

> 💡 **小贴士**：Cursor 基于 VS Code，大部分 VS Code 扩展都可以在 Cursor 中使用。如果遇到问题，尝试先禁用所有扩展，再逐个启用排查。

---

### AI 生成代码问题

#### Q: AI 使用了过时的 API？

| 常见过时 API | 替代方案 |
|-------------|---------|
| `NavigationView` | `NavigationStack` |
| `ListRowBackground` | `listRowBackground` 修饰符 |
| `@ObservedObject` + `ObservableObject` | `@Observable` 宏 |
| `UIImage` 在 SwiftUI 中 | `Image` 组件 |
| `UIColor` 在 SwiftUI 中 | `Color` 类型 |
| `UIApplication.shared` | 使用 SwiftUI 环境 |

**防范方法**：

- 在 Prompt 中明确指定 iOS 版本（如"使用 iOS 17+ API"）
- 在 `.cursorrules` 或 `CLAUDE.md` 中注明技术栈版本
- 查阅 Apple 官方文档验证 API 是否最新

---

#### Q: AI 生成的代码有安全漏洞？

| 常见漏洞 | 防范措施 |
|----------|---------|
| 硬编码密钥 | 使用 Keychain 或环境变量存储敏感信息 |
| SQL 注入 | 使用参数化查询 |
| 不安全的网络请求 | 强制使用 HTTPS |
| 数据未加密 | 敏感数据使用加密存储 |
| 权限过度申请 | 只申请必要的权限 |

---

#### Q: AI 幻觉怎么识别？

AI 幻觉是指 AI 编造了不存在的 API、方法或功能。识别方法：

| 幻觉类型 | 识别方法 | 示例 |
|----------|---------|------|
| 虚构 API | 在 Apple 文档中搜索验证 | `SwiftUI.ScrollTransition`（不存在的 API） |
| 错误参数 | 查看编译器报错 | `Text("hello").color(.red)`（应为 `.foregroundColor`） |
| 不存在的类型 | Xcode 自动补全验证 | `SwiftUI.CardView`（不存在的组件） |
| 编造的框架 | Google 搜索验证 | `SwiftUIX`（第三方库，不是官方的） |

**最佳实践**：

1. ✅ 始终在 Xcode 中编译验证
2. ✅ 不熟悉的 API 先查官方文档
3. ✅ 让 AI 解释代码逻辑
4. ❌ 不要盲目复制粘贴

---

#### Q: 如何验证 AI 代码的正确性？

| 验证方法 | 说明 | 推荐度 |
|----------|------|--------|
| Xcode 编译 | 最基本的验证 | ⭐⭐⭐⭐⭐ |
| 运行测试 | 编写单元测试验证逻辑 | ⭐⭐⭐⭐⭐ |
| 真机测试 | 在真机上运行验证 | ⭐⭐⭐⭐ |
| 代码审查 | 人工逐行审查 | ⭐⭐⭐⭐ |
| 查阅文档 | 对照官方文档验证 API | ⭐⭐⭐⭐ |
| 让 AI 自查 | 让另一个 AI 审查代码 | ⭐⭐⭐ |

---

### iOS App 集成 AI 功能

#### Q: API Key 应该存在哪里？

绝不要在客户端代码中硬编码 API Key。推荐通过自有后端代理转发 API 请求，这样 Key 只存在于服务端。开发阶段可以使用 Keychain 存储，但上线前务必切换到后端代理方案。

**方案对比**：

| 方案 | 安全性 | 实现难度 | 适用阶段 |
|------|--------|---------|---------|
| 硬编码 | ❌ 极不安全 | ⭐ | 禁止使用 |
| Keychain 存储 | ⚠️ 可被逆向 | ⭐⭐ | 开发阶段 |
| 后端代理 | ✅ 最安全 | ⭐⭐⭐ | 生产环境 |

---

#### Q: 如何实现流式输出？

使用 `URLSession.bytes(byLine:)` 逐行读取 SSE（Server-Sent Events）数据流。每个 `"data: "` 前缀的行包含一个 JSON delta 片段，解析后逐步更新 UI。`"data: [DONE]"` 表示流结束。

**核心代码**：

```swift
let (bytes, _) = try await URLSession.shared.bytes(from: url)
for try await line in bytes.lines {
    guard line.hasPrefix("data: "), line != "data: [DONE]" else { return }
    let json = line.dropFirst(6)
    if let delta = parseDelta(String(json)) {
        await MainActor.run { message.content += delta }
    }
}
```

---

#### Q: Token 成本怎么控制？

三个策略：

1. **设置 max_tokens 上限**：限制单次回复的最大 token 数
2. **截断历史上下文**：保留最近 N 条消息，避免上下文无限增长
3. **模型分级**：简单请求用便宜模型（如 DeepSeek），复杂请求用贵模型（如 GLM-4）

**成本估算参考**：

| 场景 | 模型选择 | 单次成本估算 |
|------|---------|------------|
| 简单问答 | DeepSeek Chat | ≈ ¥0.001 |
| 文本摘要 | 通义千问 qwen-plus | ≈ ¥0.002 |
| 复杂推理 | GLM-4 | ≈ ¥0.01 |

---

#### Q: AI 功能离线时怎么办？

实现降级策略：

1. **检测网络状态**：使用 `NWPathMonitor` 监听网络变化
2. **离线时显示友好提示**：告知用户当前无法使用 AI 功能
3. **可选本地模型**：使用 Core ML 部署轻量本地模型处理简单任务
4. **缓存常见回答**：对高频问题预缓存答案

```swift
import Network

let monitor = NWPathMonitor()
monitor.pathUpdateHandler = { path in
    if path.status == .satisfied {
        // 网络可用，启用 AI 功能
    } else {
        // 网络不可用，显示降级提示
    }
}
monitor.start(queue: DispatchQueue.global())
```

---

### 国内 AI 合规

#### Q: App 内集成 AI 功能需要算法备案吗？

取决于具体场景：

| 场景 | 是否需要备案 | 说明 |
|------|-------------|------|
| App 向公众提供 AI 对话/生成服务 | ✅ 通常需要 | 如内置 AI 助手、AI 写作等 |
| 仅调用第三方 API 展示结果 | ⚠️ 视情况而定 | 模型提供方已备案，App 方需评估 |
| 本地模型推理 | ❌ 通常不需要 | 不涉及云端服务 |

> 💡 **小贴士**：算法备案通过[互联网信息服务算法备案系统](https://beian.cac.gov.cn/)提交，审核周期约 20 个工作日。

---

#### Q: AI 生成的内容需要标识吗？

根据《生成式人工智能服务管理暂行办法》，AI 生成内容应进行标识。建议：

| 内容类型 | 标识方式 | 示例 |
|---------|---------|------|
| AI 生成文本 | 在内容旁显示"AI 生成"标签 | `⚠️ 此内容由 AI 生成，仅供参考` |
| AI 生成图片 | 嵌入数字水印 + 角标标识 | 右下角添加"AI"角标 |
| AI 生成代码 | 代码注释中标明 | `// AI-generated code` |

> ⚠️ **注意**：未标识 AI 生成内容可能面临监管处罚，务必在上线前完成标识方案。

---

#### Q: 调用海外 API 有什么合规风险？

调用 OpenAI/Claude 等海外 API 会导致用户数据出境，需遵守《数据出境安全评估办法》。建议面向国内用户的 App 优先使用国内大模型 API（如通义千问、DeepSeek），避免数据出境合规风险。

**风险对比**：

| API 来源 | 数据出境 | 合规风险 | 推荐度 |
|---------|---------|---------|--------|
| 国内 API（通义千问、DeepSeek 等） | ❌ 不出境 | 低 | ⭐⭐⭐⭐⭐ |
| 海外 API（OpenAI、Claude 等） | ✅ 出境 | 高 | ⭐⭐ |
| 自有部署模型 | ❌ 不出境 | 低 | ⭐⭐⭐⭐ |

**如必须使用海外 API**：

1. 完成数据出境安全评估
2. 在隐私政策中明确告知用户数据将出境
3. 获取用户明确同意
4. 确保数据传输加密

---

## 其他

#### Q: Mac 存储空间不够？

| 清理项目 | 可释放空间 | 方法 |
|----------|-----------|------|
| Xcode 缓存 | 5-30GB | 参见上方 Xcode 占用空间问题 |
| 模拟器数据 | 5-20GB | 删除不需要的模拟器 |
| Homebrew 缓存 | 1-5GB | `brew cleanup` |
| Docker 镜像 | 5-50GB | `docker system prune` |
| 系统缓存 | 1-10GB | 使用 OmniDiskSweeper 查找大文件 |
| 旧项目 | 视情况 | 归档或删除不再维护的项目 |

**推荐工具**：

| 工具 | 用途 | 价格 |
|------|------|------|
| OmniDiskSweeper | 查找大文件 | 免费 |
| CleanMyMac | 系统清理 | 付费 |
| DaisyDisk | 磁盘空间可视化 | 付费 |

---

#### Q: 如何学习更多 SwiftUI？

| 学习资源 | 类型 | 适合阶段 | 链接 |
|----------|------|---------|------|
| Apple 官方教程 | 交互式教程 | 入门 | https://developer.apple.com/tutorials/swiftui |
| Apple 官方文档 | 参考文档 | 全阶段 | https://developer.apple.com/documentation/swiftui |
| Hacking with Swift | 教程+实战 | 入门到进阶 | https://www.hackingwithswift.com |
| Swiftful Thinking | YouTube 视频 | 入门到进阶 | YouTube 搜索 "Swiftful Thinking" |
| Stanford CS193p | 大学课程 | 入门到进阶 | YouTube 搜索 "Stanford CS193p" |
| SwiftUI Lab | 实验性教程 | 进阶 | https://swiftui-lab.com |

---

#### Q: 有哪些推荐的社区？

| 社区 | 特点 | 链接 |
|------|------|------|
| Apple 开发者论坛 | 官方支持，Apple 工程师会回复 | https://developer.apple.com/forums |
| Stack Overflow | 全球最大的编程问答社区 | https://stackoverflow.com |
| Reddit r/iOSProgramming | 英文 iOS 开发社区 | https://reddit.com/r/iOSProgramming |
| 掘金 | 中文技术社区 | https://juejin.cn |
| Swift 论坛 | Swift 语言官方论坛 | https://forums.swift.org |
| V2EX | 中文技术社区 | https://v2ex.com |

---

#### Q: 如何找到远程 iOS 开发工作？

| 平台 | 特点 | 链接 |
|------|------|------|
| 远程.work | 中文远程工作平台 | https://yuancheng.work |
| Remote OK | 全球远程工作 | https://remoteok.com |
| We Work Remotely | 全球远程工作 | https://weworkremotely.com |
| LinkedIn | 职业社交+求职 | https://linkedin.com |
| 电鸭社区 | 中文远程工作社区 | https://eleduck.com |

**提高竞争力的建议**：

1. 📱 在 App Store 上架个人项目，展示实际作品
2. 📝 在 GitHub 开源项目，展示代码质量
3. ✍️ 写技术博客，展示专业能力
4. 🌐 参与开源社区，积累人脉
5. 🎯 专注某个领域（如 SwiftUI、ARKit），成为专家
