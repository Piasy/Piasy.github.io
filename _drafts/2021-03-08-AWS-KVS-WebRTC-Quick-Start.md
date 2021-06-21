---
layout: post
title: AWS KVS WebRTC 快速入门
tags:
    - 实时多媒体
    - WebRTC
---

很早就在群里听说过 [Amazon Kinesis Video Streams C WebRTC SDK](https://github.com/awslabs/amazon-kinesis-video-streams-webrtc-sdk-c)，它是 Amazon 用 C 实现的 WebRTC 协议栈，最大的优势就是库体积小，官方宣称不超过 200KB，对于资源受限的嵌入式场景基本就是不二之选。今天我就给大家带来一份 AWS KVS WebRTC 的快速入门教程。

## 编译、官方 Sample 测试

在国外的服务器上，按照官方 README 的说明，全程无坑。

```bash
git clone --recursive https://github.com/awslabs/amazon-kinesis-video-streams-webrtc-sdk-c.git
mkdir -p amazon-kinesis-video-streams-webrtc-sdk-c/build; cd amazon-kinesis-video-streams-webrtc-sdk-c/build; cmake ..
make

export AWS_ACCESS_KEY_ID= <AWS account access key>
export AWS_SECRET_ACCESS_KEY= <AWS account secret key>
./kvsWebrtcClientMaster myChannel
```

其中 AWS account access key 和 AWS account secret key 在 AWS 控制台的 My Security Credentials 里，创建 Access keys 后可以得到。

`kvsWebrtcClientMaster` 跑起来之后，打开 [WebRTC SDK Test Page](https://awslabs.github.io/amazon-kinesis-video-streams-webrtc-sdk-js/examples/index.html)，输入相同的 Access Key ID、Secret Access Key 和 Channel Name，点击 Start Viewer，等一会儿就能收到 `kvsWebrtcClientMaster` 推过来的视频了。

## 基本调用流程

`kvsWebrtcClientMaster` 会创建 KVS SignalingClient 并连接，收到的消息在 `samples/Common.c signalingMessageReceived` 函数里处理，master 会从 KVS 服务器收到 offer，收到后创建 PC、设置 remote SDP、创建 answer，再交换 ICE candidate，于是 PC 连接就建立起来了。

这个过程调用到的接口有：

+ `initKvsWebRtc`: 全局初始化；
+ `createPeerConnection`: 创建 PC；
+ `peerConnectionOnIceCandidate`: 设置本地 ICE candidate 回调；
+ `peerConnectionOnConnectionStateChange`: 设置 PC 连接状态回调；
+ `addSupportedCodec`: 添加音视频的 codec；
+ `addTransceiver`: 添加音视频的 transceiver；
+ `transceiverOnBandwidthEstimation`: 设置 transceiver 带宽估计回调，可以在回调里调整编码码率，不过 `kvsWebrtcClientMaster` 并未实际处理，只是打印了日志；
+ `setRemoteDescription`: 设置 remote SDP；
+ `setLocalDescription`: 在 `handleOffer` 中有点奇怪的是，还没有创建 answer，就先把这个指针设置为 local SDP 了；不过整个过程也就只有这里调用了一次 `setLocalDescription`，所以应该是 KVS WebRTC 就是需要先 `setLocalDescription`，再 `createAnswer`（或 `createOffer`）；
+ `createAnswer`: 创建 answer，如果远端支持 Trickle Ice，就立即创建 answer，否则在本地 ICE candidate 回调中，确定 ICE candidate 收集完毕后（回调的 `candidateJson` 参数为 `NULL`），再创建 answer；
+ `addIceCandidate`: 添加远端的 ICE candidate；

处理 offer 时（`samples/Common.c handleOffer`），还会创建从文件读取 H264、OPUS 数据并发送的线程，线程函数分别是 `samples/kvsWebRTCClientMaster.c sendVideoPackets` 和 `samples/kvsWebRTCClientMaster.c sendAudioPackets`，其中调用的是 `writeFrame` 接口，通过 transceiver 发送音视频帧。

接收音视频数据的回调则是通过 `transceiverOnFrame` 设置，最终会在 `samples/Common.c sampleFrameHandler` 里处理，不过 `kvsWebrtcClientMaster` 只是在第一帧的时候打了个日志，并未处理接收到的数据。

## 对接其他框架
