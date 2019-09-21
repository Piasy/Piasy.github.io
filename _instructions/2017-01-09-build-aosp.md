# 导入 AOSP 源码查看、开发、调试

macOS 10.14.6 (18G87), 10.3 (10G8)

``` bash
# 注意切换环境，保证 python 2.7 的环境
curl https://mirrors.tuna.tsinghua.edu.cn/git/git-repo -o repo
chmod +x repo
wget https://mirrors.tuna.tsinghua.edu.cn/aosp-monthly/aosp-latest.tar
tar xf aosp-latest.tar
cd aosp
../repo sync

# 切换分支（tag），只需重新执行 repo init && repo sync，
# 具体 tag，需要去 https://source.android.com/setup/start/build-numbers 查看欲编译的设备支持哪些 tag
../repo init -u https://aosp.tuna.tsinghua.edu.cn/platform/manifest -b android-9.0.0_r36
../repo sync

bash
make clobber # 更新代码后执行
# 下面的关闭 shell 之后都需要执行
source build/envsetup.sh
lunch
make -j4

mmm <path to the module>

# 刷入，需要继续在 make 所在的 bash 里
fastboot flashall -w
```

## 错误解决

切换分支后，可能会报如下错：

``` bash
internal error: could not open symlink hardware/qcom/sdm710/Android.bp; its target (display/os_pickup.bp) cannot be opened
```

删掉 `hardware/qcom/` 目录后，重新 sync，就可以了。

---

make 报错：

``` bash
ld: in '/usr/local/lib/libunwind.dylib', file was built for x86_64 which is not the architecture being linked (i386): /usr/local/lib/libunwind.dylib for architecture i386
```

可能是 brew 安装了新的 llvm@4，通过 `brew uninstall --ignore-dependencies llvm@4` 先临时卸载之。另，遇到上述错误后，一定要 clean 后重新编译，否则可能刷入手机后，手机无法启动。

刷入编出来的镜像之前，先刷一下对应版本的 factory image，因为编译出来的结果不包含 vendor.img，否则刷入手机后可能无法启动。

---

internal error: Could not find a supported mac sdk: ["10.10" "10.11" "10.12" "10.13"]

Xcode最新版本自带是的 MacOSX10.14.sdk，Google 未支持到该版本，可以到 https://github.com/phracker/MacOSX-SDKs/releases 下载 ”10.10” “10.11” “10.12” “10.13” 的其中一个sdk，解压后创建软链接即可：

``` bash
sudo ln -s ~/tools/MacOSX10.11.sdk /Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX10.11.sdk
```

## 编译 kernel

编译 kernel 需要在 Linux 上进行。

``` bash
# lsb_release -a
LSB Version:	core-9.20170808ubuntu1-noarch:security-9.20170808ubuntu1-noarch
Distributor ID:	Ubuntu
Description:	Ubuntu 18.04.1 LTS
Release:	18.04
Codename:	bionic
```

Pixel kernel 下载、编译：

``` bash
# 注意切换环境，保证 python 2.7 的环境
mkdir kernel && cd kernel
../repo init -u https://aosp.tuna.tsinghua.edu.cn/kernel/manifest -b android-msm-marlin-3.18-pie-qpr2
../repo sync
./build/build.sh
```

`build.sh` 最后会有个报错：

``` bash
...
  DTC     arch/arm64/boot/dts/htc/msm8996pro-v1.1-htc_sailfish-xd.dtb
  LZ4     arch/arm64/boot/Image.lz4
  CAT     arch/arm64/boot/Image.lz4-dtb
+ set +x
========================================================
 Running extra build command(s):
+ eval python build/buildinfo/buildinfo.py
++ python build/buildinfo/buildinfo.py
python: can't open file 'build/buildinfo/buildinfo.py': [Errno 2] No such file or directory
```

没关系，有了 `arch/arm64/boot/Image.lz4-dtb` 就够了，它的完整路径为 `out/android-msm-marlin-3.18/private/msm-google/arch/arm64/boot/Image.lz4-dtb`。

在已经刷入了自编 AOSP 的机器上，刷入自编 kernel（重启就会被还原）：

``` bash
adb reboot bootloader
fastboot boot Image.lz4-dtb
```

若要持久化，则把 image 放入 AOSP 代码库的 `device/google/marlin-kernel/` 目录下，重新编译、刷入 AOSP 即可。
