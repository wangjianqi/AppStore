---
name: localization
description: 涉及多语言、本地化、中英文文案、Localizable.strings、语言切换的任务
---

# 本地化 / 多语言

## 支持语言
- **主语言：简体中文（zh-Hans）**
- **次要语言：英文（en）**
- 语言文件位置：`Resources/zh-Hans.lproj/` 和 `Resources/en.lproj/`

## 文案管理规范
- 所有用户可见文字**必须走本地化**，禁止硬编码中文或英文字符串
- Key 命名规则：`模块.组件.含义`（全小写，点分隔）
  ```
  camera.permission.title = "需要相机权限"
  camera.permission.message = "请在设置中开启相机权限，以使用录制功能"
  camera.button.start = "开始录制"
  ```
- 相同含义的按钮/标签用统一 Key，禁止重复定义

## 代码使用
```swift
// 标准用法
label.text = NSLocalizedString("camera.button.start", comment: "录制按钮")

// 项目封装（推荐）
label.text = "camera.button.start".localized
```

```swift
// String 扩展（Core/Extensions/String+Localized.swift）
extension String {
    var localized: String {
        NSLocalizedString(self, bundle: .main, comment: "")
    }
    
    func localized(with arguments: CVarArg...) -> String {
        String(format: self.localized, arguments: arguments)
    }
}
```

## 中文文案规范
- **界面文案：** 简洁、口语化，避免过于书面
- **错误提示：** 说明问题 + 提供操作指引
  - ❌ "网络错误"
  - ✅ "网络连接失败，请检查网络后重试"
- **按钮文案：** 动词开头，2-4 个字
  - ✅ "开始录制" "保存" "稍后再说"
  - ❌ "点击开始录制" "确认保存文件"
- **空状态文案：** 说明原因 + 引导操作
  - ✅ "还没有录制内容\n点击下方按钮开始"

## 英文文案规范
- Title Case 用于标题和按钮
- Sentence case 用于描述和提示
- 时态：按钮用动词原形（"Start Recording"），描述用现在时

## 动态文案（带参数）
```swift
// Localizable.strings
"paywall.trial.message" = "免费试用 %d 天";

// 使用
let text = "paywall.trial.message".localized(with: 7)
// → "免费试用 7 天"
```

## 注意事项
- 中英文字符串长度差异大，UI 布局用弹性约束（不要固定宽度）
- 日期格式：使用 `DateFormatter`，不要硬编码格式
  ```swift
  formatter.dateStyle = .medium
  formatter.timeStyle = .short
  formatter.locale = Locale.current  // 自动适配语言
  ```
- 数字格式：金额使用 `NumberFormatter`，不要手动拼接
