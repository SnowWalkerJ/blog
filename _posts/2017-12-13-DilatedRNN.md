---
layout: post
title: DilatedRNN
date: 2017-12-13 16:31:47
tags:
 - 深度学习
layout: post
---


在[WaveNet](https://arxiv.org/abs/1609.03499)中使用了Dilated Convolution，通过调节卷积核中的空洞的比例（dilation），这种网络可以在不使用pooling和stride的前提下对不同尺度的信息进行处理。

<!-- more -->

传统的CNN通过pooling和stride来把输入的图像缩小，从而相同大小的卷积核可以覆盖更大的区域，使其拥有更大的感受野。但是这两种方法在缩小图像的同时也会损失信息；另外，有迹象表明*对抗样本攻击*和pooling与stride有关。

相比之下，dilated convolution不会缩小图像大小，而是通过在卷积核中设置“空洞”来让卷积核的尺寸变大同时控制参数数量。空洞越大的卷积核能覆盖更大的空间从而具有更大的感受野。

当然，Dilated Convolution不是本篇的重点，有兴趣的看官直接移步[WaveNet: A Generative Model for Raw Audio](https://arxiv.org/abs/1609.03499)和[Multi-Scale Context Aggregation by Dilated Convolutions](https://arxiv.org/abs/1511.07122)这两篇论文应该就能明白其中的机制了。

本篇要讲的是Dilated RNN. 前段时间我在应用LSTM做量化模型的时候发现LSTM实际能记忆的时间尺度不够长。考虑到模型的LSTM的每天更新状态的，早些时候的信息可能很快就被后面的信息覆盖了，于是由Dilated Covolution启发，我搞了一个Dilated LSTM，通过在时间尺度上设置不同大小的“空洞”，可以让LSTM学习不同周期上的信号。简单地说，这个Dilation就是设置每过多少天更新一次LSTM的状态，更新频率高的模块就容易记住短时间内快速变化的信息；更新慢的模块就更容易记住较远时间的内容。

巧的是，我写了这个Dialated LSTM没几天，就在arXiv上看到[Dilated Recurrent Neural Networks](https://arxiv.org/abs/1710.02224)（还上*NIPS 2017*了），思路和我的一模一样。在这篇论文中仿照*WaveNet*的结构，把RNN层由高频（dilation小）到低频（dilation大）堆叠。文章中没有解释这么做的原因，一个直观的解释是这样做可以把不同时期的局部信息在高层低频的模块中整合起来，就和在CNN中一样。但是这么做并不是唯一可行的。由低频至高频的堆叠也可以很合理：先在低频模块中提取长时间的记忆，再在高频模块中建模这些信息的变化。

我在[Gist](https://gist.github.com/SnowWalkerJ/2c0137f914a44e1bbe5b9a158ab2ce44)上放了我自己使用的DilatedLSTMCell版本，内部管理隐藏状态而不是暴露出来，会方便一些。用Cell形式的而不是Layer形式主要是为了灵活。通过控制参数`output`的值为`new`、`concat`还是`stack`可以灵活选择是只输出最新的那个值（如论文中那样）；还是把最近N天的输出都在最后一个维度连接起来（作为不同的feature map）；还是新加一个维度堆叠起来（方便求均值或者应用注意力机制等）。

