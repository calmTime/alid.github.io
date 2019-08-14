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
    - lambda
    - java
---

### CASE
> 匿名内部类等效于Lambda, 在本文中2者都有提到, 可以当做同一个概念理解

让我们来看一下问题的切入点：
```java
int i = 0;
str.forEach(s -> {
    i++; // 编译错误：Variable used in lambda expression should be final or effectively final
});
```

很明显这里idea直接飘红了，提示是 `“Variable used in lambda expression should be final or effectively final”` 意思是说：lambda表达式中使用的变量应该是final的。 

#### Q 我传入的i不是final类型的啊
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

#### Q  那如果我再之后修改i呢
我们来尝试一下
```java
int i = 0;
str.forEach(s -> {
    System.out.println(i); // 编译错误：Variable used in lambda expression should be final or effectively final
});
i++;
```

这里就会发现又出现了我们刚刚看到的错误，告诉我们i是final的。

#### Q 为什么lambda中只能使用final对象呢？

这里首先对比内部类使用的对象和直接调用一个方法传入参数的区别

```java
private void test(){
    int i = 0;
    // lambda 使用局部变量i
    str.forEach(s -> {
        System.out.println(i);
    });
    // 方法使用局部标量i
    method(i);
}

private void method(int i){
    System.out.println(i);
}
```

这里可以看到在正常方法调用的时候是传了参数`i`进入方法的, 而lambda中使用的参数`i`是可以看到没有传进去的. 

这就带来了一个问题, 因为内部类和外部类其实是处于同一个级别，内部类不会因为定义在方法中就会随着方法的执行完毕而跟随者被销毁。当外部方法执行完成, **这时局部变量`i`已经被GC了**, 而内部类还没有开始执行. 那么之后就拿不到局部变量`i`了.

如果定义为final，java会将这个变量复制一份作为成员变量内置于内部类中，这样的话，由于final所修饰的值始终无法改变，所以这个变量所指向的内存区域就不会变。

```java
public class External {
    private Internal inter;

    public static void main(String[] args) {
        External external = new External();
        // 先运行外部方法
        external.test();
        // 再外部方法执行完成后再调用内部类
        external.inter.print();
    }

    private void test() {
        int i = 200;
        System.out.println(System.identityHashCode(i));
        inter = new Internal() {
            @Override
            void print() {
                System.out.println(System.identityHashCode(i));
            }
        };
    }

    interface Internal {
        void print();
    }
}
```

在上面的例子中就可以很好的看出来, 在外部方法执行完再调用内部类. 这时如果i不是`final`的, 那么i就会被GC掉, 不可能拿到i的值了.

并且这里打印了两次`i`的地址, 得到的结果是不一样的. 也印证了Java是值传递的, 对应值传递可以[[参考]](https://juejin.im/post/5bce68226fb9a05ce46a0476)

#### Q 为什么只有局部变量有限制而属性是无限制的

很简单属性都是绑定在类上的, 不会因为方法生命周期结算而被GC. 不影响内部类中引用.

#### Q 如果我希望在lambda中修改值呢？
> Idea给出了两种修改意见

1. 修改变量所指向的对象的状态
```java
final int[] i = {0};
str.forEach(s -> {
    i[0]++;
});
```

Java只是不允许改变被lambda表达式捕获的变量，并没有限制这些变量所指向的对象的状态能不能变，原因是因为Java是值传递。

所以就可以通过修改**变量所指向的对象的状态**来实现变量的修改和外层感知。

关于Java的值传递。无论是基本类型和是引用类型，在实参传入形参时，都是值传递，**也就是说传递的都是一个副本，而不是内容本身** 值传递对于`基础变量`直接**拷贝值传入方法的虚拟机栈中**; 而对于这里的数组就是`引用变量`，传入的是**堆中地址的副本**，指向的是同一块内存，我们可以修改该内存地址对应的值，让外部的引用变量的状态改变，因为**虽然内存地址的拷贝并值传递的，但是指向的内存都一样**，被修改的内容也肯定一样了。[[参考]](https://juejin.im/post/5bce68226fb9a05ce46a0476)

这种做法可以叫做“手动boxing”，那个长度为1的数组其实就是个Box。

P.s.JDK内部自己都有些代码这么做的…

2. 使用原子类型
```java
AtomicInteger i = new AtomicInteger();
str.forEach(s -> {
    i.getAndIncrement();
});
```
其实很简单原子类型的值(Value)是`AtomicInteger`的属性, 这就和传入一个Model修改其属性一样。原因也和刚刚的引用变量的值传递是一样的.

原子类型是通过volite修饰配合CAS实现的高并发下的同步需求.[[参考]](https://juejin.im/post/5a73cbbff265da4e807783f5)






