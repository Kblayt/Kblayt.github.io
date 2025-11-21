---
title: 13、阻塞队列BlockingQueue实战及其原理分析一
date: 2023-06-16 23:39:06
categories:
    - 系统架构
    - 架构之并发编程
tags:
    - 并发编程
    - JUC
---

# **阻塞队列介绍**

## **Queue接口**

```java
public interface Queue<E> extends Collection<E> {
    //添加一个元素，添加成功返回true, 如果队列满了，就会抛出异常
    boolean add(E e);
    //添加一个元素，添加成功返回true, 如果队列满了，返回false
    boolean offer(E e);
    //返回并删除队首元素，队列为空则抛出异常
    E remove();
    //返回并删除队首元素，队列为空则返回null
    E poll();
    //返回队首元素，但不移除，队列为空则抛出异常
    E element();
    //获取队首元素，但不移除，队列为空则返回null
    E peek();
}
```

   

## **BlockingQueue接口**

BlockingQueue 继承了 Queue 接口，是队列的一种。Queue 和 BlockingQueue 都是在 Java 5 中加入的。阻塞队列（BlockingQueue）是一个在队列基础上又支持了两个附加操作的队列，常用解耦。两个附加操作:

- **支持阻塞**的插入方法**put**: 队列满时，队列会阻塞插入元素的线程，直到队列不满。
- **支持阻塞**的移除方法**take**: 队列空时，获取元素的线程会等待队列变为非空

​    ![0](13%E3%80%81%E9%98%BB%E5%A1%9E%E9%98%9F%E5%88%97BlockingQueue%E5%AE%9E%E6%88%98%E5%8F%8A%E5%85%B6%E5%8E%9F%E7%90%86%E5%88%86%E6%9E%90%E4%B8%80/1832)

BlockingQueue和JDK集合包中的Queue接口兼容，同时在其基础上增加了阻塞功能。

**入队：**

（1）offer(E e)：如果队列没满，返回true，如果队列已满，返回false（不阻塞）

（2）offer(E e, long timeout, TimeUnit unit)：可以设置阻塞时间，如果队列已满，则进行阻塞。超过阻塞时间，则返回false

（3）<font color='red'>put(E e)：队列没满的时候是正常的插入，如果队列已满，则阻塞，直至队列空出位置 </font>

**出队：**

（1）poll()：如果有数据，出队，如果没有数据，返回null   （不阻塞）

（2）poll(long timeout, TimeUnit unit)：可以设置阻塞时间，如果没有数据，则阻塞，超过阻塞时间，则返回null

（3）<font color='red'>take()：队列里有数据会正常取出数据并删除；但是如果队列里无数据，则阻塞，直到队列里有数据</font>

### **BlockingQueue常用方法示例**

当队列满了无法添加元素，或者是队列空了无法移除元素时：

1. 抛出异常：add、remove、element
2. 返回结果但不抛出异常：offer、poll、peek
3. 阻塞：put、take

| 方法         | 抛出异常  | 返回特定值 | 阻塞                            | 阻塞特定时间                                  |
| ------------ | --------- | ---------- | ------------------------------- | --------------------------------------------- |
| 入队         | add(e)    | offer(e)   | <font color='red'>put(e)</font> | <font color='red'>offer(e, time, unit)</font> |
| 出队         | remove()  | poll()     | <font color='red'>take()</font> | <font color='red'>poll(time, unit)</font>     |
| 获取队首元素 | element() | peek()     | 不支持                          | 不支持                                        |

```java
public class BlockingQueueTest {

    public static void main(String[] args) {

        addTest();

    }

    /**
     * add 方法是往队列里添加一个元素，如果队列满了，就会抛出异常来提示队列已满。
     */
    private static void addTest() {
        BlockingQueue<Integer> blockingQueue = new ArrayBlockingQueue<Integer>(2);
        System.out.println(blockingQueue.add(1));
        System.out.println(blockingQueue.add(2));
        System.out.println(blockingQueue.add(3));
    }

    /**
     * remove 方法的作用是删除元素并返回队列的头节点，如果删除的队列是空的， remove 方法就会抛出异常。
     */
    private static void removeTest() {
        ArrayBlockingQueue<Integer> blockingQueue = new ArrayBlockingQueue<Integer>(2);
        blockingQueue.add(1);
        blockingQueue.add(2);
        System.out.println(blockingQueue.remove());
        System.out.println(blockingQueue.remove());
        System.out.println(blockingQueue.remove());
    }

    /**
     * element 方法是返回队列的头部节点，但是并不删除。如果队列为空，抛出异常
     */
    private static void elementTest() {
        ArrayBlockingQueue<Integer> blockingQueue = new ArrayBlockingQueue<Integer>(2);
        blockingQueue.element();
    }

    /**
     * offer 方法用来插入一个元素。如果添加成功会返回 true，而如果队列已经满了，返回false
     */
    private static void offerTest(){
        ArrayBlockingQueue<Integer> blockingQueue = new ArrayBlockingQueue<Integer>(2);
        System.out.println(blockingQueue.offer(1));
        System.out.println(blockingQueue.offer(2));
        System.out.println(blockingQueue.offer(3));
    }

    /**
     * poll 方法作用也是移除并返回队列的头节点。 如果队列为空，返回null
     */
    private static void pollTest() {
        ArrayBlockingQueue<Integer> blockingQueue = new ArrayBlockingQueue<Integer>(3);
        blockingQueue.offer(1);
        blockingQueue.offer(2);
        blockingQueue.offer(3);
        System.out.println(blockingQueue.poll());
        System.out.println(blockingQueue.poll());
        System.out.println(blockingQueue.poll());
        System.out.println(blockingQueue.poll());
    }

    /**
     * peek 方法返回队列的头元素但并不删除。 如果队列为空，返回null
     */
    private static void peekTest() {
        ArrayBlockingQueue<Integer> blockingQueue = new ArrayBlockingQueue<Integer>(2);
        System.out.println(blockingQueue.peek());
    }

    /**
     * put 方法的作用是插入元素。如果队列已满就无法继续插入,阻塞插入线程，直至队列空出位置 
     */
    private static void putTest(){
        BlockingQueue<Integer> blockingQueue = new ArrayBlockingQueue<Integer>(2);
        try {
            blockingQueue.put(1);
            blockingQueue.put(2);
            blockingQueue.put(3);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    /**
     * take 方法的作用是获取并移除队列的头结点。如果执队列里无数据，则阻塞，直到队列里有数据
     */
    private static void takeTest(){
        BlockingQueue<Integer> blockingQueue = new ArrayBlockingQueue<Integer>(2);
        try {
            blockingQueue.take();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

 

## **阻塞队列特性**

### **阻塞**

阻塞队列区别于其他类型的队列的最主要的特点就是“阻塞”这两个字，所以下面重点介绍阻塞功能：<font color='red'>阻塞功能使得生产者和消费者两端的能力得以平衡，当有任何一端速度过快时，阻塞队列便会把过快的速度给降下来。</font>实现阻塞最重要的两个方法是 take 方法和 put 方法。

**take 方法**

take 方法的功能是获取并移除队列的头结点，通常在队列里有数据的时候是可以正常移除的。<font color='red'>可是一旦执行 take 方法的时候，队列里无数据，则阻塞，</font>直到队列里有数据。一旦队列里有数据了，就会立刻解除阻塞状态，并且取到数据。过程如图所示：

​    ![0](13%E3%80%81%E9%98%BB%E5%A1%9E%E9%98%9F%E5%88%97BlockingQueue%E5%AE%9E%E6%88%98%E5%8F%8A%E5%85%B6%E5%8E%9F%E7%90%86%E5%88%86%E6%9E%90%E4%B8%80/1831)

**put 方法**

put 方法插入元素时，如果队列没有满，那就和普通的插入一样是正常的插入，但是如果队列已满，那么就无法继续插入，则阻塞，直到队列里有了空闲空间。如果后续队列有了空闲空间，比如消费者消费了一个元素，那么此时队列就会解除阻塞状态，并把需要添加的数据添加到队列中。过程如图所示：

​    ![0](13%E3%80%81%E9%98%BB%E5%A1%9E%E9%98%9F%E5%88%97BlockingQueue%E5%AE%9E%E6%88%98%E5%8F%8A%E5%85%B6%E5%8E%9F%E7%90%86%E5%88%86%E6%9E%90%E4%B8%80/1829)

思考：阻塞队列是否有容量限制？

### **是否有界**

​	阻塞队列还有一个非常重要的属性，那就是<font color='red'>容量的大小，分为有界和无界两种。</font>无界队列意味着里面可以容纳非常多的元素，例如 LinkedBlockingQueue 的上限是 Integer.MAX_VALUE，是非常大的一个数，可以近似认为是无限容量，因为我们几乎无法把这个容量装满。但是有的阻塞队列是有界的，例如 ArrayBlockingQueue 如果容量满了，也不会扩容，所以一旦满了就无法再往里放数据了。

## **应用场景**

​	<font color='red'>BlockingQueue 是线程安全的，我们在很多场景下都可以利用线程安全的队列来优雅地解决我们业务自身的线程安全问题</font>。比如说，使用生产者/消费者模式的时候，我们生产者只需要往队列里添加元素，而消费者只需要从队列里取出它们就可以了，如图所示：

​    ![0](13%E3%80%81%E9%98%BB%E5%A1%9E%E9%98%9F%E5%88%97BlockingQueue%E5%AE%9E%E6%88%98%E5%8F%8A%E5%85%B6%E5%8E%9F%E7%90%86%E5%88%86%E6%9E%90%E4%B8%80/1830)

因为阻塞队列是线程安全的，所以生产者和消费者都可以是多线程的，不会发生线程安全问题。<font color='red'>生产者/消费者直接使用线程安全的队列就可以</font>，而不需要自己去考虑更多的线程安全问题。这也就意味着，考虑锁等线程安全问题的重任从“你”转移到了“队列”上，降低了我们开发的难度和工作量。

同时，队列它还能起到一个隔离的作用。比如说我们开发一个银行转账的程序，那么生产者线程不需要关心具体的转账逻辑，只需要把转账任务，如账户和金额等信息放到队列中就可以，而不需要去关心银行这个类如何实现具体的转账业务。而作为银行这个类来讲，它会去从队列里取出来将要执行的具体的任务，再去通过自己的各种方法来完成本次转账。这样就实现了具体任务与执行任务类之间的解耦，任务被放在了阻塞队列中，而负责放任务的线程是无法直接访问到我们银行具体实现转账操作的对象的，实现了隔离，提高了安全性。

## **常见阻塞队列**

BlockingQueue 接口的实现类都被放在了 juc 包中，它们的区别主要体现在存储结构上或对元素操作上的不同，但是对于take与put操作的原理，却是类似的。

| **队列**                  | 描述                                                      |
| ------------------------- | --------------------------------------------------------- |
| **ArrayBlockingQueue**    | 基于数组结构实现的一个有界阻塞队列                        |
| **LinkedBlockingQueue**   | 基于链表结构实现的一个有界阻塞队列                        |
| **PriorityBlockingQueue** | 支持按优先级排序的无界阻塞队列                            |
| **DelayQueue**            | 基于优先级队列（PriorityBlockingQueue）实现的无界阻塞队列 |
| **SynchronousQueue**      | 不存储元素的阻塞队列                                      |
| **LinkedTransferQueue**   | 基于链表结构实现的一个无界阻塞队列                        |
| **LinkedBlockingDeque**   | 基于链表结构实现的一个双端阻塞队列                        |

https://www.processon.com/view/link/618ce3941e0853689b0818e2

​    ![0](13%E3%80%81%E9%98%BB%E5%A1%9E%E9%98%9F%E5%88%97BlockingQueue%E5%AE%9E%E6%88%98%E5%8F%8A%E5%85%B6%E5%8E%9F%E7%90%86%E5%88%86%E6%9E%90%E4%B8%80/1833)

# **ArrayBlockingQueue**

ArrayBlockingQueue是最典型的有界阻塞队列，其内部是用数组存储元素的，初始化时需要指定容量大小，利用 ReentrantLock 实现线程安全。

在生产者-消费者模型中使用时，如果生产速度和消费速度基本匹配的情况下，使用ArrayBlockingQueue是个不错选择；当如果生产速度远远大于消费速度，则会导致队列填满，大量生产线程被阻塞。

使用独占锁ReentrantLock实现线程安全，入队和出队操作使用同一个锁对象，也就是只能有一个线程可以进行入队或者出队操作；这也就意味着生产者和消费者无法并行操作，在高并发场景下会成为性能瓶颈。

​    ![0](13%E3%80%81%E9%98%BB%E5%A1%9E%E9%98%9F%E5%88%97BlockingQueue%E5%AE%9E%E6%88%98%E5%8F%8A%E5%85%B6%E5%8E%9F%E7%90%86%E5%88%86%E6%9E%90%E4%B8%80/1840)

## **ArrayBlockingQueue使用**

```java
BlockingQueue queue = new ArrayBlockingQueue(1024);
queue.put("1");   //向队列中添加元素
```

##  **ArrayBlockingQueue的原理**

### **数据结构**

利用了Lock锁的Condition通知机制进行阻塞控制。

核心：一把锁，两个条件

```java
//数据元素数组
final Object[] items;
//下一个待取出元素索引
int takeIndex;
//下一个待添加元素索引
int putIndex;
//元素个数
int count;
//内部锁
final ReentrantLock lock-
//消费者
private final Condition notEmpty;
//生产者
private final Condition notFull;  

public ArrayBlockingQueue(int capacity) {
    this(capacity, false);
}
public ArrayBlockingQueue(int capacity, boolean fair) {
    ...
    lock = new ReentrantLock(fair); //公平，非公平
    notEmpty = lock.newCondition();
    notFull =  lock.newCondition();
}
```

   

### **入队put方法**

```java
public void put(E e) throws InterruptedException {
	//检查是否为空
    checkNotNull(e);
    final ReentrantLock lock = this.lock;
    //加锁，如果线程中断抛出异常 
    lock.lockInterruptibly();
    try {
       //阻塞队列已满，则将生产者挂起，等待消费者唤醒
       //设计注意点： 用while不用if是为了防止虚假唤醒
        while (count == items.length)
            notFull.await(); //队列满了，使用notFull等待（生产者阻塞）
        // 入队
        enqueue(e);
    } finally {
        lock.unlock(); // 唤醒消费者线程
    }
}
    
private void enqueue(E x) {
    final Object[] items = this.items;
    //入队   使用的putIndex
    items[putIndex] = x;
    if (++putIndex == items.length) 
        putIndex = 0;  //设计的精髓： 环形数组，putIndex指针到数组尽头了，返回头部
    count++;
    //notEmpty条件队列转同步队列，准备唤醒消费者线程，因为入队了一个元素，肯定不为空了
    notEmpty.signal();
}
```

​        

### **出队take方法**

```java
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    //加锁，如果线程中断抛出异常 
    lock.lockInterruptibly();
    try {
       //如果队列为空，则消费者挂起
        while (count == 0)
            notEmpty.await();
        //出队
        return dequeue();
    } finally {
        lock.unlock();// 唤醒生产者线程
    }
}
private E dequeue() {
    final Object[] items = this.items;
    @SuppressWarnings("unchecked")
    E x = (E) items[takeIndex]; //取出takeIndex位置的元素
    items[takeIndex] = null;
    if (++takeIndex == items.length)
        takeIndex = 0; //设计的精髓： 环形数组，takeIndex 指针到数组尽头了，返回头部
    count--;
    if (itrs != null)
        itrs.elementDequeued();
    //notFull条件队列转同步队列，准备唤醒生产者线程，此时队列有空位
    notFull.signal();
    return x;
}
```

思考： 为什么对数组操作要设计成双指针？

​    ![0](13%E3%80%81%E9%98%BB%E5%A1%9E%E9%98%9F%E5%88%97BlockingQueue%E5%AE%9E%E6%88%98%E5%8F%8A%E5%85%B6%E5%8E%9F%E7%90%86%E5%88%86%E6%9E%90%E4%B8%80/1835)

# **LinkedBlockingQueue**

LinkedBlockingQueue是一个基于链表实现的阻塞队列，默认情况下，该阻塞队列的大小为Integer.MAX_VALUE，由于这个数值特别大，所以 LinkedBlockingQueue 也被称作无界队列，代表它几乎没有界限，队列可以随着元素的添加而动态增长，但是如果没有剩余内存，则队列将抛出OOM错误。所以为了避免队列过大造成机器负载或者内存爆满的情况出现，我们在使用的时候建议手动传一个队列的大小。

LinkedBlockingQueue内部由单链表实现，只能从head取元素，从tail添加元素。LinkedBlockingQueue采用两把锁的锁分离技术实现入队出队互不阻塞，添加元素和获取元素都有独立的锁，也就是说LinkedBlockingQueue是读写分离的，读写操作可以并行执行。

​    ![0](13%E3%80%81%E9%98%BB%E5%A1%9E%E9%98%9F%E5%88%97BlockingQueue%E5%AE%9E%E6%88%98%E5%8F%8A%E5%85%B6%E5%8E%9F%E7%90%86%E5%88%86%E6%9E%90%E4%B8%80/1837)

## **LinkedBlockingQueue使用**

```java
//指定队列的大小创建有界队列
BlockingQueue<Integer> boundedQueue = new LinkedBlockingQueue<>(100);
//无界队列
BlockingQueue<Integer> unboundedQueue = new LinkedBlockingQueue<>();
```

## **LinkedBlockingQueue的原理**

### **数据结构**

```java
// 容量,指定容量就是有界队列
private final int capacity;
// 元素数量
private final AtomicInteger count = new AtomicInteger();
// 链表头  本身是不存储任何元素的，初始化时item指向null
transient Node<E> head;
// 链表尾
private transient Node<E> last;
// take锁   锁分离，提高效率
private final ReentrantLock takeLock = new ReentrantLock();
// notEmpty条件
// 当队列无元素时，take锁会阻塞在notEmpty条件上，等待其它线程唤醒
private final Condition notEmpty = takeLock.newCondition();
// put锁
private final ReentrantLock putLock = new  ();
// notFull条件
// 当队列满了时，put锁会会阻塞在notFull上，等待其它线程唤醒
private final Condition notFull = putLock.newCondition();

//典型的单链表结构
static class Node<E> {
    E item;  //存储元素
    Node<E> next;  //后继节点    单链表结构
    Node(E x) { item = x; }
}
```

### **构造器**

```java
public LinkedBlockingQueue() {
    // 如果没传容量，就使用最大int值初始化其容量
    this(Integer.MAX_VALUE);
}

public LinkedBlockingQueue(int capacity) {
    if (capacity <= 0) throw new IllegalArgumentException();
    this.capacity = capacity;
    // 初始化head和last指针为空值节点
    last = head = new Node<E>(null);
}
```

​       

### **入队put方法**

```java
public void put(E e) throws InterruptedException {    
    // 不允许null元素
    if (e == null) throw new NullPointerException();
    int c = -1;
    // 新建一个节点
    Node<E> node = new Node<E>(e);
    final ReentrantLock putLock = this.putLock;
    final AtomicInteger count = this.count;
    // 使用put锁加锁
    putLock.lockInterruptibly();
    try {
        // 如果队列满了，就阻塞在notFull上等待被其它线程唤醒（阻塞生产者线程）
        while (count.get() == capacity) {
            notFull.await();
        }  
        // 队列不满，就入队
        enqueue(node);
        c = count.getAndIncrement();// 队列长度加1，返回原值
        // 如果现队列长度小于容量，notFull条件队列转同步队列，准备唤醒一个阻塞在notFull条件上的线程(可以继续入队) 
        // 这里为啥要唤醒一下呢？
        // 因为可能有很多线程阻塞在notFull这个条件上,而取元素时只有取之前队列是满的才会唤醒notFull,此处不用等到取元素时才唤醒
        if (c + 1 < capacity)
            notFull.signal();
    } finally {
        putLock.unlock(); // 真正唤醒生产者线程
    }  
    // 如果原队列长度为0，现在加了一个元素后立即唤醒阻塞在notEmpty上的线程
    if (c == 0)
        signalNotEmpty();
}
private void enqueue(Node<E> node) { 
    // 直接加到last后面,last指向入队元素
    last = last.next = node;
}    
private void signalNotEmpty() {
    final ReentrantLock takeLock = this.takeLock; 
    takeLock.lock();// 加take锁
    try {  
        notEmpty.signal();// notEmpty条件队列转同步队列，准备唤醒阻塞在notEmpty上的线程
    } finally {
        takeLock.unlock();  // 真正唤醒消费者线程
    }
}
```

​       

### **出队take方法**

```java
public E take() throws InterruptedException {
    E x;
    int c = -1;
    final AtomicInteger count = this.count;
    final ReentrantLock takeLock = this.takeLock;
    // 使用takeLock加锁
    takeLock.lockInterruptibly();
    try {
        // 如果队列无元素，则阻塞在notEmpty条件上（消费者线程阻塞）
        while (count.get() == 0) {
            notEmpty.await();
        }
        // 否则，出队
        x = dequeue();
        c = count.getAndDecrement();//长度-1，返回原值
        if (c > 1)// 如果取之前队列长度大于1，notEmpty条件队列转同步队列，准备唤醒阻塞在notEmpty上的线程，原因与入队同理
            notEmpty.signal();
    } finally {
        takeLock.unlock(); // 真正唤醒消费者线程
    }
    // 为什么队列是满的才唤醒阻塞在notFull上的线程呢？
    // 因为唤醒是需要加putLock的，这是为了减少锁的次数,所以，这里索性在放完元素就检测一下，未满就唤醒其它notFull上的线程,
    // 这也是锁分离带来的代价
    // 如果取之前队列长度等于容量（已满），则唤醒阻塞在notFull的线程
    if (c == capacity)
        signalNotFull();
    return x;
}
private E dequeue() {
     // head节点本身是不存储任何元素的
    // 这里把head删除，并把head下一个节点作为新的值
    // 并把其值置空，返回原来的值
    Node<E> h = head;
    Node<E> first = h.next;
    h.next = h; // 方便GC
    head = first;
    E x = first.item;
    first.item = null;
    return x;
}
private void signalNotFull() {
    final ReentrantLock putLock = this.putLock;
    putLock.lock();
    try {
        notFull.signal();// notFull条件队列转同步队列，准备唤醒阻塞在notFull上的线程
    } finally {
        putLock.unlock(); // 解锁，这才会真正的唤醒生产者线程
    }
}
```

​     

### **LinkedBlockingQueue与ArrayBlockingQueue对比**

LinkedBlockingQueue是一个阻塞队列，内部由两个ReentrantLock来实现出入队列的线程安全，由各自的Condition对象的await和signal来实现等待和唤醒功能。它和ArrayBlockingQueue的不同点在于：

- 队列大小有所不同，ArrayBlockingQueue是有界的初始化必须指定大小，而LinkedBlockingQueue可以是有界的也可以是无界的(Integer.MAX_VALUE)，对于后者而言，当添加速度大于移除速度时，在无界的情况下，可能会造成内存溢出等问题。
- 数据存储容器不同，ArrayBlockingQueue采用的是数组作为数据存储容器，而LinkedBlockingQueue采用的则是以Node节点作为连接对象的链表。
- 由于ArrayBlockingQueue采用的是数组的存储容器，因此在插入或删除元素时不会产生或销毁任何额外的对象实例，而LinkedBlockingQueue则会生成一个额外的Node对象。这可能在长时间内需要高效并发地处理大批量数据的时，对于GC可能存在较大影响。
- 两者的实现队列添加或移除的锁不一样，ArrayBlockingQueue实现的队列中的锁是没有分离的，即添加操作和移除操作采用的同一个ReenterLock锁，而LinkedBlockingQueue实现的队列中的锁是分离的，其添加采用的是putLock，移除采用的则是takeLock，这样能大大提高队列的吞吐量，也意味着在高并发的情况下生产者和消费者可以并行地操作队列中的数据，以此来提高整个队列的并发性能。

# LinkedBlockingDeque

![image-20240319135838547](13%E3%80%81%E9%98%BB%E5%A1%9E%E9%98%9F%E5%88%97BlockingQueue%E5%AE%9E%E6%88%98%E5%8F%8A%E5%85%B6%E5%8E%9F%E7%90%86%E5%88%86%E6%9E%90%E4%B8%80/image-20240319135838547.png)

# **SynchronousQueue**

SynchronousQueue是一个没有数据缓冲的BlockingQueue，生产者线程对其的插入操作put必须等待消费者的移除操作take。

​    ![0](13%E3%80%81%E9%98%BB%E5%A1%9E%E9%98%9F%E5%88%97BlockingQueue%E5%AE%9E%E6%88%98%E5%8F%8A%E5%85%B6%E5%8E%9F%E7%90%86%E5%88%86%E6%9E%90%E4%B8%80/1838)

如图所示，SynchronousQueue 最大的不同之处在于，它的容量为 0，所以没有一个地方来暂存元素，导致每次取数据都要先阻塞，直到有数据被放入；同理，每次放数据的时候也会阻塞，直到有消费者来取。

需要注意的是，SynchronousQueue 的容量不是 1 而是 0，因为 SynchronousQueue 不需要去持有元素，它所做的就是直接传递（direct handoff）。由于每当需要传递的时候，SynchronousQueue 会把元素直接从生产者传给消费者，在此期间并不需要做存储，所以如果运用得当，它的效率是很高的。

## **应用场景**

SynchronousQueue非常适合传递性场景做交换工作，生产者的线程和消费者的线程同步传递某些信息、事件或者任务。

SynchronousQueue的一个使用场景是在线程池里。如果我们不确定来自生产者请求数量，但是这些请求需要很快的处理掉，那么配合SynchronousQueue为每个生产者请求分配一个消费线程是处理效率最高的办法。Executors.newCachedThreadPool()就使用了SynchronousQueue，这个线程池根据需要（新任务到来时）创建新的线程，如果有空闲线程则会重复使用，线程空闲了60秒后会被回收。

## **SynchronousQueue使用**

```java
BlockingQueue<Integer> synchronousQueue = new SynchronousQueue<>();
```

![0](13%E3%80%81%E9%98%BB%E5%A1%9E%E9%98%9F%E5%88%97BlockingQueue%E5%AE%9E%E6%88%98%E5%8F%8A%E5%85%B6%E5%8E%9F%E7%90%86%E5%88%86%E6%9E%90%E4%B8%80/1836)

# LinkedTransferQueue

![image-20240319140810813](13%E3%80%81%E9%98%BB%E5%A1%9E%E9%98%9F%E5%88%97BlockingQueue%E5%AE%9E%E6%88%98%E5%8F%8A%E5%85%B6%E5%8E%9F%E7%90%86%E5%88%86%E6%9E%90%E4%B8%80/image-20240319140810813.png)

# PriorityBlockingQueue

![image-20240319140829597](13%E3%80%81%E9%98%BB%E5%A1%9E%E9%98%9F%E5%88%97BlockingQueue%E5%AE%9E%E6%88%98%E5%8F%8A%E5%85%B6%E5%8E%9F%E7%90%86%E5%88%86%E6%9E%90%E4%B8%80/image-20240319140829597.png)

# **DelayQueue**

DelayQueue 是一个支持延时获取元素的阻塞队列， 内部采用优先队列 PriorityQueue 存储元素，同时元素必须实现 Delayed 接口；在创建元素时可以指定多久才可以从队列中获取当前元素，只有在延迟期满时才能从队列中提取元素。延迟队列的特点是：不是先进先出，而是会按照延迟时间的长短来排序，下一个即将执行的任务会排到队列的最前面。

它是无界队列，放入的元素必须实现 Delayed 接口，而 Delayed 接口又继承了 Comparable 接口，所以自然就拥有了比较和排序的能力，代码如下：

```java
public interface Delayed extends Comparable<Delayed> {
    //getDelay 方法返回的是“还剩下多长的延迟时间才会被执行”，
    //如果返回 0 或者负数则代表任务已过期。
    //元素会根据延迟时间的长短被放到队列的不同位置，越靠近队列头代表越早过期。
    long getDelay(TimeUnit unit);
}
```

![0](13%E3%80%81%E9%98%BB%E5%A1%9E%E9%98%9F%E5%88%97BlockingQueue%E5%AE%9E%E6%88%98%E5%8F%8A%E5%85%B6%E5%8E%9F%E7%90%86%E5%88%86%E6%9E%90%E4%B8%80/1839)

## **DelayQueue使用**

```java
DelayQueue<OrderInfo> queue = new DelayQueue<OrderInfo>();
```

## **DelayQueue的原理**

### **数据结构**

```java
//用于保证队列操作的线程安全
private final transient ReentrantLock lock = new ReentrantLock();
// 优先级队列,存储元素，用于保证延迟低的优先执行
private final PriorityQueue<E> q = new PriorityQueue<E>();
// 用于标记当前是否有线程在排队（仅用于取元素时） leader 指向的是第一个从队列获取元素阻塞的线程
private Thread leader = null;
// 条件，用于表示现在是否有可取的元素   当新元素到达，或新线程可能需要成为leader时被通知
private final Condition available = lock.newCondition();

public DelayQueue() {}
public DelayQueue(Collection<? extends E> c) {
    this.addAll(c);
}
```

​          

### **入队put方法**

```java
public void put(E e) {
    offer(e);
}
public boolean offer(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        // 入队
        q.offer(e);
        if (q.peek() == e) {
            // 若入队的元素位于队列头部，说明当前元素延迟最小
            // 将 leader 置空
            leader = null;
            // available条件队列转同步队列,准备唤醒阻塞在available上的线程
            available.signal();
        }
        return true;
    } finally {
        lock.unlock(); // 解锁，真正唤醒阻塞的线程
    }
}
```

### **出队take方法**

```java
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        for (;;) {
            E first = q.peek();// 取出堆顶元素   
            if (first == null)// 如果堆顶元素为空，说明队列中还没有元素，直接阻塞等待
                available.await();
            else {
                long delay = first.getDelay(NANOSECONDS);// 堆顶元素的到期时间             
                if (delay <= 0)// 如果小于0说明已到期，直接调用poll()方法弹出堆顶元素
                    return q.poll();
                
                // 如果delay大于0 ，则下面要阻塞了
                // 将first置为空方便gc
                first = null; 
                // 如果前面有其它线程在等待，直接进入等待
                if (leader != null)
                    available.await();
                else {
                    // 如果leader为null，把当前线程赋值给它
                    Thread thisThread = Thread.currentThread();
                    leader = thisThread;
                    try {
                        // 等待delay时间后自动醒过来
                        // 醒过来后把leader置空并重新进入循环判断堆顶元素是否到期
                        // 这里即使醒过来后也不一定能获取到元素
                        // 因为有可能其它线程先一步获取了锁并弹出了堆顶元素
                        // 条件锁的唤醒分成两步，先从Condition的队列里出队
                        // 再入队到AQS的队列中，当其它线程调用LockSupport.unpark(t)的时候才会真正唤醒
                        available.awaitNanos(delay);
                    } finally {
                        // 如果leader还是当前线程就把它置为空，让其它线程有机会获取元素
                        if (leader == thisThread)
                            leader = null;
                    }
                }
            }
        }
    } finally {
        // 成功出队后，如果leader为空且堆顶还有元素，就唤醒下一个等待的线程
        if (leader == null && q.peek() != null)
            // available条件队列转同步队列,准备唤醒阻塞在available上的线程
            available.signal();
        // 解锁，真正唤醒阻塞的线程
        lock.unlock();
    }
}
```

​      

