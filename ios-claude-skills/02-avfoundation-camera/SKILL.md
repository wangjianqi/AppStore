---
name: avfoundation-camera
description: 涉及相机、录像、预览、AVCaptureSession、多摄像头、AVFoundation 的任务
---

# AVFoundation 相机模块

## Session 架构
- **单摄：** `AVCaptureSession`
- **双摄（前+后同时）：** `AVCaptureMultiCamSession`（iOS 13+，需检查 `isMultiCamSupported`）
- Session 所有操作必须在**专用串行队列**执行：
  ```swift
  private let sessionQueue = DispatchQueue(label: "com.app.sessionQueue")
  ```
- **禁止在主线程配置 session**，预览层更新除外

## 权限处理顺序
1. 检查 `AVCaptureDevice.authorizationStatus`
2. 未授权时调用 `requestAccess(for:)`，在回调里配置 session
3. 被拒绝时展示自定义引导页（跳转设置），禁止静默失败
4. `Info.plist` 必须包含：
   - `NSCameraUsageDescription`：具体说明用途（不能是"需要相机权限"）
   - `NSMicrophoneUsageDescription`：录像功能时必填

## 输出配置
- **视频编码：** H.264（兼容性优先）或 HEVC（存储优先，iOS 11+）
- **音频：** AAC，44100 Hz，双声道
- **分辨率：** 默认 1080p，4K 可选，通过 `sessionPreset` 配置
- **帧率：** 通过 `AVCaptureDevice.Format` 的 `videoSupportedFrameRateRanges` 配置
- **多路录制：** 每路使用独立 `AVAssetWriter` + `AVAssetWriterInput`

## 预览层
- 使用 `AVCaptureVideoPreviewLayer`，`videoGravity = .resizeAspectFill`
- 预览层绑定到专用 `UIView`（`CameraPreviewView`），不直接加到 VC.view
- 旋转处理：监听 `UIDevice.orientationDidChangeNotification`，更新 `connection.videoOrientation`

## 已知陷阱 & 解决方案
- **iPhone 15 Pro 后摄默认 48MP**：录像前显式设置 `sessionPreset = .hd1920x1080`
- **后台录制**：需要 Background Modes → Audio，并配置 `AVAudioSession.Category.playAndRecord`
- **热切换前后摄（双摄 session）**：切换前必须 `beginConfiguration()`，切换后 `commitConfiguration()`
- **内存警告**：监听 `didReceiveMemoryWarning`，及时释放 buffer
- **AVCaptureMultiCamSession 功耗**：双摄开启时降低分辨率/帧率，避免热量过高

## 文件保存
- 临时文件写入 `FileManager.default.temporaryDirectory`
- 保存相册使用 `PHPhotoLibrary.requestAuthorization` + `PHAssetChangeRequest`
- 导出前检查存储空间：`URLResourceKey.volumeAvailableCapacityForImportantUsageKey`
