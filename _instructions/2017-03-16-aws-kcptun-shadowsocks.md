# AWS(ubuntu) + kcptun + Shadowsocks(Mac)

+ [Installing and running shadowsocks on Ubuntu Server](https://gist.github.com/zhiguangwang/7018fbc0a38a5b663868)
+ 注意要重新配置，listen 和 target 都要填 aws private ip；
+ 另外 EC2 console 要开启 29900 出口权限；
+ ShadowsocksX NG 客户端支持 kcptun，把端口改成 29900 即可；
+ 1080p 速度从 10 Mbps -> 20 Mbps；

## ASUS RT-AC68U VPN

参考[科学上网](https://haoel.github.io/)和 [How To Setup Your Own VPN With PPTP](https://www.digitalocean.com/community/tutorials/how-to-setup-your-own-vpn-with-pptp) 失败，PPTP VPN 要么连接失败，要么报错 unsupported protocol。

+ [Initialize OPTWARE](https://github.com/RMerl/asuswrt-merlin/wiki/Initialize-OPTWARE)
+ [Entware](https://github.com/RMerl/asuswrt-merlin/wiki/Entware)

最终决定参考[在路由器上部署 shadowsocks](https://zzz.buzz/zh/gfw/2016/02/16/deploy-shadowsocks-on-routers/)。

+ 刷入 merlin 固件，最新 `RT-AC68U_384.5_0` 可以；
+ 准备一块 U 盘，在 Windows 下用 [MiniTool Partition Wizard Home Edition](http://www.partitionwizard.com/free-partition-manager.html) 格式化为 ext2 格式，注意一定要格式化为 ext2 格式；
+ 进入管理后台，USB 相关应用 - Download Master，先安装，再卸载，参考 [Initialize OPTWARE](https://github.com/RMerl/asuswrt-merlin/wiki/Initialize-OPTWARE)；
+ iptables 配置转发；
