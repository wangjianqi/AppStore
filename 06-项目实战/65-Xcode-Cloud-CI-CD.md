# 65-Xcode Cloud CI/CD

## 本章目标

- 理解 CI/CD 的核心概念以及为什么 iOS 开发需要它
- 了解 Xcode Cloud 的定位、优势以及与其他方案的对比
- 学会在 App Store Connect 中配置 Xcode Cloud
- 掌握 `ci_scripts` 目录的用法和自定义脚本编写
- 能够配置完整的构建工作流，包括触发条件、环境变量和构建方案
- 了解如何在 Xcode Cloud 中运行自动化测试
- 掌握自动分发到 TestFlight 的方法
- 学会查看构建报告和配置通知
- 完成一个完整的 iOS 项目 CI/CD 实战配置
- 了解常见问题和成本考量

---

## 1. CI/CD 是什么

### 1.1 概念解释

**CI（Continuous Integration，持续集成）** 和 **CD（Continuous Delivery/Deployment，持续交付/部署）** 是现代软件开发中两个紧密相连的实践。

> 💡 生活类比：想象你开了一家奶茶店——
> - **CI（持续集成）**：每做一杯奶茶，立刻尝一口，确保味道没问题。而不是做了 100 杯之后才发现糖放多了，那 100 杯全得倒掉。
> - **CD（持续交付）**：尝完没问题，立刻贴标签放到柜台上，等顾客来拿。不需要再经过一道"质检-打包-上架"的繁琐流程。

| 概念 | 全称 | 核心思想 | 类比 |
|------|------|----------|------|
| CI | Continuous Integration | 代码频繁合并，自动构建+测试 | 每杯奶茶做完立刻尝 |
| CD | Continuous Delivery | 通过测试后自动打包，随时可发布 | 尝完直接放柜台 |
| CD | Continuous Deployment | 自动发布到生产环境 | 尝完直接递给顾客 |

> ⚠️ 注意：CD 有两种含义——Continuous Delivery（持续交付，需手动点"发布"）和 Continuous Deployment（持续部署，全自动发布）。在 iOS 场景中，由于 App Store 审核的存在，通常是 Continuous Delivery。

### 1.2 为什么 iOS 开发需要 CI/CD

没有 CI/CD 时的典型痛点：

| 痛点 | 描述 |
|------|------|
| "我电脑上能跑啊" | 代码在开发者机器上正常，到别人机器就报错 |
| 手动打包耗时 | 每次发版都要手动 Archive → Export → 上传，至少 20 分钟 |
| 忘记跑测试 | 改了代码忘记跑测试，结果引入了回归 Bug |
| 证书/描述文件噩梦 | 手动管理证书和 Provisioning Profile，团队协作时经常冲突 |
| 发版流程不一致 | 不同人打包方式不同，导致构建产物有差异 |

有了 CI/CD 之后：

```
代码推送 → 自动构建 → 自动测试 → 自动打包 → 自动上传 TestFlight
   🎉         🔨          ✅          📦          🚀
```

> 💡 一句话总结：CI/CD 就是让机器帮你做那些重复、容易出错的事情，让你专注于写代码。

---

## 2. Xcode Cloud 简介

### 2.1 Xcode Cloud 是什么

Xcode Cloud 是 Apple 在 WWDC21 推出的官方 CI/CD 服务，深度集成在 Xcode 和 App Store Connect 生态中。

核心特点：

| 特点 | 说明 |
|------|------|
| Apple 官方出品 | 与 Xcode、App Store Connect 无缝集成 |
| 免配置证书 | 自动管理签名证书和 Provisioning Profile |
| 云端 Mac | 不需要自己维护 Mac 服务器 |
| 内置 Xcode | 使用与本地相同的 xcodebuild 工具链 |
| 按用量计费 | 每月有免费额度 |

### 2.2 与其他方案对比

| 特性 | Xcode Cloud | GitHub Actions | Fastlane |
|------|-------------|----------------|----------|
| 提供方 | Apple | GitHub | 开源社区 |
| 配置难度 | ⭐ 简单 | ⭐⭐ 中等 | ⭐⭐⭐ 较复杂 |
| 证书管理 | 全自动 | 需手动/第三方 | 需配合 match |
| 需要 Mac 服务器 | ❌ 不需要 | ✅ 需要（macOS runner） | ✅ 需要 |
| 免费额度 | 25 小时/月 | 2000 分钟/月 | 免费（自建） |
| 自定义程度 | 中等 | 高 | 最高 |
| 与 Apple 生态集成 | 最深 | 一般 | 较深 |
| 学习曲线 | 低 | 中 | 高 |
| 适合人群 | iOS 开发新手 | 跨平台团队 | 需要高度定制的团队 |

> 💡 选型建议：
> - **新手/小团队**：Xcode Cloud，开箱即用，零配置证书管理
> - **跨平台团队**：GitHub Actions，一套 CI 跑多端
> - **需要深度定制**：Fastlane，灵活度最高但学习成本也最高
> - **混合方案**：Xcode Cloud + Fastlane 脚本，兼顾易用性和灵活性

---

## 3. 配置 Xcode Cloud

### 3.1 前提条件

| 条件 | 说明 |
|------|------|
| Apple Developer Program | 必须加入，年费 $99 |
| App Store Connect | 应用已创建记录 |
| Xcode 项目 | 使用 Xcode 14+ 创建 |
| Git 仓库 | 项目代码托管在 GitHub、GitLab 或 Bitbucket |

### 3.2 在 App Store Connect 中开启

**步骤一：连接代码仓库**

1. 登录 [App Store Connect](https://appstoreconnect.apple.com)
2. 进入你的 App → **Xcode Cloud** 标签页
3. 点击 **Get Started**
4. 选择你的 Git 托管平台（GitHub / GitLab / Bitbucket）
5. 授权访问你的仓库

> ⚠️ 如果使用 GitHub，建议只授权特定仓库的访问权限，而非所有仓库，遵循最小权限原则。

**步骤二：确认项目设置**

Xcode Cloud 会自动检测你的项目配置：

```
检测项目 → 识别 Scheme → 识别测试目标 → 生成默认工作流
```

### 3.3 工作流配置

工作流（Workflow）是 Xcode Cloud 的核心概念，定义了"什么时候触发、做什么事"。

在 App Store Connect 中配置工作流：

1. 进入 **Xcode Cloud** → **Workflows**
2. 点击 **Create Workflow** 或编辑默认工作流
3. 配置各个部分（下一节详细讲解）

工作流的基本结构：

```
┌─────────────────────────────────────────┐
│              Workflow                    │
├─────────────────────────────────────────┤
│  触发条件 (When to run)                  │
│  ├── 分支：main                         │
│  ├── Tag：v*                            │
│  └── Pull Request                       │
├─────────────────────────────────────────┤
│  环境 (Environment)                     │
│  ├── Xcode 版本                         │
│  ├── macOS 版本                         │
│  └── 环境变量                           │
├─────────────────────────────────────────┤
│  操作 (Actions)                         │
│  ├── 构建 + 测试                        │
│  └── 自动 Archive                       │
├─────────────────────────────────────────┤
│  后处理 (Post-actions)                  │
│  ├── 分发到 TestFlight                  │
│  └── 发送通知                           │
└─────────────────────────────────────────┘
```

---

## 4. ci_scripts 目录

### 4.1 目录结构

Xcode Cloud 支持在项目中添加自定义脚本，这些脚本放在 `ci_scripts` 目录下：

```
MyApp/
├── ci_scripts/
│   ├── ci_post_clone.sh        # 克隆完成后执行
│   └── ci_post_xcodebuild.sh   # 构建完成后执行
├── MyProject.xcodeproj
├── MyProject/
│   ├── AppDelegate.swift
│   └── ...
└── MyProjectTests/
    └── ...
```

> 💡 `ci_scripts` 目录必须放在仓库根目录下，Xcode Cloud 会自动识别并执行其中的脚本。

### 4.2 脚本执行时机

| 脚本名称 | 执行时机 | 典型用途 |
|----------|----------|----------|
| `ci_post_clone.sh` | 代码克隆完成后、构建开始前 | 安装依赖、配置环境、生成文件 |
| `ci_post_xcodebuild.sh` | 构建完成后 | 上传产物、发送通知、清理 |

执行顺序：

```
克隆代码 → ci_post_clone.sh → xcodebuild → ci_post_xcodebuild.sh
```

### 4.3 ci_post_clone.sh 示例

```bash
#!/bin/sh

# ci_post_clone.sh
# 在代码克隆完成后、构建开始前执行

echo "=== 开始执行 ci_post_clone.sh ==="

# 安装 CocoaPods 依赖（如果使用 CocoaPods）
if [ -f "Podfile" ]; then
    echo "检测到 Podfile，开始安装 CocoaPods 依赖..."
    gem install cocoapods
    pod install
fi

# 安装 Swift Package Manager 依赖（Xcode Cloud 通常自动处理，但可显式执行）
if [ -f "Package.swift" ]; then
    echo "检测到 Package.swift，SPM 依赖将由 Xcode 自动解析"
fi

# 设置版本号（使用 Git commit 数量）
BUILD_NUMBER=$(git rev-list --count HEAD)
echo "构建号: $BUILD_NUMBER"

# 如果使用 agvtool 更新版本号，需要先确保 Info.plist 中版本号使用变量
# agvtool new-version -all $BUILD_NUMBER

echo "=== ci_post_clone.sh 执行完毕 ==="
```

### 4.4 ci_post_xcodebuild.sh 示例

```bash
#!/bin/sh

# ci_post_xcodebuild.sh
# 在构建完成后执行

echo "=== 开始执行 ci_post_xcodebuild.sh ==="

# 检查构建是否成功
if [ "$CI_RESULT" = "success" ]; then
    echo "✅ 构建成功！"

    # 发送通知到 Slack（示例）
    # SLACK_WEBHOOK_URL 在 Xcode Cloud 环境变量中配置
    if [ -n "$SLACK_WEBHOOK_URL" ]; then
        curl -X POST "$SLACK_WEBHOOK_URL" \
            -H "Content-Type: application/json" \
            -d "{
                \"text\": \"✅ 构建成功！\n分支: $CI_BRANCH\n提交: $CI_COMMIT\n构建号: $CI_BUILD_NUMBER\"
            }"
    fi
else
    echo "❌ 构建失败！"

    if [ -n "$SLACK_WEBHOOK_URL" ]; then
        curl -X POST "$SLACK_WEBHOOK_URL" \
            -H "Content-Type: application/json" \
            -d "{
                \"text\": \"❌ 构建失败！\n分支: $CI_BRANCH\n提交: $CI_COMMIT\n构建号: $CI_BUILD_NUMBER\"
            }"
    fi
fi

echo "=== ci_post_xcodebuild.sh 执行完毕 ==="
```

### 4.5 Xcode Cloud 内置环境变量

Xcode Cloud 在执行脚本时提供了丰富的环境变量：

| 环境变量 | 说明 | 示例值 |
|----------|------|--------|
| `CI` | 标识当前在 CI 环境 | `true` |
| `CI_BUILD_NUMBER` | 构建编号 | `42` |
| `CI_BRANCH` | 当前分支 | `main` |
| `CI_COMMIT` | Git commit SHA | `abc1234...` |
| `CI_RESULT` | 构建结果 | `success` / `failure` |
| `CI_PRODUCT` | 产品名称 | `MyApp` |
| `CI_SCHEME` | 使用的 Scheme | `MyApp` |
| `CI_PLATFORM` | 目标平台 | `ios` |
| `CI_XCODE_VERSION` | Xcode 版本 | `15.0` |
| `CI_WORKFLOW` | 工作流名称 | `default` |
| `CI_PULL_REQUEST_NUMBER` | PR 编号 | `123` |

> ⚠️ 自定义环境变量（如 API Key、Webhook URL）需要在 App Store Connect 的工作流配置中添加，**不要**硬编码在脚本中。

### 4.6 脚本权限

脚本文件必须有可执行权限，否则 Xcode Cloud 不会执行：

```bash
# 在终端中为脚本添加可执行权限
chmod +x ci_scripts/ci_post_clone.sh
chmod +x ci_scripts/ci_post_xcodebuild.sh
```

> ⚠️ 忘记添加执行权限是最常见的"脚本没跑"原因！添加后记得提交到 Git。

---

## 5. 构建工作流

### 5.1 触发条件

Xcode Cloud 支持以下触发条件：

| 触发条件 | 说明 | 典型场景 |
|----------|------|----------|
| 分支推送 | 指定分支有新提交时触发 | `main` 分支推送时构建 |
| Tag 推送 | 创建指定模式的 Tag 时触发 | `v*` Tag 触发发布构建 |
| Pull Request | PR 创建或更新时触发 | PR 合并前验证 |
| 手动触发 | 在 App Store Connect 中手动点击 | 紧急修复后手动构建 |

配置示例——多触发条件组合：

```
工作流: "CI Build"
├── 触发条件:
│   ├── 分支: main, develop          → 推送时触发
│   ├── Pull Request: main ← *       → PR 时触发
│   └── Tag: v*                       → 打 Tag 时触发
```

> 💡 实践建议：为不同场景创建不同的工作流。例如：
> - `CI` 工作流：PR 触发，只跑测试，不 Archive
> - `Release` 工作流：Tag 触发，Archive + 分发到 TestFlight

### 5.2 环境配置

在 App Store Connect 的工作流编辑器中，可以配置：

| 配置项 | 选项 | 说明 |
|--------|------|------|
| Xcode 版本 | 14.x / 15.x / 16.x / Latest | 建议选 Latest 或与团队一致 |
| macOS 版本 | Ventura / Sonoma / Sequoia | 跟随 Xcode 版本自动选择 |
| Scheme | 项目中的 Scheme | 选择要构建的 Scheme |
| 设备类型 | iPhone / iPad / Mac | 选择目标设备 |

### 5.3 环境变量

在工作流中添加自定义环境变量：

**在 App Store Connect 中配置：**

1. 进入工作流编辑器 → **Environment** 部分
2. 点击 **Add Environment Variable**
3. 输入名称和值

| 变量名 | 用途 | 是否勾选 Secure |
|--------|------|-----------------|
| `SLACK_WEBHOOK_URL` | Slack 通知地址 | ✅ 是 |
| `API_KEY` | 第三方服务密钥 | ✅ 是 |
| `APP_VERSION` | 自定义版本号 | ❌ 否 |

> ⚠️ 敏感信息（API Key、密码等）务必勾选 **Secure**，这样在日志中会被脱敏显示为 `***`。

### 5.4 构建方案选择

| 构建类型 | 说明 | 产物 |
|----------|------|------|
| Build Only | 只编译，不 Archive | 无 IPA |
| Build and Test | 编译 + 运行测试 | 测试报告 |
| Build, Test, and Archive | 完整流程 | IPA + 测试报告 |
| Archive Only | 只做 Archive | IPA |

> 💡 日常开发用 "Build and Test" 就够了，发版时用 "Build, Test, and Archive"。

---

## 6. 自动化测试

### 6.1 单元测试

Xcode Cloud 会自动识别项目中的测试 Target 并运行。

**前提条件：**

- 项目中包含测试 Target（如 `MyAppTests`）
- Scheme 中勾选了测试 Target

**确认 Scheme 配置：**

1. Xcode → Product → Scheme → Edit Scheme
2. 选择 **Test** 动作
3. 确认测试 Target 已勾选

```
Edit Scheme → Test → Info:
  ☑️ MyAppTests
  ☑️ MyAppUITests (如果有 UI 测试)
```

### 6.2 UI 测试

Xcode Cloud 同样支持 UI 测试，但需要注意：

| 注意事项 | 说明 |
|----------|------|
| 运行时间 | UI 测试比单元测试慢很多，注意控制免费额度 |
| 模拟器 | Xcode Cloud 使用模拟器运行 UI 测试 |
| 截图 | UI 测试中的截图会出现在构建报告中 |
| 稳定性 | UI 测试可能因时序问题偶尔失败，建议加入重试机制 |

### 6.3 测试脚本配置

如果需要在测试前后执行自定义操作，可以在 `ci_post_clone.sh` 中配置：

```bash
#!/bin/sh

# ci_post_clone.sh

echo "=== 准备测试环境 ==="

# 如果测试需要模拟数据，可以在这里生成
# 例如：创建测试用的 JSON 文件
cat > TestFixtures/mock_data.json << EOF
{
    "id": 1,
    "name": "Test User",
    "email": "test@example.com"
}
EOF

echo "=== 测试环境准备完毕 ==="
```

### 6.4 测试结果查看

构建完成后，在 App Store Connect 中查看测试结果：

```
Xcode Cloud → 选择构建 → Test Results
├── ✅ 通过的测试（绿色）
├── ❌ 失败的测试（红色，附带失败原因和截图）
├── ⏭️ 跳过的测试（灰色）
└── 📊 测试覆盖率报告
```

> 💡 Xcode 14+ 可以直接在 Xcode 中查看 Xcode Cloud 的测试结果：Report Navigator → Cloud 标签。

---

## 7. 自动分发

### 7.1 TestFlight 分发

Xcode Cloud 构建成功后可以自动分发到 TestFlight，这是最常用的分发方式。

**配置步骤：**

1. 在工作流编辑器中找到 **Post-Actions**
2. 添加 **TestFlight** 分发动作
3. 选择分发方式

| 分发方式 | 说明 |
|----------|------|
| Internal Testing Only | 仅内部分发（最多 100 人） |
| External Testing | 外部分发（最多 10,000 人，需 Beta App Review） |

### 7.2 内部测试组 vs 外部测试组

| 对比项 | 内部测试组 | 外部测试组 |
|--------|-----------|-----------|
| 人数上限 | 100 人 | 10,000 人 |
| 审核要求 | 无需审核 | 需要 Beta App Review |
| 邀请方式 | App Store Connect 团队成员 | 邮件邀请链接 |
| 上架速度 | 即时 | 审核后（通常 1-2 天） |
| 适合场景 | 团队内部验证 | 大范围公测 |

### 7.3 自动分发配置

在 App Store Connect 的工作流 Post-Actions 中：

```
Post-Actions:
├── TestFlight Internal Testing
│   └── 分发到: 内部测试组
└── TestFlight External Testing
    ├── 分发到: 外部测试组
    └── 附带测试说明: "修复了登录页面的 Bug"
```

### 7.4 分发前的自动测试保障

推荐的工作流设计——只有测试全部通过才分发：

```
触发条件: Tag v*
    ↓
构建 + 运行所有测试
    ↓
测试通过？ ──No──→ 通知团队（构建失败）
    ↓ Yes
Archive
    ↓
上传到 App Store Connect
    ↓
分发到 TestFlight 内部测试组
    ↓
通知团队（新版本已就绪）
```

> ⚠️ 外部测试需要 Beta App Review，建议先分发到内部测试组验证，确认无问题后再推广到外部测试组。

---

## 8. 构建报告与通知

### 8.1 构建报告

每次构建完成后，Xcode Cloud 会生成详细的报告：

| 报告类型 | 内容 | 查看位置 |
|----------|------|----------|
| 构建日志 | 完整的构建输出 | App Store Connect / Xcode |
| 测试结果 | 通过/失败/跳过的测试 | App Store Connect / Xcode |
| 测试覆盖率 | 代码覆盖率百分比 | Xcode Report Navigator |
| 截图 | UI 测试截图 | App Store Connect |
| 警告 | 编译警告列表 | 构建日志 |
| 产物 | IPA / dSYM 文件 | App Store Connect |

### 8.2 在 Xcode 中查看

Xcode 14+ 可以直接查看 Xcode Cloud 的构建信息：

```
Xcode → Report Navigator (⌘9) → Cloud 标签
├── 构建列表
├── 构建详情
│   ├── 触发信息
│   ├── 构建日志
│   ├── 测试结果
│   └── 产物下载
└── 构建趋势
```

### 8.3 邮件通知

Xcode Cloud 默认发送邮件通知：

| 事件 | 通知对象 |
|------|----------|
| 构建开始 | 触发构建的人 |
| 构建成功 | 触发构建的人 + App Store Connect 用户 |
| 构建失败 | 触发构建的人 + App Store Connect 用户 |
| 测试失败 | 触发构建的人 |

### 8.4 Slack 通知

通过 `ci_post_xcodebuild.sh` 脚本发送 Slack 通知：

```bash
#!/bin/sh

# ci_post_xcodebuild.sh

if [ -z "$SLACK_WEBHOOK_URL" ]; then
    echo "未配置 SLACK_WEBHOOK_URL，跳过 Slack 通知"
    exit 0
fi

if [ "$CI_RESULT" = "success" ]; then
    EMOJI="✅"
    COLOR="good"
    STATUS="构建成功"
else
    EMOJI="❌"
    COLOR="danger"
    STATUS="构建失败"
fi

curl -X POST "$SLACK_WEBHOOK_URL" \
    -H "Content-Type: application/json" \
    -d "{
        \"attachments\": [{
            \"color\": \"$COLOR\",
            \"title\": \"$EMOJI $STATUS\",
            \"fields\": [
                { \"title\": \"分支\", \"value\": \"$CI_BRANCH\", \"short\": true },
                { \"title\": \"构建号\", \"value\": \"$CI_BUILD_NUMBER\", \"short\": true },
                { \"title\": \"Scheme\", \"value\": \"$CI_SCHEME\", \"short\": true },
                { \"title\": \"工作流\", \"value\": \"$CI_WORKFLOW\", \"short\": true }
            ]
        }]
    }"
```

### 8.5 Discord / 钉钉 / 飞书通知

其他平台的通知原理相同，都是通过 Webhook URL 发送 HTTP 请求：

```bash
# Discord Webhook 示例
curl -X POST "$DISCORD_WEBHOOK_URL" \
    -H "Content-Type: application/json" \
    -d "{
        \"content\": \"✅ 构建成功！分支: $CI_BRANCH, 构建号: $CI_BUILD_NUMBER\"
    }"

# 飞书 Webhook 示例
curl -X POST "$FEISHU_WEBHOOK_URL" \
    -H "Content-Type: application/json" \
    -d "{
        \"msg_type\": \"text\",
        \"content\": { \"text\": \"✅ 构建成功！分支: $CI_BRANCH, 构建号: $CI_BUILD_NUMBER\" }
    }"
```

---

## 9. 实战：配置一个完整的 iOS 项目 CI/CD 流程

### 9.1 项目背景

假设我们有一个名为 `TodoApp` 的 iOS 项目：

```
TodoApp/
├── ci_scripts/
│   ├── ci_post_clone.sh
│   └── ci_post_xcodebuild.sh
├── TodoApp.xcodeproj
├── TodoApp/
│   ├── App/
│   ├── Models/
│   ├── Views/
│   └── ...
├── TodoAppTests/
│   └── ...
└── TodoAppUITests/
    └── ...
```

### 9.2 创建 ci_scripts 目录和脚本

**创建目录：**

```bash
mkdir -p ci_scripts
```

**ci_post_clone.sh：**

```bash
#!/bin/sh

echo "=== ci_post_clone.sh 开始 ==="
echo "分支: $CI_BRANCH"
echo "提交: $CI_COMMIT"
echo "构建号: $CI_BUILD_NUMBER"

# 使用 Git commit 数量作为构建号
BUILD_NUMBER=$(git rev-list --count HEAD)
echo "自动构建号: $BUILD_NUMBER"

# 更新 Info.plist 中的构建号
/usr/libexec/PlistBuddy -c "Set :CFBundleVersion $BUILD_NUMBER" "$CI_PRIMARY_REPOSITORY_PATH/TodoApp/Info.plist"

# 如果使用 CocoaPods
if [ -f "Podfile" ]; then
    echo "安装 CocoaPods 依赖..."
    gem install cocoapods --no-document
    pod install
fi

echo "=== ci_post_clone.sh 完成 ==="
```

**ci_post_xcodebuild.sh：**

```bash
#!/bin/sh

echo "=== ci_post_xcodebuild.sh 开始 ==="
echo "构建结果: $CI_RESULT"

if [ "$CI_RESULT" = "success" ]; then
    echo "✅ 构建成功"

    if [ -n "$SLACK_WEBHOOK_URL" ]; then
        curl -s -X POST "$SLACK_WEBHOOK_URL" \
            -H "Content-Type: application/json" \
            -d "{
                \"text\": \"✅ TodoApp 构建成功\n分支: $CI_BRANCH\n构建号: $CI_BUILD_NUMBER\nScheme: $CI_SCHEME\"
            }"
    fi
else
    echo "❌ 构建失败"

    if [ -n "$SLACK_WEBHOOK_URL" ]; then
        curl -s -X POST "$SLACK_WEBHOOK_URL" \
            -H "Content-Type: application/json" \
            -d "{
                \"text\": \"❌ TodoApp 构建失败\n分支: $CI_BRANCH\n构建号: $CI_BUILD_NUMBER\nScheme: $CI_SCHEME\"
            }"
    fi
fi

echo "=== ci_post_xcodebuild.sh 完成 ==="
```

**添加执行权限并提交：**

```bash
chmod +x ci_scripts/ci_post_clone.sh
chmod +x ci_scripts/ci_post_xcodebuild.sh

git add ci_scripts/
git commit -m "Add Xcode Cloud CI scripts"
git push
```

### 9.3 配置工作流一：CI（持续集成）

**目的：** 每次 PR 和推送时自动跑测试，确保代码质量。

在 App Store Connect 中配置：

| 配置项 | 值 |
|--------|-----|
| 工作流名称 | CI |
| 触发条件 - 分支 | `main`, `develop` |
| 触发条件 - PR | `main` ← 任何分支 |
| 环境 - Xcode | Latest |
| 操作 | Build and Test |
| Scheme | TodoApp |
| Post-Actions | 无（只跑测试，不分发） |

### 9.4 配置工作流二：Release（发布）

**目的：** 打 Tag 时自动构建、测试、打包、分发到 TestFlight。

| 配置项 | 值 |
|--------|-----|
| 工作流名称 | Release |
| 触发条件 - Tag | `v*`（如 v1.0.0） |
| 环境 - Xcode | Latest |
| 操作 | Build, Test, and Archive |
| Scheme | TodoApp |
| Post-Actions | TestFlight Internal Testing |
| 环境变量 | `SLACK_WEBHOOK_URL`（Secure） |

### 9.5 完整发布流程

```bash
# 1. 确保代码在 main 分支且测试通过
git checkout main
git pull

# 2. 更新版本号（如需要）
# 在 Xcode 中修改 Marketing Version

# 3. 提交更改
git add .
git commit -m "Bump version to 1.0.0"
git push

# 4. 打 Tag 触发 Release 工作流
git tag v1.0.0
git push origin v1.0.0

# 5. Xcode Cloud 自动执行：
#    克隆代码 → ci_post_clone.sh → 构建+测试 → Archive
#    → ci_post_xcodebuild.sh → 上传 → 分发到 TestFlight
#    → Slack 通知

# 6. 在 TestFlight 中验证后，提交 App Store 审核
```

### 9.6 工作流全景图

```
开发者推送代码 / 打 Tag
         ↓
   Xcode Cloud 触发
         ↓
   克隆代码到云端 Mac
         ↓
  执行 ci_post_clone.sh
   (安装依赖、设置版本号)
         ↓
   xcodebuild 构建
         ↓
   运行单元测试 + UI 测试
         ↓
   测试通过？ ──No──→ 通知失败 → 开发者修复
         ↓ Yes
   Archive (如果是 Release 工作流)
         ↓
   执行 ci_post_xcodebuild.sh
   (发送通知)
         ↓
   上传到 App Store Connect
         ↓
   分发到 TestFlight
         ↓
   测试人员收到新版本
```

---

## 10. 常见问题与成本

### 10.1 常见问题

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 脚本没有执行 | 忘记 `chmod +x` | 添加可执行权限并重新提交 |
| CocoaPods 依赖安装失败 | 网络超时或版本不兼容 | 在脚本中指定 `gem install cocoapods -v '具体版本'` |
| 证书签名失败 | 项目签名配置不正确 | 确保使用 Automatic Signing，Xcode Cloud 会自动管理 |
| 测试超时 | UI 测试运行时间过长 | 减少测试用例或拆分为多个工作流 |
| 构建找不到 Scheme | Scheme 没有设为 Shared | Xcode → Scheme → Manage Schemes → 勾选 Shared |
| 环境变量读取不到 | 变量名拼写错误 | 检查 App Store Connect 中的变量名和脚本中是否一致 |
| SPM 依赖解析慢 | 依赖包较大 | 在 `ci_post_clone.sh` 中提前解析：`xcodebuild -resolvePackageDependencies` |
| 分发到外部测试被拒 | Beta App Review 未通过 | 确保应用没有明显崩溃和违规内容 |

### 10.2 成本

Xcode Cloud 按计算时长计费：

| 计划 | 每月免费额度 | 超出后价格 |
|------|-------------|-----------|
| Apple Developer Program 成员 | 25 小时/月 | $0.025/分钟 |

**典型构建耗时估算：**

| 构建类型 | 预计耗时 | 每日次数 | 月耗时估算 |
|----------|----------|----------|-----------|
| Build and Test（小项目） | 5-10 分钟 | 5 次 | ~1.5 小时 |
| Build and Test（中项目） | 10-20 分钟 | 5 次 | ~3 小时 |
| Build, Test, Archive | 15-30 分钟 | 1 次 | ~0.75 小时 |
| UI 测试 | 20-40 分钟 | 2 次 | ~2 小时 |

> 💡 省钱技巧：
> - PR 工作流只跑受影响的测试，不跑全部
> - UI 测试单独放在一个工作流，只在特定条件下触发
> - 避免频繁推送小改动，善用本地测试
> - 监控用量：App Store Connect → Xcode Cloud → Usage

**25 小时免费额度够用吗？**

| 团队规模 | 够不够 | 说明 |
|----------|--------|------|
| 个人开发者 | ✅ 绰绰有余 | 每天 2-3 次构建完全够用 |
| 2-5 人小团队 | ✅ 基本够用 | 合理配置触发条件即可 |
| 5-10 人团队 | ⚠️ 可能不够 | 需要优化工作流或购买额外时长 |
| 10+ 人团队 | ❌ 大概率不够 | 建议混合使用 GitHub Actions 或自建方案 |

### 10.3 Xcode Cloud 的局限性

| 局限性 | 说明 |
|--------|------|
| 只支持 Apple 平台 | 不能构建 Android 或其他平台 |
| 自定义程度有限 | 不如 Fastlane 灵活 |
| 依赖 Apple 生态 | 必须使用 App Store Connect |
| 构建环境固定 | 不能安装自定义系统级依赖 |
| 缓存机制有限 | SPM/CocoaPods 缓存不如其他方案成熟 |
| 并发构建数有限 | 免费计划同时只能运行 1 个构建 |

> ⚠️ 如果你的项目需要高度定制化的 CI/CD 流程（如多平台构建、自定义部署脚本等），建议考虑 GitHub Actions + Fastlane 的组合方案。

---

## 小结

| 知识点 | 关键内容 |
|--------|----------|
| CI/CD 概念 | 持续集成（自动构建+测试）+ 持续交付（自动打包+分发） |
| Xcode Cloud | Apple 官方 CI/CD 服务，免配置证书，开箱即用 |
| 配置流程 | App Store Connect 连接仓库 → 创建工作流 → 配置触发条件 |
| ci_scripts | `ci_post_clone.sh`（构建前）+ `ci_post_xcodebuild.sh`（构建后） |
| 构建工作流 | 触发条件 + 环境配置 + 构建操作 + 后处理动作 |
| 自动化测试 | 单元测试和 UI 测试自动运行，结果可在 Xcode 和 App Store Connect 中查看 |
| 自动分发 | 构建成功后自动上传到 TestFlight，支持内部/外部测试组 |
| 构建报告 | 构建日志、测试结果、覆盖率、截图 |
| 通知 | 邮件默认通知 + 自定义 Webhook（Slack/Discord/飞书等） |
| 成本 | 每月 25 小时免费，超出 $0.025/分钟，小团队基本够用 |

> 💡 一句话总结：Xcode Cloud 是 iOS 开发者入门 CI/CD 的最佳选择——零配置证书管理、与 Apple 生态深度集成、每月 25 小时免费额度，让你从第一天起就能享受自动化带来的效率提升。
