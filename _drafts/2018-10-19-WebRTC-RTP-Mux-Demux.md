---
layout: post
title: WebRTC Native 源码导读（十六）：RTP H.264 封装与解封装
tags:
    - 实时多媒体
    - WebRTC
    - 网络
---

之前我在[为 janus-pp-rec 增加视频旋正功能](/2018/10/08/Janus-pp-rec-video-rotation/#rtp-报文格式简介)一文中简单介绍了一点 RTP 协议的内容，重点关注的是视频方向的 RTP header extension，这次我们更深入的了解一下 RTP 协议的内容，看看 H.264 视频数据是如何封装和解封装的。

_其他视频编码格式的封装和解封装大体思路应该是一样的，大家可以举一反三；音频数据的封装和解封装，日后可能我会再整理一篇文章_。

## 再谈 RTP 协议

我们首先了解一下 RTP H.264 相关的 RFC，下面的内容是对两篇 RFC 的总结：[RTP: A Transport Protocol for Real-Time Applications](https://tools.ietf.org/html/rfc3550), [RTP Payload Format for H.264 Video](https://tools.ietf.org/html/rfc6184)。

### RTP 包结构

包头有固定 12 个字节部分，以及可选的 `csrc` 和 `ext` 数据（在[为 janus-pp-rec 增加视频旋正功能](/2018/10/08/Janus-pp-rec-video-rotation/#rtp-报文格式简介)一文中有更详细的介绍）：

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

## WebRTC RTP 封包解包相关数据结构

RtpPacket, RtpPacketReceived, RtpPacketToSend...

## WebRTC H.264 封包实现

了解完了理论部分，接下来我们看看 WebRTC 里是如何实现的，WebRTC 把视频数据封装成 RTP packet 的逻辑在 `RTPSenderVideo::SendVideo` 函数中。

### `RTPSenderVideo::SendVideo`

首先计算一个 packet 的最大容量，这个容量是指可以用来容纳 RTP header 和 payload 的容量，FEC、重传的开销排除在外：

``` c++
  // Maximum size of packet including rtp headers.
  // Extra space left in case packet will be resent using fec or rtx.
  int packet_capacity = rtp_sender_->MaxRtpPacketSize() - fec_packet_overhead -
                        (rtp_sender_->RtxStatus() ? kRtxHeaderSize : 0);
```

`rtp_sender_->MaxRtpPacketSize` [默认会被设置为 1460](https://webrtc.googlesource.com/src/+/6714bf9f18f2514919f7b0cdc305107076cdc65b/modules/rtp_rtcp/source/rtp_rtcp_impl.cc#116)，[但如果需要发送视频，则会被设置为 1200](https://webrtc.googlesource.com/src/+/6714bf9f18f2514919f7b0cdc305107076cdc65b/media/engine/webrtcvideoengine.cc#1546)。

接着准备四种 packet 的模板：

+ `single_packet`: 对应 NAL unit 和 STAP-A 的 packet；
+ `first_packet`: 对应 FU-A 的首个 packet；
+ `middle_packet`: 对应 FU-A 的中间 packet；
+ `last_packet`: 对应 FU-A 的最后一个 packet；

准备过程包括：

+ 在 `RTPSender::AllocatePacket` 里设置 `ssrc` 和 `csrcs` 字段，预留 `AbsoluteSendTime`, `TransmissionOffset` 和 `TransportSequenceNumber` extension 的空间，并且按需设置 `PlayoutDelayLimits` 和 `RtpMid` extension；
+ 在 `RTPSenderVideo::SendVideo` 里设置 `payload_type`, `rtp_timestamp` 和 `capture_time_ms` 字段；
+ 在 `AddRtpHeaderExtensions` 里按需设置 `VideoOrientation`, `VideoContentTypeExtension`, `VideoTimingExtension` 和 `RtpGenericFrameDescriptorExtension` extension；
+ `first_packet`, `middle_packet` 和 `last_packet` 均是拷贝自 `single_packet`，因此代码里只调用了 `AddRtpHeaderExtensions` 设置它们的 extension；

这些模板一是后续封包时可以直接拿来用，二是可以准确地知道 RTP header 需要多少空间，正如注释所言：

> Simplest way to estimate how much extensions would occupy is to set them.

知道了每种包的 header 需要多少空间后，就可以为 `RtpPacketizer::PayloadSizeLimits` 的各个字段赋值了：

+ `max_payload_len`: 最大 payload 可用空间，`packet_capacity - middle_packet->headers_size()`；
+ `single_packet_reduction_len`: 只需要封一个包时，payload 可用空间还需要在 `max_payload_len` 的基础上打个折扣，`single_packet->headers_size() - middle_packet->headers_size()`；
+ `first_packet_reduction_len`: 需要封多个包时，首包 payload 可用空间也需要在 `max_payload_len` 的基础上打个折扣，`first_packet->headers_size() - middle_packet->headers_size()`；
+ `last_packet_reduction_len`: 需要封多个包时，末包 payload 可用空间也需要在 `max_payload_len` 的基础上打个折扣，`last_packet->headers_size() - middle_packet->headers_size()`；

准备好了模板、知道了 limits 之后，就创建 `RtpPacketizer`，通过其 `NumPackets` 接口得知这一帧图像需要封装为多少个 packet，再调用其 `NextPacket` 封装每个 packet。调用 `NextPacket` 之后还不算完，还得调用 `RTPSender::AssignSequenceNumber` 分配序列号，如果需要设置 `VideoTimingExtension`，还得设置 `packetization_finish_time_ms`。最后，就是调用 FEC 处理，或直接调用 `RTPSenderVideo::SendVideoPacket` 发送 RTP 报文了。

视频编码为 H.264 时，`RtpPacketizer` 的实现类是 `RtpPacketizerH264`，接下来，我们就看一下 H.264 的封包逻辑。

### `RtpPacketizerH264`

`RtpPacketizerH264` 构造时会根据 `RTPFragmentationHeader` 的内容，生成 `RtpPacketizerH264::Fragment` 数组 `input_fragments_`，`Fragment` 里面包含了每个 NALU payload 起始字节的指针、NALU 的长度。

`RTPFragmentationHeader` 其实就是这帧图像里面每个 NALU 的信息：payload 在 buffer 里的 offset、payload 长度。这些信息在编码器输出数据之后解析生成，扫描整个 buffer，查找 NALU start code（`001` 或 `0001`），统计每个 NALU 的 offset 和长度。安卓的实现在 `sdk/android/src/jni/videoencoderwrapper.cc` 的 `VideoEncoderWrapper::ParseFragmentationHeader` 中，iOS 的实现在 `sdk/objc/components/video_codec/nalu_rewriter.cc` 的 `H264CMSampleBufferToAnnexBBuffer` 中。

H.264 规范里定义了一幅图像分片为多个 NALU 的功能，但我观察了一下 iPhone 6 编出来的数据，非关键帧都只有一个 NALU，关键帧有两个 NALU，而且前面都添加了 SPS 和 PPS，所以会有四个 NALU。

有了 `input_fragments_` 后，就会在 `GeneratePackets` 中遍历之，对每个 `Fragment`，根据 `packetization_mode` 执行不同的封包逻辑：

+ 如果是 `SingleNalUnit`，那就为这个 `Fragment`（其实就是一个 NALU）生成一个 `PacketUnit`；
+ 如果是 `NonInterleaved`（WebRTC Native SDK 实际使用的 mode），那就看这个 `Fragment` 能否放进单个 packet 里，先计算单个 packet 能容纳多少数据：

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
+ 如果 `fragment_len > single_packet_capacity`，就说明无法放进单个 packet，那就要做 Fragmentation 了，即调用 `PacketizeFuA`，否则说明可以放进单个 packet，那就可以做 Aggregation，即调用 `PacketizeStapA`；
+ `PacketizeFuA` 就是看怎么把一个 `Fragment` 分成多个 packet 了，然后生成每个 `PacketUnit`，这个分的逻辑实现在 `SplitAboutEqually` 函数中，里面处理了不少边界情况，大体思想就是把数据放进尽可能少的 packet、每个 packet 的大小尽可能相近；它生成的 `PacketUnit` 的 `aggregated` 字段都是 false；
+ `PacketizeStapA` 则是看能把多少个 `Fragment` 放进一个 packet，这里也会为每个 `Fragment` 生成一个 `PacketUnit`，但只会对 `num_packets_left_` 做一次加一操作；它生成的 `PacketUnit` 除了最后一个的 `aggregated` 字段为 false，其他都为 true；

`GeneratePackets` 执行完毕后，就算出了 `num_packets_left_` 的值，即此帧图像需要多少个 RTP packet，并且也准备好了 `PacketUnit` 数组。

之后在 `RTPSenderVideo::SendVideo` 里就会调用 `num_packets_left_` 次 `NextPacket` 来实际组装每一个 RTP packet 了，我们现在就看看 `NextPacket` 的逻辑：

+ 检查首个 `PacketUnit`：
+ 如果 `PacketUnit` 的 `first_fragment` 和 `last_fragment` 字段都是 true，那就直接把 payload 拷进去；
+ 这种情况有可能是 `SingleNalUnit`，也有可能是 `NonInterleaved` 的 STAP-A 包，因为 `NonInterleaved` 时，如果 `Fragment` 可以放进一个 packet，那就会封为 STAP-A，而如果只生成了一个 `PacketUnit`，那它的 `first_fragment` 和 `last_fragment` 都会是 true；
+ 否则，如果 `aggregated` 字段为 true，那就调用 `NextAggregatePacket` 封 STAP-A 包；
  - 这里只提一点我看了比较久才看清楚的逻辑：这个函数里通过一个循环不停的消费 `PacketUnit`，退出循环的条件是 `!packet->aggregated` 或 `packet->last_fragment`，由于需要放进一个 packet 的一系列 `PacketUnit` 里只有最后一个 `last_fragment` 字段为 true（这个逻辑在 `PacketizeStapA` 里），因此可以正确退出循环；
+ 如果 `aggregated` 字段为 false，就调用 `NextFragmentPacket` 封 FU-A 包；

---

好了，至此我们就已经看完了 H.264 封装 RTP packet 的逻辑，可以长舒一口气了 :)

## WebRTC H.264 解包实现

了解了封包的实现，我们接下来看看解包是怎么实现的，解包比封包稍微复杂一点，关键就在于 packet 的到达可能是乱序的，甚至会发生丢包重传，

WebRTC 解封装视频数据 RTP packet 的逻辑在 `RtpVideoStreamReceiver::ReceivePacket` 和 `PacketBuffer::FindFrames` 函数中，前者负责。

### `RtpPacket::ParseBuffer`

### `RtpVideoStreamReceiver::ReceivePacket`

### `PacketBuffer::FindFrames`
