---
name: coreml-vision
description: 涉及 CoreML 模型加载、Vision 框架、图像识别、对象检测、YOLO、CLIP、人脸检测、模型转换、推理优化的任务
---

# CoreML / Vision 本地推理

## 模型管理

### 目录结构
```
Resources/
└── Models/
    ├── YOLOv8n_v2.mlpackage       # 版本号在文件名中
    ├── MobileNetV3.mlmodel
    └── CLIP_Quantized.mlpackage   # 量化版标注
```

### 加载规范
- 所有 `.mlmodel` / `.mlpackage` 文件放在 `Resources/Models/` 目录
- 模型加载使用**懒加载**，禁止在 App 启动时同步加载
- 模型版本通过文件名区分（如 `YOLOv8n_v2.mlpackage`），禁止覆盖旧版本
- 启用 Metal 加速：

```swift
final class ModelManager {
    static let shared = ModelManager()

    private var yoloModel: VNCoreMLModel?

    func loadYOLO() throws -> VNCoreMLModel {
        if let model = yoloModel { return model }
        let config = MLModelConfiguration()
        config.computeUnits = .all  // CPU + GPU + Neural Engine
        let model = try YOLOv8n_v2(configuration: config)
        yoloModel = try VNCoreMLModel(for: model.model)
        yoloModel?.inputImageFeatureName = "image"
        yoloModel?.outputFeatureName = "var_894"
        return yoloModel!
    }

    func unloadAll() {
        yoloModel = nil
    }
}
```

---

## Vision Pipeline — 静态图像

### 完整推理流程

```swift
final class ImageDetector {
    private let model: VNCoreMLModel

    init(model: VNCoreMLModel) {
        self.model = model
    }

    func detect(in image: UIImage, confidenceThreshold: Float = 0.5) throws -> [Detection] {
        guard let cgImage = image.cgImage else {
            throw VisionError.invalidImage
        }

        let request = VNCoreMLRequest(model: model) { request, error in
            // 结果在下方处理
        }
        request.imageCropAndScaleOption = .scaleFill

        let handler = VNImageRequestHandler(cgImage: cgImage, options: [:])
        try handler.perform([request])

        guard let observations = request.results as? [VNRecognizedObjectObservation] else {
            return []
        }

        return observations.compactMap { observation in
            guard let label = observation.labels.first,
                  label.confidence >= confidenceThreshold else { return nil }
            return Detection(
                label: label.identifier,
                confidence: label.confidence,
                boundingBox: observation.boundingBox
            )
        }
    }
}

struct Detection {
    let label: String
    let confidence: Float
    let boundingBox: CGRect
}
```

---

## Vision Pipeline — 实时视频流

### 完整实时检测

```swift
final class RealtimeDetector: NSObject {
    private let model: VNCoreMLModel
    private let requestHandler: VNSequenceRequestHandler
    private let processingQueue = DispatchQueue(label: "com.app.vision", qos: .userInitiated)

    var onDetections: (([Detection], CVPixelBuffer) -> Void)?

    private var isProcessing = false
    private var frameCount = 0
    private let processEveryNFrames = 3  // 跳帧：每 3 帧处理 1 帧

    init(model: VNCoreMLModel) {
        self.model = model
        self.requestHandler = VNSequenceRequestHandler()
    }

    func processFrame(_ sampleBuffer: CMSampleBuffer) {
        guard !isProcessing else { return }
        frameCount += 1
        guard frameCount % processEveryNFrames == 0 else { return }

        isProcessing = true
        processingQueue.async { [weak self] in
            self?.performDetection(on: sampleBuffer)
        }
    }

    private func performDetection(on sampleBuffer: CMSampleBuffer) {
        guard let pixelBuffer = CMSampleBufferGetImageBuffer(sampleBuffer) else {
            isProcessing = false
            return
        }

        let request = VNCoreMLRequest(model: model) { [weak self] request, _ in
            let observations = request.results as? [VNRecognizedObjectObservation] ?? []
            let detections = observations.compactMap { observation -> Detection? in
                guard let label = observation.labels.first, label.confidence >= 0.5 else { return nil }
                return Detection(label: label.identifier, confidence: label.confidence, boundingBox: observation.boundingBox)
            }
            DispatchQueue.main.async {
                self?.onDetections?(detections, pixelBuffer)
            }
            self?.isProcessing = false
        }
        request.imageCropAndScaleOption = .scaleFill

        try? requestHandler.perform([request], on: pixelBuffer)
    }
}
```

### AVCaptureSession 集成

```swift
extension RealtimeDetector: AVCaptureVideoDataOutputSampleBufferDelegate {
    func captureOutput(_ output: AVCaptureOutput, didOutput sampleBuffer: CMSampleBuffer, from connection: AVCaptureConnection) {
        processFrame(sampleBuffer)
    }
}
```

---

## 性能规范

| 优化项 | 规范 | 说明 |
|--------|------|------|
| 帧率控制 | 推理分辨率 `640×480` 或 `416×416` | 通过 `sessionPreset` 限制 |
| 跳帧策略 | `alwaysDiscardsLateVideoFrames = true` | 避免帧队列堆积 |
| 跳帧频率 | 每 N 帧处理 1 帧 | N=3 为推荐值，视设备性能调整 |
| 结果防抖 | 连续 N 帧同类才输出 | 避免闪烁，推荐 N=3 |
| 模型加载 | 懒加载 + 后台队列 | 禁止启动时同步加载 |
| 内存管理 | 退出页面时 `unloadAll()` | 大模型占用 100MB+ 内存 |
| UI 更新 | 结果回调 `DispatchQueue.main.async` | 禁止在回调里直接操作 UI |

### 结果防抖

```swift
final class DetectionSmoother {
    private var history: [String: Int] = [:]
    private let requiredFrames = 3

    func shouldEmit(_ detection: Detection) -> Bool {
        let key = detection.label
        history[key] = (history[key] ?? 0) + 1
        return history[key] == requiredFrames
    }

    func reset() {
        history.removeAll()
    }
}
```

---

## 坐标系转换（重要）

Vision 框架坐标系：**左下角原点，Y 轴向上，值域 [0,1]**
UIKit 坐标系：**左上角原点，Y 轴向下，像素值**

**禁止在业务代码里手动做坐标转换**，使用项目内 `VisionCoordinateConverter.swift`：

```swift
final class VisionCoordinateConverter {
    static func convert(normalizedRect: CGRect, to view: UIView) -> CGRect {
        let x = normalizedRect.origin.x * view.bounds.width
        let y = (1 - normalizedRect.origin.y - normalizedRect.height) * view.bounds.height
        let width = normalizedRect.width * view.bounds.width
        let height = normalizedRect.height * view.bounds.height
        return CGRect(x: x, y: y, width: width, height: height)
    }

    static func convert(normalizedPoint: CGPoint, to view: UIView) -> CGPoint {
        let x = normalizedPoint.x * view.bounds.width
        let y = (1 - normalizedPoint.y) * view.bounds.height
        return CGPoint(x: x, y: y)
    }
}
```

---

## 常用模型集成规范

| 场景 | 推荐方案 | 备注 |
|------|---------|------|
| 对象检测 | YOLOv8n (CoreML 导出) | 优先用 nano/small |
| 图像分类 | MobileNetV3 / EfficientNet | Apple 预训练可直接用 |
| 图文匹配 | CLIP (量化版) | 注意内存，按需加载 |
| 人脸检测 | `VNDetectFaceRectanglesRequest` | 系统 API，无需自带模型 |
| 文字识别 | `VNRecognizeTextRequest` | 支持中英文，iOS 15+ 精度好 |
| 人体姿态 | `VNDetectHumanBodyPoseRequest` | iOS 14+ |
| 手势检测 | `VNDetectHumanHandPoseRequest` | iOS 14+，最多 2 只手 |

### 文字识别示例

```swift
func recognizeText(in image: UIImage) async throws -> [RecognizedText] {
    guard let cgImage = image.cgImage else { throw VisionError.invalidImage }

    let request = VNRecognizeTextRequest()
    request.recognitionLevel = .accurate
    request.recognitionLanguages = ["zh-Hans", "en"]
    request.usesLanguageCorrection = true

    let handler = VNImageRequestHandler(cgImage: cgImage)
    try handler.perform([request])

    return request.results?.compactMap { observation in
        observation.topCandidates(1).first.map { candidate in
            RecognizedText(text: candidate.string, confidence: candidate.confidence, boundingBox: observation.boundingBox)
        }
    } ?? []
}
```

---

## 模型转换备忘

### PyTorch → CoreML

```bash
# 1. 导出 ONNX
python export.py --weights yolov8n.pt --include onnx

# 2. 转换 CoreML
python -c "
import coremltools as ct
model = ct.converters.onnx.convert('yolov8n.onnx',
    minimum_ios_deployment_target='15.0',
    compute_units=ct.ComputeUnit.ALL)
model.save('YOLOv8n.mlpackage')
"
```

### 量化（减小包体积）

```python
import coremltools as ct
from coremltools.optimize.coreml import Palettizer, PalettizerConfig

config = PalettizerConfig(mode="kmeans", nbits=4)
model = ct.models.MLModel("YOLOv8n.mlpackage")
quantized = Palettizer.palettize_weights(model, config)
quantized.save("YOLOv8n_4bit.mlpackage")
```

### 转换注意事项
- `minimum_ios_deployment_target` 必须与项目最低版本一致
- YOLO 模型需要自定义 NMS 后处理（CoreML 不内置）
- 量化后精度可能下降，必须做 A/B 对比测试
- `.mlpackage` 比 `.mlmodel` 更灵活，推荐使用

---

## 已知陷阱

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| 坐标系翻转 | Vision Y 轴向上，UIKit 向下 | 使用 `VisionCoordinateConverter`，禁止手动转换 |
| 推理阻塞主线程 | `handler.perform()` 是同步调用 | 必须在后台队列执行 |
| 内存暴涨 | 大模型未释放 | 退出页面时调用 `unloadAll()` |
| 模型加载失败 | `.mlmodel` 编译缓存问题 | Clean Build Folder，重新编译 |
| 实时检测闪烁 | 单帧误检 | 使用 `DetectionSmoother` 防抖 |
| 热量过高 | Neural Engine 满负荷 | 降低帧率 / 分辨率，增加跳帧 |
| YOLO 输出格式不对 | CoreML 导出后需 NMS | 自行实现 NMS 或使用 ultralytics 导出脚本 |
| 人脸检测回调为空 | 图片太小或无人脸 | 检查输入图片尺寸，至少 299×299 |
