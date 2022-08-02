---
title: Spring自动装配原理
date: 2022-08-01 00:00:00
tags: Springboot
categories: Springboot
---

# 一句话解释

提到自动装配原理，就会提到Springboot的注解@SpringBootApplication，这个注解由@SpringbootConfiguration 和 @EnableAutoConfiguration以及@ComponentScan最重要的注解组成。其中@EnableAutoConfiguration注解通过读取META-INF/spring.factories文件中的信息，以读取一些基础配置，比如redis、kafka。@ComponentScan注解则是用来扫描应用程序中的配置类，比如自己写的bean的配置类。@SpringbootConfiguration，功能与@Configuration差不多，其加在启动类上的作用是，将启动类也标志为配置类，在启动时，将自己也扫描进去。



# 详细的解释

@SpringBootApplication的其中3个重要注解：

- @SpringBootConfiguration 标识是一个配置类
- @EnableAutoConfiguration 启用Spring Boot 的自动配置机制
- @ComponentScan 扫描应用程序中的配置类

## 对注解 @EnableAutoConfiguration 的解释

看了几片八股文，说一说自己的理解吧

![image-20220801230956201](https://xingqiu-tuchuang-1256524210.cos.ap-shanghai.myqcloud.com/2054/image-20220801230956201.png)

@EnabkeAutoConfiguration中有一个重要的注解--@Import ，import的内容按照字面意思则是自动配置导入选择器。

其中有个process（）方法，方法中调用getAutoConfigurationEntry方法

![image-20220801232327905](https://xingqiu-tuchuang-1256524210.cos.ap-shanghai.myqcloud.com/2054/image-20220801232327905.png)

再进入其中，其中调用了getCandidateConfigurations方法

![image-20220801232456680](https://xingqiu-tuchuang-1256524210.cos.ap-shanghai.myqcloud.com/2054/image-20220801232456680.png)

再进入getCandidateConfigurations方法，方法中调用了SpringFactoriesLoader的loadFactoryNames方法

![image-20220801232543951](https://xingqiu-tuchuang-1256524210.cos.ap-shanghai.myqcloud.com/2054/image-20220801232543951.png)

再进入loadFactoryNames其中，调用了loadSpringFactories方法

![image-20220801232943516](https://xingqiu-tuchuang-1256524210.cos.ap-shanghai.myqcloud.com/2054/image-20220801232943516.png)

最后会发现，这里加载了META-INF/spring.factories中的配置信息。

![image-20220801233037132](https://xingqiu-tuchuang-1256524210.cos.ap-shanghai.myqcloud.com/2054/image-20220801233037132.png)

我们进入spring-boot-autoconfigure包中的META-INF/spring.factories，能够看到类似于这样的一些配置类信息。

![image-20220801233331015](https://xingqiu-tuchuang-1256524210.cos.ap-shanghai.myqcloud.com/2054/image-20220801233331015.png)

# 总结

用官方的话来说，Springboot的自动配置，就是尝试根据用户添加的jar依赖项配置您的Spring应用程序。

要是想对自动配置的理解更深的话，应该去体会一下Spring是如何配置的。

# 参考的文章

[SpringBoot系列--自动装配原理1](https://learnku.com/articles/56208)

[Spring官方文档](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/using-spring-boot.html#using-boot-auto-configuration)

[自动装配脑图](https://www.processon.com/view/link/5efc4ffa6376891e81f3895d)

# Tips

create：2022/8/1 内容比较简陋，将会在后期深入学习后继续补充和更新相关内容。