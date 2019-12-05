# Kmpp

Objective-C interop

cinterop linkerOpts https://stackoverflow.com/a/58933001/3077508

安卓测例：

```gradle
        sourceSets {
            // with this assignment, test code under commonTest will be invoked,
            // without it, test code even under androidTest won't be invoked.
            androidTest {
                java.srcDirs = ['src/androidTest/kotlin',]
            }
        }
```

Robolectric 测例过滤

publish

```gradle
kotlin {
        android {
            publishLibraryVariants("release", "debug")
        }
}
```

必须要有 jvm，否则 common 依赖都会报错找不到。

https://github.com/ktorio/ktor/blob/master/gradle/publish.gradle

https://github.com/orangy/multiplatform-lib

但安卓仍找不到……

---

Ktor 用到了 coroutine，按照 coroutine 写了一遍，安卓没问题。但 iOS coroutine 只能在主线程用，不可行。

抛弃 Ktor 和 coroutine，改用 worker，但 worker 要提交 operation 就需要 freeze，但 freeze 了就不能修改了，根本不能用……

coroutine for multiple thread? 实际上只是想要一个非主线程的 coroutine…… 它确实可以 newSingleThreadContext，但在主线程调用的函数中，向其添加 coroutine，会触发异常：This dispatcher can be used only from a single thread test, but now in MainThread。

用一个 native 实现的 message queue，主线程向其发送消息数据，处理消息的对象都由 native 层创建，但怎么回调呢？callback 要传入 native 层就得 freeze，但 callback 的实现对象，肯定不能 freeze...

使用 `concurrency safe API` 就能不怕 freeze（例如 `AtomicInt`），给 TaskQueue 增加一个成员，因为要在 native 里用 `AtomicReference`，所以不能在 common 里，在提交到 TaskQueue 的 lambda 里修改（替换）之。

其他库回调 Kotlin，需要在主线程，否则会在 FreezeSubgraph 时 crash: `EXC_BAD_ACCESS (code=1, address=0x10)`（address 都是低地址）。

处处要小心，忘记 freeze 就是 crash...
