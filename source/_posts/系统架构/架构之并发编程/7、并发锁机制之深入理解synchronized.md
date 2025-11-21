---
title: 7、并发锁机制之深入理解synchronized
date: 2023-06-16 23:36:15
categories:
    - 系统架构
    - 架构之并发编程
tags:
    - 并发编程
    - JUC
---

**=========synchronized基础篇=========**

# **Java共享内存模型带来的线程安全问题**

思考： 两个线程对初始值为 0 的静态变量一个做自增，一个做自减，各做 5000 次，结果是 0 吗？

```java
public class SyncDemo {
   
    private static int counter = 0;

    public static void increment() {
        counter++;
    }

    public static void decrement() {
        counter--;
    }

    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread(() -> {
            for (int i = 0; i < 5000; i++) {
                increment();
            }
        }, "t1");
        Thread t2 = new Thread(() -> {
            for (int i = 0; i < 5000; i++) {
                decrement();
            }
        }, "t2");
        t1.start();
        t2.start();
        t1.join();
        t2.join();

        //思考： counter=？
        log.info("{}", counter);
    }
}
```

## **问题分析**

以上的结果可能是正数、负数、零。为什么呢？因为 Java 中对静态变量的自增，自减并**不是原子操作**。

我们可以查看 i++和 i--（i 为静态变量）的 JVM 字节码指令 （ **可以在idea中安装一个jclasslib插件）**

### **i++的JVM 字节码指令**

```
getstatic i // 获取静态变量i的值 
iconst_1 // 将int常量1压入操作数栈
iadd // 自增 
```

### **i--的JVM 字节码指令**

```
getstatic i // 获取静态变量i的值 
iconst_1 // 将int常量1压入操作数栈
isub // 自减 
```

如果是单线程以上 8 行代码是顺序执行（不会交错）没有问题。

但多线程下这 8 行代码可能交错运行：

​    ![0](7%E3%80%81%E5%B9%B6%E5%8F%91%E9%94%81%E6%9C%BA%E5%88%B6%E4%B9%8B%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3synchronized/1706)

## **临界区（ Critical Section）**

-  一个程序运行多个线程本身是没有问题的
-  问题出在多个线程访问共享资源 
  -  多个线程读共享资源其实也没有问题 
  -  <font color='red'>在多个线程对共享资源读写操作时发生**指令交错**，就会出现问题</font> 

<font color='red'>一段代码块内如果存在对共享资源的多线程读写操作，称这段代码块为临界区，其共享资源为临界资源</font>

```java
//临界资源
private static int counter = 0;

public static void increment() { //临界区
    counter++;
}

public static void decrement() {//临界区
    counter--;
}
```

​         

## **竞态条件（ Race Condition ）**

<font color='red'>多个线程在临界区内执行，由于代码的执行序列不同而导致结果无法预测，称之为发生了竞态条件</font>

为了避免临界区的竞态条件发生，有多种手段可以达到目的：

- 阻塞式的解决方案：synchronized，Lock 
- 非阻塞式的解决方案：原子变量

**注意：**

虽然 java 中互斥和同步都可以采用 synchronized 关键字来完成，但它们还是有区别的： 

互斥是保证临界区的竞态条件发生，同一时刻只能有一个线程执行临界区代码 

同步是由于线程执行的先后、顺序不同、需要一个线程等待其它线程运行到某个点

# **synchronized的使用**

<font color='red'>synchronized 同步块是 Java 提供的一种原子性内置锁</font>，Java 中的每个对象都可以把它当作一个同步锁来使用，这些 Java 内置的使用者看不到的锁被称为内置锁，也叫作监视器锁。

## **加锁方式**

​    ![0](7%E3%80%81%E5%B9%B6%E5%8F%91%E9%94%81%E6%9C%BA%E5%88%B6%E4%B9%8B%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3synchronized/1695)

## **解决之前的共享问题**

方式一

```java
public static synchronized void increment() {
    counter++;
}

public static synchronized void decrement() {
    counter--;
}
```

方式二

```java
private static String lock = "";

public static void increment() {
    synchronized (lock){
        counter++;
    }
}

public static void decrement() {
    synchronized (lock) {
        counter--;
    }
}
```

synchronized 实际是用对象锁保证了临界区内代码的原子性

​    ![0](7%E3%80%81%E5%B9%B6%E5%8F%91%E9%94%81%E6%9C%BA%E5%88%B6%E4%B9%8B%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3synchronized/1707)

**=========synchronized高级篇=========**

# **synchronized底层原理**

<font color='red'>synchronized是JVM内置锁</font>，基于<font color='red'>**Monitor**</font>机制实现，依赖底层操作系统的互斥原语<font color='red'>Mutex</font>（互斥量），它是一个重量级锁，性能较低。当然，<font color='red'>JVM内置锁在1.5之后版本做了重大的优化</font>，如锁粗化（Lock Coarsening）、锁消除（Lock Elimination）、轻量级锁（Lightweight Locking）、偏向锁（Biased Locking）、自适应自旋（Adaptive Spinning）等技术来减少锁操作的开销，内置锁的并发性能已经基本与Lock持平。

> The Java® Language Specification
>
> Each object is associated with a monitor (§17.1), which is used by synchronized methods (§8.4.3) and the synchronized statement (§14.19) to provide control over concurrent access to state by multiple threads (§17 (Threads and Locks)).
>
> The Java® Virtual Machine Specification
>
> The Java Virtual Machine supports synchronization of both methods and sequences of instructions within a method by a single synchronization construct: the monitor.

<font color='red'>Java虚拟机通过一个同步结构支持方法和方法中的指令序列的同步：monitor。</font>

同步方法是通过方法中的access_flags中设置<font color='red'>ACC_SYNCHRONIZED</font>标志来实现；同步代码块是通过<font color='red'>monitorenter和monitorexit</font>来实现。<font color='red'>两个指令的执行是JVM通过调用操作系统的互斥原语mutex来实现，被阻塞的线程会被挂起、等待重新调度，会导致“用户态和内核态”两个态之间来回切换，对性能有较大影响。</font>

## **查看synchronized的字节码指令序列**

​    ![0](7%E3%80%81%E5%B9%B6%E5%8F%91%E9%94%81%E6%9C%BA%E5%88%B6%E4%B9%8B%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3synchronized/1691)

Method access and property flags：

​    ![0](7%E3%80%81%E5%B9%B6%E5%8F%91%E9%94%81%E6%9C%BA%E5%88%B6%E4%B9%8B%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3synchronized/1698)

​    ![0](7%E3%80%81%E5%B9%B6%E5%8F%91%E9%94%81%E6%9C%BA%E5%88%B6%E4%B9%8B%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3synchronized/1683)

## **Monitor（管程/监视器）**

Monitor，直译为“监视器”，而操作系统领域一般翻译为“管程”。<font color="red">管程是指管理共享变量以及对共享变量操作的过程，让它们支持并发。</font>在Java 1.5之前，Java语言提供的唯一并发语言就是管程，Java 1.5之后提供的SDK并发包也是以管程为基础的。除了Java之外，C/C++、C#等高级语言也都是支持管程的。synchronized关键字和wait()、notify()、notifyAll()这三个方法是Java中实现管程技术的组成部分。

### **MESA模型**

在管程的发展史上，先后出现过三种不同的管程模型，分别是Hasen模型、Hoare模型和MESA模型。现在正在广泛使用的是MESA模型。下面我们便介绍MESA模型：

​    ![](7%E3%80%81%E5%B9%B6%E5%8F%91%E9%94%81%E6%9C%BA%E5%88%B6%E4%B9%8B%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3synchronized/1701)

管程中引入了条件变量的概念，而且每个条件变量都对应有一个等待队列。条件变量和等待队列的作用是解决线程之间的同步问题。

**wait()的正确使用姿势**

对于MESA管程来说，有一个编程范式：

```
while(条件不满足) {  
	wait();   
}      
```

唤醒的时间和获取到锁继续执行的时间是不一致的，被唤醒的线程再次执行时可能条件又不满足了，所以循环检验条件。MESA模型的wait()方法还有一个超时参数，为了避免线程进入等待队列永久阻塞。

**notify()和notifyAll()分别何时使用**

满足以下三个条件时，可以使用notify()，其余情况尽量使用notifyAll()：

1. 所有等待线程拥有相同的等待条件；
2. 所有等待线程被唤醒后，执行相同的操作；
3. 只需要唤醒一个线程。

### **Java语言的内置管程synchronized**

Java 参考了 MESA 模型，语言内置的管程（synchronized）对 MESA 模型进行了精简。MESA 模型中，条件变量可以有多个，Java 语言内置的管程里只有一个条件变量。模型如下图所示。

​    ![0](7%E3%80%81%E5%B9%B6%E5%8F%91%E9%94%81%E6%9C%BA%E5%88%B6%E4%B9%8B%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3synchronized/1702)

### **Monitor机制在Java中的实现**

java.lang.Object 类定义了 wait()，notify()，notifyAll() 方法，这些方法的具体实现，依赖于 ObjectMonitor 实现，这是 JVM 内部基于 C++ 实现的一套机制。

ObjectMonitor其主要数据结构如下（hotspot源码ObjectMonitor.hpp）：

```java
ObjectMonitor() {
    _header       = NULL; //对象头  markOop
    _count        = 0;  
    _waiters      = 0,   
    _recursions   = 0;   // 锁的重入次数 
    _object       = NULL;  //存储锁对象
    _owner        = NULL;  // 标识拥有该monitor的线程（当前获取锁的线程） 
    _WaitSet      = NULL;  // 等待线程（调用wait）组成的双向循环链表，_WaitSet是第一个节点
    _WaitSetLock  = 0 ;    
    _Responsible  = NULL ;
    _succ         = NULL ;
    _cxq          = NULL ; //多线程竞争锁会先存到这个单向链表中 （FILO栈结构）
    FreeNext      = NULL ;
    _EntryList    = NULL ; //存放在进入或重新进入时被阻塞(blocked)的线程 (也是存竞争锁失败的线程)
    _SpinFreq     = 0 ;
    _SpinClock    = 0 ;
    OwnerIsThread = 0 ;
    _previous_owner_tid = 0;
}
```

​           

​    ![0](7%E3%80%81%E5%B9%B6%E5%8F%91%E9%94%81%E6%9C%BA%E5%88%B6%E4%B9%8B%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3synchronized/1705)

<font color='red'>在获取锁时，是将当前线程插入到cxq的头部，而释放锁时，默认策略（QMode=0）是：如果EntryList为空，则将cxq中的元素按原有顺序插入到EntryList，并唤醒第一个线程，也就是当EntryList为空时，是后来的线程先获取锁。_EntryList不为空，直接从_EntryList中唤醒线程。</font>

思考：<font color='red'>synchronized加锁加在对象上，锁对象是如何记录锁状态的？</font>

## **对象的内存布局**

<font color='red'>Hotspot虚拟机中，对象在内存中存储的布局可以分为三块区域：**对象头（Header）**、**实例数据（Instance Data）**和**对齐填充（Padding）**。</font>

- 对象头：比如 hash码，对象所属的年代，对象锁，锁状态标志，偏向锁（线程）ID，偏向时间，数组长度（数组对象才有）等。
- 实例数据：存放类的属性数据信息，包括父类的属性信息；
- 对齐填充：由于虚拟机要求 <font color='red'>**对象起始地址必须是8字节的整数倍**</font>。填充数据不是必须存在的，仅仅是为了字节对齐。

​    ![0](7%E3%80%81%E5%B9%B6%E5%8F%91%E9%94%81%E6%9C%BA%E5%88%B6%E4%B9%8B%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3synchronized/1704)

### **对象头详解**

HotSpot虚拟机的对象头包括：

- Mark Word 

用于存储对象自身的运行时数据，如哈希码（HashCode）、GC分代年龄、锁状态标志、线程持有的锁、偏向线程ID、偏向时间戳等，这部分数据的长度在32位和64位的虚拟机中分别为32bit和64bit，官方称它为“Mark Word”。

-  Klass Pointer

对象头的另外一部分是klass类型指针，即对象指向它的类元数据的指针，虚拟机通过这个指针来确定这个对象是哪个类的实例。 32位4字节，64位开启指针压缩或最大堆内存<32g时4字节，否则8字节。**jdk1.8默认开启指针压缩后为4字节，当在JVM参数中关闭指针压缩（-XX:-UseCompressedOops）后，长度为8字节。**

- 数组长度（只有数组对象有）

如果对象是一个数组, 那在对象头中还必须有一块数据用于记录数组长度。 4字节

​    ![0](7%E3%80%81%E5%B9%B6%E5%8F%91%E9%94%81%E6%9C%BA%E5%88%B6%E4%B9%8B%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3synchronized/1703)

## **使用JOL工具查看内存布局**

给大家推荐一个可以查看普通java对象的内部布局工具JOL(JAVA OBJECT LAYOUT)，使用此工具可以查看new出来的一个java对象的内部布局,以及一个普通的java对象占用多少字节。

引入maven依赖

```xml
<!-- 查看Java 对象布局、大小工具 --> 
<dependency>    
    <groupId>org.openjdk.jol</groupId>    
    <artifactId>jol-core</artifactId>    
    <version>0.10</version>    
</dependency> 
```

​          

 使用方法

```
//查看对象内部信息   
ClassLayout.parseInstance(obj).toPrintable()
```

 

**测试**

```java
public static void main(String[] args) throws InterruptedException {
    Object obj = new Object();
    //查看对象内部信息
    System.out.println(ClassLayout.parseInstance(obj).toPrintable());
}
```

1. 利用jol查看64位系统java对象（空对象），默认开启指针压缩，总大小显示16字节，前12字节为对象头

​    ![0](7%E3%80%81%E5%B9%B6%E5%8F%91%E9%94%81%E6%9C%BA%E5%88%B6%E4%B9%8B%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3synchronized/1694)

- OFFSET：偏移地址，单位字节；
- SIZE：占用的内存大小，单位为字节；
- TYPE DESCRIPTION：类型描述，其中object header为对象头；
- VALUE：对应内存中当前存储的值，二进制32位；

2. 关闭指针压缩后，对象头为16字节：-XX:-UseCompressedOops

​    ![0](7%E3%80%81%E5%B9%B6%E5%8F%91%E9%94%81%E6%9C%BA%E5%88%B6%E4%B9%8B%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3synchronized/1688)

思考： 下面例子中obj对象占多少个字节?

```java
public class ObjectTest {
    public static void main(String[] args) throws InterruptedException {
        Object obj = new Test();
        //查看对象内部信息
        System.out.println(ClassLayout.parseInstance(obj).toPrintable());
    }
}
class Test{
    private long p;
}
```

回到之前的问题： synchronized加锁加在对象上，对象是如何记录锁状态的？

锁状态被记录在每个对象的对象头的Mark Word中

## **Mark Word是如何记录锁状态的**

Hotspot通过markOop类型实现Mark Word，具体实现位于markOop.hpp文件中。由于对象需要存储的运行时数据很多，考虑到虚拟机的内存使用，markOop被设计成一个非固定的数据结构，以便在极小的空间存储尽量多的数据，根据对象的状态复用自己的存储空间。

简单点理解就是：MarkWord 结构搞得这么复杂，是因为需要节省内存，让同一个内存区域在不同阶段有不同的用处。

### **Mark Word的结构**

```
//  32 bits:
//  --------
//             hash:25 ------------>| age:4    biased_lock:1 lock:2 (normal object)
//             JavaThread*:23 epoch:2 age:4    biased_lock:1 lock:2 (biased object)
//             size:32 ------------------------------------------>| (CMS free block)
//             PromotedObject*:29 ---------->| promo_bits:3 ----->| (CMS promoted object)
//
//  64 bits:
//  --------
//  unused:25 hash:31 -->| unused:1   age:4    biased_lock:1 lock:2 (normal object)
//  JavaThread*:54 epoch:2 unused:1   age:4    biased_lock:1 lock:2 (biased object)
//  PromotedObject*:61 --------------------->| promo_bits:3 ----->| (CMS promoted object)
//  size:64 ----------------------------------------------------->| (CMS free block)
//
//  unused:25 hash:31 -->| cms_free:1 age:4    biased_lock:1 lock:2 (COOPs && normal object)
//  JavaThread*:54 epoch:2 cms_free:1 age:4    biased_lock:1 lock:2 (COOPs && biased object)
//  narrowOop:32 unused:24 cms_free:1 unused:4 promo_bits:3 ----->| (COOPs && CMS promoted object)
//  unused:21 size:35 -->| cms_free:1 unused:7 ------------------>| (COOPs && CMS free block)

。。。。。。
//    [JavaThread* | epoch | age | 1 | 01]       lock is biased toward given thread
//    [0           | epoch | age | 1 | 01]       lock is anonymously biased
//
//  - the two lock bits are used to describe three states: locked/unlocked and monitor.
//
//    [ptr             | 00]  locked             ptr points to real header on stack
//    [header      | 0 | 01]  unlocked           regular object header
//    [ptr             | 10]  monitor            inflated lock (header is wapped out)
//    [ptr             | 11]  marked             used by markSweep to mark an object
```

- **hash**： 保存对象的哈希码。运行期间调用System.identityHashCode()来计算，延迟计算，并把结果赋值到这里。
- **age**： 保存对象的分代年龄。表示对象被GC的次数，当该次数到达阈值的时候，对象就会转移到老年代。
- **biased_lock**： 偏向锁标识位。由于无锁和偏向锁的锁标识都是 01，没办法区分，这里引入一位的偏向锁标识位。
- **lock**： 锁状态标识位。区分锁状态，比如11时表示对象待GC回收状态, 只有最后2位锁标识(11)有效。
- **JavaThread***： 保存持有偏向锁的线程ID。偏向模式的时候，当某个线程持有对象的时候，对象这里就会被置为该线程的ID。 在后面的操作中，就无需再进行尝试获取锁的动作。这个线程ID并不是JVM分配的线程ID号，和Java Thread中的ID是两个概念。
- **epoch**： 保存偏向时间戳。偏向锁在CAS锁操作过程中，偏向性标识，表示对象更偏向哪个锁。

**32位JVM下的对象结构描述**

​    ![0](7%E3%80%81%E5%B9%B6%E5%8F%91%E9%94%81%E6%9C%BA%E5%88%B6%E4%B9%8B%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3synchronized/1697)

**64位JVM下的对象结构描述**

​    ![0](7%E3%80%81%E5%B9%B6%E5%8F%91%E9%94%81%E6%9C%BA%E5%88%B6%E4%B9%8B%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3synchronized/1699)

- ptr_to_lock_record：轻量级锁状态下，指向栈中锁记录的指针。当锁获取是无竞争时，JVM使用原子操作而不是OS互斥，这种技术称为轻量级锁定。在轻量级锁定的情况下，JVM通过CAS操作在对象的Mark Word中设置指向锁记录的指针。
- ptr_to_heavyweight_monitor：重量级锁状态下，指向对象监视器Monitor的指针。如果两个不同的线程同时在同一个对象上竞争，则必须将轻量级锁定升级到Monitor以管理等待的线程。在重量级锁定的情况下，JVM在对象的ptr_to_heavyweight_monitor设置指向Monitor的指针

### **Mark Word中锁标记枚举**

```java
enum { locked_value             = 0,    //00 轻量级锁 
         unlocked_value           = 1,   //001 无锁
         monitor_value            = 2,   //10 监视器锁，也叫膨胀锁，也叫重量级锁
         marked_value             = 3,   //11 GC标记
         biased_lock_pattern      = 5    //101 偏向锁
     }
```

​          

更直观的理解方式：

​    ![0](7%E3%80%81%E5%B9%B6%E5%8F%91%E9%94%81%E6%9C%BA%E5%88%B6%E4%B9%8B%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3synchronized/1708)

# **测试：利用JOL工具跟踪锁标记变化**

## **偏向锁**

​	偏向锁是一种针对加锁操作的优化手段，经过研究发现，<font color='red'>在大多数情况下，锁不仅不存在多线程竞争，而且总是由同一线程多次获得，因此为了消除数据在无竞争情况下锁重入（CAS操作）的开销而引入偏向锁。对于没有锁竞争的场合，偏向锁有很好的优化效果。</font>

```java
/***StringBuffer内部同步***/
public synchronized int length() { 
   return count; 
} 
//System.out.println 无意识的使用锁 
public void println(String x) { 
  synchronized (this) {
     print(x); newLine(); 
  } 
}
```

​	当JVM启用了偏向锁模式（<font color='red'>jdk6默认开启</font>），新创建对象的Mark Word中的Thread Id为0，说明此时<font color='red'>处于可偏向但未偏向任何线程</font>，也叫做<font color='red'>匿名偏向状态(anonymously biased)</font>。

### **偏向锁延迟偏向**

<font color='red'>	偏向锁模式存在偏向锁延迟机制</font>：HotSpot 虚拟机在启动后有个 4s 的延迟才会对每个新建的对象开启偏向锁模式。JVM启动时会进行一系列的复杂活动，比如装载配置，系统类初始化等等。在这个过程中会使用大量synchronized关键字对对象加锁，且这些锁大多数都不是偏向锁。为了减少初始化时间，JVM默认延时加载偏向锁。

```
//关闭延迟开启偏向锁
-XX:BiasedLockingStartupDelay=0
//禁止偏向锁
-XX:-UseBiasedLocking 
//启用偏向锁
```

验证

```java
@Slf4j
public class LockEscalationDemo{

    public static void main(String[] args) throws InterruptedException {
        log.debug(ClassLayout.parseInstance(new Object()).toPrintable());
        Thread.sleep(4000);
        log.debug(ClassLayout.parseInstance(new Object()).toPrintable());
    }
}
```

4s后偏向锁为可偏向或者匿名偏向状态：

​    ![0](7%E3%80%81%E5%B9%B6%E5%8F%91%E9%94%81%E6%9C%BA%E5%88%B6%E4%B9%8B%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3synchronized/1690)

思考：如果锁LockEscalationDemo.class会是什么状态？

### **偏向锁状态跟踪**

```java
public class LockEscalationDemo {
    public static void main(String[] args) throws InterruptedException {
        log.debug(ClassLayout.parseInstance(new Object()).toPrintable());
        //HotSpot 虚拟机在启动后有个 4s 的延迟才会对每个新建的对象开启偏向锁模式
        Thread.sleep(4000);
        Object obj = new Object();

        new Thread(new Runnable() {
            @Override
            public void run() {
                log.debug(Thread.currentThread().getName()+"开始执行。。。\n"
                        +ClassLayout.parseInstance(obj).toPrintable());
                synchronized (obj){
                    log.debug(Thread.currentThread().getName()+"获取锁执行中。。。\n"
                            +ClassLayout.parseInstance(obj).toPrintable());
                }
                log.debug(Thread.currentThread().getName()+"释放锁。。。\n"
                        +ClassLayout.parseInstance(obj).toPrintable());
            }
        },"thread1").start();
        
        Thread.sleep(5000);
        log.debug(ClassLayout.parseInstance(obj).toPrintable());
   }
}
```

![0](7%E3%80%81%E5%B9%B6%E5%8F%91%E9%94%81%E6%9C%BA%E5%88%B6%E4%B9%8B%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3synchronized/1681)

 思考：如果对象调用了hashCode,还会开启偏向锁模式吗？

![0](7%E3%80%81%E5%B9%B6%E5%8F%91%E9%94%81%E6%9C%BA%E5%88%B6%E4%B9%8B%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3synchronized/1678)

### **偏向锁撤销之调用对象HashCode**

调用锁对象的obj.hashCode()或System.identityHashCode(obj)方法会导致该对象的偏向锁被撤销。

因为对于一个对象，其HashCode只会生成一次并保存，偏向锁是没有地方保存hashcode的。

当对象处于可偏向（也就是线程ID为0）和已偏向的状态下，调用HashCode计算将会使对象再也无法偏向：

- <font color='red'>当对象可偏向时，MarkWord将变成未锁定状态，并只能升级成轻量锁；轻量级锁会在锁记录中记录 hashCode </font>
- <font color='red'>当对象正处于偏向锁时，调用HashCode将使偏向锁强制升级成重量锁。重量级锁会在 Monitor 中记录 hashCode</font>

​    ![0](7%E3%80%81%E5%B9%B6%E5%8F%91%E9%94%81%E6%9C%BA%E5%88%B6%E4%B9%8B%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3synchronized/1680)

### **偏向锁撤销之调用wait/notify**

 偏向锁状态执行obj.notify() 会升级为轻量级锁，调用obj.wait(timeout) 会升级为重量级锁

```java
synchronized (obj) {
    // 思考：偏向锁执行过程中，调用hashcode会发生什么？
    //obj.hashCode();
    //obj.notify();
    try {
        obj.wait(100);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
   
    log.debug(Thread.currentThread().getName() + "获取锁执行中。。。\n"
            + ClassLayout.parseInstance(obj).toPrintable());
}
```

​      

测试结果：

​    ![0](7%E3%80%81%E5%B9%B6%E5%8F%91%E9%94%81%E6%9C%BA%E5%88%B6%E4%B9%8B%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3synchronized/1684)

​    ![0](7%E3%80%81%E5%B9%B6%E5%8F%91%E9%94%81%E6%9C%BA%E5%88%B6%E4%B9%8B%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3synchronized/1686)

## **轻量级锁**

倘若偏向锁失败，虚拟机并不会立即升级为重量级锁，它还会尝试使用一种称为轻量级锁的优化手段，此时Mark Word 的结构也变为轻量级锁的结构。<font color="red">轻量级锁所适应的场景是线程交替执行同步块的场合</font>，如果存在同一时间多个线程访问同一把锁的场合，就会导致**轻量级锁膨胀为重量级锁**。

### **轻量级锁跟踪**

```java
public class LockEscalationDemo {
    public static void main(String[] args) throws InterruptedException {

        logger.debug(ClassLayout.parseInstance(new Object()).toPrintable());
        //HotSpot 虚拟机在启动后有个 4s 的延迟才会对每个新建的对象开启偏向锁模式
        Thread.sleep(4000);
        Object obj = new Object();
        // 思考： 如果对象调用了hashCode,还会开启偏向锁模式吗
        obj.hashCode();
       //logger.debug(ClassLayout.parseInstance(obj).toPrintable());

        new Thread(new Runnable() {
            @Override
            public void run() {
                logger.debug(Thread.currentThread().getName()+"开始执行。。。\n"
                        +ClassLayout.parseInstance(obj).toPrintable());
                synchronized (obj){
                    logger.debug(Thread.currentThread().getName()+"获取锁执行中。。。\n"
                            +ClassLayout.parseInstance(obj).toPrintable());
                }
                logger.debug(Thread.currentThread().getName()+"释放锁。。。\n"
                        +ClassLayout.parseInstance(obj).toPrintable());
            }
        },"thread1").start();
        
        Thread.sleep(5000);
        logger.debug(ClassLayout.parseInstance(obj).toPrintable());
   }
}
```

思考： 轻量级锁是否可以降级为偏向锁？

   ![0](7%E3%80%81%E5%B9%B6%E5%8F%91%E9%94%81%E6%9C%BA%E5%88%B6%E4%B9%8B%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3synchronized/1682)

## **测试：锁升级场景**

### **偏向锁升级轻量级锁**

模拟两个线程轻微竞争场景

```java
@Slf4j
public class LockEscalationDemo {

    public static void main(String[] args) throws InterruptedException {
        logger.debug(ClassLayout.parseInstance(new Object()).toPrintable());
        //HotSpot 虚拟机在启动后有个 4s 的延迟才会对每个新建的对象开启偏向锁模式
        Thread.sleep(4000);
        Object obj = new Object();
        // 思考： 如果对象调用了hashCode,还会开启偏向锁模式吗
        //obj.hashCode();
        //logger.debug(ClassLayout.parseInstance(obj).toPrintable());

        Thread thread1 = new Thread(new Runnable() {
            @Override
            public void run() {
                logger.debug(Thread.currentThread().getName() + "开始执行。。。\n"
                        + ClassLayout.parseInstance(obj).toPrintable());
                synchronized (obj) {
                    // 思考：偏向锁执行过程中，调用hashcode会发生什么？
                    //obj.hashCode();
                    logger.debug(Thread.currentThread().getName() + "获取锁执行中。。。\n"
                            + ClassLayout.parseInstance(obj).toPrintable());

                }
                logger.debug(Thread.currentThread().getName() + "释放锁。。。\n"
                        + ClassLayout.parseInstance(obj).toPrintable());
            }
        }, "thread1");
        thread1.start();
        
        //控制线程竞争时机
        Thread.sleep(1);

        Thread thread2 = new Thread(new Runnable() {
            @Override
            public void run() {
                logger.debug(Thread.currentThread().getName()+"开始执行。。。\n"
                        +ClassLayout.parseInstance(obj).toPrintable());
                synchronized (obj){
                    logger.debug(Thread.currentThread().getName()+"获取锁执行中。。。\n"
                            +ClassLayout.parseInstance(obj).toPrintable());
                }
                logger.debug(Thread.currentThread().getName()+"释放锁。。。\n"
                        +ClassLayout.parseInstance(obj).toPrintable());
            }
        },"thread2");
        thread2.start();

        Thread.sleep(5000);
        logger.debug(ClassLayout.parseInstance(obj).toPrintable());

    }
}
```

![0](7%E3%80%81%E5%B9%B6%E5%8F%91%E9%94%81%E6%9C%BA%E5%88%B6%E4%B9%8B%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3synchronized/1685)

### **轻量级锁膨胀为重量级锁**

```java
@Slf4j
public class LockEscalationDemo {

    public static void main(String[] args) throws InterruptedException {

        logger.debug(ClassLayout.parseInstance(new Object()).toPrintable());
        //HotSpot 虚拟机在启动后有个 4s 的延迟才会对每个新建的对象开启偏向锁模式
        Thread.sleep(5000);
        Object obj = new Object();

        new Thread(new Runnable() {
            @Override
            public void run() {
                logger.debug(Thread.currentThread().getName()+"开始执行。。。\n"
                        +ClassLayout.parseInstance(obj).toPrintable());
                synchronized (obj){
                    logger.debug(Thread.currentThread().getName()+"获取锁执行中。。。\n"
                            +ClassLayout.parseInstance(obj).toPrintable());
                }
                logger.debug(Thread.currentThread().getName()+"释放锁。。。\n"
                        +ClassLayout.parseInstance(obj).toPrintable());
            }
        },"thread1").start();

        new Thread(new Runnable() {
            @Override
            public void run() {
                logger.debug(Thread.currentThread().getName()+"开始执行。。。\n"
                        +ClassLayout.parseInstance(obj).toPrintable());
                synchronized (obj){
                    logger.debug(Thread.currentThread().getName()+"获取锁执行中。。。\n"
                            +ClassLayout.parseInstance(obj).toPrintable());
                }
                logger.debug(Thread.currentThread().getName()+"释放锁。。。\n"
                        +ClassLayout.parseInstance(obj).toPrintable());
            }
        },"thread2").start();

        Thread.sleep(5000);
        logger.debug(ClassLayout.parseInstance(obj).toPrintable());
    }
}
```

​    ![0](7%E3%80%81%E5%B9%B6%E5%8F%91%E9%94%81%E6%9C%BA%E5%88%B6%E4%B9%8B%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3synchronized/1696)

思考：重量级锁释放之后变为无锁，此时有新的线程来调用同步块，会获取什么锁？

### **总结：锁对象状态转换**

​    ![0](7%E3%80%81%E5%B9%B6%E5%8F%91%E9%94%81%E6%9C%BA%E5%88%B6%E4%B9%8B%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3synchronized/1700)

![锁升级流程图](7%E3%80%81%E5%B9%B6%E5%8F%91%E9%94%81%E6%9C%BA%E5%88%B6%E4%B9%8B%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3synchronized/%E9%94%81%E5%8D%87%E7%BA%A7%E6%B5%81%E7%A8%8B%E5%9B%BE%5B666%E8%B5%84%E6%BA%90%E7%AB%99%EF%BC%9A666java%20.com%5D.png)

## **锁升级的原理分析**

会结合Hotspot源码重点分析

