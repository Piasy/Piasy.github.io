---
layout: post
title: WebRTC Native 源码导读（七）：iOS 相机采集实现分析
tags:
    - 实时多媒体
    - WebRTC
---

从上一篇开始，我们这个系列就进入了 iOS 的世界，接下来我打算先熟悉一下 iOS 相机相关的内容，包括采集、预览、编码等，本篇重点是采集。

WebRTC-iOS 的相机采集主要涉及到以下几个类：AVCaptureSession, RTCCameraVideoCapturer, RTCVideoFrame。

AVCaptureSession 是 iOS 和 macOS 系统提供的采集管理类，位于 `AVFoundation.framework` 中，在 RTCCameraVideoCapturer 中完成了对 AVCaptureSession 的使用，RTCVideoFrame 则是对视频数据的封装。

_本文的分析基于 WebRTC 的 #23295 提交_。

## AVCaptureSession

我们先来了解一下 AVCaptureSession 的基本使用。

一个 session 需要有 input 和 output，这样数据才能在其中流动（处理），下面这个 session 包含了音视频输入，预览、图片、视频输出：

![](https://imgs.piasy.com/2018-04-26-capture_session_example.png)

AVCaptureSession 的使用主要分为以下几步：

+ 创建 session；
+ 配置 session：添加 input 和 output device；
+ 启停 session；

### 创建 session

创建 session 很简单，就是构造一个对象即可：

~~~ objective-c
session = [[AVCaptureSession alloc] init];
~~~

### 配置 session

由于配置 session 是多步操作，为了保证原子性，AVCaptureSession 提供了事务机制，即先 `beginConfiguration`，再添加 device，最后 `commitConfiguration`：

~~~ objective-c
// 开始配置
[session beginConfiguration];

// 设置采集参数 preset
session.sessionPreset = AVCaptureSessionPresetHigh;

// 选择后置广角相机 AVCaptureDeviceTypeBuiltInWideAngleCamera
// 新款 iPhone 可以选择双摄相机 AVCaptureDeviceTypeBuiltInDualCamera
AVCaptureDevice* videoDevice = [AVCaptureDevice
    defaultDeviceWithDeviceType:AVCaptureDeviceTypeBuiltInWideAngleCamera
                        mediaType:AVMediaTypeVideo
                        position:AVCaptureDevicePositionBack];

// 创建 video input device
AVCaptureDeviceInput* videoDeviceInput =
    [AVCaptureDeviceInput deviceInputWithDevice:videoDevice error:&error];
if ([session canAddInput:videoDeviceInput]) {
    // 添加 video input device
    [session addInput:videoDeviceInput];
}

// 选择音频设备
AVCaptureDevice* audioDevice =
    [AVCaptureDevice defaultDeviceWithMediaType:AVMediaTypeAudio];

// 创建 audio input device
AVCaptureDeviceInput* audioDeviceInput =
    [AVCaptureDeviceInput deviceInputWithDevice:audioDevice error:&error];

if ([session canAddInput:audioDeviceInput]) {
    // 添加 audio input device
    [session addInput:audioDeviceInput];
}

// 创建视频录制
AVCaptureMovieFileOutput* movieFileOutput =
    [[AVCaptureMovieFileOutput alloc] init];

if ([session canAddOutput:movieFileOutput]) {
    // 添加视频录制
    [session addOutput:movieFileOutput];
    AVCaptureConnection* connection =
        [movieFileOutput connectionWithMediaType:AVMediaTypeVideo];
    if (connection.isVideoStabilizationSupported) {
        connection.preferredVideoStabilizationMode =
            AVCaptureVideoStabilizationModeAuto;
    }
}

// 提交配置
[session commitConfiguration];
~~~

### 启停 session

启停 session 也比较简单，就是一个接口的调用：

~~~ objective-c
// 启动 session
[session startRunning];

// 停止 session
[session stopRunning];
~~~

### 操作线程

官方文档中提到，session 相关操作（尤其是启停）比较耗时，建议切换到后台线程进行处理，以免阻塞主线程。

对于这种情况，简单又有效的做法就是切换到一个串行的后台任务队列，利用 GCD 的 `DISPATCH_QUEUE_SERIAL` 即可。这样既不会阻塞主线程，也不存在线程安全性问题，代码编写起来很简单。

其他更多详细使用说明，大家可以参考官方 demo：[AVCam-iOS: Using AVFoundation to Capture Images and Movies](https://developer.apple.com/library/content/samplecode/AVCam/Introduction/Intro.html)。

## RTCCameraVideoCapturer

iOS 的视频采集接口定义为 `RTCVideoCapturer`，目前只有 `RTCCameraVideoCapturer` 和 `RTCFileVideoCapturer` 两个实现，分别是相机采集和本地 mp4 文件“采集”。

和安卓不一样，`RTCVideoCapturer` 除了数据回调接口外，没有定义任何其他接口，选择设备、参数的逻辑，都交给了调用方，当然，iOS 的这些逻辑实现起来也确实比较简单。

选好了设备和参数之后，开始采集的逻辑实现在 `startCaptureWithDevice:format:fps:completionHandler` 中，其过程和前面介绍的 `AVCaptureSession` 使用说明基本一致，但有几个要点：

+ WebRTC 封装了一个 `RTCDispatcher` 类，用来实现三种类型的任务调度：主线程，AVCaptureSession 线程，AudioSession 线程；
+ 在 init 函数中添加 output device，但并未调用 `beginConfiguration` 和 `commitConfiguration`，因为这里只做了添加一个 output device 的操作，本身是原子的；
+ 调用了 `AVCaptureDevice` 的 `lockForConfiguration` 和 `unlockForConfiguration` 来实现对硬件资源配置的独占访问；
+ 配置 input device 时，先移除老的 device，再添加新的 device，那这就需要利用事务机制了；

### 获取采集数据

在 `setupVideoDataOutput` 函数中，把 `self` 设置为 `AVCaptureVideoDataOutput` 的 delegate，在 `captureOutput:didOutputSampleBuffer:fromConnection` 中收到采集的数据，在 `captureOutput:didDropSampleBuffer:fromConnection` 中收到丢弃数据的通知。

采集到的数据封装在 `CMSampleBufferRef` 对象中，我们可以从中获取 `CVPixelBufferRef`（关于 CoreVideo 里的各种 image buffer，后面我们再仔细介绍）。

iOS 获取图像方向的逻辑还是比安卓要简单得多，这主要得益于 Apple 对硬件和系统的强硬控制：

~~~ objective-c
#if TARGET_OS_IPHONE
  switch (_orientation) {
    case UIDeviceOrientationPortrait:
      _rotation = RTCVideoRotation_90;
      break;
    case UIDeviceOrientationPortraitUpsideDown:
      _rotation = RTCVideoRotation_270;
      break;
    case UIDeviceOrientationLandscapeLeft:
      _rotation = usingFrontCamera ? RTCVideoRotation_180 : RTCVideoRotation_0;
      break;
    case UIDeviceOrientationLandscapeRight:
      _rotation = usingFrontCamera ? RTCVideoRotation_0 : RTCVideoRotation_180;
      break;
    case UIDeviceOrientationFaceUp:
    case UIDeviceOrientationFaceDown:
    case UIDeviceOrientationUnknown:
      // Ignore.
      break;
  }
#else
  // No rotation on Mac.
  _rotation = RTCVideoRotation_0;
#endif
~~~

不过 iOS 获取图像时间戳则比安卓麻烦：

~~~ objective-c
int64_t timeStampNs =
    CMTimeGetSeconds(CMSampleBufferGetPresentationTimeStamp(sampleBuffer)) *
    kNanosecondsPerSecond;
~~~

采集到视频数据后，会封装为 `RTCVideoFrame` 对象，通过 `RTCVideoCapturerDelegate` 回调出去，至于之后的处理，且听下回分解 :)

### 切换摄像头

前面提到，`RTCCameraVideoCapturer` 是从选择完设备之后再接管工作，所以切换摄像头就需要调用方切换相机设备后重新调用 `startCaptureWithDevice:format:fps:completionHandler` 了，这个逻辑实现在 `ARDCaptureController` 类中。

## RTCVideoFrame

`RTCVideoFrame` 是对视频数据的封装，它内部用 `RTCVideoFrameBuffer` 表示实际的视频数据。`RTCVideoFrameBuffer` 是一个 protocol，它的实现有 `RTCCVPixelBuffer`, `RTCI420Buffer` 和 `RTCMutableI420Buffer`。

CoreVideo 里有多种 image buffer，`CVImageBufferRef` 算是基类，`CVPixelBufferRef`, `CVOpenGLESTextureRef`, `CVOpenGLTextureRef`, `CVOpenGLBufferRef`, `CVMetalTextureRef` 算是子类。

正如它们的名字所示：

+ `CVPixelBufferRef` 表示的是内存像素数据，格式包括 RGB YUV 等；
+ `CVOpenGLESTextureRef` 表示的是 OpenGL ES 的纹理数据；
+ `CVOpenGLTextureRef` 表示的是 OpenGL 的纹理数据；
+ `CVOpenGLBufferRef` 表示的是 OpenGL 的 buffer 数据；
+ `CVMetalTextureRef` 表示的是 Metal 的纹理数据；

在 WebRTC 里，相机采集使用 `AVCaptureVideoDataOutput` 接收数据，格式是 `CVPixelBufferRef`，而 WebRTC 内部则是使用的 I420 格式进行存储和传递，`CVPixelBufferRef` 到 I420 的转换，在 `RTCCVPixelBuffer.mm` 中实现。

iOS 不支持相机直接输出 OpenGL ES texture，这一点和安卓不同，但可以把 YUV 数据上传到 OpenGL ES texture，具体可以查看[官方 demo GLCameraRipple](https://developer.apple.com/library/content/samplecode/GLCameraRipple/Introduction/Intro.html)。

## 参考文章

+ [AVCaptureSession](https://developer.apple.com/documentation/avfoundation/avcapturesession)
+ [Setting Up a Capture Session](https://developer.apple.com/documentation/avfoundation/cameras_and_media_capture/setting_up_a_capture_session)
+ [AVCam-iOS: Using AVFoundation to Capture Images and Movies](https://developer.apple.com/library/content/samplecode/AVCam/Introduction/Intro.html)
