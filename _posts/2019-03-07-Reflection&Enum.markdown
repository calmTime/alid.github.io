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
1. getFields()只能获取public的字段，包括父类的。
2. getDeclaredFields()只能获取自己声明的各种字段，包括public，protected，private。
![img](/img/in-post/field/filds.jpeg)

**case**
测试类:
```java
public class FieldTest<T> {
    public boolean[][] b = {{true, true}, {false, false}};
    public String name = "test";
    public Integer integer = 23;
    public T type;
}
```
测试方法:
```java
@Test
public void field() {
    FieldTest<Object> objectFieldTest = new FieldTest<>();
    Class<?> aClass = objectFieldTest.getClass();
    Field[] fields = aClass.getFields();
    for (Field field : fields) {
        System.out.println(String.format("FieldTest：%s", field.getName()));
        System.out.println(String.format("Type：%s", field.getType().getCanonicalName()));
        System.out.println(String.format("GenericType:%s", field.getGenericType().toString()));
        System.out.println("\n");
    }
}
```
输出:
```
FieldTest：b
Type：boolean[][]
GenericType:class [[Z

FieldTest：name
Type：java.lang.String
GenericType:class java.lang.String

FieldTest：integer
Type：java.lang.Integer
GenericType:class java.lang.Integer

FieldTest：type
Type：java.lang.Object //泛型 T 类型，运行时被擦除为 Object
GenericType:T
```

> 枚举实现抽象方法获取field

```java
//枚举类
public enum ModuleEnum {
    //在线课程
    COURSE("0", "course", "courseindex"){
        @Override
        public ISearchService build() {
            return  new CourseSearch();
        }
    },
    //培训活动
    ACTIVITY("1", "activity", "activityindex") {
        @Override
        public ISearchService build() {
            return  new AlbumSearch();
        }
    };
    public abstract ISearchService build();
}
```

这里使用枚举实现简单的工厂方法。在返回前端的时候，有一个默认的枚举处理器。
```java
Class<? extends Enum> aClass = value.getClass();
for (Field field : aClass.getDeclaredFields()) {
    field.setAccessible(true);
    attr.put(field.getName(), field.get(value));
}
```

就是把枚举值取出来组成一个map返回前端

```json
"protocolSubject": {
   "code": 1,
   "info": "趣拿软件_Q013C_1"
},
```

**但是我在枚举中实现了抽象方法，对于每个枚举值来说，使用内部类实现**
导致了我直接getClass()获取到的是内部类的类类型，不能获取到枚举的详情，所以考虑获取其父类即我所需要的枚举（正常的枚举类的父类是Enum）

```java
Class<?> aClass = value.getClass();
Field[] declaredFields = aClass.getDeclaredFields();
while (aClass != Enum.class || declaredFields.length == 0) {
    for (Field field : aClass.getDeclaredFields()) {
        field.setAccessible(true);
        attr.put(field.getName(), field.get(value));
    }
    // 如果该类获取不到Fields则可能是内部类,查找其父类
    aClass = aClass.getSuperclass();
    declaredFields = aClass.getDeclaredFields();
}
```

这里如果 `declaredFields.length == 0` 则获取其`aClass.getSuperclass()` 就可以获取到我所需要的类类型。








