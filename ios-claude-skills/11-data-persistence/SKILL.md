---
name: data-persistence
description: 涉及数据存储、持久化、CoreData、SwiftData、UserDefaults、Keychain、FileManager、数据库迁移、缓存、沙盒文件管理的任务
---

# 数据持久化 / 存储层

## 存储方案选型

| 方案 | 适用场景 | 线程安全 | 数据量 |
|------|---------|---------|--------|
| UserDefaults | 轻量配置、用户偏好 | ✅ 主线程安全 | < 1MB |
| Keychain | Token、密钥、敏感凭证 | ✅ | 极小 |
| CoreData | 结构化关系数据、离线缓存 | 需配置 | 大 |
| SwiftData | 新项目结构化数据（iOS 17+） | ✅ | 大 |
| FileManager | 文件、图片、导出数据 | 需加锁 | 任意 |

**选择原则：**
- 能用 UserDefaults 解决的不上 CoreData
- 敏感数据（Token、密码）**必须存 Keychain**，禁止存 UserDefaults
- 新项目且最低支持 iOS 17+ 可选 SwiftData，否则用 CoreData
- 文件类数据（导出 PDF、录音文件）用 FileManager

---

## UserDefaults

### 规范
- 封装为 `UserDefaultsStorage`，禁止在业务代码中直接调用 `UserDefaults.standard`
- Key 统一管理，禁止散落字符串：

```swift
struct UserDefaultsStorage {
    private let defaults = UserDefaults.standard

    enum Key: String {
        case hasCompletedOnboarding
        case selectedTheme
        case lastSyncTimestamp
        case launchCount
    }

    var hasCompletedOnboarding: Bool {
        get { defaults.bool(forKey: Key.hasCompletedOnboarding.rawValue) }
        set { defaults.set(newValue, forKey: Key.hasCompletedOnboarding.rawValue) }
    }
}
```

### 已知陷阱
- `bool(forKey:)` 在 key 不存在时返回 `false`，无法区分"未设置"和"设置为 false"。需要区分时用 `object(forKey:) != nil` 判断
- `UserDefaults.standard` 写入是异步的，`synchronize()` 在 iOS 12+ 已无必要（系统自动处理）
- **禁止存储大数据**（图片 Base64、JSON 数组等），会导致启动变慢
- App Group 共享：必须用 `UserDefaults(suiteName: "group.com.app.shared")`

---

## Keychain

### 封装

```swift
final class KeychainStorage {
    static let shared = KeychainStorage()

    private let service = Bundle.main.bundleIdentifier ?? "com.app.default"

    func save(_ data: Data, for key: String) throws {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: key,
        ]
        SecItemDelete(query as CFDictionary)

        var addQuery = query
        addQuery[kSecValueData as String] = data
        addQuery[kSecAttrAccessible as String] = kSecAttrAccessibleAfterFirstUnlock
        let status = SecItemAdd(addQuery as CFDictionary, nil)
        guard status == errSecSuccess else {
            throw KeychainError.saveFailed(status)
        }
    }

    func load(for key: String) -> Data? {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: key,
            kSecReturnData as String: true,
            kSecMatchLimit as String: kSecMatchLimitOne,
        ]
        var result: AnyObject?
        let status = SecItemCopyMatching(query as CFDictionary, &result)
        return status == errSecSuccess ? result as? Data : nil
    }

    func delete(for key: String) {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: key,
        ]
        SecItemDelete(query as CFDictionary)
    }
}
```

### 规范
- `kSecAttrAccessible` 使用 `afterFirstUnlock`（后台访问场景），禁止 `whenUnlocked`（后台推送时无法读取）
- 存储字符串时先 `data(using: .utf8)`，读取时 `String(data:)`
- App Group 共享：添加 `kSecAttrAccessGroup`，并在 Keychain Groups 中配置
- **禁止在 Keychain 中存储大量数据**（限制 4KB 以内）

---

## CoreData

### 目录结构
```
Core/
├── Storage/
│   ├── CoreDataStack.swift        # Stack 初始化
│   ├── CoreDataModels.xcdatamodeld
│   ├── Entities/                  # NSManagedObject 子类
│   │   ├── User+CoreDataClass.swift
│   │   └── User+CoreDataProperties.swift
│   └── Migrations/               # 迁移映射模型
```

### CoreDataStack

```swift
final class CoreDataStack {
    static let shared = CoreDataStack()

    lazy var persistentContainer: NSPersistentContainer = {
        let container = NSPersistentContainer(name: "CoreDataModels")
        container.loadPersistentStores { _, error in
            if let error { fatalError("CoreData 加载失败: \(error)") }
        }
        container.viewContext.automaticallyMergesChangesFromParent = true
        container.viewContext.mergePolicy = NSMergeByPropertyObjectTrumpMergePolicy
        return container
    }()

    var viewContext: NSManagedObjectContext { persistentContainer.viewContext }

    func newBackgroundContext() -> NSManagedObjectContext {
        persistentContainer.newBackgroundContext()
    }

    func performBackgroundTask(_ block: @escaping (NSManagedObjectContext) -> Void) {
        persistentContainer.performBackgroundTask(block)
    }
}
```

### 规范
- **写操作必须在 backgroundContext**，禁止在 viewContext 上写（会阻塞 UI）
- **读操作用 viewContext**（主线程，配合 `NSFetchedResultsController`）
- `automaticallyMergesChangesFromParent = true` 必须设置，否则后台写入主线程看不到
- `mergePolicy` 统一用 `NSMergeByPropertyObjectTrumpMergePolicy`（内存版本优先）
- Entity 命名大写驼峰，属性命名小写驼峰
- **禁止在 VC 中直接操作 CoreData**，通过 Repository 层封装：

```swift
protocol UserRepositoryProtocol {
    func fetchUser(id: String) async throws -> User?
    func saveUser(name: String, email: String) async throws -> User
    func deleteUser(_ user: User) async throws
}

final class UserRepository: UserRepositoryProtocol {
    private let stack: CoreDataStack

    init(stack: CoreDataStack = .shared) {
        self.stack = stack
    }

    func fetchUser(id: String) async throws -> User? {
        let context = stack.viewContext
        let request: NSFetchRequest<User> = User.fetchRequest()
        request.predicate = NSPredicate(format: "id == %@", id)
        request.fetchLimit = 1
        return try context.fetch(request).first
    }

    func saveUser(name: String, email: String) async throws -> User {
        try await withCheckedThrowingContinuation { continuation in
            stack.performBackgroundTask { context in
                let user = User(context: context)
                user.name = name
                user.email = email
                user.createdAt = Date()
                do {
                    try context.save()
                    continuation.resume(returning: user)
                } catch {
                    continuation.resume(throwing: error)
                }
            }
        }
    }
}
```

### 数据迁移

#### 轻量迁移（自动）
- 新增可选属性、新增 Entity、新增关系 → 自动处理
- 在 `loadPersistentStores` 前设置：
```swift
let description = container.persistentStoreDescriptions.first
description?.setOption(true as NSNumber, forKey: NSMigratePersistentStoresAutomaticallyOption)
description?.setOption(true as NSNumber, forKey: NSInferMappingModelAutomaticallyOption)
```

#### 手动迁移
- 轻量迁移无法处理时（重命名属性、合并 Entity 等），创建 Mapping Model
- 迁移模型命名：`CoreDataModels v2 → v3`
- **禁止修改已发布的迁移模型**，只能追加新迁移
- 测试迁移：每次版本升级前，用旧版本数据库测试迁移路径

### 已知陷阱
- **CoreData 不支持多线程共享 NSManagedObject**，跨线程必须传 `objectID`，再用 `context.object(with:)` 获取
- `NSFetchedResultsController` 的 `delegate` 必须在主线程处理回调
- 删除关系时设置 `deleteRule`：一对一用 `.cascade`（级联删除），多对多用 `.nullify`（置空）
- CoreData 与 CloudKit 同步：开启 `NSPersistentCloudKitContainer`，但**禁止在同步场景使用 `NSPersistentStoreInMemory`**

---

## SwiftData（iOS 17+）

### 基本用法

```swift
import SwiftData

@Model
final class User {
    @Attribute(.unique) var id: String
    var name: String
    var email: String
    var createdAt: Date

    init(id: String = UUID().uuidString, name: String, email: String) {
        self.id = id
        self.name = name
        self.email = email
        self.createdAt = Date()
    }
}
```

### 规范
- 新项目且最低 iOS 17+ 可用 SwiftData 替代 CoreData
- `@Attribute(.unique)` 替代 CoreData 的唯一约束
- 关系用 `@Relationship` 标注，删除规则通过 `deleteRule` 参数指定
- **禁止在 ViewModel 之外使用 `@Query`**，`@Query` 只能在 SwiftUI View 中使用
- UIKit 项目中使用 SwiftData：手动创建 `ModelContainer` + `ModelContext`

```swift
let schema = Schema([User.self])
let config = ModelConfiguration(schema: schema, isStoredInMemoryOnly: false)
let container = try ModelContainer(for: schema, configurations: [config])
let context = container.mainContext
```

### 迁移
- 轻量迁移：`VersionedSchema` + `SchemaMigrationPlan`
- `@Migration(.lightweight)` 处理简单变更
- 手动迁移：实现 `MigrationPlan` 的 `migrate` 方法

---

## FileManager / 沙盒文件

### 目录约定

```swift
enum AppDirectory {
    static let documents = FileManager.default.urls(for: .documentDirectory, in: .userDomainMask)[0]
    static let caches = FileManager.default.urls(for: .cachesDirectory, in: .userDomainMask)[0]
    static let temporary = FileManager.default.temporaryDirectory
    static let applicationSupport = FileManager.default.urls(for: .applicationSupportDirectory, in: .userDomainMask)[0]
}
```

| 目录 | 用途 | iCloud 备份 | 系统清理 |
|------|------|-----------|---------|
| Documents | 用户生成内容（导出文件） | ✅ | ❌ |
| Caches | 可重建的缓存（下载图片） | ❌ | ✅ |
| tmp | 临时文件（处理中数据） | ❌ | ✅ |
| Application Support | App 数据库、配置 | ✅ | ❌ |

### 规范
- 缓存文件放 `Caches/`，系统存储不足时自动清理
- 临时文件用 `tmp/`，用完即删，监听 `UIApplication.didReceiveMemoryWarningNotification` 主动清理
- 需要备份但不希望被 iCloud 同步的文件，设置 `URLResourceKey.isExcludedFromBackupKey = true`
- 大文件（视频、数据库）放 Application Support，禁止放 Documents（会被 iCloud 同步占空间）
- **禁止硬编码路径**，使用 `FileManager.default.urls(for:in:)` 获取

### 存储空间检查

```swift
func availableStorageMB() -> Int? {
    let url = AppDirectory.documents
    let values = try? url.resourceValues(forKeys: [.volumeAvailableCapacityForImportantUsageKey])
    return values?.volumeAvailableCapacityForImportantUsage.map { Int($0 / 1024 / 1024) }
}
```

---

## 缓存策略

### 内存缓存
- 小数据用 `NSCache`（系统内存不足时自动清理）
- Key 用 `NSString`，Value 用 `NSObject` 子类

### 磁盘缓存
- 图片缓存：Kingfisher 自动管理（默认 200MB 上限）
- API 响应缓存：URLCache（根据 HTTP `Cache-Control` 头）
- 自定义缓存：`Caches/` 目录 + 过期时间 + 容量上限

### 缓存清理
- App 进入后台时检查缓存大小，超过阈值提示用户
- 提供"清除缓存"设置项，清理 `Caches/` 目录
- 禁止清理 `Documents/` 和 `Application Support/`

---

## 数据加密

### at-rest 加密
- iOS 默认启用 Data Protection（文件级加密）
- 敏感文件设置 `FileProtectionType.complete`（设备解锁后才可访问）：
```swift
try data.write(to: url, options: [.completeFileProtection])
```
- CoreData 数据库文件加密：`NSPersistentStoreFileProtectionKey = NSFileProtectionComplete`

### 禁止事项
- 禁止自研加密算法，使用系统 CryptoKit 或 Security 框架
- 禁止将加密密钥和加密数据存在同一位置
- 禁止在日志中打印敏感数据
