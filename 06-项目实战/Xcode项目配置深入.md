# Xcode 项目配置深入

## 本章目标

- 深入理解 Xcode 项目底层文件结构，掌握 .xcodeproj / .xcworkspace / .pbxproj 的组成与关系
- 理解 Target 与 Scheme 的概念，能进行多 Target 管理和 Scheme 配置
- 掌握 Build Configurations 的原理与自定义方法，熟练使用 xcconfig 文件管理多环境配置
- 了解关键编译选项的含义与优化策略，能根据场景调整编译参数
- 掌握 Capabilities 与 Entitlements 的配置方法，理解各类系统权限的声明方式
- 能独立搭建 Dev / Staging / Prod 三套环境配置，实现 API URL、Bundle ID、App Icon 等的自动切换
- 建立配置最佳实践意识，能排查常见配置冲突，保障团队协作一致性

---

## 1. Xcode 项目结构解析

### 1.1 .xcodeproj 与 .xcworkspace

打开一个 iOS 项目时，你在 Finder 中看到的通常是一个 `.xcodeproj` 文件。它其实不是普通文件，而是一个**目录包（bundle）**——就像一栋大楼，外表看起来是一个整体，内部却有很多房间。

```
MyApp.xcodeproj/
├── project.pbxproj       # 项目核心配置（"建筑图纸"）
├── project.xcworkspace/  # 内嵌的 workspace
│   └── contents.xcworkspacedata
└── xcuserdata/           # 用户个人数据（断点、窗口布局等）
    └── <username>.xcuserdatad/
```

而 `.xcworkspace` 是更高层级的容器，用于将多个 `.xcodeproj` 组织在一起。引入 CocoaPods 后，它会生成一个 `.xcworkspace`，把你的主项目和 Pods 项目打包在一起。

> 💡 类比：`.xcodeproj` 是一栋独立建筑，`.xcworkspace` 是一个园区——园区里可以有多栋建筑，它们共享道路和基础设施。

| 特性 | .xcodeproj | .xcworkspace |
|------|-----------|-------------|
| 层级 | 单个项目 | 可包含多个项目 |
| 使用场景 | 无第三方依赖的简单项目 | 使用 CocoaPods 或多项目协作 |
| 打开方式 | 双击直接打开 | 双击打开（推荐） |
| 生成方式 | Xcode 新建项目自动创建 | CocoaPods `pod install` 自动生成 |

> ⚠️ 使用 CocoaPods 后，务必打开 `.xcworkspace` 而非 `.xcodeproj`，否则会因找不到 Pods 依赖而编译失败。

### 1.2 project.pbxproj 关键字段解读

`project.pbxproj` 是一个旧式 plist 格式的文本文件，记录了项目的全部结构信息。它是整个项目的"基因图谱"——文件引用、编译配置、Target 定义全部存储其中。

```
// !$*UTF8*$!
archiveVersion = 1;
objects = {

/* PBXBuildFile section — 编译文件引用 */
    A1B2C3D4 /* MainViewController.swift in Sources */ = {
        isa = PBXBuildFile;
        fileRef = E5F6G7H8 /* MainViewController.swift */;
    };

/* PBXFileReference section — 文件引用 */
    E5F6G7H8 /* MainViewController.swift */ = {
        isa = PBXFileReference;
        lastKnownFileType = sourcecode.swift;
        path = MainViewController.swift;
        sourceTree = "<group>";
    };

/* PBXGroup section — 文件夹/分组 */
    I9J0K1L2 /* Views */ = {
        isa = PBXGroup;
        children = (E5F6G7H8);
        path = Views;
        sourceTree = "<group>";
    };

/* XCBuildConfiguration section — 编译配置 */
    M3N4O5P6 /* Debug */ = {
        isa = XCBuildConfiguration;
        buildSettings = {
            PRODUCT_BUNDLE_IDENTIFIER = com.example.MyApp;
            SWIFT_VERSION = 5.0;
        };
        name = Debug;
    };
};
```

关键 section 一览：

| Section | 作用 | 类比 |
|---------|------|------|
| PBXFileReference | 记录项目中每个文件的引用 | 户口本上的每个人 |
| PBXBuildFile | 记录参与编译的文件 | 实际上工的人员名单 |
| PBXGroup | 文件分组（文件夹结构） | 部门组织架构 |
| PBXNativeTarget | Target 定义 | 一条产品线 |
| XCBuildConfiguration | 编译配置项 | 产品规格参数 |
| XCConfigurationList | 配置列表（Debug/Release） | 规格参数表 |

> ⚠️ `project.pbxproj` 文件极易因合并冲突而损坏。团队协作时应避免多人同时修改项目结构，推荐使用 `.xcodeproj` 的 JSON 格式（Xcode 13+）来降低冲突概率。

---

## 2. Target 与 Scheme

### 2.1 Target 是什么

Target 是 Xcode 中构建产物的基本单元——它定义了"用什么源文件、按什么规则、产出什么东西"。一个项目可以有多个 Target，就像一家工厂的多条生产线，共享原材料（源码），但生产不同的产品。

```
MyApp.xcodeproj
├── Target: MyApp (主 App)
│   ├── Sources: 所有 Swift 文件
│   ├── Resources: Assets、Storyboard
│   └── Build Settings: Bundle ID = com.example.myapp
├── Target: MyApp-Staging (Staging App)
│   ├── Sources: 同上
│   └── Build Settings: Bundle ID = com.example.myapp.staging
└── Target: MyAppTests (单元测试)
    └── Sources: Tests/
```

| Target 属性 | 说明 |
|------------|------|
| Bundle Identifier | 唯一标识，决定 App 在设备上的身份 |
| Info.plist | 配置文件，包含权限声明、URL Scheme 等 |
| Build Settings | 编译参数集合 |
| Build Phases | 编译阶段（Sources、Resources、Frameworks） |
| Capabilities | 系统能力声明（推送、签名等） |

### 2.2 多 Target 管理

多 Target 的典型场景：

- **免费版 / 付费版**：共享核心代码，差异化功能
- **开发版 / 生产版**：不同 Bundle ID，可同时安装
- **白标产品**：同一套代码，不同品牌和资源

创建多 Target 的方式：

1. **Duplicate Target**：在 Xcode 中右键 Target → Duplicate，适合快速创建变体
2. **手动创建**：File → New → Target，更灵活可控

在代码中区分当前 Target：

```swift
#if DEBUG
let apiBaseURL = URL(string: "https://dev.api.example.com")!
#elseif STAGING
let apiBaseURL = URL(string: "https://staging.api.example.com")!
#else
let apiBaseURL = URL(string: "https://api.example.com")!
#endif
```

> 💡 多 Target 方案虽然直观，但维护成本随 Target 数量线性增长。对于仅需要切换环境参数的场景，推荐使用 Build Configuration + xcconfig 方案（见第 4 节）。

### 2.3 Scheme 配置

Scheme 定义了 Target 的运行方式——点击 Run、Test、Profile 时各执行什么操作。

```
Scheme 配置项：
├── Run          → Build Configuration: Debug
├── Test         → Build Configuration: Debug
├── Profile      → Build Configuration: Release
├── Analyze      → Build Configuration: Debug
└── Archive      → Build Configuration: Release
```

### 2.4 Shared Scheme

Scheme 默认只存在于本地（`xcuserdata` 目录），不会被 Git 追踪。勾选 **Shared** 后，Scheme 会保存到 `xcshareddata/xcschemes/` 目录，团队成员可以共享同一套 Scheme 配置。

> ⚠️ CI/CD 环境（如 Xcode Cloud）必须使用 Shared Scheme，否则无法找到对应的构建方案。团队项目务必将所有 Scheme 设为 Shared。

---

## 3. Build Configurations

### 3.1 Debug 与 Release 的区别

Xcode 默认创建两个 Build Configuration：Debug 和 Release。它们的区别就像"练习模式"和"正式比赛"——练习时你需要教练提示（日志）、保护垫（断点），比赛时你需要最佳表现（优化）。

| 配置项 | Debug | Release |
|--------|-------|---------|
| Optimization Level | `-Onone`（无优化） | `-O`（全量优化） |
| Debug Information | DWARF with dSYM | DWARF with dSYM |
| Assertions | 启用 | 禁用 |
| Code Signing | Development | Distribution |
| Strip Debug Symbols | NO | YES |
| Swift Compiler - Assertions | Enable | Disable |

### 3.2 自定义 Configuration

实际项目中，Debug 和 Release 往往不够用。添加 Staging 配置的步骤：

1. 选中项目 → Info → Configurations
2. 点击 `+` → Duplicate "Debug" Configuration → 命名为 `Staging`
3. 在 Scheme 中将 Run 的 Build Configuration 改为 `Staging`

### 3.3 配置切换

在 Scheme Editor 中，可以为每个 Action（Run / Test / Profile / Analyze / Archive）指定不同的 Build Configuration：

```
Run     → Debug（日常开发，断点调试）
Test    → Debug（单元测试）
Profile → Release（性能分析，需真实性能表现）
Analyze → Debug（静态分析）
Archive → Release（打包上架）
```

> 💡 建议为 Staging 环境创建独立 Scheme（如 `MyApp Staging`），将 Run 默认指向 Staging Configuration，避免频繁手动切换。

---

## 4. xcconfig 文件实战

### 4.1 为什么使用 xcconfig

`xcconfig` 文件是纯文本的键值对配置文件，用于替代在 Xcode Build Settings UI 中逐项修改的方式。它的优势就像把散落在各处的便签纸整理成一本手册——集中管理、版本可控、易于复用。

| 特性 | Build Settings UI | xcconfig 文件 |
|------|------------------|--------------|
| 可读性 | 分散在多层菜单中 | 纯文本，一目了然 |
| 版本控制 | 变更隐藏在 pbxproj 中 | 变更清晰可 diff |
| 复用性 | 每个 Target 单独配置 | 可 include 共享配置 |
| 冲突风险 | pbxproj 合并冲突频繁 | 纯文本，冲突易解决 |
| 团队协作 | 需口头传达修改 | Git 提交即可同步 |

### 4.2 变量定义

创建 xcconfig 文件：File → New → File → Configuration Settings File。

```
// Dev.xcconfig
API_BASE_URL = https://dev.api.example.com
APP_NAME = MyApp Dev
BUNDLE_ID_SUFFIX = .dev
```

在 Build Settings 中引用这些变量：

- `PRODUCT_BUNDLE_IDENTIFIER = com.example.myapp$(BUNDLE_ID_SUFFIX)`
- `INFOPLIST_KEY_CFBundleDisplayName = $(APP_NAME)`

### 4.3 条件编译

xcconfig 支持基于 SDK 和架构的条件赋值，语法为 `SETTING[sdk=条件] = 值`：

```
// 仅模拟器下禁用 Bitcode
ENABLE_BITCODE[sdk=iphonesimulator*] = NO

// 仅 ARM64 设备启用特定标志
OTHER_SWIFT_FLAGS[sdk=iphoneos*][arch=arm64] = -D ARM64_DEVICE

// 不同 SDK 的 Header Search Paths
HEADER_SEARCH_PATHS[sdk=iphoneos*] = $(SRCROOT)/iOS/Libraries/include
HEADER_SEARCH_PATHS[sdk=iphonesimulator*] = $(SRCROOT)/Sim/Libraries/include
```

### 4.4 #include 继承

xcconfig 支持 `#include` 指令，实现配置的继承与分层——就像面向对象编程中的父类与子类，公共配置放在"父类"，差异配置放在"子类"。

```
// Base.xcconfig（公共配置）
SWIFT_VERSION = 5.0
IPHONEOS_DEPLOYMENT_TARGET = 15.0
DEVELOPMENT_TEAM = ABC1234567
CODE_SIGN_IDENTITY = Apple Development

// Dev.xcconfig（开发环境）
#include "Base.xcconfig"
API_BASE_URL = https://dev.api.example.com
APP_NAME = MyApp Dev
BUNDLE_ID_SUFFIX = .dev
GCC_PREPROCESSOR_DEFINITIONS = DEBUG=1 $(inherited)
SWIFT_ACTIVE_COMPILATION_CONDITIONS = DEBUG $(inherited)

// Staging.xcconfig（预发布环境）
#include "Base.xcconfig"
API_BASE_URL = https://staging.api.example.com
APP_NAME = MyApp Staging
BUNDLE_ID_SUFFIX = .staging
SWIFT_ACTIVE_COMPILATION_CONDITIONS = STAGING $(inherited)

// Prod.xcconfig（生产环境）
#include "Base.xcconfig"
API_BASE_URL = https://api.example.com
APP_NAME = MyApp
BUNDLE_ID_SUFFIX =
SWIFT_ACTIVE_COMPILATION_CONDITIONS = RELEASE $(inherited)
```

> ⚠️ `$(inherited)` 非常重要！它表示在原有值的基础上追加，而非覆盖。不加 `$(inherited)` 会导致 Base 中的设置被清空。

### 4.5 多环境配置最佳实践

完整的 Dev / Staging / Prod 三环境配置结构：

```
MyApp/
├── Config/
│   ├── Base.xcconfig
│   ├── Dev.xcconfig
│   ├── Staging.xcconfig
│   └── Prod.xcconfig
```

在 Xcode 中为每个 Configuration 指定对应的 xcconfig：

1. 选中项目 → Info → Configurations
2. 展开 Debug → 选择 `Dev.xcconfig`
3. 展开 Staging → 选择 `Staging.xcconfig`
4. 展开 Release → 选择 `Prod.xcconfig`

在 Swift 代码中读取 xcconfig 变量：

```swift
enum Environment {
    static var apiBaseURL: URL {
        guard let urlString = Bundle.main.infoDictionary?["API_BASE_URL"] as? String,
              let url = URL(string: urlString) else {
            fatalError("无法获取 API_BASE_URL，请检查 xcconfig 配置")
        }
        return url
    }

    static var isDev: Bool {
        #if DEBUG
        return true
        #else
        return false
        #endif
    }

    static var isStaging: Bool {
        #if STAGING
        return true
        #else
        return false
        #endif
    }
}
```

> 💡 xcconfig 中定义的变量若要在运行时通过 `Bundle.main.infoDictionary` 读取，必须在 Info.plist 中添加对应 key 并设值为 `$(VARIABLE_NAME)`，否则运行时无法获取。

---

## 5. 编译选项优化

### 5.1 Optimization Level

Swift 编译器的优化级别直接影响编译速度和运行性能：

| 优化级别 | 标识 | 适用场景 | 说明 |
|---------|------|---------|------|
| None | `-Onone` | Debug | 无优化，编译最快，调试体验最佳 |
| Speed | `-O` | Release | 优化执行速度，标准发布配置 |
| Size | `-Osize` | Release (可选) | 优化包体积，速度略有牺牲 |

```
// Dev.xcconfig
SWIFT_OPTIMIZATION_LEVEL = -Onone

// Prod.xcconfig
SWIFT_OPTIMIZATION_LEVEL = -O
// 如果包体积敏感，可改用：
// SWIFT_OPTIMIZATION_LEVEL = -Osize
```

> 💡 `-Osize` 通常能减小 5%~30% 的二进制体积，对大型项目效果更明显。建议在 Release 配置中实测对比后再决定。

### 5.2 Swift Compiler Flags

通过 `SWIFT_ACTIVE_COMPILATION_CONDITIONS` 定义条件编译标志，在代码中用 `#if` 判断：

```
// Dev.xcconfig
SWIFT_ACTIVE_COMPILATION_CONDITIONS = DEBUG LOGGING NETWORK_LOG

// Staging.xcconfig
SWIFT_ACTIVE_COMPILATION_CONDITIONS = STAGING LOGGING

// Prod.xcconfig
SWIFT_ACTIVE_COMPILATION_CONDITIONS = RELEASE
```

```swift
#if LOGGING
func logDebug(_ message: String) {
    print("[DEBUG] \(message)")
}
#else
func logDebug(_ message: String) {}
#endif

#if NETWORK_LOG
func logNetworkRequest(_ url: String, params: [String: Any]) {
    print("[NETWORK] \(url) \(params)")
}
#else
func logNetworkRequest(_ url: String, params: [String: Any]) {}
#endif
```

### 5.3 Other Linker Flags

`OTHER_LDFLAGS` 控制链接器的行为，常见用法：

| 标志 | 作用 | 使用场景 |
|------|------|---------|
| `-ObjC` | 加载所有 Objective-C 类和分类 | 使用 OC 静态库中的分类 |
| `-all_load` | 加载所有静态库 | 解决分类方法未加载问题 |
| `-force_load` | 加载指定静态库 | 精确控制，避免全局影响 |
| `-framework` | 链接指定框架 | 手动链接系统框架 |

```
// Base.xcconfig
OTHER_LDFLAGS = $(inherited) -ObjC
```

> ⚠️ `-all_load` 会导致链接所有符号，可能产生重复符号错误。优先使用 `-ObjC`，仅在必要时使用 `-force_load` 指定具体库路径。

### 5.4 Header Search Paths

`HEADER_SEARCH_PATHS` 告诉编译器去哪里查找头文件，在使用 C/C++ 库时尤为重要：

```
// Base.xcconfig
HEADER_SEARCH_PATHS = $(inherited) $(SRCROOT)/Vendor/include

// 如果使用递归搜索（不推荐，影响编译速度）
// HEADER_SEARCH_PATHS = $(inherited) $(SRCROOT)/Vendor/include/**
```

| 路径变量 | 含义 |
|---------|------|
| `$(SRCROOT)` | 项目根目录（.xcodeproj 所在目录） |
| `$(BUILD_DIR)` | 构建输出目录 |
| `$(CONFIGURATION)` | 当前 Build Configuration 名称 |
| `$(PLATFORM_NAME)` | 当前平台名称（iphoneos / iphonesimulator） |

---

## 6. Capabilities 与 Entitlements

### 6.1 .entitlements 文件

`.entitlements` 文件是 App 向系统声明"我需要什么能力"的清单——就像去政府部门办事，你得先在申请表上勾选需要办理的业务，工作人员才会给你对应权限。

```
// MyApp.entitlements
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>aps-environment</key>
    <string>development</string>
    <key>com.apple.developer.applesignin</key>
    <true/>
    <key>com.apple.security.application-groups</key>
    <array>
        <string>group.com.example.myapp</string>
    </array>
</dict>
</plist>
```

### 6.2 常用 Capabilities 配置

| Capability | Entitlement Key | 用途 |
|-----------|----------------|------|
| Push Notifications | `aps-environment` | 接收远程推送通知 |
| Sign in with Apple | `com.apple.developer.applesignin` | Apple 登录 |
| App Groups | `com.apple.security.application-groups` | 进程间数据共享 |
| Keychain Sharing | `keychain-access-groups` | 钥匙串数据共享 |
| Associated Domains | `com.apple.developer.associated-domains` | 通用链接、App Clip |
| HealthKit | `com.apple.developer.healthkit` | 读写健康数据 |
| Camera | 无需 entitlement（Info.plist 权限即可） | 相机访问 |

### 6.3 多环境 Entitlements 配置

不同环境可能需要不同的 entitlements（如推送环境不同）：

```
// Dev.xcconfig
CODE_SIGN_ENTITLEMENTS = MyApp/Config/Dev.entitlements

// Staging.xcconfig
CODE_SIGN_ENTITLEMENTS = MyApp/Config/Staging.entitlements

// Prod.xcconfig
CODE_SIGN_ENTITLEMENTS = MyApp/Config/Prod.entitlements
```

各环境 entitlements 的差异：

```xml
<!-- Dev.entitlements -->
<key>aps-environment</key>
<string>development</string>

<!-- Prod.entitlements -->
<key>aps-environment</key>
<string>production</string>
```

> ⚠️ Capability 必须同时在 Apple Developer Portal 的 App ID 中开启，否则即使 entitlements 文件配置正确，签名也会失败。

---

## 7. 多环境配置实战

### 7.1 整体方案设计

本节将把前面学到的知识串联起来，搭建一套完整的三环境配置体系。就像一家连锁餐厅——中央厨房统一采购食材（Base 配置），各分店根据当地口味微调菜单（环境差异配置）。

```
Config/
├── Base.xcconfig          # 公共配置
├── Dev.xcconfig           # 开发环境
├── Staging.xcconfig       # 预发布环境
├── Prod.xcconfig          # 生产环境
├── Dev.entitlements       # 开发环境权限
├── Staging.entitlements   # 预发布环境权限
└── Prod.entitlements      # 生产环境权限
```

### 7.2 API URL 切换

在 xcconfig 中定义 API 地址：

```
// Dev.xcconfig
API_BASE_URL = https://dev.api.example.com
CDN_BASE_URL = https://dev-cdn.example.com

// Staging.xcconfig
API_BASE_URL = https://staging.api.example.com
CDN_BASE_URL = https://staging-cdn.example.com

// Prod.xcconfig
API_BASE_URL = https://api.example.com
CDN_BASE_URL = https://cdn.example.com
```

在 Info.plist 中添加：

```xml
<key>API_BASE_URL</key>
<string>$(API_BASE_URL)</string>
<key>CDN_BASE_URL</key>
<string>$(CDN_BASE_URL)</string>
```

在代码中封装访问：

```swift
struct APIConfig {
    static let baseURL: URL = {
        guard let string = Bundle.main.object(forInfoDictionaryKey: "API_BASE_URL") as? String,
              let url = URL(string: string) else {
            fatalError("API_BASE_URL 配置缺失或无效")
        }
        return url
    }()

    static let cdnURL: URL = {
        guard let string = Bundle.main.object(forInfoDictionaryKey: "CDN_BASE_URL") as? String,
              let url = URL(string: string) else {
            fatalError("CDN_BASE_URL 配置缺失或无效")
        }
        return url
    }()
}
```

### 7.3 Bundle ID 后缀

通过 `BUNDLE_ID_SUFFIX` 实现不同环境安装为独立 App：

```
// Dev.xcconfig
BUNDLE_ID_SUFFIX = .dev

// Staging.xcconfig
BUNDLE_ID_SUFFIX = .staging

// Prod.xcconfig
BUNDLE_ID_SUFFIX =
```

在 Build Settings 中设置：

```
PRODUCT_BUNDLE_IDENTIFIER = com.example.myapp$(BUNDLE_ID_SUFFIX)
```

效果：

| 环境 | Bundle ID | 可否共存 |
|------|-----------|---------|
| Dev | com.example.myapp.dev | ✅ |
| Staging | com.example.myapp.staging | ✅ |
| Prod | com.example.myapp | ✅ |

> 💡 三套环境可同时安装在同一台设备上，方便开发者在测试生产版本的同时对比开发版本。

### 7.4 不同环境不同 App Icon

在 xcconfig 中指定不同的 App Icon 资源名：

```
// Dev.xcconfig
APP_ICON_NAME = AppIcon-Dev

// Staging.xcconfig
APP_ICON_NAME = AppIcon-Staging

// Prod.xcconfig
APP_ICON_NAME = AppIcon
```

在 Build Settings 中设置：

```
ASSETCATALOG_COMPILER_APPICON_NAME = $(APP_ICON_NAME)
```

然后在 `Assets.xcassets` 中创建对应的 App Icon 集：

```
Assets.xcassets/
├── AppIcon.appiconset/           # 生产环境
├── AppIcon-Dev.appiconset/       # 开发环境（加红色角标 "DEV"）
└── AppIcon-Staging.appiconset/   # 预发布环境（加橙色角标 "STG"）
```

### 7.5 Info.plist 变量替换

xcconfig 变量通过 `$(VARIABLE)` 语法在 Info.plist 中生效，Xcode 在编译时会自动替换：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>CFBundleDisplayName</key>
    <string>$(APP_NAME)</string>
    <key>CFBundleIdentifier</key>
    <string>com.example.myapp$(BUNDLE_ID_SUFFIX)</string>
    <key>API_BASE_URL</key>
    <string>$(API_BASE_URL)</string>
</dict>
</plist>
```

变量替换的完整链路：

```
xcconfig 定义变量 → Build Settings 读取 → Info.plist 中 $(VAR) 替换 → 运行时 Bundle.main 读取
```

> ⚠️ 如果 Info.plist 中的变量未被替换（显示为字面 `$(VAR)`），通常是因为 xcconfig 未正确关联到对应的 Build Configuration，或变量名拼写不一致。

---

## 8. 常见配置问题与最佳实践

### 8.1 配置冲突排查

当 Build Setting 在 UI 中显示为粗体时，表示该值在 Target 级别被覆盖。排查冲突的方法：

1. **查看定义来源**：在 Build Settings 中选中某项，按 `Cmd+Option+Enter` 查看 Quick Help，会显示该值在哪个层级定义
2. **层级优先级**：Target > Project > xcconfig（后定义的优先级更高）
3. **$(inherited) 缺失**：如果追加型设置（如 `OTHER_SWIFT_FLAGS`）缺少 `$(inherited)`，会导致上层配置被覆盖

```
// ❌ 错误：覆盖了 Base 中的所有 flags
OTHER_SWIFT_FLAGS = -D MY_FLAG

// ✅ 正确：在继承基础上追加
OTHER_SWIFT_FLAGS = $(inherited) -D MY_FLAG
```

### 8.2 .gitignore 中应忽略的文件

```
# Xcode 用户数据（不应提交）
*.xcuserdata/
xcuserdata/

# 构建产物
build/
DerivedData/

# CocoaPods（Pods 目录可选择性忽略）
# Pods/  # 如果团队约定 pod install 则可忽略

# SPM 解析缓存
.build/

# macOS 系统文件
.DS_Store

# 临时文件
*.swp
*~
```

| 文件/目录 | 是否提交 | 原因 |
|-----------|---------|------|
| `*.xcuserdata/` | ❌ 忽略 | 包含个人断点、窗口布局 |
| `xcshareddata/` | ✅ 提交 | Shared Scheme 和 Breakpoint |
| `project.pbxproj` | ✅ 提交 | 项目结构定义 |
| `*.xcconfig` | ✅ 提交 | 环境配置 |
| `*.entitlements` | ✅ 提交 | 权限声明 |
| `DerivedData/` | ❌ 忽略 | 可重新生成的构建缓存 |

### 8.3 团队协作配置一致性

确保团队成员配置一致的关键措施：

| 措施 | 说明 |
|------|------|
| Shared Scheme | 所有 Scheme 设为 Shared，确保 CI 和团队成员使用相同配置 |
| xcconfig 文件 | 编译配置通过 xcconfig 管理，而非 UI 手动修改 |
| 统一 Xcode 版本 | 在 README 中注明最低 Xcode 版本要求 |
| Swift 版本锁定 | 在 Base.xcconfig 中指定 `SWIFT_VERSION` |
| Deployment Target 锁定 | 在 Base.xcconfig 中指定 `IPHONEOS_DEPLOYMENT_TARGET` |
| lint 检查 | 在 CI 中加入 `xcodebuild -showBuildSettings` 校验关键配置 |
| 提交钩子 | 使用 pre-commit hook 检查 pbxproj 格式一致性 |

### 8.4 常见问题速查表

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| "No such module" 编译错误 | Header Search Path 缺失 | 检查 `HEADER_SEARCH_PATHS` 配置 |
| Info.plist 变量未替换 | xcconfig 未关联 Configuration | 在项目 Info 页确认 xcconfig 绑定 |
| 多环境 App 同时安装冲突 | Bundle ID 相同 | 添加 `BUNDLE_ID_SUFFIX` 区分 |
| Archive 后推送不工作 | aps-environment 为 development | 确认 Prod entitlements 使用 production |
| 代码中 `#if STAGING` 不生效 | 编译条件未定义 | 检查 `SWIFT_ACTIVE_COMPILATION_CONDITIONS` |
| 合并冲突后项目打不开 | pbxproj 格式损坏 | 用 `git checkout -- <file>` 恢复或手动修复 |
| CocoaPods 签名失败 | Entitlements 路径被覆盖 | 在 Podfile 中添加 `post_install` 钩子修正 |

CocoaPods 签名修复示例：

```ruby
# Podfile
post_install do |installer|
  installer.pods_project.targets.each do |target|
    target.build_configurations.each do |config|
      config.build_settings['CODE_SIGN_IDENTITY'] = ''
      config.build_settings['EXPANDED_CODE_SIGN_IDENTITY'] = ''
    end
  end
end
```

---

## 本章小结

| 主题 | 核心要点 |
|------|---------|
| 项目结构 | .xcodeproj 是目录包，pbxproj 是核心配置文件，.xcworkspace 用于多项目组织 |
| Target 与 Scheme | Target 定义构建产物，Scheme 定义运行方式，Shared Scheme 是团队协作基础 |
| Build Configurations | Debug/Release 是默认配置，可自定义 Staging 等，Scheme 中为每个 Action 指定 Configuration |
| xcconfig 实战 | 纯文本配置文件，支持 `#include` 继承和 `$(inherited)` 追加，是环境管理的最佳工具 |
| 编译选项 | Optimization Level 影响性能与包体积，Swift Flags 控制条件编译，Linker Flags 处理 OC 兼容 |
| Capabilities | .entitlements 声明系统权限，需与 Developer Portal 同步开启，多环境可使用不同 entitlements |
| 多环境实战 | 通过 xcconfig + Info.plist 变量替换实现 API URL、Bundle ID、App Icon 的环境切换 |
| 最佳实践 | 用 xcconfig 替代 UI 配置、忽略用户数据、统一团队配置、排查 $(inherited) 缺失问题 |

← [项目搭建与架构](./项目搭建与架构.md) | [依赖管理与开源库](./依赖管理与开源库.md) →
