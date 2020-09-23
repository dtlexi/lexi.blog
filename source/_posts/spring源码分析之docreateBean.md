---
title: spring源码分析之doCreateBean.md
date: 2020-08-22
cover: https://cdn.jsdelivr.net/gh/dtlexi/lexi.blog/src/image/21.jpg
top_img: https://cdn.jsdelivr.net/gh/dtlexi/lexi.blog/src/image/22.jpg
categories:
 - java
 - spring
tags:
 - java
 - spring
 - 源码分析
 - spring 源码分析
---

`doCreateBean`方法是spring最后真正用来创建对象的方法。spring创建bean过程中的实例化对象，属性赋值，执行生命周期函数都是在当前方法中完成的。是spring生命周期中最重要的一个方法

下面我们来具体分析一下该方法
```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
      throws BeanCreationException {

   // Instantiate the bean.
   BeanWrapper instanceWrapper = null;
   if (mbd.isSingleton()) {
      // todo 确定一下这边为什么要remove,是什么时候加到这个factoryBeanInstanceCache中去的。
      instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
   }
   if (instanceWrapper == null) {
      // 这个方法内部第二次调用后置处理器

      // 实例化对象
      // 这边主要完成了俩件事
      //        1. 推断构造方法
      //        2. 实例化对象
      instanceWrapper = createBeanInstance(beanName, mbd, args);
   }

   // 得到实例化出来的对象
   final Object bean = instanceWrapper.getWrappedInstance();

   Class<?> beanType = instanceWrapper.getWrappedClass();
   if (beanType != NullBean.class) {
      mbd.resolvedTargetType = beanType;
   }

   // 第三次调用后置处理器BeanPostProcessor
   synchronized (mbd.postProcessingLock) {
      if (!mbd.postProcessed) {
         try {
            // 调用MergedBeanDefinitionPostProcessor的postProcessMergeBeanDefinition()
            // 这边典型的有CommonAnnotationBeanPostProcessor和AutowiredAnnotationBeanPostProcessor
            // CommonAnnotationBeanPostProcessor 是找到所有@Resource注解
            // AutowiredAnnotationBeanPostProcessor 找到所有@Autowired注解
            applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
         }
         catch (Throwable ex) {
            // 省略无关代码...
         }
         mbd.postProcessed = true;
      }
   }

 
   // this.allowCircularReferences是否允许循环依赖，可以通过setAllowCircularReferences()设置
   // 是否单利&&是否支持循环依赖&&是否正在创建中
   boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
         isSingletonCurrentlyInCreation(beanName));
   if (earlySingletonExposure) {
      // 第四次调用后置处理器，在getEarlyBeanReference方法里面
      // 如果允许调用循环引用，这边会往beanFactory的singletonFactories put lamb表达式
      // () -> getEarlyBeanReference(beanName, mbd, bean)
      // getEarlyBeanReference 是提前调用aop的方法产生代理对象
      // 这边也是spring aop 解决循环依赖同时又能兼顾aop的核心
      addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
   }

   // Initialize the bean instance.
   Object exposedObject = bean;
   try {
      // 填充属性
      // 里面会完成第五次第六次后置处理器调用
      populateBean(beanName, mbd, instanceWrapper);


      // 初始化Spring,不是实例化
      // 第七次第八次后置处理器调用
      exposedObject = initializeBean(beanName, exposedObject, mbd);
   }
   catch (Throwable ex) {
      //...
   }

   if (earlySingletonExposure) {
      // 这边分为俩种情况
      // 1. 非循环引用
      //   此时是创建对象后的第一次调用getSingleton.
      //   此时singleObjects==null  earlySingletonObjects==null  singletonFactories != null
      //   但是此时allowEarlyReference == false ，所以此时不会get singletonFactories
      //       此时返回null
      // 2. 循环引用
      //    此时已经在populateBean属性赋值时调用过getSingleton，并且当时allowEarlyReference==true
      //    所以populateBean中会调用singletonFactories，并且添加到earlySingletonObjects，此时earlySingletonObjects不为空
      //    此时返回不为空

      Object earlySingletonReference = getSingleton(beanName, false);
      if (earlySingletonReference != null) {
         // bean 是通过反射创建出来的对象
         // exposedObject 是调用 postProcessAfterInitialization 返回的对象
         // 在aop情况下
         // 1. 非循环引用
         //    exposedObject 是代理对象
         //        bean 原始对象
         // 2. 循环引用
         //    因为之前调用了getSingleton 的 singletonFactory 即 getEarlyBeanReference
         //    getEarlyBeanReference 方法中 earlyProxyReferences.put
         //        postProcessAfterInitialization 方法中不会在创建
         //        此时exposedObject== bean
         if (exposedObject == bean) {
            exposedObject = earlySingletonReference;
         }
         // 下面方法看不懂,在此省略
         else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
            //...
         }
      }
   }

   // Register bean as disposable.
   // 注册disposable
   try {
      registerDisposableBeanIfNecessary(beanName, bean, mbd);
   }
   catch (BeanDefinitionValidationException ex) {
      // ...
   }

   return exposedObject;
}
```
上面代码注释比较多，但是可能写的比较乱，大家不一定看得懂，下面我来分析一下`doCreateBean`的执行流程
1. 从缓存`BeanWrapper`中获取对象，并且清除缓存，但是这个缓存是什么时候添加的，在什么情况下生效的，我暂时没搞清楚。
2. 调用`createBeanInstance`创建对象。`createBeanInstance`中主要完成了推断构造方法，和实例化对象俩件事，我们将在下面一片文章中具体分析
3. 调用`MergedBeanDefinitionPostProcessor`的`postProcessMergeBeanDefinition()`。这是spring第三次调用后置处理器
4. if `单利 && 允许循环引用 && 当前创建中`，暴露`早期引用(early reference)`，用于解决循环引用
5. 属性赋值
6. 执行各种Aware和生命周期函数
7. 注册销毁逻辑

我们将在接下来的几篇文章中对2~7作详细分析。
