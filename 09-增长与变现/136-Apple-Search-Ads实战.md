# 136-Apple Search Ads实战

> 🎯 **本章目标**：掌握 Apple Search Ads 的投放和优化方法，学会创建有效的搜索广告 campaign，通过数据驱动提升获客效率。

---

## Apple Search Ads 概述

### 什么是 Apple Search Ads

Apple Search Ads（ASA）是 Apple 官方提供的搜索广告平台，允许开发者在 App Store 搜索结果中投放广告。当用户在 App Store 搜索关键词时，你的广告会出现在搜索结果的第一位。

**广告展示位置：**

```text
┌─────────────────────────────────────────┐
│  🔍 搜索：记账                           │
│                                         │
│  ┌─────────────────────────────────┐    │
│  │  📱 广告  随手记 - 记账理财        │    │  ← 搜索结果广告（Search Tab）
│  │  智能记账，轻松理财               │    │
│  │  4.8 ★  12.3K 评分    广告       │    │
│  └─────────────────────────────────┘    │
│                                         │
│  ┌─────────────────────────────────┐    │
│  │  📱 网易有钱 - 记账理财           │    │  ← 自然搜索结果
│  │  全自动记账，资产管理             │    │
│  │  4.6 ★  8.7K 评分               │    │
│  └─────────────────────────────────┘    │
│                                         │
│  ┌─────────────────────────────────┐    │
│  │  📱 挖财记账                     │    │  ← 自然搜索结果
│  │  个人记账理财助手                │    │
│  │  4.5 ★  6.2K 评分               │    │
│  └─────────────────────────────────┘    │
└─────────────────────────────────────────┘
```

### 为什么选择 Apple Search Ads

| 优势 | 说明 |
|------|------|
| **高意图用户** | 搜索用户有明确的下载意图，转化率远高于其他广告 |
| **官方平台** | Apple 官方产品，审核严格，流量质量高 |
| **精准匹配** | 基于用户搜索词匹配，相关性高 |
| **透明计费** | 按 TTR（点击率）计费，CPT（每千次点击成本）可控 |
| **隐私保护** | Apple 不追踪用户，但提供足够的投放数据 |
| **品牌保护** | 品牌词投放可防止竞品抢夺你的搜索流量 |

**Apple Search Ads 的核心数据：**

| 指标 | 数据 |
|------|------|
| 平均转化率（TTR） | 约 50%（搜索结果广告） |
| 平均 CPT | $0.5-$5.0（因类别而异） |
| 用户获取占比 | App Store 中约 65% 用户通过搜索发现 App |
| 广告覆盖率 | 覆盖所有 App Store 市场 |

### Basic vs Advanced

Apple Search Ads 提供两种版本：

| 对比维度 | Basic | Advanced |
|----------|:------:|:--------:|
| **月预算** | ≤ $10,000 | 无上限 |
| **关键词控制** | ❌ 自动匹配 | ✅ 手动选择 |
| **受众定位** | ❌ 自动 | ✅ 自定义 |
| **出价控制** | ❌ 自动 | ✅ 手动出价 |
| **Creative Sets** | ❌ | ✅ |
| **广告位选择** | ❌ | ✅ 搜索结果/推荐 |
| **详细报告** | ❌ 基础 | ✅ 完整 |
| **API 访问** | ❌ | ✅ |
| **适合对象** | 小型开发者 | 专业投放团队 |
| **最低预算** | 无 | 无 |

> 💡 **提示**：如果你刚开始接触 Apple Search Ads，建议先从 Basic 开始试水，了解基本效果后再升级到 Advanced。但如果你有专业的投放需求，直接使用 Advanced 可以获得更好的控制力和 ROI。

---

## 账户创建和付款设置

### 创建 Apple Search Ads 账户

| 步骤 | 操作 | 说明 |
|:----:|------|------|
| 1 | 访问 searchads.apple.com | Apple Search Ads 官网 |
| 2 | 点击"开始使用" | 选择 Advanced 或 Basic |
| 3 | 使用 Apple ID 登录 | 建议使用公司 Apple ID |
| 4 | 填写账户信息 | 公司名称、地址等 |
| 5 | 选择付款方式 | 信用卡或发票付款 |
| 6 | 设置账户预算 | 月度预算上限 |

### 账户结构

```text
Apple Search Ads 账户
├── Organization（组织）
│   ├── Payment Info（付款信息）
│   └── User Roles（用户角色）
│
├── Campaign Group（广告系列组）
│   └── Campaign（广告系列）
│       ├── Ad Group（广告组）
│       │   ├── Keywords（关键词）
│       │   ├── Audience（受众）
│       │   ├── Creative Sets（广告素材）
│       │   └── Bidding（出价）
│       │
│       └── Ad Group 2
│           └── ...
│
└── Campaign Group 2
    └── ...
```

### 付款设置

| 付款方式 | 说明 | 适用场景 |
|----------|------|----------|
| 信用卡 | Visa、MasterCard、American Express | 小型投放 |
| 发票付款 | 月度结算，需要信用审批 | 大型投放（月消费 > $5,000） |
| 预付余额 | 预先充值 | 控制预算 |

**付款注意事项：**

| 注意事项 | 说明 |
|----------|------|
| 货币 | 默认 USD，部分市场支持本地货币 |
| 账单周期 | 每月结算，或达到信用额度时结算 |
| 最低充值 | 无最低要求 |
| 退款政策 | 广告费一般不退款 |

> ⚠️ **警告**：国内开发者需要注意，Apple Search Ads 目前不支持人民币直接付款，需要使用支持美元支付的信用卡。建议使用双币信用卡或企业信用卡。

### 用户权限管理

| 角色 | 权限 | 适用人员 |
|------|------|----------|
| Admin | 全部权限 | 管理者 |
| Account Manager | 创建/编辑 Campaign | 投放经理 |
| Campaign Group Manager | 管理特定 Campaign Group | 投放专员 |
| Read Only | 只能查看数据 | 数据分析师 |

---

## Campaign 创建：预算、地区、受众

### 创建 Campaign 的步骤

| 步骤 | 操作 | 说明 |
|:----:|------|------|
| 1 | 点击"创建 Campaign" | 在 Campaigns 页面 |
| 2 | 选择 App | 选择要推广的 App |
| 3 | 设置 Campaign 名称 | 命名规范：日期_类型_关键词 |
| 4 | 选择广告位 | 搜索结果 / 推荐 |
| 5 | 设置预算 | 日预算和总预算 |
| 6 | 选择地区 | 投放的国家/地区 |
| 7 | 创建 Ad Group | 设置关键词和出价 |

### Campaign 命名规范

好的命名规范可以帮助你快速识别和管理 Campaign：

```text
命名格式：[日期]_[类型]_[关键词类别]_[地区]

示例：
202605_品牌词_品牌名_US
202605_竞品词_竞品A_JP
202605_通用词_记账_CN
202605_探索_自动匹配_GLOBAL
```

### 预算设置

| 预算类型 | 说明 | 建议 |
|----------|------|------|
| 日预算（Daily Budget） | 每天最多花费的金额 | 初期设置较低，逐步增加 |
| 总预算（Lifetime Budget） | Campaign 总花费上限 | 长期 Campaign 使用 |
| 无上限 | 不设预算限制 | ❌ 不推荐 |

**预算分配策略：**

| 策略 | 说明 | 适用场景 |
|------|------|----------|
| 小预算测试 | 每天几十美元测试效果 | 新手入门 |
| 逐步放量 | 测试有效后逐步增加预算 | 稳健投放 |
| 集中投放 | 短期内大量投放 | 新版本上线、促销活动 |
| 长期投放 | 持续低预算投放 | 品牌词保护 |

### 地区选择

| 地区 | 平均 CPT | 竞争程度 | 建议 |
|------|:--------:|:--------:|------|
| 美国（US） | $1.5-$5.0 | 🔴 高 | 预算充足时优先 |
| 日本（JP） | $1.0-$3.0 | 🟠 中高 | 游戏类效果好 |
| 英国（UK） | $1.0-$3.0 | 🟠 中高 | 英语市场补充 |
| 德国（DE） | $0.8-$2.5 | 🟡 中 | 欧洲主力市场 |
| 法国（FR） | $0.5-$2.0 | 🟡 中 | 法语市场 |
| 韩国（KR） | $0.5-$2.0 | 🟡 中 | 亚洲补充市场 |
| 澳大利亚（AU） | $0.5-$1.5 | 🟢 低 | 测试市场 |
| 加拿大（CA） | $0.5-$1.5 | 🟢 低 | 测试市场 |

> 💡 **提示**：建议先用低 CPT 的市场（如澳大利亚、加拿大）进行测试，验证关键词和素材效果后再扩展到高 CPT 市场（如美国、日本）。

### 受众定位

Advanced 版本支持精细的受众定位：

| 定位维度 | 选项 | 说明 |
|----------|------|------|
| **用户类型** | 所有用户 / 新用户 / 回归用户 | 新用户首次下载，回归用户曾下载过 |
| **地理位置** | 国家/地区/城市 | 精确到城市级别 |
| **设备类型** | iPhone / iPad | 分别投放 |
| **年龄段** | 18-24 / 25-34 / 35-44 / 45-54 / 55+ | 按年龄分群 |
| **性别** | 男 / 女 / 全部 | 部分类别效果明显 |
| **App 使用行为** | 已安装特定 App 的用户 | 竞品用户定向 |

**受众定位策略：**

```text
┌──────────────────────────────────────────────────────┐
│                  受众定位策略                          │
│                                                      │
│  品牌词 Campaign：                                    │
│  ├── 用户类型：全部（保护品牌流量）                     │
│  ├── 设备：全部                                       │
│  └── 其他：不限                                       │
│                                                      │
│  竞品词 Campaign：                                    │
│  ├── 用户类型：新用户（抢夺竞品用户）                   │
│  ├── 设备：根据 App 适配情况选择                       │
│  └── 年龄/性别：根据目标用户画像选择                    │
│                                                      │
│  通用词 Campaign：                                    │
│  ├── 用户类型：新用户                                  │
│  ├── 设备：全部                                       │
│  └── 年龄/性别：根据产品定位选择                       │
└──────────────────────────────────────────────────────┘
```

---

## Ad Group 设置：关键词、匹配类型、出价

### Ad Group 的作用

Ad Group 是 Campaign 下的子单元，用于管理一组相关的关键词和出价策略：

```text
Campaign: 202605_品牌词_记账App_US
├── Ad Group 1: 品牌核心词
│   ├── 关键词: 记账App, 记账软件
│   ├── 出价: $1.50
│   └── 受众: 全部用户
│
├── Ad Group 2: 品牌长尾词
│   ├── 关键词: 最好用的记账App, 免费记账软件
│   ├── 出价: $0.80
│   └── 受众: 新用户
│
└── Ad Group 3: 品牌保护
    ├── 关键词: 我App的名称
    ├── 出价: $2.00
    └── 受众: 全部用户
```

### 关键词匹配类型

Apple Search Ads 支持两种关键词匹配类型：

| 匹配类型 | 说明 | 优点 | 缺点 |
|----------|------|------|------|
| **Exact Match（精确匹配）** | 只匹配完全相同的关键词 | 精准控制，CPT 低 | 覆盖面窄 |
| **Broad Match（广泛匹配）** | 匹配相关变体和近义词 | 覆盖面广，发现新词 | CPT 较高，可能不相关 |

**匹配示例：**

| 关键词 | Exact Match 匹配 | Broad Match 匹配 |
|--------|-----------------|-----------------|
| 记账 | ✅ 记账 | ✅ 记账、记账本、记账软件、理财 |
| 照片编辑 | ✅ 照片编辑 | ✅ 照片编辑、图片编辑、修图、P图 |
| 冥想 | ✅ 冥想 | ✅ 冥想、冥想App、打坐、正念 |

> 💡 **提示**：建议用 Exact Match 投放核心关键词（品牌词、高转化词），用 Broad Match 发现新关键词。Broad Match 发现的高转化词，再转为 Exact Match 精细投放。

### 出价策略

| 出价策略 | 说明 | 适用场景 |
|----------|------|----------|
| 最高 CPT 出价 | 设置你愿意为每次点击支付的最高金额 | 大多数场景 |
| 目标 CPA 出价 | 设置你愿意为每次安装支付的目标成本 | 有明确 CPA 目标时 |

**CPT 出价建议：**

| 关键词类型 | 建议出价 | 说明 |
|------------|:--------:|------|
| 品牌词 | $0.5-$1.5 | 竞争低，转化率高 |
| 竞品词 | $1.0-$3.0 | 竞争中等，转化率中等 |
| 通用词 | $0.5-$2.0 | 竞争不一，需要测试 |
| 高竞争词 | $2.0-$5.0 | 热门词，需要高出价 |

**出价调整原则：**

```text
┌──────────────────────────────────────────────────┐
│               出价调整决策树                       │
│                                                  │
│  转化率高 + 展示量少 ──► 提高出价                  │
│  转化率高 + 展示量多 ──► 维持出价                  │
│  转化率低 + 点击率低 ──► 降低出价或暂停             │
│  转化率低 + 点击率高 ──► 优化素材/关键词            │
│  超出预算 ──► 降低出价或缩小关键词范围              │
└──────────────────────────────────────────────────┘
```

### 否定关键词

否定关键词（Negative Keywords）可以阻止广告在不相关的搜索词上展示：

```swift
struct NegativeKeywordStrategy {
    let irrelevantCategories: [String]
    let competitorNames: [String]
    let freeSeekers: [String]

    static let defaultStrategy = NegativeKeywordStrategy(
        irrelevantCategories: ["免费", "破解", "盗版"],
        competitorNames: [],
        freeSeekers: ["免费下载", "破解版", "去广告"]
    )

    func generateNegativeKeywords() -> [String] {
        var keywords: [String] = []
        keywords.append(contentsOf: irrelevantCategories)
        keywords.append(contentsOf: freeSeekers)
        return keywords
    }
}
```

---

## 搜索结果广告 vs 推荐广告

### 两种广告位对比

Apple Search Ads 提供两种广告展示位置：

| 对比维度 | 搜索结果广告（Search Results） | 推荐广告（Search Tab） |
|----------|:-----------------------------:|:---------------------:|
| **展示位置** | 搜索结果页第一位 | 搜索页"推荐"区域 |
| **触发方式** | 用户搜索特定关键词 | 用户打开搜索页 |
| **用户意图** | 🔴 高（主动搜索） | 🟡 中（浏览发现） |
| **转化率** | 高（约 50%） | 中（约 30%） |
| **CPT** | 较高 | 较低 |
| **关键词** | ✅ 可选关键词 | ❌ 自动匹配 |
| **Creative Sets** | ✅ 支持 | ✅ 支持 |
| **适合场景** | 精准获客 | 品牌曝光、发现新用户 |

### 搜索结果广告详解

搜索结果广告是最核心的广告形式，出现在用户搜索关键词后的第一位：

**优势：**

| 优势 | 说明 |
|------|------|
| 高意图 | 用户主动搜索，下载意愿强 |
| 高转化 | 平均转化率约 50% |
| 可控性强 | 可选择关键词、出价、受众 |
| 效果可衡量 | 完整的数据追踪 |

**最佳实践：**

| 实践 | 说明 |
|------|------|
| 品牌词必投 | 保护品牌搜索流量 |
| 精确匹配优先 | 控制成本，提高转化 |
| 持续优化出价 | 根据数据调整出价 |
| 监控搜索词报告 | 发现高价值新词 |

### 推荐广告详解

推荐广告出现在 App Store 搜索页的"推荐"区域，用户无需输入搜索词即可看到：

**优势：**

| 优势 | 说明 |
|------|------|
| 发现新用户 | 触达未搜索但可能感兴趣的用户 |
| 品牌曝光 | 增加品牌知名度 |
| CPT 较低 | 竞争相对较小 |
| 适合冷启动 | 新 App 获取初始用户 |

**最佳实践：**

| 实践 | 说明 |
|------|------|
| 配合搜索广告 | 搜索 + 推荐组合投放 |
| 优化素材 | 素材质量决定点击率 |
| 设置合理预算 | 避免预算浪费 |
| 关注转化率 | 推荐广告转化率较低，需要监控 |

---

## 关键词策略：品牌词、竞品词、通用词

### 三类关键词

| 类型 | 定义 | 转化率 | CPT | 竞争 | 策略 |
|------|------|:------:|:---:|:----:|------|
| **品牌词** | 你的 App 名称及相关词 | ⭐⭐⭐⭐⭐ | 低 | 低 | 必须投放，保护品牌 |
| **竞品词** | 竞品 App 的名称 | ⭐⭐⭐ | 中高 | 高 | 选择性投放，抢夺用户 |
| **通用词** | 行业/功能相关词 | ⭐⭐ | 中 | 中 | 大量测试，筛选高转化词 |

### 品牌词策略

品牌词是转化率最高、成本最低的关键词，必须优先投放：

| 策略 | 说明 |
|------|------|
| 核心品牌词 | App 名称、公司名称 |
| 品牌变体 | 常见拼写错误、缩写 |
| 品牌词 + 功能 | "XX记账 高级版" |
| 品牌词 + 平台 | "XX iOS" |

**品牌词投放的必要性：**

| 不投品牌词的风险 | 投品牌词的好处 |
|-----------------|---------------|
| 竞品可能投放你的品牌词 | 保护品牌搜索流量 |
| 搜索你品牌的用户可能被竞品抢走 | 转化率极高，CPT 极低 |
| 品牌流量流失 | 占据搜索结果第一位 |

### 竞品词策略

投放竞品词可以抢夺竞品的搜索流量，但成本较高：

| 策略 | 说明 | 注意事项 |
|------|------|----------|
| 直接竞品词 | 同类竞品的 App 名称 | 出价要高于竞品 |
| 竞品词 + 优势 | "比XX更好用的记账" | 素材要突出差异化 |
| 竞品替代词 | "XX替代品" | 转化意图明确 |

**竞品词投放建议：**

| 建议 | 说明 |
|------|------|
| 选择性投放 | 只投 3-5 个核心竞品 |
| 高出价 | 竞品词竞争激烈，需要高出价 |
| 突出差异化 | 素材中强调你的独特优势 |
| 监控转化率 | 竞品词转化率通常较低 |
| 注意法律风险 | 不要在广告文案中直接贬低竞品 |

### 通用词策略

通用词覆盖面广，是发现新用户的重要渠道：

| 通用词类型 | 示例 | 策略 |
|------------|------|------|
| 功能词 | 记账、理财、预算 | 核心通用词，必须投放 |
| 场景词 | 家庭记账、旅行记账 | 精准场景，转化率高 |
| 人群词 | 学生记账、情侣记账 | 特定人群，竞争较小 |
| 长尾词 | 简单好用的记账软件 | 竞争小，CPT 低 |
| 问题词 | 怎么记账、记账方法 | 搜索意图明确 |

**关键词发现方法：**

| 方法 | 说明 | 工具 |
|------|------|------|
| Apple Search Ads 搜索词报告 | 查看实际触发广告的搜索词 | ASA 后台 |
| App Store 搜索建议 | 输入关键词查看相关建议 | App Store |
| 竞品分析 | 分析竞品的关键词布局 | Sensor Tower 等 |
| ASO 工具 | 使用专业 ASO 工具发现关键词 | AppTweak 等 |
| 用户反馈 | 从评论中提取关键词 | 手动分析 |

---

## Creative Sets：自定义广告素材

### 什么是 Creative Sets

Creative Sets 允许你从 App Store 产品页面中选择不同的截图和预览视频组合，为不同的关键词和受众展示不同的广告素材。

**Creative Sets 的作用：**

```text
┌──────────────────────────────────────────────────────┐
│              同一个 App，不同素材                        │
│                                                      │
│  搜索"记账"的用户看到：                                │
│  ┌─────────────────────────────────────┐             │
│  │  📱 记账App - 截图1: 记账界面        │             │
│  │  📱 记账App - 截图2: 统计报表        │             │
│  └─────────────────────────────────────┘             │
│                                                      │
│  搜索"理财"的用户看到：                                │
│  ┌─────────────────────────────────────┐             │
│  │  📱 记账App - 截图3: 理财规划        │             │
│  │  📱 记账App - 截图4: 投资收益        │             │
│  └─────────────────────────────────────┘             │
└──────────────────────────────────────────────────────┘
```

### 创建 Creative Sets

| 步骤 | 操作 | 说明 |
|:----:|------|------|
| 1 | 在 Ad Group 中点击"Creative Sets" | 进入素材管理 |
| 2 | 点击"创建 Creative Set" | 新建素材组合 |
| 3 | 命名 Creative Set | 描述性名称 |
| 4 | 选择截图和视频 | 从 App Store 页面选择 |
| 5 | 保存并关联 | 关联到关键词或受众 |

### Creative Sets 优化策略

| 策略 | 说明 | 效果 |
|------|------|:----:|
| 关键词匹配素材 | 搜索"冥想"展示冥想截图 | ⭐⭐⭐⭐⭐ |
| 受众匹配素材 | 年轻用户展示时尚界面 | ⭐⭐⭐⭐ |
| 功能突出素材 | 每组素材突出一个核心功能 | ⭐⭐⭐⭐ |
| A/B 测试 | 不同素材组合对比效果 | ⭐⭐⭐⭐⭐ |
| 本地化素材 | 不同地区使用本地化截图 | ⭐⭐⭐⭐ |

**Creative Sets 最佳实践：**

| 实践 | 说明 |
|------|------|
| 至少 3 组 Creative Set | 不同关键词/受众对应不同素材 |
| 每组 3-5 张截图 | 覆盖核心卖点 |
| 包含预览视频 | 视频素材点击率更高 |
| 定期更新 | 素材疲劳后效果下降 |
| 数据驱动 | 根据点击率数据优化素材 |

---

## 数据分析：CAC、TTR、CPT、D7 ROI

### 核心指标

| 指标 | 全称 | 计算方式 | 说明 |
|------|------|----------|------|
| **TTR** | Tap-Through Rate | 点击次数 ÷ 展示次数 | 广告点击率 |
| **CPT** | Cost Per Tap | 总花费 ÷ 点击次数 | 每次点击成本 |
| **CPA** | Cost Per Acquisition | 总花费 ÷ 安装次数 | 每次安装成本 |
| **CAC** | Customer Acquisition Cost | 总花费 ÷ 新增付费用户数 | 每个付费用户获取成本 |
| **CR** | Conversion Rate | 安装次数 ÷ 点击次数 | 点击到安装的转化率 |
| **D7 ROI** | Day 7 Return on Investment | 7天内收入 ÷ 7天内广告花费 | 7天投资回报率 |
| **LTV** | Life Time Value | 用户生命周期总价值 | 用户长期价值 |

### 指标之间的关系

```text
展示次数 ──TTR──► 点击次数 ──CR──► 安装次数 ──付费率──► 付费用户数
   │                │                │                │
   │                │                │                │
   └── CPT ◄── 总花费 ──► CPA ◄─────┘──► CAC ◄──────┘
                                              │
                                         D7 ROI = 7天收入 / 7天花费
                                         LTV / CAC > 3 = 健康
```

### 各指标健康范围

| 指标 | 健康范围 | 需要优化 | 危险信号 |
|------|:--------:|:--------:|:--------:|
| TTR | > 8% | 4-8% | < 4% |
| CR | > 40% | 25-40% | < 25% |
| CPT | < $1.5 | $1.5-$3.0 | > $3.0 |
| CPA | < $2.0 | $2.0-$5.0 | > $5.0 |
| D7 ROI | > 50% | 20-50% | < 20% |
| LTV/CAC | > 3 | 1-3 | < 1 |

> 💡 **提示**：以上范围仅供参考，不同类别和地区的 App 差异很大。游戏类 App 的 CPA 可能高达 $5-10，而工具类 App 可能只需 $0.5-2。关键是找到你自己的基准线。

### 数据分析工具

**Apple Search Ads 后台报告：**

| 报告类型 | 内容 | 频率 |
|----------|------|:----:|
| Campaign 报告 | 展示、点击、花费、安装 | 每日 |
| Ad Group 报告 | 各 Ad Group 的详细数据 | 每日 |
| 关键词报告 | 各关键词的表现数据 | 每日 |
| 搜索词报告 | 实际触发广告的搜索词 | 每日 |
| Creative Sets 报告 | 不同素材组合的效果 | 每日 |
| 队列报告 | 用户安装后的行为数据 | 每日 |

**通过 API 获取数据：**

```swift
import Foundation

struct ASAReportingAPI {
    let apiKeyId: String
    let issuerId: String
    let orgId: String
    let baseUrl = "https://api.searchads.apple.com/api/v4"

    func getCampaignReport(campaignId: Int, startDate: String, endDate: String) async throws -> CampaignReport {
        let urlString = "\(baseUrl)/reports/campaigns/\(campaignId)"
        let url = URL(string: urlString)!
        var request = URLRequest(url: url)
        request.httpMethod = "POST"
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")
        request.setValue("Bearer \(try generateJWT())", forHTTPHeaderField: "Authorization")
        request.setValue(orgId, forHTTPHeaderField: "X-AP-Context-Org-ID")

        let body: [String: Any] = [
            "startTime": startDate,
            "endTime": endDate,
            "granularity": "DAILY",
            "selector": [
                "conditions": [],
                "orderBy": [
                    ["field": "impressions", "sortOrder": "DESCENDING"]
                ],
                "pagination": ["offset": 0, "limit": 1000]
            ]
        ]

        request.httpBody = try JSONSerialization.data(withJSONObject: body)
        let (data, _) = try await URLSession.shared.data(for: request)
        return try JSONDecoder().decode(CampaignReport.self, from: data)
    }

    private func generateJWT() throws -> String {
        fatalError("实现 JWT 生成逻辑")
    }
}

struct CampaignReport: Codable {
    let reportingDataResponse: ReportingData
}

struct ReportingData: Codable {
    let row: [ReportRow]?
}

struct ReportRow: Codable {
    let metadata: ReportMetadata?
    let granularities: [GranularityData]?
}

struct ReportMetadata: Codable {
    let campaignId: Int?
    let campaignName: String?
    let adGroupId: Int?
    let adGroupName: String?
    let keywordId: Int?
    let keyword: String?
}

struct GranularityData: Codable {
    let date: String?
    let impressions: Int?
    let taps: Int?
    let installs: Int?
    let spend: Double?
    let ttr: Double?
    let cpt: Double?
    let cpa: Double?
}
```

### 自定义分析仪表板

```swift
struct ASADashboard {
    let reports: [ReportRow]

    var totalImpressions: Int {
        reports.compactMap { $0.granularities?.first?.impressions }.reduce(0, +)
    }

    var totalTaps: Int {
        reports.compactMap { $0.granularities?.first?.taps }.reduce(0, +)
    }

    var totalInstalls: Int {
        reports.compactMap { $0.granularities?.first?.installs }.reduce(0, +)
    }

    var totalSpend: Double {
        reports.compactMap { $0.granularities?.first?.spend }.reduce(0, +)
    }

    var overallTTR: Double {
        guard totalImpressions > 0 else { return 0 }
        return Double(totalTaps) / Double(totalImpressions)
    }

    var overallCPT: Double {
        guard totalTaps > 0 else { return 0 }
        return totalSpend / Double(totalTaps)
    }

    var overallCPA: Double {
        guard totalInstalls > 0 else { return 0 }
        return totalSpend / Double(totalInstalls)
    }

    var overallCR: Double {
        guard totalTaps > 0 else { return 0 }
        return Double(totalInstalls) / Double(totalTaps)
    }

    struct PerformanceSummary {
        let impressions: Int
        let taps: Int
        let installs: Int
        let spend: Double
        let ttr: Double
        let cpt: Double
        let cpa: Double
        let cr: Double
    }

    var summary: PerformanceSummary {
        PerformanceSummary(
            impressions: totalImpressions,
            taps: totalTaps,
            installs: totalInstalls,
            spend: totalSpend,
            ttr: overallTTR,
            cpt: overallCPT,
            cpa: overallCPA,
            cr: overallCR
        )
    }
}
```

---

## 优化策略：出价调整、关键词扩展、否定词

### 出价优化

**出价调整策略：**

| 策略 | 触发条件 | 调整幅度 |
|------|----------|----------|
| 提高出价 | TTR 高但展示量少 | +10-20% |
| 提高出价 | 转化率高于目标 | +5-15% |
| 降低出价 | CPA 高于目标 | -10-20% |
| 降低出价 | TTR 持续下降 | -5-15% |
| 暂停关键词 | 连续 7 天无转化 | 暂停 |
| 暂停关键词 | CPA 超过目标 2 倍 | 暂停 |

**出价调整频率：**

| 阶段 | 调整频率 | 说明 |
|------|:--------:|------|
| 冷启动期（1-2 周） | 每 2-3 天 | 数据不稳定，频繁调整 |
| 优化期（3-4 周） | 每周 | 数据趋于稳定 |
| 稳定期 | 每 2 周 | 小幅微调 |

### 关键词扩展

**关键词扩展流程：**

```text
┌──────────────┐
│  初始关键词    │  品牌词 + 5-10 个通用词
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  搜索词报告    │  查看实际触发的搜索词
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  筛选高转化词  │  TTR > 8%, CR > 40%
└──────┬───────┘
       │
       ├──► 添加为 Exact Match 关键词
       │
       └──► 添加为 Broad Match 关键词（测试更多变体）
```

**关键词扩展方法：**

| 方法 | 说明 | 工具 |
|------|------|------|
| 搜索词报告 | 最直接的方法 | ASA 后台 |
| App Store 搜索建议 | 输入核心词看建议 | App Store |
| 竞品关键词 | 分析竞品投放的词 | ASO 工具 |
| 相关词扩展 | 从核心词扩展相关词 | 关键词工具 |
| 用户语言 | 从评论中提取关键词 | 手动分析 |

### 否定词优化

否定词可以避免广告在不相关的搜索词上展示，节省预算：

**否定词添加规则：**

| 规则 | 说明 | 示例 |
|------|------|------|
| 不相关词 | 与 App 功能无关的词 | 记账 App 否定"游戏" |
| 免费寻求者 | 搜索免费版的用户 | 否定"免费"、"破解" |
| 低转化词 | 有展示但无转化的词 | 搜索词报告中发现 |
| 竞品无关词 | 不想竞争的竞品词 | 选择性否定 |

**否定词管理：**

```swift
struct NegativeKeywordManager {
    var globalNegatives: Set<String> = []
    var campaignNegatives: [String: Set<String>] = [:]

    mutating func addGlobalNegative(_ keyword: String) {
        globalNegatives.insert(keyword.lowercased())
    }

    mutating func addCampaignNegative(_ keyword: String, for campaignId: String) {
        campaignNegatives[campaignId, default: []].insert(keyword.lowercased())
    }

    func shouldBlock(keyword: String, in campaignId: String) -> Bool {
        let lowerKeyword = keyword.lowercased()
        if globalNegatives.contains(lowerKeyword) { return true }
        if let campaignSet = campaignNegatives[campaignId],
           campaignSet.contains(lowerKeyword) { return true }
        return false
    }

    static let recommendedFreeSeekerNegatives: Set<String> = [
        "免费", "破解", "盗版", "去广告", "无限金币",
        "free", "crack", "hack", "mod", "cheat"
    ]
}
```

---

## Apple Search Ads 与 ASO 的配合

### ASA 和 ASO 的关系

| 维度 | ASO | ASA |
|------|-----|-----|
| **本质** | 优化自然排名 | 付费获取流量 |
| **成本** | 时间成本为主 | 直接广告花费 |
| **见效速度** | 1-4 周 | 立即 |
| **持续性** | 长期有效 | 停止投放即停止 |
| **关键词** | 优化标题/副标题/关键词字段 | 投放关键词广告 |
| **排名影响** | 直接提升自然排名 | 间接影响（下载量信号） |

### ASA + ASO 协同策略

```text
┌──────────────────────────────────────────────────────┐
│                ASA + ASO 协同策略                      │
│                                                      │
│  第一步：ASA 测试关键词                                │
│  ├── 投放大量关键词                                    │
│  ├── 收集转化数据                                      │
│  └── 筛选高转化关键词                                  │
│                                                      │
│  第二步：ASO 优化关键词                                │
│  ├── 将高转化词加入标题/副标题                          │
│  ├── 优化关键词字段                                    │
│  └── 提升自然排名                                      │
│                                                      │
│  第三步：ASA 聚焦高价值词                              │
│  ├── 减少已自然排名靠前的词的投放                       │
│  ├── 加大竞争激烈词的投放                              │
│  └── 持续发现新关键词                                  │
└──────────────────────────────────────────────────────┘
```

**具体协同方法：**

| 方法 | 说明 | 效果 |
|------|------|:----:|
| ASA 为 ASO 提供关键词数据 | 用 ASA 测试关键词转化率 | ⭐⭐⭐⭐⭐ |
| ASA 提升下载量信号 | 广告下载量间接提升自然排名 | ⭐⭐⭐⭐ |
| ASO 降低 ASA 成本 | 自然排名靠前的词减少广告投放 | ⭐⭐⭐⭐ |
| ASA 保护品牌词 | 即使自然排名第一也投放品牌词 | ⭐⭐⭐⭐⭐ |
| ASA 补充 ASO 短板 | 无法自然排名靠前的词用广告补充 | ⭐⭐⭐⭐ |

---

## 预算分配和 ROI 优化

### 预算分配框架

| 分配维度 | 建议比例 | 说明 |
|----------|:--------:|------|
| 品牌词 | 20-30% | 必须保护，CPT 低 |
| 通用词 | 40-50% | 主要获客渠道 |
| 竞品词 | 10-20% | 选择性投放 |
| 探索词 | 5-10% | 发现新关键词 |
| 推荐广告 | 10-15% | 品牌曝光补充 |

### ROI 优化模型

```swift
struct ROIOptimizer {
    let targetROI: Double
    let maxCPA: Double

    struct CampaignMetrics {
        let campaignId: String
        let spend: Double
        let installs: Int
        let revenue: Double
        let day7Revenue: Double
    }

    func optimize(metrics: [CampaignMetrics]) -> [OptimizationAction] {
        var actions: [OptimizationAction] = []

        for metric in metrics {
            let cpa = metric.installs > 0 ? metric.spend / Double(metric.installs) : .infinity
            let day7ROI = metric.spend > 0 ? metric.day7Revenue / metric.spend : 0

            if cpa > maxCPA * 2 {
                actions.append(.pauseCampaign(metric.campaignId, reason: "CPA 超出目标 2 倍"))
            } else if cpa > maxCPA {
                actions.append(.reduceBid(metric.campaignId, factor: 0.8, reason: "CPA 高于目标"))
            } else if day7ROI > targetROI * 1.5 {
                actions.append(.increaseBudget(metric.campaignId, factor: 1.2, reason: "ROI 远超目标"))
            } else if day7ROI > targetROI {
                actions.append(.maintain(metric.campaignId, reason: "ROI 达标"))
            } else if day7ROI > targetROI * 0.5 {
                actions.append(.optimizeCreatives(metric.campaignId, reason: "ROI 接近目标，优化素材"))
            } else {
                actions.append(.pauseCampaign(metric.campaignId, reason: "ROI 过低"))
            }
        }

        return actions
    }
}

enum OptimizationAction {
    case increaseBudget(String, factor: Double, reason: String)
    case reduceBid(String, factor: Double, reason: String)
    case pauseCampaign(String, reason: String)
    case maintain(String, reason: String)
    case optimizeCreatives(String, reason: String)
}
```

### 不同阶段的预算策略

| 阶段 | 预算策略 | 目标 | 时间 |
|------|----------|------|:----:|
| 测试期 | 小预算多测试 | 收集数据，发现高转化词 | 1-2 周 |
| 优化期 | 逐步增加预算 | 优化出价和关键词 | 2-4 周 |
| 扩量期 | 大幅增加预算 | 最大化获客量 | 持续 |
| 稳定期 | 维持预算 | 保持 ROI 稳定 | 持续 |

---

## 国内开发者注意事项

### 账户相关

| 注意事项 | 说明 |
|----------|------|
| 付款方式 | 需要支持美元的信用卡 |
| 公司资质 | 可能需要提供公司营业执照 |
| 税务信息 | 需要填写 W-8BEN 表格 |
| 发票 | Apple 提供英文发票，可用于报销 |
| 客服 | 英文客服为主，响应时间较长 |

### 中国大陆市场

| 注意事项 | 说明 |
|----------|------|
| 中国大陆不可用 | Apple Search Ads 目前不在中国大陆 App Store 展示 |
| 替代方案 | 国内可考虑百度SEM、腾讯广告等 |
| 海外市场 | 中国大陆开发者可以投放海外市场 |
| ICP 备案 | 投放海外市场不需要 ICP 备案 |

> ⚠️ **警告**：截至 2026 年，Apple Search Ads 仍不在中国大陆 App Store 展示广告。国内开发者只能投放海外市场。如果你的 App 只面向中国大陆用户，ASA 目前无法使用。

### 汇率和成本

| 注意事项 | 说明 |
|----------|------|
| 汇率波动 | 美元结算，注意汇率风险 |
| 信用卡手续费 | 部分银行收取跨境交易手续费 |
| 税务 | 广告费可能涉及税务处理 |
| 预算换算 | 注意人民币和美元的换算 |

### 投放建议

| 建议 | 说明 |
|------|------|
| 先投港澳台 | 中文市场，用户习惯接近 |
| 再投东南亚 | 成本低，华人用户多 |
| 最后投欧美 | 成本高，竞争激烈 |
| 本地化素材 | 不同市场使用本地化截图和文案 |
| 注意时区 | 广告投放时间要考虑目标市场时区 |

---

## Apple Search Ads 常见问题

### FAQ

| 问题 | 回答 |
|------|------|
| ASA 最低预算是多少？ | 没有最低限制，但建议至少 $50/天起步 |
| 广告审核需要多久？ | 通常 1-24 小时 |
| 可以投放中国大陆吗？ | ❌ 目前不支持 |
| 广告可以指定展示时间吗？ | ❌ 不支持时段投放 |
| 可以 A/B 测试素材吗？ | ✅ 通过 Creative Sets 实现 |
| ASA 数据和第三方数据不一致？ | ASA 使用 Apple 归因，第三方使用 SDK 归因，口径不同 |
| 品牌词已经排名第一还需要投吗？ | ✅ 需要，防止竞品抢夺流量 |
| 广告会展示给已安装用户吗？ | 默认不会，Apple 会过滤已安装用户 |
| 可以同时投放多个 App 吗？ | ✅ 可以，每个 App 创建独立 Campaign |

### 常见错误

| 错误 | 后果 | 避免方法 |
|------|------|----------|
| 不投品牌词 | 品牌流量被竞品抢走 | 品牌词必须投放 |
| 出价过低 | 没有展示量 | 参考建议出价 |
| 不看搜索词报告 | 浪费预算在不相关词上 | 每周检查搜索词报告 |
| 不设否定词 | 广告展示给不相关用户 | 设置否定词列表 |
| 频繁调整出价 | 数据不稳定，无法判断效果 | 至少观察 3 天再调整 |
| 不跟踪 ROI | 不知道广告是否赚钱 | 集成归因和收入追踪 |
| 素材不更新 | 点击率下降 | 每月更新 Creative Sets |

---

## 本章小结

本章详细介绍了 Apple Search Ads 的投放和优化方法：

| 知识点 | 要点 |
|--------|------|
| ASA 概述 | Basic vs Advanced，搜索结果广告 vs 推荐广告 |
| 账户设置 | 信用卡付款、用户权限管理 |
| Campaign 创建 | 预算、地区、受众定位 |
| Ad Group 设置 | 关键词匹配类型、出价策略、否定词 |
| 关键词策略 | 品牌词必投、竞品词选择性投放、通用词大量测试 |
| Creative Sets | 不同关键词/受众展示不同素材 |
| 数据分析 | TTR、CPT、CPA、D7 ROI 等核心指标 |
| 优化策略 | 出价调整、关键词扩展、否定词优化 |
| ASA + ASO | ASA 测试关键词 → ASO 优化排名 → ASA 聚焦高价值词 |
| 国内注意 | 中国大陆不可用，先投港澳台和东南亚 |

**核心原则：ASA 是花钱买数据的工具，最终目标是把付费流量转化为自然流量。**

> 💡 **提示**：对于独立开发者，建议从每月 $500-1000 的预算开始，专注 2-3 个核心市场，用 ASA 测试关键词效果，再把高转化词应用到 ASO 中，逐步减少对付费广告的依赖。

---

**上一章**：[135-社区运营与用户生态](135-社区运营与用户生态.md)

**下一章**：[137-visionOS入门](../10-多平台开发/137-visionOS入门.md)
