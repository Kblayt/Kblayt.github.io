---
title: 9、深入理解AQS之独占锁ReentrantLock源码分析
date: 2023-06-16 23:37:30
categories:
    - 系统架构
    - 架构之并发编程
tags:
    - 并发编程
    - JUC
---

# **AQS原理分析**

## **什么是AQS**

java.util.concurrent包中的大多数同步器实现都是围绕着共同的基础行为，比如等待队列、条件队列、独占获取、共享获取等，而这些行为的抽象就是基于<font color='red'>AbstractQueuedSynchronizer（简称AQS）</font>实现的，AQS是一个抽象同步框架，可以用来实现一个依赖状态的同步器。

JDK中提供的大多数的同步器如Lock, Latch, Barrier等，都是基于AQS框架来实现的

- 一般是通过一个内部类Sync继承 AQS
- 将同步器所有调用都映射到Sync对应的方法

​    ![0](9%E3%80%81%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3AQS%E4%B9%8B%E7%8B%AC%E5%8D%A0%E9%94%81ReentrantLock%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/1715.png)

AQS具备的特性：

- 阻塞等待队列
- 共享/独占
- 公平/非公平
- 可重入
- 允许中断 

AQS内部维护属性<font color='red'>volatile int state</font> 

- state表示资源的可用状态

State三种访问方式：

- getState() 
- setState() 
- compareAndSetState()

AQS定义两种资源共享方式

- Exclusive-独占，只有一个线程能执行，如ReentrantLock
- Share-共享，多个线程可以同时执行，如Semaphore/CountDownLatch

AQS定义两种队列

- 同步等待队列： <font color='red'>主要用于维护获取锁失败时入队的线程</font>
- 条件等待队列：<font color='red'> 调用await()的时候会释放锁，然后线程会加入到条件队列，调用signal()唤醒的时候会把条件队列中的线程节点移动到同步队列中，等待再次获得锁</font>

AQS 定义了5个队列中节点状态：

1. 值为0，初始化状态，表示当前节点在sync队列中，等待着获取锁。
2. CANCELLED，值为1，表示当前的线程被取消；
3. SIGNAL，值为-1，表示当前节点的后继节点包含的线程需要运行，也就是unpark；
4. CONDITION，值为-2，表示当前节点在等待condition，也就是在condition队列中；
5. PROPAGATE，值为-3，表示当前场景下后续的acquireShared能够得以执行；

不同的自定义同步器竞争共享资源的方式也不同。自定义同步器在实现时只需要实现共享资源state的获取与释放方式即可，至于具体线程等待队列的维护（如获取资源失败入队/唤醒出队等），AQS已经在顶层实现好了。<font color='red'>自定义同步器实现时主要实现以下几种方法：</font>

- isHeldExclusively()：该线程是否正在独占资源。只有用到condition才需要去实现它。
- <font color='red'>tryAcquire(int)：独占方式。</font>尝试获取资源，成功则返回true，失败则返回false。
- <font color='red'>tryRelease(int)：独占方式。</font>尝试释放资源，成功则返回true，失败则返回false。
- <font color='red'>tryAcquireShared(int)：共享方式。</font>尝试获取资源。负数表示失败；0表示成功，但没有剩余可用资源；正数表示成功，且有剩余资源。
- <font color='red'>tryReleaseShared(int)：共享方式。</font>尝试释放资源，如果释放后允许唤醒后续等待结点返回true，否则返回false。

## **同步等待队列**

AQS当中的同步等待队列也称CLH队列，CLH队列是Craig、Landin、Hagersten三人发明的一种<font color='red'>基于双向链表数据结构的队列，是FIFO先进先出线程等待队列，</font>Java中的CLH队列是原CLH队列的一个变种,线程由原自旋机制改为阻塞机制。

AQS 依赖CLH同步队列来完成同步状态的管理：

- 当前线程如果获取同步状态失败时，AQS则会将当前线程已经等待状态等信息构造成一个节点（Node）并将其加入到CLH同步队列，同时会阻塞当前线程
- 当同步状态释放时，会把首节点唤醒（公平锁），使其再次尝试获取同步状态。
- 通过signal或signalAll将条件队列中的节点转移到同步队列。（<font color='red'>由条件队列转化为同步队列</font>）

​    ![0](9%E3%80%81%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3AQS%E4%B9%8B%E7%8B%AC%E5%8D%A0%E9%94%81ReentrantLock%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/1716.png)

## **条件等待队列**

AQS中条件队列是使用单向列表保存的，用nextWaiter来连接:

- 调用await方法阻塞线程；
- 当前线程存在于同步队列的头结点，调用await方法进行阻塞（<font color='red'>从同步队列转化到条件队列</font>）

## **Condition接口详解**

​    ![0](9%E3%80%81%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3AQS%E4%B9%8B%E7%8B%AC%E5%8D%A0%E9%94%81ReentrantLock%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/1714.png)

1. 调用Condition#await方法会释放当前持有的锁，然后阻塞当前线程，同时向Condition队列尾部添加一个节点，所以调用Condition#await方法的时候必须持有锁。
2. 调用Condition#signal方法会将Condition队列的首节点移动到阻塞队列尾部，然后唤醒因调用Condition#await方法而阻塞的线程(唤醒之后这个线程就可以去竞争锁了)，所以调用Condition#signal方法的时候必须持有锁，持有锁的线程唤醒被因调用Condition#await方法而阻塞的线程。

### **等待唤醒机制之await/signal测试**

```java
@Slf4j
public class ConditionTest {

    public static void main(String[] args) {

        Lock lock = new ReentrantLock();
        Condition condition = lock.newCondition();

        new Thread(() -> {
            lock.lock();
            try {
                log.debug(Thread.currentThread().getName() + " 开始处理任务");
                condition.await();
                log.debug(Thread.currentThread().getName() + " 结束处理任务");
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                lock.unlock();
            }
        }).start();

        new Thread(() -> {
            lock.lock();
            try {
                log.debug(Thread.currentThread().getName() + " 开始处理任务");

                Thread.sleep(2000);
                condition.signal();
                log.debug(Thread.currentThread().getName() + " 结束处理任务");
            } catch (Exception e) {
                e.printStackTrace();
            } finally {
                lock.unlock();
            }
        }).start();
    }
}
```

​     

# JUC核心类AQS底层实现

## 一、AQS内部结构介绍

AQS其实本质就是Java中的一个抽象类：AbstractQueuedSynchronizer

JUC是Java中一个包java.util.concurrent。在这个包下，基本存放了Java中几乎所有的并发类，包括并发工具，并发集合，锁等等。

AQS是JUC下的一个基础类，大多数的并发工具都是基于AQS实现。AQS本质并没有实现太多的业务功能，只是对外提供了**三点核心**内容，来帮助实现其他的并发内容。

三个核心内容：

- int state
  - 比如锁ReentrantLock或者ReentrantReadWriteLock，他们获取锁的方式都是对state变量做修改实现的
  - 比如CountDownLatch基于state作为计数器，同样的Semaphore也是用state记录资源个数。
- Node对象组成的双向链表(CLH同步等待队列)
  - 比如ReentrantLock，有一个线程没拿到资源，当前线程需要等一会，需要将线程封装为Node对象，将Node添加到双向链表，将线程挂起，等待即可。
- Node对象组成的单向链表(Condition)
  - 比如ReentrantLock，一个线程持有锁时，执行了await方法(wait)，此时这个线程就需要封装为Node对象，并且添加到单项链表。

## 二、Lock锁和AQS的关系

ReentrantLock就是基于AQS实现的。

![image-20240501193218985](9%E3%80%81%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3AQS%E4%B9%8B%E7%8B%AC%E5%8D%A0%E9%94%81ReentrantLock%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/image-20240501193218985.png)

为什么修改地址的顺序是1、2、3？

如果是先修改2，则再修改完2之后此时不再是链表结构，无法进行数据查询。
如果先修改3，如果发生两个节点同时修改，可能会导致链表数据断开。

## 三、AQS&Lock锁的tryAcquire

从ReentrantLock的lock方法开始读源码。

当看到lock方法时发现，执行了sync的lock方法。

Sync是一个抽象类，继承了AQS。

Sync有两个子类实现：

- FairSync：公平锁
- NonFairSync：非公平锁

Sync的lokc方法实现：

- 非公平锁：

  - ```java
    final void lock() {
    	//	直接CAS尝试将state从0改为1，成功就拿到锁资源
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else
        //	如果cas失败旧执行acquire方法，走后续操作
            acquire(1);
    }
    ```

- 公平锁：

  - ```java
    final void lock() {
        acquire(1);
    }
    ```

如果lock锁没有成功拿到锁资源，需要执行acquire走后续。

acquire方法是AQS提供的，部分为公平和非公平，他两都走一套逻辑。

```java
public final void acquire(int arg) {
	//	1、查看tryAcquire：再次尝试拿锁
	//	2、查看addWaiter：没拿到锁，要去排队了
	//	3、查看acquireQueued：挂起线程和后续继唤醒继续获取锁资源的逻辑
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

tryAcquire就是基于lock没拿到所之后要继续做的事情。

AQS中提供的tryAcquire直接跑出了异常，并没有提供具体的实现逻辑。

在公平锁和非公平锁中，各自有各自的实现。

- 非公平锁：

```java
//	非公平锁再次尝试拿锁
final boolean nonfairTryAcquire(int acquires) {
	//	获取当前线程对象
    final Thread current = Thread.currentThread();
    //	获取state属性
    int c = getState();
    //	锁资源现在没有线程持有，可以尝试获取锁
    if (c == 0) {
    	//	非公平锁直接一个CAS，尝试将state从0改为1
        if (compareAndSetState(0, acquires)) {
        	//	成功代表拿锁成功，完事
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    //	锁资源现在被某个线程持有，同事看一下持有锁的线程是不是我自己
    else if (current == getExclusiveOwnerThread()) {
    	//	持有锁资源的线程，就是我自己。(ReentrantLock是可重入锁)
        int nextc = c + acquires;
        //	对state + 1
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        //	设置给state，代表锁重入成功
        setState(nextc);
        return true;
    }
    return false;
}
```

- 公平锁

```java
//	公平锁的实现逻辑
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
    	//	当前锁资源没有被其他线程持有
    	//	hasQueuedPredecessors：虽然所资源没有被持有，但是我要看下有没有线程正在排队
    	//	1、没有现成排队：抢锁
    	//	2、有线程排队，查看当前线程是否排在第一位，如果是，抢锁。
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
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
```

## 四、AQS的addWaiter

addWaiter方法，就是将当前线程封装为Node对象，并且插入到AQS的双向链表中。

```java
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    //	获取到tail
    Node pred = tail;
    //	tail还没初始化时不能走这个逻辑
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    enq(node);
    return node;
}
private Node enq(final Node node) {
    //	死循环
        for (;;) {
            // 获取到tail，用t指向
            Node t = tail;
            //	当tail还为null时，进入if初始化虚拟的Node节点
            if (t == null) { // Must initialize
                if (compareAndSetHead(new Node()))
                    tail = head;
            //	完成当前Node加入到AQS双向链表的过程
            } else {
                //	1、prev 2、tail 3、next
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
}
```

## 五、AQS的acquireQueued

这里就是线程挂起以及重新尝试获取锁资源的位置

重新尝试获取锁资源有两种情况：

- 上来就排在head.next
- 唤醒之后尝试拿锁

```java
//	在线程Node添加到AQS队列后的操作
final boolean acquireQueued(final Node node, int arg) {
	//	一个标记，代表拿锁失败了
    boolean failed = true;
    try {
    	// 死循环，代表一直尝试去拿锁资源，拿不到救死等
        boolean interrupted = false;
        for (;;) {
        	// 获取当前节点的上一个节点，相当于获取prev
            final Node p = node.predecessor();
            //	如果上一个节点是head，代表当前节点排在第一个位置。
            //	如果是第一个位置，执行tryAcquire尝试拿锁
            if (p == head && tryAcquire(arg)) {
            	//	是第一个位置并且拿到锁资源了
                setHead(node);
                //	将上一个node指向null等待GC
                p.next = null; // help GC
                //	代表拿锁成功了
                failed = false;
                return interrupted;
            }
            //	当前线程没拿到锁
            //	shouldParkAfterFailedAcquire：挂起线程前的准备
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

![image-20240503144156976](9%E3%80%81%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3AQS%E4%B9%8B%E7%8B%AC%E5%8D%A0%E9%94%81ReentrantLock%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/image-20240503144156976.png)

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    //	ws是上一个节点状态
    int ws = pred.waitStatus;
    //	如果是Node.SIGNAL -1
    if (ws == Node.SIGNAL)
    	//	返回true
        return true;
    if (ws > 0) {
    	//	上一个节点是取消状态
    	//	往前找到状态不是取消的节点，并且修改好指针状态
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
    	//	节点状态不是1，代表没取消，将节点状态从之前的值修改为Node.SIGNAL -1
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```

## 六、AQS&Lock锁的release

**详细看ReentrantLock.unlock()**

```java
public void unlock() {
    sync.release(1);
}
```

加锁，是每次对state + 1

释放锁每次对state - 1

获取锁资源，是将state从0改为1。

释放锁资源，state为0表示释放干净了。

释放锁资源干净后，可能需要唤醒后面挂起的线程。当ws=-1的时候代表后面有线程挂起了。

如果唤醒线程时，发现要唤醒的head.next为取消状态，那我们需要唤醒第一个为正常状态的节点，从tail开始从后向前找，从head开始无法找到，因为取消状态节点的node.next指向null了





# **ReentrantLock详解**

<font color="red">ReentrantLock是一种基于AQS框架的应用实现</font>，是JDK中的一种线程并发访问的同步手段，它的功能类似于synchronized<font color="red">是一种互斥锁，可以保证线程安全。</font>

相对于 synchronized， ReentrantLock具备如下特点：

- 可中断 
- 可以设置超时时间
- 可以设置为公平锁 
- 支持多个条件变量
- 与 synchronized 一样，都支持可重入

​    ![0](9%E3%80%81%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3AQS%E4%B9%8B%E7%8B%AC%E5%8D%A0%E9%94%81ReentrantLock%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/1718.png)

顺便总结了几点synchronized和ReentrantLock的区别：

- synchronized是JVM层次的锁实现，ReentrantLock是JDK层次的锁实现；
- synchronized的锁状态是无法在代码中直接判断的，但是ReentrantLock可以通过ReentrantLock#isLocked判断；
- synchronized是非公平锁，ReentrantLock是可以是公平也可以是非公平的；
- synchronized是不可以被中断的，而ReentrantLock#lockInterruptibly方法是可以被中断的；
- 在发生异常时synchronized会自动释放锁，而ReentrantLock需要开发者在finally块中显示释放锁；
- ReentrantLock获取锁的形式有多种：如立即返回是否成功的tryLock(),以及等待指定时长的获取，更加灵活；
- synchronized在特定的情况下对于已经在等待的线程是后来的线程先获得锁（回顾一下sychronized的唤醒策略），而ReentrantLock对于已经在等待的线程是先来的线程先获得锁；

## **ReentrantLock的使用**

### **同步执行，类似于synchronized**

```java
ReentrantLock lock = new ReentrantLock(); //参数默认false，不公平锁  
ReentrantLock lock = new ReentrantLock(true); //公平锁  

//加锁    
lock.lock(); 
try {  
    //临界区 
} finally { 
    // 解锁 
    lock.unlock();  
}
```

​      

测试

```java
public class ReentrantLockDemo {

    private static  int sum = 0;
    private static Lock lock = new ReentrantLock();

    public static void main(String[] args) throws InterruptedException {

        for (int i = 0; i < 3; i++) {
            Thread thread = new Thread(()->{
                //加锁
                lock.lock();
                try {
                    for (int j = 0; j < 10000; j++) {
                        sum++;
                    }
                } finally {
                    // 解锁
                    lock.unlock();
                }
            });
            thread.start();
        }
        Thread.sleep(2000);
        System.out.println(sum);
    }
}
```

### **可重入**

```java
@Slf4j
public class ReentrantLockDemo2 {

    public static ReentrantLock lock = new ReentrantLock();

    public static void main(String[] args) {
        method1();
    }


    public static void method1() {
        lock.lock();
        try {
            log.debug("execute method1");
            method2();
        } finally {
            lock.unlock();
        }
    }
    public static void method2() {
        lock.lock();
        try {
            log.debug("execute method2");
            method3();
        } finally {
            lock.unlock();
        }
    }
    public static void method3() {
        lock.lock();
        try {
            log.debug("execute method3");
        } finally {
            lock.unlock();
        }
    }
}
```

​         

### **可中断** 

```java
@Slf4j
public class ReentrantLockDemo3 {

    public static void main(String[] args) {
        ReentrantLock lock = new ReentrantLock();

        Thread t1 = new Thread(() -> {

            log.debug("t1启动...");

            try {
                lock.lockInterruptibly();
                try {
                    log.debug("t1获得了锁");
                } finally {
                    lock.unlock();
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
                log.debug("t1等锁的过程中被中断");
            }

        }, "t1");

        lock.lock();
        try {
            log.debug("main线程获得了锁");
            t1.start();
            //先让线程t1执行
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            t1.interrupt();
            log.debug("线程t1执行中断");
        } finally {
            lock.unlock();
        }
    }
}
```

​     

### **锁超时**

**立即失败**

```java
@Slf4j
public class ReentrantLockDemo4 {

    public static void main(String[] args) {
        ReentrantLock lock = new ReentrantLock();

        Thread t1 = new Thread(() -> {

            log.debug("t1启动...");
            // 注意： 即使是设置的公平锁，此方法也会立即返回获取锁成功或失败，公平策略不生效
            if (!lock.tryLock()) {
                log.debug("t1获取锁失败，立即返回false");
                return;
            }
            try {
                log.debug("t1获得了锁");
            } finally {
                lock.unlock();
            }

        }, "t1");


        lock.lock();
        try {
            log.debug("main线程获得了锁");
            t1.start();
            //先让线程t1执行
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        } finally {
            lock.unlock();
        }

    }
}

```

  

**超时失败**

```java
@Slf4j
public class ReentrantLockDemo4 {

    public static void main(String[] args) {
        ReentrantLock lock = new ReentrantLock();
        Thread t1 = new Thread(() -> {
            log.debug("t1启动...");
            //超时
            try {
                if (!lock.tryLock(1, TimeUnit.SECONDS)) {
                    log.debug("等待 1s 后获取锁失败，返回");
                    return;
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
                return;
            }
            try {
                log.debug("t1获得了锁");
            } finally {
                lock.unlock();
            }

        }, "t1");


        lock.lock();
        try {
            log.debug("main线程获得了锁");
            t1.start();
            //先让线程t1执行
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        } finally {
            lock.unlock();
        }

    }
}
```

​      

### **公平锁**

ReentrantLock 默认是不公平的

```java
@Slf4j
public class ReentrantLockDemo5 {

    public static void main(String[] args) throws InterruptedException {
        ReentrantLock lock = new ReentrantLock(true); //公平锁 

        for (int i = 0; i < 500; i++) {
            new Thread(() -> {
                lock.lock();
                try {
                    try {
                        Thread.sleep(10);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    log.debug(Thread.currentThread().getName() + " running...");
                } finally {
                    lock.unlock();
                }
            }, "t" + i).start();
        }
        // 1s 之后去争抢锁
        Thread.sleep(1000);

        for (int i = 0; i < 500; i++) {
            new Thread(() -> {
                lock.lock();
                try {
                    log.debug(Thread.currentThread().getName() + " running...");
                } finally {
                    lock.unlock();
                }
            }, "强行插入" + i).start();
        }
    }
}
```

​    

思考：ReentrantLock公平锁和非公平锁的性能谁更高？

### **条件变量**

 java.util.concurrent类库中提供Condition类来实现线程之间的协调。调用Condition.await() 方法使线程等待，其他线程调用Condition.signal() 或 Condition.signalAll() 方法唤醒等待的线程。

注意：调用Condition的await()和signal()方法，都必须在lock保护之内。

```java
@Slf4j
public class ReentrantLockDemo6 {
    private static ReentrantLock lock = new ReentrantLock();
    private static Condition cigCon = lock.newCondition();
    private static Condition takeCon = lock.newCondition();

    private static boolean hashcig = false;
    private static boolean hastakeout = false;

    //送烟
    public void cigratee(){
        lock.lock();
        try {
            while(!hashcig){
                try {
                    log.debug("没有烟，歇一会");
                    cigCon.await();

                }catch (Exception e){
                    e.printStackTrace();
                }
            }
            log.debug("有烟了，干活");
        }finally {
            lock.unlock();
        }
    }

    //送外卖
    public void takeout(){
        lock.lock();
        try {
            while(!hastakeout){
                try {
                    log.debug("没有饭，歇一会");
                    takeCon.await();

                }catch (Exception e){
                    e.printStackTrace();
                }
            }
            log.debug("有饭了，干活");
        }finally {
            lock.unlock();
        }
    }

    public static void main(String[] args) {
        ReentrantLockDemo6 test = new ReentrantLockDemo6();
        new Thread(() ->{
            test.cigratee();
        }).start();

        new Thread(() -> {
            test.takeout();
        }).start();

        new Thread(() ->{
            lock.lock();
            try {
                hashcig = true;
                //唤醒送烟的等待线程
                cigCon.signal();
            }finally {
                lock.unlock();
            }


        },"t1").start();

        new Thread(() ->{
            lock.lock();
            try {
                hastakeout = true;
                //唤醒送饭的等待线程
                takeCon.signal();
            }finally {
                lock.unlock();
            }
        },"t2").start();
    }
}
```

​          

## **ReentrantLock源码分析**

关注点： 

1. <font color='red'>ReentrantLock加锁解锁的逻辑</font>
2.  <font color='red'>公平和非公平，可重入锁的实现</font>
3.  <font color='red'>线程竞争锁失败入队阻塞逻辑和获取锁的线程释放锁唤醒阻塞线程竞争锁的逻辑实现 （ 设计的精髓：并发场景下入队和出队操作）</font>

https://www.processon.com/view/link/6191f070079129330ada1209

​    ![0](9%E3%80%81%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3AQS%E4%B9%8B%E7%8B%AC%E5%8D%A0%E9%94%81ReentrantLock%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/1802.png)

![image-20231112134518771](9%E3%80%81%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3AQS%E4%B9%8B%E7%8B%AC%E5%8D%A0%E9%94%81ReentrantLock%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/image-20231112134518771.png)
