# AWS(ubuntu) + kcptun + Shadowsocks(Mac)

+ 主要参考：[一步一步教你用Kcptun给Shadowsocks加速！看YouTube1080P一点都不卡！](http://www.jianshu.com/p/172c38ba6cee)
+ 注意要重新配置，listen 和 target 都要填 aws private ip；
+ 另外 EC2 console 要开启 29900 出口权限；
+ ShadowsocksX NG 客户端支持 kcptun，把端口改成 29900 即可；
+ 1080p 速度从 10 Mbps -> 20 Mbps；
