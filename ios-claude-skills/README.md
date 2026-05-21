# iOS Claude Code Skills 工具包

> 10 个 SKILL.md 文件，开箱即用，覆盖独立 iOS 开发者的核心场景。

---

## 使用方法

### 方式一：项目级（推荐）
把 Skills 放入项目目录，只对当前项目生效：

```
your-ios-project/
└── .claude/
    └── skills/
        ├── 01-ui-framework/SKILL.md
        ├── 02-avfoundation-camera/SKILL.md
        └── ...
```

### 方式二：全局（所有项目生效）
```bash
mkdir -p ~/.claude/skills
cp -r ios-claude-skills/* ~/.claude/skills/
```

---

## Skills 列表

| # | 文件 | 触发场景 |
|---|------|---------|
| 01 | ui-framework | 界面、布局、ViewController、导航 |
| 02 | avfoundation-camera | 相机、录像、AVCaptureSession、多摄 |
| 03 | coreml-vision | CoreML、YOLO、CLIP、图像识别 |
| 04 | appstore-compliance | 订阅支付、权限、隐私、审核准备 |
| 05 | project-architecture | 新功能、重构、模块划分 |
| 06 | network-api | 网络请求、API、后端对接 |
| 07 | localization | 中英文本地化、文案规范 |
| 08 | subscription-paywall | IAP、StoreKit 2、Paywall 页面 |
| 09 | essential-community-skills | 社区必备 Skill 安装推荐、xcodebuild、SwiftUI Pro、设备参数、HIG、后端搭建（Supabase/Firebase/Vapor/Cloudflare Workers）、认证安全 |
| 10 | backend-server | 后端搭建、API 开发、数据库设计、Supabase、Vapor、Cloudflare Workers、认证、BaaS、Serverless |

---

## 自定义指南

每个 SKILL.md 顶部有两个关键字段：

```markdown
---
name: 技能名称
description: Claude Code 识别触发条件（越具体越好）
---
```

**建议根据你的项目修改：**
- `01-ui-framework`：修改布局库（SnapKit / 手写 AutoLayout）
- `05-project-architecture`：修改为你的实际目录结构和命名习惯
- `06-network-api`：修改后端地址、认证方式、响应格式
- `08-subscription-paywall`：修改为你的实际产品 ID
- `10-backend-server`：修改为你选用的后端方案（Supabase / Vapor / Cloudflare Workers / Firebase）

**踩到新坑就更新对应 Skill**，文件会越用越准。

---

## 技术栈假设

这套 Skills 基于以下技术栈，使用前确认是否匹配：

- UI：UIKit + SnapKit（非 SwiftUI）
- 架构：MVVM
- 包管理：Swift Package Manager
- 网络：URLSession（自封装）
- 存储：UserDefaults + CoreData + Keychain
- 订阅：StoreKit 2
- AI：OpenAI Whisper + Anthropic Claude（后端代理）

---

*配套公众号文章：《用 Claude Code 开发 iOS，这几个 Skill 必须装上》*
