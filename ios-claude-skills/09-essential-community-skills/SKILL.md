---
name: essential-community-skills
description: 安装、配置、推荐社区 Claude Code Skills，或需要 xcodebuild、SwiftUI、设备参数、HIG、App Store 上架等社区技能支持
---

# 必备社区 Skills 清单

以下为公开可访问、经过社区验证、对 iOS 开发者高度实用的 Claude Code Skills。按场景分类，附带安装命令。

---

## 一、构建与项目脚手架

### 1. iOS App Builder（XcodeGen + SPM）

- **仓库**：https://github.com/daymade/claude-code-skills
- **技能名**：`developing-ios-apps`
- **用途**：XcodeGen project.yml 配置、代码签名（免费/付费账号）、iOS 版本兼容性对照、SPM 动态框架嵌入问题排查、CI/CD 签名管道
- **安装**：
  ```bash
  npx skills add https://github.com/daymade/claude-code-skills --skill developing-ios-apps
  ```
- **适用场景**：项目初始化、构建失败排查、签名配置、SPM 依赖冲突

### 2. iOS App Scaffold（XcodeGen 项目骨架）

- **仓库**：https://github.com/avwohl/claude-skills
- **技能名**：`ios-app-scaffold`
- **用途**：用 XcodeGen 从零创建 iOS / Mac Catalyst 应用，含完整 project.yml 模板和目录结构
- **安装**：
  ```bash
  npx skills add https://github.com/avwohl/claude-skills --skill ios-app-scaffold
  ```
- **适用场景**：新项目搭建、多 Target 配置、Mac Catalyst 移植

---

## 二、SwiftUI 与界面

### 3. SwiftUI Pro（Paul Hudson 出品）

- **仓库**：https://github.com/twostraws/swiftui-agent-skill
- **技能名**：`swiftui-pro`
- **用途**：捕获 SwiftUI API 的细微错误，提供 SwiftUI 最佳实践，覆盖视图生命周期、状态管理、性能优化
- **安装**：
  ```bash
  npx skills add https://github.com/twostraws/swiftui-agent-skill --skill swiftui-pro
  ```
- **适用场景**：SwiftUI 视图开发、API 用法确认、性能问题排查

### 4. Device Geometry（设备参数精确参考）

- **仓库**：https://github.com/avwohl/claude-skills
- **技能名**：`device-geometry`
- **用途**：iPhone / iPad 全系屏幕尺寸、圆角半径、安全区域、刘海/灵动岛参数，含超椭圆(n=5)公式
- **安装**：
  ```bash
  npx skills add https://github.com/avwohl/claude-skills --skill device-geometry
  ```
- **适用场景**：精确 UI 布局、适配刘海/圆角、多设备尺寸适配

### 5. Apple HIG（人机交互指南参考）

- **仓库**：https://github.com/avwohl/claude-skills
- **技能名**：`apple-hig`
- **用途**：Apple HIG 完整参考 — Dynamic Type 排版、颜色系统、8pt 网格、组件高度、无障碍清单、Liquid Glass 设计系统、12 个常见违规
- **安装**：
  ```bash
  npx skills add https://github.com/avwohl/claude-skills --skill apple-hig
  ```
- **适用场景**：UI 设计合规审查、HIG 违规修正、设计系统搭建

---

## 三、Swift 语言与架构

### 6. Swift Expert（语言专家级指导）

- **仓库**：https://github.com/jeffallan/claude-skills
- **技能名**：`swift-expert`
- **用途**：协议优先架构、async/await 正确/错误模式、SwiftUI 状态管理、Actor 线程安全、XCTest 异步测试
- **安装**：
  ```bash
  npx skills add https://github.com/jeffallan/claude-skills --skill swift-expert
  ```
- **适用场景**：Swift 高级语法、并发问题、架构设计、测试编写

### 7. Swift Concurrency（并发专项）

- **仓库**：https://github.com/jamesrochabrun/skills
- **技能名**：`swift-concurrency`
- **用途**：Swift 6+ 并发代码构建/审计/重构，async/await、actor、Sendable 最佳实践
- **安装**：
  ```bash
  npx skills add https://github.com/jamesrochabrun/skills --skill swift-concurrency
  ```
- **适用场景**：Swift 6 迁移、并发安全审计、Sendable 合规

---

## 四、全流程 iOS 技能包

### 8. iOS Agent Skills（18 个技能全覆盖）

- **仓库**：https://github.com/troyjthomas/ios-agent-skills
- **许可证**：MIT
- **技能数量**：18 个 iOS 专用技能 + 3 个 Agent 人设
- **一键安装**：
  ```bash
  curl -fsSL https://raw.githubusercontent.com/troyjthomas/ios-agent-skills/main/install.sh | bash
  ```
- **核心技能**：

| 技能 | 用途 |
|------|------|
| `app-vision` | 粗略想法 → 可执行的应用概念 |
| `app-spec` | 生成 CLAUDE.md + APP_SPEC.md |
| `design-system` | 颜色、排版、资源、品牌规则 |
| `figma-to-code` | Figma 设计稿转 SwiftUI |
| `scaffolding` | 一键生成完整 App 骨架 |
| `quality-gates` | 会话结束时的质量检查 |
| `code-review` | 自动化代码审查 |
| `app-store-prep` | App Store 提交检查清单 |
| `device-testing` | 设备测试完整清单 |
| `post-launch` | 上架后维护：崩溃报告、版本管理 |

- **适用场景**：从想法到上架全流程，特别适合独立开发者用 Claude Code 构建 SwiftUI 应用

---

## 五、App Store 与上架

### 9. iOS Accessibility（无障碍开发）

- **仓库**：https://github.com/dadederk/iOS-Accessibility-Agent-Skill
- **技能名**：`ios-accessibility`
- **用途**：VoiceOver、Dynamic Type、对比度、Reduce Motion 等 iOS 无障碍开发指导
- **安装**：
  ```bash
  npx skills add https://github.com/dadederk/iOS-Accessibility-Agent-Skill --skill ios-accessibility
  ```
- **适用场景**：App Store 审核无障碍合规、VoiceOver 适配、Dynamic Type 支持

### 10. Xcode Cloud CI/CD

- **仓库**：https://github.com/melissa-pereira-deel/claude-code-server-side-swift-skills
- **技能名**：`xcode-cloud`
- **用途**：Xcode Cloud CI/CD 配置、自定义构建脚本、工作流配置、TestFlight 集成
- **安装**：
  ```bash
  npx skills add https://github.com/melissa-pereira-deel/claude-code-server-side-swift-skills --skill xcode-cloud
  ```
- **适用场景**：CI/CD 流水线搭建、TestFlight 自动分发、构建脚本编写

---

## 六、通用工程技能

### 11. Skill Creator（技能创建工具）

- **仓库**：Anthropic 官方
- **技能名**：`skill-creator`
- **用途**：交互式创建新的 Claude Code Skill，引导你写出规范的 SKILL.md
- **安装**：Claude Code 内置，直接使用即可
- **适用场景**：为你的项目创建自定义 Skill

### 12. MCP Builder（MCP 服务器开发）

- **仓库**：Anthropic 官方（https://github.com/anthropics/skills）
- **技能名**：`mcp-builder`
- **用途**：MCP 服务器开发指南，构建自定义 MCP 集成
- **安装**：
  ```bash
  npx skills add https://github.com/anthropics/skills --skill mcp-builder
  ```
- **适用场景**：为 iOS 项目构建自定义 MCP 集成（数据库、Figma、GitHub 等）

---

## 推荐安装组合

根据开发阶段选择安装：

### 🟢 新手起步（最小集）

```bash
npx skills add https://github.com/avwohl/claude-skills --skill device-geometry
npx skills add https://github.com/avwohl/claude-skills --skill apple-hig
npx skills add https://github.com/twostraws/swiftui-agent-skill --skill swiftui-pro
```

### 🟡 独立开发者（推荐集）

```bash
# 最小集 + 构建 + 上架
npx skills add https://github.com/daymade/claude-code-skills --skill developing-ios-apps
npx skills add https://github.com/jeffallan/claude-skills --skill swift-expert
npx skills add https://github.com/dadederk/iOS-Accessibility-Agent-Skill --skill ios-accessibility
npx skills add https://github.com/melissa-pereira-deel/claude-code-server-side-swift-skills --skill xcode-cloud
```

### 🔴 全流程覆盖（完整集）

```bash
# 一键安装 troyjthomas 全套 18 个 iOS 技能
curl -fsSL https://raw.githubusercontent.com/troyjthomas/ios-agent-skills/main/install.sh | bash

# 再补充专项技能
npx skills add https://github.com/avwohl/claude-skills --skill device-geometry
npx skills add https://github.com/avwohl/claude-skills --skill apple-hig
npx skills add https://github.com/jamesrochabrun/skills --skill swift-concurrency
```

---

## 更多资源

| 资源 | 地址 | 说明 |
|------|------|------|
| awesome-claude-skills | https://github.com/travisvn/awesome-claude-skills | 最全的 Claude Skills 精选清单 |
| Claude Skills Hub | https://claudeskills.info/agent-skills/directory/ | 658+ 技能索引，按分类搜索 |
| Agent Skills 标准 | https://agentskills.io/ | Agent Skills 官方规范文档 |
| Anthropic 官方 Skills | https://github.com/anthropics/skills | 官方技能仓库（pdf/xlsx/mcp-builder 等） |
