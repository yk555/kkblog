---
title: Bean的类型以及作用域
date: 2022-08-04 00:00:00
tags: Springboot
categories: Springboot
---



## Bean的类型

### 普通Bean

像类似于这样创建的bean都是普通bean

```java
@Component
public class Child {
}
```

> @component （把普通pojo实例化到spring容器中，相当于配置文件中的 <bean id="" class=""/>）
> 泛指各种组件，就是说当我们的类不属于各种归类的时候（不属于@Controller、@Services等的时候），我们就可以使用@Component来标注这个类。
>
> 

```java
@Bean
public Child child() {
    return new Child();
}
```

```java
<bean class="com.linkedbear.spring.bean.a_type.bean.Child"/>
```

### FactoryBean



SpringFramework 考虑到创建一些复杂的Bean时利用创建简单的Bean的方式很难做到。

借助 ==**FactoryBean**== 就比较容易做到。

`FactoryBean` 本身是一个接口，它本身就是一个创建对象的工厂。如果 Bean 实现了 `FactoryBean` 接口，则它本身将不再是一个普通的 Bean ，不会在实际的业务逻辑中起作用，而是由创建的对象来起作用。

FactoryBean的接口有三个方法

```Java
public interface FactoryBean<T> {
    // 返回创建的对象
    @Nullable
    T getObject() throws Exception;

    // 返回创建的对象的类型（即泛型类型）
    @Nullable
    Class<?> getObjectType();

    // 创建的对象是单实例Bean还是原型Bean，默认单实例
    default boolean isSingleton() {
        return true;
    }
}
```

### FactoryBean 与 Bean 同时存在

如过向IOC容器预先创建一个bean，Factory再创建相同类型的时，就会抛出`NoUniqueBeanDefinitionException`异常，说明创建的Bean是直接放在IOC容器中的

### FactoryBean加载时机

**`FactoryBean` 本身的加载是伴随 IOC 容器的初始化时机一起的**。

### FactoryBean创建Bean的时机

**`FactoryBean` 生产 Bean 的机制是延迟生产**。

### 取出FactoryBean的本体

只需要在Bean的id前面加上&

```Java
System.out.println(ctx.getBean("&toyFactory"));	
```





### 【面试题】BeanFactory与FactoryBean的区别

- **BeanFactory**：beanfactory是最基础的IOC容器，从类的继承结构上看，它是最顶级的接口，也就是最顶层的容器实现；从类的组合结构上看，它则是最深层次的容器，`ApplicationContext` 在最底层组合了 `BeanFactory` ）ApplicationContext其实也是一种IOC容器。

- **FactoryBean**：创建对象的工厂 Bean ，可以使用它来直接创建一些初始化流程比较复杂的对象。该接口可以按照用户的需求来构造 `Bean` 对象，而不再遵守 `Bean` 生命周期的流程。

罗列一下他们的方法

![image-20220803165521714](https://xingqiu-tuchuang-1256524210.cos.ap-shanghai.myqcloud.com/2054/image-20220803165521714.png)

![image-20220803165608291](https://xingqiu-tuchuang-1256524210.cos.ap-shanghai.myqcloud.com/2054/image-20220803165608291.png)

## Bean的作用域



1. singleton 一个IOC容器一个
2. prototype 每次获取一个
3. request  一次请求创建一个
4. session 一次会话创建一个
5. application  一个web应用创建一个
6. websocket  一个WebSocket会话创建一个



















