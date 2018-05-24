---
layout: post
title: WebRTC Native 源码导读（八）：iOS 视频渲染实现分析
tags:
    - 实时多媒体
    - WebRTC
---

前面我们已经知道，相机数据采集之后，会通过 `RTCVideoCapturerDelegate` 的 `capturer:didCaptureVideoFrame` 回调抛出。在 WebRTC iOS 工程中，这个 delegate 只有一个实现，它就是 `RTCVideoSource`。

视频数据采集之后的处理，iOS 和 Android 有一个很大的差别。iOS 的本地视频预览没有使用自定义渲染，而是简单把 `AVCaptureSession` 和 `AVCaptureVideoPreviewLayer` 绑定了一下，利用了系统已有的预览功能；而 Android 无论是本地视频预览还是远端视频渲染，走的都是 `VideoRenderer` 的自定义渲染流程。

## 本地视频预览

WebRTC iOS 的本地视频预览利用 `AVCaptureVideoPreviewLayer` 实现，Apple 官方文档有对这个类有简单的使用示例：

~~~ objective-c
AVCaptureSession* captureSession = <#Get a capture session#>;
AVCaptureVideoPreviewLayer* previewLayer =
    [AVCaptureVideoPreviewLayer layerWithSession:captureSession];
UIView* aView = <#The view in which to present the layer#>;
previewLayer.frame =
    aView.bounds;  // Assume you want the preview layer to fill the view.
[aView.layer addSublayer:previewLayer];
~~~

其实很简单，就是两步：把 session 设置给 layer；把 layer 加到 view 中。

在 WebRTC 里这个逻辑实现在 `RTCCameraPreviewView` 中：

+ `RTCCameraPreviewView` 继承自 `UIView`，通过重载 `layerClass` 静态函数，使得自己使用 `AVCaptureVideoPreviewLayer` 作为自己的 layer 类型；
+ 在 `ARDVideoCallViewController` 的 `appClient:didCreateLocalCapturer` 函数中，为 `RTCCameraPreviewView.captureSession` 赋值，进而触发 `RTCCameraPreviewView` 的 `setCaptureSession` 函数；
+ 在 `RTCDispatcherTypeMain` 队列里获取 layer，在 `RTCDispatcherTypeCaptureSession` 队列里赋值 `previewLayer.session`，最后再在 `RTCDispatcherTypeMain` 队列里更新视频预览方向；

`setCaptureSession` 函数里切换了三次线程，前后两次是[为了确保 `UIView.layer` 的访问都在主线程](https://webrtc.googlesource.com/src/+/c288dab6e26f85b80b191b06beeffc1fcf3af5d8)，而中间这次切换则是为了保证 layer 设置 session 的操作和 session 启停的线程保持一致，我们可以在 `RTCCameraPreviewView.h` 的注释中看到这个说明：

~~~ objective-c
/** The capture session being rendered in the view. Capture session
 *  is assigned to AVCaptureVideoPreviewLayer async in the same
 *  queue that the AVCaptureSession is started/stopped.
 */
~~~

## 远端视频渲染

让我们采用分析安卓视频渲染时的思路，先看看视频数据从何而来，再看视频数据如何渲染。

### 视频数据来源

从 socket 收到 RTP 包、解封装：

![](https://imgs.piasy.com/2018-05-02-ios_rtp_socket_to_channel.png)

把解封装后的数据放到待解码队列：

![](https://imgs.piasy.com/2018-05-02-ios_rtp_channel_to_stream.png)

待解码数据送入解码器：

![](https://imgs.piasy.com/2018-05-23-ios_rtp_stream_to_decoder.png)

解码后数据进行渲染：

![](https://imgs.piasy.com/2018-05-02-ios_rtp_decoder_to_renderer.png)

_用 Xcode 看 WebRTC iOS 的代码还是很舒心的，可以加断点单步调试，一路到最底层的调用_。

这里总结一下：

+ 在 socket 数据回调线程进行解封装；
+ 解封装后异步到 `worker_thread_` 放入待解码队列（`pc/channel.cc BaseChannel::OnPacketReceived`）；
+ 在 `decode_thread_` 线程消费待解码队列，把数据送入解码器（`video/video_receive_stream.cc VideoReceiveStream::DecodeThreadFunction`）；
+ 在解码器回调中，提交任务到 `incoming_render_queue_` 中，执行渲染任务；

### 视频数据渲染

远端视频渲染有两种模式，Metal 和 OpenGL ES，通过 `RTCMTLVideoView.h` 里的 `RTC_SUPPORTS_METAL` 宏控制，而其设置的依据为 CPU 架构是否为 arm64（`#if defined(__aarch64__)`）。

无论是 Metal 还是 OpenGL ES，都是很大的话题，这里就不做过多展开，只是了解下基本概念。

#### Metal 渲染

Apple 系统为我们提供了 MetalKit 库，其中的 `MTKView` 是核心类，为我们设置、管理渲染循环，我们只需要实现 `MTKViewDelegate`，在 `drawInMTKView` 中执行绘制指令即可。

+ 创建 `MTLDevice`, `MTLCommandQueue`, `MTLLibrary`，加载 shader 代码，创建 `MTLRenderPipelineState`，配置 `MTKView`；
+ 准备 frame：
  - 创建 `MTLCommandBuffer`；
  - 获取 `view.currentRenderPassDescriptor`；
  - 获取 `MTLRenderCommandEncoder`；
  - 把数据上传到 encoder 里；
+ 提交 frame：

    ~~~ objective-c
    [commandBuffer presentDrawable:view.currentDrawable];
    [commandBuffer commit];
    ~~~

Metal 的处理过程如图：

![](https://imgs.piasy.com/2018-05-02-metal_process.png)

更多关于 Metal 的内容，可以查阅[官方文档](https://developer.apple.com/documentation/metal)。

#### OpenGL ES 渲染

和 Metal 类似，Apple 系统也为我们提供了 GLKit 库，其中的 `GLKView` 是核心类，为我们设置、管理 OpenGL 上下文，以及渲染循环，我们只需要实现 `GLKViewDelegate`，在 `glkView:drawInRect` 中执行绘制指令即可。

+ 创建 `EAGLContext`，加载 shader 代码，配置 `GLKView`；
+ 把数据上传到 GPU 中（GL 绘制指令）；

关于 OpenGL ES 的基本使用，可以查阅 [iOS 官方文档](https://developer.apple.com/library/content/documentation/3DDrawing/Conceptual/OpenGLES_ProgrammingGuide/Introduction/Introduction.html)，以及 [OpenGL ES Android 教程](/tags/#OpenGL)。
