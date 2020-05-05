---
layout: post
title: 基于 Kotlin multiplatform 的多平台 WebRTC SDK
tags:
    - 多平台
    - 实时多媒体
    - AvConf
---

将近两年前，我发布了[移动客户端跨平台开发方案探索](/2017/12/16/Mobile-Client-Cross-Platform-Development/index.html)，初步探索了 Javascript, Dart, Java, C++ 这四种语言用作多平台开发的框架。这两年的时间里，Java (J2ObjC) 的方案比较好地满足了鼎点 Android/iOS/Windows SDK 的需求，痒点虽然有，但尚能忍受，这里列几个典型的痒点：

+ iOS 如果客户想要静态库，那客户也就得准备 J2ObjC 的开发环境，磁盘空间大约需要 3GB；
+ Windows 端是启动 JVM 运行 Java 代码，所以需要 JRE 环境；
+ Windows 端开发 APP 尝试 RN Windows 时，启动 JVM 的调用一直抛出异常；

其实在我看来，前两个问题实在不值一提，都有很好的解决方案：开发环境只需要准备一次，磁盘不够换个 512GB 的 MacBook Pro 就能搞定；Windows 制作一个安装程序，自动安装 JRE 即可。

_但没办法，客户给钱就是大爷，再不爽也得舔 :(_

今年年初在 GitHub 上了解到了 Kotlin multiplatform（下面简称 KMPP），稍作尝试后发现还处于早期阶段，就只是保持了关注。两个月前再次尝试时，发现似乎已经基本可用了，过去三周里，我就深度实践了一把，把之前基于 J2ObjC 方案做的一套 WebRTC SDK 利用 KMPP 完成了初步重写，体验还是很不错的，今天在这里分享给大家。

## 概述

首先简单介绍一下这套 SDK，其名曰 AvConf，主要内容是 WebRTC 未涵盖的信令逻辑，目标是要比较方便地对接各种 WebRTC 服务端，比如 Janus Gateway, Licode, OWT 等等，甚至 WebRTC 官方 Demo 的 apprtc server。

信令是纯逻辑代码，如果能多平台复用同一套代码，当然是很好的，但这就需要把平台相关的调用，都封装为接口，比如 HTTP，长连接（WebSocket/SocketIO），WebRTC 等等，不过好在这件事并不复杂。

此外，这两年我在编写客户端 SDK 的过程中，喜欢上了一种编程模式：所有对外的接口，都不限定调用时的线程，内部统一都异步到一个单线程的消息队列中；SDK 内部其他模块触发的事件，也都异步到这个消息队列中进行处理；这样所有的代码都是串行执行的，完全不用考虑线程安全问题，非常省心。

最后还有一点：JSON 解析。在编写信令逻辑时，如果 JSON 解析不能在公共代码中编写，那工作量就多了不止半点。

在 J2ObjC 方案里，通过接口隔离平台相关代码很简单，利用依赖注入的方式，把平台相关的实现注入进去即可；由于公共代码就是纯 Java 代码，Executor 可以直接用；JSON 解析则可以把 Gson 源码都编译为 Objective-C，相当于是公共代码的一部分。

下面我就分别介绍一下 KMPP 这三块内容的处理方案。

## 平台相关代码

首先 Kotlin 语言有 `expect`/`actual` 关键字，利用 `expect` 可以声明平台相关的类或函数，然后在各个平台的实现代码中加上 `actual` 关键字即可。

比如下面这个日志输出模块，在 common 代码中通过 `expect` 进行声明：

```kotlin
expect object Logging {
  fun info(
    tag: String,
    content: String
  )
}
```

在 common 代码中就可以直接调用了：

```kotlin
override fun connect(
  rsUrl: String,
  rid: String
) {
  logInfo("connectToRoom $rsUrl $rid")
}
```

然后我们可以利用腾讯开源的 XLog 进行实现，比如安卓：

```kotlin
actual object Logging {
  @JvmStatic
  actual fun info(
    tag: String,
    content: String
  ) {
    Log.i(tag, "${Thread.currentThread().name} # $content")
  }
}
```

注意，平台相关代码仍然用 Kotlin 进行编写，但因为 Kotlin 支持和 Java/Objective-C/C 进行互相调用，所以没啥问题。

Kotlin 支持直接调用任何 Java 代码，但 Objective-C/C 则没有这么好的待遇，只有系统类/函数内置了翻译，第三方代码则需要自己定义 interop，不过这个过程也不算复杂，首先在 gradle 里进行如下配置：

```gradle
targets {
  iosArm64('ios') {
    compilations.main.cinterops {
      MarsXLog {
        defFile "${rootProject.projectDir}/AvConf/src/iosMain/cinterop/MarsXLog.def"
        includeDirs {
          allHeaders "${rootProject.projectDir}/WrapperProjects/MarsXLog/iOS/wrapper"
        }
      }
    }
    binaries {
      all {
        linkerOpts = ["-framework", "CFNetwork",
            "-framework", "CoreTelephony",
            "-framework", "SystemConfiguration",
            "-framework", "UIKit",
            "-lz",
            "-F${rootProject.projectDir}/libs".toString(),
            "-framework", "mars",
            "-L${rootProject.projectDir}/libs/WrapperLibs/iOS".toString(),
            '-lMarsXLog']
      }
      framework {
        baseName = "AvConf"
        embedBitcode("disable")
      }
    }
  }
}
```

要点如下：

+ 首先在 iosArm64 target 的 `compilations.main.cinterops` 中定义 cinterop；虽然我们定义的是 Objective-C interop，但 gradle 的关键字仍是 cinterop；
+ `MarsXLog` 是这个 interop 的名字，之后生成的 Kotlin 代码的包名将会是 `MarsXLog`；
+ `MarsXLog.def` 的内容如下：

  ```
  language = Objective-C
  headers = PSYMarsXLog.h
  headerFilter = *
  ```

+ 其实 def 文件里可以直接写 Objective-C/C 代码，但由于 XLog 有 C++ 头文件，所以我不得不引入一个 wrapper，PSYMarsXLog 的定义和实现如下：

  ```objective-c
  @interface PSYMarsXLog : NSObject
  + (void)info:(NSString*)tag content:(NSString*)content;
  @end

  @implementation PSYMarsXLog
  + (void)info:(NSString*)tag content:(NSString*)content {
    xlogger2(kLevelInfo, [tag UTF8String], "", "", 0, "%s # %s",
             [[NSThread currentThread].name UTF8String], [content UTF8String]);
  }
  @end
  ```

+ `includeDirs` 和 `linkerOpts` 都可以在 def 文件里写，网上很多示例和教程也都是这么干的，但 def 文件里不能使用变量，就得使用绝对路径，而在 gradle 里则可以不用绝对路径，更利于团队协作；

有了 interop 后，iOS 的实现代码如下：

```kotlin
actual object Logging {
  actual fun info(
    tag: String,
    content: String
  ) {
    MarsXLog.PSYMarsXLog.info(tag, content)
  }
}
```

其他模块的平台相关代码的处理，都可以按照上述套路进行处理，而且如果 iOS 的代码是纯 Objective-C 的，都无需 wrapper，直接在 def 里面编写，或者使用它们的头文件即可。

对于 HTTP 这里我多说一句，虽然有个开源的 Ktor 库支持 KMPP，但由于它用到了 suspend function，而 coroutine 在 Kotlin/Native（即 KMPP 的 iOS/macOS/Windows/Linux 等 Native 平台）上的支持还很有限，所以我就没有用它。

## 单线程消息队列

其实在重写之初，我并未意识到 coroutine 的支持性问题，所以在安卓上基于 Ktor 和 coroutine 写了一版，利用 Dispatcher 即可控制 coroutine 执行所在的线程。但在 iOS 上一跑就报错 `kotlin.IllegalStateException: There is no event loop.` 或 `IncorrectDereferenceException` crash，最终发现是 Kotlin/Native 上只能在主线程使用 coroutine，于是只得作罢。

虽然 kotlinx.coroutine 发布了一个开发者预览版本支持多线程的 coroutine，而且它确实可以 newSingleThreadContext，但在主线程调用的函数中，向其添加 coroutine 会触发异常：`This dispatcher can be used only from a single thread test, but now in MainThread`，所以还是不行。

接着我开始尝试两个月前使用过的 Worker，但发现向 Worker 提交的 lambda（及其引用的对象）必须 freeze，而一旦 freeze 了，就不能修改了，否则会触发 `kotlin.native.concurrent.InvalidMutabilityException: mutation attempt of frozen XXX` 异常。

至此我几乎心灰意冷，还好我在放弃之前去 [Kotlin Slack](https://kotlinlang.slack.com) 上问了一下，得到了一个新的思路：如果使用 `kotlin.native.concurrent` 包下的 Atomic 对象，那即便 freeze 了也可以修改。

所以我给 `WorkerTaskQueue` 增加一个 `AtomicReference` 成员，用来保存所有需要修改的状态，在每次需要修改时，deep copy 一份再做修改，然后 freeze 并替换之。因为 `AtomicReference` 只在 Kotlin/Native 里才有，所以这些逻辑需要在平台相关代码里，于是就作为 `WorkerTaskQueue` 的成员好了。

`WorkerTaskQueue` 的定义如下：

```kotlin
expect class WorkerTaskQueue<T>(state: T) {
  fun state(): T

  fun updateState(newState: T): T

  fun execute(task: () -> Unit)
}
```

安卓上其实没有上面提到的这些事，直接利用 Executor 实现即可：

```kotlin
actual class WorkerTaskQueue<T> actual constructor(private var state: T) {
  private val executor = Executors.newSingleThreadScheduledExecutor()

  actual fun state(): T = state

  actual fun updateState(newState: T): T {
    this.state = newState
    return newState
  }

  actual fun execute(task: () -> Unit) {
    executor.execute(task)
  }
}
```

iOS 上就需要用 `AtomicReference` 和 freeze 了：

```kotlin
actual class WorkerTaskQueue<T> actual constructor(state: T)  {
  private val worker = Worker.start()
  private val state = AtomicReference(state.freeze())

  actual fun state(): T = state.value

  actual fun updateState(newState: T): T {
    newState.freeze()
    while (!state.compareAndSet(state.value, newState)) {
    }
    return state.value
  }

  actual fun execute(task: () -> Unit) {
    worker.executeAfter(0, task.freeze())
  }
}
```

State 的示例如下：

```kotlin
data class ConfState(
  var rid: String
) {
  fun onJoin(rid: String): ConfState {
    val state = copy()
    state.rid = rid
    return state
  }

  private fun copy(): ConfState {
    return ConfState(rid)
  }
}
```

对 `WorkerTaskQueue` 的使用示例如下：

```kotlin
init {
  taskQueue.updateConfState(ConfState(""))
}

fun join(rid: String) {
  taskQueue.execute {
    val newState = taskQueue.confState()
        .onJoin(rid)
    taskQueue.updateConfState(newState)
    newState.roomClient.connect(config.rsUrl, rid)
  }
}
```

采用上述方案后，处处都要小心，一忘记 freeze 就是 crash...

此外，其他库回调 Kotlin 时，需要在主线程，否则会在 FreezeSubgraph 时 crash: `EXC_BAD_ACCESS (code=1, address=0x10)`（address 都是低地址）。

## JSON 解析

JSON 解析倒是非常成熟了，我两个月前在 [KmppBootstrap](https://github.com/Piasy/KmppBootstrap) 里也介绍过，这里再举个简单的例子。

data 类定义如下：

```kotlin
@Serializable
data class AvFormat(
  val codec: String,
  val sampleRate: Int? = null,
  val channelNum: Int? = null,
  val profile: String? = null
)
```

使用代码如下：

```kotlin
private val json = Json(JsonConfiguration(encodeDefaults = false, strictMode = false))
try {
  val format = json.parse(AvFormat.serializer(), str)
} catch (e: SerializationException) {
  logError("parse AvFormat fail: ${e.message}")
}
```

创建 `Json` 时，`encodeDefaults = false` 用于在序列号时跳过 null（其实是跳过 default 值，但 default 值是 null，故跳过 null），`strictMode = false` 用于在反序列化时忽略 data 类里没有定义的 key。

## 展望未来

好了，今天先分享这么多，目前重写的版本只包含了安卓和 iOS 对接 OWT 的逻辑，下一步则是在 Windows 上实现平台相关代码，完成 Windows 平台对接 OWT 的功能。

再下一步就可以按需对接更多服务端了，到时只需编写公共的信令逻辑即可，将会变得非常轻松。

再之后就要考虑工程上的优化了，比如状态更新其实类似于 JS 里的 redux，可以考虑完全按照 redux 的套路实现；再比如编写单元测试、集成测试。

最后，本项目之后考虑开源，敬请期待 :)

_细节仍有疑问？还不过瘾？_
