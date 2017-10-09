---
layout: post
title: 安卓 OpenGL ES 2.0 完全入门（三）：2D 纹理的裁剪、翻转、旋转、缩放
tags:
    - 安卓开发
    - OpenGL
---

我在去年六月份学习了 OpenGL 的一些基本概念，整理了一个 demo 和两篇文章，并在今年六月份复习修正了一番。不久前我进一步向[铁蕾兄](http://zhangtielei.com/)学习了四种常用 2D 纹理变换的实现思路（以及本文中的其他总结性文字），由于铁蕾兄实在太忙，无暇快速整理成文，因此我就在这里为他代笔了 :)

没有阅读过前两篇、对 OpenGL 基本概念不熟悉的朋友，下面是链接：

+ [安卓 OpenGL ES 2.0 完全入门（一）：基本概念和 hello world](/2016/06/07/Open-gl-es-android-2-part-1/)
+ [安卓 OpenGL ES 2.0 完全入门（二）：矩形、图片、读取显存等](/2016/06/14/Open-gl-es-android-2-part-2/)

## 整体思路

在[基本概念和 hello world](/2016/06/07/Open-gl-es-android-2-part-1/) 中我们提到着色器程序（Shader）的最终目的就是确定图形的顶点（Vertex）坐标和片元（Fragment）颜色。其实这正是 OpenGL 提供的最基本、最核心的操作原语，我们想要用 OpenGL 实现任何效果，无论是静止的光影、色彩、形状，还是运动的物理效果、粒子效果，归根结底都是要根据时间和位置确定顶点坐标和片元颜色。不过这个归根结底说得轻巧，但其实现方法在图形学相关的领域里却有大量的研究课题，是非常广阔的领域，这里我就没有能力展开了。

本文将要展开讲的四种常用 2D 纹理变换，**其核心思想就是调整纹理坐标和顶点坐标**。

理想情况下，我们的纹理、图形、视口（view port）尺寸一致，那把纹理贴到图像上、把图形绘制到视口上这两个过程都不存在任何问题。次理想情况下，这三者的尺寸不一致但宽高比一致，那在这两个过程中使用最基本的缩放操作（即 OpenGL 默认的宽高填充，`FIT_XY`）也不会导致任何问题。

但实际情况往往不是这样，例如我们做全屏相机预览，相机采集的图像（将作为纹理）尺寸是 `640*480`，但我们希望将它显示在 `1920*1080` 的屏幕上，而且安卓相机采集的画面通常都是横着的，但我们却需要竖屏预览，那这就需要我们进行旋转和缩放操作了。下面我将依次分享纹理的裁剪、翻转、旋转、缩放操作的思路。

### 裁剪

如果我们绘制的图形是一个“全屏”的矩形（即顶点的横纵坐标范围都是 `[-1, 1]`），且纹理也是“全屏”的（即纹理的横纵坐标范围都是 `[0, 1]`），那我们就可以把纹理完全贴在图形上（图形会被全屏渲染到屏幕上）。如果我们希望裁掉各个方向上一定比例的像素，那我们可以把相应方向上的纹理坐标向中点（0.5）靠拢。例如把左右各 128 个像素的内容裁掉，我们就能不变形地把 `896*360` 的图像渲染到 `640*360` 的屏幕上。

为什么这样？下面这幅图解释了左右各裁掉 20%（`128 / 640 = 0.2`）的原理：

![](https://imgs.piasy.com/2017-10-07-opengl_crop_showcase.jpeg)

### 翻转

我们继续使用上面提出的场景，如果我们要实现左右翻转（也称水平翻转），那么把纹理的左边两个点和右边两个点对调即可，如果我们要实现上下翻转（也称垂直翻转），把纹理的上边两个点和下边两个点对调即可。

我本想也画一幅示意图，但发现难度较大，所以大家还是自己脑补一下，比如把自己的手机左右翻转一下，是不是左边跑到了右边？ :)

### 旋转

我们如果把纹理的顶点按左下、右下、左上、右上编号，逆时针旋转 90° 的变化如下图所示：

![](https://imgs.piasy.com/2017-10-07-opengl_rotate_showcase.jpeg)

这里我们相当于把位置和编号进行一个错位操作，比如原图的`左下 -> 右下 -> 右上 -> 左上`是 `0132`，逆时针旋转 90° 之后则变成了 `2013`，我们传递给 OpenGL 的纹理坐标是按位置的，传入不同的编号顺序即可达到旋转的效果，但需要是 `0123` 可以旋转出来的编号顺序，包括 `0132`、`2013`、`3201`、`1320`。

所以这种思路只能实现 90/180/270 的旋转，任意角度的旋转则需要结合投影矩阵实现，不过对纹理进行非 90/180/270 角度的旋转效果会很诡异，一般用不到。比如逆时针旋转 45° 的效果如下：

![](https://imgs.piasy.com/2017-10-07-opengl_preview_rotate_315.png)

可以看到左上、右上、右下三个角的画面变成了条纹状。

### 缩放

通常缩放会有三种模式：

+ `FIT_XY`：宽高都填充满，如果宽高比不一致，则会发生变形；
+ `CENTER_CROP`：短边填充满，长边等比例缩放，超出部分两端裁掉；
+ `CENTER_INSIDE`：长边填充满，短边等比例缩放，不足部分两端留黑边；

其中 `FIT_XY` 是 OpenGL 的默认模式，我们无需做任何操作，对于 `CENTER_CROP`/`CENTER_INSIDE` 我们则可以通过调整顶点坐标进行实现。`CENTER_CROP` 是让顶点坐标取值范围超过 `[-1, 1]`，超过部分会在被 OpenGL 裁掉；`CENTER_INSIDE` 是让顶点坐标取值范围小于 `[-1, 1]`，不足部分则没有内容，如果我们可以通过 `glClearColor` 调用设置无内容区域的颜色。

裁剪、翻转、旋转这三种操作实际上都没有调整我们绘制的图形（都是“全屏”矩形），只是调整贴纹理的方式，而缩放则不调整贴纹理的方式（“全屏”纹理），它调整的是我们绘制的图形。

细心的朋友可能会发现，`CENTER_CROP` 可以利用裁剪来实现，确实是这样。但 `CENTER_INSIDE` 却无法用裁剪的思路实现，让纹理坐标取值范围超过 `[0, 1]` 则会出现边缘像素重复的效果，如下图所示：

![](https://imgs.piasy.com/2017-10-07-opengl_preview_texture_coords_exceed-1.png)

可以看到左右两边的画面变成了条纹状。

### 关于条纹状效果

如果我们指定的纹理坐标值超过 `[0, 1]` 区间，超出部分的内容是怎么确定的？上面我们看到的都是条纹状，看起来像是边缘像素的重复，一定是这种效果吗？

其实这个问题在 OpenGL 里面叫 texture wrapping，OpenGL 规范定义了四种 wrap mode：`GL_REPEAT`、`GL_MIRRORED_REPEAT`、`GL_CLAMP_TO_EDGE`、`GL_CLAMP_TO_BORDER`。这四种 mode 的效果如下图（[摘自 open.gl](https://open.gl/textures)）：

![](https://imgs.piasy.com/2017-10-07-c3_clamping.png)

由于安卓相机输出的纹理是 OES 纹理，而在 OES 纹理中，wrap mode 我们只能使用 `GL_CLAMP_TO_EDGE`，所以相机采集的预览，就一定是边缘像素重复的效果（[参考 `OES_EGL_image_external` 文档第 3.7.14 节](https://www.khronos.org/registry/OpenGL/extensions/OES/OES_EGL_image_external.txt)）。

## Show me the code

> Talk is cheap, show me the code.

下面我将修改 [Grafika](https://github.com/google/grafika) 的预览代码，为其加上上述变换的功能，完整代码可以在[我的 GitHub 获取](https://github.com/Piasy/grafika/tree/dont_touch_fragment_coordinate/)。

Grafika 的绘制逻辑封装在 `Texture2dProgram#draw` 函数中，其 Shader 代码非常简单，已经基本被[矩形、图片、读取显存等](/2016/06/14/Open-gl-es-android-2-part-2/)涵盖了，只是绘制矩形时用了另一种接口。我们之前使用的是 `GLES20.glDrawElements(GLES20.GL_TRIANGLES,,,)`，而 Grafika 使用的是 `GLES20.glDrawArrays(GLES20.GL_TRIANGLE_STRIP,,)`。

`glDrawElements` 是给定顶点列表和绘制顺序（顶点索引列表）进行绘制，而 `glDrawArrays` 则只给定顶点列表。对于非常复杂的模型，需要很多三角形构成，顶点大量重复，而顶点坐标占用的空间比索引占用的空间大得多，因此 `glDrawElements` 空间效率会更高。

另外，`glDrawElements` 和 `glDrawArrays` 都需要一个 mode 参数，以确定处理顶点序列的方式（`glDrawArrays` 给定的顶点列表即顶点序列，`glDrawElements` 则需要把顶点索引列表中的索引替换为顶点才能得到顶点序列），OpenGL 定义了三种 mode：

+ `GL_TRIANGLES`：每三个顶点构成一个三角形，即顶点 012 构成一个三角形，顶点 345 构成一个三角形，依此类推；
+ `GL_TRIANGLE_STRIP`：任意相邻三个顶点都构成一个三角形，即顶点 012 构成一个三角形，顶点 123 构成一个三角形，依此类推；
+ `GL_TRIANGLE_FAN`：首个顶点不动，后续任意相邻两个顶点与首个顶点构成一个三角形，即顶点 012 构成一个三角形，顶点 023 构成一个三角形，依此类推；

具体可以参考 [OpenGL glDrawElements 文档](https://www.khronos.org/registry/OpenGL-Refpages/gl4/html/glDrawElements.xhtml)、[OpenGL glDrawArrays 文档](https://www.khronos.org/registry/OpenGL-Refpages/gl4/html/glDrawArrays.xhtml) 和 [OpenGL wiki 的 Triangle primitives 部分](https://www.khronos.org/opengl/wiki/Primitive#Triangle_primitives)。

现在让我们回到 Grafika 的绘制代码。

在 Grafika 中，顶点坐标、纹理坐标的获取都封装在 `Drawable2d` 中，所以我的主要修改都在 `Drawable2d` 中。另外，我在[WebRTC-Android 源码导读（二）：预览实现分析](/2017/07/26/WebRTC-Android-Render-Video/#glrectdrawer)中提到过，Grafika 的 Shader 代码并没有对顶点坐标进行变换，而是对纹理坐标进行的变换，变换的逻辑比较让人费解，所以我就去掉了这一变换逻辑（忽略 `SurfaceTexture` 的 transform matrix，使用 identity matrix）。

### 处理框架

~~~ java
private static final float FULL_RECTANGLE_COORDS[] = {
    -1.0f, -1.0f,   // 0 bottom left
    1.0f, -1.0f,    // 1 bottom right
    -1.0f,  1.0f,   // 2 top left
    1.0f,  1.0f,    // 3 top right
};
private static final float FULL_RECTANGLE_TEX_COORDS[] = {
    0.0f, 0.0f,     // 0 bottom left
    1.0f, 0.0f,     // 1 bottom right
    0.0f, 1.0f,     // 2 top left
    1.0f, 1.0f      // 3 top right
};

public void setTransformation(final Transformation transformation) {
    if (mPrefab != Prefab.FULL_RECTANGLE) {
        return;
    }

    vertices = Arrays.copyOf(FULL_RECTANGLE_COORDS, FULL_RECTANGLE_COORDS.length);
    textureCoords = new float[8];

    if (transformation.cropRect != null) {
        resolveCrop(transformation.cropRect.x, transformation.cropRect.y,
                transformation.cropRect.width, transformation.cropRect.height);
    } else {
        resolveCrop(Transformation.FULL_RECT.x, Transformation.FULL_RECT.y,
                Transformation.FULL_RECT.width, Transformation.FULL_RECT.height);
    }
    resolveFlip(transformation.flip);
    resolveRotate(transformation.rotation);
    if (transformation.inputSize != null && transformation.outputSize != null) {
        resolveScale(transformation.inputSize.width, transformation.inputSize.height,
                transformation.outputSize.width, transformation.outputSize.height,
                transformation.scaleType);
    }

    mVertexArray = GlUtil.createFloatBuffer(vertices);
    mTexCoordArray = GlUtil.createFloatBuffer(textureCoords);
}
~~~

由于这里使用了 `GL_TRIANGLE_STRIP` 的模式，所以顶点列表的顺序不能是`左下 -> 右下 -> 右上 -> 左上`，而得是 `左下 -> 右下 -> 左上 -> 右上`。如果我们用 `左下 -> 右下 -> 右上 -> 左上`，那 `012` 和 `123` 将是 `左下 -> 右下 -> 右上`和`右下 -> 右上 -> 左上`，很遗憾这两个三角形无法构成一个矩形。

此外，如上文所述，变换操作都是通过修改纹理坐标和顶点坐标实现，各个变换的顺序并不重要，比如先 flip 再 crop 也是可以的。下面我们看看每种变换的代码。

### resolveCrop

~~~ java
private void resolveCrop(float x, float y, float width, float height) {
    float minX = x;
    float minY = y;
    float maxX = minX + width;
    float maxY = minY + height;

    // left bottom
    textureCoords[0] = minX;
    textureCoords[1] = minY;
    // right bottom
    textureCoords[2] = maxX;
    textureCoords[3] = minY;
    // left top
    textureCoords[4] = minX;
    textureCoords[5] = maxY;
    // right top
    textureCoords[6] = maxX;
    textureCoords[7] = maxY;
}
~~~

可以看到，确实就是限制纹理坐标的取值范围来实现裁剪的效果。

### resolveFlip

~~~ java
private void resolveFlip(int flip) {
    switch (flip) {
        case Transformation.FLIP_HORIZONTAL:
            swap(textureCoords, 0, 2);
            swap(textureCoords, 4, 6);
            break;
        case Transformation.FLIP_VERTICAL:
            swap(textureCoords, 1, 5);
            swap(textureCoords, 3, 7);
            break;
        case Transformation.FLIP_HORIZONTAL_VERTICAL:
            swap(textureCoords, 0, 2);
            swap(textureCoords, 4, 6);

            swap(textureCoords, 1, 5);
            swap(textureCoords, 3, 7);
            break;
        case Transformation.FLIP_NONE:
        default:
            break;
    }
}
~~~

水平翻转时，`swap(0, 2)` 是交换了左下、右下两点的 x 坐标，`swap(4, 6)` 则是交换了左上、右上两点的 x 坐标，即完成了左边两点和右边两点的交换。垂直翻转同理，大家可以自己推导一下。

### resolveRotate

~~~ java
private void resolveRotate(int rotation) {
    float x, y;
    switch (rotation) {
        case Transformation.ROTATION_90:
            x = textureCoords[0];
            y = textureCoords[1];
            textureCoords[0] = textureCoords[4];
            textureCoords[1] = textureCoords[5];
            textureCoords[4] = textureCoords[6];
            textureCoords[5] = textureCoords[7];
            textureCoords[6] = textureCoords[2];
            textureCoords[7] = textureCoords[3];
            textureCoords[2] = x;
            textureCoords[3] = y;
            break;
        case Transformation.ROTATION_180:
            swap(textureCoords, 0, 6);
            swap(textureCoords, 1, 7);
            swap(textureCoords, 2, 4);
            swap(textureCoords, 3, 5);
            break;
        case Transformation.ROTATION_270:
            x = textureCoords[0];
            y = textureCoords[1];
            textureCoords[0] = textureCoords[2];
            textureCoords[1] = textureCoords[3];
            textureCoords[2] = textureCoords[6];
            textureCoords[3] = textureCoords[7];
            textureCoords[6] = textureCoords[4];
            textureCoords[7] = textureCoords[5];
            textureCoords[4] = x;
            textureCoords[5] = y;
            break;
        case Transformation.ROTATION_0:
        default:
            break;
    }
}
~~~

这里我们只分析选择 90° 的情况，剩余情况大家参照着分析。这段代码相当于是依次把左上移到左下，右上移到左上，右下移到右上，左下移到右下，是不是就相当于逆时针旋转了 90° ？

### resolveScale

~~~ java
private void resolveScale(int inputWidth, int inputHeight, int outputWidth, 
        int outputHeight, int scaleType) {
    if (scaleType == Transformation.SCALE_TYPE_FIT_XY) {
        // The default is FIT_XY
        return;
    }

    // Note: scale type need to be implemented by adjusting
    // the vertices (not textureCoords).
    if (inputWidth * outputHeight == inputHeight * outputWidth) {
        // Optional optimization: If input w/h aspect is the same as output's,
        // there is no need to adjust vertices at all.
        return;
    }

    float inputAspect = inputWidth / (float) inputHeight;
    float outputAspect = outputWidth / (float) outputHeight;

    if (scaleType == Transformation.SCALE_TYPE_CENTER_CROP) {
        if (inputAspect < outputAspect) {
            float heightRatio = outputAspect / inputAspect;
            vertices[1] *= heightRatio;
            vertices[3] *= heightRatio;
            vertices[5] *= heightRatio;
            vertices[7] *= heightRatio;
        } else {
            float widthRatio = inputAspect / outputAspect;
            vertices[0] *= widthRatio;
            vertices[2] *= widthRatio;
            vertices[4] *= widthRatio;
            vertices[6] *= widthRatio;
        }
    } else if (scaleType == Transformation.SCALE_TYPE_CENTER_INSIDE) {
        if (inputAspect < outputAspect) {
            float widthRatio = inputAspect / outputAspect;
            vertices[0] *= widthRatio;
            vertices[2] *= widthRatio;
            vertices[4] *= widthRatio;
            vertices[6] *= widthRatio;
        } else {
            float heightRatio = outputAspect / inputAspect;
            vertices[1] *= heightRatio;
            vertices[3] *= heightRatio;
            vertices[5] *= heightRatio;
            vertices[7] *= heightRatio;
        }
    }
}
~~~

如果 `inputWidth * outputHeight == inputHeight * outputWidth`，那三种缩放模式其实效果一样，所以我们无需做事。

`CENTER_CROP` 时，如果 `inputAspect < outputAspect`，即输入宽高比小于输出宽高比，则说明（竖屏时）输入宽是短边高是长边，宽填充满即顶点横坐标取值范围充满 `[-1, 1]`，高按比例放大即各顶点纵坐标值乘以放大系数，系数为 `outputAspect / inputAspect`（推理过程见下图）。`inputAspect > outputAspect` 说明高是长边宽是短边，其处理逻辑同理可得。

![](https://imgs.piasy.com/2017-10-09-opengl_center_crop_height_ratio.jpeg)

`CENTER_INSIDE` 时，ratio 的计算结果小于 1，即限制顶点坐标的取值范围，使其小于 `[-1, 1]`，其逻辑类似，大家可以自行推导。

## 总结

在本文中，我代铁蕾兄分享了 OpenGL 四种常用 2D 变换的实现思路，实现代码则基于 Grafika 项目，3D 变换相关的内容，我会在今后涉及到之后再进行总结，时间未定，不过铁蕾兄近期应该会分享 OpenGL 3D 坐标变换的内容，[大家敬请期待](http://zhangtielei.com/)！
