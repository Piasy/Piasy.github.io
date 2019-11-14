# 带宽分配

+ 网络估计带宽更新后，会调用 `BitrateAllocator::OnNetworkEstimateChanged`；
+ 进程生命周期内，应该会记录最后一次的估计带宽，这样下次就不会从头估计；
+ 从头估计都是从 300kbps 起步；
+ `goog_cc` 是在 `DelayBasedBwe::SetStartBitrate` 设置起步估计带宽；
+ start bitrate 应该和具体 CC 算法无关，都是在 `PeerConnectionFactory::CreateCall_w` 中解析 field trial 得来，设置方式为：`WebRTC-PcFactoryDefaultBitrates/min:30,start:800,max:2000/`；
