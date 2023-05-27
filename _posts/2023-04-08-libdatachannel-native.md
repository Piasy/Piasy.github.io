---
layout: post
title: 封装 libdatachannel Android 库
tags:
    - 实时多媒体
    - WebRTC
---

差不多三年前，我和 [cdnbye](https://cdnbye.com/en/) 合作了[对 WebRTC 的极致裁剪](/2020/05/10/WebRTC-Build-DataChannel-Only/index.html)，今天我们再次合作，对 [paullouisageneau/libdatachannel](https://github.com/paullouisageneau/libdatachannel) 做一个封装，以便在 Android 平台上方便的使用。

因为这个库是对 WebRTC 网络传输层的独立实现，短小精悍，而且据 Pion 作者说 libdatachannel 的内存开销只需要 libWebRTC 的 1/3。

话不多说，咱们直接开始 :)

## 编译 libdatachannel

libdatachannel 项目本身有很好的 cmake 支持，所以我们只需要创建一个 gradle 项目，引用它的 `CMakeLists.txt` 即可。

不过我在配置好之后，sync 报错：

```bash
CMake Error at /Users/piasy/tools/android-sdk/cmake/3.22.1/share/cmake-3.22/
Modules/FindPackageHandleStandardArgs.cmake:230 (message):
Could NOT find OpenSSL, try to set the path to OpenSSL root folder in the
system variable OPENSSL_ROOT_DIR (missing: OPENSSL_CRYPTO_LIBRARY
OPENSSL_INCLUDE_DIR) (Required is at least version "1.1.0")
Call Stack (most recent call first):
/Users/piasy/tools/android-sdk/cmake/3.22.1/share/cmake-3.22/Modules/
FindPackageHandleStandardArgs.cmake:594 (_FPHSA_FAILURE_MESSAGE)
/Users/piasy/tools/android-sdk/cmake/3.22.1/share/cmake-3.22/Modules/
FindOpenSSL.cmake:574 (find_package_handle_standard_args)
deps/libsrtp/CMakeLists.txt:77 (find_package)
```

看错误信息，想必是项目用到了 OpenSSL，想想也是合理的，WebRTC 项目是自己带了 BoringSSL，那我们这里就需要自己配置好 OpenSSL 路径了。

不过我在搜索 Android NDK 怎么配置 OpenSSL 的时候，意外发现了一个好东西：在 [Native Dependencies in Android Studio 4.0](https://android-developers.googleblog.com/2020/02/native-dependencies-in-android-studio-40.html) 这篇官方博客中，发布了 [prefab](https://github.com/google/prefab)。使用这套工具发布包含 native 库的依赖，在其他项目里也可以利用这套工具很方便的引用 native 库，工具会自动下载头文件、native 库，并在调用 cmake 时传入相关查找路径，在 cmake 中可以直接 `find_package`。而且常用的项目，比如 openssl, jsoncpp, curl 都已经预先发布了，我们添加一个依赖就行。

在 AGP 7.x 中，我们可以这样启用 prefab：

```gradle
android {
  defaultConfig {
    ...
  }

  buildFeatures.prefab = true
}
```

并添加 openssl 依赖：

```gradle
dependencies {
  implementation 'com.android.ndk.thirdparty:openssl:1.1.1q-beta-1'
}
```

但是配置好之后，还是报上面的错误。于是我尝试在新建的空白项目里实验 prefab，按照上面的方法配置好 prefab 之后，在 `CMakeLists.txt` 里查找 openssl：

```cmake
find_package(openssl REQUIRED CONFIG)
```

结果也报错：

```bash
CMake Error at /Users/piasy/Downloads/MyApplication/app/.cxx/Debug/w1j4b3t4/prefab/
arm64-v8a/prefab/lib/aarch64-linux-android/cmake/curl/curlConfig.cmake:1 (find_package):
Could not find a package configuration file provided by "openssl" with any
of the following names:
```

仔细看还有这个报错：

```bash
[CXX1212] /Users/piasy/Downloads/MyApplication/app/src/main/cpp/CMakeLists.txt
debug|arm64-v8a : User is using a static STL but library requires a shared STL [//curl/curl]
```

经过和官方 [ndk-sample prefab/curl-ssl](https://github.com/android/ndk-samples/tree/master/prefab/curl-ssl) 的一番比对，最终发现是少了这个配置：

```gradle
android {
  defaultConfig {
    externalNativeBuild {
      cmake {
        arguments '-DANDROID_STL=c++_shared'
      }
    }
  }
}
```

也就是必须指定 STL 为 `c++_shared`，其实一开始也搜出了 prefab 的相关 issue：[OpenSSL forces c++_shared – why?](https://github.com/google/prefab/issues/134)，不过一开始我没仔细看出缘由来。

不过当我在编译 libdatachannel 的项目里加上这个配置之后，还是报最开始的错误。经过一番比对、尝试，最终发现主要是 libdatachannel 及其依赖的项目里，查找 openssl 时使用的名字和 prefab 这套工具链准备的名字不一样：prefab 是全小写（openssl），libdatachannel 查找的是 OpenSSL。

libdatachannel 里的使用我就加了个判断，对 Android 平台使用不同的名字：

```cmake
if(ANDROID)
    find_package(openssl REQUIRED)
    target_link_libraries(datachannel PRIVATE openssl::ssl)
    target_link_libraries(datachannel-static PRIVATE openssl::ssl)
else()
    find_package(OpenSSL REQUIRED)
    target_link_libraries(datachannel PRIVATE OpenSSL::SSL)
    target_link_libraries(datachannel-static PRIVATE OpenSSL::SSL)
endif()
```

而对于它依赖项目里使用到的地方，刚好 WebSocket 和 media transport 功能目前都不需要，所以就可以通过禁用相关功能规避：

```gradle
    externalNativeBuild {
      cmake {
        arguments '-DANDROID_STL=c++_shared',
            '-DNO_WEBSOCKET=ON', '-DNO_MEDIA=ON',
            '-DNO_EXAMPLES=ON', '-DNO_TESTS=ON'
      }
    }
```

这样 sync 终于通过了，也顺利编译起来了，不过链接的时候报错：

```bash
undefined symbol: BIO_s_mem
undefined symbol: BIO_new
undefined symbol: BIO_write
undefined symbol: PEM_read_bio_X509
undefined symbol: X509_free
undefined symbol: BIO_free
undefined symbol: PEM_read_bio_PrivateKey
undefined symbol: EVP_PKEY_free
undefined symbol: X509_new
undefined symbol: BN_new
undefined symbol: BN_free
undefined symbol: X509_NAME_new
undefined symbol: X509_NAME_free
undefined symbol: EVP_PKEY_new
undefined symbol: EC_KEY_new_by_curve_name
undefined symbol: EC_KEY_free
undefined symbol: EC_KEY_set_asn1_flag
undefined symbol: EC_KEY_generate_key
undefined symbol: EVP_PKEY_assign
undefined symbol: RSA_new
```

看来是链接还是出了问题。

在 Android Studio 里看了一下链接命令，只有 `libssl.so`，而用 nm 命令看 `libssl.so` 里的符号，确实没有 `X509_new`，因为它在 `libcrypto.so` 里！

所以还需要在 cmake 里添加 crypto 依赖。_因为只有安卓平台默认是需要链接时找到所有的符号，所以原项目里不加 crypto 依赖也还合理。_

关于控制是否「需要链接时找到所有的符号」，可以在 cmake 里可以通过 `target_link_options(name PRIVATE "LINKER:-z,defs")` 或 `-z,undefs` 控制（defs 是需要，undefs 是不需要）。更多说明，可以阅读 [Dependency related linker options](https://maskray.me/blog/2021-06-13-dependency-related-linker-options#z-defs) 这篇文章。

所以，在 cmake 里增加这个配置即可：

```cmake
target_link_libraries(datachannel PRIVATE openssl::crypto)
```

## 封装 JNI 接口

libdatachannel 提供了原始的 C 接口，但也提供了很好的 C++ 接口，而且 C++ 封装中也有做了很多不错的工作，比如用了消息队列来处理多线程问题，所以还是对 C++ 接口进行封装更合适。

这里我首先就想到了之前用过的 [dropbox/djinni](https://github.com/dropbox/djinni)，_虽然这个项目三年前停止维护了，但项目一直都很稳定，真要遇见什么小问题，自己上手也能处理_。djinni 的介绍和使用说明这里就不展开了，感兴趣的朋友可以看看项目主页。

封装的代码这里也不做赘述，[具体可以查看项目代码](https://github.com/swarm-cloud/datachannel-native/commit/8cc00aaf888c155da896ef655f51886d4f5c41f2)，不过这个过程有两个点值得一提。

首先是 libdatachannel 的 C++ 回调接口都是支持 lambda 表达式的，而 djinni 生成的回调都是 `shared_ptr`，传入的 `shared_ptr` callback 在 lambda 里不能按引用捕获。

比如下面这样按引用去捕获 callback：

```cpp
void PeerConnectionImpl::onLocalDescription(const std::shared_ptr<SdpCallback>& callback) {
    pc_.onLocalDescription([&callback](const rtc::Description& sdp) {
        callback->onSdp(std::string(sdp));
    });
}
```

因为这个 callback 不会在别处有引用，函数返回之后就会被释放，那等到 pc 回调 `onLocalDescription` 时，我们再使用 callback 就会 crash 了。

第二个点就是需要打开 djinni 的 `EXPERIMENTAL_AUTO_CPP_THREAD_ATTACH` 选项，否则 pc 回调时，回调所在的线程没有 attach 到 JVM，就会触发 djinni 的 abort 了。

## 和 libWebRTC 对比

封装完毕功能测试通过之后，当然是需要和 libWebRTC 做一个对比的。

首先我们看看包大小：libdatachannel 关闭 WebSocket 和 media transport 之后，包括我们的 djinni 封装代码，一共是编译了 120 个文件，release aar 里的 `armeabi-v7a/libdatachannel.so` 是 1690716 B，比[极致裁剪的 libWebRTC](/2020/05/10/WebRTC-Build-DataChannel-Only/index.html#%E4%BE%9D%E8%B5%96%E5%88%86%E6%9E%90) 的 1779752 B 小了一丢丢。但 libdatachannel 还有三个额外的库，分别是 `libdatachannel_jni.so` 233552 B，`libssl.so` 469924 B 和 `libcrypto.so` 2197668 B，综合下来包大小倒是没有优势。

然后我们再看看性能测量。

测试时我是起了一百对 pc，每对 pc 之间每隔 50ms 发送两条比较短小的测试消息，然后用 Android Studio 的 Profiler 看内存和 CPU 占用。

| lib                     | mem idle    | mem perf      | cpu perf |
| -----------             | ----------- | -----------   | -------  |
| libDC-debug             | 142MB       | 209MB (303MB) | 56%      |
| libDC-release           | 142MB       | 209MB (303MB) | 44%      |
| libDC-release (native)  | 144MB       | 210MB         | 30%      |
| libWebRTC-release       | 139MB       | 188MB (282MB) | 20%      |

测量结果还是比较出乎意料的：

+ 因为两个 demo 代码几乎一样，所以 idle 状态的内存区别，可能就是多加载了几个库；
+ perf 状态下，libDC 内存增量为 67MB，libWebRTC 的内存增量只有 49MB，括号内是第一次 gc 之前的内存峰值；
+ perf 状态下，libDC 的 CPU 占用也比 libWebRTC 高很多；
+ 我也怀疑过是本地 module 依赖的 debug 配置问题，改为依赖 release aar，发现 CPU 占用有所降低，但是内存占用基本没区别；

那么究竟是 libdatachannel 性能存在问题，还是封装实现的问题，还是测试 demo 的问题，亦或是 Pion 作者的结论有误？朋友们就且听下回分解啦 :)

_文章发布后，花了一点时间快速试了一下抛开封装代码，直接在 native 层做性能测量，测量结果排除了封装代码的问题。_

![内存占用详情](https://imgs.piasy.com/2023-04-11-mem-detail-compare.png)

性能测量截图：

![libWebRTC demo 空闲状态](https://imgs.piasy.com/2023-04-09-libWebRTC-idle.png)
![libWebRTC demo 测试状态](https://imgs.piasy.com/2023-04-09-libWebRTC-perf.png)
![libDC demo 空闲状态](https://imgs.piasy.com/2023-04-09-libDC-idle.png)
![libDC demo 测试状态](https://imgs.piasy.com/2023-04-09-libDC-perf.png)

## 参考文章

+ [Using openSSL for encryption in Android NDK](https://groups.google.com/g/android-ndk/c/jexf1pzREnk)
