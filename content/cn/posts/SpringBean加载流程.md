---
title: "Spring Bean 加载"
date: 2022-10-05
categories: ['Spring']
draft: false
---

本文探索Spring Bean实例化的过程，由于Spring源码很复杂，所以这里只是摘取一些网上道听途说的文字讲解。

## 容器启动

Spring的类加载用到了工厂模式，Spring中有一个接口定义了这个工厂要干什么，这就是BeanFactory。

Spring 中用于类加载的IOC容器有DefaultListAbleFactory、XmlBeanFactory、ApplicationContext等。这些都实现了BeanFactory，并在其基础上添加了一些功能。

## 读取Bean元数据
两种扫描元数据的方法，一是扫描指定包路径下的Spring注解，如@Component、@Service等，二是读取xml配置的属性。Spring会将读取到的数据装配到BeanDefinition中，BeanDefinition中存储了加载类的各种属性，如属性值、是否是单例。再将这些BeanDefinition存储到一个Map中——BeanDefinationRegistry。

如果不是懒加载，则在容器启动完成后会执行finishBeanFactoryInitialization()方法，隐式调用所有对象的getBean()方法，实例化所有Bean。

## Bean实例化

doGetBean()方法会先去缓存或BeanFactory中查看是否存在Bean，如果存在则返回，否则创建。

创建过程：

1. 检查缓存中是否存在实例化Bean，若存在则直接返回
2. 如果没有，检查是否存在依赖循环，存在则报错。获取父工厂。
3. 通过父工厂创建
   a) 从一级缓存中获取，若为空，则检查是否在创建中
   b) 检查的时候要对一级缓存加锁，防止并发创建
   c) 如果正在加载，则无需操作。否则创建Bean，并将其存放在三级缓存中

## Bean初始化

1. 处理Aware接口
   > ①如果这个Bean实现了BeanNameAware接口，会调用它实现的setBeanName(String beanId)方法，传入Bean的名字；
②如果这个Bean实现了BeanClassLoaderAware接口，调用setBeanClassLoader()方法，传入ClassLoader对象的实例。
②如果这个Bean实现了BeanFactoryAware接口，会调用它实现的setBeanFactory()方法，传递的是Spring工厂自身。
③如果这个Bean实现了ApplicationContextAware接口，会调用setApplicationContext(ApplicationContext)方法，传入Spring上下文；
————————————————
版权声明：本文为CSDN博主「张维鹏」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/a745233700/article/details/113840727
2. 执行BeanPostProcessor前置处理
3. 调用自定义的 init 方法
4. 执行BeanPostProcessor后置处理
5. 注册 disposableBean