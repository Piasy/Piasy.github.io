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
