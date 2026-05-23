---
name: multimedia
description: 涉及音频录制、音频播放、视频编辑、Photos Framework、图片压缩、图片缓存、AVAudioPlayer、AVAudioRecorder、AVPlayer、AVAssetExportSession 的任务
---

# 多媒体处理

## 音频

### 音频会话配置

```swift
import AVFoundation

func configureAudioSession(category: AVAudioSession.Category = .playAndRecord) {
    let session = AVAudioSession.sharedInstance()
    do {
        try session.setCategory(category, mode: .default, options: [.defaultToSpeaker, .allowBluetooth])
        try session.setActive(true)
    } catch {
        Logger.error("音频会话配置失败: \(error)")
    }
}
```

### 场景与 Category 对照

| 场景 | Category | Mode | Options |
|------|----------|------|---------|
| 录音 | `.playAndRecord` | `.default` | `.defaultToSpeaker` |
| 播放音乐 | `.playback` | `.default` | - |
| 语音通话 | `.playAndRecord` | `.voiceChat` | `.allowBluetooth` |
| 后台播放 | `.playback` | `.default` | 需 Background Modes → Audio |
| 静音开关跟随 | `.ambient` | `.default` | - |

### 音频录制

```swift
final class AudioRecorder: NSObject {
    private var recorder: AVAudioRecorder?
    private var recordingURL: URL?

    func startRecording(to url: URL) throws {
        configureAudioSession(category: .playAndRecord)
        let settings: [String: Any] = [
            AVFormatIDKey: Int(kAudioFormatMPEG4AAC),
            AVSampleRateKey: 44100.0,
            AVNumberOfChannelsKey: 1,
            AVEncoderAudioQualityKey: AVAudioQuality.high.rawValue
        ]
        recorder = try AVAudioRecorder(url: url, settings: settings)
        recorder?.record()
        recordingURL = url
    }

    func stopRecording() -> URL? {
        recorder?.stop()
        return recordingURL
    }

    var isRecording: Bool { recorder?.isRecording ?? false }

    var currentTime: TimeInterval { recorder?.currentTime ?? 0 }
}
```

### 音频播放

```swift
final class AudioPlayer {
    private var player: AVAudioPlayer?

    func play(url: URL) {
        configureAudioSession(category: .playback)
        do {
            player = try AVAudioPlayer(contentsOf: url)
            player?.play()
        } catch {
            Logger.error("音频播放失败: \(error)")
        }
    }

    func stop() {
        player?.stop()
        player = nil
    }

    var isPlaying: Bool { player?.isPlaying ?? false }

    var duration: TimeInterval { player?.duration ?? 0 }

    var currentTime: TimeInterval { player?.currentTime ?? 0 }
}
```

### 远程音频流播放

```swift
final class StreamingPlayer {
    private var player: AVPlayer?
    private var playerItem: AVPlayerItem?

    func play(url: URL) {
        playerItem = AVPlayerItem(url: url)
        player = AVPlayer(playerItem: playerItem)
        player?.play()
    }

    func pause() {
        player?.pause()
    }

    func seek(to time: CMTime) {
        player?.seek(to: time)
    }
}
```

### 音频权限
- Info.plist：`NSMicrophoneUsageDescription`（具体说明用途）
- 检查权限：`AVAudioApplication.shared.recordPermission`（iOS 17+）或 `AVCaptureDevice.authorizationStatus(for: .audio)`
- 被拒绝后引导去系统设置

---

## 视频编辑

### AVAsset 基础操作

```swift
func trimVideo(sourceURL: URL, startTime: CMTime, endTime: CMTime, outputURL: URL) async throws {
    let asset = AVAsset(url: sourceURL)
    let composition = AVMutableComposition()

    let tracks = try await asset.loadTracks(withMediaType: .video)
    guard let videoTrack = tracks.first else { throw MediaError.noVideoTrack }

    let compositionTrack = composition.addMutableTrack(withMediaType: .video, preferredTrackID: kCMPersistentTrackID_Invalid)
    try compositionTrack?.insertTimeRange(CMTimeRange(start: startTime, end: endTime), of: videoTrack, at: .zero)

    if let audioTracks = try? await asset.loadTracks(withMediaType: .audio), let audioTrack = audioTracks.first {
        let compositionAudio = composition.addMutableTrack(withMediaType: .audio, preferredTrackID: kCMPersistentTrackID_Invalid)
        try compositionAudio?.insertTimeRange(CMTimeRange(start: startTime, end: endTime), of: audioTrack, at: .zero)
    }

    guard let exportSession = AVAssetExportSession(asset: composition, presetName: AVAssetExportPresetHighestQuality) else {
        throw MediaError.exportSessionFailed
    }
    exportSession.outputURL = outputURL
    exportSession.outputFileType = .mp4
    try await exportSession.export()
}
```

### 视频合成（多段拼接）

```swift
func mergeVideos(urls: [URL], outputURL: URL) async throws {
    let composition = AVMutableComposition()
    var currentTime = CMTime.zero

    for url in urls {
        let asset = AVAsset(url: url)
        let duration = try await asset.load(.duration)

        let videoTracks = try await asset.loadTracks(withMediaType: .video)
        if let videoTrack = videoTracks.first, let compositionTrack = composition.addMutableTrack(withMediaType: .video, preferredTrackID: kCMPersistentTrackID_Invalid) {
            try compositionTrack.insertTimeRange(CMTimeRange(start: .zero, duration: duration), of: videoTrack, at: currentTime)
        }

        let audioTracks = try? await asset.loadTracks(withMediaType: .audio)
        if let audioTrack = audioTracks?.first, let compositionAudio = composition.addMutableTrack(withMediaType: .audio, preferredTrackID: kCMPersistentTrackID_Invalid) {
            try compositionAudio.insertTimeRange(CMTimeRange(start: .zero, duration: duration), of: audioTrack, at: currentTime)
        }

        currentTime = CMTimeAdd(currentTime, duration)
    }

    guard let exportSession = AVAssetExportSession(asset: composition, presetName: AVAssetExportPresetHighestQuality) else {
        throw MediaError.exportSessionFailed
    }
    exportSession.outputURL = outputURL
    exportSession.outputFileType = .mp4
    try await exportSession.export()
}
```

### 视频导出预设

| 预设 | 分辨率 | 适用场景 |
|------|--------|---------|
| `AVAssetExportPresetLowQuality` | ~480p | 快速预览 |
| `AVAssetExportPresetMediumQuality` | ~720p | 社交分享 |
| `AVAssetExportPresetHighestQuality` | 原始 | 本地保存 |
| `AVAssetExportPreset1280x720` | 720p | 精确控制 |
| `AVAssetExportPreset1920x1080` | 1080p | 精确控制 |

### 视频缩略图

```swift
func generateThumbnail(url: URL, at time: CMTime = .zero) async throws -> UIImage {
    let asset = AVAsset(url: url)
    let generator = AVAssetImageGenerator(asset: asset)
    generator.appliesPreferredTrackTransform = true
    generator.maximumSize = CGSize(width: 400, height: 400)
    let cgImage = try await generator.image(at: time).image
    return UIImage(cgImage: cgImage)
}
```

---

## Photos Framework

### 权限

```swift
func requestPhotoLibraryPermission() async -> Bool {
    let status = PHPhotoLibrary.authorizationStatus(for: .readWrite)
    switch status {
    case .notDetermined:
        let granted = await PHPhotoLibrary.requestAuthorization(for: .readWrite)
        return granted == .authorized || granted == .limited
    case .authorized, .limited:
        return true
    default:
        return false
    }
}
```

- Info.plist：
  - `NSPhotoLibraryUsageDescription`：读取相册
  - `NSPhotoLibraryAddUsageDescription`：保存到相册（iOS 11+ 自动授权，但仍需声明）

### 保存到相册

```swift
func saveToPhotoLibrary(url: URL) async throws {
    try await PHPhotoLibrary.shared().performChanges {
        PHAssetChangeRequest.creationRequestForAssetFromVideo(atFileURL: url)
    }
}

func saveToPhotoLibrary(image: UIImage) async throws {
    try await PHPhotoLibrary.shared().performChanges {
        PHAssetChangeRequest.creationRequestForAsset(from: image)
    }
}
```

### 读取相册

```swift
func fetchPhotos(limit: Int = 20) -> [PHAsset] {
    let options = PHFetchOptions()
    options.sortDescriptors = [NSSortDescriptor(key: "creationDate", ascending: false)]
    options.fetchLimit = limit
    return PHAsset.fetchAssets(with: .image, options: options).objects()
}

func loadImage(for asset: PHAsset, targetSize: CGSize) async throws -> UIImage {
    let options = PHImageRequestOptions()
    options.deliveryMode = .highQualityFormat
    options.isNetworkAccessAllowed = true
    return try await withCheckedThrowingContinuation { continuation in
        PHImageManager.default().requestImage(for: asset, targetSize: targetSize, contentMode: .aspectFill, options: options) { image, _ in
            if let image {
                continuation.resume(returning: image)
            } else {
                continuation.resume(throwing: MediaError.imageLoadFailed)
            }
        }
    }
}
```

### 规范
- PHAsset 操作必须在主线程发起
- `PHImageManager` 请求图片时设置 `targetSize`，**禁止请求全尺寸图片**（内存爆炸）
- 大量图片使用 `PHCachingImageManager` 预缓存
- `PHFetchResult` 是懒加载的，不要一次性转为数组（数据量大时内存飙升）
- Limited Photo Library（iOS 14+）：用户可能只授权部分照片，需处理 `.limited` 状态

---

## 图片处理

### 图片压缩

```swift
func compressImage(_ image: UIImage, maxLengthKB: Int = 500) -> Data? {
    var compression: CGFloat = 1.0
    var data = image.jpegData(compressionQuality: compression)

    let maxBytes = maxLengthKB * 1024
    while let d = data, d.count > maxBytes, compression > 0.1 {
        compression -= 0.1
        data = image.jpegData(compressionQuality: compression)
    }
    return data
}
```

### 图片缩放

```swift
func resizeImage(_ image: UIImage, to size: CGSize) -> UIImage {
    let renderer = UIGraphicsImageRenderer(size: size)
    return renderer.image { _ in
        image.draw(in: CGRect(origin: .zero, size: size))
    }
}
```

### 图片缓存（Kingfisher）

```swift
// 加载网络图片
imageView.kf.setImage(with: url, placeholder: UIImage(named: "placeholder"))

// 自定义缓存配置
let cache = ImageCache.default
cache.memoryStorage.config.countLimit = 100
cache.memoryStorage.config.expiration = .seconds(300)
cache.diskStorage.config.sizeLimit = 200 * 1024 * 1024

// 清理缓存
cache.clearMemoryCache()
cache.clearDiskCache()
```

---

## 规范

### 通用
- 所有媒体操作必须先检查权限，**禁止静默失败**
- 大文件操作（视频导出、图片处理）在后台队列执行，**禁止阻塞主线程**
- 临时文件用完即删，监听 `UIApplication.didReceiveMemoryWarningNotification` 主动清理
- 导出前检查存储空间

### 音频
- 切换音频场景时必须重新配置 `AVAudioSession`（录音→播放需要切换 Category）
- 后台播放需要 Background Modes → Audio, AirPlay, and Picture in Picture
- 蓝牙耳机场景设置 `.allowBluetooth` option
- 音频中断（来电）监听 `AVAudioSession.interruptionNotification`，中断后暂停，恢复后继续

### 视频
- `AVAssetExportSession` 同一时间只能有一个导出任务
- 导出大视频时监听进度：`exportSession.progress`
- 视频导出目标文件如果已存在会失败，导出前先删除旧文件
- `AVAssetImageGenerator` 生成缩略图时设置 `maximumSize`，避免全尺寸解码

### 图片
- 列表中加载图片必须用异步加载 + 占位图
- 图片缓存大小上限建议 200MB
- HEIF 格式兼容性：iOS 11+ 支持，上传到服务端时转 JPEG
- 图片方向：使用 `UIImage` 的 `imageOrientation` 处理，禁止手动旋转像素

---

## 已知陷阱

- **AVAudioSession 切换 Category 时会中断正在播放的音频**，需在切换后恢复
- **AVAssetExportSession 在 iOS 16+ 废弃了部分预设**，使用前检查 `compatibleFileTypes`
- **PHAsset 的 `creationDate` 可能为 nil**（iCloud 照片未下载时）
- **Kingfisher 在 iOS 15+ 默认使用 URLSession 缓存**，可能与自定义 URLCache 冲突
- **视频导出时 App 进入后台会被系统挂起**，需要 `beginBackgroundTask` 保护
- **`UIImage(contentsOfFile:)` 不缓存**，适合大图一次性使用；`UIImage(named:)` 会缓存，适合小图标
- **录音文件格式 `.caf` 只在 Apple 平台可播放**，跨平台场景用 `.m4a`（AAC）
