# 64-Apple Intelligence 与 Controls API

## 本章目标

- 理解 Apple Intelligence 的核心理念与设备端 AI 架构
- 掌握 iOS 18 App Intents 增强 API 与 AppEntity 改进
- 学会让 Siri 通过 AppEntity 理解你的 App 数据
- 掌握 Controls API 创建控制中心自定义控件
- 实战创建控制中心开关与音乐播放控件
- 了解 Spotlight 与 App Indexing 的新增强
- 掌握 Apple Intelligence 隐私框架与审核要求
- 学会渐进式集成 Apple Intelligence 的最佳实践

---

## 1. Apple Intelligence 概述

### 1.1 什么是 Apple Intelligence

> 💡 **生活类比**：Apple Intelligence 就像给你配了一个"贴身管家"——这个管家住在你的手机里（设备端），不需要把你的私事告诉外面的人（隐私保护），但遇到复杂问题时，可以请外面的专家帮忙（私有云计算），而且专家看完就忘，不会留记录。

Apple Intelligence 是 Apple 在 WWDC24 推出的个人智能系统，深度集成于 iOS 18、iPadOS 18 和 macOS Sequoia 中。它不是单一的模型或 API，而是一套**跨设备的智能体验框架**，涵盖：

- **系统级智能**：通知摘要、邮件智能回复、照片清理等
- **Siri 增强**：更自然的对话、屏幕感知、App Intents 深度集成
- **写作工具**：全系统可用的改写、校对、摘要
- **图像智能**：Image Playground、Genmoji、照片智能搜索
- **开发者 API**：App Intents 增强、Controls API、Spotlight 增强

### 1.2 设备端 AI 理念

Apple Intelligence 的核心设计哲学是**设备端优先**：

| 特性 | 说明 |
|------|------|
| 设备端模型 | 约 3B 参数的语言模型，运行在 Apple Silicon 的 Neural Engine 上 |
| 语义索引 | 设备端构建个人语义索引，理解照片、邮件、日历等内容 |
| 无需联网 | 基础智能功能完全离线运行，不发送数据到云端 |
| 低延迟 | 设备端推理延迟远低于云端，响应更即时 |

### 1.3 隐私保护架构

Apple Intelligence 的隐私保护是分层设计的：

```
用户请求
    │
    ├─ 简单任务 → 设备端模型直接处理（数据不出设备）
    │
    ├─ 复杂任务 → 私有云计算（Private Cloud Compute）
    │               │
    │               ├─ 数据仅在处理时存在
    │               ├─ 处理完毕立即删除
    │               ├─ 代码公开可审计
    │               └─ 无法被 Apple 员工访问
    │
    └─ 第三方服务 → 用户明确授权后才调用
```

### 1.4 与云端 AI 的区别

| 对比项 | Apple Intelligence | 云端 AI（ChatGPT 等） |
|--------|-------------------|---------------------|
| 运行位置 | 设备端 + 私有云 | 远程服务器 |
| 数据留存 | 不留存 | 可能用于训练 |
| 隐私等级 | 极高 | 取决于服务商 |
| 延迟 | 低（设备端） | 较高（网络） |
| 模型能力 | 中等（3B 参数） | 强大（数百B 参数） |
| 个人上下文 | 深度理解 | 需手动提供 |
| 离线可用 | 部分可用 | 不可用 |
| 成本 | 免费 | 可能收费 |

> ⚠️ Apple Intelligence 不是 ChatGPT 的替代品，而是互补关系。Apple Intelligence 擅长个人上下文理解和设备端操作，ChatGPT 擅长复杂推理和知识问答。iOS 18 已集成 ChatGPT 作为可选的外部能力。

---

## 2. App Intents 增强

### 2.1 iOS 18 App Intents 新 API

iOS 18 对 App Intents 框架进行了重大增强，使其成为 Apple Intelligence 与 App 交互的核心桥梁。

> 💡 **生活类比**：如果说 iOS 16 的 App Intents 是给 App 装了一扇"门"，那 iOS 18 就是给这扇门装上了"智能门铃"——Siri 不仅知道门在哪，还知道门后有什么、该怎么敲。

| 新特性 | 说明 | 最低版本 |
|--------|------|---------|
| `@Dependency` | Intent 间共享依赖，减少重复代码 | iOS 18 |
| `IntentParameter` 增强 | 更丰富的参数类型与交互 | iOS 18 |
| `PredictableIntent` | 让 Siri 预测用户可能要执行的操作 | iOS 18 |
| `EntityProperty` | 为 Entity 属性添加丰富元数据 | iOS 18 |
| `AssistantIntent` | 为 Siri 提供更精确的语义描述 | iOS 18 |

### 2.2 AppEntity 改进

iOS 18 中 `AppEntity` 新增了 `EntityProperty`，让每个属性都能携带语义信息：

```swift
import AppIntents

struct SongEntity: AppEntity {
    var id: UUID
    var title: String
    var artist: String
    var duration: TimeInterval
    var isFavorite: Bool

    static var typeDisplayRepresentation = TypeDisplayRepresentation(name: "歌曲")
    static var defaultQuery = SongEntityQuery()

    static var properties = SchemaProperties {
        Property(\.title, name: "歌曲名称", description: "歌曲的标题")
        Property(\.artist, name: "艺术家", description: "演唱者或乐队")
        Property(\.duration, name: "时长", description: "歌曲时长（秒）")
        Property(\.isFavorite, name: "已收藏", description: "是否被用户收藏")
    }

    var displayRepresentation: DisplayRepresentation {
        DisplayRepresentation(title: "\(title)", subtitle: "\(artist)")
    }
}
```

### 2.3 EntityQuery 增强

`EntityQuery` 在 iOS 18 中支持更丰富的查询语义，让 Siri 能更精准地找到用户想要的数据：

```swift
struct SongEntityQuery: EntityQuery {
    func entities(for identifiers: [UUID]) async throws -> [SongEntity] {
        await MusicManager.shared.songs
            .filter { identifiers.contains($0.id) }
            .map { $0.toEntity() }
    }

    func suggestedEntities() async throws -> [SongEntity] {
        await MusicManager.shared.recentlyPlayed
            .map { $0.toEntity() }
    }

    func entities(matching query: String) async throws -> [SongEntity] {
        await MusicManager.shared.songs
            .filter {
                $0.title.localizedCaseInsensitiveContains(query) ||
                $0.artist.localizedCaseInsensitiveContains(query)
            }
            .map { $0.toEntity() }
    }
}
```

### 2.4 App Shortcuts Provider

iOS 18 的 `AppShortcutsProvider` 支持更灵活的短语定义和视觉定制：

```swift
struct PlaySongShortcut: AppShortcut {
    static var intent: PlaySongIntent { PlaySongIntent() }

    static var phrases: [AppShortcutPhrase] = [
        "用\(.applicationName)播放\(\.$song)",
        "在\(.applicationName)里播放\(\.$song)",
        "\(.applicationName)放首歌"
    ]

    static var shortTitle: LocalizedStringResource = "播放歌曲"
    static var systemImageName = "play.circle.fill"
}

struct MusicShortcuts: AppShortcutsProvider {
    static var appShortcuts: [AppShortcut] = [
        PlaySongShortcut(),
        PauseMusicShortcut(),
        ToggleFavoriteShortcut()
    ]

    static var shortcutTileColor: ShortcutTileColor = .purple
}
```

> 💡 短语中使用 `\(\.$parameterName)` 可以让 Siri 在识别时直接提取参数，体验更自然。

---

## 3. Siri 增强集成

### 3.1 SiriKit vs App Intents

| 对比项 | SiriKit | App Intents (iOS 18) |
|--------|---------|---------------------|
| 定义方式 | .intentdefinition 文件 | 纯 Swift 代码 |
| 支持领域 | 仅限苹果预定义的十几个领域 | 完全自定义 |
| Siri 对话 | 模板化 | 自然语言交互 |
| 上下文理解 | 无 | 支持屏幕感知和对话上下文 |
| Apple Intelligence | 不支持 | 深度集成 |
| 维护成本 | 高 | 低 |
| 未来方向 | 维护模式，不再更新 | 苹果主推方向 |

> ⚠️ SiriKit 仍然可用，但不会获得新功能。如果你的 App 已有 SiriKit 实现，建议逐步迁移到 App Intents。

### 3.2 AppEntity 让 Siri 理解你的 App 数据

> 💡 **生活类比**：AppEntity 就像给 Siri 一本"App 词典"——没有这本词典，Siri 只知道你的 App 叫什么名字；有了这本词典，Siri 就知道你的 App 里有"歌曲"、"播放列表"、"艺术家"这些概念，以及它们之间的关系。

```swift
struct PlaylistEntity: AppEntity {
    var id: UUID
    var name: String
    var songCount: Int
    var isSmartPlaylist: Bool

    static var typeDisplayRepresentation = TypeDisplayRepresentation(name: "播放列表")
    static var defaultQuery = PlaylistEntityQuery()

    var displayRepresentation: DisplayRepresentation {
        DisplayRepresentation(
            title: "\(name)",
            subtitle: "\(songCount) 首歌曲"
        )
    }
}

struct AddToPlaylistIntent: AppIntent {
    static var title: LocalizedStringResource = "添加到播放列表"
    static var description = IntentDescription("将歌曲添加到指定播放列表")

    @Parameter(title: "歌曲")
    var song: SongEntity

    @Parameter(title: "播放列表")
    var playlist: PlaylistEntity

    func perform() async throws -> some IntentResult {
        try await MusicManager.shared.add(song: song.id, to: playlist.id)
        return .result(dialog: "已将「\(song.title)」添加到「\(playlist.name)」")
    }
}
```

用户可以说："嘿 Siri，把这首歌加到我的跑步歌单"——Siri 能理解"这首歌"（屏幕感知）和"跑步歌单"（EntityQuery 匹配）。

### 3.3 自然语言交互设计

设计 Siri 交互时，要考虑用户会怎么**自然地说话**，而不是怎么**精确地输入**：

| 设计原则 | 说明 | 示例 |
|---------|------|------|
| 短语多样化 | 提供多种说法 | "播放音乐" / "放首歌" / "来点音乐" |
| 参数可推断 | 能从上下文推断参数 | "播放这首歌" → 屏幕上的歌 |
| 容错设计 | 接受模糊输入 | "播放周杰伦" → 搜索该艺术家 |
| 渐进确认 | 复杂操作分步确认 | "删除播放列表" → "确定要删除吗？" |
| 友好反馈 | 结果用自然语言回复 | "好的，正在播放晴天" |

```swift
struct PlaySongIntent: AppIntent {
    static var title: LocalizedStringResource = "播放歌曲"
    static var description = IntentDescription("播放指定歌曲或随机播放")
    static var openAppWhenRun = true

    @Parameter(
        title: "歌曲",
        description: "要播放的歌曲，不指定则随机播放",
        requestValueDialog: "你想听哪首歌？"
    )
    var song: SongEntity?

    func perform() async throws -> some IntentResult {
        if let song = song {
            try await MusicManager.shared.play(song: song.id)
            return .result(dialog: "正在播放「\(song.title)」")
        } else {
            try await MusicManager.shared.shuffle()
            return .result(dialog: "随机播放开始")
        }
    }
}
```

---

## 4. Controls API 详解

### 4.1 控制中心自定义控件

> 💡 **生活类比**：Controls API 就像给用户一排"智能开关面板"——用户可以把你 App 的功能钉在控制中心，就像在家里装一个灯的开关，不用走到灯旁边，抬手就能控制。

iOS 18 引入的 Controls API 允许开发者创建自定义控件，用户可以将这些控件添加到**控制中心**、**锁屏**和**操作按钮**（iPhone 15 Pro 及以上）。

| 控件位置 | 说明 | 用户操作 |
|---------|------|---------|
| 控制中心 | 下拉控制中心的自定义页面 | 长按编辑，添加/移除控件 |
| 锁屏 | 替换锁屏底部左右按钮 | 长按锁屏编辑 |
| 操作按钮 | iPhone 15 Pro+ 的侧边按钮 | 设置 → 操作按钮 |

### 4.2 ControlWidget 协议

`ControlWidget` 是 Controls API 的核心协议，类似于 WidgetKit 的 `Widget` 协议：

```swift
import AppIntents
import WidgetKit

struct SmartLightControl: ControlWidget {
    var body: some ControlWidgetConfiguration {
        StaticControlConfiguration(
            kind: "com.example.SmartHome.light"
        ) {
            ControlWidgetToggle(
                "客厅灯",
                isOn: SmartLightIntent.isOn,
                action: SmartLightIntent()
            ) { isOn in
                Label(
                    isOn ? "客厅灯已开" : "客厅灯已关",
                    systemImage: isOn ? "lightbulb.fill" : "lightbulb"
                )
            }
        }
        .displayName("客厅灯开关")
        .description("控制客厅灯的开关状态")
    }
}
```

### 4.3 ControlWidgetConfiguration

Controls API 提供两种配置方式：

| 配置类型 | 协议 | 说明 | 适用场景 |
|---------|------|------|---------|
| StaticConfiguration | `StaticControlConfiguration` | 固定控件，编译时确定 | 开关、按钮等固定功能 |
| IntentConfiguration | `IntentControlConfiguration` | 动态控件，用户可配置 | 选择特定设备、播放列表等 |

```swift
struct DynamicMusicControl: ControlWidget {
    var body: some ControlWidgetConfiguration {
        IntentControlConfiguration(
            kind: "com.example.MusicApp.play",
            intent: SelectPlaylistIntent.self
        ) { playlist in
            ControlWidgetButton(
                action: PlayPlaylistIntent(playlist: playlist)
            ) {
                Label(
                    "播放 \(playlist.name)",
                    systemImage: "play.circle.fill"
                )
            }
        }
        .displayName("播放播放列表")
        .description("选择一个播放列表快速播放")
    }
}
```

### 4.4 多种控件类型

Controls API 提供三种内置控件类型：

| 控件类型 | 协议 | 交互方式 | 典型场景 |
|---------|------|---------|---------|
| `ControlWidgetButton` | 按钮 | 点击触发一次操作 | 播放/暂停、开始导航 |
| `ControlWidgetToggle` | 切换 | 开/关两种状态 | 灯光开关、静音切换 |
| `ControlWidgetPicker` | 选择器 | 从多个选项中选择 | 选择设备、选择模式 |

**按钮控件**：

```swift
ControlWidgetButton(
    action: StartWorkoutIntent()
) {
    Label("开始跑步", systemImage: "figure.run")
}
```

**切换控件**：

```swift
ControlWidgetToggle(
    "专注模式",
    isOn: FocusModeIntent.isFocused,
    action: FocusModeIntent()
) { isOn in
    Label(
        isOn ? "专注中" : "未专注",
        systemImage: isOn ? "brain.head.profile.fill" : "brain.head.profile"
    )
}
```

**选择器控件**：

```swift
ControlWidgetPicker(
    "空调模式",
    items: [
        .init(title: "制冷", action: SetACModeIntent(mode: .cool)),
        .init(title: "制热", action: SetACModeIntent(mode: .heat)),
        .init(title: "自动", action: SetACModeIntent(mode: .auto))
    ]
) {
    Label("空调模式", systemImage: "thermostat")
}
```

> ⚠️ 控件的 UI 由系统渲染，开发者只能提供 Label 内容和动作。不要试图自定义控件的大小、颜色或布局。

---

## 5. Controls API 实战

### 5.1 创建控制中心开关控件

以下是一个完整的智能家居灯控开关控件：

**第一步：定义 Intent**

```swift
import AppIntents

struct ToggleLightIntent: SetValueIntent {
    static var title: LocalizedStringResource = "切换灯光"
    static var description = IntentDescription("开灯或关灯")

    @Parameter(title: "灯光状态")
    var value: Bool

    init(value: Bool = false) {
        self.value = value
    }

    func perform() async throws -> some IntentResult {
        await LightManager.shared.setLight(on: value)
        return .result(dialog: value ? "灯已打开" : "灯已关闭")
    }
}
```

**第二步：创建 ControlWidget**

```swift
import WidgetKit
import AppIntents

struct LightControlWidget: ControlWidget {
    var body: some ControlWidgetConfiguration {
        StaticControlConfiguration(kind: "com.example.SmartHome.light-toggle") {
            ControlWidgetToggle(
                "客厅灯",
                isOn: ToggleLightIntent.value,
                action: ToggleLightIntent()
            ) { isOn in
                Label(
                    isOn ? "客厅灯 · 开" : "客厅灯 · 关",
                    systemImage: isOn ? "lightbulb.fill" : "lightbulb"
                )
                .tint(isOn ? .yellow : .gray)
            }
        }
        .displayName("客厅灯")
        .description("控制客厅灯的开关")
    }
}
```

**第三步：注册控件**

在 App 的 `Info.plist` 中添加控件组：

```xml
<key>UIControlGroups</key>
<array>
    <dict>
        <key>UIControlGroupIdentifier</key>
        <string>com.example.SmartHome.controls</string>
        <key>UIControlGroupDisplayName</key>
        <string>智能家居</string>
    </dict>
</array>
```

### 5.2 控制中心音乐播放控件

```swift
struct MusicPlayControlWidget: ControlWidget {
    var body: some ControlWidgetConfiguration {
        StaticControlConfiguration(kind: "com.example.MusicApp.play-pause") {
            ControlWidgetButton(
                action: PlayPauseMusicIntent()
            ) {
                Label("播放/暂停", systemImage: "playpause.fill")
            }
        }
        .displayName("音乐播放")
        .description("快速播放或暂停音乐")
    }
}

struct PlayPauseMusicIntent: AppIntent {
    static var title: LocalizedStringResource = "播放/暂停音乐"

    func perform() async throws -> some IntentResult {
        let isPlaying = await MusicManager.shared.togglePlayPause()
        return .result(dialog: isPlaying ? "继续播放" : "已暂停")
    }
}

struct MusicNextTrackControlWidget: ControlWidget {
    var body: some ControlWidgetConfiguration {
        StaticControlConfiguration(kind: "com.example.MusicApp.next-track") {
            ControlWidgetButton(
                action: NextTrackIntent()
            ) {
                Label("下一首", systemImage: "forward.fill")
            }
        }
        .displayName("下一首")
        .description("切换到下一首歌曲")
    }
}

struct NextTrackIntent: AppIntent {
    static var title: LocalizedStringResource = "下一首"

    func perform() async throws -> some IntentResult {
        let song = try await MusicManager.shared.nextTrack()
        return .result(dialog: "正在播放「\(song.title)」")
    }
}
```

### 5.3 与 App Intents 联动

Controls API 和 App Intents 共享同一套 Intent 定义，一个 Intent 可以同时被控件、Siri、快捷指令调用：

```swift
struct SetThermostatIntent: SetValueIntent {
    static var title: LocalizedStringResource = "设置温度"
    static var description = IntentDescription("设置恒温器目标温度")
    static var openAppWhenRun = false

    @Parameter(title: "目标温度")
    var value: Double

    init(value: Double = 22.0) {
        self.value = value
    }

    func perform() async throws -> some IntentResult {
        try await ThermostatManager.shared.setTemperature(value)
        return .result(dialog: "温度已设置为 \(Int(value)) 度")
    }
}

struct ThermostatControlWidget: ControlWidget {
    var body: some ControlWidgetConfiguration {
        StaticControlConfiguration(kind: "com.example.SmartHome.thermostat") {
            ControlWidgetPicker(
                "温度调节",
                items: [
                    .init(title: "20°", action: SetThermostatIntent(value: 20)),
                    .init(title: "22°", action: SetThermostatIntent(value: 22)),
                    .init(title: "24°", action: SetThermostatIntent(value: 24)),
                    .init(title: "26°", action: SetThermostatIntent(value: 26))
                ]
            ) {
                Label("温度调节", systemImage: "thermostat")
            }
        }
        .displayName("恒温器")
        .description("快速调节室内温度")
    }
}
```

> 💡 同一个 `SetThermostatIntent` 既能在控制中心使用，也能通过 Siri 说"把温度调到 22 度"，还能在快捷指令中组合使用——写一次 Intent，三处可用。

---

## 6. Spotlight 与 App Indexing 增强

### 6.1 CSSearchableItem 新用法

iOS 18 增强了 `CSSearchableItem`，支持更丰富的元数据和更智能的索引：

```swift
import CoreSpotlight

func indexSong(_ song: Song) {
    let attributes = CSSearchableItemAttributeSet(contentType: .audio)
    attributes.title = song.title
    attributes.contentDescription = "\(song.artist) · \(song.album)"
    attributes.keywords = [song.title, song.artist, song.album, "音乐"]
    attributes.identifier = song.id.uuidString

    let item = CSSearchableItem(
        uniqueIdentifier: song.id.uuidString,
        domainIdentifier: "com.example.MusicApp.songs",
        attributeSet: attributes
    )

    CSSearchableIndex.default().indexSearchableItems([item])
}
```

### 6.2 Indexed Core Spotlight

iOS 18 引入了 `CSIndexExtension`，允许在后台持续索引 App 内容：

| 特性 | 旧方式 | iOS 18 新方式 |
|------|--------|-------------|
| 索引时机 | App 前台运行时 | 后台 Extension 持续索引 |
| 数据量 | 受前台时间限制 | 可处理大量数据 |
| 增量更新 | 需手动管理 | 支持增量索引 |
| 性能影响 | 可能影响 App 性能 | 独立进程，不影响 App |

### 6.3 App Shortcuts 自动出现在 Spotlight

从 iOS 18 开始，App Shortcuts 会**自动**出现在 Spotlight 搜索结果中，无需额外配置：

```swift
struct PlaySongShortcut: AppShortcut {
    static var intent: PlaySongIntent { PlaySongIntent() }

    static var phrases: [AppShortcutPhrase] = [
        "用\(.applicationName)播放歌曲"
    ]

    static var shortTitle: LocalizedStringResource = "播放歌曲"
    static var systemImageName = "play.circle.fill"
}
```

用户在 Spotlight 搜索"播放歌曲"时，会自动出现你的 App Shortcut。

| 索引方式 | 适用场景 | 是否自动 |
|---------|---------|---------|
| App Shortcuts | 让 App 操作出现在 Spotlight | ✅ 自动 |
| CSSearchableItem | 让 App 数据出现在 Spotlight | ❌ 手动索引 |
| CSIndexExtension | 大量数据的后台持续索引 | ❌ 需配置 Extension |

> ⚠️ Spotlight 索引有数量限制（通常数千条），请只索引最重要的内容。索引过多会降低搜索质量和设备性能。

---

## 7. 隐私与审核要求

### 7.1 Apple Intelligence 隐私框架

Apple Intelligence 的隐私框架对开发者有明确要求：

| 要求 | 说明 | 违规后果 |
|------|------|---------|
| 数据最小化 | 只收集 Intent 必需的数据 | 审核被拒 |
| 本地优先 | 能在设备端处理的不要上云 | 功能受限 |
| 透明告知 | 涉及数据传输要明确告知用户 | 审核被拒 |
| 不得缓存 | 私有云计算返回的数据不得持久化 | 审核被拒 |
| 用户控制 | 用户必须能关闭智能功能 | 审核被拒 |

### 7.2 App Intents 审核规范

| 规范 | 说明 |
|------|------|
| 功能真实 | Intent 必须执行实际操作，不能是空壳 |
| 描述准确 | title 和 description 必须与实际行为一致 |
| 短语合规 | 至少一个短语包含 `\(.applicationName)` |
| 错误处理 | perform() 中的异常必须妥善处理 |
| 无误导 | 不能用 Intent 诱导用户执行非预期操作 |
| 隐私合规 | 不能在 Intent 中偷偷收集用户数据 |

### 7.3 Controls API 限制

| 限制 | 说明 |
|------|------|
| 控件数量 | 每个 App 最多注册有限数量的控件 |
| UI 不可自定义 | 控件外观由系统决定，只能提供 Label |
| 执行时间 | Intent 必须快速完成（通常 < 5 秒） |
| 后台限制 | 控件触发的 Intent 受后台执行时间限制 |
| 审核要求 | 控件功能必须与 App 核心功能相关 |
| 不得模拟系统控件 | 不能创建与系统控件外观/功能相似的控件 |

> ⚠️ 不要创建"打开 App"这种控件——控件的意义在于不打开 App 就能完成操作。如果你的控件只是跳转到 App，审核很可能会被拒。

---

## 8. 适配 Apple Intelligence 的最佳实践

### 8.1 如何设计 Intent 让 AI 更好理解

> 💡 **生活类比**：设计 Intent 就像写菜谱——菜谱写得越清晰、越结构化，AI 这个"厨师"就越容易照着做。模糊的菜谱（"加适量盐"）让 AI 犯难，精确的菜谱（"加 5 克盐"）让 AI 高效。

| 设计原则 | 做法 | 反面示例 | 正面示例 |
|---------|------|---------|---------|
| 语义清晰 | title 用动宾结构 | "操作" | "播放歌曲" |
| 参数明确 | 每个参数都有描述 | `@Parameter(title: "输入")` | `@Parameter(title: "歌曲名称", description: "要播放的歌曲标题")` |
| 类型具体 | 用 Entity 而非 String | `@Parameter var target: String` | `@Parameter var song: SongEntity` |
| 结果可预测 | 返回有意义的值 | 返回空 result | 返回 `.result(value: entity, dialog: "...")` |
| 渐进式 | 先实现核心 Intent | 一开始就做 20 个 Intent | 先做 3 个最常用的 Intent |

### 8.2 数据建模最佳实践

```swift
struct RecipeEntity: AppEntity {
    var id: UUID
    var name: String
    var cuisine: CuisineType
    var cookingTime: Int
    var difficulty: DifficultyLevel
    var ingredients: [IngredientEntity]

    static var typeDisplayRepresentation = TypeDisplayRepresentation(name: "菜谱")
    static var defaultQuery = RecipeEntityQuery()

    static var properties = SchemaProperties {
        Property(\.name, name: "菜名", description: "菜谱的名称")
        Property(\.cuisine, name: "菜系", description: "中餐、西餐、日料等")
        Property(\.cookingTime, name: "烹饪时间", description: "预计烹饪时间（分钟）")
        Property(\.difficulty, name: "难度", description: "简单、中等、困难")
    }

    var displayRepresentation: DisplayRepresentation {
        DisplayRepresentation(
            title: "\(name)",
            subtitle: "\(cuisine.displayName) · \(cookingTime)分钟"
        )
    }
}
```

建模要点：

| 要点 | 说明 |
|------|------|
| 用有意义的 ID | UUID 或业务 ID，不要用索引 |
| 属性即语义 | 每个属性都加 `Property` 描述，让 AI 理解含义 |
| 枚举优于字符串 | `CuisineType` 枚举比 `String` 更精确 |
| 关联关系清晰 | `ingredients` 用 `[IngredientEntity]` 而非 `[String]` |
| displayRepresentation 完整 | 提供 title + subtitle，让 Siri 展示更丰富 |

### 8.3 渐进式集成策略

> 💡 不要试图一步到位——Apple Intelligence 的集成是一个渐进过程，从最简单、最常用的功能开始。

```
第一阶段：基础集成（1-2 天）
├── 定义 2-3 个核心 AppEntity
├── 实现 3-5 个最常用的 Intent
└── 注册 AppShortcutsProvider

第二阶段：Controls API（1-2 天）
├── 为核心操作创建 ControlWidget
├── 实现 Toggle / Button 控件
└── 与现有 Intent 联动

第三阶段：深度集成（3-5 天）
├── 添加 EntityProperty 语义信息
├── 完善 EntityQuery 的搜索逻辑
├── 添加 Spotlight 索引
└── 优化 Siri 对话体验

第四阶段：持续优化
├── 根据用户反馈调整短语
├── 监控 Intent 执行成功率
├── 添加更多 Entity 和 Intent
└── 适配新系统特性
```

### 8.4 集成检查清单

| 检查项 | 是否完成 |
|--------|---------|
| AppEntity 有 `properties` 定义 | ☐ |
| EntityQuery 实现了三个核心方法 | ☐ |
| Intent 的 title 使用动宾结构 | ☐ |
| Intent 的 description 清晰准确 | ☐ |
| 短语至少一个包含 App 名 | ☐ |
| ControlWidget 的 kind 唯一 | ☐ |
| 控件功能与 App 核心功能相关 | ☐ |
| perform() 有错误处理 | ☐ |
| 隐私数据未在 Intent 中泄露 | ☐ |
| 真机测试所有 Intent 和控件 | ☐ |

---

## 本章小结

| 概念 | 一句话总结 |
|------|-----------|
| Apple Intelligence | Apple 的设备端优先个人智能系统，隐私保护为核心 |
| App Intents 增强 | iOS 18 新增 EntityProperty、PredictableIntent 等 API |
| AppEntity | 让 Siri 和 AI 理解你的 App 数据模型 |
| EntityQuery | 告诉系统如何搜索和列出你的 Entity |
| Siri 增强集成 | App Intents 取代 SiriKit，支持自然语言和屏幕感知 |
| Controls API | iOS 18 新框架，创建控制中心/锁屏/操作按钮的自定义控件 |
| ControlWidgetToggle | 开关控件，适合二态操作 |
| ControlWidgetButton | 按钮控件，适合一次性操作 |
| ControlWidgetPicker | 选择器控件，适合多选一操作 |
| Spotlight 增强 | App Shortcuts 自动出现在 Spotlight，CSSearchableItem 支持后台索引 |
| 隐私框架 | 数据最小化、本地优先、透明告知、不得缓存 |
| 渐进式集成 | 从核心 Intent 开始，逐步添加 Entity、Controls、Spotlight |

核心集成路径：

```
定义 Entity + Query（让 AI 理解你的数据）
    ↓
定义 Intent（让 AI 能操作你的 App）
    ↓
创建 AppShortcut（让用户通过 Siri 触发）
    ↓
创建 ControlWidget（让用户通过控制中心触发）
    ↓
添加 Spotlight 索引（让用户通过搜索发现）
    ↓
持续优化（根据用户反馈迭代）
```

Apple Intelligence 代表了 Apple 生态的未来方向——让 AI 深度理解用户的 App 和数据，同时坚守隐私底线。作为开发者，通过 App Intents 和 Controls API 让你的 App 与系统智能深度集成，不仅提升了用户体验，也让你的 App 在 AI 时代更具竞争力。