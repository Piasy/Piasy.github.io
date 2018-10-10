---
layout: post
title: 解决A/libc Fatal signal 11 (SIGSEGV)错误，这可能是目前最鲁棒的Android声音录制和播放封装库了
tags:
    - 安卓开发
    - 实时多媒体
    - Reactive eXtention
---

安卓开发过程中一旦开始和硬件打交道，以及涉及到一定的native代码之后，各种闪退就开始浮出水面了，声音录制和播放当然不例外，其中最摸不着头脑的就是A/libc: Fatal signal 11 (SIGSEGV) at了。本文总结了YOLO安卓客户端大半年来的安卓音频实践，整理出一套系统API的封装，命名为[RxAndroidAudio](https://github.com/Piasy/RxAndroidAudio)。

## 概览
安卓平台和声音录制与播放相关的主要是4个类：MediaRecorder，MediaPlayer，AudioRecord和AudioTrack。

MediaRecorder可以录制视频和音频到文件，MediaPlayer可以播放视频和音频文件，AudioRecord可以提供接口读取音频流数据（byte数组或者short数组），AudioTrack提供接口用于播放音频流数据。

小结一下，其中MediaRecorder和AudioRecord用于声音录制，MediaPlayer和AudioTrack用于声音播放。AudioRecord和AudioTrack用于操作音频流数据，操作对象是byte数组（或者short数组），而MediaRecorder和MediaPlayer提供了经过更高层抽象和封装接口，直接对文件进行操作，而且他俩功能更丰富，同时支持音频和视频。

本文会涉及到部分关于声音录制和播放更底层的实现，详细的分析并不是本文的内容范围了，此外本文主要目标并不是声音录制和播放的使用教程，本文主要关注的是上述4个类的使用过程中需要注意的地方。

## 基于文件的操作

**[使用MediaRecorder录制声音到文件](https://github.com/Piasy/RxAndroidAudio/blob/master/rxandroidaudio%2Fsrc%2Fmain%2Fjava%2Fcom%2Fgithub%2Fpiasy%2Frxandroidaudio%2FAudioRecorder.java)：**

+  `mRecorder.prepare()`调用需要[捕获`IOException`和`RuntimeException`](https://github.com/Piasy/RxAndroidAudio/blob/f9c840e3aaf0a4512aee433d250c570cb441e4d8/rxandroidaudio%2Fsrc%2Fmain%2Fjava%2Fcom%2Fgithub%2Fpiasy%2Frxandroidaudio%2FAudioRecorder.java#L131)，注意需要捕获`RuntimeException`而不是`IllegalStateException`，尽管Java doc中只声明了会抛出IllegalStateException，但是查看[jni层代码](http://androidxref.com/6.0.1_r10/xref/frameworks/base/media/jni/android_media_MediaRecorder.cpp#330)可以看到，prepare的调用也是可能会触发RuntimeException的；
+  `mRecorder.start()`，`mRecorder.stop()`，`mRecorder.reset()`调用也需要捕获`RuntimeException`，理由同上；
+  无论是prepare还是start，抛出异常之后[都需要reset和release](https://github.com/Piasy/RxAndroidAudio/blob/f9c840e3aaf0a4512aee433d250c570cb441e4d8/rxandroidaudio%2Fsrc%2Fmain%2Fjava%2Fcom%2Fgithub%2Fpiasy%2Frxandroidaudio%2FAudioRecorder.java#L145)；
+  需要保证不会对jni层进行多线程的调用，以免出现下面这样的“静默闪退”（[参考资料](http://stackoverflow.com/questions/14023291/fatal-signal-11-sigsegv-at-0x00000000-code-1-phonegap)），RxAndroidAudio通过[单例](https://github.com/Piasy/RxAndroidAudio/blob/f9c840e3aaf0a4512aee433d250c570cb441e4d8/rxandroidaudio%2Fsrc%2Fmain%2Fjava%2Fcom%2Fgithub%2Fpiasy%2Frxandroidaudio%2FAudioRecorder.java#L61)和[synchronized方法](https://github.com/Piasy/RxAndroidAudio/blob/f9c840e3aaf0a4512aee433d250c570cb441e4d8/rxandroidaudio%2Fsrc%2Fmain%2Fjava%2Fcom%2Fgithub%2Fpiasy%2Frxandroidaudio%2FAudioRecorder.java#L116)来保证这一点：

``` java
A/libc: Fatal signal 11 (SIGSEGV) at 0x00000010 (code=1), thread 9302 (RxComputationTh)
```

+  当用户录完声音，需要停止录音，调用stop的时候，[需要sleep一段时间](https://github.com/Piasy/RxAndroidAudio/blob/f9c840e3aaf0a4512aee433d250c570cb441e4d8/rxandroidaudio%2Fsrc%2Fmain%2Fjava%2Fcom%2Fgithub%2Fpiasy%2Frxandroidaudio%2FAudioRecorder.java#L239)，以免最后几百毫秒录不上，这有可能是安卓系统音频编码器的bug，[参考资料](http://stackoverflow.com/a/24092524/3077508)；
+  当prepare返回后，有些低端的设备需要再延迟一段时间开始说话，以免开头几百毫秒录不上，可以[在prepare返回后延迟几百毫秒（例如300ms）再显示初始化完毕的UI](https://github.com/Piasy/RxAndroidAudio/blob/f9c840e3aaf0a4512aee433d250c570cb441e4d8/app%2Fsrc%2Fmain%2Fjava%2Fcom%2Fgithub%2Fpiasy%2Frxandroidaudio%2Fexample%2FFileActivity.java#L210)，原因还需要继续寻找；

**[使用MediaPlayer播放声音文件](https://github.com/Piasy/RxAndroidAudio/blob/master/rxandroidaudio%2Fsrc%2Fmain%2Fjava%2Fcom%2Fgithub%2Fpiasy%2Frxandroidaudio%2FRxAudioPlayer.java)：**

+  和MediaRecorder一样，prepare，start，stop，reset，release函数的调用都需要捕获异常；
+  和MediaRecorder一样，需要保证不会对jni层进行多线程的调用；
+  MediaPlayer提供了两种音频文件播放方式：通过文件绝对路径指定播放文件，或者使用资源文件id指定；绝对路径的方式[需要调用`mPlayer.prepare()`](https://github.com/Piasy/RxAndroidAudio/blob/f9c840e3aaf0a4512aee433d250c570cb441e4d8/rxandroidaudio%2Fsrc%2Fmain%2Fjava%2Fcom%2Fgithub%2Fpiasy%2Frxandroidaudio%2FRxAudioPlayer.java#L167)，而资源文件id方式不需要；
+  在`MediaPlayer.OnCompletionListener`的`onCompletion`回调中，需要[延迟一定时间再释放MediaPlayer](https://github.com/Piasy/RxAndroidAudio/blob/f9c840e3aaf0a4512aee433d250c570cb441e4d8/rxandroidaudio%2Fsrc%2Fmain%2Fjava%2Fcom%2Fgithub%2Fpiasy%2Frxandroidaudio%2FRxAudioPlayer.java#L199)，否则可能导致下次紧接着的播放无法成功（静默失败，不会抛出异常），原因还需要继续寻找；

## 基于数据流的操作

**[使用AudioRecord录制音频流数据](https://github.com/Piasy/RxAndroidAudio/blob/master/rxandroidaudio%2Fsrc%2Fmain%2Fjava%2Fcom%2Fgithub%2Fpiasy%2Frxandroidaudio%2FStreamAudioRecorder.java)：**

+  和MediaRecorder一样，startRecording需要捕获异常；
+  和MediaRecorder一样，需要保证不会对jni层进行多线程的调用；
+  `mAudioRecord.read`的返回值需要进行错误检查；
+  [`mAudioRecord.read`传入的参数类型需要进行区分](https://github.com/Piasy/RxAndroidAudio/blob/f9c840e3aaf0a4512aee433d250c570cb441e4d8/rxandroidaudio%2Fsrc%2Fmain%2Fjava%2Fcom%2Fgithub%2Fpiasy%2Frxandroidaudio%2FStreamAudioRecorder.java#L129)，`ENCODING_PCM_16BIT`格式的录音需要传入short数组，`ENCODING_PCM_8BIT`格式的录音需要传入byte数组，尽管在[jni层的实现](http://androidxref.com/6.0.1_r10/xref/frameworks/base/core/jni/android_media_AudioRecord.cpp#android_media_AudioRecord_readInArray)都是一样的，但是[Java doc](http://developer.android.com/reference/android/media/AudioRecord.html#read(byte[], int, int))明确说明了这一注意事项，理应遵循；
+  RxAndroidAudio使用`ExecutorService`来执行异步任务（从AudioRecord中循环读取数据）；

**[使用AudioTrack播放音频流数据](https://github.com/Piasy/RxAndroidAudio/blob/master/rxandroidaudio%2Fsrc%2Fmain%2Fjava%2Fcom%2Fgithub%2Fpiasy%2Frxandroidaudio%2FStreamAudioPlayer.java)：**

+  和MediaRecorder一样，write需要捕获异常；
+  和MediaRecorder一样，需要保证不会对jni层进行多线程的调用；

## Bonus Part
Reactive!

RxAndroidAudio之所以叫Rx，就是因为它尽可能的提供了Reactive的API。

RxAudioPlayer播放声音文件：

``` java
public Single<Boolean> play(@NonNull final File audioFile);

public Single<Boolean> play(final Context context, @RawRes final int audioRes);
```

当然RxAudioPlayer也提供了传统的API：

``` java
public boolean playNonRxy(@NonNull final File audioFile,
        final MediaPlayer.OnCompletionListener onCompletionListener,
        final MediaPlayer.OnErrorListener onErrorListener);
        
public boolean playNonRxy(final Context context, @RawRes final int audioRes,
        final MediaPlayer.OnCompletionListener onCompletionListener,
        final MediaPlayer.OnErrorListener onErrorListener);
```

RxAmplitude获取当前说话音量等级：

``` java
public static Observable<Integer> from(@NonNull AudioRecorder audioRecorder);
```

当然RxAmplitude使用的是AudioRecorder的传统API：

``` java
public synchronized int getMaxAmplitude();
```

## 使用
详情请见[Github主页](https://github.com/Piasy/RxAndroidAudio)以及[demo app工程](https://github.com/Piasy/RxAndroidAudio/tree/master/app)。[demo apk下载地址](https://www.pgyer.com/rsyU)。
