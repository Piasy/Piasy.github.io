---
layout: post
title: OWT Server 快速入门
tags:
    - 实时多媒体
    - WebRTC
---

去年做了一个多平台的（Android/iOS/Windows）基于 WebRTC 的多人音视频通话的项目，客户端基于 WebRTC 最新的客户端代码封装业务逻辑，自己写信令对接 SFU，SFU 最初是对接的 Janus Gateway（所以我才会去给 Janus 提 PR），但 Janus 在中弱网下（ping 4% 丢包 100ms RTT）的表现非常差，一直处于卡死状态。

几经波折，前两周我开始对接 OWT（Open WebRTC Toolkit，Intel 团队开源）4.1.1 发布版，对接过程比较流畅，当然并不是没坑，而是我在对接 Licode 时已经趟平了，所以一天就接完了。但测试过程中遇到一个问题，iPhone 8 plus 或 iPhone X 推的流，Google Glass 收起来只有音频没有视频，为了分析这个问题，我开始了 RTFSC :)

## OWT Server 源码编译、部署

由于 GitHub README 的说明太少，我花了一整晚才成功把 v4.2 的源码编译部署起来，在这里手把手分享给大家。

服务器环境：Ubuntu 18.04 x64 和 Ubuntu 16.04 x64 我均成功编译部署。

_注：由于用于测试，所以下面的命令都是以 root 用户执行_。

### 安装 nodejs 环境

``` bash
cd ~ && \
wget https://nodejs.org/dist/v8.15.1/node-v8.15.1-linux-x64.tar.gz && \
mkdir -p /usr/local/lib/nodejs && \
tar xf node-v8.15.1-linux-x64.tar.gz -C /usr/local/lib/nodejs && \
echo 'export PATH=/usr/local/lib/nodejs/node-v8.15.1-linux-x64/bin:$PATH' >> ~/.bashrc && \
source ~/.bashrc
```

### 下载源码

``` bash
apt-get update && apt-get install -y git htop unzip && \
git config --global user.name Piasy && \
git config --global user.email xz4215@gmail.com && \
wget https://github.com/open-webrtc-toolkit/owt-server/archive/v4.2.zip -O owt-server-4.2.zip && \
unzip owt-server-4.2.zip
```

说明：

+ git config 命令设置用户名和邮箱可自行替换，不替换也没问题，这个设置是为了可以成功给 Licode 打 patch；

### 编译 owt-server

``` bash
cd owt-server-4.2 && \
./scripts/installDepsUnattended.sh && \
npm install -g node-gyp graceful-fs grunt-cli && \
./scripts/build.js -t mcu --check
```

说明：

+ GitHub README 上写的是 `-t all`，但对于没有硬件加速需求/环境的情况，这样 build 会失败，对于不需要硬件加速的情况，`-t mcu` 即可；

### 编译 owt-client-javascript

``` bash
cd ~ && \
wget https://github.com/open-webrtc-toolkit/owt-client-javascript/archive/v4.2.zip -O owt-client-javascript-4.2.zip && \
unzip owt-client-javascript-4.2.zip && \
cd owt-client-javascript-4.2/scripts/ && \
npm install && \
grunt
```

说明：

+ 这个项目里包含了官方 demo，v4.2 这个 tag 的代码没毛病，直接用即可；
+ 对于 AWS 环境，最开始通过 `sudo su` 切换到 root 用户后，执行 `npm install` 可能会提示 `git clone` 权限不足，这时执行一下 `sudo -s`，再重试 `npm install && grunt` 即可；

### 打包 owt-server

``` bash
cd ~/owt-server-4.2/ && \
./scripts/pack.js -t all -f -a -s ~/owt-client-javascript-4.2/dist/samples/conference/
```

说明：

+ pack 成功后，会在当前目录生成 dist 目录，其中就是我们后面运行的内容了；
+ 之后如果修改了代码，需要重新 build pack，那需要先删掉 dist 目录，再重新 build pack；

### 配置 owt-server

配置、运行都是在 dist 目录下，为了运行 SFU，我们需要修改以下两处配置：

+ 编辑 `webrtc_agent/agent.toml`：修改 `[webrtc]` 部分的 `network_interfaces`，添加 `{name = "eth2", replaced_ip_address = "192.0.2.2"}`（需要把 `name` 设置为网卡实际名称，`replaced_ip_address` 设置为服务器公网 IP 地址）, `maxport`, `minport`；注意配置文件里 max 在前，min 在后，别配反了；
+ 编辑 `portal/portal.toml`：修改 `[portal]` 部分里的 `ip_address` 为服务器公网 IP 地址，`ssl` 按需设置为 true 或 false；

服务器开放 TCP 3000~3004, 8080 端口，UDP minport~maxport。

### 运行 owt-server

``` bash
cd ~/owt-server-4.2/dist && \
./bin/init-all.sh && \
./bin/start-all.sh
```

如果上述过程没有发生任何错误，那么恭喜你，owt-server 就成功运行起来了，如果配置时 ssl 为 true，那可以在浏览器里访问 `https://<ip>:3004/`，浏览器会推流，并且会收 MCU 合成的流，访问 `https://<ip>:3004/?forward=true` 则不会收合成流，而是收原流。

## OWT Server 关键逻辑流程

编译部署只是第一步，为了分析问题，我们还得搞清楚关键的逻辑流程：publish, subscribe, 音视频数据。

_owt-server 所有的源码都在 source 子目录，下文路径都已省略_。

### 信令处理

信令（SocketIO）监听代码在 `portal/socketIOServer.js` 和 `portal/v10Client.js` 中，收到消息后，都将调用 `portal/portal.js` 的函数，最后通过 RabbitMQ 发消息给 conference agent 执行实际逻辑。

`common/amqp_client.js` 和 `common/rpcChannel.js` 对 RabbitMQ 的 pub-sub 模式做了一个封装，提供了一套 RPC 的通讯方式：server 端（controller）定义一个 `rpcAPI` 字典，然后调用 `rpc.asRpcServer`，就能开始处理调用请求，client 端则通过 `rpcChannel.makeRPC` 发起调用请求，第二个参数就是方法名，也就是 `rpcAPI` 的 key。例如 `agent/conference/conference.js` 里定义了 join, publish, subscribe 等各种函数，并在 `agent/workingNode.js` 里调用了 `rpc.asRpcServer`，则 `portal/rpcRequest.js` 里发出的相应请求就会在 `agent/conference/conference.js` 里进行处理。`rpcAPI` 字典也不是必须的，如果不定义，那就直接调用 controller 的同名函数。

### 主要事件的处理流程

客户端发出 publish 事件发起推流，发出 subscribe 事件发起收流，发出 soac 事件上传 offer 和 ICE candidate，服务端则通过 progress 事件下发 answer。

+ publish:

  ```
  portal/v10Client.js ->
  portal/portal.js publish ->
  agent/conference/conference.js publish ->
  agent/conference/accessController.js initiate ->
  agent/webrtc/index.js publish, 创建 WrtcConnection
  ```

+ subscribe:

  ```
  portal/v10Client.js ->
  portal/portal.js subscribe ->
  agent/conference/conference.js subscribe ->
  agent/conference/accessController.js initiate ->
  agent/webrtc/index.js subscribe, 创建 WrtcConnection
  ```

+ soac:

  ```
  portal/v10Client.js ->
  portal/portal.js onSessionSignaling ->
  agent/conference/conference.js onSessionSignaling ->
  agent/conference/accessController.js onSessionSignaling ->
  agent/webrtc/index.js onSessionSignaling ->
  agent/webrtc/wrtcConnection.js onSignalling, 调用 C++ 接口
  ```

+ progress (answer):

  ```
  WebRtcConnection::notifyEvent (answer) ->
  agent/webrtc/wrtcConnection.js wrtc.init 回调 ->
  agent/webrtc/index.js notifyStatus ->
  agent/conference/conference.js onSessionProgress ->
  agent/conference/accessController.js onSessionStatus ->
  agent/conference/accessController.js onSignaling ->
  agent/conference/conference.js onLocalSessionSignaling ->
  agent/conference/participant.js notify ->
  portal/socketIOServer.js notify, 发送 SocketIO 事件给客户端
  ```

### subscribe 建立数据转发关系

subscribe 时，先建立好收流端和服务端的 PC 连接，连接成功后，C++ 层会抛出 ready 事件到 JS 层：

```
WebRtcConnection::notifyEvent (ready) ->
agent/webrtc/wrtcConnection.js wrtc.init 回调 ->
agent/webrtc/index.js notifyStatus ->
agent/conference.js onSessionProgress ->
agent/conference/accessController.js onSessionStatus ->
agent/conference.js onSessionEstablished
```

之后，服务端建立推流端的数据到收流端的转发关系：

```
agent/conference.js onSessionEstablished ->
agent/conference.js addSubscription ->
agent/conference/roomController.js subscribe ->
agent/conference/roomController.js spreadStream ->
agent/conference/roomController.js linkup ->
agent/webrtc/index.js linkup ->
agent/connections.js linkupConnection ->
agent/webrtc/wrtcConnection.js addDestination ->
NAN_METHOD(AudioFrameConstructor::addDestination), NAN_METHOD(VideoFrameConstructor::addDestination) ->
FrameSource::addAudioDestination, FrameSource::addVideoDestination
```

### 音视频数据转发

推流端的数据会回调到 Audio/VideoFrameConstructor，进而到 FrameSource，最后转发给 Audio/VideoFramePacketizer，发送给收流端。

```
AudioFrameConstructor::deliverAudioData_, VideoFrameConstructor::Decode ->
FrameSource::deliverFrame ->
AudioFramePacketizer::onFrame, VideoFramePacketizer::onFrame
```

## 总结

有了上述基础后，就可以开始加日志排查我最初的问题了，然而当我部署好之后请人测试时，却发现问题不复现了，看来是 4.1.1 到 4.2 之间的某些修改解决了我的问题 :)

最后，上述源码分析并不全面，是出于排查问题的目的进行的，分享出来权当抛转好了。
