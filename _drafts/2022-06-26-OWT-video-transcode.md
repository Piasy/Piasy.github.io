
+ `FFmpegFrameDecoder::onFrame`, 创建 buffer 是在 `FFmpegFrameDecoder::AVGetBuffer`
+ `FrameProcesser::onFrame`，会调用 `FrameConverter::convert` 对 frame 进行处理（等宽高就 copy，否则 scale），另外还有帧率控制，在 `FrameProcesser::SendFrame` 里调用 `deliverFrame` 送入下一环节
+ `VCMFrameEncoder::onFrame`
+ `VCMFrameEncoder::encode`
