---
layout: post
title: 安卓 OpenGL ES 2.0 完全入门（一）：基本概念和 hello world
tags:
    - 安卓开发
    - OpenGL
---

做安卓开发满打满算也有 3 年了，OpenGL 这块之前完全没有涉及过，这两周一直在整理安卓相机预览、用 GPUImage 进行美颜处理以及美颜后的数据传输这块内容，结果 GPUImage 的美颜原理基本一窍不通，因此就把 OpenGL ES 好好入了个门，并且整理为 _安卓 OpenGL ES 2.0 完全入门_ 系列。本文是系列第一篇，主要是介绍了 OpenGL 的一些基本概念，并且包含了对一个 hello world 程序的完全解析，注意，并不是有一个 hello world，而是对其进行了完全解析！

## 1. 基本概念

+ OpenGL 绘制的都是图形，包括形状和填充，基本形状是三角形。
+ 每个形状都有顶点，Vertix，顶点的序列就是一个图形。
+ 图形有所谓的正反面，如果我们看向一个图形，它的顶点序列是逆时针方向，那我们看到的就是正面。
+ Shader，着色器，用来描述如何绘制（渲染），GLSL 是 OpenGL 的编程语言，全称就叫 OpenGL Shader Language。OpenGL 渲染需要两种 shader，vertex 和 fragment。
+ Vertex shader，控制顶点的绘制，指定坐标、变换等。
+ Fragment shader，控制形状内区域渲染，纹理填充内容。

## 2. 坐标系

弄清楚坐标系很重要，不然会找不着东南西北。

### 2.1. 安卓手机坐标系

二维坐标系，原点在左上角，x 轴向右，y 轴向下，x y 取值范围为屏幕分辨率：

<img src="/img/201606/android-coordinates.png" alt="/img/201606/android-coordinates.png" style="height:300px">

### 2.2. OpenGL 坐标系

三维坐标系，原点在中间，x 轴向右，y 轴向上，z 轴朝向我们，x y z 取值范围都是 [-1, 1]：

<img src="/img/201606/open-gl-coordinates.png" alt="/img/201606/open-gl-coordinates.png" style="height:300px">

### 2.3. OpenGL 纹理（texture）坐标系

二维坐标系，原点在左下角，s（x）轴向右，t（y）轴向上，x y 取值范围都是 [0, 1]：

<img src="/img/201606/open-gl-texture-coordinates.png" alt="/img/201606/open-gl-texture-coordinates.png" style="height:300px">

## 3. Hello world

先看一下最简单的完整例子（下文有详细分析，完整代码可以在 [GitHub 获取](https://github.com/Piasy/AndroidPlayground/blob/90030a2ebfc70e948106a2db451df2db801742eb/try/TryOpenGL/src/main/java/com/github/piasy/tryopengl/MainActivity.java){:target="_blank"}），绘制一个三角形，效果如图一：

<img src="/img/201606/open_gl_triangle.png" alt="/img/201606/open_gl_triangle.png" style="height:400px">

~~~ java
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        GLSurfaceView glSurfaceView =
                (android.opengl.GLSurfaceView) findViewById(R.id.mGLSurfaceView);

        glSurfaceView.setEGLContextClientVersion(2);
        glSurfaceView.setRenderer(new MyRenderer());
        glSurfaceView.setRenderMode(GLSurfaceView.RENDERMODE_CONTINUOUSLY);
    }

    static class MyRenderer implements GLSurfaceView.Renderer {
        private static final String VERTEX_SHADER = "attribute vec4 vPosition;\n"
                + "void main() {\n"
                + "  gl_Position = vPosition;\n"
                + "}";
        private static final String FRAGMENT_SHADER = "precision mediump float;\n"
                + "void main() {\n"
                + "  gl_FragColor = vec4(0.5,0,0,1);\n"
                + "}";
        private static final float[] VERTEX = {   // in counterclockwise order:
                0,      1,      0,  // top
                -0.5,   -1,     0,  // bottom left
                1,      -1,     0,  // bottom right
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

        @Override
        public void onSurfaceCreated(GL10 unused, EGLConfig config) {
        }

        @Override
        public void onSurfaceChanged(GL10 unused, int width, int height) {
            mProgram = GLES20.glCreateProgram();
            int vertexShader = loadShader(GLES20.GL_VERTEX_SHADER, VERTEX_SHADER);
            int fragmentShader = loadShader(GLES20.GL_FRAGMENT_SHADER, FRAGMENT_SHADER);
            GLES20.glAttachShader(mProgram, vertexShader);
            GLES20.glAttachShader(mProgram, fragmentShader);
            GLES20.glLinkProgram(mProgram);
            
            mPositionHandle = GLES20.glGetAttribLocation(mProgram, "vPosition");
        }

        @Override
        public void onDrawFrame(GL10 unused) {
            GLES20.glClear(GLES20.GL_COLOR_BUFFER_BIT | GLES20.GL_DEPTH_BUFFER_BIT);
    
            GLES20.glUseProgram(mProgram);

            GLES20.glEnableVertexAttribArray(mPositionHandle);
            GLES20.glVertexAttribPointer(mPositionHandle, 3, GLES20.GL_FLOAT, false,
                    12, mVertexBuffer);

            GLES20.glDrawArrays(GLES20.GL_TRIANGLES, 0, 3);

            GLES20.glDisableVertexAttribArray(mPositionHandle);
        }

        static int loadShader(int type, String shaderCode) {
            int shader = GLES20.glCreateShader(type);
            GLES20.glShaderSource(shader, shaderCode);
            GLES20.glCompileShader(shader);
            return shader;
        }
    }
}
~~~

### 3.1. set up

首先我们需要一个 `GLSurfaceView`，它是让我们渲染的“画布”，更准确来说，是一个 surface。然后我们需要一个 `GLSurfaceView.Renderer`，它将实现我们的渲染逻辑。此外我们还将设置 GL ES 版本，并将 surface view 和 renderer 连接起来：

~~~ java
glSurfaceView.setEGLContextClientVersion(2);
glSurfaceView.setRenderer(new MyRenderer());
glSurfaceView.setRenderMode(GLSurfaceView.RENDERMODE_CONTINUOUSLY);
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

有的时候，我们的初始化工作可能需要依赖 surface 的尺寸，所以这里我们把初始化工作放到了 `onSurfaceChanged` 方法中。

### 3.3. GLSL 程序

和普通的 view 利用 canvas 来绘制不一样，OpenGL 需要加载 GLSL 程序，让 GPU 进行绘制。所以我们需要定义 shader 代码，并在 `onSurfaceChanged` 回调中加载：

~~~ java
@Override
public void onSurfaceChanged(GL10 unused, int width, int height) {
    mProgram = GLES20.glCreateProgram();
    int vertexShader = loadShader(GLES20.GL_VERTEX_SHADER, VERTEX_SHADER);
    int fragmentShader = loadShader(GLES20.GL_FRAGMENT_SHADER, FRAGMENT_SHADER);
    GLES20.glAttachShader(mProgram, vertexShader);
    GLES20.glAttachShader(mProgram, fragmentShader);
    GLES20.glLinkProgram(mProgram);
    
    mPositionHandle = GLES20.glGetAttribLocation(mProgram, "vPosition");
}

static int loadShader(int type, String shaderCode) {
    int shader = GLES20.glCreateShader(type);
    GLES20.glShaderSource(shader, shaderCode);
    GLES20.glCompileShader(shader);
    return shader;
}
~~~

GLSL 的语法并不是本文的主要内容，所以就不深入展开。我们的 Java 代码需要获取 shader 代码中定义的变量索引，并在稍后的绘制代码中进行赋值。

+ 创建 GLSL 程序：`mProgram = GLES20.glCreateProgram()`
+ 加载 shader 代码：`vertexShader = loadShader(GLES20.GL_VERTEX_SHADER, VERTEX_SHADER)`
+ attatch shader 代码：`GLES20.glAttachShader(mProgram, vertexShader)`
+ 链接 shader 代码：`GLES20.glLinkProgram(mProgram)`
+ 获取 shader 代码中的变量索引：`mPositionHandle = GLES20.glGetAttribLocation(mProgram, "vPosition")`

需要指出的是，shader 代码中的变量索引，在 GLSL 程序的生命周期内（两次链接之间），都是固定的，只需要获取一次。

### 3.4. 绘制

我们在 `onDrawFrame` 回调中执行绘制操作，绘制的过程其实就是为 shader 代码变量赋值，并调用绘制命令的过程：

~~~ java
@Override
public void onDrawFrame(GL10 unused) {
    GLES20.glClear(GLES20.GL_COLOR_BUFFER_BIT | GLES20.GL_DEPTH_BUFFER_BIT);
    
    GLES20.glUseProgram(mProgram);

    GLES20.glEnableVertexAttribArray(mPositionHandle);
    GLES20.glVertexAttribPointer(mPositionHandle, 3, GLES20.GL_FLOAT, false,
            12, mVertexBuffer);

    GLES20.glDrawArrays(GLES20.GL_TRIANGLES, 0, 3);

    GLES20.glDisableVertexAttribArray(mPositionHandle);
}
~~~

首先需要应用 GLSL 程序，对于 `attribute` 类型的变量，我们需要先 enable，再赋值，绘制完毕之后再 disable。我们可以通过 `GLES20.glDrawArrays` 或者 `GLES20.glDrawElements` 开始绘制。注意，执行完毕之后，GPU 就在显存中处理好帧数据了，但此时并没有更新到 surface 上，是 `GLSurfaceView` 会在调用 `renderer.onDrawFrame` 之后，调用 `mEglHelper.swap()`，来把显存的帧数据更新到 surface 上的。

这里我们绘制的是一个三角形，OpenGL 坐标原点在屏幕中心，三个顶点分别是：

+ `(0, 1, 0)`，位于屏幕顶部中心点；
+ `(-0.5, -1, 0)`，位于屏幕底部四分之一点；
+ `(1, -1, 0)`，位于屏幕右下角；

所以也就是上面图一的效果了。

## 4. 投影变换和相机视觉（Camera view）

你肯定已经注意到，OpenGL 坐标系和安卓手机坐标系不是线性对应的，因为手机的宽高比几乎都不是 1。因此我们绘制的形状是变形的，所以我们需要进行投影变换和相机视觉，使得渲染出来的图形不变形，并且更接近我们真实的视觉效果。关于投影变换的原理，可以参考[这篇博客](http://blog.csdn.net/popy007/article/details/1797121){:target="_blank"}。

根据 [Developer 文档](https://developer.android.com/training/graphics/opengl/projection.html#projection){:target="_blank"}，如果仅仅应用一个投影矩阵，将导致绘制出来的内容是空的，**必须要和相机视觉矩阵一起使用**。

投影变换和相机视觉实际上是用一个变换矩阵对我们的顶点坐标矩阵进行一个左乘，因此我们需要修改我们的 vertex shader 代码：

~~~ java
private static final String VERTEX_SHADER = "attribute vec4 vPosition;\n"
        + "uniform mat4 uMVPMatrix;\n"
        + "void main() {\n"
        + "  gl_Position = uMVPMatrix * vPosition;\n"
        + "}";
~~~

然后我们需要获取 `uMVPMatrix` 的索引并设置变换矩阵的值：

~~~ java
@Override
public void onSurfaceChanged(GL10 unused, int width, int height) {
    // ...
    
    mPositionHandle = GLES20.glGetAttribLocation(mProgram, "vPosition");
    mMatrixHandle = GLES20.glGetUniformLocation(mProgram, "uMVPMatrix");

    float ratio = (float) width / height;
    Matrix.frustumM(mProjectionMatrix, 0, -ratio, ratio, -1, 1, 3, 7);
    Matrix.setLookAtM(mCameraMatrix, 0, 0, 0, 3, 0, 0, 0, 0, 1, 0);
    Matrix.multiplyMM(mMVPMatrix, 0, mProjectionMatrix, 0, mCameraMatrix, 0);
}
~~~

`Matrix.frustumM`，`Matrix.setLookAtM`，`Matrix.multiplyMM`？先别急，我将在下文进行详细解释。

最后我们在绘制的时候为 `uMVPMatrix` 赋值：

~~~ java
@Override
public void onDrawFrame(GL10 unused) {
    // ...

    GLES20.glUniformMatrix4fv(mMatrixHandle, 1, false, mMVPMatrix, 0);

    GLES20.glDrawArrays(GLES20.GL_TRIANGLES, 0, 3);

    // ...
}
~~~

经过这样的变换之后，绘制的效果如图二：

<img src="/img/201606/open_gl_triangle_projected_camera.png" alt="/img/201606/open_gl_triangle_projected_camera.png" style="height:400px">

### 4.1. `Matrix.frustumM`

这个函数就是上文提到的[介绍投影变换原理的博客](http://blog.csdn.net/popy007/article/details/1797121){:target="_blank"}中介绍的投影算法的封装，我们只需要提供几个参数，就可以得到投影矩阵，用于投影变换了。下面我将详细分析每个参数的含义。

~~~ java
public static void frustumM(float[] m, int offset,
        float left, float right, float bottom, float top,
        float near, float far)
        
float ratio = (float) width / height;
Matrix.frustumM(mProjectionMatrix, 0, -ratio, ratio, -1, 1, 3, 7);
~~~

前两个参数不用多说，Javadoc 里面就有，`m` 是保存变换矩阵的数组，`offset` 是开始保存的下标偏移量。后面的 6 个参数就是那篇博客中提到的参数了，那我们为什么要为它们设置这样的值呢？

surface view 的宽高为 `width` 和 `height`，我们要把 OpenGL 坐标系投影到 surface view 上。首先我们忽略坐标系的方向问题（方向问题由相机视觉解决），以 OpenGL 的方向为准，我们把坐标原点置于 view 中心，这样 view 的 y 轴范围就是 `[- height / 2, height / 2]`，x 轴范围就是 `[- width / 2, width / 2]`。

如果我们希望 OpenGL 坐标系 y 坐标范围充满 surface view 的高，那我们就需要让 `[-1, 1]` 和 `[- height / 2, height / 2]` 映射起来。怎么做呢？除以 `height/ 2` 即可。此时 x 轴范围就变成了 `[- width / height, width / height]`，也就是 `[-ratio, ratio]` 了。这样我们就把 OpenGL 坐标系保持宽高比地投影到了屏幕上，高度占满全屏，宽度则超出了屏幕范围，所以我们看到图二的效果是，三角形比例正常，但左右均被截去了一部分。

我们当然可以把 x 坐标范围归一化为 `[-1, 1]`，这时我们的矩阵代码需要变成这样：

~~~ java
float ratio = (float) height / width;
Matrix.frustumM(mProjectionMatrix, 0, -1, 1, -ratio, ratio, 3, 7);
~~~

这时的效果就是图三了：

<img src="/img/201606/open_gl_triangle_projected_camera_x.png" alt="/img/201606/open_gl_triangle_projected_camera_x.png" style="height:400px">

这里我们可以看到，宽度占满全屏，高度则出现了富余，所以左右两个顶点的位置比例正常，宽高比也正常，但上下出现了空白。

那最后的 `near` 和 `far` 是怎么确定的呢？它们是投影时的近平面和远平面的 z 坐标。这两个值其实是和相机视觉一起用的，所以我们在下节中再介绍，现在只需要记住，`0 < near < far` 即可。

### 4.2. `Matrix.setLookAtM`

~~~ java
public static void setLookAtM(float[] rm, int rmOffset,
        float eyeX, float eyeY, float eyeZ,
        float centerX, float centerY, float centerZ, float upX, float upY,
        float upZ)
        
Matrix.setLookAtM(mCameraMatrix, 0, 0, 0, 3, 0, 0, 0, 0, 1, 0);
~~~

我们需要传入 9 个坐标值，(eyeX, eyeY, eyeZ)，(centerX, centerY, centerZ)，(upX, upY, upZ)。eye 表示相机的坐标点，center 表示物体（目标，或者图形）的中心坐标点，up 表示方向向量。

通常情况下，我们都把 center 设置为坐标原点。而由于上节中介绍的投影变换是投影到 x, y 平面，所以相机都在 z 轴上，至于在 z 轴的哪个点，就要结合调用 `Matrix.frustumM` 时的 `near` 和 `far` 参数了，~~`near <= eyeZ <= far` 时我们才能看到渲染的内容，否则屏幕上就是空白了~~。

`near`/`far` 和 eye 的坐标值没有必然联系，两者在不同的坐标系，所以 `near <= eyeZ <= far` 这是一个没有意义的不等式关系，非常感谢[铁蕾](http://zhangtielei.com/)纠正这一错误。这篇文章访问量还挺高，已经误导了不少朋友了，实在抱歉...另外这个问题我现在没有时间深入理解，所以只能贴出铁蕾勘误原文，待日后有时间理解了再更新。另外强烈推荐 [Learn OpenGL](https://learnopengl.com/)。

> `near <= eyeZ <= far` 并不是必须的。
>
> 现在的代码之所以有这个限制，是因为凑巧了。举两个例子：
>
>（1）首先说 eyeZ 比 near 小的情况。假设 `eyeZ = 2 < near = 3`，这个表达式这么比较其实本身没有什么意义，因为 2 在这里是世界坐标里的数值，而 3 是相机坐标里的数值；真正要展示的图片在世界坐标中 z 坐标都是 0；near = 3 如果转换回世界坐标那么 z 坐标对应 -1，因此图片 z 坐标 0 落在了视锥体 near plane 的前面，所以看不见。
>
>（2）再说 eyeZ 比 far 大的情况。假设 `eyeZ = 8 > far = 7`，同样，8 在这里是世界坐标里的数值，而 7 是相机坐标里的数值；真正要展示的图片在世界坐标中 z 坐标都是 0；far = 7 如果转换回世界坐标那么 z 坐标对应 1，因此图片 z 坐标 0 落在了视锥体 far plane 的后面，所以也看不见。
>
> 其实很容易举出反例，让 `near <= eyeZ <= far` 不成立。比如取 eyeZ = -4，其它参数都不变，那么相当于我们绕到了图片背后往回看，这时候也是能看到图片的，不过相当于是在电影幕布的后面看，所以看到的图片是左右镜像过的。这个你可以跑一下就能看到结果。

而 up 向量的设置我们就需要先看一下 OpenGL 坐标系了：

<img src="/img/201606/open_gl_triangle_projected_camera_coordinate.png" alt="/img/201606/open_gl_triangle_projected_camera_coordinate.png">

看清楚这幅图需要一点空间想象力 =_=

up 向量的长度无关紧要，重要的是方向。左图中，我们把 up 设置为 y 轴方向，即 (0, 1, 0)，我们让 up 向量指向我们的正上方（这时不需要动），那此时我们看到的样式就是 OpenGL 渲染出来的样子。而在右图中，我们把 up 设置为 x 轴方向，即 (1, 0, 0)，这时我们就需要先把图逆时针旋转 90°，所以 OpenGL 渲染出来的将是逆时针旋转 90° 之后的。有图有真相：

<img src="/img/201606/open_gl_triangle_projected_camera_up_vec_comp.png" alt="/img/201606/open_gl_triangle_projected_camera_up_vec_comp.png">

### 4.3. `Matrix.multiplyMM`

这个就不用多说了，矩阵乘法的封装函数。

## 5. 小结

讲完 hello world 文章有已经这么长了，所以其他的还是放到后面的部分吧，包括：渲染矩形、渲染图片纹理、读取显存数据（保存为图片或者网络传输）、GLSL 简介等，GLSL 可能不会太深入，不要太期待 :)
