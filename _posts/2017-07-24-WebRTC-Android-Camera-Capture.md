---
layout: post
title: WebRTC Native 源码导读（一）：安卓相机采集实现分析
tags:
    - 实时多媒体
    - WebRTC
---

前面分享了一套[开箱即用的 WebRTC 开发环境](/2017/06/17/out-of-the-box-webrtc-dev-env/)，希望能给对 WebRTC 感兴趣的朋友带来帮助。不过有了开发环境只是迈出了万里长征第一步，后面的事情还得仔细研读源码才行，所以这里给大家先带来 WebRTC 的安卓 demo 工程—— AppRTC-Native 的源码导读系列。（[十个月前说好的拆 Dagger2](/2016/09/15/Understand-RxJava/) 看来又要等等了，海涵海涵...）

WebRTC 的代码量不小，一次性看明白不太现实，在本系列中，我将试图搞清楚三个问题：

1. 客户端之间如何建立连接？
2. 客户端之间如何实现数据传输？
3. 音视频数据的采集、预览、编码、传输、解码、渲染完整流程。

本文是第一篇，我将从最熟悉的采集入手，分析一下 WebRTC-Android 相机采集的实现。

WebRTC-Android 的相机采集主要涉及到以下几个类：Enumerator，Capturer，Session，SurfaceTextureHelper。

其中 Enumerator 创建 Capturer，Capturer 创建 Session，实现对相机的操作，SurfaceTextureHelper 实现用 SurfaceTexture 接收数据。

## Enumerator

`CameraEnumerator` 接口如下：

~~~ java
public interface CameraEnumerator {
  public String[] getDeviceNames();
  public boolean isFrontFacing(String deviceName);
  public boolean isBackFacing(String deviceName);
  public List<CaptureFormat> getSupportedFormats(String deviceName);
  public CameraVideoCapturer createCapturer(
      String deviceName, CameraVideoCapturer.CameraEventsHandler eventsHandler);
}
~~~

主要是获取设备列表、检查朝向、创建 Capturer。

这是典型的抽象处理，它用 API 无关的接口统一（也封装）了 Camera1 和 Camera2 不同的 API，而实现这个统一时，`deviceName` 是核心。把不同 API 下的相机设备都映射到各自的 `deviceName`，后续的操作就可以都指定 `deviceName` 了。当然也可以不用字符串，用整数 id 也可以，只不过可读性就会变差了。

## Capturer

WebRTC 视频采集的接口定义为 `VideoCapturer`，其中定义了初始化、启停、销毁等操作，以及接收启停事件、数据的回调。相机采集的实现是 `CameraCapturer`，针对不同的相机 API 又分为 `Camera1Capturer` 和 `Camera2Capturer`。相机采集大部分逻辑都封装在 `CameraCapturer` 中，只有创建 `CameraSession` 的代码在两个子类中有不同的实现。

下面分别看看 `VideoCapturer` 几个重要的 API 实现逻辑。

### `initialize`

`initialize` 比较简单，只是保存一下传入的相关对象。

### `startCapture`

`startCapture` 则会先检查当前是否正在创建 session，或者已有 session 正在运行，这里保证了不会同时存在多个 session 在运行。而众多状态成员的访问都通过 `stateLock` 进行保护，避免多线程安全问题。

如果需要创建 session，则在相机操作线程创建 session，同时在主线程检测相机操作的超时。所有相机的操作都切换到了单独的相机线程，以避免造成主线程阻塞，而检查超时自然不能在相机线程，否则相机线程被阻塞住之后超时回调也不会执行。

我们发现 capturer 中并没有实际相机操作的代码，开启相机、预览的代码都封装在了 `CameraSession` 中，那这样  capturer 的逻辑就得到了简化，切换摄像头、失败重试都只需要创建 session 即可，capturer 可以专注于状态维护和错误处理的逻辑。

`CameraCapturer` 状态维护和错误处理的逻辑还是非常全面的：相机开启状态、相机运行状态、切换摄像头状态、错误重试、相机开启超时，全部都考虑到了。另外相机切换、开关相机、错误事件，统统都有回调通知。这里就充分体现出了 demo 和产品的差别，开启相机预览的 demo 十行代码就能搞定，而要全面考虑各种异常情况，就需要费一番苦心了。

不过这里仍有一点小瑕疵，错误回调的参数是字符串，虽然可以很方便的打入日志，但不利于代码判断错误类型。最好是参数使用错误码，然后准备一个错误码到错误信息的转换函数。

### `stopCapture`

`stopCapture` 时会先判断是否正在创建 session，如果正在创建，那就需要等待其创建完毕。通过检查后，如果当前有 session 正在运行，就在相机线程关闭 session。

### `changeCaptureFormat`

改变采集格式需要重启采集，即先 `stopCapture`，再 `startCapture`。这俩操作都是异步的，会不会有问题？这就涉及到 Handler 的一点知识了，向 Handler 提交的消息、任务，都会被加入到同一个队列中，提交到队列中的任务会保证按序执行，即先提交一定会先执行，所以这里我们不必担心关闭相机和开启相机顺序错乱。

### `switchCamera`

`switchCamera` 也会先停止老的 session，再创建新的 session，只不过还需要检查相机个数、实现切换状态通知逻辑。

这块代码应该有个小问题：`startCapture` 会把 `openAttemptsRemaining` 设置为 `MAX_OPEN_CAMERA_ATTEMPTS`，但切换摄像头时只会将其设置为 1，这个不对称应该没什么道理，所以我认为应该保持一致。

## Session

前面我们已经知道，和相机 API 实际打交道的代码都在 `CameraSession` 中，这里我们就一探其究竟。

开启相机、开启预览、设置事件回调的代码都在创建 session 的工厂方法 `Camera1Session.create` 和  `Camera2Session.create` 中。停止相机和预览则定义了一个 `stop` 接口。

具体的相机 API 使用就比较简单了。

### Camera1

+ 创建 `Camera` 对象：`Camera.open`；
+ 设置预览 SurfaceTexture，用来接收帧数据（位于显存中）：`camera.setPreviewTexture`；
+ 选择合适的相机预览参数（尺寸、帧率、对焦）：`Parameters` 和 `camera.setParameters`；
+ 如果需要获取内存数据回调，则需要设置 buffer 和 listener：`camera.addCallbackBuffer` 和 `camera.setPreviewCallbackWithBuffer`；
+ 如果需要相机服务为我们调整数据方向，则可以设置旋转角度：`camera.setDisplayOrientation`；
+ 开启预览：`camera.startPreview`；
+ 停止预览：`camera.stopPreview` 和 `camera.release`；

### Camera2

+ 创建 `CameraManager` 对象，相机操作始于“相机管家”：`context.getSystemService(Context.CAMERA_SERVICE)`；
+ 创建 `CameraDevice` 对象：`cameraManager.openCamera`；
+ 和 Camera1 不同，Camera2 的操作都是异步的，调用 `openCamera` 时我们会传入一个回调，在其中接收相机操作状态的事件；
+ 创建成功：`CameraDevice.StateCallback#onOpened`；
+ 创建相机对象后，开启预览 session，设置数据回调：`camera.createCaptureSession`，同样，这个操作也会传入一个回调；
+ session 开启成功：`CameraCaptureSession.StateCallback#onConfigured`；
+ 开启 session 后，设置数据格式（尺寸、帧率、对焦），发出数据请求：`CaptureRequest.Builder` 和 `session.setRepeatingRequest`；
+ 停止预览：`cameraCaptureSession.stop` 和 `cameraDevice.close`；

**2017.07.27 update：才发现漏掉了一块会困扰很多人的内容：图像方向问题。**

### 图像方向

通常前置摄像头输出的图像方向是逆时针旋转 270° 的，后置摄像头是 90°，但存在一些意外情况，例如 Nexus 5X 前后置都是 270°。

在 Camera1 里我们可以通过 `camera.setDisplayOrientation` 接口来控制相机的输出图像角度，但实际上无论是获取内存数据，还是获取显存数据（SurfaceTexture），这个调用都不会改变数据，它只是影响了相机输出数据时携带的变换矩阵的方向。Camera2 里没有相应的接口，但相机服务会自动为我们合理调整变换矩阵方向，所以相当于我们正确地调用了类似的接口。

如果利用 `camera.setPreviewDisplay` 或者 `camera.setPreviewTexture` 实现预览，那 `camera.setDisplayOrientation` 确实会让预览出来的图像方向发生变化，因为相机服务在渲染到 SurfaceView/TextureView 时会应用变换矩阵，使得预览画面是旋转之后的画面。

除了方向还有一个镜像的问题，Camera1 在前置摄像头时会自动为我们翻转一下画面（当然也只是修改了变换矩阵），例如前置摄像头输出的图像方向是逆时针旋转 270° 时，那就会把图像上下翻转，如果我们再设置一个旋转 90°，把图像旋正，那就相当于是左右翻转，也就达到了镜像的效果，即：前置摄像头我们用左手摸左边的脸，预览里也是显示在屏幕左边（但预览在和我们四目相对，所以实际是“他”的右边，是有点绕...）。

至于怎么设置 `camera.setPreviewDisplay` 的参数，使得直接预览可以方向正确，可以使用以下代码：

~~~ java
private static int getRotationDegree(int cameraId) {
    int orientation = 0;

    WindowManager wm = (WindowManager) applicationContext
        .getSystemService(Context.WINDOW_SERVICE);
    switch (wm.getDefaultDisplay().getRotation()) {
      case Surface.ROTATION_90:
        orientation = 90;
        break;
      case Surface.ROTATION_180:
        orientation = 180;
        break;
      case Surface.ROTATION_270:
        orientation = 270;
        break;
      case Surface.ROTATION_0:
      default:
        orientation = 0;
        break;
    }

    if (cameraInfo.facing == Camera.CameraInfo.CAMERA_FACING_FRONT) {
        return (720 - (cameraInfo.orientation + orientation)) % 360;
    } else {
        return (360 - orientation + cameraInfo.orientation) % 360;
    }
}
~~~

## `SurfaceTextureHelper`

`SurfaceTextureHelper` 负责创建 `SurfaceTexture`，接收 `SurfaceTexture` 数据，相机线程的管理。

创建 `SurfaceTexture` 有几点注意事项：

+ 创建 OpenGL texture 时所在的线程需要准备好 GL 上下文，WebRTC 中将这部分逻辑封装在 `EglBase` 类中；
+ 创建 `SurfaceTexture` 所在的线程，将是其数据回调 `onFrameAvailable` 发生的线程；不过 API 21 引入了一个新的重载版本，支持指定回调所在线程的 Handler；

    ~~~ java
    // The onFrameAvailable() callback will be executed on the SurfaceTexture ctor thread. 
    // See: http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/
    // android/5.1.1_r1/android/graphics/SurfaceTexture.java#195.
    // Therefore, in order to control the callback thread on API lvl < 21, 
    // the SurfaceTextureHelper is constructed on the |handler| thread.
    ~~~

## 有哪些坑

+ 低版本（5.0 以前）的系统上，Camera1 停止预览时，不要手贱地调用下列接口设置 null 值：`setPreviewDisplay`/`setPreviewCallback`/`setPreviewTexture`（文档中确实也说过不要调用...），否则可能导致系统服务全线崩溃，最终导致手机重启：

    ![](https://imgs.piasy.com/2017-07-24-set_null_cause_system_services_crash.png)

+ Camera1 停止预览可能存在死锁（没有解决）：

    ~~~ java
    // Note: stopPreview or other driver code might deadlock. Deadlock in
    // android.hardware.Camera._stopPreview(Native Method) has been observed on
    // Nexus 5 (hammerhead), OS version LMY48I.
    camera.stopPreview();
    ~~~

+ Camera2 相关的代码在 4.4.2 之前的系统上遇到 VerifyError：

    ~~~ java
    try {
        return cameraManager.getCameraIdList();
        // On Android OS pre 4.4.2, a class will not load because of VerifyError if it contains a
        // catch statement with an Exception from a newer API, even if the code is never executed.
        // https://code.google.com/p/android/issues/detail?id=209129
    } catch (/* CameraAccessException */ AndroidException e) {
        Logging.e(TAG, "Camera access exception: " + e);
        return new String[] {};
    }
    ~~~

+ 利用 SurfaceTexture 接收帧数据，有些机型可能获取到的数据是黑屏（MX5 遇到过）：[需要设置 SurfaceTexture 的 buffer size，`surfaceTexture.setDefaultBufferSize`](http://stackoverflow.com/a/34337226/3077508)
+ 利用 SurfaceTexture 接收帧数据，通过 `SurfaceTexture.getTimestamp` 接口获取时间戳，这个时间戳是相对时间，而且前面会有几帧值为 0：相对时间的问题可以在首帧记录下和物理时间的差值，然后计算后续每帧的物理时间戳，但头几帧时间戳为 0，所以我们记下差值就得等到非零时，而头几帧则可以直接使用物理时间作为时间戳；
+ `surfaceTexture.updateTexImage` 和 `eglSwapBuffers` 会发生死锁，我们需要自行加锁：

    ~~~ java
    // SurfaceTexture.updateTexImage apparently can compete and deadlock with eglSwapBuffers,
    // as observed on Nexus 5. Therefore, synchronize it with the EGL functions.
    // See https://bugs.chromium.org/p/webrtc/issues/detail?id=5702 for more info.
    synchronized (EglBase.lock) {
    surfaceTexture.updateTexImage();
    }

    synchronized (EglBase.lock) {
    EGL14.eglSwapBuffers(eglDisplay, eglSurface);
    }
    ~~~

+ 有些机型上，用 `TextureView` 实现预览，`onSurfaceTextureAvailable` 回调不会被调用，导致无法开启预览，这个问题有可能可以通过开启硬件加速得以解决（参考 [StackOverflow 这个问题](https://stackoverflow.com/a/28895727/3077508)，我还顶过），但有可能这个办法也不管用，那么恭喜你，得再费一番脑细胞了。我就遇到过这种情况，在一款 OPPO 4.3 的手机上，折腾半天发现延迟一会儿重设一次 `LayoutParams` 就能触发，所以就先这么搞了；

## 内存抖动优化

运行 AppRTC-Android 程序，我们会发现内存抖动非常严重：

![](https://imgs.piasy.com/2017-07-23-monitor_1.png)

这块我们可以利用 Allocation Tracker 进行分析和优化，具体内容等其他部分的基础内容发布后再整理发布，敬请期待 :)

## 后续文章

+ [WebRTC-Android 源码导读（二）：预览实现分析](/2017/07/26/WebRTC-Android-Render-Video/)
+ [WebRTC-Android 源码导读（三）：视频硬编码实现分析](/2017/08/08/WebRTC-Android-HW-Encode-Video/)
