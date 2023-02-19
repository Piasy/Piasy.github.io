---
layout: post
title: mediasoup 快速入门
tags:
    - 实时多媒体
    - WebRTC
    - mediasoup
---

差不多四年前，我开始了和 OWT 的故事，并分享了 [OWT Server 快速入门](/2019/04/14/OWT-Server-Quick-Start/index.html)，这四年间 mediasoup 当然也是时常接触到的，不过因为一直没有需求，也就基本没碰过，今天就和 mediasoup 来一个虽迟但到的相会吧 :)

## 编译、运行 demo

官方 demo 项目的组成比 OWT 简单许多，就是 [mediasoup-demo](https://github.com/versatica/mediasoup-demo)，一个 server 目录，一个 app 目录，官方的 README 说得也比较清楚。但是我在把 demo 跑起来的过程中还是遇到了一些问题的。

我一开始是在 Linux docker 里搞的（因为他也没说能直接在 macOS 上搞啊），用的当然是 [rtcdev-docker](https://github.com/HackWebRTC/rtcdev-docker) 了。

首先是安装 nvm 以及 node（这里我安装的是 v16）：

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.3/install.sh | bash
source ~/.bashrc
nvm install 16
```

遇到的第一个问题就是要把 ssh key 添加到 GitHub 账户，否则会报错：

```bash
npm ERR! Error while executing:
npm ERR! /usr/bin/git ls-remote -h -t ssh://git@github.com/versatica/mediasoup.git
npm ERR!
npm ERR! fatal: failed to stat '/root/mediasoup-demo/server': Permission denied
npm ERR!
npm ERR! exited with error code: 128
```

接着，需要安装 python3-pip，否则会报错（注意第一行 `--system` 那个可以忽略，关键是最后的 `No module named pip`）：

```bash
npm ERR! # `--system` is not present everywhere and is only needed as workaround for
npm ERR! # Debian-specific issue (copied from https://github.com/gluster/gstatus/pull/33),
npm ERR! # fallback to command without `--system` if the first one fails.
npm ERR! /usr/bin/python3 -m pip install --system --target=/home/ubuntu/.npm/_cacache/tmp/git-cloneKmWOcu/worker/out/pip pip setuptools || \
npm ERR! 	/usr/bin/python3 -m pip install --target=/home/ubuntu/.npm/_cacache/tmp/git-cloneKmWOcu/worker/out/pip pip setuptools || \
npm ERR! 	echo "Installation failed, likely because PIP is unavailable, if you are on Debian/Ubuntu or derivative please install the python3-pip package"
npm ERR! Installation failed, likely because PIP is unavailable, if you are on Debian/Ubuntu or derivative please install the python3-pip package
npm ERR! # Install `meson` and `ninja` using `pip` into custom location, so we don't
npm ERR! # depend on system-wide installation.
npm ERR! /usr/bin/python3 -m pip install --upgrade --target=/home/ubuntu/.npm/_cacache/tmp/git-cloneKmWOcu/worker/out/pip  meson==0.61.5 ninja==1.10.2.4
npm ERR! Makefile:94: recipe for target 'meson-ninja' failed
npm ERR! make: Leaving directory '/home/ubuntu/.npm/_cacache/tmp/git-cloneKmWOcu/worker'
npm ERR! npm WARN using --force Recommended protections disabled.
npm ERR! /usr/bin/python3: No module named pip
npm ERR! /usr/bin/python3: No module named pip
npm ERR! /usr/bin/python3: No module named pip
```

然后是 app 执行 npm install 的时候会报错：

```bash
npm ERR! code EINVALIDTAGNAME
npm ERR! Invalid tag name ">=^16.0.0" of package "react@>=^16.0.0": Tags may not have any characters that encodeURIComponent encodes.
```

需要把命令改为 `npm install --legacy-peer-deps`。

至此编译（`npm install`）环节就成功了，要运行 demo 还需要做一些配置，这一点在官方 README 里也有提到。

主要是创建自签名 SSL 证书，可以参考 [mediasoup-demo 实践](https://blog.csdn.net/aggresss/article/details/104858479/)这篇文章里[分享的脚本](https://github.com/aggresss/playground-cpp/blob/master/certs/autogen.sh)创建（[更多关于自签名证书的介绍，可以看这篇文章](https://devopscube.com/create-self-signed-certificates-openssl/)）：

```bash
# CA key
openssl genrsa -out ca.key 2048
# CA csr
openssl req -new -subj "/CN=ca" -key ca.key -out ca.csr
# CA crt
openssl x509 -req -in ca.csr -out ca.crt -signkey ca.key -days 3650

# server key
openssl genrsa -out server.key 2048
# server.csr
openssl req -new -subj "/CN=server" -key server.key -out server.csr
# server.crt
openssl x509 -req -in server.csr -out server.crt -CA ca.crt -CAkey ca.key -CAcreateserial -days 3650
# server.crt verify
openssl verify -CAfile ca.crt  server.crt
```

上述命令会创建出 `server.crt` 和 `server.key` 两个文件，分别是自签名证书的公钥和私钥，运行 demo 时必须用到。

创建好证书（并放到 mediasoup-demo/server 目录）之后，在 shell 里用如下命令即可运行 server 程序（注意使用实际的路径和 IP）：

```bash
cd mediasoup-demo/server
cp config.example.js config.js

export HTTPS_CERT_FULLCHAIN=/root/src/mediasoup-demo/server/server.crt
export HTTPS_CERT_PRIVKEY=/root/src/mediasoup-demo/server/server.key
export MEDIASOUP_ANNOUNCED_IP=172.16.11.239

npm start
```

运行 app 需要在另一个 shell 里，也需要 export 上面三个环境变量，命令还是 npm start，但是会报错：

```bash
/root/src/mediasoup-demo/app/gulpfile.js
  58:21  error  Unexpected use of file extension "json" for "./package.json"  import/extensions

/root/src/mediasoup-demo/app/lib/components/PeerView.jsx
  455:6  error  Invalid property 'playsInline' found on tag 'audio', but it is only allowed on: video  react/no-unknown-property
```

这就是 demo 代码的一点小问题了，文件和行号错误信息里都有，第一个需要把 `require('./package.json');` 改成 `require('./package');`，第二个是把 `playsInline` 删掉。

到这里 demo 就顺利跑起来了，在浏览器里打开 app shell 里打印的地址，在安全提示的页面点继续访问即可。

但这里我还遇到了另一个问题，那就是 websocket 连接失败，浏览器调试模式也没看到 connection refused 之类的错误信息，而 Google 出来[他们官方论坛里的帖子](https://mediasoup.discourse.group/t/websocket-connection-failed-error-in-connection-establishment-net-err-connection-refused/1240)，也只说让用有效的证书，我一度怀疑是 mediasoup 不支持自签名证书。不过我后来发现我运行 docker 镜像的时候，没有打开 4443 端口的映射，加上之后 websocket 连接就没问题了。

最后，其实 mediasoup demo 是可以在 macOS 上编译、运行的，而且可以把 [mediasoup](https://github.com/versatica/mediasoup) 项目单独 clone 下来、编译，然后在 demo 里引用本地仓库，避免 `npm install` 过程中网络原因导致失败耽误更多时间。

具体做法就是 clone mediasoup 仓库后，在 mediasoup 目录中执行 `npm install`，再修改 `mediasoup-demo/server/package.json`，把里面的 `github:versatica/mediasoup#v3` 改成 `file:../../mediasoup`，再 `npm install` 即可。当然，在 macOS 上编译、运行 mediasoup demo，上面遇到的问题也一样会遇到，解决方法也是一样的。

用两个浏览器 tab 打开同一个 url，即可实现两人通话的效果：

![](https://imgs.piasy.com/2023-02-18-mediasoup-demo-screenshot.jpg)

## mediasoup 关键逻辑流程

网上已经有了不少相关文章，这里我简单汇总一下：

+ mediasoup 的整体架构设计，看[官方的 mediasoup v3 Design](https://mediasoup.org/documentation/v3/mediasoup/design/)；
+ mediasoup demo server 的代码结构、大概流程分析：[WebRTC 进阶流媒体服务器开发（三）Mediasoup 源码分析之应用层（代码组成、Server.js、Room.js）](https://www.cnblogs.com/ssyfj/p/14847097.html)；
+ mediasoup JS 进程和 C++ 进程的通信原理分析、C++ 大概流程分析：[mediasoup 源码分析-初始化、建立连接及媒体数据的处理流程](https://blog.csdn.net/qiuguolu1108/article/details/115362653)、[WebRTC 进阶流媒体服务器开发（四）Mediasoup 源码分析之底层库](https://www.cnblogs.com/ssyfj/p/14850041.html)、[WebRTC 进阶流媒体服务器开发（五）Mediasoup 源码分析之 Mediasoup 启动过程](https://www.cnblogs.com/ssyfj/p/14851442.html)、[WebRTC 进阶流媒体服务器开发（六）Mediasoup 源码分析之 Mediasoup 主业务流程](https://www.cnblogs.com/ssyfj/p/14855454.html)

## mediasoup 水平扩缩容

这次和 mediasoup 的相会，主要是为了做自动的水平扩缩容，把上面这些文章基本看明白之后，再结合官方 API 文档，以及 [Autoscaling WebRTC with mediasoup](https://www.centedge.io/post/autoscaling-webrtc-with-mediasoup) 这篇文章，基本就可以看明白 [mediasoup horizontal scaling](https://gist.github.com/gurupras/c9ac6609f4c22a515b3aadea8d65bad3) 里的示例代码了，具体的思路和这个过程中遇到的坑，那就未完待续了。

朋友们，再会 :)
