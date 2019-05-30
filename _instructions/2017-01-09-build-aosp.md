# 导入 AOSP 源码查看、开发、调试

macOS 10.14.4 (18E226), Xcode 9.4 (必须 9.4)

``` bash
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
# 注意切换环境，保证 python 2.7 的环境
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

make 报错：

``` bash
ld: in '/usr/local/lib/libunwind.dylib', file was built for x86_64 which is not the architecture being linked (i386): /usr/local/lib/libunwind.dylib for architecture i386
```

可能是 brew 安装了新的 llvm@4，通过 `brew uninstall --ignore-dependencies llvm@4` 先临时卸载之。另，遇到上述错误后，一定要 clean 后重新编译，否则可能刷入手机后，手机无法启动。

刷入编出来的镜像之前，先刷一下对应版本的 factory image，因为编译出来的结果不包含 vendor.img，否则刷入手机后可能无法启动。
