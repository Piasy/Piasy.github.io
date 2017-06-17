---
layout: post
title: 开箱即用的 WebRTC 开发环境
tags:
    - 流媒体
    - WebRTC
---

> 在刚刚落幕的 WWDC17 上，苹果为我们带来了一个不小的惊喜 —— 其浏览器内核WebKit将正式支持 WebRTC，而未来基于 WebKit 内核的苹果浏览器，比如macOS High Sierra、iOS 11 中的 Safari 浏览器和Safari 技术预览版 32，都将使用到 WebRTC 技术。
>
> ——[苹果终于入伙 WebRTC](https://mp.weixin.qq.com/s?__biz=MjM5MjAwODM4MA==&mid=2650689194&idx=1&sn=997dbb3a92daae03482ca8502f145ceb&chksm=bea63f7989d1b66fe8a1e484d1d4a18ffb2810e69a16bce01c5780c95264bc3f94395d5563bf&mpshare=1&scene=1&srcid=0612TAozPhE88lObIrNkhzFD&key=5e973e5d3cae484af799943e3987b5adc5b8149e99f20cb5aea564465da13525fe121e5d65e7301e3169394061a522986df8b6781b3a4e7231c2200826827631986e3226bebb2d49d679a21683cc8460&ascene=0&uin=MzMwMTQwNjU1&devicetype=iMac+MacBookPro13%2C3+OSX+OSX+10.12.5+build(16F73)&version=12020810&nettype=WIFI&fontScale=100&pass_ticket=xYBB6JseSePyVd3Fu%2B7LTCQZRCx1ebQmLZ4pwGPyccIdlW4e2GW2zS%2FmsfLrusfI)

适逢我也正在倒腾一些 WebRTC 的东西，万事开头难，搞事情最怕的就是开始的拦路虎，编译环境、demo 工程、Server 如何部署，这三个问题（尤其是最后一个）想必会浇灭很多朋友的热情之火。经过近两周的不懈奋斗，我总算把这几头拦路虎一一解决，今天我就在这里把这一套开箱即用的 WebRTC 开发环境分享给大家。

_注意：这里我假设大家具备 Docker 的基本使用能力，如不具备，请自行搜索相关教程。_

## WebRTC 编译环境

一开始我是用的 [webrtc-build-scripts](https://github.com/Piasy/webrtc-build-scripts) 这个工具，它利用的是 Vagrant，为了和后面的其他工具统一，我就基于它构建了一个 Docker 镜像：[piasy/webrtc-build](https://hub.docker.com/r/piasy/webrtc-build/)。

首先是 pull 镜像：

~~~ bash
docker pull piasy/webrtc-build
~~~

然后就是运行 Docker 镜像了：

~~~ bash
docker run --rm \
  -e ENABLE_SHADOW_SOCKS=false \
  -v <path to place webrtc source>:/webrtc \
  -t -i piasy/webrtc-build
~~~

单纯对 webrtc-build-scripts 做一层封装肯定没啥意思，针对国情，我在 Docker 镜像里面加上了使用 Shadowsocks 代理的支持，上面的命令不启用 Shadowsocks 代理，如需启用，则运行下面的命令：

~~~ bash
docker run --rm \
  -e ENABLE_SHADOW_SOCKS=true \
  -e SHADOW_SOCKS_SERVER_ADDR=<your shadowsocks server ip> \
  -e SHADOW_SOCKS_SERVER_PORT=<your shadowsocks server port> \
  -e SHADOW_SOCKS_ENC_METHOD=<your shadowsocks encrypt method> \
  -e SHADOW_SOCKS_ENC_PASS=<your shadowsocks encrypt password> \
  -v <path to place webrtc source>:/webrtc \
  -t -i piasy/webrtc-build
~~~

不过现在还有两点小瑕疵：

+ ~~proxychains 在 Docker 镜像中还有问题，所以 Shadowsocks 代理暂时不能用；~~ 换用 `polipo` 之后就 OK 了，感谢 [技术小黑屋](http://droidyue.com/blog/2016/04/04/set-shadowsocks-proxy-for-terminal/index.html)；
+ 目前只能编译 Android 环境，iOS 的我还没搞过；

_注意：如果 Shadowsocks 的密码有特殊字符，请用 `\` 进行转义，例如 `&bDmc!` 变为 `\&bDmc\!`。_

请大家保持关注 :)

### 编译命令

~~~ bash
# 安装相关依赖
install_dependencies

# 下载 WebRTC 代码
get_webrtc

# 编译 WebRTC 代码
build_apprtc
~~~

更多编译指令，可以参考 [webrtc-build-scripts](https://github.com/pristineio/webrtc-build-scripts) 和 [WebRTC 项目官网](https://webrtc.org/native-code/android/)。

### 命令行使用 Shadowsocks 代理

在这里顺便分享一下如何在命令行挂上 Shadowsocks 代理：

~~~ bash
sudo apt-get install python-pip
sudo pip install shadowsocks
sudo apt-get install proxychains

wget https://download.libsodium.org/libsodium/releases/LATEST.tar.gz
tar zxf LATEST.tar.gz && cd libsodium*
./configure && make && make install

# 修复关联
echo /usr/local/lib > /etc/ld.so.conf.d/usr_local_lib.conf && ldconfig
~~~

`/etc/shadowsocks.json` 配置内容：

~~~ json
{
"server":"server-ip",
"server_port":8000,
"local_address": "127.0.0.1",
"local_port":1080,
"password":"your-password",
"timeout":600,
"method":"aes-256-cfb"
}
~~~

`~/.proxychains/proxychains.conf` 配置内容：

~~~
strict_chain
proxy_dns
remote_dns_subnet 224
tcp_read_time_out 15000
tcp_connect_time_out 8000
localnet 127.0.0.0/255.0.0.0
quiet_mode

[ProxyList]
socks5  127.0.0.1 1080
~~~

执行下述命令之后，命令行程序就会使用 Shadowsocks 代理了：

~~~ bash
sudo sslocal -c /etc/shadowsocks.json -d start
proxychains bash
~~~

## Android demo 工程

Android demo 可以利用上面的 WebRTC 源码编译出 APK，但我们如何查阅代码、调试代码呢？最好自然是有一个 Android Studio 工程了，这里为大家送上：[AppRTC-Android](https://github.com/Piasy/AppRTC-Android)。

这个项目其实就是把相关路径的源码摘出来，并把编译完成的 so 库文件也拿出来，放到了一个 Android Studio 工程中，相关源码路径：

+ [webrtc/sdk/android/api/org/webrtc](https://chromium.googlesource.com/external/webrtc/+/master/webrtc/sdk/android/api/org/webrtc)
+ [webrtc/sdk/android/src/java/org/webrtc](https://chromium.googlesource.com/external/webrtc/+/master/webrtc/sdk/android/src/java/org/webrtc)
+ [webrtc/base/java/src/org/webrtc](https://chromium.googlesource.com/external/webrtc/+/master/webrtc/base/java/src/org/webrtc)
+ [webrtc/modules/audio_device/android/java/src/org/webrtc/voiceengine](https://chromium.googlesource.com/external/webrtc/+/master/webrtc/modules/audio_device/android/java/src/org/webrtc/voiceengine)
+ [webrtc/examples/androidapp](https://chromium.googlesource.com/external/webrtc/+/master/webrtc/examples/androidapp/)

## AppRTC-Server

AppRTC 是 WebRTC 的一个 demo 应用，它需要和 Server 配合才能运行。这个 Server 的搭建是最令人头疼的了，不过不用担心，咱这不是有开箱即用的工具嘛：[piasy/apprtc-server](https://hub.docker.com/r/piasy/apprtc-server/)。

pull 镜像：

~~~ bash
docker pull piasy/apprtc-server
~~~

运行 Docker 镜像：

~~~ bash
docker run --rm \
  -p 8080:8080 -p 8089:8089 -p 3478:3478 -p 3033:3033 \
  --expose=59000-65000 \
  -e PUBLIC_IP=<server public IP> \
  -v <path to constants.py>:/apprtc_configs \
  -t -i piasy/apprtc-server
~~~

运行之后，在 Android demo 的设置界面中，把 Server 地址设置为 `http://<server public IP>:8080`，demo 即可成功**跨网**视频通话。很多人自己部署完服务器之后发现，只能在同一 WiFi 下通话，跨 WiFi 就不行了。大家放心，咱们可没这个问题 :)

### AppRTC-Server 部署简介

开箱即用当然好，不过这里我也把构建过程中遇到的一些问题分享出来，希望对大家有所帮助。

AppRTC 需要三个 Server：

+ Room Server: 负责处理加入房间、获取配置等请求；
+ Signal Server: 长连接服务器，用于聊天过程中实时下发信息；
+ TURN/STUN Server: 打洞服务器，用于 NAT 打洞；

Room Server 和 Signal Server 都在 [apprtc](https://github.com/webrtc/apprtc) 这个项目中，部署说明很详细，需要注意的是关于 TURN/STUN Server 的配置问题：

AppRTC Android demo 中，会尝试从房间配置中读取 `pc_config` 域，以取得 TURN/STUN Server 信息；如果没有获取到，就会向配置中的 `ice_server_url` 指向的服务器获取 TURN/STUN Server 信息。但 apprtc 的配置说明中，让我们把 TURN/STUN Server 配置写在 `TURN_SERVER_OVERRIDE` 中，这是不行的，因为客户端的逻辑并不会读取 `TURN_SERVER_OVERRIDE` 这个域，此外 Android demo 读取 `pc_config` 的代码中，无法正确把 TURN Server 的用户名解析出来，所以我们不得不自己搭一个 ICE Server，尽管这会多一次网络请求，不过用于测试也没啥问题。最后，[这里是示例的 constants.py 地址](https://github.com/Piasy/WebRTC-Docker/blob/master/apprtc-server/constants.py)。

因此我们还需要第四个 Server：ICE Server。它用于下发 TURN/STUN Server 配置信息，代码如下（nodejs）：

~~~ javascript
var express = require('express')
var crypto = require('crypto')
var app = express()

var hmac = function (key, content) {
  var method = crypto.createHmac('sha1', key)
  method.setEncoding('base64')
  method.write(content)
  method.end()
  return method.read()
}

app.get('/iceconfig', function (req, resp) {
  var query = req.query
  var key = '4080218913'
  var time_to_live = 600
  var timestamp = Math.floor(Date.now() / 1000) + time_to_live
  var turn_username = timestamp + ':ninefingers'
  var password = hmac(key, turn_username)

  return resp.send({
    iceServers: [
      {
        urls: [
          'turn:ICE_SERVER_ADDR:3478?transport=udp',
          'turn:ICE_SERVER_ADDR:3478?transport=tcp',
          'turn:ICE_SERVER_ADDR:3479?transport=udp',
          'turn:ICE_SERVER_ADDR:3479?transport=tcp'
        ],
        username: turn_username,
        credential: password
      }
    ]
  })
})

app.listen('3033', function () {
  console.log('server started')
})
~~~

几点说明：

+ 这里面有两个配置写死了：`key = '4080218913'`，用户名 `ninefingers`，它们都是在部署 Coturn 时配置的；
+ `username`/`credential` 并不是 Coturn 创建用户的 username/password，而是按照上述逻辑计算出来的值，否则 Coturn 会报错 401 Unauthorized；
+ 上面的 ICE Server 部署之后，Android demo 请求时会报 404，最终定位是因为 demo 用 HttpURLConnection 时设置了 `connection.setDoOutput(true);`，注释掉就好了，[详见这个 commit](https://github.com/Piasy/AppRTC-Android/commit/88f18b8142392ec683c6df4db64d004632b60af7)；

四个 Server 都部署好之后，就可以开心的开始视频聊天啦！

最后再解释一下 Docker 镜像开放的端口：

+ `8080` 用于 Room Server；
+ `8089` 用于 Signal Server；
+ `3033` 用于 ICE Server；
+ `3478` 和 `59000-65000` 用于 TURN/STUN Server；
