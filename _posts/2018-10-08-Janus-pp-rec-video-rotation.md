---
layout: post
title: 为 janus-pp-rec 增加视频旋正功能
tags:
    - 实时多媒体
    - WebRTC
    - Janus
---

Janus Gateway 支持 server 端录制，保存的文件格式是对 RTP 报文的一种自定义封装格式（MJR），音视频数据单独存储，官方提供了一个 `janus-pp-rec` 的程序，可以把 MJR 格式的文件转换为其他封装格式的文件，然后我们可以利用 ffmpeg 把音视频文件合并为一个文件，[命令如下（以 H.264 和 OPUS 编码为例）](https://stackoverflow.com/a/41744149/3077508)：

``` bash
janus-pp-rec video.mjr video.mp4
janus-pp-rec audio.mjr audio.opus
ffmpeg -i audio.opus -i video.mp4 -c:v copy -c:a opus \
    -strict experimental merged_output.mp4
```

但是此前我发现转换视频数据时，janus-pp-rec 程序不是 crash，就是转换出来的视频无法播放，最后发现[是 `janus-pp-rec.c` 包含的一个 `pp-rtp.h` 里面少了对字节序的定义/引用](https://github.com/meetecho/janus-gateway/pull/1398)。

这个问题修复之后，又暴露出一个新问题：画面方向不对，需要顺时针旋转 90 度才行。

通过测试我发现，WebRTC Android/iOS SDK 推流端没有对视频数据做旋转操作，而是把图像旋转角度放在了 RTP header 里，由接收端在渲染时做旋转操作。而浏览器端在 Android 平台上，推流端对视频数据做了旋转操作。鉴于此，我们保存的 RTP 数据在转换为 mp4 格式时，需要做旋转操作。

首先我们简单了解下 janus-pp-rec 的主体流程。

## janus-pp-rec 主体流程介绍

janus-pp-rec 的主体流程在 `postprocessing/janus-pp-rec.c` 中：

+ 程序先扫描文件，解析出所有 RTP header，形成一个有序的 `janus_pp_frame_packet` 列表；
  - 这个过程中，对时间戳的处理还是挺复杂的，不过这次我们的关注点不在这里，先跳过；
+ 接着，如果是视频文件，则调用 `janus_pp_webm_preprocess` 或 `janus_pp_h264_preprocess` 解析视频元数据（宽高、帧率）；
+ 接着，调用 `janus_pp_XXX_create` 创建数据文件、环境；
+ 接着，调用 `janus_pp_XXX_process` 处理 RTP 报文，生成目标文件；
+ 最后，调用 `janus_pp_XXX_close` 进行清理工作，释放资源；

这次我们重点关注 h264 相关的函数，在 `postprocessing/pp-h264.c` 中，不过在开始改代码之前，我们需要对 MJR 和 RTP 的数据格式有个基本的了解。

## MJR 格式简介

由于 RTP 报文 header 没有 payload 长度字段，也不像 H.264 的 NAL unit 定义了 start code 来分隔报文，所以保存到文件时需要自己加上分隔信息。start code 是一种思路，通过一个特殊的 magic number 来作为分隔符，内容中如果出现 magic number，则进行转义，另一种思路则是在 header 里增加长度字段。MJR 格式结合了这两种思路，magic number 发挥一点区分类型、错误发现的作用，不过这只是作者的一种选择，未必最优。

MJR 数据分块存储，包括 metadata 块、RTP 数据块，每块前八个字节是 magic number，紧接着两个字节为块长度，接下来则为块数据。

```
    0                   1                   2                   3
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                          magic number...                      
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                           ...magic number                         |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |         block length          |            block data...      |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                        ....block data                         
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

magic number 有两种：

+ ASCII `MJR00001` 表明为 metadata 块；
+ ASCII `MEETECHO` 表明为 RTP 报文块；

下面是一个视频录制文件的开头：

![](https://imgs.piasy.com/2018-10-08-mjr_video_example-1.png)

我们可以看到，`MJR00001` 之后的两个字节是 0x0045，即 69 字节，而下一个 `MEETECHO` 出现在第 79 个字节（从 0 开始），前面一共 79 字节，即 8 + 2 + 69 字节。

上面的例子中，metadata 为：

``` json
{
  "t": "v",
  "c": "h264",
  "s": 1538926050574237,
  "u": 1538926051708146
}
```

各字段分别表示类型、编码格式、创建时间、最后修改时间。

## RTP 报文格式简介

RTP 报文分为 header 和 payload 两部分，其中 header 又分为 fixed header 和 header extension 两部分。

### fixed header

```
    0                   1                   2                   3
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |V=2|P|X|  CC   |M|     PT      |       sequence number         |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                           timestamp                           |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |           synchronization source (SSRC) identifier            |
   +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
   |            contributing source (CSRC) identifiers             |
   |                             ....                              |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

+ X bit 表明 fixed header 后面是否跟有 header extension；
+ CC 是 CSRC 的数量，每个 CSRC 4 字节；

### header extension

如果 fixed header 的 X bit 为 1，则表明有 ext header。

```
    0                   1                   2                   3
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |      defined by profile       |           length              |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                        header extension                       |
   |                             ....                              |
```

+ 一个 RTP 报文里只允许一个 header extension，「defined by profile」的取值用来标识后续的 ext header 的含义；
+ length 的单位为 4 字节，表明 header extension 部分的长度，允许为 0；
+ header extension 部分大体为 ID + len + data 的格式，len 为 data 字节数减一；

header extension RFC 8285 定义了两种：单字节 header 和双字节 header。

### 单字节 header

单字节 header 的「defined by profile」字段固定为 0xBEDE。

```
       0
       0 1 2 3 4 5 6 7
      +-+-+-+-+-+-+-+-+
      |  ID   |  len  |
      +-+-+-+-+-+-+-+-+


       0                   1                   2                   3
       0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |       0xBE    |    0xDE       |           length=3            |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |  ID   | L=0   |     data      |  ID   |  L=1  |   data...
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
            ...data   |    0 (pad)    |    0 (pad)    |  ID   | L=3   |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |                          data                                 |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

### 双字节 header

单字节 header 的「defined by profile」字段固定为 0x100X，最后四个 bit 为 app bits。

```
       0                   1
       0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |         0x100         |appbits|
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+


       0                   1
       0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |       ID      |     length    |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+


       0                   1                   2                   3
       0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |       0x10    |    0x00       |           length=3            |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |      ID       |     L=0       |     ID        |     L=1       |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |       data    |    0 (pad)    |       ID      |      L=4      |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |                          data                                 |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

### 实际例子

好了，我们结合实际例子理解一下上述内容。

上面第一幅截图中，第一个 RTP 报文块的 header 为（前 12 字节）：

```
806B0001 00000000 37589AF0
```

十六进制 `806B` 即为二进制 `1000 0000 0110 1011`，即 X bit 为 0，没有 header extension，CC 为 0，没有 CSRC，PT 为 107。

下面我们看一个有 header extension 的例子：

![](https://imgs.piasy.com/2018-10-08-mjr_video_example_rtp_header_ext.png)

header 的前 20 字节为：

```
90EB0005 00000000 37589AF0 BEDE0001 40010000
```

十六进制 `90EB` 即为二进制 `1001 0000 1110 1011`，即 X bit 为 1，有 header extension，CC 为 0，没有 CSRC，PT 为 107。

「defined by profile」字段为 `0xBEDE`，即为单字节 header，length 字段为 1，即 header extension 部分有 4 字节。

十六进制 `40` 即为二进制 `0100 0000`，即 ID 字段为 4，len 字段为 0，即 data 长度为 1 字节。

[各个 extension 的 ID 值是通过 SDP 协商出来的](https://groups.google.com/d/msg/discuss-webrtc/f5944a56jx8/-NOBbmO0GwAJ)，[但 Google 的 WebRTC 实现是提供固定值 offer](https://webrtc.googlesource.com/src/+/ab09039d2ab6d66d3a57c6caa626034d9effb62e/modules/rtp_rtcp/include/rtp_rtcp_defines.h#100)，这次测试的 answer 有如下内容：

```
a=extmap:4 urn:3gpp:video-orientation
a=extmap:6 http://www.webrtc.org/experiments/rtp-hdrext/playout-delay
```

因此 ID 4 表示是图像方向（Coordination of Video Orientation, CVO），[其格式为](https://webrtc.googlesource.com/src/+/ab09039d2ab6d66d3a57c6caa626034d9effb62e/modules/rtp_rtcp/source/rtp_header_extensions.cc#154)：

```
    0                   1
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |  ID   | len=1 |0 0 0 0 C F R R|
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

其各个 bit 的含义如下：

![](https://imgs.piasy.com/2018-10-08-rtp_header_ext_cvo-1.png)

十六进制 `01` 即为二进制 `0000 0001`，R1 R0 为 `01`，即播放端需要顺时针旋转 90 度。

## 修改思路

了解了处理流程和数据格式之后，我们先简化一下需求：不考虑同一个视频文件中视频方向发生变化的情况。

其实这个功能只需要做两件事：解析出视频旋转角度，转换格式时应用旋转角度。

原本我打算这两件事分别在 `janus_pp_h264_preprocess` 和 `janus_pp_h264_process` 里完成，不过 janus-pp-rec 调用 ffpmeg 的方式是 `av_write_frame`，它无法直接实现旋转操作，简单 google 一番无果后，我决定改变思路：只在 janus-pp-rec 里解析出旋转角度，最后在合并音视频文件时利用 transpose filter 实现旋转处理（[参考](https://stackoverflow.com/a/9570992/3077508)）：

``` bash
# 顺时针旋转 90 度：-vf "transpose=1"
# 顺时针旋转 180 度：-vf "transpose=2,transpose=2"
# 顺时针旋转 270 度：-vf "transpose=2"
ffmpeg -i audio.opus -i video.mp4 -c:a opus \
     -vf "transpose=1" \
    -strict experimental merged_output.mp4
```

怎么把 janus-pp-rec 解析出的旋转角度传递给命令行呢？打印到控制台即可，解析 stdout 就能得到旋转角度啦！

具体代码这里就不赘述了，[敬请关注 GitHub PR](https://github.com/meetecho/janus-gateway/pull/1400)。

## 后记

好了，今天这篇短文就到这里，不过信息量还是不少的，重点是 RTP 报文格式、RTP header extension 机制，结合实际需求理解这些内容才更轻松 :)

## 参考文章

+ [RTP: A Transport Protocol for Real-Time Applications](https://tools.ietf.org/html/rfc3550#section-5.1)
+ [A General Mechanism for RTP Header Extensions](https://tools.ietf.org/html/rfc8285#section-4.2)
+ [3GPP TS 26.114 version 12.7.0 Release 12, 7.4.5](https://www.etsi.org/deliver/etsi_ts/126100_126199/126114/12.07.00_60/ts_126114v120700p.pdf)

---

欢迎大家加入 Hack WebRTC 星球，和我一起钻研 WebRTC。

<img src="https://imgs.piasy.com/2019-11-14-piasy-knowladge-planet.jpeg" alt="piasy-knowladge-planet" style="height:400px">
