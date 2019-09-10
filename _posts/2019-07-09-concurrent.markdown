---
layout:     post
title:      "并发编程"
subtitle:   "让我们的代码拒绝等待"
date:       2019-05-09 12:00:00
author:     "ALID"
header-img: "img/post-wedding.jpg"
catalog: false
tags:
    - java
    - 并发
    - case
---

# 提纲
1. 基础并行 future.get()
2. 为什么要等待? listenableFuture
3. case 异步缓存

## BASE

并发我们需要什么?
1. 开一个线程异步 -> 线程池
2. 拿到返回结果 -> Future

### Future


1. cancle    可以停止任务的执行 但不一定成功 看返回值true or  false
2. get       阻塞获取callable的任务结果,即get阻塞住调用线程，直至计算完成返回结果
3. isCancelled  是否取消成功
4. isDone    是否完成

为什么要`Future`, 因为业务代码基本上都是要返回值的, `Future`就可以使我们很好的拿到返回值




