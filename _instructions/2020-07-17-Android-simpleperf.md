# Android simpleperf

## 使用

[建议在 Android N 及以上设备使用](https://android.googlesource.com/platform/system/extras/+/master/simpleperf/doc/README.md#why-we-suggest-profiling-on-android-n-devices)，但不是必须。

建议用最新的 Android NDK 里的 `app_profiler.py` 脚本，比如 r21d。

运行步骤：

1. 编译、安装 debug 版 apk，略；
1. 收集 profiling data：`python <ndk 目录>/simpleperf/app_profiler.py -p <app 包名> -a <启动 activity 包名类名> --compile_java_code -r '-e task-clock:u -f 1000 --duration 10 -g'`；等命令行不再刷输出后，就可以操作 APP 了，期间的 CPU 占用，都会被记录下来；
1. 生成报告：在第 2 步的相同目录下，执行 `python <ndk 目录>/simpleperf/report_html.py`；

第 2 步参数说明：

+ `--compile_java_code` 用来把 Java 代码预编译为 native 指令，在 Android P 及以上的设备上 profiling 时，无需此选项；
+ `-a` 的参数取值，包名类名之间是 `.` 分隔；
+ `-r` 里的 `-f` 指定一秒内最多收集的事件数量，默认 4000；
+ `-r` 里的 `--duration 10` 用来指定 `simpleperf` 在手机上的运行时间，单位秒，可以指定很长，然后通过 `Ctrl + c` 来停止收集；
+ `-r` 里的 `-g` 可以用来生成调用栈（call graph），但可能有的设备不支持；
+ 还可以通过 `-lib` 选项指定 debug .so 的目录（包含架构名子目录），用来符号化 .so 的调用栈，非必须；

## 可能的问题

+ `Event type 'XXX' is not supported on the device`: 设备不支持 `-r` 选项里 `-e` 后指定的事件类型，可在 adb shell 里运行 `simpleperf list` 查看支持的事件类型，一般 `task-clock` 都支持，`cpu-cycles` 可能有的硬件不支持；
+ `dwarf callchain sampling is not supported on this device`: 把 `-r` 选项里的 `-g` 去掉；
+ 如果第 2 步还有什么其他的问题，可以查看控制台的日志，找到出错那步执行的命令，然后在 adb shell 里执行，也许可以看到更多的输出信息；
+ profiling 过一次后，下次 profiling 前，需要删掉当前目录的 `binary_cache` 目录，否则脚本会报错 `No such file or directory`；不过只要第 2 步里把 `perf.data` pull 下来了，即便有这个报错，也可以忽略；

可能用最新的 NDK 里的 `app_profiler.py` 会在手机上执行 simpleperf 时报错 `Illegal instruction (core dumped)`。这时可以试试使用老版本的 NDK，比如 r15c。

r15c 的 `app_profiler.py` 只接收 `--config` 参数，可以复制 `<ndk 目录>/simpleperf/app_profiler.config`，并修改其中的 `app_package_name`、`launch_activity` 和 `record_options`，它们依次对应于新版本的 `-p`、`-a` 和 `-r` 参数。

## 参考

+ [Simpleperf: README](https://android.googlesource.com/platform/system/extras/+/master/simpleperf/doc/README.md)
+ [Simpleperf: Android application profiling](https://android.googlesource.com/platform/system/extras/+/master/simpleperf/doc/android_application_profiling.md)
+ [SimplePerf: Difference between Hardware events and raw events?](https://github.com/android/ndk/issues/550)
