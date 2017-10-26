---
layout: post
title: 我需要知道：RTP
tags:
    - 基础知识
    - 实时多媒体
---

## RTP 包结构

+ 包头有固定 12 个字节部分，以及可选的 `csrc` 和 `ext` 数据；

![](https://imgs.piasy.com/2017-10-19-rtp_header_format.png)

+ 接着是载荷数据，载荷长度在包头中有记录；
+ 载荷数据第一个字节格式和 NAL 头一样，其 type 定义如下：

![](https://imgs.piasy.com/2017-10-19-rtp_h264_header_type.png)

+ 如果 type 为 `[1, 23]`，则该 RTP 包只包含一个 NALU；

![](https://imgs.piasy.com/2017-10-19-rtp_h264_single_nalu.png)

### 包聚合（Aggregation Packets）

为了体现/应对有线网络和无线网络的 MTU 巨大差异，RTP 协议定义了包聚合策略；

+ STAP-A：聚合的 NALU 时间戳都一样，无 DON（decoding order number）；
+ STAP-B：聚合的 NALU 时间戳都一样，有 DON；
+ MTAP16：聚合的 NALU 时间戳不同，时间戳差值用 16 bit 记录；
+ MTAP24：聚合的 NALU 时间戳不同，时间戳差值用 24 bit 记录；
+ 包聚合时，RTP 的时间戳是所有 NALU 时间戳的最小值；

![](https://imgs.piasy.com/2017-10-19-rtp_h264_aggregation_nalu.png)

+ STAP-A 示例：

![](https://imgs.piasy.com/2017-10-19-rtp_h264_stap_a.png)

+ STAP-B 示例：
  - DON 计算规则：[section-5.7.1](https://tools.ietf.org/html/rfc6184#section-5.7.1)

![](https://imgs.piasy.com/2017-10-19-rtp_h264_stap_b.png)

+ MTAP 示例：
  - DON 计算规则、时间戳计算规则：[section-5.7.2](https://tools.ietf.org/html/rfc6184#section-5.7.2)

![](https://imgs.piasy.com/2017-10-19-rtp_h264_mtap.png)

+ MTAP16 示例：

![](https://imgs.piasy.com/2017-10-19-rtp_h264_mtap16-1.png)

+ MTAP24 示例：

![](https://imgs.piasy.com/2017-10-19-rtp_h264_mtap24-1.png)

### 包拆分（Fragmentation Units，FUs）

在应用层实现包拆分而不是依赖下层网络的拆分机制，好处有二：

+ 可以支持超过 64 KB（IPv4 包最大长度为 64 KB）的 NALU，高清视频文件可能有超大的 NALU；
+ 可以利用 FEC（forward error correction）；

每个分包都有一个编号，一个 NALU 拆分的 RTP 包其序列必须顺序且连续，中间不得插入其他数据的 RTP 包序号。FU 只能拆分 NALU，STAP 和 MTAP 不能拆分，FU 也不能嵌套。

FU-A 和 FU-B 的格式如下：

![](https://imgs.piasy.com/2017-10-19-rtp_h264_fu_a.png)

![](https://imgs.piasy.com/2017-10-19-rtp_h264_fu_b.png)

FU indicator 格式和 NAL 头结构一致。

FU header 格式如下：

![](https://imgs.piasy.com/2017-10-19-rtp_h264_fu_header.png)
