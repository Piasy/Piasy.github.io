# 导入 AOSP 源码查看、开发、调试

* https://mirrors.tuna.tsinghua.edu.cn/help/AOSP/
* [Mac 10.12 编译 Android 源码](http://www.jianshu.com/p/1513fc9e1a74)
* http://szysky.com/2016/07/12/mac%E7%B3%BB%E7%BB%9Fandroid%E7%BC%96%E8%AF%91%E6%BA%90%E7%A0%81/
* https://android.googlesource.com/platform/development/+/master/tools/idegen/README
* http://ronubo.blogspot.jp/2016/01/debugging-aosp-platform-code-with.html
* https://www.ibm.com/developerworks/cn/opensource/os-cn-android-build/

~~~ bash
curl https://storage.googleapis.com/git-repo-downloads/repo > repo
wget https://mirrors.tuna.tsinghua.edu.cn/aosp-monthly/aosp-latest.tar
tar xf aosp-latest.tar
cd AOSP
../repo sync

bash
# 注意切换环境，保证 python 2.7 的环境
make clobber # 更新代码后执行
# 下面的关闭 shell 之后都需要执行
source build/envsetup.sh
lunch
make -j4

mmm <path to the module>
~~~


## 错误解决

~~~ bash
build/core/combo/mac_version.mk:26: none of the installed SDKs (wifi-serviceac_sdk_versions_installed) match supported versions (10.8 10.9 10.10 10.11), trying 10.8
build/core/combo/mac_version.mk:36: no SDK 10.8 at /Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX10.8.sdk, trying legacy dir
build/core/combo/mac_version.mk:40: *****************************************************
build/core/combo/mac_version.mk:41: * Can not find SDK 10.8 at /Developer/SDKs/MacOSX10.8.sdk
build/core/combo/mac_version.mk:42: *****************************************************
build/core/combo/mac_version.mk:43: *** Stop..  Stop.
~~~

~~~修改 `build/core/combo/mac_version.mk`，修改 `mac_sdk_versions_supported := 10.12`（系统安装的 XCode 版本，`ls /Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/` 获得）~~~

10.12 弃用了 syscall，所以还是得安装老的 SDK，[下载地址](https://github.com/phracker/MacOSX-SDKs/releases)，解压拷贝到`/Applications/XCode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs`。为了避免下次升级的时候再被删除，可以放到 `~/tools/MacOSX10.11.sdk`，再给它创建一个软链接：

~~~ bash
ln -s ~/tools/MacOSX10.11.sdk /Applications/XCode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX10.11.sdk
~~~
