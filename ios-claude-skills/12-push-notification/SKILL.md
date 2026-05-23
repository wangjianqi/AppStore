---
name: push-notification
description: 涉及推送通知、APNs、UNUserNotificationCenter、Background Tasks、Silent Push、Rich Notification、Notification Extension、后台刷新、后台下载的任务
---

# 推送通知与后台任务

## 通知权限

### 请求时机
- **禁止在 App 启动时立即请求通知权限**，应在用户触发相关操作后请求（如完成注册、关注内容后）
- 只请求一次，被拒绝后禁止反复弹窗，引导用户去系统设置开启

### 权限请求

```swift
UNUserNotificationCenter.current().requestAuthorization(options: [.alert, .badge, .sound]) { granted, error in
    DispatchQueue.main.async {
        if granted {
            UIApplication.shared.registerForRemoteNotifications()
        }
    }
}
```

### 权限检查与引导

```swift
func checkNotificationAuthorization() async -> Bool {
    let settings = await UNUserNotificationCenter.current().notificationSettings()
    if settings.authorizationStatus == .denied {
        showSettingsAlert(message: "请在系统设置中开启通知权限")
        return false
    }
    return settings.authorizationStatus == .authorized
}

func showSettingsAlert(message: String) {
    guard let url = URL(string: UIApplication.openSettingsURLString) else { return }
    if UIApplication.shared.canOpenURL(url) {
        UIApplication.shared.open(url)
    }
}
```

---

## APNs 注册与 Token 管理

### 注册

```swift
func application(_ application: UIApplication, didRegisterForRemoteNotificationsWithDeviceToken deviceToken: Data) {
    let token = deviceToken.map { String(format: "%02x", $0) }.joined()
    TokenStorage.shared.saveAPNsToken(token)
    Task { await sendTokenToBackend(token) }
}

func application(_ application: UIApplication, didFailToRegisterForRemoteNotificationsWithError error: Error) {
    Logger.error("APNs 注册失败: \(error)")
}
```

### Token 管理
- Token 每次安装和系统更新都可能变化，**每次启动都必须注册**
- Token 发送到后端后本地缓存，下次对比是否变化，变化则重新上报
- 模拟器无法获取 APNs Token，Debug 时需跳过或使用测试 Token

---

## 本地通知

### 基本发送

```swift
func scheduleLocalNotification(title: String, body: String, in seconds: TimeInterval) {
    let content = UNMutableNotificationContent()
    content.title = title
    content.body = body
    content.sound = .default
    content.badge = NSNumber(value: UIApplication.shared.applicationIconBadgeNumber + 1)

    let trigger = UNTimeIntervalNotificationTrigger(timeInterval: seconds, repeats: false)
    let request = UNNotificationRequest(identifier: UUID().uuidString, content: content, trigger: trigger)

    UNUserNotificationCenter.current().add(request)
}
```

### 规范
- Identifier 必须唯一（用 UUID），否则会覆盖同 ID 的通知
- 触发时间最小 1 秒，重复通知最小间隔 60 秒
- 地理围栏通知用 `UNLocationNotificationTrigger`
- 日历通知用 `UNCalendarNotificationTrigger`

---

## 远程通知

### Payload 格式

```json
{
    "aps": {
        "alert": {
            "title": "新消息",
            "body": "你收到了一条新消息"
        },
        "sound": "default",
        "badge": 1,
        "mutable-content": 1,
        "category": "MESSAGE_CATEGORY"
    },
    "custom_key": "custom_value"
}
```

### 前台展示

```swift
func userNotificationCenter(_ center: UNUserNotificationCenter,
    willPresent notification: UNNotification) async -> UNNotificationPresentationOptions {
    return [.banner, .sound, .badge]
}
```

### 点击处理

```swift
func userNotificationCenter(_ center: UNUserNotificationCenter,
    didReceive response: UNNotificationResponse) async {
    let userInfo = response.notification.request.content.userInfo
    guard let action = userInfo["action"] as? String else { return }

    switch action {
    case "open_chat":
        let chatId = userInfo["chat_id"] as? String
        navigateToChat(chatId)
    case "open_detail":
        let itemId = userInfo["item_id"] as? String
        navigateToDetail(itemId)
    default:
        break
    }
}
```

---

## Rich Notification（Notification Service Extension）

### 创建 Extension
- Xcode → File → New → Target → Notification Service Extension
- Extension 有独立的 Bundle Identifier：`com.app.NotificationService`
- Extension 有独立的 Provisioning Profile

### 实现

```swift
class NotificationService: UNNotificationServiceExtension {
    var contentHandler: ((UNNotificationContent) -> Void)?
    var bestAttemptContent: UNMutableNotificationContent?

    override func didReceive(_ request: UNNotificationRequest,
        withContentHandler contentHandler: @escaping (UNNotificationContent) -> Void) {
        self.contentHandler = contentHandler
        bestAttemptContent = (request.content.mutableCopy() as? UNMutableNotificationContent)

        guard let content = bestAttemptContent else { return }

        if let imageURLString = content.userInfo["image_url"] as? String,
           let imageURL = URL(string: imageURLString) {
            downloadAndAttach(imageURL, to: content)
        } else {
            contentHandler(content)
        }
    }

    private func downloadAndAttach(_ url: URL, to content: UNMutableNotificationContent) {
        let task = URLSession.shared.downloadTask(with: url) { location, _, _ in
            guard let location, let fileURL = self.moveFile(at: location) else {
                self.contentHandler?(content)
                return
            }
            if let attachment = try? UNNotificationAttachment(identifier: "image", url: fileURL) {
                content.attachments = [attachment]
            }
            self.contentHandler?(content)
        }
        task.resume()
    }

    override func serviceExtensionTimeWillExpire() {
        contentHandler?(bestAttemptContent ?? UNNotificationContent())
    }
}
```

### 规范
- Payload 必须包含 `"mutable-content": 1` 才会触发 Extension
- Extension 运行时间有限（约 30 秒），超时调用 `serviceExtensionTimeWillExpire`
- 图片限制：小于 10MB，格式 PNG/JPEG/GIF
- 视频限制：小于 50MB，格式 MPEG/MP4/MOV
- **Extension 和主 App 不共享内存和沙盒**，需要共享数据用 App Group

---

## 通知分类与交互操作

### 注册分类

```swift
let replyAction = UNTextInputNotificationAction(
    identifier: "REPLY_ACTION",
    title: "回复",
    options: []
)

let markReadAction = UNNotificationAction(
    identifier: "MARK_READ_ACTION",
    title: "标为已读",
    options: []
)

let category = UNNotificationCategory(
    identifier: "MESSAGE_CATEGORY",
    actions: [replyAction, markReadAction],
    intentIdentifiers: [],
    options: .customDismissAction
)

UNUserNotificationCenter.current().setNotificationCategories([category])
```

### 处理操作

```swift
func userNotificationCenter(_ center: UNUserNotificationCenter,
    didReceive response: UNNotificationResponse) async {
    let action = response.actionIdentifier

    switch action {
    case "REPLY_ACTION":
        if let textResponse = response as? UNTextInputNotificationResponse {
            let replyText = textResponse.userText
            await sendReply(replyText)
        }
    case "MARK_READ_ACTION":
        await markAsRead()
    case UNNotificationDefaultActionIdentifier:
        handleDefaultTap(userInfo: response.notification.request.content.userInfo)
    default:
        break
    }
}
```

---

## Silent Push（后台静默推送）

### Payload

```json
{
    "aps": {
        "content-available": 1
    },
    "sync_type": "messages"
}
```

### 规范
- `content-available: 1` 触发后台唤醒，不显示通知
- 系统不保证每次都唤醒（受电量、频率等因素影响）
- 唤醒后约 30 秒执行时间，必须调用 `fetchCompletionHandler`
- **禁止用 Silent Push 做实时通讯**，仅用于数据同步
- 发送频率限制：每小时不超过几次，否则系统可能限流

```swift
func application(_ application: UIApplication,
    didReceiveRemoteNotification userInfo: [AnyHashable: Any],
    fetchCompletionHandler completionHandler: @escaping (UIBackgroundFetchResult) -> Void) {
    Task {
        do {
            try await syncData()
            completionHandler(.newData)
        } catch {
            completionHandler(.failed)
        }
    }
}
```

---

## Background Tasks

### 注册

```swift
func application(_ application: UIApplication,
    didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
    BGTaskScheduler.shared.register(forTaskWithIdentifier: "com.app.refresh", using: nil) { task in
        self.handleAppRefresh(task: task as! BGAppRefreshTask)
    }
    BGTaskScheduler.shared.register(forTaskWithIdentifier: "com.app.processing", using: nil) { task in
        self.handleProcessing(task: task as! BGProcessingTask)
    }
    scheduleAppRefresh()
    return true
}
```

### BGAppRefreshTask（后台刷新）

```swift
func scheduleAppRefresh() {
    let request = BGAppRefreshTaskRequest(identifier: "com.app.refresh")
    request.earliestBeginDate = Date(timeIntervalSinceNow: 3600)
    do {
        try BGTaskScheduler.shared.submit(request)
    } catch {
        Logger.error("后台刷新调度失败: \(error)")
    }
}

func handleAppRefresh(task: BGAppRefreshTask) {
    scheduleAppRefresh()

    task.expirationHandler = {
        task.setTaskCompleted(success: false)
    }

    Task {
        do {
            try await syncData()
            task.setTaskCompleted(success: true)
        } catch {
            task.setTaskCompleted(success: false)
        }
    }
}
```

### BGProcessingTask（后台处理）
- 适合耗时操作：数据库清理、ML 模型训练、大文件处理
- 系统在设备充电+空闲时执行
- `requiresNetworkConnectivity = true` 表示需要网络
- `requiresExternalPower = true` 表示需要充电

### 规范
- Info.plist 中 `BGTaskSchedulerPermittedIdentifiers` 必须声明所有 Task Identifier
- Background Task 不保证执行时机和频率，**禁止依赖它做关键业务**
- `earliestBeginDate` 不是精确时间，系统根据电量、使用习惯决定
- 任务结束前必须调用 `setTaskCompleted`，否则系统可能限制后续调度

---

## App Group 数据共享

### 配置
- 主 Target 和 Extension 都开启 App Group
- Group Identifier：`group.com.app.shared`

### 共享 UserDefaults

```swift
let sharedDefaults = UserDefaults(suiteName: "group.com.app.shared")
sharedDefaults?.set(value, forKey: "key")
```

### 共享文件

```swift
guard let groupURL = FileManager.default.containerURL(
    forSecurityApplicationGroupIdentifier: "group.com.app.shared") else { return }
let sharedFile = groupURL.appendingPathComponent("shared_data.json")
```

---

## 已知陷阱

- **通知权限被拒绝后无法再次弹窗**，只能引导用户去系统设置
- **Badge 数字不会自动清零**，进入前台时必须手动清零：`UIApplication.shared.applicationIconBadgeNumber = 0`
- **Rich Notification 图片下载失败**时静默降级为纯文本通知，不会崩溃
- **Extension 的内存限制约 24MB**，禁止在 Extension 中做图片处理
- **iOS 15+ 通知摘要**：用户可能将通知折叠为摘要，测试时需验证摘要展示效果
- **后台任务调度在 Debug 模式下行为不同**，测试时用 `e -l objc -- (void)[[BGTaskScheduler sharedScheduler] _simulateLaunchForTaskWithIdentifier:@"com.app.refresh"]`
- **APNs Token 在模拟器上无法获取**，Debug 构建需做容错处理
