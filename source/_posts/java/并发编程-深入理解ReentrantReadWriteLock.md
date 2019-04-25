---
title: 并发编程-深入理解ReentrantReadWriteLock读写锁
date: 2019-03-21 12:52:09
tags: ReentrantLock
categories: 并发编程
---

## 概述

- `ReentrantReadWriteLock`内部维护了一对锁，读锁和写锁。支持重入和公以及平非公平模式。读锁是共享式的，多个线程可以并发的读取。写锁是独占式的，在写线程访问时，所有的读线程和其他写线程均被阻塞。通过分离读锁和写锁，使得并发性相比一般的排他锁有了很大提升
- 锁降级：遵循获取写锁，获取读锁在释放写锁的次序，写锁可以降级为读锁
- 读取锁和写入锁都支持锁获取期间的中断
- `ReentrantLock` 中的同步状态state表示一个锁被获取的次数，而读写锁也是基于`AQS`队列同步器实现的，内部也有一个帮助类Sync继承`AQS`，读写锁的自定义同步器需要在同步状态上state维护多个读线程和一个写线程的状态，将变量切分成了两个部分，高16位表示读，低16位表示写 。 

## 程序中使用

```java
public class ReentrantReadWriteLockTest {
    static Map<String, Object> map = new HashMap<String, Object>();
    static ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();
    static Lock r = rwl.readLock();
    static Lock w = rwl.writeLock();

    // 获取一个key对应的value
    public static final Object get(String key) {
        r.lock();
        try {
            return map.get(key);
        } finally {
            r.unlock();
        }
    }
    // 设置key对应的value，并返回旧的value
    public static final Object put(String key, Object value) {
        w.lock();
        try {
            return map.put(key, value);
        } finally {
            w.unlock();
        }
    }
}

```

* 在读操作`get(String key)`方法中，需要获取读锁，这使得并发访问该方法时不会被阻塞。写操作`put(String key,Object value)`方法，在更新`HashMap`时必须提前获取写锁，当获取写锁后，其他线程对于读锁和写锁的获取均被阻塞，而只有写锁被释放之后，其他读写操作才能继续 .



## 读锁源码分析

当`new  ReentrantReadWriteLock`时 执行的如下代码：

```java
//默认非同步的
public ReentrantReadWriteLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
        readerLock = new ReadLock(this);
        writerLock = new WriteLock(this);
    }
```

> 默认非同步的获取一个读锁的对象

**执行 `r.lock()`获或读锁**

```java
  public void lock() {
      		//调用AQS中的acquireShared方法
            sync.acquireShared(1);
   }
	//AQS类的
   public final void acquireShared(int arg) {
       //tryAcquireShared需要自定义同步组件具体提供实现，
       //所以这里调用的就是读写锁内的tryAcquireShared方法
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
    }
```

* `lock`方法会调用AQS中的acquireShared方法。acquireShared方法需要调用tryAcquireShare。
* tryAcquireShared需要自定义同步组件具体提供实现，所以这里调用的就是读写锁内的tryAcquireShared方法

> tryAcquireShared的源码

```java
 protected final int tryAcquireShared(int unused) {
     		//临时变量 记录当前访问的线程
            Thread current = Thread.currentThread();
     		//获同步状态 AQS中的getState
            int c = getState();
     		/**
     			exclusiveCount:计算写锁
     			getExclusiveOwnerThread：获取当前锁的持有线程
     		*/
     		 //如果存在写锁，且锁的持有者不是当前线程，直接返回-1
            if (exclusiveCount(c) != 0 &&getExclusiveOwnerThread() != current)
                return -1;
     		//返回读锁被获取的总数
            int r = sharedCount(c);
        /*
         * readerShouldBlock():读锁是否需要等待（公平锁原则）
         * r < MAX_COUNT：持有线程小于最大数（65535）
         * compareAndSetState(c, c + SHARED_UNIT)：设置读取锁状态
         */
            if (!readerShouldBlock() && r < MAX_COUNT 
                &&compareAndSetState(c, c + SHARED_UNIT)) {
                // r=0说明当前读锁处于空闲，还没有线程持有
                if (r == 0) {
                    //firstReader是第一个获得读锁定的线程。
                    firstReader = current;
                    firstReaderHoldCount = 1;
                } else if (firstReader == current) {
                    //重入锁，累加持有总数
                    firstReaderHoldCount++;
                } else {
                    //这里处理读锁的共享式获取，记录每个线程获取锁的线程ID以及次数
                    HoldCounter rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current))
                        cachedHoldCounter = rh = readHolds.get();
                    else if (rh.count == 0)
                        readHolds.set(rh);
                    rh.count++;
                }
                return 1;
            }
            return fullTryAcquireShared(current);
        }
```

* 在读锁获取锁和释放锁的过程中，我们一直都可以看到一个变量rh （HoldCounter ），该变量在读锁中扮演着非常重要的作用。
* 我们了解读锁的内在机制其实就是一个共享锁，为了更好理解HoldCounter ，我们暂且认为它不是一个锁的概率，而相当于一个计数器。一次共享锁的操作就相当于在该计数器的操作。获取共享锁，则该计数器 + 1，释放共享锁，该计数器 – 1。只有当线程获取共享锁后才能对共享锁进行释放、重入操作。所以HoldCounter的作用就是当前线程持有共享锁的数量，这个数量必须要与线程绑定在一起，否则操作其他线程锁就会抛出异常。我们先看HoldCounter的定义：

```java
static final class HoldCounter {
            int count = 0;
            final long tid = getThreadId(Thread.currentThread());
  }
```

* HoldCounter 定义非常简单，就是一个计数器count 和线程 id tid 两个变量。
* 判断读锁是否需要阻塞，读锁持有线程数小于最大值（65535），且设置锁状态成功，并返回1。如果不满足改条件，执行fullTryAcquireShared()。
* fullTryAcquireShared(Thread current)会根据“是否需要阻塞等待”，“读取锁的共享计数是否超过限制”等等进行处理。如果不需要阻塞等待，并且锁的共享计数没有超过限制，则通过CAS尝试获取锁，并返回1

## 读锁的获取与释放

* 读锁是一个支持重进入的共享锁，它能够被多个线程同时获取，在没有其他写线程访问（或者写状态为0）时，读锁总会被成功地获取，而所做的也只是（线程安全的）增加读状态。如果当前线程已经获取了读锁，则增加读状态。如果当前线程在获取读锁时，写锁已被其他线程获取，则进入等待状态。获取读锁的实现从Java 5到Java 6变得复杂许多，主要原因是新增了一些功能，例如getReadHoldCount()方法，作用是返回当前线程获取读锁的次数。读状态是所有线程获取读锁次数的总和，而每个线程各自获取读锁的次数只能选择保存在ThreadLocal中，由线程自身维护，这使获取读锁的实现变得复杂 



![觉得本文不错的话，分享一下给小伙伴吧~](http://wx1.sinaimg.cn/large/006b7Nxngy1g1eu6ewhl9j30760763yz.jpg)