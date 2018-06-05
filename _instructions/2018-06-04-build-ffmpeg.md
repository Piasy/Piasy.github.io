# 编译 ffmpeg

## Android

编译环境：

~~~ bash
macOS   10.13.4 (17E202)
NDK     r15c
ffmpeg  3.4.2
~~~

编译命令：

~~~ bash
./configure \
--enable-cross-compile \
--cross-prefix=$ANDROID_NDK/toolchains/arm-linux-androideabi-4.9/prebuilt/darwin-x86_64/bin/arm-linux-androideabi- \
--sysroot=$ANDROID_NDK/platforms/android-16/arch-arm/ \
--target-os=android \
--arch=arm \
--enable-shared \
--disable-doc \
--disable-programs \
--disable-symver \
--prefix=`pwd`/out

make -j16 install

./configure \
--enable-cross-compile \
--cross-prefix=$ANDROID_NDK/toolchains/x86-4.9/prebuilt/darwin-x86_64/bin/i686-linux-android- \
--sysroot=$ANDROID_NDK/platforms/android-16/arch-x86/ \
--target-os=android \
--arch=x86 \
--enable-shared \
--disable-doc \
--disable-programs \
--disable-symver \
--prefix=`pwd`/out

make -j16 install

./configure \
--enable-cross-compile \
--cross-prefix=$ANDROID_NDK/toolchains/aarch64-linux-android-4.9/prebuilt/darwin-x86_64/bin/aarch64-linux-android- \
--sysroot=$ANDROID_NDK/platforms/android-21/arch-arm64/ \
--target-os=android \
--arch=arm64 \
--enable-shared \
--disable-doc \
--disable-programs \
--disable-symver \
--prefix=`pwd`/out

make -j16 install

./configure \
--enable-cross-compile \
--cross-prefix=$ANDROID_NDK/toolchains/x86_64-4.9/prebuilt/darwin-x86_64/bin/x86_64-linux-android- \
--sysroot=$ANDROID_NDK/platforms/android-21/arch-x86_64/ \
--target-os=android \
--arch=x86_64 \
--enable-shared \
--disable-doc \
--disable-programs \
--disable-symver \
--prefix=`pwd`/out

make -j16 install
~~~

`out/lib` 目录里会有动态库和静态库。

遇到的问题：

mac 下编译报错：`undefined reference to 'swr_alloc()'`，通过 `greadelf` 查看 `.so` 和 `.a` 都确定有相关的符号，最终定位到**因为 ffmpeg 是纯 C 代码，在 C++ 中引用时，`include` 需要用 `extern "C"` 包起来**；

使用静态库时 mac 下链接报错：

~~~ bash
libavformat/hls.c:773: error: undefined reference to 'atof'
libavformat/hlsproto.c:149: error: undefined reference to 'atof'
libavutil/file.c:85: error: undefined reference to 'mmap64'
libavutil/log.c:188: error: undefined reference to 'stderr'
libavutil/log.c:362: error: undefined reference to 'stderr'
libavutil/log.c:362: error: undefined reference to 'stderr'
~~~

是 NDK unified headers 导致，在创建 standalone toolchain 时加上 `--deprecated-headers` 即可。

## iOS

[FFmpeg iOS build script](https://github.com/kewlbear/FFmpeg-iOS-build-script) 亲测可用：

~~~ bash
macOS   10.13.4 (17E202)
Xcode   9.4 (9F1027a)
ffmpeg  3.4.2
~~~

## 参考文章

+ [Decoding audio files with ffmpeg](https://www.targodan.de/post/decoding-audio-files-with-ffmpeg/)
