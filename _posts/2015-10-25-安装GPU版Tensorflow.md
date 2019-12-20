---
title: 安装GPU版Tensorflow
layout: post
date: 2016-10-25 10:47:35
tags:
 - Tensorflow
 - 深度学习
---
搞了台HP Gen8 MicroServer，150W的功率，双核CPU，配了8GB内存，就开始跑深度学习模型。随便写了个程序，训练一次epoch要一个半小时，整个心都凉了！最近都在淘宝边逛显卡边流口水，买是肯定买不起了。

三年前买了台笔记本，当时是为了打游戏，配了个GeForce 650M，到NVIDIA管网查了一下，Computation Capability刚好等于3，达到了Tensorflow的最低标准，泪奔！

<!-- more -->

跟着管网的教程先到[CUDA](https://developer.nvidia.com/cuda-downloads)下载安装CUDA，再去[CUDNN](https://developer.nvidia.com/cudnn)下载安装CUDNN，这个是要注册以后才能下载的。整个安装都很顺利，没有任何问题。这两个都装好以后配置一下环境变量：
```
export PATH=$PATH:/usr/local/cuda
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/cuda/lib64
```
就可以安装GPU版的Tensorflow了！一开始我安装完就直接使用了，结果报错，各种驱动没装好，找不到合适的显卡之类的错误，我还捣鼓了好一会儿，其实只要重启一下就可以了。

我的显卡的显存是2GB的，略有点小，加上笔记本散热不好，感觉用来跑深度学习还是有点捉急T T。

-----------
**2016-10-25更新**
今天下班把正在跑的模型放到笔记本上用显卡跑，发现散热问题没有预想的严重，并不烫，比之前玩某些烧显卡的游戏强多了。原来用双核CPU跑需要80分钟/epoch的，现在只需要20分钟了，快了4倍！不过2GB的显存确实有点寒酸，batch size只能用区区的32，64就OOM了。validation也做不了，正在思考解决办法～
