# 编译 ffmpeg 安卓库

编译环境：

~~~ bash
macOS   10.13.1
NDK     r15c
ffmpeg  3.4
~~~

编译命令：

~~~ bash
$ANDROID_NDK/build/tools/make_standalone_toolchain.py \
    --arch arm \
    --api 16 \
    --stl libc++ \
    --install-dir /Users/piasy/tools/standalone-r15c-arm-16-libc++/

./configure \
--enable-cross-compile \
--cross-prefix=/Users/piasy/tools/standalone-r15c-arm-16-libc++/bin/arm-linux-androideabi- \
--sysroot=/Users/piasy/tools/standalone-r15c-arm-16-libc++/sysroot/ \
--target-os=android \
--arch=arm \
--enable-shared \
--disable-doc \
--disable-ffmpeg \
--disable-ffplay \
--disable-ffprobe \
--disable-ffserver \
--disable-avdevice \
--disable-doc \
--disable-symver \
--extra-cflags="-Os -fpic -marm" \
--prefix=`pwd`/out

make -j16
make install
~~~

遇到的问题：

+ mac 下编译报错：`undefined reference to 'swr_alloc()'`，通过 `greadelf` 查看 `.so` 和 `.a` 都确定有相关的符号，最终定位到**因为 ffmpeg 是纯 C 代码，在 C++ 中引用时，`include` 需要用 `extern "C"` 包起来**；

## 参考文章

+ [Decoding audio files with ffmpeg](https://www.targodan.de/post/decoding-audio-files-with-ffmpeg/)
