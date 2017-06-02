---
layout: post
title: 在安卓平台上应用深度学习（一）：MNIST 手写数字识别
tags:
    - 安卓开发
    - 深度学习
---

''' 还是先总结一篇深度学习的博文比较合适，先有个铺垫

自从去年 AlphaGo 战胜了李世乭，人工智能开始全方位火爆起来。学术界和工业界的大 V 与媒体们都在强调同一个观点：AI 是行业未来，是下一个风口。在这样的高呼声中，直觉告诉我该有所行动了。适逢 Udacity 推出了一个深度学习的纳米微学位课程，社区内也有好些朋友都报名参加了，如此良机怎能错过？

参加这个项目的初衷，主要是作为门外汉，面临网上海量的信息、教程，完全不知如何下手。Udacity 的课程团队经过精挑细选、仔细编排之后的课程安排，还是很值的。上周这个课程终于完成，提交最后一个项目之后，也顺利拿到了证书，现在回过头来看这将近四个月的学习，收获还是很大的，虽然不能立即投身 AI 行业中去，但也算是领进了门，知道接下来该怎么逐步学习了。

首先澄清几个概念：人工智能，机器学习，深度学习。

+ 人工智能（AI）：这就是我们听得最多的词了，我的理解是，AI 就是要让程序表现得像具备只能一样，能够解决一些特定的问题；
+ 机器学习（ML）：机器学习是实现人工智能的一种方式；
+ 深度学习（DL）：深度学习是机器学习的一个子集，指的是具备某种特征（“深”）的模型/方法；

'''

接下来，我将分享一系列在安卓平台上应用深度学习的例子，今天先从深度学习领域的 hello world 开始：MNIST 手写数字识别。项目完整的代码请参见 [GitHub Piasy/mnist-android-tensorflow](https://github.com/Piasy/mnist-android-tensorflow)。

在安卓（或者 iOS）平台上应用深度学习主要分为两步：准备模型，使用模型。其中只有第二步发生在安卓平台中，模型的准备（训练）需要很大的计算量，需要在 PC 甚至服务器集群上才能完成。

## 准备模型

在准备模型阶段，我们需要准备数据、定义模型、训练模型，最后导出模型，以便在安卓上使用。这里我们使用 Google 开源的深度学习库：[TensorFlow](https://www.tensorflow.org/)。TensorFlow 是深度学习领域最流行、使用最广泛的工具，既有 Google 官方团队的开发，又有广大社区的支持，是开发者们的不二之选。

这里我们需要一些 Python、深度学习、TensorFlow 相关的知识，对此还不熟悉的朋友，建议看看 [TensorFlow 官网的教程](https://www.tensorflow.org/get_started/)，非常详细。下文中我基本不会对基本概念进行解释，对代码/概念有疑惑的朋友，请自行查阅相关文档。

MNIST 的数据准备工作 TensorFlow 已经帮我们完成了，直接代码导入即可：

~~~ python
# Python 3.6.0
# tensorflow 1.1.0

import os
import os.path as path

import tensorflow as tf
from tensorflow.python.tools import freeze_graph
from tensorflow.python.tools import optimize_for_inference_lib

from tensorflow.examples.tutorials.mnist import input_data

MODEL_NAME = 'mnist_convnet'
NUM_STEPS = 3000
BATCH_SIZE = 16

mnist = input_data.read_data_sets("MNIST_data/", one_hot=True)
~~~

接下来就是定义模型了，首先我们定义模型的输入：

~~~ python
def model_input(input_node_name, keep_prob_node_name):
    x = tf.placeholder(tf.float32, shape=[None, 28*28], name=input_node_name)
    keep_prob = tf.placeholder(tf.float32, name=keep_prob_node_name)
    y_ = tf.placeholder(tf.float32, shape=[None, 10])
    return x, keep_prob, y_
~~~

然后是定义模型：

~~~ python
def build_model(x, keep_prob, y_, output_node_name):
    x_image = tf.reshape(x, [-1, 28, 28, 1])
    # 28*28*1

    conv1 = tf.layers.conv2d(x_image, 64, 3, 1, 'same', activation=tf.nn.relu)
    # 28*28*64
    pool1 = tf.layers.max_pooling2d(conv1, 2, 2, 'same')
    # 14*14*64

    conv2 = tf.layers.conv2d(pool1, 128, 3, 1, 'same', activation=tf.nn.relu)
    # 14*14*128
    pool2 = tf.layers.max_pooling2d(conv2, 2, 2, 'same')
    # 7*7*128

    conv3 = tf.layers.conv2d(pool2, 256, 3, 1, 'same', activation=tf.nn.relu)
    # 7*7*256
    pool3 = tf.layers.max_pooling2d(conv3, 2, 2, 'same')
    # 4*4*256

    flatten = tf.reshape(pool3, [-1, 4*4*256])
    fc = tf.layers.dense(flatten, 1024, activation=tf.nn.relu)
    dropout = tf.nn.dropout(fc, keep_prob)
    logits = tf.layers.dense(dropout, 10)
    outputs = tf.nn.softmax(logits, name=output_node_name)

    # loss
    loss = tf.reduce_mean(
        tf.nn.softmax_cross_entropy_with_logits(labels=y_, logits=logits))

    # train step
    train_step = tf.train.AdamOptimizer(1e-4).minimize(loss)

    # accuracy
    correct_prediction = tf.equal(tf.argmax(outputs, 1), tf.argmax(y_, 1))
    accuracy = tf.reduce_mean(tf.cast(correct_prediction, tf.float32))

    tf.summary.scalar("loss", loss)
    tf.summary.scalar("accuracy", accuracy)
    merged_summary_op = tf.summary.merge_all()

    return train_step, loss, accuracy, merged_summary_op
~~~

这里我们定义了一个 CNN 模型，使用的也都是 TensorFlow 封装程度较高的 layers API。但对初学的朋友我有一个建议，先熟悉基础的 API，搞清楚了基本原理之后，再使用封装好的 API，基础知识要是不打牢，后面很容易犯迷糊。

接下来就是训练模型了：

~~~ python
def train(x, keep_prob, y_, train_step, loss, accuracy,
        merged_summary_op, saver):
    print("training start...")

    init_op = tf.global_variables_initializer()

    with tf.Session() as sess:
        sess.run(init_op)

        tf.train.write_graph(sess.graph_def, 'out',
            MODEL_NAME + '.graph.bin', False)

        # op to write logs to Tensorboard
        summary_writer = tf.summary.FileWriter('logs/',
            graph=tf.get_default_graph())

        for step in range(NUM_STEPS):
            batch = mnist.train.next_batch(BATCH_SIZE)
            if step % 100 == 0:
                train_accuracy = accuracy.eval(feed_dict={
                    x: batch[0], y_: batch[1], keep_prob: 1.0})
                print('step %d, training accuracy %f' % (step, train_accuracy))
            _, summary = sess.run([train_step, merged_summary_op],
                feed_dict={x: batch[0], y_: batch[1], keep_prob: 0.5})
            summary_writer.add_summary(summary, step)

        saver.save(sess, 'out/' + MODEL_NAME + '.ckpt')

        test_accuracy = accuracy.eval(feed_dict={x: mnist.test.images,
                                    y_: mnist.test.labels,
                                    keep_prob: 1.0})
        print('test accuracy %g' % test_accuracy)

    print("training finished!")
~~~

我们在训练的过程中保存了三个东西：模型结构图（`tf.train.write_graph`），训练过程的中间数据（`summary_writer.add_summary`），训练完成之后的各个变量值（`saver.save`）。中间数据我们可以利用 TensorBoard 工具观察/分析训练过程，而模型结构图和变量值则用于接下来的模型导出：

~~~ python
def export_model(input_node_names, output_node_name):
    freeze_graph.freeze_graph('out/' + MODEL_NAME + '.graph.bin', None, True,
        'out/' + MODEL_NAME + '.ckpt', output_node_name, "save/restore_all",
        "save/Const:0", 'out/frozen_' + MODEL_NAME + '.pb', True, "")

    input_graph_def = tf.GraphDef()
    with tf.gfile.Open('out/frozen_' + MODEL_NAME + '.pb', "rb") as f:
        input_graph_def.ParseFromString(f.read())

    output_graph_def = optimize_for_inference_lib.optimize_for_inference(
            input_graph_def, input_node_names, [output_node_name],
            tf.float32.as_datatype_enum)

    with tf.gfile.FastGFile('out/opt_' + MODEL_NAME + '.pb', "wb") as f:
        f.write(output_graph_def.SerializeToString())

    print("graph saved!")
~~~

我们利用 `freeze_graph` 和 `optimize_for_inference_lib` 把 graph 和 checkpoint 处理成可以单独使用的模型文件，这个处理的主要目的是优化模型，去掉一些训练过程才需要的东西，提高模型使用时的运行效率。`input_node_names` 和 `output_node_name` 就是我们后面在 Java 代码里面将要使用到的输入输出 Tensor 名。

最后就是我们的 main 函数了：

~~~ python
def main():
    if not path.exists('out'):
        os.mkdir('out')

    input_node_name = 'input'
    keep_prob_node_name = 'keep_prob'
    output_node_name = 'output'

    x, keep_prob, y_ = model_input(input_node_name, keep_prob_node_name)

    train_step, loss, accuracy, merged_summary_op = build_model(x, keep_prob,
        y_, output_node_name)
    saver = tf.train.Saver()

    train(x, keep_prob, y_, train_step, loss, accuracy,
        merged_summary_op, saver)

    export_model([input_node_name, keep_prob_node_name], output_node_name)

if __name__ == '__main__':
    main()
~~~

上面的 python 代码执行完毕后，会在 out 目录下生成 `opt_mnist_convnet.pb` 文件，这个文件就是我们将要在安卓平台中使用的模型文件了。

## 使用模型

在安卓平台上使用 TensorFlow 导出的模型，我们需要引入 TensorFlow 依赖，之前这个库并没有发布到 bintray，为此我还搞了一个 double-tf-android 项目，不过现在不需要啦，直接使用官方发布的库即可：

~~~ gradle
compile 'org.tensorflow:tensorflow-android:1.2.0-rc0'
~~~

我们主要使用的是 `TensorFlowInferenceInterface` 这个类，用它来加载模型、输入数据、执行推断（inference）、取出结果等。

### 加载模型

我们需要把之前导出的模型文件放到项目的 assets 目录中，供运行时加载模型：

~~~ java
TensorFlowInferenceInterface tfHelper = new TensorFlowInferenceInterface(assetManager, "opt_mnist_convnet.pb");
~~~

### 输入数据，执行推断，取出结果

我们在训练时使用的数据都是 28*28 的图像，而且只有单一通道，因此我们运行模型时也要用同样的数据作为模型的输入，如何编写一个“手写板”类，以及把手写的数据转换为 `float[]` 这里就不展开了，感兴趣的朋友可以[查看源码](https://github.com/Piasy/mnist-android-tensorflow/blob/master/MnistAndroid/app/src/main/java/mariannelinhares/mnistandroid/views/)。

另外，输入数据除了图像数据，还有一个 dropout 层的 `keep_prob` 值，这里我们需要模型发挥全部实力，就不需要 dropout 了，所以传 1 即可。

~~~ java
// 1, 28, 28, 1 是输入 tensor 的各个维度
// batch_size = 1，width = height = 28，channels = 1
tfHelper.feed("input", pixels, 1, 28, 28, 1);
tfHelper.feed("keep_prob", new float[] {1.0f});

tfHelper.run(new String[]{ "output" });

float[] output = new float[10];
tfHelper.fetch("output", output);
~~~

执行完之后，`output` 数组里就是模型认为输入数据属于各个输出分类的概率了，选出最大的作为模型的输出即可。

## 导出模型优化：去掉 dropout 层

上面我们在使用模型时，还固定输入了一个 `keep_prob = 1.0`，要是能把这个输入去掉，一方面能让代码更简洁（没必要的代码，就应该去掉），另一方面也能加快模型运行速度。这个问题在 [GitHub 上有 issue](https://github.com/tensorflow/tensorflow/issues/5867)，遗憾的是，[issue 评论中的办法](https://github.com/tensorflow/tensorflow/issues/5867#issuecomment-282954429)我没有使用成功，他的代码应该是依赖于模型定义的，没法像移除 batch normalization 那样移除 dropout。

## 导出 Keras 训练的模型

Keras 对 TensorFlow 的代码做了进一步的封装，极大地简化了我们编写模型的代码，如何导出用 Keras 编写的模型？
