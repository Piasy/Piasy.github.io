---
layout: post
title: 安卓 NDK 入门指南
tags:
    - NDK
---

_本文的前身是一篇笔记，比较零碎，发布出来是为了让后续的文章可以有一个基本的参考，本文会持续更新。_

NDK 的高性能最常见的场景：多媒体，游戏。此外，利用 NDK 还能练习 C/C++，一举两得。

## 基本概念

+ shared library, `.so`
+ static library, `.a`
+ JNI: Java Native Interface
+ Application Binary Interface, ABI：CPU 指令集，字节序（大小端）……
  - `armeabi`
  - `armeabi-v7a`
  - `arm64-v8a`
  - `x86`
+ JNI function v.s. native method：前者是 JNI 系统（Java）提供的函数，后者则是 Java 类里面定义的 native 函数；

## JNI

### Design overview

+ JNI interface pointer：**native 代码通过它使用 JVM 的功能（调用 JNI functions）**；
  - 它就是 native 方法的 `JNIEnv* env` 参数，它是一个指针的指针，所以使用都是 `(*env)->XXX`；C++ 语法层面可以简化为 `env->XXX`；
  - 可以把它理解为一个线程局部的指针，只在传入时的线程内有效，不要把它存起来给其他线程用；JVM 保证同一个线程内多次调用（不同的）native 方法，env 指针都是同一个，不同线程调用，env 指针将不同；

![](https://imgs.piasy.com/2017-03-16-designa.gif)

+ 函数命名规则，参数对应规则，参数使用之前的转换，现在 Android Studio 都有了很好的支持；
+ 参数列表：第一个参数是 JNI interface pointer `JNIEnv* env`，如果方法是 static，第二个参数就是该类的 class 对象引用，否则是该对象的引用；其他参数的对应关系见 [JNI Types and Data Structures](http://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/types.html)；
+ 访问 Java 对象：primitive 类型传值，对象传引用，这和 Java 的机制一致；native 代码需要显式告知 JVM，何时引用了 Java 对象，何时释放了引用；
  - Global References：有效期不限于本次函数调用，需显式释放；
  - Local References：有效期仅限于本次函数调用，无需显式释放；为了允许提前 GC，或者防止 local ref 导致 OOM 时，可以显式释放；local ref 只允许在被创建的线程使用；
+ 访问 primitive 数组：_JNI 引入了“pinning”机制，可以让 JVM “固化”数组的地址，允许 native 代码获得直接访问的指针_；
+ 访问成员变量和成员函数：根据名字和签名，取得函数/变量的 id，再利用 id 进行调用；持有函数/变量的 id 不能阻止类被卸载，持有类的 class 对象引用可以阻止；
+ JNI 允许在 native 代码中抛出异常，也能捕获异常；

### [JNI Functions](http://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html)

+ 访问 Java 的方法，需要指定签名（signature，包含参数列表、返回值类型），可以用 `javap –s -classpath <path to class file> <import path>` 获取；
+ 类操作、异常处理、reference、对象操作、成员变量、成员函数、字符串、数组、monitor、NIO、反射……具体可以参考 [spec](http://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html)；

### [The Invocation API](http://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/invocation.html)

+ native 线程可以创建 JavaVM 和 JNIEnv 对象，用于运行 Java 的代码；
+ JNIEnv 只在创建的线程内有效，如果要如果要保存起来在其他线程使用，都需要先 AttachCurrentThread，下面的代码参考自 [StackOverflow](http://stackoverflow.com/q/12900695/3077508)：

``` java
// 在 JNI_OnLoad 中直接保存 g_vm，或者在初始化函数中利用 JNIEnv 获取并保存 g_vm
static JavaVM *g_vm;

jint JNI_OnLoad(JavaVM *vm, void *reserved) {
    g_vm = vm;
}

...init(JNIEnv *env...) {
    env->GetJavaVM(&g_vm);
}

// 利用 g_vm 获取 JNIEnv，并判断是否需要 attach
void nativeFunc(char *data, int len) {
    JNIEnv *env;
    int getEnvStat = g_vm->GetEnv((void **) &env, JNI_VERSION_1_6);
    if (getEnvStat == JNI_EDETACHED) {
        int attached = g_vm->AttachCurrentThread(&env, NULL);
        LOGI("AttachCurrentThread: %s", (attached ? "false" : "true"));
    } else if (getEnvStat == JNI_OK) {
        LOGI("thread already attached");
    } else if (getEnvStat == JNI_EVERSION) {
        LOGI("Unsupported JVM version");
        return;
    }
    callJNIFunc(env, data, len);
    if (getEnvStat == JNI_EDETACHED) {
        g_vm->DetachCurrentThread();
    }
}
```

+ native 库被加载的时候（`System.loadLibrary`），会调用 `JNI_OnLoad` 函数；库被 GC 的时候，会调用 `JNI_OnUnload`；
+ 调用 JVM（JNI）方法都需要 JNIEnv 指针，但 JNIEnv 不能跨线程共享，我们只能共享 JavaVM 指针，并用它来获取各自线程的 JNIEnv；

## [JNI tips](https://developer.android.com/training/articles/perf-jni.html)

+ 不要跨线程共享 JNIEnv，要共享就共享 JavaVM；
+ 如果头文件会被 C/C++ 同时 include，那最好里面不要引用 JNIEnv；
+ `pthread_create` 创建的线程，需要调用 `AttachCurrentThread` 才能拥有 JNIEnv，才能调用 JNI 函数；attach 会为该 native 线程创建一个 `java.lang.Thread` 对象；attach 之后的线程，在退出之前需要 detach，当然，也可以更早 detach；

### jclass, jmethodID, and jfieldID

+ native 代码中访问 Java 成员变量或者调用 Java 函数，需要先找到 `jclass`，`jmethodID` 或 `jfieldID`，找到这些变量会比较耗时，但找到之后访问/调用是很快的；
+ `jclass` 如果要存起来，必须用 `NewGlobalRef` 包装一层，一是防止被 GC，二是为了在作用域之外使用；如果要缓存 `jmethodID` 或 `jfieldID`，可以在 Java 类的 static 代码块中调用 native 函数执行缓存操作；

### Local and Global References

+ native 方法的参数，以及绝大多数 JNI 方法的返回值，都是 local reference，即便被引用的对象还存在，local reference 也将在作用域外变得非法（不能使用）；可以通过 `NewGlobalRef` 或者 `NewWeakGlobalRef` 创建 global reference；常用的保存 `jclass` 方法就是如下：

``` java
jclass localClass = env->FindClass("MyClass");
jclass globalClass = reinterpret_cast<jclass>(env->NewGlobalRef(localClass));
```

+ 在 native 代码中，同一对象的引用值可能不同，因此不要用 `==` 判等，而要用 `IsSameObject` 函数；
+ 对象的引用既不是固定不变的，也不是唯一的，因此不要用 `jobject` 作为 key；
+ 通常 local reference 可以被 JVM 自动释放，但由于存在数量上限，而且手动 attach 的线程不会自动释放 local reference，因此最好还是手动释放 local reference，尤其是在循环中；

### UTF-8, UTF-16 Strings, Primitive Arrays

+ Get 之后都需要 Release，否则内存泄漏；Get 可能失败，不要 Release NULL；
+ `Get<PrimitiveType>ArrayElements` 可能会直接返回 Java heap 上的地址，也可能会做拷贝；无论是否拷贝，都需要 Release；
+ Release 时有 mode 参数，0，`JNI_COMMIT`，`JNI_ABORT`；
+ `Get<PrimitiveType>ArrayRegion` 和 `Set<PrimitiveType>ArrayRegion` 用于单纯的数组拷贝；

### Exceptions，Extended Checking

+ 异常发生后，就不要调用 JNI 函数了，除了 Release/Delete 等函数；
+ `adb shell setprop debug.checkjni 1` 或在 manifest 中设置 `android:debuggable`；

### Java 和 native 代码共享数组

+ `GetByteArrayElements` 可能直接返回堆地址，也可能会进行拷贝，后者就存在性能开销；
+ `java.nio.ByteBuffer.allocateDirect` 分配的数组，一定不需要拷贝（通过 `GetDirectBufferAddress`）；但在 Java 代码中访问 direct ByteBuffer 可能会很慢；
+ 所以需要考虑：数组主要在哪一层代码访问？（native 层就用 direct ByteBuffer，Java 层就用 byte[]）如果数据最终都要交给系统 API，数据必须是什么形式？（最好能用 byte[]）

## 开发技巧

+ native obj 做 Java wrapper
  - 把 native obj 指针 `jlong handle = reinterpret_cast<intptr_t>(ptr)` 得到一个 jlong 值；
  - Java obj 保存一个这个 long 值，后续 native 接口调用都带着它；
  - native 函数中 `reinterpret_cast<NativeType*>(j_ptr)` 得到 native obj 的指针，就可以调用它的方法了；

## 参考链接

+ [JNI spec](http://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/jniTOC.html)
+ [JNI tips](https://developer.android.com/training/articles/perf-jni.html)
+ [添加、引入、引用 C/C++ 代码、模块](https://developer.android.com/studio/projects/add-native-code.html)
+ [NDK 提供的可直接使用的库](https://developer.android.com/ndk/guides/stable_apis.html)
+ [googlesamples/android-ndk](https://github.com/googlesamples/android-ndk)
