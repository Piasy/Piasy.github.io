---
layout: post
title: WebRTC Native 源码导读（四）：VideoCRE 与内存抖动优化
tags:
    - 性能优化
    - 实时多媒体
    - WebRTC
---

前面三篇中，我们依次分析了 WebRTC Android 的[视频采集](/2017/07/24/WebRTC-Android-Camera-Capture/index.html)、[视频渲染](/2017/07/26/WebRTC-Android-Render-Video/index.html)和[视频硬编码](/2017/08/08/WebRTC-Android-HW-Encode-Video/index.html)，Live Streaming 视频的前段就已经全了。WebRTC 是个宝，初窥这部分代码时就被它的 Capturer 类的设计惊艳到了，仔细品鉴后越发佩服起来，里面简直填了太多坑了，如此宝贝，如不能为我所用，岂非一大憾事！而前三篇的解读，正是为了今天能将其剥离出来所做的铺垫，现在就有请我们今天的主角——[VideoCRE, _Video Capture, Render and Encode_](https://github.com/Piasy/VideoCRE)——闪亮登场。

## VideoCRE 结构

我们当然可以直接使用 Capturer/Renderer/Encoder，但如果能将它们进行一定的封装，让基本的需求实现起来更加简单，岂不妙哉。

下面介绍一下 VideoCRE 的结构：

![](https://imgs.piasy.com/2017-08-11-video_cre_structure.jpeg)

+ 视频数据由 `VideoCapturer` 采集，例如 `Camera1Capturer`；
+ `VideoCapturer`、`SurfaceTextureHelper` 等由 `VideoSource` 类管理；
+ `VideoCapturer` 采集到的数据会回调给 `VideoCapturer.CapturerObserver`，`VideoSink` 实现了该接口；
+ `VideoSink` 会把数据发送给多个 `VideoRenderer.Callbacks`，例如 `SurfaceViewRenderer` 负责预览，`HwAvcEncoder` 负责视频编码；
+ `HwAvcEncoder` 则会把编码后的数据发送给多个 `MediaCodecCallback`，例如由 `Streamer` 进行网络传输实现直播功能，`Mp4Recorder` 负责本地录制；

同一路视频数据可以被多路消费，例如预览、低码率编码、高码率编码，而同一路编码数据，也可以被多路消费，例如推流、存文件。

## VideoCRE 使用

demo 工程里实现了高低码率两路本地 MP4 录制功能，下面我们看看如何一步步实现这个功能。

首先是配置参数，标清和高清：

``` java
VideoConfig config = VideoConfig.builder()
        .previewWidth(1280)
        .previewHeight(720)
        .outputWidth(448)
        .outputHeight(800)
        .fps(30)
        .outputBitrate(800)
        .build();
VideoConfig hdConfig = VideoConfig.builder()
        .previewWidth(1280)
        .previewHeight(720)
        .outputWidth(720)
        .outputHeight(1280)
        .fps(30)
        .outputBitrate(2000)
        .build();
```

接下来是创建 `VideoCapturer`：

``` java
VideoCapturer capturer = createVideoCapturer();

private VideoCapturer createVideoCapturer() {
    switch (MainActivity.sVideoSource) {
        case VideoSource.SOURCE_CAMERA1:
            return VideoCapturers.createCamera1Capturer(true);
        case VideoSource.SOURCE_CAMERA2:
            return VideoCapturers.createCamera2Capturer(this);
        default:
            return null;
    }
}
```

准备 Renderer 和 Encoder：

``` java
mVideoView = (SurfaceViewRenderer) findViewById(R.id.mVideoView1);
try {
    String filename = "video_source_record_" + System.currentTimeMillis();
    mMp4Recorder = new Mp4Recorder(
            new File(Environment.getExternalStorageDirectory(), filename + ".mp4"));
    mHdMp4Recorder = new Mp4Recorder(
            new File(Environment.getExternalStorageDirectory(), filename + "-hd.mp4"));
} catch (IOException e) {
    e.printStackTrace();
    Toast.makeText(this, "start Mp4Recorder fail!", Toast.LENGTH_SHORT).show();
    finish();
    return;
}
mHwAvcEncoder = new HwAvcEncoder(config, mMp4Recorder);
mHdHwAvcEncoder = new HwAvcEncoder(hdConfig, mHdMp4Recorder);
```

创建 `VideoSink` 和 `VideoSource`，`VideoSource` 也需要视频配置，但只需要使用预览尺寸、帧率，所以用 `config` 或者 `hdConfig` 都可以：

``` java
mVideoSink = new VideoSink(mVideoView, mHwAvcEncoder, mHdHwAvcEncoder);
mVideoSource = new VideoSource(getApplicationContext(), config, capturer, mVideoSink);
```

初始化：

``` java
mVideoView.init(mVideoSource.getRootEglBase().getEglBaseContext(), null);
mHwAvcEncoder.start(mVideoSource.getRootEglBase());
mHdHwAvcEncoder.start(mVideoSource.getRootEglBase());
```

开始采集、录制：

``` java
@Override
protected void onStart() {
    super.onStart();

    mVideoSource.start();
}
```

## 内存抖动优化

完成了 VideoCRE 的剥离后，我发现内存抖动非常严重，CPU 占用也很高：

![](https://imgs.piasy.com/2017-07-23-monitor_1.png)

排查内存抖动，当然首选 Allocation Tracker 了，结果如下：

![](https://imgs.piasy.com/2017-07-23-allocation_track_result1.png)

这里我们可以看到，60% 的内存分配都发生在 BufferInfo 对象上，但这个对象非常小，只有几个 primitive 数据成员，怎么会出现这么多分配呢？我们看次数，15s 内发生了 2.6 万次，每毫秒分配了 1.7 次。看代码发现，是我在单独的线程调用 `dequeueOutputBuffer` 时传入的 timeout 为 0，所以在疯狂的创建 BufferInfo 对象。

单独的线程设置 timeout 为 0 显然不合理，除了这里的内存分配，CPU 占用也会更高，所以我们可以设置一个合适的值，这里我换成 3000，也就是 3ms，结果如下：

![](https://imgs.piasy.com/2017-07-23-monitor_2.png)

我们可以看到，内存抖动减缓了很多，但仍比较明显。CPU 占用率倒是下降很多了。

这时 Allocation Tracker 的结果如下：

![](https://imgs.piasy.com/2017-07-23-allocation_track_result2_2.png)
![](https://imgs.piasy.com/2017-07-23-allocation_track_result2_3.png)

**优化性能切忌盲目，要找准瓶颈，并且测量对比成效。**

我们发现最大的分配竟是由一条日志代码引起的！所以不要以为在日志工具函数内通过变量控制是否打日志就够了，即便日志最终没有打印出来，但拼接日志字符串就可能已经成为瓶颈。

除了日志，还存在两处很高的分配：allocateDirect 和 BufferInfo。

其实 BufferInfo 没必要每次创建，我们消费 MediaCodec 输出是单线程行为，只需要分配一次即可。同理，容纳输出数据的 buffer 也没必要每次分配，只有需要扩容时创建即可。

经过上述优化，内存抖动再次减弱：

![](https://imgs.piasy.com/2017-07-23-monitor_3.png)

分析 Allocation Tracker，较高的内存分配分别为：

+ `Display#getRotation`：19.58%；
+ 取到 MediaCodec 输出后，构造 ByteBuffer 对象的拷贝：17.31%；
+ 帧数据传递过程中 matrix 数组分配：16.32%；

上面这几点都改了之后，再剩下的就是 I420Frame 的创建、日志字符串拼接、Runnable 对象创建了。此外还发现了一个注意点：使用 ExecutorService 时，每次 submit 任务，还会创建一个链表节点对象，而 Handler 会复用 Message 对象，所以我把 ExecutorService 换成了 HandlerThread + Handler 的组合。当然，for each 遍历每次都会创建一个 Iterator 对象，虽然没有成为瓶颈，但也确实可观，何况可以一行代码进行优化，顺手就给做了。

I420Frame 也许可以用对象池来优化，Runnable 则可以把局部变量成员化，但现在其实已经优化了很多（对比测试一分钟的内存分配从 2613976 优化到 240352，优化为了 9.2%），而这些做法需要比较复杂的处理才能确保不会发生消费者-生产者的竞争问题，所以就先告一段落啦！另外，这里并没有贴出具体优化代码，想看代码的朋友，可以查看 [GitHub 仓库的这个 commit](https://github.com/Piasy/VideoCRE/commit/e48dd49b20205fb80ea0aa9602269f4ccd4e2369)。

最后在 Nexus 5X 安卓 7.1.2 上测试发现，Camera2 采集时，存在大量的 Binder 通信，内存抖动严重得多，而其中 48.86% 都是由 Binder 通信导致的。

+ Camera1 采集：一分钟内存增长 0.32MB；
+ Camera2 采集：一分钟内存增长 3.13MB；

对此我就真是黔驴技穷了，只能寄希望于大谷歌了 :(

## 总结

至此，WebRTC（安卓流媒体）视频的前段就已经差不多了，我们了解了采集、预览、编码的大体实现思路，也详细分析了这些步骤里面可能遇到的坑，最后将这三块相关的代码剥离成为了一个可以单独使用的模块：VideoCRE，并对其运行过程中的内存抖动进行了极大的优化，一分钟内存分配优化为了 9.2%。

当然，流媒体前段还有不少内容没有涵盖：美颜、特效（结合人脸识别、场景分割）、更复杂的渲染……这些内容我还需要更深入的学习和理解，才敢分享，而流媒体的中段（传输）、后段（解码播放）则还有更多的内容等着我们，好戏才刚刚开始 :)
