---
layout: post
title: 安卓 OpenGL ES 2.0 完全入门（四）：Texture 和 FrameBuffer
tags:
    - 安卓开发
    - OpenGL
---

+ `glGenFramebuffers(1, &fbo)` 创建 FBO；
+ `glBindFramebuffer(GL_FRAMEBUFFER, fbo)` 绑定、激活 FBO；
+ 添加 attachment，例如 texture，准备好 texture 之后，调用 `glFramebufferTexture2D` 把 texture attach 到 FBO 中，这样绘制的结果就会保存到 texture 中；
+ 后续所有的 read/write 操作，都会作用于该 FBO 之上；
+ `glBindFramebuffer(GL_FRAMEBUFFER, 0)` 解绑 FBO；
+ `glDeleteFramebuffers(1, &fbo)` 销毁 FBO；
