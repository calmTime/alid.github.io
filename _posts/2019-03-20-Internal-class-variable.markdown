---
layout:     post
title:      "Java的Lambda中的变量"
subtitle:   "从Stream.foreach的计数器说起"
date:       2019-03-20
author:     "ALID"
header-img: "img/home-bg-o.jpg"
catalog: true
tags:
    - case
    - 内部类
    - lambda
    - java
---

### 知识准备
1. Java中闭包的实现在没有Lambda的时候是使用匿名内部类的方式，Java8中新增的Lambda取代了部分匿名内部类，同样实现了闭包。
2. Java的匿名内部类和lambda表达式都是 **只有值捕获(capture-by-value)** 只需要在创建闭包的地方把捕获的值拷贝一份到对象里即可。与之相对的是 **有引用捕获(capture-by-reference)** 把被捕获的局部变量“提升”（hoist）到对象里。C#的匿名函数（匿名委托/lambda表达式）就是这样实现的。[[参考]](https://www.zhihu.com/question/28190927/answer/39786939)
3. 如果变量（variable）是**不可变(immutable)**的，那么使用者无法感知值捕获和引用捕获的区别。
4. Java只是不允许改变被lambda表达式捕获的变量，并没有限制这些变量所指向的对象的状态能不能变，原因是因为Java是值传递。
5. 关于Java的值传递。无论是基本类型和是引用类型，在实参传入形参时，都是值传递，**也就是说传递的都是一个副本，而不是内容本身。**对于引用变量传入的是堆中地址的副本，对于基础变量直接拷贝值传入。[[参考]](https://juejin.im/post/5bce68226fb9a05ce46a0476)
6. 关于原子类型…

### CASE
让我们来看一下问题的切入点：
```java
int i = 0;
str.forEach(s -> {
    i++; // 编译错误：Variable used in lambda expression should be final or effectively final
});
```

很明显这里idea直接飘红了，提示是 `“Variable used in lambda expression should be final or effectively final”` 意思是说：lambda表达式中使用的变量应该是final的。 

##### Q 我传入的i不是final类型的啊
这里我们来看一下反编译后的内容：

**源代码**
```
int i = 0;
str.forEach(s -> {
    System.out.println(i);
});
```
**反编译**
```java
final int i = 0;
private static /* synthetic */ void lambda$stream$0(int i, String s) {
    System.out.println((int)i);
}
```
可以看到编译后我们不是final的变量i被自动加上了final修饰。即如果lambda中引用某局部变量，则直接将其视为final。

##### Q  那如果我再之后修改i呢
我们来尝试一下
```java
int i = 0;
str.forEach(s -> {
    System.out.println(i); // 编译错误：Variable used in lambda expression should be final or effectively final

});
i++;
```

这里就会发现又出现了我们刚刚看到的错误，告诉我们i是final的。

##### Q 为什么lambda中只能使用final对象呢？

这就涉及到知识储备的第一点和第二点， 再来详细说一下：
> Java 8语言上的lambda表达式只实现了capture-by-value，也就是说它捕获的局部变量都会拷贝一份到lambda表达式的实体里，然后在lambda表达式里要变也只能变自己的那份拷贝而无法影响外部原本的变量；但是Java语言的设计者又要挂牌坊不明说自己是capture-by-value，为了以后语言能进一步扩展成支持capture-by-reference留下后路，所以现在干脆不允许向捕获的变量赋值，而且可以捕获的也只有“效果上不可变”（effectively final）的参数/局部变量。

这就解释了为什么lambda(就是闭包)中为什么必须使用final对象了。

##### Q 如果我希望在lambda中修改值呢？
> Idea给出了两种修改意见

1.  修改变量所指向的对象的状态
```java
final int[] i = {0};
str.forEach(s -> {
    i[0]++;
});
```
就是使用知识储备中第三点的方法，我不能直接修改变量的状态，但是我可以通过修改**变量所指向的对象的状态**来实现变量的修改和外层感知。

这种做法可以叫做“手动boxing”，那个长度为1的数组其实就是个Box。

P.s.JDK内部自己都有些代码这么做嗯…

2. 使用原子类型
```java
AtomicInteger i = new AtomicInteger();
str.forEach(s -> {
    i.getAndIncrement();
});
```
原子类型的实现方法中是这样的：`private volatile int value;`
我们可以看到原子类型的值是加了`volatile`关键词的…

##### Q 为什么只有局部变量有限制而属性是无限制的

 **源代码**
```java
public class Lambda {
    private int in = 0;
    public void test() {
        Test test = new Test() {
            @Override
            public void fx() {
                in = in + 1;
            }
        };
    }
}

```
**反编译**
```java
public class Lambda {
    private int in = 0;
    public void test() {
         Test test = new Test() {
            public void fx() {
                ((Lambda)Lambda.this).in = (int)(((Lambda)Lambda.this).in + 1);
            }
        };
    }
}
```
可以发现变压器帮我们把属性in变成的`Lambda.this.in` 
…

### 进一步思考
为什么Java并没有限制这些变量所指向的对象的状态能不能变，留出了长度为1的数组这种方法呢？

我个人认为有两个原因第一点是因为Java是值传递的，传入的是该数组的堆中地址，我们**基于此地址在方法中(lambda中)修改该地址的内容**会影响**同样指向改地址的数组的内容**。

第二点是，我希望在流中修改传入`List<Model>`中每一个Model的值如果不允许我修改，那么lambda的使用范围将受到很大限制。





