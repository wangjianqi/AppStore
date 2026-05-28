# RAG 与知识库问答

> 🎯 **本章目标**：
> - 理解 RAG（检索增强生成）的核心原理与适用场景
> - 掌握 RAG 完整 Pipeline 的每个环节及其关键参数
> - 学会文档处理与分块策略的选择与实现
> - 理解文本嵌入（Embedding）的原理与模型选型
> - 了解向量数据库的选型与 iOS 端轻量方案
> - 掌握检索、重排与查询优化的实战技巧
> - 使用 Swift 实现完整的 RAG Service
> - 实现无需服务器的本地知识库方案
> - 掌握 RAG 效果优化的核心策略
> - 了解生产环境的注意事项与最佳实践

---

## 1. 什么是 RAG

### 1.1 RAG 的定义与原理

RAG（Retrieval-Augmented Generation，检索增强生成）是一种将**信息检索**与**大语言模型生成**相结合的技术架构。其核心思想是：在 LLM 生成回答之前，先从外部知识库中检索与问题相关的文档片段，将这些片段作为上下文注入 Prompt，让 LLM 基于真实数据生成答案。

```
传统 LLM 回答流程：
  用户问题 → LLM → 回答（可能产生幻觉）

RAG 回答流程：
  用户问题 → 检索知识库 → 获取相关文档 → 拼接 Prompt → LLM → 基于事实的回答
```

RAG 的三个核心阶段：

| 阶段 | 作用 | 关键技术 |
|------|------|---------|
| **检索（Retrieval）** | 从知识库中找到与问题最相关的文档片段 | 向量相似度搜索、关键词检索 |
| **增强（Augmentation）** | 将检索结果注入 LLM 的 Prompt 中 | Prompt 模板设计、上下文窗口管理 |
| **生成（Generation）** | LLM 基于增强后的上下文生成回答 | 大语言模型推理、引用溯源 |

### 1.2 为什么需要 RAG

LLM 虽然强大，但存在三个根本性局限，RAG 正是为了解决这些问题而生：

**1. 知识截止问题**

LLM 的训练数据有截止日期，无法回答训练之后发生的事情。例如，一个 2024 年 1 月训练的模型，无法回答 2024 年 6 月的产品更新内容。

**2. 幻觉问题**

LLM 在缺乏确切知识时，会"编造"看似合理但实际错误的答案。这在法律咨询、医疗问答等场景中尤其危险。

**3. 私有数据问题**

LLM 无法访问企业的内部文档、产品手册、用户数据等私有信息。这些数据不会出现在训练集中，LLM 对其一无所知。

```
没有 RAG 时的典型问题：

用户：我们公司的退货政策是什么？
LLM：一般来说，大多数公司的退货政策是...（泛泛而谈，不准确）

有了 RAG：

用户：我们公司的退货政策是什么？
检索：找到公司退货政策文档第 3 条
LLM：根据公司退货政策第 3 条，商品在购买后 7 天内可无理由退货...（准确引用）
```

### 1.3 RAG vs 微调 vs 预训练

三种让 LLM 获取新知识的方式各有优劣：

| 维度 | RAG | 微调（Fine-tuning） | 预训练（Pre-training） |
|------|-----|--------------------|-----------------------|
| **知识更新** | 实时更新，修改知识库即可 | 需重新训练，成本高 | 需重新训练，成本极高 |
| **知识溯源** | ✅ 可追溯到原文档 | ❌ 知识融入模型权重 | ❌ 知识融入模型权重 |
| **幻觉控制** | ✅ 基于检索结果生成 | ⚠️ 仍可能产生幻觉 | ⚠️ 仍可能产生幻觉 |
| **私有数据** | ✅ 数据保留在本地 | ⚠️ 数据需上传训练 | ⚠️ 数据需上传训练 |
| **实现成本** | 低（检索系统 + Prompt） | 中（训练数据 + GPU） | 极高（海量数据 + 大量 GPU） |
| **响应延迟** | 略高（需检索步骤） | 低（直接推理） | 低（直接推理） |
| **适用场景** | 事实性问答、知识库 | 风格调整、领域适配 | 基础能力提升 |
| **数据量要求** | 数十~数万文档 | 数百~数千条样本 | 数十亿 Token |

> 💡 **提示**：RAG 和微调并非互斥，两者可以组合使用。先用微调让模型适配特定领域的表达风格，再用 RAG 注入最新知识，效果最佳。

### 1.4 RAG 的典型应用场景

| 应用场景 | 描述 | 示例 |
|---------|------|------|
| **企业知识库** | 员工可查询公司制度、流程、FAQ | "年假怎么请？" → 返回 HR 制度文档 |
| **智能客服** | 基于产品文档自动回答用户问题 | "路由器怎么重置？" → 返回操作指南 |
| **文档问答** | 对长文档进行精准问答 | "合同第 5 条写了什么？" → 返回合同条款 |
| **法律咨询** | 基于法条和案例提供法律参考 | "劳动合同解除有哪些规定？" → 返回相关法条 |
| **医疗辅助** | 基于医学文献辅助诊断参考 | "这个症状可能是什么病？" → 返回相关文献 |
| **教育辅导** | 基于教材内容解答学生问题 | "牛顿第三定律是什么？" → 返回教材解释 |
| **代码助手** | 基于项目代码库回答开发问题 | "这个 API 怎么用？" → 返回代码示例 |
| **金融分析** | 基于研报和公告进行分析 | "这家公司营收情况？" → 返回财报数据 |

---

## 2. RAG 架构详解

### 2.1 完整 RAG Pipeline 流程图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        RAG 完整 Pipeline                                │
│                                                                         │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐          │
│  │  原始文档  │───→│  文档解析  │───→│  文本分块  │───→│  元数据标注 │          │
│  │ PDF/Word  │    │  提取文本  │    │ Chunking  │    │ 来源/页码  │          │
│  └──────────┘    └──────────┘    └──────────┘    └──────────┘          │
│                                                       │                 │
│                                                       ▼                 │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐          │
│  │  向量数据库 │←───│  向量存储  │←───│  向量化   │←───│  文本块    │          │
│  │  Index    │    │  持久化   │    │ Embedding │    │ + 元数据   │          │
│  └──────────┘    └──────────┘    └──────────┘    └──────────┘          │
│       │                                                                 │
│       │                     ┌──────────┐                                │
│       │                     │  用户问题  │                                │
│       │                     └────┬─────┘                                │
│       │                          │                                      │
│       │                          ▼                                      │
│       │                     ┌──────────┐                                │
│       │                     │ 问题向量化 │                                │
│       │                     │ Embedding │                                │
│       │                     └────┬─────┘                                │
│       │                          │                                      │
│       ▼                          ▼                                      │
│  ┌────────────────────────────────────┐                                 │
│  │          向量相似度检索              │                                 │
│  │   Top-K 最相似的文档块 + 关键词检索   │                                 │
│  └──────────────┬─────────────────────┘                                 │
│                 │                                                        │
│                 ▼                                                        │
│  ┌────────────────────────────────────┐                                 │
│  │            重排（Reranking）         │                                 │
│  │   对检索结果按相关性重新排序          │                                 │
│  └──────────────┬─────────────────────┘                                 │
│                 │                                                        │
│                 ▼                                                        │
│  ┌────────────────────────────────────┐                                 │
│  │          Prompt 构建                │                                 │
│  │  System + 检索上下文 + 用户问题      │                                 │
│  └──────────────┬─────────────────────┘                                 │
│                 │                                                        │
│                 ▼                                                        │
│  ┌────────────────────────────────────┐                                 │
│  │          LLM 生成回答               │                                 │
│  │   基于上下文生成 + 引用溯源          │                                 │
│  └────────────────────────────────────┘                                 │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 每个环节的作用与关键参数

| Pipeline 环节 | 作用 | 关键参数 | 常见问题 |
|-------------|------|---------|---------|
| **文档解析** | 从各种格式提取纯文本 | 格式支持、编码检测 | PDF 表格丢失、OCR 错误 |
| **文本分块** | 将长文档切分为适合检索的小块 | chunk_size、overlap、策略 | 块太大检索不精确，太小丢失上下文 |
| **元数据标注** | 为每个块附加来源信息 | 来源、页码、章节、时间 | 元数据不全会影响溯源 |
| **向量化** | 将文本转换为高维向量 | 模型选择、维度、批大小 | 模型选择影响检索质量 |
| **向量存储** | 将向量持久化并建立索引 | 索引类型、存储引擎 | 索引构建耗时、存储空间 |
| **向量检索** | 根据问题向量找最相似的文档块 | Top-K、相似度阈值 | 召回率不足或噪声过多 |
| **重排** | 对检索结果精确排序 | 重排模型、分数阈值 | 重排模型延迟 |
| **Prompt 构建** | 将检索结果组装为 LLM 输入 | 模板设计、Token 预算 | 上下文过长超出窗口 |
| **LLM 生成** | 基于上下文生成最终答案 | 模型选择、温度、最大 Token | 幻觉、引用不准确 |

### 2.3 端到端数据流示意

以一个具体例子展示完整数据流：

```
输入：用户上传公司员工手册（50 页 PDF）

1. 文档解析 → 提取 50 页文本内容
2. 文本分块 → 切分为 200 个 Chunk（每个约 500 字，重叠 50 字）
3. 元数据标注 → 每个 Chunk 标注：来源="员工手册", 页码=3, 章节="请假制度"
4. 向量化 → 200 个 Chunk → 200 个 1536 维向量
5. 向量存储 → 存入向量数据库，建立索引

--- 用户提问 ---

6. 问题向量化 → "年假可以累计吗？" → 1536 维向量
7. 向量检索 → 找到 Top-5 最相似 Chunk
8. 重排 → 按精确相关性重新排序，取 Top-3
9. Prompt 构建：
   System: 你是公司 HR 助手，请基于以下文档回答问题...
   Context: [Chunk 1: 年假制度...] [Chunk 2: 假期累计...] [Chunk 3: 年假清零...]
   Question: 年假可以累计吗？
10. LLM 生成 → "根据员工手册第 12 页'假期制度'章节，年假不可累计..."
```

---

## 3. 文档处理与分块

### 3.1 支持的文档格式

| 格式 | 解析方式 | 优势 | 挑战 |
|------|---------|------|------|
| **PDF** | PDFKit / PyMuPDF | 最常见的文档格式 | 表格解析难、扫描件需 OCR |
| **Word (.docx)** | python-docx / 原生解析 | 保留格式信息 | 嵌入对象处理复杂 |
| **Markdown** | 直接读取 | 结构清晰，天然分块 | 需处理链接和图片 |
| **HTML** | HTML 解析器 | 网页内容抓取 | 需去除标签和广告 |
| **TXT** | 直接读取 | 最简单 | 无结构信息 |
| **Excel** | 读取行列 | 表格数据 | 需转为文本描述 |
| **PPT** | 提取幻灯片文本 | 演示内容 | 文本碎片化 |

### 3.2 分块策略对比

分块（Chunking）是 RAG 中最关键的预处理步骤，直接影响检索质量：

| 策略 | 原理 | 优点 | 缺点 | 适用场景 |
|------|------|------|------|---------|
| **固定长度** | 按 Token/字符数切割 | 实现简单，块大小均匀 | 可能切断句子/段落 | 通用场景 |
| **语义分块** | 按语义边界（段落/章节）切割 | 保留语义完整性 | 块大小不均匀 | 结构化文档 |
| **递归分块** | 按分隔符层级递归切割 | 兼顾大小和语义 | 需要调参 | 最常用的通用策略 |
| **句子级分块** | 每个句子一个块 | 粒度最细 | 上下文不足 | 精确匹配场景 |

> 💡 **提示**：LangChain 的 `RecursiveCharacterTextSplitter` 是最常用的分块工具，它按 `["\n\n", "\n", "。", "！", "？", "；", " "]` 的优先级递归切割，优先保留段落完整性。

### 3.3 分块参数选择

```
chunk_size（块大小）与 overlap（重叠）的关系：

文档文本：AAAAABBBBBCCCCCDDDDDEEEEE

chunk_size = 10, overlap = 2：
  Chunk 1: AAAAABBBBB
  Chunk 2: BBCCCCCDD  ← 与 Chunk 1 重叠 2 字符
  Chunk 3: CCDDDDDEEE
  Chunk 4: EEEEE

chunk_size 越大：
  ✅ 每个块包含更多上下文
  ❌ 检索精度下降（噪声增加）
  ❌ Token 消耗增加

chunk_size 越小：
  ✅ 检索精度更高
  ❌ 上下文可能不足
  ❌ 块数量增加，检索变慢
```

不同场景的推荐参数：

| 场景 | chunk_size | overlap | 说明 |
|------|-----------|---------|------|
| FAQ 问答 | 200-300 | 0-50 | 短问答，精确匹配 |
| 技术文档 | 500-800 | 50-100 | 需要一定上下文 |
| 法律合同 | 800-1200 | 100-200 | 条款需要完整上下文 |
| 长篇报告 | 1000-1500 | 100-200 | 保留章节结构 |

### 3.4 元数据管理

每个 Chunk 都应附带元数据，用于溯源和过滤：

```swift
struct DocumentChunk: Identifiable, Codable {
    let id: UUID
    let content: String
    let metadata: ChunkMetadata

    struct ChunkMetadata: Codable {
        let source: String
        let fileName: String
        let pageIndex: Int?
        let section: String?
        let chunkIndex: Int
        let createdAt: Date
        let charCount: Int
        let tags: [String]
    }
}
```

元数据的用途：

| 元数据字段 | 用途 | 示例 |
|-----------|------|------|
| `source` | 标识文档来源 | "员工手册.pdf" |
| `pageIndex` | 定位原文位置 | 第 12 页 |
| `section` | 按章节过滤 | "请假制度" |
| `tags` | 分类检索 | ["HR", "假期"] |
| `createdAt` | 时效性过滤 | 2024-06-01 |

### 3.5 Swift 端文档处理实现

```swift
import PDFKit

struct DocumentProcessor {

    static func extractText(from url: URL) async throws -> [DocumentChunk] {
        let ext = url.pathExtension.lowercased()
        switch ext {
        case "pdf":
            return try processPDF(url: url)
        case "txt", "md":
            return try processPlainText(url: url)
        default:
            throw DocumentError.unsupportedFormat(ext)
        }
    }

    private static func processPDF(url: URL) throws -> [DocumentChunk] {
        guard let document = PDFDocument(url: url) else {
            throw DocumentError.pdfLoadFailed
        }

        var chunks: [DocumentChunk] = []
        let totalPageCount = document.pageCount

        for pageIndex in 0..<totalPageCount {
            guard let page = document.page(at: pageIndex) else { continue }
            let pageText = page.string ?? ""

            let pageChunks = splitText(
                pageText,
                metadata: DocumentChunk.ChunkMetadata(
                    source: url.lastPathComponent,
                    fileName: url.lastPathComponent,
                    pageIndex: pageIndex,
                    section: nil,
                    chunkIndex: 0,
                    createdAt: Date(),
                    charCount: pageText.count,
                    tags: []
                )
            )

            chunks.append(contentsOf: pageChunks)
        }

        return chunks
    }

    private static func processPlainText(url: URL) throws -> [DocumentChunk] {
        let text = try String(contentsOf: url, encoding: .utf8)

        return splitText(
            text,
            metadata: DocumentChunk.ChunkMetadata(
                source: url.lastPathComponent,
                fileName: url.lastPathComponent,
                pageIndex: nil,
                section: nil,
                chunkIndex: 0,
                createdAt: Date(),
                charCount: text.count,
                tags: []
            )
        )
    }

    static func splitText(
        _ text: String,
        chunkSize: Int = 500,
        overlap: Int = 50,
        metadata: DocumentChunk.ChunkMetadata
    ) -> [DocumentChunk] {
        let separators = ["\n\n", "\n", "。", "！", "？", "；", " "]
        return recursiveSplit(
            text: text,
            separators: separators,
            chunkSize: chunkSize,
            overlap: overlap,
            baseMetadata: metadata
        )
    }

    private static func recursiveSplit(
        text: String,
        separators: [String],
        chunkSize: Int,
        overlap: Int,
        baseMetadata: DocumentChunk.ChunkMetadata,
        chunkIndexOffset: Int = 0
    ) -> [DocumentChunk] {
        if text.count <= chunkSize {
            return [DocumentChunk(
                id: UUID(),
                content: text,
                metadata: DocumentChunk.ChunkMetadata(
                    source: baseMetadata.source,
                    fileName: baseMetadata.fileName,
                    pageIndex: baseMetadata.pageIndex,
                    section: baseMetadata.section,
                    chunkIndex: chunkIndexOffset,
                    createdAt: baseMetadata.createdAt,
                    charCount: text.count,
                    tags: baseMetadata.tags
                )
            )]
        }

        guard let separator = separators.first else {
            return splitByFixedSize(
                text: text,
                chunkSize: chunkSize,
                overlap: overlap,
                baseMetadata: baseMetadata
            )
        }

        let parts = text.components(separatedBy: separator)
        var chunks: [DocumentChunk] = []
        var currentChunk = ""
        var currentIndex = chunkIndexOffset

        for part in parts {
            if currentChunk.count + part.count + separator.count > chunkSize && !currentChunk.isEmpty {
                let metadata = DocumentChunk.ChunkMetadata(
                    source: baseMetadata.source,
                    fileName: baseMetadata.fileName,
                    pageIndex: baseMetadata.pageIndex,
                    section: baseMetadata.section,
                    chunkIndex: currentIndex,
                    createdAt: baseMetadata.createdAt,
                    charCount: currentChunk.count,
                    tags: baseMetadata.tags
                )
                chunks.append(DocumentChunk(id: UUID(), content: currentChunk, metadata: metadata))
                currentIndex += 1

                if overlap > 0 && currentChunk.count > overlap {
                    currentChunk = String(currentChunk.suffix(overlap)) + separator + part
                } else {
                    currentChunk = part
                }
            } else {
                currentChunk += currentChunk.isEmpty ? part : separator + part
            }
        }

        if !currentChunk.isEmpty {
            let metadata = DocumentChunk.ChunkMetadata(
                source: baseMetadata.source,
                fileName: baseMetadata.fileName,
                pageIndex: baseMetadata.pageIndex,
                section: baseMetadata.section,
                chunkIndex: currentIndex,
                createdAt: baseMetadata.createdAt,
                charCount: currentChunk.count,
                tags: baseMetadata.tags
            )
            chunks.append(DocumentChunk(id: UUID(), content: currentChunk, metadata: metadata))
        }

        return chunks
    }

    private static func splitByFixedSize(
        text: String,
        chunkSize: Int,
        overlap: Int,
        baseMetadata: DocumentChunk.ChunkMetadata
    ) -> [DocumentChunk] {
        var chunks: [DocumentChunk] = []
        let chars = Array(text)
        var start = 0
        var index = 0

        while start < chars.count {
            let end = min(start + chunkSize, chars.count)
            let chunkText = String(chars[start..<end])

            let metadata = DocumentChunk.ChunkMetadata(
                source: baseMetadata.source,
                fileName: baseMetadata.fileName,
                pageIndex: baseMetadata.pageIndex,
                section: baseMetadata.section,
                chunkIndex: index,
                createdAt: baseMetadata.createdAt,
                charCount: chunkText.count,
                tags: baseMetadata.tags
            )
            chunks.append(DocumentChunk(id: UUID(), content: chunkText, metadata: metadata))

            start += chunkSize - overlap
            index += 1
        }

        return chunks
    }
}

enum DocumentError: LocalizedError {
    case unsupportedFormat(String)
    case pdfLoadFailed

    var errorDescription: String? {
        switch self {
        case .unsupportedFormat(let ext): return "不支持的文件格式: .\(ext)"
        case .pdfLoadFailed: return "PDF 文件加载失败"
        }
    }
}
```

---

## 4. 向量化（Embedding）

### 4.1 什么是文本嵌入

文本嵌入（Text Embedding）是将文本转换为高维数值向量的过程。语义相近的文本在向量空间中距离更近，语义不同的文本距离更远。

```
文本嵌入示意：

"年假怎么请？"     → [0.12, -0.34, 0.56, ..., 0.78]  (1536 维)
"如何申请年假？"   → [0.11, -0.33, 0.55, ..., 0.77]  (语义相似，向量接近)
"路由器怎么重启？" → [0.87, 0.21, -0.45, ..., 0.03]  (语义不同，向量远离)

余弦相似度：
  "年假怎么请？" vs "如何申请年假？"   → 0.95 (高度相似)
  "年假怎么请？" vs "路由器怎么重启？" → 0.12 (几乎无关)
```

### 4.2 嵌入模型对比

| 模型 | 提供方 | 维度 | 最大输入 | 中英文能力 | 价格 | 适用场景 |
|------|--------|------|---------|-----------|------|---------|
| **text-embedding-3-small** | OpenAI | 1536 | 8191 Token | 中等 | $0.02/1M Token | 通用场景 |
| **text-embedding-3-large** | OpenAI | 3072 | 8191 Token | 较好 | $0.13/1M Token | 高质量需求 |
| **text-embedding-v3** | 通义千问 | 1024 | 8192 Token | 优秀（中文优化） | ¥0.7/1M Token | 中文为主场景 |
| **embedding-3** | 智谱 AI | 2048 | 512 Token | 优秀（中文优化） | ¥0.5/1M Token | 中文短文本 |
| **bge-large-zh-v1.5** | 本地部署 | 1024 | 512 Token | 优秀（中文优化） | 免费（需 GPU） | 私有化部署 |
| **all-MiniLM-L6-v2** | 本地 / Core ML | 384 | 256 Token | 一般 | 免费 | iOS 端本地嵌入 |

> 💡 **提示**：如果应用主要面向中文用户，推荐使用通义千问或智谱的中文嵌入模型，中文语义理解明显优于 OpenAI 的模型。如果需要完全离线运行，可以考虑将 MiniLM 模型转换为 Core ML 格式在设备端运行。

### 4.3 调用嵌入 API 的 Swift 实现

```swift
struct EmbeddingService {
    let apiKey: String
    let baseURL: String
    let model: String

    static let openAI = EmbeddingService(
        apiKey: "",
        baseURL: "https://api.openai.com/v1",
        model: "text-embedding-3-small"
    )

    static let dashscope = EmbeddingService(
        apiKey: "",
        baseURL: "https://dashscope.aliyuncs.com/api/v1",
        model: "text-embedding-v3"
    )

    static let zhipu = EmbeddingService(
        apiKey: "",
        baseURL: "https://open.bigmodel.cn/api/paas/v4",
        model: "embedding-3"
    )

    func embed(texts: [String]) async throws -> [[Float]] {
        var request = URLRequest(url: URL(string: "\(baseURL)/embeddings")!)
        request.httpMethod = "POST"
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")
        request.setValue("Bearer \(apiKey)", forHTTPHeaderField: "Authorization")

        let body: [String: Any] = [
            "model": model,
            "input": texts
        ]
        request.httpBody = try JSONSerialization.data(withJSONObject: body)

        let (data, response) = try await URLSession.shared.data(for: request)

        guard let httpResponse = response as? HTTPURLResponse,
              (200...299).contains(httpResponse.statusCode) else {
            throw EmbeddingError.apiError(String(data: data, encoding: .utf8) ?? "Unknown error")
        }

        let result = try JSONDecoder().decode(EmbeddingResponse.self, from: data)
        return result.data.map { $0.embedding }
    }

    func embed(text: String) async throws -> [Float] {
        let embeddings = try await embed(texts: [text])
        guard let first = embeddings.first else {
            throw EmbeddingError.emptyResult
        }
        return first
    }
}

struct EmbeddingResponse: Codable {
    let data: [EmbeddingData]
    let model: String
    let usage: Usage

    struct EmbeddingData: Codable {
        let embedding: [Float]
        let index: Int
    }

    struct Usage: Codable {
        let promptTokens: Int
        let totalTokens: Int

        enum CodingKeys: String, CodingKey {
            case promptTokens = "prompt_tokens"
            case totalTokens = "total_tokens"
        }
    }
}

enum EmbeddingError: LocalizedError {
    case apiError(String)
    case emptyResult

    var errorDescription: String? {
        switch self {
        case .apiError(let msg): return "嵌入 API 错误: \(msg)"
        case .emptyResult: return "嵌入结果为空"
        }
    }
}
```

### 4.4 本地嵌入方案（Core ML 模型）

对于需要完全离线运行或对隐私要求极高的场景，可以使用 Core ML 在设备端运行嵌入模型：

```swift
import CoreML

struct LocalEmbeddingService {
    private let model: MLModel?

    init() {
        guard let modelURL = Bundle.main.url(forResource: "MiniLM_L6_v2", withExtension: "mlmodelc"),
              let model = try? MLModel(contentsOf: modelURL) else {
            self.model = nil
            return
        }
        self.model = model
    }

    var isAvailable: Bool { model != nil }

    func embed(text: String) async throws -> [Float] {
        guard let model = model else {
            throw LocalEmbeddingError.modelNotAvailable
        }

        let inputFeatures = try MLFeatureProvider(dictionary: [
            "text": MLFeatureValue(string: text)
        ])

        let prediction = try model.prediction(from: inputFeatures)

        guard let output = prediction.featureValue(for: "embedding")?.multiArrayValue else {
            throw LocalEmbeddingError.predictionFailed
        }

        var embedding: [Float] = []
        embedding.reserveCapacity(output.count)
        for i in 0..<output.count {
            embedding.append(output[i].floatValue)
        }

        return embedding
    }

    func embed(texts: [String]) async throws -> [[Float]] {
        try await withThrowingTaskGroup(of: [Float].self) { group in
            for text in texts {
                group.addTask {
                    try await embed(text: text)
                }
            }
            var results: [[Float]] = []
            for try await result in group {
                results.append(result)
            }
            return results
        }
    }
}

enum LocalEmbeddingError: LocalizedError {
    case modelNotAvailable
    case predictionFailed

    var errorDescription: String? {
        switch self {
        case .modelNotAvailable: return "本地嵌入模型不可用"
        case .predictionFailed: return "本地嵌入推理失败"
        }
    }
}
```

⚠️ **警告**：将嵌入模型转换为 Core ML 格式需要使用 `coremltools`，且模型文件通常有 20-100MB，会增加 App 包体积。建议仅在确实需要离线场景时使用本地嵌入方案。

### 4.5 嵌入维度与质量的关系

| 维度 | 存储开销 | 检索质量 | 检索速度 | 适用场景 |
|------|---------|---------|---------|---------|
| 384 | 低 | 一般 | 快 | 简单分类、粗排 |
| 768 | 中 | 较好 | 中 | 通用检索 |
| 1024 | 中 | 好 | 中 | 中文优化场景 |
| 1536 | 较高 | 很好 | 较慢 | 高质量检索 |
| 3072 | 高 | 最佳 | 慢 | 最高质量需求 |

> 💡 **提示**：OpenAI 的 `text-embedding-3-small/large` 支持维度缩减（Matryoshka 嵌入），可以在不显著损失质量的情况下降低维度。例如 3072 维可以缩减到 512 维，质量损失约 5%，但存储和检索效率大幅提升。

---

## 5. 向量数据库

### 5.1 向量数据库选型对比

| 数据库 | 类型 | 维度支持 | 索引算法 | 规模 | 特色 | 适用场景 |
|--------|------|---------|---------|------|------|---------|
| **Pinecone** | 云托管 | 任意 | 自研 | 百万~十亿 | 全托管，零运维 | 快速上线、不想运维 |
| **Qdrant** | 开源 | 任意 | HNSW | 百万~十亿 | Rust 实现，高性能 | 自建部署、Rust 生态 |
| **Weaviate** | 开源 | 任意 | HNSW | 百万~十亿 | 内置多模态 | 多模态检索 |
| **Milvus** | 开源 | 任意 | IVF/HNSW | 十亿+ | 超大规模 | 超大规模知识库 |
| **Chroma** | 开源 | 任意 | HNSW | 十万~百万 | 轻量，Python 友好 | 原型开发、小规模 |
| **pgvector** | PG 扩展 | 任意 | IVFFlat/HNSW | 百万 | 复用 PostgreSQL | 已有 PG 基础设施 |

### 5.2 云端 vs 本地部署对比

| 维度 | 云端（Pinecone 等） | 本地部署（Qdrant 等） |
|------|-------------------|---------------------|
| **运维成本** | 零 | 需要专人维护 |
| **初始成本** | 按用量付费 | 服务器 + 人力 |
| **数据安全** | 数据在第三方 | 数据完全自控 |
| **扩展性** | 自动扩缩容 | 手动扩容 |
| **延迟** | 取决于网络 | 局域网低延迟 |
| **合规性** | 需确认数据出境 | 满足数据本地化要求 |

> 💡 **提示**：对于面向中国大陆用户的应用，数据出境合规是重要考量。如果知识库包含用户隐私数据，建议使用国内云服务商的向量数据库（如阿里云 Milvus、腾讯云 VectorDB），或自建部署。

### 5.3 iOS 端轻量方案

在 iOS 端，我们无法运行完整的向量数据库，但可以实现轻量级的向量存储和检索：

**方案一：SQLite + 向量扩展**

```swift
import SQLite

struct VectorStore {
    private let db: Connection
    private let table = Table("vectors")

    private let idCol = Expression<String>("id")
    private let contentCol = Expression<String>("content")
    private let vectorCol = Expression<Data>("vector")
    private let sourceCol = Expression<String>("source")
    private let pageIndexCol = Expression<Int?>("page_index")
    private let sectionCol = Expression<String?>("section")
    private let chunkIndexCol = Expression<Int>("chunk_index")
    private let createdAtCol = Expression<Date>("created_at")

    init(dbPath: String) throws {
        self.db = try Connection(dbPath)
        try createTable()
    }

    private func createTable() throws {
        try db.run(table.create(ifNotExists: true) { t in
            t.column(idCol, primaryKey: true)
            t.column(contentCol)
            t.column(vectorCol)
            t.column(sourceCol)
            t.column(pageIndexCol)
            t.column(sectionCol)
            t.column(chunkIndexCol)
            t.column(createdAtCol)
        })
    }

    func insert(chunk: DocumentChunk, vector: [Float]) throws {
        let vectorData = vector.withUnsafeBufferPointer { Data(buffer: $0) }

        try db.run(table.insert(
            idCol <- chunk.id.uuidString,
            contentCol <- chunk.content,
            vectorCol <- vectorData,
            sourceCol <- chunk.metadata.source,
            pageIndexCol <- chunk.metadata.pageIndex,
            sectionCol <- chunk.metadata.section,
            chunkIndexCol <- chunk.metadata.chunkIndex,
            createdAtCol <- chunk.metadata.createdAt
        ))
    }

    func search(queryVector: [Float], topK: Int = 5) throws [(chunk: DocumentChunk, score: Float)] {
        let queryData = queryVector.withUnsafeBufferPointer { Data(buffer: $0) }

        var results: [(chunk: DocumentChunk, score: Float)] = []

        for row in try db.prepare(table) {
            let storedData = row[vectorCol]
            let storedVector: [Float] = storedData.withUnsafeBytes {
                Array(UnsafeBufferPointer<Float>(start: $0.baseAddress!.assumingMemoryBound(to: Float.self),
                                                 count: storedData.count / MemoryLayout<Float>.size))
            }

            let score = VectorMath.cosineSimilarity(queryVector, storedVector)
            let chunk = try rowToChunk(row)
            results.append((chunk: chunk, score: score))
        }

        results.sort { $0.score > $1.score }
        return Array(results.prefix(topK))
    }

    private func rowToChunk(_ row: Row) throws -> DocumentChunk {
        DocumentChunk(
            id: UUID(uuidString: row[idCol]) ?? UUID(),
            content: row[contentCol],
            metadata: DocumentChunk.ChunkMetadata(
                source: row[sourceCol],
                fileName: row[sourceCol],
                pageIndex: row[pageIndexCol],
                section: row[sectionCol],
                chunkIndex: row[chunkIndexCol],
                createdAt: row[createdAtCol],
                charCount: row[contentCol].count,
                tags: []
            )
        )
    }

    func delete(source: String) throws {
        let rows = table.filter(sourceCol == source)
        try db.run(rows.delete())
    }

    func count() throws -> Int {
        try db.scalar(table.count)
    }
}
```

**方案二：本地 JSON 存储（适合小型知识库）**

```swift
struct JSONVectorStore {
    let storageDir: URL

    struct VectorEntry: Codable {
        let id: String
        let content: String
        let vector: [Float]
        let metadata: DocumentChunk.ChunkMetadata
    }

    init(storageDir: URL) throws {
        self.storageDir = storageDir
        try FileManager.default.createDirectory(at: storageDir, withIntermediateDirectories: true)
    }

    func insert(chunk: DocumentChunk, vector: [Float]) throws {
        let entry = VectorEntry(
            id: chunk.id.uuidString,
            content: chunk.content,
            vector: vector,
            metadata: chunk.metadata
        )
        let data = try JSONEncoder().encode(entry)
        let fileURL = storageDir.appendingPathComponent("\(chunk.id.uuidString).json")
        try data.write(to: fileURL)
    }

    func search(queryVector: [Float], topK: Int = 5) throws -> [(chunk: DocumentChunk, score: Float)] {
        let files = try FileManager.default.contentsOfDirectory(at: storageDir, includingPropertiesForKeys: nil)
        var results: [(chunk: DocumentChunk, score: Float)] = []

        for fileURL in files where fileURL.pathExtension == "json" {
            let data = try Data(contentsOf: fileURL)
            let entry = try JSONDecoder().decode(VectorEntry.self, from: data)

            let score = VectorMath.cosineSimilarity(queryVector, entry.vector)
            let chunk = DocumentChunk(
                id: UUID(uuidString: entry.id) ?? UUID(),
                content: entry.content,
                metadata: entry.metadata
            )
            results.append((chunk: chunk, score: score))
        }

        results.sort { $0.score > $1.score }
        return Array(results.prefix(topK))
    }

    func deleteAll() throws {
        let files = try FileManager.default.contentsOfDirectory(at: storageDir, includingPropertiesForKeys: nil)
        for fileURL in files where fileURL.pathExtension == "json" {
            try FileManager.default.removeItem(at: fileURL)
        }
    }
}
```

⚠️ **警告**：JSON 存储方案仅适合数百到数千条记录的小型知识库。当记录超过 1 万条时，全量加载和遍历的性能会显著下降，应改用 SQLite 方案或远程向量数据库。

### 5.4 向量相似度计算

```swift
struct VectorMath {
    static func cosineSimilarity(_ a: [Float], _ b: [Float]) -> Float {
        guard a.count == b.count, !a.isEmpty else { return 0 }

        var dotProduct: Float = 0
        var normA: Float = 0
        var normB: Float = 0

        for i in 0..<a.count {
            dotProduct += a[i] * b[i]
            normA += a[i] * a[i]
            normB += b[i] * b[i]
        }

        let denominator = sqrt(normA) * sqrt(normB)
        guard denominator > 0 else { return 0 }
        return dotProduct / denominator
    }

    static func euclideanDistance(_ a: [Float], _ b: [Float]) -> Float {
        guard a.count == b.count else { return .infinity }

        var sum: Float = 0
        for i in 0..<a.count {
            let diff = a[i] - b[i]
            sum += diff * diff
        }
        return sqrt(sum)
    }

    static func dotProduct(_ a: [Float], _ b: [Float]) -> Float {
        guard a.count == b.count else { return 0 }

        var result: Float = 0
        for i in 0..<a.count {
            result += a[i] * b[i]
        }
        return result
    }

    static func normalize(_ vector: [Float]) -> [Float] {
        let norm = sqrt(vector.reduce(0) { $0 + $1 * $1 })
        guard norm > 0 else { return vector }
        return vector.map { $0 / norm }
    }
}
```

三种相似度度量对比：

| 度量方式 | 范围 | 特点 | 适用场景 |
|---------|------|------|---------|
| **余弦相似度** | [-1, 1] | 忽略向量长度，关注方向 | 文本语义检索（最常用） |
| **欧几里得距离** | [0, +∞) | 考虑向量绝对距离 | 需要考虑幅度的场景 |
| **点积** | (-∞, +∞) | 同时考虑方向和长度 | 已归一化的向量检索 |

> 💡 **提示**：大多数嵌入模型输出的向量已经归一化（长度为 1），此时余弦相似度与点积等价。如果向量已归一化，使用点积可以省去归一化计算，速度更快。

---

## 6. 检索与重排

### 6.1 语义检索实现

语义检索是 RAG 的核心步骤——根据用户问题的向量，在知识库中找到最相关的文档块：

```swift
struct SemanticRetriever {
    let vectorStore: VectorStore
    let embeddingService: EmbeddingService

    struct RetrievalResult {
        let chunks: [DocumentChunk]
        let scores: [Float]
    }

    func retrieve(query: String, topK: Int = 5, minScore: Float = 0.5) async throws -> RetrievalResult {
        let queryVector = try await embeddingService.embed(text: query)

        let results = try vectorStore.search(queryVector: queryVector, topK: topK)

        let filtered = results.filter { $0.score >= minScore }

        return RetrievalResult(
            chunks: filtered.map { $0.chunk },
            scores: filtered.map { $0.score }
        )
    }
}
```

### 6.2 混合检索（关键词 + 语义）

纯语义检索可能遗漏关键词精确匹配的结果，混合检索结合两者优势：

```
混合检索流程：

用户问题："SwiftUI 中 @State 的用法"
        │
        ├──→ 语义检索 → 找到语义相关的文档块
        │
        ├──→ 关键词检索 → 找到包含 "@State" 的文档块
        │
        └──→ 融合排序 → 合并去重，按综合分数排序
```

```swift
struct HybridRetriever {
    let semanticRetriever: SemanticRetriever
    let keywordRetriever: KeywordRetriever

    struct HybridResult {
        let chunk: DocumentChunk
        let semanticScore: Float
        let keywordScore: Float
        var combinedScore: Float {
            0.7 * semanticScore + 0.3 * keywordScore
        }
    }

    func retrieve(query: String, topK: Int = 5) async throws -> [HybridResult] {
        async let semanticResults = semanticRetriever.retrieve(query: query, topK: topK * 2)
        let keywordResults = keywordRetriever.search(query: query, topK: topK * 2)

        let semantic = try await semanticResults

        var resultMap: [String: HybridResult] = [:]

        for (index, chunk) in semantic.chunks.enumerated() {
            let score = semantic.scores[index]
            resultMap[chunk.id.uuidString] = HybridResult(
                chunk: chunk,
                semanticScore: score,
                keywordScore: 0
            )
        }

        for result in keywordResults {
            let key = result.chunk.id.uuidString
            if var existing = resultMap[key] {
                existing.keywordScore = result.score
                resultMap[key] = existing
            } else {
                resultMap[key] = HybridResult(
                    chunk: result.chunk,
                    semanticScore: 0,
                    keywordScore: result.score
                )
            }
        }

        let sorted = resultMap.values.sorted { $0.combinedScore > $1.combinedScore }
        return Array(sorted.prefix(topK))
    }
}

struct KeywordRetriever {
    let vectorStore: VectorStore

    struct KeywordResult {
        let chunk: DocumentChunk
        let score: Float
    }

    func search(query: String, topK: Int = 5) -> [KeywordResult] {
        let keywords = extractKeywords(from: query)

        var results: [KeywordResult] = []

        do {
            let allChunks = try vectorStore.getAllChunks()
            for chunk in allChunks {
                let score = computeKeywordScore(keywords: keywords, content: chunk.content)
                if score > 0 {
                    results.append(KeywordResult(chunk: chunk, score: score))
                }
            }
        } catch {
            return []
        }

        results.sort { $0.score > $1.score }
        return Array(results.prefix(topK))
    }

    private func extractKeywords(from query: String) -> [String] {
        let stopWords: Set<String> = ["的", "了", "是", "在", "有", "和", "与", "或", "吗", "呢", "怎么", "如何", "什么"]
        return query
            .components(separatedBy: .whitespacesAndNewlines)
            .flatMap { $0.components(separatedBy: CharacterSet(charactersIn: "，。！？、；：""''")) }
            .filter { !$0.isEmpty && !stopWords.contains($0) }
    }

    private func computeKeywordScore(keywords: [String], content: String) -> Float {
        var score: Float = 0
        let lowerContent = content.lowercased()
        for keyword in keywords {
            let lowerKeyword = keyword.lowercased()
            var count: Float = 0
            var searchStart = lowerContent.startIndex
            while let range = lowerContent.range(of: lowerKeyword, range: searchStart..<lowerContent.endIndex) {
                count += 1
                searchStart = range.upperBound
            }
            score += count
        }
        return score
    }
}
```

### 6.3 重排模型（Reranker）的作用

检索阶段使用向量相似度进行粗排，速度快但精度有限。重排模型（Reranker）对粗排结果进行精确排序，显著提升最终结果质量：

```
检索 + 重排流程：

知识库（10000 条）
      │
      ▼
  向量检索（Top-20）  ← 粗排，速度快
      │
      ▼
  重排模型（Top-5）   ← 精排，精度高
      │
      ▼
  最终结果
```

| 重排模型 | 提供方 | 特点 | 延迟 |
|---------|--------|------|------|
| **bge-reranker-v2-m3** | BAAI | 开源，中英文 | 中 |
| **cohere-rerank** | Cohere | 商业 API，多语言 | 低 |
| **dashscope-rerank** | 通义千问 | 中文优化 | 低 |
| **bce-reranker-base_v1** | 网易 | 开源，中文优化 | 中 |

```swift
struct RerankerService {
    let apiKey: String
    let baseURL: String
    let model: String

    static let dashscope = RerankerService(
        apiKey: "",
        baseURL: "https://dashscope.aliyuncs.com/api/v1",
        model: "gte-rerank"
    )

    struct RerankResult {
        let index: Int
        let score: Float
        let chunk: DocumentChunk
    }

    func rerank(query: String, chunks: [DocumentChunk], topN: Int = 5) async throws -> [RerankResult] {
        var request = URLRequest(url: URL(string: "\(baseURL)/services/aigc/text-ranking/generation")!)
        request.httpMethod = "POST"
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")
        request.setValue("Bearer \(apiKey)", forHTTPHeaderField: "Authorization")

        let documents = chunks.map { $0.content }
        let body: [String: Any] = [
            "model": model,
            "parameters": [
                "top_n": topN
            ],
            "input": [
                "query": query,
                "documents": documents
            ]
        ]
        request.httpBody = try JSONSerialization.data(withJSONObject: body)

        let (data, response) = try await URLSession.shared.data(for: request)

        guard let httpResponse = response as? HTTPURLResponse,
              (200...299).contains(httpResponse.statusCode) else {
            throw RerankError.apiError
        }

        let result = try JSONDecoder().decode(RerankResponse.self, from: data)

        return result.results.map { rerankItem in
            RerankResult(
                index: rerankItem.index,
                score: rerankItem.relevanceScore,
                chunk: chunks[rerankItem.index]
            )
        }
    }
}

struct RerankResponse: Codable {
    let results: [RerankItem]

    struct RerankItem: Codable {
        let index: Int
        let relevanceScore: Float

        enum CodingKeys: String, CodingKey {
            case index
            case relevanceScore = "relevance_score"
        }
    }
}

enum RerankError: LocalizedError {
    case apiError

    var errorDescription: String? {
        switch self {
        case .apiError: return "重排 API 调用失败"
        }
    }
}
```

### 6.4 Top-K 参数调优

Top-K 决定了检索返回多少个文档块，直接影响生成质量：

| Top-K 值 | 优点 | 缺点 | 适用场景 |
|----------|------|------|---------|
| **1-3** | 上下文精确，Token 消耗低 | 可能遗漏相关信息 | 简单问答 |
| **5** | 平衡精度和召回 | 适中 | 通用场景（推荐） |
| **10-20** | 召回率高 | 噪声增加，Token 消耗高 | 复杂问题 + 重排 |

> 💡 **提示**：推荐使用"大 Top-K 检索 + 重排后取小 Top-N"的策略。例如先检索 Top-20，重排后取 Top-5，兼顾召回率和精度。

### 6.5 检索质量评估

评估 RAG 检索质量的常用指标：

| 指标 | 含义 | 计算方式 |
|------|------|---------|
| **召回率（Recall）** | 相关文档被检索到的比例 | 检索到的相关文档数 / 总相关文档数 |
| **精确率（Precision）** | 检索结果中相关文档的比例 | 检索到的相关文档数 / 检索结果总数 |
| **MRR** | 第一个正确结果的排名倒数 | 1 / 第一个相关结果的排名 |
| **nDCG** | 考虑排序位置的增益 | 归一化的折损累积增益 |

```swift
struct RetrievalEvaluator {
    struct EvaluationResult {
        let recall: Float
        let precision: Float
        let mrr: Float
    }

    static func evaluate(
        retrieved: [String],
        relevant: Set<String>
    ) -> EvaluationResult {
        let retrievedSet = Set(retrieved)
        let truePositives = retrievedSet.intersection(relevant).count

        let recall = relevant.isEmpty ? 0 : Float(truePositives) / Float(relevant.count)
        let precision = retrieved.isEmpty ? 0 : Float(truePositives) / Float(retrieved.count)

        var mrr: Float = 0
        for (index, id) in retrieved.enumerated() {
            if relevant.contains(id) {
                mrr = 1.0 / Float(index + 1)
                break
            }
        }

        return EvaluationResult(recall: recall, precision: precision, mrr: mrr)
    }
}
```

---

## 7. 完整 RAG Pipeline 实现

### 7.1 RAGService 架构设计

```
┌──────────────────────────────────────────────────────┐
│                    RAGService                         │
│                                                      │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │
│  │ DocumentProc │  │ EmbeddingSer│  │ VectorStore  │  │
│  │ 文档处理+分块│  │ 向量化服务   │  │ 向量存储     │  │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  │
│         │                │                │          │
│  ┌──────┴────────────────┴────────────────┴──────┐  │
│  │              索引流程 (Index)                   │  │
│  │  文档 → 分块 → 嵌入 → 存储                      │  │
│  └───────────────────────────────────────────────┘  │
│                                                      │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │
│  │ HybridRetri │  │ RerankerSer │  │ LLMService   │  │
│  │ 混合检索     │  │ 重排服务     │  │ 大模型服务    │  │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  │
│         │                │                │          │
│  ┌──────┴────────────────┴────────────────┴──────┐  │
│  │              查询流程 (Query)                   │  │
│  │  问题 → 嵌入 → 检索 → 重排 → Prompt → LLM      │  │
│  └───────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────┘
```

### 7.2 文档索引流程

```swift
actor RAGService {
    private let documentProcessor = DocumentProcessor.self
    private let embeddingService: EmbeddingService
    private let vectorStore: VectorStore
    private let rerankerService: RerankerService?
    private let llmService: LLMService

    struct IndexProgress {
        let totalChunks: Int
        let processedChunks: Int
        var progress: Float {
            totalChunks == 0 ? 0 : Float(processedChunks) / Float(totalChunks)
        }
    }

    init(
        embeddingService: EmbeddingService,
        vectorStore: VectorStore,
        rerankerService: RerankerService? = nil,
        llmService: LLMService
    ) {
        self.embeddingService = embeddingService
        self.vectorStore = vectorStore
        self.rerankerService = rerankerService
        self.llmService = llmService
    }

    func indexDocument(url: URL, progressHandler: ((IndexProgress) -> Void)? = nil) async throws {
        let chunks = try await DocumentProcessor.extractText(from: url)

        let batchSize = 20
        var processed = 0

        for batchStart in stride(from: 0, to: chunks.count, by: batchSize) {
            let batchEnd = min(batchStart + batchSize, chunks.count)
            let batch = Array(chunks[batchStart..<batchEnd])

            let texts = batch.map { $0.content }
            let embeddings = try await embeddingService.embed(texts: texts)

            for (index, chunk) in batch.enumerated() {
                guard index < embeddings.count else { break }
                try vectorStore.insert(chunk: chunk, vector: embeddings[index])
            }

            processed = batchEnd
            progressHandler?(IndexProgress(totalChunks: chunks.count, processedChunks: processed))
        }
    }

    func indexDocuments(urls: [URL], progressHandler: ((IndexProgress) -> Void)? = nil) async throws {
        var totalChunks = 0
        var processedChunks = 0

        var allChunks: [DocumentChunk] = []
        for url in urls {
            let chunks = try await DocumentProcessor.extractText(from: url)
            allChunks.append(contentsOf: chunks)
        }
        totalChunks = allChunks.count

        let batchSize = 20
        for batchStart in stride(from: 0, to: allChunks.count, by: batchSize) {
            let batchEnd = min(batchStart + batchSize, allChunks.count)
            let batch = Array(allChunks[batchStart..<batchEnd])

            let texts = batch.map { $0.content }
            let embeddings = try await embeddingService.embed(texts: texts)

            for (index, chunk) in batch.enumerated() {
                guard index < embeddings.count else { break }
                try vectorStore.insert(chunk: chunk, vector: embeddings[index])
            }

            processedChunks = batchEnd
            progressHandler?(IndexProgress(totalChunks: totalChunks, processedChunks: processedChunks))
        }
    }
}
```

### 7.3 查询流程

```swift
extension RAGService {
    struct QueryResult {
        let answer: String
        let sources: [SourceReference]
        let retrievedChunks: [DocumentChunk]
        let retrievalScores: [Float]
    }

    struct SourceReference {
        let source: String
        let pageIndex: Int?
        let section: String?
        let relevantText: String
    }

    func query(
        _ question: String,
        topK: Int = 20,
        topN: Int = 5,
        minScore: Float = 0.3
    ) async throws -> QueryResult {
        let queryVector = try await embeddingService.embed(text: question)

        let semanticResults = try vectorStore.search(queryVector: queryVector, topK: topK)

        let filteredResults = semanticResults.filter { $0.score >= minScore }

        let finalChunks: [DocumentChunk]
        let finalScores: [Float]

        if let reranker = rerankerService, filteredResults.count > topN {
            let chunksToRerank = filteredResults.map { $0.chunk }
            let reranked = try await reranker.rerank(
                query: question,
                chunks: chunksToRerank,
                topN: topN
            )
            finalChunks = reranked.map { $0.chunk }
            finalScores = reranked.map { $0.score }
        } else {
            finalChunks = Array(filteredResults.prefix(topN).map { $0.chunk })
            finalScores = Array(filteredResults.prefix(topN).map { $0.score })
        }

        let prompt = buildPrompt(question: question, chunks: finalChunks)

        let answer = try await llmService.complete(prompt: prompt)

        let sources = finalChunks.map { chunk in
            SourceReference(
                source: chunk.metadata.source,
                pageIndex: chunk.metadata.pageIndex,
                section: chunk.metadata.section,
                relevantText: String(chunk.content.prefix(100))
            )
        }

        return QueryResult(
            answer: answer,
            sources: sources,
            retrievedChunks: finalChunks,
            retrievalScores: finalScores
        )
    }

    private func buildPrompt(question: String, chunks: [DocumentChunk]) -> String {
        var context = ""
        for (index, chunk) in chunks.enumerated() {
            context += "[文档\(index + 1)]"
            if let section = chunk.metadata.section {
                context += " 章节: \(section)"
            }
            if let page = chunk.metadata.pageIndex {
                context += " 第\(page + 1)页"
            }
            context += "\n\(chunk.content)\n\n"
        }

        return """
        你是一个专业的知识库问答助手。请基于以下参考文档回答用户的问题。

        要求：
        1. 只基于参考文档中的信息回答，不要编造内容
        2. 如果参考文档中没有相关信息，请明确说明
        3. 引用信息时请注明来源文档编号
        4. 回答要准确、完整、有条理

        参考文档：
        \(context)

        用户问题：\(question)
        """
    }
}
```

### 7.4 完整 Swift 代码实现

将所有组件整合为可运行的完整实现：

```swift
import Foundation

struct LLMService {
    let apiKey: String
    let baseURL: String
    let model: String

    static let openAI = LLMService(
        apiKey: "",
        baseURL: "https://api.openai.com/v1",
        model: "gpt-4o-mini"
    )

    static let dashscope = LLMService(
        apiKey: "",
        baseURL: "https://dashscope.aliyuncs.com/compatible-mode/v1",
        model: "qwen-plus"
    )

    func complete(prompt: String) async throws -> String {
        var request = URLRequest(url: URL(string: "\(baseURL)/chat/completions")!)
        request.httpMethod = "POST"
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")
        request.setValue("Bearer \(apiKey)", forHTTPHeaderField: "Authorization")

        let body: [String: Any] = [
            "model": model,
            "messages": [
                ["role": "user", "content": prompt]
            ],
            "temperature": 0.3,
            "max_tokens": 2000
        ]
        request.httpBody = try JSONSerialization.data(withJSONObject: body)

        let (data, response) = try await URLSession.shared.data(for: request)

        guard let httpResponse = response as? HTTPURLResponse,
              (200...299).contains(httpResponse.statusCode) else {
            throw LLMError.requestFailed
        }

        let result = try JSONDecoder().decode(LLMResponse.self, from: data)
        return result.choices.first?.message.content ?? ""
    }

    func completeStream(prompt: String) -> AsyncThrowingStream<String, Error> {
        AsyncThrowingStream { continuation in
            Task {
                var request = URLRequest(url: URL(string: "\(baseURL)/chat/completions")!)
                request.httpMethod = "POST"
                request.setValue("application/json", forHTTPHeaderField: "Content-Type")
                request.setValue("Bearer \(apiKey)", forHTTPHeaderField: "Authorization")

                let body: [String: Any] = [
                    "model": model,
                    "messages": [
                        ["role": "user", "content": prompt]
                    ],
                    "temperature": 0.3,
                    "max_tokens": 2000,
                    "stream": true
                ]
                request.httpBody = try JSONSerialization.data(withJSONObject: body)

                let (bytes, response) = try await URLSession.shared.bytes(for: request)

                guard let httpResponse = response as? HTTPURLResponse,
                      (200...299).contains(httpResponse.statusCode) else {
                    continuation.finish(throwing: LLMError.requestFailed)
                    return
                }

                for try await line in bytes.lines {
                    guard line.hasPrefix("data: ") else { continue }
                    let jsonString = String(line.dropFirst(6))
                    guard jsonString != "[DONE]" else {
                        continuation.finish()
                        return
                    }
                    if let jsonData = jsonString.data(using: .utf8),
                       let chunk = try? JSONDecoder().decode(SSEChunk.self, from: jsonData),
                       let content = chunk.choices.first?.delta.content {
                        continuation.yield(content)
                    }
                }
                continuation.finish()
            }
        }
    }
}

struct LLMResponse: Codable {
    let choices: [Choice]

    struct Choice: Codable {
        let message: Message
    }

    struct Message: Codable {
        let content: String
    }
}

struct SSEChunk: Codable {
    let choices: [SSEChoice]

    struct SSEChoice: Codable {
        let delta: Delta
    }

    struct Delta: Codable {
        let content: String?
    }
}

enum LLMError: LocalizedError {
    case requestFailed

    var errorDescription: String? {
        switch self {
        case .requestFailed: return "LLM 请求失败"
        }
    }
}
```

VectorStore 扩展方法：

```swift
extension VectorStore {
    func getAllChunks() throws -> [DocumentChunk] {
        var chunks: [DocumentChunk] = []
        for row in try db.prepare(table) {
            let chunk = try rowToChunk(row)
            chunks.append(chunk)
        }
        return chunks
    }

    func batchInsert(chunks: [DocumentChunk], vectors: [[Float]]) throws {
        try db.transaction {
            for (index, chunk) in chunks.enumerated() {
                guard index < vectors.count else { break }
                try insert(chunk: chunk, vector: vectors[index])
            }
        }
    }
}
```

使用示例：

```swift
@MainActor
class KnowledgeBaseViewModel: ObservableObject {
    @Published var isIndexing = false
    @Published var indexingProgress: Float = 0
    @Published var answer: String = ""
    @Published var sources: [RAGService.SourceReference] = []
    @Published var isQuerying = false

    private var ragService: RAGService?

    func setup() throws {
        let dbPath = try FileManager.default
            .url(for: .applicationSupportDirectory, in: .userDomainMask, appropriateFor: nil, create: true)
            .appendingPathComponent("rag_vectors.db")
            .path

        let vectorStore = try VectorStore(dbPath: dbPath)
        let embeddingService = EmbeddingService.dashscope
        let llmService = LLMService.dashscope

        ragService = RAGService(
            embeddingService: embeddingService,
            vectorStore: vectorStore,
            llmService: llmService
        )
    }

    func importDocument(url: URL) async throws {
        guard let service = ragService else { return }
        isIndexing = true

        try await service.indexDocument(url: url) { progress in
            Task { @MainActor in
                self.indexingProgress = progress.progress
            }
        }

        isIndexing = false
        indexingProgress = 1.0
    }

    func ask(_ question: String) async throws {
        guard let service = ragService else { return }
        isQuerying = true

        let result = try await service.query(question)
        answer = result.answer
        sources = result.sources

        isQuerying = false
    }
}
```

---

## 8. 本地知识库方案

### 8.1 无需服务器的轻量 RAG

对于小型知识库（数百条 FAQ、产品说明等），可以实现完全离线的本地 RAG 方案：

```
本地 RAG 方案架构：

┌─────────────────────────────────────────┐
│              iOS App                     │
│                                         │
│  ┌───────────┐    ┌───────────────┐     │
│  │ 预计算嵌入  │    │ 本地向量存储    │     │
│  │ (构建时)   │    │ (SQLite/JSON) │     │
│  └─────┬─────┘    └───────┬───────┘     │
│        │                  │             │
│        ▼                  ▼             │
│  ┌───────────────────────────────┐      │
│  │        本地检索引擎            │      │
│  └───────────────┬───────────────┘      │
│                  │                      │
│  ┌───────────┐   │   ┌───────────────┐  │
│  │ Core ML   │───┤   │ 远程 LLM API  │  │
│  │ 嵌入模型   │   │   │ (仅生成步骤)  │  │
│  └───────────┘   │   └───────┬───────┘  │
│                  ▼           │          │
│          ┌───────────────────┘          │
│          ▼                              │
│  ┌───────────────┐                      │
│  │   最终回答     │                      │
│  └───────────────┘                      │
└─────────────────────────────────────────┘
```

### 8.2 预计算嵌入 + 本地存储

在 App 构建时预计算所有文档的嵌入向量，打包进 App：

```bash
# 使用 Python 预计算嵌入
python3 precompute_embeddings.py \
  --input ./knowledge_base/ \
  --output ./ios_app/Resources/knowledge_vectors.json \
  --model text-embedding-v3 \
  --chunk-size 500 \
  --overlap 50
```

```python
# precompute_embeddings.py
import json
import os
import sys

def main():
    import argparse
    parser = argparse.ArgumentParser()
    parser.add_argument("--input", required=True)
    parser.add_argument("--output", required=True)
    parser.add_argument("--model", default="text-embedding-v3")
    parser.add_argument("--chunk-size", type=int, default=500)
    parser.add_argument("--overlap", type=int, default=50)
    args = parser.parse_args()

    chunks = []
    for root, dirs, files in os.walk(args.input):
        for filename in files:
            filepath = os.path.join(root, filename)
            ext = filename.split(".")[-1].lower()
            if ext in ["txt", "md"]:
                with open(filepath, "r", encoding="utf-8") as f:
                    text = f.read()
                text_chunks = split_text(text, args.chunk_size, args.overlap)
                for i, chunk in enumerate(text_chunks):
                    chunks.append({
                        "content": chunk,
                        "source": filename,
                        "chunk_index": i,
                    })

    embeddings = compute_embeddings(chunks, args.model)

    result = []
    for chunk, embedding in zip(chunks, embeddings):
        chunk["vector"] = embedding
        result.append(chunk)

    with open(args.output, "w", encoding="utf-8") as f:
        json.dump(result, f, ensure_ascii=False)

    print(f"Processed {len(chunks)} chunks, saved to {args.output}")

def split_text(text, chunk_size, overlap):
    chunks = []
    start = 0
    while start < len(text):
        end = min(start + chunk_size, len(text))
        chunks.append(text[start:end])
        start += chunk_size - overlap
    return chunks

def compute_embeddings(chunks, model):
    import openai
    client = openai.OpenAI()
    texts = [c["content"] for c in chunks]
    embeddings = []
    batch_size = 20
    for i in range(0, len(texts), batch_size):
        batch = texts[i:i+batch_size]
        response = client.embeddings.create(model=model, input=batch)
        for item in response.data:
            embeddings.append(item.embedding)
    return embeddings

if __name__ == "__main__":
    main()
```

### 8.3 App 内置知识库

将预计算的向量数据打包进 App Bundle，启动时加载：

```swift
struct BundledKnowledgeBase {
    private let entries: [KnowledgeEntry]
    private let vectorStore: JSONVectorStore

    struct KnowledgeEntry: Codable {
        let content: String
        let source: String
        let chunkIndex: Int
        let vector: [Float]
    }

    init() throws {
        guard let url = Bundle.main.url(forResource: "knowledge_vectors", withExtension: "json") else {
            throw KnowledgeError.bundleNotFound
        }

        let data = try Data(contentsOf: url)
        entries = try JSONDecoder().decode([KnowledgeEntry].self, from: data)

        let cacheDir = FileManager.default.urls(for: .cachesDirectory, in: .userDomainMask)[0]
            .appendingPathComponent("knowledge_base", isDirectory: true)
        vectorStore = try JSONVectorStore(storageDir: cacheDir)
    }

    func search(query: String, embeddingService: EmbeddingService, topK: Int = 5) async throws -> [(entry: KnowledgeEntry, score: Float)] {
        let queryVector = try await embeddingService.embed(text: query)

        var results: [(entry: KnowledgeEntry, score: Float)] = []
        for entry in entries {
            let score = VectorMath.cosineSimilarity(queryVector, entry.vector)
            results.append((entry: entry, score: score))
        }

        results.sort { $0.score > $1.score }
        return Array(results.prefix(topK))
    }

    func searchLocally(query: String, localEmbedding: LocalEmbeddingService, topK: Int = 5) async throws -> [(entry: KnowledgeEntry, score: Float)] {
        let queryVector = try await localEmbedding.embed(text: query)

        var results: [(entry: KnowledgeEntry, score: Float)] = []
        for entry in entries {
            let score = VectorMath.cosineSimilarity(queryVector, entry.vector)
            results.append((entry: entry, score: score))
        }

        results.sort { $0.score > $1.score }
        return Array(results.prefix(topK))
    }

    var totalEntries: Int { entries.count }
}

enum KnowledgeError: LocalizedError {
    case bundleNotFound

    var errorDescription: String? {
        switch self {
        case .bundleNotFound: return "内置知识库文件未找到"
        }
    }
}
```

### 8.4 适合小型知识库的方案

| 方案 | 数据量 | 离线支持 | 更新方式 | 复杂度 |
|------|--------|---------|---------|--------|
| **内置 JSON** | < 1000 条 | ✅ 完全离线 | App 更新 | 低 |
| **内置 SQLite** | < 10000 条 | ✅ 完全离线 | App 更新 | 中 |
| **远程下载 + 本地缓存** | < 50000 条 | ⚠️ 首次需网络 | 增量更新 | 中 |
| **远程向量数据库** | 无限制 | ❌ 需网络 | 实时 | 高 |

> 💡 **提示**：对于 FAQ 类应用，推荐使用"内置 JSON + 远程 LLM"方案。嵌入向量预计算打包进 App（通常只有几 MB），检索在本地完成，只有最终生成步骤需要调用 LLM API。这样既保证了检索速度，又减少了 API 调用成本。

---

## 9. RAG 效果优化

### 9.1 Chunk 策略优化

分块策略是影响 RAG 效果的首要因素：

| 优化方向 | 方法 | 效果 |
|---------|------|------|
| **大小调优** | 根据文档类型选择合适的 chunk_size | 减少噪声，提升精度 |
| **语义边界** | 按段落/章节而非固定长度切割 | 保留语义完整性 |
| **父子块** | 检索小块，返回大块上下文 | 兼顾精度和上下文 |
| **滑动窗口** | 增大 overlap 比例 | 避免关键信息被切断 |

**父子块策略**（Parent-Child Chunking）是一种高级优化：

```
父块（1000 字）：用于提供完整上下文
  └── 子块（200 字）：用于精确检索

检索流程：
1. 用子块做向量检索（精度高）
2. 找到匹配的子块后，返回对应的父块（上下文完整）
3. 将父块作为 LLM 的输入
```

```swift
struct ParentChildChunker {
    struct ParentChunk: Identifiable {
        let id: UUID
        let content: String
        let children: [ChildChunk]
        let metadata: DocumentChunk.ChunkMetadata
    }

    struct ChildChunk: Identifiable {
        let id: UUID
        let content: String
        let parentId: UUID
        let metadata: DocumentChunk.ChunkMetadata
    }

    static func chunk(
        text: String,
        parentSize: Int = 1000,
        childSize: Int = 200,
        parentOverlap: Int = 100,
        childOverlap: Int = 20,
        baseMetadata: DocumentChunk.ChunkMetadata
    ) -> [ParentChunk] {
        let parentTexts = fixedSizeSplit(text: text, chunkSize: parentSize, overlap: parentOverlap)

        var parents: [ParentChunk] = []
        for (index, parentText) in parentTexts.enumerated() {
            let parentId = UUID()
            let childTexts = fixedSizeSplit(text: parentText, chunkSize: childSize, overlap: childOverlap)

            let children = childTexts.enumerated().map { childIndex, childText in
                ChildChunk(
                    id: UUID(),
                    content: childText,
                    parentId: parentId,
                    metadata: DocumentChunk.ChunkMetadata(
                        source: baseMetadata.source,
                        fileName: baseMetadata.fileName,
                        pageIndex: baseMetadata.pageIndex,
                        section: baseMetadata.section,
                        chunkIndex: childIndex,
                        createdAt: baseMetadata.createdAt,
                        charCount: childText.count,
                        tags: baseMetadata.tags
                    )
                )
            }

            parents.append(ParentChunk(
                id: parentId,
                content: parentText,
                children: children,
                metadata: DocumentChunk.ChunkMetadata(
                    source: baseMetadata.source,
                    fileName: baseMetadata.fileName,
                    pageIndex: baseMetadata.pageIndex,
                    section: baseMetadata.section,
                    chunkIndex: index,
                    createdAt: baseMetadata.createdAt,
                    charCount: parentText.count,
                    tags: baseMetadata.tags
                )
            ))
        }

        return parents
    }

    private static func fixedSizeSplit(text: String, chunkSize: Int, overlap: Int) -> [String] {
        var chunks: [String] = []
        let chars = Array(text)
        var start = 0

        while start < chars.count {
            let end = min(start + chunkSize, chars.count)
            chunks.append(String(chars[start..<end]))
            start += chunkSize - overlap
        }

        return chunks
    }
}
```

### 9.2 查询改写（Query Rewriting）

用户的问题可能表述模糊或不完整，查询改写可以优化检索效果：

```swift
struct QueryRewriter {
    let llmService: LLMService

    func rewrite(query: String) async throws -> [String] {
        let prompt = """
        用户提出了以下问题，请生成 3 个不同角度的改写版本，以便更全面地检索相关信息。
        每行一个改写版本，不要编号。

        原始问题：\(query)
        """

        let response = try await llmService.complete(prompt: prompt)
        let rewrites = response
            .components(separatedBy: "\n")
            .map { $0.trimmingCharacters(in: .whitespaces) }
            .filter { !$0.isEmpty }

        return [query] + rewrites
    }

    func decompose(query: String) async throws -> [String] {
        let prompt = """
        将以下复杂问题分解为 2-4 个简单的子问题，每行一个，不要编号。

        问题：\(query)
        """

        let response = try await llmService.complete(prompt: prompt)
        let subQueries = response
            .components(separatedBy: "\n")
            .map { $0.trimmingCharacters(in: .whitespaces) }
            .filter { !$0.isEmpty }

        return subQueries.isEmpty ? [query] : subQueries
    }

    func hyDE(query: String) async throws -> String {
        let prompt = """
        请针对以下问题，写一段可能包含答案的文档内容（约 100 字）。
        不需要答案完全正确，只需语义相关即可。

        问题：\(query)
        """

        return try await llmService.complete(prompt: prompt)
    }
}
```

三种查询改写策略对比：

| 策略 | 原理 | 优点 | 缺点 | 适用场景 |
|------|------|------|------|---------|
| **多角度改写** | 生成多个不同表述的查询 | 增加召回率 | API 调用增加 | 表述模糊的问题 |
| **问题分解** | 将复杂问题拆为子问题 | 处理多跳推理 | 需要合并答案 | 复杂多步问题 |
| **HyDE** | 先生成假设性答案，用答案检索 | 提升语义匹配度 | 假设答案可能偏题 | 专业术语问题 |

> 💡 **提示**：HyDE（Hypothetical Document Embedding）的巧妙之处在于——用户的问题可能很短（如"年假怎么请？"），但假设性答案会包含更多相关术语（如"年假申请流程、审批人、所需材料"），用假设答案去检索，比用短问题检索效果更好。

### 9.3 上下文压缩

检索到的文档块可能很长，包含大量与问题无关的内容。上下文压缩可以提取关键信息，减少 Token 消耗：

```swift
struct ContextCompressor {
    let llmService: LLMService

    func compress(chunks: [DocumentChunk], query: String) async throws -> [String] {
        var compressed: [String] = []

        for chunk in chunks {
            let prompt = """
            从以下文本中提取与问题" \(query) "相关的关键信息。
            只输出相关信息，删除无关内容。保持原文表述，不要改写。

            文本：\(chunk.content)
            """

            let result = try await llmService.complete(prompt: prompt)
            compressed.append(result)
        }

        return compressed
    }
}
```

⚠️ **警告**：上下文压缩需要额外的 LLM 调用，会增加延迟和成本。建议仅在检索结果很长（单块超过 1000 字）且 Token 预算紧张时使用。

### 9.4 引用溯源

让 RAG 回答标注信息来源，增强可信度：

```swift
struct CitedAnswer {
    let answer: String
    let citations: [Citation]
}

struct Citation {
    let documentIndex: Int
    let source: String
    let pageIndex: Int?
    let section: String?
    let quotedText: String
}

extension RAGService {
    func queryWithCitation(_ question: String, topK: Int = 5) async throws -> CitedAnswer {
        let queryVector = try await embeddingService.embed(text: question)
        let results = try vectorStore.search(queryVector: queryVector, topK: topK)

        let chunks = results.map { $0.chunk }

        var contextBlock = ""
        for (index, chunk) in chunks.enumerated() {
            contextBlock += "[文档\(index + 1)] 来源: \(chunk.metadata.source)"
            if let page = chunk.metadata.pageIndex {
                contextBlock += " 第\(page + 1)页"
            }
            if let section = chunk.metadata.section {
                contextBlock += " 章节: \(section)"
            }
            contextBlock += "\n\(chunk.content)\n\n"
        }

        let prompt = """
        基于以下参考文档回答问题。回答时请在引用信息后标注来源，格式为[文档编号]。

        参考文档：
        \(contextBlock)

        问题：\(question)
        """

        let answer = try await llmService.complete(prompt: prompt)

        var citations: [Citation] = []
        let pattern = #"\[文档(\d+)\]"#
        if let regex = try? NSRegularExpression(pattern: pattern) {
            let matches = regex.matches(in: answer, range: NSRange(answer.startIndex..., in: answer))
            var citedIndices = Set<Int>()
            for match in matches {
                if let range = Range(match.range(at: 1), in: answer),
                   let index = Int(answer[range]) {
                    citedIndices.insert(index - 1)
                }
            }
            for index in citedIndices where index < chunks.count {
                let chunk = chunks[index]
                citations.append(Citation(
                    documentIndex: index,
                    source: chunk.metadata.source,
                    pageIndex: chunk.metadata.pageIndex,
                    section: chunk.metadata.section,
                    quotedText: String(chunk.content.prefix(100))
                ))
            }
        }

        return CitedAnswer(answer: answer, citations: citations)
    }
}
```

### 9.5 多轮对话 RAG

在多轮对话中，用户的问题可能依赖上下文，需要结合对话历史进行检索：

```swift
struct ConversationalRAG {
    let ragService: RAGService
    let llmService: LLMService

    struct ConversationContext {
        var history: [ChatMessage] = []
        var maxHistoryTurns: Int = 5
    }

    struct ChatMessage {
        let role: Role
        let content: String

        enum Role: String {
            case user, assistant
        }
    }

    func query(
        _ question: String,
        context: inout ConversationContext
    ) async throws -> String {
        let contextualizedQuery = await contextualizeQuery(
            question: question,
            history: context.history
        )

        let result = try await ragService.query(contextualizedQuery)

        context.history.append(ChatMessage(role: .user, content: question))
        context.history.append(ChatMessage(role: .assistant, content: result.answer))

        if context.history.count > context.maxHistoryTurns * 2 {
            context.history = Array(context.history.suffix(context.maxHistoryTurns * 2))
        }

        return result.answer
    }

    private func contextualizeQuery(question: String, history: [ChatMessage]) async -> String {
        guard !history.isEmpty else { return question }

        let historyText = history.suffix(4).map { "\($0.role.rawValue): \($0.content)" }.joined(separator: "\n")

        let prompt = """
        基于以下对话历史，将用户最新问题改写为独立的、包含必要上下文的问题。
        只输出改写后的问题，不要解释。

        对话历史：
        \(historyText)

        最新问题：\(question)
        """

        do {
            return try await llmService.complete(prompt: prompt)
        } catch {
            return question
        }
    }
}
```

多轮对话 RAG 的关键挑战：

| 挑战 | 说明 | 解决方案 |
|------|------|---------|
| **指代消解** | "它怎么用？"中的"它"指什么 | 用 LLM 改写为完整问题 |
| **上下文窗口** | 对话历史越来越长 | 截断历史 + 摘要压缩 |
| **检索偏移** | 话题切换后仍检索旧话题 | 基于改写后的问题检索 |
| **重复检索** | 相似问题重复检索相同内容 | 缓存检索结果 |

---

## 10. 生产环境注意事项

### 10.1 知识库更新策略

知识库内容会随时间变化，需要建立更新机制：

| 策略 | 方式 | 适用场景 | 复杂度 |
|------|------|---------|--------|
| **全量重建** | 删除旧索引，重新构建 | 文档变动大 | 低 |
| **增量更新** | 只处理新增/修改的文档 | 文档频繁更新 | 中 |
| **版本管理** | 为每个文档维护版本号 | 需要回滚能力 | 高 |
| **双写切换** | 新旧索引并行，切换读取 | 零停机更新 | 高 |

```swift
struct KnowledgeBaseUpdater {
    let vectorStore: VectorStore
    let embeddingService: EmbeddingService

    struct DocumentVersion: Codable {
        let fileName: String
        let hash: String
        let updatedAt: Date
        let chunkCount: Int
    }

    func incrementalUpdate(
        urls: [URL],
        versions: inout [String: DocumentVersion]
    ) async throws -> UpdateResult {
        var addedCount = 0
        var updatedCount = 0
        var removedCount = 0

        let currentFiles = Set(urls.map { $0.lastPathComponent })
        let storedFiles = Set(versions.keys)

        let deletedFiles = storedFiles.subtracting(currentFiles)
        for fileName in deletedFiles {
            try vectorStore.delete(source: fileName)
            versions.removeValue(forKey: fileName)
            removedCount += 1
        }

        for url in urls {
            let fileName = url.lastPathComponent
            let currentHash = try computeFileHash(url: url)

            if let existing = versions[fileName] {
                if existing.hash == currentHash { continue }

                try vectorStore.delete(source: fileName)

                let chunks = try await DocumentProcessor.extractText(from: url)
                let texts = chunks.map { $0.content }
                let embeddings = try await embeddingService.embed(texts: texts)

                for (index, chunk) in chunks.enumerated() {
                    guard index < embeddings.count else { break }
                    try vectorStore.insert(chunk: chunk, vector: embeddings[index])
                }

                versions[fileName] = DocumentVersion(
                    fileName: fileName,
                    hash: currentHash,
                    updatedAt: Date(),
                    chunkCount: chunks.count
                )
                updatedCount += 1
            } else {
                let chunks = try await DocumentProcessor.extractText(from: url)
                let texts = chunks.map { $0.content }
                let embeddings = try await embeddingService.embed(texts: texts)

                for (index, chunk) in chunks.enumerated() {
                    guard index < embeddings.count else { break }
                    try vectorStore.insert(chunk: chunk, vector: embeddings[index])
                }

                versions[fileName] = DocumentVersion(
                    fileName: fileName,
                    hash: currentHash,
                    updatedAt: Date(),
                    chunkCount: chunks.count
                )
                addedCount += 1
            }
        }

        return UpdateResult(added: addedCount, updated: updatedCount, removed: removedCount)
    }

    private func computeFileHash(url: URL) throws -> String {
        let data = try Data(contentsOf: url)
        var hash = [UInt8](repeating: 0, count: Int(CC_SHA256_DIGEST_LENGTH))
        data.withUnsafeBytes {
            _ = CC_SHA256($0.baseAddress, CC_LONG(data.count), &hash)
        }
        return hash.map { String(format: "%02x", $0) }.joined()
    }
}

struct UpdateResult {
    let added: Int
    let updated: Int
    let removed: Int
}
```

### 10.2 大规模文档处理

当文档数量达到数万甚至数十万时，需要考虑性能优化：

| 优化方向 | 方法 | 效果 |
|---------|------|------|
| **批量嵌入** | 一次 API 调用嵌入多条文本 | 减少 API 调用次数 |
| **并行处理** | 多线程/协程并行处理文档 | 加速索引构建 |
| **增量索引** | 只处理新增/修改的文档 | 避免全量重建 |
| **索引分片** | 按类别/时间分片存储 | 减少检索范围 |
| **缓存机制** | 缓存热门问题的检索结果 | 减少重复计算 |

```swift
struct BatchEmbeddingService {
    let embeddingService: EmbeddingService
    let batchSize: Int
    let maxConcurrency: Int

    func embedAll(texts: [String]) async throws -> [[Float]] {
        try await withThrowingTaskGroup(of: [[Float]].self) { group in
            let batches = texts.chunked(into: batchSize)

            let semaphore = AsyncSemaphore(limit: maxConcurrency)

            for (index, batch) in batches.enumerated() {
                group.addTask {
                    await semaphore.wait()
                    defer { semaphore.signal() }
                    return try await self.embeddingService.embed(texts: batch)
                }
            }

            var allEmbeddings: [[Float]] = []
            allEmbeddings.reserveCapacity(texts.count)
            for try await batchResult in group {
                allEmbeddings.append(contentsOf: batchResult)
            }
            return allEmbeddings
        }
    }
}

final class AsyncSemaphore {
    private let limit: Int
    private var count: Int
    private var waiters: [CheckedContinuation<Void, Never>] = []

    init(limit: Int) {
        self.limit = limit
        self.count = limit
    }

    func wait() async {
        if count > 0 {
            count -= 1
            return
        }
        await withCheckedContinuation { continuation in
            waiters.append(continuation)
        }
    }

    func signal() {
        if waiters.isEmpty {
            count += 1
        } else {
            waiters.removeFirst().resume()
        }
    }
}

extension Array {
    func chunked(into size: Int) -> [[Element]] {
        stride(from: 0, to: count, by: size).map {
            Array(self[$0..<Swift.min($0 + size, count)])
        }
    }
}
```

### 10.3 成本控制

RAG 系统的主要成本来源：

| 成本项 | 计费方式 | 优化策略 |
|--------|---------|---------|
| **嵌入 API** | 按 Token 计费 | 批量调用、缓存结果、本地嵌入 |
| **LLM API** | 按 Token 计费 | 压缩上下文、选择合适模型、缓存回答 |
| **向量数据库** | 按存储/查询计费 | 选择合适规模、本地方案 |
| **重排 API** | 按次数计费 | 仅在必要时使用、控制 Top-K |
| **带宽** | 数据传输 | 本地缓存、压缩传输 |

```swift
struct CostEstimator {
    struct CostBreakdown {
        var embeddingCost: Float = 0
        var llmCost: Float = 0
        var rerankCost: Float = 0
        var totalCost: Float { embeddingCost + llmCost + rerankCost }
    }

    static func estimateQueryCost(
        queryTokens: Int,
        contextTokens: Int,
        responseTokens: Int,
        rerankCount: Int
    ) -> CostBreakdown {
        var cost = CostBreakdown()

        cost.embeddingCost = Float(queryTokens) / 1_000_000 * 0.02

        let totalLLMTokens = queryTokens + contextTokens + responseTokens
        cost.llmCost = Float(totalLLMTokens) / 1_000_000 * 0.15

        cost.rerankCost = Float(rerankCount) * 0.001

        return cost
    }

    static func estimateIndexingCost(
        totalTokens: Int,
        embeddingPricePerMillion: Float = 0.02
    ) -> Float {
        Float(totalTokens) / 1_000_000 * embeddingPricePerMillion
    }
}
```

### 10.4 隐私与安全

RAG 系统处理的数据可能包含敏感信息，需要特别注意隐私安全：

| 安全维度 | 风险 | 防护措施 |
|---------|------|---------|
| **数据传输** | 中间人攻击 | HTTPS 加密传输 |
| **数据存储** | 数据泄露 | 向量数据库加密、访问控制 |
| **API 调用** | 密钥泄露 | 服务端代理、密钥轮换 |
| **用户输入** | Prompt 注入 | 输入过滤、Prompt 加固 |
| **检索结果** | 越权访问 | 基于用户权限过滤 |
| **日志记录** | 敏感信息泄露 | 脱敏处理、日志加密 |

Prompt 注入防护：

```swift
struct PromptGuard {
    static let injectionPatterns = [
        "忽略以上指令",
        "ignore previous instructions",
        "disregard all above",
        "你现在是",
        "you are now",
        "system:",
        "###",
    ]

    static func checkInput(_ input: String) -> PromptCheckResult {
        let lowerInput = input.lowercased()

        for pattern in injectionPatterns {
            if lowerInput.contains(pattern) {
                return PromptCheckResult(isSafe: false, reason: "检测到潜在注入攻击: \(pattern)")
            }
        }

        if input.count > 2000 {
            return PromptCheckResult(isSafe: false, reason: "输入过长，可能包含隐藏指令")
        }

        return PromptCheckResult(isSafe: true, reason: nil)
    }

    static func sanitizePrompt(_ prompt: String) -> String {
        var sanitized = prompt
        sanitized = sanitized.replacingOccurrences(of: "```", with: "")
        sanitized = sanitized.replacingOccurrences(of: "---", with: "")
        return sanitized
    }
}

struct PromptCheckResult {
    let isSafe: Bool
    let reason: String?
}
```

基于权限的检索过滤：

```swift
struct PermissionFilteredRetriever {
    let vectorStore: VectorStore
    let embeddingService: EmbeddingService

    struct UserPermission {
        let accessibleSources: Set<String>?
        let accessibleSections: Set<String>?
        let accessLevel: AccessLevel

        enum AccessLevel {
            case full, restricted, readonly
        }
    }

    func retrieve(
        query: String,
        permission: UserPermission,
        topK: Int = 5
    ) async throws -> [(chunk: DocumentChunk, score: Float)] {
        let queryVector = try await embeddingService.embed(text: query)
        let results = try vectorStore.search(queryVector: queryVector, topK: topK * 3)

        let filtered = results.filter { result in
            if let allowedSources = permission.accessibleSources {
                guard allowedSources.contains(result.chunk.metadata.source) else { return false }
            }
            if let allowedSections = permission.accessibleSections,
               let section = result.chunk.metadata.section {
                guard allowedSections.contains(section) else { return false }
            }
            return true
        }

        return Array(filtered.prefix(topK))
    }
}
```

### 10.5 用户体验设计

RAG 应用的用户体验设计要点：

**1. 引用来源展示**

```swift
struct CitationView: View {
    let citations: [Citation]

    var body: some View {
        VStack(alignment: .leading, spacing: 8) {
            Text("参考来源")
                .font(.caption)
                .foregroundStyle(.secondary)

            ForEach(Array(citations.enumerated()), id: \.offset) { index, citation in
                HStack(alignment: .top, spacing: 6) {
                    Text("[\(index + 1)]")
                        .font(.caption2)
                        .foregroundStyle(.accentColor)
                        .frame(width: 24)

                    VStack(alignment: .leading, spacing: 2) {
                        Text(citation.source)
                            .font(.caption)
                            .fontWeight(.medium)
                        if let page = citation.pageIndex {
                            Text("第 \(page + 1) 页")
                                .font(.caption2)
                                .foregroundStyle(.secondary)
                        }
                        if let section = citation.section {
                            Text(section)
                                .font(.caption2)
                                .foregroundStyle(.secondary)
                        }
                    }
                }
            }
        }
        .padding(12)
        .background(Color(.systemGray6))
        .clipShape(RoundedRectangle(cornerRadius: 8))
    }
}
```

**2. 置信度展示**

```swift
struct ConfidenceIndicator: View {
    let score: Float

    var body: some View {
        HStack(spacing: 4) {
            Circle()
                .fill(confidenceColor)
                .frame(width: 8, height: 8)
            Text(confidenceLabel)
                .font(.caption2)
                .foregroundStyle(.secondary)
        }
    }

    private var confidenceColor: Color {
        switch score {
        case 0.8...1.0: return .green
        case 0.6..<0.8: return .orange
        default: return .red
        }
    }

    private var confidenceLabel: String {
        switch score {
        case 0.8...1.0: return "高置信度"
        case 0.6..<0.8: return "中等置信度"
        case 0.4..<0.6: return "低置信度"
        default: return "参考信息不足"
        }
    }
}
```

**3. 知识库管理界面**

```swift
struct KnowledgeBaseManagementView: View {
    @StateObject private var viewModel = KnowledgeBaseViewModel()
    @State private var showImporter = false

    var body: some View {
        List {
            Section("知识库状态") {
                HStack {
                    Text("文档数量")
                    Spacer()
                    Text("\(viewModel.documentCount) 个文档")
                        .foregroundStyle(.secondary)
                }
                HStack {
                    Text("索引块数")
                    Spacer()
                    Text("\(viewModel.chunkCount) 个块")
                        .foregroundStyle(.secondary)
                }
                HStack {
                    Text="存储大小"
                    Spacer()
                    Text(viewModel.storageSize)
                        .foregroundStyle(.secondary)
                }
            }

            Section("文档列表") {
                ForEach(viewModel.documents) { doc in
                    HStack {
                        VStack(alignment: .leading) {
                            Text(doc.fileName)
                                .font(.subheadline)
                            Text(doc.updatedAt.formatted(date: .abbreviated, time: .shortened))
                                .font(.caption2)
                                .foregroundStyle(.secondary)
                        }
                        Spacer()
                        Text("\(doc.chunkCount) 块")
                            .font(.caption)
                            .foregroundStyle(.secondary)
                    }
                }
                .onDelete { indexSet in
                    viewModel.deleteDocuments(at: indexSet)
                }
            }

            Section("操作") {
                Button {
                    showImporter = true
                } label: {
                    Label("导入文档", systemImage: "doc.badge.plus")
                }

                Button {
                    Task { await viewModel.rebuildIndex() }
                } label: {
                    Label("重建索引", systemImage: "arrow.clockwise")
                }

                Button(role: .destructive) {
                    viewModel.clearAll()
                } label: {
                    Label("清空知识库", systemImage: "trash")
                }
            }
        }
        .navigationTitle("知识库管理")
        .fileImporter(isPresented: $showImporter, allowedContentTypes: [.pdf, .plainText]) { result in
            switch result {
            case .success(let url):
                Task { try await viewModel.importDocument(url: url) }
            case .failure:
                break
            }
        }
    }
}
```

**4. 检索过程可视化**

让用户了解 RAG 的工作过程，增强信任感：

```swift
struct RAGProcessView: View {
    let stage: RAGStage
    let isComplete: Bool

    enum RAGStage: Int, CaseIterable {
        case embedding = 0
        case retrieving = 1
        case reranking = 2
        case generating = 3

        var label: String {
            switch self {
            case .embedding: return "理解问题"
            case .retrieving: return "检索文档"
            case .reranking: return "排序筛选"
            case .generating: return "生成回答"
            }
        }

        var icon: String {
            switch self {
            case .embedding: return "brain"
            case .retrieving: return "magnifyingglass"
            case .reranking: return "arrow.up.arrow.down"
            case .generating: return "text.bubble"
            }
        }
    }

    var body: some View {
        HStack(spacing: 12) {
            ForEach(RAGStage.allCases, id: \.rawValue) { s in
                VStack(spacing: 4) {
                    ZStack {
                        Circle()
                            .fill(stageColor(for: s))
                            .frame(width: 32, height: 32)

                        Image(systemName: s.icon)
                            .font(.caption)
                            .foregroundStyle(.white)
                    }

                    Text(s.label)
                        .font(.caption2)
                        .foregroundStyle(stage >= s ? .primary : .secondary)
                }
            }
        }
        .padding()
    }

    private func stageColor(for s: RAGStage) -> Color {
        if stage.rawValue > s.rawValue {
            return .green
        } else if stage.rawValue == s.rawValue {
            return .accentColor
        } else {
            return Color(.systemGray4)
        }
    }
}
```

---

## 小结

本章全面介绍了 RAG（检索增强生成）与知识库问答的核心原理与实战实现，关键要点如下：

| 主题 | 关键要点 |
|------|---------|
| **RAG 原理** | 检索→增强→生成三阶段，解决 LLM 知识截止、幻觉、私有数据三大问题 |
| **RAG 架构** | 完整 Pipeline：文档→分块→嵌入→存储→检索→重排→生成 |
| **文档分块** | 递归分块最常用，chunk_size 500-800、overlap 50-100 是通用起点 |
| **文本嵌入** | 中文场景推荐通义/智谱模型，离线场景可用 Core ML 本地模型 |
| **向量数据库** | 云端选 Pinecone/国内云，iOS 端用 SQLite 轻量方案 |
| **检索与重排** | 混合检索（语义+关键词）+ 重排模型，Top-20 检索 + 重排取 Top-5 |
| **完整实现** | RAGService 统一管理索引和查询流程，Swift 完整代码可直接使用 |
| **本地方案** | 预计算嵌入打包进 App，适合小型知识库的离线 RAG |
| **效果优化** | 父子块、查询改写、HyDE、上下文压缩、引用溯源、多轮对话 |
| **生产环境** | 增量更新、成本控制、隐私安全、权限过滤、用户体验设计 |

RAG 是让 LLM 从"通才"变为"专家"的关键技术。对于 iOS 开发者来说，从轻量本地方案起步，逐步演进到云端方案，是最务实的路径。建议先实现核心 Pipeline（分块→嵌入→检索→生成），验证效果后再逐步添加重排、查询改写等优化。

下一章我们将探讨国内大模型与 AI 生态，了解如何选择和集成适合国内用户的大模型服务。

---

← [构建AI对话界面](./构建AI对话界面.md) | [国内大模型与AI生态](./国内大模型与AI生态.md) →