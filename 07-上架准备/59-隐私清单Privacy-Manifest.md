# 59-隐私清单（Privacy Manifest）

## 本章目标

- 理解什么是 Privacy Manifest，以及 Apple 为什么要强制要求它
- 掌握 PrivacyInfo.xcprivacy 文件的结构和每个字段的含义
- 学会声明 App 收集的数据类型和使用的 API 原因
- 能够在 Xcode 中创建和配置隐私清单文件
- 了解第三方 SDK 的隐私清单处理方式
- 避免因隐私清单问题导致的审核被拒

---

## 1. Privacy Manifest 是什么

### 1.1 一句话解释

Privacy Manifest（隐私清单）就是你的 App 交给 Apple 的一份"隐私声明书"——告诉 Apple 和用户：**我的 App 收集了哪些数据、用了哪些可能涉及隐私的 API、为什么用**。

### 1.2 生活类比

想象你开了一家餐厅（App），卫生局（Apple）要求你贴一张公示牌（隐私清单），上面写清楚：

- 🥩 你用了哪些食材（收集了哪些数据）
- 🔪 你用了哪些特殊厨具（调用了哪些隐私 API）
- 📋 你为什么需要这些（使用原因）

顾客（用户）看了公示牌，就能决定要不要进你的餐厅。

### 1.3 Apple 的强制要求

> ⚠️ **重要时间线**
>
> - **2023 年 WWDC**：Apple 宣布隐私清单要求
> - **2024 年 5 月 1 日起**：新 App 和更新必须包含隐私清单
> - 如果你的 App 使用了指定的隐私 API 但没有声明，**审核会被拒**！

Apple 这样做的原因很简单：保护用户隐私。以前很多 App 暗中收集用户信息（比如设备指纹、键盘信息等），用户根本不知道。现在 Apple 要求开发者"摊牌"，让一切透明化。

---

## 2. 哪些 App 必须提供隐私清单

### 2.1 需要提供的情况

| 情况 | 是否需要隐私清单 | 说明 |
|------|:---:|------|
| App 使用了指定的隐私相关 API | ✅ 必须 | 如 UserDefaults、文件时间戳等 |
| App 包含第三方 SDK | ✅ 必须 | SDK 也需要自己的隐私清单 |
| App 收集用户数据 | ✅ 必须 | 需要声明收集了哪些数据 |
| App 追踪用户行为 | ✅ 必须 | 需要声明追踪域名 |
| 纯本地、不使用任何隐私 API 的极简 App | ❌ 不强制 | 但建议还是添加 |

### 2.2 哪些 API 需要声明

Apple 划定了一批"需要声明原因"的 API，这些 API 本身可能被用来做用户追踪或指纹识别。主要包括：

| API 类别 | 典型用途 | 风险 |
|----------|---------|------|
| File timestamp API | 获取文件创建/修改时间 | 可用于设备指纹 |
| System boot time API | 获取系统启动时间 | 可用于设备指纹 |
| Disk space API | 获取磁盘空间信息 | 可用于设备指纹 |
| Active keyboards API | 获取当前激活的键盘列表 | 可推断用户语言/习惯 |
| User defaults API | 读写 UserDefaults | 可用于跨 App 追踪 |
| System boot time API | 获取系统启动时间 | 可用于设备指纹 |

> 💡 **简单判断原则**：如果你的代码里用了 `UserDefaults`、`FileManager`、`ProcessInfo`、`StatFS` 等，大概率需要声明。

---

## 3. PrivacyInfo.xcprivacy 文件结构

### 3.1 文件本质

`PrivacyInfo.xcprivacy` 本质上就是一个 **plist 文件**（XML 格式），里面包含四个顶级键：

```
PrivacyInfo.xcprivacy
├── NSPrivacyTracking              → 是否追踪用户（布尔值）
├── NSPrivacyTrackingDomains       → 追踪域名列表（数组）
├── NSPrivacyCollectedDataTypes    → 收集的数据类型（数组）
└── NSPrivacyAccessedAPITypes      → 使用的隐私 API（数组）
```

### 3.2 完整文件示例

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
 "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>NSPrivacyTracking</key>
    <false/>

    <key>NSPrivacyTrackingDomains</key>
    <array>
    </array>

    <key>NSPrivacyCollectedDataTypes</key>
    <array>
        <dict>
            <key>NSPrivacyCollectedDataType</key>
            <string>NSPrivacyCollectedDataTypeEmailAddress</string>
            <key>NSPrivacyCollectedDataTypeLinked</key>
            <true/>
            <key>NSPrivacyCollectedDataTypeTracking</key>
            <false/>
            <key>NSPrivacyCollectedDataTypePurposes</key>
            <array>
                <string>NSPrivacyCollectedDataTypePurposeAnalytics</string>
            </array>
        </dict>
    </array>

    <key>NSPrivacyAccessedAPITypes</key>
    <array>
        <dict>
            <key>NSPrivacyAccessedAPIType</key>
            <string>NSPrivacyAccessedAPICategoryUserDefaults</string>
            <key>NSPrivacyAccessedAPITypeReasons</key>
            <array>
                <string>CA92.1</string>
            </array>
        </dict>
    </array>
</dict>
</plist>
```

### 3.3 四个顶级键详解

| 键名 | 类型 | 说明 |
|------|------|------|
| `NSPrivacyTracking` | Boolean | 你的 App 或第三方 SDK 是否使用追踪功能。如果为 `true`，还必须在 `NSPrivacyTrackingDomains` 中列出追踪域名 |
| `NSPrivacyTrackingDomains` | Array | 追踪域名列表。只有当 `NSPrivacyTracking` 为 `true` 时才需要填写 |
| `NSPrivacyCollectedDataTypes` | Array | 声明你的 App 收集了哪些用户数据，以及收集目的、是否关联用户身份等 |
| `NSPrivacyAccessedAPITypes` | Array | 声明你的 App 使用了哪些需要说明原因的 API，以及使用原因代码 |

---

## 4. 声明数据收集类型：NSPrivacyCollectedDataTypes

### 4.1 每条数据声明的结构

每条数据声明是一个字典，包含以下字段：

| 字段 | 类型 | 说明 |
|------|------|------|
| `NSPrivacyCollectedDataType` | String | 数据类型标识符（见下表） |
| `NSPrivacyCollectedDataTypeLinked` | Boolean | 该数据是否与用户身份关联 |
| `NSPrivacyCollectedDataTypeTracking` | Boolean | 该数据是否用于追踪 |
| `NSPrivacyCollectedDataTypePurposes` | Array | 收集目的（可多个） |

### 4.2 常用数据类型标识符

| 标识符 | 含义 | 常见场景 |
|--------|------|---------|
| `NSPrivacyCollectedDataTypeEmailAddress` | 电子邮箱 | 注册、登录 |
| `NSPrivacyCollectedDataTypePhoneNumber` | 电话号码 | 注册、验证 |
| `NSPrivacyCollectedDataTypeName` | 姓名 | 用户资料 |
| `NSPrivacyCollectedDataTypeUserID` | 用户 ID | 账号系统 |
| `NSPrivacyCollectedDataTypeDeviceID` | 设备 ID | 设备识别 |
| `NSPrivacyCollectedDataTypeLocation` | 精确位置 | 地图、外卖 |
| `NSPrivacyCollectedDataTypeCoarseLocation` | 大致位置 | 天气、本地内容 |
| `NSPrivacyCollectedDataTypeBrowsingHistory` | 浏览历史 | 内容推荐 |
| `NSPrivacyCollectedDataTypeSearchHistory` | 搜索历史 | 搜索建议 |
| `NSPrivacyCollectedDataTypeUserContent` | 用户生成内容 | 社交、笔记 |
| `NSPrivacyCollectedDataTypePhotosOrVideos` | 照片或视频 | 相册 App |
| `NSPrivacyCollectedDataTypeContacts` | 通讯录 | 社交 App |
| `NSPrivacyCollectedDataTypeFinancialInfo` | 金融信息 | 支付、理财 |
| `NSPrivacyCollectedDataTypeHealthInfo` | 健康信息 | 健康 App |
| `NSPrivacyCollectedDataTypeSensitiveInfo` | 敏感信息 | 种族、宗教等 |
| `NSPrivacyCollectedDataTypeDiagnostics` | 诊断数据 | 崩溃日志 |
| `NSPrivacyCollectedDataTypePerformanceData` | 性能数据 | ANR 监控 |
| `NSPrivacyCollectedDataTypeProductInteraction` | 产品交互数据 | 点击、浏览统计 |
| `NSPrivacyCollectedDataTypeAdvertisingData` | 广告数据 | 广告投放 |
| `NSPrivacyCollectedDataTypeOtherUserData` | 其他用户数据 | 不属于以上分类的数据 |

### 4.3 收集目的标识符

| 标识符 | 含义 |
|--------|------|
| `NSPrivacyCollectedDataTypePurposeAnalytics` | 分析——了解 App 使用情况 |
| `NSPrivacyCollectedDataTypePurposeAppFunctionality` | App 功能——核心功能必需 |
| `NSPrivacyCollectedDataTypePurposeProductPersonalization` | 个性化——为用户定制体验 |
| `NSPrivacyCollectedDataTypePurposeAdvertising` | 广告——投放或衡量广告效果 |
| `NSPrivacyCollectedDataTypePurposeThirdPartyAdvertising` | 第三方广告 |
| `NSPrivacyCollectedDataTypePurposeDeveloperAdvertising` | 开发者广告 |
| `NSPrivacyCollectedDataTypePurposeOther` | 其他目的 |

### 4.4 实际示例

假设你的 App 收集了邮箱（用于登录）和产品交互数据（用于分析）：

```xml
<key>NSPrivacyCollectedDataTypes</key>
<array>
    <dict>
        <key>NSPrivacyCollectedDataType</key>
        <string>NSPrivacyCollectedDataTypeEmailAddress</string>
        <key>NSPrivacyCollectedDataTypeLinked</key>
        <true/>
        <key>NSPrivacyCollectedDataTypeTracking</key>
        <false/>
        <key>NSPrivacyCollectedDataTypePurposes</key>
        <array>
            <string>NSPrivacyCollectedDataTypePurposeAppFunctionality</string>
        </array>
    </dict>
    <dict>
        <key>NSPrivacyCollectedDataType</key>
        <string>NSPrivacyCollectedDataTypeProductInteraction</string>
        <key>NSPrivacyCollectedDataTypeLinked</key>
        <false/>
        <key>NSPrivacyCollectedDataTypeTracking</key>
        <false/>
        <key>NSPrivacyCollectedDataTypePurposes</key>
        <array>
            <string>NSPrivacyCollectedDataTypePurposeAnalytics</string>
        </array>
    </dict>
</array>
```

> 💡 **`Linked` 和 `Tracking` 怎么选？**
>
> - **Linked（是否关联身份）**：如果数据能直接或间接识别到具体用户，选 `true`。比如邮箱、用户 ID 关联的数据就是 `true`；匿名统计就是 `false`。
> - **Tracking（是否用于追踪）**：如果数据用于跨 App/网站追踪用户行为用于广告，选 `true`。大多数 App 不做广告追踪，选 `false` 即可。

---

## 5. 声明 API 使用原因：NSPrivacyAccessedAPITypes

### 5.1 每条 API 声明的结构

| 字段 | 类型 | 说明 |
|------|------|------|
| `NSPrivacyAccessedAPIType` | String | API 类别标识符 |
| `NSPrivacyAccessedAPITypeReasons` | Array | 使用原因代码列表 |

### 5.2 API 类别标识符

| 标识符 | API 类别 | 涉及的系统 API |
|--------|---------|---------------|
| `NSPrivacyAccessedAPICategoryUserDefaults` | UserDefaults | `UserDefaults.standard` 等 |
| `NSPrivacyAccessedAPICategoryFileTimestamp` | 文件时间戳 | `FileManager.attributesOfItem`、`stat()` 等 |
| `NSPrivacyAccessedAPICategorySystemBootTime` | 系统启动时间 | `ProcessInfo.processInfo.systemUptime`、`sysctl()` 等 |
| `NSPrivacyAccessedAPICategoryDiskSpace` | 磁盘空间 | `FileManager.attributesOfFileSystem`、`statfs()` 等 |
| `NSPrivacyAccessedAPICategoryActiveKeyboards` | 活跃键盘 | `UITextInputMode.activeInputModes` 等 |
| `NSPrivacyAccessedAPICategoryClipboard` | 剪贴板 | `UIPasteboard.general` 等 |

### 5.3 声明示例

```xml
<key>NSPrivacyAccessedAPITypes</key>
<array>
    <dict>
        <key>NSPrivacyAccessedAPIType</key>
        <string>NSPrivacyAccessedAPICategoryUserDefaults</string>
        <key>NSPrivacyAccessedAPITypeReasons</key>
        <array>
            <string>CA92.1</string>
        </array>
    </dict>
    <dict>
        <key>NSPrivacyAccessedAPIType</key>
        <string>NSPrivacyAccessedAPICategoryFileTimestamp</string>
        <key>NSPrivacyAccessedAPITypeReasons</key>
        <array>
            <string>C617.1</string>
        </array>
    </dict>
</array>
```

> ⚠️ **一个类别可以填多个原因代码**，如果你的 App 出于不同目的使用了同一个类别的 API，应该把所有适用原因都列上。

---

## 6. 常见 API 使用原因代码速查表

### 6.1 UserDefaults（NSPrivacyAccessedAPICategoryUserDefaults）

| 原因代码 | 含义 | 适用场景 |
|---------|------|---------|
| `CA92.1` | 读写用户配置、偏好设置 | 存储主题偏好、登录状态等 |
| `1C8F.1` | 在 App 内管理共享状态 | App 内部组件间共享数据 |
| `C56D.1` | 读写用户指定的文件 | 用户主动选择的数据 |

> 💡 **绝大多数 App 用 `CA92.1` 就够了**——因为大部分 UserDefaults 的用途就是存偏好设置。

### 6.2 File Timestamp（NSPrivacyAccessedAPICategoryFileTimestamp）

| 原因代码 | 含义 | 适用场景 |
|---------|------|---------|
| `C617.1` | 显示文件时间信息给用户 | 文件管理器、照片 App |
| `3B52.1` | 调试/性能分析 | 开发调试工具 |
| `DDA9.1` | 保护用户安全 | 安全检测、反欺诈 |
| `0A2A.1` | 访问 App 自身创建的文件的时间戳 | App 沙盒内文件管理 |

### 6.3 System Boot Time（NSPrivacyAccessedAPICategorySystemBootTime）

| 原因代码 | 含义 | 适用场景 |
|---------|------|---------|
| `35F9.1` | 计算绝对时间或时间间隔 | 计时器、闹钟 |
| `8FFB.1` | 调试/性能分析 | 性能监控工具 |
| `3D61.1` | 保护用户安全 | 安全检测 |

### 6.4 Disk Space（NSPrivacyAccessedAPICategoryDiskSpace）

| 原因代码 | 含义 | 适用场景 |
|---------|------|---------|
| `E174.1` | 显示磁盘空间信息给用户 | 清理工具、设置页面 |
| `7D9E.1` | 在写入前检查是否有足够空间 | 下载、缓存管理 |
| `85F4.1` | 调试/性能分析 | 性能监控 |
| `B728.1` | 保护用户安全 | 安全检测 |

### 6.5 Active Keyboards（NSPrivacyAccessedAPICategoryActiveKeyboards）

| 原因代码 | 含义 | 适用场景 |
|---------|------|---------|
| `3EC4.1` | 自定义键盘相关功能 | 自定义键盘 App |
| `54BD.1` | 根据键盘调整 UI 布局 | 聊天 App 适配不同键盘高度 |

### 6.6 Clipboard（NSPrivacyAccessedAPICategoryClipboard）

| 原因代码 | 含义 | 适用场景 |
|---------|------|---------|
| `2A09.1` | 读取剪贴板内容（用户主动触发） | 粘贴按钮 |
| `4E4A.1` | 写入剪贴板 | 复制按钮 |
| `0DD4.1` | 安全检测 | 检测剪贴板中的恶意内容 |

> ⚠️ **选错原因代码可能导致审核被拒！** 请根据实际用途选择最匹配的原因代码，不要随便填。

---

## 7. 第三方 SDK 的隐私清单

### 7.1 SDK 也需要隐私清单

Apple 不仅要求 App 本身提供隐私清单，还要求**第三方 SDK 也必须提供自己的隐私清单**。这意味着：

- 如果你集成了第三方 SDK，该 SDK 应该自带 `PrivacyInfo.xcprivacy`
- 如果 SDK 没有提供，你需要在 App 的隐私清单中替它声明

### 7.2 生活类比

这就像开餐厅时，你用的调料供应商（第三方 SDK）也需要提供自己的成分表。如果供应商不提供，你就得自己把它的成分写进你的公示牌里。

### 7.3 如何检查第三方 SDK 是否有隐私清单

**方法一：查看 SDK 的 XCFramework 或源码**

```bash
# 在你的项目目录下搜索
find . -name "PrivacyInfo.xcprivacy"
```

如果 SDK 自带了隐私清单，你会在它的 `.framework` 或 `.xcframework` 目录下找到这个文件。

**方法二：查看 SDK 官方文档**

主流 SDK 一般会在更新日志或文档中说明是否已添加隐私清单。

### 7.4 常见 SDK 的隐私清单状态

| SDK | 是否自带隐私清单 | 备注 |
|-----|:---:|------|
| Alamofire | ✅ | 5.8+ 版本已包含 |
| Firebase | ✅ | 10.x+ 版本已包含 |
| Google Ads SDK | ✅ | 已包含 |
| Facebook SDK | ✅ | 已包含 |
| Kingfisher | ✅ | 7.x+ 版本已包含 |
| SnapKit | ✅ | 已包含 |
| AFNetworking | ⚠️ | 较旧版本可能没有 |
| 一些小众/不再维护的 SDK | ❌ | 需要你手动补充 |

> ⚠️ **如果你的 App 使用了没有隐私清单的 SDK**，你需要在 App 自己的 `PrivacyInfo.xcprivacy` 中，把该 SDK 使用的隐私 API 和收集的数据也一并声明。

### 7.5 如何为缺少隐私清单的 SDK 补充声明

1. 查看 SDK 的源码或文档，了解它使用了哪些隐私 API
2. 在你 App 的 `PrivacyInfo.xcprivacy` 中添加对应的 API 声明和数据收集声明
3. 如果不确定 SDK 用了什么，可以用以下方法排查：

```bash
# 搜索 SDK 源码中是否使用了相关 API
grep -r "UserDefaults" Path/To/SDK/
grep -r "FileManager" Path/To/SDK/
grep -r "systemUptime" Path/To/SDK/
grep -r "statfs" Path/To/SDK/
grep -r "activeInputModes" Path/To/SDK/
```

---

## 8. 在 Xcode 中创建和配置 PrivacyInfo.xcprivacy

### 8.1 创建文件

**方法一：Xcode 自动创建（推荐）**

1. 在 Xcode 中，选择 **File → New → File...**
2. 搜索 **Privacy**，选择 **Privacy Manifest File**
3. 点击 Next，确保 Target 勾选了你的主 App Target
4. 点击 Create

Xcode 会自动创建一个模板文件，包含四个顶级键的框架。

**方法二：手动创建**

1. 在 Xcode 中，选择 **File → New → File...**
2. 选择 **Property List**
3. 文件名输入 `PrivacyInfo`
4. 创建后手动添加四个顶级键

### 8.2 编辑文件

创建完成后，在 Xcode 中点击 `PrivacyInfo.xcprivacy` 文件，你会看到可视化的 plist 编辑器：

**步骤一：设置 NSPrivacyTracking**

| Key | Type | Value |
|-----|------|-------|
| NSPrivacyTracking | Boolean | NO（大多数 App 选 NO） |

**步骤二：设置 NSPrivacyTrackingDomains**

| Key | Type | Value |
|-----|------|-------|
| NSPrivacyTrackingDomains | Array | 留空（除非你做广告追踪） |

**步骤三：添加 NSPrivacyCollectedDataTypes**

1. 点击 `NSPrivacyCollectedDataTypes` 左侧三角展开
2. 点击 + 号添加条目
3. 每个条目是一个 Dictionary，包含四个字段

**步骤四：添加 NSPrivacyAccessedAPITypes**

1. 点击 `NSPrivacyAccessedAPITypes` 左侧三角展开
2. 点击 + 号添加条目
3. 每个条目是一个 Dictionary，包含 API 类别和原因代码

### 8.3 以源码方式编辑

如果你更习惯直接编辑 XML，可以右键点击文件 → **Open As → Source Code**，然后直接编辑 XML 内容。

一个最常见的最小配置（只用了 UserDefaults 的 App）：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
 "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>NSPrivacyTracking</key>
    <false/>
    <key>NSPrivacyTrackingDomains</key>
    <array/>
    <key>NSPrivacyCollectedDataTypes</key>
    <array>
        <dict>
            <key>NSPrivacyCollectedDataType</key>
            <string>NSPrivacyCollectedDataTypeProductInteraction</string>
            <key>NSPrivacyCollectedDataTypeLinked</key>
            <false/>
            <key>NSPrivacyCollectedDataTypeTracking</key>
            <false/>
            <key>NSPrivacyCollectedDataTypePurposes</key>
            <array>
                <string>NSPrivacyCollectedDataTypePurposeAppFunctionality</string>
            </array>
        </dict>
    </array>
    <key>NSPrivacyAccessedAPITypes</key>
    <array>
        <dict>
            <key>NSPrivacyAccessedAPIType</key>
            <string>NSPrivacyAccessedAPICategoryUserDefaults</string>
            <key>NSPrivacyAccessedAPITypeReasons</key>
            <array>
                <string>CA92.1</string>
            </array>
        </dict>
    </array>
</dict>
</plist>
```

### 8.4 确认文件被包含在构建中

> ⚠️ **关键步骤！** 创建文件后，务必确认它被包含在 App Target 的 Build Phase 中：

1. 点击项目文件 → 选择你的 App Target → **Build Phases**
2. 展开 **Copy Bundle Resources**
3. 检查 `PrivacyInfo.xcprivacy` 是否在列表中
4. 如果没有，点击 + 号手动添加

---

## 9. 审核被拒案例与解决方案

### 9.1 常见被拒原因

| 被拒原因 | 典型错误信息 | 解决方案 |
|---------|------------|---------|
| 缺少隐私清单 | "Your app uses API that requires a reason" | 添加 PrivacyInfo.xcprivacy 并声明对应 API |
| 原因代码不匹配 | "The selected reason doesn't match the API usage" | 选择与实际用途匹配的原因代码 |
| 缺少 API 声明 | "API X is used but not declared" | 在 NSPrivacyAccessedAPITypes 中补充声明 |
| 数据收集未声明 | "Your app collects data but doesn't declare it" | 在 NSPrivacyCollectedDataTypes 中补充 |
| 第三方 SDK 缺少声明 | "SDK X uses API Y without declaration" | 更新 SDK 或在 App 清单中补充 |

### 9.2 案例一：使用了 UserDefaults 但未声明

**错误信息：**

> Your app uses the UserDefaults API, which requires a declared reason in the Privacy Manifest.

**解决方案：**

在 `PrivacyInfo.xcprivacy` 中添加：

```xml
<key>NSPrivacyAccessedAPITypes</key>
<array>
    <dict>
        <key>NSPrivacyAccessedAPIType</key>
        <string>NSPrivacyAccessedAPICategoryUserDefaults</string>
        <key>NSPrivacyAccessedAPITypeReasons</key>
        <array>
            <string>CA92.1</string>
        </array>
    </dict>
</array>
```

### 9.3 案例二：第三方 SDK 使用了文件时间戳 API

**错误信息：**

> The SDK "SomeAnalytics" uses file timestamp APIs that require a privacy manifest declaration.

**解决方案：**

1. 首先尝试更新该 SDK 到最新版本，看是否已包含隐私清单
2. 如果 SDK 未更新，在你的 App 清单中补充声明：

```xml
<dict>
    <key>NSPrivacyAccessedAPIType</key>
    <string>NSPrivacyAccessedAPICategoryFileTimestamp</string>
    <key>NSPrivacyAccessedAPITypeReasons</key>
    <array>
        <string>3B52.1</string>
    </array>
</dict>
```

### 9.4 案例三：原因代码与实际用途不符

**错误信息：**

> The declared reason for API usage doesn't align with the app's functionality.

**解决方案：**

仔细检查你的 App 实际使用该 API 的目的，选择最匹配的原因代码。比如你用 UserDefaults 是为了存用户偏好设置，就选 `CA92.1`，而不是随便选一个。

> 💡 **预防建议**：提交审核前，用 `xcodebuild` 的隐私报告功能检查：
>
> ```bash
> xcodebuild -project YourProject.xcodeproj \
>   -scheme YourScheme \
>   -destination generic/platform=ios \
>   -privacyManifest \
>   build
> ```
>
> 构建完成后在构建日志中查看隐私 API 使用报告。

---

## 10. 最佳实践与检查清单

### 10.1 最佳实践

**1. 尽早添加，不要等审核被拒**

项目一开始就创建 `PrivacyInfo.xcprivacy`，随着功能开发逐步补充。不要等到提交审核时才发现缺少声明。

**2. 声明要完整，宁可多声明不要遗漏**

如果你不确定是否使用了某个 API，先声明上。多声明不会导致被拒，但遗漏会。

**3. 保持与 App Store Connect 的隐私说明一致**

你在 `PrivacyInfo.xcprivacy` 中声明的数据收集类型，应该与 App Store Connect 中填写的"App 隐私"信息一致。两者矛盾也会导致审核问题。

**4. 及时更新第三方 SDK**

保持第三方 SDK 为最新版本，确保它们包含了最新的隐私清单。

**5. 原因代码要诚实**

不要为了通过审核而选择不匹配的原因代码。Apple 会审查你的实际代码用途。

### 10.2 提交前检查清单

提交审核前，对照以下清单逐项检查：

| # | 检查项 | 是否完成 |
|---|--------|:-------:|
| 1 | 已创建 `PrivacyInfo.xcprivacy` 文件 | ☐ |
| 2 | 文件已包含在 App Target 的 Copy Bundle Resources 中 | ☐ |
| 3 | `NSPrivacyTracking` 已正确设置（大多数 App 为 false） | ☐ |
| 4 | `NSPrivacyTrackingDomains` 已填写（如适用） | ☐ |
| 5 | `NSPrivacyCollectedDataTypes` 已声明所有收集的数据类型 | ☐ |
| 6 | 每条数据声明的 Linked/Tracking/Purposes 已正确填写 | ☐ |
| 7 | `NSPrivacyAccessedAPITypes` 已声明所有使用的隐私 API | ☐ |
| 8 | 每个 API 类别的原因代码与实际用途匹配 | ☐ |
| 9 | 所有第三方 SDK 已更新到包含隐私清单的版本 | ☐ |
| 10 | 缺少隐私清单的 SDK 已在 App 清单中补充声明 | ☐ |
| 11 | 隐私清单内容与 App Store Connect 隐私说明一致 | ☐ |
| 12 | 已通过构建日志检查隐私 API 使用报告 | ☐ |

### 10.3 快速排查命令

在项目根目录运行以下命令，快速排查可能遗漏的隐私 API：

```bash
echo "=== UserDefaults ==="
grep -rn "UserDefaults" --include="*.swift" --include="*.m" .

echo "=== File Timestamp ==="
grep -rn "attributesOfItem\|attributesOfFileSystem\|stat(" --include="*.swift" --include="*.m" .

echo "=== System Boot Time ==="
grep -rn "systemUptime\|sysctl\|KERN_BOOTTIME" --include="*.swift" --include="*.m" .

echo "=== Disk Space ==="
grep -rn "statfs\|volumeAvailableCapacity\|volumeTotalCapacity" --include="*.swift" --include="*.m" .

echo "=== Active Keyboards ==="
grep -rn "activeInputModes" --include="*.swift" --include="*.m" .

echo "=== Clipboard ==="
grep -rn "UIPasteboard\|NSPasteboard" --include="*.swift" --include="*.m" .
```

> 💡 **小技巧**：把上面的命令保存成脚本（如 `check_privacy.sh`），每次提交前跑一遍，省时省力。

---

## 小结

| 知识点 | 核心要点 |
|--------|---------|
| Privacy Manifest 是什么 | App 的"隐私声明书"，Apple 2024 年起强制要求 |
| 谁需要提供 | 使用指定 API 的 App、包含第三方 SDK 的 App |
| 文件结构 | 四个顶级键：Tracking、TrackingDomains、CollectedDataTypes、AccessedAPITypes |
| 数据收集声明 | 声明收集了什么数据、是否关联身份、是否追踪、收集目的 |
| API 使用声明 | 声明用了哪类 API、使用原因代码 |
| 原因代码选择 | 根据实际用途选择，UserDefaults 大多选 CA92.1 |
| 第三方 SDK | 优先更新到自带清单的版本，否则在 App 清单中补充 |
| Xcode 操作 | File → New → Privacy Manifest File，注意加入 Copy Bundle Resources |
| 审核被拒 | 多因缺少声明或原因代码不匹配，诚实完整是关键 |
| 最佳实践 | 尽早添加、完整声明、与 Connect 一致、及时更新 SDK |

> 💡 **一句话总结**：隐私清单就是给 App 的隐私行为做一次"如实申报"——用了什么 API 就声明什么，收集了什么数据就写什么，诚实完整就不会出问题。
