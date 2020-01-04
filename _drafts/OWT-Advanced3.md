# OWT 进阶 3

## RTCP

+ 通过 `WebRtcConnection::deliverFeedback_` 发送？
+ `WebRtcConnection::read` 里读到收流端的 RTCP 报文后就调用到推流端的 `WebRtcConnection::deliverFeedback_`？
+ RR 是在 `RtcpFeedbackGenerationHandler::read` 里发送的？
+ 看代码：OWT 只会主动发 PLI？和 RR？
+ 看日志：只会转发收流端的 FIR？
+ DTLS 只有握手包才是 `DtlsTransport::isDtlsPacket`？RTP/RTCP 包不是？DTLS 和 SRTP 的关系是？
