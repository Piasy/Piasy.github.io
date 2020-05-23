---
layout: post
title: WebRTC Native 源码导读（十五）：RTP H.264 封包与解包
tags:
    - 实时多媒体
    - WebRTC
    - 网络
---

之前我在[为 janus-pp-rec 增加视频旋正功能](/2018/10/08/Janus-pp-rec-video-rotation/index.html#rtp-报文格式简介)一文中简单介绍了一点 RTP 协议的内容，重点关注的是视频方向的 RTP header extension，这次我们更深入的了解一下 RTP 协议的内容，看看 H.264 视频数据是如何封装和解封装的。

_其他视频编码格式的封装和解封装大体思路应该是一样的，大家可以举一反三；音频数据的封装和解封装，日后可能我会再整理一篇文章_。

_注：本文是我对这块代码、协议理解的总结，如果你也打算钻研这块代码，那本文可能有一点助益，如果你没有这个打算，那我建议趁早关闭本页面_。

2019.11.3 update: 文章 URL 里有 Mux Demux 字样，实乃谬误，本文分析的是封包和解包，与 Mux/Demux（多路复用和解多路复用）完全是两回事，特此纠正。

## 再谈 RTP 协议

我们首先了解一下 RTP H.264 相关的 RFC，下面的内容是对两篇 RFC 的总结：[RTP: A Transport Protocol for Real-Time Applications](https://tools.ietf.org/html/rfc3550), [RTP Payload Format for H.264 Video](https://tools.ietf.org/html/rfc6184)。

### RTP 包结构

包头有固定 12 个字节部分，以及可选的 `csrc` 和 `ext` 数据（在[为 janus-pp-rec 增加视频旋正功能](/2018/10/08/Janus-pp-rec-video-rotation/index.html#rtp-报文格式简介)一文中有更详细的介绍）：

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

接着是载荷数据，载荷长度在包头中有记录。载荷数据的格式，由不同的 profile 单独定义，profile 的 payload type 值，通过 SDP 协商确定。

下面我们了解一下 H.264 载荷的格式。

### H.264 载荷

H.264 载荷数据的第一个字节格式和 NAL 头一样，其 type 定义如下：

```
      Table 1.  Summary of NAL unit types and the corresponding packet
                types

      NAL Unit  Packet    Packet Type Name               Section
      Type      Type
      -------------------------------------------------------------
      0        reserved                                     -
      1-23     NAL unit  Single NAL unit packet             5.6
      24       STAP-A    Single-time aggregation packet     5.7.1
      25       STAP-B    Single-time aggregation packet     5.7.1
      26       MTAP16    Multi-time aggregation packet      5.7.2
      27       MTAP24    Multi-time aggregation packet      5.7.2
      28       FU-A      Fragmentation unit                 5.8
      29       FU-B      Fragmentation unit                 5.8
      30-31    reserved                                     -
```

H.264 载荷数据的封包有三种模式：Single NAL unit mode (0), Non-interleaved mode (1), Interleaved mode (2)。它们各自支持的 type 见下表：

```
      Table 3.  Summary of allowed NAL unit types for each packetization
                mode (yes = allowed, no = disallowed, ig = ignore)

      Payload Packet    Single NAL    Non-Interleaved    Interleaved
      Type    Type      Unit Mode           Mode             Mode
      -------------------------------------------------------------
      0      reserved      ig               ig               ig
      1-23   NAL unit     yes              yes               no
      24     STAP-A        no              yes               no
      25     STAP-B        no               no              yes
      26     MTAP16        no               no              yes
      27     MTAP24        no               no              yes
      28     FU-A          no              yes              yes
      29     FU-B          no               no              yes
      30-31  reserved      ig               ig               ig
```

注意：**WebRTC iOS H.264 编码时，[无论是 baseline 还是 high profile，都是使用的 Non-interleaved mode](https://webrtc.googlesource.com/src/+/6714bf9f18f2514919f7b0cdc305107076cdc65b/sdk/objc/components/video_codec/RTCVideoEncoderFactoryH264.m#18)，[WebRTC Android 也是如此](https://webrtc.googlesource.com/src/+/6714bf9f18f2514919f7b0cdc305107076cdc65b/sdk/android/src/java/org/webrtc/H264Utils.java#30)**。

因此 WebRTC 里实际使用的只有三种封包模式：NAL unit, STAP-A, FU-A。那我们接下来就看一下这三种模式。

### NAL unit

如果 type 为 `[1, 23]`，则该 RTP 包只包含一个 NALU：

```
     0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |F|NRI|  Type   |                                               |
    +-+-+-+-+-+-+-+-+                                               |
    |                                                               |
    |               Bytes 2..n of a single NAL unit                 |
    |                                                               |
    |                               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                               :...OPTIONAL RTP padding        |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

    Figure 2.  RTP payload format for single NAL unit packet
```

### 包聚合（Aggregation Packets）

为了体现/应对有线网络和无线网络的 MTU 巨大差异，RTP 协议定义了包聚合策略：

+ STAP-A：聚合的 NALU 时间戳都一样，无 DON（decoding order number）；
+ STAP-B：聚合的 NALU 时间戳都一样，有 DON；
+ MTAP16：聚合的 NALU 时间戳不同，时间戳差值用 16 bit 记录；
+ MTAP24：聚合的 NALU 时间戳不同，时间戳差值用 24 bit 记录；
+ 包聚合时，RTP 的时间戳是所有 NALU 时间戳的最小值；

```
     0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |F|NRI|  Type   |                                               |
    +-+-+-+-+-+-+-+-+                                               |
    |                                                               |
    |             one or more aggregation units                     |
    |                                                               |
    |                               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                               :...OPTIONAL RTP padding        |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

    Figure 3.  RTP payload format for aggregation packets
```

STAP-A 示例：

```
     0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                          RTP Header                           |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |STAP-A NAL HDR |         NALU 1 Size           | NALU 1 HDR    |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                         NALU 1 Data                           |
    :                                                               :
    +               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |               | NALU 2 Size                   | NALU 2 HDR    |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                         NALU 2 Data                           |
    :                                                               :
    |                               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                               :...OPTIONAL RTP padding        |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

    Figure 7.  An example of an RTP packet including an STAP-A
               containing two single-time aggregation units
```

### 包拆分（Fragmentation Units，FUs）

在应用层实现包拆分而不是依赖下层网络的拆分机制，好处有二：

+ 可以支持超过 64 KB（IPv4 包最大长度为 64 KB）的 NALU，高清视频文件可能有超大的 NALU；
+ 可以利用 FEC（forward error correction）；

每个分包都有一个编号，一个 NALU 拆分的 RTP 包其序列必须顺序且连续，中间不得插入其他数据的 RTP 包序号。FU 只能拆分 NALU，STAP 和 MTAP 不能拆分，FU 也不能嵌套。FU-A 没有 DON，FU-B 有 DON。

FU-A 格式如下：

```
     0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    | FU indicator  |   FU header   |                               |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+                               |
    |                                                               |
    |                         FU payload                            |
    |                                                               |
    |                               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                               :...OPTIONAL RTP padding        |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

    Figure 14.  RTP payload format for FU-A
```

FU header 格式如下：

```
      +---------------+
      |0|1|2|3|4|5|6|7|
      +-+-+-+-+-+-+-+-+
      |S|E|R|  Type   |
      +---------------+
```

+ S: start bit, 置一表明这是 NALU 的首个 fragment；
+ E: end bit, 置一表明是 NALU 的最后一个 fragment；
+ R: reserved，必须置零；
+ Type: 取值含义与 NALU header 的 type 字段一致；

## WebRTC H.264 封包实现

了解完了理论部分，接下来我们看看 WebRTC 里是如何实现的，WebRTC 把视频数据封装成 RTP packet 的逻辑在 `RTPSenderVideo::SendVideo` 函数中。

### `RTPSenderVideo::SendVideo`

其实封包的过程，就是计算一帧数据需要封多少个包、每个包放多少载荷，为此我们需要知道各种封包模式下，每个包的最大载荷（包大小减去头部大小）。

首先计算一个包的最大容量，这个容量是指可以用来容纳 RTP 头部和载荷的容量，FEC、重传的开销排除在外：

``` c++
// Maximum size of packet including rtp headers.
// Extra space left in case packet will be resent using fec or rtx.
int packet_capacity = rtp_sender_->MaxRtpPacketSize() - fec_packet_overhead -
                    (rtp_sender_->RtxStatus() ? kRtxHeaderSize : 0);
```

`rtp_sender_->MaxRtpPacketSize` [默认会被设置为 1460](https://webrtc.googlesource.com/src/+/6714bf9f18f2514919f7b0cdc305107076cdc65b/modules/rtp_rtcp/source/rtp_rtcp_impl.cc#116)，[但如果需要发送视频，则会被设置为 1200](https://webrtc.googlesource.com/src/+/6714bf9f18f2514919f7b0cdc305107076cdc65b/media/engine/webrtcvideoengine.cc#1546)。（_why ???_）

接着准备四种包的模板：

+ `single_packet`: 对应 NAL unit 和 STAP-A 的包；
+ `first_packet`: 对应 FU-A 的首个包；
+ `middle_packet`: 对应 FU-A 的中间包；
+ `last_packet`: 对应 FU-A 的最后一个包；

准备过程包括：

+ 在 `RTPSender::AllocatePacket` 里设置 `ssrc` 和 `csrcs` 字段，预留 `AbsoluteSendTime`, `TransmissionOffset` 和 `TransportSequenceNumber` extension 的空间，并且按需设置 `PlayoutDelayLimits` 和 `RtpMid` extension；
+ 在 `RTPSenderVideo::SendVideo` 里设置 `payload_type`, `rtp_timestamp` 和 `capture_time_ms` 字段；
+ 在 `AddRtpHeaderExtensions` 里按需设置 `VideoOrientation`, `VideoContentTypeExtension`, `VideoTimingExtension` 和 `RtpGenericFrameDescriptorExtension` extension；
+ `first_packet`, `middle_packet` 和 `last_packet` 均是拷贝自 `single_packet`，因此代码里只调用了 `AddRtpHeaderExtensions` 设置它们的 extension；

这些模板一是后续封包时可以直接拿来用，二是可以准确地知道 RTP 头部需要多少空间，正如注释所言：

> Simplest way to estimate how much extensions would occupy is to set them.

知道了每种包的头部需要多少空间后，就知道每个包最多可以容纳多少载荷了（为 `RtpPacketizer::PayloadSizeLimits` 的各个字段赋值）：

+ `max_payload_len`，最大载荷可用空间：包的最大容量减去中包头部大小；
+ `single_packet_reduction_len`，封单包时，载荷可用空间还需要在 `max_payload_len` 的基础上打个折扣：单包与中包头部大小之差；即包的最大容量减去单包头部大小；
+ `first_packet_reduction_len`，封多包时，首包载荷可用空间也需要在 `max_payload_len` 的基础上打个折扣：首包与中包头部大小之差；
+ `last_packet_reduction_len`，封多包时，末包载荷可用空间也需要在 `max_payload_len` 的基础上打个折扣：末包与中包头部大小之差；

准备好了模板、知道了 limits 之后，就创建 `RtpPacketizer`，通过其 `NumPackets` 接口得知这一帧图像需要封装为多少个包，再调用其 `NextPacket` 封装每个包。调用 `NextPacket` 之后还不算完，还得调用 `RTPSender::AssignSequenceNumber` 分配序列号，如果需要设置 `VideoTimingExtension`，还得设置 `packetization_finish_time_ms`。最后，就是调用 FEC 处理，或直接调用 `RTPSenderVideo::SendVideoPacket` 发送 RTP 报文了。

视频编码为 H.264 时，`RtpPacketizer` 的实现类是 `RtpPacketizerH264`，接下来，我们就看一下 H.264 的封包逻辑。

### `RtpPacketizerH264`

`RtpPacketizerH264` 构造时会根据 `RTPFragmentationHeader` 的内容，生成 `RtpPacketizerH264::Fragment` 数组 `input_fragments_`，`Fragment` 里面包含了每个 NALU 载荷起始字节的指针、NALU 的长度。

`RTPFragmentationHeader` 其实就是这帧图像里面每个 NALU 的信息：载荷在 buffer 里的 offset、载荷长度。这些信息在编码器输出数据之后解析生成，扫描整个 buffer，查找 NALU start code（`001` 或 `0001`），统计每个 NALU 的 offset 和长度。安卓的实现在 `sdk/android/src/jni/videoencoderwrapper.cc` 的 `VideoEncoderWrapper::ParseFragmentationHeader` 中，iOS 的实现在 `sdk/objc/components/video_codec/nalu_rewriter.cc` 的 `H264CMSampleBufferToAnnexBBuffer` 中。

H.264 规范里定义了一幅图像分片为多个 NALU 的功能，但我观察了一下 iPhone 6 编出来的数据，非关键帧都只有一个 NALU，关键帧有两个 NALU，而且前面都添加了 SPS 和 PPS，所以关键帧会有四个 NALU。

有了 `input_fragments_` 后，就会在 `GeneratePackets` 中遍历之，对每个 `Fragment`，根据 `packetization_mode` 执行不同的封包逻辑：

+ 如果是 `SingleNalUnit`，那就为这个 `Fragment`（其实就是一个 NALU）生成一个 `PacketUnit`；
+ 如果是 `NonInterleaved`（WebRTC Native SDK 实际使用的 mode），那就看这个 `Fragment` 能否放进单个包里，先计算单个包能容纳多少数据：

    ``` c++
    int single_packet_capacity = limits_.max_payload_len;
    if (input_fragments_.size() == 1)
        single_packet_capacity -= limits_.single_packet_reduction_len;
    else if (i == 0)
        single_packet_capacity -= limits_.first_packet_reduction_len;
    else if (i + 1 == input_fragments_.size())
        single_packet_capacity -= limits_.last_packet_reduction_len;
    ```

+ 逻辑并不复杂，`max_payload_len` 扣除各种情况的折扣之后，剩下的就是 `single_packet_capacity`；
+ 如果 `fragment_len > single_packet_capacity`，就说明无法放进单个包，那就要做 Fragmentation 了，即调用 `PacketizeFuA`，否则说明可以放进单个包，那就可以做 Aggregation，即调用 `PacketizeStapA`；
+ `PacketizeFuA` 就是看怎么把一个 `Fragment` 分成多个包了，然后生成每个 `PacketUnit`，这个分的逻辑实现在 `SplitAboutEqually` 函数中，里面处理了不少边界情况，大体思想就是把数据放进尽可能少的包、每个包的大小尽可能相近；它生成的 `PacketUnit` 的 `aggregated` 字段都是 false；
+ `PacketizeStapA` 则是看能把多少个 `Fragment` 放进一个包，这里也会为每个 `Fragment` 生成一个 `PacketUnit`，但只会对 `num_packets_left_` 做一次加一操作；它生成的 `PacketUnit` 除了最后一个的 `aggregated` 字段为 false，其他都为 true；

`GeneratePackets` 执行完毕后，就算出了 `num_packets_left_` 的值，即此帧图像需要多少个 RTP包，并且也准备好了 `PacketUnit` 数组。

之后在 `RTPSenderVideo::SendVideo` 里就会调用 `num_packets_left_` 次 `NextPacket` 来实际组装每一个 RTP 包了，我们现在就看看 `NextPacket` 的逻辑：

+ 检查首个 `PacketUnit`：
+ 如果 `PacketUnit` 的 `first_fragment` 和 `last_fragment` 字段都是 true，那就直接把载荷拷进去；
+ 这种情况有可能是 `SingleNalUnit`，也有可能是 `NonInterleaved` 的 STAP-A 包，因为 `NonInterleaved` 时，如果 `Fragment` 可以放进一个包，那就会封为 STAP-A，而如果只生成了一个 `PacketUnit`，那它的 `first_fragment` 和 `last_fragment` 都会是 true；
+ 否则，如果 `aggregated` 字段为 true，那就调用 `NextAggregatePacket` 封 STAP-A 包；
  - 这里只提一点我看了比较久才看清楚的逻辑：这个函数里通过一个循环不停的消费 `PacketUnit`，退出循环的条件是 `!packet->aggregated` 或 `packet->last_fragment`，由于需要放进一个包的一系列 `PacketUnit` 里只有最后一个 `last_fragment` 字段为 true（这个逻辑在 `PacketizeStapA` 里），因此可以正确退出循环；
+ 如果 `aggregated` 字段为 false，就调用 `NextFragmentPacket` 封 FU-A 包；

---

好了，至此我们就已经看完了 H.264 封装 RTP 包的逻辑，可以长舒一口气了 :)

## WebRTC H.264 解包实现

了解了封包的实现，我们接下来看看解包是怎么实现的，解包比封包稍微复杂一点，关键就在于包的到达可能是乱序的（丢包重传也可以认为是一种乱序）。

解包过程包括两大步：先解析出 RTP 的头部和载荷；再解析载荷部分，根据不同的封包模式，对封包过程做一个逆操作，就能得到一帧完整的数据。前者在 `Call::DeliverRtp` 中调用 `RtpPacket::ParseBuffer` 中实现，后者则比较复杂，因为需要处理乱序问题，逻辑起始点是 `RtpVideoStreamReceiver::ReceivePacket` 函数。

### `RtpPacket::ParseBuffer`

`ParseBuffer` 的任务有三点：

+ 解析 RTP 标准头的各个字段，包括 `payload_type_`, `sequence_number_`, `timestamp_`, `ssrc_` 等；
+ 解析 RTP 扩展头的元数据，即偏移量和长度；
+ 确定 RTP 载荷的偏移量和长度，完成了第二点后做个减法就可以得到；

### `RtpVideoStreamReceiver::ReceivePacket`

首先根据不同的 payload type，创建不同的 `RtpDepacketizer` 去解析载荷内容，H.264 的解析逻辑在 `RtpDepacketizerH264::Parse` 中实现，其主要任务就是找到实际数据的位置和大小：

+ 检查载荷的第一个字节里的 type 字段（低五位），以判断包类型（NAL unit, STAP-A, FU-A）；
+ FU-A 的解析在 `ParseFuaNalu` 里完成；
+ NAL unit 和 STAP-A 的解析在 `ProcessStapAOrSingleNalu` 里完成；
+ _具体解析代码这里不做展开；_

然后解析 RTP 扩展头的实际数据，包括 `VideoOrientation` 等。

最后构造 `VCMPacket`，并调用 `PacketBuffer::InsertPacket` 放入包缓冲区中。

### `PacketBuffer::InsertPacket`

`PacketBuffer` 封装了 RTP 包处理乱序到达的逻辑，大体思路就是：

+ 收到每个包之后，检查序列号：
  - 确定已经收到过的包，就会直接丢弃；
  - 否则就把包放进 `data_buffer_` 数组里，并在 `sequence_buffer_` 数组里记下这个序列号的一些属性；
  - 每个包在上述两个数组里存放的下标是序列号模以数组大小，因此是按序列号顺序存放的；
+ 调用 `FindFrames`，从已收到的包列表中，找出完整的帧；
+ 把完整的帧交给 `RtpFrameReferenceFinder::ManageFrame`，由其确保帧可以解码后，再回调出去，进入后续的解码环节；

### `PacketBuffer::FindFrames`

每次收到包后，会触发 `FindFrames`，我们会从刚收到的包的序列号向后查找：

+ 只有一个包满足以下两个条件之一才会进行检查：
  - 该包是「帧起始」包；
  - 该包前一个序号的包是连续的，何谓连续？就是说它是帧起始包，或它之前的所有序列号都已经收到了；
  - 举个例子，假设 1 是帧起始包，那收到 1 的时候肯定会检查，之后收到 2 时，由于 1 是连续的，所以 2 也会检查，但如果收到 4（假设 4 不是帧起始），那 4 就不会检查，再收到 3 时，就会依次检查 3 和 4；
+ 我们首先感兴趣的是「帧末尾」包，即有 `packet->is_last_packet_in_frame` 标志；
+ 找到帧末尾包后，再反过来向前查找「帧起始」包；
  - VP8/VP9 靠 `frame_begin`（即 `packet->is_first_packet_in_frame`）标志判断帧起始，H.264 则靠时间戳的变化来判断帧起始；
  - 正常情况下，由于我们只检查帧起始包和连续包，所以一旦找到了帧末尾包，向前就一定能找到帧起始包；
+ 找到了帧末尾包和帧起始包，就可以构造完整的帧了，不过这里只是记录元数据，不会做帧数据的拷贝；

### `RtpFrameReferenceFinder::ManageFrame`

从载荷里解析出来的帧数据都是完整的帧，但不一定能解码，比如 H.264 有前向参考（P 帧需要参考前面的 I 帧才能解码），也有后向参考（B 帧需要参考前面的 I/P 帧和后面的 P 帧才能解码），因此需要等这一帧的参考帧都收到之后，才能回调出去。

虽然 `PacketBuffer` 处理了 RTP 报文乱序到达的问题，输出了一个个完整的帧，但并没有保证帧是按序到达的，所以仍需 `RtpFrameReferenceFinder` 来处理帧乱序到达的问题。

_`RtpFrameReferenceFinder` 的代码细节这里就不展开了，有兴趣/需求的朋友可以自行阅读。_

---

好了，至此我们就已经看完了 H.264 解封装 RTP 包的逻辑，可以再长舒一口气了 :)

## WebRTC RTP 封包解包相关数据结构

最后，我们再总结一下 WebRTC RTP 封包解包相关数据结构：

+ `RtpPacket`: RTP 报文的数据结构，里面定义了各种标准头部字段、扩展头部、数据缓冲区等；
+ `RtpPacketToSend`: 发送端封包用到的数据结构，继承自 `RtpPacket`，加了一些扩展头部设置逻辑的封装；
+ `RtpPacketReceived`: 接收端解包用到的数据结构，也继承自 `RtpPacket`，加了获取扩展头部逻辑的封装；

---

最后的最后，我再分享一个内容：序列号的比较算法。

## 序列号的比较算法

由于序列号可能发生回绕，所以不能直接比较，有一个 RFC 文档专门定义了这个比较算法：[Serial Number Arithmetic](https://tools.ietf.org/html/rfc1982)。

这个 RFC 里首先定义了序列号的定义法：n 位无符号数，最低序列号为 0，最高序列号为 `2^n-1`，序列号没有最大最小值，每个序列号至少需要 n 位来保存。

接着它定义了序列号的加法：在 `[0, 2^n-1]` 范围内的合法序列号值 s，加 m 的值为 `(s+m) % (2^n)`，这里的加法和取模，都是常规定义的加法和取模。

最后它定义了序列号的比较算法（RFC 里为了严谨，引入了另外两个普通正整数，这里简单起见我们就不引入了）：

+ 判等：序列号 `s` 和 `s+m`（`m` 为普通正整数），只有 `m` 为 0 时，它们才相等；即给定两个序列号值，完全无法判断其是否相等，不过通常我们不需要判等，而是判断大小；
+ 判小：当且仅当 `(s1 < s2 && s2 - s1 < 2^(n-1)) || (s1 > s2 && s1 - s2 > 2^(n-1))` 时，序列号 `s1` 小于 `s2`；即值小不过一半范围，或大过一半范围，例如 n=3，`2-1 < 4`，故 1 比 2 小，`7-2 > 4`，故 7 比 2 小；
+ 判大：当且仅当 `(s1 < s2 && s2 - s1 > 2^(n-1)) || (s1 > s2 && s1 - s2 < 2^(n-1))` 时，序列号 `s1` 大于 `s2`；即值小过一半范围，或大不过一半范围，例如 n=3，`7-2 > 4`，故 2 比 7 大，`2-1 < 4`，故 2 比 1 大；

细心的朋友也许会举出一个例子：7 和 3 谁大谁小？它们其实无法区分大小。就像 3 和 3 是否相等一样，无法区分。RFC 里故意不对这种序列号对的大小问题作出定义，因为着实不好定义。

WebRTC 的实现逻辑主要在 `rtc_base/numerics/sequence_number_util.h` 和 `rtc_base/numerics/mod_ops.h` 中：

``` cpp
template <typename T, T M = 0>
inline bool AheadOf(T a, T b) {
  static_assert(std::is_unsigned<T>::value,
                "Type must be an unsigned integer.");
  return a != b && AheadOrAt<T, M>(a, b);
}

template <typename T, T M>
inline typename std::enable_if<(M == 0), bool>::type AheadOrAt(T a, T b) {
  static_assert(std::is_unsigned<T>::value,
                "Type must be an unsigned integer.");
  const T maxDist = std::numeric_limits<T>::max() / 2 + T(1);
  if (a - b == maxDist)
    return b < a;
  return ForwardDiff(b, a) < maxDist;
}

template <typename T, T M>
inline typename std::enable_if<(M == 0), T>::type ForwardDiff(T a, T b) {
  static_assert(std::is_unsigned<T>::value,
                "Type must be an unsigned integer.");
  return b - a;
}
```

+ 首先序列号必须是无符号数；
+ 然后 WebRTC 定义了「前向距离」这个概念，即后数加多少能加到前数（考虑无符号数的溢出）；
+ 还定义了「最大距离」这个概念，可以理解为两个数之差的绝对值的最大可能取值，也就是最大取值范围的一半（向上取整）；
+ 最后，a 领先于 b 的的条件就是：若 ab 前向距离为最大距离，那 a 大于 b 就是领先于 b，否则，若 ab 前向距离小于最大距离，那 a 就领先于 b；

_其实就是通过无符号数减法的溢出，把 RFC 定义的两种或起来的情况统一了，以及对于 RFC 未定义的情况，定义成了值大小的比较_。

## 总结

好了，从 18 年 10 月到 19 年，这篇（_又臭又长_）的文章，总算完成了。

RTP 协议 WebRTC 里用到了的部分，以及 WebRTC 封包和解包的代码主要流程，我算是搞清楚了个大概，希望能在你钻研这块代码的过程中有一点助益 :)
