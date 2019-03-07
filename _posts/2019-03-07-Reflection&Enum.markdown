---
layout:     post
title:      "反射获取Field"
subtitle:   "记录一次遇到反射获取不到Field的问题"
date:       2019-03-07
author:     "ALID"
header-img: "post-bg-infinity.jpg"
catalog: true
tags:
    - case
    - 反射
    - java
    - enum
---

## 反射

#### Field
> `java.lang.reflect.Field` 为我们提供了获取当前对象的成员变量的类型，和重新设值的方法

可以做到:
1. 获取变量类型
2. 获取成员变量的修饰符
3. 获取成员变量的值和名称,修改成员变量的值

方法: