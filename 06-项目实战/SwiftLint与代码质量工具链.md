# SwiftLint 与代码质量工具链

> 🎯 **本章目标**：掌握 SwiftLint 代码规范检查与 SwiftFormat 代码格式化的配置与使用，学会 Git Hooks 自动化代码质量检查，建立团队级代码质量保障工作流。

---

## 1. 为什么需要代码质量工具

### 1.1 没有工具链的痛点

想象一个团队里五个人写代码——有人用 Tab 缩进，有人用空格；有人大括号换行，有人不换行；有人变量名用驼峰，有人用下划线。代码评审时，一半时间在争论风格，另一半时间在找 Bug。这不是夸张，而是没有代码质量工具的真实写照。

没有工具链的团队通常会面临以下问题：

| 痛点 | 具体表现 | 后果 |
|------|---------|------|
| 风格不统一 | 缩进、命名、括号位置各异 | 代码像"多人拼凑"，阅读成本高 |
| 潜在 Bug 难发现 | 未使用的变量、强制解包、过长函数 | Bug 藏在混乱的代码中，评审时容易遗漏 |
| 代码评审效率低 | 评审者花大量时间指出风格问题 | 真正的逻辑问题反而没时间审查 |
| 新人上手慢 | 不知道团队规范是什么 | 写出的代码反复被退回修改 |
| 技术债累积 | 代码越来越乱，没人敢重构 | 项目越往后越难维护 |

### 1.2 代码质量工具的价值

代码质量工具就像"语法检查器 + 拼写检查器"。写文章时，Word 会自动在你拼错单词的地方标红、在语法不通顺的地方画波浪线——你不需要等编辑来指出这些低级错误。代码质量工具做的事情完全一样：在你写代码时自动标出风格问题和潜在 Bug，把代码评审的时间留给真正重要的事情。

| 工具 | 解决的问题 | 收益 |
|------|-----------|------|
| SwiftLint | 代码规范违反、潜在 Bug、命名不当 | 统一风格、提前发现问题、减少评审负担 |
| SwiftFormat | 代码格式不一致、缩进混乱 | 自动格式化、消除格式争论、保持代码整洁 |
| Git Hooks | 提交前未检查代码质量 | 防止不规范代码进入仓库 |
| periphery | 死代码、未使用的类型/函数 | 减少包体积、降低维护成本 |
| EditorConfig | 不同编辑器格式不一致 | 跨编辑器基础格式统一 |

> 💡 **提示**：代码质量工具不是替代代码评审，而是把评审从"挑格式毛病"升级为"讨论架构设计"。工具处理机械性问题，人关注创造性问题。

---

## 2. SwiftLint 代码规范检查

### 2.1 SwiftLint 是什么

SwiftLint 是由 Realm 开源的一个 Swift 代码规范检查工具。它基于 Clang 和 SourceKit，能够解析 Swift 源码并检查是否符合预定义的规则集。SwiftLint 内置了 100+ 条规则，覆盖命名规范、代码复杂度、潜在 Bug 等多个维度。

### 2.2 安装方式

**方式一：Homebrew（推荐）**

```bash
brew install swiftlint
```

**方式二：CocoaPods**

```ruby
pod 'SwiftLint'
```

**方式三：SPM**

在 `Package.swift` 中添加：

```swift
dependencies: [
    .package(url: "https://github.com/realm/SwiftLint.git", from: "0.55.0")
]
```

三种安装方式对比：

| 方式 | 优点 | 缺点 | 适用场景 |
|------|------|------|---------|
| Homebrew | 全局可用，安装简单 | 团队成员版本可能不一致 | 个人项目、快速体验 |
| CocoaPods | 版本锁定在 Podfile | 仅 CocoaPods 项目可用 | CocoaPods 管理的项目 |
| SPM | 原生集成，版本可控 | 需要 Package.swift | SPM 管理的项目 |

> 💡 **提示**：团队项目建议使用 CocoaPods 或 SPM 方式安装，这样可以确保所有成员和 CI 环境使用同一版本的 SwiftLint。

### 2.3 .swiftlint.yml 配置文件详解

在项目根目录创建 `.swiftlint.yml` 文件，SwiftLint 会自动读取配置。

```yaml
included:
  - MyApp
  - MyAppKit

excluded:
  - Pods
  - MyApp/Generated
  - .build
  - DerivedData

opt_in_rules:
  - empty_count
  - closure_spacing
  - force_unwrapping
  - implicitly_unwrapped_optional
  - overridden_super_call
  - private_outlet
  - vertical_whitespace_closing_braces
  - nimble_operator

disabled_rules:
  - trailing_whitespace
  - todo

analyzer_rules:
  - explicit_self
  - unused_import
  - unused_declaration

type_body_length:
  warning: 300
  error: 500

function_body_length:
  warning: 40
  error: 100

file_length:
  warning: 500
  error: 1000

type_name:
  min_length: 3
  max_length:
    warning: 40
    error: 50

identifier_name:
  min_length:
    warning: 2
  excluded:
    - id
    - x
    - y
    - z

custom_rules:
  no_print:
    name: "No Print"
    regex: "\\bprint\\("
    match_kinds:
      - identifier
    message: "请使用日志框架代替 print"
    severity: warning
```

配置文件核心字段说明：

| 字段 | 作用 | 说明 |
|------|------|------|
| `included` | 指定检查目录 | 只检查这些目录下的文件 |
| `excluded` | 排除检查目录 | 排除生成代码、第三方库等 |
| `opt_in_rules` | 启用可选规则 | 默认关闭的规则需手动启用 |
| `disabled_rules` | 禁用默认规则 | 关闭不需要的默认规则 |
| `analyzer_rules` | 分析型规则 | 需要 Swift 编译信息，检查更深入 |
| `custom_rules` | 自定义规则 | 用正则表达式定义项目专属规则 |

### 2.4 常用规则解读

| 规则名 | 默认级别 | 说明 | 违规示例 |
|--------|---------|------|---------|
| `force_cast` | 错误 | 禁止强制类型转换 | `let x = obj as! String` |
| `force_try` | 错误 | 禁止强制 try | `try! someFunction()` |
| `force_unwrapping` | 可选 | 禁止强制解包 | `let x = optionalValue!` |
| `implicit_return` | 可选 | 闭包隐式返回 | `map { return $0 }` |
| `vertical_whitespace` | 默认 | 禁止多余空行 | 连续两个以上空行 |
| `trailing_newline` | 默认 | 文件末尾需换行 | 文件末尾无换行符 |
| `colon` | 默认 | 冒号紧跟类型 | `let x : Int` |
| `comma` | 默认 | 逗号后需空格 | `f(a,b,c)` |
| `mark` | 默认 | MARK 注释格式 | `// MARK:-` |
| `nesting` | 默认 | 嵌套层级限制 | 类型嵌套超过 1 层 |
| `function_body_length` | 默认 | 函数体长度限制 | 函数超过 40 行 |
| `type_body_length` | 默认 | 类型体长度限制 | 类型超过 300 行 |
| `cyclomatic_complexity` | 默认 | 圈复杂度限制 | if/for/while 分支过多 |
| `unused_optional_binding` | 默认 | 未使用的可选绑定 | `if let x = value {}` 但未使用 x |
| `redundant_optional_initialization` | 默认 | 冗余的可选初始化 | `var x: Int? = nil` |
| `prohibited_super_call` | 可选 | 必须调用 super | 重写 viewDidLoad 未调 super |
| `empty_count` | 可选 | 使用 isEmpty 代替 count | `array.count == 0` |

### 2.5 自定义规则编写

当内置规则无法满足团队需求时，可以通过 `custom_rules` 编写自定义规则。自定义规则基于正则表达式匹配。

```yaml
custom_rules:
  no_objc_members:
    name: "No @objc Members"
    regex: "@objc\\b"
    match_kinds:
      - keyword
    message: "Swift 代码中避免使用 @objc，如需桥接请说明原因"
    severity: warning

  no_direct_uiappearance:
    name: "No Direct UIAppearance"
    regex: "appearance\\(\\)"
    message: "请使用 AppearanceManager 统一管理 UIAppearance"
    severity: error

  no_hardcoded_colors:
    name: "No Hardcoded Colors"
    regex: "UIColor\\(red:|UIColor\\(displayP3Red:|#colorLiteral"
    message: "请使用 ColorPalette 中定义的颜色常量"
    severity: warning

  no_magic_numbers:
    name: "No Magic Numbers"
    regex: "(?<!\\.|\\d)\\d{2,}(?!\\.|\\d|px)"
    match_kinds:
      - number
    message: "请将魔法数字提取为常量"
    severity: warning
    excluded: "*.strings"
```

> ⚠️ **警告**：自定义规则的正则表达式需要仔细测试，过于宽泛的匹配会产生大量误报。建议先设为 `warning` 级别，观察一段时间确认无误后再提升为 `error`。

### 2.6 禁用规则的方式

**行内禁用**

```swift
let name = user.name! // swiftlint:disable:this force_unwrapping

// swiftlint:disable:next force_cast
let view = controller as! UIViewController

// swiftlint:disable force_unwrapping
let a = optional1!
let b = optional2!
// swiftlint:enable force_unwrapping
```

**文件级禁用**

在文件头部添加：

```swift
// swiftlint:disable file_length
// swiftlint:disable type_body_length
```

**项目级禁用**

在 `.swiftlint.yml` 中添加：

```yaml
disabled_rules:
  - trailing_whitespace
  - todo
```

> 💡 **提示**：行内禁用应尽量少用，每次使用都应附带注释说明原因。项目级禁用需团队达成共识，不能个人随意添加。

### 2.7 Xcode 集成：Build Phase 自动运行

将 SwiftLint 集成到 Xcode 的 Build Phase 中，每次编译时自动运行检查：

1. 在 Xcode 中选择项目 Target → Build Phases
2. 点击 "+" → New Run Script Phase
3. 添加以下脚本：

```bash
if which swiftlint >/dev/null; then
  swiftlint lint --quiet
else
  echo "warning: SwiftLint not installed, download from https://github.com/realm/SwiftLint"
fi
```

4. 将 Run Script 拖到 Compile Sources 之前，这样可以在编译前就发现问题

### 2.8 输出格式与报告

SwiftLint 支持多种输出格式，方便与不同工具集成：

```bash
swiftlint lint --reporter csv > lint_report.csv
swiftlint lint --reporter json > lint_report.json
swiftlint lint --reporter html > lint_report.html
swiftlint lint --reporter checkstyle > lint_report.xml
swiftlint lint --reporter codeclimate > lint_report.json
swiftlint lint --reporter emoji
```

| 格式 | 用途 | 说明 |
|------|------|------|
| `csv` | 数据分析 | 逗号分隔，可导入 Excel |
| `json` | 程序处理 | 结构化数据，方便脚本解析 |
| `html` | 人类阅读 | 生成可视化报告页面 |
| `checkstyle` | CI 集成 | Jenkins 等工具可解析 |
| `codeclimate` | GitLab 集成 | Code Quality 报告格式 |
| `emoji` | 终端美化 | 用表情符号标记严重程度 |

---

## 3. SwiftFormat 代码格式化

### 3.1 SwiftFormat 是什么

SwiftFormat 是由 Nick Lockwood 开发的 Swift 代码格式化工具。与 SwiftLint 不同，SwiftLint 只检查不修复，而 SwiftFormat 可以自动修改代码使其符合格式规范。SwiftFormat 内置 50+ 条格式化规则，覆盖缩进、空格、换行、括号等各个方面。

### 3.2 安装方式

**Homebrew**

```bash
brew install swiftformat
```

**CocoaPods**

```ruby
pod 'SwiftFormat'
```

**SPM**

```swift
dependencies: [
    .package(url: "https://github.com/nicklockwood/SwiftFormat.git", from: "0.54.0")
]
```

### 3.3 .swiftformat 配置文件

在项目根目录创建 `.swiftformat` 文件：

```
--swift 5.9
--indent 4
--tabwidth 4
--smarttabs enabled
--self remove
--importgrouping alpha
--semicolons never
--commas always
--decimalgrouping 3,6
--exponentcase lowercase
--header ignore
--ifdef indent
--operatorfunc spaced
--ranges spaced
--stripunusedargs closure-only
--trimwhitespace always
--voidtype void
--wraparguments before-first
--wrapcollections before-first
--disable redundantReturn
--disable trailingClosures
--exclude Pods,Generated,.build,DerivedData
```

常用配置项说明：

| 配置项 | 作用 | 示例值 |
|--------|------|--------|
| `--indent` | 缩进空格数 | `4` |
| `--self` | self 的处理方式 | `remove`（移除不必要的 self） |
| `--importgrouping` | import 排序方式 | `alpha`（字母排序） |
| `--semicolons` | 分号处理 | `never`（移除分号） |
| `--commas` | 尾逗号 | `always`（多行时添加尾逗号） |
| `--wraparguments` | 参数换行方式 | `before-first` |
| `--voidtype` | Void 类型写法 | `void`（用 Void 而非 ()） |
| `--ranges` | 范围运算符空格 | `spaced`（`0 ..< 10`） |

### 3.4 常用格式化规则

SwiftFormat 的格式化规则覆盖了代码的方方面面：

**缩进与空格**

```swift
// 格式化前
func calculate(  a:Int,b: Int)->Int{
let result=a+b
    return result
}

// 格式化后
func calculate(a: Int, b: Int) -> Int {
    let result = a + b
    return result
}
```

**Import 排序**

```swift
// 格式化前
import UIKit
import Foundation
import SnapKit
import Alamofire

// 格式化后
import Alamofire
import Foundation
import SnapKit
import UIKit
```

**Self 移除**

```swift
// 格式化前
class ViewModel {
    var title: String
    init(title: String) {
        self.title = title
    }
    func update() {
        self.refresh()
    }
}

// 格式化后（self.refresh() 保留，因为必要）
class ViewModel {
    var title: String
    init(title: String) {
        self.title = title
    }
    func update() {
        refresh()
    }
}
```

### 3.5 与 SwiftLint 的区别与配合

| 对比维度 | SwiftLint | SwiftFormat |
|---------|-----------|-------------|
| 核心功能 | 检查代码规范 | 自动格式化代码 |
| 工作方式 | 只报告问题，不自动修改 | 直接修改源码 |
| 规则侧重 | 命名规范、潜在 Bug、复杂度 | 缩进、空格、换行、排序 |
| 规则数量 | 100+ | 50+ |
| 误报率 | 较高（需配置排除） | 极低（格式化结果确定性强） |
| 运行时机 | 编译时 / CI | 保存时 / 提交前 |
| 是否可逆 | 不涉及修改 | 可用 git diff 查看变更 |

两者配合使用的最佳实践：

1. **SwiftFormat 先行**：先格式化代码，消除格式问题
2. **SwiftLint 后行**：再检查规范，发现逻辑层面的问题
3. **避免规则冲突**：确保 SwiftLint 的格式类规则与 SwiftFormat 的格式化规则一致

```yaml
# .swiftlint.yml 中禁用与 SwiftFormat 冲突的规则
disabled_rules:
  - trailing_whitespace
  - vertical_whitespace
  - opening_brace
  - closing_brace
  - colon
  - comma
```

> ⚠️ **警告**：如果不禁用冲突规则，SwiftFormat 格式化后的代码可能仍被 SwiftLint 报错，形成"格式化 → 报错 → 再格式化"的死循环。

### 3.6 Xcode 集成

**方式一：Build Phase**

与 SwiftLint 类似，在 Build Phase 中添加 Run Script：

```bash
if which swiftformat >/dev/null; then
  swiftformat --lint --quiet .
else
  echo "warning: SwiftFormat not installed"
fi
```

注意这里用了 `--lint` 参数，只检查不修改。自动修改源码应该放在 Git Hooks 中，而不是 Build Phase。

**方式二：Xcode 扩展**

安装 SwiftFormat 的 Xcode 扩展后，可以在 Xcode 编辑器菜单中一键格式化当前文件：

1. 安装 SwiftFormat for Xcode 扩展
2. 系统设置 → 隐私与安全 → 扩展 → Xcode Source Editor → 启用 SwiftFormat
3. Xcode 菜单 Editor → SwiftFormat 即可使用

### 3.7 命令行使用

```bash
swiftformat .                          # 格式化当前目录所有文件
swiftformat Sources/                   # 格式化指定目录
swiftformat --lint .                   # 只检查不修改
swiftformat --dryrun .                 # 预览会修改什么
swiftformat --verbose .                # 显示详细输出
swiftformat --exclude Pods,Generated . # 排除目录
swiftformat --config .swiftformat .    # 指定配置文件
swiftformat --stdin < file.swift       # 从标准输入读取
```

> 💡 **提示**：首次在已有项目上运行 SwiftFormat 时，务必使用 `--dryrun` 先预览变更，确认无误后再正式执行。大规模格式化建议单独提交，不要与功能代码混在一起。

---

## 4. Git Hooks 自动化

### 4.1 Git Hooks 原理简介

Git Hooks 是 Git 内置的钩子机制，在特定 Git 操作（如 commit、push）前后自动执行自定义脚本。钩子脚本存放在 `.git/hooks/` 目录下，默认以 `.sample` 后缀提供示例。

| 钩子 | 触发时机 | 常见用途 |
|------|---------|---------|
| `pre-commit` | `git commit` 之前 | 代码检查、格式化 |
| `commit-msg` | 编辑 commit 信息时 | 校验 commit 信息格式 |
| `pre-push` | `git push` 之前 | 运行测试 |
| `post-merge` | `git pull` 之后 | 安装依赖 |

> ⚠️ **警告**：`.git/hooks/` 目录不会被 Git 跟踪，所以直接修改钩子脚本无法在团队间共享。需要借助 Husky 或 lefthook 等工具来管理。

### 4.2 pre-commit 钩子：提交前自动 lint + format

pre-commit 是最常用的钩子，在代码提交前自动运行 SwiftLint 检查和 SwiftFormat 格式化。

手动创建 pre-commit 钩子：

```bash
#!/bin/sh

CHANGED_SWIFT_FILES=$(git diff --cached --name-only --diff-filter=ACM | grep '\.swift$')

if [ -z "$CHANGED_SWIFT_FILES" ]; then
  exit 0
fi

echo "Running SwiftFormat..."
swiftformat $CHANGED_SWIFT_FILES

echo "Running SwiftLint..."
swiftlint lint --quiet --path $CHANGED_SWIFT_FILES

LINT_RESULT=$?

if [ $LINT_RESULT -ne 0 ]; then
  echo "❌ SwiftLint found issues. Please fix them before committing."
  exit 1
fi

git add $CHANGED_SWIFT_FILES

echo "✅ All checks passed!"
exit 0
```

### 4.3 使用 Husky / lefthook 管理钩子

**Husky 方案**

Husky 是最流行的 Git Hooks 管理工具，通过 npm 安装：

```bash
npm init -y
npm install husky --save-dev
npx husky init
```

创建 pre-commit 钩子：

```bash
echo 'npx swiftlint-lint-and-format' > .husky/pre-commit
```

或者直接编写脚本：

```bash
# .husky/pre-commit
#!/bin/sh

FILES=$(git diff --cached --name-only --diff-filter=ACM | grep '\.swift$')
[ -z "$FILES" ] && exit 0

swiftformat $FILES
swiftlint lint --quiet $FILES

if [ $? -ne 0 ]; then
  echo "❌ Lint failed. Fix issues before committing."
  exit 1
fi

git add $FILES
```

**lefthook 方案**

lefthook 是一个用 Go 编写的 Git Hooks 管理工具，配置更灵活，执行速度更快：

```bash
brew install lefthook
lefthook install
```

创建 `lefthook.yml` 配置文件：

```yaml
pre-commit:
  parallel: true
  commands:
    swiftformat:
      glob: "*.swift"
      run: swiftformat {staged_files} && git add {staged_files}
    swiftlint:
      glob: "*.swift"
      run: swiftlint lint --quiet {staged_files}

commit-msg:
  commands:
    conventional:
      run: |
        MSG=$(cat {1})
        if ! echo "$MSG" | grep -qE "^(feat|fix|docs|style|refactor|test|chore|ci|perf|build)(\(.+\))?: .+"; then
          echo "❌ Commit message must follow Conventional Commits format"
          echo "   Example: feat: add login screen"
          exit 1
        fi
```

Husky 与 lefthook 对比：

| 对比维度 | Husky | lefthook |
|---------|-------|----------|
| 语言 | Node.js | Go |
| 安装依赖 | 需要 Node.js 环境 | 独立二进制，零依赖 |
| 配置方式 | 钩子脚本文件 | YAML 配置文件 |
| 并行执行 | 不支持 | 支持 |
| 执行速度 | 较慢（Node.js 启动开销） | 快 |
| 社区生态 | 更成熟，文档更多 | 新兴但增长迅速 |

> 💡 **提示**：如果项目已经有 Node.js 环境（如使用 React Native），Husky 是自然选择。纯 iOS 项目推荐 lefthook，安装简单且执行更快。

### 4.4 CI 中的代码质量检查

Git Hooks 只在本地生效，无法保证所有开发者都正确配置。因此还需要在 CI 中添加代码质量检查作为兜底。

**GitHub Actions 示例**

```yaml
name: Code Quality

on:
  pull_request:
    paths:
      - "**.swift"

jobs:
  lint:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install SwiftLint
        run: brew install swiftlint
      - name: Install SwiftFormat
        run: brew install swiftformat
      - name: Run SwiftLint
        run: swiftlint lint --reporter github-actions-logging
      - name: Run SwiftFormat (check only)
        run: swiftformat --lint .
```

**Xcode Cloud 示例**

在项目根目录创建 `ci_scripts/ci_post_clone.sh`：

```bash
#!/bin/sh

brew install swiftlint swiftformat

echo "Running SwiftLint..."
swiftlint lint --strict

echo "Running SwiftFormat check..."
swiftformat --lint .
```

### 4.5 团队成员的初始化流程

新成员加入团队时，需要一键完成所有工具的安装和配置：

```bash
#!/bin/bash
# scripts/setup.sh

set -e

echo "🔧 Installing code quality tools..."
brew install swiftlint swiftformat lefthook

echo "🔧 Installing lefthook hooks..."
lefthook install

echo "🔧 Verifying installations..."
swiftlint version
swiftformat --version
lefthook version

echo "✅ Setup complete! Code quality tools are ready."
```

在项目 README 中说明初始化步骤：

```bash
git clone https://github.com/your-org/your-app.git
cd your-app
./scripts/setup.sh
```

> ⚠️ **警告**：确保 `scripts/setup.sh` 有执行权限（`chmod +x scripts/setup.sh`），否则新成员运行时会报权限错误。

---

## 5. 其他代码质量工具

### 5.1 periphery：死代码检测

periphery 是一个 Swift 死代码检测工具，它通过分析整个项目的类型引用关系，找出未被使用的函数、类型、协议和属性。

```bash
brew install peripheryapp/periphery/periphery

cd /path/to/your/project
periphery scan
```

periphery 能检测出以下类型的死代码：

| 类型 | 示例 |
|------|------|
| 未使用的函数 | 只有声明，从未被调用 |
| 未使用的类型 | 定义了 struct/class，但从未实例化 |
| 未使用的协议 | 协议定义了但无类型遵循 |
| 未使用的属性 | 实例属性从未被读取 |
| 冗余的 public 声明 | 声明为 public 但模块外未使用 |
| 未使用的 import | import 了框架但未使用其中的类型 |

```yaml
# .periphery.yml
targets:
  - MyApp
  - MyAppKit

exclude:
  - "**/Mock*"
  - "**/*+Generated*"
  - "**/Preview Content/**"
```

> 💡 **提示**：periphery 需要完整的编译环境才能分析，运行时间较长。建议在 CI 的定时任务中运行（如每周一次），而不是每次提交都运行。

### 5.2 SonarQube：代码质量平台

SonarQube 是一个企业级代码质量平台，提供代码质量度量、技术债追踪、安全漏洞检测等功能。

对于 Swift 项目，可以通过 SonarQube + SwiftLint 的组合实现：

```bash
swiftlint lint --reporter json > swiftlint-report.json
sonar-scanner \
  -Dsonar.projectKey=MyApp \
  -Dsonar.sources=MyApp \
  -Dsonar.swift.swiftlint.reportPath=swiftlint-report.json
```

SonarQube 的核心功能：

| 功能 | 说明 |
|------|------|
| Quality Gate | 代码质量门槛，不达标则阻止合并 |
| 技术债追踪 | 量化技术债，计算修复时间 |
| 趋势分析 | 追踪代码质量变化趋势 |
| 安全热点 | 检测潜在安全漏洞 |
| 重复代码 | 检测代码重复率 |

### 5.3 EditorConfig：跨编辑器基础格式统一

EditorConfig 是一个跨编辑器的格式统一方案。团队成员可能使用 Xcode、VS Code、AppCode 等不同编辑器，EditorConfig 确保它们在基础格式上保持一致。

在项目根目录创建 `.editorconfig`：

```ini
root = true

[*]
indent_style = space
indent_size = 4
end_of_line = lf
charset = utf-8
trim_trailing_whitespace = true
insert_final_newline = true

[*.swift]
indent_size = 4

[*.yml]
indent_size = 2

[*.json]
indent_size = 2

[*.md]
trim_trailing_whitespace = false
```

Xcode 从 14.1 开始原生支持 EditorConfig，VS Code 需要安装 EditorConfig 扩展。

### 5.4 Xcode 内置的代码检查功能

除了第三方工具，Xcode 本身也提供了一些代码检查功能：

| 功能 | 位置 | 说明 |
|------|------|------|
| 编译器警告 | Build Settings | 设置警告级别，开启额外警告 |
| Analyzer | Product → Analyze | 静态分析，检测内存泄漏、逻辑错误 |
| Warning Policies | Build Settings | `-Wall -Wextra` 开启更多警告 |
| Swift Concurrency | Build Settings | 严格并发检查模式 |

在 Build Settings 中推荐开启的警告选项：

```
SWIFT_TREAT_WARNINGS_AS_ERRORS = YES
SWIFT_WARN_ON_CONCURRENCY = YES
GCC_TREAT_WARNINGS_AS_ERRORS = NO
```

> 💡 **提示**：`SWIFT_TREAT_WARNINGS_AS_ERRORS = YES` 会将所有 Swift 警告视为错误，确保零警告编译。新项目建议一开始就开启，已有项目可以逐步修复警告后再开启。

---

## 6. 团队代码规范实践

### 6.1 制定团队代码规范文档

代码规范文档是团队共识的书面化，应该简洁明确，避免模糊表述。文档结构建议：

1. **命名规范**：变量、函数、类型的命名规则
2. **格式规范**：缩进、空格、换行的具体要求
3. **架构规范**：文件组织、模块划分原则
4. **最佳实践**：错误处理、可选值、并发等推荐做法
5. **禁止事项**：明确列出不允许的编码模式

```markdown
# 团队 Swift 代码规范

## 命名
- 类型用 UpperCamelCase：`UserProfile`
- 函数和变量用 lowerCamelCase：`fetchUserData()`
- 常量用 lowerCamelCase：`let maxRetryCount = 3`
- 协议用 UpperCamelCase，描述能力时用 -able/-ible：`Cacheable`

## 格式
- 缩进 4 个空格
- 冒号紧跟变量名：`let name: String`
- 函数体超过 40 行需拆分
- 文件超过 500 行需考虑拆分

## 禁止
- 禁止强制解包（`!`），测试代码除外
- 禁止强制类型转换（`as!`）
- 禁止在主线程执行网络请求
- 禁止硬编码颜色值和字符串
```

### 6.2 规范与工具的对应关系

每条规范都应该有对应的工具来自动检查，否则规范只是一纸空文：

| 规范 | 工具 | 配置 |
|------|------|------|
| 缩进 4 空格 | SwiftFormat | `--indent 4` |
| 冒号格式 | SwiftFormat + SwiftLint | `--colon` / `colon` 规则 |
| 禁止强制解包 | SwiftLint | `force_unwrapping` 规则 |
| 禁止强制类型转换 | SwiftLint | `force_cast` 规则 |
| 函数体长度限制 | SwiftLint | `function_body_length` |
| import 排序 | SwiftFormat | `--importgrouping alpha` |
| 尾逗号 | SwiftFormat | `--commas always` |
| 提交信息格式 | lefthook / Husky | commit-msg 钩子 |
| 编码无警告 | Xcode | `TREAT_WARNINGS_AS_ERRORS` |

> 💡 **提示**：如果某条规范没有对应的工具可以自动检查，考虑是否真的需要这条规范。无法自动执行的规范往往会被遗忘。

### 6.3 新成员 Onboarding 流程

新成员加入团队时，代码质量工具的配置应该是自动化的：

```
Day 1: 运行 setup.sh，一键安装所有工具
Day 1: 首次提交代码，Git Hooks 自动运行检查
Day 2: 阅读《团队 Swift 代码规范》文档
Day 3: 第一次代码评审，了解规范的实际应用
```

`setup.sh` 脚本应包含：

```bash
#!/bin/bash
set -e

brew install swiftlint swiftformat lefthook

if [ -f "lefthook.yml" ]; then
  lefthook install
fi

swiftlint version
swiftformat --version

echo "✅ Code quality tools installed successfully!"
echo "📖 Please read docs/code-style-guide.md for team conventions."
```

### 6.4 规范演进与更新机制

代码规范不是一成不变的，需要随项目演进不断调整：

1. **提案机制**：任何人都可以提出修改规范的提案
2. **讨论阶段**：团队讨论，权衡利弊
3. **试验阶段**：先在 `.swiftlint.yml` 中设为 warning，观察效果
4. **正式采纳**：确认无误后提升为 error，更新规范文档
5. **通知全员**：通过 Slack/飞书通知规范变更

```yaml
# 试验阶段：先设为 warning
opt_in_rules:
  - explicit_self
  - no_magic_numbers

custom_rules:
  no_print:
    regex: "\\bprint\\("
    message: "请使用日志框架代替 print"
    severity: warning
```

### 6.5 代码评审中的质量工具辅助

代码评审时，质量工具可以大大减轻评审者的负担：

| 评审关注点 | 是否应由工具处理 | 工具 |
|-----------|----------------|------|
| 缩进不一致 | ✅ 是 | SwiftFormat |
| 命名不规范 | ✅ 是 | SwiftLint |
| 强制解包 | ✅ 是 | SwiftLint |
| 函数过长 | ✅ 是 | SwiftLint |
| 架构设计问题 | ❌ 否 | 人工评审 |
| 业务逻辑正确性 | ❌ 否 | 人工评审 |
| API 设计合理性 | ❌ 否 | 人工评审 |
| 性能优化方向 | ❌ 否 | 人工评审 |

理想的工作流：工具自动处理格式和规范问题，评审者专注于架构、逻辑和设计。

> 💡 **提示**：在 PR 模板中加入检查清单，提醒评审者工具已经检查了哪些方面，他们应该重点关注哪些方面。

---

## 小结

| 知识点 | 核心内容 |
|--------|---------|
| 代码质量工具的价值 | 统一风格、提前发现问题、提升评审效率，工具处理机械性问题，人关注创造性问题 |
| SwiftLint 安装 | Homebrew / CocoaPods / SPM 三种方式，团队项目建议锁定版本 |
| .swiftlint.yml 配置 | included/excluded 控制范围，opt_in_rules 启用可选规则，custom_rules 自定义规则 |
| 常用规则 | force_cast、force_unwrapping、function_body_length、cyclomatic_complexity 等 |
| 自定义规则 | 基于正则表达式，先 warning 后 error，注意避免误报 |
| 禁用规则 | 行内（`:this`/`:next`）、文件级、项目级，应尽量少用 |
| Xcode 集成 | Build Phase 添加 Run Script，每次编译自动检查 |
| 输出格式 | csv/json/html/checkstyle/codeclimate/emoji，方便与不同工具集成 |
| SwiftFormat 安装 | Homebrew / CocoaPods / SPM，与 SwiftLint 安装方式一致 |
| .swiftformat 配置 | indent/self/importgrouping/commas/wraparguments 等核心配置项 |
| SwiftLint vs SwiftFormat | Lint 只检查不修改，Format 自动修改；先 Format 后 Lint，禁用冲突规则 |
| Git Hooks 原理 | pre-commit 最常用，.git/hooks 不被 Git 跟踪，需工具管理 |
| Husky / lefthook | Husky 需要 Node.js，lefthook 独立二进制更快，纯 iOS 项目推荐 lefthook |
| CI 代码质量检查 | GitHub Actions / Xcode Cloud 中运行 SwiftLint + SwiftFormat，作为本地 Hooks 的兜底 |
| 团队初始化 | setup.sh 一键安装，确保新成员开箱即用 |
| periphery | 死代码检测，建议在 CI 定时任务中运行 |
| SonarQube | 企业级代码质量平台，Quality Gate + 技术债追踪 |
| EditorConfig | 跨编辑器基础格式统一，Xcode 14.1+ 原生支持 |
| Xcode 内置检查 | 编译器警告、Analyzer、TREAT_WARNINGS_AS_ERRORS |
| 规范与工具对应 | 每条规范都应有工具自动检查，无法自动执行的规范容易被遗忘 |
| 规范演进 | 提案 → 讨论 → 试验（warning）→ 采纳（error）→ 通知全员 |
| 评审辅助 | 工具处理格式和规范，评审者专注架构、逻辑和设计 |

← [依赖管理与开源库](./依赖管理与开源库.md) | [Xcode 项目配置深入](./Xcode项目配置深入.md) →