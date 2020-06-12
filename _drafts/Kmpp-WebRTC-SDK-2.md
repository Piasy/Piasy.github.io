# 要点

+ ObjC 静态库：回调，对象 ObjC -> Kotlin -> ObjC，`__bridge` cast 导致 `EXC_BAD_ACCESS`；
+ native 共享源码；
+ native 回调 Kotlin，其他线程，`kotlin.native.initRuntimeIfNeeded()`；
+ native 返回字符串；
+ HTTP，HTTPS；
+ SocketIO，HTTPS；
+ Windows 静态库？
+ FFmpeg

## Windows

运行 gradle build 时可能报错 `Java Could not reserve enough space for object heap error`，[需要添加环境变量 `_JAVA_OPTIONS` 取值 `-Xmx512M`](https://stackoverflow.com/a/24406013/3077508)。

另需要安装 64 位 jdk，否则可能报错 `Can't load AMD 64-bit .dll on a IA 32-bit platform`。

ld: DWARF error: mangled line number section (bad file number)

## Linux

std::regex std::bad_cast

https://github.com/yhirose/cpp-httplib/issues/422

## mars xlog

```bash
comm/xlogger/xlogger.h          xlogger2
comm/xlogger/xlogger.h          __xlogger_c_write
comm/xlogger/xloggerbase.c      xlogger_Write
comm/xlogger/xloggerbase.c      __xlogger_Write_impl
log/src/appender.cc             xlogger_appender
```

在 `XLogger::VPrintf` 和 `xlogger_appender` 中，均对日志长度做了限定：不超过 4096。

## profiling

```bash
git clone https://github.com/brendangregg/FlameGraph

perf record -a -F 1000 --call-graph dwarf ./LinuxExample
perf record -a -F 1000 -g ./LinuxExample
perf script | ../FlameGraph/stackcollapse-perf.pl > out.perf-folded
../FlameGraph/flamegraph.pl out.perf-folded > perf-kernel.svg
```

`--call-graph dwarf` 和 `-g` 结果略有不同，可以都试试。

+ [CPU Flame Graphs](http://www.brendangregg.com/FlameGraphs/cpuflamegraphs.html)

## runtime assert: Must be newly frozen

/mnt/agent/work/4d622a065c544371/runtime/src/main/cpp/Memory.cpp:2441: runtime assert: Must be newly frozen

真的需要 freeze 吗？单线程可变，符合原则的。

需要在主线程获取的，确实会有问题。

提交到 worker 的 task 需要 freeze，而 task 肯定是要引用 state 的。

task 靠 queue 提供 state 呢？
