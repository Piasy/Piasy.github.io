---
layout: post
title: Janus Gateway HTTPS
tags:
    - 安卓开发
    - OpenGL
---

## 安装

~~~ bash
apt-get -y update && apt-get install -y libmicrohttpd-dev \
    libjansson-dev \
    libnice-dev \
    libssl-dev \
    libsrtp-dev \
    libsofia-sip-ua-dev \
    libglib2.0-dev \
    libopus-dev \
    libogg-dev \
    libini-config-dev \
    libcollection-dev \
    pkg-config \
    gengetopt \
    libtool \
    automake \
    build-essential \
    subversion \
    git \
    cmake \
    unzip \
    zip \
    lsof wget vim sudo rsync cron mysql-client openssh-server supervisor locate

apt-get -y install autoconf automake build-essential libass-dev libfreetype6-dev \
    libsdl1.2-dev libtheora-dev libtool libva-dev libvdpau-dev libvorbis-dev libxcb1-dev libxcb-shm0-dev \
    libxcb-xfixes0-dev pkg-config texinfo zlib1g-dev nasm

apt-get -y update && apt-get install -y --no-install-recommends \
        g++ \
        gcc \
        libc6-dev \
        make \
        pkg-config \

mkdir ~/ffmpeg_sources

cd ~/ffmpeg_sources && \
    wget http://www.tortall.net/projects/yasm/releases/yasm-1.3.0.tar.gz && \
    tar xzvf yasm-1.3.0.tar.gz && \
    cd yasm-1.3.0 && \
    ./configure --prefix="$HOME/ffmpeg_build" --bindir="$HOME/bin"  && \
    make && \
    make install && \
    make distclean

cd ~/ffmpeg_sources && \
    wget http://download.videolan.org/pub/x264/snapshots/last_x264.tar.bz2 && \
    tar xjvf last_x264.tar.bz2 && \
    cd x264-snapshot* && \
    PATH="$HOME/bin:$PATH" ./configure --prefix="$HOME/ffmpeg_build" --bindir="$HOME/bin" --enable-static --disable-opencl --disable-asm && \
    PATH="$HOME/bin:$PATH" make && \
    make install && \
    make distclean

cd ~/ffmpeg_sources && \
    wget http://storage.googleapis.com/downloads.webmproject.org/releases/webm/libvpx-1.5.0.tar.bz2 && \
    tar xjvf libvpx-1.5.0.tar.bz2 && \
    cd libvpx-1.5.0 && \
    PATH="$HOME/bin:$PATH" ./configure --prefix="$HOME/ffmpeg_build" --disable-examples --disable-unit-tests && \
    PATH="$HOME/bin:$PATH" make && \
    make install && \
    make clean

cd ~/ffmpeg_sources && \
    wget -O fdk-aac.tar.gz https://github.com/mstorsjo/fdk-aac/tarball/master && \
    tar xzvf fdk-aac.tar.gz && \
    cd mstorsjo-fdk-aac* && \
    autoreconf -fiv && \
    ./configure --prefix="$HOME/ffmpeg_build" --disable-shared && \
    make && \
    make install && \
    make distclean

cd ~/ffmpeg_sources && \
    wget http://downloads.sourceforge.net/project/lame/lame/3.99/lame-3.99.5.tar.gz && \
    tar xzvf lame-3.99.5.tar.gz && \
    cd lame-3.99.5 && \
    ./configure --prefix="$HOME/ffmpeg_build" --enable-nasm --disable-shared && \
    make && \
    make install && \
    make distclean

cd ~/ffmpeg_sources && \
    wget http://downloads.xiph.org/releases/opus/opus-1.1.2.tar.gz && \
    tar xzvf opus-1.1.2.tar.gz && \
    cd opus-1.1.2 && \
    ./configure --prefix="$HOME/ffmpeg_build" --disable-shared && \
    make && \
    make install && \
    make clean

cd ~/ffmpeg_sources && \
    wget https://ffmpeg.org/releases/ffmpeg-snapshot.tar.bz2 && \
    tar xjvf ffmpeg-snapshot.tar.bz2 && \
    cd ffmpeg && \
    PATH="$HOME/bin:$PATH" PKG_CONFIG_PATH="$HOME/ffmpeg_build/lib/pkgconfig" ./configure \
    --prefix="$HOME/ffmpeg_build" \
    --pkg-config-flags="--static" \
    --extra-cflags="-I$HOME/ffmpeg_build/include" \
    --extra-ldflags="-L$HOME/ffmpeg_build/lib" \
    --bindir="$HOME/bin" \
    --enable-gpl \
    --enable-libass \
    --enable-libfdk-aac \
    --enable-libfreetype \
    --enable-libopus \
    --enable-libtheora \
    --enable-libvorbis \
    --enable-libvpx \
    --enable-libx264 \
    --enable-nonfree && \
    PATH="$HOME/bin:$PATH" make && \
    make install && \
    make distclean && \
    hash -r

cd ~/ffmpeg_sources && \
    COTURN="4.5.0.6" && \
    wget https://github.com/coturn/coturn/archive/$COTURN.tar.gz && \
    tar xzvf $COTURN.tar.gz && \
    cd coturn-$COTURN && \
    ./configure && \
    make && make install

cd ~/ffmpeg_sources && \
    LIBWEBSOCKET="2.2.1" && vLIBWEBSOCKET="v2.2.1" && \
    wget https://github.com/warmcat/libwebsockets/archive/$vLIBWEBSOCKET.tar.gz && \
    tar xzvf $vLIBWEBSOCKET.tar.gz && \
    cd libwebsockets-$LIBWEBSOCKET && \
    mkdir build && \
    cd build && \
    cmake -DCMAKE_INSTALL_PREFIX:PATH=/usr -DCMAKE_C_FLAGS="-fpic" -DLWS_MAX_SMP=1 -DLWS_IPV6="ON" .. && \
    make && make install

cd ~ && \
    wget https://storage.googleapis.com/golang/go1.9.1.linux-amd64.tar.gz && \
    tar -C /usr/local -xzf go$VERSION.$OS-$ARCH.tar.gz

mkdir -p "/go/src" "/go/bin" && chmod -R 777 "/go"

cd ~/ffmpeg_sources && \
    git clone https://boringssl.googlesource.com/boringssl && \
    cd boringssl && \
    sed -i s/" -Werror"//g CMakeLists.txt && \
    mkdir -p build  && \
    cd build  && \
    cmake -DCMAKE_CXX_FLAGS="-lrt" ..  && \
    make  && \
    cd ..  && \
    sudo mkdir -p /opt/boringssl  && \
    sudo cp -R include /opt/boringssl/  && \
    sudo mkdir -p /opt/boringssl/lib  && \
    sudo cp build/ssl/libssl.a /opt/boringssl/lib/  && \
    sudo cp build/crypto/libcrypto.a /opt/boringssl/lib/

apt-get remove -y libsrtp0-dev
cd ~/ffmpeg_sources && \
    wget https://github.com/cisco/libsrtp/archive/v2.0.0.tar.gz && \
    tar xfv v2.0.0.tar.gz && \
    cd libsrtp-2.0.0 && \
    ./configure --prefix=/usr --enable-openssl && \
    make shared_library && sudo make install

cd ~/ffmpeg_sources && \
    GDB="8.0" && \
    wget ftp://sourceware.org/pub/gdb/releases/gdb-$GDB.tar.gz && \
    tar xzvf gdb-$GDB.tar.gz && \
    cd gdb-$GDB && \
    ./configure && \
    make && \
    make install

cd ~ && \
    git clone https://github.com/meetecho/janus-gateway.git && \
    cd janus-gateway && \
    sh autogen.sh && \
    ./configure --prefix=/opt/janus && \
    make && \
    make install
~~~

修改 `~/.bashrc`：

~~~ sh
export GOPATH=/go
export PATH=$GOPATH/bin:/usr/local/go/bin:/opt/janus/bin:$PATH
~~~

## HTTPS 模式运行

+ 修改 `/opt/janus/etc/janus/janus.transport.http.cfg`，设置 `https yes`；
+ 运行 `/opt/janus/bin/janus`；
+ 安装 [Caddy](https://caddyserver.com/)，下载后解压到 `~/caddy`；
+ 编辑 Caddyfile，注意把 `192.168.50.4` 替换为实际 IP：

~~~ bash
192.168.50.4:4443 {
    gzip
    root /vagrant/janus-gateway/html/
    log /vagrant/caddy/access.log
    tls /opt/janus/share/janus/certs/mycert.pem /opt/janus/share/janus/certs/mycert.key
}
~~~

+ 进入 `~/janus-gateway/html`，运行 `~/caddy/caddy -conf=~/caddy/Caddyfile`；
+ 打开 chrome，访问 `https://192.168.50.4:4443`，提示证书错误，选择继续前往；

## RTMP 推收流

+ 部署 SRS 服务器：

~~~ bash
git clone https://github.com/ossrs/srs
cd srs/trunk
./configure && make
./objs/srs -c conf/rtmp.conf
~~~

+ ffmpeg 推流：`ffmpeg -re -i video.mp4 -acodec copy -vcodec copy -f flv -y rtmp://192.168.50.4/live/livestream`；
+ ffplay 收流：`ffplay rtmp://192.168.50.4/live/livestream`；

## RTMP -> RTP -> WEBRTC

~~~ bash
~/bin/ffmpeg -y -i "rtmp://0.0.0.0:8898/rtmp_relay/test live=1" \
-c:v libx264 -profile:v main -s 640x480 -an -preset ultrafast \
-tune zerolatency -f rtp rtp://0.0.0.0:8888
~~~

## 包格式分析

### MJR

开头是 meta 信息，后面每个 `MEETECHO` 是分隔符，接着两个字节为 RTP 包长度，接着就是 RTP 包了。

`janus_pp_h264_preprocess` 是为了计算视频尺寸、帧率。

## MISC

+ [Vagrant ubuntu 账号密码](https://askubuntu.com/a/875659)；
+ [Vagrant mac 下启用桥接网络](https://apple.stackexchange.com/a/201184/207466)，但共享目录就不能用 NFS 了，去掉 Vagrantfile 中的 `type nfs` 即可；
