---
layout:     post
title:      Android Neural Networks框架分析及应用
subtitle:   ANN API
date:       2020-06-20
author:     WMY
header-img: img/2020-01-26_114950.png
catalog: 	 true
tags: 
    - 工作
---

### Deep Learning 简介

近年来，以人工神经网络为核心技术的深度学习取得了突破性进展，应用于图片分类、目标检测、语义分割、自然语言处理等领域的各种算法不断涌现。
深度学习是学习样本数据的内在规律和表示层次，它的最终目标是让机器能够像人一样能够识别文字、图像和声音等数据。
下图是一个3层的神经网络，其中：每一个⭕代表一个神经元，也可以叫做神经节点、神经结点，每个神经元有一个偏置bias。每一条线，有一个权重weight。神经网络学习的目标：就是通过减少损失loss或cost，来确定权重和偏置。
理论上，如果有足够多的隐藏层和足够大的训练集，则可以模拟出任何方程


![](https://wwmmyy2023.github.io/img/ANNAPI/pic0.png)

![](https://wwmmyy2023.github.io/img/ANNAPI/pic01.png)


激活函数：通过激活函数会将数据压缩到一定的范围区间内，得到的数据的大小将决定该神经元是否处于活跃状态，即：是否被激活。这使得神经网络可以更好地解决较为复杂的问题
常见的激励函数：sigmoid函数、tanh函数、ReLu函数、SoftMax函数等等。


![](https://wwmmyy2023.github.io/img/ANNAPI/pic2.png)


### Android NNAPI

深度学习的应用的一大特点是计算量极大，采用终端上的 GPU、DSP、ASIC等硬件设备计算的需求愈发广泛。另一方面，用于开发应用的上层框架也不断丰富。

丰富的上层框架应用和底层硬件设备带来了一个问题——每一种上层框架都要尝试适配主流的硬件设备，每一种底层硬件设备上都要适配主流的框架应用。这种工程任务是如此的繁重，因而对产业中所有的角色都是不利的。

针对该问题，多种中间层被提出。这样的中间层可以支持多个应用框架和多个设备。框架和设备都只需要对接到这样的中间层，大大降低了开发成本。

中间层的代表为： Android Neural Networks API。
总体而言，Android Neural Networks API 简洁有效，符合软件系统的设计方法学。

当前上层框架代表 TensorFlow 几乎成为了事实标准，TensorFlow 模型实际上已成为了这种「中间型」模型交换格式。
Google 颇有携 Android 和 TensorFlow 两大神器号令「移动+人工智能」的趋势。

#### Android NNAPI 的软件架构-架构图


Android Neural Networks API是和底层设备高度相关的一套系统，其接口包括两部分：高层用户侧的 NDK 接口（图中 Android NN API）、和底层硬件设备侧的 HAL 接口（图中 Android NN HAL）。

![](https://wwmmyy2023.github.io/img/ANNAPI/pic1.png)


#### Android NNAPI 编程模型

对于任何机器学习系统，我们都可以将其任务分为三个步骤：
1.从用户那里获得计算模型的定义
2.对计算模型施以某些优化，使得计算能更快地进行
3.在硬件设备上执行计算模型

![](https://wwmmyy2023.github.io/img/ANNAPI/pic3.png)

![](https://wwmmyy2023.github.io/img/ANNAPI/pic4.png)


#### Android NNAPI 模块结构

源码路径：frameworks/ml/nn/

![](https://wwmmyy2023.github.io/img/ANNAPI/pic5.png)


#### Android NNAPI 具体运行过程

![](https://wwmmyy2023.github.io/img/ANNAPI/pic6.png)

框架和驱动程序之间的接口遵循的一般流程：

![](https://wwmmyy2023.github.io/img/ANNAPI/pic7.png)


#### Qualcomm DSP HIDL驱动层

源码路径： android/platform/hardware/qcom/neuralnetworks/hvxservice/master/./1.0

HIDL 驱动的实现
HIDL 驱动的工作主要是将 Android Neural Networks API 表示的模型、数据转换成 DSP 的表示，并使用 DSP 提供的接口来运行计算任务。

HIDL驱动是中间层，这里又有两层接口：DSP 驱动的接口和 HIDL 驱动的接口。

DSP 驱动定义了又一套神经网络模型的接口，包括算子、张量、性能等数据结构，模型的准备、运行等接口。由于接口已经比较底层，算子的定义综合了精度等信息。

HIDL 驱动在 HexagonController 中将 DSP 驱动的 C 接口封装成 C++ 形式。底层的服务由 libhexagon_nn_controller.so 提供。然后通过该so会调用到 Hexagon NN lib 里面的具体算子实现方法。

获取每个运行算子最佳的硬件设备：

![](https://wwmmyy2023.github.io/img/ANNAPI/pic8.png)


获取所有的底层硬件驱动设备：

![](https://wwmmyy2023.github.io/img/ANNAPI/pic9.png)

上层接口调用底层DSP算子执行调用关系图：

![](https://wwmmyy2023.github.io/img/ANNAPI/pic10.png)

Qualcomm Hexagon NN 自定义算子：

![](https://wwmmyy2023.github.io/img/ANNAPI/pic11.png)

### Deep Learning 应用案例：

目标：采用深度学习神经网络，建立手机内部硬件温度与手机外壳壳温关系模型，通过手机内部各硬件实时温度，预测当前手机表面用户可感知温度。并将训练好的模型移植到手机系统中，进行实际应用。

深度学习框架：采用当前主流的TensorFlow2.0深度学习框架，建立模型，手机端采用Tensor Flow Lite。
Tensor Flow Lite 是Google推出的专门针对移动设备上可运行的深度网络模型简单版。
Tensor Flow Lite 使用Android NN API，可在没有加速硬件时直接调用CPU 来处理，确保其可以兼容不同设备。

1，准备数据       --> 2，定义模型 --> 3，训练模型 --> 4，评估模型


![](https://wwmmyy2023.github.io/img/ANNAPI/pic12.png)


![](https://wwmmyy2023.github.io/img/ANNAPI/pic13.png)


5，使用模型： 移植到android手机系统 使用运行，将训练好的模型转为“.rfile”格式，在Android端调用该模型，测试实际预测结果：


![](https://wwmmyy2023.github.io/img/ANNAPI/pic14.png)


![](https://wwmmyy2023.github.io/img/ANNAPI/pic15.png)

![](https://wwmmyy2023.github.io/img/ANNAPI/pic16.png)



参考文献：https://zhenhuaw.me/blog/2018/on-android-nnapi.html