# 66-CloudKit 与 iCloud 同步

## 本章目标

- 理解多设备同步的必要性，掌握 CloudKit 与其他同步方案的对比选型
- 深入理解 CloudKit 核心概念：Container / Database / Zone / Record / Record Type / Query
- 学会使用 CloudKit Dashboard 管理开发与生产环境、Schema 和记录
- 掌握 CKDatabase 的 CRUD 操作，理解 Public / Private Database 的选择策略
- 掌握 CKQuery 查询与 CKSubscription 订阅机制，实现实时数据推送
- 掌握 CloudKit 与 SwiftUI 集成：NSPersistentCloudKitContainer + Core Data、SwiftData + CloudKit
- 学会使用 NSUbiquitousKeyValueStore 进行轻量键值同步
- 理解离线支持策略与冲突解决方案
- 了解 CloudKit 配额与限制，掌握优化策略
- 掌握 iCloud 权限声明与隐私合规要求

---

## 1. iCloud 同步概述

### 1.1 为什么需要多设备同步

想象一下：你在 iPhone 上记了一条备忘录，切换到 iPad 时却找不到——这就像你在家里写了一份购物清单，到了超市却发现忘带了。多设备同步的核心目标就是：**让用户的数据像影子一样，走到哪里跟到哪里**。

现代用户通常拥有多台 Apple 设备，对数据同步的期望包括：

| 需求 | 描述 | 典型场景 |
|------|------|---------|
| **无缝同步** | 数据变更自动推送到所有设备 | 备忘录、提醒事项 |
| **离线可用** | 无网络时仍可正常使用 | 飞行模式下编辑文档 |
| **实时感知** | 一台设备修改后其他设备立即感知 | 协作编辑、消息 |
| **隐私安全** | 敏感数据端到端加密 | 健康数据、密码 |

### 1.2 同步方案对比

Apple 生态下有多种数据同步方案，各有适用场景：

| 方案 | 数据类型 | 同步方式 | 数据量 | 复杂度 | 典型场景 |
|------|---------|---------|--------|--------|---------|
| **NSUbiquitousKeyValueStore** | 键值对 | 自动 | ≤ 1MB | ⭐ | 用户偏好、设置 |
| **iCloud Document Storage** | 文件 | 自动 | 大文件 | ⭐⭐ | 文档编辑器 |
| **CloudKit** | 结构化数据 | 手动+自动 | ≤ 1GB/用户 | ⭐⭐⭐ | 社交、笔记、任务 |
| **Core Data + CloudKit** | 结构化数据 | 自动 | ≤ 1GB/用户 | ⭐⭐ | 通用数据持久化 |
| **SwiftData + CloudKit** | 结构化数据 | 自动 | ≤ 1GB/用户 | ⭐⭐ | 现代数据持久化 |

> 💡 **选型建议**：如果你的 App 已经使用 Core Data 或 SwiftData，优先选择它们自带的 CloudKit 集成方案，可以零代码实现同步。只有在需要精细控制同步逻辑或跨平台共享数据时，才直接使用 CloudKit API。

---

## 2. CloudKit 基础概念

### 2.1 核心架构

CloudKit 的架构就像一个大型图书馆系统：

```
┌─────────────────────────────────────────────────┐
│                  Container                       │
│            （图书馆大楼）                          │
│  ┌───────────────────────────────────────────┐  │
│  │         Public Database                   │  │
│  │       （公共阅览室——所有人可见）             │  │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  │  │
│  │  │  Zone   │  │  Zone   │  │  Zone   │  │  │
│  │  │(书架区) │  │(书架区) │  │(书架区) │  │  │
│  │  └─────────┘  └─────────┘  └─────────┘  │  │
│  └───────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────┐  │
│  │        Private Database                   │  │
│  │     （私人保险柜——仅用户自己可见）           │  │
│  │  ┌─────────┐  ┌─────────┐                │  │
│  │  │  Zone   │  │  Zone   │                │  │
│  │  └─────────┘  └─────────┘                │  │
│  └───────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────┐  │
│  │       Shared Database                     │  │
│  │     （共享书架——用户间共享）                 │  │
│  └───────────────────────────────────────────┘  │
└─────────────────────────────────────────────────┘
```

### 2.2 核心概念详解

| 概念 | 类比 | 说明 |
|------|------|------|
| **CKContainer** | 图书馆大楼 | 最顶层容器，每个 App 默认有一个，可创建自定义容器 |
| **CKDatabase** | 楼层 | 每个容器包含 Public / Private / Shared 三个数据库 |
| **CKRecordZone** | 书架区 | 数据库内的分区，默认有 `_defaultZone`，自定义 Zone 支持原子提交 |
| **CKRecord** | 一本书 | 一条数据记录，键值对结构，类似字典 |
| **CKRecordType** | 书的分类 | 记录的类型定义，类似数据库表结构 |
| **CKQuery** | 检索条件 | 按条件查询记录，支持排序和分页 |

### 2.3 三种数据库的选择

| 数据库 | 可见性 | 用途 | 典型数据 |
|--------|--------|------|---------|
| **Public Database** | 所有用户可读 | 共享的公共数据 | 城市列表、公告、排行榜 |
| **Private Database** | 仅当前用户 | 用户的私有数据 | 个人笔记、待办事项 |
| **Shared Database** | 被分享的用户 | 用户间共享数据 | 共享相册、协作文档 |

> ⚠️ **重要**：Public Database 的写入需要用户登录 iCloud 账户，但读取不需要。Private Database 的所有操作都需要用户登录 iCloud。

---

## 3. CloudKit Dashboard

### 3.1 开发与生产环境

CloudKit Dashboard（https://icloud.developer.apple.com）是管理 CloudKit 数据的 Web 工具。它有两个环境：

| 环境 | 用途 | 数据 | Schema 修改 |
|------|------|------|------------|
| **Development** | 开发调试 | 模拟数据 | ✅ 可自由修改 |
| **Production** | 正式上线 | 真实用户数据 | ❌ 只能添加字段，不能删除 |

> 💡 **生活类比**：开发环境就像服装设计师的工作室——可以随意裁剪修改；生产环境就像工厂流水线——一旦开模，修改成本极高。

### 3.2 Schema 管理

Schema 定义了 Record Type 的字段结构。在开发阶段，代码中创建的 Record 会自动注册 Schema 到 Development 环境。部署到 Production 前需要通过 Dashboard 操作：

```
Development Schema → Deploy to Production → Production Schema
```

> ⚠️ **关键规则**：部署到 Production 后，Record Type 和字段**不可删除**，只能添加新字段。这就像盖楼——建好的楼层不能拆，只能往上加盖。因此开发阶段要仔细设计数据结构。

### 3.3 记录浏览与操作

在 Dashboard 中可以：

- **浏览记录**：按 Record Type 查看所有记录，支持筛选和排序
- **创建/编辑记录**：手动添加测试数据
- **管理订阅**：查看和删除已注册的订阅
- **查看日志**：监控 API 请求量和错误日志
- **管理用户**：重置开发环境的用户数据

---

## 4. CKDatabase 操作

### 4.1 获取 Database 引用

```swift
import CloudKit

let container = CKContainer(identifier: "iCloud.com.example.myapp")
let publicDB = container.publicCloudDatabase
let privateDB = container.privateCloudDatabase
```

### 4.2 检查 iCloud 账户状态

在执行任何 CloudKit 操作前，应先检查用户的 iCloud 登录状态：

```swift
func checkiCloudStatus() async {
    do {
        let status = try await container.accountStatus()
        switch status {
        case .available:
            print("iCloud 可用")
        case .noAccount:
            print("用户未登录 iCloud")
        case .restricted:
            print("iCloud 受限")
        case .couldNotDetermine:
            print("无法确定 iCloud 状态")
        case .temporarilyUnavailable:
            print("iCloud 暂时不可用")
        @unknown default:
            break
        }
    } catch {
        print("检查 iCloud 状态失败: \(error)")
    }
}
```

### 4.3 创建记录（Create）

```swift
func createNote(title: String, content: String) async throws -> CKRecord {
    let recordID = CKRecord.ID(recordName: UUID().uuidString)
    let record = CKRecord(recordType: "Note", recordID: recordID)
    record["title"] = title
    record["content"] = content
    record["createdAt"] = Date()

    let savedRecord = try await privateDB.save(record)
    return savedRecord
}
```

### 4.4 读取记录（Read）

```swift
func fetchNote(recordName: String) async throws -> CKRecord {
    let recordID = CKRecord.ID(recordName: recordName)
    let record = try await privateDB.record(for: recordID)
    return record
}
```

### 4.5 更新记录（Update）

```swift
func updateNote(record: CKRecord, newTitle: String) async throws -> CKRecord {
    record["title"] = newTitle
    let updatedRecord = try await privateDB.save(record)
    return updatedRecord
}
```

> 💡 **乐观锁机制**：CloudKit 使用 `CKRecord` 的 `recordChangeTag` 实现乐观并发控制。保存时如果 tag 不匹配（说明其他设备已修改），会返回冲突错误，需要手动解决。

### 4.6 删除记录（Delete）

```swift
func deleteNote(recordName: String) async throws {
    let recordID = CKRecord.ID(recordName: recordName)
    try await privateDB.deleteRecord(withID: recordID)
}
```

### 4.7 批量操作

```swift
func batchSave(records: [CKRecord]) async throws -> [CKRecord] {
    let (savedRecords, _) = try await privateDB.modifyRecords(saving: records, deleting: [])
    return savedRecords
}
```

> ⚠️ **批量操作限制**：单次 `modifyRecords` 最多操作 400 条记录。超出需要分批处理。

---

## 5. CKQuery 与 CKSubscription

### 5.1 CKQuery 查询

CKQuery 类似于数据库的 SQL 查询，但使用 `NSPredicate` 和 `NSSortDescriptor`：

```swift
func queryNotes(searchText: String) async throws -> [CKRecord] {
    let predicate = NSPredicate(format: "title CONTAINS %@", searchText)
    let query = CKQuery(recordType: "Note", predicate: predicate)
    query.sortDescriptors = [NSSortDescriptor(key: "createdAt", ascending: false)]

    let (matchResults, _) = try await privateDB.records(matching: query)
    return matchResults.compactMap { try? $0.1.get() }
}
```

### 5.2 查询限制

| 限制项 | 说明 |
|--------|------|
| **索引** | 查询字段必须在 Schema 中标记为可查询（Queryable）或可排序（Sortable） |
| **分页** | 单次查询最多返回 100 条结果，通过 cursor 分页获取更多 |
| **不支持** | 不支持聚合查询、JOIN、子查询 |
| **大小写** | 字符串比较默认不区分大小写 |

### 5.3 分页查询

```swift
func queryAllNotes() async throws -> [CKRecord] {
    let query = CKQuery(recordType: "Note", predicate: NSPredicate(value: true))
    query.sortDescriptors = [NSSortDescriptor(key: "createdAt", ascending: false)]

    var allRecords: [CKRecord] = []
    var cursor: CKQueryOperation.Cursor?

    let (matchResults, queryCursor) = try await privateDB.records(matching: query)
    allRecords += matchResults.compactMap { try? $0.1.get() }
    cursor = queryCursor

    while let currentCursor = cursor {
        let (moreResults, nextCursor) = try await privateDB.records(
            continuingMatchFrom: currentCursor
        )
        allRecords += moreResults.compactMap { try? $0.1.get() }
        cursor = nextCursor
    }

    return allRecords
}
```

### 5.4 CKSubscription 订阅

订阅机制让服务端在数据变化时主动推送通知给客户端，就像订阅了报纸——有新内容自动送到家门口。

#### 查询订阅

当满足条件的记录被创建或修改时触发：

```swift
func subscribeToSharedNotes() async throws {
    let predicate = NSPredicate(format: "isShared = %@", NSNumber(value: true))
    let subscription = CKQuerySubscription(
        recordType: "Note",
        predicate: predicate,
        subscriptionID: "shared-notes-subscription",
        options: [.firesOnRecordCreation, .firesOnRecordUpdate]
    )

    let notificationInfo = CKSubscription.NotificationInfo()
    notificationInfo.title = "新共享笔记"
    notificationInfo.body = "有人分享了新笔记"
    notificationInfo.shouldSendContentAvailable = true
    subscription.notificationInfo = notificationInfo

    try await privateDB.save(subscription)
}
```

#### 区域订阅

当指定 Zone 内任何记录变化时触发：

```swift
func subscribeToZone(zoneID: CKRecordZone.ID) async throws {
    let subscription = CKRecordZoneSubscription(
        zoneID: zoneID,
        subscriptionID: "zone-changes-subscription"
    )

    let notificationInfo = CKSubscription.NotificationInfo()
    notificationInfo.shouldSendContentAvailable = true
    subscription.notificationInfo = notificationInfo

    try await privateDB.save(subscription)
}
```

> 💡 **推送通知前提**：订阅触发的远程推送需要 App 配置了推送通知能力（Push Notification capability），且用户授予了通知权限。`shouldSendContentAvailable = true` 会发送静默推送，让 App 在后台静默同步数据。

---

## 6. CloudKit 与 SwiftUI 集成

### 6.1 NSPersistentCloudKitContainer + Core Data

这是最常用的零代码同步方案。Core Data 负责本地持久化，CloudKit 负责云端同步，两者通过 `NSPersistentCloudKitContainer` 桥接。

> 💡 **生活类比**：Core Data 是你的本地笔记本，CloudKit 是云端备份服务，`NSPersistentCloudKitContainer` 是自动同步的智能笔——你在本地写的每一笔，都会自动同步到云端。

#### 数据模型配置

```swift
import CoreData

class PersistenceController {
    static let shared = PersistenceController()

    let container: NSPersistentContainer

    init(inMemory: Bool = false) {
        container = NSPersistentCloudKitContainer(name: "MyAppModel")

        if inMemory {
            container.persistentStoreDescriptions.first?.url = URL(fileURLWithPath: "/dev/null")
        }

        guard let description = container.persistentStoreDescriptions.first else {
            fatalError("Failed to load store description")
        }

        description.setOption(true as NSNumber,
                             forKey: NSPersistentStoreRemoteChangeNotificationPostOptionKey)

        container.loadPersistentStores { _, error in
            if let error = error {
                fatalError("Unresolved error: \(error)")
            }
        }

        container.viewContext.automaticallyMergesChangesFromParent = true
        container.viewContext.mergePolicy = NSMergeByPropertyObjectTrumpMergePolicy
    }
}
```

#### SwiftUI 中使用

```swift
@main
struct MyApp: App {
    @StateObject private var persistence = PersistenceController.shared

    var body: some Scene {
        WindowGroup {
            ContentView()
                .environment(\.managedObjectContext, persistence.container.viewContext)
        }
    }
}
```

#### 监听远程变更

```swift
func observeRemoteChanges() {
    NotificationCenter.default.addObserver(
        forName: .NSPersistentStoreRemoteChange,
        object: container.persistentStoreCoordinator,
        queue: .main
    ) { notification in
        print("收到远程数据变更: \(notification)")
    }
}
```

### 6.2 SwiftData + CloudKit

SwiftData 是 Apple 在 iOS 17 推出的现代数据框架，与 CloudKit 的集成更加简洁：

```swift
import SwiftData

@Model
class Note {
    var title: String
    var content: String
    var createdAt: Date

    init(title: String, content: String) {
        self.title = title
        self.content = content
        self.createdAt = Date()
    }
}
```

```swift
@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .modelContainer(for: Note.self)
    }
}
```

> ⚠️ **SwiftData + CloudKit 要求**：
> - iOS 17 及以上
> - Xcode 中开启 iCloud 和 CloudKit 能力
> - Model 类必须符合特定要求：属性类型必须是 CloudKit 支持的类型
> - 目前 SwiftData 的 CloudKit 同步仍有一些已知限制，生产环境需充分测试

### 6.3 两种方案对比

| 对比项 | Core Data + CloudKit | SwiftData + CloudKit |
|--------|---------------------|---------------------|
| 最低系统版本 | iOS 13+ | iOS 17+ |
| 代码量 | 较多（需配置 Container） | 极少（声明式） |
| 自定义 Zone | ✅ 支持 | ✅ 支持 |
| 同步粒度控制 | ✅ 灵活 | ⚠️ 较受限 |
| 调试工具 | 成熟 | 仍在完善 |
| 生产稳定性 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |

---

## 7. NSUbiquitousKeyValueStore

### 7.1 概述

`NSUbiquitousKeyValueStore` 是一个轻量级的键值同步方案，就像一个"云端 UserDefaults"——你在 iPhone 上设置了偏好，iPad 上自动生效。

```swift
let store = NSUbiquitousKeyValueStore.default
```

### 7.2 基本操作

```swift
store.set("dark", forKey: "app_theme")
store.set(true, forKey: "notifications_enabled")
store.set(42, forKey: "last_read_chapter")
store.synchronize()

let theme = store.string(forKey: "app_theme")
let enabled = store.bool(forKey: "notifications_enabled")
let chapter = store.longLong(forKey: "last_read_chapter")
```

### 7.3 监听同步变更

```swift
NotificationCenter.default.addObserver(
    forName: NSUbiquitousKeyValueStore.didChangeExternallyNotification,
    object: NSUbiquitousKeyValueStore.default,
    queue: .main
) { notification in
    guard let changedKeys = notification.userInfo?
        [NSUbiquitousKeyValueStoreChangedKeysKey] as? [String] else { return }

    for key in changedKeys {
        print("云端键值变更: \(key)")
    }
}
```

### 7.4 与 UserDefaults 对比

| 对比项 | UserDefaults | NSUbiquitousKeyValueStore |
|--------|-------------|--------------------------|
| 同步范围 | 仅本设备 | 所有 iCloud 设备 |
| 存储限制 | 无硬性限制 | ≤ 1MB 总量 |
| 数据类型 | 基本类型 + PropertyList | 基本类型 + PropertyList |
| 同步延迟 | 无（本地读写） | 数秒到数分钟 |
| 离线可用 | ✅ | ✅（本地缓存） |
| App 卸载后 | 清除 | 保留 |
| 适用数据 | 非同步偏好 | 需同步的偏好、小型配置 |

> ⚠️ **使用限制**：NSUbiquitousKeyValueStore 总量不能超过 1MB，单个键值不宜过大。它不适合存储大量数据或频繁变更的数据——那是 CloudKit 的工作。

---

## 8. 离线支持与冲突解决

### 8.1 离线策略

CloudKit 本身不提供离线支持——它是一个网络 API。离线支持需要本地持久化配合：

```
┌──────────────────────────────────────────────────┐
│                  用户操作                          │
│                      │                            │
│                      ▼                            │
│              ┌──────────────┐                     │
│              │  本地数据库    │ ← Core Data /      │
│              │  (始终可用)    │   SwiftData        │
│              └──────┬───────┘                     │
│                     │                             │
│              ┌──────▼───────┐                     │
│              │  同步引擎     │ ← CloudKit /        │
│              │ (网络可用时)  │   NSPersistentCloud  │
│              └──────┬───────┘   KitContainer      │
│                     │                             │
│              ┌──────▼───────┐                     │
│              │   iCloud     │                     │
│              └──────────────┘                     │
└──────────────────────────────────────────────────┘
```

### 8.2 同步策略

| 策略 | 说明 | 适用场景 |
|------|------|---------|
| **即时同步** | 每次修改立即推送 | 协作类 App |
| **延迟同步** | 积攒一批变更后同步 | 日记、笔记类 App |
| **按需同步** | 用户手动触发同步 | 设置类数据 |
| **混合策略** | 重要数据即时、次要数据延迟 | 大多数 App |

### 8.3 冲突解决

当多个设备同时修改同一条记录时，就会产生冲突。就像两个人同时编辑同一份文档——谁的修改为准？

#### Core Data 的合并策略

```swift
container.viewContext.mergePolicy = NSMergeByPropertyObjectTrumpMergePolicy
```

| 合并策略 | 说明 |
|----------|------|
| `NSMergeByPropertyStoreTrumpMergePolicy` | 服务端数据优先（Store Trump） |
| `NSMergeByPropertyObjectTrumpMergePolicy` | 内存中对象优先（Object Trump） |
| `NSOverwriteMergePolicyPolicy` | 内存对象直接覆盖 |
| `NSRollbackMergePolicy` | 放弃内存修改，回滚到服务端版本 |

#### 手动冲突解决

```swift
func resolveConflict(record: CKRecord, savePolicy: CKModifyRecordsOperation.RecordSavePolicy) {
    let operation = CKModifyRecordsOperation(
        recordsToSave: [record],
        recordIDsToDelete: []
    )
    operation.savePolicy = savePolicy
}
```

| Save Policy | 说明 |
|-------------|------|
| `.ifServerRecordUnchanged` | 仅当服务端记录未变时保存（默认，可能冲突） |
| `.changedKeys` | 仅覆盖修改过的字段（推荐，减少冲突） |
| `.allKeys` | 强制覆盖所有字段（慎用） |

> 💡 **最佳实践**：使用 `.changedKeys` 策略可以最大程度减少冲突——只同步你实际修改的字段，而不是整条记录。就像编辑文档时只提交修改的段落，而不是整个文档。

---

## 9. CloudKit 配额与限制

### 9.1 免费额度

CloudKit 对每个 App 提供了慷慨的免费额度：

| 资源 | 免费额度 | 说明 |
|------|---------|------|
| **资产存储** | 1 GB / 用户 | 文件、图片等大文件 |
| **数据库存储** | 10 MB / 用户 | 结构化数据 |
| **请求量** | 40 次 / 秒 | API 请求频率 |
| **推送量** | 无明确限制 | 订阅推送通知 |
| **传输量** | 2 GB / 天 / 用户 | 数据传输总量 |

### 9.2 API 限制

| 操作 | 限制 |
|------|------|
| 单次查询返回记录数 | 100 条 |
| 单次批量操作 | 400 条 |
| 单条 Record 大小 | 1 MB |
| 单个 Asset 大小 | 250 MB（Private）/ 100 MB（Public） |
| Record Type 字段数 | 最多 400 个 |
| 订阅数量 | 每个数据库 100 个 |

### 9.3 优化策略

```swift
let operation = CKQueryOperation(query: query)
operation.resultsLimit = 100
operation.qualityOfService = .utility
```

| 优化方向 | 具体措施 |
|----------|---------|
| **减少请求次数** | 使用批量操作合并多个请求 |
| **按需查询** | 只查询需要的字段，使用 `desiredKeys` |
| **合理分页** | 使用 cursor 分页，避免一次加载过多数据 |
| **缓存策略** | 本地缓存 + 增量同步，减少全量查询 |
| **QoS 设置** | 非关键同步使用 `.utility`，降低优先级 |
| **Asset 优化** | 大文件使用 CKAsset，上传前压缩 |

```swift
func queryWithDesiredKeys() async throws -> [CKRecord] {
    let query = CKQuery(recordType: "Note", predicate: NSPredicate(value: true))
    let (results, _) = try await privateDB.records(
        matching: query,
        desiredKeys: ["title", "createdAt"]
    )
    return results.compactMap { try? $0.1.get() }
}
```

---

## 10. 审核与隐私

### 10.1 iCloud 权限声明

使用 CloudKit 需要在 Xcode 中配置以下能力：

| 配置项 | 说明 |
|--------|------|
| **iCloud** | 开启 iCloud 能力 |
| **CloudKit** | 勾选 CloudKit 服务 |
| **Container** | 选择或创建 Container ID |

### 10.2 Info.plist 隐私声明

```xml
<key>NSUbiquitousKeyValueStore</key>
<string>需要 iCloud 同步您的偏好设置</string>
<key>NSCloudKitContainerDescription</key>
<string>需要 iCloud 存储和同步您的数据</string>
```

### 10.3 隐私政策要求

App Store 审核对使用 iCloud 同步的 App 有以下要求：

| 要求 | 说明 |
|------|------|
| **隐私政策** | 必须提供隐私政策 URL，说明数据收集和使用方式 |
| **数据声明** | 在 App Store Connect 中声明收集的数据类型 |
| **用户知情** | 首次同步前应告知用户数据将上传至 iCloud |
| **最小化原则** | 只同步必要的数据，不上传无关信息 |
| **数据删除** | 提供让用户删除云端数据的方式 |

> ⚠️ **审核常见拒审原因**：
> - 未提供隐私政策链接
> - Private Database 数据未正确隔离（用户 A 能看到用户 B 的数据）
> - 未声明 iCloud 使用目的
> - 强制要求用户登录 iCloud 才能使用 App 基本功能

### 10.4 合规检查清单

```swift
func checkPrivacyCompliance() -> [String] {
    var issues: [String] = []

    if !hasPrivacyPolicy() {
        issues.append("缺少隐私政策")
    }
    if !hasDataDeletionOption() {
        issues.append("缺少数据删除功能")
    }
    if !informsUserBeforeSync() {
        issues.append("首次同步前未告知用户")
    }

    return issues
}
```

---

## 本章小结

| 主题 | 核心要点 |
|------|---------|
| **iCloud 同步概述** | 多设备同步是现代 App 的基本需求，根据数据类型选择合适的同步方案 |
| **CloudKit 基础** | Container → Database → Zone → Record 四层架构，Public/Private/Shared 三种数据库 |
| **CloudKit Dashboard** | 开发环境可自由修改 Schema，生产环境只能添加字段，部署前务必仔细设计 |
| **CKDatabase CRUD** | 使用 async/await 的现代 API，注意批量操作限制和乐观锁机制 |
| **CKQuery 与订阅** | 查询需要字段索引，订阅实现服务端推送，支持查询订阅和区域订阅 |
| **SwiftUI 集成** | NSPersistentCloudKitContainer 零代码同步，SwiftData 更简洁但需 iOS 17+ |
| **NSUbiquitousKeyValueStore** | 轻量键值同步，≤ 1MB，适合偏好设置，不适合大量数据 |
| **离线与冲突** | 本地持久化 + 增量同步，使用 `.changedKeys` 策略减少冲突 |
| **配额与限制** | 免费额度通常足够，注意单次查询和批量操作限制，合理使用 QoS |
| **审核与隐私** | 必须声明 iCloud 使用目的，提供隐私政策和数据删除功能，确保数据隔离 |

← [-Core ML 与设备端 AI](./65-Core-ML与设备端AI.md) | [-音频与视频处理](./67-音频与视频处理.md) →
