# build WebRTC

+ depot tools 是 chromium 代码库管理工具，包括代码管理、依赖管理、工作流程管理等；
+ Android/Linux、Windows、iOS/macOS WebRTC 本身的代码是同一个仓库，但依赖工具不同，所以不可能搞到放到一起；
+ depot tools 的运行基于 python 2.x 环境，且需要是官方 build（`--version` 选项不能输出额外信息）；
+ 各个系统的 buildtools 是需要运行 `gclient runhooks` 进行下载的，而且是自动检测运行时的系统，只下载当前系统的；

## iOS AppRTCMobile

+ `src/example/BUILD.gn` 中，搜索 `ios_app_bundle("AppRTCMobile")`，为其中增加以下内容：

``` gn
      extra_substitutions = [
        "PRODUCT_BUNDLE_IDENTIFIER=com.google.AppRTCMobile",
      ]
```

+ `src` 目录下执行 `gn gen out/ios --args='target_os="ios" target_cpu="arm64"' --ide=xcode`；
+ 用 Xcode 打开 `src/out/ios/all.xcworkspace`；
+ 选择 `AppRTCMobile` 这个 target，先 run 一下，但最后会提示 `"AppRTCMobile" isn't code signed but requires entitlements. It is not possible to add entitlements to a binary without signing it.`；
+ 在工程文件的 general tab 中，手动选择 Info.plist 为 `src/examples/objc/AppRTCMobile/ios/Info.plist`，设置一个独特的 bundle id；`src/examples/objc/AppRTCMobile/ios/Info.plist` 也改下 bundle id；
+ 勾选「Automatically manage signing」
+ 在工程文件的 general tab 最底部，「Embedded Binaries」添加 WebRTC.framework；
+ 再次 run 即可；
+ Xcode beta 经常提示「can not write to device」，clean 一下即可（clean 并不会清除 ninja 脚本编译的结果，所以不会耗时）；
