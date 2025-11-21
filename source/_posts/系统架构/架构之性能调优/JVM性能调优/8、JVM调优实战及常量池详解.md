---
title: 8、JVM调优实战及常量池详解
date: 2023-06-16 16:50:41
categories:
    - 系统架构
    - 架构之性能调优
    - JVM性能调优
tags:
    - JVM性能调优
---

# **阿里巴巴Arthas详解**

------

**Arthas** 是 Alibaba 在 2018 年 9 月开源的 **Java 诊断**工具。支持 JDK6+， 采用命令行交互模式，可以方便的定位和诊断线上程序运行问题。**Arthas** 官方文档十分详细，详见：https://arthas.aliyun.com/doc/

##  **Arthas**使用场景

得益于 **Arthas** 强大且丰富的功能，让 **Arthas** 能做的事情超乎想象。下面仅仅列举几项常见的使用情况，更多的使用场景可以在熟悉了 **Arthas** 之后自行探索。

1. 是否有一个全局视角来查看系统的运行状况？
2. 为什么 CPU 又升高了，到底是哪里占用了 CPU ？
3. 运行的多线程有死锁吗？有阻塞吗？
4. 程序运行耗时很长，是哪里耗时比较长呢？如何监测呢？
5. 这个类从哪个 jar 包加载的？为什么会报各种类相关的 Exception？
6. 我改的代码为什么没有执行到？难道是我没 commit？分支搞错了？
7. 遇到问题无法在线上 debug，难道只能通过加日志再重新发布吗？
8. 有什么办法可以监控到 JVM 的实时运行状态？

##  **Arthas**使用

```makefile
# github下载arthas
wget https://alibaba.github.io/arthas/arthas-boot.jar
# 或者 Gitee 下载
wget https://arthas.gitee.io/arthas-boot.jar
```

用java -jar运行即可，可以识别机器上所有Java进程(我们这里之前已经运行了一个Arthas测试程序，代码见下方)

​    ![0](8%E3%80%81JVM%E8%B0%83%E4%BC%98%E5%AE%9E%E6%88%98%E5%8F%8A%E5%B8%B8%E9%87%8F%E6%B1%A0%E8%AF%A6%E8%A7%A3/96552)

```java
package com.tuling.jvm;

import java.util.HashSet;

public class Arthas {

    private static HashSet hashSet = new HashSet();

    public static void main(String[] args) {
        // 模拟 CPU 过高
        cpuHigh();
        // 模拟线程死锁
        deadThread();
        // 不断的向 hashSet 集合增加数据
        addHashSetThread();
    }

    /**
     * 不断的向 hashSet 集合添加数据
     */
    public static void addHashSetThread() {
        // 初始化常量
        new Thread(() -> {
            int count = 0;
            while (true) {
                try {
                    hashSet.add("count" + count);
                    Thread.sleep(1000);
                    count++;
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }).start();
    }

    public static void cpuHigh() {
        new Thread(() -> {
            while (true) {

            }
        }).start();
    }

    /**
     * 死锁
     */
    private static void deadThread() {
        /** 创建资源 */
        Object resourceA = new Object();
        Object resourceB = new Object();
        // 创建线程
        Thread threadA = new Thread(() -> {
            synchronized (resourceA) {
                System.out.println(Thread.currentThread() + " get ResourceA");
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(Thread.currentThread() + "waiting get resourceB");
                synchronized (resourceB) {
                    System.out.println(Thread.currentThread() + " get resourceB");
                }
            }
        });

        Thread threadB = new Thread(() -> {
            synchronized (resourceB) {
                System.out.println(Thread.currentThread() + " get ResourceB");
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(Thread.currentThread() + "waiting get resourceA");
                synchronized (resourceA) {
                    System.out.println(Thread.currentThread() + " get resourceA");
                }
            }
        });
        threadA.start();
        threadB.start();
    }
}
```

选择进程序号1，进入进程信息操作

​    ![0](8%E3%80%81JVM%E8%B0%83%E4%BC%98%E5%AE%9E%E6%88%98%E5%8F%8A%E5%B8%B8%E9%87%8F%E6%B1%A0%E8%AF%A6%E8%A7%A3/96569)

输入**dashboard**可以查看整个进程的运行情况，线程、内存、GC、运行环境信息：

​    ![0](8%E3%80%81JVM%E8%B0%83%E4%BC%98%E5%AE%9E%E6%88%98%E5%8F%8A%E5%B8%B8%E9%87%8F%E6%B1%A0%E8%AF%A6%E8%A7%A3/96574)

输入**thread**可以查看线程详细情况

​    ![0](8%E3%80%81JVM%E8%B0%83%E4%BC%98%E5%AE%9E%E6%88%98%E5%8F%8A%E5%B8%B8%E9%87%8F%E6%B1%A0%E8%AF%A6%E8%A7%A3/96580)

输入 **thread加上线程ID** 可以查看线程堆栈

​    ![0](8%E3%80%81JVM%E8%B0%83%E4%BC%98%E5%AE%9E%E6%88%98%E5%8F%8A%E5%B8%B8%E9%87%8F%E6%B1%A0%E8%AF%A6%E8%A7%A3/96586)

输入 **thread -b** 可以查看线程死锁

​    ![0](8%E3%80%81JVM%E8%B0%83%E4%BC%98%E5%AE%9E%E6%88%98%E5%8F%8A%E5%B8%B8%E9%87%8F%E6%B1%A0%E8%AF%A6%E8%A7%A3/96592)

**输入 jad加类的全名 可以反编译，这样可以方便我们查看线上代码是否是正确的版本**

​    ![0](8%E3%80%81JVM%E8%B0%83%E4%BC%98%E5%AE%9E%E6%88%98%E5%8F%8A%E5%B8%B8%E9%87%8F%E6%B1%A0%E8%AF%A6%E8%A7%A3/96602)

**使用 ognl 命令可以查看线上系统变量的值，甚至可以修改变量的值**

​    ![0](8%E3%80%81JVM%E8%B0%83%E4%BC%98%E5%AE%9E%E6%88%98%E5%8F%8A%E5%B8%B8%E9%87%8F%E6%B1%A0%E8%AF%A6%E8%A7%A3/96607)

​    ![0](8%E3%80%81JVM%E8%B0%83%E4%BC%98%E5%AE%9E%E6%88%98%E5%8F%8A%E5%B8%B8%E9%87%8F%E6%B1%A0%E8%AF%A6%E8%A7%A3/96615)

更多命令使用可以用help命令查看，或查看文档：https://alibaba.github.io/arthas/commands.html#arthas

# **GC日志详解**

------

对于java应用我们可以通过一些配置把程序运行过程中的gc日志全部打印出来，然后分析gc日志得到关键性指标，分析GC原因，调优JVM参数。

打印GC日志方法，在JVM参数里增加参数，%t 代表时间

```
-Xloggc:./gc-%t.log -XX:+PrintGCDetails -XX:+PrintGCDateStamps  -XX:+PrintGCTimeStamps -XX:+PrintGCCause  
-XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=10 -XX:GCLogFileSize=100M
```

Tomcat则直接加在JAVA_OPTS变量里。

**如何分析GC日志**

运行程序加上对应gc日志

```
java -jar -Xloggc:./gc-%t.log -XX:+PrintGCDetails -XX:+PrintGCDateStamps  -XX:+PrintGCTimeStamps -XX:+PrintGCCause  
-XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=10 -XX:GCLogFileSize=100M microservice-eureka-server.jar
```

​       

下图中是我截取的JVM刚启动的一部分GC日志 

​    ![0](8%E3%80%81JVM%E8%B0%83%E4%BC%98%E5%AE%9E%E6%88%98%E5%8F%8A%E5%B8%B8%E9%87%8F%E6%B1%A0%E8%AF%A6%E8%A7%A3/80226)

我们可以看到图中第一行红框，是项目的配置参数。这里不仅配置了打印GC日志，还有相关的VM内存参数。 

第二行红框中的是在这个GC时间点发生GC之后相关GC情况。 

1、对于**2.909：**  这是从jvm启动开始计算到这次GC经过的时间，前面还有具体的发生时间日期。 

2、Full GC(Metadata GC Threshold)指这是一次full gc，括号里是gc的原因， PSYoungGen是年轻代的GC，ParOldGen是老年代的GC，Metaspace是元空间的GC

3、 6160K->0K(141824K)，这三个数字分别对应GC之前占用年轻代的大小，GC之后年轻代占用，以及整个年轻代的大小。 

4、112K->6056K(95744K)，这三个数字分别对应GC之前占用老年代的大小，GC之后老年代占用，以及整个老年代的大小。 

5、6272K->6056K(237568K)，这三个数字分别对应GC之前占用堆内存的大小，GC之后堆内存占用，以及整个堆内存的大小。 

6、20516K->20516K(1069056K)，这三个数字分别对应GC之前占用元空间内存的大小，GC之后元空间内存占用，以及整个元空间内存的大小。 

7、0.0209707是该时间点GC总耗费时间。 

从日志可以发现几次fullgc都是由于元空间不够导致的，所以我们可以将元空间调大点

```sh
java -jar -Xloggc:./gc-adjust-%t.log -XX:MetaspaceSize=256M -XX:MaxMetaspaceSize=256M -XX:+PrintGCDetails -XX:+PrintGCDateStamps  
-XX:+PrintGCTimeStamps -XX:+PrintGCCause  -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=10 -XX:GCLogFileSize=100M 
microservice-eureka-server.jar
```

​        

调整完我们再看下gc日志发现已经没有因为元空间不够导致的fullgc了

对于CMS和G1收集器的日志会有一点不一样，也可以试着打印下对应的gc日志分析下，可以发现gc日志里面的gc步骤跟我们之前讲过的步骤是类似的

```java
public class HeapTest {

    byte[] a = new byte[1024 * 100];  //100KB

    public static void main(String[] args) throws InterruptedException {
        ArrayList<HeapTest> heapTests = new ArrayList<>();
        while (true) {
            heapTests.add(new HeapTest());
            Thread.sleep(10);
        }
    }
}
```

# **CMS**

```
-Xloggc:d:/gc-cms-%t.log -Xms50M -Xmx50M -XX:MetaspaceSize=256M -XX:MaxMetaspaceSize=256M -XX:+PrintGCDetails -XX:+PrintGCDateStamps  
 -XX:+PrintGCTimeStamps -XX:+PrintGCCause  -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=10 -XX:GCLogFileSize=100M 
 -XX:+UseParNewGC -XX:+UseConcMarkSweepGC   
```

# **G1**

```
-Xloggc:d:/gc-g1-%t.log -Xms50M -Xmx50M -XX:MetaspaceSize=256M -XX:MaxMetaspaceSize=256M -XX:+PrintGCDetails -XX:+PrintGCDateStamps  
 -XX:+PrintGCTimeStamps -XX:+PrintGCCause  -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=10 -XX:GCLogFileSize=100M -XX:+UseG1GC 
```

# GCEasy工具

上面的这些参数，能够帮我们查看分析GC的垃圾收集情况。但是如果GC日志很多很多，成千上万行。就算你一目十行，看完了，脑子也是一片空白。所以我们可以借助一些功能来帮助我们分析，这里推荐一个gceasy([https://gceasy.io](https://gceasy.io/))，可以上传gc文件，然后他会利用可视化的界面来展现GC情况。具体下图所示 

​    ![0](8%E3%80%81JVM%E8%B0%83%E4%BC%98%E5%AE%9E%E6%88%98%E5%8F%8A%E5%B8%B8%E9%87%8F%E6%B1%A0%E8%AF%A6%E8%A7%A3/80219)

上图我们可以看到年轻代，老年代，以及永久代的内存分配，和最大使用情况。 

​    ![0](8%E3%80%81JVM%E8%B0%83%E4%BC%98%E5%AE%9E%E6%88%98%E5%8F%8A%E5%B8%B8%E9%87%8F%E6%B1%A0%E8%AF%A6%E8%A7%A3/80221)

上图我们可以看到堆内存在GC之前和之后的变化，以及其他信息。

这个工具还提供基于机器学习的JVM智能优化建议，当然现在这个功能需要付费

​    ![0](8%E3%80%81JVM%E8%B0%83%E4%BC%98%E5%AE%9E%E6%88%98%E5%8F%8A%E5%B8%B8%E9%87%8F%E6%B1%A0%E8%AF%A6%E8%A7%A3/80273)

​    ![0](8%E3%80%81JVM%E8%B0%83%E4%BC%98%E5%AE%9E%E6%88%98%E5%8F%8A%E5%B8%B8%E9%87%8F%E6%B1%A0%E8%AF%A6%E8%A7%A3/80276)

# **JVM参数汇总查看命令**

------

java -XX:+PrintFlagsInitial 表示打印出所有参数选项的默认值

java -XX:+PrintFlagsFinal 表示打印出所有参数选项在运行程序时生效的值

# **Class常量池与运行时常量池**

------

Class常量池可以理解为是Class文件中的资源仓库。 Class文件中除了包含类的版本、字段、方法、接口等描述信息外，还有一项信息就是**常量池(constant pool table)**，用于存放编译期生成的各种**字面量(Literal)和符号引用(Symbolic References)**。

一个class文件的16进制大体结构如下图：

​    ![0](8%E3%80%81JVM%E8%B0%83%E4%BC%98%E5%AE%9E%E6%88%98%E5%8F%8A%E5%B8%B8%E9%87%8F%E6%B1%A0%E8%AF%A6%E8%A7%A3/80047)

对应的含义如下，细节可以查下oracle官方文档

​    ![0](8%E3%80%81JVM%E8%B0%83%E4%BC%98%E5%AE%9E%E6%88%98%E5%8F%8A%E5%B8%B8%E9%87%8F%E6%B1%A0%E8%AF%A6%E8%A7%A3/80124)

当然我们一般不会去人工解析这种16进制的字节码文件，我们一般可以通过javap命令生成更可读的JVM字节码指令文件：

**javap -v Math.class**![0](8%E3%80%81JVM%E8%B0%83%E4%BC%98%E5%AE%9E%E6%88%98%E5%8F%8A%E5%B8%B8%E9%87%8F%E6%B1%A0%E8%AF%A6%E8%A7%A3/80062)

红框标出的就是class常量池信息，常量池中主要存放两大类常量：**字面量和符号引用**。

## **字面量**

**字面量就是指由字母、数字等构成的字符串或者数值常量**

字面量只可以右值出现，所谓右值是指等号右边的值，如：int a=1 这里的a为左值，1为右值。在这个例子中1就是字面量。

```java 
int a = 1;
int b = 2;
int c = "abcdefg";
int d = "abcdefg";
```

## **符号引用**

符号引用是编译原理中的概念，是相对于直接引用来说的。主要包括了以下三类常量：

- 类和接口的全限定名 
- 字段的名称和描述符 
- 方法的名称和描述符

上面的a，b就是字段名称，就是一种符号引用，还有Math类常量池里的 Lcom/tuling/jvm/Math 是类的全限定名，main和compute是方法名称，()是一种UTF8格式的描述符，这些都是符号引用。

这些常量池现在是静态信息，只有到运行时被加载到内存后，这些符号才有对应的内存地址信息，这些常量池一旦被装入内存就变成**运行时常量池**，对应的符号引用在程序加载或运行时会被转变为被加载到内存区域的代码的直接引用，也就是我们说的**动态链接了。例如，compute()这个符号引用在运行时就会被转变为compute()方法具体代码在内存中的地址，主要通过对象头里的类型指针去转换直接引用。**

## **字符串常量池**

------

### **字符串常量池的设计思想**

1. 字符串的分配，和其他的对象分配一样，耗费高昂的时间与空间代价，作为最基础的数据类型，大量频繁的创建字符串，极大程度地影响程序的性能
2. JVM为了提高性能和减少内存开销，在实例化字符串常量的时候进行了一些优化

- 为字符串开辟一个字符串常量池，类似于缓存区
- 创建字符串常量时，首先查询字符串常量池是否存在该字符串
- 存在该字符串，返回引用实例，不存在，实例化该字符串并放入池中

### **三种字符串操作(Jdk1.7 及以上版本)**

- 直接赋值字符串

```java
String s = "zhuge";  // s指向常量池中的引用       
```

这种方式创建的字符串对象，只会在常量池中。

因为有"zhuge"这个字面量，创建对象s的时候，JVM会先去常量池中通过 equals(key) 方法，判断是否有相同的对象

如果有，则直接返回该对象在常量池中的引用；

如果没有，则会在常量池中创建一个新对象，再返回引用。

- new String();

```java
String s1 = new String("zhuge");  // s1指向内存中的对象引用      
```

这种方式会保证字符串常量池和，没有就创建，最后返回堆内存中的对象引用。

步骤大致如下：

因为有"zhuge"这个字面量，所以会先检查字符串常量池中是否存在字符串"zhuge"

不存在，先在字符串常量池里创建一个字符串对象；再去内存中创建一个字符串对象"zhuge"；

存在的话，就直接去堆内存中创建一个字符串对象"zhuge"；

最后，将内存中的引用返回。

- intern方法

```java
String s1 = new String("zhuge");   
String s2 = s1.intern();

System.out.println(s1 == s2);  //false
```

String中的intern方法是一个 native 的方法，当调用 intern方法时，如果池已经包含一个等于此String对象的字符串（用equals(oject)方法确定），则返回池中的字符串。**否则，将intern返回的引用指向当前字符串 s1**(**jdk1.6版本需要将 s1 复制到字符串常量池里**)。

### **字符串常量池位置**

Jdk1.6及之前： 有永久代, 运行时常量池在永久代，运行时常量池包含字符串常量池

Jdk1.7：有永久代，但已经逐步“去永久代”，字符串常量池从永久代里的运行时常量池分离到堆里

Jdk1.8及之后： 无永久代，运行时常量池在元空间，字符串常量池里依然在堆里

用一个程序证明下字符串常量池在哪里：

```java
/**
 * jdk6：-Xms6M -Xmx6M -XX:PermSize=6M -XX:MaxPermSize=6M  
 * jdk8：-Xms6M -Xmx6M -XX:MetaspaceSize=6M -XX:MaxMetaspaceSize=6M
 */
public class RuntimeConstantPoolOOM{
    public static void main(String[] args) {
        ArrayList<String> list = new ArrayList<String>();
        for (int i = 0; i < 10000000; i++) {
            String str = String.valueOf(i).intern();
            list.add(str);
        }
    }
}

运行结果：
jdk7及以上：Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
jdk6：Exception in thread "main" java.lang.OutOfMemoryError: PermGen space
```

​        

### **字符串常量池设计原理**

　　字符串常量池底层是hotspot的C++实现的，底层类似一个 HashTable， 保存的本质上是字符串对象的引用。

看一道比较常见的面试题，下面的代码创建了多少个 String 对象？

```java
String s1 = new String("he") + new String("llo");
String s2 = s1.intern();
 
System.out.println(s1 == s2);
// 在 JDK 1.6 下输出是 false，创建了 6 个对象
// 在 JDK 1.7 及以上的版本输出是 true，创建了 5 个对象
// 当然我们这里没有考虑GC，但这些对象确实存在或存在过
```

　　为什么输出会有这些变化呢？主要还是字符串池从永久代中脱离、移入堆区的原因， intern() 方法也相应发生了变化：

1、在 JDK 1.6 中，调用 intern() 首先会在字符串池中寻找 equal() 相等的字符串，假如字符串存在就返回该字符串在字符串池中的引用；假如字符串不存在，虚拟机会重新在永久代上创建一个实例，将 StringTable 的一个表项指向这个新创建的实例。

​    ![0](8%E3%80%81JVM%E8%B0%83%E4%BC%98%E5%AE%9E%E6%88%98%E5%8F%8A%E5%B8%B8%E9%87%8F%E6%B1%A0%E8%AF%A6%E8%A7%A3/96690)

 

2、在 JDK 1.7 (及以上版本)中，由于字符串池不在永久代了，intern() 做了一些修改，更方便地利用堆中的对象。字符串存在时和 JDK 1.6一样，但是字符串不存在时不再需要重新创建实例，可以直接指向堆上的实例。

​    ![0](8%E3%80%81JVM%E8%B0%83%E4%BC%98%E5%AE%9E%E6%88%98%E5%8F%8A%E5%B8%B8%E9%87%8F%E6%B1%A0%E8%AF%A6%E8%A7%A3/96692)

 

　　由上面两个图，也不难理解为什么 JDK 1.6 字符串池溢出会抛出 OutOfMemoryError: PermGen space ，而在 JDK 1.7 及以上版本抛出 OutOfMemoryError: Java heap space 。

**String常量池问题的几个例子**

示例1：

```java
String s0="zhuge";
String s1="zhuge";
String s2="zhu" + "ge";
System.out.println( s0==s1 ); //true
System.out.println( s0==s2 ); //true
```

  

分析：因为例子中的 s0和s1中的”zhuge”都是字符串常量，它们在编译期就被确定了，所以s0==s1为true；而”zhu”和”ge”也都是字符串常量，当一个字 符串由多个字符串常量连接而成时，它自己肯定也是字符串常量，所以s2也同样在编译期就被优化为一个字符串常量"zhuge"，所以s2也是常量池中” zhuge”的一个引用。所以我们得出s0==s1==s2；

示例2：

```java
String s0="zhuge";
String s1=new String("zhuge");
String s2="zhu" + new String("ge");
System.out.println( s0==s1 );　　// false
System.out.println( s0==s2 )；　 // false
System.out.println( s1==s2 );　　// false
```

​        

分析：用new String() 创建的字符串不是常量，不能在编译期就确定，所以new String() 创建的字符串不放入常量池中，它们有自己的地址空间。

s0还是常量池 中"zhuge”的引用，s1因为无法在编译期确定，所以是运行时创建的新对象”zhuge”的引用，s2因为有后半部分 new String(”ge”)所以也无法在编译期确定，所以也是一个新创建对象”zhuge”的引用;明白了这些也就知道为何得出此结果了。

示例3：

```java
  String a = "a1";
  String b = "a" + 1;
  System.out.println(a == b); // true 
  
  String a = "atrue";
  String b = "a" + "true";
  System.out.println(a == b); // true 
  
  String a = "a3.4";
  String b = "a" + 3.4;
  System.out.println(a == b); // true
```

​      

分析：JVM对于字符串常量的"+"号连接，将在程序编译期，JVM就将常量字符串的"+"连接优化为连接后的值，拿"a" + 1来说，经编译器优化后在class中就已经是a1。在编译期其字符串常量的值就确定下来，故上面程序最终的结果都为true。

示例4：

```java
String a = "ab";
String bb = "b";
String b = "a" + bb;

System.out.println(a == b); // false
```

​        

分析：JVM对于字符串引用，由于在字符串的"+"连接中，有字符串引用存在，而引用的值在程序编译期是无法确定的，即"a" + bb无法被编译器优化，只有在程序运行期来动态分配并将连接后的新地址赋给b。所以上面程序的结果也就为false。

示例5：

```java
String a = "ab";
final String bb = "b";
String b = "a" + bb;

System.out.println(a == b); // true
```

​            

分析：和示例4中唯一不同的是bb字符串加了final修饰，对于final修饰的变量，它在编译时被解析为常量值的一个本地拷贝存储到自己的常量池中或嵌入到它的字节码流中。所以此时的"a" + bb和"a" + "b"效果是一样的。故上面程序的结果为true。

示例6：

```java
String a = "ab";
final String bb = getBB();
String b = "a" + bb;

System.out.println(a == b); // false

private static String getBB() 
{  
    return "b";  
 }
```

分析：JVM对于字符串引用bb，它的值在编译期无法确定，只有在程序运行期调用方法后，将方法的返回值和"a"来动态连接并分配地址为b，故上面 程序的结果为false。

 

**关于String是不可变的**

​    通过上面例子可以得出得知：

```java
String  s  =  "a" + "b" + "c";  //就等价于String s = "abc";
String  a  =  "a";
String  b  =  "b";
String  c  =  "c";
String  s1  =   a  +  b  +  c;
```

　　s1 这个就不一样了，可以通过观察其**JVM指令码**发现s1的"+"操作会变成如下操作：

```java
StringBuilder temp = new StringBuilder();
temp.append(a).append(b).append(c);
String s = temp.toString();
```

**最后再看一个例子**：

```java
//字符串常量池："计算机"和"技术"     堆内存：str1引用的对象"计算机技术"  
//堆内存中还有个StringBuilder的对象，但是会被gc回收，StringBuilder的toString方法会new String()，这个String才是真正返回的对象引用
String str2 = new StringBuilder("计算机").append("技术").toString();   //没有出现"计算机技术"字面量，所以不会在常量池里生成"计算机技术"对象
System.out.println(str2 == str2.intern());  //true
//"计算机技术" 在池中没有，但是在heap中存在，则intern时，会直接返回该heap中的引用

//字符串常量池："ja"和"va"     堆内存：str1引用的对象"java"  
//堆内存中还有个StringBuilder的对象，但是会被gc回收，StringBuilder的toString方法会new String()，这个String才是真正返回的对象引用
String str1 = new StringBuilder("ja").append("va").toString();    //没有出现"java"字面量，所以不会在常量池里生成"java"对象
System.out.println(str1 == str1.intern());  //false
//java是关键字，在JVM初始化的相关类里肯定早就放进字符串常量池了

String s1=new String("test");  
System.out.println(s1==s1.intern());   //false
//"test"作为字面量，放入了池中，而new时s1指向的是heap中新生成的string对象，s1.intern()指向的是"test"字面量之前在池中生成的字符串对象

String s2=new StringBuilder("abc").toString();
System.out.println(s2==s2.intern());  //false
//同上
```

### **八种基本类型的包装类和对象池**

------

java中基本类型的包装类的大部分都实现了常量池技术(严格来说应该叫**对象池，**在堆上)，这些类是Byte,Short,Integer,Long,Character,Boolean,另外两种浮点数类型的包装类则没有实现。另外Byte,Short,Integer,Long,Character这5种整型的包装类也只是在对应值小于等于127时才可使用对象池，也即对象不负责创建和管理大于127的这些类的对象。因为一般这种比较小的数用到的概率相对较大。

```java
public class Test {

    public static void main(String[] args) {
        //5种整形的包装类Byte,Short,Integer,Long,Character的对象，  
        //在值小于127时可以使用对象池  
        Integer i1 = 127;  //这种调用底层实际是执行的Integer.valueOf(127)，里面用到了IntegerCache对象池
        Integer i2 = 127;
        System.out.println(i1 == i2);//输出true  

        //值大于127时，不会从对象池中取对象  
        Integer i3 = 128;
        Integer i4 = 128;
        System.out.println(i3 == i4);//输出false  
        
        //用new关键词新生成对象不会使用对象池
        Integer i5 = new Integer(127);  
        Integer i6 = new Integer(127);
        System.out.println(i5 == i6);//输出false 

        //Boolean类也实现了对象池技术  
        Boolean bool1 = true;
        Boolean bool2 = true;
        System.out.println(bool1 == bool2);//输出true  

        //浮点类型的包装类没有实现对象池技术  
        Double d1 = 1.0;
        Double d2 = 1.0;
        System.out.println(d1 == d2);//输出false  
    }
} 
```

​        
