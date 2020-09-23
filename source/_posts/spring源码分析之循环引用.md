---
title: spring源码分析-循环引用
date: 2020-09-22
cover: https://cdn.jsdelivr.net/gh/dtlexi/lexi.blog/src/image/spring源码分析之循环引用/cover.jpg
cover: https://cdn.jsdelivr.net/gh/dtlexi/lexi.blog/src/image/spring源码分析之循环引用/top.jpg
categories:
 - java
 - spring
tags:
 - java
 - spring
 - 源码分析
 - spring 源码分析
---



本文，我们来看一下 spring 是如何解决循环依赖问题的。在本篇文章中，首先我会向大家介绍一下什么是循环依赖，然后会从`getBean(String)`方法开始，梳理一下bean的什么周期，给大家分析一下spring的循环依赖问题的可能解法，最后在通过spirng的源码分析一下spring如何解决循环周期

spring循环依赖在面试中是一个很高频的问题，循环依赖相关的源码，通过三个map解决问题。可是真正要搞懂循环依赖确不简单，需要大量前置知识和对spring知识有深入了解，否则大家对spring解决循环依赖的了解也仅限已三个map而已



## 什么是循环依赖



所谓的循环依赖是指，A 依赖 B，B 又依赖 A，它们之间形成了循环依赖。或者是 A 依赖 B，B 依赖 C，C 又依赖 A。它们之间的依赖关系如下：




spring循环引用又分为属性循环引用和构造方法循环引用，他们的代码实现关系如下

### 构造方法循环引用

```java
@Component
public class A {
   public A(B b)
   {}
}
```
```java
@Component
public class B {
   public B(A a)
   {}
}
```
构造方法循环引用在实例化对象A时会使用到对象B，需要先创建对象B，但是创建对象B时又需要先创建对象A，这就陷入了一个死循环。
 spring对这种循环引用是无能为力的，会直接报错。本文讨论的循环引用指下面的属性循环引用

### 属性循环引用
```java
@Component
public class A {
   @Autowired
   private B b;
}
```

```java
@Component
public class B {
   @Autowired
   private A a;
}
```

## 解决循环依赖

### 模拟spring基本功能
为了让大家更加透彻的了解循环引用，接下来我将自己模拟一个Spring，并且尝试一下解决循环依赖问题,
首先我们定义一个bean工厂`BeanFactory`类
```java
public class BeanFactory {
}
```
spring容器的核心就是一个map，下面为自己的bean工厂类添加一个map容器
```java
private HashMap<String,Object> singletonObjects;
```
下面来为bean工厂提供一个创建对象方法
```java
// 创建对象
private  <T> T createBean(String beanName,Class<T> beanCls) throws Exception {
    Constructor constructor=beanCls.getDeclaredConstructor();
    T bean= (T)constructor.newInstance();
    this.populateBean(bean);
    return bean;
}

// 属性赋值
private void populateBean(Object obj) throws Exception {
    Class beanCla=obj.getClass();
    Field[] fields=beanCla.getDeclaredFields();

    for (Field field:
            fields) {
        Class fieldType=field.getType();
        Object fieldBean=getBean(fieldType);
        field.set(obj,fieldBean);
    }
}
```
最后我们在为bean工厂提供getBean方法

```java

public  <T> T getBean(Class<T> cls) throws Exception {
    String beanName=cls.toString();
    Object singletonObject=getSingleton(beanName);
    if(singletonObject!=null)
    {
        return (T)singletonObject;
    }
    singletonObject= createBean(beanName,cls);
    this.addSingleton(beanName,singletonObject);
    return (T) singletonObject;
}
private void addSingleton(String beanName, Object singletonObject) {
    this.singletonObjects.put(beanName, singletonObject);
}
```

接下来我们定义一个Bean来测试一下基本功能

```java

public static void main(String[] args) throws Exception {
   BeanFactory beanFactory=new BeanFactory();
   System.out.println(beanFactory.getBean(A.class));
}
```

下面我们来看一下运行结果

```

com.lexi.service.A@7ea987ac



Process finished with exit code 0

```



### 模拟循环引用



从上面的结果可以看出，我们自己的bean工厂类已经能生产对象，并且给属性赋值，但是如果遇到属性循环引用是会出现什么结果？我们来看看

创建对象A，和对象B，如下:

```java

public class A {
   public B b;
}
public class B {
   public A a;
}
```

运行一下上面代码,不出意外的报错了
```
/Library/Java/JavaVirtualMachines/jdk1.8.0_101.jdk/Contents/Home/bin/java 
Exception in thread "main" java.lang.StackOverflowError
    at java.lang.Class.privateGetDeclaredFields(Class.java:2577)
    at java.lang.Class.getDeclaredFields(Class.java:1916)
    at com.lexi.factory.BeanFactory1.populateBean(BeanFactory.java:26)
    at com.lexi.factory.BeanFactory1.createBean(BeanFactory.java:19)
    at com.lexi.factory.BeanFactory1.getBean(BeanFactory.java:56)
    at com.lexi.factory.BeanFactory1.populateBean(BeanFactory.java:31)
    ...
```
接下来尝试画一下程序执行流程图，如下：

从上面的流程图我们可以看出`createBean(a)`,`popilateBean(b)`,`createBean(B)`,`popilateBean(a)`,`createBean(a)`,`popilateBean(b)`...的循环中，从而导致`StackOverflowError`。

下面我们来尝试解决一下当前问题，我们试着加入一个早期缓存`earlySingletonObjects`,用来存放刚刚创建,还没有完成生命周期的早期bean对象。

修改相关代码如下：
```java
public class BeanFactory {
   private HashMap<String,Object> singletonObjects;
   private HashMap<String,Object> earlySingletonObjects;

   public BeanFactory()
   {
      this.singletonObjects=new HashMap<String, Object>();
      this.earlySingletonObjects=new HashMap<String, Object>();
   }

   // 创建对象
   private  <T> T createBean(String beanName,Class<T> beanCls) throws Exception {
      Constructor constructor=beanCls.getDeclaredConstructor();
      T bean= (T)constructor.newInstance();
      this.earlySingletonObjects.put(beanName,bean);
      this.populateBean(bean);
      return bean;
   }

   // 属性赋值
   private void populateBean(Object obj) throws Exception {
      Class beanCla=obj.getClass();
      Field[] fields=beanCla.getDeclaredFields();

      for (Field field:
          fields) {
         Class typeField=field.getType();
         Object fieldBean=getBean(typeField);
         field.set(obj,fieldBean);
      }
   }

   public Object getSingleton(String beanName)
   {
      Object obj=this.singletonObjects.get(beanName);
      if(obj==null)
      {
         obj=this.earlySingletonObjects.get(beanName);
      }
      return  obj;
   }

   //获取对象
   public  <T> T getBean(Class<T> cls) throws Exception {
      String beanName=cls.toString();
      Object singletonObject=getSingleton(beanName);
      if(singletonObject!=null)
      {
         return (T)singletonObject;
      }
      singletonObject= createBean(beanName,cls);
      this.addSingleton(beanName,singletonObject);
      return (T) singletonObject;
   }
   
   private void addSingleton(String beanName, Object singletonObject) {
      this.singletonObjects.put(beanName, singletonObject);
      if(this.earlySingletonObjects.containsKey(beanName))
      {
         this.earlySingletonObjects.remove(beanName);
      }
   }
}
```
上面的代码在`singletonObjects`的基础上，引入二级缓存`earlySingletonObjects`。将刚刚创建的bean对象存入其中，`getSingleton`时，如果一级缓存中没有，再到二级缓存中获取，这样就避免了重复创建对象。
运行一下测试代码。
```java
public static void main(String[] args) throws Exception {
   BeanFactory beanFactory=new BeanFactory();
   System.out.println(beanFactory.getBean(B.class));
   System.out.println(beanFactory.getBean(A.class).b);
   System.out.println("");
   System.out.println(beanFactory.getBean(A.class));
   System.out.println(beanFactory.getBean(B.class).a);
}
```
运行结果：
```
/Library/Java/JavaVirtualMachines/jdk1.8.0_101.jdk/Contents/Home/bin/java
com.lexi.service.B@7ea987ac
com.lexi.service.B@7ea987ac

com.lexi.service.A@12a3a380
com.lexi.service.A@12a3a380

Process finished with exit code 0
```
上面的代码貌似解决了循环引用的问题，那么spring为什么要使用三级缓存呢，二级缓存就能解决的问题，为什么要使用三级缓存？难道是spring多此一举了？我们带着这个问题进一步研究

### 模拟spring aop

aop是spring除了ioc之外提供的另外一个强大的功能，下面我们模拟一下aop
在项目中引入`CglibProxyFactory`类，来产生CGLib代理，代码如下：
```java
public class CglibProxyFactory implements MethodInterceptor {

    private Object target;
    public CglibProxyFactory(Object obj)
    {
        this.target=obj;
    }

    public Object createProxyInstance()
    {
        Enhancer enhancer=new Enhancer();
        enhancer.setSuperclass(target.getClass());
        enhancer.setCallback(this);
        return  enhancer.create();
    }

    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
        Object result = method.invoke(target, objects);
        return result;
    }
}
```
在项目中添加`ProxyCreator`类，提供`wrapInstance`方法，用来创建代理
```java
public class ProxyCreator {
    public Object wrapInstance(String beanName,Object bean)
    {
        CglibProxyFactory proxyFactory=new CglibProxyFactory(bean);
        return proxyFactory.createProxyInstance();
    }
}
```
修改`BeanFactory`的`createBean`方法，在属性赋值完成后完成代理
```java
private  <T> T createBean(String beanName,Class<T> beanCls) throws Exception {
   Constructor constructor=beanCls.getDeclaredConstructor();
   T bean= (T)constructor.newInstance();

   earlySingletonObjects.put(beanName,bean);

   this.populateBean(bean);

   // 判断当前是否需要aop,我们这边写死
   boolean warpIfNecessary=true;
   if(warpIfNecessary)
   {
      bean= (T) this.proxyCreator.wrapInstance(bean);
   }

   return bean;
}
```
运行一下测试代码：
```java
public static void main(String[] args) throws Exception {
      BeanFactory beanFactory=new BeanFactory();

      System.out.println(beanFactory.getBean(A.class));
      System.out.println(beanFactory.getBean(B.class).getA());
      System.out.println();
      System.out.println(beanFactory.getBean(B.class));
      System.out.println(beanFactory.getBean(A.class).getB());
   }
```
```
"C:\Program Files\Java\jdk1.8.0_161\bin\java.exe" 
com.lexi.service.A@497470ed
com.lexi.service.A@497470ed

com.lexi.service.B@63c12fb0
com.lexi.service.B@63c12fb0

Process finished with exit code 0
```
上面的代码看上去没问题，但是熟悉CGlib的同学还是能够发现问题的，此时打印的对象不是CGLib产生的对象，这是什么原因导致的呢？刚开始我也有点蒙，后来才发现是`toString()`被Cglib代理了，执行的是原始对象的`toString()`

知道什么问题就好解决了，修改一下`CglibProxyFactory`的`createProxyInstance`，添加`callbackFilter`。什么？什么是`callbackFilter`，CGLib动态代理.md
```java
public Object createProxyInstance()
{
    Enhancer enhancer=new Enhancer();
    enhancer.setSuperclass(target.getClass());
    enhancer.setCallbackFilter(new MyCallFilter());
    enhancer.setCallbacks(new Callback[]{
            NoOp.INSTANCE,
            this
    });
    return  enhancer.create();
}
```

下面我们再来看一下运行结果：
```
"C:\Program Files\Java\jdk1.8.0_161\bin\java.exe" "-javaagent:C:\Program Files\JetBrains\IntelliJ 
com.lexi.service.A$$EnhancerByCGLIB$$a9ea6b86@6477463f
com.lexi.service.A@6477463f

com.lexi.service.B$$EnhancerByCGLIB$$71ff370b@3d71d552
com.lexi.service.B$$EnhancerByCGLIB$$71ff370b@3d71d552
```

其中`beanFactory.getBean(A.class)`和`beanFactory.getBean(B.class).getA()`出现了不同的结果，这是什么原因导致的呢？

`beanFactory.getBean(A.class)`获取的是`singletonObjects`中的A对象，是完整的对象。
`beanFactory.getBean(B.class).getA()`获取的是B对象中的a，如下图所示，B.a 赋值发生在`warp(a)`之前,此时a对象还没有添加代理，是原始的a对象
流程图如下：



找到问题了，那么该如何解决问题呢?将`wrapInstance`提升到`earlySingletonObjects.put(beanName,bean)`之前？
这样看上去行得通，可是此时对象刚创建完成，对象还没有完全初始化，此时修改对象，可能新的对象不是我们想要的对象。而且如果将`wrapInstance`提前到，那么之后的所有操作都是针对于`warpInstance`。会导致不可预料的问题。
那么我们该如何解决这个问题呢？这是后第三级缓存`singletonFactories`就派上用场了。

先添加一个`ObjectFactory`接口，用来获取前期引用对象
```java
public interface ObjectFactory<T> {
    T getObject() throws Exception;
}
```

在`BeanFactory`中添加第三级缓存`singletonFactories`
```java
private final Map<String, ObjectFactory<?>> singletonFactories;
```

在`ProxyCreator`添加暴露早期引用方法，用来提前暴露代理对象引用。并且引入早期引用缓存`earlyProxyReferences`
```java
private final Map<String, Object> earlyProxyReferences;
public Object getEarlyBeanReference(String beanName,Object bean)
{
    CglibProxyFactory proxyFactory=new CglibProxyFactory(bean);
    Object warpBean= proxyFactory.createProxyInstance();
    earlyProxyReferences.put(beanName,warpBean);
    return warpBean;
}
```
修改`wrapInstance`方法
```java
public Object wrapInstance(String beanName,Object bean)
{
    if(this.earlyProxyReferences.containsKey(beanName))
    {
        return this.earlyProxyReferences.get(beanName);
    }

    CglibProxyFactory proxyFactory=new CglibProxyFactory(bean);
    return proxyFactory.createProxyInstance();
}
```
最后修改`BeanFactory`的`createBean`方法，对象创建完成后，将对象的早期引用方法存储进`singletonFactories`,并且修改`getSingleton`方法，如果`earlySingletonObjects`获取不到对象，到三级缓存`singletonFactories`获取对象。
```java
// 创建对象
private  <T> T createBean(String beanName,Class<T> beanCls) throws Exception {
    Constructor constructor=beanCls.getDeclaredConstructor();
    final T bean= (T)constructor.newInstance();

    // 是否暴露早起引用，spring这边判断的是
    // 1. 当前对象是否是单例
    // 2. 当前对象是否正在创建中，
    // 3. 是否支持循环引用
    // 我们这边就直接简单的都默认为true
    boolean earlySingletonExposure = true;

    if(earlySingletonExposure)
    {
        // addSingletonFactory(beanName,ObjectFactory);
        addSingletonFactory(beanName,()->{
            return this.proxyCreator.getEarlyBeanReference(beanName,bean);
        });
    }
    this.populateBean(bean);

    T exposedObject= (T) this.proxyCreator.wrapInstance(beanName,bean);

    return exposedObject;
}
```
```java
private Object getSingleton(String beanName) throws Exception {
    Object singletonObject=this.singletonObjects.get(beanName);
    if(singletonObject==null)
    {
        singletonObject = this.earlySingletonObjects.get(beanName);
        if(singletonObject==null)
        {
            ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
            if (singletonFactory != null) {
                singletonObject = singletonFactory.getObject();
                this.earlySingletonObjects.put(beanName, singletonObject);
                this.singletonFactories.remove(beanName);
            }
        }
    }
    return  singletonObject;
}
```
测试代码：
```java
public static void main(String[] args) throws Exception {
      BeanFactory beanFactory=new BeanFactory();
      System.out.println(beanFactory.getBean(A.class));
      System.out.println(beanFactory.getBean(B.class).getA());
}
```
```
"C:\Program Files\Java\jdk1.8.0_161\bin\java.exe" "-javaagent:C:\Program Files\JetBrains\IntelliJ IDEA 2020.1.2\lib\idea_rt.jar=50861:C:\Program Files\JetBrains\IntelliJ IDEA 2020.1.2\bin" 
com.lexi.service.A$$EnhancerByCGLIB$$438ef93a@123772c4
com.lexi.service.A$$EnhancerByCGLIB$$438ef93a@123772c4

Process finished with exit code 0
```
下面我们来分析一下上面代码
1. 我们首先获取对象A`getBean(A)`，此时对象A还没有实例化，三个缓存都为空，所以此时`getSingleton(A)`返回null。
2. 进入常见对象A流程`createBean(A)`。获取默认构造方法，调用默认构造方法创建对象。
3. 在第三级缓存`singletonFactories`中加入获取A对象早期引用的ObjectFactory对象，注意此时加入的是ObjectFactory对象，不是A对象
4. 给A对象属性赋值
    1. 赋值A.b，调用getBean(B)获取B对象，此时B对象还没有实例化，加入`createBean(B)`
    2. 获取B的默认构造方法，创建B对象
    3. 在第三级缓存`singletonFactories`中加入获取B对象早期引用的ObjectFactory对象
    4. 给B属性赋值
        1. 赋值B.a，此时第三级缓存中已经存在A的早期引用`ObjectFactory`，获取`ObjectFactory`，调用其`get()`方法参数A对象，并且将其方法早期引用`earlySingletonObjects`,这边为什么还有存在`earlySingletonObjects`？
            因为此时如果存在对象C，C中有A，A中有C，那么给C.a赋值时，如果没有`earlySingletonObjects`，那么此时会继续调用`ObjectFactory,get()`，产生一个新的对象。
    5. Warp(B)，获取B的代理对象，并且添加到缓存中去，返回代理对象
5. 获取A对像的代理对象，此时已经在`ProxyCreator`的`earlyProxyReferences`存在早期引用，直接返回早期引用

## Spring解决循环依赖
spring解决循环依赖的方案其实和我们差不多（嘿嘿，我就是照着spring抄袭的）

```java
@Nullable
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
   // 从容器当中获取bean
   // 第一次调用这个方法时，singletonObject肯定为空

   // 第二次调用：
   //     循环引用是：为空
   Object singletonObject = this.singletonObjects.get(beanName);
   // 如果bean==null && bean正在创建中
   // isSingletonCurrentlyInCreation 是判断当前对象是否正在创建中
   // 其主要是判断this.singletonsCurrentlyInCreation对象中是否包含beanName,
   // 第一次调用是肯定不存在
   // 所以第一次调用这个方法时肯定不会进入这个if
   // 第二次调用时：
   //     循环引用：会进入这个if
   if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
      synchronized (this.singletonObjects) {
         /**
          * spring解决循环依赖用了三个map。
          * 有人可能会好奇明明俩个map就能解决的事为啥非得用三个map来解决呢？
          * 其实这边三个map都是有他的用处的
          *
          * 第一个map singletonObjects 用来放完全体的Bean，
          * 这边的完全体指实例化出来，完成属性赋值，有代理的完成了代理可以直接使用的对象
          *
          * 第二个map earlySingletonObjects 用来存放已经实例化，但还没有可以直接使用的对象
          *
          * 第三个map singletonFactories 用来存放对象工厂，这个对象工厂不是生产对象，而是对已经存在的对象进行加工，
          * 这个比较抽象，举个例子：
          * class A{
          *        @Autowired
          *        private B b;
          * }
          * class B{
          *        @Autowired
          *        private A a;
          * }
          * 循环依赖在属性填充时，a,b都需要添加代理，那么A.b和B.a都应该是代理后的对象
          * 但是spring aop 添加代理是在属性填充完成之后
          * 此时工厂的作用就是生产一个代理对象，详见AbstractAutoProxyCreator
          *
          * 有人可能会问为什么不直接使用 singletonObjects 和 singletonFactories
          * 为什么还要再加一个 earlySingletonObjects？
          * 如果不适用earlySingletonObjects有俩种解决方法：我们来分析一下是否可行
          *     1.     每次都调用singletonFactories.get(),此时会重新生产一个代理对象
          *        这种方案明显行不通，每次重新get一个相当于每次都执行 getEarlyBeanReference，重新产生一个代理对象，效率低
          * 为什么不直接将aop提到populateBean之前
          * 我哪知道，问spring去
          */
         singletonObject = this.earlySingletonObjects.get(beanName);
         if (singletonObject == null && allowEarlyReference) {
            // this.singletonFactories.put is in addSingletonFactory
            // AbstractAutowireCapableBeanFactory getEarlyBeanReference
            // 这边的 singletonFactory.getObject 不是new 对象，而是处理对象 具体是执行
            ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
            if (singletonFactory != null) {
               singletonObject = singletonFactory.getObject();
               this.earlySingletonObjects.put(beanName, singletonObject);
               this.singletonFactories.remove(beanName);
            }
         }
      }
   }
   // 第一次调用这个方法是，肯定返回null
   return singletonObject;
}
```
上面就是spring解决循环依赖的源码，代码很简单，和我们自己写的（抄袭）的`getSingleton`方法几乎一模一样，我想现在大家都应该能看清楚这段代码了吧。
spring解决循环依赖其实很简单，难就难在大家不清楚spring创建和初始化对象的流程，只要清楚spring初始化对象的流程，我想大部分人还是能看懂这段代码的。
哈哈，这篇文章就到这边了