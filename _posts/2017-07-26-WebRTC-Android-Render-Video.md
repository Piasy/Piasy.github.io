---
layout: post
title: WebRTC-Native 源码导读（二）：安卓预览实现分析
tags:
    - 实时多媒体
    - WebRTC
    - OpenGL
---

在[本系列第一篇](/2017/07/24/WebRTC-Android-Camera-Capture/)中，我们分析了 WebRTC-Android 相机采集的实现，本文中我们将分析预览的实现。

有过一定相机开发经验的朋友可能会疑惑，预览还有什么好分析的，不是直接 `camera.setPreviewDisplay` 或者 `camera.setPreviewTexture` 就能在 `SurfaceView`/`TextureView` 上预览了吗？实际上预览还有更高级的玩法，尤其是需要加上图像处理功能（美颜、特效）时。WebRTC 使用了 OpenGL 进行渲染（预览），涉及下面三个问题：

+ 数据怎么来？
+ 渲染到哪儿？
+ 怎么渲染？

接下来我们就逐步寻找这三个问题的答案。

## 数据怎么来？

在第一篇中我们已经知道，WebRTC 的数据采集由 `VideoCapturer` 完成，`VideoCapturer` 定义了一个 `CapturerObserver` 来接收采集到的数据。而相机数据的输出，无外乎两个途径：`Camera.PreviewCallback`（Camera1） 和 `SurfaceTexture`（Camera1 和 Camera2）。当然，[Camera2 也可以获取 YUV 内存数据](https://github.com/Piasy/CameraCompat/blob/master/CameraCompat/src/main/java/com/github/piasy/cameracompat/compat/Camera2PreviewCallback.java#L45)，但这里就不展开了。

`camera.setPreviewCallbackWithBuffer` 的调用在 `Camera1Session` 中，取得内存数据后将一路回调通知到 `VideoCapturer#onByteBufferFrameCaptured`。

为 `SurfaceTexture` 设置数据回调 `surfaceTexture.setOnFrameAvailableListener` 的调用则在 `SurfaceTextureHelper` 中，显存数据更新后将一路回调通知到`VideoCapturer#onTextureFrameCaptured`。

数据的来源已经一清二楚了，那我们看看 `CapturerObserver` 有哪些实现。

_这里我就忍不住想要吐个槽，Java 的 IDE IntelliJ IDEA，以及基于它开发的 Android Studio，阅读代码那是杠杠的，但要看 WebRTC 的 native 代码时，Mac 下无论是 Xcode 还是 Visual Studio，都渣得不行，看看一个类有哪些子类都做不到。_

言归正传，`CapturerObserver` 只有一个实现类，那就是 `AndroidVideoTrackSourceObserver`，而它在收到数据之后负责把数据抛到 native 层，WebRTC 在 native 层做了很多事情，包括图像处理、编码（软）、传输等，`AndroidVideoTrackSourceObserver` 是帧数据从 Java 层到 native 层的起点。

帧数据在 native 层的旅途我们暂且跳过，下面我们从渲染的终点继续探寻数据流动的线索。

## 渲染到哪儿？

我们要实现预览肯定得有一个 View 来显示，WebRTC 里用的是 `SurfaceView`，虽然 WebRTC 使用了 OpenGL，但它并没有使用 `GLSurfaceView`。其实 `GLSurfaceView` 是 `SurfaceView` 的子类，它实现了 OpenGL 环境的管理，如果不用它，我们就得自己管理 OpenGL 环境。

那为什么好好的代码放着不用呢？因为使用框架/已有代码虽然能省却一番工夫，但它肯定也会带来一些限制，例如使用 `GLSurfaceView` 我们的渲染模式就只有 continously 和 when dirty 了，而如果我们自己管理 OpenGL 环境，那我们的渲染将是完全自定义的。这和舍弃 TCP 保证的可靠传输，自己基于 UDP 实现可靠传输，是一个道理，图的就是灵活性。

实际上 WebRTC 的渲染不需要局限在 `SurfaceView` 及其子类上，OpenGL 只是利用了 `SurfaceView` 提供的 `Surface`，除了 `Surface`，OpenGL 也可以用 `SurfaceTexture`，而 `TextureView` 就能提供 `SurfaceTexture`，所以我们也可以渲染在 `TextureView` 上。

WebRTC 的渲染接口定义为 `VideoRenderer`，它用于预览的实现就是 `SurfaceViewRenderer`，接下来就让我们看看它究竟是如何渲染的。

## 怎么渲染？

既然渲染是用 OpenGL 实现的，那我们就需要了解一下 OpenGL 的一些基础知识。

### OpenGL 和 EGL

OpenGL（Open Graphics Library）是一套跨平台的渲染 2D、3D 计算机图形的库，通常用于视频、游戏，利用 GPU 进行硬件加速处理。OpenGL ES（Open Graphics Library for Embedded Systems，也叫 GLES）是 OpenGL 的一个子集，用于嵌入式系统，在安卓平台上，我们使用的实际上是 GLES API。GLES 也是跨平台的，既然跨平台，那就一定有连接跨平台 API 和具体平台实现的东西，这就是 EGL。EGL 是连接 OpenGL/GLES API 和底层系统 window system（或者叫做“操作系统的窗口系统”）的桥梁（抽象层），它负责上下文管理、窗口/缓冲区绑定、渲染同步（上层绘制 API 和下层渲染 API），让我们可以利用 OpenGL/GLES 实现高性能、硬件加速的 2D/3D 图形开发。

> EGL™ is an interface between Khronos rendering APIs such as OpenGL ES or OpenVG and the underlying native platform window system. It handles graphics context management, surface/buffer binding, and rendering synchronization and enables high-performance, accelerated, mixed-mode 2D and 3D rendering using other Khronos APIs. 

所谓的 OpenGL 环境管理，其实就是 EGL 环境的管理：`EGLContext`，`EGLSurface` 和 `EGLDisplay`。

+ `EGLContext` 是一个容器，里面存储着各种内部的状态（view port，texture 等）以及对这个 context 待执行的 GL 指令，可以说它存储着渲染的输入（配置和指令）；
+ `EGLSurface` 则是一个 buffer，存储着渲染的输出（a color buffer, a depth buffer, and a stencil buffer），它有两种类型，`EGL_SINGLE_BUFFER` 和 `EGL_BACK_BUFFER`，single 就是只有一个 buffer，在里面画了就立即显示到了 display 上，而 back 则有两个 buffer，一个用于在前面显示，一个用于在后面绘制，绘制完了就用 `eglSwapBuffers` 进行切换；
+ `EGLDisplay` 是和“操作系统的窗口系统”的一个连接，它代表了一个显示窗口，我们最常用的是系统默认的显示窗口（屏幕）；

我们首先在渲染线程创建 `EGLContext`，它的各种状态都是 ThreadLocal 的，所以 GLES API 的调用都需要在创建了 `EGLContext` 的线程调用。有了上下文还不够，我们还需要创建 `EGLDisplay`，我们用 `eglGetDisplay` 获取 display，参数通常用 `EGL_DEFAULT_DISPLAY`，表明我们要获取的是系统默认的显示窗口。最后就是利用 `EGLDisplay` 创建 `EGLSurface` 了：`eglCreateWindowSurface`，这个接口除了需要 `EGLDisplay` 参数，还需要一个 surface 参数，它的类型可以是 `Surface` 或者 `SurfaceTexture`，这就是前面说的 OpenGL 既能用 `Surface` 也能用 `SurfaceTexture` 的原因了。

### SurfaceViewRenderer 和 EglRenderer

现在让我们回到 WebRTC 的代码中。WebRTC 把 EGL 的操作封装在了 `EglBase` 中，并针对 EGL10 和 EGL14 提供了不同的实现，而 OpenGL 的绘制操作则封装在了 `EglRenderer` 中。视频数据在 native 层处理完毕后会抛出到 `VideoRenderer.Callbacks#renderFrame` 回调中，在这里也就是 `SurfaceViewRenderer#renderFrame`，而 `SurfaceViewRenderer` 又会把数据交给 `EglRenderer` 进行渲染。所以实际进行渲染工作的主角就是 `EglRenderer` 和 `EglBase14`（EGL14 实现）了。

`EglRenderer` 实际的渲染代码在 `renderFrameOnRenderThread` 中，前面已经提到，GLES API 的调用都需要在创建了 `EGLContext` 的线程调用，在 `EglRenderer` 中这个线程就是 RenderThread，也就是 `renderThreadHandler` 对应的线程。

由于这里出现了异步，而且提交的 `Runnable` 并不是每次创建一个匿名对象，所以我们就需要考虑如何传递帧数据，`EglRenderer` 的实现还是比较巧妙的：它先把需要渲染的帧保存在 `pendingFrame` 成员变量中，保存好后异步执行 `renderFrameOnRenderThread`，在其中首先把 `pendingFrame` 的值保存在局部变量中，然后将其置为 null，这样就实现了一个“接力”的效果，利用一个成员变量，把帧数据从 `renderFrame` 的参数传递到了 `renderFrameOnRenderThread` 的局部变量中。当然这个接力的过程需要加锁，以保证多线程安全，一旦完成接力，双方的操作就无需加锁了，这样能有效减少加锁的范围，提升性能。

_在第一篇的结尾，我们提到了内存抖动的问题，内存抖动肯定是由不合理的内存分配导致的，如果我们分析定位渲染每帧数据时创建的 `Runnable`、`I420Frame` 对象成为了瓶颈，那我们就可以按照这种技巧避免每次创建新的对象。具体怎么实现这里我们先按下不表，且听下回分解_。

`renderFrameOnRenderThread` 中会调用 `GlDrawer` 的 `drawOes`/`drawYuv` 来绘制 OES 纹理数据/YUV 内存数据。绘制完毕后，调用 `eglBase.swapBuffers` 交换 Surface 的前后 buffer，把绘制的内容显示到屏幕上。

### GlRectDrawer

`GlDrawer` 的实现是 `GlRectDrawer`，在这里我们终于见到了期待已久的 shader 代码、vertex 坐标和 texture 坐标。

~~~ java
private static final String VERTEX_SHADER_STRING =
      "varying vec2 interp_tc;\n"
    + "attribute vec4 in_pos;\n"
    + "attribute vec4 in_tc;\n"
    + "\n"
    + "uniform mat4 texMatrix;\n"
    + "\n"
    + "void main() {\n"
    + "    gl_Position = in_pos;\n"
    + "    interp_tc = (texMatrix * in_tc).xy;\n"
    + "}\n";

private static final String OES_FRAGMENT_SHADER_STRING =
      "#extension GL_OES_EGL_image_external : require\n"
    + "precision mediump float;\n"
    + "varying vec2 interp_tc;\n"
    + "\n"
    + "uniform samplerExternalOES oes_tex;\n"
    + "\n"
    + "void main() {\n"
    + "  gl_FragColor = texture2D(oes_tex, interp_tc);\n"
    + "}\n";

private static final FloatBuffer FULL_RECTANGLE_BUF = GlUtil.createFloatBuffer(new float[] {
    -1.0f, -1.0f, // Bottom left.
    1.0f, -1.0f, // Bottom right.
    -1.0f, 1.0f, // Top left.
    1.0f, 1.0f, // Top right.
});
~~~

正如其名，`GlRectDrawer` 封装了绘制矩形的操作，而我们的预览/渲染也确实只需要绘制一个矩形。WebRTC 用到的 shader 代码非常简单，几乎和我在[安卓 OpenGL ES 2.0 完全入门（二）：矩形、图片、读取显存等](/2016/06/14/Open-gl-es-android-2-part-2/)中编写的代码一样简单。不过有一点不同寻常的是，这里并没有对 vertex 坐标进行变换，而是对 texture 坐标进行的变换，所以如果我们需要对图像进行旋转操作，直接使用 `Matrix.rotateM` 会导致十分诡异的效果，必须搭配 `Matrix.translateM` 才能正常。例如下图：

<img src="https://imgs.piasy.com/2017-07-26-mk82T.png" alt="mk82T.png" style="height:400px">

说到这里我就不得不提另一个开源项目 [Grafika](https://github.com/google/grafika) 了，那里面预览绘制的 shader 代码和 WebRTC 如出一辙，也对 texture 坐标做了变换，之前我尝试旋转图像时就遇到了上图的窘境，最后在一位商汤“老大哥”的帮助下才解决了问题，当然，他也是从 [StackOverflow](https://stackoverflow.com/a/34668611/3077508) 上找到的答案。如果大家打开了这个 StackOverflow 的链接，而且知道 fadden 这个 id，一定会感叹，原来大神也会瞎扯淡。fadden 在媒体开发领域的地位，应该不逊于 JakeWharton 在应用开发领域的地位，[bigflake](http://bigflake.com/mediacodec/)、[Grafika](https://github.com/google/grafika)、[Graphics architecture](https://source.android.com/devices/graphics/architecture) 都是 fadden 的大作，但 fadden 大神对这个问题的回答确实有失水准 :)

好了让我们继续看 `GlRectDrawer` 的代码。以 `drawOes` 为例，我们发现确实都是比较基础的 OpenGL 调用了：

~~~ java
@Override
public void drawOes(int oesTextureId, float[] texMatrix, int frameWidth, int frameHeight,
    int viewportX, int viewportY, int viewportWidth, int viewportHeight) {
  prepareShader(OES_FRAGMENT_SHADER_STRING, texMatrix);
  GLES20.glActiveTexture(GLES20.GL_TEXTURE0);
  // updateTexImage() may be called from another thread in another EGL context, so we need to
  // bind/unbind the texture in each draw call so that GLES understads it's a new texture.
  GLES20.glBindTexture(GLES11Ext.GL_TEXTURE_EXTERNAL_OES, oesTextureId);
  drawRectangle(viewportX, viewportY, viewportWidth, viewportHeight);
  GLES20.glBindTexture(GLES11Ext.GL_TEXTURE_EXTERNAL_OES, 0);
}

private void prepareShader(String fragmentShader, float[] texMatrix) {
  final Shader shader;
  if (shaders.containsKey(fragmentShader)) {
    shader = shaders.get(fragmentShader);
    shader.glShader.useProgram();
  } else {
    // Lazy allocation.
    shader = new Shader(fragmentShader);
    shaders.put(fragmentShader, shader);
    shader.glShader.useProgram();
    // ...
    GlUtil.checkNoGLES2Error("Initialize fragment shader uniform values.");
    // Initialize vertex shader attributes.
    shader.glShader.setVertexAttribArray("in_pos", 2, FULL_RECTANGLE_BUF);
    shader.glShader.setVertexAttribArray("in_tc", 2, FULL_RECTANGLE_TEX_BUF);
  }
  // Copy the texture transformation matrix over.
  GLES20.glUniformMatrix4fv(shader.texMatrixLocation, 1, false, texMatrix, 0);
}
~~~

为 uniform 变量赋值、为顶点 attribute 赋值、绑定 texture、绘制矩形……当然这里对代码做了适当的封装，增加了代码的复用性，使得 `drawYuv`/`drawRgb` 的流程也基本相同。

## TextureViewRenderer

WebRTC 中 实现了 Renderer 的 View 只有 SurfaceView 版本，如果我们有多个视频同时渲染叠加显示，我们会发现拖动小窗口时会留下黑色残影，我推测这是因为 SurfaceView 的 Surface 和 View 树是独立的，两者位置的更新没有保持同步，所以出现了残影。不过 Nexus 5X 7.1.1 不存在此问题，应该是 7.1.1 解决了这个问题。

好消息是 TextureView 不存在拖动残影的问题，坏消息是 WebRTC 并没有 TextureViewRenderer。不过这点小问题肯定难不倒技术小能手们，对 SurfaceViewRenderer 稍作修改就可以得到 TextureViewRenderer 了。具体代码我将在后续的文章中发布。

## 总结

在本文中，我们接着第一篇采集的内容，理清楚了帧数据在预览过程中的流动，以及预览实现过程的细节，OpenGL 相关的内容占了较大的篇幅。接下来第三篇我将分析 WebRTC 视频硬编码的实现，敬请期待 :)

## 后续文章

+ [WebRTC-Android 源码导读（三）：视频硬编码实现分析](/2017/08/08/WebRTC-Android-HW-Encode-Video/)
