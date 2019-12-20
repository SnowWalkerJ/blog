---
title: 模仿tensorflow实现符号运算及自动求导
tags:
 - Python
 - 编程
 - 深度学习
layout: post
date: 2017-09-30
---
Tensorflow用了好久了，它的符号运算和自动求导功能都非常有意思，实现起来好像也不难，只是很简单的链式法则而已。终于有空下来稍微搞了一下。相关代码在[github](https://github.com/SnowWalkerJ/fake_ft)上。

<!-- more -->

代码并没有特别难的或者tricky的部分。我觉得受到Haskell的影响非常大。Tensorflow使用的符号计算和Haskell的惰性求值如出一辙。不要直接计算x+y的结果，而是定义一种加运算AddOp储存两个操作数，直到eval的时候才真正去计算。

自动求导也不算太难，为每个操作Op定义其导数（也是一个Op），再利用链式法则把顶层的导数往底层传，最终构造出目标函数关于变量的骗导。

```Python
from faketf import reduce_mean, Placeholder
from faketf.highend.layers import fully_connected
from faketf.train import SGD
import numpy as np

x = Placeholder([None, 5], name="x")
y_real = Placeholder([None, 1], name="real")
y = fully_connected(x, 1)
loss = reduce_mean((y - y_real)**2)  # for now power is not realized yet
trainer = SGD(loss, learning_rate=0.01)

for i in range(3000):
    x_values = np.random.randn(1000, 5)
    true_weights = np.array([[1.0], [0.5], [-0.5], [2.0], [-0.3]])
    noise = np.random.randn(1000, 1) * 0.3
    y_values = x_values @ true_weights + noise
    feed_dict = {x: x_values, y_real: y_values}
    trainer.train(feed_dict)
    if i % 100 == 0:
        trainer.learning_rate *= 0.9
        print("Loss: %0.4f" % np.asscalar(loss.eval(feed_dict)))
```