---
title: Spring源码分析-BeanDefintion
date: 2020-06-30 00:00:00
cover: https://ss2.bdstatic.com/70cFvnSh_Q1YnxGkpoWK1HF6hhy/it/u=3827979352,1501043192&fm=26&gp=0.jpg
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


## 一、BeanDefinition源码解析


### 常量

```java
// 单例
String SCOPE_SINGLETON = ConfigurableBeanFactory.SCOPE_SINGLETON;

// 原型
String SCOPE_PROTOTYPE = ConfigurableBeanFactory.SCOPE_PROTOTYPE;
```

### 方法/属性

#### 1. parentName
parentName在AbstractBeanDefinition没有实现，而是在各个子类中实现的。

BeanDefinition
```java
void setParentName(@Nullable String parentName);

@Nullable
String getParentName();
```
ChildBeanDefinition
ChildBeanDefinition在实例化对象时，必须传入ParentName
```java
@Nullable
private String parentName;
public ChildBeanDefinition(String parentName) {
   super();
   this.parentName = parentName;
}
@Override
public void setParentName(@Nullable String parentName) {
   this.parentName = parentName;
}

@Override
@Nullable
public String getParentName() {
   return this.parentName;
}
```
RootBeanDefinition
RootBeanDefinition无法设置parentName
```java
@Override
public String getParentName() {
   return null;
}

@Override
public void setParentName(@Nullable String parentName) {
   if (parentName != null) {
      throw new IllegalArgumentException("Root bean cannot be changed into a child bean with parent reference");
   }
}
```
GenericBeanDefinition
GenericBeanDefinition是Spring2.5之后新加入的一个类，GenericBeanDefinition类似于RootBeanDefinition和ChildBeanDefinition的结合体，它既可以表示"父类"模板也可以表示"子类"对象
RootBeanDefinition和ChildBeanDefinition是Spring 2.5之前的，ChildBeanDefinition必须要有parentName，RootBeanDefinition不能设置parentName

```java
@Nullable
private String parentName;
@Override
public void setParentName(@Nullable String parentName) {
   this.parentName = parentName;
}

@Override
@Nullable
public String getParentName() {
   return this.parentName;
}
```

#### 2. BeanClassName
类全名

```java
void setBeanClassName(@Nullable String beanClassName);

@Nullable
String getBeanClassName();
```
setBeanClassName和getBeanClassName是在BeanDefinition的字了AbstractBeanDefinition中实现的
```java
@Nullable
private volatile Object beanClass;

@Override
public void setBeanClassName(@Nullable String beanClassName) {
   this.beanClass = beanClassName;
}

@Override
@Nullable
public String getBeanClassName() {
   Object beanClassObject = this.beanClass;
   if (beanClassObject instanceof Class) {
      return ((Class<?>) beanClassObject).getName();
   }
   else {
      return (String) beanClassObject;
   }
}

public void setBeanClass(@Nullable Class<?> beanClass) {
   this.beanClass = beanClass;
}

public Class<?> getBeanClass() throws IllegalStateException {
   Object beanClassObject = this.beanClass;
   if (beanClassObject == null) {
      throw new IllegalStateException("No bean class specified on bean definition");
   }
   if (!(beanClassObject instanceof Class)) {
      throw new IllegalStateException(
            "Bean class name [" + beanClassObject + "] has not been resolved into an actual Class");
   }
   return (Class<?>) beanClassObject;
}
```
这边不管是beanClassName还是beanClass，都是通过`beanClass`对象实现的



#### 3. scope

设置bean作用域

使用
```java
<bean id="testConstArgService" class="com.lexi.service.TestConstArgService" scope="singleton">
</bean>
```
或者
```java
@Scope(BeanDefinition.SCOPE_SINGLETON)
```
BeanDefinition
```java
void setScope(@Nullable String scope);

@Nullable
String getScope();
```
实现 AbstractBeanDefinition
```java
@Nullable
private String scope = SCOPE_DEFAULT;

@Override
public void setScope(@Nullable String scope) {
   this.scope = scope;
}


@Override
@Nullable
public String getScope() {
   return this.scope;
}

// 是否是单例
@Override
public boolean isSingleton() {
   return SCOPE_SINGLETON.equals(this.scope) || SCOPE_DEFAULT.equals(this.scope);
}
// 是否是多例
@Override
public boolean isPrototype() {
   return SCOPE_PROTOTYPE.equals(this.scope);
}
```
#### 4. lazyInit
xml 
```java
<bean id="testConstArgService" class="com.lexi.service.TestConstArgService" lazy-init="true">
</bean>
```
注解
```java
@Lazy(true)
```

BeanDefinition
```java
void setLazyInit(boolean lazyInit);

boolean isLazyInit();
​```java
AbstractBeanDefinition
​```java
@Nullable
private Boolean lazyInit;
@Override
public void setLazyInit(boolean lazyInit) {
   this.lazyInit = lazyInit;
}

@Override
public boolean isLazyInit() {
   return (this.lazyInit != null && this.lazyInit.booleanValue());
}
```

#### 5. DependsOn
我们有这么一个需求，实例化某一个类之前，需要先实例化另外一个类。这时就用到了DependsOn

XML
```java
<bean id="testConstArgService" class="com.lexi.service.TestConstArgService" depends-on="testService">
</bean>
```
注解
```java
@DependsOn("testService")
```
BeanDefinition
```java
void setDependsOn(@Nullable String... dependsOn);

@Nullable
String[] getDependsOn();
```
AbstractBeanDefinition
```java
@Nullable
private String[] dependsOn;
@Override
public void setDependsOn(@Nullable String... dependsOn) {
   this.dependsOn = dependsOn;
}

@Override
@Nullable
public String[] getDependsOn() {
   return this.dependsOn;
}
```
#### 6. autowireCandidate
设置是否作为自动装配的候选对象

如果`HelloService`和`HelloService2`同时实现自`IHelloService`接口我们使用`getBean`或者`@Autowired`自动装配`IHelloService`时会报错

此时如果在`HelloService2`上加上`autowire-candidate="false"`表示自动装配时不考虑`HelloService2`，此时自动装配时只会使用`HelloService`
此属性只可以在XML设置

XML
```java
<bean id="helloService2" class="com.lexi.service.HelloService2" autowire-candidate="false">
</bean>
```

BeanDefinition
```java
void setAutowireCandidate(boolean autowireCandidate);

boolean isAutowireCandidate();
```

AbstractBeanDefinition

```java
private boolean autowireCandidate = true;

@Override
public void setAutowireCandidate(boolean autowireCandidate) {
   this.autowireCandidate = autowireCandidate;
}

@Override
public boolean isAutowireCandidate() {
   return this.autowireCandidate;
}
```

#### 7. Primary
和上面一样的场景，只是Primary表示自动装配时优先装配当前对象

BeanDefinition
```java
void setPrimary(boolean primary);

boolean isPrimary();
```

AbstractBeanDefinition
```java
private boolean primary = false;
@Override
public void setPrimary(boolean primary) {
   this.primary = primary;
}

/**
 * Return whether this bean is a primary autowire candidate.
 */
@Override
public boolean isPrimary() {
   return this.primary;
}
```
#### 8. FactoryBeanName(待完善)
BeanDefinition
```java
void setFactoryBeanName(@Nullable String factoryBeanName);

@Nullable
String getFactoryBeanName();
```
AbstractBeanDefinition
```java
@Nullable
private String factoryBeanName;
@Override
public void setFactoryBeanName(@Nullable String factoryBeanName) {
   this.factoryBeanName = factoryBeanName;
}

@Override
@Nullable
public String getFactoryBeanName() {
   return this.factoryBeanName;
}
```
#### 9. FactoryMethodName（待完善）
BeanDefinition
```java
void setFactoryMethodName(@Nullable String factoryMethodName);

@Nullable
String getFactoryMethodName();
```
AbstractBeanDefinition
```java
@Nullable
private String factoryMethodName;
@Override
public void setFactoryMethodName(@Nullable String factoryMethodName) {
   this.factoryMethodName = factoryMethodName;
}

/**
 * Return a factory method, if any.
 */
@Override
@Nullable
public String getFactoryMethodName() {
   return this.factoryMethodName;
}
```
#### 10. ConstructorArgumentValues
实例化对象时的构造方法的参数，只有XML配置constructor-arg时才有此参数，注解不会有此参数

```java
<bean id="helloService" class="com.lexi.service.HelloService">
    <constructor-arg name="name" value="lexi"></constructor-arg>
</bean>
```
or
```java
<bean id="helloService" class="com.lexi.service.HelloService">
    <constructor-arg index="0" value="lexi"></constructor-arg>
</bean>
```
or
```java
<bean id="helloService" class="com.lexi.service.HelloService">
    <constructor-arg value="lexi"></constructor-arg>
</bean>
```

BeanDefinition
```java
ConstructorArgumentValues getConstructorArgumentValues();

default boolean hasConstructorArgumentValues() {
   return !getConstructorArgumentValues().isEmpty();
}
```

AbstractBeanDefinition
```java
@Nullable
private ConstructorArgumentValues constructorArgumentValues;

@Override
public ConstructorArgumentValues getConstructorArgumentValues() {
   if (this.constructorArgumentValues == null) {
      this.constructorArgumentValues = new ConstructorArgumentValues();
   }
   return this.constructorArgumentValues;
}

@Override
public boolean hasConstructorArgumentValues() {
   return (this.constructorArgumentValues != null && !this.constructorArgumentValues.isEmpty());
}
```
其中ConstructorArgumentValues中是通过
```java
private final Map<Integer, ValueHolder> indexedArgumentValues = new LinkedHashMap<>();

private final List<ValueHolder> genericArgumentValues = new ArrayList<>();
```
来存储参数的
如果是
```java
<bean id="helloService" class="com.lexi.service.HelloService">
    <constructor-arg value="lexi" index="0"></constructor-arg>
    <constructor-arg value="1" index="1"></constructor-arg>
</bean>
```
这种格式，那么在`indexedArgumentValues`存储数据

其他会存储到`genericArgumentValues`


#### 11. PropertyValues

Bean中定义的属性的值，注意，这边只有XML定义的`<property>`才会有值，直接在类中通过`@Autowired`标注的不会出现在这边

XML
```java
<bean id="testService" class="com.lexi.service.TestService">
    <property name="helloService" ref="helloService"></property>
</bean>
```

BeanDefinition

```java
MutablePropertyValues getPropertyValues();

default boolean hasPropertyValues() {
   return !getPropertyValues().isEmpty();
}
```
AbstarctBeanDefinition

```java
@Nullable
private MutablePropertyValues propertyValues;

public MutablePropertyValues getPropertyValues() {
   if (this.propertyValues == null) {
      this.propertyValues = new MutablePropertyValues();
   }
   return this.propertyValues;
}

@Override
public boolean hasPropertyValues() {
   return (this.propertyValues != null && !this.propertyValues.isEmpty());
}
```
其中MutablePropertyValues是通过一个`List`来存储数据的
```java
private final List<PropertyValue> propertyValueList;
```
PropertyValue部分源码如下
```java
public class PropertyValue extends BeanMetadataAttributeAccessor implements Serializable {

   private final String name;

   @Nullable
   private final Object value;
    ...
}
```
#### 12. initMethod

Bean的初始化方法
注意：这边只有通过XML的`init-method`配置时，当字段才会有值，如果通过注解的`@PostConstruct`配置的，当前字段不会有值

XML
```java
<bean id="helloService" class="com.lexi.service.HelloService" init-method="init">
</bean>
```

BeanDefinition
```java
void setInitMethodName(@Nullable String initMethodName);

@Nullable
String getInitMethodName();
```

AbstractBeanDefinition
```java
@Nullable
private String initMethodName;
/**
 * Set the name of the initializer method.
 * <p>The default is {@code null} in which case there is no initializer method.
 */
@Override
public void setInitMethodName(@Nullable String initMethodName) {
   this.initMethodName = initMethodName;
}

/**
 * Return the name of the initializer method.
 */
@Override
@Nullable
public String getInitMethodName() {
   return this.initMethodName;
}
```
#### 13. destroyMethodName
Bean的生命周期销毁方法，同上，只有当通过XML的`destroy-method`配置时，当前字段才有值，如果通过注解的`@PreDestroy`，当前字段无值。

XML配置
```java
<bean id="helloService" class="com.lexi.service.HelloService" destroy-method="destory">
</bean>
```

BeanDefinition
```java
void setDestroyMethodName(@Nullable String destroyMethodName);

@Nullable
String getDestroyMethodName();
```
AbstractBeanDefinition
```java
@Nullable
private String destroyMethodName;

@Override
public void setDestroyMethodName(@Nullable String destroyMethodName) {
   this.destroyMethodName = destroyMethodName;
}

@Override
@Nullable
public String getDestroyMethodName() {
   return this.destroyMethodName;
}
```
#### 14. Description
添加Bean的注解描述,目前只能通过注解添加，在此猜测将来注解会不会完全取代xml

```java
@Description("这是个描述")
```
BeanDefinition

```java
void setDescription(@Nullable String description);

@Nullable
String getDescription();
```
AbstarctBeanDefinition
```java
@Nullable
private String description;

@Override
public void setDescription(@Nullable String description) {
   this.description = description;
}


@Override
@Nullable
public String getDescription() {
   return this.description;
}
```
#### 15. isSingleton & isPrototype
是否是单例 or 是否是多例


BeanDefinition
```java
boolean isSingleton();

boolean isPrototype();
```
AbstractBeanDefinition

```java
@Override
public boolean isSingleton() {
   return SCOPE_SINGLETON.equals(this.scope) || SCOPE_DEFAULT.equals(this.scope);
}

@Override
public boolean isPrototype() {
   return SCOPE_PROTOTYPE.equals(this.scope);
}
```

#### 16. isAbstract(待完善)

一般是和parent搭配使用，表示当前BeanDefinition是抽象的BeanDefinition。这边的抽象不一定是抽象类，这边的抽象是指当前BeanDefiniton无需设置beanClassName。

abstract一般是提起多个BeanDefiniton公共的部分，比如`scope`,`lazy`,`init`,`destory`还有相同的`属性`等等。然后当做父类BeanDefintion。

子BeanDefinition通过parent来继承自当前BeanDefinition。

abstract和parent一样，只能用在xml上

XML

```java
<bean id="abstarctService" lazy-init="true" scope="prototype" abstract="true"></bean>
<bean id="helloService" parent="abstarctService" class="com.lexi.client.HelloService"></bean>
```

BeanDefinition

```java
boolean isPrototype();

boolean isAbstract();
```

AbstractBeanDefinition

```java
private boolean abstractFlag = false;

public void setAbstract(boolean abstractFlag) {
   this.abstractFlag = abstractFlag;
}

@Override
public boolean isAbstract() {
   return this.abstractFlag;
}
```



## 二、BeanDefinition关系


BeanDefinition继承关系如上图所示。

### 1. AttributeAccessor 和 AttributeAccessorSupport
`AttributeAccessor`提供对数据元数据操作，元数据可以简单理解为Bean的额外描述信息，比如`BeanDefinition`的`@Configuration`是`full`还是`lite`，代理信息...





```java

public interface AttributeAccessor {

   void setAttribute(String name, @Nullable Object value);

   @Nullable
   Object getAttribute(String name);

   @Nullable
   Object removeAttribute(String name);

   boolean hasAttribute(String name);

   String[] attributeNames();
}
```



AttributeAccessorSupport 是对接口AttributeAccessor的实现，其内部维护了一个

由AttributeAccessorSupport实现，内部维护了一个

```java

private final Map<String, Object> attributes = new LinkedHashMap<>();

```

用来存储元数据



### 2. BeanMetadataElement
BeanDefinition的另一个接口是BeanMetadataElement，定义了对BeanDefinition所描述的类的源文件

```java

public interface BeanMetadataElement {
   @Nullable
   default Object getSource() {
      return null;
   }
}
```

其实现类是`BeanMetadataAttributeAccessor`



### 3. BeanMetadataAttributeAccessor
BeanMetadataAttributeAccessor实现了`BeanMetadataElement`表示其拥有操作BeanDefinition所描述类源文件的能力

```java
@Nullable
private Object source;

public void setSource(@Nullable Object source) {
   this.source = source;
}

@Override
@Nullable
public Object getSource() {
   return this.source;
}
```

`BeanMetadataAttributeAccessor`同时还继承自`AttributeAccessorSupport`，表示其还拥有操作元数据的能力
### 4. AbstarctBeanDefinition
`AbstarctBeanDefinition`实现了BeanDefinition，同时他也继承了` BeanMetadataAttributeAccessor`。

### 5. RootBeanDefinition（待完善）
Spring 2.5之前用来表示普通的beanDefinition或者父类beanDefinition,Spring2.5字后，被`GenericBeanDefinition`替换，无法设置parentName，即无法作为ChindBeanDefinition


### 5. ChildBeanDefinition
Spring 2.5之前用来表示子类Definition，Spring2.5字后被`GenericBeanDefinition`代替。实例化必要时提供parentName。即必须要有父类BeanDefinition

### 6. GenericBeanDefinition
Spring2.5之后代替`RootBeanDefinition`和`ChildBeanDefinition`,既可以作为parentBeanDefinition也可以作为childBeanDefinition
Spring2.5之后所有在XML中定义的BeanDefinition，用的都是GenericBeanDefinition，不管是parent，还是child，还是普通的beanDefinition，都是用GenericBeanDefinition来实现的

### 7. AnnotatedGenericBeanDefinition（待完善）
通过`AnnotationConfigApplicationContext`的`register`注册的类用的是`AnnotatedGenericBeanDefinition`,包括我们才实例化`AnnotationConfigApplicationContext`传的config类，它内部也调用的`register`方法



### 8. ScannedGenericBeanDefinition(待完善)
Spring加了`@Component`注解扫描出来的类默认使用此BeanDefinition

### 9. ConfigurationClassBeanDefinition（待完善）
使用@Bean注解生成的BeanDefinition