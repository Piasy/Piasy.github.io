---
layout: post
title: Android 使用 OpenSSL 库
tags:
    - 实时多媒体
---

## 背景

不久前我在[封装 libdatachannel Android 库](/2023/04/08/libdatachannel-native/index.html)中介绍了通过 prefab 在项目中引入 OpenSSL 依赖，当时确实是解决了项目所需，不过后来碰到了一个兼容性问题，在 Android 5.x 系统上，加载依赖了 OpenSSL 的库时，会 crash：

```bash
E  dlopen("/xxxxxx/lib/arm/libdatachannel.so", RTLD_LAZY) failed: 
dlopen failed: cannot locate symbol "X509_getm_notBefore" referenced by "libdatachannel.so"...
```

是说找不到 `X509_getm_notBefore` 这个符号，但是 apk 打包的 `libcrypto.so` 里确实是有这个符号的。

经过一番搜索，找到了解释，而且就是在 [prefab 的 GitHub issue](https://github.com/google/prefab/issues/112#issuecomment-678484426) 里找到的：

> The libcrypto.so in the OpenSSL package definitely has that symbol.
> IIRC before M system libraries would be loaded preferentially to libraries in the app if they had the same name, so we're getting /system/lib/libcrypto.so instead of the one in the app :(

是说在 Android M 之前的系统上，因为和系统带的 OpenSSL 库符号重名冲突了，所以导致了问题。

prefab 官方始终没有解决这个问题，但这个 issue 下有人也提出了规避方式：要么就是把符号重命名，避免冲突；要么就是使用 OpenSSL 静态库。

看到这里我不禁想到，既然系统里有 OpenSSL 库，为什么我们不能直接用呢？自己打包一份 OpenSSL 的库，反倒是把包体积变大了。

## 使用系统 OpenSSL 库

其实我们自己项目的编译和链接过程，只要提供 OpenSSL 的头文件和 so 库文件，就能成功，至于这些文件是怎么来的，无论是自己编译的，还是从 Android 设备上导出来，都可以。然后把我们自己的库编译出来之后，打包 apk 的时候别带着 OpenSSL 库，而是去加载系统里的 libcrypto.so 和 libssl.so 就行。此外，OpenSSL 的接口是向前兼容的，比如我们使用 Android API 19 使用的 OpenSSL 版本的头文件和库，在新版的 Android 系统上应该也是没有兼容性问题的，当然前提是我们的代码没有用到新版本 OpenSSL 添加的接口。

不过在验证这个方案的时候，也遇到了问题：

```bash
E  library "/system/lib64/libcrypto.so" ("/system/lib64/libcrypto.so") 
needed or dlopened by "/apex/com.android.art/lib64/libnativeloader.so" is not accessible 
for the namespace: xxxxxxx
```

原来是从 Android 7.0 开始，系统的动态库，只有写在 /system/etc/public.libraries.txt 里的库，普通 app 才能够使用，否则加载就会失败，报上面的错误。更详细的说明，可以看[这个 reddit 讨论](https://www.reddit.com/r/androiddev/comments/119gzjv/android_native_development_shared_library_loading/)。

_其实之前在嵌入式设备上有遇到过类似的情况，当时是需要使用 libtinyalsa.so。_

所以使用系统 OpenSSL 库还是不行。

## 使用 OpenSSL 静态库

我们当然可以制作一个 prefab 的 OpenSSL 静态库组件，不过这个工作量有点大，我希望能先直接本地编译出 OpenSSL 静态库之后，在 CMake 项目里使用。_编译命令我是基于 [subins2000/android-openssl-cmake](https://github.com/subins2000/android-openssl-cmake) 稍作修改的_。

在引入编译好的 OpenSSL 库的过程中，我又遇到了最初的 `find_package` 报错问题：

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

就算按照提示设置了 `OPENSSL_ROOT_DIR` 变量，也还是不管用，最终还是经过一番搜索，找到了两种解决方案。

其实我们也不是必须使用 CMake 提供的 `FindOpenSSL.cmake`，自己写一个就好了。

方案一能支持使用 CONFIG 模式的 `find_package` 调用，把如下内容写入 `OpenSSLConfig.cmake`：

```cmake
if (NOT TARGET OpenSSL::Crypto)
    add_library(OpenSSL::Crypto STATIC IMPORTED)
    set_target_properties(OpenSSL::Crypto PROPERTIES
            IMPORTED_LOCATION "${OPENSSL_ROOT_DIR}/lib/libcrypto.a"
            INTERFACE_INCLUDE_DIRECTORIES "${OPENSSL_ROOT_DIR}/include"
            INTERFACE_LINK_LIBRARIES ""
            )
endif ()

if (NOT TARGET OpenSSL::SSL)
    add_library(OpenSSL::SSL STATIC IMPORTED)
    set_target_properties(OpenSSL::SSL PROPERTIES
            IMPORTED_LOCATION "${OPENSSL_ROOT_DIR}/lib/libssl.a"
            INTERFACE_INCLUDE_DIRECTORIES "${OPENSSL_ROOT_DIR}/include"
            INTERFACE_LINK_LIBRARIES ""
            )
endif ()
```

并把 `OpenSSLConfig.cmake` 所在的路径设置给 `OpenSSL_DIR`，或者添加到 `CMAKE_PREFIX_PATH` 里：

```cmake
set(OpenSSL_DIR path-to-openssl-config)
list(APPEND CMAKE_PREFIX_PATH path-to-openssl-config)
```

这样就可以通过 `find_package(OpenSSL REQUIRED CONFIG)` 找到 OpenSSL 库了。

不过 libdatachannel 的 CMakeLists.txt 里用的是 MODULE 模式，也就是 `find_package(OpenSSL REQUIRED)`，上述配置方式不管用，依然还是会调用到 CMake 提供的 `FindOpenSSL.cmake`，依然会报上面的错。

方案二就可以解决，把和上面 `OpenSSLConfig.cmake` 的一样的内容写入 `FindOpenSSL.cmake`，并把它所在的路径添加到 `CMAKE_MODULE_PATH` 中即可：

```cmake
list(APPEND CMAKE_MODULE_PATH path-to-find-openssl)
```

CONFIG 模式和 MODULE 模式，其实只是 CMake 查找时使用的 cmake 文件名规则不一样，一个是 `<LibraryName>Config.cmake` 或者 `<lower-case-library-name>-config.cmake`，一个是 `Find<LibraryName>.cmake`，用哪个都可以。
