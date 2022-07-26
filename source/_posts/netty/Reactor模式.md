---
title: Reacotor模式
date: 2022-01-08 03:00:00
tags: Reactor模式
categories: netty
---

## 什么是Reactor模式？

Reactor模式有两大角色组成：

1. Reactor反应器线程，负责响应IO事件，并且分发到Handles处理器中
2. Handles处理器，非阻塞的执行业务处理逻辑



## 为什么需要Reactor模式？

### 多线程OIO的缺陷

在最初的Java OIO程序中，使用的方式是while循环，不断地监听是否有新的连接。相当于，如果前面的handle没有处理完，就不能处理后面的连接。吞吐很低。

为了解决这种情况，后来出现了一个叫做connection per thread模式。就是为每一个连接分配一个线程，包括监听也是独立的线程。

如果在这种多线程OIO的情况下，每一个线程处理多个连接可以吗？当然是不可以的，因为这里的每个socket连接的读写操作都是阻塞的，无论如何都只能处理一个IO操作。缺点是，需要消耗大量的线程资源。

为了做到一个线程处理多个连接，引出反应器模式的简单版本

### 单线程Reactor反应器

即reactor反应器和handler处理器在同一个线程

缺点：由于在同一个线程，当某个handle阻塞时其他handle得不到执行。



为了真正解决问题，引入

### 多线程Reactor反应器

**升级：**

1. 升级handler处理器，线程池
2. 多个reactor处理器

将IOHandler和监听事件的反应器线程隔离，不会相互阻塞。



**反应器模式和生产者消费者模式的对比**

反应器是基于查询的，没有专门的队列去缓冲存储IO事件，查询到事件后，反应器会根据不同IO选择器分发。



**反应器模式和观察者模式的对比**

观察者模式，发布一个主题后，会通知所有观察着处理。

反应器模式一般一个事件绑定一个handler