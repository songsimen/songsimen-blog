---
title: 并发编程-深入理解阻塞队列ArrayBlockingQueue源码
date: 2019-03-30 10:21:09
tags: ArrayBlockingQueue
categories: 并发编程
---

## 概述
* `ArrayBlockingQueue`是一个由数组构成的有界阻塞队列，此队列按 `FIFO`（先进先出）原则对元素进行排序,支持公平和非公平模式，默认情况下不保证线程公平的访问队列。新元素插入到队列的尾部，队列获取操作则是从队列头部开始获得元素
  ![继承AbstractQueue类,实现BlockingQueue接口](http://wx1.sinaimg.cn/large/006b7Nxngy1g1w8723xluj30fg01mmy3.jpg)
## 常用方法
* `ArrayBlockingQueue(int capacity)`  创建一个固定容量和默认非公司访问策略队列
* `ArrayBlockingQueue(int capacity, boolean fair)`  创建一个具有固定容量和指定访问策略的 队列
* `ArrayBlockingQueue(int capacity, boolean fair, Collection<? extends E> c)` 创建一个具有固定容量和指定访问策略，并且制定元素类型的 队列
* `boolean add(E e)` 插入指定元素到队列尾部，成功返回true,队列已满会直接抛出异常
* `boolean	offer(E e)` 插入指定元素到队列尾部，成功时返回 true，如果此队列已满，则返回 false
* `E	peek()`  获取但不移除此队列的头；如果此队列为空，则返回 null
* `E	poll()`  获取并移除此队列的头，如果此队列为空，则返回 null
* `void	put(E e)`    将指定的元素插入此队列的尾部，如果该队列已满，则等待可用的空间
* `E	take()` 获取并移除此队列的头部，在元素变得可用之前一直等待

## 源码阅读
## 构造函数

```java
     /** 存放队列的 */
    final Object[] items;
    int takeIndex;
    int putIndex;
    int count;
    /** 可重入锁*/
    final ReentrantLock lock;
    private final Condition notEmpty;
    private final Condition notFull;
```

![构造函数](http://wx1.sinaimg.cn/large/006b7Nxngy1g1w8814qlcj30h90astgb.jpg)
* `ArrayBlockingQueue`的公平模式是使用`ReentrantLock`可重入锁实现的。并使用Condition使用队列的阻塞和唤醒
## put入队
* put操作比较简单。阻塞的添加。首先判断插入元素是否为空，如果是空则抛出控指针
* 然后获取可重入的排他锁，根据初始化时选择是公平还是非公平模式的锁，加的锁是一个支持可中断的锁。当队列的count等于数组的长度，此时队列已满。则使用`await`的方法是当前线程进入阻塞模式。未满就执行增加元素，并释放锁。
  ![put](http://wx1.sinaimg.cn/large/006b7Nxngy1g1w88ty60dj30c005wtbi.jpg)
* 队列未满，则添加元素到队列尾巴，当`++putIndex == items.length`条件成立说明此时队列已满，`putIndex`赋值为0 从头开始。然后累加队列的总个数，并唤醒一个阻塞的线程

![队列未满添加元素](http://wx1.sinaimg.cn/large/006b7Nxngy1g1w89x8h87j30au04q0us.jpg)

## take出队

* 出队的操作也很简单，首先拿到锁，然后判断队列是否为空，为空则进入阻塞等待。不为空则执行`dequeue`出队，并释放锁。
  ![出队](http://wx1.sinaimg.cn/large/006b7Nxngy1g1w8ahc4xij30ck05odil.jpg)
```java
    private E dequeue() {
    	//首先获取队列数组
        final Object[] items = this.items;
        @SuppressWarnings("unchecked")
        //拿到出队的元素，临时存储
        E x = (E) items[takeIndex];
        //出队的元素置为null
        items[takeIndex] = null;
        if (++takeIndex == items.length)
        //当出队元素到队列的最后一个元素，takeIndex还原为0
            takeIndex = 0;
        count--;
        //迭代器操作
        if (itrs != null)
            itrs.elementDequeued();
         //唤醒一个阻塞线程，返回出队的元素
        notFull.signal();
        return x;
    }
```

![觉得本文不错的话，分享一下给小伙伴吧~](http://wx1.sinaimg.cn/large/006b7Nxngy1g1eu6ewhl9j30760763yz.jpg)