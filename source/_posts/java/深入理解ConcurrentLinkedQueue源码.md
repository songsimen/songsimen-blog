---
title: 并发编程-深入理解ConcurrentLinkedQueue源码
date: 2019-03-29 10:21:09
tags: ConcurrentLinkedQueue
categories: 并发编程

---

## 概述
* `ConcurrentLinkedQueue`是一个基于链接节点的无边界的线程安全队列，它采用先进先出原则对元素进行排序，插入元素放入队列尾部，出队时从队列头部返回元素，利用CAS方式实现的
* `ConcurrentLinkedQueue`的结构由头节点和尾节点组成的，都是使用`volatile`修饰的。每个节点由节点元素`item`和指向下一个节点的`next`引用组成，组成一张链表结构。
* `ConcurrentLinkedQueue`继承自`AbstractQueue`类，实现`Queue`接口

## 常用方法
 * `boolean	add(E e)` 将指定元素插入此队列的尾部，当队列满时，抛出异常
* ` boolean	contains(Object o) ` 判断队列是否包含次元素
 * `boolean	isEmpty()` 判断队列是否为空 
* `boolean	offer(E e)`  将元素插入队列尾部，当队列满时返回false
* `E	peek() `     获取队列头部元素但不删除
* `E	poll()`   获取队列头部元素，并删除
* `boolean	remove(Object o)` 从队列中移指定元素
 * `int	size()`  返回此队列中的元素数量,需要遍历一遍集合。判断队列是否为空时，不推荐此方法

## 源码分析

```java
// 头节点
 private transient volatile Node<E> head;
 //尾节点
 private transient volatile Node<E> tail;
 
 public ConcurrentLinkedQueue() {
 		//初始化构造时 头结点等于尾结点
        head = tail = new Node<E>(null);
 }
 //创建一个最初包含给定 collection 元素的 ConcurrentLinkedQueue，按照此 collection 迭代器的遍历顺序来添加元素。
 public ConcurrentLinkedQueue(Collection<? extends E> c) {
        Node<E> h = null, t = null;
        for (E e : c) {
            checkNotNull(e);
            Node<E> newNode = new Node<E>(e);
            if (h == null)
                h = t = newNode;
            else {
                t.lazySetNext(newNode);
                t = newNode;
            }
        }
        if (h == null)
            h = t = new Node<E>(null);
        head = h;
        tail = t;
    }
    
  private static class Node<E> {
    volatile E item;
    volatile Node<E> next;
    //构造一个新节点
   Node(E item) {
            UNSAFE.putObject(this, itemOffset, item);
    }
    boolean casItem(E cmp, E val) {
           return UNSAFE.compareAndSwapObject(this, itemOffset, cmp, val);
     }
     void lazySetNext(Node<E> val) {
     UNSAFE.putOrderedObject(this, nextOffset, val);
}
    boolean casNext(Node<E> cmp, Node<E> val) {
    return UNSAFE.compareAndSwapObject(this, nextOffset, cmp, val);
  }
 private static final sun.misc.Unsafe UNSAFE;
 //当前结点的偏移量
 private static final long itemOffset;
 //下一个结点的偏移量
 private static final long nextOffset;
     static {
            try {
                UNSAFE = sun.misc.Unsafe.getUnsafe();
                Class<?> k = Node.class;
                itemOffset = UNSAFE.objectFieldOffset
                    (k.getDeclaredField("item"));
                nextOffset = UNSAFE.objectFieldOffset
                    (k.getDeclaredField("next"));
            } catch (Exception e) {
                throw new Error(e);
            }
        }
    }

```
### offer入队操作
### 初始化
* 初始化操作就是创建一个新结点，并且`head`和`tail`相等,结点的数据域为空。
  ![初始化操作](https://img-blog.csdnimg.cn/20190328214940402.png)
* 当第一次入队操作时，检查插入的值是否为空，为空则抛空指针，然后用当前的值创建一个新`Node`结点。然后执行死循环开始入队操作
  ![死循环入队操作](https://img-blog.csdnimg.cn/20190328220017332.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAzOTEzNDI=,size_16,color_FFFFFF,t_70)
* 首先定义了两个指针`p和t`,都指向`tail`
* 然后定义`q`结点存储`p`的next指向的结点,此时p的next是为空没有结点的
  ![q指向p的next](https://img-blog.csdnimg.cn/20190328220810259.png)
* 此时`q==null` 条件成立。执行`p.casNext(null, newNode)`.以cas方式把p的下一个节点指向新创建出来的结点，然后往下执行,`p=t` 直接返回true。此时初始化构造的结点的next指向第一次入队成功的结点
  ![第一次入队成功结构](https://img-blog.csdnimg.cn/2019032822184113.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAzOTEzNDI=,size_16,color_FFFFFF,t_70)
* 第二次入队操作，首先也是非空检查，然后创建一个新结点。此时死循环入队操作。定义了两个指针`p和t`,都指向`tail`。定义`q`结点存储`p`的next指向的结点,此时p的next是不为空的，指向了上面创建的结点。所以`q==null`不成立。执行else操作

![第二次入队q==null不成立](https://img-blog.csdnimg.cn/20190328222604551.png)
* 此时`p`也不等于`q`。`p！=t`不成立，p和t都是指向`tail`。因为不成立所以让`p=q`,此时p和q都是指向第二个结点。再次循循环操作。
  ![](https://img-blog.csdnimg.cn/2019032822351839.png)
* 然后再次`p和t`,都指向`tail`。定义`q`结点存储`p`的next指向的结点。此时p的next指向还是空，所以`q=null`成立。执行`p.casNext(null, newNode)`.以cas方式把`p`的`next`指向新创建出来的结点。
  ![](https://img-blog.csdnimg.cn/20190328224115536.png)
* 此时`  if (p != t)`是成立的 执行`casTail(t, newNode); ` 期望值是`t`,更新值新创建的结点。于是更新了tail结点移动到最后添加的结点

![](https://img-blog.csdnimg.cn/20190328224628328.png)
大概的入队流程就是这样重复上述操作。直到入队成功。`tail`结点并不是每次都是尾结点。所以每次入队都要通过`tail`定位尾结点。
```java
    public boolean offer(E e) {
    	//检查结点是为null，如果插入null则抛出空指针
        checkNotNull(e);
        //构造一个新结点
        final Node<E> newNode = new Node<E>(e);
        //死循换，一直到入队成功
        for (Node<E> t = tail, p = t;;) {
        	//p表示队列尾结点，默认情况尾巴=结点就是taill结点
			//获取p结点的下一个节点
            Node<E> q = p.next;
            //q为空,说明p就是taill结点
            if (q == null) {
            	//case方式设置p(p=t)节点的next指向当前节点
                if (p.casNext(null, newNode)) {
                    if (p != t) 
                    	//p不等于t更新尾结点,
                        casTail(t, newNode); //失败了也是没事的，因为表示有其他线程成功更新了tail节点
                    return true;
                }
                //其他线程抢先完成入队，需要重新尝试
            }
            //q不为空，p和相等
            else if (p == q)
                p = (t != (t = tail)) ? t : head;
            else
            	// // 在两跳之后检查尾部更新.
                p = (p != t && t != (t = tail)) ? t : q;
        }
    }
```
### 出队操作

```java
    public E poll() {
    	//一个标号
        restartFromHead:
        //死循环方
        for (;;) {
        	// 定义p，h两个指针 都指向head
            for (Node<E> h = head, p = h, q;;) {
            	//获取当前p的值
                E item = p.item;
				//值不为空，且以cas方式设置p的item赋值为空。两个条件成立向下执行
                if (item != null && p.casItem(item, null)) {
                	// p和h不相等则更新头结点，否则直接返回值
                    if (p != h) 
                    	//更新头结点，预期值是h,当p的next指向不为空，更新值是q,为空则是p
                        updateHead(h, ((q = p.next) != null) ? q : p);
                        //返回当前p的值
                    return item;
                }
                //如果item为空说明已经被出队了，然后判断q是否null,是空则说明当前队列为空了。但是q = p.next赋值语句已经执行了
                else if ((q = p.next) == null) {
                //更新头结点，预期值h=head,更新p.此时p的item是空，说明已经被出队了
                    updateHead(h, p);
                    return null;
                }
                else if (p == q)
                    continue restartFromHead;
                else
                    p = q;
            }
        }
    }
```
* 出队操作是以死循环的方式直到出队成功。 第一次出队首先执行` for (Node<E> h = head, p = h, q;;)` 定义两个指针`p`和`h`都指向`head`
  ![](https://img-blog.csdnimg.cn/2019032900283295.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAzOTEzNDI=,size_16,color_FFFFFF,t_70)
* 然后定义一个item存储p(这里就是head)的值，然后判断item是否为空，此时第一次出队时为空的，则执行 `else if ((q = p.next) == null)` ,此条件不成立，因为head的next有结点。执行 `else if (p == q)`，此时不相等，因为上个操作已经`把q赋值为p的next结点了`。所以执行最后的else语句  `p = q;`在次循环执行。
  ![](https://img-blog.csdnimg.cn/20190329004142245.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAzOTEzNDI=,size_16,color_FFFFFF,t_70)
* 此时`p.item`不为空条件成立且以`cas`方式更新`p`的i`tem`为空 `p.casItem(item, null)`。如果都两个条件都成立，判断 `if (p != h)`此时不成立的，更新`updateHead(h, ((q = p.next) != null) ? q : p);` 预期值是`h,`更新值是`q` 因为不为空。并返回item,第一次出队成功。

![第一次出队成功结构](https://img-blog.csdnimg.cn/2019032900524254.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAzOTEzNDI=,size_16,color_FFFFFF,t_70)
## 总结
* `CoucurrentLinkedQueue`的结构由头节点和尾节点组成的，都是使用`volatile`修饰的。每个节点由节点元素`item`和指向下一个节点的`next`引用组成.
* `入队`:先检查插入的值是否为空，如果是空则抛出异常。然后以死循坏的方式执行一直到入队成功，整个过程大概就是把`tail`结点的`next`指向新结点，然后更新`tail`为新结点即可。但是`tail`结点并不是每次都是尾结点。所以每次入队都要通过`tail`定位尾结点。
* `出队`：出队操作就是从队列里返回一个最早插入的节点元素，并清空该节点对元素的引用。并不是每次出队都更新`head`节点，当`head`节点有元素时，直接弹出`head`节点的元素，并以`cas`方式设置节点的`item`为`null`,不会更新`head`节点。只有当`head`节点没有元素值时，出队操作才会更新`head`节点，这种做法是为了减少`cas`方式更新`head`节点的消耗，提供出队的效率



![觉得本文不错的话，分享一下给小伙伴吧~](http://wx1.sinaimg.cn/large/006b7Nxngy1g1eu6ewhl9j30760763yz.jpg)