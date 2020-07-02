---
title: Spring源码分析-BeanFactoryPostProcessor
data: 2020-06-30
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


`BeanFactoryPostProcessor`是实现spring容器功能扩展的重要接口，例如修改bean属性值。很多框架都是通过此接口实现对spring容器的扩展，例如Mybatis(后面会说)



BeanFactoryPostProcessor 源码如下：



```java

@FunctionalInterface
public interface BeanFactoryPostProcessor {
   void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;
}
```



### BeanFactoryPostProcessor 使用



实现一个类继承自BeanFactoryPostProcessor接口



```java

public class TestBeanFactoryPostProcessor implements BeanFactoryPostProcessor {
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        // 这边可以拿到beanFactory对象，可以通过beanFactory对象获取到对应的beanDefinition。并且可以对BeanDefinition进行修改

        BeanDefinition helloDefinition = beanFactory.getBeanDefinition("helloService");
        helloDefinition.setScope(BeanDefinition.SCOPE_PROTOTYPE);
    }
}
```



这边可以拿到beanFactory对象，可以通过beanFactory对象获取到对应的beanDefinition。并且可以对BeanDefinition进行修改



### BeanFactoryPostProcessor 的执行流程源码分析

```java

1. org.springframework.context.support.AbstractApplicationContext#refresh

2. org.springframework.context.support.AbstractApplicationContext#invokeBeanFactoryPostProcessors

3. org.springframework.context.support.PostProcessorRegistrationDelegate#invokeBeanFactoryPostProcessors(org.springframework.beans.factory.config.ConfigurableListableBeanFactory, java.util.List<org.springframework.beans.factory.config.BeanFactoryPostProcessor>)

```



invokeBeanFactoryPostProcessors 源码如下（含详细注释）：



```java

public static void invokeBeanFactoryPostProcessors(
        ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {

    // 这边是保存的已经处理过的
    Set<String> processedBeans = new HashSet<>();

    // 判断当前bean工厂是否继承自BeanDefinitionRegistry接口
    // 因为BeanDefinitionRegistryPostProcessor接口的postProcessBeanDefinitionRegistry方法需要传递BeanDefinitionRegistry参数
    if (beanFactory instanceof BeanDefinitionRegistry) {
        // 将beanFactory强转为BeanDefinitionRegistry对象
        // postProcessBeanDefinitionRegistry(registry)
        BeanDefinitionRegistry registry = (BeanDefinitionRegistry) beanFactory;

        // beanFactoryPostProcessors中没有实现BeanDefinitionRegistryPostProcessor部分的集合
        // 这边指通过context.addBeanFactoryPostProcessor添加没有实现BeanDefinitionRegistryPostProcessor部分的集合
        // 会在执行完BeanDefinitionRegistryPostProcessor的postProcessBeanFactory方法后首先执行当前队列中对象的postProcessBeanFactory方法
        List<BeanFactoryPostProcessor> regularPostProcessors = new ArrayList<>();

        List<BeanDefinitionRegistryPostProcessor> registryProcessors = new ArrayList<>();

        // 第一波 postProcessBeanDefinitionRegistry 调用
        // 只我们在refresh方法之前调用context对象的addBeanFactoryPostProcessor方法传递的对象
        //        AnnotationConfigApplicationContext context=new AnnotationConfigApplicationContext();
        //        context.register(SpringConfig.class);
        //        context.addBeanFactoryPostProcessor(new TestBeanDefinitionRegistryPostProcessor());
        //        context.refresh();
        for (BeanFactoryPostProcessor postProcessor : beanFactoryPostProcessors) {
            // 判断当前对象是否实现自BeanDefinitionRegistryPostProcessor接口
            if (postProcessor instanceof BeanDefinitionRegistryPostProcessor) {
                // 如果当前对象实现了BeanDefinitionRegistryPostProcessor
                // 那么就会调用当前对象的postProcessBeanDefinitionRegistry方法
                BeanDefinitionRegistryPostProcessor registryProcessor =
                        (BeanDefinitionRegistryPostProcessor) postProcessor;
                // 执行postProcessBeanDefinitionRegistry方法
                // 注意：此时因为还没执行ConfigurationClassPostProcessor
                // 所以此时还没有扫描@Component注解，所以辞职无法操作相关BeanDefinition
                // 此时只有beanDefinitionMap中，只有我们通过register注册的配置类和其他几个spring自带带系统类
                registryProcessor.postProcessBeanDefinitionRegistry(registry);
                //
                registryProcessors.add(registryProcessor);
            }
            else {
                //
                regularPostProcessors.add(postProcessor);
            }
        }

        // Do not initialize FactoryBeans here: We need to leave all regular beans
        // uninitialized to let the bean factory post-processors apply to them!
        // Separate between BeanDefinitionRegistryPostProcessors that implement
        // PriorityOrdered, Ordered, and the rest.

        // 临时容器,放置一些内容,排序后,归档到  registryProcessors 中后,就重置清理了.
        List<BeanDefinitionRegistryPostProcessor> currentRegistryProcessors = new ArrayList<>();

        // 第二波 postProcessBeanDefinitionRegistry 调用
        // 这一波调用同时实现了BeanDefinitionRegistryPostProcessor和PriorityOrdered接口的类
        // 核心类ConfigurationClassPostProcessor的postProcessBeanDefinitionRegistry在此处被调用
        // 如果是程序员自定义的BeanDefinitionRegistryPostProcessor的实现类，如果同时实现了PriorityOrdered接口，
        // 有如下俩种情况：
        //      1.  如果在refresh()之前，调用了context.registerBeanDefinition()，注册了当前BeanDefinitionRegistryPostProcessor对呀的BeanDefinition
        //          那么此时会一同执行
        //      2.  如果只是加了@Component注解的BeanDefinitionRegistryPostProcessor的实现类
        //          因为@Component注解在ConfigurationClassPostProcessor的postProcessBeanDefinitionRegistry扫描的
        //          所以此时程序员自定义的BeanDefinitionRegistryPostProcessor接口的类，不管其是否实现了PriorityOrdered接口

        String[] postProcessorNames =
                beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
        for (String ppName : postProcessorNames) {
            if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
                // getBean实例化对象，并且将其放到currentRegistryProcessors好集合中去
                currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                processedBeans.add(ppName);
            }
        }
        // 排序
        sortPostProcessors(currentRegistryProcessors, beanFactory);
        registryProcessors.addAll(currentRegistryProcessors);
        // 调用第二波ConfigurationClassPostProcessor的postProcessBeanDefinitionRegistry
        invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
        // 清空
        currentRegistryProcessors.clear();

        // 第三波 postProcessBeanDefinitionRegistry 调用
        // 这一波是调用实现了Ordered接口类型的BeanDefinitionRegistryPostProcessor
        //      如果是程序员自己实现的BeanDefinitionRegistryPostProcessor,同时实现了PriorityOrdered，,并且加了@Component注解
        //      但是由于在第二波执行时，@Component未被扫描出来从而未被执行
        //      那么此时也会一同执行（PriorityOrdered继承自Ordered）
        postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
        for (String ppName : postProcessorNames) {
            if (!processedBeans.contains(ppName) && beanFactory.isTypeMatch(ppName, Ordered.class)) {
                currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                processedBeans.add(ppName);
            }
        }
        sortPostProcessors(currentRegistryProcessors, beanFactory);
        registryProcessors.addAll(currentRegistryProcessors);
        invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
        currentRegistryProcessors.clear();

        // 第四波 postProcessBeanDefinitionRegistry 调用
        // 调用其他postProcessBeanDefinitionRegistry
        // 这一波采用双重循环
        // 第一次循环取出所有能获取到的其他的BeanDefinitionRegistryPostProcessor接口（比如@Component,context添加的...）
        // 第二次循环是解决在第一次循环过程中postProcessBeanDefinitionRegistry方法中新添加的
        //  public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
        //        GenericBeanDefinition beanDefinition=new GenericBeanDefinition();
        //        beanDefinition.setBeanClass(TestBeanDefinitionRegistryPostProcessor5.class);
        //        registry.registerBeanDefinition("testBeanDefinitionRegistryPostProcessor5",beanDefinition);
        //  }
        boolean reiterate = true;
        while (reiterate) {
            reiterate = false;
            postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
            for (String ppName : postProcessorNames) {
                if (!processedBeans.contains(ppName)) {
                    currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                    processedBeans.add(ppName);
                    reiterate = true;
                }
            }
            sortPostProcessors(currentRegistryProcessors, beanFactory);
            registryProcessors.addAll(currentRegistryProcessors);
            invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
            currentRegistryProcessors.clear();
        }

        // 第一波 postProcessBeanFactory 调用，此时所有的postProcessBeanDefinitionRegistry已全部执行完成
        // 这一波BeanFactoryPostProcessor对象主要由俩部分组成
        //    1. 同时实现了BeanDefinitionRegistryPostProcessor接口的
        //    2. context.addBeanFactoryPostProcessor(new TestBeanDefinitionRegistryPostProcessor());
        // spring开发人员认为这种通过编程方式直接加入的执行的优先级都比较高


        // 执行所有BeanDefinitionRegistryPostProcessor的postProcessBeanFactory
        invokeBeanFactoryPostProcessors(registryProcessors, beanFactory);

        // 执行regularPostProcessors集合中的postProcessBeanFactory方法
        invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
    }

    else {
        // Invoke factory processors registered with the context instance.
        invokeBeanFactoryPostProcessors(beanFactoryPostProcessors, beanFactory);
    }

    // 第二波 postProcessBeanFactory 调用
    // 这边没什么好说的
    // 执行顺序是：
    //      1. PriorityOrdered
    //      2. Ordered
    //      3. Other

    // Do not initialize FactoryBeans here: We need to leave all regular beans
    // uninitialized to let the bean factory post-processors apply to them!
    String[] postProcessorNames =
            beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);

    // Separate between BeanFactoryPostProcessors that implement PriorityOrdered,
    // Ordered, and the rest.
    List<BeanFactoryPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
    List<String> orderedPostProcessorNames = new ArrayList<>();
    List<String> nonOrderedPostProcessorNames = new ArrayList<>();
    for (String ppName : postProcessorNames) {
        if (processedBeans.contains(ppName)) {
            // skip - already processed in first phase above
        }
        else if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
            priorityOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanFactoryPostProcessor.class));
        }
        else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
            orderedPostProcessorNames.add(ppName);
        }
        else {
            nonOrderedPostProcessorNames.add(ppName);
        }
    }

    // First, invoke the BeanFactoryPostProcessors that implement PriorityOrdered.
    sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
    invokeBeanFactoryPostProcessors(priorityOrderedPostProcessors, beanFactory);

    // Next, invoke the BeanFactoryPostProcessors that implement Ordered.
    List<BeanFactoryPostProcessor> orderedPostProcessors = new ArrayList<>(orderedPostProcessorNames.size());
    for (String postProcessorName : orderedPostProcessorNames) {
        orderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
    }
    sortPostProcessors(orderedPostProcessors, beanFactory);
    invokeBeanFactoryPostProcessors(orderedPostProcessors, beanFactory);

    // Finally, invoke all other BeanFactoryPostProcessors.
    List<BeanFactoryPostProcessor> nonOrderedPostProcessors = new ArrayList<>(nonOrderedPostProcessorNames.size());
    for (String postProcessorName : nonOrderedPostProcessorNames) {
        nonOrderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
    }
    invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);

    // Clear cached merged bean definitions since the post-processors might have
    // modified the original metadata, e.g. replacing placeholders in values...
    beanFactory.clearMetadataCache();
}
```