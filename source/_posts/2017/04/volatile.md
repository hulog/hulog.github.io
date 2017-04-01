---
title: Java 之 volatile 关键字
date: 2017-04-02 01:32:17
description:
categories: [java, multithread]
tags: [多线程, 并发, 锁]
---

## 前言
 
　　要想进军多线程，玩玩高并发，那么你肯定知道线程同步，同步是为了协调多个竞争者对资源的同时访问。对应Java，我们第一反应可能会跳出`sychronized`关键字，这个关键字能够修饰`类`，`方法`，`静态方法`以及`代码块`。但是它的性能在高并发下是相当低下的，属于重量级锁。有的业务场景可能不需要这么重量级的锁（比如读多写少，我们如果能够保证获取到的值是最新的就OK），随着对JVM的深入了解，发现CPU不是每次都从内存中取值，各个内核都有自己的缓存，若每个内核中的线程同一时刻对共享变量进行操作，谁的结果才是正确的呢？这样就出现了各个线程的数据同步问题。
<!--more-->
## 情景

　　与synchronized相比，volatile所需的编码较少，运行时的开销也小，但是其功能也只有synchronized的一部分。程序员比较关心的无非是其使用场景，以及如何在代码中使用，本文接下来着重介绍这两方面。
　　在目前大多数的处理器架构上，volatile 读操作开销非常低 —— 几乎和非 volatile 读操作一样。而 volatile 写操作的开销要比非 volatile 写操作多很多，因为要保证可见性需要实现内存界定（Memory Fence），即便如此，volatile 的总开销仍然要比锁获取低。
　　我们听到的关于volatile最多的特性是`可见性`，但又不能保证`原子性`，什么鬼？这两个说的是什么意思？那么，我们用一个例子来表述。

```java
class A{
	private static int a;
	private static countA(){
		a++;
	}
	// ...
}
```

　　我们来分析一下`a++`这个操作，这个等同于`a=a+1`，这个里面包含了几个步骤呢？大家先思考一下～

---

　　`1.`首先，我们的CPU是要去取出a的当前值；`2.`然后执行+1操作；`3.`最后把结果赋值给左边的a。
　　接下来，我们根据这个步骤，解释上面的两个名词。

1. **取值**
　　现在的计算机处理器几乎都是多核，不是我们在学校里书本上学的单核，多个核就相当于多个CPU，能够并行地处理任务。假设4 个核，那么就有可能出现，四个线程分别在四个核中同时执行`a++`。相对于内存io，cpu的执行速度是相当快的，二者的速度不匹配，所以在cpu和内存中间，缓存主要解决这个问题。如图
![@CPU与内存之间的缓存 | center](http://ojxisatrc.bkt.clouddn.com/image/multithread/volatile01.jpgafsd)  
图中，L1，L2，L3为三级缓存，越靠近cpu，速度越快，容量也越小。对于多核，情况如下：
![@CPU与内存之间的缓存(多核) | center](http://ojxisatrc.bkt.clouddn.com/image/multithread/volatile02.jpgzz)
　　cpu将会默认从缓存中取a的值，四个处理器中都缓存了a的值，那么谁的值才是最新的/正确的呢？volatile就能保证每个线程取到的值是最新的，即`可见性`。那么volatile如何保证可见性的呢？
　　在x86处理器下[(原文)](http://www.infoq.com/cn/articles/ftf-java-volatile)通过工具获取JIT编译器生成的汇编指令来看看对Volatile进行写操作CPU会做什么事情。

| Java代码 | instance = new Singleton();//instance是volatile变量 |
| --------: | :-------- |
| 汇编代码 | 0x01a3de1d: movb $0x0,0x1104800(%esi); |
| |0x01a3de24: **lock** addl $0x0,(%esp); |

　　有volatile变量修饰的共享变量进行写操作的时候会多第二行汇编代码，通过查IA-32架构软件开发者手册可知，**lock**前缀的指令在多核处理器下会引发了两件事情。
　　`a`. **将当前处理器缓存行的数据会写回到系统内存**。
　　`b`. **这个写回内存的操作会引起在其他CPU里缓存了该内存地址的数据无效**。
　　【`强烈推荐看完==>`】*处理器为了提高处理速度，不直接和内存进行通讯，而是先将系统内存的数据读到内部缓存（L1,L2或其他）后再进行操作，但操作完之后不知道何时会写到内存，如果对声明了volatile变量进行写操作，JVM就会向处理器发送一条Lock前缀的指令，将这个变量所在缓存行的数据写回到系统内存。但是就算写回到内存，如果其他处理器缓存的值还是旧的，再执行计算操作就会有问题，所以在多处理器下，为了保证各个处理器的缓存是一致的，就会实现**缓存一致性协议**，每个处理器通过嗅探在总线上传播的数据来检查自己缓存的值是不是过期了，当处理器发现自己缓存行对应的内存地址被修改，就会将当前处理器的缓存行设置成无效状态，当处理器要对这个数据进行修改操作的时候，会强制重新从系统内存里把数据读到处理器缓存里。*


2. **赋值**

　　从上面，每个线程获取的值都是最新的，cpu拿到数据后，执行`+1`操作，最后写会内存。问题来了，每个线程如果拿到的值都为最新值`5`，四个线程同时执行`+1`后，将值`a`写回内存，最终结果却成为`6`，而正确的结果应该是`9`。所以它不能保证`原子性`，原子性是指不可中断的一个或一系列操作。

## 使用场景及注意事项

　　您只能在有限的一些情形下使用 volatile 变量替代锁。要使 volatile 变量提供理想的线程安全，必须同时满足下面两个条件[(原文)](https://www.ibm.com/developerworks/cn/java/j-jtp06197.html)：
　　`a`. 对变量的写操作**不依赖于当前值**。
　　`b`. 该变量没有包含在具有其他变量的不变式中。
**【解析】：**第一个条件的限制使 volatile 变量不能用作线程安全计数器。虽然增量操作（x++）看上去类似一个单独操作，实际上它是一个由 *`读取－修改－写入`* 操作序列组成的组合操作，必须以原子方式执行，而 volatile 不能提供必须的原子特性。实现正确的操作需要使 x 的值在操作期间保持不变，而 volatile 变量无法实现这点。
　　volatile 操作不会像锁一样造成阻塞，因此，在能够安全使用 volatile 的情况下，volatile 可以提供一些优于锁的可伸缩特性。如果读操作的次数要远远超过写操作（`读多写少`），与锁相比，volatile 变量通常能够减少同步的性能开销。
　　很多并发性专家事实上往往引导用户远离 volatile 变量，因为使用它们要比使用锁更加容易出错。然而，如果谨慎地遵循一些良好定义的模式，就能够在很多场合内安全地使用 volatile 变量。要始终牢记使用 volatile 的限制 —— 只有在状态真正独立于程序内其他内容时才能使用 volatile —— 这条规则能够避免将这些模式扩展到不安全的用例。
　　接下来介绍常见的适用场景。

### 1. 状态标志

```java
volatile boolean shutdownRequested;

...

public void shutdown() { shutdownRequested = true; }

public void doWork() { 
    while (!shutdownRequested) { 
        // do stuff
    }
}
```

　　由于 volatile 简化了编码，并且状态标志并不依赖于程序内任何其他状态，因此此处非常适合使用 volatile。

### 2. 一次性安全发布（one-time safe publication）

　　在缺乏同步的情况下，可能会遇到某个对象引用的更新值（由另一个线程写入）和该对象状态的旧值同时存在。（这就是造成著名的双重检查锁定（double-checked-locking）问题的根源，其中对象引用在没有同步的情况下进行读操作，产生的问题是您可能会看到一个更新的引用，但是仍然会通过该引用看到不完全构造的对象）。

```java
public class BackgroundFloobleLoader {
    public volatile Flooble theFlooble;

    public void initInBackground() {
        // do lots of stuff
        theFlooble = new Flooble();  // this is the only write to theFlooble
        /*
        //此操作包含了（伪代码）：
        
        //memory = allocate();   //1：分配对象的内存空间
		//ctorInstance(memory);  //2：初始化对象
		//instance = memory;     //3：设置instance指向刚分配的内存地址
		
		//上面三行伪代码中的2和3之间，可能会被重排序(在一些JIT编译器上，这种重排序是真实发生的)
		//2和3之间重排序之后的执行时序如下
		
		//memory = allocate();   //1：分配对象的内存空间
		//instance = memory;     //3：设置instance指向刚分配的内存地址
					//注意，此时对象还没有被初始化！
		//ctorInstance(memory);  //2：初始化对象
		*/
    }
}

public class SomeOtherClass {
    public void doWork() {
        while (true) { 
            // do some stuff...
            // use the Flooble, but only if it is ready
            if (floobleLoader.theFlooble != null) 
                doSomething(floobleLoader.theFlooble);
        }
    }
}
```

　　如果 theFlooble 引用不是 volatile 类型，doWork() 中的代码在解除对 theFlooble 的引用时，将会得到一个不完全构造的 Flooble。亦即出现重排序后，虽然拿到了地址值，但是对象还没有初始化结束，如果此时调用该对象，将会出现错误。关于 double-checked locking，强烈推荐[《双重检查锁定与延迟初始化》](http://www.infoq.com/cn/articles/double-checked-locking-with-delay-initialization)。
　　与锁相比，Volatile 变量是一种非常简单但同时又非常脆弱的同步机制，它在某些情况下将提供优于锁的性能和伸缩性。如果严格遵循 volatile 的使用条件 —— 即变量真正独立于其他变量和自己以前的值 —— 在某些情况下可以使用 volatile 代替 synchronized 来简化代码。
