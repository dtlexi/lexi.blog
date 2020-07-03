---
title: Spring源码分析-ClassPathBeanDefinitionScanner
data: 2020-07-02
cover: https://ss2.bdstatic.com/70cFvnSh_Q1YnxGkpoWK1HF6hhy/it/u=3827979352,1501043192&fm=26&gp=0.jpg
top_img: https://spring.io/images/spring-logo-9146a4d3298760c2e7e49595184e1975.svg
categories:
 - java
 - spring
tags:
 - java
 - spring
 - 源码分析
---

在阅读spring源码时，发现在创建`AnnotationConfigApplicationContext`对象是，spring在其构造方法中，给我们实例化了俩个对象。
```java
public AnnotationConfigApplicationContext() {
    this.reader = new AnnotatedBeanDefinitionReader(this);
    //这边初始化了一个spring的类扫描器，
    // 其实通过分析后面`ConfigurationClassPostProcessor`中源码可知
    // spring默认扫描包时不是使用的这个对象，而是重新new了一个`ClassPathBeanDefinitionScanner`对象。
    // 这里的scanner仅仅是为了程序员可以手动调用AnnotationConfigApplicationContext对象的scan方法。
    this.scanner = new ClassPathBeanDefinitionScanner(this);
}
```
`AnnotatedBeanDefinitionReader`是AnnotatedBeanDefinition读取器，这个是一个非常重要的类，这不是今天的重点，我们会在后面专门介绍他。

下面我们就来分析一下今天的重点`ClassPathBeanDefinitionScanner`

### ClassPathBeanDefinitionScanner作用

`ClassPathBeanDefinitionScanner`的作用就像一个雷达，它能从一个指定包中扫描出带有某些注解的类，并且将他们转化为`BeanDefinition`对象。它主要干了俩件事：
1. 扫描类路径下的候选BeanDefinition
2. 将BeanDefinition注册到BeanDefinitionRegister中

默认情况想，他仅仅会扫描带有如下注解的类：
1. `@Component`,包括其子类（`@Service`,`@Dao`,`@Controller`）
2. `@ManagedBean`

### ClassPathBeanDefinitionScanner源码分析

```java
public ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry) {
   this(registry, true);
}

public ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry, boolean useDefaultFilters) {
   this(registry, useDefaultFilters, getOrCreateEnvironment(registry));
}

public ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry, boolean useDefaultFilters,
      Environment environment) {

   this(registry, useDefaultFilters, environment,
         (registry instanceof ResourceLoader ? (ResourceLoader) registry : null));
}

public ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry, boolean useDefaultFilters,
      Environment environment, @Nullable ResourceLoader resourceLoader) {

   Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
   this.registry = registry;
    
    //是否采用Spring默认的Filters，这边的Filters指需要扫描的注解
    if (useDefaultFilters) {
        // 注册默认扫描的注解
        registerDefaultFilters();
    }

   setEnvironment(environment);
   setResourceLoader(resourceLoader);
}
```
上面代码是`ClassPathBeanDefinitionScanner`的构造方法，通过分析上面代码我们发现，`ClassPathBeanDefinitionScanner`虽然有四个构造方法，但是其最后都会调用最后一个四个构造参数的构造方法

在最后一个构造方法用，如果使用默认的Filters（useDefaultFilters=true）,那么会调用`registerDefaultFilters`注册默认的Filters
这个方法在`ClassPathBeanDefinitionScanner`的父类中`ClassPathScanningCandidateComponentProvider`中

```java
// 扫描包时需要包括的TypeFilter
private final List<TypeFilter> includeFilters = new LinkedList<>();

// 扫描包时排除的TypeFilter
private final List<TypeFilter> excludeFilters = new LinkedList<>();

@SuppressWarnings("unchecked")
protected void registerDefaultFilters() {
    // 添加new AnnotationTypeFilter(Component.class),
    // 这是一个注解扫描类，即扫描时扫描当前注解
    // 这边不光包括扫描@Component,还包括其子类，@Service,@Dao,@Controller
    this.includeFilters.add(new AnnotationTypeFilter(Component.class));
    ClassLoader cl = ClassPathScanningCandidateComponentProvider.class.getClassLoader();
    try {
        // 添加扫描注册@ManagedBean
        this.includeFilters.add(new AnnotationTypeFilter(
                ((Class<? extends Annotation>) ClassUtils.forName("javax.annotation.ManagedBean", cl)), false));
        logger.trace("JSR-250 'javax.annotation.ManagedBean' found and supported for component scanning");
    }
    catch (ClassNotFoundException ex) {
        // JSR-250 1.1 API (as included in Java EE 6) not available - simply skip.
    }
    try {
        // 添加扫描注册@Named
        this.includeFilters.add(new AnnotationTypeFilter(
                ((Class<? extends Annotation>) ClassUtils.forName("javax.inject.Named", cl)), false));
        logger.trace("JSR-330 'javax.inject.Named' annotation found and supported for component scanning");
    }
    catch (ClassNotFoundException ex) {
        // JSR-330 API not available - simply skip.
    }
}
```

`ClassPathBeanDefinitionScanner`另一个重要的方法就是`Scan`方法，源码如下：
```java
public int scan(String... basePackages) {
    int beanCountAtScanStart = this.registry.getBeanDefinitionCount();

    // 调用doScan方法进行扫描
    doScan(basePackages);

    // Register annotation config processors, if necessary.
    if (this.includeAnnotationConfig) {
        AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
    }

    return (this.registry.getBeanDefinitionCount() - beanCountAtScanStart);
}
```
这个方法主要是调用doScan()方法进行扫描，然后计算扫描出来的`BeanDefinition`数量
```java
protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
    Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();
    for (String basePackage : basePackages) {
        // 扫描包中加了对应注解的类，并且将其转换成BeanDefinition
        Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
        // 循环扫描到的BeanDefinition
        for (BeanDefinition candidate : candidates) {
            //如果这个类是AbstractBeanDefinition的子类
            //则为他设置默认值，比如lazy，init destory
            ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
            candidate.setScope(scopeMetadata.getScopeName());
            String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
            if (candidate instanceof AbstractBeanDefinition) {
                postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
            }
            if (candidate instanceof AnnotatedBeanDefinition) {
                AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
            }
            if (checkCandidate(beanName, candidate)) {
                BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
                definitionHolder =
                        AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
                beanDefinitions.add(definitionHolder);
                registerBeanDefinition(definitionHolder, this.registry);
            }
        }
    }
    return beanDefinitions;
}
```
在doScan方法中，调用父类的findCandidateComponents方法扫描数据

```java
public Set<BeanDefinition> findCandidateComponents(String basePackage) {
    if (this.componentsIndex != null && indexSupportsIncludeFilters()) {
        // 上面是引入spring index生成扫描索引时
        return addCandidateComponentsFromIndex(this.componentsIndex, basePackage);
    }
    else {
        // 直接扫描，一般采用这个
        return scanCandidateComponents(basePackage);
    }
}
```
scanCandidateComponents源码
```java
private Set<BeanDefinition> scanCandidateComponents(String basePackage) {
    Set<BeanDefinition> candidates = new LinkedHashSet<>();
    try {
        // classpath+包名+.class
        // classpath*:com/lexi/service/**/*.class
        String packageSearchPath = ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX +
                resolveBasePackage(basePackage) + '/' + this.resourcePattern;
        // 获取资源文件，采用的asm，没有详细研究
        // 大体思路是读取文件流，使用asm解析
        Resource[] resources = getResourcePatternResolver().getResources(packageSearchPath);
        boolean traceEnabled = logger.isTraceEnabled();
        boolean debugEnabled = logger.isDebugEnabled();
        for (Resource resource : resources) {
            if (traceEnabled) {
                logger.trace("Scanning " + resource);
            }
            if (resource.isReadable()) {
                try {
                    // 使用asm解析成MetadataReader
                    MetadataReader metadataReader = getMetadataReaderFactory().getMetadataReader(resource);
                    // 扫描是扫描全部类，这边是判断当前资源是否符合扫描规则
                    // spring 定义includeFilter和excludeFilters两个list
                    //     private final List<TypeFilter> includeFilters = new LinkedList<>();
                    // private final List<TypeFilter> excludeFilters = new LinkedList<>();
                    // 其中includeFilter表示应该包含哪些规则，excludeFilters表示应该排除哪些规则
                    // TypeFilter是定义规则的，一般包含如下：
                    // 1. AnnotationTypeFilter：定义包含或者排除哪些注解
                    // 2. AspectJTypeFilter: 使用AspectJ来定义规则
                    // 3. RegexPatternTypeFilter: 类名的正则表达式
                    // 我们还可以自定义FilterType，只要定义一个类实现TypeFilter即可
                    if (isCandidateComponent(metadataReader)) {
                        ScannedGenericBeanDefinition sbd = new ScannedGenericBeanDefinition(metadataReader);
                        sbd.setResource(resource);
                        sbd.setSource(resource);
                        // 这边是扫描判断生成的beanDefinition是否符合规则
                        // 是否是独立的类，抽象类是否添加了@LookUp接口等等..
                        if (isCandidateComponent(sbd)) {
                            if (debugEnabled) {
                                logger.debug("Identified candidate component class: " + resource);
                            }
                            candidates.add(sbd);
                        }
                        else {
                            if (debugEnabled) {
                                logger.debug("Ignored because not a concrete top-level class: " + resource);
                            }
                        }
                    }
                    else {
                        if (traceEnabled) {
                            logger.trace("Ignored because not matching any filter: " + resource);
                        }
                    }
                }
                catch (Throwable ex) {
                    throw new BeanDefinitionStoreException(
                            "Failed to read candidate component class: " + resource, ex);
                }
            }
            else {
                if (traceEnabled) {
                    logger.trace("Ignored because not readable: " + resource);
                }
            }
        }
    }
    catch (IOException ex) {
        throw new BeanDefinitionStoreException("I/O failure during classpath scanning", ex);
    }
    return candidates;
}
```
下面是俩个isCandidateComponent源码
```java
protected boolean isCandidateComponent(MetadataReader metadataReader) throws IOException {
   for (TypeFilter tf : this.excludeFilters) {
      if (tf.match(metadataReader, getMetadataReaderFactory())) {
         return false;
      }
   }
   for (TypeFilter tf : this.includeFilters) {
      if (tf.match(metadataReader, getMetadataReaderFactory())) {
         return isConditionMatch(metadataReader);
      }
   }
   return false;
}
```
```java
protected boolean isCandidateComponent(AnnotatedBeanDefinition beanDefinition) {
   AnnotationMetadata metadata = beanDefinition.getMetadata();
   return (metadata.isIndependent() && (metadata.isConcrete() ||
         (metadata.isAbstract() && metadata.hasAnnotatedMethods(Lookup.class.getName()))));
}
```