---
title: 并发编程-深入理解ThreadLocal源码
date: 2019-03-28 19:52:09
tags: ThreadLocal
categories: 并发编程
---

## 概述
 - `ThreadLocal` 是一个本地线程副本变量工具类。主要用于将私有线程和该线程存放的副本对象做一个映射，各个线程之间的变量互不干扰。在高并发场景下，可以实现无状态的调用，适用于各个线程不共享变量值的操作。
 - 内部使用静态内部类`ThreadLocalMap`存储每个线程变量副本的方法，key存储的是当前线程的`ThreadLocal`对象，value就是当前`ThreadLocal`对应的线程变量的的副本值。

## 提供方法
* `T	get()`   返回此线程局部变量的当前线程副本中的值。
* `protected  T	initialValue()`   返回此线程局部变量的当前线程的“`初始值`”。线程第一次使用 `get()` 方法访问变量时将调用此方法，但如果线程之前调用了 `set(T)` 方法，则不会对该线程再调用 `initialValue` 方法。通常，此方法对每个线程最多调用一次，但如果在调用 `get()` 后又调用了 `remove()`，则可能再次调用此方法。
 * `void	remove()` 移除此线程局部变量当前线程的值。
 * `void	set(T value)`   将此线程局部变量的当前线程副本中的值设置为指定值。

##  怎嘛使用
```java
public class ThreadLocalTest {
    ThreadLocal<Integer> threadLocal=new ThreadLocal<Integer>(){
        //返回此线程局部变量的当前线程的“初始值”。
        @Override
        protected Integer initialValue() {
            return 0;
        }
    };

    //   返回此线程局部变量的当前线程副本中的值。
    public int get(){
        //将此线程局部变量的当前线程副本中的值设置为指定值。
        threadLocal.set(threadLocal.get()+1);
        return  threadLocal.get();
    }

    public static void main(String[] args) {
        ThreadLocalTest test=new ThreadLocalTest();
            new Thread(()->{
                for (int i = 0; i <3 ; i++) {
                    int state=test.get();
                    System.out.println(Thread.currentThread().getName()+"获取值："+state);
                }

            }).start();

            new Thread(()->{
                for (int i = 0; i <3 ; i++) {
                    int state=test.get();
                    System.out.println(Thread.currentThread().getName()+"获取值："+state);
                }
            }).start();
    }
}

//输出

Thread-0获取值：1
Thread-0获取值：2
Thread-0获取值：3
Thread-1获取值：1
Thread-1获取值：2
Thread-1获取值：3
```
## 源码分析
### set()方法
```java
    public void set(T value) {
    	//记录当前线程
        Thread t = Thread.currentThread();
        //获取当前线》的ThreadLocalMap 
        ThreadLocalMap map = getMap(t);
        if (map != null)
        	//ThreadLocalMap  不为空则直接设置当前变成的副本值，
            map.set(this, value);
        else
        	//创建ThreadLocalMap  key当前线程对象,value：副本值
            createMap(t, value);
    }
```
> `ThreadLocalMap` 内部类

```java
    static class ThreadLocalMap {
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** 与此ThreadLocal关联的值.  */
            Object value;
		
            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
```
* 源码中可以看出  `ThreadLocalMap` 依靠`Entry` 来存储`ThreadLocal`和副本值，key就是`ThreadLocal`，`value`就是`ThreadLocal`的变量副本值。`Entry` 集成`WeakReference`，说明是一个弱引用关系。当一个对象仅仅被弱引用指向, 而没有任何其他强引用指向的时候, 如果这时GC运行, 那么这个对象就会被回收，不论当前的内存空间是否足够，这个对象都会被回收。

```java
	//获取与ThreadLocal关联的Thread中的ThreadLocal。
	
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
```
* `ThreadLocal`是包含在`Thread`类中的

`ThreadLocalMap`的`set`方法



```java
        private void set(ThreadLocal<?> key, Object value) {
            Entry[] tab = table;
            int len = tab.length;
           //计算ThreadLocal 散列值 找到存储位置
            int i = key.threadLocalHashCode & (len-1);
			//利用线性探测法找到合适的存储位置
            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal<?> k = e.get();
				//如果找到的k和传入的key相等，说明存在，覆盖更新即可
                if (k == key) {
                    e.value = value;
                    return;
                }
			   // key == null，但是存在值（因为此处的e != null），说明之前的ThreadLocal对象已经被回收了
                if (k == null) {
                 // 替换之前的的元素
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }
			//不存在对应key的实例，则创建一个新的
            tab[i] = new Entry(key, value);
           //增加容量大小
            int sz = ++size;
            //        // 如果没有清理陈旧的 Entry 并且数组中的元素大于了阈值，则进行 rehash
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
            //整表格的大小。 首先扫描整个表，删除过时的条目。 如果这不足以缩小表的大小，则将表大小加倍。
                rehash();
        }
```
### get()操作
```java
    public T get() {
    	// 记录当前访问线程
        Thread t = Thread.currentThread();
        //获取当前线程的ThreadLocalMap对象
        //Thread的    ThreadLocal.ThreadLocalMap threadLocals参数
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            //存在ThreadLocalMap 则获取相对应的Entry
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                //ThreadLocalMap内部类中有一个Entry内部类
                //依靠Entry`来存储`ThreadLocal`和副本值。直接以ThreadLocal为key获取副本值
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
    
	//getEntry方法
    private Entry getEntry(ThreadLocal<?> key) {
    		//计算ThreadLocal的在数组中的位置，采用了开放定址法
            int i = key.threadLocalHashCode & (table.length - 1);
            Entry e = table[i];
            //存在则返回
            if (e != null && e.get() == key)      
                return e;
            else
            	//不在在的操作
                return getEntryAfterMiss(key, i, e);
        }
      	/**
      	key:线程本地对象
      	i:哈希表的索引
      	e: 对应的Entry
      	*/
        private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
            Entry[] tab = table;
            int len = tab.length;

            while (e != null) {
                ThreadLocal<?> k = e.get();
                if (k == key)
                    return e;
                if (k == null)
                  //key == null，有利于GC回收，能够有效地避免内存泄漏。
                    expungeStaleEntry(i);
                else
                    i = nextIndex(i, len);
                e = tab[i];
            }
            return null;
        }
```

参考：http://cmsblogs.com/?p=2442



![觉得本文不错的话，分享一下给小伙伴吧~](http://wx1.sinaimg.cn/large/006b7Nxngy1g1eu6ewhl9j30760763yz.jpg)