---
title: 揭开神秘面纱——深入浅出ThreadLocal
date: 2017-12-08 16:02:32
description:
categories: [java,threadlocal,多线程]
tags: [java,threadlocal,多线程]
---

能够找到这篇文章，说明你已开始学习Java的多线程了，也了解多线程的同步、锁等概念。但，ThreadLocal虽出现在多线程的环境中，对于它的使用，并不涉及到锁和同步的概念。它生于多线程，伴随着多线程的热点，而并不沾染多线程的常见问题，是不是莫名的小清新呢？如果你对它有所了解，听说过内存泄露，如何才能更好的驾驭它呢？带着好奇和疑惑，一起深入ThreadLocal吧！

<!--more-->

# 1.背景

随便举两个具体的例子：

1. 在一个web项目中，从请求一进来就为之生成一个 `uuid`，无论系统是否报异常，返回给客户的必须是同一 `uuid`，可能首先会想到当作方法的参数来传递，这样任何地方可以成功的获取到这个 `uuid`，但这个 `uuid` 会在系统中几乎各个方法参数中都会出现，但 `uuid` 又非主要业务参数，这样势必会与业务耦合性太强。

2. 对于很多非线程安全的类而言，如工具类：SimpleDateFormat和JDBC的Connection，它们经常出现在并发环境中，例如Connection，大家刚接触JDBC的时候，都是在方法中完成Connection的 `init/commit/close`，如多个线程都想连接数据库执行sql，方法有如下：
    1. 对Connection进行同步加锁，协调各个线程操作DB的顺序，没错但很低效。
    2. 每个线程自己创建Connection，会造成频繁的创建和释放连接，线程结束，Connection也就结束。

# 2. 与并发/同步的区别

什么，你突然想到了并发中的同步？先立一个flag，其实他们有本质的区别，同步是协调多个线程对同一个变量的修改，而ThreadLocal则是将这个变量的副本据为线程己有，各个线程操作的是各自的threadlocal变量，各个线程互不影响，自然不会涉及到同步。

# 3. 易混名词释疑

大家容易搞混`Thread/ThreadLocal/ThreadLocalMap`三者的关系，其实很简单，如图:

![关系图](https://github.com/hulog/hulog.github.io/blob/blog/source/images/post/threadlocal-1.png?raw=true)

为了便于大家记住依赖关系，煞费苦心的我编了个故事：从前，有一个雷劈出来一个线程小天(Thread)，有一天，遇到了小乐(threadLocal)，线程小天说：“如果想发挥价值，你必须初始化一个ThreadLocalMap放我这托管！” 小乐调用`initialValue()`不一会儿就初始化了ThreaLocalMap——小明(ThreadLocal的静态内部类)，然后乐转身给天说，这是小明，小天看着欢喜地说：“明明，我给你一个小名threadLocals吧（将其赋值给当前线程的threadLocals变量），以后呢，你就跟我小天混，只要我还在，你就会有肉吃。你的工作内容也很简单，如果以后小乐调用`get()`方法获取值的时候，你就将他的 `threadLocalHashCode` 作为key，在你的Map中找到对应的value。当然 `set()` 方法也差不多，顶多处理一下hash冲突的问题，不过这是你的内务，我就不干预了。”  当然，小天后面也有遇到其他的ThreadLocal，不过它已经有Map小明了，直接让小明干活就可以了。

# 4. 源码时间

作为专业的看官，等的就是代码，静下心来，15min后，让你感受到咸鱼翻身，虽然还是咸鱼，哈哈

来看ThreadLocal这个类，其中包括 `get/set/remove` 等方法，为了避免码字嫌疑，只贴关键代码(其中加入了笔者[Norman](https://hulog.github.io)的中文注释，帮助理解)，下面逐个介绍：

## 4.1 set()

代码包含set()方法，同时包括方法体内所调用的其他方法(后同)

```java
    // 代码段1
    public class ThreadLocal<T> {
        //...
        public void set(T value) {
            // 获取当前线程t
            Thread t = Thread.currentThread();
            // 获取当前线程对应的 ThreadLocalMap，它是ThreadLocal的一个内部类
            ThreadLocal.ThreadLocalMap map = getMap(t);
            // 如果map之前被创建，则直接进map中取值
            if (map != null)
                map.set(this, value);
            // 创建ThreadLocalMap
            else
                createMap(t, value);
        }

        // 获取线程t的ThreadLocalMap，无则return null
        ThreadLocal.ThreadLocalMap getMap(Thread t) {
            return t.threadLocals;
        }

        // 创建线程t的ThreadLocalMap，并设定初始值
        void createMap(Thread t, T firstValue) {
            // 创建新的ThreadLocalMap，并将其与当前线程进行关联，构造方法往下翻
            t.threadLocals = new ThreadLocal.ThreadLocalMap(this, firstValue);
        }

        static class ThreadLocalMap {
            //...
            /**
             * Construct a new map initially containing (firstKey, firstValue).
             * ThreadLocalMaps are constructed lazily, so we only create
             * one when wse have at least one entry to put in it.
             */
            ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
                // 初始化table，INITIAL_CAPACITY为16
                table = new Entry[INITIAL_CAPACITY];
                // 将threadLocal的threadLocalHashCode除以16取模(下面这种骚操作是因为除数是2^n)，得到桶的下标
                int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
                // 将生成的Entry 放置于table对应的桶中
                table[i] = new Entry(firstKey, firstValue);
                size = 1;
                // 设置扩容阈值为table length的 2/3(负载因子2/3)
                setThreshold(INITIAL_CAPACITY);
            }

            private void setThreshold(int len) {
                threshold = len * 2 / 3;
            }

            private void set(ThreadLocal<?> key, Object value) {

                // We don't use a fast path as with get() because it is at
                // least as common to use set() to create new entries as
                // it is to replace existing ones, in which case, a fast
                // path would fail more often than not.

                Entry[] tab = table;
                int len = tab.length;
                int i = key.threadLocalHashCode & (len-1);

                for (Entry e = tab[i];
                     e != null;
                     e = tab[i = nextIndex(i, len)]) {
                    ThreadLocal<?> k = e.get();

                    if (k == key) {
                        e.value = value;
                        return;
                    }

                    if (k == null) {
                        replaceStaleEntry(key, value, i);
                        return;
                    }
                }

                tab[i] = new Entry(key, value);
                int sz = ++size;
                if (!cleanSomeSlots(i, sz) && sz >= threshold)
                    rehash();
            }
            //...
        }

        private final int threadLocalHashCode = nextHashCode();

        private static AtomicInteger nextHashCode = new AtomicInteger();

        /**
         * The difference between successively generated hash codes - turns
         * implicit sequential thread-local IDs into near-optimally spread
         * multiplicative hash values for power-of-two-sized tables.
         * 这个数字就不得不提了，以他为步长生成的2^n个数字序列，
         * 除以2^n取模后，得到的模居然可以逐个均匀的落在2^n个桶中，
         * 与传统步长(+1)的不同在于，逐个均匀分布(而非连续分布)可以减小碰撞的几率，
         * (可以拿着这个数应用在其他类似场景中)
         */
        private static final int HASH_INCREMENT = 0x61c88647;

        private static int nextHashCode() {
            // 从0开始递增
            return nextHashCode.getAndAdd(HASH_INCREMENT);
        }

        //...
    }
```

```java
// 代码段2
public class Thread implements Runnable {
//...
    /* ThreadLocal values pertaining to this thread. This map is maintained
     * by the ThreadLocal class. */
    ThreadLocal.ThreadLocalMap threadLocals = null;
//...
}
```

整体流程可以看出，当调用`set(T value)`方法时，会先取出本线程的ThreadLocalMap，对于Map：

  + 如果不为空，则以ThreadLocal实例为key， 将value存储在此Map中
  + 如果为空，就创建一个Map，并将其赋值给此线程的成员变量threadLocals

对于ThreadLocalMap是由谁来维护，其定义的代码如下:

```java
// 代码段3
public class ThreadLocal<T> {
//...
    static class ThreadLocalMap {
        /**
         * The entries in this hash map extend WeakReference, using
         * its main ref field as the key (which is always a
         * ThreadLocal object).  Note that null keys (i.e. entry.get()
         * == null) mean that the key is no longer referenced, so the
         * entry can be expunged from table.  Such entries are referred to
         * as "stale entries" in the code that follows.
         */
        static class Entry extends WeakReference<ThreadLocal> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal k, Object v) {
                super(k);
                value = v;
            }
        }
        //...
    }
//...
}
```

结合代码段2，可以看出，ThreadLocalMap其实是定义在ThreadLocal中的静态内部类，然后由Thread类来维护，依附于Thread的生命周期。读过HashMap源码的童鞋知道Entry是什么东东，这个Entry 继承了 WeakReference类，其实就Entry的key继承了它，从构造函数就可以看出，顺带简单回顾下java 的引用：

 + 强引用：不受GC影响，即时OOM也不回收；eg. Person p = new Person("Norman")
 + 软引用：只会在内存不足时，由GC回收；
 + 弱引用：不论内存是否够用，一旦GC，则回收，不过GC的线程优先级，不一定很快的发现；
 + 虚引用：形同虚设，与前三不同，它必须配合WeakReferenceQueue，跟踪对象被垃圾回收的活动

那么，为什么要用到弱引用呢？官方文档如是说：

>To help deal with very large and long-lived usages, the hash table entries use WeakReferences for keys. However, since reference queues are not used, stale entries are guaranteed to be removed only when the table starts running out of space.
>
> [Norman](https://hulog.github.io)译：为了应对非常大的和长寿命的对象使用，哈希表 `Entry` 使用 `WeakReferences` 作为键。但是，由于没有使用引用队列(Reference类中的队列)， 因此只有当表快耗尽空间时， 才保证将陈旧 `Entry` 删除。

如下场景很好的解释了这样设计的好处(感谢[xiaohansong](http://blog.xiaohansong.com/2016/08/06/ThreadLocal-memory-leak))：

**强引用:** 当对象A中引用ThreadLocal的对象B，A被回收，则B变为垃圾，但线程对Map是强引用，Map对B是 **强** 引用，只要线程存活，则B始终不会被回收。

**弱引用:** 当对象A中引用ThreadLocal的对象B，A被回收，则B变为垃圾，由于线程对Map是强引用，Map对B是 **弱** 引用，即使没有手动删除，在下一个GC周期，B也会被回收掉。而Map中的value会在调用set/get/remove方法后断掉强引用，等待GC后续回收(见 **4.4 内存泄露**)。

## 4.2 get()

```java
public class ThreadLocal<T> {
//...
    /**
     * Returns the value in the current thread's copy of this
     * thread-local variable.  If the variable has no value for the
     * current thread, it is first initialized to the value returned
     * by an invocation of the {@link #initialValue} method.
     *
     * @return the current thread's value of this thread-local
     */
    public T get() {
        // 1.获取当前线程t
        Thread t = Thread.currentThread();
        // 2.获取当前线程对应的 ThreadLocalMap， 它是ThreadLocal的一个内部类
        ThreadLocal.ThreadLocalMap map = getMap(t);
        // 3.如果map之前被创建，则直接进map中取值
        if (map != null) {
            // 3.1 以当前ThreadLocal实例作为key，在map中获取对应的Entry
            ThreadLocal.ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        // 4.如果map之前未被创建，或创建后未得到该ThreadLocal实例的Entry
        //   则调用此方法进行初始化，而后返回结果
        return setInitialValue();
    }

    static class ThreadLocalMap {
        //...
        private Entry getEntry(ThreadLocal<?> key) {
            int i = key.threadLocalHashCode & (table.length - 1);
            Entry e = table[i];
            if (e != null && e.get() == key)
                // 如果在该桶中取到，直接返回
                return e;
            else
                // 如果不在，则按规则寻找
                return getEntryAfterMiss(key, i, e);
        }

        private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
            Entry[] tab = table;
            int len = tab.length;

            while (e != null) {
                ThreadLocal<?> k = e.get();
                if (k == key)
                    return e;
                if (k == null)
                    expungeStaleEntry(i);
                else
                    i = nextIndex(i, len);
                e = tab[i];
            }
            return null;
        }
//...
}
```

如果，当前线程没有threadLocal值，则默认调用initialValue()方法，其中的取值可以看到，ThreadLocalMap中处理Hash冲突的方法是线性探测法，顺带回顾下数据结构中，**Hash冲突的解决办法：**

  1. 开放地址法
     + 线性探测 (ThreadLocalMap)
     + 二次探测
     + 再哈希
  2. 再哈希法
  3. 链地址法 (HashMap)
  4. 建立一个公共溢出区

## 4.3 remove()

```java
    public class ThreadLocal<T> {
        // ...
        public void remove() {
            ThreadLocalMap m = getMap(Thread.currentThread());
            if (m != null)
                // 调用ThreadLocalMap的remove方法
                m.remove(this);
        }
        // ...

        static class ThreadLocalMap {
            // ...

            private void remove(ThreadLocal<?> key) {
                Entry[] tab = table;
                int len = tab.length;
                // 按hashcode计算出应该在的位置
                int i = key.threadLocalHashCode & (len - 1);
                // 考虑到hash冲突，按线性探测查找应该在的位置
                for (Entry e = tab[i];
                     e != null;
                     e = tab[i = nextIndex(i, len)]) {
                    // 判断是否是目标Entry
                    if (e.get() == key) {
                        // 调用Reference中的clear()方法，Clears this reference object.
                        e.clear();
                        // 将该位置的Entry清除掉后，对table重新整理
                        expungeStaleEntry(i);
                        return;
                    }
                }
            }

            private int expungeStaleEntry(int staleSlot) {
                Entry[] tab = table;
                int len = tab.length;

                // expunge entry at staleSlot
                // 先将value引用置空
                tab[staleSlot].value = null;
                tab[staleSlot] = null;
                size--;

                // Rehash until we encounter null
                // 因为后续连续的元素可能是之前hash冲突引起的，
                // 所以，对table后续连续的元素，进行重新hash
                Entry e;
                int i;
                for (i = nextIndex(staleSlot, len);
                     (e = tab[i]) != null;
                     i = nextIndex(i, len)) {
                    ThreadLocal<?> k = e.get();
                    // 如果key为空，顺便清除它
                    if (k == null) {
                        e.value = null;
                        tab[i] = null;
                        size--;
                    } else {
                        int h = k.threadLocalHashCode & (len - 1);
                        // 如果一个元素按照hashcode运算后，
                        // 实际位置不在应该在的位置，则对其重新hash
                        if (h != i) {
                            tab[i] = null;

                            // Unlike Knuth 6.4 Algorithm R, we must scan until
                            // null because multiple entries could have been stale.
                            while (tab[h] != null)
                                h = nextIndex(h, len);
                            tab[h] = e;
                        }
                    }
                }
                return i;
            }
        }
    }

```

## 4.4 内存泄露

从源码中看，无论是get()，set()还是remove()操作，都会包含对ThreadLocalMap 中key为null的Entry清除，那么泄露会出现在什么地方呢？仔细来看各部分依赖图：

![内部关联](https://github.com/hulog/hulog.github.io/blob/blog/source/images/post/threadlocal-2.png?raw=true)

ThreadLocal可手动置为null，也可以由GC置null(因为弱引用)，但这只是针对key，对于value，当前Entry的value被Entry引用，而Entry被当前Map引用，而Map则被当前线程实例Thread引用，如果当前线程不退出，则value是不会被GC，造成内存泄露。

更加准确的说，是发生在 ：**当Map中的key(ThreadLocal)为null后到线程结束** 这期间。当遇到线程池，线程会被重复利用，如果使用 `set` 后不再使用 `get/set/remove`，这个强应用会一直存在，造成内存泄露。(PS:当value是大对象时尤为严重)

那补救措施有哪些呢？
  1. 首先，jdk本身get/set/remove操作会清除key为null的Entry，但属于被动清除，不调用此方法，依然会内存泄露
  2. 其次，当用完threadLocal后，应该主动调用remove方法，主动断掉value到thread的引用链

# 5. 总结

使用ThreadLocal有一些建议：

1. 使用static修饰，使之属于类而不是实例，因为它持有的对象，生效范围一般在用户会话/web请求周期期间。
    > One web request => one Persistence session.
    >Not one web request => one persistence session per object.

2. 如上文提到，使用结束后调用remove()方法进行清除，避免造成内存泄露。
