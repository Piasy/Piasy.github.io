---
layout: post
title: 安卓 OpenGL ES 2.0 完全入门（一）：基本概念和 hello world
tags:
    - 安卓开发
    - OpenGL
---

做安卓开发满打满算也有 3 年了，OpenGL 这块之前完全没有涉及过，这两周一直在整理安卓相机预览、用 GPUImage 进行美颜处理以及美颜后的数据传输这块内容，结果 GPUImage 的美颜原理基本一窍不通，因此就把 OpenGL ES 好好入了个门，并且整理为 [安卓 OpenGL ES 2.0 完全入门](/tags/#OpenGL) 系列。本文是系列第一篇，主要是介绍了 OpenGL 的一些基本概念，并且包含了对一个 hello world 程序的完全解析，注意，并不是有一个 hello world，而是对其进行了完全解析！

_Update 2017.07.16：时隔一年，工作再次涉及安卓平台 OpenGL 相关内容，趁此机会又学习了一次[基础知识](http://www.learnopengles.com/tag/opengl-es-2-for-android-a-quick-start-guide/)，并对本文做了[更新](https://github.com/Piasy/Piasy.github.io/commits/master/_posts/2016-06-07-Open-gl-es-android-2-part-1.md)。_

## 1. 基本概念

在 OpenGL 的世界里，我们只能画点、线、三角形，复杂的图形都是由三角形构成的。

在 OpenGL 里有两个最基本的概念：Vertex 和 Fragment。一切图形都从 Vertix 开始，Vertix 序列围成了一个图形。那什么是 Fragment 呢？为此我们需要了解一下光栅化（Rasterization）：光栅化是把点、线、三角形映射到屏幕上的像素点的过程（每个映射区域叫一个 Fragment），也就是生成 Fragment 的过程。通常一个 Fragment 对应于屏幕上的一个像素，但高分辨率的屏幕可能会用多个像素点映射到一个 Fragment，以减少 GPU 的工作。

![](https://imgs.piasy.com/2017-07-06-rasterization_generate_fragments.png)

接下来介绍 Shader（着色器程序）：Shader 用来描述如何绘制（渲染），GLSL 是 OpenGL 的编程语言，全称 OpenGL Shader Language，它的语法类似于 C 语言。OpenGL 渲染需要两种 Shader：Vertex Shader 和 Fragment Shader。

每个 Vertex 都会执行一遍 Vertex Shader，以确定 Vertex 的最终位置，其 main 函数中必须设置 `gl_Position` 全局变量，它将作为该 Vertex 的最终位置，进而把 Vertex 组合（assemble）成点、线、三角形。光栅化之后，每个 Fragment 都会执行一次 Fragment Shader，以确定每个 Fragment 的颜色，其 main 函数中必须设置 `gl_FragColor` 全局变量，它将作为该 Fragment 的最终颜色。

下面是 OpenGL 的处理过程：

![](https://imgs.piasy.com/2017-07-06-opengl_pipeline.png)

## 2. 坐标系

弄清楚坐标系很重要，不然会找不着东南西北。下面这张图展示了 OpenGL 通常的处理流程中各个环节的坐标系，以及坐标系之间的转换操作：

![](https://imgs.piasy.com/2017-07-17-opengl_coordinate_systems.png)

下面对图中的几个概念作简单说明：

+ Local space：我们为每个物体建好模型的时候，它们的坐标就是 Local space 坐标；
+ World space：当我们要绘制多个物体时，如果直接使用 Local space 的坐标（把所有物体的原点放在一起），那它们很可能会发生重叠，因此我们需要把它们进行合理的移动、排布，最终各自的坐标就是 World space 的坐标了；
+ Model matrix：把 Local space 坐标转换到 World space 坐标所使用的变换矩阵，它是针对每个物体做不同的变换；
+ View space：通常也叫 Camera space 或者 Eye space，是从观察者（也就是我们自己）所在的位置出发，所看到的空间；
+ View matrix：把 World space 坐标转换到 View space 坐标所使用的变换矩阵，它相当于是在移动相机位置，实际上是反方向移动整个场景（所有物体）；
+ Clip space：OpenGL 只会渲染坐标值范围在 `[-1, 1]` 的内容，超出这个范围的内容都会被裁剪掉，这个范围的空间就叫 Clip space，Clip space 的坐标系也叫 normalized device coordinate（NDC）；
+ Projection matrix：把 View space 坐标转换到 Clip space 坐标所使用的变换矩阵，它会指定一个可见的范围，只有这个范围内的点才会转换到 NDC 中，而这个范围被称作视锥（frustum）；projection matrix 有三种创建方式：正投影（Orthographic projection），透视投影（Perspective projection），以及 3D 投影（3D projection）；前两种比较常用；
+ Screen space：屏幕上的空间，`glViewport` 调用指定的区域；
+ Viewport transform：这一步是 Open GL 自动完成的，把 Clip space 坐标转换到 Screen space 坐标；

通常说的 Model，View，Projection 这三种变换都是针对 Vertex 坐标做的变换，也就是：

![](https://imgs.piasy.com/2017-07-17-opengl_transform_together.png)

更多关于坐标系的内容，可以阅读 [Learn OpenGL 的 Coordinate Systems 部分](https://learnopengl.com/#!Getting-started/Coordinate-Systems)。

## 3. Hello world

先看一下最简单的完整例子（下文有详细分析，完整代码可以在 [GitHub 获取](https://github.com/Piasy/OpenGLESTutorial-Android/blob/blog_demo/app/src/main/java/com/github/piasy/openglestutorial_android/MainActivity.java){:target="_blank"}），绘制一个三角形，效果如图一：

<img src="https://imgs.piasy.com/2017-07-18-open_gl_triangle.png" alt="open_gl_triangle.png" style="height:400px">

~~~ java
public class MainActivity extends AppCompatActivity {

    private GLSurfaceView mGLSurfaceView;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        if (!Utils.supportGlEs20(this)) {
            Toast.makeText(this, "GLES 2.0 not supported!", Toast.LENGTH_LONG).show();
            finish();
            return;
        }

        mGLSurfaceView = (GLSurfaceView) findViewById(R.id.surface);

        mGLSurfaceView.setEGLContextClientVersion(2);
        mGLSurfaceView.setEGLConfigChooser(8, 8, 8, 8, 16, 0);
        mGLSurfaceView.setRenderer(new MyRenderer());
        mGLSurfaceView.setRenderMode(GLSurfaceView.RENDERMODE_CONTINUOUSLY);
    }

    @Override
    protected void onPause() {
        super.onPause();
        mGLSurfaceView.onPause();
    }

    @Override
    protected void onResume() {
        super.onResume();
        mGLSurfaceView.onResume();
    }

    private static class MyRenderer implements GLSurfaceView.Renderer {

        private static final String VERTEX_SHADER =
                "attribute vec4 vPosition;\n"
                + "void main() {\n"
                + "  gl_Position = vPosition;\n"
                + "}";
        private static final String FRAGMENT_SHADER =
                "precision mediump float;\n"
                + "void main() {\n"
                + "  gl_FragColor = vec4(0.5, 0, 0, 1);\n"
                + "}";
        private static final float[] VERTEX = {   // in counterclockwise order:
                0, 1, 0,  // top
                -0.5f, -1, 0,  // bottom left
                1, -1, 0,  // bottom right
        };

        private final FloatBuffer mVertexBuffer;

        private int mProgram;
        private int mPositionHandle;

        MyRenderer() {
            mVertexBuffer = ByteBuffer.allocateDirect(VERTEX.length * 4)
                    .order(ByteOrder.nativeOrder())
                    .asFloatBuffer()
                    .put(VERTEX);
            mVertexBuffer.position(0);
        }

        static int loadShader(int type, String shaderCode) {
            int shader = GLES20.glCreateShader(type);
            GLES20.glShaderSource(shader, shaderCode);
            GLES20.glCompileShader(shader);
            return shader;
        }

        @Override
        public void onSurfaceCreated(GL10 unused, EGLConfig config) {
            GLES20.glClearColor(0.0f, 0.0f, 0.0f, 0.0f);

            mProgram = GLES20.glCreateProgram();
            int vertexShader = loadShader(GLES20.GL_VERTEX_SHADER, VERTEX_SHADER);
            int fragmentShader = loadShader(GLES20.GL_FRAGMENT_SHADER, FRAGMENT_SHADER);
            GLES20.glAttachShader(mProgram, vertexShader);
            GLES20.glAttachShader(mProgram, fragmentShader);
            GLES20.glLinkProgram(mProgram);

            GLES20.glUseProgram(mProgram);

            mPositionHandle = GLES20.glGetAttribLocation(mProgram, "vPosition");

            GLES20.glEnableVertexAttribArray(mPositionHandle);
            GLES20.glVertexAttribPointer(mPositionHandle, 3, GLES20.GL_FLOAT, false,
                    12, mVertexBuffer);
        }

        @Override
        public void onSurfaceChanged(GL10 unused, int width, int height) {
            GLES20.glViewport(0, 0, width, height);
        }

        @Override
        public void onDrawFrame(GL10 unused) {
            GLES20.glClear(GLES20.GL_COLOR_BUFFER_BIT);

            GLES20.glDrawArrays(GLES20.GL_TRIANGLES, 0, 3);
        }
    }
}
~~~

### 3.1. set up

首先我们需要一个 `GLSurfaceView`，它是让我们渲染的“画布”。然后我们需要一个 `GLSurfaceView.Renderer`，它将实现我们的渲染逻辑。此外我们还将设置 GL ES 版本，并将 GLSurfaceView 和 Renderer 连接起来：

~~~ java
mGLSurfaceView.setEGLContextClientVersion(2);
mGLSurfaceView.setEGLConfigChooser(8, 8, 8, 8, 16, 0);
mGLSurfaceView.setRenderer(new MyRenderer());
mGLSurfaceView.setRenderMode(GLSurfaceView.RENDERMODE_CONTINUOUSLY);
~~~

RenderMode 有两种，`RENDERMODE_WHEN_DIRTY` 和 `RENDERMODE_CONTINUOUSLY`，前者是懒惰渲染，需要手动调用 `glSurfaceView.requestRender()` 才会进行更新，而后者则是不停渲染。

### 3.2. `GLSurfaceView.Renderer`

Renderer 包含三个接口：

~~~ java
public interface Renderer {
    void onSurfaceCreated(GL10 gl, EGLConfig config);
    void onSurfaceChanged(GL10 gl, int width, int height);
    void onDrawFrame(GL10 gl);
}
~~~

`onSurfaceCreated` 在 surface 创建时被回调，通常用于进行初始化工作，只会被回调一次；`onSurfaceChanged` 在每次 surface 尺寸变化时被回调，注意，第一次得知 surface 的尺寸时也会回调；`onDrawFrame` 则在绘制每一帧的时候回调。

### 3.3. GLSL 程序

和普通的 view 利用 canvas 来绘制不一样，OpenGL 需要加载 GLSL 程序，让 GPU 进行绘制。所以我们需要定义 shader 代码，并在初始化时（也就是 `onSurfaceChanged` 回调中）加载：

~~~ java
@Override
public void onSurfaceCreated(GL10 unused, EGLConfig config) {
    GLES20.glClearColor(0.0f, 0.0f, 0.0f, 0.0f);

    mProgram = GLES20.glCreateProgram();
    int vertexShader = loadShader(GLES20.GL_VERTEX_SHADER, VERTEX_SHADER);
    int fragmentShader = loadShader(GLES20.GL_FRAGMENT_SHADER, FRAGMENT_SHADER);
    GLES20.glAttachShader(mProgram, vertexShader);
    GLES20.glAttachShader(mProgram, fragmentShader);
    GLES20.glLinkProgram(mProgram);

    GLES20.glUseProgram(mProgram);

    mPositionHandle = GLES20.glGetAttribLocation(mProgram, "vPosition");

    GLES20.glEnableVertexAttribArray(mPositionHandle);
    GLES20.glVertexAttribPointer(mPositionHandle, 3, GLES20.GL_FLOAT, false,
            12, mVertexBuffer);
}

static int loadShader(int type, String shaderCode) {
    int shader = GLES20.glCreateShader(type);
    GLES20.glShaderSource(shader, shaderCode);
    GLES20.glCompileShader(shader);
    return shader;
}
~~~

GLSL 的语法并不是本文的主要内容，这里就不深入展开了。

+ 创建 GLSL 程序：`glCreateProgram`
+ 加载 shader 代码：`glShaderSource` 和 `glCompileShader`
+ attatch shader 代码：`glAttachShader`
+ 链接 GLSL 程序：`glLinkProgram`
+ 使用 GLSL 程序：`glUseProgram`
+ 获取 shader 代码中的变量索引：`glGetAttribLocation`
+ 启用 vertex：`glEnableVertexAttribArray`
+ 绑定 vertex 坐标值：`glVertexAttribPointer`

需要指出的是，我们的 Java 代码需要获取 shader 代码中定义的变量索引，用于在后面的绘制代码中进行赋值，变量索引在 GLSL 程序的生命周期内（链接之后和销毁之前），都是固定的，只需要获取一次。

### 3.4. 设置 Screen space 的大小

我们可以利用 `glViewport` 设置 Screen space 的大小，通常在 `onSurfaceChanged` 中调用：

~~~ java
@Override
public void onSurfaceChanged(GL10 unused, int width, int height) {
    GLES20.glViewport(0, 0, width, height);
}
~~~

### 3.5. 绘制

我们在 `onDrawFrame` 回调中执行绘制操作，绘制的过程其实就是为 shader 代码变量赋值，并调用绘制命令的过程：

~~~ java
@Override
public void onDrawFrame(GL10 unused) {
    GLES20.glClear(GLES20.GL_COLOR_BUFFER_BIT);

    GLES20.glDrawArrays(GLES20.GL_TRIANGLES, 0, 3);
}
~~~

由于顶点坐标已经绑定过了，所以这里无需进行变量赋值，直接调用绘制指令即可。我们可以通过 `GLES20.glDrawArrays` 或者 `GLES20.glDrawElements` 开始绘制。注意，执行完毕之后，GPU 就在显存中处理好帧数据了，但此时并没有更新到 surface 上，是 `GLSurfaceView` 会在调用 `renderer.onDrawFrame` 之后，调用 `eglSwapBuffers`，来把显存的帧数据更新到 surface 上的。

这里我们绘制的是一个三角形，OpenGL 坐标原点在屏幕中心，三个顶点分别是：

+ `(0, 1, 0)`，位于屏幕顶部中心点；
+ `(-0.5f, -1, 0)`，位于屏幕底部四分之一点；
+ `(1, -1, 0)`，位于屏幕右下角；

它们的 z 轴坐标都是 0，所以也就是上面图一的效果了。

## 4. 投影变换

你肯定已经注意到，OpenGL 坐标系和安卓手机坐标系不是线性对应的，因为手机的宽高比几乎都不是 1。因此我们绘制的形状是变形的，怎么解决这个问题呢？答案是投影变换（projection）。前面我们已经知道，投影变换用于把 View space 的坐标转换为 Clip space 的坐标，在这个转换过程中，它还能顺带处理宽高比的问题。

使用较多的是正投影和透视投影，这里我们使用透视投影：`Matrix.perspectiveM`。通常坐标系的变换都是对顶点坐标进行矩阵左乘运算，因此我们需要修改我们的 vertex shader 代码：

~~~ java
private static final String VERTEX_SHADER = 
        "attribute vec4 vPosition;\n"
        + "uniform mat4 uMVPMatrix;\n"
        + "void main() {\n"
        + "  gl_Position = uMVPMatrix * vPosition;\n"
        + "}";
~~~

然后我们需要在 `onSurfaceCreated` 中获取 `uMVPMatrix` 的索引：

~~~ java
@Override
public void onSurfaceCreated(GL10 unused, EGLConfig config) {
    // ...

    mPositionHandle = GLES20.glGetAttribLocation(mProgram, "vPosition");
    mMatrixHandle = GLES20.glGetUniformLocation(mProgram, "uMVPMatrix");

    // ...
}
~~~

并在 `onSurfaceChanged` 中计算变换矩阵：

~~~ java
@Override
public void onSurfaceChanged(GL10 unused, int width, int height) {
    GLES20.glViewport(0, 0, width, height);

    Matrix.perspectiveM(mMVPMatrix, 0, 45, (float) width / height, 0.1f, 100f);
    Matrix.translateM(mMVPMatrix, 0, 0f, 0f, -2.5f);
}
~~~

`Matrix.perspectiveM`，`Matrix.translateM`？先别急，我将在下文进行详细解释。

最后我们在绘制的时候为 `uMVPMatrix` 赋值：

~~~ java
@Override
public void onDrawFrame(GL10 unused) {
    GLES20.glClear(GLES20.GL_COLOR_BUFFER_BIT);

    GLES20.glUniformMatrix4fv(mMatrixHandle, 1, false, mMVPMatrix, 0);

    GLES20.glDrawArrays(GLES20.GL_TRIANGLES, 0, 3);
}
~~~

经过这样的变换之后，绘制的效果如图二：

<img src="https://imgs.piasy.com/2017-07-18-open_gl_triangle_projected_camera.png" alt="open_gl_triangle_projected_camera.png" style="height:400px">

### 4.1. `perspectiveM` 和 `translateM`

perspectiveM 就是我们所说的透视投影了，我们只需要提供几个参数，就可以得到投影矩阵，用于投影变换了。下面我将详细分析每个参数的含义：

~~~ java
public static void perspectiveM(float[] m, int offset,
        float fovy, float aspect, float zNear, float zFar)
        
Matrix.perspectiveM(mMVPMatrix, 0, 45, (float) width / height, 0.1f, 100f);
~~~

前两个参数不用多说，Javadoc 里面就有，`m` 是保存变换矩阵的数组，`offset` 是开始保存的下标偏移量。

fovy 是 y 轴的 field of view 值，也就是视角大小，视角越大，我们看到的范围就越广，例如下面的 90° 和 45°：

![](https://imgs.piasy.com/2017-07-18-open_gl_perspective_fov.png)

`aspect` 是 Screen space 的宽高比。`zNear` 和 `zFar` 则是视锥体近平面和远平面的 z 轴坐标了。

由于历史原因，`Matrix.perspectiveM` 会让 z 轴方向倒置，所以左乘投影矩阵之后，顶点 z 坐标需要在 `-zNear~-zFar` 范围内才会可见。

前面我们顶点的 z 坐标都是 0，我们可以把它修改为 `-0.1f~-100f` 之间的值，也可以通过一个位移变换来达到此目的：

~~~ java
public static void translateM(
        float[] m, int mOffset,
        float x, float y, float z)

Matrix.translateM(mMVPMatrix, 0, 0f, 0f, -2.5f);
~~~

我们沿着 z 轴的反方向移动 2.5，这样就能把 z 坐标移到 `-0.1f~-100f` 了。

## 5. 小结

讲完 hello world 文章有已经这么长了，所以其他的还是放到后面的部分吧，包括：渲染矩形、渲染图片纹理、读取显存数据（保存为图片或者网络传输）、GLSL 简介等，GLSL 可能不会太深入，不要太期待 :)
