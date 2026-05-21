# 94-App Extension 全景

## 本章目标

- 理解 App Extension 的概念、生命周期与通信机制
- 掌握各类 Extension 的用途与适用场景
- 学会开发 Share Extension、Safari Web Extension、Notification Service/Content Extension
- 了解 Sticker Pack Extension 的创建方式
- 掌握 App Group 数据共享的三种方式
- 熟悉 Extension 审核注意事项与最佳实践

---

## 1. App Extension 概述

### 1.1 什么是 App Extension

> 💡 生活类比：App Extension 就像汽车上的"挂件"——导航仪、行车记录仪、车载充电器。它们不是汽车本身，但能让汽车在特定场景下拥有额外能力。每个挂件独立工作，但都依附于这辆车。

App Extension 是 iOS 8 引入的机制，允许你的 App 在系统其他位置提供功能——分享菜单、通知、键盘、Siri 等。Extension 不是独立 App，而是嵌入在宿主 App 中的一个独立二进制 Target。

```
┌─────────────────────────────────────────┐
│              宿主 App (Containing App)    │
│                                          │
│   ┌──────────────┐  ┌──────────────┐    │
│   │ Share Ext    │  │ Widget Ext   │    │
│   └──────────────┘  └──────────────┘    │
│   ┌──────────────┐  ┌──────────────┐    │
│   │ Notif Ext    │  │ Safari Ext   │    │
│   └──────────────┘  └──────────────┘    │
│                                          │
│   主 App 代码与资源                       │
└─────────────────────────────────────────┘
```

### 1.2 Extension 与主 App 的关系

| 关系维度 | 说明 |
|----------|------|
| 进程 | Extension 运行在**独立进程**中，与主 App 内存隔离 |
| 生命周期 | 由**宿主应用**（系统 App）控制，不由你的主 App 控制 |
| 安装 | 随主 App 一起安装，不能单独分发 |
| 代码共享 | 通过 Framework 或 App Group 共享代码和数据 |
| 体积 | 每个 Extension 都会增加 App 包体积 |

> ⚠️ Extension 和主 App **永远不在同一进程**中运行。即使主 App 正在前台，Extension 也可能在另一个进程中独立运行。

### 1.3 Extension 生命周期

```
用户触发 Extension
       │
       ▼
  系统启动 Extension 进程
       │
       ▼
  调用 Extension 的 viewDidLoad / viewDidAppear
       │
       ▼
  Extension 执行任务（展示 UI / 处理数据）
       │
       ▼
  用户完成操作 或 系统终止
       │
       ▼
  Extension 进程被销毁
```

关键要点：

- Extension 的生命周期**由宿主应用管理**，不由你的代码控制
- 系统可能在任何时刻终止 Extension（内存压力等）
- Extension 应该**快速响应**，避免耗时操作
- 每次启动都是全新的，不保留上次运行的状态

### 1.4 通信机制

Extension 与外界有两种通信方向：

| 通信方向 | 机制 | 说明 |
|----------|------|------|
| 宿主 App → Extension | `NSExtensionContext` | 宿主通过 context 传递数据给 Extension |
| Extension → 主 App | App Group / URL Scheme | 通过共享容器或打开主 App |
| Extension → 宿主 App | `NSExtensionContext.completeRequest` | 返回处理结果并关闭 Extension |

```swift
// Extension 接收宿主数据
override func viewDidLoad() {
    super.viewDidLoad()

    guard let item = extensionContext?.inputItems.first as? NSExtensionItem,
          let itemProvider = item.attachments?.first else { return }

    if itemProvider.hasItemConformingToTypeIdentifier(UTType.url.identifier) {
        itemProvider.loadItem(forTypeIdentifier: UTType.url.identifier) { url, error in
            guard let sharedURL = url as? URL else { return }
            print("收到分享链接：\(sharedURL)")
        }
    }
}

// Extension 返回结果给宿主
extensionContext?.completeRequest(returningItems: [], completionHandler: nil)
```

---

## 2. Extension 类型全览

### 2.1 类型总表

| Extension 类型 | 引入版本 | 用途 | 典型场景 |
|----------------|----------|------|----------|
| Share | iOS 8 | 分享内容到你的 App | 分享网页/图片到社交平台 |
| Today / Widget | iOS 8 / 14 | 桌面小组件 | 天气、待办、股票 |
| WatchKit App | watchOS 2 | Apple Watch 应用 | 健身、导航 |
| Custom Keyboard | iOS 8 | 自定义键盘 | 表情键盘、滑行输入 |
| Photo Editing | iOS 8 | 照片编辑扩展 | 滤镜、修图工具 |
| Document Provider | iOS 8 | 文件提供/打开 | 云盘 App 提供文件 |
| Safari Web Extension | iOS 15 | 浏览器扩展 | 广告拦截、密码管理 |
| Notification Service | iOS 10 | 修改远程推送内容 | 下载推送图片/视频 |
| Notification Content | iOS 10 | 自定义通知 UI | 富媒体通知界面 |
| Intents / Siri | iOS 10 | Siri 集成 | 语音指令操作 App |
| WidgetKit | iOS 14 | 现代小组件框架 | 主屏幕/锁屏小组件 |
| App Clip | iOS 14 | 轻量版体验 | 扫码即用 |
| Sticker Pack | iOS 10 | iMessage 贴纸 | 表情包 |
| iMessage App | iOS 10 | iMessage 内应用 | 群聊游戏、协作工具 |
| Driver | iOS 11 | 驾驶场景集成 | 导航、车载控制 |
| CallKit / Call Directory | iOS 10 | 来电识别/拦截 | 骚扰电话拦截 |
| Audio Unit | iOS 9 | 音频处理插件 | GarageBand 乐器 |
| Broadcast Upload | iOS 11 | 屏幕直播 | 游戏直播推流 |

### 2.2 如何选择

```
你的需求是什么？
│
├─ 分享内容 → Share Extension
├─ 桌面信息展示 → WidgetKit Extension
├─ 浏览器增强 → Safari Web Extension
├─ 推送增强 → Notification Service / Content Extension
├─ 键盘输入 → Custom Keyboard
├─ 照片编辑 → Photo Editing
├─ 文件访问 → Document Provider
├─ Siri 语音 → Intents Extension
├─ iMessage → Sticker Pack / iMessage App
├─ 来电识别 → Call Directory Extension
└─ 扫码即用 → App Clip
```

> 💡 一个 App 可以包含**多个 Extension**，但每个 Extension 应该只做一件事，保持职责单一。

---

## 3. Share Extension

### 3.1 概述

> 💡 生活类比：Share Extension 就像快递柜——别人（系统分享菜单）把包裹（内容）投递到你的柜子里，你取出后按自己的方式处理。

Share Extension 让用户从任何 App 的分享菜单中，将内容发送到你的 App。

### 3.2 创建 Share Extension

1. Xcode → File → New → Target
2. 选择 **Share Extension**
3. 填写 Product Name（如 `MyShareExtension`）
4. 点击 Finish

Xcode 会自动生成 `ShareViewController.swift`，它继承自 `SLComposeServiceViewController`。

### 3.3 数据提取

分享的核心是从 `NSExtensionContext` 中提取数据：

```swift
import UIKit
import Social
import UniformTypeIdentifiers

class ShareViewController: SLComposeServiceViewController {

    override func viewDidLoad() {
        super.viewDidLoad()
        extractSharedContent()
    }

    private func extractSharedContent() {
        guard let extensionItem = extensionContext?.inputItems.first
                as? NSExtensionItem,
              let itemProvider = extensionItem.attachments?.first else {
            return
        }

        // 按优先级尝试提取不同类型
        if itemProvider.hasItemConformingToTypeIdentifier(UTType.url.identifier) {
            itemProvider.loadItem(forTypeIdentifier: UTType.url.identifier) { [weak self] item, error in
                guard let url = item as? URL else { return }
                self?.handleSharedURL(url)
            }
        } else if itemProvider.hasItemConformingToTypeIdentifier(UTType.image.identifier) {
            itemProvider.loadItem(forTypeIdentifier: UTType.image.identifier) { [weak self] item, error in
                guard let imageURL = item as? URL else { return }
                self?.handleSharedImage(url: imageURL)
            }
        } else if itemProvider.hasItemConformingToTypeIdentifier(UTType.text.identifier) {
            itemProvider.loadItem(forTypeIdentifier: UTType.text.identifier) { [weak self] item, error in
                guard let text = item as? String else { return }
                self?.handleSharedText(text)
            }
        }
    }

    private func handleSharedURL(_ url: URL) {
        saveToAppGroup(content: url.absoluteString)
    }

    private func handleSharedImage(url: URL) {
        guard let data = try? Data(contentsOf: url) else { return }
        let fileName = "shared_\(Date().timeIntervalSince1970).jpg"
        let sharedDir = FileManager.default.containerURL(
            forSecurityApplicationGroupIdentifier: "group.com.example.myapp"
        )
        guard let dest = sharedDir?.appendingPathComponent(fileName) else { return }
        try? data.write(to: dest)
    }

    private func handleSharedText(_ text: String) {
        saveToAppGroup(content: text)
    }

    private func saveToAppGroup(content: String) {
        let defaults = UserDefaults(suiteName: "group.com.example.myapp")
        defaults?.set(content, forKey: "sharedContent")
    }

    override func didSelectPost() {
        extensionContext?.completeRequest(returningItems: [], completionHandler: nil)
    }
}
```

### 3.4 配置 Info.plist

Share Extension 通过 `NSExtension` 字典声明支持的类型：

```xml
<key>NSExtension</key>
<dict>
    <key>NSExtensionAttributes</key>
    <dict>
        <key>NSExtensionActivationRule</key>
        <dict>
            <key>NSExtensionActivationSupportsWebURLWithMaxCount</key>
            <integer>1</integer>
            <key>NSExtensionActivationSupportsImageWithMaxCount</key>
            <integer>10</integer>
            <key>NSExtensionActivationSupportsText</key>
            <true/>
        </dict>
    </dict>
    <key>NSExtensionMainStoryboard</key>
    <string>MainInterface</string>
    <key>NSExtensionPointIdentifier</key>
    <string>com.apple.share-services</string>
</dict>
```

| 激活规则键 | 含义 |
|------------|------|
| `NSExtensionActivationSupportsWebURLWithMaxCount` | 支持网页链接的最大数量 |
| `NSExtensionActivationSupportsImageWithMaxCount` | 支持图片的最大数量 |
| `NSExtensionActivationSupportsText` | 是否支持纯文本 |
| `NSExtensionActivationSupportsMovieWithMaxCount` | 支持视频的最大数量 |
| `NSExtensionActivationSupportsFileWithMaxCount` | 支持文件的最大数量 |

> ⚠️ 激活规则决定了你的 Extension 何时出现在分享菜单中。如果规则太严格，可能不会显示；太宽松则会在不合适的场景出现。

---

## 4. Safari Web Extension

### 4.1 概述

> 💡 生活类比：Safari Web Extension 就像给浏览器装上"外挂工具"——翻译插件帮你读外语，广告拦截器帮你挡广告，密码管理器帮你填密码。

Safari Web Extension（iOS 15+）使用 Web 技术开发，可以修改网页内容、添加工具栏按钮、管理隐私权限。

### 4.2 架构组成

```
Safari Web Extension
├── content.js          ← 注入到网页的脚本
├── background.js       ← 后台常驻脚本
├── popup.html / .js    ← 点击工具栏按钮弹出的面板
├── manifest.json       ← 权限与配置声明
└── resources/          ← 图标等资源
```

### 4.3 创建 Safari Web Extension

1. Xcode → File → New → Target
2. 选择 **Safari Extension App** 或 **Safari Extension**
3. Xcode 自动生成 Web 资源目录和原生容器

### 4.4 manifest.json 配置

```json
{
    "manifest_version": 3,
    "name": "My Safari Extension",
    "description": "一个示例 Safari 扩展",
    "version": "1.0",
    "permissions": [
        "activeTab",
        "scripting",
        "storage"
    ],
    "background": {
        "scripts": ["background.js"],
        "persistent": false
    },
    "content_scripts": [
        {
            "matches": ["<all_urls>"],
            "js": ["content.js"],
            "css": ["style.css"]
        }
    ],
    "action": {
        "default_popup": "popup.html",
        "default_icon": {
            "16": "icon-16.png",
            "48": "icon-48.png",
            "128": "icon-128.png"
        }
    },
    "icons": {
        "48": "icon-48.png",
        "128": "icon-128.png"
    }
}
```

### 4.5 Content Script

Content Script 注入到用户访问的网页中，可以读取和修改 DOM：

```javascript
// content.js - 高亮页面中的关键词
const keywords = ["重要", "紧急", "注意"];

function highlightKeywords() {
    const body = document.body;
    keywords.forEach(keyword => {
        const walker = document.createTreeWalker(
            body,
            NodeFilter.SHOW_TEXT,
            null,
            false
        );

        const nodesToReplace = [];
        while (walker.nextNode()) {
            if (walker.currentNode.textContent.includes(keyword)) {
                nodesToReplace.push(walker.currentNode);
            }
        }

        nodesToReplace.forEach(node => {
            const span = document.createElement("span");
            span.innerHTML = node.textContent.replace(
                new RegExp(keyword, "g"),
                `<mark style="background:#ffeb3b;padding:2px 4px;border-radius:3px">${keyword}</mark>`
            );
            node.parentNode.replaceChild(span, node);
        });
    });
}

highlightKeywords();
```

### 4.6 Popup 页面

```html
<!-- popup.html -->
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <style>
        body {
            width: 300px;
            padding: 16px;
            font-family: -apple-system, sans-serif;
        }
        .toggle { display: flex; justify-content: space-between; align-items: center; padding: 8px 0; }
        button { background: #007AFF; color: white; border: none; padding: 8px 16px; border-radius: 8px; cursor: pointer; }
    </style>
</head>
<body>
    <h3>关键词高亮</h3>
    <div class="toggle">
        <span>启用高亮</span>
        <input type="checkbox" id="enableToggle" checked>
    </div>
    <button id="applyBtn">应用到当前页面</button>
    <script src="popup.js"></script>
</body>
</html>
```

```javascript
// popup.js
document.getElementById("applyBtn").addEventListener("click", async () => {
    const enabled = document.getElementById("enableToggle").checked;
    const [tab] = await browser.tabs.query({ active: true, currentWindow: true });
    if (tab.id) {
        browser.tabs.sendMessage(tab.id, { action: enabled ? "highlight" : "clear" });
    }
});
```

### 4.7 权限声明

| 权限 | 说明 | 风险等级 |
|------|------|----------|
| `activeTab` | 访问当前标签页 | 低 |
| `scripting` | 在页面中执行脚本 | 中 |
| `storage` | 本地存储 | 低 |
| `tabs` | 访问标签页信息 | 中 |
| `<all_urls>` | 匹配所有网页 | 高 |
| `webRequest` | 拦截网络请求 | 高 |

> ⚠️ 权限声明越少越好。App Store 审核会检查权限合理性，过度申请权限可能导致被拒。

---

## 5. Notification Service Extension

### 5.1 概述

> 💡 生活类比：Notification Service Extension 就像快递分拣员——推送通知是包裹，分拣员在包裹送达前可以"拆开看看"，往里面塞一张图片或修改地址标签，然后再投递。

Notification Service Extension 允许你在远程推送**展示给用户之前**修改其内容，最常用的场景是**下载推送中的图片/视频附件**。

### 5.2 创建

1. Xcode → File → New → Target
2. 选择 **Notification Service Extension**
3. 填写 Product Name（如 `MyNotificationService`）

### 5.3 核心代码

```swift
import UserNotifications

class NotificationService: UNNotificationServiceExtension {

    var contentHandler: ((UNNotificationContent) -> Void)?
    var bestAttemptContent: UNMutableNotificationContent?

    override func didReceive(_ request: UNNotificationRequest,
                             withContentHandler contentHandler: @escaping (UNNotificationContent) -> Void) {
        self.contentHandler = contentHandler
        bestAttemptContent = (request.content.mutableCopy() as? UNMutableNotificationContent)

        guard let content = bestAttemptContent else { return }

        // 修改标题
        content.title = "📬 \(content.title)"

        // 下载图片附件
        if let imageURLString = content.userInfo["image-url"] as? String,
           let imageURL = URL(string: imageURLString) {
            downloadAndAttachImage(url: imageURL, to: content)
        } else {
            contentHandler(content)
        }
    }

    private func downloadAndAttachImage(url: URL, to content: UNMutableNotificationContent) {
        let task = URLSession.shared.downloadTask(with: url) { [weak self] location, _, error in
            guard let self = self,
                  let location = location,
                  error == nil else {
                self?.contentHandler?(self?.bestAttemptContent ?? content)
                return
            }

            let tmpDir = NSTemporaryDirectory()
            let tmpFile = "attachment_\(url.lastPathComponent)"
            let tmpPath = tmpDir + tmpFile

            try? FileManager.default.moveItem(atPath: location.path, toPath: tmpPath)

            if let attachment = try? UNNotificationAttachment(
                identifier: "image",
                url: URL(fileURLWithPath: tmpPath),
                options: [UNNotificationAttachmentOptionsTypeHintKey: "public.jpeg"]
            ) {
                content.attachments = [attachment]
            }

            self.contentHandler?(content)
        }
        task.resume()
    }

    override func serviceExtensionTimeWillExpire() {
        if let contentHandler = contentHandler, let bestAttemptContent = bestAttemptContent {
            bestAttemptContent.title = "⏰ \(bestAttemptContent.title)"
            contentHandler(bestAttemptContent)
        }
    }
}
```

### 5.4 推送 Payload 格式

```json
{
    "aps": {
        "alert": {
            "title": "新消息",
            "body": "你收到一张图片"
        },
        "mutable-content": 1,
        "sound": "default"
    },
    "image-url": "https://example.com/photo.jpg"
}
```

> ⚠️ 必须设置 `"mutable-content": 1`，系统才会启动 Notification Service Extension。否则推送会直接展示，Extension 不会被调用。

### 5.5 关键时间线

| 阶段 | 时间限制 | 说明 |
|------|----------|------|
| `didReceive` 调用到 `contentHandler` 回调 | 约 30 秒 | 在此期间完成附件下载 |
| `serviceExtensionTimeWillExpire` | 30 秒到期前调用 | 必须在此回调中尽快返回内容 |

> 💡 如果下载超时，系统会调用 `serviceExtensionTimeWillExpire`，你应该在此方法中返回当前已有的最佳内容，否则通知可能不会显示。

---

## 6. Notification Content Extension

### 6.1 概述

> 💡 生活类比：Notification Content Extension 就像快递柜的"定制展示柜"——普通快递直接塞进标准格口，但贵重物品可以放在带灯光和旋转台的专属展柜里。

Notification Content Extension 允许你**自定义通知的展示 UI**，包括自定义布局、图片、视频、交互按钮等。

### 6.2 创建

1. Xcode → File → New → Target
2. 选择 **Notification Content Extension**
3. 填写 Product Name（如 `MyNotificationContent`）

### 6.3 自定义通知 UI

```swift
import UIKit
import UserNotifications
import UserNotificationsUI

class NotificationViewController: UIViewController, UNNotificationContentExtension {

    @IBOutlet weak var titleLabel: UILabel!
    @IBOutlet weak var bodyLabel: UILabel!
    @IBOutlet weak var imageView: UIImageView!

    override func viewDidLoad() {
        super.viewDidLoad()
        view.backgroundColor = .systemBackground
        setupLabels()
    }

    private func setupLabels() {
        titleLabel.font = .systemFont(ofSize: 18, weight: .bold)
        bodyLabel.font = .systemFont(ofSize: 15)
        bodyLabel.numberOfLines = 0
        imageView.contentMode = .scaleAspectFit
    }

    func didReceive(_ notification: UNNotification) {
        let content = notification.request.content

        titleLabel.text = content.title
        bodyLabel.text = content.body

        if let attachment = content.attachments.first {
            if attachment.url.startAccessingSecurityScopedResource() {
                if let data = try? Data(contentsOf: attachment.url),
                   let image = UIImage(data: data) {
                    imageView.image = image
                }
                attachment.url.stopAccessingSecurityScopedResource()
            }
        }
    }

    // 处理通知上的交互按钮
    func didReceive(_ response: UNNotificationResponse,
                    completionHandler completion: @escaping (UNNotificationContentExtensionResponseOption) -> Void) {
        switch response.actionIdentifier {
        case "REPLY_ACTION":
            if let textResponse = response as? UNTextInputNotificationResponse {
                let replyText = textResponse.userText
                sendReply(replyText)
            }
            completion(.dismiss)
        case "LIKE_ACTION":
            handleLike()
            completion(.dismiss)
        default:
            completion(.doNotDismiss)
        }
    }

    private func sendReply(_ text: String) {
        // 将回复发送到服务器
    }

    private func handleLike() {
        // 处理点赞
    }
}
```

### 6.4 配置 Info.plist

```xml
<key>NSExtension</key>
<dict>
    <key>NSExtensionAttributes</key>
    <dict>
        <key>UNNotificationExtensionCategory</key>
        <string>MESSAGE_CATEGORY</string>
        <key>UNNotificationExtensionDefaultContentHidden</key>
        <true/>
        <key>UNNotificationExtensionInitialContentSizeRatio</key>
        <real>1.5</real>
        <key>UNNotificationExtensionUserInteractionEnabled</key>
        <true/>
    </dict>
    <key>NSExtensionMainStoryboard</key>
    <string>MainInterface</string>
    <key>NSExtensionPointIdentifier</key>
    <string>com.apple.usernotifications.content-extension</string>
</dict>
```

| 配置键 | 说明 |
|--------|------|
| `UNNotificationExtensionCategory` | 对应推送 payload 中的 category 标识 |
| `UNNotificationExtensionDefaultContentHidden` | `true` 隐藏默认通知内容，只显示自定义 UI |
| `UNNotificationExtensionInitialContentSizeRatio` | 自定义 UI 的初始高度比例（相对于默认高度） |
| `UNNotificationExtensionUserInteractionEnabled` | 是否允许用户交互 |

### 6.5 交互按钮（Category）

在主 App 中注册通知 Category：

```swift
import UserNotifications

func registerNotificationCategories() {
    let replyAction = UNTextInputNotificationAction(
        identifier: "REPLY_ACTION",
        title: "回复",
        options: []
    )

    let likeAction = UNNotificationAction(
        identifier: "LIKE_ACTION",
        title: "点赞",
        options: []
    )

    let category = UNNotificationCategory(
        identifier: "MESSAGE_CATEGORY",
        actions: [replyAction, likeAction],
        intentIdentifiers: [],
        options: .customDismissAction
    )

    UNUserNotificationCenter.current().setNotificationCategories([category])
}
```

---

## 7. Sticker Pack Extension

### 7.1 概述

> 💡 生活类比：Sticker Pack 就像聊天时用的"贴纸本"——别人发文字，你甩一张表情贴纸，沟通更有趣。

Sticker Pack Extension 让你在 iMessage 中提供静态或动态贴纸，无需编写任何代码即可创建基础贴纸包。

### 7.2 创建方式

1. Xcode → File → New → Target
2. 选择 **Sticker Pack Extension**
3. 填写 Product Name（如 `MyStickerPack`）

Xcode 会生成一个 `Stickers.xcstickers` 文件夹，直接拖入图片即可。

### 7.3 贴纸规格

| 类型 | 格式 | 尺寸要求 | 大小限制 |
|------|------|----------|----------|
| 静态贴纸 | PNG / APNG / GIF / JPEG | 最小 300×300，建议 618×618 | 每张 < 500KB |
| 动态贴纸 | APNG / GIF | 同上 | 每张 < 500KB |

### 7.4 贴纸序列帧（动态贴纸）

在 `Stickers.xcstickers` 中：

1. 右键 → New Sticker Sequence
2. 拖入序列帧图片（帧按顺序命名）
3. 设置帧率（Frames Per Second）和循环方式

### 7.5 自定义贴纸（iMessage App）

如果需要更复杂的交互（如用户生成贴纸、购买贴纸），需要创建 **iMessage App** 而非简单 Sticker Pack：

```swift
import Messages

class MessagesViewController: MSMessagesAppViewController {

    override func viewDidLoad() {
        super.viewDidLoad()
        setupStickerBrowser()
    }

    private func setupStickerBrowser() {
        let layout = MSStickerBrowserViewLayout()
        let browser = MSStickerBrowserView(frame: view.bounds, stickerBrowserViewLayout: layout)
        browser.dataSource = self
        browser.stickerSize = .regular
        view.addSubview(browser)
        browser.translatesAutoresizingMaskIntoConstraints = false
        NSLayoutConstraint.activate([
            browser.topAnchor.constraint(equalTo: view.topAnchor),
            browser.bottomAnchor.constraint(equalTo: view.bottomAnchor),
            browser.leadingAnchor.constraint(equalTo: view.leadingAnchor),
            browser.trailingAnchor.constraint(equalTo: view.trailingAnchor)
        ])
    }
}

extension MessagesViewController: MSStickerBrowserViewDataSource {
    func numberOfStickers(in stickerBrowserView: MSStickerBrowserView) -> Int {
        return stickers.count
    }

    func stickerBrowserView(_ stickerBrowserView: MSStickerBrowserView,
                            stickerAt index: Int) -> MSSticker {
        return stickers[index]
    }
}
```

| 方式 | 编码需求 | 交互能力 | 适用场景 |
|------|----------|----------|----------|
| Sticker Pack Extension | 无 | 仅展示贴纸 | 简单表情包 |
| iMessage App | Swift | 完整交互 | 贴纸商店、用户生成内容 |

---

## 8. App Group 数据共享

### 8.1 为什么需要 App Group

> 💡 生活类比：App Group 就像两个房间之间的"共享储物柜"——主 App 和 Extension 各自住在不同房间（进程），但都可以打开中间的储物柜取放物品。

由于 Extension 和主 App 运行在不同进程中，它们**无法直接共享内存或文件**。App Group 提供了一个共享的沙盒区域，让两者可以交换数据。

### 8.2 配置 App Group

1. 打开 Apple Developer Portal → Certificates, Identifiers & Profiles
2. 在 App ID 和 Extension 的 App ID 中都启用 **App Groups**
3. 创建相同的 Group Identifier（如 `group.com.example.myapp`）
4. 在 Xcode 中：Target → Signing & Capabilities → + App Groups → 勾选对应 Group

> ⚠️ 主 App 和 Extension 必须配置**完全相同**的 Group Identifier，否则无法共享数据。

### 8.3 UserDefaults 共享

```swift
let suiteName = "group.com.example.myapp"

// 主 App 写入
let sharedDefaults = UserDefaults(suiteName: suiteName)
sharedDefaults?.set("Hello from Main App", forKey: "greeting")
sharedDefaults?.synchronize()

// Extension 读取
let sharedDefaults = UserDefaults(suiteName: suiteName)
let greeting = sharedDefaults?.string(forKey: "greeting")
print(greeting ?? "无数据")
```

| 操作 | 主 App | Extension |
|------|--------|-----------|
| 写入 | `sharedDefaults.set(value, forKey:)` | 同左 |
| 读取 | `sharedDefaults.string(forKey:)` | 同左 |
| 删除 | `sharedDefaults.removeObject(forKey:)` | 同左 |
| 同步 | `sharedDefaults.synchronize()` | 同左 |

> 💡 `UserDefaults(suiteName:)` 返回的是可选值，使用时注意解包。建议封装一个共享的数据管理类。

### 8.4 文件共享

```swift
let suiteName = "group.com.example.myapp"

// 获取共享容器目录
guard let sharedDir = FileManager.default.containerURL(
    forSecurityApplicationGroupIdentifier: suiteName
) else { return }

// 主 App 写入文件
let fileURL = sharedDir.appendingPathComponent("shared_data.json")
let data = try JSONEncoder().encode(myData)
try data.write(to: fileURL)

// Extension 读取文件
let readData = try Data(contentsOf: fileURL)
let decoded = try JSONDecoder().decode(MyData.self, from: readData)
```

文件共享适合大数据量场景（图片、数据库等）：

| 场景 | 推荐方式 | 说明 |
|------|----------|------|
| 少量键值对 | UserDefaults | 简单快速 |
| 结构化数据 | 文件共享 + JSON/Codable | 灵活可控 |
| 图片/媒体 | 文件共享 | Share Extension 传递图片 |
| 数据库 | 文件共享 + SQLite/SwiftData | 复杂查询场景 |

### 8.5 Keychain 共享

Keychain 共享需要使用 **Keychain Sharing** Capability，而非 App Group：

```swift
import Security

struct KeychainHelper {
    static let sharedAccessGroup = "TEAMID.com.example.myapp"

    static func save(key: String, data: Data) {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key,
            kSecAttrAccessGroup as String: sharedAccessGroup,
            kSecValueData as String: data
        ]

        SecItemDelete(query as CFDictionary)
        SecItemAdd(query as CFDictionary, nil)
    }

    static func load(key: String) -> Data? {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key,
            kSecAttrAccessGroup as String: sharedAccessGroup,
            kSecReturnData as String: true,
            kSecMatchLimit as String: kSecMatchLimitOne
        ]

        var result: AnyObject?
        SecItemCopyMatching(query as CFDictionary, &result)
        return result as? Data
    }
}
```

> ⚠️ Keychain Sharing 的 Access Group 格式为 `TEAMID.com.example.myapp`，其中 `TEAMID` 是你的开发者团队 ID，与 App Group 的格式不同。

### 8.6 三种共享方式对比

| 方式 | 适用数据 | 安全性 | 容量 | 配置 |
|------|----------|--------|------|------|
| UserDefaults (App Group) | 简单键值对 | 低（明文） | 小 | App Groups Capability |
| 文件共享 (App Group) | 任意文件 | 中（沙盒保护） | 大 | App Groups Capability |
| Keychain Sharing | 敏感数据（Token/密码） | 高（加密存储） | 极小 | Keychain Sharing Capability |

---

## 9. Extension 审核注意事项

### 9.1 独立审核

Extension 会作为 App 提交的一部分接受**独立审核**。审核员会单独测试每个 Extension 的功能。

| 审核维度 | 要求 |
|----------|------|
| 功能完整性 | Extension 必须提供完整、可用的功能 |
| 功能独立性 | 不能仅作为主 App 的启动入口 |
| 崩溃与性能 | 不能崩溃、卡顿或占用过多内存 |
| UI 一致性 | 界面风格应与系统一致，不能过于突兀 |

### 9.2 功能独立性

> ⚠️ Extension **不能只是打开主 App**。审核指南明确要求 Extension 必须提供独立的、有价值的功能。

```
❌ 错误做法：
Share Extension 点击后 → 直接 openURL 跳转主 App

✅ 正确做法：
Share Extension → 在 Extension 内完成内容预览/编辑 → 保存到 App Group → 主 App 下次打开时读取
```

### 9.3 隐私合规

| 隐私要点 | 说明 |
|----------|------|
| 数据收集声明 | Extension 收集的数据也必须在隐私政策中声明 |
| 权限说明 | 访问相册、相机等需要在 Info.plist 中提供用途说明 |
| 网络请求 | Extension 中的网络请求也受 App Transport Policy 约束 |
| 数据最小化 | 只收集必要的数据，不要"顺便"收集额外信息 |

### 9.4 各类 Extension 常见被拒原因

| Extension 类型 | 常见被拒原因 |
|----------------|-------------|
| Share Extension | 只跳转主 App，不处理分享内容 |
| Custom Keyboard | 未提供"完全访问"提示，或收集用户输入 |
| Notification Service | 修改通知内容后与原始意图不符 |
| Safari Web Extension | 权限过度申请，或注入恶意脚本 |
| Sticker Pack | 贴纸涉及侵权、色情或歧视内容 |
| Widget | 点击后跳转无关页面，或展示广告 |

### 9.5 最佳实践清单

- [ ] 每个 Extension 都有独立且完整的功能
- [ ] Extension UI 风格与系统协调
- [ ] 快速响应，避免耗时操作
- [ ] 正确处理内存警告和生命周期
- [ ] App Group 配置正确，数据共享正常
- [ ] 隐私政策涵盖 Extension 的数据收集
- [ ] Info.plist 权限说明完整
- [ ] 测试 Extension 在主 App 未安装时的行为（App Clip 场景）

---

## 本章小结

| 主题 | 核心要点 |
|------|----------|
| App Extension 概述 | Extension 是独立进程的插件，由宿主应用控制生命周期，通过 NSExtensionContext 通信 |
| Extension 类型 | iOS 提供十余种 Extension 类型，覆盖分享、通知、浏览器、键盘等场景 |
| Share Extension | 从系统分享菜单接收内容，通过 NSExtensionContext 提取数据，App Group 传递给主 App |
| Safari Web Extension | 基于 Web 技术开发，包含 content script / background / popup，需合理声明权限 |
| Notification Service | 在推送展示前修改内容，下载图片/视频附件，需设置 mutable-content: 1 |
| Notification Content | 自定义通知 UI 和交互按钮，通过 Category 注册交互动作 |
| Sticker Pack | 无代码创建 iMessage 贴纸，复杂交互需使用 iMessage App |
| App Group | 三种共享方式：UserDefaults（键值）、文件（大数据）、Keychain（敏感数据） |
| 审核注意事项 | Extension 独立审核，功能必须独立完整，隐私合规，避免仅作跳转入口 |
