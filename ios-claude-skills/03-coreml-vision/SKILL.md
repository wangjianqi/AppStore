---
name: coreml-vision
description: 涉及 CoreML 模型加载、Vision 框架、图像识别、对象检测、YOLO、CLIP、人脸检测的任务
---

# CoreML / Vision 本地推理

## 模型管理
- 所有 `.mlmodel` / `.mlpackage` 文件放在 `Models/` 目录
- 模型加载使用**懒加载**，禁止在 App 启动时同步加载
- 启用 Metal 加速：
  ```swift
  let config = MLModelConfiguration()
  config.computeUnits = .all  // CPU + GPU + Neural Engine
  ```
- 模型版本通过文件名区分（如 `YOLOv8n_v2.mlpackage`），禁止覆盖旧版本

## Vision Pipeline — 静态图像
```swift
let request = VNRecognizeObjectsRequest(completionHandler: handleResults)
let handler = VNImageRequestHandler(cgImage: cgImage, options: [:])
try handler.perform([request])
```

## Vision Pipeline — 实时视频流
- 使用 `AVCaptureVideoDataOutput` + `VNSequenceRequestHandler`
- 实现 `AVCaptureVideoDataOutputSampleBufferDelegate`：
  ```swift
  func captureOutput(_ output:, didOutput sampleBuffer:, from connection:)
  ```
- 推理必须在**后台队列**执行，禁止在 `captureOutput` 回调里直接更新 UI

## 性能规范
- **帧率控制：** 通过 `videoSettings` 限制输入分辨率（推理推荐 `640×480` 或 `416×416`）
- **跳帧策略：** 使用 `alwaysDiscardsLateVideoFrames = true`，避免帧队列堆积
- **结果防抖：** 高频推理结果需做时间窗口平滑（连续 N 帧同类才输出）
- **UI 更新：** 结果回调必须 `DispatchQueue.main.async` 后再操作视图

## 坐标系转换（重要）
Vision 框架坐标系：**左下角原点，Y 轴向上，值域 [0,1]**  
UIKit 坐标系：**左上角原点，Y 轴向下，像素值**

**禁止在业务代码里手动做坐标转换**，使用项目内 `VisionCoordinateConverter.swift`：
```swift
// 标准转换方法
func convert(normalizedRect: CGRect, to view: UIView) -> CGRect
func convert(normalizedPoint: CGPoint, to view: UIView) -> CGPoint
```

## 常用模型集成规范
| 场景 | 推荐方案 | 备注 |
|------|---------|------|
| 对象检测 | YOLOv8n (CoreML 导出) | 优先用 nano/small |
| 图像分类 | MobileNetV3 / EfficientNet | Apple 预训练可直接用 |
| 图文匹配 | CLIP (量化版) | 注意内存，按需加载 |
| 人脸检测 | `VNDetectFaceRectanglesRequest` | 系统 API，无需自带模型 |
| 文字识别 | `VNRecognizeTextRequest` | 支持中英文，iOS 15+ 精度好 |
| 人体姿态 | `VNDetectHumanBodyPoseRequest` | iOS 14+ |

## 模型转换备忘
- PyTorch → CoreML：使用 `coremltools` + `ct.convert()`
- ONNX → CoreML：先 ONNX，再 `coremltools`
- 量化：`ct.optimize.coreml.palettize_weights()`（减小包体积）
