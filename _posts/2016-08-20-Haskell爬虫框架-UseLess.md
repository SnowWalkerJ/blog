---
title: Haskell爬虫框架 - UseLess
date: 2016-08-20 16:36:21
tags:
 - Haskell
 - 编程
layout: post
---

用Haskell写了个爬虫框架，支持open-closed table任务调度，目前还是单线程的，包含HTML解析和标签筛选。用起来还挺方便的。跑了一下CPU占用率有点高，可能是HTML解析的效率太低。目前的做法是把整个HTML解析成标签树，但其实有点浪费，因为很多分支都不会被用到。考虑可以修改成惰性解析，当需要检索某个标签的时候才解析相应的标签。这里有待改进。

源码在[Github](https://github.com/SnowWalkerJ/UseLess/)上，也写了相应的[Wiki](https://github.com/SnowWalkerJ/UseLess/wiki)

<!-- more -->