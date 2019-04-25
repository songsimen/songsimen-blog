---
title: 并发编程-深入理解ReentrantLock
date: 2019-03-20 12:52:09
tags: ReentrantLock
categories: 并发编程

---

### 什么是ReentrantLock
* ReentrantLock是一个可重入的互斥锁锁， 实现Lock接口。具有与使用 synchronized 方法和语句所访问的隐式监视器锁相同的一些基本行为和语义。ReentrantLock是显示的获取或释放锁，并且有锁超时，锁中断等功能。

* 内部维户了一个Sync的内部类，继承AQS队列同步器。

* ReentrantLock 将由最近成功获得锁，并且还没有释放该锁的线程所拥有。当锁没有被另一个线程所拥有时，调用 lock 的线程将成功获取该锁并返回。如果当前线程已经拥有该锁，此方法将立即返回。可以使用 isHeldByCurrentThread() 和 getHoldCount() 方法来检查此情况是否发生。

* 默认是非公平锁的实现方式

  

### 非公平锁获取和释放流程
 **加锁** 
*  执行`lock`方法加锁时调用内部`NonfairSync`的`lock`方法，第一次会快速尝试获取锁，执行`AQS`类的`compareAndSetState`方法（CAS）更改同步状态成员变量`state`，如果获取成功 则设置当前线程为锁的持有者。失败则执行`AQS`类的`acquire`方法，`acquire`会调用的`AQS`中的`tryAcquire`方法。这个`tryAcquire`方法需要自定义同步组件提供实现。
*  `tryAcquire`的具体流程是执行`Sync`类的`nonfairTryAcquire`方法：首先记录当前加锁线程，然后调用`getState`获取同步状态，如果为0时 说明锁处于空闲状态，可以获取，会以`CAS`方式修改`state`变量。成功则设置当前线程 返回`true`。否则执行重入判断，判断当前访问线程和已经持有锁的线程是否是同一个。如果相同，将同步状态值进行增加，并返回true。否则返回加锁失败`false`

 **解锁** 

 * 解锁`unlock`方法会调用内部类`Sync`的`tryRelease`方法。`tryRelease`首先调用`getState`方法获取同步状态，并进行了减法操作。在判断释放操作是不是当前线程，不是则抛出异常，然后判断同步状态是否等于0，如果是0，说明没有线程持有，锁是空闲的，则将当前锁的持有者设置为`null`， 方便其它线程获取，并返回`true`。否则返回false

### ReentrantLock常用方法介绍

* `getHoldCount() ` 查询当前线程保持此锁的次数。
* `getOwner()`  返回目前拥有此锁的线程
* `getQueueLength()`  返回正等待获取此锁的线程估计数
* `getWaitingThreads(Condition condition)`  返回一个 collection，它包含可能正在等待与此锁相关给定条件的那些线程。
* `boolean hasQueuedThread(Thread thread)`  查询给定线程是否正在等待获取此锁。
* `boolean hasQueuedThreads()`  查询是否有些线程正在等待获取此锁。
* `boolean hasWaiters(Condition condition)`   查询是否有些线程正在等待与此锁有关的给定条件
* `boolean isHeldByCurrentThread()` 查询当前线程是否保持此锁。
* `void lock()`   获取锁。
* `void lockInterruptibly() ` 如果当前线程未被中断，则获取锁。
* `Condition newCondition()` 返回用来与此 Lock 实例一起使用的 Condition 实例。
* `boolean  tryLock()`    仅在调用时锁未被另一个线程保持的情况下，才获取该锁。
* `void unlock() ` 释放锁

> 程序中使用 

``` java
  private  ReentrantLock lock=new ReentrantLock();
    private  int i=0;
    public  void  a(){
        lock.lock();
        i++;
        lock.unlock();

    }
```
## ReentrantLock源码分析

``` java
package java.util.concurrent.locks;
import java.util.concurrent.TimeUnit;
import java.util.Collection;
public class ReentrantLock implements Lock, java.io.Serializable {
    private static final long serialVersionUID = 7373984872572414699L;
    private final Sync sync;

    /**内部维护的一个帮助类，继承成AQS  锁的获取和释放主要靠它**/
    abstract static class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = -5179523762034025860L;

    
        abstract void lock();

        /**
         * 执行非公平的t加锁
         */
        final boolean nonfairTryAcquire(int acquires) {
            //记录当前加锁线程
            final Thread current = Thread.currentThread();
            //获取同步状态 AQS中的volatile修饰的int类型成员变量 state  
            int c = getState();
            //为0时 说明锁处于空闲状态，可以获取
            if (c == 0) {
                // CAS方式修改state。成功则设置当前线程 返回true
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            //线程重入判断，判断当前访问线程和已经持有锁的线程是否是同一个
            else if (current == getExclusiveOwnerThread()) {
            //将同步状态值进行增加

                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
				//设置同步状态，重入锁的话就累加，并返回true
                setState(nextc);
                return true;
            }
            return false;
        }
		//释放锁，就是把AQS中的同步状态变量就行类减直到0 就是出于空闲状态了
        protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            //如果释放操作不是当前线程 则抛出异常
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            //同步状态等于0，说明没有线程持有，锁是空闲的
            if (c == 0) {
                free = true;
                //当前锁的持有者 设置为null 方便其它线程获取
                setExclusiveOwnerThread(null);
            }
          //如果该锁被获取了n次，那么前(n-1)次tryRelease(int releases)方法必须返回false
            setState(c);
            return free;
        }
	
        protected final boolean isHeldExclusively() {
            return getExclusiveOwnerThread() == Thread.currentThread();
        }

        final ConditionObject newCondition() {
            return new ConditionObject();
        }
        //返回目前拥有此锁的线程，如果此锁不被任何线程拥有，则返回 null。
        final Thread getOwner() {
            return getState() == 0 ? null : getExclusiveOwnerThread();
        }
		//查询当前线程保持此锁的次数。
        final int getHoldCount() {
            return isHeldExclusively() ? getState() : 0;
            
        //查询锁是否被持有
        final boolean isLocked() {
            return getState() != 0;
        }


        private void readObject(java.io.ObjectInputStream s)
            throws java.io.IOException, ClassNotFoundException {
            s.defaultReadObject();
            setState(0); // reset to unlocked state
        }
    }

	//非公平的  Sync的子类
    static final class NonfairSync extends Sync {
        private static final long serialVersionUID = 7316153563782823691L;

        final void lock() {
            //第一次快速获取锁，使用CAS 方式 成功设置当前线程为锁的持有者
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                //锁获取失败时，调用AQS的acquire去获取锁，
               //acquire会调用tryAcquire方法，tryAcquire需要自定义同步组件提供实现,
                //所以这里的调用逻辑是acquire-》tryAcquire（NonfairSync类的）-》Sync的nonfairTryAcquire方法
                acquire(1);
        }
		//调用父类nonfairTryAcquire 实现加锁
        protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
    }
	//公平的 Sync的子类
    static final class FairSync extends Sync {
        private static final long serialVersionUID = -3000897897090466540L;
		// 加锁 调用AQS中的acquire方法，acquire会调用下面的tryAcquire方法
        final void lock() {
            acquire(1);
        }
		//加锁的过程，和父类的调用父类nonfairTryAcquire方法大致一样
        //唯一不同的位置为判断条件多了hasQueuedPredecessors()方法
        protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                //公平锁实现的关键点hasQueuedPredecessors
                /**
                 即加入了同步队列中当前节点是否有前驱节点的判断
                 如果该方法返回true，则表示有线程比当前线程更早地请求获取锁
                 因此需要等待前驱线程获取并释放锁之后才能继续获取锁
                */
                if (!hasQueuedPredecessors() && compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
    }
    //AQS中的方法 判断当前线程是否位于CLH同步队列中的第一个。如果是则返回true，否则返回false。
    public final boolean hasQueuedPredecessors() {
        Node t = tail; // Read fields in reverse initialization order
        Node h = head;
        Node s;
        return h != t &&((s = h.next) == null || s.thread != Thread.currentThread());
    }
        
	//默认的构造函数  非公平锁
    public ReentrantLock() { sync = new NonfairSync();}
    //为true 公平锁
    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }
    public void lock() { sync.lock();}
    public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
    }
    public boolean tryLock() { return sync.nonfairTryAcquire(1);}
    public boolean tryLock(long timeout, TimeUnit unit) throws InterruptedException {
        return sync.tryAcquireNanos(1, unit.toNanos(timeout));
    }
	//释放锁
    public void unlock() {  sync.release(1);}
    public Condition newCondition() {return sync.newCondition();}
    

}

```

![觉得本文不错的话，分享一下给小伙伴吧~](http://wx1.sinaimg.cn/large/006b7Nxngy1g1eu6ewhl9j30760763yz.jpg)