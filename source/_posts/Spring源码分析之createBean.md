---
title: spring源码分析之createBean.md
date: 2020-08-22
cover: https://cdn.jsdelivr.net/gh/dtlexi/lexi.blog/src/image/20.jpg
top_img: https://cdn.jsdelivr.net/gh/dtlexi/lexi.blog/src/image/19.jpg
categories:
 - java
 - spring
tags:
 - java
 - spring
 - 源码分析
 - spring 源码分析
---

`createBean`是spirng用来实例化bean对象的。spring实例化对象简单可以分为三步
1. 通过反射创建对象
2. 给对象属性赋值
3. 调用对象生命周期函数
下面我们就来想`createBean`方法进行详细分析。下面是spring createBean的源码


```java
@Override
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
      throws BeanCreationException {

   if (logger.isTraceEnabled()) {
      logger.trace("Creating instance of bean '" + beanName + "'");
   }
   RootBeanDefinition mbdToUse = mbd;

   // Make sure bean class is actually resolved at this point, and
   // clone the bean definition in case of a dynamically resolved Class
   // which cannot be stored in the shared merged bean definition.

   // 拿到当前bd的clss
   Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
   if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
      mbdToUse = new RootBeanDefinition(mbd);
      mbdToUse.setBeanClass(resolvedClass);
   }

   
   // Prepare method overrides.
   // 处理 lookup-method 和 replace-method 配置，Spring 将这两个配置统称为 override method
   // <bean id="testLookUpService" class="com.lexi.service.TestLookUpService">
   //    <lookup-method name="getHelloService"></lookup-method>
   // </bean>
   // 如上：
   // 这边只是检查name为getHelloService的方法的数量，如果当前方法数量为1,表示lookup方法没有重载。，
   // 设置当前overloaded false。表示当前没有重载，方便后面处理
   // 注意：这边只处理xml中的'<lookup-method name="getHelloService"></lookup-method>'
   // 不会处理@Method注解
   // 其实这也好理解，只有xml中才有可能出现重载，而@Lookup注解直接添加在方法上面，不会有这个问题
   try {
      mbdToUse.prepareMethodOverrides();
   }
   catch (BeanDefinitionValidationException ex) {
      throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
            beanName, "Validation of method overrides failed", ex);
   }

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

   try {
      // 这边是真正的开始创建bean
      Object beanInstance = doCreateBean(beanName, mbdToUse, args);
      if (logger.isTraceEnabled()) {
         logger.trace("Finished creating instance of bean '" + beanName + "'");
      }
      return beanInstance;
   }
   catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
      // A previously detected exception with proper bean creation context already,
      // or illegal singleton state to be communicated up to DefaultSingletonBeanRegistry.
      throw ex;
   }
   catch (Throwable ex) {
      throw new BeanCreationException(
            mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
   }
}
```
上面的代码比较简单，下面我们简单说一下`createBean`方法做的事情
1. 解析bd对应的class
2. 处理`lookup-method` 和 `replace-method`配置，Spring 将这两个配置统称为`override method`
3. 调用后置处理器`InstantiationAwareBeanPostProcessor`的`postProcessBeforeInstantiation`方法,这边是Spring第一次调用后置处理器，目的是确定程序员有没有提供对象创建逻辑。如果postProcessBeforeInstantiation返回不为空，那么spring将直接返回程序员提供的对象
4. 调用 `doCreateBean` 创建 bean 实例

## 处理override方法
```java
try {
   mbdToUse.prepareMethodOverrides();
}
catch (BeanDefinitionValidationException ex) {
   throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
         beanName, "Validation of method overrides failed", ex);
}

public void prepareMethodOverrides() throws BeanDefinitionValidationException {
   // Check that lookup methods exist and determine their overloaded status.
   if (hasMethodOverrides()) {
      // 获取所有override method
      // 循环调用prepareMethodOverride
      getMethodOverrides().getOverrides().forEach(this::prepareMethodOverride);
   }
}

protected void prepareMethodOverride(MethodOverride mo) throws BeanDefinitionValidationException {
   // 获取当前方法数量
   int count = ClassUtils.getMethodCountForName(getBeanClass(), mo.getMethodName());
   if (count == 0) {
      throw new BeanDefinitionValidationException(
            "Invalid method override: no method with name '" + mo.getMethodName() +
            "' on class [" + getBeanClassName() + "]");
   }
   // 当前没有重载
   // 如果当前没有重载，set overLoaded false
   // 方便接下来处理lookup
   else if (count == 1) {
      // Mark override as not overloaded, to avoid the overhead of arg type checking.
      mo.setOverloaded(false);
   }
}
```

上面的代码逻辑比较简单，这边只是简单的判断了一下`<lookup-method name="xxx"></lookup-method>`中对应的name的方法的数量，如果 count=1 表示当前没有重载。这边只是方便后面处理lookup，注意这边不会处理`@Lookup`注解

## 第一次调用后置处理器
接下来是Spring生命周期中的第一次调用后置处理器。
```java
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
```
```java
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
```java
@Nullable
protected Object applyBeanPostProcessorsBeforeInstantiation(Class<?> beanClass, String beanName) {
   for (BeanPostProcessor bp : getBeanPostProcessors()) {
      if (bp instanceof InstantiationAwareBeanPostProcessor) {
         InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
         Object result = ibp.postProcessBeforeInstantiation(beanClass, beanName);
         if (result != null) {
            return result;
         }
      }
   }
   return null;
}
```
这边调用的是`InstantiationAwareBeanPostProcessor`的`postProcessorBeforeInstantiation`方法。这边是Spring提供给程序员的一个钩子，程序员可以通过实现当前后置处理器，来自定义bean的创建。
如果程序员返回了自定义对象，那么spring将不会走spring bean的生命周期

## doCreateBean 方法创建 bean
```java
try {
   // 这边是真正的开始创建bean
   Object beanInstance = doCreateBean(beanName, mbdToUse, args);
   if (logger.isTraceEnabled()) {
      logger.trace("Finished creating instance of bean '" + beanName + "'");
   }
   return beanInstance;
}
catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
   // A previously detected exception with proper bean creation context already,
   // or illegal singleton state to be communicated up to DefaultSingletonBeanRegistry.
   throw ex;
}
catch (Throwable ex) {
   throw new BeanCreationException(
         mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
}
```

doCreateBean 是Spring真正创建Bean的方法，我将会在下一篇博客中对此在进行详细分析。

[传送门](http://www.renjilin.online/java/spring/spring源码分析之docreateBean.html)