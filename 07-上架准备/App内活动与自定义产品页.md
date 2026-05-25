# App内活动与自定义产品页

> 🎯 **本章目标**：掌握 App Store 的 In-App Events 和 Custom Product Pages 功能，学会创建和管理活动与自定义产品页，提升 App 的获客和留存能力。

---

## 1. In-App Events 概述

### 1.1 什么是 In-App Events

In-App Events（App 内活动）是 Apple 在 2021 年推出的 App Store 功能，允许开发者在 App Store 中展示 App 内的限时活动，帮助用户发现 App 内的精彩内容。

> 💡 生活类比：App 内活动就像商场的"促销海报"——
> - 贴在门口（App Store 产品页）吸引路人进店
> - 告诉顾客"本周有限时特惠"（活动信息）
> - 不同海报吸引不同人群（精准获客）
> - 活动结束后自动撤下（时效性）

**App 内活动的展示位置：**

| 展示位置 | 说明 |
|----------|------|
| App 产品页 | 活动卡片直接展示在产品页上 |
| App Store 搜索结果 | 搜索时可能展示相关活动 |
| App Store Today 标签 | Apple 编辑可能推荐优质活动 |
| "为你推荐" | 个性化推荐中展示活动 |
| 通知 | 用户可订阅活动通知 |

### 1.2 活动类型

Apple 定义了以下活动类型：

| 类型 | 英文名 | 适用场景 | 示例 |
|------|--------|----------|------|
| 挑战 | Challenge | 竞技、排名、成就 | 健身挑战赛、编程马拉松 |
| 竞赛 | Competition | 对抗性比赛 | 电竞锦标赛、知识竞赛 |
| 线上活动 | Live Event | 实时互动 | 直播、线上演唱会 |
| 重大更新 | Major Update | 大版本更新 | 新赛季、新地图、新功能 |
| 新季内容 | New Season | 周期性内容更新 | 新赛季、新学期课程 |
| 首发 | Premiere | 首次发布 | 新电影上映、新专辑发布 |
| 特别活动 | Special Event | 特殊场合 | 周年庆、节日活动 |
| 促销 | Promotion | 限时优惠 | 限时折扣、买一送一 |

### 1.3 活动的价值

| 价值维度 | 说明 |
|----------|------|
| 获客 | 在 App Store 中展示活动，吸引新用户下载 |
| 留存 | 活动通知召回已安装用户 |
| 转化 | 限时活动促进付费转化 |
| 曝光 | 活动可能被 Apple 编辑推荐 |
| ASO | 活动关键词增加搜索曝光 |

---

## 2. 创建 In-App Events

### 2.1 创建前的准备

**创建活动前需要准备：**

| 准备项 | 说明 | 规格 |
|--------|------|------|
| 活动图片 | 活动的视觉展示 | 1024x1024px，JPG/PNG |
| 活动视频（可选） | 活动预览视频 | MP4，最长 30 秒 |
| 深链接 | 点击活动后跳转到 App 内指定页面 | URL Scheme / Universal Link |
| 活动文案 | 活动名称和描述 | 名称最多 30 字符，简短描述最多 50 字符，详细描述最多 120 字符 |
| 活动时间 | 开始和结束时间 | 最多 31 天 |

### 2.2 在 App Store Connect 中创建活动

**步骤：**

1. 登录 [App Store Connect](https://appstoreconnect.apple.com)
2. 选择「我的 App」→ 选择你的 App
3. 在左侧菜单中选择「App 内活动」
4. 点击「+」按钮创建新活动
5. 填写活动信息

**活动信息字段：**

| 字段 | 说明 | 限制 |
|------|------|------|
| 参考名称 | 内部标识名称，不显示给用户 | 最多 64 字符 |
| 活动类型 | 选择活动类型 | 必选 |
| 优先级 | 在产品页上的展示优先级 | 标准优先级 / 高优先级 |
| 开始时间 | 活动开始时间 | 必填 |
| 结束时间 | 活动结束时间 | 必填，最多 31 天 |

### 2.3 活动本地化信息

每个活动需要为每个支持的语言提供本地化信息：

| 字段 | 说明 | 限制 |
|------|------|------|
| 活动名称 | 展示给用户的活动标题 | 最多 30 字符 |
| 简短描述 | 活动的简要说明 | 最多 50 字符 |
| 详细描述 | 活动的详细说明 | 最多 120 字符 |
| 活动图片 | 活动的视觉展示 | 1024x1024px |

**各语言文案示例（中文）：**

| 字段 | 示例 |
|------|------|
| 活动名称 | 春节限时挑战赛 |
| 简短描述 | 完成每日任务赢取新年限定奖励 |
| 详细描述 | 参与春节特别挑战，连续7天完成每日任务，即可获得限定皮肤和金币奖励！ |

**各语言文案示例（英文）：**

| 字段 | 示例 |
|------|------|
| 活动名称 | Lunar New Year Challenge |
| 简短描述 | Complete daily tasks for exclusive rewards |
| 详细描述 | Join our special Lunar New Year challenge! Complete daily tasks for 7 consecutive days to earn exclusive skins and coin rewards! |

### 2.4 活动图片设计规范

**图片规格：**

| 属性 | 要求 |
|------|------|
| 尺寸 | 1024 x 1024 像素 |
| 格式 | JPG 或 PNG |
| 色彩空间 | sRGB |
| 文件大小 | 不超过 8 MB |
| 圆角 | 系统自动添加，设计时不要自带圆角 |

**设计建议：**

| 建议 | 说明 |
|------|------|
| 视觉冲击 | 使用鲜明的颜色和清晰的图形 |
| 文字精简 | 图片上尽量少放文字 |
| 品牌一致 | 与 App 图标和产品页风格一致 |
| 避免截图 | 不要使用 App 截图作为活动图片 |
| 避免边框 | 不要添加装饰性边框 |
| 安全区域 | 重要内容避开边缘（系统可能裁剪） |

### 2.5 深链接配置

深链接让用户点击活动后直接跳转到 App 内对应页面。

**深链接类型：**

| 类型 | 格式 | 示例 |
|------|------|------|
| URL Scheme | myapp:// | myapp://event/spring-challenge |
| Universal Link | https:// | https://myapp.com/event/spring-challenge |

**在 App 中处理深链接：**

```swift
// UIKit - AppDelegate
func application(
    _ application: UIApplication,
    continue userActivity: NSUserActivity,
    restorationHandler: @escaping ([UIUserActivityRestoring]?) -> Void
) -> Bool {
    guard userActivity.activityType == NSUserActivityTypeBrowsingWeb,
          let url = userActivity.webpageURL else {
        return false
    }

    handleDeepLink(url)
    return true
}

// URL Scheme
func application(
    _ app: UIApplication,
    open url: URL,
    options: [UIApplication.OpenURLOptionsKey: Any] = [:]
) -> Bool {
    handleDeepLink(url)
    return true
}

func handleDeepLink(_ url: URL) {
    guard let host = url.host else { return }

    switch host {
    case "event":
        if let eventId = url.pathComponents.last {
            navigateToEvent(eventId: eventId)
        }
    default:
        break
    }
}
```

```swift
// SwiftUI
@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
                .onOpenURL { url in
                    handleDeepLink(url)
                }
        }
    }

    func handleDeepLink(_ url: URL) {
        guard let host = url.host else { return }

        switch host {
        case "event":
            if let eventId = url.pathComponents.last {
                NotificationCenter.default.post(
                    name: .navigateToEvent,
                    object: nil,
                    userInfo: ["eventId": eventId]
                )
            }
        default:
            break
        }
    }
}
```

### 2.6 活动优先级

| 优先级 | 说明 | 展示效果 |
|--------|------|----------|
| 标准优先级 | 默认优先级 | 正常展示 |
| 高优先级 | 重要活动 | 更大的展示卡片、更靠前的位置 |

⚠️ **警告**：高优先级活动每个 App 同时只能有一个。滥用高优先级可能导致审核被拒。

**适合高优先级的活动：**

| 适合 | 不适合 |
|------|--------|
| 年度大型活动 | 常规促销 |
| 重大版本更新 | 每周例行活动 |
| 独家首发 | 重复性活动 |

---

## 3. 活动的审核和发布流程

### 3.1 审核流程

```
创建活动 → 提交审核 → 等待审核 → 审核通过 → 活动发布
                              │
                              └── 审核被拒 → 修改后重新提交
```

**审核时间：**

| 情况 | 预计时间 |
|------|----------|
| 首次提交 | 24-48 小时 |
| 后续提交 | 12-24 小时 |
| 加急审核 | 几小时（需申请） |

### 3.2 审核要求

Apple 对 In-App Events 的审核要求：

| 要求 | 说明 |
|------|------|
| 活动必须真实 | 活动必须在 App 内真实存在 |
| 时间准确 | 活动时间必须与实际一致 |
| 深链接有效 | 点击活动必须能跳转到正确页面 |
| 内容一致 | 活动描述必须与 App 内实际内容一致 |
| 图片规范 | 图片不得包含误导性内容 |
| 无违规内容 | 不得包含违法、暴力、色情等内容 |

### 3.3 常见审核被拒原因

| 被拒原因 | 说明 | 解决方法 |
|----------|------|----------|
| 活动不存在 | App 内找不到对应活动 | 确保活动在 App 内已上线 |
| 深链接无效 | 点击后无法跳转 | 测试深链接，确保正确跳转 |
| 时间不一致 | 活动时间与实际不符 | 更新活动时间 |
| 图片误导 | 图片与实际内容不符 | 使用真实活动截图或设计图 |
| 类型不匹配 | 活动类型与实际不符 | 选择正确的活动类型 |
| 重复提交 | 同一活动重复创建 | 删除重复活动 |
| 过度营销 | 文案过于夸张 | 使用客观描述 |

### 3.4 活动状态

| 状态 | 说明 |
|------|------|
| 草稿 | 尚未提交审核 |
| 等待审核 | 已提交，等待 Apple 审核 |
| 审核被拒 | 审核未通过，需要修改 |
| 已批准 | 审核通过，等待发布 |
| 即将开始 | 活动即将开始（开始前 24 小时内） |
| 进行中 | 活动正在进行 |
| 已结束 | 活动已结束 |

### 3.5 活动时间管理

```
提交流程时间线：

Day -7:  创建活动，提交审核
Day -5:  审核通过
Day -3:  活动在 App Store 预热展示
Day 0:   活动正式开始
Day 7:   活动结束
Day 14:  活动从 App Store 移除
```

💡 **提示**：建议在活动开始前至少 5-7 天提交审核，留出审核和预热时间。

### 3.6 活动修改和取消

| 操作 | 说明 | 限制 |
|------|------|------|
| 修改草稿 | 随时修改 | 无限制 |
| 修改已批准活动 | 修改后需重新审核 | 进行中的活动不能修改 |
| 取消活动 | 撤下活动 | 已开始的活动不能取消 |
| 提前结束 | 提前结束活动 | 需要联系 Apple |

⚠️ **警告**：活动开始后无法修改或取消。请务必在活动开始前仔细检查所有信息。

---

## 4. 自定义产品页（Custom Product Pages）

### 4.1 什么是自定义产品页

自定义产品页（Custom Product Pages，简称 CPP）允许你为同一个 App 创建最多 35 个不同的产品页变体，每个变体可以有不同的截图、描述和推广文案，用于针对不同用户群体进行精准推广。

> 💡 生活类比：自定义产品页就像"个性化橱窗"——
> - 同一家店的不同橱窗展示不同商品
> - 面向年轻人的橱窗展示潮流款
> - 面向商务人士的橱窗展示经典款
> - 每个橱窗都能吸引对应的客户进店

### 4.2 自定义产品页 vs 默认产品页

| 特性 | 默认产品页 | 自定义产品页 |
|------|------------|--------------|
| 数量 | 1 个 | 最多 35 个 |
| 截图 | 固定 | 可自定义 |
| 描述 | 固定 | 可自定义 |
| 推广文案 | 固定 | 可自定义 |
| App 预览 | 固定 | 可自定义 |
| 图标 | 固定 | 固定（不可修改） |
| 评分 | 共享 | 共享 |
| 年龄分级 | 共享 | 共享 |
| 类别 | 共享 | 共享 |
| 版本信息 | 共享 | 共享 |

### 4.3 创建自定义产品页

**步骤：**

1. 登录 App Store Connect
2. 选择「我的 App」→ 选择你的 App
3. 在左侧菜单中选择「自定义产品页」
4. 点击「+」创建新变体
5. 输入变体名称（内部使用，不显示给用户）
6. 编辑本地化内容

**可自定义的内容：**

| 内容 | 说明 | 限制 |
|------|------|------|
| App 预览 | 视频预览 | 最多 3 个 |
| 截图 | 产品页截图 | 必填，按设备尺寸 |
| 推广文案 | 产品页顶部文案 | 最多 170 字符 |
| 描述 | App 详细描述 | 最多 4000 字符 |
| 关键词 | 搜索关键词 | 最多 100 字符 |

### 4.4 不同用户群体的产品页策略

**按用户画像定制：**

| 用户群体 | 截图策略 | 文案策略 | 关键词策略 |
|----------|----------|----------|------------|
| 学生 | 展示学习功能、学生优惠 | 强调学习效率、考试备考 | 学习、备考、笔记 |
| 职场人士 | 展示办公功能、效率工具 | 强调工作效率、团队协作 | 办公、效率、协作 |
| 创作者 | 展示创作工具、作品展示 | 强调创作自由、社区分享 | 创作、设计、分享 |
| 游戏玩家 | 展示游戏场景、角色 | 强调游戏体验、竞技乐趣 | 游戏、竞技、挑战 |

**按获客渠道定制：**

| 渠道 | 产品页策略 |
|------|------------|
| 搜索广告 | 突出核心卖点，与搜索词匹配 |
| 社交媒体 | 展示社交功能，使用社交语言 |
| KOL 推荐 | 展示 KOL 推荐内容 |
| 邮件营销 | 展示优惠信息 |
| 内容营销 | 展示深度功能 |

### 4.5 自定义产品页的 URL

每个自定义产品页有唯一的 URL，用于追踪不同渠道的流量：

```
默认产品页：
https://apps.apple.com/app/id1234567890

自定义产品页：
https://apps.apple.com/app/id1234567890?ppid=abc123def456
```

**URL 参数说明：**

| 参数 | 说明 |
|------|------|
| id | App 的 Apple ID |
| ppid | 自定义产品页标识符 |

**在营销链接中使用：**

```swift
// 生成带来源追踪的链接
func generateMarketingLink(pageId: String, source: String, campaign: String) -> URL {
    var components = URLComponents(string: "https://apps.apple.com/app/id1234567890")!
    components.queryItems = [
        URLQueryItem(name: "ppid", value: pageId),
        URLQueryItem(name: "ct", value: campaign),
        URLQueryItem(name: "mt", value: "8")
    ]
    return components.url!
}
```

### 4.6 自定义产品页与 App 的关联

```swift
import StoreKit

// 获取用户通过哪个自定义产品页下载的 App
Task {
    if let attribution = try? await Product.SubscriptionInfo.Attribution {
        // 用户通过自定义产品页下载
    }
}

// iOS 15+ 获取产品页归属
if let scene = UIApplication.shared.connectedScenes.first as? UIWindowScene {
    Task {
        if let attribution = try? await AppStore.attribution(scene) {
            // 处理归属数据
        }
    }
}
```

### 4.7 自定义产品页的管理

| 操作 | 说明 |
|------|------|
| 创建变体 | 最多 35 个 |
| 编辑变体 | 修改截图、文案等 |
| 归档变体 | 暂时停用（不影响已下载用户） |
| 删除变体 | 永久删除（需先归档） |
| 复制变体 | 基于现有变体创建新变体 |

⚠️ **警告**：自定义产品页的修改需要经过审核。修改期间，原版本继续展示。

---

## 5. 产品页 A/B 测试（Product Page Optimization）

### 5.1 什么是产品页优化

产品页优化（Product Page Optimization，简称 PPO）是 Apple 提供的 A/B 测试功能，允许你对默认产品页的不同变体进行测试，找出转化率最高的版本。

**PPO vs 传统 A/B 测试：**

| 特性 | PPO | 传统 A/B 测试 |
|------|-----|---------------|
| 测试平台 | App Store 内 | 自建落地页 |
| 测试对象 | 产品页元素 | 任意页面 |
| 流量来源 | App Store 自然流量 | 广告流量 |
| 可测试元素 | 截图、图标、推广文案 | 任意元素 |
| 统计显著性 | Apple 自动计算 | 需自行计算 |
| 费用 | 免费 | 可能有成本 |

### 5.2 可测试的元素

| 元素 | 可测试 | 说明 |
|------|--------|------|
| App 图标 | ✅ | 最多 3 个变体 |
| 截图 | ✅ | 最多 3 个变体 |
| 推广文案 | ✅ | 最多 3 个变体 |
| App 描述 | ❌ | 不可测试 |
| 关键词 | ❌ | 不可测试 |
| App 预览 | ❌ | 不可测试 |
| 评分 | ❌ | 不可测试 |

### 5.3 创建 A/B 测试

**步骤：**

1. 登录 App Store Connect
2. 选择「我的 App」→ 选择你的 App
3. 在左侧菜单中选择「产品页优化」
4. 点击「创建测试」
5. 选择要测试的元素
6. 创建变体（最多 3 个，含对照组）
7. 设置流量分配
8. 启动测试

**流量分配：**

| 分配方式 | 说明 |
|----------|------|
| 均匀分配 | 每个变体获得相同流量 |
| 自定义分配 | 手动设置每个变体的流量比例 |

**流量分配示例：**

```
对照组（原始版本）：40%
变体 A（新截图）：30%
变体 B（新图标）：30%
```

### 5.4 测试结果分析

Apple 会自动计算每个变体的转化率，并标注统计显著性。

**关键指标：**

| 指标 | 说明 |
|------|------|
| 展示次数 | 产品页被查看的次数 |
| 下载次数 | 用户点击下载的次数 |
| 转化率 | 下载次数 / 展示次数 |
| 提升幅度 | 变体相对于对照组的提升百分比 |
| 置信度 | 统计显著性水平 |

**结果判定：**

| 置信度 | 判定 |
|--------|------|
| < 90% | 结果不显著，继续测试 |
| 90% - 95% | 结果可能显著 |
| > 95% | 结果显著，可以采纳 |

### 5.5 应用测试结果

当测试结果显著时：

1. 选择表现最好的变体
2. 点击「应用」
3. 该变体替换默认产品页
4. 其他变体自动停止

**注意事项：**

| 注意 | 说明 |
|------|------|
| 测试时长 | 建议至少运行 7 天 |
| 流量要求 | 每个变体至少需要 100 次展示 |
| 同时测试 | 同一时间只能运行一个测试 |
| 审核要求 | 变体内容需要通过审核 |
| 版本限制 | 每个版本最多 5 个测试 |

### 5.6 A/B 测试最佳实践

| 实践 | 说明 |
|------|------|
| 一次只测一个变量 | 同时修改截图和图标无法判断哪个因素起作用 |
| 测试时长足够 | 至少 7 天，覆盖完整周期 |
| 避免节假日干扰 | 节假日流量异常，影响测试结果 |
| 对照组保留 | 始终保留原始版本作为对照 |
| 迭代测试 | 第一轮找方向，第二轮优化细节 |

---

## 6. 活动和自定义页的数据分析

### 6.1 App Store Connect 数据

**In-App Events 数据：**

| 指标 | 说明 |
|------|------|
| 活动展示次数 | 活动在 App Store 中的展示次数 |
| 活动点击次数 | 用户点击活动的次数 |
| 活动转化率 | 点击后下载 App 的比例 |
| 通知订阅数 | 订阅活动通知的用户数 |
| 通知打开率 | 活动通知的打开率 |

**自定义产品页数据：**

| 指标 | 说明 |
|------|------|
| 页面展示次数 | 自定义页被查看的次数 |
| 下载次数 | 通过自定义页下载的次数 |
| 转化率 | 下载次数 / 展示次数 |
| 来源渠道 | 流量来源分布 |

### 6.2 数据查看方式

**在 App Store Connect 中：**

1. 选择「我的 App」→ 选择 App
2. 「度量」标签 → 查看各项指标
3. 「App 内活动」→ 查看活动数据
4. 「自定义产品页」→ 查看页面数据

**使用 API 获取数据：**

```bash
# 获取 App 分析数据
curl -H "Authorization: Bearer $JWT_TOKEN" \
    "https://api.appstoreconnect.apple.com/v1/apps/$APP_ID/appEvents"
```

### 6.3 数据分析维度

| 维度 | 说明 | 用途 |
|------|------|------|
| 时间维度 | 按日/周/月查看趋势 | 发现周期性规律 |
| 地区维度 | 按国家/地区查看 | 优化本地化策略 |
| 来源维度 | 按流量来源查看 | 优化获客渠道 |
| 设备维度 | 按设备类型查看 | 优化截图尺寸 |
| 活动维度 | 按活动类型查看 | 找出最有效的活动类型 |

### 6.4 数据驱动优化

**优化闭环：**

```
数据收集 → 数据分析 → 提出假设 → A/B 测试 → 验证假设 → 应用结果
    ↑                                                        │
    └────────────────────────────────────────────────────────┘
```

**常见优化场景：**

| 场景 | 分析方法 | 优化方向 |
|------|----------|----------|
| 转化率低 | 分析截图和文案 | 优化截图顺序和文案 |
| 活动点击率低 | 分析活动类型和图片 | 更换活动类型或图片 |
| 渠道效果差 | 分析来源数据 | 调整渠道投放策略 |
| 地区差异大 | 分析地区数据 | 优化本地化内容 |

---

## 7. 营销策略

### 7.1 活动日历规划

建议提前规划全年活动日历，确保活动节奏合理：

| 月份 | 活动建议 | 活动类型 |
|------|----------|----------|
| 1 月 | 元旦、春节预热 | 特别活动 |
| 2 月 | 春节活动 | 特别活动 / 促销 |
| 3 月 | 女神节 | 促销 |
| 4 月 | 春季更新 | 重大更新 |
| 5 月 | 劳动节 | 促销 |
| 6 月 | 618、儿童节 | 促销 / 特别活动 |
| 7 月 | 暑期活动 | 新季内容 |
| 8 月 | 暑期活动延续 | 挑战 |
| 9 月 | 秋季更新、开学季 | 重大更新 / 新季内容 |
| 10 月 | 国庆节 | 特别活动 |
| 11 月 | 双 11 | 促销 |
| 12 月 | 双 12、圣诞、跨年 | 促销 / 特别活动 |

### 7.2 节日营销策略

**节日活动设计要点：**

| 要点 | 说明 |
|------|------|
| 提前准备 | 至少提前 2 周创建活动 |
| 节日元素 | 使用节日主题的图片和文案 |
| 限时优惠 | 节日限定优惠增加紧迫感 |
| 深链接直达 | 确保用户点击后直达活动页面 |
| 多语言 | 针对不同地区使用不同节日元素 |

**节日活动示例（春节）：**

| 元素 | 内容 |
|------|------|
| 活动类型 | 特别活动 |
| 活动名称 | 春节限定：集福赢大奖 |
| 简短描述 | 集齐五福，赢取新年限定奖励 |
| 详细描述 | 参与春节集福活动，每日登录领取福卡，集齐五福即可兑换限定皮肤和 888 金币！活动持续 7 天，不要错过！ |
| 深链接 | myapp://event/spring-festival-2024 |
| 图片 | 红色主题，福字元素，App 角色 |

### 7.3 版本更新活动

每次重大版本更新都应配合活动：

| 版本类型 | 活动策略 | 活动类型 |
|----------|----------|----------|
| 大版本更新 | 创建活动展示新功能 | 重大更新 |
| 赛季更新 | 创建新赛季活动 | 新季内容 |
| 功能更新 | 突出核心新功能 | 重大更新 |
| 修复更新 | 一般不需要活动 | - |

**版本更新活动示例：**

| 元素 | 内容 |
|------|------|
| 活动类型 | 重大更新 |
| 活动名称 | 3.0 全新上线：AI 智能助手 |
| 简短描述 | 全新 AI 助手功能，智能帮你规划每一天 |
| 详细描述 | 我们带来了全新的 AI 智能助手！自动分析你的习惯，智能推荐最优方案。更新即可免费体验 7 天！ |
| 深链接 | myapp://update/v3-ai-assistant |

### 7.4 活动与自定义产品页配合

**组合策略：**

```
场景：游戏 App 推出新赛季

1. 创建"新赛季"活动（In-App Event）
   → 在 App Store 中展示新赛季信息

2. 创建"游戏玩家"自定义产品页
   → 截图展示新赛季内容
   → 文案突出竞技体验

3. 创建"休闲玩家"自定义产品页
   → 截图展示休闲玩法
   → 文案突出轻松娱乐

4. 不同渠道使用不同产品页 URL
   → 游戏论坛链接到"游戏玩家"页
   → 休闲社区链接到"休闲玩家"页
```

### 7.5 活动与搜索广告配合

Apple Search Ads 支持自定义产品页：

| 配合方式 | 说明 |
|----------|------|
| 搜索广告 + 自定义页 | 广告展示自定义产品页 |
| 搜索广告 + 活动 | 广告配合活动时间投放 |
| 搜索广告 + PPO | 先测试再投放广告 |

**搜索广告自定义产品页设置：**

1. 在 Apple Search Ads 后台创建广告组
2. 选择「自定义产品页」
3. 选择要使用的变体
4. 设置关键词和出价

---

## 8. 实战：为一个 App 创建活动和完善产品页

### 8.1 实战场景

假设我们有一个健身 App「FitLife」，需要：

1. 为春节创建一个限时活动
2. 创建两个自定义产品页（跑步用户 / 瑜伽用户）
3. 对截图进行 A/B 测试

### 8.2 创建春节活动

**活动信息：**

| 字段 | 值 |
|------|-----|
| 参考名称 | 2024 春节健身挑战 |
| 活动类型 | 挑战 |
| 优先级 | 高优先级 |
| 开始时间 | 2024-02-10 00:00 |
| 结束时间 | 2024-02-17 00:00 |

**本地化信息（中文）：**

| 字段 | 值 |
|------|-----|
| 活动名称 | 春节燃脂挑战赛 |
| 简短描述 | 7天打卡，赢取新年限定徽章 |
| 详细描述 | 春节不停歇！连续7天完成每日运动目标，即可获得新年限定徽章和7天VIP体验卡。快来和万千用户一起，用运动开启新一年！ |
| 深链接 | fitlife://event/spring-challenge-2024 |

**本地化信息（英文）：**

| 字段 | 值 |
|------|-----|
| 活动名称 | Spring Fitness Challenge |
| 简短描述 | 7-day streak for exclusive badge |
| 详细描述 | Keep moving this Spring Festival! Complete daily fitness goals for 7 consecutive days to earn an exclusive badge and 7-day VIP trial. Join thousands of users and start the new year with fitness! |
| 深链接 | fitlife://event/spring-challenge-2024 |

**活动图片设计要点：**

| 要点 | 说明 |
|------|------|
| 主色调 | 红色 + 金色（春节氛围） |
| 核心元素 | 奔跑的人形 + 烟花/灯笼 |
| 文字 | "7天挑战" 大字 |
| 风格 | 与 App 整体风格一致 |

### 8.3 创建跑步用户自定义产品页

**变体名称：** Runner Page

**截图策略：**

| 截图顺序 | 内容 |
|----------|------|
| 截图 1 | 跑步记录界面，展示 GPS 轨迹 |
| 截图 2 | 跑步数据统计，展示配速和距离 |
| 截图 3 | 跑步训练计划 |
| 截图 4 | 跑步社区/排行榜 |
| 截图 5 | 跑步音乐播放 |

**文案策略：**

| 字段 | 内容 |
|------|------|
| 推广文案 | GPS 精准追踪，让每一步都有意义 |
| 描述 | FitLife 是你的专业跑步伙伴。精准 GPS 追踪记录你的每一次跑步，智能配速分析帮你突破 PB，海量训练计划从 5K 到全马全覆盖。加入百万跑者社区，一起跑出精彩！ |
| 关键词 | 跑步,GPS,配速,马拉松,5K,训练计划,跑步记录 |

### 8.4 创建瑜伽用户自定义产品页

**变体名称：** Yoga Page

**截图策略：**

| 截图顺序 | 内容 |
|----------|------|
| 截图 1 | 瑜伽课程列表 |
| 截图 2 | 瑜伽跟练界面 |
| 截图 3 | 冥想功能 |
| 截图 4 | 瑜伽日历/打卡 |
| 截图 5 | 身心数据统计 |

**文案策略：**

| 字段 | 内容 |
|------|------|
| 推广文案 | 从内到外的身心修行，从今天开始 |
| 描述 | FitLife 带你走进瑜伽的世界。数百节专业瑜伽课程，从入门到进阶全覆盖。冥想引导帮你释放压力，瑜伽日历记录你的修行之路。在繁忙生活中找到内心的平静。 |
| 关键词 | 瑜伽,冥想,身心,放松,拉伸,课程,入门 |

### 8.5 设置 A/B 测试

**测试目标：** 找出转化率最高的截图风格

**测试变量：** 截图第一张（功能展示 vs 情感化场景）

| 变体 | 截图 1 内容 | 预期效果 |
|------|-------------|----------|
| 对照组 | 功能展示：跑步记录界面 | 吸引理性用户 |
| 变体 A | 情感场景：晨跑中的用户 + 日出 | 吸引感性用户 |

**测试设置：**

| 设置 | 值 |
|------|-----|
| 测试元素 | 截图 |
| 流量分配 | 对照组 50%，变体 A 50% |
| 测试时长 | 14 天 |
| 成功指标 | 转化率提升 > 5% |

### 8.6 App 内深链接处理

```swift
final class DeepLinkHandler {
    static let shared = DeepLinkHandler()

    enum Destination {
        case event(String)
        case challenge(String)
        case workout(String)
    }

    func handle(_ url: URL) -> Destination? {
        guard let scheme = url.scheme, scheme == "fitlife" else { return nil }
        guard let host = url.host else { return nil }

        switch host {
        case "event":
            if let eventId = url.pathComponents.last {
                return .event(eventId)
            }
        case "challenge":
            if let challengeId = url.pathComponents.last {
                return .challenge(challengeId)
            }
        case "workout":
            if let workoutId = url.pathComponents.last {
                return .workout(workoutId)
            }
        default:
            break
        }
        return nil
    }
}
```

### 8.7 活动效果追踪

```swift
import FirebaseAnalytics

final class EventTracker {
    static let shared = EventTracker()

    func trackEventView(eventId: String, source: String) {
        Analytics.logEvent("in_app_event_viewed", parameters: [
            "event_id": eventId,
            "source": source
        ])
    }

    func trackEventJoin(eventId: String) {
        Analytics.logEvent("in_app_event_joined", parameters: [
            "event_id": eventId
        ])
    }

    func trackEventComplete(eventId: String, reward: String) {
        Analytics.logEvent("in_app_event_completed", parameters: [
            "event_id": eventId,
            "reward": reward
        ])
    }

    func trackCustomPageView(pageId: String, source: String) {
        Analytics.logEvent("custom_product_page_viewed", parameters: [
            "page_id": pageId,
            "source": source
        ])
    }
}
```

---

## 9. 高级技巧和注意事项

### 9.1 活动与推送通知配合

用户可以订阅活动通知，当活动开始时收到推送：

```swift
import UserNotifications

func requestNotificationPermission() {
    UNUserNotificationCenter.current().requestAuthorization(options: [.alert, .sound]) { granted, error in
        if granted {
            print("通知权限已获取")
        }
    }
}
```

**活动通知最佳实践：**

| 实践 | 说明 |
|------|------|
| 提前通知 | 活动开始前 1-2 天发送提醒 |
| 即时通知 | 活动开始时发送通知 |
| 进度通知 | 活动进行中发送进度提醒 |
| 截止通知 | 活动即将结束时发送提醒 |

### 9.2 活动与 StoreKit 配合

```swift
import StoreKit

// iOS 16+ 展示 App 内活动
if let windowScene = UIApplication.shared.connectedScenes.first as? UIWindowScene {
    Task {
        do {
            let appStoreEvents = try await AppStore.getEvents(in: windowScene)
            for event in appStoreEvents {
                print("活动: \(event.title), 状态: \(event.status)")
            }
        } catch {
            print("获取活动失败: \(error)")
        }
    }
}
```

### 9.3 自定义产品页的 SEO 优化

虽然自定义产品页的关键词不直接影响 ASO，但合理的文案可以提升转化率：

| 优化项 | 说明 |
|--------|------|
| 推广文案 | 前 30 字符最重要（搜索结果中可见） |
| 截图顺序 | 第一张截图决定用户是否继续浏览 |
| 截图文字 | 截图上的文字要简洁有力 |
| 本地化 | 为每个市场提供本地化内容 |

### 9.4 活动素材管理

建议建立素材管理系统：

```
素材目录结构：
├── events/
│   ├── 2024-spring/
│   │   ├── image_zh_CN.png
│   │   ├── image_en_US.png
│   │   └── metadata.json
│   ├── 2024-summer/
│   │   ├── image_zh_CN.png
│   │   ├── image_en_US.png
│   │   └── metadata.json
│   └── ...
├── custom_pages/
│   ├── runner/
│   │   ├── screenshots/
│   │   └── metadata.json
│   ├── yoga/
│   │   ├── screenshots/
│   │   └── metadata.json
│   └── ...
└── templates/
    ├── event_image_template.psd
    └── screenshot_template.psd
```

### 9.5 常见错误和避坑

| 错误 | 说明 | 避免方法 |
|------|------|----------|
| 活动时间太长 | 超过 31 天限制 | 规划好活动周期 |
| 图片不符合规范 | 尺寸或格式错误 | 严格按照 1024x1024 制作 |
| 深链接失效 | 活动上线但深链接未准备好 | 提前测试深链接 |
| 审核时间不足 | 活动快开始了才提交 | 提前 5-7 天提交 |
| 自定义页太多 | 超过 35 个限制 | 定期清理不用的变体 |
| A/B 测试时间太短 | 数据不显著 | 至少运行 7 天 |
| 活动类型选错 | 类型与实际不符 | 仔细阅读类型说明 |

---

## 10. 与 App Store Connect API 的配合

### 10.1 通过 API 管理活动

```bash
# 获取 App 内活动列表
curl -H "Authorization: Bearer $JWT_TOKEN" \
    "https://api.appstoreconnect.apple.com/v1/apps/$APP_ID/appEvents"

# 创建新活动
curl -X POST \
    -H "Authorization: Bearer $JWT_TOKEN" \
    -H "Content-Type: application/json" \
    -d '{
        "data": {
            "type": "appEvents",
            "attributes": {
                "referenceName": "Spring Challenge 2024",
                "primaryLocale": "zh-Hans",
                "eventType": "CHALLENGE",
                "priority": "HIGH",
                "startDate": "2024-02-10T00:00:00+08:00",
                "endDate": "2024-02-17T00:00:00+08:00"
            }
        }
    }' \
    "https://api.appstoreconnect.apple.com/v1/apps/$APP_ID/appEvents"
```

### 10.2 通过 API 管理自定义产品页

```bash
# 获取自定义产品页列表
curl -H "Authorization: Bearer $JWT_TOKEN" \
    "https://api.appstoreconnect.apple.com/v1/apps/$APP_ID/appCustomProductPages"

# 创建自定义产品页
curl -X POST \
    -H "Authorization: Bearer $JWT_TOKEN" \
    -H "Content-Type: application/json" \
    -d '{
        "data": {
            "type": "appCustomProductPages",
            "attributes": {
                "name": "Runner Page",
                "url": "runner-page"
            }
        }
    }' \
    "https://api.appstoreconnect.apple.com/v1/apps/$APP_ID/appCustomProductPages"
```

💡 **提示**：通过 API 管理活动和自定义产品页可以实现自动化，特别是对于需要频繁创建活动的 App。

---

## 本章小结

| 主题 | 核心要点 |
|------|----------|
| In-App Events | 限时活动展示在 App Store，8 种活动类型，提升获客和留存 |
| 创建活动 | 准备图片+深链接+文案，在 App Store Connect 中创建，需审核 |
| 审核发布 | 活动需审核通过，建议提前 5-7 天提交，开始后不可修改 |
| 自定义产品页 | 最多 35 个变体，不同截图和文案针对不同用户群 |
| A/B 测试 | PPO 测试截图/图标/推广文案，至少 7 天，一次只测一个变量 |
| 数据分析 | 关注展示/点击/转化率，按时间/地区/来源/设备维度分析 |
| 营销策略 | 规划活动日历，节日营销+版本更新+渠道配合 |
| 实战 | 健身 App 案例：春节活动+跑步/瑜伽自定义页+截图 A/B 测试 |

> 💡 一句话总结：App 内活动和自定义产品页是 App Store 给开发者的"营销武器"——用好它们，让你的 App 在海量应用中脱颖而出。

← [国内服务器部署与网站备案](./国内服务器部署与网站备案.md) | [App Store Connect API自动化](./App-Store-Connect-API自动化.md) →
