---
name: essential-community-skills
description: 安装、配置、推荐社区 Claude Code Skills，或需要 xcodebuild、SwiftUI、设备参数、HIG、App Store 上架、后端搭建、Supabase、Firebase、Vapor、Cloudflare Workers 等社区技能支持
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

## 六、后端搭建（BaaS / Serverless / 自建）

> 💡 后端编码规范（Supabase RLS / Vapor 控制器 / Cloudflare Workers / 数据库设计 / 认证流程等）已独立为 **`10-backend-server`** Skill，开发后端时请同时启用。

以下为社区可安装的后端相关 Skills：

### 11. Supabase Agent Skills（官方出品 ⭐ 必装）

- **仓库**：https://github.com/supabase/agent-skills
- **技能名**：`supabase`（完整包）+ `postgres-best-practices`（PostgreSQL 专项）
- **用途**：覆盖 Supabase 全产品线 — PostgreSQL RLS 策略、索引/迁移、Auth 认证、Edge Functions、Realtime、Storage、Vectors、Cron、Queues。内联安全检查清单（禁止 service_role 暴露前端、视图必须 security_invoker=true 等），自动查文档再实现
- **安装**：
  ```bash
  npx skills add supabase/agent-skills
  ```
- **适用场景**：iOS 应用后端首选 BaaS，PostgreSQL + RLS 天然适合移动端，免费额度充足，官方维护

### 12. Server-Side Swift（Vapor + Hummingbird ⭐ 必装）

- **仓库**：https://github.com/melissa-pereira-deel/claude-code-server-side-swift-skills
- **技能名**：`server-side-swift`
- **用途**：Vapor 4 路由/控制器/中间件/WebSocket、Hummingbird 2 模式、Fluent ORM 数据库模式、JWT 认证/Sign in with Apple 服务端、Docker 多阶段部署、Fly.io/Railway 部署、Swift 客户端↔服务端共享代码包
- **安装**：
  ```bash
  npx skills add https://github.com/melissa-pereira-deel/claude-code-server-side-swift-skills --skill server-side-swift
  ```
- **适用场景**：用 Swift 写后端，共享 Model/DTO 代码，Sign in with Apple 服务端验证

### 13. Cloudflare Skills（官方出品 ⭐ 必装）

- **仓库**：https://github.com/cloudflare/skills
- **技能名**：`cloudflare`（综合）+ `agents-sdk` + `durable-objects` 等 21 个子技能
- **用途**：覆盖 Workers、Pages、KV、D1、R2、Workers AI、Vectorize、Agents SDK、Email、WAF 等全部 Cloudflare 产品，自动查文档
- **安装**：
  ```bash
  npx skills add cloudflare/skills
  ```
- **适用场景**：最轻量 Serverless 方案，免费额度大，适合 iOS 独立开发者快速搭建 API 后端

### 14. Firebase Firestore Setup

- **仓库**：https://github.com/TovTechOrg/tovtech-skills
- **技能名**：`setup-firebase`
- **用途**：自动创建 firebase.json / firestore.rules / firestore.indexes.json，引导创建 Firebase 项目，部署安全规则，测试读写连接
- **安装**：
  ```bash
  # 手动复制到项目
  cp -r skills/setup-firebase ~/.claude/skills/
  ```
- **适用场景**：Firebase 是 iOS 开发者最常用的 BaaS，此 Skill 自动化 Firestore 从零搭建

### 15. Backend API（RESTful 设计）

- **仓库**：https://github.com/maksimtereshin/biz-assess-platform
- **技能名**：`backend-api`
- **用途**：RESTful API 端点设计，HTTP 方法/状态码/资源路由，API 版本化、限流、嵌套资源、过滤/排序/分页，支持 Express / NestJS / FastAPI / Django REST Framework
- **安装**：
  ```bash
  npx skills add https://github.com/maksimtereshin/biz-assess-platform --skill backend-api
  ```
- **适用场景**：为 iOS 应用设计 REST API，确保 API 设计规范、URL 结构清晰

### 16. Hono Routing（轻量级 API 框架）

- **仓库**：https://github.com/secondsky/claude-skills
- **技能名**：`hono-routing`
- **用途**：Hono 框架路由模式，支持 Cloudflare Workers / Bun / Node.js 多运行时
- **安装**：
  ```bash
  npx skills add https://github.com/secondsky/claude-skills --skill hono-routing
  ```
- **适用场景**：配合 Cloudflare Workers 用极低成本搭建 iOS 应用 API 后端

---

## 七、认证与安全

### 17. Better Auth（现代认证框架）

- **仓库**：https://github.com/secondsky/claude-skills
- **技能名**：`better-auth`
- **用途**：Better Auth 框架集成，支持多种 OAuth 提供商、邮箱/密码、双因素认证、会话管理，支持 Sign in with Apple
- **安装**：
  ```bash
  npx skills add https://github.com/secondsky/claude-skills --skill better-auth
  ```
- **适用场景**：iOS 应用后端认证，Sign in with Apple 服务端集成

### 18. Security 全套（API + Backend + Database）

- **仓库**：https://github.com/steveantini/claude-templates
- **技能名**：`api-security` + `backend-security` + `database-security`
- **用途**：JWT 验证、API Key 管理/轮换、OAuth2 Authorization Code + PKCE、CORS、速率限制、密码哈希（Argon2id）、SQL 注入防护、HTTPS/TLS 配置
- **安装**：
  ```bash
  # 手动复制 skills/ 目录
  cp -r skills/api-security ~/.claude/skills/
  cp -r skills/backend-security ~/.claude/skills/
  cp -r skills/database-security ~/.claude/skills/
  ```
- **适用场景**：iOS 后端安全全面覆盖，特别是 OAuth2 PKCE 流程（iOS 原生认证推荐方式）

---

## 八、DevOps 与部署

### 19. DevOps Engineer

- **仓库**：https://github.com/jeffallan/claude-skills
- **技能名**：`devops-engineer`
- **用途**：Dockerfile 编写、CI/CD 管道（GitHub Actions / GitLab CI / Jenkins）、Kubernetes 部署、Terraform / Pulumi IaC、蓝绿/金丝雀部署、云平台配置（AWS / GCP / Azure）
- **安装**：
  ```bash
  npx skills add https://github.com/jeffallan/claude-skills --skill devops-engineer
  ```
- **适用场景**：后端容器化部署、CI/CD 流水线搭建、基础设施即代码

### 20. AWS Solution Architect

- **仓库**：https://github.com/alirezarezvani/claude-code-skill-factory
- **技能名**：`aws-solution-architect`
- **用途**：AWS 全栈架构设计 — Lambda / API Gateway / DynamoDB 无服务器、CloudFormation / CDK / Terraform IaC、Cognito 认证、ECS / EKS 容器、CloudFront CDN、成本优化
- **安装**：
  ```bash
  # 手动复制
  cp -r generated-skills/aws-solution-architect ~/.claude/skills/
  ```
- **适用场景**：选择 AWS 作为后端基础设施时的架构设计和 IaC 模板

---

## 九、通用工程技能

### 21. Skill Creator（技能创建工具）

- **仓库**：Anthropic 官方
- **技能名**：`skill-creator`
- **用途**：交互式创建新的 Claude Code Skill，引导你写出规范的 SKILL.md
- **安装**：Claude Code 内置，直接使用即可
- **适用场景**：为你的项目创建自定义 Skill

### 22. MCP Builder（MCP 服务器开发）

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

### 🖥️ 后端开发（按方案选择）

**方案 A：Supabase（推荐，开箱即用）**
```bash
npx skills add supabase/agent-skills
```

**方案 B：Server-Side Swift（Swift 全栈）**
```bash
npx skills add https://github.com/melissa-pereira-deel/claude-code-server-side-swift-skills --skill server-side-swift
```

**方案 C：Cloudflare Workers（最轻量 Serverless）**
```bash
npx skills add cloudflare/skills
npx skills add https://github.com/secondsky/claude-skills --skill hono-routing
```

**方案 D：Firebase（Google 生态）**
```bash
# 手动复制
cp -r skills/setup-firebase ~/.claude/skills/
```

**安全加固（所有方案通用）**
```bash
npx skills add https://github.com/secondsky/claude-skills --skill better-auth
cp -r skills/api-security ~/.claude/skills/
cp -r skills/backend-security ~/.claude/skills/
```

---

## 更多资源

| 资源 | 地址 | 说明 |
|------|------|------|
| awesome-claude-skills | https://github.com/travisvn/awesome-claude-skills | 最全的 Claude Skills 精选清单 |
| Claude Skills Hub | https://claudeskills.info/agent-skills/directory/ | 658+ 技能索引，按分类搜索 |
| Agent Skills 标准 | https://agentskills.io/ | Agent Skills 官方规范文档 |
| Anthropic 官方 Skills | https://github.com/anthropics/skills | 官方技能仓库（pdf/xlsx/mcp-builder 等） |
