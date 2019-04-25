---
title: 并发编程-深入理解AQS(队列同步器)
date: 2019-03-19 11:52:09
tags: AQS
categories: 并发编程
---

## 什么是AQS
- AbstractQueuedSynchronizer是一个队列同步器，是用来构建锁和其它同步组件的基础框架，它使用一个volatile修饰的int成员变量表示同步状态，通过内置的FIFO队列来完成资源获取线程排队的工作
- 通过改变int成员变量state来表示锁是否获取成功，当state>0表示锁获取成功，当state=0时说明锁释放成功。提供了三个方法（`getState()`、`setState(int newState)`、`compareAndSetState(int expect,int update)`）来对同步状态state进行操作，AQS确保对state操作时线程安全的。
- 主要使用方式是继承，子类通过继承同步器并实现它的抽像方法来管理同步状态。
- 提供独占式和共享式两种方式来操作同步状态的获取与释放
- ReentrantLock、ReentrantReadWriteLock、Semaphore等就并发工具就是基于护一个内部帮助器类集成AQS来实现的的

## AQS提供的方法(列出主要几个)
- `acquire(int arg)`   以独占模式获取对象，忽略中断。
- `acquireInterruptibly(int arg) ` 以独占模式获取对象，如果被中断则中止。
- `acquire(int arg)`   以独占模式获取对象，忽略中断。
- `acquireShared(int arg) ` 以共享模式获取对象，忽略中断。
- `acquireSharedInterruptibly(int arg)  ` 以共享模式获取对象，如果被中断则中止。
- `compareAndSetState(int expect, int update) ` 如果当前状态值等于预期值，则以原子方式将同步状态设置为给定的更新值。
- `getState() ` 返回同步状态的当前值。
- `release(int arg)`   以独占模式释放对象。
- `releaseShared(int arg)`    以共享模式释放对象。
- `setState(int newState)`  设置同步状态的值。
- `tryAcquire(int arg)` 试图在独占模式下获取对象状态。
- `tryAcquireNanos(int arg, long nanosTimeout)`   试图以独占模式获取对象，如果被中断则中止，如果到了给定超时时间，则会失败。
- `tryAcquireShared(int arg) `  试图在共享模式下获取对象状态。
- `tryAcquireSharedNanos(int arg, long nanosTimeout) ` 试图以共享模式获取对象，如果被中断则中止，如果到了给定超时时间，则会失败。
- `tryReleaseShared(int arg)`  试图设置状态来反映共享模式下的一个释放。

![同步器可重写的方法](http://wx1.sinaimg.cn/large/006b7Nxngy1g1g30xqlk9j30tk0dogs7.jpg)

## 队列同步器的实现分析
* 同步器依赖内部的同步队列（一个FIFO双向队列）来完成同步状态的管理，当前线程获取同步状态失败时，同步器会将当前线程以及等待状态等信息构造成为一个节点（Node）并将其加入同步队列，同时会阻塞当前线程，当同步状态释放时，会把首节点中的线程唤醒，使其再次尝试获取同步状态。

``` java
 static final class Node {
        /** 表示节点正在共享模式中等待 */
        static final Node SHARED = new Node();
        /** 表示节点正在独占模式下等待 */
        static final Node EXCLUSIVE = null;

        /** 表示取消状态，同步队列中等待的线程等待超时或中断，需要从同步队列中取消等待，节点进入该值不会发生变化 */
        static final int CANCELLED =  1;
        /** 后续节点的线程处于等待状态，而当前节点的线程如果释放了同步状态或者取消，将会通知后续节点运行*/
        static final int SIGNAL    = -1;
        /** 节点在等待中，节点线程等待在Conditions上。当其他线程对Condition调用了signal()后，该节点将会从等待队列中转移到同步队列中，加入到同步状态的获取中 */
        static final int CONDITION = -2;
        /**
         * 表示下一次共享式同步状态获取将会无条件传播下去
         */
        static final int PROPAGATE = -3;

        volatile int waitStatus;

       /**前驱节点**/
        volatile Node prev;
        /**后继节点**/
        volatile Node next;

  
        volatile Thread thread;


        Node nextWaiter;

        final boolean isShared() {
            return nextWaiter == SHARED;
        }

        final Node predecessor() throws NullPointerException {
            Node p = prev;
            if (p == null)
                throw new NullPointerException();
            else
                return p;
        }

        Node() {    // Used to establish initial head or SHARED marker
        }

        Node(Thread thread, Node mode) {     // Used by addWaiter
            this.nextWaiter = mode;
            this.thread = thread;
        }

        Node(Thread thread, int waitStatus) { // Used by Condition
            this.waitStatus = waitStatus;
            this.thread = thread;
        }
    }

```
节点是构成同步队列的基础，同步器拥有首节点（head）和尾节点（tail），没有成功获取同步状态的线程将会成为节点加入该队列的尾部，同步队列的

## 同步器的acquire方法（获取） 

``` java
  public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```
- 调用自定义同步器实现的tryAcquire(int arg)方法，该方法保证线程安全的获取同步状态，如果同步状态获取失败，则构造同步节点并通过addWaiter(Node node)方法将该节点加入到同步队列的尾部，最后调用acquireQueued(Node node,int arg)方法，使得该
  节点以"死循环"(自旋)的方式获取同步状态。

## 独占式的获取与释放总结
- 在获取同步状态时，同步器维护一个同步队列，获取状态失败的线程都会被加入到队列中并在队列中进行自旋；移出队列（或停止自旋）的条件是前驱节点为头节点且成功获取了同步状态。在释放同步状态时，同步器调用tryRelease(int arg)方法释放同步状态，然后唤醒头节点的后继节点


## 同步器的release方法（释放）

``` java
  public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```
- 该方法执行时，会唤醒头节点的后继节点线程，unparkSuccessor(Node node)方法使用LockSupport来唤醒处于等待状态的线程。




## 基于AQS实现一个简单的可重入的独占式锁的获取与释放

``` java
package com.example.juc;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.AbstractQueuedSynchronizer;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;

/**
 *  基于AQS实现一个简单的锁
 *
 * @author qinxuewu
 * @create 19/3/18下午11:44
 * @since 1.0.0
 */
public class MyAQSLock implements Lock {
    private  final  MySync sync=new MySync();

    /**
     * 构建一个内部帮助器 集成AQS
     */
    private  static  class MySync extends AbstractQueuedSynchronizer{
        //状态为0时获取锁，

        /***
         * 一个线程进来时，如果状态为0，就更改state变量，返回true表示拿到锁
         *
         * 当state大于0说明当前锁已经被持有，直接返回false,如果重复进来，就累加state,返回true
         * @param arg
         * @return
         */
        @Override
        protected boolean tryAcquire(int arg) {
            //获取同步状态状态的成员变量的值
            int state=getState();
            Thread cru=Thread.currentThread();
            if(state==0){
                //CAS方式更新state，保证原子性，期望值，更新的值
                if( compareAndSetState(0,arg)){
                    //设置成功
                    //设置当前线程
                    setExclusiveOwnerThread(Thread.currentThread());
                    return  true;
                }
            }else if(Thread.currentThread()==getExclusiveOwnerThread()){
                    //如果还是当前线程进来，累加state,返回true  可重入
                    setState(state+1);
                    return  true;
            }
            return false;
        }

        /**
         * 释放同步状态
         * @param arg
         * @return
         */
        @Override
        protected boolean tryRelease(int arg) {
            boolean flag=false;
            //判断释放操作是否是当前线程，
            if(Thread.currentThread()==getExclusiveOwnerThread()){

                    //获取同步状态成员变量，如果大于0 才释放
                    int state=getState();
                    if(getState()==0){
                        //当前线程置为null
                        setExclusiveOwnerThread(null);
                        flag=true;
                    }
                    setState(arg);

            }else{
                //不是当线程抛出异常
                throw  new RuntimeException();
            }
            return flag;
        }
        Condition newCondition(){
            return  new ConditionObject();
        }
    }

    @Override
    public void lock() {
            sync.acquire(1);
    }

    @Override
    public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
    }

    /**
     * 加锁
     * @return
     */
    @Override
    public boolean tryLock() {
        return sync.tryAcquire(1);
    }

    @Override
    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
        return sync.tryAcquireNanos(1,unit.toNanos(time));
    }

    /**
     * 释放锁
     */
    @Override
    public void unlock() {
        sync.tryRelease(1);
    }

    @Override
    public Condition newCondition() {
        return sync.newCondition();
    }
}

```
测试

``` java
public class MyAQSLockTest {
    MyAQSLock lock=new MyAQSLock();
    private    int i=0;
    public  int  next() {
        try {
            lock.lock();
            try {
                Thread.sleep(300);

            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return i++;
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
        return 0;
    }

    public void test1(){
        System.out.println("test1");
        test2();
    }
    public  void  test2(){
        System.out.println("test2");
    }

    public static void main(String[] args){
        MyAQSLockTest test=new MyAQSLockTest();
//         Thread thread = new Thread(new Runnable() {
//            @Override
//            public void run() {
//                while (true) {
//
//                    System.out.println(Thread.currentThread().getName() + "-" + test.next());
//
//                }
//
//            }
//        });
//        thread.start();
//
//        Thread thread2 = new Thread(new Runnable() {
//            @Override
//            public void run() {
//                while (true) {
//
//                    System.out.println(Thread.currentThread().getName() + "-" + test.next());
//
//                }
//
//            }
//        });
//        thread2.start();

        //可重复锁演示
        Thread thread3 = new Thread(new Runnable() {
            @Override
            public void run() {
                test.test1();

            }
        });
        thread3.start();
    }
}

```

![觉得本文不错的话，分享一下给小伙伴吧~](http://wx1.sinaimg.cn/large/006b7Nxngy1g1eu6ewhl9j30760763yz.jpg)