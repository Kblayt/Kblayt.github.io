---
title: 8、并发锁机制之深入理解synchronize二
date: 2023-06-16 23:36:43
categories:
    - 系统架构
    - 架构之并发编程
tags:
    - 并发编程
    - JUC
---

# **进阶：synchronized锁优化**

## **偏向锁批量重偏向&批量撤销**

​	从偏向锁的加锁解锁过程中可看出，当只有一个线程反复进入同步块时，偏向锁带来的性能开销基本可以忽略，但是当有其他线程尝试获得锁时，就需要等到safe point时，再将偏向锁撤销为无锁状态或升级为轻量级，会消耗一定的性能，所以<font color='red'>在多线程竞争频繁的情况下，偏向锁不仅不能提高性能，还会导致性能下降</font>。于是，就有了批量重偏向与批量撤销的机制。

### **原理**

​	以class为单位，为每个class维护一个偏向锁撤销计数器，每一次该class的对象发生偏向撤销操作时，该计数器+1，当这个值达到重偏向阈值（默认20）时，JVM就认为该class的偏向锁有问题，因此会进行批量重偏向。

​	每个class对象会有一个对应的epoch字段，每个处于偏向锁状态对象的Mark Word中也有该字段，其初始值为创建该对象时class中的epoch的值。每次发生批量重偏向时，就将该值+1，同时遍历JVM中所有线程的栈，找到该class所有正处于加锁状态的偏向锁，将其epoch字段改为新值。下次获得锁时，发现当前对象的epoch值和class的epoch不相等，那就算当前已经偏向了其他线程，也不会执行撤销操作，而是直接通过CAS操作将其Mark Word的Thread Id 改成当前线程Id。

​	当达到重偏向阈值（默认20）后，假设该class计数器继续增长，当其达到批量撤销的阈值后（默认40），JVM就认为该class的使用场景存在多线程竞争，会标记该class为不可偏向，之后，对于该class的锁，直接走轻量级锁的逻辑。

### **应用场景**

**批量重偏向（bulk rebias）机制**是为了解决：<font color='red'>一个线程创建了大量对象并执行了初始的同步操作，后来另一个线程也来将这些对象作为锁对象进行操作，这样会导致大量的偏向锁撤销操作。 </font>

**批量撤销（bulk revoke）机制**是为了解决：<font color='red'>在明显多线程竞争剧烈的场景下使用偏向锁是不合适的。</font>

**JVM的默认参数值**

设置JVM参数-XX:+PrintFlagsFinal，在项目启动时即可输出JVM的默认参数值

```java
intx BiasedLockingBulkRebiasThreshold  = 20  //默认偏向锁批量重偏向阈值  
intx BiasedLockingBulkRevokeThreshold  = 40  //默认偏向锁批量撤销阈值 
```

我们可以通过-XX:BiasedLockingBulkRebiasThreshold 和 -XX:BiasedLockingBulkRevokeThreshold 来手动设置阈值

### **测试：批量重偏向**

当撤销偏向锁阈值超过 20 次后，jvm 会这样觉得，我是不是偏向错了，于是会在给这些对象加锁时重新偏向至加锁线程，重偏向会重置对象 的 Thread ID

```java
package org.example;

import lombok.extern.slf4j.Slf4j;
import org.openjdk.jol.info.ClassLayout;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.locks.LockSupport;


@Slf4j
public class BiasedLockingTest {
    public static void main(String[] args) throws InterruptedException {

        //延时产生可偏向对象
        Thread.sleep(5000);
        // 创建一个list，来存放锁对象
        List<Object> list = new ArrayList<>();

        // 线程1
        new Thread(() -> {
            for (int i = 0; i < 50; i++) {
                // 新建锁对象
                Object lock = new Object();
                synchronized (lock) {
                    list.add(lock);
                }
            }
            try {
                //为了防止JVM线程复用，在创建完对象后，保持线程thead1状态为存活
                Thread.sleep(100000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }, "thead1").start();

        //睡眠3s钟保证线程thead1创建对象完成
        Thread.sleep(3000);
        log.debug("打印thead1，list中第20个对象的对象头：");
        log.debug((ClassLayout.parseInstance(list.get(19)).toPrintable()));

        // 线程2
        new Thread(() -> {
            for (int i = 0; i < 40; i++) {
                Object obj = list.get(i);
                synchronized (obj) {
                    if(i>=15&&i<=21||i>=38){
                        log.debug("thread2-第" + (i + 1) + "次加锁执行中\t"+
                                ClassLayout.parseInstance(obj).toPrintable());
                    }
                }
                if(i==17||i==19){
                    log.debug("thread2-第" + (i + 1) + "次释放锁\t"+
                            ClassLayout.parseInstance(obj).toPrintable());
                }
            }
            try {
                Thread.sleep(100000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }, "thead2").start();

        LockSupport.park();
    }

}
```

测试结果：

thread1:  创建50个偏向线程thread1的偏向锁     1-50 偏向锁

​    ![0](8%E3%80%81%E5%B9%B6%E5%8F%91%E9%94%81%E6%9C%BA%E5%88%B6%E4%B9%8B%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3synchronized%E4%BA%8C/1687)

thread2：

1-18 偏向锁撤销，升级为轻量级锁  （thread1释放锁之后为偏向锁状态）

19-40 偏向锁撤销达到阈值（20），执行了批量重偏向 （测试结果在第19就开始批量重偏向了）

​    ![0](8%E3%80%81%E5%B9%B6%E5%8F%91%E9%94%81%E6%9C%BA%E5%88%B6%E4%B9%8B%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3synchronized%E4%BA%8C/1676)

### **测试：批量撤销**

当撤销偏向锁阈值超过 40 次后，jvm 会认为不该偏向，于是整个类的所有对象都会变为不可偏向的，新建的对象也是不可偏向的。

**注意**：时间-XX:BiasedLockingDecayTime=25000ms范围内没有达到40次，撤销次数清为0，重新计时

```java
@Slf4j
public class BiasedLockingTest {
    public static void main(String[] args) throws  InterruptedException {
        //延时产生可偏向对象
        Thread.sleep(5000);
        // 创建一个list，来存放锁对象
        List<Object> list = new ArrayList<>();
        
        // 线程1
        new Thread(() -> {
            for (int i = 0; i < 50; i++) {
                // 新建锁对象
                Object lock = new Object();
                synchronized (lock) {
                    list.add(lock);
                }
            }
            try {
                //为了防止JVM线程复用，在创建完对象后，保持线程thead1状态为存活
                Thread.sleep(100000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }, "thead1").start();

        //睡眠3s钟保证线程thead1创建对象完成
        Thread.sleep(3000);
        logger.debug("打印thead1，list中第20个对象的对象头：");
        logger.debug((ClassLayout.parseInstance(list.get(19)).toPrintable()));
        
        // 线程2
        new Thread(() -> {
            for (int i = 0; i < 40; i++) {
                Object obj = list.get(i);
                synchronized (obj) {
                    if(i>=15&&i<=21||i>=38){
                        logger.debug("thread2-第" + (i + 1) + "次加锁执行中\t"+
                                ClassLayout.parseInstance(obj).toPrintable());
                    }
                }
                if(i==17||i==19){
                    logger.debug("thread2-第" + (i + 1) + "次释放锁\t"+
                            ClassLayout.parseInstance(obj).toPrintable());
                }
            }
            try {
                Thread.sleep(100000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }, "thead2").start();


        Thread.sleep(3000);

        new Thread(() -> {
            for (int i = 0; i < 50; i++) {
                Object lock =list.get(i);
                if(i>=17&&i<=21||i>=35&&i<=41){
                    logger.debug("thread3-第" + (i + 1) + "次准备加锁\t"+
                            ClassLayout.parseInstance(lock).toPrintable());
                }
                synchronized (lock){
                    if(i>=17&&i<=21||i>=35&&i<=41){
                        logger.debug("thread3-第" + (i + 1) + "次加锁执行中\t"+
                                ClassLayout.parseInstance(lock).toPrintable());
                    }
                }
            }
        },"thread3").start();

        Thread.sleep(3000);
        logger.debug("查看新创建的对象");
        logger.debug((ClassLayout.parseInstance(new Object()).toPrintable()));

        LockSupport.park();
    }
}
```

测试结果：

thread3：

1-18  从无锁状态直接获取轻量级锁  （thread2释放锁之后变为无锁状态）

​    ![0](8%E3%80%81%E5%B9%B6%E5%8F%91%E9%94%81%E6%9C%BA%E5%88%B6%E4%B9%8B%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3synchronized%E4%BA%8C/1677)

19-40 偏向锁撤销，升级为轻量级锁   （thread2释放锁之后为偏向锁状态）

​    ![0](8%E3%80%81%E5%B9%B6%E5%8F%91%E9%94%81%E6%9C%BA%E5%88%B6%E4%B9%8B%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3synchronized%E4%BA%8C/1675)

41-50   达到偏向锁撤销的阈值40，批量撤销偏向锁，升级为轻量级锁     （thread1释放锁之后为偏向锁状态）

​    ![0](8%E3%80%81%E5%B9%B6%E5%8F%91%E9%94%81%E6%9C%BA%E5%88%B6%E4%B9%8B%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3synchronized%E4%BA%8C/1679)

新创建的对象： 无锁状态

​    ![0](8%E3%80%81%E5%B9%B6%E5%8F%91%E9%94%81%E6%9C%BA%E5%88%B6%E4%B9%8B%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3synchronized%E4%BA%8C/1689)

### **总结**

1. <font color='red'>批量重偏向和批量撤销是针对类的优化，和对象无关。</font>
2. <font color='red'>偏向锁重偏向一次之后不可再次重偏向。</font>
3. <font color='red'>当某个类已经触发批量撤销机制后，JVM会默认当前类产生了严重的问题，剥夺了该类的新实例对象使用偏向锁的权利</font>

## **自旋优化**

重量级锁竞争的时候，还可以使用自旋来进行优化，如果当前线程自旋成功（即这时候持锁线程已经退出了同步块，释放了锁），这时当前线程就可以避免阻塞。

- 自旋会占用 CPU 时间，单核 CPU 自旋就是浪费，多核 CPU 自旋才能发挥优势。 
- 在 Java 6 之后自旋是自适应的，比如对象刚刚的一次自旋操作成功过，那么认为这次自旋成功的可能性会高，就多自旋几次；反之，就少自旋甚至不自旋，比较智能。 
- Java 7 之后不能控制是否开启自旋功能

注意：<font color='red'>自旋的目的是为了减少线程挂起的次数，尽量避免直接挂起线程（挂起操作涉及系统调用，存在用户态和内核态切换，这才是重量级锁最大的开销） </font>

## **锁粗化**

假设一系列的连续操作都会<font color='red'>对同一个对象反复加锁及解锁，甚至加锁操作是出现在循环体中的</font>，即使没有出现线程竞争，频繁地进行互斥同步操作也会导致不必要的性能损耗。<font color='red'>如果JVM检测到有一连串零碎的操作都是对同一对象的加锁，将会扩大加锁同步的范围（即锁粗化）到整个操作序列的外部。</font>

```
StringBuffer buffer = new StringBuffer();
/**
 * 锁粗化
 */
public void append(){
    buffer.append("aaa").append(" bbb").append(" ccc");
}
```

上述代码每次调用 buffer.append 方法都需要加锁和解锁，如果JVM检测到有一连串的对同一个对象加锁和解锁的操作，就会将其合并成一次范围更大的加锁和解锁操作，即在第一次append方法时进行加锁，最后一次append方法结束后进行解锁。

## **锁消除**

锁消除即删除不必要的加锁操作。<font color='red'>锁消除是Java虚拟机在JIT编译期间，通过对运行上下文的扫描，去除不可能存在共享资源竞争的锁，通过锁消除，可以节省毫无意义的请求锁时间。</font>

```java
public class LockEliminationTest {
    /**
     * 锁消除
     * -XX:+EliminateLocks 开启锁消除(jdk8默认开启）
     * -XX:-EliminateLocks 关闭锁消除
     * @param str1
     * @param str2
     */
    public void append(String str1, String str2) {
        StringBuffer stringBuffer = new StringBuffer();
        stringBuffer.append(str1).append(str2);
    }

    public static void main(String[] args) throws InterruptedException {
        LockEliminationTest demo = new LockEliminationTest();
        long start = System.currentTimeMillis();
        for (int i = 0; i < 100000000; i++) {
            demo.append("aaa", "bbb");
        }
        long end = System.currentTimeMillis();
        System.out.println("执行时间：" + (end - start) + " ms");
    }

```

StringBuffer的append是个同步方法，但是append方法中的 StringBuffer 属于一个局部变量，不可能从该方法中逃逸出去，因此其实这过程是线程安全的，可以将锁消除。

测试结果： 关闭锁消除执行时间4688 ms   开启锁消除执行时间：2601 ms

### **逃逸分析(Escape Analysis）**

​	逃逸分析，是一种可以有效减少Java 程序中同步负载和内存堆分配压力的跨函数全局数据流分析算法。<font color='red'>通过逃逸分析，Java Hotspot编译器能够分析出一个新的对象的引用的使用范围从而决定是否要将这个对象分配到堆上。逃逸分析的基本行为就是分析对象动态作用域。</font>

**方法逃逸(对象逃出当前方法)**

```
当一个对象在方法中被定义后，它可能被外部方法所引用，例如作为调用参数传递到其他地方中。
```

**线程逃逸((对象逃出当前线程)**

 	这个对象甚至可能被其它线程访问到，例如赋值给类变量或可以在其它线程中访问的实例变量。

使用逃逸分析，编译器可以对代码做如下优化：

1. 同步省略或锁消除(Synchronization Elimination)。如果一个对象被发现只能从一个线程被访问到，那么对于这个对象的操作可以不考虑同步。

2. 将堆分配转化为栈分配(Stack Allocation)。如果一个对象在子程序中被分配，要使指向该对象的指针永远不会逃逸，对象可能是栈分配的候选，而不是堆分配。

3. 分离对象或标量替换(Scalar Replacement)。有的对象可能不需要作为一个连续的内存结构存在也可以被访问到，那么对象的部分（或全部）可以不存储在内存，而是存储在CPU寄存器中。

​	jdk6才开始引入该技术，jdk7开始默认开启逃逸分析。在Java代码运行时，可以通过JVM参数指定是否开启逃逸分析：

```
-XX:+DoEscapeAnalysis  //表示开启逃逸分析 (jdk1.8默认开启）
-XX:-DoEscapeAnalysis //表示关闭逃逸分析。
-XX:+EliminateAllocations   //开启标量替换(默认打开)
```

**测试**

```java
/**
 * @author  Fox
 *
 * 进行两种测试
 * 关闭逃逸分析，同时调大堆空间，避免堆内GC的发生，如果有GC信息将会被打印出来
 * VM运行参数：-Xmx4G -Xms4G -XX:-DoEscapeAnalysis -XX:+PrintGCDetails -XX:+HeapDumpOnOutOfMemoryError
 *
 * 开启逃逸分析  jdk8默认开启
 * VM运行参数：-Xmx4G -Xms4G -XX:+DoEscapeAnalysis -XX:+PrintGCDetails -XX:+HeapDumpOnOutOfMemoryError
 *
 * 执行main方法后
 * jps 查看进程
 * jmap -histo 进程ID
 *
 */
@Slf4j
public class EscapeTest {

    public static void main(String[] args) {
        long start = System.currentTimeMillis();
        for (int i = 0; i < 500000; i++) {
            alloc();
        }

        long end = System.currentTimeMillis();

        log.info("执行时间：" + (end - start) + " ms");
        try {
            Thread.sleep(Integer.MAX_VALUE);
        } catch (InterruptedException e1) {
            e1.printStackTrace();
        }
    }

    /**
     * JIT编译时会对代码进行逃逸分析
     * 并不是所有对象存放在堆区，有的一部分存在线程栈空间
     * Ponit没有逃逸
     */
    private static String alloc() {
        Point point = new Point();
        return point.toString();
    }

    /**
     *同步省略（锁消除）  JIT编译阶段优化，JIT经过逃逸分析之后发现无线程安全问题，就会做锁消除
     */
    public void append(String str1, String str2) {
        StringBuffer stringBuffer = new StringBuffer();
        stringBuffer.append(str1).append(str2);
    }

    /**
     * 标量替换
     *
     */
    private static void test2() {
        Point point = new Point(1,2);
        System.out.println("point.x="+point.getX()+"; point.y="+point.getY());

//        int x=1;
//        int y=2;
//        System.out.println("point.x="+x+"; point.y="+y);
    }
}

@Data
@AllArgsConstructor
@NoArgsConstructor
class Point{
    private int x;
    private int y;
}

```

​           

测试结果：开启逃逸分析，部分对象会在栈上分配

​    ![0](8%E3%80%81%E5%B9%B6%E5%8F%91%E9%94%81%E6%9C%BA%E5%88%B6%E4%B9%8B%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3synchronized%E4%BA%8C/1693)

​    ![0](8%E3%80%81%E5%B9%B6%E5%8F%91%E9%94%81%E6%9C%BA%E5%88%B6%E4%B9%8B%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3synchronized%E4%BA%8C/1692)

