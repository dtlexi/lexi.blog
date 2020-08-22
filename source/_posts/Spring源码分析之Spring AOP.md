---
title: spring源码分析-spring aop 
date: 2020-07-01
cover: https://cdn.jsdelivr.net/gh/dtlexi/lexi.blog/src/image/3.jpg
top_img: https://cdn.jsdelivr.net/gh/dtlexi/lexi.blog/src/image/big5.jpeg
categories:
 - java
 - spring
tags:
 - java
 - spring
 - 源码分析
 - spring 源码分析
---

### 引入

SpringAop使用的是`@EnableAspectJAutoProxy`注解，点进去看看源码
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(AspectJAutoProxyRegistrar.class)
public @interface EnableAspectJAutoProxy {
   boolean proxyTargetClass() default false;

   boolean exposeProxy() default false;
}
```

发现`EnableAspectJAutoProxy`注解内部引用了一个`@Import`注解，引入了一个`AspectJAutoProxyRegistrar`类，具体作用详见 Spring @Import.md

我们看一下`AspectJAutoProxyRegistrar`类
```java
class AspectJAutoProxyRegistrar implements ImportBeanDefinitionRegistrar {

   /**
    * Register, escalate, and configure the AspectJ auto proxy creator based on the value
    * of the @{@link EnableAspectJAutoProxy#proxyTargetClass()} attribute on the importing
    * {@code @Configuration} class.
    */
   @Override
   public void registerBeanDefinitions(
      AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
      AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry);

      AnnotationAttributes enableAspectJAutoProxy =
            AnnotationConfigUtils.attributesFor(importingClassMetadata, EnableAspectJAutoProxy.class);
      if (enableAspectJAutoProxy != null) {
         if (enableAspectJAutoProxy.getBoolean("proxyTargetClass")) {
            AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
         }
         if (enableAspectJAutoProxy.getBoolean("exposeProxy")) {
            AopConfigUtils.forceAutoProxyCreatorToExposeProxy(registry);
         }
      }
   }

}
```
发现`AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry);`注入了一个`ProxyCreator`,继续跟进

```java
@Nullable
public static BeanDefinition registerAspectJAnnotationAutoProxyCreatorIfNecessary(BeanDefinitionRegistry registry) {
   return registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry, null);
}
```

```java
@Nullable
public static BeanDefinition registerAspectJAnnotationAutoProxyCreatorIfNecessary(
      BeanDefinitionRegistry registry, @Nullable Object source) {

   return registerOrEscalateApcAsRequired(AnnotationAwareAspectJAutoProxyCreator.class, registry, source);
}
```

```java
private static BeanDefinition registerOrEscalateApcAsRequired(
      Class<?> cls, BeanDefinitionRegistry registry, @Nullable Object source) {

   Assert.notNull(registry, "BeanDefinitionRegistry must not be null");

   if (registry.containsBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME)) {
      BeanDefinition apcDefinition = registry.getBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME);
      if (!cls.getName().equals(apcDefinition.getBeanClassName())) {
         int currentPriority = findPriorityForClass(apcDefinition.getBeanClassName());
         int requiredPriority = findPriorityForClass(cls);
         if (currentPriority < requiredPriority) {
            apcDefinition.setBeanClassName(cls.getName());
         }
      }
      return null;
   }

   RootBeanDefinition beanDefinition = new RootBeanDefinition(cls);
   beanDefinition.setSource(source);
   beanDefinition.getPropertyValues().add("order", Ordered.HIGHEST_PRECEDENCE);
   beanDefinition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
   registry.registerBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME, beanDefinition);
   return beanDefinition;
}
```

通过上面的代码我们发现Import主要是注册了一个`AnnotationAwareAspectJAutoProxyCreator`,通过查看该方法的继承关系，我们发现该方法继承自`AbstractAutoProxyCreator`,最终继承自`BeanPostProcesser`,结合`BeanPostProcesser`的知识，我们猜想SpringAop是在`BeanPostProcesser`的`postProcessAfterInitialization`中对目标类完成改造的。我们在`AbstractAutoProxyCreator`类中找到了该方法

### BeanPostProcessor 

BeanPostProcessor 接口:
```java
public interface BeanPostProcessor {

   @Nullable
   default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
      return bean;
   }

   @Nullable
   default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
      return bean;
   }

}
```

AbstractAutoProxyCreator抽象类简化：
```java
public abstract class AbstractAutoProxyCreator extends ProxyProcessorSupport
        implements SmartInstantiationAwareBeanPostProcessor, BeanFactoryAware {
    
    @Override
    /** bean 初始化后置处理方法 */
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        if (bean != null) {
            Object cacheKey = getCacheKey(bean.getClass(), beanName);
            if (!this.earlyProxyReferences.contains(cacheKey)) {
                // 如果需要，为 bean 生成代理对象
                return wrapIfNecessary(bean, beanName, cacheKey);
            }
        }
        return bean;
    }
    
    protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
        if (beanName != null && this.targetSourcedBeans.contains(beanName)) {
            return bean;
        }
         // 判断是否不应该代理这个bean
        if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
            return bean;
        }

        /*
         * 如果是基础设施类（Pointcut、Advice、Advisor 等接口的实现类），或是应该跳过的类，
         * 则不应该生成代理，此时直接返回 bean
         */ 
        if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
            // 将 <cacheKey, FALSE> 键值对放入缓存中，供上面的 if 分支使用
            this.advisedBeans.put(cacheKey, Boolean.FALSE);
            return bean;
        }

        // 为目标 bean 查找合适的通知器
        Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
        /*
         * 若 specificInterceptors != null，即 specificInterceptors != DO_NOT_PROXY，
         * 则为 bean 生成代理对象，否则直接返回 bean
         */ 
        if (specificInterceptors != DO_NOT_PROXY) {
            this.advisedBeans.put(cacheKey, Boolean.TRUE);
            // 创建代理
            Object proxy = createProxy(
                    bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
            this.proxyTypes.put(cacheKey, proxy.getClass());
            /*
             * 返回代理对象，此时 IOC 容器输入 bean，得到 proxy。此时，
             * beanName 对应的 bean 是代理对象，而非原始的 bean
             */ 
            return proxy;
        }

        this.advisedBeans.put(cacheKey, Boolean.FALSE);
        // specificInterceptors = null，直接返回 bean
        return bean;
    }
}
```
以上就是 Spring AOP 创建代理对象的入口方法分析，过程比较简单，这里简单总结一下：
1. 若 bean 是 AOP 基础设施类型，则直接返回
2. 为 bean 查找合适的通知器
3. 如果通知器数组不为空，则为 bean 生成代理对象，并返回该对象
4. 若数组为空，则返回原始 bean

### 创建代理对象

通过上面的代码我们可以知道代理对象是通过`createProxy`方法创建的，那我们就看看这个方法
```java
protected Object createProxy(
        Class<?> beanClass, String beanName, Object[] specificInterceptors, TargetSource targetSource) {

    if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
        AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
    }

    ProxyFactory proxyFactory = new ProxyFactory();
    proxyFactory.copyFrom(this);

    /*
     * 默认配置下，或用户显式配置 proxy-target-class = "false" 时，
     * 这里的 proxyFactory.isProxyTargetClass() 也为 false
     */
    if (!proxyFactory.isProxyTargetClass()) {
        if (shouldProxyTargetClass(beanClass, beanName)) {
            proxyFactory.setProxyTargetClass(true);
        }
        else {
            /*
             * 检测 beanClass 是否实现了接口，若未实现，则将 
             * proxyFactory 的成员变量 proxyTargetClass 设为 true
             */ 
            evaluateProxyInterfaces(beanClass, proxyFactory);
        }
    }

    // specificInterceptors 中若包含有 Advice，此处将 Advice 转为 Advisor
    Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
    proxyFactory.addAdvisors(advisors);
    proxyFactory.setTargetSource(targetSource);
    customizeProxyFactory(proxyFactory);

    proxyFactory.setFrozen(this.freezeProxy);
    if (advisorsPreFiltered()) {
        proxyFactory.setPreFiltered(true);
    }

    // 创建代理
    return proxyFactory.getProxy(getProxyClassLoader());
}

public Object getProxy(ClassLoader classLoader) {
    // 先创建 AopProxy 实现类对象，然后再调用 getProxy 为目标 bean 创建代理对象
    return createAopProxy().getProxy(classLoader);
}
```

getProxy 这里有两个方法调用，一个是调用 createAopProxy 创建 AopProxy 实现类对象，然后再调用 AopProxy 实现类对象中的 getProxy 创建代理对象。这里我们先来看一下创建 AopProxy 实现类对象的过程，如下：

```java
protected final synchronized AopProxy createAopProxy() {
    if (!this.active) {
        activate();
    }
    return getAopProxyFactory().createAopProxy(this);
}

public class DefaultAopProxyFactory implements AopProxyFactory, Serializable {

    @Override
    public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
        /*
         * 下面的三个条件简单分析一下：
         *
         *   条件1：config.isOptimize() - 是否需要优化，这个属性没怎么用过，
         *         细节我不是很清楚
         *   条件2：config.isProxyTargetClass() - 检测 proxyTargetClass 的值，
         *         前面的代码会设置这个值
         *   条件3：hasNoUserSuppliedProxyInterfaces(config) 
         *         - 目标 bean 是否实现了接口
         */
        if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
            Class<?> targetClass = config.getTargetClass();
            if (targetClass == null) {
                throw new AopConfigException("TargetSource cannot determine target class: " +
                        "Either an interface or a target is required for proxy creation.");
            }
            if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
                return new JdkDynamicAopProxy(config);
            }
            // 创建 CGLIB 代理，ObjenesisCglibAopProxy 继承自 CglibAopProxy
            return new ObjenesisCglibAopProxy(config);
        }
        else {
            // 创建 JDK 动态代理
            return new JdkDynamicAopProxy(config);
        }
    }
}
```
其中`AopProxy `的类结构如下



如上，DefaultAopProxyFactory 根据一些条件决定生成什么类型的 AopProxy 实现类对象。生成好 AopProxy 实现类对象后，下面就要为目标 bean 创建代理对象了。这里以 JdkDynamicAopProxy 为例，我们来看一下，该类的 getProxy 方法的逻辑是怎样的。如下：

```java
public Object getProxy() {
    return getProxy(ClassUtils.getDefaultClassLoader());
}

public Object getProxy(ClassLoader classLoader) {
    if (logger.isDebugEnabled()) {
        logger.debug("Creating JDK dynamic proxy: target source is " + this.advised.getTargetSource());
    }
    Class<?>[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised, true);
    findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);
    
    // 调用 newProxyInstance 创建代理对象
    return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
}
```

如上，请把目光移至最后一行有效代码上，会发现 JdkDynamicAopProxy 最终调用 Proxy.newProxyInstance 方法创建代理对象。


本节，我来分析一下 JDK 动态代理逻辑。对于 JDK 动态代理，代理逻辑封装在 InvocationHandler 接口实现类的 invoke 方法中。JdkDynamicAopProxy 实现了 InvocationHandler 接口，下面我们就来分析一下 JdkDynamicAopProxy 的 invoke 方法。如下：

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    MethodInvocation invocation;
    Object oldProxy = null;
    boolean setProxyContext = false;

    TargetSource targetSource = this.advised.targetSource;
    Class<?> targetClass = null;
    Object target = null;

    try {
        // 省略部分代码
        Object retVal;

        // 如果 expose-proxy 属性为 true，则暴露代理对象
        if (this.advised.exposeProxy) {
            // 向 AopContext 中设置代理对象
            oldProxy = AopContext.setCurrentProxy(proxy);
            setProxyContext = true;
        }

        // 获取适合当前方法的拦截器
        List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

        // 如果拦截器链为空，则直接执行目标方法
        if (chain.isEmpty()) {
            Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
            // 通过反射执行目标方法
            retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
        }
        else {
            // 创建一个方法调用器，并将拦截器链传入其中
            invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
            // 执行拦截器链
            retVal = invocation.proceed();
        }

        // 获取方法返回值类型
        Class<?> returnType = method.getReturnType();
        if (retVal != null && retVal == target &&
                returnType != Object.class && returnType.isInstance(proxy) &&
                !RawTargetAccess.class.isAssignableFrom(method.getDeclaringClass())) {
            // 如果方法返回值为 this，即 return this; 则将代理对象 proxy 赋值给 retVal 
            retVal = proxy;
        }
        // 如果返回值类型为基础类型，比如 int，long 等，当返回值为 null，抛出异常
        else if (retVal == null && returnType != Void.TYPE && returnType.isPrimitive()) {
            throw new AopInvocationException(
                    "Null return value from advice does not match primitive return type for: " + method);
        }
        return retVal;
    }
    finally {
        if (target != null && !targetSource.isStatic()) {
            targetSource.releaseTarget(target);
        }
        if (setProxyContext) {
            AopContext.setCurrentProxy(oldProxy);
        }
    }
}
```
如上，上面的代码我做了比较详细的注释。下面我们来总结一下 invoke 方法的执行流程，如下：

1. 检测 expose-proxy 是否为 true，若为 true，则暴露代理对象
2. 获取适合当前方法的拦截器
3. 如果拦截器链为空，则直接通过反射执行目标方法
4. 若拦截器链不为空，则创建方法调用 ReflectiveMethodInvocation 对象
5. 调用 ReflectiveMethodInvocation 对象的 proceed() 方法启动拦截器链
6. 处理返回值，并返回该值

在以上6步中，我们重点关注第2步和第5步中的逻辑。第2步用于获取拦截器链，第5步则是启动拦截器链。下面先来分析获取拦截器链的过程。


### 获取所有的拦截器

所谓的拦截器，顾名思义，是指用于对目标方法的调用进行拦截的一种工具。拦截器的源码比较简单，所以我们直接看源码好了。下面以前置通知拦截器为例，如下：

```java
public class MethodBeforeAdviceInterceptor implements MethodInterceptor, Serializable {
    
    /** 前置通知 */
    private MethodBeforeAdvice advice;

    public MethodBeforeAdviceInterceptor(MethodBeforeAdvice advice) {
        Assert.notNull(advice, "Advice must not be null");
        this.advice = advice;
    }

    @Override
    public Object invoke(MethodInvocation mi) throws Throwable {
        // 执行前置通知逻辑
        this.advice.before(mi.getMethod(), mi.getArguments(), mi.getThis());
        // 通过 MethodInvocation 调用下一个拦截器，若所有拦截器均执行完，则调用目标方法
        return mi.proceed();
    }
}
```

如上，前置通知的逻辑在目标方法执行前被执行。这里先简单向大家介绍一下拦截器是什么，关于拦截器更多的描述将放在下一节中。本节我们先来看看如何如何获取拦截器，如下：

```java
public List<Object> getInterceptorsAndDynamicInterceptionAdvice(Method method, Class<?> targetClass) {
    MethodCacheKey cacheKey = new MethodCacheKey(method);
    // 从缓存中获取
    List<Object> cached = this.methodCache.get(cacheKey);
    // 缓存未命中，则进行下一步处理
    if (cached == null) {
        // 获取所有的拦截器
        cached = this.advisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice(
                this, method, targetClass);
        // 存入缓存
        this.methodCache.put(cacheKey, cached);
    }
    return cached;
}

public List<Object> getInterceptorsAndDynamicInterceptionAdvice(
        Advised config, Method method, Class<?> targetClass) {

    List<Object> interceptorList = new ArrayList<Object>(config.getAdvisors().length);
    Class<?> actualClass = (targetClass != null ? targetClass : method.getDeclaringClass());
    boolean hasIntroductions = hasMatchingIntroductions(config, actualClass);
    // registry 为 DefaultAdvisorAdapterRegistry 类型
    AdvisorAdapterRegistry registry = GlobalAdvisorAdapterRegistry.getInstance();

    // 遍历通知器列表
    for (Advisor advisor : config.getAdvisors()) {
        if (advisor instanceof PointcutAdvisor) {
            PointcutAdvisor pointcutAdvisor = (PointcutAdvisor) advisor;
            /*
             * 调用 ClassFilter 对 bean 类型进行匹配，无法匹配则说明当前通知器
             * 不适合应用在当前 bean 上
             */
            if (config.isPreFiltered() || pointcutAdvisor.getPointcut().getClassFilter().matches(actualClass)) {
                // 将 advisor 中的 advice 转成相应的拦截器
                MethodInterceptor[] interceptors = registry.getInterceptors(advisor);
                MethodMatcher mm = pointcutAdvisor.getPointcut().getMethodMatcher();
                // 通过方法匹配器对目标方法进行匹配
                if (MethodMatchers.matches(mm, method, actualClass, hasIntroductions)) {
                    // 若 isRuntime 返回 true，则表明 MethodMatcher 要在运行时做一些检测
                    if (mm.isRuntime()) {
                        for (MethodInterceptor interceptor : interceptors) {
                            interceptorList.add(new InterceptorAndDynamicMethodMatcher(interceptor, mm));
                        }
                    }
                    else {
                        interceptorList.addAll(Arrays.asList(interceptors));
                    }
                }
            }
        }
        else if (advisor instanceof IntroductionAdvisor) {
            IntroductionAdvisor ia = (IntroductionAdvisor) advisor;
            // IntroductionAdvisor 类型的通知器，仅需进行类级别的匹配即可
            if (config.isPreFiltered() || ia.getClassFilter().matches(actualClass)) {
                Interceptor[] interceptors = registry.getInterceptors(advisor);
                interceptorList.addAll(Arrays.asList(interceptors));
            }
        }
        else {
            Interceptor[] interceptors = registry.getInterceptors(advisor);
            interceptorList.addAll(Arrays.asList(interceptors));
        }
    }

    return interceptorList;
}

public MethodInterceptor[] getInterceptors(Advisor advisor) throws UnknownAdviceTypeException {
    List<MethodInterceptor> interceptors = new ArrayList<MethodInterceptor>(3);
    Advice advice = advisor.getAdvice();
    /*
     * 若 advice 是 MethodInterceptor 类型的，直接添加到 interceptors 中即可。
     * 比如 AspectJAfterAdvice 就实现了 MethodInterceptor 接口
     */
    if (advice instanceof MethodInterceptor) {
        interceptors.add((MethodInterceptor) advice);
    }

    /*
     * 对于 AspectJMethodBeforeAdvice 等类型的通知，由于没有实现 MethodInterceptor 
     * 接口，所以这里需要通过适配器进行转换
     */ 
    for (AdvisorAdapter adapter : this.adapters) {
        if (adapter.supportsAdvice(advice)) {
            interceptors.add(adapter.getInterceptor(advisor));
        }
    }
    if (interceptors.isEmpty()) {
        throw new UnknownAdviceTypeException(advisor.getAdvice());
    }
    return interceptors.toArray(new MethodInterceptor[interceptors.size()]);
}
```
以上就是获取拦截器的过程，代码有点长，不过好在逻辑不是很复杂。这里简单总结一下以上源码的执行过程，如下：

1. 从缓存中获取当前方法的拦截器链
2. 若缓存未命中，则调用 getInterceptorsAndDynamicInterceptionAdvice 获取拦截器链
3. 遍历通知器列表
4. 对于 PointcutAdvisor 类型的通知器，这里要调用通知器所持有的切点（Pointcut）对类和方法进行匹配，匹配成功说明应向当前方法织入通知逻辑
5. 调用 getInterceptors 方法对非 MethodInterceptor 类型的通知进行转换
6. 返回拦截器数组，并在随后存入缓存中

### 执行拦截器链

本节的开始，我们先来说说 ReflectiveMethodInvocation。ReflectiveMethodInvocation 贯穿于拦截器链执行的始终，可以说是核心。该类的 proceed 方法用于启动启动拦截器链，下面我们去看看这个方法的逻辑。

```java
public class ReflectiveMethodInvocation implements ProxyMethodInvocation {

    private int currentInterceptorIndex = -1;

    public Object proceed() throws Throwable {
        // 拦截器链中的最后一个拦截器执行完后，即可执行目标方法
        if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
            // 执行目标方法
            return invokeJoinpoint();
        }

        Object interceptorOrInterceptionAdvice =
                this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
        if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
            InterceptorAndDynamicMethodMatcher dm =
                    (InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
            /*
             * 调用具有三个参数（3-args）的 matches 方法动态匹配目标方法，
             * 两个参数（2-args）的 matches 方法用于静态匹配
             */
            if (dm.methodMatcher.matches(this.method, this.targetClass, this.arguments)) {
                // 调用拦截器逻辑
                return dm.interceptor.invoke(this);
            }
            else {
                // 如果匹配失败，则忽略当前的拦截器
                return proceed();
            }
        }
        else {
            // 调用拦截器逻辑，并传递 ReflectiveMethodInvocation 对象
            return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
        }
    }
}
```

如上，proceed 根据 currentInterceptorIndex 来确定当前应执行哪个拦截器，并在调用拦截器的 invoke 方法时，将自己作为参数传给该方法。前面的章节中，我们看过了前置拦截器的源码，这里来看一下后置拦截器源码。如下：

```java
public class AspectJAfterAdvice extends AbstractAspectJAdvice
        implements MethodInterceptor, AfterAdvice, Serializable {

    public AspectJAfterAdvice(
            Method aspectJBeforeAdviceMethod, AspectJExpressionPointcut pointcut, AspectInstanceFactory aif) {

        super(aspectJBeforeAdviceMethod, pointcut, aif);
    }


    @Override
    public Object invoke(MethodInvocation mi) throws Throwable {
        try {
            // 调用 proceed
            return mi.proceed();
        }
        finally {
            // 调用后置通知逻辑
            invokeAdviceMethod(getJoinPointMatch(), null, null);
        }
    }

    //...
}
```
如上，由于后置通知需要在目标方法返回后执行，所以 AspectJAfterAdvice 先调用 mi.proceed() 执行下一个拦截器逻辑，等下一个拦截器返回后，再执行后置通知逻辑。如果大家不太理解的话，先看个图。这里假设目标方法 method 在执行前，需要执行两个前置通知和一个后置通知。下面我们看一下由三个拦截器组成的拦截器链是如何执行的，如下：



这边不好理解，假如当前获取到了`before`,`after`,`around`三个`advice`，
1. 首先第一个获取到的是`after`，他会去执行`after`的`invoke`方法，
```java
try {
    // 调用 proceed
    return mi.proceed();
}
finally {
    // 调用后置通知逻辑
    invokeAdviceMethod(getJoinPointMatch(), null, null);
}
```
他会先回调`processd`方法，最后在finally中执行`after`的切入方法，这个方式是执行链执行后完成
2. 第一次拿到的是before
他会在执行before的通知方法，在执行processed方法
3. 拿到around

这边得细品

获取到的拦截器是有顺序的
1. AfterThrowing
2. AfterReturn
3. After
4. Around
5. Before

所以执行顺序是
1. AroundBefore
2. Before
3. 目标方法
4. ArdoudAfter
5. After
6. AfterReturn
7. AfterThrowing

### 执行目标方法

```java
protected Object invokeJoinpoint() throws Throwable {
    return AopUtils.invokeJoinpointUsingReflection(this.target, this.method, this.arguments);
}

public abstract class AopUtils {
    public static Object invokeJoinpointUsingReflection(Object target, Method method, Object[] args)
            throws Throwable {

        try {
            ReflectionUtils.makeAccessible(method);
            // 通过反射执行目标方法
            return method.invoke(target, args);
        }
        catch (InvocationTargetException ex) {...}
        catch (IllegalArgumentException ex) {...}
        catch (IllegalAccessException ex) {...}
    }
}
```
