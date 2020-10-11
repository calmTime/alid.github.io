---
layout:     post
title:      "spring bean的加载过程"
subtitle:   "spring是如何帮我们管理bean的"
date:       2020-05-02 12:00:00
author:     "ALID"
header-img: "img/psot-bg-piano.jpg"
catalog: true
tags:
    - java
    - 源码
    - spring
---

在bean被加载之前, 需要创建spring上下文, 之后才能加载bean. 创建上下文的过程就是创建ICO容器的部分.

如果细分的话, 可以分为三步
1. 创建IOC容器
2. 从注解和xml, 解析并注册bean
3. 开始创建bean

对于BeanFactory来说，对象实例化默认采用延迟初始化。通常情况下，当对象A被请求而需要第一次实例化的时候，如果它所依赖的对象B之前同样没有被实例化，那么容器会先实例化对象A所依赖的对象。这时容器内部就会首先实例化对象B，以及对象 A依赖的其他还没有实例化的对象。这种情况是容器内部调用getBean()，对于本次请求的请求方是隐式的。

ApplicationContext启动之后会实例化所有的bean定义，但ApplicationContext在实现的过程中依然遵循Spring容器实现流程的两个阶段，只不过它会在启动阶段的活动完成之后，紧接着调用注册到该容器的所有bean定义的实例化方法getBean()。这就是为什么当你得到ApplicationContext类型的容器引用时，容器内所有对象已经被全部实例化完成。

1. AbstractApplicationContext.refresh 方法, 在其中会实例化所有bean.
2. 实例化bean是调用 AbstractBeanFactory.doGetBean 方法实现的.
3. 首先要转换beanName, 因为可能是别名也可能是FactoryBean
4. 会尝试从第一个map中获取bean, 如果没有就创建.
5. 单例会在**实例化但未初始化**的时候将Bean包装后的ObjectFactory加入第二个map.
6. 这样, 可以先返回未初始化完成的类, 而不是每次都重新创建. 避免循环依赖问题.
7. 其实还有第三个map, 会在使用到第二个map的时候将其取出加入第三个map中. 注意第二个map是包装后的, 而这个map是实例化对象.
8. 原型bean, 如果发现循环依赖就**直接抛出异常**
9. 在创建bean之前会记录正在创建, 用来判断循环引用
10. 之后就是创建实例, 并缓存实例化但未初始化的bean, 之后再注入属性
11. 还有一种bean叫做FactoryBean, 用于扩展beanFactory, 实现定制化的bean实例创建逻辑


## 源码
> 加载bean, 需要先从注解或者xml解析bean. 之后就是创建bean的过程.  

### doGetBean
首先初始化bean的入库是getbean方法, 其中调用了doGetBean方法.

```java
protected  T doGetBean(final String name, @Nullable final Class requiredType,
    @Nullable final Object[] args, boolean typeCheckOnly) throws BeansException{

	 // 首先是转换格式化beanName
  final String beanName = transformedBeanName(name);
  Object bean;

  // 尝试通过bean名称获取目标bean对象
  Object sharedInstance = getSingleton(beanName);
	 // 尝试加载工厂bean
  if (sharedInstance != null && args == null) {
		// log...
		bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
  }
  else{
		// 如果不是单例模式(原型模式) 并发送循环依赖 则直接抛异常
  	if (isPrototypeCurrentlyInCreation(beanName)) {
			throw new BeanCurrentlyInCreationException(beanName);
		}

		//... 省略部分代码 看单例部分的实现
		
		// 单例
		if (mbd.isSingleton()) {
    		// 这里就尝试创建目标对象，第二个参数传的就是一个ObjectFactory类型的对象，
    		// 这里是使用Java8的lamada
    		// 表达式书写的，只要上面的getSingleton()方法返回值为空，
   		// 则会调用这里的getSingleton()方法来创建目标对象
    		sharedInstance = getSingleton(beanName, () -> {
      		try {
        			// 尝试创建目标对象
        			return createBean(beanName, mbd, args);
      		} catch (BeansException ex) {
        			throw ex;
      		}
    		});
  	}
		// ... 省略了部分代码
  }

	// 检查需要的类型是否符合bean的实际类型，对应getBean时指定的requireType
  if (requiredType != null && bean != null && !requiredType.isAssignableFrom(bean.getClass())) {
     try {
         // 执行类型转换，转换成期望的类型
         return this.getTypeConverter().convertIfNecessary(bean, requiredType);
     } catch (TypeMismatchException ex) {
         if (logger.isDebugEnabled()) {
             logger.debug(“Failed to convert bean ‘” + name + “’ to required type ‘” + ClassUtils.getQualifiedName(requiredType) + “’”, ex);
         }
         throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
     }
  }
  return (T) bean;
}
```

上面的代码是省略了一部分. 重点展示了主流程. 我们先来看第一个getSingleton方法.

### getSingleton 缓存

我们需要知道这三个Map
* singletonFactories: Map<String, ObjectFactory<?>> 类型，用于记录创建bean的工厂
* singletonObjects: Map<String, Object> 类型，用于记录bean实例
* earlySingletonObjects: Map<String, Object> 类型，用于记录原始bean实例(未创建完成的)

```java
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
  // 首先尝试从singletonObjects获取是否存在以及初始化好的bean
  Object singletonObject = this.singletonObjects.get(beanName);
  // 如果为空, 且发现已经在初始化
  if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
    synchronized (this.singletonObjects) {
      // 从未初始化好的map中查询
      singletonObject = this.earlySingletonObjects.get(beanName);
      // 如果为空, 且允许早期依赖
      if (singletonObject == null && allowEarlyReference) {
        // 获取创建对象的工厂
        ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
        if (singletonFactory != null) {
          // 获取对象实例(可能是没加载好的)
          singletonObject = singletonFactory.getObject();
          // 将其提前暴露, 放入为初始化好的map中
          this.earlySingletonObjects.put(beanName, singletonObject);
          this.singletonFactories.remove(beanName);
        }
      }
    }
  }
  return singletonObject;
}

// 这里会在执行创建之前将beanName加入set中
public boolean isSingletonCurrentlyInCreation(String beanName) {
	return this.singletonsCurrentlyInCreation.contains(beanName);
}
```

这里的思路其实很简单, 就是先查询已经创建好的缓存(`singletonObjects`), 如果查到就直接返回; 如果没有已经创建好的对象, 则在**未创建好的对象缓存**(`earlySingletonObjects`)中查找; 如果其中还没有对象则会取其对应的创建工厂, 从创建工厂(`singletonFactories`)中取出**还没有创建好的缓存**并插入刚刚的 `earlySingletonObjects` 缓存中. 

这里如果allowEarlyReference为true且是单例模式, 则会使用上述的方法. 在对象实例创建完成但没有初始化其中field的时候, 将这个半成品的封装到 `singletonFactories` 中. 这样在之后尝试获取的时候就会发现其中在创建bean了. 将半成品取出放到半成品Map中. 可以先将半成品设置为其他类的属性.

**核心点如下**
- 这里的创建工厂其实就像对于未完全创建好的bean的封装.
- 未完全创建好的bean, 这个半成品其实是实例化了但没有初始化其中的field.
- 这样的意义是, 可以在让A中的field赋值未初始化的B. 这样如果A/B循环引用, 都可以先引用未初始化完成的bean, 就不会无限循环其初始化了.
- 只有单例可以使用这个逻辑, 也就是只有单例可以解决循环依赖问题.

> 这样大致的思路就有了, 之后我们再看下怎么去创建bean的.  

### createBean

```java
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) throws BeanCreationException {
   //... 校验是否可以被classLoader加载

   try {
    // bean加载记录, 用以判断循环引用和是否已经被加载
    mbdToUse.prepareMethodOverrides();
   }
   catch (BeanDefinitionValidationException ex) {
     throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
       beanName, “Validation of method overrides failed”, ex);
   }
    
   try {
     // 创建bean
     Object beanInstance = doCreateBean(beanName, mbdToUse, args);
     if (logger.isTraceEnabled()) {
       logger.trace("Finished creating instance of bean '" + beanName + "'");
     }
     return beanInstance;
   }
   catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
     throw ex;
   }
   catch (Throwable ex) {
     throw new BeanCreationException(
       mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
   }
}
```

所以说, 具体实现还要看doCreateBean方法.

### doCreateBean

```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
  throws BeanCreationException {
  // 实例化当前尝试获取的bean对象，比如A对象和B对象都是在这里实例化的
  BeanWrapper instanceWrapper = null;
  if (mbd.isSingleton()) {
    instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
  }
  if (instanceWrapper == null) {
    // 创建实例
    instanceWrapper = createBeanInstance(beanName, mbd, args);
  }

  // 调用所有的bean后置处理器
  synchronized (mbd.postProcessingLock) {
    if (!mbd.postProcessed) {
      try {
        applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
      }
      catch (Throwable ex) {
        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
          “Post-processing of merged bean definition failed”, ex);
      }
      mbd.postProcessed = true;
    }
  }

  // 判断Spring是否配置了支持提前暴露目标bean，也就是是否支持提前暴露半成品的bean
  boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences 
    && isSingletonCurrentlyInCreation(beanName));
  if (earlySingletonExposure) {
    // 如果支持，这里就会将当前生成的半成品的bean放到singletonFactories中，这个singletonFactories
    // 就是前面第一个getSingleton()方法中所使用到的singletonFactories属性，也就是说，这里就是
    // 封装半成品的bean的地方。而这里的getEarlyBeanReference()本质上是直接将放入的第三个参数，也就是目标bean直接返回
    addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
  }
  try {
    // 在初始化实例之后，这里就是判断当前bean是否依赖了其他的bean，如果依赖了，
    // 就会递归的调用getBean()方法尝试获取目标bean
    populateBean(beanName, mbd, instanceWrapper);
	   // 调用 init-method
    exposedObject = initializeBean(beanName, exposedObject, mbd);
  } catch (Throwable ex) {
    // 省略...
  }
  return exposedObject;
}
```

这里首先是**创建实例**, 将BeanDefinition转换为BeanWrapper。
- 如果存在工厂方法则使用工厂方法进行初始化
- 一个类有多个构造函数，每个构造函数都有不同的参数，所以需要根据参数锁定构造函数进行 bean 的实例化
- 如果即不存在工厂方法，也不存在带有参数的构造函数，会使用默认的构造函数进行 bean 的实例化

接着**调用所有的bean后置处理器, 其中就有@Autowired的实现**

之后处理了**循环依赖问题**, 就是我们之前说过的提前暴露实例化但未初始化半成品的策略

开始**属性注入**
- autowireByName 通过名字加载bean
- autowireByType 加载同一类型的所有bean (加载出来一个list)

最后还会**调用客户设定的初始化方法**, 以及**注册DisposableBean**

### getSingleton 创建

这样在来看一下调用创建bean的代码

```java
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
	Assert.notNull(beanName, "Bean name must not be null");
 // 加锁
 synchronized (this.singletonObjects) {
    // 检查是否已经被加载了，单例模式就是可以复用已经创建的 bean
    Object singletonObject = this.singletonObjects.get(beanName);
    if (singletonObject == null) {
        // 初始化前操作，校验是否 beanName 是否有别的线程在初始化，并加入初始化状态中
        beforeSingletonCreation(beanName);
        boolean newSingleton = false;
        boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
        if (recordSuppressedExceptions) {
            this.suppressedExceptions = new LinkedHashSet<>();
        }
        // 初始化 bean，这个就是刚才的回调接口调用的方法，实际执行的是 createBean 方法
        singletonObject = singletonFactory.getObject();
        newSingleton = true;
        if (recordSuppressedExceptions) {
            this.suppressedExceptions = null;
        }
        // 初始化后的操作，移除初始化状态
        afterSingletonCreation(beanName);
        if (newSingleton) {
            // 加入缓存
            addSingleton(beanName, singletonObject);
        }
    }
    return singletonObject;
 }
}
```

这里就是加锁并调用 createBean 方法.

### getObjectForBeanInstance

一般情况下Spring通过bean中的class属性，通过反射创建Bean的实例。但在某些情况下，实例化Bean的过程比较复杂，如果按照传统的方式，则需要在bean标签中提供大量的配置信息，配置方式的灵活性是受限的，这时采用编码的方式可能会得到一个简单的方案。Spring为此提供了一个 `FactoryBean` 的接口。**用户可以实例化该接口，实现定制化bean实例创建逻辑**

FactoryBean接口对应Spring框架来说占有重要的地位，Spring本身就提供了70多个FactoryBean的实现。他们隐藏了实例化一些复杂的细节，给上层应用带来了便利。从Spring3.0开始，FactoryBean开始支持泛型。

> 比如 Dubbo 的服务导入就是实现了 FactoryBean 接口, 在 getObject 方法中实现了服务导入.  

```java
protected Object getObjectForBeanInstance(
		Object beanInstance, String name, String beanName, @Nullable RootBeanDefinition mbd) {

	// 校验...

	// 如果不是FactoryBean 或者带有&前缀 返回类本身
	if (!(beanInstance instanceof FactoryBean) || BeanFactoryUtils.isFactoryDereference(name)) {
		return beanInstance;
	}

	Object object = null;
	if (mbd == null) {
		// 尝试缓存中获取
		object = getCachedObjectForFactoryBean(beanName);
	}
	if (object == null) {
		// 这里就确定是FactoryBean了
		FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
		// 再次尝试从缓存中获取
		if (mbd == null && containsBeanDefinition(beanName)) {
			mbd = getMergedLocalBeanDefinition(beanName);
		}
		boolean synthetic = (mbd != null && mbd.isSynthetic());
		object = getObjectFromFactoryBean(factory, beanName, !synthetic);
	}
	return object;
}
```

需要注意的是, 这个方法返回的是 `FactoryBean.getObject()`, 而不是FactoryBean本身. 如果想要获取的是FactoryBean则需要加上’&’符, 这样就会返回FactoryBean本身了.



### 参考
1. [高频面试题：Spring 如何解决循环依赖？ - 知乎](https://zhuanlan.zhihu.com/p/84267654)
2. [Spring解密 - Bean的加载流程 - 个人文章 - SegmentFault 思否](https://segmentfault.com/a/1190000012887776)
3. [Spring 源码学习(四) bean 的加载 - 掘金](https://juejin.im/post/5d0a1c50f265da1bbe5e08b8#heading-15)
4. [创建Bean · Spring源码阅读](https://gavinzhang1.gitbooks.io/spring/content/chuang_jian_bean.html)
5. [Spring源码解析：简单容器中Bean的加载过程初探 - 指间 - OSCHINA](https://my.oschina.net/wangzhenchao/blog/915897)
