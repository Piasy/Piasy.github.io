# build WebRTC

+ depot tools 是 chromium 代码库管理工具，包括代码管理、依赖管理、工作流程管理等；
+ Android/Linux、Windows、iOS/macOS WebRTC 本身的代码是同一个仓库，但依赖工具不同，所以不可能搞到放到一起；
+ depot tools 的运行基于 python 2.x 环境，且需要是官方 build（`--version` 选项不能输出额外信息）；
+ 各个系统的 buildtools 是需要运行 `gclient runhooks` 进行下载的，而且是自动检测运行时的系统，只下载当前系统的；
+ gn/clang format 下载地址：https://storage.googleapis.com/chromium-clang-format/0679b295e2ce2fce7919d1e8d003e497475f24a3 ,https://storage.googleapis.com/chromium-gn/9be792dd9010ce303a9c3a497a67bcc5ac8c7666 ，替换 hash 值即可，其他 `download_from_google_storage` 的步骤都可以这样解决（替换 bucket 和 hash）；

## iOS AppRTCMobile

+ `src/examples/BUILD.gn` 中，搜索 `ios_app_bundle("AppRTCMobile")`，为其中增加以下内容（bundle id 设置为实际使用的独特 id）：

``` gn
      extra_substitutions = [
        "PRODUCT_BUNDLE_IDENTIFIER=com.github.piasy.AppRTCMobile",
      ]
```

+ `src` 目录下执行 `gn gen out/xcode_ios_arm64 --args='target_os="ios" target_cpu="arm64"' --ide=xcode`；
+ 用 Xcode 打开 `src/out/ios/all.xcworkspace`，run target 选择 `AppRTCMobile`，工程文件的设置 target 也选择 `AppRTCMobile`；

![](https://imgs.piasy.com/2018-04-01-apprtc_ios_setup.png)

+ 在工程文件的 general tab 中，手动选择 `Info.plist` 为 `src/examples/objc/AppRTCMobile/ios/Info.plist`，设置一个独特的 bundle id，该 `Info.plist` 文件也改下 bundle id，否则会提示 `The executable was signed with invalid entitlements.`；
+ 勾选「Automatically manage signing」，选择合适的 team；
+ 在工程文件的 general tab 最底部，「Embedded Binaries」添加 WebRTC.framework；
+ 点击 run 即可；
+ clean 工程并不会清除 ninja 脚本编译的结果，所以不必担心耗时；
+ 更新代码后，可能需要删掉老的 `src/third_party/llvm-build/` 目录，然后执行 `gclient run_hooks` 下载新的 llvm；
+ 在 `examples/objc/AppRTCMobile/ARDAppEngineClient.m` 里，修改 `kARDRoomServerHostUrl`, `kARDRoomServerJoinFormat`, `kARDRoomServerJoinFormatLoopback`, `kARDRoomServerMessageFormat`, `kARDRoomServerLeaveFormat` 这四个变量的域名为实际部署的 AppRTC server 域名/地址；
