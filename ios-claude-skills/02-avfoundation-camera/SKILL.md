---
name: avfoundation-camera
description: 涉及相机、录像、预览、AVCaptureSession、多摄像头、前后摄切换、权限处理、AVFoundation 的任务
---

# AVFoundation 相机模块

## Session 架构

### Session 类型选择

| 场景 | Session 类型 | 最低版本 |
|------|-------------|---------|
| 单摄（前或后） | `AVCaptureSession` | iOS 4+ |
| 双摄（前+后同时） | `AVCaptureMultiCamSession` | iOS 13+ |
| 深度采集 | `AVCaptureSession` + depth data output | iOS 11+ |

- Session 所有操作必须在**专用串行队列**执行
- **禁止在主线程配置 session**，预览层更新除外

```swift
private let sessionQueue = DispatchQueue(label: "com.app.sessionQueue")
```

---

## 完整权限处理流程

```swift
final class CameraPermissionManager {
    static func checkAuthorization(completion: @escaping (Bool) -> Void) {
        let status = AVCaptureDevice.authorizationStatus(for: .video)
        switch status {
        case .authorized:
            completion(true)
        case .notDetermined:
            requestAccess(completion: completion)
        case .denied, .restricted:
            showPermissionDeniedAlert()
            completion(false)
        @unknown default:
            completion(false)
        }
    }

    private static func requestAccess(completion: @escaping (Bool) -> Void) {
        AVCaptureDevice.requestAccess(for: .video) { granted in
            DispatchQueue.main.async {
                if granted {
                    completion(true)
                } else {
                    showPermissionDeniedAlert()
                    completion(false)
                }
            }
        }
    }

    private static func showPermissionDeniedAlert() {
        guard let settingsURL = URL(string: UIApplication.openSettingsURLString) else { return }
        let alert = UIAlertController(
            title: "需要相机权限".localized,
            message: "请在设置中开启相机权限，以使用录制功能".localized,
            preferredStyle: .alert
        )
        alert.addAction(UIAlertAction(title: "去设置".localized, style: .default) { _ in
            UIApplication.shared.open(settingsURL)
        })
        alert.addAction(UIAlertAction(title: "取消".localized, style: .cancel))
        (UIApplication.shared.connectedScenes.first?.delegate as? SceneDelegate)?
            .window?.rootViewController?.present(alert, animated: true)
    }
}
```

### Info.plist 必填
- `NSCameraUsageDescription`：具体说明用途（不能是"需要相机权限"）
- `NSMicrophoneUsageDescription`：录像功能时必填

---

## Session 完整配置

```swift
final class CameraSessionManager: NSObject {
    private let session = AVCaptureSession()
    private let sessionQueue = DispatchQueue(label: "com.app.sessionQueue")
    private var videoDeviceInput: AVCaptureDeviceInput?
    private var photoOutput: AVCapturePhotoOutput?
    private var movieOutput: AVCaptureMovieFileOutput?

    var onPreviewLayerReady: ((AVCaptureVideoPreviewLayer) -> Void)?

    func setupSession() {
        sessionQueue.async { [weak self] in
            guard let self = self else { return }
            self.session.beginConfiguration()
            self.session.sessionPreset = .hd1920x1080

            do {
                try self.setupVideoInput()
                try self.setupAudioInput()
                self.setupPhotoOutput()
                self.setupMovieOutput()
            } catch {
                print("Session 配置失败: \(error)")
                self.session.commitConfiguration()
                return
            }

            self.session.commitConfiguration()
            self.session.startRunning()

            DispatchQueue.main.async {
                let previewLayer = AVCaptureVideoPreviewLayer(session: self.session)
                previewLayer.videoGravity = .resizeAspectFill
                self.onPreviewLayerReady?(previewLayer)
            }
        }
    }

    func stopSession() {
        sessionQueue.async { [weak self] in
            self?.session.stopRunning()
        }
    }

    private func setupVideoInput() throws {
        guard let device = AVCaptureDevice.default(.builtInWideAngleCamera, for: .video, position: .back) else {
            throw CameraError.deviceUnavailable
        }
        let input = try AVCaptureDeviceInput(device: device)
        if session.canAddInput(input) {
            session.addInput(input)
            videoDeviceInput = input
        }
    }

    private func setupAudioInput() throws {
        guard let device = AVCaptureDevice.default(for: .audio) else {
            throw CameraError.deviceUnavailable
        }
        let input = try AVCaptureDeviceInput(device: device)
        if session.canAddInput(input) {
            session.addInput(input)
        }
    }

    private func setupPhotoOutput() {
        let output = AVCapturePhotoOutput()
        if session.canAddOutput(output) {
            session.addOutput(output)
            photoOutput = output
        }
    }

    private func setupMovieOutput() {
        let output = AVCaptureMovieFileOutput()
        if session.canAddOutput(output) {
            session.addOutput(output)
            movieOutput = output
        }
    }
}
```

---

## 前后摄切换

```swift
func switchCamera() {
    sessionQueue.async { [weak self] in
        guard let self = self,
              let currentInput = self.videoDeviceInput else { return }

        self.session.beginConfiguration()

        self.session.removeInput(currentInput)

        let newPosition: AVCaptureDevice.Position = currentInput.device.position == .back ? .front : .back
        guard let device = AVCaptureDevice.default(.builtInWideAngleCamera, for: .video, position: newPosition),
              let newInput = try? AVCaptureDeviceInput(device: device) else {
            self.session.addInput(currentInput)
            self.session.commitConfiguration()
            return
        }

        if self.session.canAddInput(newInput) {
            self.session.addInput(newInput)
            self.videoDeviceInput = newInput
        } else {
            self.session.addInput(currentInput)
        }

        self.session.commitConfiguration()
    }
}
```

- 切换前必须 `beginConfiguration()`，切换后 `commitConfiguration()`
- 切换失败时恢复原 Input

---

## 输出配置

### 视频编码

| 编码 | 优势 | 劣势 | 适用场景 |
|------|------|------|---------|
| H.264 | 兼容性最好 | 文件较大 | 默认选择 |
| HEVC (H.265) | 文件小约 40% | iOS 11+，部分设备不支持 | 存储优先 |

### 音频配置
- 格式：AAC，44100 Hz，双声道
- 后台录制需配置 `AVAudioSession.Category.playAndRecord`

### 分辨率与帧率

```swift
// 设置分辨率
session.sessionPreset = .hd1920x1080  // 1080p

// 设置帧率
if let device = videoDeviceInput?.device {
    try device.lockForConfiguration()
    let targetFrameRate = CMTime(value: 1, timescale: 30)
    let format = device.formats.first { format in
        format.videoSupportedFrameRateRanges.contains { $0.minFrameRate <= 30 && $0.maxFrameRate >= 30 }
    }
    if let format = format {
        device.activeFormat = format
        device.activeVideoMinFrameDuration = targetFrameRate
        device.activeVideoMaxFrameDuration = targetFrameRate
    }
    device.unlockForConfiguration()
}
```

---

## 预览层

```swift
final class CameraPreviewView: UIView {
    var previewLayer: AVCaptureVideoPreviewLayer? {
        layer as? AVCaptureVideoPreviewLayer
    }

    override class var layerClass: AnyClass {
        AVCaptureVideoPreviewLayer.self
    }

    func setSession(_ session: AVCaptureSession) {
        previewLayer?.session = session
        previewLayer?.videoGravity = .resizeAspectFill
    }
}
```

- 预览层绑定到专用 `UIView`（`CameraPreviewView`），不直接加到 VC.view
- 旋转处理：监听 `UIDevice.orientationDidChangeNotification`，更新 `connection.videoOrientation`

```swift
@objc private func handleDeviceRotation() {
    guard let connection = previewLayer?.connection,
          connection.isVideoOrientationSupported else { return }
    connection.videoOrientation = UIDevice.current.orientation.videoOrientation
}
```

---

## 录制控制

```swift
func startRecording(to url: URL) {
    sessionQueue.async { [weak self] in
        guard let self = self, let output = self.movieOutput else { return }
        output.startRecording(to: url, recordingDelegate: self)
    }
}

func stopRecording() {
    sessionQueue.async { [weak self] in
        self?.movieOutput?.stopRecording()
    }
}
```

---

## 已知陷阱 & 解决方案

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| iPhone 15 Pro 后摄默认 48MP | 默认分辨率过高 | 录像前显式设置 `sessionPreset = .hd1920x1080` |
| 后台录制中断 | App 挂起后 session 停止 | 配置 Background Modes → Audio，设置 `AVAudioSession.Category.playAndRecord` |
| 前后摄切换闪烁 | 未用 `beginConfiguration` | 切换前 `beginConfiguration()`，切换后 `commitConfiguration()` |
| 内存警告 | Buffer 堆积 | 监听 `didReceiveMemoryWarning`，及时释放 buffer |
| 双摄功耗过高 | 两个摄像头同时工作 | 降低分辨率/帧率，避免热量过高 |
| 录制文件损坏 | App 被杀但未 stopRecording | 在 `applicationWillTerminate` 中调用 `stopRecording()` |
| 预览画面拉伸 | `videoGravity` 设置不当 | 使用 `.resizeAspectFill`，禁止 `.resize` |
| 前摄镜像 | 前摄默认不镜像预览 | 预览层设置 `previewLayer?.isMirrored = true`（仅预览，录制不镜像） |

---

## 文件保存

### 临时文件
- 临时文件写入 `FileManager.default.temporaryDirectory`

### 保存到相册

```swift
func saveToPhotoLibrary(url: URL, completion: @escaping (Bool) -> Void) {
    PHPhotoLibrary.requestAuthorization(for: .addOnly) { status in
        guard status == .authorized || status == .limited else {
            completion(false)
            return
        }
        PHPhotoLibrary.shared().performChanges({
            PHAssetChangeRequest.creationRequestForAssetFromVideo(atFileURL: url)
        }) { success, _ in
            DispatchQueue.main.async { completion(success) }
        }
    }
}
```

### 存储空间检查

```swift
func availableStorageMB() -> Int? {
    let url = FileManager.default.temporaryDirectory
    let values = try? url.resourceValues(forKeys: [.volumeAvailableCapacityForImportantUsageKey])
    return values?.volumeAvailableCapacityForImportantUsage.map { Int($0 / 1024 / 1024) }
}
```

- 录制前检查存储空间，低于 500MB 提示用户
