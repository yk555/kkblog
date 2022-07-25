---
title: 并发基础中的Future异步回调模式
date: 2022-01-09 03:00:00
tags: 
 - 异步回调
 - Future
categories: netty
---

在netty中使用了很多回调技术，并且基于java的回调设计了一整套异步回调接口和实现。

这篇文章从Java Future入手，再到第三方异步回调技术--谷歌的Guava Future，最后介绍Netty的异步回调技术。



在文章中的所有demo都采用“泡茶”的案例

主线程：负责启动和等待其他线程

清洗线程

烧水线程

## Join异步阻塞

join的原理与作用是，阻塞当前线程，直到准备合并的目标线程的执行完成。

```java
package org.example;

public class JoinDemo {
    public static final int SLEEP_GAP = 5000;
    public static String getCurThreadName() {
        return Thread.currentThread().getName();
    }
    static class HotwaterThread extends Thread {
        public HotwaterThread() {
            super("** 烧水-Thread");
        }
        public void run() {
            try {
                System.out.println("洗好茶壶");
                System.out.println("灌水");
                System.out.println("放在火上");
                Thread.sleep(SLEEP_GAP);
                System.out.println("水烧开了");
            } catch (InterruptedException e) {
                System.out.println("发送异常被中断");
            }
            System.out.println("运行结束");
        }
    }
    static class WashThread extends Thread {
        public WashThread() {
            super("$$ 清洗-Thread");
        }
        public void run() {
            try {
                System.out.println("洗茶壶");
                System.out.println("洗茶杯");
                System.out.println("拿茶叶");
                Thread.sleep(SLEEP_GAP);
                System.out.println("洗完了");
            } catch (InterruptedException e) {
                System.out.println("发送异常被中断");
            }
            System.out.println("运行结束");
        }
    }
    public static void main(String[] args) {
        Thread h = new HotwaterThread();
        Thread w = new WashThread();
        h.start();
        w.start();
        try {
            h.join();
            w.join();
            Thread.currentThread().setName("主线程");
            System.out.println("泡茶喝");
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
        System.out.println("end");
    }
}
```

### join有三个重载版本：

- void join () : A线程等待B线程执行结束后，A线程重新恢复执行
- void join(long millis) ：A线程等待B线程最长millis毫秒，超过后A重新恢复执行
- void join(long millis, int nanos)A线程等待B线程最长mills毫秒 + nanos纳秒

### 容易混淆的地方：

1. join是实例方法，不是静态方法，所以需要用线程对象去调用join，例如thread.join
2. join调用时，不是线程所指向的目标线程阻塞，而是当前线程阻塞。
3. 只有等到目标线程完成或超时，当前线程才能恢复执行

### join的问题:

被合并的线程没有返回值，即调用join后，没有返回值。

例如，在上述代码中，main进程无法得知其他线程的运行结果。



所以想要获得异步线程的执行结果，下面就介绍Java的FutureTask系列



## FutureTask异步回调



FutureTask方式在java.util.concurrent包中。最重要的就是**FutureTask 类**和 **Callable接口**



### Callable接口

新事物的出现，一定是为了解决旧事物的一些缺陷。

提到Callable接口，就离不开Runnable接口，Runnable有一个重要的问题，即它的run方法没有返回值的。所以Runnable不能用于需要有返回值的场景。

Runnable接口是在Java多线程中表示线程的业务代码的抽象接口

为了解决Runnable的问题，所以才有了Callable

```java
package java.util.concurrent;
@FunctionalInterface
public interface Callable<V> {
    V call() throws Exception;
}
package java.lang;
@FunctionalInterface
public interface Runnable {
    public abstract void run();
}
```

Callable是一个泛型接口，唯一方法call的返回值类型为泛型形参的实际类型，并且还有一个Exception，容许方法内部的异常不经过捕获。

阿Callable方法可以与Runnable相对应，可以看成更强大的Runnable。

**但他们也有不同的地方：**

- Callable接口的实例不能作为Thread线程实例的target来使用。Runnable接口实例可以作为Thread线程实例的target构造参数，开启一个Thread线程。

target是什么？

查看Thread源码可以得知，target是**Runable**类的，在Thread初始化时有使用。target作为参数传入Thread构造函数，构造函数调用init函数初始时也将target传入。并且在thread run中调用的也是Runnable的run方法。

```java
public class Thread implements Runnable {
    
    /* What will be run. */
    private Runnable target;
    @Override
    protected Object clone() throws CloneNotSupportedException {
        throw new CloneNotSupportedException();
    }
	//Thread的构造器有很多种，只列了一种
    public Thread(Runnable target) {
        init(null, target, "Thread-" + nextThreadNum(), 0);
    }   
    @Override
    public void run() {
        if (target != null) {
            target.run();
        }
    }

 
   
}
```

但是呢，java的线程类型只有Thread，callable要是想利用Thread的话，就要想办法将自己赋值给Runnable类型的target，所以需要一个FutureTask类来进行所谓的搭桥。



### FutureTask类

```java
public class FutureTask<V> implements RunnableFuture<V>
public interface RunnableFuture<V> extends Runnable, Future<V> 
public FutureTask(Callable<V> callable) {
        if (callable == null)
            throw new NullPointerException();
        this.callable = callable;
        this.state = NEW;       // ensure visibility of callable
    }
```

直接查看源码可以发现，FutureTask实现了RunnableFuture，而RunnableFuture则继承了Runnable和Future

所以可以作为target传递给Thread。但是想要获取到Callable的异步执行结果就不应该使用它的call方法，而是通过FutureTask类的相应方法去获取。FutureTask实现了Future接口，Future接口则是提供了判断、获取、取消并发任务结果的功能。

```java
package java.util.concurrent;
public interface Future<V> {
    boolean cancel(boolean mayInterruptIfRunning);
    boolean isCancelled();
    boolean isDone();
    V get() throws InterruptedException, ExecutionException;
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```

- Future接口提供的get方法是阻塞的，如果并发任务没有完成就会等待并发任务完成。可以增加时间约束。

总结一下，FutureTask通过间接继承了Runnable，从而能够作为Thread的target, 然后Callable作为参数传入其中，此外Future提供了获取异步任务执行结果的方法，FutureTask实现了其方法。到这里Future成功搭起来看Callable和Thread的桥梁，并且实现获取异步任务结果的方式。



回到FutureTask中，变量Callable代表异步执行的逻辑，我们需要保存结果，保存在哪呢？

查看源码可以看到，FutureTask中有一个run方法是Runnable接口的，它能作为Target目标去执行，然后

```java
 public void run() {
        if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                         null, Thread.currentThread()))
            return;
        try {
            Callable<V> c = callable;
            if (c != null && state == NEW) {
                V result;
                boolean ran;
                try {
                    result = c.call();
                    ran = true;
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    setException(ex);
                }
                if (ran)
                    set(result);
            }
        } finally {
            // runner must be non-null until state is settled to
            // prevent concurrent calls to run()
            runner = null;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts
            int s = state;
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
    }
```

run方法中有个set方法

```java
protected void set(V v) {
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
            outcome = v;
            UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
            finishCompletion();
        }
    }
```

最终保存在outcome属性中

private Object outcome

```java
package org.example;

import java.util.concurrent.Callable;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.FutureTask;

public class JavaFutureDemo {
    public static final int SLEEP_GAP = 5000;
    public static String getCurThreadName() {
        return Thread.currentThread().getName();
    }
    static class HotWaterJob implements Callable<Boolean> {
        @Override
        public Boolean call() throws Exception {
            try {
                System.out.println("洗好茶壶");
                System.out.println("灌水");
                System.out.println("放在火上");
                Thread.sleep(SLEEP_GAP);
                System.out.println("水烧开了");
            } catch (InterruptedException e) {
                System.out.println("发送异常被中断");
                return false;
            }
            System.out.println("运行结束");
            return true;
        }
    }
    static class WashJon implements Callable<Boolean> {
        @Override
        public Boolean call() throws Exception {
            try {
                System.out.println("洗茶壶");
                System.out.println("洗茶杯");
                System.out.println("拿茶叶");
                Thread.sleep(SLEEP_GAP);
                System.out.println("洗完了");
            } catch (InterruptedException e) {
                System.out.println("发送异常被中断");
                return false;
            }
            System.out.println("运行结束");
            return true;
        }
    }
    public static void drinkTea(boolean waterOK, boolean cupOk) {
        if (waterOK && cupOk) {
            System.out.println("泡茶喝");
        } else if (!waterOK) {
            System.out.println("烧水失败，没有茶喝了");
        } else if (!cupOk) {
            System.out.println("杯子洗不了，没有茶喝了");
        }
    }
    public static void main(String args[]) {
        Callable<Boolean> hJob = new HotWaterJob();
        FutureTask<Boolean> hTask = new FutureTask<Boolean>(hJob);
        Thread hThread = new Thread(hTask, "** 烧水-Thread");

        Callable<Boolean> wJob = new WashJon();
        FutureTask<Boolean> wTask = new FutureTask<Boolean>(wJob);
        Thread wThread = new Thread(wTask, "$$ 清洗-Thread" );

        hThread.start();
        wThread.start();

        Thread.currentThread().setName("主线程");
        try {
            boolean watereOk = hTask.get();
            boolean cupOk = wTask.get();
            drinkTea(watereOk, cupOk);
        } catch (ExecutionException e) {
            throw new RuntimeException(e);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
        System.out.println("end");
    }
}
```



虽然可以通过get获取异步执行的结果，但是并不高明，因为get实际上还是阻塞的，就是主线程调用get时，主线程也会阻塞，和join差不多。

原生JavaAPI除了阻塞模式的获取结果外，没有实现非阻塞的，需要引入别的框架比如谷歌的Guava。



## 谷歌Guava的异步回调

Guava是谷歌公司提供的Java扩展包，是一种异步回调的解决方案。

- 新增了一个ListenableFuture接口，继承了Java的Future接口，使得Java的Future异步任务，在Guava中能够被监控和获得非阻塞异步执行的结果
- 引入一个新的接口FutureCallback，根据异步执行结果完成不同的回调处理，能够处理异步结果啦。

### FutureCallback

负责异步任务完成后的监听逻辑，相当于处理异步结果

**有两个方法：**

- onSuccess 异步任务成功后被回调，异步任务的结果作为onSuccess的参数
- onFailure 异步任务异常被回调，异步任务的结果作为onFailure的参数

```java
@GwtCompatible
public interface FutureCallback<V> {
  void onSuccess(@Nullable V result);
  void onFailure(Throwable t);
}
```

**看起来与Callable相似，但完全不一样：**

Callable代表异步执行逻辑

FutureCallback代表的是异步任务完成后对结果的处理工作



Guava FutureCallback只是对Java Future异步执行的增强，所以使用Guava需要Java Callable，简单来说只有Java的Callable任务执行的结果出来以后才能执行FutureCallback

如何实现这种监控关系呢？

引入ListenableFuture，获取异步执行结果

### ListenableFuture

这里直接上源码

```java
@GwtCompatible
public interface ListenableFuture<V> extends Future<V> {
  void addListener(Runnable listener, Executor executor);
}
```

ListenableFuture在Future的基础上增加了addListener方法，目的是将FutureCallback的任务封装成一个内部的Runnable异步回调任务。在Callable异步完成后，回调。但是这个方法只在内部使用，一般不调用。



那如何将FutureCallback绑定到ListenableFuture任务呢？

使用Guava的Futures工具类，addcallback可以绑定。

```java
ListenableFuture<Boolean> hotFuture = gPool.submit(hotJob);
        Futures.addCallback(hotFuture, new FutureCallback<Boolean>() {
            public void onSuccess(Boolean r) {
                if (r) {
                    mainJob.waterOk = true;
                }
            }

            public void onFailure(Throwable throwable) {
                System.out.println("烧水失败");
            }
        });
```

如果Guava的ListenableFuture 是 对Java Future 的扩展，都表示异步任务，那Guava的异步任务从何而来？



ListenableFuture的异步任务异步任务的实例主要是通过向线程池提交Callable任务的方式获取，这里的线程池是Guava的不是Java的，另外FutureTask的异步任务实例是直接将Callable传入构造器。



```java
ExecutorService jPool = Executors.newFixedThreadPool(10);
ListeningExecutorService gPool = MoreExecutors.listeningDecorator(jPool);
```

流程：

1. 创建Callable异步任务
2. 创建Guava线程池
3. 提交Callable获取ListenableFuture异步任务实例
4. 创建FutureCallback，绑定ListenableFuture



泡茶实例

```java
package org.example;

import com.google.common.util.concurrent.*;

import javax.annotation.Nullable;
import java.util.concurrent.Callable;
import java.util.concurrent.Executor;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class GuavaFutureDemo {
    public static final int SLEEP_GAP = 5000;
    public static String getCurThreadName() {
        return Thread.currentThread().getName();
    }

    static class HotWaterJob implements Callable<Boolean> {
        @Override
        public Boolean call() throws Exception {
            try {
                System.out.println("洗好茶壶");
                System.out.println("灌水");
                System.out.println("放在火上");
                Thread.sleep(SLEEP_GAP);
                System.out.println("水烧开了");
            } catch (InterruptedException e) {
                System.out.println("发送异常被中断");
                return false;
            }
            System.out.println("运行结束");
            return true;
        }
    }
    static class WashJon implements Callable<Boolean> {
        @Override
        public Boolean call() throws Exception {
            try {
                System.out.println("洗茶壶");
                System.out.println("洗茶杯");
                System.out.println("拿茶叶");
                Thread.sleep(SLEEP_GAP);
                System.out.println("洗完了");
            } catch (InterruptedException e) {
                System.out.println("发送异常被中断");
                return false;
            }
            System.out.println("运行结束");
            return true;
        }
    }
    static class MainJob implements Runnable {
        boolean waterOk = false;
        boolean cupOk = false;
        int gap = SLEEP_GAP / 10;
        @Override
        public  void run() {
            while (true) {
                try {
                    Thread.sleep(gap);
                    System.out.println("读书中");
                } catch (InterruptedException e) {
                    System.out.println(getCurThreadName() + "中断");
                }
                if (waterOk && cupOk) {
                    drinkTea(waterOk, cupOk);
                    break;
                }
            }
        }
        public  void drinkTea(boolean waterOK, boolean cupOk) {
            if (waterOK && cupOk) {
                System.out.println("泡茶喝");
            } else if (!waterOK) {
                System.out.println("烧水失败，没有茶喝了");
            } else if (!cupOk) {
                System.out.println("杯子洗不了，没有茶喝了");
            }
        }
    }
    public static void main(String[] args) {
        final MainJob  mainJob = new MainJob();
        Thread mainThread = new Thread(mainJob);
        mainThread.setName("主线程");
        mainThread.start();

        Callable<Boolean> hotJob = new HotWaterJob();
        Callable<Boolean> washJob = new WashJon();

        ExecutorService jPool = Executors.newFixedThreadPool(10);
        ListeningExecutorService gPool = MoreExecutors.listeningDecorator(jPool);

        ListenableFuture<Boolean> hotFuture = gPool.submit(hotJob);
        Futures.addCallback(hotFuture, new FutureCallback<Boolean>() {
            public void onSuccess(Boolean r) {
                if (r) {
                    mainJob.waterOk = true;
                }
            }

            public void onFailure(Throwable throwable) {
                System.out.println("烧水失败");
            }
        });
        ListenableFuture<Boolean> washFuture = gPool.submit(washJob);
        Futures.addCallback(hotFuture, new FutureCallback<Boolean>() {
            public void onSuccess(Boolean r) {
                if (r) {
                    mainJob.cupOk = true;
                }
            }

            public void onFailure(Throwable throwable) {
                System.out.println("杯子洗不了");
            }
        });
    }
}
```



Guava非阻塞的原因与理解



## Netty的异步回调模式

在netty源码中大量使用了异步回调模式，在netty业务开发层面，netty应用的是handle处理器中的业务处理代码也是异步的。所以了解netty的异步回调是十分重要的。

Netty扩展了Java Future的异步任务：

- 继承Java Future接口并增强，使其方法可以非阻塞。
- 引入GeneriFutureListener，用于表示异步任务完成的监控器，与Guava的FutureCallback不同，Netty使用的是监控器模式，异步任务完成后回调逻辑抽象成了Lisener监听器接口。可以将GenericFutureListener监听器加入Netty的异步任务Future中，实现对异步任务执行状态的事件监听。

在异步非阻塞回调思路上和Guava是一致的。

### GenericFutureListener接口

该接口位于io.netty.util.concurrent

直接上源码



GenericFutureListener拥有一个回调方法operationComplete,回调代码写在这。

而父接口EventListner是空接口，没有任何方法，起标识作用。

### Netty Future 接口

netty对Future进行了扩展，对执行的过程进行监控，对异步回调完成事件进行监听。

```java
 
```

Future接口一般不会直接使用，而是会使用子接口。Netty有一系列的子接口，代表不同类型的异步任务如ChannelFuture接口。

ChannelFuture表示通道IO操作的异步任务，如果在通道的异步IO操作完成后执行回调就需要ChannelFuture