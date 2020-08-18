---
title: Spring扩展之CGLib应用
date: 2020-08-18 00:00:00
cover: https://cdn.jsdelivr.net/gh/dtlexi/lexi.blog/src/image/17.jpg
top_img: https://cdn.jsdelivr.net/gh/dtlexi/lexi.blog/src/image/18.jpg
categories:
 - java
 - cglib
tags:
 - java
 - cglib
---


Spring中大量使用的CGLib实现代理，为了方便大家理解源码，在此写一篇关于CGLib使用的博客。

CGLib和JDK动态代理都是动态代理的实现，和JDK动态代理要求被代理对象必须实现一个接口不同的是CGLib允许对象是一个单独的对象。



Cglib代理,也叫作子类代理,它是在内存中构建一个子类对象从而实现对目标对象功能的扩展.



CGLib依赖如下

```xml
<dependencies>
    <!-- https://mvnrepository.com/artifact/cglib/cglib -->
    <dependency>
        <groupId>cglib</groupId>
        <artifactId>cglib</artifactId>
        <version>3.3.0</version>
    </dependency>
</dependencies>
```



### CGLib基本使用



HelloService.cs

```java
public class HelloService {

    public void sayHello() {
        System.out.println("Hello Java");
    }
}
```



ProxyFactory.cs



```java
public class ProxyFactory implements MethodInterceptor {
    private Object target;
    public ProxyFactory(Object target)
    {
        this.target=target;
    }

    public Object createInstance()
    {
       Enhancer enhancer=new Enhancer();
       enhancer.setCallback(this);
       enhancer.setSuperclass(target.getClass());
       return enhancer.create();
    }

    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
        System.out.println("before");
        Object returnValue=method.invoke(target,objects);
        System.out.println("after");

        return  returnValue;
    }
}
```



Main方法

```java
HelloService helloService= (HelloService) new ProxyFactory(new HelloService()).createInstance();
helloService.sayHello();
```



### invoke和invokeSuper



这边大家思考一个问题，有如下情况，在一个类中，有方法1和方法2中，如果在方法2中调用方法1，此时会调用方法1的代理逻辑吗？



#### Invoke



TestService.cs

```java
public class TestService {
    public void m1()
    {
        System.out.println("m1");
    }

    public void m2()
    {
        this.m1();
        System.out.println("m2");
    }
}
```



ProxyFactory.cs

```java
public class ProxyFactory implements MethodInterceptor {
    private Object target;
    public ProxyFactory(Object target)
    {
        this.target=target;
    }

    public Object createInstance()
    {
       Enhancer enhancer=new Enhancer();
       enhancer.setCallback(this);
       enhancer.setSuperclass(target.getClass());
       return enhancer.create();
    }

    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
        System.out.println("before");
        // 这边是target对象
        // 调用对象的方法
        // method 是目标方法
        Object returnValue=method.invoke(target,objects);
        System.out.println("after");

        return  returnValue;
    }
}
```



Main方法

```java
TestService testService=new TestService();
TestService proxyService= (TestService) new ProxyFactory(testService).createInstance();
proxyService.m2();
```



执行结果



```
before
m1
m2
after
```



#### InvokeSuper



`invokeSuper` 和上面基本上差不多，主要对`ProxyFactory` 的`intercept` 进行简单的修改



```java
// o指代理对象
    // method 指目标对象方法
    // methodProxy 代理方法
    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
        System.out.println("before");
        // 调用代理对象的代理方法
        Object returnValue=methodProxy.invokeSuper(o,objects);
        System.out.println("after");

        return  returnValue;
    }
```



运行结果



```
before
before
m1
after
m2
after
```





### CallbackFilter

CallbackFilter可以实现不同的方法使用不同的回调方法，在Spring中`@Configuration` `@Lazy` `@Lookup` 等使用了该特性



HelloService.cs



```java
public class HelloService {
    public void sayHello()
    {
        System.out.println("hello");
    }

    public void sayGoodbye()
    {
        System.out.println("goodbye");
    }

    public void say()
    {
        System.out.println("say");
    }
}
```



CallbackFilterImpl.cs



```java
public class CallbackFilterImpl implements CallbackFilter {
    @Override
    public int accept(Method method) {
        if(method.getName().equals("sayHello"))
        {
            // 返回1 代表调用第2个callback
            // index从0开始
            return 1;
        }
        if(method.getName().equals("sayGoodbye"))
        {
            return 2;
        }
        return 0;
    }
}
```





SayHelloInvokeImpl.cs



```java
public class SayHelloInvokeImpl implements MethodInterceptor {
    @Override
    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
        System.out.println("SayHelloInvokeImpl");
        return methodProxy.invokeSuper(o,objects);
    }
}
```



SayGoodbyeInvokeImpl.cs



```java
public class SayGoodbyeInvokeImpl implements MethodInterceptor {
    @Override
    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
        System.out.println("SayGoodbyeInvokeImpl");
        return methodProxy.invokeSuper(o,objects);
    }
}
```



Main方法



```java
public static void main(String[] args) {
    Enhancer enhancer = new Enhancer();
    enhancer.setSuperclass(HelloService.class);
    // NoOp这边指空的callback
    enhancer.setCallbacks(new Callback[]{NoOp.INSTANCE, new SayHelloInvokeImpl(),new SayGoodbyeInvokeImpl()});
    enhancer.setCallbackFilter(new CallbackFilterImpl());
    HelloService helloService= (HelloService) enhancer.create();
    helloService.sayHello();
    helloService.sayGoodbye();
    helloService.say();
}
```



执行结果：

```
SayHelloInvokeImpl

hello

SayGoodbyeInvokeImpl

goodbye

say
```





### Mixin

这是一种将多个接口混合在一起的方式, 实现了多个接口

这种方式是一种多继承的替代方案, 很大程度上解决了多继承的很多问题, 实现和理解起来都比较易



```java
Class[] interfaces={IHelloService.class, IHelloService2.class};
Object[] objects={new HelloService(),new HelloService2()};

Object o = Mixin.create(interfaces, objects);

IHelloService helloService=(IHelloService)o;
helloService.sayHello();

IHelloService2 helloService2=(IHelloService2)o;
helloService2.sayHello2();
```

