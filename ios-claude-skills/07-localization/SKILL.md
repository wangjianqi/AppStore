---
name: localization
description: 涉及多语言、本地化、中英文文案、Localizable.strings、复数处理、语言切换、国际化测试的任务
---

# 本地化 / 多语言

## 支持语言
- **主语言：简体中文（zh-Hans）**
- **次要语言：英文（en）**
- 语言文件位置：`Resources/zh-Hans.lproj/` 和 `Resources/en.lproj/`

---

## 文案管理规范

### Key 命名规则
- 所有用户可见文字**必须走本地化**，禁止硬编码中文或英文字符串
- Key 命名：`模块.组件.含义`（全小写，点分隔）
- 相同含义的按钮/标签用统一 Key，禁止重复定义

```
camera.permission.title = "需要相机权限"
camera.permission.message = "请在设置中开启相机权限，以使用录制功能"
camera.button.start = "开始录制"
camera.button.stop = "停止录制"
settings.title = "设置"
settings.button.logout = "退出登录"
paywall.title = "升级 Pro"
paywall.trial.days = "免费试用 %d 天"
```

---

## 代码使用

### 标准用法

```swift
label.text = NSLocalizedString("camera.button.start", comment: "录制按钮")
```

### 项目封装（推荐）

```swift
extension String {
    var localized: String {
        NSLocalizedString(self, bundle: .main, comment: "")
    }

    func localized(with arguments: CVarArg...) -> String {
        String(format: self.localized, arguments: arguments)
    }
}

// 使用
label.text = "camera.button.start".localized
let text = "paywall.trial.days".localized(with: 7)
```

---

## 复数处理（.stringsdict）

### 创建 Localizable.stringsdict

文件位置：`Resources/zh-Hans.lproj/Localizable.stringsdict` 和 `Resources/en.lproj/Localizable.stringsdict`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>%d items selected</key>
    <dict>
        <key>NSStringLocalizedFormatKey</key>
        <string>%#@items@</string>
        <key>items</key>
        <dict>
            <key>NSStringFormatSpecifierKey</key>
            <string>%d</string>
            <key>NSStringFormatValueTypeKey</key>
            <string>d</string>
            <key>one</key>
            <string>%d 项已选中</string>
            <key>other</key>
            <string>%d 项已选中</string>
        </dict>
    </dict>
    <key>%d hours remaining</key>
    <dict>
        <key>NSStringLocalizedFormatKey</key>
        <string>%#@hours@</string>
        <key>hours</key>
        <dict>
            <key>NSStringFormatSpecifierKey</key>
            <string>%d</string>
            <key>NSStringFormatValueTypeKey</key>
            <string>d</string>
            <key>one</key>
            <string>%d hour remaining</string>
            <key>other</key>
            <string>%d hours remaining</string>
        </dict>
    </dict>
</dict>
</plist>
```

### 代码使用

```swift
let text = String(format: NSLocalizedString("%d items selected", comment: ""), count)
```

### 中文复数规则
- 中文只有 `one` 和 `other`，且通常 `one` 和 `other` 文案相同
- 英文有 `zero` / `one` / `other`，需分别处理

---

## 中文文案规范

### 界面文案
- 简洁、口语化，避免过于书面
- 错误提示：说明问题 + 提供操作指引
  - ❌ "网络错误"
  - ✅ "网络连接失败，请检查网络后重试"
- 按钮文案：动词开头，2-4 个字
  - ✅ "开始录制" "保存" "稍后再说"
  - ❌ "点击开始录制" "确认保存文件"
- 空状态文案：说明原因 + 引导操作
  - ✅ "还没有录制内容\n点击下方按钮开始"

---

## 英文文案规范

- Title Case 用于标题和按钮
- Sentence case 用于描述和提示
- 时态：按钮用动词原形（"Start Recording"），描述用现在时
- 缩写：App 内文案不用缩写（用 "Settings" 不用 "Sett."）

---

## 动态文案（带参数）

```swift
// Localizable.strings
"paywall.trial.message" = "免费试用 %d 天";
"user.welcome" = "欢迎回来，%@";

// 使用
let text = "paywall.trial.message".localized(with: 7)
let welcome = "user.welcome".localized(with: userName)
```

---

## 日期与数字格式

### 日期格式

```swift
let formatter = DateFormatter()
formatter.dateStyle = .medium
formatter.timeStyle = .short
formatter.locale = Locale.current  // 自动适配语言
let text = formatter.string(from: date)
```

- 禁止硬编码日期格式（如 `"yyyy-MM-dd"`），使用 `DateFormatter.Style`
- 如需自定义格式，根据 locale 区分：

```swift
if Locale.current.languageCode == "zh" {
    formatter.dateFormat = "yyyy年MM月dd日"
} else {
    formatter.dateFormat = "MMM d, yyyy"
}
```

### 数字格式

```swift
let numberFormatter = NumberFormatter()
numberFormatter.numberStyle = .currency
numberFormatter.locale = Locale.current
let price = numberFormatter.string(from: 9.99)
// 中文：¥9.99  英文：$9.99
```

- 金额使用 `NumberFormatter`，不要手动拼接
- 百分比使用 `.percent` style

---

## 图片与资源本地化

### Asset Catalog 本地化
- 在 Xcode 中勾选图片的 Localizable
- 为不同语言提供不同变体（如中文用中文标注的截图）

### 文件资源本地化
- 法律文档、用户协议等按语言分目录：
  ```
  Resources/
  ├── zh-Hans.lproj/
  │   ├── terms_of_service.html
  │   └── privacy_policy.html
  └── en.lproj/
      ├── terms_of_service.html
      └── privacy_policy.html
  ```

---

## 语言切换

### 应用内语言切换

```swift
final class LanguageManager {
    static let shared = LanguageManager()

    enum Language: String {
        case zhHans = "zh-Hans"
        case en = "en"
    }

    var currentLanguage: Language {
        guard let code = UserDefaults.standard.string(forKey: "app_language") else {
            return .zhHans
        }
        return Language(rawValue: code) ?? .zhHans
    }

    func switchTo(_ language: Language) {
        UserDefaults.standard.set(language.rawValue, forKey: "app_language")
    }

    func currentBundle: Bundle {
        guard let path = Bundle.main.path(forResource: currentLanguage.rawValue, ofType: "lproj"),
              let bundle = Bundle(path: path) else {
            return Bundle.main
        }
        return bundle
    }
}
```

### 切换后生效
- 语言切换后需要重启 App 才能完全生效
- 提示用户"语言将在重启后生效"

---

## 多语言测试

### 伪语言测试（Pseudolanguage）
- Xcode → Scheme → Run → Arguments → `-AppleLanguages "(en)" -AppleLocale "en_US"`
- 或使用 Xcode 的 Preview 功能切换语言

### 测试 Checklist

```markdown
- [ ] 所有 Key 在两种语言文件中都存在
- [ ] 带参数的文案参数数量一致
- [ ] 英文长文案不超出 UI 布局
- [ ] 日期/数字格式随语言正确切换
- [ ] 复数规则正确（英文 one/other）
- [ ] 切换语言后所有页面文案更新
- [ ] 截图/引导图随语言切换
```

### 自动化检查

```bash
# 检查缺失的 Key
diff <(grep -o '"[^"]*"' Resources/zh-Hans.lproj/Localizable.strings | sort) \
     <(grep -o '"[^"]*"' Resources/en.lproj/Localizable.strings | sort)
```

---

## 已知陷阱

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| 英文文案超长 | 英文比中文长 30-50% | UI 布局用弹性约束，不固定宽度 |
| Key 拼写错误 | 手动输入 Key 易错 | 用 `String` 扩展 + 常量管理 Key |
| 缺失 Key | 新增文案忘记加英文 | CI 中加入 Key 一致性检查 |
| 复数不生效 | 缺少 .stringsdict 文件 | 创建对应的 Localizable.stringsdict |
| 语言切换不生效 | Bundle 缓存 | 使用自定义 Bundle 或重启 App |
| 日期格式硬编码 | `"yyyy-MM-dd"` 不适配语言 | 使用 `DateFormatter.Style` |
| 金额符号错误 | 手动拼接 `¥` / `$` | 使用 `NumberFormatter` + `.currency` |
| NSLocalizedString 返回 Key | Key 不存在时返回 Key 本身 | 开发阶段开启 `showNonLocalizedStrings` |
