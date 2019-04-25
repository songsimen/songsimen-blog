---
title: volatile的内存语义
date: 2019-04-05 14:29:09
tags: volatile
categories: 并发编程
---

## volatile的特性
- `volatile`修饰的变量可以禁止指令重排序和保证了内存可见性和单一操作的原子性，类似`i++`这样的复合操作的原子性保证不了
- 有`volatile`关键字修饰的共享变量进行写操作数，会多出一个`lock`前缀指令。`lock`前缀指令其实就相当于一个内存屏障。在多处理器下，会将当前处理器工作内存的数据回写到主内存中，并且这个回写操作会其它线程中缓存该内存地址的数据无效。相当于会在写操作后，发出一个信号给缓存了这个数的线程，告诉它们值更新了，需要从主内存中从新获取
 - 在`JVM`底层`volatile`是采用“`内存屏障`”来实现的。
- `volatile`经常用于两个两个场景：状态标记两、单列模式中的`DCL`

## volatile写-读建立的happens-before关系

```java
  private  int  count;  //普通变量
  private  volatile  boolean falg;  //volatile 修饰的变量
    //写操作
    public  void  writer(){
        count=1;   // 1
        falg=true;  //2
    }
    // 读操作
    public  void reader(){
        if(falg){                   //3
            int  sum=count+1;       // 4
        }
    }
```
* 假设有两个线程：线程`A`调用读方法， 线程`B`调用写方法
  根据happens-before规则，这个过程的建立分为三类：
1. `程序次序规则`： 1 happens-before 2,3 happens-before 4
2. `volatile规则`：2 happens-before 3 。对一个volatile变量的写操作先行发生于后面对这个变量的读操作
3. `传递规则`： 1 happens-before 4 ；

![](https://img-blog.csdnimg.cn/20190405130152624.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAzOTEzNDI=,size_16,color_FFFFFF,t_70)
* 如果`falg`不是volatile修饰的，那么`操作1`和`操作2`之间没有数据依赖性，处理器可能会对这两个操作进行`重排序`，这时`线程A`正好执行先执行了`操作2`，然后这时`线程B`抢先执行了`操作3`, 发现为`true`就执行`if语句`里的代码， 得到值可能就是`1`，而不是我们所预想的输出`sum=2`。

##  volatile写-读的内存语义

* `volatile写操作`：当对一个volatile共享变量写操作时，JMM会当前线程对应的更新的后的本地内存中的值强制刷新到主内存中
* `volatile读操作`：当读一个`volatile`共享变量时，JMM会把当前线程对应的本地内存`标记为无效`，然后线程会从主内存中加载最新的值到工作内存中进行操作。
  ![](https://img-blog.csdnimg.cn/20190405131645165.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAzOTEzNDI=,size_16,color_FFFFFF,t_70)
* 线程A写一个`volatile`变量，其实就是新城A向接下来要读取这个共享变量的某个线程，发送了一个信号，告诉它我已经修改了共享变量，你的工作内存的值要被标记无效。
* 线程B读一个`volatile`变量，其实就是接收了之前线程A发出的修改共享变量的信号。
* 对一个volatile变量的写操作，随后对这个变量的读操作，其实就是两个线程之间的进行了通讯。

## volatile的内存语义的实现
* 重排序分为编译器重排序和处理器重排序，为了实现volatile内存语义，JMM会分别限制这两种重排序的内型。
> `volatilec`重排序规则

| 第一个操作 | 第二个操作                                                   |
| ---------- | ------------------------------------------------------------ |
| 普通读/写  | 普通读/写: yes ,      `volatile`读 ：yes,           `volatile`写 ：no, |
| volatile读 | 普通读/写: no ,      `volatile`读 ：no,           `volatile`写 ：no, |
| volatile写 | 普通读/写: yes ,      `volatile`读 ：no,           `volatile`写 ：no, |

* 当第一个操作为普通变量的读/写时，如果第二个操作是`volatile`写，则编译器不能重排序这个两个操作。
* 当第一个操作是`volatile`读时,第二个操作不管是什么都不能重排序，这个规则确保volatile读之后的操作不会排序的它之前。
* 当一个操作是volatile写时，第二个操作时volatile读时，不能重排序

> 为了实现`volatile`内存语义，编译器生成字节码时，会在指令序列中插入内存屏障来禁止特定类型的处理器重排序。

* 在每个`volatile写`之前插入一个`StoreStore`屏障
* 在每个`volatile写`操作的后面插入一个StoreLoad屏障
* 在每个`volatile读`操作的后面插入一个LoadLoad屏障
* 在每个`volatile读`操作的后面插入一个LoadStore屏障
![volatile写指令序列示意图](http://wx1.sinaimg.cn/large/006b7Nxngy1g1w7ueh8mdj30nk0ed427.jpg)

![volatile读指令序列示意图](http://wx1.sinaimg.cn/large/006b7Nxngy1g1w833hp28j30ot0eh0wk.jpg)
