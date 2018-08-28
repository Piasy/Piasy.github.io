
``` objc
RTCRtpTransceiver* videoTransceiver = nil;
for (RTCRtpTransceiver* transceiver in _peerConnection.transceivers) {
    if (transceiver.mediaType == RTCRtpMediaTypeVideo) {
        videoTransceiver = transceiver;
    }
}
RTCVideoTrack* track =
    (RTCVideoTrack*)(videoTransceiver.receiver.track);

[track addRenderer:_remoteRenderer];
```

调用栈：

```
RTCVideoTrack addRenderer
            ↓ proxy
VideoTrack::AddOrUpdateSink
            ↓ proxy
VideoTrackSource::AddOrUpdateSink
            ↓
VideoBroadcaster::AddOrUpdateSink
```