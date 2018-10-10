---
layout: post
title: Android Runtime Permission测试
tags:
    - 安卓开发
    - PoC
---

Android 6.0引入了Runtime Permission模型，一方面用户不必在安装APP时便授予所有权限，另一方面，对于第三方ROM，APP自身也能方便地判断是否有某项权限了。在本文中，我将以读取通讯录为例对运行时权限进行一次全面的测试，完整代码可以[在Github下载](https://github.com/Piasy/AndroidPlayground/blob/master/try/AppMarshmallow/src/main/java/com/github/piasy/androidplayground/appmarshmallow/MainActivity.java)。

## 快速使用运行时权限
+  在AndroidManifest.xml中声明权限，就算是运行时权限，这一步也是不能忘记的：

``` xml
<uses-permission android:name="android.permission.READ_CONTACTS"/>
```

+  检查系统版本：

``` java
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
    // 继续
}
```

+  检查是否被授权：

``` java
switch (checkSelfPermission(Manifest.permission.READ_CONTACTS)) {
    case PackageManager.PERMISSION_GRANTED:
        // 已有授权
        break;
    case PackageManager.PERMISSION_DENIED:
        // 没有权限：尚未请求过权限，或者请求授权被拒绝，或者曾经授权过，
        // 但被用户在设置中禁用权限
        break;
    default:
        // 其实只会返回上述两种情况
        break;
}
```

+  如果当前没有权限，则请求权限：

``` java
requestPermissions(new String[] { Manifest.permission.READ_CONTACTS },
                        REQUEST_PERMISSION);
```

+  处理授权请求回调：

``` java
@Override
public void onRequestPermissionsResult(int requestCode, 
        @NonNull String[] permissions, @NonNull int[] grantResults) {
    if (requestCode == REQUEST_PERMISSION) {
        if (permissions.length == 1 &&
                permissions[0].equals(Manifest.permission.READ_CONTACTS) &&
                grantResults.length == 1) {
            if (grantResults[0] == PackageManager.PERMISSION_GRANTED) {
                // 授权请求被通过，读取通讯录
            } else {
                // 授权请求被拒绝
            }
        } else {
            // 其他情况
        }
    }
    super.onRequestPermissionsResult(requestCode, permissions, grantResults);
}
```

## 进一步深入
系统还提供了一个API：

``` java
public boolean shouldShowRequestPermissionRationale(
        @NonNull String permission)
```

文档中的描述比较晦涩，“显示Rationale是为了解释不那么明显的权限请求，该方法就是检查是否需要显示Rationale，例如相机应用请求定位权限”，那么这个函数到底是干什么用的呢？它的行为有何规律？

此外，请求权限的时候，系统会有一个弹出对话框提示用户是否授权，那么是否每次请求都会弹出这个对话框呢？对话框里面会不会有“不再询问”选择框呢？选择“不再询问”之后是否真的就不会再弹对话框了呢？如果应用卸载了，权限记录是否会保留？

带着这些问题，我针对`checkSelfPermission`和`shouldShowRequestPermissionRationale`进行了全面的测试，用X.Y来标记一次实验，其中X相同的实验是连续进行的，每次实验都把两个函数的返回值输出到log，然后请求权限，然后读取通讯录，实验结果记录如下，下面的列表每一项分别记录`checkSelfPermission`，`shouldShowRequestPermissionRationale`的返回值，以及请求权限时是否弹出对话框，是否有“不再询问”，最后是执行的操作：

+  安装APP，实验1.1，PERMISSION_DENIED，false，弹出/没有，允许
+  实验1.2，PERMISSION_GRANTED，false，弹出/没有，允许
+  实验1.3，PERMISSION_GRANTED，false，弹出/没有，允许
+  实验1.4，PERMISSION_GRANTED，false，弹出/没有，拒绝
+  实验1.5，PERMISSION_DENIED，true，弹出/有，拒绝
+  实验1.6，PERMISSION_DENIED，true，弹出/有，允许
+  实验1.7，PERMISSION_GRANTED，false，弹出/没有，拒绝
+  实验1.8，PERMISSION_DENIED，true，弹出/有，选择不再询问，结果允许按钮变得不可点，那么点拒绝吧
+  实验1.9，PERMISSION_DENIED，false，不弹/-，拒绝
+  卸载重装APP，实验2.1，PERMISSION_DENIED，false，弹出/没有，允许
+  实验2.2，PERMISSION_GRANTED，false，弹出/没有，拒绝
+  实验2.3，PERMISSION_DENIED，true，弹出/有，选择不再询问后拒绝
+  在设置里面允许APP权限，实验2.4，PERMISSION_GRANTED，false，弹出/没有，允许
+  在设置里面拒绝APP权限，实验2.5，PERMISSION_DENIED，false，弹出/没有，-

## 小结
+  从实验1.1、2.1可以得知，初始状态下，APP是没有权限的；
+  无论APP是否有权限，只要用户没有在授权对话框中选择不再询问，那么调用`requestPermissions`都会弹出授权请求对话框；所以如果`checkSelfPermission`函数返回了`PERMISSION_GRANTED`，我们就不应该再调用`requestPermissions`，但是检查权限是必须的，因为用户可以随时拒绝已允许的权限（revoke）；
+  从上述所有实验可以得知，`shouldShowRequestPermissionRationale`的行为规律如下：
  +  用户没有通过弹出的授权请求对话框来拒绝APP权限时，都返回false，包括从未请求过权限、用户在弹出对话框中允许了权限；
  +  用户通过弹出对话框拒绝APP权限，但未选择不再询问时，将返回true；
  +  用户通过弹出对话框拒绝APP权限，同时选择了不再询问，将返回false，同时以后的`requestPermissions`调用都不会触发弹出对话框，而是立即在`onRequestPermissionsResult`回调中返回`PERMISSION_DENIED`；
  +  用户如果在设置里面允许或者拒绝APP权限，都返回false，其性质类似于用户通过弹出对话框进行允许/拒绝APP权限；
+  从实验1.4~1.7可以看出，一旦用户在弹出对话框中选择了拒绝，那么后续的弹出对话框都会包含“不再询问”选项，而一旦用户在后续的弹出对话框中选择了允许，那么“不再询问”选项都会被隐藏；同时，如果选择了“不再询问”，将只能选择拒绝，允许按钮将不可点；如果选择了“不再询问”+拒绝，那么以后的`requestPermissions`调用都不会触发弹出对话框了，且立即在`onRequestPermissionsResult`回调中返回`PERMISSION_DENIED`；
+  从实验2.1可以看出，卸载APP将会导致一切都还原，不仅仅是卸载会导致权限行为复原，直接在设置里面允许权限后再拒绝权限，也会导致权限行为复原；

好了，上面的实验和小结对上文提出的问题进行了一一解答，相信可以解开一部分开发者的疑惑，当然这个小结肯定是不全面的，如果有什么补充的建议，可以在下面评论，或者[发邮件给我](mailto:xz4215@gmail.com)。

另外还有一位开发者的一篇译文写的还不错，可以参考一下，[链接](http://jijiaxin89.com/2015/08/30/Android-s-Runtime-Permission/)。
