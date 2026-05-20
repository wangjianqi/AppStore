# 12-MCP 协议与 AI 工具集成

## 本章目标

- 理解 MCP（Model Context Protocol）是什么，以及它如何让 AI 与工具"对话"
- 掌握 MCP 的核心架构：Host → Client → Server
- 学会在 Claude Desktop、Cursor、Trae 中配置 MCP Server
- 能够安装和使用常见的 MCP Server（文件系统、GitHub、Figma、Context7 等）
- 了解如何用 TypeScript / Python 创建自定义 MCP Server
- 掌握 MCP 使用中的安全注意事项和最佳实践

---

## 1. MCP 是什么

### 1.1 从 USB-C 说起

你一定用过 USB-C 接口——不管你是充电、传数据、接显示器，一根线搞定。在 USB-C 出现之前，我们有 Lightning、Micro-USB、Mini-USB……各种接口互不兼容，出门得带一坨转接头。

**MCP 就是 AI 世界的 USB-C。**

在 MCP 出现之前，每家 AI 应用想连接一个工具（比如 GitHub、数据库、文件系统），就得单独写一套集成代码。AI 应用 A 连 GitHub 用一套接口，AI 应用 B 连 GitHub 又得写另一套。这就像每个手机厂商都用自己的充电口一样混乱。

MCP 定义了一个**统一的协议标准**，让任何 AI 应用都能通过同一种方式连接任何工具服务。写一次 MCP Server，所有支持 MCP 的 AI 应用都能用。

| 类比 | USB-C 世界 | MCP 世界 |
|------|-----------|----------|
| 统一标准 | USB-C 协议 | MCP 协议 |
| 提供接口的设备 | 充电器、硬盘、显示器 | GitHub、Figma、数据库 |
| 使用接口的设备 | 手机、电脑 | Claude、Cursor、Trae |
| 连接线 | USB-C 线缆 | MCP Client |
| 好处 | 一根线连所有设备 | 一套协议连所有工具 |

### 1.2 为什么 MCP 对 iOS 开发者重要

作为 iOS 开发者，你的日常工作涉及大量工具：

- 🎨 **Figma**：看设计稿、量间距、取颜色
- 🐙 **GitHub**：提 PR、Review 代码、管理 Issue
- 📚 **第三方库文档**：查 Alamofire 怎么用、看 SwiftUI 最新 API
- 🗄️ **数据库**：查数据、验证接口返回
- 📁 **项目文件**：改配置、查日志、整理目录

传统方式下，你得在各个工具之间来回切换，手动复制粘贴信息给 AI，再把 AI 的输出搬回工具。**MCP 让 AI 直接操作这些工具**，你只需要说一句话：

> "帮我把 Figma 上这个页面的设计稿转成 SwiftUI 代码，然后创建一个 PR。"

AI 就能自动：读取 Figma → 生成代码 → 写入文件 → 创建 PR。全程无需你手动切换。

### 1.3 MCP 架构

MCP 采用**客户端-服务器**架构，由三个核心角色组成：

```
┌─────────────────────────────────────────────────┐
│                  Host（宿主）                     │
│            Claude Desktop / Cursor / Trae        │
│                                                  │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐   │
│  │ MCP Client│  │ MCP Client│  │ MCP Client│   │
│  │  (GitHub) │  │ (Figma)   │  │  (文件系统)│   │
│  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘   │
└────────┼──────────────┼──────────────┼──────────┘
         │              │              │
    MCP 协议        MCP 协议       MCP 协议
         │              │              │
  ┌──────▼──────┐ ┌─────▼──────┐ ┌────▼───────┐
  │ MCP Server  │ │ MCP Server │ │ MCP Server  │
  │  (GitHub)   │ │  (Figma)   │ │ (文件系统)   │
  └─────────────┘ └────────────┘ └─────────────┘
```

| 角色 | 说明 | 举例 |
|------|------|------|
| **Host** | 运行 AI 模型的应用程序 | Claude Desktop、Cursor、Trae |
| **Client** | Host 内部与 Server 通信的客户端，每个 Server 对应一个 Client | 内置在 Host 中，无需手动配置 |
| **Server** | 提供具体工具能力的服务进程 | GitHub MCP Server、Figma MCP Server |

> 💡 **简单理解**：Host 是你用的 AI 软件，Server 是你给 AI 装的"插件"，Client 是它们之间的"数据线"。你只需要关心装什么 Server，Client 是自动管理的。

MCP Server 通过两种方式向 AI 暴露能力：

| 能力类型 | 说明 | 举例 |
|---------|------|------|
| **Tools（工具）** | AI 可以调用的函数 | `create_issue`、`read_file`、`search_figma` |
| **Resources（资源）** | AI 可以读取的数据 | 项目文件内容、数据库表结构、API 文档 |

---

## 2. MCP 在 iOS 开发中的常用场景

### 2.1 Figma MCP：设计稿 → SwiftUI 代码

这是 iOS 开发者最想要的超能力——直接从设计稿生成代码。

| 传统方式 | MCP 方式 |
|---------|---------|
| 打开 Figma → 手动量间距 → 手写 SwiftUI | 告诉 AI "把这个 Figma 页面转成 SwiftUI" |
| 容易看错颜色值、字号 | AI 自动提取精确的色值、字号、间距 |
| 设计改了，手动改代码 | AI 重新读取 Figma，自动更新代码 |

使用示例（在 AI 对话中）：

```
请读取 Figma 文件 "iOS App - 首页" 中的登录页面设计稿，
将其转换为 SwiftUI View，使用我们项目的设计系统。
```

AI 会通过 Figma MCP Server 读取设计稿的节点信息（颜色、字体、布局），然后生成对应的 SwiftUI 代码。

### 2.2 GitHub MCP：PR 管理、Issue 跟踪、代码审查

| 功能 | 对话示例 |
|------|---------|
| 创建 Issue | "帮我在仓库创建一个 Bug：登录页面点击无响应" |
| 列出 PR | "看看今天有哪些待 Review 的 PR" |
| 审查代码 | "Review 一下 PR #42，重点关注内存管理" |
| 合并 PR | "PR #42 的 CI 通过了，帮我合并" |

### 2.3 Context7 MCP：实时查询第三方库文档

iOS 开发中经常需要查第三方库的用法。传统方式是打开浏览器搜索，现在可以直接问 AI：

```
查一下 Alamofire 5.x 怎么上传 multipart 表单数据？
```

AI 通过 Context7 MCP Server 实时获取最新的库文档，给出准确的代码示例，而不是凭训练数据"猜"一个可能过时的答案。

> ⚠️ AI 模型的训练数据有截止日期。如果你问"SwiftUI 最新的 NavigationStack 怎么用"，AI 可能给出旧的 NavigationView 方案。Context7 解决了这个问题——它实时查询最新文档。

### 2.4 Puppeteer MCP：浏览器自动化测试

虽然 iOS 开发主要用 Xcode，但很多项目也有配套的 Web 页面（营销页、管理后台等）。Puppeteer MCP 可以让 AI 自动操作浏览器：

```
打开我们的测试环境网站 http://localhost:3000，
点击"登录"按钮，输入测试账号，验证登录是否成功。
```

### 2.5 MySQL / 数据库 MCP：数据操作与验证

后端 API 返回数据不对？直接让 AI 查数据库：

```
查一下数据库中用户 ID 为 123 的最近 10 条订单记录，
看看状态是否都是"已完成"。
```

| 操作 | 说明 |
|------|------|
| 查询数据 | SELECT 查询，验证 API 返回 |
| 查看表结构 | 了解数据库 Schema |
| 统计分析 | 快速统计用户量、订单量等 |

> ⚠️ 数据库 MCP 建议只连接开发/测试环境的数据库，**绝对不要连接生产数据库**！

### 2.6 文件系统 MCP：项目文件操作

让 AI 直接读写你项目中的文件：

```
帮我在项目中找到所有使用了 deprecated API 的文件，
列出文件名和行号。
```

```
把 Localizable.strings 中的所有中文翻译整理成一个表格给我看。
```

---

## 3. 配置 MCP Server

MCP Server 的配置方式因 AI 应用而异，但核心都是写一个 JSON 配置文件，告诉 Host："请启动这个 MCP Server，参数是这些。"

### 3.1 Claude Desktop 配置 MCP

配置文件路径：

```
~/Library/Application Support/Claude/claude_desktop_config.json
```

配置示例：

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "/Users/yourname/Projects/MyApp"
      ]
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "ghp_xxxxxxxxxxxx"
      }
    }
  }
}
```

> 💡 修改配置后需要**重启 Claude Desktop** 才能生效。

### 3.2 Cursor 配置 MCP

Cursor 支持项目级和全局级两种 MCP 配置：

**项目级配置**（仅对当前项目生效）：

```
你的项目/.cursor/mcp.json
```

**全局配置**（对所有项目生效）：

```
~/.cursor/mcp.json
```

配置示例：

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "/Users/yourname/Projects/MyApp"
      ]
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "ghp_xxxxxxxxxxxx"
      }
    },
    "context7": {
      "command": "npx",
      "args": ["-y", "@upstash/context7-mcp@latest"]
    }
  }
}
```

> 💡 Cursor 中配置 MCP 后，在聊天窗口底部会显示 MCP 工具图标，点击可以查看已连接的 Server 和可用工具。

### 3.3 Trae 配置 MCP

Trae 同样支持 MCP 配置，配置文件位于项目目录或全局配置中：

**项目级配置**：

```
你的项目/.trae/mcp.json
```

**全局配置**：

```
~/.trae/mcp.json
```

配置格式与其他 AI 应用一致：

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "/Users/yourname/Projects/MyApp"
      ]
    }
  }
}
```

### 3.4 完整配置示例

以下是一个面向 iOS 开发者的完整 MCP 配置，你可以根据需要选用：

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "/Users/yourname/Projects/MyiOSApp"
      ]
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "ghp_your_token_here"
      }
    },
    "figma": {
      "command": "npx",
      "args": ["-y", "@anthropic/mcp-figma"],
      "env": {
        "FIGMA_ACCESS_TOKEN": "figd_your_token_here"
      }
    },
    "context7": {
      "command": "npx",
      "args": ["-y", "@upstash/context7-mcp@latest"]
    },
    "mysql": {
      "command": "npx",
      "args": ["-y", "@benborla29/mcp-server-mysql"],
      "env": {
        "MYSQL_HOST": "127.0.0.1",
        "MYSQL_PORT": "3306",
        "MYSQL_USER": "root",
        "MYSQL_PASSWORD": "your_password",
        "MYSQL_DATABASE": "myapp_dev"
      }
    }
  }
}
```

> ⚠️ **安全提醒**：配置文件中可能包含 API Token 等敏感信息。务必将配置文件加入 `.gitignore`，不要提交到 Git 仓库！

---

## 4. 常用 MCP Server 安装与使用

### 4.1 文件系统 MCP Server

**包名**：`@modelcontextprotocol/server-filesystem`

**功能**：让 AI 读写指定目录下的文件

**安装与配置**：

无需单独安装，`npx` 会自动下载并运行。在配置文件中添加：

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "/Users/yourname/Projects/MyiOSApp",
        "/Users/yourname/Projects/SharedResources"
      ]
    }
  }
}
```

> 💡 `args` 中可以指定多个目录，AI 可以访问这些目录及其子目录下的所有文件。

**使用示例**：

```
读取 MyApp/ContentView.swift 的内容，帮我找出所有硬编码的字符串，
提取到 Localizable.strings 中。
```

### 4.2 GitHub MCP Server

**包名**：`@modelcontextprotocol/server-github`

**功能**：操作 GitHub 仓库、Issue、PR 等

**前置准备**：创建 GitHub Personal Access Token

1. 打开 GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic)
2. 点击 "Generate new token"
3. 勾选所需权限：`repo`、`read:org`、`workflow`
4. 生成并复制 Token

**安装与配置**：

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "ghp_your_token_here"
      }
    }
  }
}
```

**使用示例**：

```
列出 myorg/my-ios-app 仓库中所有 open 状态的 Issue，
按创建时间排序。
```

```
在 myorg/my-ios-app 仓库创建一个 PR，
标题是"feat: 添加暗黑模式支持"，
从 feature/dark-mode 分支合并到 main 分支。
```

### 4.3 Figma MCP Server

**包名**：`@anthropic/mcp-figma`

**功能**：读取 Figma 设计稿的节点信息，辅助生成代码

**前置准备**：获取 Figma Access Token

1. 打开 Figma → Settings → Account → Personal access tokens
2. 点击 "Generate new token"
3. 复制 Token

**安装与配置**：

```json
{
  "mcpServers": {
    "figma": {
      "command": "npx",
      "args": ["-y", "@anthropic/mcp-figma"],
      "env": {
        "FIGMA_ACCESS_TOKEN": "figd_your_token_here"
      }
    }
  }
}
```

**使用示例**：

```
读取这个 Figma 文件的登录页面设计：
https://www.figma.com/design/xxxxx/MyApp?node-id=123-456

把设计稿转成 SwiftUI View，使用我们的设计 Token。
```

### 4.4 Context7 MCP Server

**包名**：`@upstash/context7-mcp`

**功能**：实时查询第三方库的最新文档

**安装与配置**：

```json
{
  "mcpServers": {
    "context7": {
      "command": "npx",
      "args": ["-y", "@upstash/context7-mcp@latest"]
    }
  }
}
```

**使用示例**：

```
查一下 Kingfisher 最新的缓存策略配置方式，
给我一个 SwiftUI 中使用 Kingfisher 加载网络图片的示例。
```

> 💡 Context7 不需要 API Key，开箱即用。它会自动从官方源获取最新文档。

### 4.5 MCP Server 速查表

| MCP Server | 包名 | 需要 Token | 主要用途 |
|------------|------|-----------|---------|
| 文件系统 | `@modelcontextprotocol/server-filesystem` | ❌ | 读写项目文件 |
| GitHub | `@modelcontextprotocol/server-github` | ✅ GitHub Token | Issue / PR 管理 |
| Figma | `@anthropic/mcp-figma` | ✅ Figma Token | 读取设计稿 |
| Context7 | `@upstash/context7-mcp` | ❌ | 查询库文档 |
| MySQL | `@benborla29/mcp-server-mysql` | ✅ 数据库密码 | 数据库操作 |
| Puppeteer | `@modelcontextprotocol/server-puppeteer` | ❌ | 浏览器自动化 |

---

## 5. 自定义 MCP Server

当现有的 MCP Server 不能满足需求时，你可以创建自己的 MCP Server。比如：

- **Xcode 项目分析 MCP**：自动分析 `.xcodeproj` 文件，检查 Target 配置
- **本地化检查 MCP**：扫描所有 `.strings` 文件，找出缺失的翻译
- **CocoaPods / SPM 管理 MCP**：查询和更新依赖版本

### 5.1 用 TypeScript 创建 MCP Server

#### 环境准备

```bash
# 安装 Node.js（如果还没有）
brew install node

# 确认版本
node --version   # 需要 v18+
npm --version
```

#### 创建项目

```bash
# 创建项目目录
mkdir mcp-ios-localization
cd mcp-ios-localization

# 初始化项目
npm init -y

# 安装 MCP SDK
npm install @modelcontextprotocol/sdk

# 安装 TypeScript 依赖
npm install -D typescript @types/node

# 初始化 TypeScript 配置
npx tsc --init
```

#### 编写 MCP Server

创建 `index.ts`：

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";
import * as fs from "fs";
import * as path from "path";

const server = new McpServer({
  name: "ios-localization-checker",
  version: "1.0.0",
});

server.tool(
  "check_missing_translations",
  "检查 iOS 项目中缺失的本地化翻译",
  {
    projectPath: z.string().describe("Xcode 项目根目录路径"),
    baseLanguage: z.string().default("en").describe("基准语言"),
  },
  async ({ projectPath, baseLanguage }) => {
    const baseStringsPath = path.join(
      projectPath,
      `${baseLanguage}.lproj`,
      "Localizable.strings"
    );

    if (!fs.existsSync(baseStringsPath)) {
      return {
        content: [
          {
            type: "text",
            text: `未找到基准语言文件: ${baseStringsPath}`,
          },
        ],
      };
    }

    const baseContent = fs.readFileSync(baseStringsPath, "utf-8");
    const baseKeys = parseStringsKeys(baseContent);

    const lprojDirs = fs
      .readdirSync(projectPath)
      .filter((d) => d.endsWith(".lproj") && d !== `${baseLanguage}.lproj`);

    const results: string[] = [];

    for (const dir of lprojDirs) {
      const lang = dir.replace(".lproj", "");
      const stringsPath = path.join(
        projectPath,
        dir,
        "Localizable.strings"
      );

      if (!fs.existsSync(stringsPath)) {
        results.push(`❌ ${lang}: Localizable.strings 文件不存在`);
        continue;
      }

      const langContent = fs.readFileSync(stringsPath, "utf-8");
      const langKeys = parseStringsKeys(langContent);

      const missingKeys = baseKeys.filter((k) => !langKeys.includes(k));

      if (missingKeys.length === 0) {
        results.push(`✅ ${lang}: 翻译完整`);
      } else {
        results.push(
          `⚠️ ${lang}: 缺失 ${missingKeys.length} 条翻译:\n` +
            missingKeys.map((k) => `  - ${k}`).join("\n")
        );
      }
    }

    return {
      content: [{ type: "text", text: results.join("\n\n") }],
    };
  }
);

function parseStringsKeys(content: string): string[] {
  const regex = /^"([^"]+)"\s*=/gm;
  const keys: string[] = [];
  let match;
  while ((match = regex.exec(content)) !== null) {
    keys.push(match[1]);
  }
  return keys;
}

async function main() {
  const transport = new StdioServerTransport();
  await server.connect(transport);
}

main().catch(console.error);
```

#### 配置 TypeScript 编译

修改 `tsconfig.json`：

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "Node16",
    "moduleResolution": "Node16",
    "outDir": "./dist",
    "strict": true,
    "esModuleInterop": true
  },
  "include": ["index.ts"]
}
```

修改 `package.json`，添加构建和启动脚本：

```json
{
  "name": "mcp-ios-localization",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "build": "tsc",
    "start": "node dist/index.js"
  }
}
```

#### 编译并配置

```bash
# 编译 TypeScript
npm run build
```

然后在 AI 应用的 MCP 配置中添加：

```json
{
  "mcpServers": {
    "ios-localization": {
      "command": "node",
      "args": ["/Users/yourname/mcp-ios-localization/dist/index.js"]
    }
  }
}
```

### 5.2 用 Python 创建 MCP Server

如果你更熟悉 Python，也可以用 Python 来写 MCP Server。

#### 环境准备

```bash
# 安装 Python MCP SDK
pip install mcp
```

#### 编写 MCP Server

创建 `xcode_analyzer.py`：

```python
import json
import subprocess
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp.types import Tool, TextContent

server = Server("xcode-project-analyzer")

@server.list_tools()
async def list_tools():
    return [
        Tool(
            name="analyze_xcode_project",
            description="分析 Xcode 项目配置，检查常见问题",
            inputSchema={
                "type": "object",
                "properties": {
                    "project_path": {
                        "type": "string",
                        "description": "Xcode 项目 (.xcodeproj) 路径",
                    }
                },
                "required": ["project_path"],
            },
        )
    ]

@server.call_tool()
async def call_tool(name, arguments):
    if name == "analyze_xcode_project":
        project_path = arguments["project_path"]
        result = analyze_project(project_path)
        return [TextContent(type="text", text=result)]

def analyze_project(project_path: str) -> str:
    findings = []

    try:
        output = subprocess.check_output(
            ["xcodebuild", "-project", project_path, "-list"],
            stderr=subprocess.STDOUT,
            text=True,
        )
        findings.append(f"📋 项目信息:\n{output}")
    except subprocess.CalledProcessError as e:
        findings.append(f"❌ 分析失败: {e.output}")

    try:
        output = subprocess.check_output(
            ["xcodebuild", "-project", project_path, "-showBuildSettings"],
            stderr=subprocess.STDOUT,
            text=True,
        )

        if "IPHONEOS_DEPLOYMENT_TARGET = 12.0" in output:
            findings.append(
                "⚠️ 部署目标版本较低 (12.0)，"
                "建议升级到 16.0+ 以使用最新 SwiftUI 特性"
            )

        if "SWIFT_VERSION = 5.0" in output:
            findings.append(
                "⚠️ Swift 版本为 5.0，建议升级到 5.9+ "
                "以使用宏 (Macro) 等新特性"
            )

    except subprocess.CalledProcessError:
        pass

    return "\n\n".join(findings) if findings else "✅ 未发现明显问题"

async def main():
    async with stdio_server() as (read_stream, write_stream):
        await server.run(read_stream, write_stream)

if __name__ == "__main__":
    import asyncio
    asyncio.run(main())
```

#### 配置使用

```json
{
  "mcpServers": {
    "xcode-analyzer": {
      "command": "python3",
      "args": ["/Users/yourname/mcp-xcode-analyzer/xcode_analyzer.py"]
    }
  }
}
```

### 5.3 MCP Server 开发基础结构

无论用什么语言，MCP Server 的核心结构都是一样的：

```
┌─────────────────────────────────┐
│         MCP Server              │
│                                 │
│  1. 创建 Server 实例            │
│     - 名称、版本号              │
│                                 │
│  2. 注册 Tools                  │
│     - 工具名称                  │
│     - 参数 Schema (Zod/JSON)    │
│     - 处理函数                  │
│                                 │
│  3. 注册 Resources（可选）      │
│     - 资源 URI                  │
│     - 读取函数                  │
│                                 │
│  4. 绑定传输层                  │
│     - Stdio（标准输入输出）      │
│     - SSE（HTTP 流）            │
│                                 │
└─────────────────────────────────┘
```

| 组件 | 说明 |
|------|------|
| **Server 实例** | MCP Server 的入口，定义名称和版本 |
| **Tool** | AI 可以调用的函数，需定义参数 Schema |
| **Resource** | AI 可以读取的数据源，类似 REST API 的 GET |
| **Transport** | 通信方式，本地用 Stdio，远程用 SSE |

> 💡 **开发建议**：先从简单的 Tool 开始，比如只实现一个"检查项目配置"的功能。验证能跑通后，再逐步添加更多 Tool。

---

## 6. MCP 最佳实践

### 6.1 安全注意事项

| 风险 | 说明 | 建议 |
|------|------|------|
| Token 泄露 | 配置文件中的 API Token 被提交到 Git | 将配置文件加入 `.gitignore` |
| 数据库暴露 | MCP 连接了生产数据库 | 只连接开发/测试环境 |
| 文件越权 | 文件系统 MCP 暴露了整个用户目录 | 只指定项目目录，不要用 `/` 或 `~` |
| 命令注入 | 自定义 MCP Server 未校验输入 | 对用户输入做严格校验和转义 |

**.gitignore 中添加**：

```
# MCP 配置文件（包含敏感 Token）
.claude/
.cursor/mcp.json
.trae/mcp.json
claude_desktop_config.json
```

**文件系统 MCP 的安全配置示例**：

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "/Users/yourname/Projects/MyiOSApp"
      ]
    }
  }
}
```

> ⚠️ **绝对不要**把文件系统 MCP 的路径设为 `/` 或 `~`，这会让 AI 能读写你电脑上的所有文件！

### 6.2 性能优化

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| 启动慢 | 每个 MCP Server 都要启动一个进程 | 只配置你当前需要的 Server |
| 内存占用高 | 同时运行太多 MCP Server | 按项目配置，不用全局配所有 |
| 响应慢 | MCP Server 首次调用需要初始化 | 使用 `npx` 的 `-y` 参数避免交互提示 |

**按需配置建议**：

- 日常编码：文件系统 + Context7
- 团队协作：文件系统 + GitHub
- UI 开发：文件系统 + Figma
- 全栈开发：文件系统 + GitHub + MySQL + Context7

### 6.3 常见问题排查

#### 问题 1：MCP Server 无法启动

**症状**：AI 应用中看不到 MCP 工具

**排查步骤**：

```bash
# 1. 检查 Node.js 是否安装
node --version

# 2. 手动运行 MCP Server 看报错
npx -y @modelcontextprotocol/server-filesystem /tmp/test

# 3. 检查配置文件 JSON 格式是否正确
cat ~/.cursor/mcp.json | python3 -m json.tool
```

#### 问题 2：MCP Server 启动但工具不可用

**症状**：配置了 MCP 但 AI 说"我没有这个工具"

**可能原因**：

| 原因 | 解决方案 |
|------|---------|
| 配置文件路径错误 | 确认路径是否正确，区分项目级和全局级 |
| Token 无效 | 重新生成 GitHub / Figma Token |
| Server 崩溃 | 查看 AI 应用的日志（Cursor: Output 面板） |
| 未重启应用 | 修改配置后重启 AI 应用 |

#### 问题 3：npx 下载慢

**解决方案**：

```bash
# 设置 npm 镜像源
npm config set registry https://registry.npmmirror.com

# 或者使用已全局安装的包
npm install -g @modelcontextprotocol/server-filesystem
```

全局安装后，配置中可以直接用命令名：

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "mcp-server-filesystem",
      "args": ["/Users/yourname/Projects/MyiOSApp"]
    }
  }
}
```

#### 问题 4：自定义 MCP Server 调试

在开发自定义 MCP Server 时，可以开启调试日志：

```json
{
  "mcpServers": {
    "my-server": {
      "command": "node",
      "args": ["/path/to/my-server/dist/index.js"],
      "env": {
        "DEBUG": "mcp:*",
        "LOG_LEVEL": "debug"
      }
    }
  }
}
```

---

## 小结

| 内容 | 关键要点 |
|------|---------|
| **MCP 是什么** | AI 世界的 USB-C——统一协议，让 AI 连接任何工具 |
| **架构** | Host（AI 应用）→ Client → Server（工具服务） |
| **常用场景** | Figma 设计稿转代码、GitHub PR 管理、文档查询、数据库操作、文件读写 |
| **配置方式** | 在 AI 应用的 JSON 配置文件中声明 MCP Server |
| **常用 Server** | 文件系统、GitHub、Figma、Context7——开箱即用 |
| **自定义开发** | 用 TypeScript 或 Python，注册 Tool + 绑定 Stdio 传输 |
| **安全** | 不暴露 Token、不连生产库、限制文件系统范围 |
| **性能** | 按需配置，不要一次装太多 |

MCP 正在快速改变开发者与 AI 协作的方式。对于 iOS 开发者来说，掌握 MCP 意味着你可以让 AI 真正融入你的工作流——从设计稿到代码、从 Issue 到 PR、从文档到实现，AI 不再只是"聊天助手"，而是你的"开发搭档"。

> 💡 **下一步**：从最简单的文件系统 MCP 开始配置，体验 AI 直接操作项目文件的能力。然后逐步添加 GitHub、Figma 等 Server，让 AI 的能力越来越强。
