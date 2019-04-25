---
title: 并发编程-深入理解线程池ThreadPoolExecutor
date: 2019-03-31 10:21:09
tags: ThreadPoolExecutor
categories: 并发编程
---

## 概述
* 使用线程池技术可以降低资源的消耗，提高响应速度和线程的可重复利用性
* 当提交一个新任务到线程池后，线程池首先会判断核心线程池(`corePoolSize`）里的线程是否都在执行任务，如果不是则创建一个新的工作线程来执行任务。如果核心线程池`corePoolSize`的线程都被占用在执行任务，线程判断工作队列是否已满，如果工作队列没有满：则将新提交的任务存储到工作队列中，如果工作队列已满：判断线程池（`maximumPoolSize`）的线程是否处于工作状态，如果没有，则创建一个新的工作线程来执行任务。如果线程池已满，则交给饱和策略处理这个任务
  ![线程池处理的流程](http://wx1.sinaimg.cn/large/006b7Nxngy1g1w864isphj30p50a8dhl.jpg)

### 线程池的五种状态
- `RUNNING(运行中)`：线程池处在`RUNNING`状态时，能够接收新任务，以及对已添加的任务进行处理。线程池的初始化状态是RUNNING。
- `SHUTDOWN(关掉)`：调用线程池的`shutdown()`接口时，线程池由`RUNNING -> SHUTDOWN`。处在`SHUTDOWN`状态时，不接收新任务，但能处理已添加的任务。
- `STOP(停止):`调用线程池的`shutdownNow`()接口时，线程池由(`RUNNING or SHUTDOWN` ) `-> STOP`。处在`STOP`状态时，不接收新任务，不处理已添加的任务，并且会中断正在处理的任务。 
- `tidying`：当线程池在`SHUTDOWN`状态下，阻塞队列为空并且线程池中执行的任务也为空时，就会由 `SHUTDOWN -> TIDYING`。 当线程池在`STOP`状态下，线程池中执行的任务为空时，就会由`STOP -> TIDYING`。
- `terminated`(终止)：线程池彻底终止，就变成`terminated`状态。 线程池处在`tidying`状态时，执行完`terminated()`之后，就会由 `tidying -> terminated`。

### 线程池的参数
* `corePoolSize`: 初始化指定的核心线程数量
* `maximumPoolSize`:允许的最大线程数。当前的线程数小于`maximumPoolSize`，则会新建线程来执行任务
* `keepAliveTime`:线程空闲的时间
* `unit`:keepAliveTime的单位
* `workQueue`:保存等待执行的任务的阻塞队列。初始化核心线程池已满时，队列未满会吧任务存储到队列中。
> 可供选择的几种阻塞队列
> `ArrayBlockingQueue`：是一个基于数组结构的有界阻塞队列，此队列按 FIFO（先进先出）原则对元素进行排序。
> `LinkedBlockingQueue`：一个基于链表结构的阻塞队列，此队列按FIFO （先进先出） 排序元素，吞吐量通常要高于ArrayBlockingQueue。静态工厂方法Executors.newFixedThreadPool()使用了这个队列。
> `SynchronousQueue`：一个不存储元素的阻塞队列。每个插入操作必须等到另一个线程调用移除操作，否则插入操作一直处于阻塞状态，吞吐量通常要高于LinkedBlockingQueue，静态工厂方法Executors.newCachedThreadPool使用了这个队列。
> `PriorityBlockingQueue`：一个具有优先级得无限阻塞队列

* `threadFactory`：用于设置创建线程的工厂。不指定 则是默认
* `handler`:线程池的拒绝策略。线程池中的线程已经饱和了，而且阻塞队列也已经满了，则线程池会选择一种拒绝策略来处理该任务

> 线程池提供的`四种拒绝策略`，也可以实现自己的拒绝策略： 

 - `AbortPolicy`：默认策略  抛出异常
 - `CallerRunsPolicy`：由当前调用者所在的线程来执行任务
 - `DiscardOldestPolicy`：丢弃阻塞队列中靠最前的任务，并执行当前任务
 - `DiscardPolicy`：直接丢弃多余的任务

```java
 ThreadPoolExecutor executor=new
                ThreadPoolExecutor(1,1,
                10,TimeUnit.SECONDS,new ArrayBlockingQueue<>(1),new ThreadPoolExecutor.CallerRunsPolicy());
        for (int i = 0; i <5 ; i++) {
            executor.execute(new Runnable() {
                @Override
                public void run() {
                    System.out.println(Thread.currentThread().getName());
                }
            });
        }
        executor.shutdown();
    }
```
## 常用方法
* `getCorePoolSize()`  返回核心线程数。
* `getMaximumPoolSize()`  返回允许的最大线程数。
* `getPoolSize()`   返回池中的当前线程数。
* `getQueue()`     返回此执行程序使用的任务队列。
* `isShutdown()`  如果此执行程序已关闭，则返回 true。
* `isTerminated()`   如果关闭后所有任务都已完成，则返回 true。
* `execute(Runnable command)`  在将来某个时间执行给定任务
* `shutdown()`   按过去执行已提交任务的顺序发起一个有序的关闭，但是不接受新任务。
* `shutdownNow()`    尝试停止所有的活动执行任务、暂停等待任务的处理，并返回等待执行的任务列表。

## 自定义线程名称

```java 
public class ThreadPoolExecutorTest {

    // 命名线程工厂
    static class NamedThreadFactory implements ThreadFactory {
        private static final AtomicInteger poolNumber = new AtomicInteger(1);
        private final ThreadGroup group;
        private final AtomicInteger threadNumber = new AtomicInteger(1);
        private final String namePrefix;

        NamedThreadFactory(String name) {

            SecurityManager s = System.getSecurityManager();
            group = (s != null) ? s.getThreadGroup() : Thread.currentThread().getThreadGroup();
            if (null == name || name.isEmpty()) {
                name = "pool";
            }

            namePrefix = name + "-" + poolNumber.getAndIncrement() + "-thread-";
        }

        public Thread newThread(Runnable r) {
            Thread t = new Thread(group, r, namePrefix + threadNumber.getAndIncrement(), 0);
            if (t.isDaemon())
                t.setDaemon(false);
            if (t.getPriority() != Thread.NORM_PRIORITY)
                t.setPriority(Thread.NORM_PRIORITY);
            return t;
        }
    }

    public static void main(String[] args) {
        ThreadPoolExecutor executor=new
                ThreadPoolExecutor(1,1,
                10,TimeUnit.SECONDS,new ArrayBlockingQueue<>(1),new NamedThreadFactory("test-poll"),new ThreadPoolExecutor.CallerRunsPolicy());
        for (int i = 0; i <5 ; i++) {
            executor.execute(new Runnable() {
                @Override
                public void run() {
                    System.out.println(Thread.currentThread().getName());
                }
            });
        }


        executor.shutdown();
    }
}

```

## 源码分析
### 类中其他属性

```java

    // 线程池的控制状态:用来表示线程池的运行状态（整型的高3位）和运行的worker数量（低29位）
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
    // 29位的偏移量
    private static final int COUNT_BITS = Integer.SIZE - 3;
    // 最大容量（2^29 - 1）
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

    // runState is stored in the high-order bits
    // 线程运行状态，总共有5个状态，需要3位来表示（所以偏移量的29 = 32 - 3）
   /**
    * RUNNING    :    接受新任务并且处理已经进入阻塞队列的任务
    * SHUTDOWN    ：    不接受新任务，但是处理已经进入阻塞队列的任务
    * STOP        :    不接受新任务，不处理已经进入阻塞队列的任务并且中断正在运行的任务
    * TIDYING    :    所有的任务都已经终止，workerCount为0， 线程转化为TIDYING状态并且调用terminated钩子函数
    * TERMINATED:    terminated钩子函数已经运行完成
    **/
    private static final int RUNNING    = -1 << COUNT_BITS;
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    private static final int STOP       =  1 << COUNT_BITS;
    private static final int TIDYING    =  2 << COUNT_BITS;
    private static final int TERMINATED =  3 << COUNT_BITS;
    // 阻塞队列
    private final BlockingQueue<Runnable> workQueue;
    // 可重入锁
    private final ReentrantLock mainLock = new ReentrantLock();
    // 存放工作线程集合
    private final HashSet<Worker> workers = new HashSet<Worker>();
    // 终止条件
    private final Condition termination = mainLock.newCondition();
    // 最大线程池容量
    private int largestPoolSize;
    // 已完成任务数量
    private long completedTaskCount;
    // 线程工厂
    private volatile ThreadFactory threadFactory;
    // 拒绝执行处理器
    private volatile RejectedExecutionHandler handler;
    // 线程等待运行时间
    private volatile long keepAliveTime;
    // 是否运行核心线程超时
    private volatile boolean allowCoreThreadTimeOut;
    // 核心池的大小
    private volatile int corePoolSize;
    // 最大线程池大小
    private volatile int maximumPoolSize;
    // 默认拒绝执行处理器
    private static final RejectedExecutionHandler defaultHandler =
        new AbortPolicy();
```

### 构造方法

```java
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||                                                // 核心大小不能小于0
            maximumPoolSize <= 0 ||                                            // 线程池的初始最大容量不能小于0
            maximumPoolSize < corePoolSize ||                                // 初始最大容量不能小于核心大小
            keepAliveTime < 0)                                                // keepAliveTime不能小于0
            throw new IllegalArgumentException();                                
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        // 初始化相应的域
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```

### 提交任务

```java
/*
* 进行下面三步
*
* 1. 如果运行的线程小于corePoolSize,则尝试使用用户定义的Runnalbe对象创建一个新的线程
*     调用addWorker函数会原子性的检查runState和workCount，通过返回false来防止在不应
*     该添加线程时添加了线程
* 2. 如果一个任务能够成功入队列，在添加一个线城时仍需要进行双重检查（因为在前一次检查后
*     该线程死亡了），或者当进入到此方法时，线程池已经shutdown了，所以需要再次检查状态，
*    若有必要，当停止时还需要回滚入队列操作，或者当线程池没有线程时需要创建一个新线程
* 3. 如果无法入队列，那么需要增加一个新线程，如果此操作失败，那么就意味着线程池已经shut
*     down或者已经饱和了，所以拒绝任务
*/
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    // 获取线程池控制状态
    int c = ctl.get();
    if (workerCountOf(c) < corePoolSize) { // worker数量小于corePoolSize
        if (addWorker(command, true)) // 添加worker
            // 成功则返回
            return;
        // 不成功则再次获取线程池控制状态
        c = ctl.get();
    }
    // 线程池处于RUNNING状态，将用户自定义的Runnable对象添加进workQueue队列
    if (isRunning(c) && workQueue.offer(command)) { 
        // 再次检查，获取线程池控制状态
        int recheck = ctl.get();
        // 线程池不处于RUNNING状态，将自定义任务从workQueue队列中移除
        if (! isRunning(recheck) && remove(command)) 
            // 拒绝执行命令
            reject(command);
        else if (workerCountOf(recheck) == 0) // worker数量等于0
            // 添加worker
            addWorker(null, false);
    }
    else if (!addWorker(command, false)) // 添加worker失败
        // 拒绝执行命令
        reject(command);
}
```

#### addWorker

1. 原子性的增加workerCount。

2. 将用户给定的任务封装成为一个worker，并将此worker添加进workers集合中。

3. 启动worker对应的线程，并启动该线程，运行worker的run方法。

4. 回滚worker的创建动作，即将worker从workers集合中删除，并原子性的减少workerCount。

```java
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) { // 外层无限循环
        // 获取线程池控制状态
        int c = ctl.get();
        // 获取状态
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
        if (rs >= SHUTDOWN &&            // 状态大于等于SHUTDOWN，初始的ctl为RUNNING，小于SHUTDOWN
            ! (rs == SHUTDOWN &&        // 状态为SHUTDOWN
               firstTask == null &&        // 第一个任务为null
               ! workQueue.isEmpty()))     // worker队列不为空
            // 返回
            return false;

        for (;;) {
            // worker数量
            int wc = workerCountOf(c);
            if (wc >= CAPACITY ||                                // worker数量大于等于最大容量
                wc >= (core ? corePoolSize : maximumPoolSize))    // worker数量大于等于核心线程池大小或者最大线程池大小
                return false;
            if (compareAndIncrementWorkerCount(c))                 // 比较并增加worker的数量
                // 跳出外层循环
                break retry;
            // 获取线程池控制状态
            c = ctl.get();  // Re-read ctl
            if (runStateOf(c) != rs) // 此次的状态与上次获取的状态不相同
                // 跳过剩余部分，继续循环
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }

    // worker开始标识
    boolean workerStarted = false;
    // worker被添加标识
    boolean workerAdded = false;
    // 
    Worker w = null;
    try {
        // 初始化worker
        w = new Worker(firstTask);
        // 获取worker对应的线程
        final Thread t = w.thread;
        if (t != null) { // 线程不为null
            // 线程池锁
            final ReentrantLock mainLock = this.mainLock;
            // 获取锁
            mainLock.lock();
            try {
                // Recheck while holding lock.
                // Back out on ThreadFactory failure or if
                // shut down before lock acquired.
                // 线程池的运行状态
                int rs = runStateOf(ctl.get());

                if (rs < SHUTDOWN ||                                    // 小于SHUTDOWN
                    (rs == SHUTDOWN && firstTask == null)) {            // 等于SHUTDOWN并且firstTask为null
                    if (t.isAlive()) // precheck that t is startable    // 线程刚添加进来，还未启动就存活
                        // 抛出线程状态异常
                        throw new IllegalThreadStateException();
                    // 将worker添加到worker集合
                    workers.add(w);
                    // 获取worker集合的大小
                    int s = workers.size();
                    if (s > largestPoolSize) // 队列大小大于largestPoolSize
                        // 重新设置largestPoolSize
                        largestPoolSize = s;
                    // 设置worker已被添加标识
                    workerAdded = true;
                }
            } finally {
                // 释放锁
                mainLock.unlock();
            }
            if (workerAdded) { // worker被添加
                // 开始执行worker的run方法
                t.start();
                // 设置worker已开始标识
                workerStarted = true;
            }
        }
    } finally {
        if (! workerStarted) // worker没有开始
            // 添加worker失败
            addWorkerFailed(w);
    }
    return workerStarted;
}
```

### 执行任务

runWorker函数中会实际执行给定任务（即调用用户重写的run方法），并且当给定任务完成后，会继续从阻塞队列中取任务，直到阻塞队列为空（即任务全部完成）。在执行给定任务时，会调用钩子函数，利用钩子函数可以完成用户自定义的一些逻辑。在runWorker中会调用到getTask函数和processWorkerExit钩子函数

```java
final void runWorker(Worker w) {
    // 获取当前线程
    Thread wt = Thread.currentThread();
    // 获取w的firstTask
    Runnable task = w.firstTask;
    // 设置w的firstTask为null
    w.firstTask = null;
    // 释放锁（设置state为0，允许中断）
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
        while (task != null || (task = getTask()) != null) { // 任务不为null或者阻塞队列还存在任务
            // 获取锁
            w.lock();
            // If pool is stopping, ensure thread is interrupted;
            // if not, ensure thread is not interrupted.  This
            // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt
            if ((runStateAtLeast(ctl.get(), STOP) ||    // 线程池的运行状态至少应该高于STOP
                 (Thread.interrupted() &&                // 线程被中断
                  runStateAtLeast(ctl.get(), STOP))) &&    // 再次检查，线程池的运行状态至少应该高于STOP
                !wt.isInterrupted())                    // wt线程（当前线程）没有被中断
                wt.interrupt();                            // 中断wt线程（当前线程）
            try {
                // 在执行之前调用钩子函数
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    // 运行给定的任务
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    // 执行完后调用钩子函数
                    afterExecute(task, thrown);
                }
            } finally {
                task = null;
                // 增加给worker完成的任务数量
                w.completedTasks++;
                // 释放锁
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        // 处理完成后，调用钩子函数
        processWorkerExit(w, completedAbruptly);
    }
}
```

此函数用于从workerQueue阻塞队列中获取Runnable对象，由于是阻塞队列，所以支持有限时间等待（poll）和无限时间等待（take）。在该函数中还会响应shutDown和、shutDownNow函数的操作，若检测到线程池处于SHUTDOWN或STOP状态，则会返回null，而不再返回阻塞队列中的Runnalbe对象。

```java
    private Runnable getTask() {
        boolean timedOut = false; // Did the last poll() time out?

        for (;;) { // 无限循环，确保操作成功
            // 获取线程池控制状态
            int c = ctl.get();
            // 运行的状态
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) { // 大于等于SHUTDOWN（表示调用了shutDown）并且（大于等于STOP（调用了shutDownNow）或者worker阻塞队列为空）
                // 减少worker的数量
                decrementWorkerCount();
                // 返回null，不执行任务
                return null;
            }
            // 获取worker数量
            int wc = workerCountOf(c);

            // Are workers subject to culling?
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize; // 是否允许coreThread超时或者workerCount大于核心大小

            if ((wc > maximumPoolSize || (timed && timedOut))     // worker数量大于maximumPoolSize
                && (wc > 1 || workQueue.isEmpty())) {            // workerCount大于1或者worker阻塞队列为空（在阻塞队列不为空时，需要保证至少有一个wc）
                if (compareAndDecrementWorkerCount(c))            // 比较并减少workerCount
                    // 返回null，不执行任务，该worker会退出
                    return null;
                // 跳过剩余部分，继续循环
                continue;
            }

            try {
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :    // 等待指定时间
                    workQueue.take();                                        // 一直等待，直到有元素
                if (r != null)
                    return r;
                // 等待指定时间后，没有获取元素，则超时
                timedOut = true;
            } catch (InterruptedException retry) {
                // 抛出了被中断异常，重试，没有超时
                timedOut = false;
            }
        }
    }
```

processWorkerExit函数是在worker退出时调用到的钩子函数，而引起worker退出的主要因素如下

1. 阻塞队列已经为空，即没有任务可以运行了。

2. 调用了shutDown或shutDownNow函数

此函数会根据是否中断了空闲线程来确定是否减少workerCount的值，并且将worker从workers集合中移除并且会尝试终止线程池。

```java
    private void processWorkerExit(Worker w, boolean completedAbruptly) {
        if (completedAbruptly) // 如果被中断，则需要减少workCount    // If abrupt, then workerCount wasn't adjusted
            decrementWorkerCount();
        // 获取可重入锁
        final ReentrantLock mainLock = this.mainLock;
        // 获取锁
        mainLock.lock();
        try {
            // 将worker完成的任务添加到总的完成任务中
            completedTaskCount += w.completedTasks;
            // 从workers集合中移除该worker
            workers.remove(w);
        } finally {
            // 释放锁
            mainLock.unlock();
        }
        // 尝试终止
        tryTerminate();
        // 获取线程池控制状态
        int c = ctl.get();
        if (runStateLessThan(c, STOP)) { // 小于STOP的运行状态
            if (!completedAbruptly) {
                int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
                if (min == 0 && ! workQueue.isEmpty()) // 允许核心超时并且workQueue阻塞队列不为空
                    min = 1;
                if (workerCountOf(c) >= min) // workerCount大于等于min
                    // 直接返回
                    return; // replacement not needed
            }
            // 添加worker
            addWorker(null, false);
        }
    }
```

### 关闭线程池

```java
    public void shutdown() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            // 检查shutdown权限
            checkShutdownAccess();
            // 设置线程池控制状态为SHUTDOWN
            advanceRunState(SHUTDOWN);
            // 中断空闲worker
            interruptIdleWorkers();
            // 调用shutdown钩子函数
            onShutdown(); // hook for ScheduledThreadPoolExecutor
        } finally {
            mainLock.unlock();
        }
        // 尝试终止
        tryTerminate();
    }
```

```java
    final void tryTerminate() {
        for (;;) { // 无限循环，确保操作成功
            // 获取线程池控制状态
            int c = ctl.get();
            if (isRunning(c) ||                                            // 线程池的运行状态为RUNNING
                runStateAtLeast(c, TIDYING) ||                            // 线程池的运行状态最小要大于TIDYING
                (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))    // 线程池的运行状态为SHUTDOWN并且workQueue队列不为null
                // 不能终止，直接返回
                return;
            if (workerCountOf(c) != 0) { // 线程池正在运行的worker数量不为0    // Eligible to terminate
                // 仅仅中断一个空闲的worker
                interruptIdleWorkers(ONLY_ONE);
                return;
            }
            // 获取线程池的锁
            final ReentrantLock mainLock = this.mainLock;
            // 获取锁
            mainLock.lock();
            try {
                if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) { // 比较并设置线程池控制状态为TIDYING
                    try {
                        // 终止，钩子函数
                        terminated();
                    } finally {
                        // 设置线程池控制状态为TERMINATED
                        ctl.set(ctlOf(TERMINATED, 0));
                        // 释放在termination条件上等待的所有线程
                        termination.signalAll();
                    }
                    return;
                }
            } finally {
                // 释放锁
                mainLock.unlock();
            }
            // else retry on failed CAS
        }
    }
```

```java
    private void interruptIdleWorkers(boolean onlyOne) {
        // 线程池的锁
        final ReentrantLock mainLock = this.mainLock;
        // 获取锁
        mainLock.lock();
        try {
            for (Worker w : workers) { // 遍历workers队列
                // worker对应的线程
                Thread t = w.thread;
                if (!t.isInterrupted() && w.tryLock()) { // 线程未被中断并且成功获得锁
                    try {
                        // 中断线程
                        t.interrupt();
                    } catch (SecurityException ignore) {
                    } finally {
                        // 释放锁
                        w.unlock();
                    }
                }
                if (onlyOne) // 若只中断一个，则跳出循环
                    break;
            }
        } finally {
            // 释放锁
            mainLock.unlock();
        }
    }
```
* ![觉得本文不错的话，分享一下给小伙伴吧~](http://wx1.sinaimg.cn/large/006b7Nxngy1g1eu6ewhl9j30760763yz.jpg)