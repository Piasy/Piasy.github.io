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
