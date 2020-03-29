# 要点

+ ObjC 静态库：回调，对象 ObjC -> Kotlin -> ObjC，`__bridge` cast 导致 `EXC_BAD_ACCESS`；
+ native 共享源码；
+ native 回调 Kotlin，其他线程，`kotlin.native.initRuntimeIfNeeded()`；
+ native 返回字符串；
+ HTTP，HTTPS；
+ SocketIO，HTTPS；
+ Windows 静态库？

## Windows

`.\gradlew assemble` 时可能报错 `Java Could not reserve enough space for object heap error`，[需要添加环境变量 `_JAVA_OPTIONS` 取值 `-Xmx512M`](https://stackoverflow.com/a/24406013/3077508)。

另需要安装 64 位 jdk，否则可能报错 `Can't load AMD 64-bit .dll on a IA 32-bit platform`。
