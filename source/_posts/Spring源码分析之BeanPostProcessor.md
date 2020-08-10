---
title: Spring源码分析之BeanPostProcessor
date: 2020-08-10
cover: https://cdn.jsdelivr.net/gh/dtlexi/lexi.blog/src/image/15.jfif
top_img: https://cdn.jsdelivr.net/gh/dtlexi/lexi.blog/src/image/16.jfif
categories:
 - java
 - spring
tags:
 - java
 - spring
 - 源码分析
 - spring 源码分析
---


BeanFactoryPostProcessor和BeanPostProcessor这两个接口都是初始化bean时对外暴露的入口之一，和Aware类似。是Spring预先埋好的钩子。弄清楚Spring的各个BeanPostProcess的执行时机和作用有助于我们更好地理解spring的生命周期。同时能更好的方便我们对spring进行扩展。

Spring在实例化Bean时一共调用了八次后置处理器，下面我们来对他们一一进行分析。

### 1. 第一次调用后置处理器
Spring第一次调用后置处理器实在`Spring`实例化`Bean`对象之前。Spring此时将会调用实现了`InstantiationAwareBeanPostProcessors`接口的后置处理器的`postProcessBeforeInstantiation`方法。如果此时返回不为空，那么Spring将会返回当前对象，不会再重新创建对象。并且不会给对象内部属性赋值

```java
@Nullable
default Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
   return null;
}
```

程序员可以实现`InstantiationAwareBeanPostProcessors`接口，来完成一些特定类的初始化工作(但是好像一般不这样做)。

Spring第一次调用后置处理器的源码如下：

```java
@Override
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
      throws BeanCreationException {
   //...省略部分类容
   try {
      // Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.

      // 第一次调用spring后置处理器
      // InstantiationAwareBeanPostProcessor postProcessBeforeInstantiation
      // 此时spring还没有开始实例化对象
      // 程序员可以接管spring的创建对象流程，返回自定义对象（spring建议返回代理对象）
      Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
      if (bean != null) {
         return bean;
      }
   }
   catch (Throwable ex) {
      throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
            "BeanPostProcessor before instantiation of bean failed", ex);
   }
   //...省略部分类容
}
```
```java
@Nullable
protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
   Object bean = null;
   if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
      // Make sure bean class is actually resolved at this point.

      /**
       * mbd.isSynthetic() 判断当前类是否是合成类，合成类指的是
       * class A{
       *     class B{
       *         private B()
       *         {
       *
       *         }
       *     }
       *     psvm(){
       *         B b=new B();
       *
       *     }
       * }
       * 因为B的构造方法私有化，但是在内的类的外部类中，有可以调用内部类的私有方法，字段等，
       * 此时可以在A中实例化B
       * JVM通过构建内外一个类来实现的，此时有三个class文件。而spring的扫描时扫描类文件。
       * 所以此时要判断是否是合成类
       *
       * hasInstantiationAwareBeanPostProcessors()：
       * 判断当前是否有后置处理器实现了InstantiationAwareBeanPostProcessors，
       * AspectJAutoProxyCreator就是实现了这个类
       *
       * 这边第一次调用了后置处理器
       */
      if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
         //获取当前bd对应的Class
         Class<?> targetType = determineTargetType(beanName, mbd);
         if (targetType != null) {
            // 调用InstantiationAwareBeanPostProcessor postProcessBeforeInstantiation方法
            bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
            if (bean != null) {
               // 调用BeanPostProcessor的postProcessAfterInitialization方法
               bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
            }
         }
      }
      mbd.beforeInstantiationResolved = (bean != null);
   }
   return bean;
}
```

### 2. 第二次调用后置处理器

Sping第二次调用的是`SmartInstantiationAwareBeanPostProcessor`的`determineCandidateConstructors`方法。Spring第二次调用后置处理器也是在实例化对象之前。

```java
@Nullable
default Constructor<?>[] determineCandidateConstructors(Class<?> beanClass, String beanName)
      throws BeansException {

   return null;
}
```
Spring提供这个扩展是让后置处理器参与到Spring构造方法的判断中去

这边最主要的实现就是`AutowiredAnnotationBeanPostProcessor`，它是Spring用来推断构造方法的。

Spirng第二次调用后置处理器的源码：

```java
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
   // ... 省略部分内容
   // 第二次执行执行后置处理器BeanPostProcessor
   // 用来确定构造方法，主要逻辑如下
   // 1. 这边的逻辑是如果存在@Autowired(required=true)的构造方法。直接返回当前构造方法
   // 2. 如果不存在required=true的构造方法，但是存在required=false的构造方法，返回默认构造方法和@Autowired(required=false)的构造方法
   // 3. 如果有且只存在一个构造方法，并且当前构造方法的参数>0 那么返回当前构造方法
   Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
   // ... 省略部分内容
}
```
```java
protected Constructor<?>[] determineConstructorsFromBeanPostProcessors(@Nullable Class<?> beanClass, String beanName)
      throws BeansException {

   // 调用SmartInstantiationAwareBeanPostProcessor determineCandidateConstructors
   // 推断构造方法
   if (beanClass != null && hasInstantiationAwareBeanPostProcessors()) {
      for (BeanPostProcessor bp : getBeanPostProcessors()) {
         if (bp instanceof SmartInstantiationAwareBeanPostProcessor) {
            SmartInstantiationAwareBeanPostProcessor ibp = (SmartInstantiationAwareBeanPostProcessor) bp;
            Constructor<?>[] ctors = ibp.determineCandidateConstructors(beanClass, beanName);
            if (ctors != null) {
               return ctors;
            }
         }
      }
   }
   return null;
}
```

### 3. 第三次调用后置处理器
Spring第三次执行`MergedBeanDefinitionPostProcessor`的`postProcessMergedBeanDefinition`方法，此时`Spring`刚刚实例化完成Bean对象
Spring不仅有`RootBeanDefinition`和`ChildBeanDefinition`的概念，Spring还有`Merged（合并）`的概念，即合并`root`和`child`的内容，使bd内容完整。
Spring第三次后置处理器可以简单理解为bd合并完成后（此时bd在之前就已经合并完成，不是此处合并的），此时是一个完整的bd，程序员可以进行相关操作。

```java
void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName);
```

这边典型的后置处理器有`CommonAnnotationBeanPostProcessor`和`AutowiredAnnotationBeanPostProcessor`。
其中`CommonAnnotationBeanPostProcessor`是找到所有`@Resource`注解，而`AutowiredAnnotationBeanPostProcessor`是找到所有`@Autowired`注解的

Spring第三次调用后置处理器源码：

```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
      throws BeanCreationException {

   //...删除相关代码
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
            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                  "Post-processing of merged bean definition failed", ex);
         }
         mbd.postProcessed = true;
      }
   }
   //...删除相关代码
}
```

```java
protected void applyMergedBeanDefinitionPostProcessors(RootBeanDefinition mbd, Class<?> beanType, String beanName) {
   for (BeanPostProcessor bp : getBeanPostProcessors()) {
      if (bp instanceof MergedBeanDefinitionPostProcessor) {
         MergedBeanDefinitionPostProcessor bdp = (MergedBeanDefinitionPostProcessor) bp;
         bdp.postProcessMergedBeanDefinition(mbd, beanType, beanName);
      }
   }
}
```

### 4. 第四次调用后置处理器
紧接着第三次后置处理器之后是Spring第四次调用后置处理器。

当然，此处不应该说调用后置处理器，因为此处没有调用后置处理器。此处只是将后置处理器封装lamb表达式封装到一个map中去了，这个map就是spring解决循环依赖中的`singleFactory`。

Spring将在解决循环依赖的`getSingle`方法中调用此后置处理器。

第四次循环依赖的作用是，当我们在对bean对象本身就行修改时（参考aop代理）。此时创建的对象和给其他对象属性赋值的可能是不同的对象。那么此时就需要提前拿到他的代理后的对象。

这边也是spring 解决循环依赖同时又能兼顾aop的核心，将在Spring 循环依赖中详细介绍。

Spring在此调用的是`SmartInstantiationAwareBeanPostProcessor`的`getEarlyBeanReference`方法

```java
default Object getEarlyBeanReference(Object bean, String beanName) throws BeansException {
   return bean;
}
```

Spring第四次调用后置处理器源码：
```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
      throws BeanCreationException {
   //...省略部代码
   // this.allowCircularReferences是否允许循环依赖，可以通过setAllowCircularReferences()设置
   // 这边主要是检查当前是否是单例对象，是否允许循环依赖
   boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
         isSingletonCurrentlyInCreation(beanName));
   if (earlySingletonExposure) {
      if (logger.isTraceEnabled()) {
         logger.trace("Eagerly caching bean '" + beanName +
               "' to allow for resolving potential circular references");
      }
      // 第四次调用后置处理器，在getEarlyBeanReference方法里面
      // 如果允许调用循环引用，这边会往beanFactory的singletonFactories put lamb表达式
      // () -> getEarlyBeanReference(beanName, mbd, bean)
      // getEarlyBeanReference 是提前调用aop的方法产生代理对象
      // 这边也是spring aop 解决循环依赖同时又能兼顾aop的核心
      addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
   }
   //...省略无关代码
}
```

```java
protected Object getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean) {
   Object exposedObject = bean;
   if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
      for (BeanPostProcessor bp : getBeanPostProcessors()) {
         if (bp instanceof SmartInstantiationAwareBeanPostProcessor) {
            SmartInstantiationAwareBeanPostProcessor ibp = (SmartInstantiationAwareBeanPostProcessor) bp;
            exposedObject = ibp.getEarlyBeanReference(exposedObject, beanName);
         }
      }
   }
   return exposedObject;
}
```



### 5. 第五次调用后置处理器
Spring的第五次后置处理器调用发生在Spring填充属性之前。Spring此时会调用`InstantiationAwareBeanPostProcessor`的`postProcessAfterInstantiation`方法。

```java
default boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException {
   return true;
}
```

Spring提供当前后置处理器的目的有俩个：
1. 程序员可以取消某些bean的属性赋值
2. 程序员可以手动给某些bean的属性赋值

Sping第五次调用后置处理器源码：

```java
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
   // ...删除无关代码
   // 执行后置处理器InstantiationAwareBeanPostProcessor的postProcessAfterInstantiation方法
   // 如果postProcessAfterInstantiation方法返回false，那么将直接退出该方法，不会执行下面的赋值操作
   // 这边spring默认的后置处理器都是直接返回true
   // 这边应该是spring开发给程序员使用的，
   // 控制 autowired by type 但是有的又不想注入
   // spring提供此后置处理器的目的还有一个，让程序员自己给bean赋值，然后返回false,取消spring自动赋值
   if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
      for (BeanPostProcessor bp : getBeanPostProcessors()) {
         if (bp instanceof InstantiationAwareBeanPostProcessor) {
            // 执行后置处理器的 postProcessAfterInstantiation
            // 这是spring第五次调用后置处理器
            InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
            if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
               return;
            }
         }
      }
   }
   // ...删除无关代码
}
```




### 6. 第六次调用后置处理器

Spring第六次后置处理器调用是发生在Spring给bean属性赋值时。Spring此时会调用`InstantiationAwareBeanPostProcessor`的`postProcessProperties`方法,这边还调用了`InstantiationAwareBeanPostProcessor`的`postProcessPropertyValues`，只不过这个方法Spring已经废弃掉了。

```java
@Nullable
default PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName)
      throws BeansException {

   return null;
}
```

Spirng此时调用后置处理器是想让程序员参与到spring bean的属性注入中去。

这边比较典型的就是`CommonAnnotationBeanPostProcessor`和`AutowiredAnnotationBeanPostProcessor`，前者完成了`@Resource`注解标注的属性的注入。后者完成了`Autowired`标注的属性的注入

Spring第六次后置处理器的调用源码如下：

```java
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
   //... 删除无关代码
   PropertyDescriptor[] filteredPds = null;
   if (hasInstAwareBpps) {
      // spring第六次调用后置处理器
      // 调用 InstantiationAwareBeanPostProcessor postProcessProperties 完成属性注入
      // 其中@CommonAnnotationBeanPostProcessor 完成@Resource属性注入
      // @AutowiredAnnotationBeanPostProcessor 完成@Autowired属性注入
      for (BeanPostProcessor bp : getBeanPostProcessors()) {
         if (bp instanceof InstantiationAwareBeanPostProcessor) {
            InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
            // 执行postProcessProperties
            PropertyValues pvsToUse = ibp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
            if (pvsToUse == null) {
               if (filteredPds == null) {
                  filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
               }
               // 执行postProcessPropertyValues，但是Spring已经废弃了此方法
               pvsToUse = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
               if (pvsToUse == null) {
                  return;
               }
            }
            pvs = pvsToUse;
         }
      }
   }
   //... 删除无关代码
}
```

### 7. 第七次调用后置处理器
Spring第八次后置处理器是在Spring完成了属性注入，并且完成了部分Aware方法的调用后执行的，这边为什么要说部分，因为有的Aware是在后置处理器中调用的

Spring此时调用的是`BeanPostProcessor`的`postProcessBeforeInitialization`方法,此时还没有调用bean的生命周期函数方法。

```java
@Nullable
default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
   return bean;
}
```

spring提供当前后置处理器的目的是让程序员参与到spring的初始化中去。

这边比较典型的实现类是`ApplicationContextAwareProcessor`和`InitDestroyAnnotationBeanPostProcessor`。其中`ApplicationContextAwareProcessor`负责处理其他Aware接口方法的调用。而`InitDestroyAnnotationBeanPostProcessor`负责处理`@PostConstruct`注解。

Spring第七次调用后置处理器的源码
```java
protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
   // ...删除部分源码
   // 第七次后置处理器的调用
   // 执行postProcessBeforeInitialization
   // ApplicationContextAwareProcessor 在这边处理各种Aware
   // InitDestroyAnnotationBeanPostProcessor 在这边处理@PostConstruct注解
   Object wrappedBean = bean;
   if (mbd == null || !mbd.isSynthetic()) {
      wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
   }
   // ...删除部分源码
}
```
```java
@Override
public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
      throws BeansException {

   Object result = existingBean;
   for (BeanPostProcessor processor : getBeanPostProcessors()) {
      Object current = processor.postProcessAfterInitialization(result, beanName);
      if (current == null) {
         return result;
      }
      result = current;
   }
   return result;
}
```

### 8. 第八次调用后置处理器

Spring第八次调用后置处理器是发生在Spring完全初始化完Bean对象后，执行的。这也是Sprig最后一次调用后置处理器。

Spring此时会调用`BeanPostProcessor`的`postProcessAfterInitialization`方法

```java
@Nullable
default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
   return bean;
}
```

Spring提供当前后置处理器是想让Spirng对完成初始化的对象进行修改，比如代理。

这边比较典型的是Sprng Aop的实现。

Spring第八次调用后置处理器代码实现：

```java
protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
   // ... 删除部分无关代码
   // 第八次后置处理器的调用
   // 执行postProcessAfterInitialization
   // spring aop就是在这边添加代理的
   if (mbd == null || !mbd.isSynthetic()) {
      wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
   }

   return wrappedBean;
}
```

```java
public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
      throws BeansException {

   Object result = existingBean;
   for (BeanPostProcessor processor : getBeanPostProcessors()) {
      Object current = processor.postProcessAfterInitialization(result, beanName);
      if (current == null) {
         return result;
      }
      result = current;
   }
   return result;
}
```

下面我们来对Spring的生命周期做个简单的总结：

![](https://cdn.jsdelivr.net/gh/dtlexi/lexi.blog/src/image/beanPostProcessor.png)

