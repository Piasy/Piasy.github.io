# AWS(ubuntu) + kcptun + Shadowsocks(Mac)

+ [Installing and running shadowsocks on Ubuntu Server](https://gist.github.com/zhiguangwang/7018fbc0a38a5b663868)
+ 注意要重新配置，listen 和 target 都要填 aws private ip；
+ 另外 EC2 console 要开启 29900 出口权限；
+ ShadowsocksX NG 客户端支持 kcptun，把端口改成 29900 即可；
+ 1080p 速度从 10 Mbps -> 20 Mbps；

## ASUS RT-AC68U VPN

RT-AC68U + koolshare 380.63_0-X7.2（升级后 format jffs at next boot 并重启） + 离线安装科学上网插件。
