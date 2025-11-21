---
title: 10、Semaphorer&CountDownLatch详解
date: 2023-06-16 23:37:57
categories:
    - 系统架构
    - 架构之并发编程
tags:
    - 并发编程
    - JUC
---

# **Semaphore介绍**

Semaphore，俗称信号量，它是操作系统中PV操作的原语在java的实现，它也是基于AbstractQueuedSynchronizer实现的。

Semaphore的功能非常强大，大小为1的信号量就类似于互斥锁，通过同时只能有一个线程获取信号量实现。大小为n（n>0）的信号量可以实现限流的功能，它可以实现只能有n个线程同时获取信号量。

​    ![0](10%E3%80%81Semaphorer-CountDownLatch%E8%AF%A6%E8%A7%A3/1731)

> <font color="red">PV操作是操作系统一种实现进程互斥与同步的有效方法。</font>PV操作与信号量（S）的处理相关，P表示通过的意思，V表示释放的意思。用PV操作来管理共享资源时，首先要确保PV操作自身执行的正确性。
>
> P操作的主要动作是：
>
> ①S减1；
>
> ②若S减1后仍大于或等于0，则进程继续执行；
>
> ③若S减1后小于0，则该进程被阻塞后放入等待该信号量的等待队列中，然后转进程调度。
>
> V操作的主要动作是：
>
> ①S加1； 
>
> ②若相加后结果大于0，则进程继续执行；
>
> ③若相加后结果小于或等于0，则从该信号的等待队列中释放一个等待进程，然后再返回原进程继续执行或转进程调度。

## **Semaphore 常用方法**

### **构造器**

​    ![0](10%E3%80%81Semaphorer-CountDownLatch%E8%AF%A6%E8%A7%A3/1730)

- permits 表示许可证的数量（资源数）
- fair 表示公平性，如果这个设为 true 的话，下次执行的线程会是等待最久的线程

### **常用方法**

```java
public void acquire() throws InterruptedException //表示阻塞并获取许可
public boolean tryAcquire() //方法在没有许可的情况下会立即返回 false，要获取许可的线程不会阻塞
public void release() //释放许可
public int availablePermits() //返回此信号量中当前可用的许可证数。
public final int getQueueLength() //返回正在等待获取许可证的线程数。
public final boolean hasQueuedThreads() //是否有线程正在等待获取许可证。
protected void reducePermits(int reduction) //减少 reduction 个许可证
protected Collection<Thread> getQueuedThreads() //返回所有等待获取许可证的线程集合
```

## **应用场景**

可以用于做流量控制，特别是公用资源有限的应用场景

### **限流**

```java
public class SemaphoneTest2 {

    /**
     * 实现一个同时只能处理5个请求的限流器
     */
    private static Semaphore semaphore = new Semaphore(5);

    /**
     * 定义一个线程池
     */
    private static ThreadPoolExecutor executor = new ThreadPoolExecutor(10, 50, 60, TimeUnit.SECONDS, new LinkedBlockingDeque<>(200));

    /**
     * 模拟执行方法
     */
    public static void exec() {
        try {
            semaphore.acquire(1);
            // 模拟真实方法执行
            System.out.println("执行exec方法" );
            Thread.sleep(2000);
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            semaphore.release(1);
        }
    }

    public static void main(String[] args) throws InterruptedException {
        {
            for (; ; ) {
                Thread.sleep(100);
                // 模拟请求以10个/s的速度
                executor.execute(() -> exec());
            }
        }
    }
}
```

​      

## **Semaphore源码分析**

建议： 先跟上节课ReentrantLock的源码，再来跟Semaphore的源码

关注点：

1. Semaphore的加锁解锁（共享锁）逻辑实现
2.  线程竞争锁失败入队阻塞逻辑和获取锁的线程释放锁唤醒阻塞线程竞争锁的逻辑实现

https://www.processon.com/view/link/61950f6e5653bb30803c5bd2

​    ![0](10%E3%80%81Semaphorer-CountDownLatch%E8%AF%A6%E8%A7%A3/1741)

# **CountDownLatch介绍**

<font color="red">CountDownLatch（闭锁）是一个同步协助类，允许一个或多个线程等待，直到其他线程完成操作集。</font>

CountDownLatch使用给定的计数值（count）初始化。<font color="red">await方法会阻塞直到当前的计数值（count）由于countDown方法的调用达到0，count为0之后所有等待的线程都会被释放，并且随后对await方法的调用都会立即返回。这是一个一次性现象 —— count不会被重置。</font>如果你需要一个重置count的版本，那么请考虑使用CyclicBarrier。

​    ![0](10%E3%80%81Semaphorer-CountDownLatch%E8%AF%A6%E8%A7%A3/1729)

## **CountDownLatch的使用**

### **构造器**

​    ![0](10%E3%80%81Semaphorer-CountDownLatch%E8%AF%A6%E8%A7%A3/1733)

### **常用方法**

```java
 // 调用 await() 方法的线程会被挂起，它会等待直到 count 值为 0 才继续执行
public void await() throws InterruptedException { };  
// 和 await() 类似，若等待 timeout 时长后，count 值还是没有变为 0，不再等待，继续执行
public boolean await(long timeout, TimeUnit unit) throws InterruptedException { };  
// 会将 count 减 1，直至为 0
```

## **CountDownLatch应用场景**

CountDownLatch一般用作多线程倒计时计数器，强制它们等待其他一组（CountDownLatch的初始化决定）任务执行完成。

CountDownLatch的两种使用场景：

- 场景1：让多个线程等待
- 场景2：让单个线程等待。

### **场景1 让多个线程等待：模拟并发，让并发线程一起执行**

```java
public class CountDownLatchTest {
    
    public static void main(String[] args) throws InterruptedException {
        CountDownLatch countDownLatch = new CountDownLatch(1);
        for (int i = 0; i < 5; i++) {
            new Thread(() -> {
                try {
                    //准备完毕……运动员都阻塞在这，等待号令
                    countDownLatch.await();
                    String parter = "【" + Thread.currentThread().getName() + "】";
                    System.out.println(parter + "开始执行……");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }).start();
        }

        Thread.sleep(2000);// 裁判准备发令
        countDownLatch.countDown();// 发令枪：执行发令
    }
}
```

​           

### **场景2 让单个线程等待：多个线程(任务)完成后，进行汇总合并**

很多时候，我们的并发任务，存在前后依赖关系；比如数据详情页需要同时调用多个接口获取数据，并发请求获取到数据后、需要进行结果合并；或者多个数据操作完成后，需要数据check；这其实都是：在多个线程(任务)完成后，进行汇总合并的场景。

```java
public class CountDownLatchTest2 {
    public static void main(String[] args) throws Exception {

        CountDownLatch countDownLatch = new CountDownLatch(5);
        for (int i = 0; i < 5; i++) {
            final int index = i;
            new Thread(() -> {
                try {
                    Thread.sleep(1000 + ThreadLocalRandom.current().nextInt(1000));
                    System.out.println(Thread.currentThread().getName()+" finish task" + index );

                    countDownLatch.countDown();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }).start();
        }

        // 主线程在阻塞，当计数器==0，就唤醒主线程往下执行。
        countDownLatch.await();
        System.out.println("主线程:在所有任务运行完成后，进行结果汇总");

    }
}
```

​          

## **CountDownLatch实现原理**

底层基于 AbstractQueuedSynchronizer 实现，CountDownLatch 构造函数中指定的count直接赋给AQS的state；<font color="red">每次countDown()则都是release(1)减1，最后减到0时unpark阻塞线程；这一步是由最后一个执行countdown方法的线程执行的。</font>

而调用await()方法时，当前线程就会判断state属性是否为0，如果为0，则继续往下执行，如果不为0，则使当前线程进入等待状态，直到某个线程将state属性置为0，其就会唤醒在await()方法中等待的线程。

### **CountDownLatch与Thread.join的区别**

- CountDownLatch的作用就是<font color="red">允许一个或多个线程等待其他线程完成操作</font>，看起来有点类似join() 方法，但其提供了比 join() 更加灵活的API。
- CountDownLatch可以<font color="red">手动控制在n个线程里调用n次countDown()方法使计数器进行减一操作，也可以在一个线程里调用n次执行减一操作。</font>
- 而<font color="red"> join() 的实现原理是不停检查join线程是否存活，如果 join 线程存活则让当前线程永远等待。</font>所以两者之间相对来说还是CountDownLatch使用起来较为灵活。

### **CountDownLatch与CyclicBarrier的区别**

CountDownLatch和CyclicBarrier都能够实现线程之间的等待，只不过它们侧重点不同：

1. <font color="red">CountDownLatch的计数器只能使用一次，而CyclicBarrier的计数器可以使用reset() 方法重置。</font>所以CyclicBarrier能处理更为复杂的业务场景，比如如果计算发生错误，可以重置计数器，并让线程们重新执行一次
2. CyclicBarrier还提供getNumberWaiting(可以获得CyclicBarrier阻塞的线程数量)、isBroken(用来知道阻塞的线程是否被中断)等方法。
3. CountDownLatch会阻塞主线程，CyclicBarrier不会阻塞主线程，只会阻塞子线程。
4. CountDownLatch和CyclicBarrier都能够实现线程之间的等待，只不过它们侧重点不同。CountDownLatch一般用于一个或多个线程，等待其他线程执行完任务后，再执行。CyclicBarrier一般用于一组线程互相等待至某个状态，然后这一组线程再同时执行。
5. CyclicBarrier 还可以提供一个 barrierAction，合并多线程计算结果。
6. <font color="red">CyclicBarrier是通过ReentrantLock的"独占锁"和Conditon来实现一组线程的阻塞唤醒的，而CountDownLatch则是通过AQS的“共享锁”实现</font>

