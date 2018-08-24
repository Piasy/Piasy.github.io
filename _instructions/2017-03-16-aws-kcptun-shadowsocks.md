# AWS(ubuntu) + kcptun + Shadowsocks(Mac)

+ [Installing and running shadowsocks on Ubuntu Server](https://gist.github.com/zhiguangwang/7018fbc0a38a5b663868)
+ 注意要重新配置，listen 和 target 都要填 aws private ip；
+ 另外 EC2 console 要开启 29900 出口权限；
+ ShadowsocksX NG 客户端支持 kcptun，把端口改成 29900 即可；
+ 1080p 速度从 10 Mbps -> 20 Mbps；

## ASUS RT-AC68U VPN

RT-AC68U + koolshare 380.63_0-X7.2（升级后 format jffs at next boot 并重启） + 离线安装科学上网插件。

详细步骤：

+ 登录管理后台 - 系统管理（页面左侧栏高级设置倒数第三项） - 固件升级 - 上传，选择 `RT-AC68U_380.63_X7.2.trx` 文件；
+ 升级完成后，登录管理后台 - 系统管理 - 系统设置，勾选「Format JFFS partition at next boot」，「Enable JFFS custom scripts and configs」，点击「应用本页面设置」（页面最底部），再点击「重新启动」（页面上方）；
+ 重启完毕后，登录管理后台 - Software center（左侧栏最底部）- 更新；
+ 更新完毕后，Software center - 离线安装 - 选择 `shadowsocks_3.9.9.tar.gz` - 上传并安装；
+ 界面显示「离线安装插件成功」后，点击 Software center，可以看到「科学上网」插件；
+ 进入科学上网插件 - 手动添加 - SS 节点，设置节点别名、服务器地址、服务器端口、密码、加密方式，其他选项保持默认，点击添加；节点管理 - 勾上「科学上网开关」，点击应用；
