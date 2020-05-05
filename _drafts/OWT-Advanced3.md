# OWT 进阶 3

## RTCP

RTCP packet type:

+ SR, sender report: 200;
+ RR, receiver report: 201;
+ RTPFB: 205;

OWT Server 流程：

+ 通过 `WebRtcConnection::deliverFeedback_` 发送？
+ `WebRtcConnection::read` 里读到收流端的 RTCP 报文后就调用到推流端的 `WebRtcConnection::deliverFeedback_`？
+ RR 是在 `RtcpFeedbackGenerationHandler::read` 里发送的？
+ 看代码：OWT 只会主动发 PLI？和 RR？
+ 看日志：只会转发收流端的 FIR？
+ DTLS 只有握手包才是 `DtlsTransport::isDtlsPacket`？RTP/RTCP 包不是？DTLS 和 SRTP 的关系是？

客户端流程：

+ 在 `PeerConnection::Initialize` 中创建了 `JsepTransportController`，创建时通过 `JsepTransportController::Config` 设置了 `rtcp_handler`，处理收到的 RTCP 报文；
+ RTCP 报文是在 `network_thread` 收到的，而处理 RTCP 报文则切换到了 `worker_thread`，由 `Call::DeliverRtcp` 负责；
+ 由于在 `rtcp_handler` 中调用 `Call::DeliverRtcp` 时，`media_type` 是写死的 `MediaType::ANY`，所以 RTCP 报文会交给每个 `VideoReceiveStream`/`AudioReceiveStream`/`VideoSendStream`/`AudioSendStream`，任其按需处理；
+ 这四种 stream 都是把 RTCP 报文交给 `ModuleRtpRtcpImpl` 进行处理，并且是不同的实例；
+ 最终 RTCP 报文会在 `RTCPReceiver::ParseCompoundPacket` 函数中，根据不同的 block 类型，进行不同的处理；

音频的 `remote-inbound-rtp` 在 `RTCStatsCollector::ProduceAudioRTPStreamStats_n` 中生成，源于 `voice_sender_info.report_block_datas`，而 `report_block_datas` 则最终来源于 `ModuleRtpRtcpImpl::GetLatestReportBlockData`，也就是 `RTCPReceiver::GetLatestReportBlockData`（`received_report_blocks_`），而 `received_report_blocks_` 则会在 `RTCPReceiver::HandleReportBlock` 中填充，`HandleReportBlock` 则会在收到 `SenderReport` 和 `ReceiverReport` 后被调用。

`registered_ssrcs_` audio 和 video 不一样，但收到的 RR 报文，ssrc 都是 video 的，即只收到了 video 的 RR 报文。
