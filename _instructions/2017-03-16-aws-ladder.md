# AWS(ubuntu) Ladder

## ShadowSocks

+ [Installing and running shadowsocks on Ubuntu Server](https://gist.github.com/zhiguangwang/7018fbc0a38a5b663868)
+ 注意要重新配置，listen 和 target 都要填 aws private ip；
+ 另外 EC2 console 要开启 29900 出口权限；
+ ShadowsocksX NG 客户端支持 kcptun，把端口改成 29900 即可；
+ 1080p 速度从 10 Mbps -> 20 Mbps；

## V2Ray

## ASUS RT-AC68U VPN

RT-AC68U + koolshare 380.63_0-X7.2（升级后 format jffs at next boot 并重启） + 离线安装科学上网插件。

详细步骤：

+ 登录管理后台 - 系统管理（页面左侧栏高级设置倒数第三项） - 固件升级 - 上传，选择 [`RT-AC68U_380.63_X7.2.trx`](https://firmware.koolshare.cn/Koolshare_Merlin_Legacy_380/ASUS/RT-AC68U/X7.2/) 文件；
+ 升级完成后，登录管理后台 - 系统管理 - 系统设置，勾选「Format JFFS partition at next boot」，「Enable JFFS custom scripts and configs」，点击「应用本页面设置」（页面最底部），再点击「重新启动」（页面上方）；
+ 重启完毕后，登录管理后台 - Software center（左侧栏最底部）- 更新；
+ 更新完毕后，Software center - 离线安装 - 选择 `shadowsocks_3.9.9.tar.gz`（或从 https://github.com/heweiye/Merlin_Shadowsocks 下载最新版本） - 上传并安装；
+ 界面显示「离线安装插件成功」后，点击 Software center，可以看到「科学上网」插件；
+ 进入科学上网插件 - 手动添加 - SS 节点，设置节点别名、服务器地址、服务器端口、密码、加密方式，其他选项保持默认，点击添加；节点管理 - 勾上「科学上网开关」，点击应用；

---

koolshare 新版软件中心屏蔽了某些软件的安装，具体报错信息参考如下：

```bash
检测到离线安装包：koolss 含非法关键词！！！
根据法律规定，koolshare 软件中心将不会安装此插件！！！
```

解决办法是登陆软路由 ssh（开路由器 ssh，装 webshell，一定要把路由器界面语言改成英文）取消该限制：

```bash
sed -i 's/\tdetect_package/\t# detect_package/g' /koolshare/scripts/ks_tar_install.sh
```

然后就可以愉快的玩耍啦~
