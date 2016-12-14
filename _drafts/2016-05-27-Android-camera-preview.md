---
layout: post
title: 安卓相机预览
tags:
    - 安卓开发
---

## `SurfaceTexture` 和 `Surface`

通常来说，拍照都需要先提供预览，预览的是帧数据，显示在 `SurfaceTexture` 上，或者 `Surface` 上。`SurfaceTexture` 把图像作为 OpenGL ES 纹理进行渲染，而数据来源则可以是相机或者解码的视频流。`Surface` 则负责管理用来显示的数据帧，`Surface` 渲染数据实际上也是通过其持有的 `SurfaceTexture` 对象实现的。

`SurfaceTexture` 结合 `Surface`，可以渲染 camera2 api、MediaCodec、MediaPlayer、renderscript.Allocation 的输出数据。`Surface` 结合 `SurfaceHolder` 则可以用于渲染 camera api 的输出数据。由此可见，`SurfaceTexture` 是预览以及视频播放的核心。

## `SurfaceView` 和 `TextureView`

`SurfaceTexture` 和 `Surface` 都不是 view，无法显示在界面中。

寻常 view 的绘制都是在主线程的，而 `SurfaceView` 则使用单独的线程进行绘制（渲染在 `Surface` 上，所以实际上还是通过 `SurfaceTexture` 渲染），但它的 measure 和 layout 都发生在主线程，它有一个“占位 window”，就像寻常 view 的内容区域一样，只不过它是透明的，单独线程绘制的内容将透过这个透明的”占位 window“让用户看见。

`TextureView` 和 `SurfaceView` 类似，也使用单独的线程绘制（通过 `SurfaceTexture` 渲染），不过它没有“占位 window”。而且 `TextureView` 只能用于硬件加速的 window 中。

`GLSurfaceView` 和 `VideoView` 都是 `SurfaceView` 的子类。前者封装了 Open GL 渲染的功能，后者则封装了对一个视频文件的播放功能。

使用 `GLSurfaceView` 时，我们可以指定自定义的 `Renderer`，在把图像数据渲染到 `Surface` 之前，进行一些处理，例如美颜滤镜、趣味滤镜等。

## `Camera` API

`Camera` 是 Camera service 的客户端，而后者则负责管理相机硬件。利用 `Camera` 可以拍照（capture image）、预览（preview）、获取帧数据（用于编码传输或者保存为视频）。

### `TextureView` 预览相机

简单完整的预览示例 [developer 网站](https://developer.android.com/reference/android/view/TextureView.html){:target="_blank"}上可以找到。

设置 SurfaceTexture 回调：

~~~ java
mTextureView.setSurfaceTextureListener(listener);
~~~

在 SurfaceTexture 回调中开始预览：

~~~ java
@Override
public void onSurfaceTextureAvailable(SurfaceTexture surface, int width, int height) {
    mCamera = Camera.open();

    try {
        mCamera.setPreviewTexture(surface);
        mCamera.startPreview();
    } catch (IOException ioe) {
        // Something bad happened
    }
}

@Override
public boolean onSurfaceTextureDestroyed(SurfaceTexture surface) {
    mCamera.stopPreview();
    mCamera.release();
    return true;
}
~~~

当然，在打开 `Camera` 对象之后，可以通过设置，调整预览以及 `TextureView` 的尺寸，以免预览变形。

设置帧数据回调：

~~~ java
mCamera.setPreviewCallback(callback)
~~~

在帧数据回调中获取帧数据：

~~~ java
@Override
public void onPreviewFrame(final byte[] data, final Camera camera) {
    // data 就是帧数据，可以进行处理（编码传输等）
    // 处理完毕之后，执行下面这段代码
    camera.addCallbackBuffer(data);
}
~~~

### `GLSurfaceView` 美颜预览

结合使用 [android-gpuimage](https://github.com/CyberAgent/android-gpuimage){:target="_blank"}。

在 `onResume` 中初始化 `GLSurfaceView`、初始化 `GPUImageRenderer`、开启相机、开始预览：

~~~ java
@Override
public void onResume() {
    super.onResume();

    mRenderer = new GPUImageRenderer(new GPUImageColorInvertFilter());
    if (Utils.isSupportOpenGLES2(getContext())) {
        mGLSurfaceView.setEGLContextClientVersion(2);
    }
    mGLSurfaceView.setEGLConfigChooser(8, 8, 8, 8, 16, 0);
    mGLSurfaceView.getHolder().setFormat(PixelFormat.RGBA_8888);
    mGLSurfaceView.setRenderer(mRenderer);
    mGLSurfaceView.setRenderMode(GLSurfaceView.RENDERMODE_WHEN_DIRTY);
    mGLSurfaceView.requestRender();
    
    mCamera = Camera.open();
    
    mGLSurfaceView.setRenderMode(GLSurfaceView.RENDERMODE_CONTINUOUSLY);
    mRenderer.setUpSurfaceTexture(new SurfaceTextureInitCallback() {
        @Override
        public void onSurfaceTextureInitiated(SurfaceTexture surfaceTexture) {
            camera.setPreviewCallback(callback);
            camera.setPreviewTexture(surfaceTexture);
            camera.startPreview();
        }
    });
}
~~~

注意，尽管我们是用自定义 render 绘制到 `GLSurfaceView` 上的，但我们仍需要调用 `camera.setPreviewTexture`，否则有的手机上可能无法显示预览（例如三星 A7）。

`mRenderer.setUpSurfaceTexture` 的工作原理是在绘制到 `GLSurfaceView` 时调用 `onSurfaceTextureInitiated`，此时表明 `GLSurfaceView` 已经准备好，所以可以开始相机预览了。

帧数据回调中处理数据：

~~~ java
@Override
public void onPreviewFrame(final byte[] data, final Camera camera) {
    Camera.Size size = camera.getParameters().getPreviewSize();
    mRenderer.onFrameData(data, new Size(size.width, size.height), new Runnable() {
        @Override
        public void run() {
            camera.addCallbackBuffer(data);
        }
    });
}
~~~

## `Camera2` API

【补充描述】

### `TextureView` 预览相机

普通预览完整例子可以参考 [Android Camera2Basic Sample](https://github.com/googlesamples/android-Camera2Basic){:target="_blank"}。

在 `onResume` 开启相机：

~~~ java
@Override
public void onResume() {
    super.onResume();
    
    CameraManager manager = (CameraManager) activity.getSystemService(Context.CAMERA_SERVICE);
    try {
        manager.openCamera(mCameraId, mStateCallback, mBackgroundHandler);
    } catch (CameraAccessException e) {
        e.printStackTrace();
    }
}
~~~

在回调中开启预览：

~~~ java
@Override
public void onOpened(@NonNull CameraDevice cameraDevice) {
    createCameraPreviewSession();
}

private void createCameraPreviewSession() {
    try {
        SurfaceTexture texture = mTextureView.getSurfaceTexture();

        // This is the output Surface we need to start preview.
        Surface surface = new Surface(texture);

        // We set up a CaptureRequest.Builder with the output Surface.
        mPreviewRequestBuilder
                = mCameraDevice.createCaptureRequest(CameraDevice.TEMPLATE_PREVIEW);
        mPreviewRequestBuilder.addTarget(surface);

        // Here, we create a CameraCaptureSession for camera preview.
        mCameraDevice.createCaptureSession(Collections.singletonList(surface),
                new CameraCaptureSession.StateCallback() {

                    @Override
                    public void onConfigured(@NonNull CameraCaptureSession cameraCaptureSession) {
                        // The camera is already closed
                        if (null == mCameraDevice) {
                            return;
                        }

                        // When the session is ready, we start displaying the preview.
                        mCaptureSession = cameraCaptureSession;
                        try {
                            // Auto focus should be continuous for camera preview.
                            mPreviewRequestBuilder.set(CaptureRequest.CONTROL_AF_MODE,
                                    CaptureRequest.CONTROL_AF_MODE_CONTINUOUS_VIDEO);
                            // Finally, we start displaying the camera preview.
                            mPreviewRequest = mPreviewRequestBuilder.build();
                            mCaptureSession.setRepeatingRequest(mPreviewRequest,
                                    null, mBackgroundHandler);
                        } catch (CameraAccessException e) {
                            e.printStackTrace();
                        }
                    }

                    @Override
                    public void onConfigureFailed(
                            @NonNull CameraCaptureSession cameraCaptureSession) {
                        showToast("Failed");
                    }
                }, null);
    } catch (CameraAccessException e) {
        e.printStackTrace();
    }
}
~~~

如果我们需要获取预览每一帧的数据，我们需要利用 `ImageReader` 类，我们需要修改 `createCameraPreviewSession` 方法：

~~~ java
// 初始化 ImageReader
mImageReader = ImageReader.newInstance(largest.getWidth(), largest.getHeight(),
        ImageFormat.YUV_420_888, 2);
mImageReader.setOnImageAvailableListener(
        mOnImageAvailableListener, mBackgroundHandler);

private void createCameraPreviewSession() {
    try {
        // ...
        
        mPreviewRequestBuilder
                = mCameraDevice.createCaptureRequest(CameraDevice.TEMPLATE_PREVIEW);
        mPreviewRequestBuilder.addTarget(surface);
        mPreviewRequestBuilder.addTarget(mImageReader.getSurface());

        // Here, we create a CameraCaptureSession for camera preview.
        mCameraDevice.createCaptureSession(Arrays.asList(surface, mImageReader.getSurface()),
                /*...*/, null);
    } catch (CameraAccessException e) {
        e.printStackTrace();
    }
}
~~~

然后我们就可以在 `mOnImageAvailableListener` 回调中处理预览帧数据了：

~~~ java
@Override
public void onImageAvailable(ImageReader reader) {
    Image image = reader.acquireLatestImage();
    if (image != null) {
        // image 对象封装了帧数据，可以用来处理（编码推流、美颜滤镜等）
        // 此回调在指定的后台线程 `mBackgroundHandler` 执行
        // 处理完毕之后需要调用 close 方法，否则预览将会停止
        image.close();
    }
}
~~~

需要注意的是，通过 `Image` 提供的 API，我们获得的是三个 `byte` 数组，我们需要将它们组合为一个 yuv 格式数组，具体方法可以参考[这篇文章](http://www.polarxiong.com/archives/Android-YUV_420_888%E7%BC%96%E7%A0%81Image%E8%BD%AC%E6%8D%A2%E4%B8%BAI420%E5%92%8CNV21%E6%A0%BC%E5%BC%8Fbyte%E6%95%B0%E7%BB%84.html){:target="_blank"}。

### `GLSurfaceView` 美颜预览

和正常预览相比，我们只需要在开启相机之前，初始化 `GLSurfaceView` 和 `GPUImageRenderer` 即可：

~~~ java
@Override
public void onResume() {
    super.onResume();
    
    mRenderer = new GPUImageRenderer(new GPUImageColorInvertFilter());
    if (Utils.isSupportOpenGLES2(getContext())) {
        mGLSurfaceView.setEGLContextClientVersion(2);
    }
    mGLSurfaceView.setEGLConfigChooser(8, 8, 8, 8, 16, 0);
    mGLSurfaceView.getHolder().setFormat(PixelFormat.RGBA_8888);
    mGLSurfaceView.setRenderer(mRenderer);
    mGLSurfaceView.setRenderMode(GLSurfaceView.RENDERMODE_WHEN_DIRTY);
    mGLSurfaceView.requestRender();
    
    // 开启相机
    // ...
}
~~~

在开启相机回调中初始化 texture 并开启预览：

~~~ java
@Override
public void onOpened(@NonNull CameraDevice cameraDevice) {
    mGLSurfaceView.setRenderMode(GLSurfaceView.RENDERMODE_CONTINUOUSLY);
    mRenderer.setUpSurfaceTexture(new SurfaceTextureInitCallback() {
        @Override
        public void onSurfaceTextureInitiated(SurfaceTexture surfaceTexture) {
            createCameraPreviewSession(surfaceTexture);
        }
    });
}
~~~

与正常预览版本不同的是，我们把在 `createCameraPreviewSession` 函数中创建的 `texture` 对象改为参数传入了，因为它会在 GPUImage 库中进行创建，其他就和获取预览帧数据的版本完全一样了。

但是我们需要在数据处理时调用 GPUImage 的 API：

~~~ java
@Override
public void onImageAvailable(ImageReader reader) {
    final Image image = reader.acquireLatestImage();
    if (image != null) {
        final byte[] data = ImageUtils.getDataFromImage(image, ImageUtils.COLOR_FormatI420);
        Size size = new Size(image.getWidth(), image.getHeight());
        mRenderer.onFrameData(data, size, new Runnable() {
            @Override
            public void run() {
                image.close();
            }
        });
    }
}
~~~

## 方向问题

### Activity 显示方向

~~~ java
int rotation = getWindowManager().getDefaultDisplay().getRotation();
~~~

返回值可能为 `Surface.ROTATION_0`，`Surface.ROTATION_90`，`Surface.ROTATION_180`，`Surface.ROTATION_270`。

根据其 Javadoc，手机基本上都是长方形，物理上有上下左右之分（菜单、home、返回键认为在下方），如果界面元素的顶部也在物理上方，那就是 0，如果界面元素顶部在物理右方，那就是 90，顺时针依次类推。

### Camera 方向

#### Camera1

~~~ java
Camera.CameraInfo info = new Camera.CameraInfo();
Camera.getCameraInfo(mIsFront ? Camera.CameraInfo.CAMERA_FACING_FRONT
        : Camera.CameraInfo.CAMERA_FACING_BACK, info);
// info.orientation 就是相机的角度
~~~

在三星 A7 手机上实测，后置相机，读出的结果为 90，前置相机读出的结果为 270，它们预览时在屏幕上方向是一样的，图像顶部都在物理左侧。

Javadoc 中的描述有些难以理解：如果后置相机图像的顶部和手机物理右方对齐（aligned），那就是 90，如果前置相机图像的顶部和手机物理右方对齐，那就是 270。 _但和我们实测不太一致，是手机硬件问题，还是“对齐”意思理解的问题？我测了三星 A7，魅族 MX5，华为 P8，结果均一样，我怀疑是理解问题，或者是文档错误。_

#### Camera2

~~~ java
CameraCharacteristics characteristics;
Integer orientation = null;
try {
    characteristics = mCameraManager.getCameraCharacteristics(cameraId);
    orientation = characteristics.get(CameraCharacteristics.SENSOR_ORIENTATION);
} catch (CameraAccessException e) {
    e.printStackTrace();
}
int sensorOrientation = orientation == null ? 0 : orientation;
// sensorOrientation 就是相机的角度
~~~


http://mp.weixin.qq.com/s?__biz=MzAxMzYyNDkyNA==&mid=2651332618&idx=1&sn=9192fd1f027f2e1ae287e6f630634b5f&chksm=80633339b714ba2fed52c1350bfdc681ac9386020540e29eaa4cd8727f2b6182fc9ad8c62c3b&mpshare=1&scene=1&srcid=1124GrEPm9TUpxW5nwmjQSSH#rd