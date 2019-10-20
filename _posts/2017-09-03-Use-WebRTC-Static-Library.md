---
layout: post
title: 使用 WebRTC 静态库进行 NDK 开发
tags:
    - NDK
    - WebRTC
---

前面我们分析 [WebRTC P2P 连接过程](/2017/08/30/WebRTC-P2P-part1/index.html)时，在 C++ 代码的世界里徜徉了那么久，其中有各种各样的功能模块，难道大家看着不心动？反正我是很想把它们剥离出来用的，第一个拿来开刀的当然就是 P2P 模块了。

不过在这之前，我还得好好补补 NDK 开发的相关知识，在这篇文章中，我不会涉及 WebRTC P2P 模块的代码，而是简单用一用它的多线程模块，力图先把路给趟平了。

所谓剥离使用，其实有好几种做法，最干净的就是把相关的源码文件摘出来单独进行编译，不过这件事工作量显然不小，而且未必能在不修改 WebRTC 源码的前提下做到。而最笨的办法就是带着完整的 `libjingle_peerconnection_so.so` 了，但这显然不太符合“剥离”这一词的含义。有没有折衷的办法？当然有，那就是使用 WebRTC 静态库作为依赖，这样我们的代码实际用到了哪些模块，相关源码编译出来的目标文件才会被带进我们的库里面。

**2019.09.11 update**:

WebRTC 从 m73~m74 之间的某个提交开始，安卓编译出来的静态库使用时会报如下错误：

```bash
../../modules/audio_processing/agc2/interpolated_gain_curve.cc:0:
error: undefined reference to 'std::__1::basic_string<char, std::__1::char_traits<char>,
std::__1::allocator<char> > std::__1::operator+<char, std::__1::char_traits<char>,
std::__1::allocator<char> >(char const*, std::__1::basic_string<char, std::__1::char_traits<char>,
std::__1::allocator<char> > const&)'
```

原因是 WebRTC 编译时不再使用 NDK 里的 libc++ 导致的，为了修复这个问题，需要给 src/build 目录[打个 patch](https://gist.github.com/Piasy/4effa9057eb0faff8231d34e589478c3)。相关讨论可以查看[这个 Google Group 讨论：Android libc++ namespace change__1 in m74](https://groups.google.com/forum/#!topic/discuss-webrtc/o-zDdrq09yk)。

## 使用 WebRTC 静态库

### 编译

苦于配置 WebRTC 开发环境的朋友，福音来了！[开箱即用的 WebRTC 开发环境](/2017/06/17/out-of-the-box-webrtc-dev-env/index.html)。

``` bash
gn gen out/android_arm/Debug --args=--args='target_os="android" target_cpu="arm"'
ninja -C out/android_arm/Debug webrtc:webrtc
```

编译完毕后，静态库路径为 `out/android_arm/Debug/obj/webrtc/libwebrtc.a`。

### 头文件

头文件可以从 [sourcey.com](https://sourcey.com/precompiled-webrtc-libraries) 下载，如果没有对应 WebRTC 的版本，则可以自己提取（third party 变化应该不会太大，就用下载的好了）：

``` bash
find webrtc -name "*.h" | xargs -I {} cp --parents {} <path to store headers>
```

### 配置 AndroidStudio 工程

#### 目录结构

``` bash
├──app
   ├──libs
   |  └──webrtc
   |     ├──include
   |     |  ├──third_party
   |     |  └──webrtc
   |     └──lib
   |        └──libwebrtc.a
   ├──build.gradle
   └──CMakeLists.txt
```

#### 测试代码

``` cpp
#include <jni.h>
#include <android/log.h>
#include <webrtc/rtc_base/thread.h>

#define LOGI(...) __android_log_print(ANDROID_LOG_INFO, "TRY_WEBRTC", ##__VA_ARGS__)

struct TestFunctor {
    void operator()() {
        LOGI("TestFunctor run");
    }
};

extern "C" {

JNIEXPORT void JNICALL
Java_com_github_piasy_try_1webrtc_MainActivity_testWebrtc(JNIEnv *env, jclass type) {
    LOGI("test start");

    auto thread = rtc::Thread::Create();
    thread->Start();
    thread->Invoke<void>(RTC_FROM_HERE, TestFunctor());
    thread->Stop();

    LOGI("test end");
}

}
```

要点：

+ `LOGI(...)` 和 `##__VA_ARGS__` 一起实现宏定义的不定长参数列表；
+ C++ 代码里面声明的 JNI 函数，都要用 `extern "C"` 包起来，否则运行时会崩溃：`java.lang.UnsatisfiedLinkError: No implementation found ...`；

#### `CMakeLists.txt`

``` CMake
cmake_minimum_required(VERSION 3.4.1)

set(CWD ${CMAKE_CURRENT_LIST_DIR})

add_library(try-webrtc SHARED
            src/main/cpp/try-webrtc.cpp
            )

include_directories(libs/webrtc/include)
add_definitions(-DWEBRTC_POSIX=1 -DWEBRTC_ANDROID=1 -DWEBRTC_LINUX=1)

# Include libraries needed for try-webrtc lib
target_link_libraries(try-webrtc
                      android
                      log
                      ${CWD}/libs/webrtc/lib/libwebrtc.a
                      )
```

要点：

+ `include_directories` 添加头文件查找路径，否则编译时会找不到头文件；
+ `add_definitions` 添加基础宏定义，否则编译时会报错：`Must define either WEBRTC_WIN or WEBRTC_POSIX.`；
+ `target_link_libraries` 添加预编译的静态库需要用绝对路径，可以通过 `CMAKE_CURRENT_LIST_DIR` 变量获取当前 CMakeLists 文件路径；
+ 更多关于 CMake 的说明，可以查阅[安卓 NDK 入门指南：CMake 基本使用](/2017/08/26/NDK-Start-Guide/#cmake-)，或 [Developer 官网](https://developer.android.com/studio/projects/add-native-code.html)，以及 [CMake 官网](https://cmake.org)；

#### `build.gradle`

``` gradle
android {
    //...
    defaultConfig {
        //...
        ndk.abiFilters = ['armeabi-v7a']
        externalNativeBuild {
            cmake {
                arguments = ['-DANDROID_TOOLCHAIN=clang', '-DANDROID_STL=c++_shared']
                cppFlags '-std=c++11 -fno-rtti'
            }
        }
    }
    externalNativeBuild {
        cmake {
            path "CMakeLists.txt"
        }
    }
    //...
}
```

要点：

+ `-std=c++11` 启用 C++11，否则编译时会报错：`This file requires compiler and library support for the ISO C++ 2011 standard. This support is currently experimental, and must be enabled with the -std=c++11 or -std=gnu++11 compiler options.`；
+ `-DANDROID_STL=c++_static` 使用 libc++ runtime，否则编译时会报一大堆 `undefined reference` 错误（[完整报错见附录](#stl--undefined-reference-buildgradle)，详细解释见后文）；
+ `-fno-rtti` 禁用 RTTI，因为 WebRTC 没有启用 RTTI，我们需要保持一致，否则编译时会报 `undefined reference to 'typeinfo for rtc::MessageHandler'`；

## 异步 RESTful API Client

上面的测试代码显然太无趣，所以我想在 native 层实现一个异步的 RESTful API Client，网络部分用 [restclient-cpp](https://github.com/mrtazz/restclient-cpp) 实现，异步部分则用 WebRTC 封装的线程完成。在实现这一目标的过程中，我遇到的最大的问题就是 STL 不匹配的问题，下面且听我慢慢道来。

### 选择 runtime 版本

安卓系统默认只提供了一个非常简单的 C++ 运行时环境：`system`。它不包含 STL、异常、RTTI 等特性，那我们的代码里面就不能使用这些特性，例如不能使用 `std::string` 或者 `std::vector`，不能使用 `try-catch`，不能使用 `typeid` 操作符。不过好在 NDK 提供了其他辅助的运行时环境，它们能提供不同的 STL 实现，异常和 RTTI 支持。

+ `system`：最基本的 C++ 运行时；
+ `gabi++_static`/`gabi++_shared`：GAbi++ 运行时，包括异常、RTTI 支持；
+ `stlport_static`/`stlport_shared`：STLport 运行时，包括异常、RTTI、STLport 的 STL 实现；
+ `gnustl_static`/`gnustl_shared`：GNU STL 运行时，包括异常、RTTI、GNU STL 的实现；
+ `c++_static`/`c++_shared`：LLVM libc++ 运行时，包括异常、RTTI、LLVM libc++ 的 STL 实现；

如何选择运行时环境，主要考虑两个问题：哪个 STL 实现？静态还是动态？

选择哪个 STL 实现，可以参考以下方面：

+ 是否活跃维护：[STLport 已经好几年没有更新了](https://groups.google.com/d/msg/android-ndk/HZ6I5XfOnP8/i06SgfkJbRoJ)，source forge 上面最新版本还是 14 年 7 月的版本；
+ License：[GNU STL 使用 GPLv3 许可](https://libcxx.llvm.org/#why)，这可是“病毒许可”，小公司也许可以无所谓，人家根本不会注意到你，但大公司就要小心了；
+ libc++ 虽然有将它们都取而代之的雄心壮志，[但仍不够稳定](https://developer.android.com/ndk/guides/cpp-support.html#cs)；
+ 最后但也同样重要的：编译依赖库时能统一为哪一 STL；

由于我们这里并不追求极致的稳定性，当然更主要还是因为 WebRTC 是用 libc++ 编译的，所以我选用了 libc++ 这一运行时环境。前面就已经提到，我们必须使用 libc++ runtime，否则会报一大堆 `undefined reference` 错误，这是因为各个运行时库的二进制接口并不兼容，编译的时候混用 STL 实现，很容易遇到 `undefined reference` 错误。

链接静态依赖库（英文里叫做 link against）时，会把库中的目标文件打包到自己的库里面来，这样就可以不带着依赖库了，但如果我们有多个库都依赖了同一个库，那链接静态依赖库就会导致同样的目标文件被包含了多份，这样既占用了磁盘空间，也会占用运行时内存，[而且 C++ 运行时库如果同时存在多份，可能会导致各种诡异的问题](https://developer.android.com/ndk/guides/cpp-support.html#sr)。此外，我们使用的依赖库可能别的程序也使用了（尤其是 C++ 运行时库），而如果操作系统中运行的多个程序如果要加载同一个动态库，那实际上只会加载一份，所以链接动态依赖库还有可能减少整个系统的内存占用。

最后，依赖库可以动态与静态混用，只要编译使用的 STL 一致即可，而 C++ 运行时库其实也是我们的依赖库，因此我们使用静态还是动态版本，与其他依赖库没有直接关系，即使用 `c++_shared` 或者 `c++_static` 与其他的依赖库没有直接关系。

### 交叉编译安卓平台可用的库

很多开源项目提供的都是利用 Autotools 构建，即 `./autogen.sh && ./configure && make install` 的方式，但这样编出来的库目标平台并不是安卓，无法直接用于 NDK 开发。如果直接使用这样编出来的静态库，编译时可能会报错 `no archive symbol table (run ranlib)`。

因此我们需要交叉编译到安卓目标平台，[NDK 为我们提供了 `make_standalone_toolchain.py` 工具](https://developer.android.com/ndk/guides/standalone_toolchain.html)，可以创建 standalone toolchain，然后我们在 configure 和 make 时使用我们创建的 toolchain 即可。其中最关键的一步就是环境变量的设置（把创建好的 toolchain 目录加入到 PATH 中，且确保环境变量中正确设置 `CC`，`CXX` 等）。

``` bash
# 生成 toolchain，放到 /vagrant/standalone-r15c-arm-libc++/ 目录下
# 没错，我搞了一个 Linux 虚拟机进行编译
$ANDROID_NDK/build/tools/make_standalone_toolchain.py \
    --arch arm \
    --api 16 \
    --stl libc++ \
    --install-dir /vagrant/standalone-r15c-arm-libc++/

./autogen.sh

# 使用 arm-linux-androideabi 作为 host 和 target 进行 configure
env PATH=/vagrant/standalone-r15c-arm-libc++/bin:$PATH \
CC=arm-linux-androideabi-clang \
CXX=arm-linux-androideabi-clang++ \
RANLIB=arm-linux-androideabi-ranlib \
LD=arm-linux-androideabi-ld \
AR=arm-linux-androideabi-ar \
CROSS_COMPILE=arm-linux-androideabi \
./configure --host=arm-linux-androideabi \
--target=arm-linux-androideabi \
--prefix=`pwd`/out

# 编译
env PATH=/vagrant/standalone-r15c-arm-libc++/bin:$PATH \
CC=arm-linux-androideabi-clang \
CXX=arm-linux-androideabi-clang++ \
RANLIB=arm-linux-androideabi-ranlib \
LD=arm-linux-androideabi-ld \
AR=arm-linux-androideabi-ar \
CROSS_COMPILE=arm-linux-androideabi \
make install
```

`make install` 之后，编译好的库会在 `out` 中，可以直接用于 NDK 开发了（ndk-build 或者 CMake）。由于 restclient-cpp 依赖于 curl，而 curl 又依赖于 zlib，因此我们最终会编译三个库，[具体编译步骤见附录](#section-8)。

### 目录结构

``` bash
├──app
   ├──libs
   |  ├──curl-7.55.1
   |  |  ├──include
   |  |  └──lib
   |  ├──restclient-0.4.4
   |  |  ├──include
   |  |  └──lib
   |  ├──webrtc
   |  |  ├──include
   |  |  └──lib
   |  └──zlib-1.2.11
   |     ├──include
   |     └──lib
   ├──build.gradle
   └──CMakeLists.txt
```

### `CMakeLists.txt`

``` CMake
cmake_minimum_required(VERSION 3.4.1)

set(CWD ${CMAKE_CURRENT_LIST_DIR})


add_library(restclient
            STATIC
            #SHARED
            IMPORTED
            )
set_target_properties( # Specifies the target library.
                       restclient

                       # Specifies the parameter you want to define.
                       PROPERTIES IMPORTED_LOCATION

                       # Provides the path to the library you want to import.
                       ${CWD}/libs/restclient-0.4.4/lib/librestclient-cpp.a
                       #${CWD}/src/main/jniLibs/armeabi-v7a/librestclient-cpp.so
                       )
include_directories(${CWD}/libs/restclient-0.4.4/include)


include_directories(${CWD}/libs/zlib-1.2.11/include)
include_directories(${CWD}/libs/curl-7.55.1/include)
include_directories(${CWD}/libs/webrtc/include)


add_definitions(-DWEBRTC_POSIX)

add_library(hack_webrtc SHARED
            src/main/cpp/hack_webrtc.cpp
            src/main/cpp/async_rest_client.cpp
            )

# Include libraries needed for hack_webrtc lib
target_link_libraries(hack_webrtc
                      android
                      log

                      ${CWD}/libs/webrtc/lib/libwebrtc.a

                      restclient
                      ${CWD}/src/main/jniLibs/armeabi-v7a/libcurl.so
                      ${CWD}/src/main/jniLibs/armeabi-v7a/libz.so
                      )
```

要点：

+ 指定预编译的依赖库，有多种方式，可以用 `add_library` + `set_target_properties` 的方式定义一个 library，再在 `target_link_libraries` 加上这个 library 的名字；也可以直接把库文件的绝对路径加到 `target_link_libraries` 中；
+ **有依赖关系的库，在 `target_link_libraries` 里面的顺序是很关键的！例如，restclient 依赖 curl，curl 又依赖 zlib，它们的顺序就必须是 restclient、curl、zlib，否则就会报错 `undefined reference`！**

### 测试代码和完整项目

实现这样一个异步 RESTful API Client 的代码并不算复杂，但也有一些代码量，就不在这里贴出来了，[完整项目可以在 GitHub 获取](https://github.com/Piasy/HackWebRTC)。

## 附录

### [STL 不匹配导致的 `undefined reference` 错误](#buildgradle)

``` bash
Error:error: undefined reference to 'std::__ndk1::ios_base::getloc() const'
Error:error: undefined reference to 'std::__ndk1::locale::use_facet(std::__ndk1::locale::id&) const'
Error:error: undefined reference to 'std::__ndk1::ctype<char>::id'
Error:error: undefined reference to 'std::__ndk1::ios_base::getloc() const'
Error:error: undefined reference to 'std::__ndk1::locale::use_facet(std::__ndk1::locale::id&) const'
Error:error: undefined reference to 'std::__ndk1::locale::~locale()'
Error:error: undefined reference to 'std::__ndk1::ctype<char>::id'
Error:error: undefined reference to 'std::__ndk1::ios_base::getloc() const'
Error:error: undefined reference to 'std::__ndk1::locale::use_facet(std::__ndk1::locale::id&) const'
Error:error: undefined reference to 'std::__ndk1::locale::~locale()'
Error:error: undefined reference to 'std::__ndk1::ios_base::getloc() const'
Error:error: undefined reference to 'std::__ndk1::locale::use_facet(std::__ndk1::locale::id&) const'
Error:error: undefined reference to 'std::__ndk1::locale::~locale()'
Error:error: undefined reference to 'std::__ndk1::ios_base::clear(unsigned int)'
Error:error: undef
Error:error: undefined reference to 'std::__ndk1::ctype<char>::id'
Error:error: undefined reference to 'std::__ndk1::ios_base::clear(unsigned int)'
Error:error: undefined reference to 'std::__ndk1::num_put<char, std::__ndk1::ostreambuf_iterator<char, std::__ndk1::char_traits<char> > >::id'
Error:error: undefined reference to 'std::__ndk1::ctype<char>::id'
Error:error: undefined reference to 'std::__ndk1::ios_base::clear(unsigned int)'
Error:error: undefined reference to 'std::__ndk1::ios_base::clear(unsigned int)'
Error:error: undefined reference to 'std::__ndk1::ios_base::init(void*)'
Error:error: undefined reference to 'std::__ndk1::num_put<char, std::__ndk1::ostreambuf_iterator<char, std::__ndk1::char_traits<char> > >::id'
Error:error: undefined reference to 'std::__ndk1::ios_base::~ios_base()'
Error:error: undefined reference to 'std::__ndk1::ios_base::~ios_base()'
Error:error: undefined reference to 'std::__ndk1::ios_base::~ios_base()'
Error:error: undefined reference to 'std::__ndk1::ios_base::~ios_base()'
Error:error: undefined reference to 'std::__ndk1::locale::locale()'
Error:error: undefined reference to 'std::__ndk1::ios_base::init(void*)'
Error:error: undefined reference to 'std::__ndk1::ios_base::init(void*)'
Error:error: undefined reference to 'std::__ndk1::ios_base::init(void*)'
Error:error: undefined reference to 'std::__ndk1::num_put<char, std::__ndk1::ostreambuf_iterator<char, std::__ndk1::char_traits<char> > >::id'
```

### [交叉编译安卓平台库](#section-4)

`librestclient-cpp.a`：

``` bash
cd /vagrant/restclient-cpp-0.4.4

env PATH=/vagrant/standalone-r15c-arm-libc++/bin:$PATH \
CC=arm-linux-androideabi-clang \
CXX=arm-linux-androideabi-clang++ \
RANLIB=arm-linux-androideabi-ranlib \
LD=arm-linux-androideabi-ld \
AR=arm-linux-androideabi-ar \
CROSS_COMPILE=arm-linux-androideabi \
./configure --host=arm-linux-androideabi \
--target=arm-linux-androideabi \
--prefix=`pwd`/out

# 编辑 Makefile，增加 -I/vagrant/curl-7.55.1/out/include

env PATH=/vagrant/standalone-r15c-arm-libc++/bin:$PATH \
CC=arm-linux-androideabi-clang \
CXX=arm-linux-androideabi-clang++ \
RANLIB=arm-linux-androideabi-ranlib \
LD=arm-linux-androideabi-ld \
AR=arm-linux-androideabi-ar \
CROSS_COMPILE=arm-linux-androideabi \
make install
```

`libcurl.a`：

``` bash
cd /vagrant/curl-7.55.1

env PATH=/vagrant/standalone-r15c-arm-libc++/bin:$PATH \
CC=arm-linux-androideabi-clang \
CXX=arm-linux-androideabi-clang++ \
RANLIB=arm-linux-androideabi-ranlib \
LD=arm-linux-androideabi-ld \
AR=arm-linux-androideabi-ar \
CROSS_COMPILE=arm-linux-androideabi \
./configure --host=arm-linux-androideabi \
--target=arm-linux-androideabi \
--prefix=`pwd`/out

env PATH=/vagrant/standalone-r15c-arm-libc++/bin:$PATH \
CC=arm-linux-androideabi-clang \
CXX=arm-linux-androideabi-clang++ \
RANLIB=arm-linux-androideabi-ranlib \
LD=arm-linux-androideabi-ld \
AR=arm-linux-androideabi-ar \
CROSS_COMPILE=arm-linux-androideabi \
make install
```

`libz.a`：

``` bash
cd /vagrant/zlib-1.2.11

env PATH=/vagrant/standalone-r15c-arm-libc++/bin:$PATH \
CC=arm-linux-androideabi-clang \
CXX=arm-linux-androideabi-clang++ \
RANLIB=arm-linux-androideabi-ranlib \
LD=arm-linux-androideabi-ld \
AR=arm-linux-androideabi-ar \
CROSS_COMPILE=arm-linux-androideabi \
./configure --prefix=`pwd`/out

env PATH=/vagrant/standalone-r15c-arm-libc++/bin:$PATH \
CC=arm-linux-androideabi-clang \
CXX=arm-linux-androideabi-clang++ \
RANLIB=arm-linux-androideabi-ranlib \
LD=arm-linux-androideabi-ld \
AR=arm-linux-androideabi-ar \
CROSS_COMPILE=arm-linux-androideabi \
make install
```

**2018.06.03 Update**: zlib 不需要自己编译，安卓系统已经带着了，在 `target_link_libraries` 里增加 `z` 即可。
