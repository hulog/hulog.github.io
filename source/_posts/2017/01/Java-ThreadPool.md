---
title: 一文带你进入Java之ThreadPool
date: 2017-01-19 23:10:21
categories: java
tags: [java,线程池]
---


## 1.简介
> 　　在计算机程序设计中，线程池是一个在计算机程序中实现并发执行的软件设计模式。一个线程池保持多个线程等待任务分配给并发执行的监督程序。通过维护一个线程池的模型，提高性能，例如，对于执行时间较短的任务，避免了由于频繁创建和销毁线程造成的系统消耗。——[维基百科](https://en.wikipedia.org/wiki/Thread_pool)

<!--more-->

　　**个人理解：**线程池就相当于一个处理任务的线程工厂，里面有很多工人（线程），当任务来了的时候，可以让工人立即开始工作（线程执行），当任务处理完了，则可以让工人休息（sleep）。所以，处理任务时，我们不用花时间单独去外面请工人（线程的创建），完事后不用辞退工人（线程的销毁），在任务量比较庞大的时候，能够显著的提高系统的处理能力。

　　其作用总结如下：
1. 控制和管理线程；
2. 显著减少CPU闲置时间；
3. 提升吞吐能力。

*`tips`:本讲的线程池主要是针对Java自带的java.util.concurrent包*。

## 2.使用场景

那什么时候可以考虑上线程池呢？首先，对于线程，可以粗略的分为三个周期：

|T<sub>1</sub>|T<sub>2</sub>|T<sub>3</sub>|
|:--:|:--:|:--:|
|线程创建|线程执行|线程销毁|

当**T<sub>1</sub>+T<sub>3</sub>>>T<sub>2</sub>**时，可以考虑上线程池。对于如何估算各个周期的执行时间，可以粗略分析是否是CPU密集型任务，如果不是，举个极端例子：求1+1=?，那么线程执行周期T<sub>2</sub>就明显很短，创建和销毁时间远大于执行时间。此时就可以考虑上线程池了。

那么，很多童鞋会有个疑惑，线程池与`new Thread()`有什么区别呢？线程池的好处在于：

1. 重用存在的线程，减少对象创建、消亡的开销，性能佳。
2. 可有效控制最大并发线程数，提高系统资源的使用率，同时避免过多资源竞争，避免堵塞。
3. 提供定时执行、定期执行、单线程、并发数控制等功能。

相反，`new Thread()`方法只是单纯的创建线程，注重单个线程本身。当启动多个线程时，需循环调用`new Thread()`方法，耗费大量时间在创建和销毁线程上。

## 3. 重要组成部分（类）

Java中线程池的顶级接口是`Executor`，里面`只`提供了一个方法`void execute(Runnable command);`,可以看出来它只是提供了一个线程执行的工具类，所以我们更认同地将其子类`ExecutorService`视为线程池真正的接口。


![线程池相关类](http://upload-images.jianshu.io/upload_images/2803944-443efc5a7ea3facb.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


具体介绍下面继续，废话不多说，赶紧的先建个线程池出来溜溜～～～
创建线程池的方法有很多种，我们快马加鞭，来个最省事儿的，傻瓜式的创建线程池，不得不先提出`Executors`类（注意带`s`），本类为创建线程池的工具类（了解Java集合的童鞋，可以类比`Collections`类与`Collection`接口）。

### 3.1 Executors类

该类提供了创建线程池的方法，比较常用的如下：

1. `newSingleThreadExecutor();`
2. `newFixedThreadPool(int nThreads);`
3. `newCachedThreadPool();`
4. `newScheduledThreadPool(int corePoolSize);`

以上方法都会返回一个线程池，只是各自的功能不一样，下面分别介绍各自的实现和使用场景。

#### 3.1.1 newSingleThreadExecutor();
```java
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
          (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
}
```

创建一个单线程的线程池，池中保持单个线程串行执行任务，如果线程因异常结束，则会创建一个新的线程来替代它，可以保证所有任务的执行顺序按照任务的提交顺序执行。

#### 3.1.2 newFixedThreadPool(int nThreads)

```java
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}
```

创建一个固定大小可重用线程的线程池，任何时候，顶多有nThreads个线程处于活跃状态执行任务。当nThreads个线程满负荷运转时，新增的任务会加到无界队列里等候，直到有空闲线程来处理。当线程因异常退出后，会创建一个新线程来替代。在某个线程被显式地关闭之前，池中的线程将一直存在。

#### 3.1.3 newCachedThreadPool();

```java
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}
```

创建一个可根据需要创建新线程的线程池，优先重用已创建的可用的线程，该线程池可以显著的提高程序的性能。当没有可用的线程时，则会在池中创建新的线程。当线程没有被使用超过60s，则会从池中remove掉，最低数量为0。因此，长时间保持空闲的线程池不会消耗任何资源。但是，当出现新任务时，又要创建一新的工作线程，又要一定的系统开销。并且，在使用CachedThreadPool时，一定要注意控制任务的数量，否则，由于大量线程同时运行，很有会造成系统瘫痪。可以使用` ThreadPoolExecutor `构造方法（**后文会重点讲到**）创建具有类似属性但细节不同（例如超时参数）的线程池。

#### 3.1.4 newScheduledThreadPool(int corePoolSize);

```java
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
    return new ScheduledThreadPoolExecutor(corePoolSize);
}
```

```java
public class ScheduledThreadPoolExecutor
    extends ThreadPoolExecutor
    implements ScheduledExecutorService {

	//ScheduledThreadPoolExecutor类的构造方法，其余方法和变量略
    public ScheduledThreadPoolExecutor(int corePoolSize) {
        super(corePoolSize, Integer.MAX_VALUE, 0, TimeUnit.NANOSECONDS,
              new DelayedWorkQueue());
    }
 }
```

创建一个能在指定时间后或周期性地执行任务的线程池，池中会保持corePoolSize个线程，即使处于空闲状态。


### 3.2 ThreadPoolExecutor类

可以看出，上面四种线程池都基本上是基于`ThreadPoolExecutor`和`ScheduledThreadPoolExecutor`来实现的。在此，我们主要讲解前者，了解其构造函数的各个参数的实际意义。

一切没有源码的解释都是耍流氓。

```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {
    if (corePoolSize < 0 ||
        maximumPoolSize <= 0 ||
        maximumPoolSize < corePoolSize ||
        keepAliveTime < 0)
        throw new IllegalArgumentException();
    if (workQueue == null || threadFactory == null || handler == null)
        throw new NullPointerException();
    this.corePoolSize = corePoolSize;
    this.maximumPoolSize = maximumPoolSize;
    this.workQueue = workQueue;
    this.keepAliveTime = unit.toNanos(keepAliveTime);
    this.threadFactory = threadFactory;
    this.handler = handler;
}
```

|参数名|作用|
|---|---|
|`corePoolSize`|线程池维护的核心线程数量。当超过这个范围的时候，就需要将新的Runnable放入到等待队列workQueue中了|
|`maximumPoolSize`|线程池维护的最大线程数量。如果队列满了，并且已创建的线程数小于最大线程数，则线程池会再创建新的线程执行任务。如果使用了无界的`workQueue`任务队列这个参数就没效果|
|`keepAliveTime`|线程池中超过`corePoolSize`的线程的存活时间|
|`unit`|`keepAliveTime`的时间单位|
|`workQueue`|线程池所使用的缓冲队列。用于保存等待执行的任务，常见的队列有`ArrayBlockingQueue`，`LinkedBlockingQueue`和`SynchronousQueue`(*区别见**注1***)|
|`threadFactory`|创建新线程所使用的线程工厂。可以通过线程工厂给每个创建出来的线程设置自定义名字，主要实现newThread方法即可|
|`handler`|参数`maximumPoolSize`达到后丢弃处理的方法，常见的策略有`AbortPolicy`，`CallerRunsPolicy`，`DiscardOldestPolicy`和`DiscardPolicy`(*区别见**注2***)。可以根据应用场景需要来实现RejectedExecutionHandler接口的rejectedExecution方法，来实现自定义策略，如记录日志或持久化不能处理的任务|

+ **注1**：
 1. `ArrayBlockingQueue`: 基于数组的有界队列。有助于防止资源耗尽，但较难控制大小，需要考虑池大小和队列的大小的折衷，大型池小型队列cpu使用率较高，但是请求量很大时，可能遇到不可接受的调度开销。小型池大型队列会降低cpu使用率，避免频繁的线程切换导致的系统消耗，但处理速率也就下降了。值得注意的是，在生产者放入数据和消费者获取数据，都是共用同一个锁对象，由此也意味着两者无法真正并行运行，这点尤其不同于LinkedBlockingQueue。
 2. `LinkedBlockingQueue`: 基于链表的“无界”队列。实际上具有*类似无限大小*的容量(Integer.MAX_VALUE)，也可以在构造函数中指定大小。LinkedBlockingQueue之所以能够高效的处理并发数据，还因为其对于生产者端和消费者端分别采用了独立的锁来控制数据同步，这也意味着在高并发的情况下生产者和消费者可以并行地操作队列中的数据，以此来提高整个队列的并发性能。
 3. `SynchronousQueue`: 无缓冲的等待队列，类似于无中介的直接交易，其特点是读取交替完成，没有实际容量，它将任务直接提交。对于SynchronousQueue的作用jdk中写的很清楚：此策略可以避免在处理可能具有内部依赖性的请求集时出现锁。举个例子，如果你的任务A<sub>1</sub>，A<sub>2</sub>有内部关联，A<sub>1</sub>需要先运行，那么先提交A<sub>1</sub>，再提交A<sub>2</sub>，当使用SynchronousQueue我们可以保证，A<sub>1</sub>必定先被执行，在A<sub>1</sub>没有被执行前，A<sub>2</sub>不可能添加入queue中。
+ **注2**：
 1. `AbortPolicy` : java默认，抛出一个异常：RejectedExecutionException。
 2. `CallerRunsPolicy` : 如果发现线程池还在运行，就直接运行这个线程的run()方法。
 3. `DiscardOldestPolicy` : 在线程池的等待队列中，将队首任务抛弃，使用当前任务来替换。
 4. `DiscardPolicy` : 什么也不做。

这一块不清楚的可以参看[Java线程池架构(一)原理和源码解析](http://segmentfault.com/a/1190000000394999?utm_source=tuicool&utm_medium=referral)

*tips：下面就是我看完某篇博文收到启发，举一个经典的例子，大家可以按照这个思路去理解。*

把线程池理解成一个医院，在医院成立之初，医生数量为 **0**，当有患者时，没有医生来诊疗患者，医院会去招聘新的医生，一旦这些医生忙不过来时，继续招聘，直到达到`corePoolSize`数量，停止招聘。此时的`corePoolSize`个医生为正式员工，即使没有患者，也不会辞退他们（*销毁线程*）。

医生达到`corePoolSize`后，当有新患者来就诊，医生忙不过来时，直接让他们在候诊区（`workQueue`）取号等候，当医生看完上一个病人时，会去候诊区叫下一个号进去，如果没有患者，则可以休息。

当患者数量急剧上升，候诊区座位数不够了，这时，医院会再去招聘临时工医生，这些临时工医生会让没有座位的患者立即就诊，医院按需求逐个招聘，直到达到`maximumPoolSize`数量，停止招聘。

当临时招聘的医生长时间（`keepAliveTime`）处于空闲状态时，医院就会解雇他们，毕竟要额外付工资啊～

## 4. 总结 ##

**综上**，文中提到创建线程池的方式有两种:

1. 通过Executors类提供的静态工厂方法，例如：
```java
ExecutorService es = Executors.newFixedThreadPool(nThreads);
```
2. 通过ThreadPoolExecutor来构造，例如：
```java
ExecutorService es =
    new ThreadPoolExecutor(corePoolSize,maximumPoolSize,
        keepAliveTime,timeUnit,workQueue);
```
其中，如果没有特殊要求，使用第一种方法可以快速构建出线程池。如果根据业务不同，需要自定义线程池，第二种方法将给你充分的发挥空间。

下篇博文将会利用线程池基于Socket实现**客户端->服务器**文件的传输，将会有大量实例代码。
