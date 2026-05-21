# 109-Fastlane 自动化构建

## 本章目标

- 理解为什么 iOS 开发需要自动化构建以及 Fastlane 的定位
- 掌握 Fastlane 的安装与初始化流程
- 熟练编写 Fastfile，理解 lane、action、参数传递等核心语法
- 掌握 gym、sigh、match、pilot 等常用 Action 的用法
- 深入理解 match 证书管理方案及其团队协作流程
- 能够编写从代码提交到 App Store 上架的完整自动化流水线
- 学会将 Fastlane 与 GitHub Actions、GitLab CI、Jenkins 集成
- 了解国内开发者的常见问题与最佳实践

---

## 1. Fastlane 概述

### 1.1 为什么需要自动化构建

iOS 发布流程堪称"手动操作重灾区"——从修改版本号、切换证书、Archive、导出 IPA、上传 App Store Connect、提交审核，每一步都依赖人工操作，稍有不慎就得重来。

> 💡 生活类比：想象你在开一家面包店——
> - **手动构建**：每天早上你亲手和面、发酵、整形、烘烤、装袋、贴标签、摆上货架。一步出错，整批报废。
> - **自动化构建**：你设计了一条流水线，原料进去，成品出来。你只需要按一下"启动"按钮，机器自动完成所有步骤。

没有自动化时的典型痛点：

| 痛点 | 描述 |
|------|------|
| 手动打包耗时 | Archive + Export + Upload 一次至少 20 分钟，全程不能离开 |
| 证书管理混乱 | 团队多人手动管理证书，经常出现"描述文件过期"的紧急状况 |
| 流程不一致 | 张三的打包方式和李四不同，导致构建产物有差异 |
| 容易遗漏步骤 | 忘记改版本号、忘记更新 Changelog、忘记截图 |
| 发版焦虑 | 每次发版如临大敌，生怕哪个环节出错 |

有了 Fastlane 之后：

```
一条命令 → 版本号递增 → 证书同步 → Archive → 导出 IPA → 上传 App Store → 提交审核
   🚀         🔢           🔐         🔨         📦           🚀            ✅
```

### 1.2 Fastlane vs Xcode Cloud 对比

| 特性 | Fastlane | Xcode Cloud |
|------|----------|-------------|
| 提供方 | 开源社区（现属 Google） | Apple 官方 |
| 配置方式 | Ruby 代码（Fastfile） | App Store Connect 界面 |
| 灵活度 | ⭐⭐⭐ 最高 | ⭐⭐ 中等 |
| 学习曲线 | ⭐⭐⭐ 较陡 | ⭐ 低 |
| 证书管理 | 需配合 match | 全自动 |
| 需要 Mac 服务器 | ✅ 需要（自建或 CI runner） | ❌ 不需要 |
| 自定义 Action | ✅ 支持 | ❌ 仅限脚本 |
| 跨平台支持 | ✅ iOS + Android | ❌ 仅 iOS |
| 社区生态 | 丰富（400+ 插件） | 有限 |
| 费用 | 免费（自建） | 按用量计费 |
| 适合团队 | 需要深度定制的团队 | iOS 新手 / 小团队 |

> 💡 选型建议：
> - **小团队 / 新手**：Xcode Cloud 开箱即用，零配置证书管理
> - **中大型团队**：Fastlane + match，灵活度和可控性更高
> - **跨平台项目**：Fastlane 一套工具管 iOS + Android
> - **混合方案**：Xcode Cloud 日常构建 + Fastlane 处理发版流程

### 1.3 Fastlane 适用场景

| 场景 | 说明 |
|------|------|
| 日常发版 | 一条命令完成打包上传，告别手动操作 |
| 多渠道分发 | 同时上传 TestFlight、内测分发、企业分发 |
| 证书统一管理 | match 让团队所有人使用同一套证书和描述文件 |
| 多 App 管理 | 一个 Fastfile 管理多个 Target 或多个 App |
| CI/CD 集成 | 与 GitHub Actions / GitLab CI / Jenkins 无缝配合 |
| 自动截图 | generate_screenshots 自动生成多语言截图 |

---

## 2. 安装与初始化

### 2.1 Ruby 环境配置

Fastlane 基于 Ruby 开发，macOS 自带 Ruby，但建议使用版本管理工具避免权限问题。

> ⚠️ 不要使用系统自带的 Ruby（`/usr/bin/ruby`），用 `sudo gem install` 可能导致系统文件损坏。请使用 rbenv 或 rvm 管理独立版本。

**推荐方式：使用 rbenv**

```bash
# 安装 rbenv
brew install rbenv ruby-build

# 配置 shell
echo 'eval "$(rbenv init - zsh)"' >> ~/.zshrc
source ~/.zshrc

# 安装 Ruby 3.x
rbenv install 3.2.2
rbenv global 3.2.2

# 验证
ruby --version
# => ruby 3.2.2
which ruby
# => /Users/xxx/.rbenv/shims/ruby
```

### 2.2 安装 Fastlane

```bash
# 方式一：gem 安装（推荐）
gem install fastlane -NV

# 方式二：Homebrew 安装
brew install fastlane

# 验证安装
fastlane --version
```

> 💡 推荐使用 gem 方式安装，版本更新更及时。`-NV` 参数表示不安装文档、显示详细输出。

### 2.3 fastlane init

在项目根目录执行初始化：

```bash
cd /path/to/your/project
fastlane init
```

初始化时会出现交互菜单：

```
✅  What would you like to use fastlane for?
1. 📸  Automate screenshots
2. 👩‍✈️  Automate beta distribution to TestFlight
3. 🚀  Automate App Store distribution
4. 🛠️  Manual setup - manually create your Fastfile
```

| 选项 | 适用场景 |
|------|----------|
| 1 | 需要自动生成 App Store 截图 |
| 2 | 只需要 TestFlight 分发 |
| 3 | 需要完整的 App Store 发布流程 |
| 4 | 自定义配置，最灵活 |

> 💡 建议选择 4（Manual setup），手动配置更灵活，也更容易理解 Fastlane 的工作原理。

### 2.4 目录结构

初始化完成后，项目根目录会生成 `fastlane/` 文件夹：

```
your-project/
├── fastlane/
│   ├── Fastfile          # 核心：定义 lane（自动化流程）
│   ├── Appfile           # 配置：App 标识、Apple ID 等基本信息
│   ├── Matchfile         # 配置：match 证书管理设置
│   ├── .env              # 环境变量（敏感信息，加入 .gitignore）
│   └── report/           # 构建报告
├── Gemfile               # Ruby 依赖管理（推荐）
└── ...
```

**Appfile 示例：**

```ruby
app_identifier("com.yourcompany.yourapp")
apple_id("your@email.com")
itc_team_id("123456789")
team_id("ABCDEFGHIJ")
```

> 💡 建议同时创建 `Gemfile` 来锁定 Fastlane 版本，确保团队所有人使用相同版本：

```ruby
# Gemfile
source "https://rubygems.org"

gem "fastlane", "~> 2.220"
```

然后使用 `bundle exec fastlane` 代替 `fastlane` 命令。

---

## 3. Fastfile 核心语法

### 3.1 lane 定义

`lane` 是 Fastlane 的核心概念，相当于一个"自动化任务"。

> 💡 生活类比：lane 就像工厂流水线上的一个"工位"，每个工位负责一道工序。你可以单独运行某个工位，也可以把多个工位串联起来。

```ruby
# fastlane/Fastfile

default_platform(:ios)

platform :ios do
  desc "运行测试"
  lane :test do
    run_tests(
      scheme: "MyApp",
      devices: ["iPhone 15"]
    )
  end

  desc "打包并上传到 TestFlight"
  lane :beta do
    build_ios_app(
      scheme: "MyApp",
      export_method: "app-store"
    )
    upload_to_testflight
  end
end
```

运行 lane：

```bash
fastlane test
fastlane beta
```

### 3.2 Action 调用

Action 是 Fastlane 的"原子操作"，每个 Action 完成一个小任务。Fastlane 内置了 400+ 个 Action。

```ruby
lane :demo do
  # Action 调用方式一：方法调用风格
  cocoapods
  swiftlint

  # Action 调用方式二：带参数
  gym(
    scheme: "MyApp",
    export_method: "ad-hoc"
  )

  # Action 调用方式三：使用 Action 类名
  Actions::SlackAction.run(
    message: "构建完成！"
  )
end
```

### 3.3 参数传递

lane 支持接收外部参数，使用 `lane_context` 或 `params`：

```ruby
lane :build do |options|
  scheme = options[:scheme] || "MyApp"
  env = options[:environment] || "staging"

  UI.message("构建 Scheme: #{scheme}, 环境: #{env}")

  gym(
    scheme: scheme,
    export_options: {
      compileBitcode: false,
      provisioningProfiles: {
        "com.yourcompany.#{scheme}" => "match AppStore com.yourcompany.#{scheme}"
      }
    }
  )
end
```

调用时传参：

```bash
fastlane build scheme:"MyApp" environment:"production"
```

### 3.4 before_all / after_all / error

Fastlane 提供了三个生命周期钩子，用于在 lane 执行前后做统一处理：

```ruby
platform :ios do
  before_all do |lane, options|
    UI.message("🚀 开始执行 lane: #{lane}")
    cocoapods
    ensure_git_status_clean
  end

  after_all do |lane, options|
    UI.success("✅ lane #{lane} 执行成功！")
    slack(
      message: "✅ #{lane} 构建成功！",
      slack_url: ENV["SLACK_WEBHOOK_URL"]
    )
  end

  error do |lane, exception, options|
    UI.error("❌ lane #{lane} 执行失败：#{exception.message}")
    slack(
      message: "❌ #{lane} 构建失败：#{exception.message}",
      slack_url: ENV["SLACK_WEBHOOK_URL"],
      success: false
    )
  end

  lane :beta do
    gym(scheme: "MyApp")
    upload_to_testflight
  end
end
```

| 钩子 | 触发时机 | 典型用途 |
|------|----------|----------|
| `before_all` | 所有 lane 执行前 | 检查 Git 状态、安装依赖 |
| `after_all` | 所有 lane 执行成功后 | 发送成功通知、清理临时文件 |
| `error` | 任何 lane 执行失败时 | 发送失败通知、记录错误日志 |

---

## 4. 常用 Action 详解

### 4.1 gym —— 构建与打包

`gym` 是 Fastlane 中最核心的 Action，负责 Archive + Export IPA。

> 💡 生活类比：gym 就像面包房里的"烤箱"，把原材料（源代码）变成成品（IPA）。

```ruby
lane :build_release do
  gym(
    workspace: "MyApp.xcworkspace",
    scheme: "MyApp",
    configuration: "Release",
    export_method: "app-store",
    output_directory: "./build",
    output_name: "MyApp.ipa",
    include_bitcode: false,
    export_options: {
      compileBitcode: false,
      provisioningProfiles: {
        "com.yourcompany.MyApp" => "match AppStore com.yourcompany.MyApp"
      }
    }
  )
end
```

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `workspace` | .xcworkspace 路径 | 自动检测 |
| `scheme` | 构建方案 | 自动检测 |
| `export_method` | 导出方式 | app-store |
| `output_directory` | IPA 输出目录 | . |
| `include_bitcode` | 是否包含 Bitcode | false |
| `clean` | 构建前是否 clean | false |

`export_method` 可选值：

| 值 | 用途 |
|----|------|
| `app-store` | App Store 发布 |
| `ad-hoc` | 有限设备分发 |
| `enterprise` | 企业证书分发 |
| `development` | 开发测试 |

### 4.2 sigh —— 描述文件管理

`sigh` 自动管理 Provisioning Profile，下载、续期、修复一条龙。

```ruby
lane :renew_profile do
  sigh(
    app_identifier: "com.yourcompany.MyApp",
    type: "appstore",
    force: true
  )
end
```

| 参数 | 说明 |
|------|------|
| `app_identifier` | Bundle Identifier |
| `type` | 类型：development / ad_hoc / appstore / enterprise |
| `force` | 是否强制续期 |
| `username` | Apple ID |

> 💡 通常不需要单独使用 sigh，match 已经内置了描述文件管理功能。sigh 更适合"急救"场景——描述文件突然过期，需要快速续期。

### 4.3 match —— 证书与描述文件统一管理

`match` 是 Fastlane 中最推荐的证书管理方案，将在第 5 节详细讲解。这里先给出基本用法：

```ruby
lane :sync_certs do
  match(
    type: "appstore",
    app_identifier: "com.yourcompany.MyApp",
    git_url: "https://github.com/yourteam/certificates",
    readonly: true
  )
end
```

### 4.4 pilot —— TestFlight 上传

`pilot`（即 `upload_to_testflight`）将 IPA 上传到 App Store Connect 并分发到 TestFlight。

```ruby
lane :upload_beta do
  gym(scheme: "MyApp")
  pilot(
    app_identifier: "com.yourcompany.MyApp",
    distribute_external: false,
    groups: ["内部测试组"],
    changelog: "修复了若干 Bug",
    skip_waiting_for_build_processing: false
  )
end
```

| 参数 | 说明 |
|------|------|
| `distribute_external` | 是否分发给外部测试员 |
| `groups` | TestFlight 测试组名称 |
| `changelog` | 更新日志 |
| `skip_waiting_for_build_processing` | 是否跳过等待构建处理 |
| `demo_account_required` | 是否需要演示账号 |

### 4.5 upload_to_app_store —— 提交审核

`upload_to_app_store`（别名 `deliver`）负责上传元数据、截图和 IPA，并提交审核。

```ruby
lane :release do
  match(type: "appstore", readonly: true)
  gym(scheme: "MyApp")

  upload_to_app_store(
    app_identifier: "com.yourcompany.MyApp",
    submit_for_review: true,
    automatic_release: false,
    force: true,
    precheck_include_in_app_purchases: false,
    submission_information: {
      add_id_info_uses_idfa: false
    }
  )
end
```

| 参数 | 说明 |
|------|------|
| `submit_for_review` | 是否自动提交审核 |
| `automatic_release` | 审核通过后是否自动发布 |
| `force` | 跳过 HTML 预览确认 |
| `precheck_include_in_app_purchases` | 是否检查内购配置 |
| `submission_information` | 提交审核时的附加信息 |

---

## 5. match 证书管理

### 5.1 证书管理的痛点

> 💡 生活类比：证书就像公司的"公章"——
> - **没有 match**：每个人自己刻一个公章，用的时候发现"这个章不对"，然后重新刻，旧的作废。最后抽屉里一堆公章，不知道哪个能用。
> - **有了 match**：公司统一保管一枚公章，谁要用都从同一个地方取，用完放回去。永远只有一枚有效的公章。

传统证书管理的噩梦：

| 问题 | 描述 |
|------|------|
| 证书数量上限 | Apple 限制每种类型最多 3 个证书，超出就得撤销旧的 |
| 描述文件过期 | 经常在发版当天发现描述文件过期了 |
| 团队协作困难 | 新成员加入后无法构建，因为没有证书和描述文件 |
| 撤销连锁反应 | 撤销一个证书，所有使用该证书的描述文件全部失效 |
| 本地存储 | 证书只存在某人的电脑上，换电脑就丢失 |

### 5.2 match 的工作原理

```
match 命令
   ↓
从 Git 仓库拉取加密的证书和描述文件
   ↓
使用 passphrase 解密
   ↓
安装到本地钥匙串和 Xcode
   ↓
构建时使用已安装的证书签名
```

| 特性 | 说明 |
|------|------|
| 统一存储 | 所有证书和描述文件存储在 Git 仓库中 |
| 加密保护 | 使用 passphrase 加密，即使仓库泄露也无法使用 |
| 版本控制 | Git 历史记录每次证书变更 |
| 团队共享 | 新成员只需 clone 仓库 + 输入 passphrase |
| 自动续期 | 描述文件过期时自动续期 |

### 5.3 match 初始化

```bash
fastlane match init
```

选择存储方式：

```
✅  How would you like to store certificates?
1. 🔒  Git repository (recommended)
2. 📦  Google Cloud Storage
3. ☁️  Amazon S3
```

> 💡 推荐选择 Git 仓库，最简单也最常用。Google Cloud Storage 和 Amazon S3 适合大型团队。

初始化后生成 `Matchfile`：

```ruby
# fastlane/Matchfile

git_url("https://github.com/yourteam/certificates")
storage_mode("git")
type("appstore")
app_identifier("com.yourcompany.MyApp")
username("your@email.com")
```

### 5.4 match 命令详解

**首次使用——生成证书和描述文件：**

```bash
fastlane match appstore
fastlane match development
fastlane match adhoc
```

首次运行会提示输入 passphrase（加密密码），请务必妥善保管！

**日常使用——同步证书和描述文件：**

```bash
fastlane match appstore --readonly
```

> ⚠️ 日常构建时务必加 `--readonly`，防止意外撤销或重新生成证书。

**在 Fastfile 中使用：**

```ruby
lane :build do
  match(type: "appstore", readonly: true)
  gym(scheme: "MyApp")
end
```

**撤销并重新生成：**

```bash
fastlane match nuke appstore
fastlane match appstore
```

> ⚠️ `nuke` 会撤销该类型的所有证书和描述文件，请确保团队其他人不需要当前证书后再执行！

### 5.5 团队协作流程

```
管理员：
1. fastlane match appstore          → 生成证书，推送到 Git 仓库
2. 将 passphrase 告知团队成员       → 通过 1Password 等安全渠道传递

新成员：
1. fastlane match appstore --readonly → 从 Git 拉取并安装证书
2. 输入 passphrase                    → 解密证书文件
3. 开始构建                           → 证书已安装到钥匙串
```

| 角色 | 操作 | 注意事项 |
|------|------|----------|
| 管理员 | `fastlane match appstore` | 首次运行设置 passphrase |
| 开发者 | `fastlane match appstore --readonly` | 只读模式，不会修改证书 |
| CI 服务器 | `fastlane match appstore --readonly` | 通过环境变量传入 passphrase |

---

## 6. 完整自动化流水线

### 6.1 从代码提交到上架的完整 Fastfile

下面是一个生产环境可用的完整 Fastfile：

```ruby
# fastlane/Fastfile

default_platform(:ios)

platform :ios do
  before_all do |lane, options|
    UI.message("🚀 开始执行: #{lane}")
    ensure_git_status_clean unless lane == :test
  end

  desc "运行测试"
  lane :test do
    run_tests(
      workspace: "MyApp.xcworkspace",
      scheme: "MyApp",
      devices: ["iPhone 15"],
      code_coverage: true
    )
  end

  desc "构建并上传到 TestFlight"
  lane :beta do
    ensure_git_branch(branch: "develop")
    bump_version
    sync_certificates
    build_app
    upload_beta
    tag_release
    push_git_tags
  end

  desc "构建并提交 App Store 审核"
  lane :release do
    ensure_git_branch(branch: "main")
    bump_version
    sync_certificates
    build_app
    upload_release
    tag_release
    push_git_tags
  end

  # ── 私有 lane（辅助方法）──

  desc "自动递增版本号"
  private_lane :bump_version do
    current_version = get_version_number(
      xcodeproj: "MyApp.xcodeproj",
      target: "MyApp"
    )
    new_build_number = increment_build_number(
      xcodeproj: "MyApp.xcodeproj"
    )
    UI.message("版本: #{current_version}, Build: #{new_build_number}")

    commit_version_bump(
      message: "Bump build number to #{new_build_number}",
      xcodeproj: "MyApp.xcodeproj"
    )
  end

  desc "同步证书"
  private_lane :sync_certificates do
    match(
      type: "appstore",
      readonly: true,
      app_identifier: [
        "com.yourcompany.MyApp",
        "com.yourcompany.MyApp.Widget"
      ]
    )
  end

  desc "构建 IPA"
  private_lane :build_app do
    gym(
      workspace: "MyApp.xcworkspace",
      scheme: "MyApp",
      configuration: "Release",
      export_method: "app-store",
      output_directory: "./build",
      include_bitcode: false,
      export_options: {
        compileBitcode: false,
        provisioningProfiles: {
          "com.yourcompany.MyApp" => "match AppStore com.yourcompany.MyApp",
          "com.yourcompany.MyApp.Widget" => "match AppStore com.yourcompany.MyApp.Widget"
        }
      }
    )
  end

  desc "上传到 TestFlight"
  private_lane :upload_beta do
    pilot(
      distribute_external: false,
      groups: ["内部测试"],
      changelog: read_changelog
    )
  end

  desc "上传到 App Store 并提交审核"
  private_lane :upload_release do
    deliver(
      submit_for_review: true,
      automatic_release: false,
      force: true,
      precheck_include_in_app_purchases: false,
      submission_information: {
        add_id_info_uses_idfa: false
      }
    )
  end

  desc "打 Git Tag"
  private_lane :tag_release do
    version = get_version_number(xcodeproj: "MyApp.xcodeproj", target: "MyApp")
    build = get_build_number(xcodeproj: "MyApp.xcodeproj")
    add_git_tag(tag: "v#{version}-#{build}")
  end

  after_all do |lane, options|
    UI.success("✅ #{lane} 执行成功！")
  end

  error do |lane, exception, options|
    UI.error("❌ #{lane} 执行失败：#{exception.message}")
  end
end
```

### 6.2 版本号自动递增

Fastlane 提供了多种版本号管理策略：

| 策略 | Action | 说明 |
|------|--------|------|
| 递增 Build Number | `increment_build_number` | 每次构建自动 +1 |
| 指定版本号 | `increment_version_number` | 手动指定新版本号 |
| 基于 Git Commit 数 | 自定义 | 用 commit 数作为 build number |

**基于 Git Commit 数的 Build Number：**

```ruby
private_lane :bump_version do
  build_number = sh("git rev-list --count HEAD").strip
  increment_build_number(
    build_number: build_number,
    xcodeproj: "MyApp.xcodeproj"
  )
end
```

### 6.3 Changelog 生成

```ruby
def read_changelog
  changelog_path = "./CHANGELOG.md"
  if File.exist?(changelog_path)
    content = File.read(changelog_path)
    sections = content.split(/^## /)
    return sections[1].strip if sections.length > 1
  end
  "请查看 Git 提交记录"
end
```

也可以使用 Fastlane 的 `changelog_from_git_commits` Action 自动从 Git 提交记录生成：

```ruby
private_lane :generate_changelog do
  changelog = changelog_from_git_commits(
    between: [last_git_tag, "HEAD"],
    pretty: "- %s",
    match_lightweight_tag: false
  )
  UI.message("Changelog:\n#{changelog}")
  changelog
end
```

---

## 7. 与 CI/CD 平台集成

### 7.1 GitHub Actions + Fastlane

在项目根目录创建 `.github/workflows/build.yml`：

```yaml
name: iOS Build

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: macos-14
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.2'
          bundler-cache: true

      - name: Install dependencies
        run: |
          bundle install
          pod install

      - name: Run tests
        run: bundle exec fastlane test

      - name: Build and upload to TestFlight
        if: github.ref == 'refs/heads/develop'
        env:
          MATCH_PASSWORD: ${{ secrets.MATCH_PASSWORD }}
          APP_STORE_CONNECT_API_KEY_ID: ${{ secrets.API_KEY_ID }}
          APP_STORE_CONNECT_ISSUER_ID: ${{ secrets.ISSUER_ID }}
          APP_STORE_CONNECT_API_KEY: ${{ secrets.API_PRIVATE_KEY }}
        run: bundle exec fastlane beta
```

> 💡 推荐使用 App Store Connect API Key 认证，而非 Apple ID + 密码。API Key 不需要 2FA 验证，更适合 CI 环境。

### 7.2 GitLab CI + Fastlane

创建 `.gitlab-ci.yml`：

```yaml
stages:
  - test
  - build

variables:
  LC_ALL: "en_US.UTF-8"
  LANG: "en_US.UTF-8"

test:
  stage: test
  tags:
    - macos
  script:
    - bundle install
    - bundle exec fastlane test
  only:
    - merge_requests

build_beta:
  stage: build
  tags:
    - macos
  script:
    - bundle install
    - bundle exec fastlane beta
  only:
    - develop
  variables:
    MATCH_PASSWORD: $MATCH_PASSWORD

build_release:
  stage: build
  tags:
    - macos
  script:
    - bundle install
    - bundle exec fastlane release
  only:
    - main
  when: manual
  variables:
    MATCH_PASSWORD: $MATCH_PASSWORD
```

### 7.3 Jenkins 集成

**Jenkins Pipeline 脚本：**

```groovy
pipeline {
    agent { label 'macos' }

    environment {
        MATCH_PASSWORD = credentials('match-password')
        FASTLANE_USER = credentials('apple-id')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'bundle install'
                sh 'pod install'
            }
        }

        stage('Test') {
            steps {
                sh 'bundle exec fastlane test'
            }
        }

        stage('Build & Upload') {
            when {
                branch 'main'
            }
            steps {
                sh 'bundle exec fastlane release'
            }
        }
    }

    post {
        success {
            slackSend(message: "✅ 构建成功: ${env.JOB_NAME} #${env.BUILD_NUMBER}")
        }
        failure {
            slackSend(message: "❌ 构建失败: ${env.JOB_NAME} #${env.BUILD_NUMBER}")
        }
    }
}
```

### 7.4 CI/CD 平台对比

| 特性 | GitHub Actions | GitLab CI | Jenkins |
|------|---------------|-----------|---------|
| macOS Runner | ✅ GitHub 托管 | ❌ 需自建 | ❌ 需自建 |
| 配置文件 | `.github/workflows/*.yml` | `.gitlab-ci.yml` | `Jenkinsfile` |
| 密钥管理 | Secrets | CI/CD Variables | Credentials Plugin |
| 免费额度 | 2000 分钟/月 | 400 分钟/月 | 免费（自建） |
| 学习曲线 | 低 | 中 | 高 |
| 适合团队 | 开源 / 小团队 | 中型团队 | 大型企业 |

---

## 8. 常见问题与最佳实践

### 8.1 Ruby 版本管理

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| `permission denied` | 使用了系统 Ruby | 安装 rbenv，使用独立 Ruby |
| 版本不一致 | 团队成员 Ruby 版本不同 | Gemfile 锁定版本，使用 `bundle exec` |
| gem 安装缓慢 | 默认源在国外 | 切换到国内镜像源 |

**切换 Ruby 镜像源：**

```bash
gem sources --remove https://rubygems.org/
gem sources --add https://gems.ruby-china.com/
gem sources -l
```

### 8.2 Fastlane 更新策略

```bash
# 查看当前版本
fastlane --version

# 更新到最新版
gem update fastlane

# 更新到指定版本
gem install fastlane -v 2.220.0

# 使用 bundle 管理版本（推荐）
# 修改 Gemfile 中的版本号后执行
bundle update fastlane
```

> 💡 建议在 Gemfile 中锁定 Fastlane 次版本号（如 `~> 2.220`），团队统一版本，避免"我电脑上能跑，你电脑上不行"的问题。

### 8.3 国内网络问题解决方案

国内开发者使用 Fastlane 常遇到网络超时，主要瓶颈在 Apple API 和 CocoaPods。

| 问题 | 解决方案 |
|------|----------|
| CocoaPods 安装慢 | 使用国内 CDN 镜像 |
| App Store Connect API 超时 | 设置代理或使用 API Key 认证 |
| gem install 超时 | 切换 Ruby 镜像源 |
| Git 仓库访问慢 | 使用 Gitee 镜像或配置代理 |

**CocoaPods 镜像配置：**

```ruby
# Podfile 顶部添加
source 'https://mirrors.tuna.tsinghua.edu.cn/git/CocoaPods/Specs.git'
```

**Fastlane 代理设置：**

```bash
# 设置 HTTP 代理
export http_proxy=http://127.0.0.1:7890
export https_proxy=http://127.0.0.1:7890

# 或在 Fastfile 中设置
ENV["http_proxy"] = "http://127.0.0.1:7890"
ENV["https_proxy"] = "http://127.0.0.1:7890"
```

### 8.4 App Store Connect API Key 认证

传统方式使用 Apple ID + 密码认证，在 CI 环境中需要处理 2FA，非常麻烦。推荐使用 API Key 认证：

**生成 API Key：**

1. 登录 App Store Connect → Users and Access → Integrations → App Store Connect API
2. 点击 Generate API Key
3. 记录 Key ID、Issuer ID，下载 .p8 文件

**在 Fastfile 中使用：**

```ruby
lane :release do
  api_key = app_store_connect_api_key(
    key_id: ENV["APP_STORE_CONNECT_API_KEY_ID"],
    issuer_id: ENV["APP_STORE_CONNECT_ISSUER_ID"],
    key_filepath: "./fastlane/AuthKey.p8"
  )

  match(
    type: "appstore",
    readonly: true,
    api_key: api_key
  )

  gym(
    scheme: "MyApp",
    export_method: "app-store",
    api_key: api_key
  )

  deliver(
    submit_for_review: true,
    api_key: api_key
  )
end
```

> ⚠️ .p8 文件是敏感信息，不要提交到 Git 仓库。在 CI 环境中通过 Secrets 注入。

### 8.5 最佳实践清单

| 实践 | 说明 |
|------|------|
| 使用 Gemfile | 锁定 Fastlane 和依赖版本 |
| 使用 bundle exec | 确保使用项目级别的 gem |
| 使用 API Key 认证 | 避免 2FA 问题，CI 友好 |
| match readonly | 日常构建只读，防止意外修改证书 |
| 私有 lane | 辅助方法标记为 `private_lane`，避免被直接调用 |
| 环境变量 | 敏感信息通过 `.env` 或 CI Secrets 传入 |
| before_all 检查 | 构建前检查 Git 状态、分支等前置条件 |
| 错误处理 | 使用 `error` 钩子统一处理失败通知 |
| dry run | 先用 `--dry-run` 模拟执行，确认无误后再正式运行 |
| 定期更新 | Fastlane 更新频繁，建议每月更新一次 |

---

## 本章小结

| 主题 | 核心要点 |
|------|----------|
| Fastlane 概述 | iOS 自动化构建利器，灵活度最高，适合需要深度定制的团队 |
| 安装与初始化 | rbenv 管理 Ruby → gem install fastlane → fastlane init → 生成 Fastfile/Appfile |
| Fastfile 语法 | lane 定义任务、action 执行操作、params 传参、before_all/after_all/error 生命周期 |
| 常用 Action | gym 打包、sigh 管理描述文件、match 统一证书、pilot 上传 TestFlight、deliver 提交审核 |
| match 证书管理 | Git 仓库加密存储、passphrase 保护、团队共享、readonly 日常使用 |
| 完整流水线 | 版本号递增 → 证书同步 → 构建 IPA → 上传 → 提交审核 → 打 Tag |
| CI/CD 集成 | GitHub Actions / GitLab CI / Jenkins 均可集成，推荐 API Key 认证 |
| 最佳实践 | Gemfile 锁版本、API Key 认证、match readonly、环境变量管理敏感信息 |

> 💡 一句话总结：Fastlane 是 iOS 开发者的"自动化瑞士军刀"——前期投入学习成本，后期回报是每次发版从 30 分钟手动操作变成一条命令搞定。
