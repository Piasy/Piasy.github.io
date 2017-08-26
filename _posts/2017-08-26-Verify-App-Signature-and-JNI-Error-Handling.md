---
layout: post
title: Native 层校验 APP 签名，以及 JNI 异常处理
tags:
    - NDK
    - 信息安全
---

有时候我们需要在 APP 运行时验证当前 APP 是否被篡改，而 SDK 提供方通常也需要验证 APP 是否被授权，今天我们就来探讨一下如何在 native 层实现这一功能，以及在这个过程中的一些要点和技巧。

## 认证方案

我们先探讨一下有哪些认证方案。

### 授权码

最容易想到的方案就是用户名密码认证了，稍加引申，可以想到用 token 进行认证，本质都是一样的，通过授权码进行认证，但这存在授权码泄漏的风险。

最危险的做法就是把授权码以完整的字符串常量硬编码在客户端代码中，攻击者将能轻易取得授权码。稍稍改进一点的做法是不保存完整字符串，而是通过一段绕来绕去的代码，运行时拼出授权码，这样静态分析就变得很困难了，不过一旦攻击者在运行时读取内存，还是可以轻易取得。

再进一步，我们可以让授权码从服务端下发（_这将引入一个“鸡生蛋”的问题，访问服务器也得授权，我们先不考虑这个问题_），不过这里也存在抓包的风险，即便是 HTTPS，也是可以抓包的，Google "charles android https" 就能找到教程。

这里的 HTTPS 抓包其实是中间人攻击，charles 先伪装为客户端向服务器取到数据，再伪装为服务器把数据交给客户端，这样它作为中间人就得到了通信的明文数据。那我们能否区分真正的服务器和 charles 扮演的服务器呢？可以的。校验 SSL 证书即可，**当然前提是我们服务器上的证书私钥没有泄露**。

那如果加上了 SSL 证书校验，是不是就无懈可击了呢？不是的。攻击者依然可以在客户端运行时读取内存。

那这是不是说明授权码的方式绝对不安全呢？这就要看我们保护的是什么东西了。例如我们用账号密码登录微信，如果网络没有被劫持，手机没有被攻破，攻击者能窃取我们的账号密码吗？不会呀！但如果攻击者用自己的账号密码登录微信后，想要窃取微信使用的某个 SDK 的授权码，那这就确实可以被窃取了。**注意，攻击者可以窃取的，是微信使用的某个 SDK 的授权码，既不是他自己的账号密码，也不是别人的账号密码**。

所以，授权码的方式对于 SDK 认证来说基本上是不安全的。当然也不绝对，如果每个用户都有不同的授权码，而不是某个客户的所有用户使用同一个授权码，并且对授权码的使用频率做出限制，那攻击者窃取到的授权码，价值也就比较有限了。

### 安全目标

经过了前面的实际探讨，现在我们来点理论知识。

通常定义的信息安全主要有三大目标：

+ 保密性（Confidentiality）：保护信息内容不会被泄露给未授权的实体，防止被动攻击；
+ 完整性（Integrity）：保证信息不被未授权地修改，或者如果被修改可以检测出来，防止主动攻击，比如篡改、插入、重放；
+ 可用性（Availability）：保证资源的授权用户能够访问到应得资源或服务，防止拒绝服务攻击；

以上三大目标简称 CIA，当然此 CIA 非彼 CIA（美国中央情报局，Central Intelligence Agency）。除了这三点，有时大家也会加上另外两点要求：

+ 可控性（Controllability）：限制实体的访问权限，通常是经过认证的合法的实体才可以访问，标识与认证是访问控制的前提；
+ 不可抵赖性（Non-repudiation）：防止发送方或者接收方否认传输或者接收过某条消息；

现在看来，实际上我们的目标就是实现可控性，前面讨论的办法就是利用授权码，但授权码在保密性方面存在不可避免的漏洞。实际上在我们讨论的场景下，任何信息都无法实现保密性，在我们最终使用涉密信息的时候，读取内存这一终极大招将无往不利。

但实现认证必须保证保密性吗？其实不需要，我们只需要认证用户身份，认证信息可以是公开的信息，恰好有一个工具就是专门做这件事情的：数字签名。

### 数字签名

我们知道，安卓 APP 打包过程中有一个签名的步骤，我们用私钥对 APK 文件进行签名，公钥会被一起打包到 APK 中，这样安卓系统就可以对 APK 的完整性进行校验了。除了安卓系统，我们作为 SDK 的提供方，也可以对 APK 的完整性进行校验。

这正是很多第三方 SDK 所使用的认证方式，我们在其管理后台提交签名公钥的 SHA1，他们为我们生成 AppId 和 AppSecret，在客户端初始化 SDK 时，传入 AppId 和 AppSecret，APP 的签名公钥则由 SDK 自行获取。

实际上 AppId 和 AppSecret 没什么必要，只需要检验公钥即可，检验一个值和检验三个值，没有什么区别。但要注意，一定不能让 APP 传入，否则攻击者就可以随意传入伪造的公钥了。

## 获取应用签名

### Java 代码实现

在使用 native 代码实现之前，我们先用 Java 代码实现以下，便于我们理解原理：

~~~ java
private byte[] getCertificateSHA1Fingerprint(Context context) {
    PackageManager pm = context.getPackageManager();
    String packageName = context.getPackageName();

    try {
        PackageInfo packageInfo = pm.getPackageInfo(packageName, 
            PackageManager.GET_SIGNATURES);
        Signature[] signatures = packageInfo.signatures;
        byte[] cert = signatures[0].toByteArray();
        X509Certificate x509 = X509Certificate.getInstance(cert);
        MessageDigest md = MessageDigest.getInstance("SHA1");
        return md.digest(x509.getEncoded());
    } catch (PackageManager.NameNotFoundException | CertificateException |
            NoSuchAlgorithmException e) {
        e.printStackTrace();
    }
    return null;
}
~~~

这段代码在 Android Studio 中会有一个 Lint 警告，“android-fake-id-vulnerability”，经过搜索我们可以发现在安卓 2.1~4.4 的系统中存在安全漏洞，使得应用签名可以被伪造。不过这已经是三年前的事情了，而且 Google 已经发出了补丁，所以让我们暂且认为这个漏洞影响不大。

上面的代码做了三件事：

+ 获取 PackageInfo 中的 Signature；
+ 获取 Signature 的公钥；
+ 计算公钥的 SHA1；

计算出 SHA1 之后，我们就可以进行对比了。下面我们看看对应的 native 代码。

### Native 代码实现

~~~ cpp
#include <jni.h>
#include <stddef.h>

extern "C" {

JNIEXPORT jbyteArray JNICALL
Java_com_github_piasy_MainActivity_nativeGetSig(
        JNIEnv *env, jclass type, jobject context) {
    // context.getPackageManager()
    jclass context_clazz = env->GetObjectClass(context);
    jmethodID getPackageManager = env->GetMethodID(context_clazz, 
        "getPackageManager", "()Landroid/content/pm/PackageManager;");
    jobject packageManager = env->CallObjectMethod(context, 
        getPackageManager);

    // context.getPackageName()
    jmethodID getPackageName = env->GetMethodID(context_clazz, 
        "getPackageName", "()Ljava/lang/String;");
    jstring packageName = (jstring) env->CallObjectMethod(context, 
        getPackageName);

    // packageManager->getPackageInfo(packageName, GET_SIGNATURES);
    jclass package_manager_clazz = env->GetObjectClass(packageManager);
    jmethodID getPackageInfo = env->GetMethodID(package_manager_clazz, 
        "getPackageInfo", "(Ljava/lang/String;I)Landroid/content/pm/PackageInfo;");
    jint flags = 0x00000040;
    jobject packageInfo = env->CallObjectMethod(packageManager, 
        getPackageInfo, packageName, flags);

    jthrowable exception = env->ExceptionOccurred();
    env->ExceptionClear();
    if (exception) {
        return NULL;
    }

    // packageInfo.signatures[0]
    jclass package_info_clazz = env->GetObjectClass(packageInfo);
    jfieldID fid = env->GetFieldID(package_info_clazz, "signatures",
        "[Landroid/content/pm/Signature;");
    jobjectArray signatures = (jobjectArray) env->GetObjectField(
        packageInfo, fid);
    jobject signature = env->GetObjectArrayElement(signatures, 0);

    // signature.toByteArray()
    jclass signature_clazz = env->GetObjectClass(signature);
    jmethodID signature_toByteArray = env->GetMethodID(signature_clazz, 
        "toByteArray", "()[B");
    jbyteArray sig_bytes = (jbyteArray) env->CallObjectMethod(
        signature, signature_toByteArray);

    // X509Certificate appCertificate = X509Certificate.getInstance(sig_bytes);
    jclass x509_clazz = env->FindClass("javax/security/cert/X509Certificate");
    jmethodID x509_getInstance = env->GetStaticMethodID(x509_clazz, 
        "getInstance", "([B)Ljavax/security/cert/X509Certificate;");
    jobject x509 = (jstring) env->CallStaticObjectMethod(x509_clazz, 
        x509_getInstance, sig_bytes);

    exception = env->ExceptionOccurred();
    env->ExceptionClear();
    if (exception) {
        return NULL;
    }

    // x509.getEncoded()
    jmethodID getEncoded = env->GetMethodID(x509_clazz, 
        "getEncoded", "()[B");
    jbyteArray public_key = (jbyteArray) env->CallObjectMethod(x509, getEncoded);

    exception = env->ExceptionOccurred();
    env->ExceptionClear();
    if (exception) {
        return NULL;
    }

    // MessageDigest.getInstance("SHA1")
    jclass message_digest_clazz = env->FindClass("java/security/MessageDigest");
    jmethodID message_digest_getInstance = env->GetStaticMethodID(
        message_digest_clazz, "getInstance", 
        "(Ljava/lang/String;)Ljava/security/MessageDigest;");
    jstring sha1_name = env->NewStringUTF("SHA1");
    jobject sha1 = env->CallStaticObjectMethod(message_digest_clazz, 
        message_digest_getInstance, sha1_name);

    exception = env->ExceptionOccurred();
    env->ExceptionClear();
    if (exception) {
        return NULL;
    }

    // sha1.digest(public_key)
    jmethodID digest = env->GetMethodID(message_digest_clazz, 
        "digest", "([B)[B");
    jbyteArray sha1_bytes = (jbyteArray) env->CallObjectMethod(
        sha1, digest, public_key);

    return sha1_bytes;
}

}
~~~

其实 native 代码并没有什么高深的技巧，就是啰嗦！在[安卓 NDK 入门指南](/2017/08/26/NDK-Start-Guide/#jclass-jmethodid-and-jfieldid)中我们提到了 native 代码调用 Java 函数的套路：先找到 `jclass` 和 `jmethodID`，再 `CallXXXMethod`。

在 Java 代码中我们捕获了好几种异常，Native 代码怎么处理异常？没有异常处理代码自然是可以编译通过，但运行期一旦发生异常，等着我们的将是崩溃。

## JNI 异常处理

在 Java 代码中，如果发生了异常，程序的控制流会立即交给最近的一个 catch block，并且暂停当前代码的执行。也就是说，连续的两条语句，第一条执行时抛出了异常，第二行将不会执行，而是开始执行 catch block 中的代码。

如果没有 catch block，那就会把控制流交给 `Thread.getDefaultUncaughtExceptionHandler` 返回的 `UncaughtExceptionHandler`，然后终止线程。安卓主线程的 `UncaughtExceptionHandler` 会弹出一个提示程序异常关闭的对话框，用户可以选择重新打开 APP。

但如果在 native 代码中，触发了 JVM 抛出异常，native 代码的控制流不会发生变化，后面的代码将会继续执行，直到程序执行从 native 层返回到 Java 层。但异常发生后，如果 native 代码没有处理异常，继续调用 JNI 函数，那就会导致程序崩溃（_这应该也是抛出的一个异常，但是会直接终止 native 代码的执行_），通常是 `SIGABRT`，且崩溃日志中会带有 `Pending exception` 关键字。所以，在 native 代码中，调用 JNI 函数后，一定要进行异常检查和处理。

有些情况下我们可以通过函数返回值判断是否发生了异常，例如 `env->FindClass` 如果返回 `NULL`，则说明发生了异常。但通常更多情况下我们无法利用返回值，而是需要调用 `env->ExceptionOccurred()` 或者 `env->ExceptionCheck()` 来进行检查。前者会在异常发生时返回异常对象，否则返回 `NULL`，后者则会返回 `JNI_TRUE` 或者 `JNI_FALSE`。

检查出发生了异常后，通常有三种做法：

+ 执行相关清理工作，return native 函数，把程序控制流交给 Java 层，进而触发 Java 层的异常处理流程；
+ 直接在 native 层对异常进行处理，异常不会传递到 Java 层；
+ 在 native 层做一些处理，然后向 Java 层抛出一个新的异常（也得 return）；

如果要在 native 层做异常处理，那我们就可以先调用 `env->ExceptionClear()` 清除 pending exception，清除后我们就可以安全地调用 JNI 函数了。如果要重新抛出异常，则可以调用 `env->Throw` 或者 `env->ThrowNew`，但要注意，throw 执行后，程序控制流依然在 native 层，必须 return 才能回到 Java 层。

此外，我们还可以调用 `env->ExceptionDescribe` 把调用栈等信息打印到 stderr 中，或者直接调用 `env->FatalError` 终止进程，这时我们就无需 return 了，因为这个调用就不会 return，最后，更多关于 JNI 异常处理函数的信息，最好的文档莫过于 [spec 的 JNI Functions 部分](http://docs.oracle.com/javase/7/docs/technotes/guides/jni/spec/functions.html)了。

## 校验实现技巧

前面我们获取了当前 APP 的签名公钥 SHA1，但这只是第一步，如何做校验呢？提交到服务端还是客户端本地检验？

如果提交到服务端，那我们势必要从服务端获取校验结果，然后使用校验结果。通讯过程我们可以通过 HTTPS + SSL 证书校验来保证安全性。使用校验结果时有一个小技巧：不要简单地用一个判断语句，这样攻击者可以比较容易根据代码执行流程定位判断位置，进而绕过。我们可以把结果参与到相关运算过程中，校验成功与失败的执行流程完全一致，那要进行定位就更困难了。当然，这只是个小花样，并不能彻底解决安全隐患。

客户端本地校验安全性就要差一些了，除了使用校验结果的过程存在安全隐患外，校验过程也存在安全隐患，攻击者可以修改我们的校验代码，把我们的对比对象修改为他们的签名值。

最后我们需要树立一个正确的观点：**安全是相对的**，RSA 算法也是可以破解的呢，只不过攻击者需要具备的能力、需要付出的代价很高而已。不要因为我们的校验方案存在安全隐患就不做校验，破罐破摔肯定是不对的，使用一个即便存在安全隐患但更安全的方案，就能挡住更多攻击者。当然，我们也应该继续思考，如何解决安全隐患，提升安全性。

## 参考文章

+ [The Dangers of the Android FakeID Vulnerability](http://blog.trendmicro.com/trendlabs-security-intelligence/the-dangers-of-the-android-fakeid-vulnerability/)
+ [Exception Handling in JNI](http://www.developer.com/java/data/exception-handling-in-jni.html)
