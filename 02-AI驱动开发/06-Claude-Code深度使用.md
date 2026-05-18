# 06-Claude Code 深度使用

> 🎯 **本章目标**：掌握 Claude Code 的安装配置、项目上下文管理、CLAUDE.md 规范文件编写、常用工作流和最佳实践，能够用 Claude Code 高效完成日常开发任务。

---

## Claude Code 是什么？

### Anthropic 推出的终端 AI 编程助手

Claude Code 是 Anthropic 官方推出的**终端 AI 编程助手**。简单来说，它就是一个住在终端里的"AI 程序员"——你用自然语言告诉它要做什么，它就能直接读取你的项目文件、理解代码逻辑、修改文件、运行命令。

它和你在浏览器里用的 Claude 网页版完全不同：

| 对比维度 | Claude 网页版 | Claude Code |
|---------|-------------|-------------|
| 运行环境 | 浏览器 | 终端（命令行） |
| 访问你的文件 | ❌ 不能，需要手动上传 | ✅ 能，直接读取本地文件 |
| 修改你的文件 | ❌ 不能 | ✅ 能，直接编辑和创建文件 |
| 运行命令 | ❌ 不能 | ✅ 能，执行终端命令 |
| 理解项目上下文 | ❌ 有限，只能看上传的文件 | ✅ 自动扫描整个项目 |
| 适合场景 | 问答、讨论方案 | 实际编码、项目开发 |

💡 **一句话理解**：Claude 网页版是"AI 顾问"，你问它答；Claude Code 是"AI 程序员"，你说它做。

### 核心优势：理解整个项目上下文、直接操作文件

Claude Code 有三大核心优势，让它在 AI 编程工具中独树一帜：

1. **理解整个项目**：Claude Code 会自动扫描你的项目目录，理解文件结构、代码关系、依赖关系，而不是只看单个文件
2. **直接操作文件**：它可以读取、创建、编辑、删除文件，不需要你手动复制粘贴
3. **执行终端命令**：它可以运行测试、安装依赖、启动服务器等，完成完整的开发循环

```
传统方式：
  你：在浏览器问 Claude → Claude 生成代码 → 你复制 → 你粘贴到编辑器 → 你手动运行

Claude Code 方式：
  你：在终端说"给用户模块加上登录功能" → Claude Code 读完项目 → 直接改文件 → 自动运行测试
```

⚠️ **注意**：Claude Code 通过 API 调用 Claude 模型，需要按使用量付费。费用取决于你的使用量，通常每天几美元左右。使用前请确保你了解计费方式。

---

## 安装与配置

### 环境要求

在安装 Claude Code 之前，你需要确保电脑上已经安装了以下软件：

| 软件 | 要求 | 如何检查 | 如何安装 |
|------|------|---------|---------|
| Node.js | 18.0 或更高版本 | 终端输入 `node -v` | 访问 [nodejs.org](https://nodejs.org) 下载安装 |
| npm | 随 Node.js 一起安装 | 终端输入 `npm -v` | 安装 Node.js 后自动可用 |

💡 **提示**：如果你不确定是否已安装，先在终端里运行检查命令。如果显示版本号（如 `v20.11.0`），说明已安装；如果显示 `command not found`，说明还没安装。

检查命令：

```bash
node -v
npm -v
```

如果 Node.js 版本低于 18，需要先升级。推荐使用 `nvm`（Node Version Manager）来管理 Node.js 版本：

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.zshrc
nvm install 20
nvm use 20
```

### 安装步骤

确认环境就绪后，一行命令安装 Claude Code：

```bash
npm install -g @anthropic-ai/claude-code
```

安装完成后，验证是否安装成功：

```bash
claude --version
```

如果显示版本号，说明安装成功 ✅

⚠️ **常见问题**：

| 问题 | 原因 | 解决方法 |
|------|------|---------|
| `EACCES` 权限错误 | npm 全局安装权限不足 | 在命令前加 `sudo`：`sudo npm install -g @anthropic-ai/claude-code` |
| `command not found: claude` | 安装路径未加入 PATH | 重新打开终端，或检查 npm 全局安装路径 |
| 网络超时 | 国内网络访问 npm 较慢 | 使用镜像：`npm install -g @anthropic-ai/claude-code --registry=https://registry.npmmirror.com` |

### API Key 配置

Claude Code 需要你的 Anthropic API Key 才能工作。API Key 就像是你的"通行证"，让 Claude Code 知道是谁在调用。

**获取 API Key 的步骤**：

1. 访问 [console.anthropic.com](https://console.anthropic.com)
2. 注册/登录 Anthropic 账号
3. 进入 API Keys 页面
4. 点击 "Create Key" 创建新的 API Key
5. 复制生成的 Key（以 `sk-ant-` 开头）

⚠️ **重要**：API Key 是你的私密信息，**绝对不要**分享给别人或提交到 Git 仓库！一旦泄露，别人可以用你的账号产生费用。

**配置 API Key**：

方法一：设置环境变量（推荐）

```bash
# 临时设置（当前终端窗口有效）
export ANTHROPIC_API_KEY=sk-ant-your-key-here

# 永久设置（写入 shell 配置文件）
echo 'export ANTHROPIC_API_KEY=sk-ant-your-key-here' >> ~/.zshrc
source ~/.zshrc
```

方法二：首次运行时输入

```bash
claude
# 首次运行会提示你输入 API Key
```

💡 **提示**：推荐使用方法一，这样每次打开终端都不需要重新输入。记得把 `sk-ant-your-key-here` 替换成你自己的真实 Key。

### 首次运行

配置好 API Key 后，进入你的项目目录，启动 Claude Code：

```bash
# 进入你的项目目录
cd /path/to/your/project

# 启动 Claude Code
claude
```

首次启动时，Claude Code 会：
1. 扫描当前目录的项目结构
2. 读取关键文件（如 package.json、README 等）
3. 进入交互模式，等待你的指令

你会看到类似这样的界面：

```
╭──────────────────────────────────────╮
│                                      │
│   Welcome to Claude Code!            │
│                                      │
│   Type your request and press Enter  │
│                                      │
╰──────────────────────────────────────╯
>
```

试试输入你的第一个指令：

```
> 请介绍一下这个项目的结构
```

Claude Code 会分析你的项目并给出回答。如果项目是空的，它会告诉你当前目录没有项目文件。

---

## 项目上下文管理

### Claude Code 如何理解项目

Claude Code 不是"瞎猜"你的项目结构，它会主动扫描和分析你的项目。理解它的工作方式，能帮你更好地使用它：

```
Claude Code 的"理解"过程：

1. 扫描目录结构 → 知道有哪些文件和文件夹
2. 读取关键文件 → package.json、README、配置文件等
3. 分析代码关系 → 哪个文件引用了哪个文件
4. 理解技术栈 → 用了什么框架、什么语言
5. 结合你的指令 → 在理解项目的基础上执行任务
```

### 上下文窗口概念

Claude Code 能"看到"的信息量是有限的，这个限制叫做**上下文窗口**。你可以把它想象成 Claude 的"短期记忆"：

| 概念 | 解释 | 类比 |
|------|------|------|
| 上下文窗口 | Claude 一次能处理的最大文本量 | 人的短期记忆容量 |
| Token | 文本的最小单位（约 1 个中文字 ≈ 2-3 token） | 记忆的基本单位 |
| 200K token | Claude 能处理约 15 万字的内容 | 相当于一本中等长度的书 |

💡 **关键点**：Claude Code 会智能地选择最相关的文件放入上下文窗口，而不是把所有文件都塞进去。但如果你项目特别大，它可能无法同时"看到"所有文件。

### 如何引导 Claude Code 关注关键文件

当项目较大时，你可以主动引导 Claude Code 关注最重要的文件：

**方法一：在对话中直接指定文件**

```
> 请看一下 src/utils/auth.js 和 src/pages/Login.vue，帮我修复登录逻辑的 bug
```

**方法二：使用 CLAUDE.md 文件（下一节详细介绍）**

在项目根目录创建 `CLAUDE.md` 文件，告诉 Claude Code 项目的关键信息。

**方法三：先让 Claude Code 了解项目**

```
> 先阅读 README.md 和 package.json，了解这个项目的技术栈和结构
```

⚠️ **注意**：不要一次让 Claude Code 处理太多不相关的文件。聚焦于当前任务相关的文件，效果会更好。

---

## CLAUDE.md 项目规范文件

### 什么是 CLAUDE.md

`CLAUDE.md` 是 Claude Code 的**项目规范文件**。你可以把它理解为给 AI 程序员的"入职手册"——它告诉 Claude Code 你的项目是什么、用什么技术栈、遵循什么编码规范、目录结构是怎样的。

每次启动 Claude Code 时，它都会**自动读取**项目根目录的 `CLAUDE.md` 文件，并按照里面的规范来工作。

💡 **为什么需要 CLAUDE.md？**

没有 CLAUDE.md：
```
你：帮我写一个用户注册接口
Claude：好的，我用 Express 写一个...（但你项目用的是 Koa！）
```

有 CLAUDE.md：
```
你：帮我写一个用户注册接口
Claude：根据项目规范，我用 Koa + TypeScript 写一个注册接口...
```

### 放在哪里

```
your-project/
├── CLAUDE.md          ← 放在项目根目录
├── package.json
├── src/
├── public/
└── ...
```

Claude Code 启动时会自动查找并读取这个文件。

### 应该包含什么内容

一个完整的 `CLAUDE.md` 通常包含以下部分：

| 内容 | 重要性 | 说明 |
|------|--------|------|
| 项目描述 | ⭐⭐⭐⭐⭐ | 一句话说清楚项目是什么 |
| 技术栈 | ⭐⭐⭐⭐⭐ | 用的语言、框架、数据库等 |
| 编码规范 | ⭐⭐⭐⭐ | 命名规则、代码风格等 |
| 目录结构说明 | ⭐⭐⭐⭐ | 每个目录放什么文件 |
| 常用命令 | ⭐⭐⭐ | 构建、测试、启动命令 |
| 注意事项 | ⭐⭐⭐ | 特殊约定、不要做的事 |

### CLAUDE.md 模板示例

下面是一个可以直接使用的模板，根据你的项目修改即可：

```markdown
# 项目规范

## 项目描述
这是一个基于 React + TypeScript 的任务管理 Web 应用，支持任务的创建、编辑、删除和分类管理。

## 技术栈
- 前端框架：React 18 + TypeScript
- 状态管理：Zustand
- 样式方案：Tailwind CSS
- 构建工具：Vite
- 后端 API：RESTful，基路径 /api/v1
- 数据库：SQLite（开发环境）

## 编码规范
- 使用函数式组件和 Hooks，不使用 class 组件
- 文件命名：组件用 PascalCase（如 TaskList.tsx），工具函数用 camelCase（如 formatDate.ts）
- 变量命名：使用 camelCase，常量使用 UPPER_SNAKE_CASE
- 类型定义：优先使用 interface，复杂联合类型使用 type
- 错误处理：使用 try-catch 包裹异步操作，统一用 toast 提示错误
- 注释：只在复杂逻辑处添加注释，不写显而易见的注释

## 目录结构
- src/components/ — 可复用的 UI 组件
- src/pages/ — 页面级组件，每个页面一个文件夹
- src/stores/ — Zustand 状态管理
- src/utils/ — 工具函数
- src/types/ — TypeScript 类型定义
- src/api/ — API 请求封装

## 常用命令
- npm run dev — 启动开发服务器
- npm run build — 构建生产版本
- npm run test — 运行测试
- npm run lint — 代码检查

## 注意事项
- 不要修改 src/api/config.ts 中的 API 基础地址
- 所有 API 请求必须经过 src/utils/request.ts 统一处理
- 样式统一使用 Tailwind 类名，不要写内联样式
- 提交代码前必须运行 npm run lint 确保没有错误
```

💡 **提示**：CLAUDE.md 不需要一次写完美。你可以先写基本内容，随着项目开发逐步补充。每次发现 Claude Code 犯了同样的错误，就把相关规范加到 CLAUDE.md 里。

---

## 常用命令与工作流

### claude：启动交互模式

交互模式是最常用的方式，适合需要多轮对话的复杂任务：

```bash
# 进入项目目录
cd /path/to/your/project

# 启动交互模式
claude
```

进入交互模式后，你可以连续输入多个指令，Claude Code 会记住之前的对话上下文：

```
> 请看一下这个项目的结构
（Claude Code 分析项目并回答）

> 帮我在 src/components/ 下创建一个 Button 组件
（Claude Code 创建文件）

> 再给它加上 loading 状态的支持
（Claude Code 修改刚才创建的文件）
```

💡 **提示**：在交互模式中，你可以用 `/` 开头的快捷命令：

| 快捷命令 | 功能 |
|---------|------|
| `/help` | 查看帮助信息 |
| `/clear` | 清除对话历史 |
| `/compact` | 压缩对话历史，释放上下文空间 |
| `/cost` | 查看当前会话的费用 |
| `/quit` | 退出 Claude Code |

### claude "task"：直接执行任务

如果你只需要执行一个简单任务，不需要多轮对话，可以直接在命令后面跟上任务描述：

```bash
# 一次性任务：直接执行完就退出
claude "给 package.json 添加 dayjs 依赖"

# 查看某个文件的逻辑
claude "解释 src/utils/auth.js 的登录流程"

# 修复 lint 错误
claude "运行 npm run lint 并修复所有错误"
```

💡 **提示**：单次任务模式适合简单、明确的操作。如果任务复杂需要多轮交互，还是用交互模式更好。

### 常用工作流示例

**工作流 1：创建新功能**

```
> 我需要添加一个"用户个人设置"页面，包含修改头像、昵称和密码的功能

（Claude Code 会：）
1. 分析现有项目结构，了解路由和组件的组织方式
2. 创建页面组件
3. 添加路由配置
4. 创建 API 请求函数
5. 添加状态管理
6. 运行构建检查是否有错误
```

**工作流 2：修复 Bug**

```
> 用户反馈：点击"保存"按钮后页面没有反应，控制台报错 "Cannot read property 'id' of undefined"

（Claude Code 会：）
1. 定位到相关的保存逻辑代码
2. 分析错误原因
3. 修复代码
4. 检查是否有类似的问题
```

**工作流 3：代码重构**

```
> src/utils/helpers.js 文件太大了（800行），请把它拆分成多个模块，按功能分类

（Claude Code 会：）
1. 分析文件中的所有函数
2. 按功能分类
3. 创建多个新文件
4. 更新所有引用
5. 运行测试确保没有破坏
```

---

## 多文件编辑与重构

### 让 Claude Code 修改多个文件

Claude Code 的一大优势是能同时修改多个文件。你只需要清楚地描述需求，它会自动处理文件间的关联：

```
> 请把项目中的所有 console.log 调试语句替换成统一的 logger 工具函数
```

Claude Code 会：
1. 扫描项目中所有包含 `console.log` 的文件
2. 创建 `src/utils/logger.ts` 工具函数
3. 逐一替换每个文件中的 `console.log`
4. 确保所有引用正确

💡 **提示**：让 Claude Code 修改多个文件时，最好明确说明你期望的结果，而不是只说"改一下"。比如：

| ❌ 模糊的指令 | ✅ 清晰的指令 |
|-------------|-------------|
| "改一下用户模块" | "在用户模块中添加邮箱验证功能，修改注册接口和前端表单" |
| "优化代码" | "把 UserService 类拆分成 UserAuthService 和 UserProfileService" |
| "修一下样式" | "把登录页面的按钮颜色改成蓝色，并增加 hover 效果" |

### 重构代码的最佳实践

重构是 Claude Code 非常擅长的工作，但需要遵循一些最佳实践：

**1. 一次只重构一件事**

```
✅ 好的做法：
> 把 UserService 拆分成 AuthService 和 ProfileService

❌ 不好的做法：
> 重构整个用户模块，顺便加上权限系统和日志系统
```

**2. 先测试，再重构**

```
> 先运行一下现有的测试，确保都通过。然后开始重构 UserService，重构完再跑一次测试。
```

**3. 分步进行，逐步确认**

```
> 第一步：分析 UserService 的所有方法，列出拆分方案
（确认方案后）
> 第二步：按照刚才的方案，创建 AuthService
（确认后）
> 第三步：创建 ProfileService
（确认后）
> 第四步：更新所有引用，运行测试
```

### 审查修改内容

Claude Code 在修改文件前会显示它打算做的修改，你可以选择接受或拒绝：

```
Claude Code: 我将修改以下文件：
  - src/services/UserService.ts（添加邮箱验证方法）
  - src/pages/Register.tsx（添加邮箱输入框）
  - src/api/user.ts（添加验证接口）

是否继续？[Y/n]
```

💡 **建议**：养成审查修改的习惯。不要无脑按 Y，至少看看它改了哪些文件，确保没有误改重要文件。

你也可以在对话中要求 Claude Code 解释它的修改：

```
> 修改前，先告诉我你打算怎么改，等我确认后再动手
```

---

## 调试与错误修复

### 把错误信息发给 Claude Code

遇到报错时，最有效的方法是直接把错误信息发给 Claude Code：

```
> 运行 npm run dev 时报错了，错误信息如下：

TypeError: Cannot read properties of undefined (reading 'map')
    at TaskList (src/components/TaskList.tsx:25:18)
    at renderWithHooks (node_modules/react-dom/cjs/react-dom.development.js:16317:18)
```

Claude Code 会：
1. 定位到 `src/components/TaskList.tsx` 第 25 行
2. 分析为什么数据是 `undefined`
3. 修复代码（比如添加空值检查或确保数据初始化）

💡 **提示**：给 Claude Code 错误信息时，尽量提供**完整的错误堆栈**，不要只截取第一行。堆栈信息能帮助它精确定位问题。

### 让 Claude Code 分析崩溃日志

如果应用崩溃了，你可以让 Claude Code 分析日志：

```
> 应用启动后访问 /dashboard 页面会崩溃，这是浏览器控制台的日志：

Error: Minified React error #301
    at dg (main.a1b2c3.js:2:345678)
    at ...

> 同时这是服务端的日志：

[ERROR] 2025-01-15 10:23:45 - GET /api/dashboard - 500 Internal Server Error
    TypeError: Cannot read property 'user_id' of null
        at DashboardController.getDashboard (src/controllers/dashboard.ts:42:25)
```

Claude Code 会综合分析前端和后端的错误信息，找到根本原因。

### 逐步调试

对于难以定位的问题，可以让 Claude Code 逐步排查：

```
> 用户反馈"偶尔会丢失数据"，我无法稳定复现。请帮我排查可能的原因。

（Claude Code 会：）
1. 检查数据存储逻辑，看是否有竞态条件
2. 检查网络请求的错误处理
3. 检查状态管理是否有覆盖问题
4. 列出所有可能的原因和修复建议
```

你也可以让 Claude Code 添加调试代码：

```
> 在 saveTask 函数中添加详细的日志，帮我追踪数据保存的完整流程
```

⚠️ **注意**：调试完成后，记得让 Claude Code 移除调试代码，不要把临时的 console.log 留在生产代码中。

---

## 最佳实践与避坑指南

### 10 条最佳实践

**1. 写好 CLAUDE.md**

这是最重要的一条。好的 CLAUDE.md 能让 Claude Code 的输出质量提升 50% 以上。每次发现 Claude Code 犯了重复错误，就把规范加到 CLAUDE.md 里。

**2. 一次只做一件事**

不要在一个指令里塞太多需求。复杂任务拆分成多个步骤，逐步执行。

**3. 先理解，再动手**

让 Claude Code 先分析代码再修改，而不是上来就改：

```
✅ "先看一下 src/auth.ts 的登录逻辑，然后告诉我你打算怎么添加邮箱验证功能"
❌ "给登录加上邮箱验证"
```

**4. 提供足够的上下文**

```
✅ "在 src/pages/Dashboard.tsx 中，用户点击'导出'按钮后，调用 src/api/export.ts 的 exportData 方法，将数据导出为 CSV 文件"
❌ "加个导出功能"
```

**5. 审查每一处修改**

不要盲目接受所有修改。至少检查它修改了哪些文件，关键逻辑是否正确。

**6. 用 Git 管理变更**

在让 Claude Code 做大改动之前，先提交当前代码：

```bash
git add .
git commit -m "保存当前状态"
```

这样如果 Claude Code 改坏了，你可以轻松回退：

```bash
git checkout .
```

**7. 让它写测试**

```
> 给刚才写的 formatDate 函数添加单元测试
```

**8. 善用 /compact 命令**

对话太长时，Claude Code 的上下文会被占满。使用 `/compact` 压缩历史对话，释放空间：

```
> /compact
```

**9. 明确说"不要做什么"**

```
> 给用户列表添加搜索功能，但不要修改现有的分页逻辑
```

**10. 渐进式开发**

不要让 Claude Code 一次写完整个功能。先搭框架，再逐步完善：

```
> 第一步：创建用户设置页面的基本框架，只包含页面结构和路由
（确认后）
> 第二步：添加修改昵称的功能
（确认后）
> 第三步：添加修改密码的功能
```

### 常见陷阱

| 陷阱 | 表现 | 解决方法 |
|------|------|---------|
| 🕳️ 幻觉代码 | Claude Code 引用了不存在的函数或库 | 让它先检查文件是否存在，运行代码验证 |
| 🕳️ 过度修改 | 你只想改一个地方，它改了十个文件 | 明确说"只修改 xxx，不要动其他文件" |
| 🕳️ 忘记上下文 | 对话太长后，它忘了之前的约定 | 使用 CLAUDE.md 记录关键规范，用 /compact 压缩对话 |
| 🕳️ 不一致的代码风格 | 生成的代码风格和项目不一致 | 在 CLAUDE.md 中明确编码规范 |
| 🕳️ 破坏现有功能 | 修改后其他功能不工作了 | 每次修改后运行测试，用 Git 管理变更 |
| 🕳️ 敏感信息泄露 | 把 API Key 写进代码 | 在 CLAUDE.md 中注明"不要在代码中硬编码密钥" |
| 🕳️ 盲目接受修改 | 不看就按 Y，导致问题 | 养成审查习惯，重要修改先看 diff |

⚠️ **最重要的提醒**：Claude Code 是你的助手，不是你的替代者。**你始终是代码的负责人**。每一行提交到项目的代码，你都应该理解它在做什么。

---

## 小结

本章我们深入学习了 Claude Code 的使用方法：

1. **Claude Code 是终端里的 AI 程序员**，能直接读取和修改你的项目文件，执行终端命令
2. **安装只需一行命令**：`npm install -g @anthropic-ai/claude-code`，配置好 API Key 即可使用
3. **CLAUDE.md 是关键**——写好项目规范文件，能让 Claude Code 的输出质量大幅提升
4. **交互模式适合复杂任务**，单次命令适合简单操作
5. **多文件编辑和重构**是 Claude Code 的强项，但要注意审查修改内容
6. **调试时提供完整错误信息**，让 Claude Code 精确定位问题
7. **遵循最佳实践**：写好 CLAUDE.md、一次做一件事、用 Git 管理变更、审查每一处修改

> 📖 **下一步**：掌握了 AI 编程工具的选择和 Claude Code 的使用后，让我们进入实战环节，学习如何用 AI 驱动的方式完成一个完整的 App 开发。👉 [07-AI驱动开发实战](./07-AI驱动开发实战.md)
