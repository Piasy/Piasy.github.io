# OWT 录制

```
management_api/api.js 定义 RESTful API 的处理函数 ->
management_api/resource/recordingsResource.js exports.add ->
management_api/requestHandler.js exports.addServerSideSubscription ->
（agent/conference/conference.js that.rpcAPI 注册了 RPC server）
agent/conference/conference.js that.addServerSideSubscription ->
agent/conference/accessController.js that.initiate 根据 sessionOptions.type 找到 RPC server node 为 recording node ->
agent/conference/rpcRequest.js that.initiate: direction 是 out，故 RPC 调用（recording node 的）subscribe ->
agent/recording/index.js that.subscribe 创建 AVStreamOut
```

AVStreamOut 是 `agent/addons/avstreamLib` 扩展里的类型，其 C++ 实现为 `agent/addons/avstreamLib/AVStreamOutWrap.cc`，在 `AVStreamOutWrap::New` 中, type 是 `file`，故实际创建的是 `owt_base::MediaFileOut`。另一种 type 是 `streaming`，用于转推 `rtsp`/`rtmp`/`hls`/`dash`。

`owt_base::MediaFileOut` 继承自 `owt_base::AVStreamOut`。在 `AVStreamOut::onFrame` 中，会把帧放入 `m_frameQueue` 中；在构造函数里，会创建一个线程，线程函数为 `AVStreamOut::sendLoop`，其中不停从 `m_frameQueue` 里取帧，取到后调用 `AVStreamOut::writeFrame`；其中就是调用 ffmpeg 的 `av_interleaved_write_frame` 写入音视频帧。

TODO: OWT 完整的音视频数据流程。
