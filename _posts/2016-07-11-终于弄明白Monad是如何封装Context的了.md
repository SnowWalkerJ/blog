---
title: 终于弄明白Monad是如何封装Context的了
date: 2016-07-11 01:11:49
tags:
  - Haskell
  - 编程
layout: post
---
Monad是函数式编程中一个饱受争议的特性。一方面，它确实非常强大，给了函数式编程很多灵活性，也允许像Haskell这样的语言拥有封装副作用的机制；另一方面，它的古怪性质导致很多学习者始终抓不住它的精髓。

Monad配合do语句可以让Haskell代码像命令式编程一样具有顺序结构。然而最牛逼的还是它封装Context的能力，就好像IO、STRef和Random那样。但是我始终没有明白它是怎么做到的，查了很多文档也没有把这个问题搞明白。刚才失眠突然想到了：原来封装的机制比想象的复杂，do语句使这个抽象的过程看起来很简单！

<!-- more -->

大家都知道Monad其实是个容器，就好像List Monad就包含了一堆元素。但其实像IO和STRef这样的Monad，其元素并不是真的数据，而是函数。当我们使用do语句的时候并没有真正进行计算，只是把一个又一个函数链接起来，然后用Monad包装起来。对于STRef来说，最后需要用runST来调用这个函数计算出真正的结果;对于Random来说则要调用runRandom。

下面用了很短的几句代码来模拟一个Random类型：
```haskell
module Random where
    data Random a = Random (Double->(Double, a))
    instance Monad Random where
        return x = Random (\s->(s, x))
        Random f0 >>= f = Random f' 
            where f' x = (x, runRandom seed' $ f y) 
                     where (seed', y) = f0 x
    getRandom::Random Double
    getRandom = Random (\x->(cos x, sin x))
    runRandom::Double->Random a->a
    runRandom seed (Random f) = snd $ f seed
    initialSeed = 0.1
    main = print $ runRandom initialSeed $ do
        x <- getRandom
        y <- getRandom
        return $ (x, y)
```

其中`getRandom`函数利用正弦、余弦函数来计算随机数和下一个随机数种子，简化了真实的随机数发生器机制。

这样一来，就可以把函数包裹在Monad内部，随机数种子作为全局参数，由runRandom传入，避免了在中间函数传递seed参数，起到了封装的效果。
