---
title: "OpenClaw 屏幕录制技术实现分析"
description: "深入分析开源 AI 助手 OpenClaw 如何在 macOS、iOS、Android 上实现屏幕录制功能，以及 Windows 平台的实现方案"
date: 2026-03-04
draft: true
categories: ["开源项目分析", "跨平台开发"]
tags: ["OpenClaw", "屏幕录制", "ScreenCaptureKit", "ReplayKit", "MediaProjection", "macOS", "iOS", "Android"]
---


# OpenClaw 屏幕录制技术实现分析

> 本文深入分析了开源 AI 助手 OpenClaw 如何在不同操作系统上实现屏幕录制功能。

## 概述

OpenClaw 是一个跨平台的个人 AI 助手项目，支持在 macOS、iOS 和 Android 设备上运行。其屏幕录制功能采用了**平台原生 API** 的实现策略，每个平台使用各自系统提供的最佳屏幕捕获方案。

**重要发现**：截至分析时，OpenClaw **没有实现 Windows 原生应用** 的屏幕录制功能。项目仅提供了 macOS、iOS 和 Android 三个平台的原生实现。

## 架构设计

### 整体架构

```
┌─────────────────────────────────────────────────────────────────┐
│                        Gateway / CLI                            │
│  (TypeScript - 跨平台命令行接口)                                  │
│                                                                 │
│  nodes-cli/register.screen.ts                                   │
│  └── 发送 screen.record 命令到节点                                │
│      参数：{ screenIndex, durationMs, fps, format, includeAudio }│
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ WebSocket / IPC
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Node Runtime (Native App)                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │   macOS     │  │    iOS      │  │   Android   │             │
│  │             │  │             │  │             │             │
│  │ ScreenCap-  │  │  ReplayKit  │  │ MediaPro-   │             │
│  │ tureKit     │  │             │  │ jection     │             │
│  │             │  │             │  │             │             │
│  │ SCStream    │  │ RPScreen    │  │ Media-      │             │
│  │             │  │ Recorder    │  │ Recorder    │             │
│  └─────────────┘  └─────────────┘  └─────────────┘             │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ 返回 base64 编码的 MP4
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        结果处理                                  │
│  nodes-screen.ts                                               │
│  └── 解析 payload { format, base64, durationMs, fps, ... }     │
│      └── 写入文件：openclaw-screen-record-*.mp4                 │
└─────────────────────────────────────────────────────────────────┘
```

### 命令协议

屏幕录制功能通过统一的命令协议暴露：

```typescript
// 命令名称
"screen.record"

// 参数结构
interface ScreenRecordParams {
  screenIndex?: number;    // 屏幕索引（0 = 主屏幕）
  durationMs?: number;     // 录制时长（毫秒），默认 10000ms
  fps?: number;            // 帧率，默认 10fps，最大 60fps
  format?: string;         // 输出格式，固定为 "mp4"
  includeAudio?: boolean;  // 是否包含音频
}

// 返回结构
interface ScreenRecordPayload {
  format: string;          // "mp4"
  base64: string;          // base64 编码的视频数据
  durationMs?: number;     // 实际录制时长
  fps?: number;            // 实际帧率
  screenIndex?: number;    // 录制的屏幕索引
  hasAudio?: boolean;      // 是否包含音频轨道
}
```

---

## macOS 实现

### 技术选型：ScreenCaptureKit

macOS 版本使用 **ScreenCaptureKit** 框架，这是 Apple 在 macOS 12+ 引入的现代屏幕捕获 API。

**核心文件**：
- `apps/macos/Sources/OpenClaw/ScreenRecordService.swift` (256 行)
- `apps/macos/Sources/OpenClaw/NodeMode/MacNodeScreenCommands.swift` (13 行)
- `apps/macos/Sources/OpenClaw/PermissionManager.swift` (482 行)

### 关键 API

```swift
import ScreenCaptureKit

// 1. 获取可共享的屏幕内容
let content = try await SCShareableContent.current
let displays = content.displays.sorted { $0.displayID < $1.displayID }

// 2. 创建内容过滤器（选择要录制的屏幕）
let filter = SCContentFilter(display: display, excludingWindows: [])

// 3. 配置录制参数
let config = SCStreamConfiguration()
config.width = display.width
config.height = display.height
config.queueDepth = 8
config.showsCursor = true
config.minimumFrameInterval = CMTime(value: 1, timescale: CMTimeScale(fps))
config.capturesAudio = true  // 可选：录制系统音频

// 4. 创建 SCStream 并添加输出
let stream = SCStream(filter: filter, configuration: config, delegate: recorder)
try stream.addStreamOutput(recorder, type: .screen, sampleHandlerQueue: queue)
try stream.addStreamOutput(recorder, type: .audio, sampleHandlerQueue: queue)

// 5. 开始/停止捕获
try await stream.startCapture()
try await Task.sleep(nanoseconds: durationMs * 1_000_000)
try await stream.stopCapture()
```

### 视频编码

使用 **AVAssetWriter** 进行 H.264 编码：

```swift
let writer = try AVAssetWriter(outputURL: outputURL, fileType: .mp4)

let videoSettings: [String: Any] = [
    AVVideoCodecKey: AVVideoCodecType.h264,
    AVVideoWidthKey: width,
    AVVideoHeightKey: height,
]
let videoInput = AVAssetWriterInput(mediaType: .video, outputSettings: videoSettings)
videoInput.expectsMediaDataInRealTime = true
writer.add(videoInput)

// 音频配置（可选）
if includeAudio {
    let audioSettings: [String: Any] = [
        AVFormatIDKey: kAudioFormatMPEG4AAC,
        AVNumberOfChannelsKey: 1,
        AVSampleRateKey: 44100,
        AVEncoderBitRateKey: 96000,
    ]
    let audioInput = AVAssetWriterInput(mediaType: .audio, outputSettings: audioSettings)
    writer.add(audioInput)
}
```

### 权限处理

macOS 10.15+ 需要用户授予**屏幕录制权限**：

```swift
// 检查权限
func isAuthorized() -> Bool {
    if #available(macOS 10.15, *) {
        return CGPreflightScreenCaptureAccess()
    }
    return true
}

// 请求权限
@MainActor
func requestAuthorization() async {
    if #available(macOS 10.15, *) {
        _ = CGRequestScreenCaptureAccess()
    }
}
```

权限状态通过 `PermissionManager` 统一管理，支持交互式请求和系统设置跳转。

### 参数限制

```swift
// CaptureRateLimits.swift
enum CaptureRateLimits {
    static func clampDurationMs(
        _ ms: Int?,
        defaultMs: Int = 10_000,
        minMs: Int = 250,
        maxMs: Int = 60_000
    ) -> Int
    
    static func clampFps(
        _ fps: Double?,
        defaultFps: Double = 10,
        minFps: Double = 1,
        maxFps: Double = 60
    ) -> Double
}
```

- **时长范围**：250ms ~ 60,000ms (1 分钟)
- **帧率范围**：1 ~ 60 fps
- **音频**：可选，默认关闭

---

## iOS 实现

### 技术选型：ReplayKit

iOS 版本使用 **ReplayKit** 框架，这是 Apple 提供的 iOS 屏幕录制 API。

**核心文件**：
- `apps/ios/Sources/Screen/ScreenRecordService.swift` (351 行)

### 关键 API

```swift
import ReplayKit

// 1. 获取单例录制器
let recorder = RPScreenRecorder.shared()
recorder.isMicrophoneEnabled = includeAudio

// 2. 开始捕获（需要 MainActor）
@MainActor
func startCapture(
    includeAudio: Bool,
    handler: @escaping (CMSampleBuffer, RPSampleBufferType, Error?) -> Void,
    completion: @escaping (Error?) -> Void
) {
    recorder.startCapture(handler: handler, completionHandler: completion)
}

// 3. 停止捕获
@MainActor
func stopCapture(completion: @escaping (Error?) -> Void) {
    recorder.stopCapture { error in completion(error) }
}
```

### 样本处理

ReplayKit 通过回调提供视频和音频样本：

```swift
// 处理视频样本
private func handleVideoSample(
    _ sample: CMSampleBuffer,
    state: CaptureState,
    config: RecordConfig
) {
    // 帧率控制：跳过间隔过短的帧
    let pts = CMSampleBufferGetPresentationTimeStamp(sample)
    let shouldSkip = state.withLock { state in
        if let lastVideoTime = state.lastVideoTime {
            let delta = CMTimeSubtract(pts, lastVideoTime)
            return delta.seconds < (1.0 / config.fpsValue)
        }
        return false
    }
    if shouldSkip { return }
    
    // 写入视频
    if state.withLock({ $0.writer == nil }) {
        prepareWriter(sample: sample, state: state, config: config, pts: pts)
    }
    
    let vInput = state.withLock { $0.videoInput }
    guard let vInput, vInput.isReadyForMoreMediaData else { return }
    vInput.append(sample)
}

// 处理音频样本（应用音频或麦克风）
private func handleAudioSample(
    _ sample: CMSampleBuffer,
    state: CaptureState,
    includeAudio: Bool
) {
    guard includeAudio, let aInput = state.audioInput else { return }
    aInput.append(sample)
}
```

### 限制

- **单屏幕**：iOS 设备只有一个屏幕，`screenIndex` 必须为 0
- **帧率限制**：最大 30 fps（相比 macOS 的 60 fps）
- **MainActor 要求**：ReplayKit API 必须在主线程调用

---

## Android 实现

### 技术选型：MediaProjection + MediaRecorder

Android 版本使用 **MediaProjection API** 获取屏幕内容，配合 **MediaRecorder** 进行编码。

**核心文件**：
- `apps/android/app/src/main/java/ai/openclaw/android/node/ScreenRecordManager.kt` (165 行)
- `apps/android/app/src/main/java/ai/openclaw/android/node/ScreenHandler.kt` (25 行)

### 关键 API

```kotlin
// 1. 请求屏幕捕获权限（需要用户确认）
val capture = requester.requestCapture()
    ?: throw IllegalStateException("SCREEN_PERMISSION_REQUIRED")

// 2. 获取 MediaProjection
val mgr = context.getSystemService(Context.MEDIA_PROJECTION_SERVICE) 
    as MediaProjectionManager
val projection = mgr.getMediaProjection(capture.resultCode, capture.data)

// 3. 获取屏幕参数
val metrics = context.resources.displayMetrics
val width = metrics.widthPixels
val height = metrics.heightPixels
val densityDpi = metrics.densityDpi

// 4. 创建 MediaRecorder
val recorder = MediaRecorder(context)
recorder.setVideoSource(MediaRecorder.VideoSource.SURFACE)
recorder.setOutputFormat(MediaRecorder.OutputFormat.MPEG_4)
recorder.setVideoEncoder(MediaRecorder.VideoEncoder.H264)
recorder.setVideoSize(width, height)
recorder.setVideoFrameRate(fpsInt)
recorder.setVideoEncodingBitRate(estimateBitrate(width, height, fpsInt))
recorder.setOutputFile(file.absolutePath)

// 5. 创建 VirtualDisplay 输出到 Recorder Surface
val surface = recorder.surface
val virtualDisplay = projection.createVirtualDisplay(
    "openclaw-screen",
    width,
    height,
    densityDpi,
    DisplayManager.VIRTUAL_DISPLAY_FLAG_AUTO_MIRROR,
    surface,
    null,
    null
)

// 6. 开始录制
recorder.start()
delay(durationMs.toLong())

// 7. 清理资源
recorder.stop()
recorder.reset()
recorder.release()
virtualDisplay.release()
projection.stop()
```

### 权限处理

Android 需要两个权限：

1. **屏幕捕获权限**：通过 `MediaProjectionManager` 请求，需要用户确认弹窗
2. **麦克风权限**：如果录制音频，需要 `RECORD_AUDIO` 权限

```kotlin
private suspend fun ensureMicPermission() {
    val granted = ContextCompat.checkSelfPermission(
        context,
        Manifest.permission.RECORD_AUDIO
    ) == PackageManager.PERMISSION_GRANTED
    
    if (granted) return
    
    val results = requester.requestIfMissing(listOf(Manifest.permission.RECORD_AUDIO))
    if (results[Manifest.permission.RECORD_AUDIO] != true) {
        throw IllegalStateException("MIC_PERMISSION_REQUIRED")
    }
}
```

### 比特率估算

```kotlin
private fun estimateBitrate(width: Int, height: Int, fps: Int): Int {
    val pixels = width.toLong() * height.toLong()
    val raw = (pixels * fps.toLong() * 2L).toInt()
    return raw.coerceIn(1_000_000, 12_000_000)  // 1~12 Mbps
}
```

### 限制

- **单屏幕**：`screenIndex` 必须为 0
- **时长范围**：250ms ~ 60,000ms
- **帧率范围**：1 ~ 60 fps
- **格式**：仅支持 MP4

---

## Windows 支持现状

### 当前状态：**未实现**

截至分析时，OpenClaw 项目**没有 Windows 原生应用**，因此不存在 Windows 屏幕录制实现。

代码库中仅有少量 Windows 相关的工具文件：
- `src/security/windows-acl.ts` - Windows ACL 安全相关
- `src/plugin-sdk/windows-spawn.ts` - Windows 进程生成
- `src/cli/windows-argv.ts` - Windows 命令行参数处理

这些文件与屏幕捕获无关。

### 推荐实现方案

如果需要在 Windows 上实现屏幕录制，建议使用以下 API：

#### 方案 1：Desktop Duplication API (DXGI)

适用于 Windows 8+ 的高性能屏幕捕获：

```cpp
// 伪代码示例
IDXGIDevice* dxgiDevice;
IDXGIOutputDuplication* duplication;
DXGI_OUTDUPL_FRAME_INFO frameInfo;
IDXGIResource* desktopResource;

// 获取桌面资源
duplication->AcquireNextFrame(timeout, &frameInfo, &desktopResource);

// 映射到 CPU 可读内存
ID3D11Texture2D* texture;
desktopResource->QueryInterface(__uuidof(ID3D11Texture2D), (void**)&texture);
```

#### 方案 2：Windows.Graphics.Capture API

适用于 Windows 10 1803+ 的现代 API：

```cpp
// 伪代码示例
auto interopFactory = get_activation_factory<GraphicsCaptureItem, IGraphicsCaptureItemInterop>();
GraphicsCaptureItem item;
interopFactory->CreateForWindow(hwnd, guid_of<GraphicsCaptureItem>(), (void**)&item);

auto framePool = Direct3D11CaptureFramePool::CreateFreeThreaded(
    device, 
    pixelFormat, 
    numberOfBuffers, 
    size
);

auto session = framePool.CreateCaptureSession(item);
session.StartCapture();
```

#### 权限要求

Windows 10/11 不需要特殊权限即可捕获屏幕，但：
- 无法捕获 UWP 应用窗口（除非是桌面桥应用）
- 受 DRM 保护的内容会被黑屏处理
- 需要管理员权限才能捕获某些系统窗口

---

## 平台对比

| 特性 | macOS | iOS | Android | Windows |
|------|-------|-----|---------|---------|
| **API** | ScreenCaptureKit | ReplayKit | MediaProjection | 未实现 |
| **最低版本** | macOS 12+ | iOS 11+ | Android 5.0+ | - |
| **多屏幕** | ✅ 支持 | ❌ 单屏 | ❌ 单屏 | - |
| **系统音频** | ✅ 支持 | ❌ 仅麦克风 | ❌ 仅麦克风 | - |
| **最大帧率** | 60 fps | 30 fps | 60 fps | - |
| **权限要求** | 屏幕录制权限 | 无需额外权限 | 用户确认弹窗 | - |
| **编码格式** | H.264 (MP4) | H.264 (MP4) | H.264 (MP4) | - |

---

## 关键设计模式

### 1. 统一命令接口

所有平台通过统一的 `screen.record` 命令暴露功能，参数和返回值结构完全一致。

### 2. 平台适配层

每个平台有独立的实现文件，通过 IPC/Gateway 与主程序通信：

```
TypeScript CLI (跨平台)
       │
       │ "screen.record"
       ▼
┌──────────────────────┐
│  Native App (Swift)  │  macOS
├──────────────────────┤
│  Native App (Swift)  │  iOS
├──────────────────────┤
│  Native App (Kotlin) │  Android
└──────────────────────┘
```

### 3. 资源管理

所有实现都采用了严格的资源管理模式：

```swift
// macOS/iOS (Swift)
do {
    try await stream.startCapture()
    started = true
    try await Task.sleep(nanoseconds: duration)
    try await stream.stopCapture()
} catch {
    if started { try? await stream.stopCapture() }
    throw error
}
```

```kotlin
// Android (Kotlin)
try {
    recorder.start()
    delay(durationMs)
} finally {
    recorder.stop()
    recorder.reset()
    recorder.release()
    virtualDisplay?.release()
    projection.stop()
}
```

### 4. 参数验证

所有平台在入口处验证参数范围，使用统一的限制值：

```typescript
// CaptureRateLimits.swift / 对应 Kotlin 实现
durationMs: 250 ~ 60,000
fps: 1 ~ 60
screenIndex: 0 (或多屏幕索引)
```

---

## 总结

OpenClaw 的屏幕录制实现展示了**原生跨平台开发**的最佳实践：

1. **使用平台最佳 API**：每个平台选择官方推荐的屏幕捕获方案
2. **统一接口设计**：跨平台命令协议保持一致
3. **严格的资源管理**：确保捕获会话正确清理
4. **完善的权限处理**：引导用户授予必要权限

对于 Windows 支持，建议未来使用 **Desktop Duplication API** 或 **Windows.Graphics.Capture API** 实现，保持与其他平台一致的命令接口和参数结构。

---

## 参考资料

- OpenClaw GitHub: https://github.com/openclaw/openclaw
- ScreenCaptureKit Documentation: https://developer.apple.com/documentation/screencapturekit
- ReplayKit Documentation: https://developer.apple.com/documentation/replaykit
- MediaProjection API: https://developer.android.com/reference/android/media/projection/MediaProjectionManager
