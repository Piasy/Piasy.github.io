---
layout: post
title: WebRTC Native 源码导读（十二）：P2P 连接过程完全解析
tags:
    - 实时多媒体
    - WebRTC
    - 网络
---

一年前我初步分析了 WebRTC 的 P2P 连接过程，并总结为了[安卓 P2P 连接过程和 DataChannel 使用](/2017/08/30/WebRTC-P2P-part1/)一文，那会儿我刚接触 WebRTC C++ 的代码，看起来着实头大，而且安卓的代码要调试、测试也很麻烦，所以很多细节就没有展开，今天就让我们在 iOS 的工程里，对 P2P 连接的过程进行一个彻底的剖析。

## 概览

首先我们从宏观上了解一下 P2P 连接的过程，以及一些关键类之间的关系，这样在看代码时就不至于迷失在细节里。此外，没看过[安卓 P2P 连接过程和 DataChannel 使用](/2017/08/30/WebRTC-P2P-part1/)的朋友，也建议先看一下。

_注：除非你对这个话题有很大兴趣，否则很可能无法读完，那我建议尽早放弃；如果确实需要研究这块内容，那我建议打开源码，反复阅读此文，应当会有些收获_。

### 宏观流程

+ 设置 local sdp；
+ 创建一个 transport 对象（启用了 bundle）；
+ 收集 local candidates；
+ 设置 remote sdp，添加 remote candidates；
+ ICE 连通性检查，建立 P2P 连接；
+ P2P 数据传输；

### P2P 关键类

+ PeerConnection: WebRTC 核心类；
+ JsepTransportController: 管理 P2P 连接；
+ 各种 transport 类：P2P 连接的封装，封装了加解密、mux/demux 等逻辑，提供收发数据的接口；
+ BasicPortAllocator, PortAllocator: 保存各种配置，管理 PortAllocatorSession；
+ BasicPortAllocatorSession, PortAllocatorSession: 遍历所有网络设备（Network 对象），分配 port；
+ AllocationSequence: 负责对单个网络设备（Network 对象）分配 port，分阶段进行；
+ 各种 port 类：代表的是一种通讯机制的本地实例，它可以和远端的类似实例一起实现数据通讯；
+ Connection: 代表的是一个 local port 和一个 remote port 的通讯链接；

_注：列在同一点里的类，是继承关系，左侧是子类，右侧是基类，下同_。

### 各种 transport 类的关系

+ PacketTransportInternal, PacketTransportInterface: packet base transport 的基类；
+ P2PTransportChannel, IceTransportInternal: 继承自 PacketTransportInternal，负责 ICE 相关的功能，包括：收集本地 candidate、和远端 candidate 做连通性检查、数据传输；
+ DtlsTransport, DtlsTransportInternal: 继承自 PacketTransportInternal，负责 DTLS 相关的功能，包括：DTLS 握手、DTLS 加解密；内含一个 IceTransportInternal (P2PTransportChannel)，收发数据通过它实现；
  - 不加密时，写入 DtlsTransport 的数据，直接交给 P2PTransportChannel；
  - 加密时，数据会交给 SSLStreamAdapter -> StreamInterfaceChannel -> P2PTransportChannel，这是为了桥接加解密的流式接口与 DtlsTransport 的包式接口；
+ RtpTransport, RtpTransportInternal, SrtpTransportInterface, RtpTransportInterface: 提供了收发 RTP, RTCP 包的接口，其内部包了两个实际收发 RTP 和 RTCP 数据的 PacketTransportInternal (DtlsTransport)；
+ DtlsSrtpTransport, SrtpTransport: DTLS-SRTP, SRTP 的实现类，继承自 RtpTransport； _有了 DTLS，为何还要 SRTP？_
+ JsepTransport: JsepTransportController 管理 transport 的辅助类，sdp 里每个 m line 都对应于一个数据流（音频、视频、应用数据），每个数据流都需要一个 transport，但可以通过 bundle 技术复用同一个 transport，m line 里的 attribute 描述了 transport 的属性；
  - 根据 transport 的加密属性，构造它时会准备无加密的 RtpTransport，或 SDES 加密的 SrtpTransport，或 DTLS 加密的 DtlsSrtpTransport；这一逻辑在 `JsepTransportController::MaybeCreateJsepTransport` 函数里；
  - transport 对象将在设置 sdp 时创建，一个 transport 对象将会对应于一个最终的 P2P 网络连接（socket）；

### 关键类的数量关系

一个 PeerConnection - 一个 JsepTransportController - 一个 JsepTransport（启用了 bundle） - 一个 DtlsSrtpTransport - 一个 DtlsTransport - 一个 P2PTransportChannel。

一个 JsepTransportController - 一个 BasicPortAllocator - 多个 BasicPortAllocatorSession，但一次分配过程只会有一个 session。

一个 BasicPortAllocatorSession - 多个 AllocationSequence。

一个 AllocationSequence - 多个 port。

一个 P2PTransportChannel - 多个 Connection，但最终会选出一个 Connection 使用。

---

接下来我们就对宏观过程的代码细节进行展开。

_再次预警，如果此时你已经有些倦意，那我建议立刻关闭这个页面_。

## 收集本地 candidate

设置 local sdp，开始收集 candidate：

```
PeerConnection::SetLocalDescription
                ↓
JsepTransportController::MaybeStartGathering
                ↓
P2PTransportChannel::MaybeStartGathering
                ↓
BasicPortAllocatorSession::StartGettingPorts
                ↓
BasicPortAllocatorSession::DoAllocate
```

### DoAllocate

DoAllocate 里会遍历所有网络设备（Network 对象），创建 AllocationSequence 对象，调用其 Init Start 函数，分配 port。

```
BasicPortAllocatorSession::DoAllocate
              ↓
AllocationSequence::Start
              ↓ message
AllocationSequence::OnMessage
```

AllocationSequence 分配 port 分为三个 phase：UDP, RELAY, TCP。每个 phase 之间间隔一个 step delay。_一年前我在分析[安卓 P2P 连接过程和 DataChannel 使用](/2017/08/30/WebRTC-P2P-part1/)时还有一个 SslTcp phase，现在已经删掉了_。

### UDP phase

UDP phase 会收集两种类型的 candidate：host 和 srflx。

#### host candidate

一旦创建了 AsyncPacketSocket 对象，有了本地 IP 和端口，host 类型的 candidate 也就已经就绪了，而 AsyncPacketSocket 对象在 `AllocationSequence::Init` 里就已经创建好了，所以可以直接发出 host candidate。

```
AllocationSequence::OnMessage
              ↓
AllocationSequence::CreateUDPPorts
              ↓
BasicPortAllocatorSession::AddAllocatedPort
              ↓
UDPPort::PrepareAddress
              ↓
UDPPort::OnLocalAddressReady
              ↓
      Port::AddAddress
              ↓ sig slot (SignalCandidateReady)
BasicPortAllocatorSession::OnCandidateReady
```

#### srflx candidate

收集 srflx candidate 的原理是，向 STUN server 发送一个 UDP 包（叫 STUN Binding request），server 会把这个包里的源 IP 地址、UDP 端口返回给客户端（叫 STUN Binding response），这个 IP 和端口将来可能可以用来和其他客户端建立 P2P 连接。关于 STUN 协议的具体内容，可以查阅 [RFC Session Traversal Utilities for NAT (STUN)](https://tools.ietf.org/html/rfc5389)。

收集 srflx candicate 时可以复用收集 host candidate 时创建的 socket 对象，这一逻辑通过 `PORTALLOCATOR_ENABLE_SHARED_SOCKET` flag 控制，默认是开启的。

复用 socket 的情况下，`AllocationSequence::CreateStunPorts` 函数会直接返回，因为早在 `AllocationSequence::CreateUDPPorts` 函数的执行过程中，就已经执行了 STUN Binding request 的发送逻辑。

发送 STUN Binding request：

```
UDPPort::OnLocalAddressReady
            ↓
UDPPort::MaybePrepareStunCandidate
            ↓
UDPPort::SendStunBindingRequest
            ↓
StunRequestManager::SendDelayed
            ↓ message
StunRequest::OnMessage
            ↓ sig slot (SignalSendPacket)
UDPPort::OnSendPacket
            ↓
AsyncUDPSocket::SendTo
            ↓
PhysicalSocket::SendTo
            ↓
系统 socket sendto
```

收到 STUN Binding response：

```
PhysicalSocketServer::WaitSelect
                ↓
SocketDispatcher::OnEvent
                ↓ sig slot (SignalReadEvent)
AsyncUDPSocket::OnReadEvent
                ↓ sig slot (SignalReadPacket)
AllocationSequence::OnReadPacket
                ↓
UDPPort::HandleIncomingPacket
                ↓
StunRequestManager::CheckResponse
                ↓
StunBindingRequest::OnResponse
                ↓
UDPPort::OnStunBindingRequestSucceeded
                ↓
        Port::AddAddress
                ↓ sig slot (SignalCandidateReady)
BasicPortAllocatorSession::OnCandidateReady
```

### RELAY phase

WebRTC 目前支持两种中继协议：GTURN 和 TURN。现在基本都是使用标准的 TURN 协议。TURN 协议是 STUN 协议的一个扩展，它利用一个中继服务器，使得无法建立 P2P 连接的客户端（NAT 严格限制导致）也能实现通讯。_关于 NAT 类型与 P2P 连接的可行性，可参考[附录一：NAT 类型与 P2P 连接的可行性](#附录一nat-类型与-p2p-连接的可行性)_。

TURN 协议的工作流程如下：

客户端发送 Allocate request 到 server，server 返回 401 未授权错误（带有 realm 和 nonce），客户端再发送带上认证信息的 Allocate request，server 返回成功分配的 relay address。分配成功后，客户端需要通过发送机制（Send Mechanism）或信道机制（Channels）在 server 上配置和其他 peer 的转发信息。此外 allocation 和 channel 都需要保活。

WebRTC 使用的是信道机制，因为这一机制的数据开销更低。

收集 TURN relay candidate 时也可以复用收集 host candidate 时创建的 socket 对象，这一逻辑通过 `PORTALLOCATOR_ENABLE_SHARED_SOCKET` flag 控制，前面我们就已经知道，默认情况下它是开启的。

由于 TURN 协议是 STUN 协议的扩展，所以基本的发送请求、接收响应的代码是复用的，下面只描述 TURN 协议独特的部分：

```
AllocationSequence::OnMessage
              ↓
AllocationSequence::CreateRelayPorts
              ↓
BasicPortAllocatorSession::AddAllocatedPort
              ↓
TurnPort::PrepareAddress
              ↓
TurnPort::SendRequest
              ↓ message
StunRequest::OnMessage
              ↓ 发送请求、接收响应
StunRequestManager::CheckResponse
              ↓
TurnAllocateRequest::OnErrorResponse
              ↓
TurnAllocateRequest::OnAuthChallenge
              ↓
TurnPort::SendRequest
              ↓ 发送请求、接收响应
StunRequestManager::CheckResponse
              ↓
TurnAllocateRequest::OnResponse
              ↓
TurnPort::OnAllocateSuccess
              ↓
        Port::AddAddress
              ↓ sig slot (SignalCandidateReady)
BasicPortAllocatorSession::OnCandidateReady
```

### StunRequest 和 StunRequestManager

上面提到的 STUN Binding, TURN Allocate 的 request response 处理，以及后面要遇到的 STUN ping request response 处理，都由 StunRequest 和 StunRequestManager 类负责。

StunRequest 类是对 STUN request 的定义和封装，基类里实现了 request 超时管理、重发的逻辑，各种特定类型的逻辑由子类实现，例如 StunBindingRequest 和 TurnAllocateRequest。

StunRequestManager 则实现了 response 和 request 匹配的逻辑：manager 按 transaction id => request 的 hash 保存了所有的 request，收到 response 后，根据 transaction id 即可找到对应的 request，进而可以执行 request 对象的回调。

### TCP phase

TCP phase 不是主流，这里就不展开了。

### OnCandidateReady

OnCandidateReady 里会调用两个重要的函数（通过 sig slot）：`P2PTransportChannel::OnPortReady` 和 `P2PTransportChannel::OnCandidatesReady`。

在 OnPortReady 里，P2PTransportChannel 会把 port 存入 `ports_` 数组，供后续收到 remote candidate 后建立 Connection 用。此外，这里也会立即尝试用这个 port 和每个远端 candidate 建立 Connection。

在 OnCandidatesReady 里，P2PTransportChannel 会把 candidate 一路回调给 APP 层：

```
P2PTransportChannel::OnCandidatesReady
                  ↓
JsepTransportController::OnTransportCandidateGathered_n
                  ↓
PeerConnection::OnTransportControllerCandidatesGathered
                  ↓
PeerConnection::OnIceCandidate
                  ↓
PeerConnectionObserver::OnIceCandidate
```

这里我们看到，port 和 candidate 都送到了 P2PTransportChannel 这里，因为它就是对 ICE 逻辑的封装，接下来我们很快会看到，remote candidate 也是交给了 P2PTransportChannel。

## 设置远端 candidate

收到远端的 candidate 后，我们调用 `PeerConnection::AddIceCandidate` 接口进行设置，其内部调用栈为：

```
PeerConnection::AddIceCandidate
              ↓
JsepTransportController::AddRemoteCandidates
              ↓
JsepTransport::AddRemoteCandidates
              ↓
P2PTransportChannel::AddRemoteCandidate
              ↓
P2PTransportChannel::CreateConnections
```

CreateConnections 会遍历本地所有的 port（在 OnCandidateReady 中保存），尝试与这个远端 candicate 建立连接。本文中我们只分析了 UDP 和 TURN 两种 port，所以会调用到 `UDPPort::CreateConnection` 和 `TurnPort::CreateConnection` 创建 Connection。

创建了 Connection 之后，怎么做连通性检查呢？这就是 ICE 协议定义的内容了。

## ICE 连通性检查

关于 ICE 连通性检查的介绍，可以阅读[「安卓 P2P 连接过程和 DataChannel 使用」的「连通性检查」部分](/2017/08/30/WebRTC-P2P-part1/#连通性检查)，当然，最好是看看 ICE 协议的 RFC 了：[Interactive Connectivity Establishment (ICE): A Protocol for Network Address Translator (NAT) Traversal for Offer/Answer Protocols](https://tools.ietf.org/html/rfc5245)。

前面我们提到，在 OnCandidateReady 里我们会用刚分配好的 Port 与每个 remote candidate 建立 Connection，此外，如果收到了对方的 STUN ping request，那就会立即创建一个 Connection，再加上添加 remote candidate 的情况，这三种情况下，创建完 Connection 之后，P2PTransportChannel 都会立即执行 SortConnectionsAndUpdateState 函数，其中首先会对 Connection 进行排序（见下文），此外也会尝试开始 ping Connection。

STUN ping request 其实就是 STUN binding request，所以它的发送、response 的接收，前面都已经分析过了，这里只展示不同的部分：

```
P2PTransportChannel::AddRemoteCandidate
                  ↓
P2PTransportChannel::SortConnectionsAndUpdateState
                  ↓
P2PTransportChannel::MaybeStartPinging
                  ↓
P2PTransportChannel::PingConnection
                  ↓
StunRequestManager::SendDelayed
                  ↓ 发送请求、接收响应
Connection::OnConnectionRequestResponse
                  ↓
Connection::set_write_state
                  ↓ sig slot (SignalStateChange)
P2PTransportChannel::OnConnectionStateChange
                  ↓
P2PTransportChannel::RequestSortAndStateUpdate
                  ↓ message
P2PTransportChannel::SortConnectionsAndUpdateState
```

创建 Connection 会触发 ping，ping 成功后会触发 Connection 的状态切换（见下文）。排序后，我们最终会选出一个合适的 Connection，通知上层可以进行数据通讯了。

```
P2PTransportChannel::SwitchSelectedConnection
                    ↓ sig slot (SignalReadyToSend)
DtlsTransport::OnReadyToSend
                    ↓ sig slot (SignalReadyToSend)
RtpTransport::OnReadyToSend
```

在这个过程中，Connection 的状态如何变迁？ICE 状态如何变迁？Connection 如何排序？如何选择？接下来我们就仔细展开分析。

### Connection 状态变迁

因为 Connection 的排序用到了状态，所以我们就先搞清楚 Connection 的状态及其变迁：

+ `write_state_`：一共有四种状态，取值依次变大，其切换逻辑在 `Connection::ReceivedPingResponse` 和 `Connection::UpdateState` 中：
  - `STATE_WRITABLE`：只要收到 ping response 就会切换到这个状态；
  - `STATE_WRITE_UNRELIABLE`：当前处于 WRITABLE 状态，且最近发生过一定量的 ping 失败/超时，则切换到此状态；
  - `STATE_WRITE_INIT`：初始化状态；
  - `STATE_WRITE_TIMEOUT`：如果当前处于 UNRELIABLE 或 INIT 状态，且最近超过一定时间没有收到 response，则切换到此状态；
+ writable: `write_state_ == STATE_WRITABLE`，即最近成功收到了 ping response；
+ PresumedWritable: 如果配置允许，就认为处于 INIT 状态、local candidate 是 relay 类型、remote candidate 是 relay 或 prflx 类型的连接也 writable；这里也允许 prflx 的原因是，prflx 是从收到的包里看到的地址，这个地址在各种 NAT 类型下都是可以发包的；
+ receiving: 上次发送的 ping 收到了响应，则认为处于 receiving 状态；否则如果此 Connection 上次收到数据（data, ping request, ping response）的时间距离现在小于阈值，则也认为处于 receiving 状态；
+ connected: UDP 一直都是 connected 状态，TCP 三次握手完成后处于 connected 状态；
+ weak: 其反义 strong 的定义是 `writable && receiving && connected`;
+ active: `write_state_ != STATE_WRITE_TIMEOUT`；
+ rtt：收到 ping response 后会根据历史的 ping request response 时间间隔估算 RTT，具体估算算法在 `Connection::ReceivedPingResponse` 函数中；
+ `nomination`: 提名过程的取值，关于提名过程，简言之就是 initiator 让某个已建立的 Connection 被选中，优先进行连通性检查，详情可以查阅 [ICE rfc](https://tools.ietf.org/html/rfc5245#section-8)；

### ICE 状态变迁

ICE 状态 IceConnectionState 定义在 `api/peerconnectioninterface.h` 中，状态定义如下：

+ `kIceConnectionNew`: PeerConnection 构造时的初始状态；
+ `kIceConnectionChecking`: ICE 连通性检查状态，包括收集 candidate、ICE 连通性检查；
  - 当 initiator 应用 remote sdp（即 answer）时，如果 sdp 里有 media section，且当前状态为 New，那就切换到 Checking 状态；
  - 当添加 remote candidate 时（从 sdp 中，或者信令通道），如果当前状态是 New 或者 Disconnected，那就切换到 Checking 状态；
+ `kIceConnectionConnected`, `kIceConnectionCompleted`, `kIceConnectionFailed`, `kIceConnectionDisconnected` 则根据 JsepTransportController 的 IceConnectionState 来进行设置，基本上一一对应，除了 Disconnected：如果当前处于 Connected 或 Completed，但 JsepTransportController 的 IceConnectionState 切换到了 Connecting，那就认为是发生了断连，故切换到 Disconnected 状态；
+ `kIceConnectionClosed`: PC 被关闭时，切换到 Closed 状态；

JsepTransportController 的 IceConnectionState 的计算状态的逻辑在 `JsepTransportController::UpdateAggregateStates_n` 函数中：

+ 默认处于 `kIceConnectionConnecting` 状态；
+ 遍历所有的 DtlsTransportInternal（即 DtlsTransport，其内部有个 P2PTransportChannel），分别统计以下几种状态：
  - `any_failed`: 是否有任一 P2PTransportChannel 处于 `cricket::IceTransportState::STATE_FAILED` 状态；
  - `all_connected`: 是否所有 DTLS transport 都处于 writable 状态；（关于 DtlsTransport 的 writable 状态，请看下文）
  - `all_completed`: 是否所有的 DTLS transport 都处于 writable 状态，且 P2PTransportChannel 处于 `cricket::IceTransportState::STATE_COMPLETED` 状态（关于 IceTransportState，请见下文），且自己是 initiator，且 candidate 收集状态处于 `cricket::kIceGatheringComplete` 状态；（关于 candidate 收集状态，请看下文）
  - `any_gathering`: 是否有任一 P2PTransportChannel candidate 收集状态未处于 `cricket::kIceGatheringNew` 状态；（_注：`any_gathering` 的计算逻辑有些绕，它并不是判断等于 `kIceGatheringChecking`，原因可以参考[这个 patch](https://webrtc-review.googlesource.com/c/src/+/93060)_）
  - `all_done_gathering`: 是否所有 P2PTransportChannel candidate 收集状态均处于 `cricket::kIceGatheringComplete` 状态；
+ 如果 `any_failed`，则切换到 `cricket::kIceConnectionFailed` 状态；
+ 如果 `all_completed`，则切换到 `cricket::kIceConnectionCompleted` 状态；
+ 如果 `all_connected`，则切换到 `cricket::kIceConnectionConnected` 状态；

DtlsTransport 的 writable 状态：

+ 如果没有禁用 DTLS，那 writable 就意味着完成了 DTLS 握手，而开始 DTLS 握手的前提就是其底层的 P2PTransportChannel 处于 writable 状态；
+ 如果禁用了 DTLS，那 writable 就等同于其底层的 P2PTransportChannel 处于 writable 状态；
+ 而 P2PTransportChannel 处于 writable 的条件则是选出了 Connection，且处于 `writable || PresumedWritable` 状态；

P2PTransportChannel 的 IceTransportState：

+ P2PTransportChannel 创建时处于 `STATE_INIT` 状态，之后的状态更新逻辑都在 `P2PTransportChannel::ComputeState` 函数中，这个函数在 `P2PTransportChannel::SortConnectionsAndUpdateState` 函数的最后调用，这是为了既保证 Connection 更新后状态更新的及时性，又避免同时多个 Connection 更新导致多次状态更新；
+ 如果执行过 AddConnection（收到未知地址的 STUN Binding request，或者 CreateConnection），但是当前没有任何 Connection 处于 active 状态，那就处于 `STATE_FAILED` 状态；
+ 如果有某个 Network 有多个 active connection，那就处于 `STATE_CONNECTING` 状态；
+ 如果所有 active connection 都有不同的 Network，即每个 Network 至多有一个 active connection，那就处于 `STATE_COMPLETED` 状态；

candidate 收集状态：

+ P2PTransportChannel 创建时，`gathering_state_` 为 kIceGatheringNew；
+ 在 `P2PTransportChannel::MaybeStartGathering` 里开始收集 candidate 时，如果当前未处于 Gathering 状态，则切换到 kIceGatheringGathering 状态；
+ 在收到 OnCandidatesAllocationDone 回调时，切换到 kIceGatheringComplete 状态；
  - 当 `BasicPortAllocatorSession::CandidatesAllocationDone` 为 true 时，就会触发这个回调；
  - 这意味着创建了 AllocationSequence，且所有的 AllocationSequence 都不处于 kRunning 状态，且所有的 port 都不处于 `STATE_INPROGRESS` 状态；
  - AllocationSequence 创建时处于 kInit 状态，`AllocationSequence::Start` 里切换为 kRunning 状态，TCP phase 结束后切换为 kCompleted 状态，如果调用了 `AllocationSequence::Stop`，则会切换到 kStopped 状态；
  - port 创建时就处于 `STATE_INPROGRESS` 状态，当被 prune、发生错误时，分别切换到 `STATE_PRUNED`、`STATE_ERROR` 状态，TurnPort 和 TcpPort 收集到 candidate 后调用 `Port::AddAddress` 时，就会切换到 `STATE_COMPLETE` 状态，RelayPort（GTURN）和 UdpPort 也会在收集到 candidate 后切换到 `STATE_COMPLETE` 状态，StunPort 则会在收集完 candidate（即向所有 STUN server 完成了 binding request）之后切换到 `STATE_COMPLETE` 状态；

### Connection 排序

Connection 的排序采用的是 stable sort，即原本排在前面的，如果两者比较不分伯仲，就保留原有顺序，这是为了避免对顺序做不必要的扰动。

而比较逻辑主要实现在 `P2PTransportChannel::CompareConnections` 函数中：

+ 首先通过 `P2PTransportChannel::CompareConnectionStates` 对比状态：简言之就是 writable、receiving 的排在 non-writable、non-receiving 的前面；
  - 首先比较 `writable || PresumedWritable`：如果 a 是 b 不是，那就 a 排前面；如果 a 不是 b 是，那就 b 排前面；如果 ab 相同，则继续比较；
  - 再比较 `write_state_` 值：值小的排前面；如果值相同，则继续比较；
  - 再比较 receiving：如果 a 是 b 不是，则 a 排前面；如果 a 不是 b 是，这时不能直接决定 b 在前面，因为 receiving 是一个比较弱的条件，这种情况下可能仍希望让 a 继续排在前面一段时间；不过目前实际上没有允许这种情况，因此如果 a 不是 b 是，那就是 b 排在前面；如果 ab 相同，则继续比较；
  - 最后如果 ab 的 `write_state_` 都是 `STATE_WRITABLE`，那就比较 connected：如果 a 是 b 不是，那就 a 排前面；如果 a 不是 b 是，那就 b 排前面；如果值相同，则认为不相伯仲；
+ 如果 `CompareConnectionStates` 认为不相伯仲，且自己不是主动发起 ICE 请求的一方（即不是 initiator），那就比较 initiator 设置的 nomination：nomination 大的排在前面；
+ 如果 nomination 相同，则比较最近收到数据的时间，更近接收到数据的排在前面；
+ 如果仍未分出高下，就比较 network cost 和 priority：cost 考虑的是网络类型，以太网、loopback 最低，WiFi 较低，4G 较高，未指定类型最高；priority 考虑的是 candidate 类型、网络 IP 地址，其计算有一个公式，具体可查阅 [ICE rfc](https://tools.ietf.org/html/rfc5245#section-4.1.2)，candidate 类型权值定义在 `IcePriorityValue` 里，网络 IP 类型权值的计算在 `network.cc SortNetworks` 函数中；

如果 CompareConnections 的结果表明不相伯仲，那就 rtt 大的 Connection 排在后面。

### Connection 选择和淘汰

对 Connection 排完序之后，会在 `P2PTransportChannel::MaybeSwitchSelectedConnection` 决定是否选中排在最前面的 Connection。

+ 首先只有处于 writable，或 PresumedWritable，或处于 UNRELIABLE 状态的 Connection 才可能被选中；
+ 如果当前没有选中的 Connection，且新的 Connection 可以被选中，那不用犹豫，直接选中它；
+ 如果当前已经有了选中的 Connection，那就得二者一较高下了：先比较 network cost，如果新的 Connection cost 更高，且未处于 receiving 状态，那就妥妥的维持现状；
+ 否则，就让已选中的 Connection 和新 Connection 重新走一下排序的对比逻辑，如果新的 Connection 应该排在前面，就选中它；但比较时这里有一个细微差别：当到最后一步比较 rtt 时，新 Connection 的 rtt 不仅要更低，还得比已选中 Connection 的 rtt 低得超过一个阈值，才会选中它；

搞清楚了选中 Connection 的逻辑，那选中一个 Connection 意味着什么呢？还记得上文提到的 transport writable 状态吗，transport writable 最基本的前提就是选出了 Connection。

其实除了选中 Connection 用到了排序的结果，STUN ping 的过程也用到了排序结果，此外，一个 Connection 要想被选中，那就必须变成 writable 状态，为此也就需要进行 STUN ping。

那如何确定该 ping 哪个 connection 呢？其逻辑实现在 `P2PTransportChannel::FindNextPingableConnection` 中：

+ 每次检查只会选出一个 Connection 执行 ping 操作（注意这不等价于同时只 ping 一个 Connection），此外，已处于 writable 状态的 Connection 也需要定期进行 ping，以确认健康度；
+ 如果已有选中的 Connection，且其处于 writable 状态，且需要 ping，那就优先 ping 它；
+ 如果没有选中的 Connection，或者其处于 weak 状态，那就每个 Network 选出一个排在最前面的 writable 且 connected 的 Connection（C++ `std::map::insert` 同一 key 只会加一次），对其中需要 ping 的，选出上次 ping 的时间最早的 Connection，进行本次 ping 操作；
+ 如果上一步还没选出一个 Connection，那就从那些非 writable、收到 ping 但未回 ping 的 Connection 里，选出收到 ping 的时间最早的 Connection，进行本次 ping 操作；
+ 如果还没选到，那就从所有未 ping 过的 Connection 里（如果都 ping 过，那就认为都没 ping 过），选出 most pingable 的 Connection，依次按下述三种指标挑选：最可能连通（两端都是 relay 的 Connection 更可能连通，在此前提下，UDP 更可能连通），最近收到过 ping，排序时排在前面；

收集 local candidate 或者添加 remote candidate 时，每个 port 都会和 remote candidate 创建 Connection，因此很可能每个 Network 创建多个 Connection，那什么时候销毁 Connection 呢？收到无法处理的 STUN error，或者超过一定时间未收到任何数据，那就会销毁 connection。

此外，也不是每个 Connection 都需要尝试进行连通性检查，比如同一个 Network 的多个 Connection 里，如果 best Connection 已经处于非 weak 状态了，那其他不如 best 的 Connection 就都不必继续尝试，可以提前剪枝（prune）了，被剪枝的 Connection 由于不会被继续使用，因此在一段时间后就会被超时机制销毁。剪枝逻辑实现在 `P2PTransportChannel::PruneConnections` 函数中。

## 连接使用

_好了，连接终于建立成功，恭喜你 :)_

连接建立成功后，就可以收发应用层的数据了，数据的收发将会通过选出来的 Connection 对象完成，Connection 则是调用 Port，Port 则是调用 AsyncPacketSocket，而这里用到的 Port 和 AsyncPacketSocket 对象，都是在收集本地 candidate 过程中创建的，并不会重新创建。

P2P 连接使用的细节分析，限于篇幅就留在下一篇里进行分析了，敬请期待 :)

---

## 附录一：NAT 类型与 P2P 连接的可行性

### 终端所处网络类型（UDP 可用性）

+ Opened：开放式，拥有公网 ip，且没有防火墙，可自由与外部通信；
+ Full Cone NAT：完全圆锥型 NAT，NAT 规则如下：从主机 UDP 端口 A 发出的数据包都会对应到 NAT 设备出口 ip 和端口 B，并且从任意外部地址（ip port 二元组）发送到该 NAT 设备 UDP 端口 B 的包都会被转到主机端口 A；
+ Restricted Cone NAT：地址限制圆锥型 NAT，相比于 Full Cone NAT，只有从之前该主机发出包的目的 ip 发送到该 NAT 设备 UDP 端口 B 的包才会被转到主机端口 A；
+ Port Restricted cone NAT：端口限制圆锥型 NAT，相比于 Full Cone NAT，只有从之前该主机发出包的目的 ip 和 port 发送到该 NAT 设备 UDP 端口 B 的包才会被转到主机端口 A；
+ Symmetric NAT：对称型 NAT，在 Port Restricted cone NAT 的基础上，即使数据包都从主机 UDP 端 A 发出，但只要目的地址（ip port 二元组）不同，NAT 设备就会为之分配不同的出端口 B；
+ Symmetric UDP Firewall：对称型 UDP 防火墙，防火墙规则如下：从主机 UDP 端口 A 发出的数据包保持源地址，但只有从之前该主机发出包的目的 ip 和 port 发送到该主机端口 A 的包才能通过防火墙；
+ Blocked：防火墙限制 UDP 通信；

利用有两个固定公网 ip 的 STUN server，客户端可以通过数次测试，探测自己所处的网络类型：

```

                        +--------+
                        |  Test  |
                        |   I    |
                        +--------+
                             |
                             |
                             V
                            /\              /\
                         N /  \ Y          /  \ Y             +--------+
          UDP     <-------/Resp\--------->/ IP \------------->|  Test  |
          Blocked         \ ?  /          \Same/              |   II   |
                           \  /            \? /               +--------+
                            \/              \/                    |
                                             | N                  |
                                             |                    V
                                             V                    /\
                                         +--------+  Sym.      N /  \
                                         |  Test  |  UDP    <---/Resp\
                                         |   II   |  Firewall   \ ?  /
                                         +--------+              \  /
                                             |                    \/
                                             V                     |Y
                  /\                         /\                    |
   Symmetric  N  /  \       +--------+   N  /  \                   V
      NAT  <--- / IP \<-----|  Test  |<--- /Resp\               Open
                \Same/      |   I    |     \ ?  /               Internet
                 \? /       +--------+      \  /
                  \/                         \/
                  |                           |Y
                  |                           |
                  |                           V
                  |                           Full
                  |                           Cone
                  V              /\
              +--------+        /  \ Y
              |  Test  |------>/Resp\---->Restricted
              |   III  |       \ ?  /
              +--------+        \  /
                                 \/
                                  |N
                                  |       Port
                                  +------>Restricted

                 Figure 2: Flow for type discovery process
```

1. STUN 客户端从向 STUN 服务器发送请求，要求得到自身经 NAT 映射后的地址：
    1. 收不到服务器回复，则认为 UDP 被防火墙阻断，不能通信，网络类型：Blocked；
    1. 收到服务器回复，对比本地地址，如果相同，则认为无 NAT 设备，进入第 2 步，否则认为有 NAT 设备，进入第 3 步；
1. （已确认无 NAT 设备）STUN 客户端向 STUN 服务器发送请求，要求服务器从其他 ip 和 port 向客户端回复包：
    1. 收不到服务器从其他 ip 地址的回复，认为包被前置防火墙阻断，网络类型：Symmetric UDP Firewall；
    1. 收到则认为客户端处在一个开放的网络上，网络类型：Opened；
1. （已确认有 NAT 设备）STUN 客户端向 STUN 服务器发送请求，要求服务器从其他 ip 和 port 向客户端回复包：
    1. 收不到服务器从其他 ip 地址的回复，认为包被前置 NAT 设备阻断，进入第 4 步；
    1. 收到服务器回复，则网络类型：Full Cone NAT；
1. STUN 客户端向 STUN 服务器的另外一个 ip 地址发送请求（本地端口不变），要求得到自身经 NAT 映射后的地址，并和第 1 步得到的地址对比：
    1. 地址不相同，则网络类型：Symmetric NAT；
    1. 地址相同，则认为是 Restricted NAT，进入第 5 步，进一步确认类型；
1. （已确认 Restricted NAT 设备）STUN 客户端向 STUN 服务器发送请求，要求服务器从相同 ip 的其他 port 向客户端回复包：
    1. 收不到服务器从其他 port 地址的回复，认为包被前置 NAT 设备阻断，网络类型：Port Restricted Cone NAT；
    1. 收到则认为网络类型：Restricted Cone NAT；

NAT 针对 TCP 的实现基本是一致的，并不存在太大差异，所以也就没有类型一说，这是因为 TCP 协议本身便是面向连接的，因此无需考虑网络连接无状态所带来的复杂性。

### NAT traversal / hole punching

记 STUN binding 结果为：A `private_port_a public_ip_a:public_port_a`，B `private_port_b public_ip_b:public_port_b`。

+ Full Cone NAT：A `private_port_a => public_ip_b:public_port_b`，B `private_port_b => public_ip_a:public_port_a`，即可收发数据；
+ Restricted Cone NAT
  - A `private_port_a => public_ip_b:public_port_b`，由于此包的 src ip 与 STUN server 不同，故被丢弃；
  - B `private_port_b => public_ip_a:public_port_a`，由于 A 已经向 B（`public_ip_b`）发过包，因而此包会通过；
  - A `private_port_a => public_ip_b:public_port_b`，由于 B 已经向 A（`public_ip_a`）发过包，因而此包会通过；
  - 综上：只要双方不停发包，且没有某一方的包全部丢失，就能打洞成功；
+ Port Restricted Cone NAT
  - A `private_port_a => public_ip_b:public_port_b`，由于此包的 src ip/port 与 STUN server 均不同，故被丢弃；
  - B `private_port_b => public_ip_a:public_port_a`，由于 A 已经向 B（`public_ip_b:public_port_b`）发过包，因而此包会通过；
  - A `private_port_a => public_ip_b:public_port_b`，由于 B 已经向 A（`public_ip_a:public_port_a`）发过包，因而此包会通过；
  - 综上：只要双方不停发包，且没有某一方的包全部丢失，就能打洞成功；
+ Symmetric NAT：由于 A 发往 B 映射的外部端口已经不是 A 发往 STUN server 映射的外部端口，因此 B 发往 A 的包也会被丢弃，对称型 NAT 无法打洞成功；

## 附录二：收集 candidate 过程日志分析

DoAllocate 开始：

```
[000:367] [3335] (basicportallocator.cc:753): Allocate ports on 4 networks
```

某个 AllocationSequence 的 UDP host 分配过程：

```
[000:374] [3335] (basicportallocator.cc:1446): Net[en0:192.168.50.0/24:Wifi:id=1]: Allocation Phase=Udp
[000:377] [3335] (port.cc:319): Port[0x1128ac600::1:0:local:Net[en0:192.168.50.0/24:Wifi:id=1]]: Port created with network cost 10
[000:378] [3335] (basicportallocator.cc:1517): AllocationSequence: UDPPort will be handling the STUN candidate generation.
[000:378] [3335] (basicportallocator.cc:873): Adding allocated port for 0
[000:378] [3335] (basicportallocator.cc:892): Port[0x1128ac600:0:1:0:local:Net[en0:192.168.50.0/24:Wifi:id=1]]: Added port to allocator
[000:379] [3335] (basicportallocator.cc:909): Port[0x1128ac600:0:1:0:local:Net[en0:192.168.50.0/24:Wifi:id=1]]: Gathered candidate: Cand[:2047277109:1:udp:2122260223:192.168.50.49:65400:local::0:0qbZ:dcSLq5Bs44BYu8yWN8mtlU48:1:10:0]
[000:380] [3335] (basicportallocator.cc:956): Port[0x1128ac600:0:1:0:local:Net[en0:192.168.50.0/24:Wifi:id=1]]: Port ready.
[000:383] [771] (RTCLogging.mm:31): (ARDAppEngineClient.m:94 -[ARDAppEngineClient sendMessage:forRoomId:clientId:completionHandler:]): C->RS POST: {
  "label" : 0,
  "id" : "0",
  "candidate" : "candidate:2047277109 1 udp 2122260223 192.168.50.49 65400 typ host generation 0 ufrag 0qbZ network-id 1 network-cost 10",
  "type" : "candidate"
}
```

UDP srflx 分配过程：

```
[000:399] [3335] (basicportallocator.cc:909): Port[0x1128ac600:0:1:0:local:Net[en0:192.168.50.0/24:Wifi:id=1]]: Gathered candidate: Cand[:4216258177:1:udp:1686052607:59.109.147.213:54541:stun:192.168.50.49:65400:0qbZ:dcSLq5Bs44BYu8yWN8mtlU48:1:10:0]
[000:410] [3335] (basicportallocator.cc:1044): Port[0x1128ac600:0:1:0:local:Net[en0:192.168.50.0/24:Wifi:id=1]]: Port completed gathering candidates.
[000:412] [771] (RTCLogging.mm:31): (ARDAppEngineClient.m:94 -[ARDAppEngineClient sendMessage:forRoomId:clientId:completionHandler:]): C->RS POST: {
  "label" : 0,
  "id" : "0",
  "candidate" : "candidate:4216258177 1 udp 1686052607 59.109.147.213 54541 typ srflx raddr 192.168.50.49 rport 65400 generation 0 ufrag 0qbZ network-id 1 network-cost 10",
  "type" : "candidate"
}
```

RELAY 分配过程：

```
[000:437] [3335] (basicportallocator.cc:1446): Net[en0:192.168.50.0/24:Wifi:id=1]: Allocation Phase=Relay
[000:438] [3335] (port.cc:319): Port[0x1128b4e00::1:0:relay:Net[en0:192.168.50.0/24:Wifi:id=1]]: Port created with network cost 10
[000:439] [3335] (basicportallocator.cc:873): Adding allocated port for 0
[000:439] [3335] (basicportallocator.cc:892): Port[0x1128b4e00:0:1:0:relay:Net[en0:192.168.50.0/24:Wifi:id=1]]: Added port to allocator
[000:439] [3335] (turnport.cc:335): Port[0x1128b4e00:0:1:0:relay:Net[en0:192.168.50.0/24:Wifi:id=1]]: Trying to connect to TURN server via udp @ 123.56.66.149:3478
[000:443] [3335] (turnport.cc:1281): Port[0x1128b4e00:0:1:0:relay:Net[en0:192.168.50.0/24:Wifi:id=1]]: TURN allocate request sent, id=61426b36764d637244316e47
[000:454] [3335] (turnport.cc:1333): Port[0x1128b4e00:0:1:0:relay:Net[en0:192.168.50.0/24:Wifi:id=1]]: Received TURN allocate error response, id=61426b36764d637244316e47, code=401, rtt=14
[000:455] [3335] (turnport.cc:1281): Port[0x1128b4e00:0:1:0:relay:Net[en0:192.168.50.0/24:Wifi:id=1]]: TURN allocate request sent, id=47305a4e534736355a4e6945
[000:468] [3335] (turnport.cc:1287): Port[0x1128b4e00:0:1:0:relay:Net[en0:192.168.50.0/24:Wifi:id=1]]: TURN allocate requested successfully, id=47305a4e534736355a4e6945, code=0, rtt=13
[000:468] [3335] (basicportallocator.cc:909): Port[0x1128b4e00:0:1:0:relay:Net[en0:192.168.50.0/24:Wifi:id=1]]: Gathered candidate: Cand[:425266556:1:udp:41885439:172.17.7.46:61226:relay:59.109.147.213:54541:0qbZ:dcSLq5Bs44BYu8yWN8mtlU48:1:10:0]
[000:468] [3335] (basicportallocator.cc:956): Port[0x1128b4e00:0:1:0:relay:Net[en0:192.168.50.0/24:Wifi:id=1]]: Port ready.
[000:468] [3335] (basicportallocator.cc:1044): Port[0x1128b4e00:0:1:0:relay:Net[en0:192.168.50.0/24:Wifi:id=1]]: Port completed gathering candidates.
[000:468] [3335] (turnport.cc:1050): Port[0x1128b4e00:0:1:0:relay:Net[en0:192.168.50.0/24:Wifi:id=1]]: Scheduled refresh in 540000ms.
[000:472] [771] (RTCLogging.mm:31): (ARDAppEngineClient.m:94 -[ARDAppEngineClient sendMessage:forRoomId:clientId:completionHandler:]): C->RS POST: {
  "label" : 0,
  "id" : "0",
  "candidate" : "candidate:425266556 1 udp 41885439 172.17.7.46 61226 typ relay raddr 59.109.147.213 rport 54541 generation 0 ufrag 0qbZ network-id 1 network-cost 10",
  "type" : "candidate"
}
```
