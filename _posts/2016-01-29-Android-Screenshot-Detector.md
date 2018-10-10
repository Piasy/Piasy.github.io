---
layout: post
title: RxScreenshotDetector：Android 截屏检测
tags:
    - 安卓开发
    - MISC
    - Reactive eXtention
---

应PM需求，YOLO可能会对直播过程中的截屏进行检测并通知其他人，类似于Snapchat，此时iOS同事再次表达了先天优势，iOS系统提供了API呀！Google无果之后决定再次造轮子，为了持续表达对Rx的敬意，命名为`RxScreenshotDetector`，[github 源码地址](https://github.com/Piasy/RxScreenshotDetector)。

## 效果有图有真相

<img src="/img/9/screenshot-detector-demo.gif" alt="screenshot-detector-demo" style="height:400px">

## 原理

安卓系统并没有提供任何截屏检测相关的API，网上针对Snapchat的这项功能进行了分析，大致猜测可能有以下几种途径：

+  使用`FileObserver`，监听`Screenshots`目录下的文件变化；
+  使用`ContentObserver`，监听`MediaStore.Images.Media.EXTERNAL_CONTENT_URI`资源的变化；
+  重载（hook）截屏组合键（不靠谱），有的机型使用的是特殊手势进行截屏；

主要参考了[StackOverflow上面的这个回答](http://stackoverflow.com/a/29624090/3077508)。

核心代码如下：

``` java
private static final String TAG = "RxScreenshotDetector";
private static final String EXTERNAL_CONTENT_URI_MATCHER =
        MediaStore.Images.Media.EXTERNAL_CONTENT_URI.toString();
private static final String[] PROJECTION = new String[] {
        MediaStore.Images.Media.DISPLAY_NAME, MediaStore.Images.Media.DATA,
        MediaStore.Images.Media.DATE_ADDED
};
private static final String SORT_ORDER = MediaStore.Images.Media.DATE_ADDED + " DESC";
private static final long DEFAULT_DETECT_WINDOW_SECONDS = 10;

final ContentResolver contentResolver = context.getContentResolver();
final ContentObserver contentObserver = new ContentObserver(null) {
    @Override
    public void onChange(boolean selfChange, Uri uri) {
        Log.d(TAG, "onChange: " + selfChange + ", " + uri.toString());
        if (uri.toString().matches(EXTERNAL_CONTENT_URI_MATCHER)) {
            Cursor cursor = null;
            try {
                cursor = contentResolver.query(uri, PROJECTION, null, null,
                        SORT_ORDER);
                if (cursor != null && cursor.moveToFirst()) {
                    String path = cursor.getString(
                            cursor.getColumnIndex(MediaStore.Images.Media.DATA));
                    long dateAdded = cursor.getLong(cursor.getColumnIndex(
                            MediaStore.Images.Media.DATE_ADDED));
                    long currentTime = System.currentTimeMillis() / 1000;
                    Log.d(TAG, "path: " + path + ", dateAdded: " + dateAdded +
                            ", currentTime: " + currentTime);
                    if (path.toLowerCase().contains("screenshot") &&
                            Math.abs(currentTime - dateAdded) <=
                                    DEFAULT_DETECT_WINDOW_SECONDS) {
                        // screenshot added!
                    }
                }
            } catch (Exception e) {
                Log.d(TAG, "open cursor fail");
            } finally {
                if (cursor != null) {
                    cursor.close();
                }
            }
        }
        super.onChange(selfChange, uri);
    }
};
contentResolver.registerContentObserver(
        MediaStore.Images.Media.EXTERNAL_CONTENT_URI, true, contentObserver);
```

主要有以下几点需要注意：

+  权限，读取资源的时候需要`READ_EXTERNAL_STORAGE`权限，这里我使用了[RxPermissions](https://github.com/tbruyelle/RxPermissions)来以reactive的方式请求权限；
+  从`ContentResolver`查询资源的时候，需要按照资源创建时间降序排列，针对最新的一个资源判断是否为截屏的图片，为`contentResolver.query`的最后一个参数传递`MediaStore.Images.Media.DATE_ADDED + " DESC"`即可，而判断图片是否为截图则比较简单，路径包含`screenshot`关键字，且添加时间在10s之内；

## 使用示例

`RxScreenshotDetector`完整使用代码如下：

``` java
RxScreenshotDetector.start(getApplicationContext())
        .subscribeOn(Schedulers.io())
        .observeOn(AndroidSchedulers.mainThread())
        .compose(this.<String>bindUntilEvent(ActivityEvent.PAUSE))
        .subscribe(new Subscriber<String>() {
            @Override
            public void onCompleted() {

            }

            @Override
            public void onError(Throwable e) {
                e.printStackTrace();
            }

            @Override
            public void onNext(String path) {
                mTextView.setText(mTextView.getText() + "\nScreenshot: " + path);
            }
        });
```

这里使用了[RxLifecycle](https://github.com/trello/RxLifecycle)，在Activity onPause之后unsubscribe，以保证不会发生内存泄漏。此外subscribe传入的是完整的Subscriber，是为了防止授权失败时没有onError处理器，导致crash。

最后，安利一发[YOLO直播APP](https://www.yoloyolo.tv/)。
