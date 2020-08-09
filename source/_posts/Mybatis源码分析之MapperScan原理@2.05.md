---
title: Mybatis源码分析-MapperScan原理@2.05
date: 2020-07-07 00:00:00
cover: https://cdn.jsdelivr.net/gh/dtlexi/lexi.blog/src/image/8.jpg
top_img: https://cdn.jsdelivr.net/gh/dtlexi/lexi.blog/src/image/big5.jpeg
categories:
 - java
 - mybatis
tags:
 - java
 - spring
 - 源码分析
 - spring 源码分析
 - mybatis 源码分析
 - mybatis
---


> 当前源码分析是基于`mybatis-spring 2.0.5`版本，和`mybatis-spring 2.0.2`之前版本有所不同

## 所需了解的知识点

如果想要完全理解@MapperScan原理，你需要了解如下知识点：

1. `@Import`作用和`@ImportBeanDefinitionRegistrar`的使用场景
2. `BeanFactoryPostProcessor`和`BeanDefinitionRegistryPostProcessor`执行时机
3. `ClassPathBeanDefinitionScanner`的原理和如何扩展`ClassPathBeanDefinitionScanner`实现自定义的扫描器
4. `BeanFactory`的用法和原理


## 源码分析

### MapperScan注解

先从`@MapperScan`注解说起

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(MapperScannerRegistrar.class)
@Repeatable(MapperScans.class)
public @interface MapperScan {
}
```
在`@MapperScan`注解引入了`Import`注解，在`@Import`注解中引入了`MapperScannerRegistrar`类，这个类继承自`ImportBeanDefinitionRegistrar`类

`@Import`和`ImportBeanDefinitionRegistrar`的详细作用详见Spring @Import.md，这边可以简单的理解为可以在`ImportBeanDefinitionRegistrar`的`registerBeanDefinitions`方法中向Spring的bdMap容器中注入`BeanDefinition`，从而将`BeanDefinition`对呀的类交给Spring管理

### MapperScannerRegistrar
接下来我们就来分析一下`MapperScannerRegistrar`的`registerBeanDefinitions`方法


```java
@Override
public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
    // 解析MapperScan
    AnnotationAttributes mapperScanAttrs = AnnotationAttributes
            .fromMap(importingClassMetadata.getAnnotationAttributes(MapperScan.class.getName()));
    if (mapperScanAttrs != null) {
        // 核心代码
        registerBeanDefinitions(importingClassMetadata, mapperScanAttrs, registry,
                generateBaseBeanName(importingClassMetadata, 0));
    }
}
```
继续跟进`registerBeanDefinitions(importingClassMetadata, mapperScanAttrs, registry,generateBaseBeanName(importingClassMetadata, 0));`
```java
void registerBeanDefinitions(AnnotationMetadata annoMeta, AnnotationAttributes annoAttrs,
                             BeanDefinitionRegistry registry, String beanName) {

    // 这一步其实就是构造一个MapperScannerConfigurer的BeanDefinition
    BeanDefinitionBuilder builder = BeanDefinitionBuilder.genericBeanDefinition(MapperScannerConfigurer.class);
    builder.addPropertyValue("processPropertyPlaceHolders", true);

    // 设置annotationClass属性
    // 这个属性将在实例化MapperScannerConfigurer对象时用到
    Class<? extends Annotation> annotationClass = annoAttrs.getClass("annotationClass");
    if (!Annotation.class.equals(annotationClass)) {
        builder.addPropertyValue("annotationClass", annotationClass);
    }

    Class<?> markerInterface = annoAttrs.getClass("markerInterface");
    if (!Class.class.equals(markerInterface)) {
        builder.addPropertyValue("markerInterface", markerInterface);
    }

    Class<? extends BeanNameGenerator> generatorClass = annoAttrs.getClass("nameGenerator");
    if (!BeanNameGenerator.class.equals(generatorClass)) {
        builder.addPropertyValue("nameGenerator", BeanUtils.instantiateClass(generatorClass));
    }

    Class<? extends MapperFactoryBean> mapperFactoryBeanClass = annoAttrs.getClass("factoryBean");
    if (!MapperFactoryBean.class.equals(mapperFactoryBeanClass)) {
        builder.addPropertyValue("mapperFactoryBeanClass", mapperFactoryBeanClass);
    }

    String sqlSessionTemplateRef = annoAttrs.getString("sqlSessionTemplateRef");
    if (StringUtils.hasText(sqlSessionTemplateRef)) {
        builder.addPropertyValue("sqlSessionTemplateBeanName", annoAttrs.getString("sqlSessionTemplateRef"));
    }

    String sqlSessionFactoryRef = annoAttrs.getString("sqlSessionFactoryRef");
    if (StringUtils.hasText(sqlSessionFactoryRef)) {
        builder.addPropertyValue("sqlSessionFactoryBeanName", annoAttrs.getString("sqlSessionFactoryRef"));
    }

    List<String> basePackages = new ArrayList<>();
    basePackages.addAll(
            Arrays.stream(annoAttrs.getStringArray("value")).filter(StringUtils::hasText).collect(Collectors.toList()));

    basePackages.addAll(Arrays.stream(annoAttrs.getStringArray("basePackages")).filter(StringUtils::hasText)
            .collect(Collectors.toList()));

    basePackages.addAll(Arrays.stream(annoAttrs.getClassArray("basePackageClasses")).map(ClassUtils::getPackageName)
            .collect(Collectors.toList()));

    if (basePackages.isEmpty()) {
        basePackages.add(getDefaultBasePackage(annoMeta));
    }

    String lazyInitialization = annoAttrs.getString("lazyInitialization");
    if (StringUtils.hasText(lazyInitialization)) {
        builder.addPropertyValue("lazyInitialization", lazyInitialization);
    }

    // 设置需要扫描的包
    builder.addPropertyValue("basePackage", StringUtils.collectionToCommaDelimitedString(basePackages));

    // 注册MapperScannerConfigurer的BeanDefinition
    registry.registerBeanDefinition(beanName, builder.getBeanDefinition());

}
```
在上面的方法中，其实基本没干什么事，主要就是注册了一个`MapperScannerConfigurer`的类的`BeanDefinition`，并且设置初始化时的各种属性

### MapperScannerConfigurer

有的人可能会问：What? 就这么简单？那么Mybatis是怎么将扫描@Mapper注解，并且注册到Spring中去的呢？别急，下面我们来分析分析`MapperScannerConfigurer`类

在`MapperScannerConfigurer`类中，我们发现了其一个很重要的父类

```java
public class MapperScannerConfigurer
    implements BeanDefinitionRegistryPostProcessor, InitializingBean, ApplicationContextAware, BeanNameAware 
{
}
```
`BeanDefinitionRegistryPostProcessor`这个类有没有很熟悉？这个类在之前Spring源码分析之BeanFactoryPostProcessor.md一文中详细讲解过这个类。
这个类有俩个很重要的方法`postProcessBeanDefinitionRegistry`和`postProcessBeanFactory`在初始化Spring容器是被调用，通过源码可知，postProcessBeanFactory方法啥也没干，

所以我们来重点分析`postProcessBeanDefinitionRegistry`方法
```java
@Override
public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
    if (this.processPropertyPlaceHolders) {
        processPropertyPlaceHolders();
    }
    // 实例化一个ClassPathMapperScanner
    ClassPathMapperScanner scanner = new ClassPathMapperScanner(registry);
    scanner.setAddToConfig(this.addToConfig);
    scanner.setAnnotationClass(this.annotationClass);
    scanner.setMarkerInterface(this.markerInterface);
    scanner.setSqlSessionFactory(this.sqlSessionFactory);
    scanner.setSqlSessionTemplate(this.sqlSessionTemplate);
    scanner.setSqlSessionFactoryBeanName(this.sqlSessionFactoryBeanName);
    scanner.setSqlSessionTemplateBeanName(this.sqlSessionTemplateBeanName);
    scanner.setResourceLoader(this.applicationContext);
    scanner.setBeanNameGenerator(this.nameGenerator);
    scanner.setMapperFactoryBeanClass(this.mapperFactoryBeanClass);
    if (StringUtils.hasText(lazyInitialization)) {
        scanner.setLazyInitialization(Boolean.valueOf(lazyInitialization));
    }
    // 设置扫描Filter
    scanner.registerFilters();
    // 调用父类的scan方法进行扫描
    scanner.scan(
            StringUtils.tokenizeToStringArray(this.basePackage, ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS));
}
```
这个方法也比较简单，就是实例化了一个扫描器，设置了扫描器扫描过滤的Filter，调用了scan方法进行扫描。

### ClassPathMapperScanner 

下面我们将对初始化`ClassPathMapperScanner`，`registerFilters()`和`scan()`进行分析

```java
public class ClassPathMapperScanner extends ClassPathBeanDefinitionScanner {
    public ClassPathMapperScanner(BeanDefinitionRegistry registry) {
        super(registry, false);
    }
}
```
`ClassPathMapperScanner`继承自`ClassPathBeanDefinitionScanner`类
在之前的学习中，我们已经知道`ClassPathBeanDefinitionScanner`能够扫描项目中的类，并且生成`BeanDefinition`添加的`bdmap`中去

在上面的构造方法中，`ClassPathMapperScanner`调用父类的默认构造方法，并且不使用默认的`incluideFilters`。即不使用spring默认的扫描规则（spring默认扫描@component注解）

下面我们来分析一下`registerFilters()`方法

```java
/**
 * 这个方法主要是来添加扫描Filter的
 * 主要是定义需要扫描那些类，或者排除哪些类
 */
public void registerFilters() {
    // 是否扫描全部接口
    // 如果用户为指定需要扫描的注解，或者父类，那么就扫描全部类
    boolean acceptAllInterfaces = true;

    // 设置需要扫描的接口
    if (this.annotationClass != null) {
        addIncludeFilter(new AnnotationTypeFilter(this.annotationClass));
        acceptAllInterfaces = false;
    }

    // 设置需要扫描的哪个接口的子接口，并且排除当前接口
    if (this.markerInterface != null) {
        addIncludeFilter(new AssignableTypeFilter(this.markerInterface) {
            @Override
            protected boolean matchClassName(String className) {
                return false;
            }
        });
        acceptAllInterfaces = false;
    }
    // 如果当前没有设置任何扫描规则，那么扫描全部类
    if (acceptAllInterfaces) {
        // default include filter that accepts all classes
        addIncludeFilter((metadataReader, metadataReaderFactory) -> true);
    }

    // 排除package-info.java
    addExcludeFilter((metadataReader, metadataReaderFactory) -> {
        String className = metadataReader.getClassMetadata().getClassName();
        return className.endsWith("package-info");
    });
}
```
`registerFilters`这个方法也补交好理解，主要是设置扫描添加了哪个注解的接口或者哪个接口的子类，如果当前未设置任何规则，那么扫描全部接口

下面就是我们的重头戏，`doScan`方法
```java
// 这个方法主要有俩个功能
// 1. 调用父类的doscan方法，扫描指定规则的类,并且生成BeanDefiniton
// 2. 处理BeanDefinition，这边db的class为factorybean
@Override
public Set<BeanDefinitionHolder> doScan(String... basePackages) {
    // 扫描
    Set<BeanDefinitionHolder> beanDefinitions = super.doScan(basePackages);

    if (beanDefinitions.isEmpty()) {
        LOGGER.warn(() -> "No MyBatis mapper was found in '" + Arrays.toString(basePackages)
                + "' package. Please check your configuration.");
    } else {
        //处理BeanDefinition
        processBeanDefinitions(beanDefinitions);
    }

    return beanDefinitions;
}
```
在doScan方法中，主要完成了俩件事
1. 调用父类的doscan方法，扫描指定规则的类,并且生成BeanDefiniton
2. 处理BeanDefinition，这边db的class为factorybean

其中`super.doScan(basePackages);`，已经在之前的Spring源码分析之ClassPathBeanDefinitionScanner.md中详细分析过了，在此就不在重复了。
这边还有一个问题，Mybatis是怎么控制扫描的结果都是接口的呢？答案就是重写父类中的`isCandidateComponent`方法

不知道大家是否还记得`ClassPathBeanDefinitionScanner`类中的俩个`isCandidateComponent`类，它们主要的作用就是在对scan扫描出的Matedata和BeanDfeinition做筛选

```java
@Override
protected boolean isCandidateComponent(AnnotatedBeanDefinition beanDefinition) {
  return beanDefinition.getMetadata().isInterface() && beanDefinition.getMetadata().isIndependent();
}
```

### FactoryBean

接下来就是最精彩的部分，Mybatis是如何将扫描出的接口转换为对象的呢？答应就是FactoryBean。

下面我们来看具体实现代码
```java
private void processBeanDefinitions(Set<BeanDefinitionHolder> beanDefinitions) {
    GenericBeanDefinition definition;
    for (BeanDefinitionHolder holder : beanDefinitions) {
        // 拿到当前循环的BeanDefinition
        definition = (GenericBeanDefinition) holder.getBeanDefinition();
        String beanClassName = definition.getBeanClassName();
        LOGGER.debug(() -> "Creating MapperFactoryBean with name '" + holder.getBeanName() + "' and '" + beanClassName
                + "' mapperInterface");

        // 加入BeanDefinition对应Bean的构造方法参数
        definition.getConstructorArgumentValues().addGenericArgumentValue(beanClassName); // issue #59
        // 设置BeanDefinition的BeanClass为MapperFactoryBean
        definition.setBeanClass(this.mapperFactoryBeanClass);

        // 设置BeanDefinition的参数 addToConfig
        definition.getPropertyValues().add("addToConfig", this.addToConfig);

        boolean explicitFactoryUsed = false;
        // 下面是一个判断，判断sqlSessionFactoryBeanName是否为空
        if (StringUtils.hasText(this.sqlSessionFactoryBeanName)) {
            // 如果sqlSessionFactoryBeanName不为空，
            // 就将BeanDefinition的sqlSessionFactory属性设置为sqlSessionFactoryBeanName对应引用的对象
            // 这边可以简单理解为<bean id="xxx" ref="xxx" />
            definition.getPropertyValues().add("sqlSessionFactory",
                    new RuntimeBeanReference(this.sqlSessionFactoryBeanName));
            explicitFactoryUsed = true;
        } else if (this.sqlSessionFactory != null) {
            // 如果当前没有设置sqlSessionFactoryBeanName
            // 那么就将当前sqlSessionFactory设置给BeanDefinition
            definition.getPropertyValues().add("sqlSessionFactory", this.sqlSessionFactory);
            explicitFactoryUsed = true;
        }

        // 和上面差不多
        // 主要是设置sqlSessionTemplate
        if (StringUtils.hasText(this.sqlSessionTemplateBeanName)) {
            if (explicitFactoryUsed) {
                LOGGER.warn(
                        () -> "Cannot use both: sqlSessionTemplate and sqlSessionFactory together. sqlSessionFactory is ignored.");
            }
            definition.getPropertyValues().add("sqlSessionTemplate",
                    new RuntimeBeanReference(this.sqlSessionTemplateBeanName));
            explicitFactoryUsed = true;
        } else if (this.sqlSessionTemplate != null) {
            if (explicitFactoryUsed) {
                LOGGER.warn(
                        () -> "Cannot use both: sqlSessionTemplate and sqlSessionFactory together. sqlSessionFactory is ignored.");
            }
            definition.getPropertyValues().add("sqlSessionTemplate", this.sqlSessionTemplate);
            explicitFactoryUsed = true;
        }

        if (!explicitFactoryUsed) {
            LOGGER.debug(() -> "Enabling autowire by type for MapperFactoryBean with name '" + holder.getBeanName() + "'.");
            definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);
        }
        definition.setLazyInit(lazyInitialization);
    }
}
```
`processBeanDefinitions`主要做的就是修改扫描出来的`BeanDefinition`，从之前的代码中，我们知道`@MapperScan`扫描出来的都是接口，而接口都是没法实例化的。而Mybatis巧妙的通过修改扫描的BeanDefinition,将其对呀的Class修改为`MapperFactoryBean`对象，并且设置起实例化所需要的参数和属性。

而`MapperFactoryBean`又是继承自`FactoryBean`类，`FactoryBean`的作用大家应该清楚吧？不懂的百度走起

```java
public class MapperFactoryBean<T> implements FactoryBean<T> {

    private Class<T> mapperInterface;


    public MapperFactoryBean(Class<T> mapperInterface) {
        this.mapperInterface = mapperInterface;
    }

    /**
     * {@inheritDoc}
     */
    @Override
    public T getObject() throws Exception {
        return getSqlSession().getMapper(this.mapperInterface);
    }

    /**
     * {@inheritDoc}
     */
    @Override
    public Class<T> getObjectType() {
        return this.mapperInterface;
    }

    /**
     * {@inheritDoc}
     */
    @Override
    public boolean isSingleton() {
        return true;
    }
}
```
`return getSqlSession().getMapper(this.mapperInterface);`有没有很熟悉？这不就是`Mybatis`获取mapper对象的方法吗？而在Spring容器实例化当前BD时，将调用当前`getObject()`方法，返回Mybatis生成的代理对象。

## 总结
到这，Myatis整合Spring是不是就清楚了？下面我们对此做个总结

1. 首先，Mybait定义一个`@MapperScan`注解供开发者使用，`@MapperScan`注解上`Import`一个继承自`ImportBeanDefinitionRegistrar`的`MapperScannerRegistrar`类
2. `MapperScannerRegistrar`类中注入一个`MapperScannerConfigurer`类的BeanDefinition，`MapperScannerConfigurer`实现了`BeanDefinitionRegistryPostProcessor`,将会在Spring容器初始化时被执行
3. `MapperScannerConfigurer`类的`postProcessBeanDefinitionRegistry`方法中实例化了一个`ClassPathMapperScanner`类，用来扫描Dao接口
    1. 定义一个`registerFilters`方法，用来添加扫描Filter
    2. 重写父类的`doScan`方法，在`doScan`方法中对扫描出来的BD就行处理，将BeanDefinition的Class设置为`MapperFactoryBean`
    3. 重写父类的`isCandidateComponent`，对扫描出来的bd就行赛选，确保扫描的是接口
4. 在`MapperFactoryBean`的getObject()方法中返回Mybatis的`getMapper(this.mapperInterface)`，Spring容器实例化Mybatis扫描出来的Bd时，将会调用`MapperFactoryBean`的`getObject()`方法，调用Mybatis原生的`getMapper()`方法产生代理对象