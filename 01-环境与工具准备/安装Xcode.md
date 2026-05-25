# 软件准备：安装 Xcode

> 🎯 **本章目标**：从零开始，完成 Xcode 的安装与配置，让你的 Mac 变成一台 iOS 开发机器。读完本章后，你将拥有一个可以运行、调试 iOS 应用的完整开发环境。

---

## macOS 系统要求与升级

Xcode 是 Apple 官方的集成开发环境（IDE），它对 macOS 版本有严格要求。版本不匹配的话，连安装都无法进行。

### Xcode 与 macOS 版本对应关系

| Xcode 版本 | 最低 macOS 要求 | 推荐 macOS |
|:---:|:---:|:---:|
| Xcode 16 | macOS 15 Sequoia | macOS 15 Sequoia |
| Xcode 15 | macOS 14 Sonoma | macOS 14 Sonoma |
| Xcode 14 | macOS 13 Ventura | macOS 13 Ventura |
| Xcode 13 | macOS 12 Monterey | macOS 12 Monterey |

> 💡 **提示**：建议始终使用最新版 Xcode，因为 App Store Connect 上架新 App 时，通常要求使用最新版 Xcode 构建。截至 2026 年，Xcode 16 是主流版本，需要 macOS 15 Sequoia。

### 如何检查当前 macOS 版本

1. 点击屏幕左上角的  **Apple 图标**
2. 选择 **关于本机（About This Mac）**
3. 在弹出的窗口中即可看到 macOS 版本号

<!-- 📸 截图位置：关于本机窗口，标注版本号位置 -->

或者用命令行查看：

```bash
sw_vers
```

输出示例：

```
ProductName:            macOS
ProductVersion:         15.3.1
BuildVersion:           24D70
```

### 如何升级 macOS

1. 点击  **Apple 图标 → 系统设置（System Settings）**
2. 点击左侧 **通用（General） → 软件更新（Software Update）**
3. 等待系统检查更新，点击 **立即更新（Update Now）**

<!-- 📸 截图位置：系统设置中的软件更新界面 -->

升级过程大约需要 30 分钟到 1 小时，期间 Mac 会重启数次。

> ⚠️ **警告**：升级前务必备份！详见下方备份建议。

### 升级前的备份建议

| 备份方式 | 操作路径 | 适合场景 |
|:---|:---|:---|
| **Time Machine** | 系统设置 → 通用 → Time Machine | 全量备份，可恢复整个系统 |
| **iCloud 云盘** | 系统设置 → Apple ID → iCloud | 备份桌面和文档文件夹 |
| **手动拷贝** | 将重要文件复制到外接硬盘 | 只需备份少量关键文件 |

> 💡 **提示**：强烈推荐使用 Time Machine + 外接硬盘做一次完整备份，这是最稳妥的方式。一块 1TB 的移动硬盘即可，价格在 200 元左右。

---

## Apple ID 注册

### 为什么需要 Apple ID

Apple ID 在 iOS 开发中扮演多个关键角色：

- **Xcode 登录**：签名和运行 App 到真机必须登录 Apple ID
- **App Store Connect**：管理 App 的上架、定价、分析数据
- **Apple Developer Program**：加入开发者计划（上架必备）需要 Apple ID
- **TestFlight**：内测分发 App 给测试用户

> 💡 **提示**：即使暂时不付费加入开发者计划，免费的 Apple ID 也能让你在真机上运行自己写的 App（有 7 天签名限制）。

### 注册步骤（网页注册）

1. 打开 Apple ID 注册页面：[https://appleid.apple.com](https://appleid.apple.com)
2. 点击右上角 **创建你的 Apple ID**
3. 按照页面提示填写：

<!-- 📸 截图位置：Apple ID 注册页面 -->

| 填写项 | 说明 |
|:---|:---|
| 姓名和出生日期 | 建议使用真实信息 |
| 国家/地区 | 选择 **中国**（国内支付更方便） |
| 邮箱 | 此邮箱就是你的 Apple ID，建议使用常用邮箱 |
| 密码 | 至少 8 位，包含大小写字母和数字 |
| 手机号 | 用于双重认证，必须能正常接收短信 |

4. 填写验证码完成邮箱和手机验证
5. 注册完成！

### 双重认证设置

双重认证（Two-Factor Authentication）是 Apple 强制要求的安全机制，在新设备登录时会要求输入验证码。

**开启方式**：

1. 打开 **系统设置 → Apple ID → 登录与安全性**
2. 点击 **双重认证**
3. 如果显示"已开启"，则无需操作；如果显示"未开启"，按提示开启

<!-- 📸 截图位置：双重认证设置界面 -->

> ⚠️ **警告**：务必记住你的受信任设备号！如果更换手机号，请提前更新 Apple ID 绑定的手机号，否则可能无法登录。

### 中国区 Apple ID 注意事项

- **支付方式**：中国区支持支付宝、微信支付，绑定更方便
- **App 内容合规**：中国区 App Store 有内容审核差异，但对你开发 App 不影响
- **开发者计划付款**：加入 Apple Developer Program（99 美元/年）需要使用支持国际支付的信用卡或支付宝
- **不建议频繁切换区**：切换 Apple ID 区域可能导致已下载 App 无法更新

> 💡 **提示**：如果你已有中国区 Apple ID，直接使用即可，无需重新注册。

---

## Xcode 下载安装

### 从 App Store 下载（推荐）

这是最简单、最推荐的方式，适合绝大多数开发者。

**操作步骤**：

1. 打开 **App Store**（Dock 栏或 Launchpad 中可以找到）
2. 在搜索框中输入 **Xcode**，按回车搜索

<!-- 📸 截图位置：App Store 搜索 Xcode 的界面 -->

3. 找到第一个结果（开发者显示为 Apple），点击 **获取（Get）** 或 **安装（Install）**

<!-- 📸 截图位置：Xcode 详情页，标注安装按钮 -->

4. 输入 Apple ID 密码或使用 Touch ID / Face ID 确认
5. 等待下载完成

**下载须知**：

| 项目 | 详情 |
|:---|:---|
| 下载大小 | 约 12 GB（压缩包） |
| 安装后占用 | 约 35 GB（含模拟器等组件后可达 50 GB+） |
| 下载时间 | 取决于网速，通常 30 分钟 ~ 数小时 |
| 存储位置 | 自动安装到 `/Applications/Xcode.app` |

> ⚠️ **警告**：确保 Mac 至少有 **50 GB 可用空间**再开始安装，否则可能中途失败。可以在"关于本机 → 储存空间"中查看剩余空间。

> 💡 **提示**：下载期间可以继续做其他事情，App Store 会在后台下载。下载完成后会自动安装。

### 从 Apple Developer 网站下载

某些情况下，你需要从 Apple Developer 官网下载 Xcode：

| 场景 | 说明 |
|:---|:---|
| 需要 **Beta 版** Xcode | 测试新版 iOS API |
| 需要 **旧版** Xcode | 维护老项目兼容性 |
| App Store 下载失败 | 网络问题等 |
| 需要多个版本共存 | 同时安装 Xcode 15 和 16 |

**操作步骤**：

1. 打开 Apple Developer 下载页面：[https://developer.apple.com/download/all/](https://developer.apple.com/download/all/)
2. 使用 Apple ID 登录
3. 在搜索框中输入 **Xcode**，筛选需要的版本

<!-- 📸 截图位置：Apple Developer 下载页面 -->

4. 点击对应版本旁的 **下载** 按钮，下载 `.xip` 格式文件

**解压 .xip 文件**：

```bash
# 进入下载目录
cd ~/Downloads

# 解压 Xcode（将文件名替换为你下载的实际文件名）
xip -x Xcode_16.xip
```

解压后会得到一个 `Xcode.app` 文件，将它移动到应用程序文件夹：

```bash
sudo mv Xcode.app /Applications/
```

> ⚠️ **警告**：如果你已经通过 App Store 安装了 Xcode，官网下载的版本会覆盖它。如需多版本共存，请将新版重命名，例如 `Xcode-16-Beta.app`。

**多版本共存时设置默认 Xcode**：

```bash
# 查看当前默认 Xcode 路径
xcode-select -p

# 切换默认 Xcode（替换为你的实际路径）
sudo xcode-select -s /Applications/Xcode-16-Beta.app/Contents/Developer
```

---

## 命令行工具安装

### 什么是命令行工具

Command Line Tools（命令行工具）是 Xcode 附带的一组开发工具，包括：

- **编译器**：clang、swiftc（编译 C 和 Swift 代码）
- **构建工具**：make、cmake
- **版本控制**：git
- **SDK 头文件**：开发所需的系统库头文件
- **调试工具**：lldb

即使你不使用 Xcode 的图形界面，很多第三方工具（如 CocoaPods、Fastlane）也依赖这些命令行工具。

> 💡 **提示**：从 App Store 安装 Xcode 后，命令行工具通常会自动安装。但为了确保万无一失，建议手动执行一次安装命令。

### 安装命令

打开 **终端（Terminal）**，输入以下命令：

```bash
xcode-select --install
```

执行后会弹出一个安装确认对话框：

<!-- 📸 截图位置：命令行工具安装确认对话框 -->

1. 点击 **安装（Install）**
2. 同意许可协议
3. 等待下载和安装完成（约 1-3 分钟）

### 验证安装

安装完成后，在终端中逐条运行以下命令来验证：

```bash
# 检查 Xcode 命令行工具路径
xcode-select -p
```

预期输出：

```
/Applications/Xcode.app/Contents/Developer
```

```bash
# 检查 git 是否可用
git --version
```

预期输出（版本号可能不同）：

```
git version 2.39.5 (Apple Git-158)
```

```bash
# 检查 Swift 编译器是否可用
swift --version
```

预期输出：

```
swift-driver version: 1.115 Apple Swift version 6.0.3 (swiftlang-6.0.3.1.10 clang-1600.0.30.1)
Target: arm64-apple-macosx15.0
```

> ⚠️ **警告**：如果 `xcode-select -p` 返回的路径不是 `/Applications/Xcode.app/Contents/Developer`，说明默认开发者目录指向有误，请使用以下命令修复：

```bash
sudo xcode-select -s /Applications/Xcode.app/Contents/Developer
```

---

## Xcode 首次启动与配置

### 同意许可协议

首次打开 Xcode 时，会弹出许可协议对话框：

1. 在 Launchpad 或 `/Applications` 文件夹中找到 **Xcode**，双击打开
2. 阅读许可协议，点击 **Agree（同意）**
3. 输入 macOS 管理员密码确认

<!-- 📸 截图位置：Xcode 许可协议对话框 -->

> 💡 **提示**：也可以通过命令行同意许可协议（适合远程操作）：

```bash
sudo xcodebuild -license accept
```

### 安装额外组件

同意协议后，Xcode 会请求安装额外的系统组件：

1. 弹出安装提示，点击 **Install（安装）**
2. 输入管理员密码
3. 等待安装完成

<!-- 📸 截图位置：额外组件安装对话框 -->

这些组件包括模拟器运行时、设备调试支持包等，是 Xcode 正常工作所必需的。

### 首次启动设置

安装完额外组件后，Xcode 会显示欢迎界面（Welcome to Xcode）：

<!-- 📸 截图位置：Xcode 欢迎界面 -->

**推荐配置**：

1. 打开 **Xcode → Settings（设置）**（快捷键 `⌘,`）
2. **General（通用）** 标签页：
   - ✅ 勾选 **Show live issues**（实时显示代码问题）
   - ✅ 勾选 **Show inline issues**（行内显示问题）
3. **Accounts（账户）** 标签页：
   - 点击左下角 **+** → 选择 **Apple ID** → 登录你的 Apple ID
   - 登录后即可在真机上运行 App

<!-- 📸 截图位置：Xcode Settings 的 Accounts 界面，标注已登录的 Apple ID -->

### 界面简介

Xcode 主界面由以下几个核心区域组成：

```
┌──────────────────────────────────────────────────┐
│  工具栏（Toolbar）                                  │
├──────┬───────────────────────────┬───────────────┤
│      │                           │               │
│ 导航  │      编辑区（Editor）       │  检查器       │
│ 区域  │                           │ (Inspector)   │
│(Nav) │                           │               │
│      │                           │               │
├──────┴───────────────────────────┴───────────────┤
│         调试区域（Debug Area）                      │
└──────────────────────────────────────────────────┘
```

| 区域 | 功能 | 快捷键切换 |
|:---|:---|:---|
| **导航区** | 浏览项目文件、搜索、断点管理等 | `⌘0` 显示/隐藏 |
| **编辑区** | 编写代码、设计界面 | — |
| **检查器** | 查看和修改选中元素的属性 | `⌥⌘0` 显示/隐藏 |
| **调试区** | 运行日志、变量查看、内存分析 | `⇧⌘Y` 显示/隐藏 |
| **工具栏** | 运行/停止按钮、模拟器选择 | — |

---

## 模拟器设置

模拟器（Simulator）让你无需真机就能在 Mac 上运行和测试 iOS App，是开发过程中最常用的工具之一。

### 下载 iOS 模拟器运行时

Xcode 默认只安装最新版 iOS 模拟器，如需测试其他版本需要手动下载：

1. 打开 **Xcode → Settings（设置）**（`⌘,`）
2. 点击 **Platforms（平台）** 标签页

<!-- 📸 截图位置：Xcode Settings 的 Platforms 界面 -->

3. 点击左下角 **+** 按钮
4. 选择需要的 iOS 版本（如 iOS 17 Simulator）
5. 点击 **Download & Install**

> 💡 **提示**：每个模拟器运行时约 3-5 GB，建议只下载你需要测试的版本。通常保留当前最新版和上一个版本即可。

### 创建模拟器

1. 打开模拟器管理器：**Xcode → Settings → Platforms** 或菜单 **Window → Devices and Simulators**（`⇧⌘2`）

<!-- 📸 截图位置：Devices and Simulators 界面 -->

2. 切换到 **Simulators** 标签页
3. 点击左下角 **+** 按钮
4. 配置模拟器：

| 设置项 | 说明 | 示例 |
|:---|:---|:---|
| 模拟器名称 | 自定义名称 | "iPhone 16 Pro - iOS 18" |
| 设备类型 | 选择 iPhone/iPad 型号 | iPhone 16 Pro |
| iOS 版本 | 选择运行时版本 | iOS 18.0 |

5. 点击 **Create**

### 模拟器基本操作

| 操作 | 方法 |
|:---|:---|
| 启动模拟器 | Xcode 工具栏选择模拟器 → 点击 Run（`⌘R`） |
| 主屏幕 | `⇧⌘H` |
| 锁屏 | `⌃⌘L`（Control + Command + L） |
| 旋转屏幕 | `⌘←` / `⌘→` |
| 截图 | `⌘S` |
| 返回主屏幕 | 在模拟器窗口菜单：Hardware → Home |
| 模拟触摸 | 鼠标点击 = 手指触摸 |
| 模拟捏合缩放 | 按住 Option 键 + 拖动 |
| 输入文字 | 直接用 Mac 键盘输入 |
| 复制粘贴 | `⌘C` / `⌘V`（模拟器内） |

### 常用快捷键

| 快捷键 | 功能 |
|:---|:---|
| `⌘R` | 运行 App |
| `⌘.` | 停止运行 |
| `⇧⌘K` | 清理构建缓存 |
| `⌘B` | 仅编译（不运行） |
| `⌃⌘⇧C` | 模拟器截屏到剪贴板 |
| `⌘1~6` | 切换导航区面板 |
| `⌥⌘1~6` | 切换检查器面板 |

> 💡 **提示**：模拟器性能不如真机，尤其是动画和 GPU 相关功能。最终上线前务必在真机上测试！

---

## 常见安装问题排查

### 下载速度慢

App Store 下载 Xcode 可能非常慢，尤其是在国内网络环境下。

**解决方案**：

| 方案 | 操作 | 效果 |
|:---|:---|:---|
| **更换 DNS** | 系统设置 → 网络 → Wi-Fi → 详细信息 → DNS → 改为 `114.114.114.114` 或 `8.8.8.8` | ⭐⭐⭐ |
| **使用代理** | 配置系统代理或使用代理工具 | ⭐⭐⭐⭐⭐ |
| **从官网下载** | 使用浏览器 + 下载工具从 Developer 网站下载 `.xip` 文件 | ⭐⭐⭐⭐ |
| **使用 aria2c 多线程下载** | `brew install aria2` 后用 aria2c 下载 `.xip` 链接 | ⭐⭐⭐⭐ |

> 💡 **提示**：从 Apple Developer 网站下载 `.xip` 文件通常比 App Store 更稳定，且支持断点续传。

### 磁盘空间不足

Xcode 及其相关组件非常占空间，以下是清理建议：

```bash
# 查看当前 Xcode 相关占用
du -sh ~/Library/Developer/*
```

**清理项目**：

| 清理目标 | 路径 | 说明 |
|:---|:---|:---|
| 模拟器运行时 | `~/Library/Developer/CoreSimulator` | 删除不用的 iOS 版本 |
| 派生数据 | `~/Library/Developer/Xcode/DerivedData` | 可安全删除，会自动重建 |
| 旧版设备支持 | `~/Library/Developer/Xcode/iOS DeviceSupport` | 删除旧版 iOS 的调试支持 |
| 归档 | `~/Library/Developer/Xcode/Archives` | 已发布的旧归档可删除 |

**在 Xcode 中清理派生数据**：

1. 菜单 **Xcode → Settings → Locations**
2. 点击 Derived Data 路径旁的 **箭头**，在 Finder 中打开
3. 选中所有文件夹，删除

> ⚠️ **警告**：删除前请确认没有正在运行的项目。删除 DerivedData 会导致下次打开项目时重新索引，可能需要几分钟。

### 安装卡住

**症状**：App Store 显示"正在安装"但进度不动。

**解决方案**：

1. **强制退出 App Store** 后重新打开
   - `⌘Q` 退出 App Store → 重新打开 → 查看是否继续安装

2. **重启 Mac**
   - 有时系统缓存导致卡住，重启可以解决

3. **取消并重新下载**
   - 在 Launchpad 中找到 Xcode → 长按（或 Option + 点击）→ 点击删除
   - 重新到 App Store 下载

4. **检查系统日志**
   ```bash
   # 查看 App Store 安装日志
   log show --predicate 'subsystem == "com.apple.AppStore"' --last 30m
   ```

### Xcode 版本冲突

**症状**：安装了多个版本 Xcode 后，命令行工具指向错误版本。

**解决方案**：

```bash
# 查看当前默认 Xcode
xcode-select -p

# 切换到指定版本
sudo xcode-select -s /Applications/Xcode.app/Contents/Developer

# 验证切换成功
xcodebuild -version
```

**多版本管理建议**：

| 做法 | 说明 |
|:---|:---|
| 重命名 App | 如 `Xcode-16.app`、`Xcode-15.app`，避免覆盖 |
| 使用 xcodes 工具 | `brew install xcodes`，命令行管理多版本 |
| 使用 xcode-version 工具 | 轻量级版本切换工具 |

```bash
# 使用 xcodes 安装和切换版本
xcodes install "16.0"
xcodes select "16.0"
```

---

## 小结

🎉 恭喜！到这里，你的 Mac 已经具备了完整的 iOS 开发环境。让我们回顾一下本章的要点：

| 步骤 | 关键操作 | 验证方式 |
|:---|:---|:---|
| macOS 升级 | 升级到 macOS 15 Sequoia+ | `sw_vers` |
| Apple ID | 注册并开启双重认证 | Xcode Settings → Accounts |
| Xcode 安装 | App Store 下载或官网下载 | 打开 Xcode 能正常启动 |
| 命令行工具 | `xcode-select --install` | `xcode-select -p` 返回正确路径 |
| 首次配置 | 同意协议 + 安装组件 + 登录 Apple ID | 真机能识别 |
| 模拟器 | 下载运行时 + 创建模拟器 | 能在模拟器中运行 App |

> 📌 **下一步**：环境准备就绪后，我们将正式创建第一个 iOS 项目！请继续阅读 [开发者账号注册](./开发者账号注册.md)。

← [硬件准备：你需要一台 Mac](./硬件准备.md) | [开发者账号注册](./开发者账号注册.md) →
