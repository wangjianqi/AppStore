# 57-Core ML 与设备端 AI

## 本章目标

- 理解设备端 AI 的核心优势，掌握与云端 AI 的适用场景对比
- 掌握 Core ML 框架基础：MLModel、预测流程、模型格式
- 学会使用 Create ML 零代码训练图像分类、文本分类、表格回归、声音分类模型
- 掌握将 Core ML 模型集成到 SwiftUI App 的完整流程
- 熟练使用 Vision 框架进行人脸检测、文字识别、物体追踪、条码扫描
- 熟练使用 Natural Language 框架进行分词、语言识别、情感分析、文本嵌入
- 掌握 Speech 框架的语音识别与实时转写
- 了解 Core ML 模型优化：量化、压缩与性能分析
- 学会使用 coremltools 从 PyTorch/TensorFlow 转换模型
- 理解设备端 AI 的隐私优势与隐私清单合规声明

---

## 1. 设备端 AI 概述

### 1.1 为什么在设备上跑 AI

传统 AI 推理依赖云端服务器——用户数据上传到远端，模型在服务器上运算后返回结果。但越来越多的场景要求 AI 在本地设备上直接运行。

> 💡 **生活类比**：云端 AI 像去医院做检查——样本送远端实验室，等结果回来；设备端 AI 像家里备了体温计——即测即得，数据不出门。

设备端 AI 的核心优势：

| 优势 | 说明 |
|------|------|
| **低延迟** | 无网络往返，推理在毫秒级完成 |
| **离线可用** | 无网络时仍可正常使用 AI 功能 |
| **隐私保护** | 数据不离开设备，无需上传服务器 |
| **零成本** | 不依赖云服务器，无 API 调用费用 |
| **一致性** | 不受网络波动影响，体验稳定 |

### 1.2 设备端 AI vs 云端 AI

| 对比项 | 设备端 AI | 云端 AI |
|-------|:---------:|:-------:|
| 延迟 | 毫秒级 | 百毫秒~秒级 |
| 离线能力 | ✅ | ❌ |
| 隐私性 | ⭐⭐⭐⭐⭐ | ⭐⭐ |
| 模型大小 | 受限（几十 MB） | 无限制（数十 GB） |
| 算力 | 有限 | 几乎无限 |
| 推理精度 | 适中（模型较小） | 高（可用大模型） |
| 运行成本 | 零 | 按调用计费 |
| 适用场景 | 实时、隐私敏感、离线 | 复杂推理、大模型、批量处理 |

> ⚠️ **注意**：设备端和云端不是非此即彼。很多 App 采用**混合策略**——简单任务本地处理，复杂任务上传云端。例如照片 App 在本地做基础分类，复杂场景识别交给云端。

### 1.3 Apple 芯片与 Neural Engine

Apple 自 A11 仿生芯片起内置了 **Neural Engine（NPU）**，专门加速神经网络运算：

| 芯片 | Neural Engine 核心数 | 设备 |
|------|:---:|------|
| A11 | 2 核 | iPhone 8 / X |
| A12 | 8 核 | iPhone XS / XR |
| A14 | 16 核 | iPhone 12 |
| A15 | 16 核 | iPhone 13 |
| A16 | 16 核 | iPhone 14 Pro |
| A17 Pro | 16 核 | iPhone 15 Pro |
| M1 | 16 核 | Mac |
| M2 | 16 核 | Mac |
| M3 / M4 | 16/18 核 | Mac / iPad |

Core ML 会自动将模型运算分配到最合适的处理器：

```
┌──────────────────────────────────────────────┐
│              Core ML 调度引擎                  │
│  根据模型特征自动选择最优计算单元               │
├──────────┬──────────┬──────────┬─────────────┤
│  Neural  │   GPU    │   CPU    │  Neural     │
│  Engine  │ (Metal)  │          │  Engine     │
│  (NPU)   │          │          │  (ANE)      │
│          │          │          │             │
│ 卷积/循环 │  图像处理 │ 通用计算  │ 矩阵运算    │
│ 神经网络  │  着色器   │ 逻辑控制  │ 批量推理    │
└──────────┴──────────┴──────────┴─────────────┘
```

> 💡 **关键点**：开发者无需手动指定运算设备，Core ML 会根据模型类型和设备能力自动调度。Neural Engine 的功耗仅为 GPU 的 1/10，非常适合持续运行的 AI 任务。

---

## 2. Core ML 框架基础

### 2.1 MLModel 核心概念

Core ML 是 Apple 推出的设备端机器学习框架，核心类是 `MLModel`。它封装了模型的输入输出规格和推理逻辑：

```swift
import CoreML

let config = MLModelConfiguration()
config.computeUnits = .all  // 允许使用所有计算单元（NPU + GPU + CPU）

let model = try MyClassifier(configuration: config)
```

### 2.2 预测流程

Core ML 的预测流程非常简洁——构造输入、调用预测、处理输出：

```swift
// 1. 构造输入
let input = MyClassifierInput(feature1: 0.5, feature2: 1.2)

// 2. 调用预测
let prediction = try model.prediction(input: input)

// 3. 处理输出
print(prediction.label)       // 分类标签
print(prediction.probability) // 各类别概率
```

对于异步预测（避免阻塞主线程）：

```swift
Task {
    let prediction = try await model.prediction(input: input)
    await MainActor.run {
        resultLabel = prediction.label
    }
}
```

### 2.3 模型格式

| 格式 | 扩展名 | 说明 |
|------|--------|------|
| **编译模型** | `.mlmodelc` | Xcode 编译后的二进制格式，打包进 App |
| **源模型** | `.mlmodel` | 文本描述 + 权重的中间格式，Xcode 自动编译 |
| **包模型** | `.mlpackage` | 目录结构，支持更大模型，Xcode 13+ |

> ⚠️ **注意**：拖入 Xcode 项目的 `.mlmodel` 文件会在编译时自动转为 `.mlmodelc`。运行时加载的模型需要手动使用 `MLModel.compileModel(at:)` 编译。

---

## 3. 使用 Create ML 训练模型

Create ML 是 Apple 提供的**零代码**模型训练工具，适合常见 ML 任务。你可以通过 Xcode 的 Create ML App 或 Swift Playgrounds 使用。

> 💡 **生活类比**：Create ML 像微波炉——放食材进去，按预设按钮，出来就是成品。不需要懂底层原理，但能解决 80% 的日常需求。

### 3.1 图像分类

**场景**：识别 App 中用户上传的图片类别（如美食、风景、宠物）。

**步骤**：
1. 打开 Xcode → File → New → Create ML Project
2. 选择 Image Classification 模板
3. 准备训练数据：按类别分文件夹组织图片
   ```
   Training/
   ├── 美食/
   │   ├── img001.jpg
   │   └── img002.jpg
   ├── 风景/
   │   ├── img003.jpg
   │   └── img004.jpg
   └── 宠物/
       ├── img005.jpg
       └── img006.jpg
   ```
4. 拖入训练集和测试集
5. 点击 Train，等待训练完成
6. 在 Preview 标签页用测试图片验证效果
7. 点击 Output → Get 导出 `.mlmodel` 文件

### 3.2 文本分类

**场景**：对用户反馈自动分类（Bug 报告、功能请求、投诉表扬）。

**数据格式**（JSON）：

```json
[
  {"text": "App 闪退了，打开就崩溃", "label": "Bug报告"},
  {"text": "希望能增加暗黑模式", "label": "功能请求"},
  {"text": "体验非常好，五星好评", "label": "表扬"},
  {"text": "为什么登录不了", "label": "Bug报告"}
]
```

在 Create ML 中选择 Text Classification 模板，导入 JSON 数据即可训练。

### 3.3 表格回归

**场景**：根据房屋面积、房间数、地段等特征预测房价。

**数据格式**（CSV）：

```csv
area,rooms,district,price
89,3,朝阳,650
120,4,海淀,980
60,2,丰台,420
```

选择 Tabular Regression 模板，指定目标列（price），Create ML 自动选择算法并训练。

### 3.4 声音分类

**场景**：识别环境音（婴儿哭声、门铃、警报器）。

选择 Sound Classification 模板，按类别组织 `.wav` / `.m4a` 音频文件，训练方式与图像分类类似。

### 3.5 Create ML 支持的任务类型

| 任务 | 模板名称 | 输入 | 输出 |
|------|---------|------|------|
| 图像分类 | Image Classification | 图片 | 类别标签 + 概率 |
| 物体检测 | Object Detection | 图片 | 边界框 + 类别 |
| 文本分类 | Text Classification | 文本 | 类别标签 |
| 表格分类 | Tabular Classification | 表格行 | 类别标签 |
| 表格回归 | Tabular Regression | 表格行 | 数值 |
| 声音分类 | Sound Classification | 音频 | 类别标签 |
| 活动分类 | Activity Classification | 传感器数据 | 活动类型 |
| 风格迁移 | Style Transfer | 图片 | 风格化图片 |
| 推荐系统 | Recommender | 用户-物品交互 | 推荐列表 |

> 💡 **建议**：训练数据质量比数量更重要。每个类别至少 50 张图片/条文本，数据越多样化，模型泛化能力越强。

---

## 4. 集成 Core ML 模型到 App

### 4.1 拖入模型

将导出的 `.mlmodel` 文件直接拖入 Xcode 项目导航器中。Xcode 会自动：

1. 编译模型为 `.mlmodelc`
2. 生成 Swift 类型安全的封装类

在 Xcode 中点击 `.mlmodel` 文件，可以查看模型的元信息、输入输出规格和预览界面。

### 4.2 自动生成的 Swift 类

假设模型文件名为 `FoodClassifier.mlmodel`，Xcode 会自动生成：

```swift
// 输入类
class FoodClassifierInput: MLFeatureProvider {
    var image: CVPixelBuffer
    var featureNames: Set<String> { ["image"] }
    func featureValue(for featureName: String) -> MLFeatureValue? { ... }
}

// 输出类
class FoodClassifierOutput: MLFeatureProvider {
    let classLabel: String
    let foodLabel: [String: Double]  // 各类别概率
}

// 模型类
class FoodClassifier {
    let model: MLModel
    func prediction(input: FoodClassifierInput) throws -> FoodClassifierOutput
    func prediction(image: CVPixelBuffer) throws -> FoodClassifierOutput
}
```

### 4.3 在 SwiftUI 中调用预测

```swift
import SwiftUI
import CoreML
import Vision

struct ContentView: View {
    @State private var resultText = "等待识别..."
    @State private var selectedImage: UIImage?

    var body: some View {
        VStack(spacing: 20) {
            if let image = selectedImage {
                Image(uiImage: image)
                    .resizable()
                    .scaledToFit()
                    .frame(height: 300)
            }

            Text(resultText)
                .font(.headline)

            Button("选择图片并识别") {
                classifyImage()
            }
        }
        .padding()
    }

    func classifyImage() {
        guard let image = selectedImage,
              let pixelBuffer = image.toCVPixelBuffer() else { return }

        do {
            let model = try FoodClassifier(configuration: MLModelConfiguration())
            let prediction = try model.prediction(image: pixelBuffer)
            resultText = "\(prediction.classLabel)（置信度：\(String(format: "%.1f%%", (prediction.foodLabel[prediction.classLabel] ?? 0) * 100))）"
        } catch {
            resultText = "识别失败：\(error.localizedDescription)"
        }
    }
}

extension UIImage {
    func toCVPixelBuffer() -> CVPixelBuffer? {
        let width = Int(size.width)
        let height = Int(size.height)
        let attrs = [kCVPixelBufferCGImageCompatibilityKey: kCFBooleanTrue,
                     kCVPixelBufferCGBitmapContextCompatibilityKey: kCFBooleanTrue] as CFDictionary
        var pixelBuffer: CVPixelBuffer?
        CVPixelBufferCreate(kCFAllocatorDefault, width, height, kCVPixelFormatType_32ARGB, attrs, &pixelBuffer)
        guard let buffer = pixelBuffer else { return nil }
        CVPixelBufferLockBaseAddress(buffer, [])
        defer { CVPixelBufferUnlockBaseAddress(buffer, []) }
        guard let context = CGContext(data: CVPixelBufferGetBaseAddress(buffer),
                                      width: width, height: height,
                                      bitsPerComponent: 8, bytesPerRow: CVPixelBufferGetBytesPerRow(buffer),
                                      space: CGColorSpaceCreateDeviceRGB(),
                                      bitmapInfo: CGImageAlphaInfo.noneSkipFirst.rawValue) else { return nil }
        UIGraphicsPushContext(context)
        draw(in: CGRect(x: 0, y: 0, width: width, height: height))
        UIGraphicsPopContext()
        return buffer
    }
}
```

> ⚠️ **注意**：模型预测是 CPU/GPU 密集型操作，务必在后台线程执行，避免阻塞 UI。使用 `Task {}` 或 `DispatchQueue.global()` 即可。

---

## 5. Vision 框架

Vision 是 Apple 提供的计算机视觉框架，内置多种预训练模型，**开箱即用**，无需自己训练。

> 💡 **生活类比**：Vision 框架像一台多功能扫描仪——不需要你造机器，放进去就能出结果。

### 5.1 人脸检测

```swift
import Vision

func detectFaces(in image: UIImage) {
    guard let cgImage = image.cgImage else { return }

    let request = VNDetectFaceRectanglesRequest { request, error in
        guard let observations = request.results as? [VNFaceObservation] else { return }
        print("检测到 \(observations.count) 张人脸")
        for face in observations {
            print("人脸边界：\(face.boundingBox)")
        }
    }

    let handler = VNImageRequestHandler(cgImage: cgImage, options: [:])
    try? handler.perform([request])
}
```

### 5.2 文字识别（OCR）

```swift
func recognizeText(in image: UIImage) {
    guard let cgImage = image.cgImage else { return }

    let request = VNRecognizeTextRequest { request, error in
        guard let observations = request.results as? [VNRecognizedTextObservation] else { return }
        let text = observations.compactMap { observation in
            observation.topCandidates(1).first?.string
        }.joined(separator: "\n")
        print("识别结果：\n\(text)")
    }

    request.recognitionLevel = .accurate
    request.recognitionLanguages = ["zh-Hans", "en"]

    let handler = VNImageRequestHandler(cgImage: cgImage, options: [:])
    try? handler.perform([request])
}
```

### 5.3 物体追踪

```swift
func startTracking(objectIn image: UIImage, boundingBox: CGRect) {
    guard let cgImage = image.cgImage else { return }

    let request = VNTrackObjectRequest(rect: boundingBox) { request, error in
        guard let observation = request.results?.first as? VNDetectedObjectObservation else { return }
        print("追踪位置：\(observation.boundingBox)")
    }

    request.trackingLevel = .accurate

    let handler = VNImageRequestHandler(cgImage: cgImage, options: [:])
    try? handler.perform([request])
}
```

### 5.4 条码扫描

```swift
func detectBarcodes(in image: UIImage) {
    guard let cgImage = image.cgImage else { return }

    let request = VNDetectBarcodesRequest { request, error in
        guard let observations = request.results as? [VNBarcodeObservation] else { return }
        for barcode in observations {
            print("类型：\(barcode.symbology.rawValue)")
            print("内容：\(barcode.payloadStringValue ?? "无")")
        }
    }

    request.symbologies = [.qr, .ean13, .code128]

    let handler = VNImageRequestHandler(cgImage: cgImage, options: [:])
    try? handler.perform([request])
}
```

### 5.5 Vision 内置请求一览

| 请求类 | 功能 | 典型场景 |
|-------|------|---------|
| `VNDetectFaceRectanglesRequest` | 人脸检测 | 相机滤镜、身份验证 |
| `VNDetectFaceLandmarksRequest` | 人脸关键点 | 表情识别、美颜 |
| `VNDetectTextRectanglesRequest` | 文字区域检测 | 文档扫描 |
| `VNRecognizeTextRequest` | 文字识别（OCR） | 名片识别、票据扫描 |
| `VNTrackObjectRequest` | 物体追踪 | 视频标注、AR |
| `VNDetectBarcodesRequest` | 条码扫描 | 二维码、商品码 |
| `VNClassifyImageRequest` | 图像分类 | 照片管理 |
| `VNDetectHumanRectanglesRequest` | 人体检测 | 姿态识别 |
| `VNRecognizeAnimalsRequest` | 动物识别 | 宠物 App |

---

## 6. Natural Language 框架

Natural Language 框架提供文本分析能力，同样基于 Apple 内置模型，无需训练。

### 6.1 分词

```swift
import NaturalLanguage

let text = "SwiftUI是苹果推出的声明式UI框架"
let tokenizer = NLTokenizer(unit: .word)
tokenizer.string = text

tokenizer.enumerateTokens(in: text.startIndex..<text.endIndex) { range, _ in
    print(String(text[range]))
    return true
}
```

### 6.2 语言识别

```swift
let recognizer = NLLanguageRecognizer()
recognizer.processString("This is an English sentence")
if let language = recognizer.dominantLanguage {
    print("识别语言：\(language.rawValue)")  // "en"
}
```

### 6.3 情感分析

```swift
let sentimentPredictor = try NLModel(mlModel: NLModel.sentimentModel())

func analyzeSentiment(_ text: String) -> (label: String, score: Double) {
    let prediction = sentimentPredictor.prediction(from: text)
    let score = sentimentPredictor.predictedLabelHypotheses(for: text, maximumCount: 3)
    return (prediction, score[prediction] ?? 0)
}

let result = analyzeSentiment("这个产品太棒了，强烈推荐！")
print("情感：\(result.label)，分数：\(result.score)")
```

### 6.4 文本嵌入

```swift
if let embedding = NLEmbedding.wordEmbedding(for: .simplifiedChinese) {
    let similarWords = embedding.neighbors(for: "快乐", maximumCount: 5)
    for (word, distance) in similarWords {
        print("\(word) - 距离：\(distance)")
    }

    if let distance = embedding.distance(between: "快乐", and: "开心") {
        print("快乐与开心的距离：\(distance)")
    }
}
```

### 6.5 Natural Language 功能一览

| 功能 | API | 说明 |
|------|-----|------|
| 分词 | `NLTokenizer` | 按词/句/段落切分 |
| 语言识别 | `NLLanguageRecognizer` | 识别文本语言 |
| 词性标注 | `NLTagger` | 名词、动词、形容词等 |
| 命名实体识别 | `NLTagger` | 人名、地名、组织名 |
| 情感分析 | `NLModel.sentimentModel()` | 正面/负面/中性 |
| 文本嵌入 | `NLEmbedding` | 语义相似度计算 |

---

## 7. Speech 框架

Speech 框架提供语音识别能力，支持实时流式识别和离线识别。

### 7.1 语音识别基础

```swift
import Speech

class SpeechRecognizer: ObservableObject {
    @Published var transcript = ""
    private var recognitionTask: SFSpeechRecognitionTask?
    private var recognitionRequest: SFSpeechAudioBufferRecognitionRequest?
    private let speechRecognizer = SFSpeechRecognizer(locale: Locale(identifier: "zh-CN"))

    func startRecording() throws {
        guard speechRecognizer?.isAvailable == true else {
            throw NSError(domain: "Speech", code: -1, userInfo: [NSLocalizedDescriptionKey: "语音识别不可用"])
        }

        recognitionRequest = SFSpeechAudioBufferRecognitionRequest()
        guard let recognitionRequest = recognitionRequest else { return }

        let audioSession = AVAudioSession.sharedInstance()
        try audioSession.setCategory(.record, mode: .measurement)
        try audioSession.setActive(true, options: .notifyOthersOnDeactivation)

        let inputNode = audioEngine.inputNode
        recognitionRequest.shouldReportPartialResults = true

        recognitionTask = speechRecognizer?.recognitionTask(with: recognitionRequest) { [weak self] result, error in
            if let result = result {
                self?.transcript = result.bestTranscription.formattedString
            }
            if error != nil || result?.isFinal == true {
                self?.stopRecording()
            }
        }

        let format = inputNode.outputFormat(forBus: 0)
        inputNode.installTap(onBus: 0, bufferSize: 1024, format: format) { buffer, _ in
            self.recognitionRequest?.append(buffer)
        }

        audioEngine.prepare()
        try audioEngine.start()
    }

    func stopRecording() {
        audioEngine.stop()
        recognitionRequest?.endAudio()
        recognitionTask?.cancel()
    }
}
```

### 7.2 权限请求

使用 Speech 框架前必须请求用户授权：

```swift
SFSpeechRecognizer.requestAuthorization { status in
    switch status {
    case .authorized:
        print("语音识别已授权")
    case .denied:
        print("用户拒绝了语音识别权限")
    case .notDetermined:
        print("尚未请求权限")
    case .restricted:
        print("设备不支持语音识别")
    @unknown default:
        break
    }
}
```

同时在 Info.plist 中添加：

```xml
<key>NSSpeechRecognitionUsageDescription</key>
<string>需要语音识别权限来提供语音输入功能</string>
<key>NSMicrophoneUsageDescription</key>
<string>需要麦克风权限来录制语音</string>
```

### 7.3 离线识别

iOS 17+ 支持完全离线的语音识别：

```swift
let recognizer = SFSpeechRecognizer(locale: Locale(identifier: "zh-CN"))!
recognizer.supportsOnDeviceRecognition  // 检查是否支持离线识别

let request = SFSpeechAudioBufferRecognitionRequest()
request.requiresOnDeviceRecognition = true  // 强制使用离线模型
```

| 对比项 | 在线识别 | 离线识别 |
|-------|:-------:|:-------:|
| 需要网络 | ✅ | ❌ |
| 识别精度 | 高 | 中等 |
| 支持语言 | 广泛 | 有限 |
| 延迟 | 较高 | 低 |
| 隐私性 | ⭐⭐ | ⭐⭐⭐⭐⭐ |

> ⚠️ **注意**：离线识别需要用户在 设置 → 辅助功能 → 语音控制 中下载离线语言包。首次使用前应检查 `supportsOnDeviceRecognition`。

---

## 8. Core ML 模型优化

模型体积和推理速度直接影响用户体验。Core ML 提供多种优化手段。

### 8.1 量化

量化是将模型权重从高精度（Float32）降低到低精度（Float16 / Int8 / Int4），牺牲少量精度换取更小的体积和更快的推理速度。

> 💡 **生活类比**：量化像图片压缩——原图 10MB 的 PNG 压缩成 500KB 的 JPEG，肉眼几乎看不出差别，但传输快了很多。

| 量化类型 | 精度 | 体积缩减 | 精度损失 | 速度提升 |
|---------|:----:|:--------:|:-------:|:-------:|
| 无量化 | Float32 | 基准 | 无 | 基准 |
| 半精度 | Float16 | ~50% | 极小 | ~2x |
| 线性量化 | Int8 | ~75% | 小 | ~3x |
| 混合量化 | Float16+Int8 | ~60% | 极小 | ~2.5x |

### 8.2 使用 coremltools 量化模型

```python
import coremltools as ct

model = ct.models.MLModel("FoodClassifier.mlmodel")

# 半精度量化
model_fp16 = ct.optimize.coreml.linear_quantize_weights(
    model,
    mode="linear"
)
model_fp16.save("FoodClassifier_fp16.mlmodel")

# Int8 量化
model_int8 = ct.optimize.coreml.linear_quantize_weights(
    model,
    mode="linear_sympmetric",
    dtype=np.int8
)
model_int8.save("FoodClassifier_int8.mlmodel")
```

### 8.3 模型压缩

除了量化，coremltools 还支持权重剪枝和调色板优化：

```python
# 权重剪枝：将接近零的权重设为零
pruned_model = ct.optimize.coreml.prune_weights(
    model,
    threshold=0.01
)

# 调色板优化：将权重聚类为少量离散值
palette_model = ct.optimize.coreml.palettize_weights(
    model,
    mode="kmeans",
    nbits=4
)
```

### 8.4 性能分析工具

Xcode Instruments 提供了 Core ML 性能分析模板：

1. **Product → Profile** 启动 Instruments
2. 选择 **Core ML** 模板
3. 运行 App 并触发预测
4. 查看每次预测的耗时、计算单元分配、内存占用

也可以在代码中测量：

```swift
let start = CFAbsoluteTimeGetCurrent()
let prediction = try model.prediction(input: input)
let elapsed = CFAbsoluteTimeGetCurrent() - start
print("推理耗时：\(String(format: "%.2f", elapsed * 1000)) ms")
```

---

## 9. 从 PyTorch/TensorFlow 转换模型

当你需要使用自定义训练的深度学习模型时，需要通过 coremltools 将 PyTorch 或 TensorFlow 模型转换为 Core ML 格式。

### 9.1 转换流程

```
PyTorch/TensorFlow 模型
        │
        ▼
   导出为中间格式
   (.pt / .onnx / .h5)
        │
        ▼
   coremltools 转换
        │
        ▼
   .mlmodel / .mlpackage
        │
        ▼
   拖入 Xcode 项目
```

### 9.2 PyTorch 模型转换

```python
import torch
import coremltools as ct

class MyModel(torch.nn.Module):
    def forward(self, x):
        return x * 2

model = MyModel()
model.eval()

example_input = torch.rand(1, 3, 224, 224)
traced_model = torch.jit.trace(model, example_input)

mlmodel = ct.convert(
    traced_model,
    inputs=[ct.TensorType(shape=example_input.shape, name="input")]
)

mlmodel.save("MyModel.mlmodel")
```

### 9.3 TensorFlow / Keras 模型转换

```python
import coremltools as ct

# 从 SavedModel 转换
mlmodel = ct.convert(
    "path/to/saved_model",
    inputs=[ct.ImageType(name="input", shape=(1, 224, 224, 3))]
)

# 从 .h5 文件转换
import tensorflow as tf
keras_model = tf.keras.models.load_model("model.h5")
mlmodel = ct.convert(
    keras_model,
    inputs=[ct.ImageType(name="input", shape=(1, 224, 224, 3))]
)

mlmodel.save("MyTFModel.mlmodel")
```

### 9.4 配置输入输出类型

```python
mlmodel = ct.convert(
    traced_model,
    inputs=[
        ct.ImageType(
            name="image",
            shape=(1, 3, 224, 224),
            scale=1.0 / 255.0,
            bias=[0, 0, 0],
            color_layout="RGB"
        )
    ],
    outputs=[
        ct.ClassifierConfig(
            class_labels=["cat", "dog", "bird"]
        )
    ],
    minimum_deployment_target=ct.target.iOS16
)
```

### 9.5 常见转换问题

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| 不支持的算子 | 模型使用了 Core ML 不支持的层 | 用 `ct.convert(..., pass_pipeline=...)` 自定义转换管线 |
| 输入维度不匹配 | Core ML 默认使用固定 batch size | 指定 `ct.TensorType(shape=...)` 或使用动态形状 |
| 精度下降 | Float32 → Float16 转换损失 | 先转换再量化，逐步验证精度 |
| 模型过大 | 超过 App 包大小限制 | 使用量化 + 剪枝 + 调色板优化 |
| 转换超时 | 模型结构复杂 | 使用 ONNX 中间格式分步转换 |

> ⚠️ **重要**：转换后务必用测试数据验证 Core ML 模型的输出与原始模型一致，精度差异应控制在 1% 以内。

---

## 10. 隐私与合规

### 10.1 设备端 AI 的隐私优势

设备端 AI 最大的价值之一是**隐私保护**——用户数据不离开设备，从根本上杜绝了数据泄露风险：

| 隐私维度 | 云端 AI | 设备端 AI |
|---------|:-------:|:---------:|
| 数据传输 | 需上传到服务器 | 不传输 |
| 数据存储 | 服务器留存 | 仅本地 |
| 数据泄露风险 | 服务器被攻击/滥用 | 极低 |
| 合规难度 | 需满足多地法规 | 天然合规 |
| 用户信任度 | 中等 | 高 |

> 💡 **Apple 的隐私承诺**：Apple 明确声明不会通过 Core ML 收集用户数据。所有设备端推理完全在本地完成，Apple 无法访问你的模型输入或输出。

### 10.2 隐私清单声明

如果你的 App 使用了 Core ML 或相关 AI 框架，需要在 `PrivacyInfo.xcprivacy` 中声明：

```xml
<key>NSPrivacyAccessedAPITypes</key>
<array>
    <dict>
        <key>NSPrivacyAccessedAPIType</key>
        <string>NSPrivacyAccessedAPICategoryFileTimestamp</string>
        <key>NSPrivacyAccessedAPITypeReasons</key>
        <array>
            <string>C617.1</string>
        </array>
    </dict>
</array>
```

### 10.3 AI 功能的隐私声明要点

| 声明项 | 说明 | 是否必须 |
|-------|------|:-------:|
| 数据收集声明 | AI 功能是否收集用户数据 | ✅ |
| 数据用途说明 | 收集的数据如何使用 | ✅ |
| 本地处理声明 | 说明 AI 在设备端运行 | 推荐 |
| 模型来源说明 | 模型是自训练还是第三方 | 推荐 |
| 输出准确性免责 | AI 结果可能不准确的提示 | 推荐 |

### 10.4 App Store 审核中的 AI 合规

从 2024 年起，Apple 对 AI 功能的审核更加严格：

1. **不得隐瞒 AI 使用**：如果 App 使用 AI 生成内容，必须在描述中说明
2. **不得用 AI 生成虚假信息**：深度伪造（Deepfake）类 App 将被拒
3. **医疗/金融 AI 需免责**：AI 诊断或投资建议必须声明仅供参考
4. **儿童安全**：面向儿童的 App 中 AI 生成内容需额外审核
5. **模型来源合规**：使用第三方模型需确保训练数据合法

> ⚠️ **注意**：如果你的 App 使用了云端 AI（如调用 OpenAI API），还需要在隐私政策中说明数据如何传输、存储和处理。设备端 AI 则无需这些声明——这是设备端 AI 在合规方面的巨大优势。

---

## 本章小结

| 主题 | 核心要点 |
|------|---------|
| 设备端 AI 概述 | 低延迟、离线可用、隐私保护、零成本；Neural Engine 自动加速 |
| Core ML 基础 | MLModel 封装推理逻辑；.mlmodel → .mlmodelc 自动编译；预测流程：构造输入→调用预测→处理输出 |
| Create ML | 零代码训练：图像分类、文本分类、表格回归、声音分类等；按类别组织数据即可 |
| 集成模型 | 拖入 .mlmodel → Xcode 自动生成 Swift 类 → 调用 prediction API；注意后台线程执行 |
| Vision 框架 | 内置模型开箱即用：人脸检测、OCR、物体追踪、条码扫描、图像分类 |
| Natural Language | 分词、语言识别、情感分析、文本嵌入；基于 Apple 内置模型 |
| Speech 框架 | 语音识别与实时转写；iOS 17+ 支持离线识别；需请求权限 |
| 模型优化 | 量化（Float16/Int8）、剪枝、调色板优化；Instruments 分析性能 |
| 模型转换 | coremltools 从 PyTorch/TensorFlow 转换；注意算子兼容性和精度验证 |
| 隐私合规 | 设备端 AI 天然隐私友好；隐私清单声明；App Store AI 审核要点 |

> 💡 **一句话总结**：Core ML 让 AI 推理在设备上高效运行——Create ML 零代码训练，Vision/NL/Speech 开箱即用，coremltools 桥接生态，设备端运行天然保护隐私。掌握这套工具链，就能在 App 中构建既智能又安全的 AI 功能。

← [-iPad 适配与多窗口](./56-iPad适配与多窗口.md) | [-CloudKit 与 iCloud 同步](./58-CloudKit与iCloud同步.md) →
