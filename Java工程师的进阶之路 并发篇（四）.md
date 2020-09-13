> 白菜Java自习室 涵盖核心知识

> [Java工程师的进阶之路 并发篇（一）](https://juejin.im/post/6844904192658636814)<br>
> [Java工程师的进阶之路 并发篇（二）](https://juejin.im/post/6844904192738328589)<br>
> [Java工程师的进阶之路 并发篇（三）](https://juejin.im/post/6844904193065484301)<br>
> [Java工程师的进阶之路 并发篇（四）](https://juejin.im/post/6844904193174536199)<br>
> [Java工程师的进阶之路 并发篇（五）](https://juejin.im/post/6844904193182924808)<br>

## Java并发-并发工具

### 1. AQS

所谓AQS，指的是AbstractQueuedSynchronizer，它提供了一种实现阻塞锁和一系列依赖FIFO等待队列的同步器的框架，ReentrantLock、Semaphore、CountDownLatch、CyclicBarrier等并发类均是基于AQS来实现的，具体用法是通过继承AQS实现其模板方法，然后将子类作为同步组件的内部类。

AQS基本框架如下图所示：
![](https://user-gold-cdn.xitu.io/2020/6/16/172bdb112fa03473?w=794&h=356&f=webp&s=7624)

AQS维护了一个volatile语义(支持多线程下的可见性)的共享资源变量state和一个FIFO线程等待队列(多线程竞争state被阻塞时会进入此队列)。

#### 1.1. State

首先说一下共享资源变量state，它是int数据类型的，其访问方式有3种：

1. getState()
2. setState(int newState)
3. compareAndSetState(int expect, int update)

上述3种方式均是原子操作，其中compareAndSetState()的实现依赖于Unsafe的compareAndSwapInt()方法。
```
private volatile int state;

// 具有内存读可见性语义
protected final int getState() {
    return state;
}

// 具有内存写可见性语义
protected final void setState(int newState) {
    state = newState;
}

// 具有内存读/写可见性语义
protected final boolean compareAndSetState(int expect, int update) {
    // See below for intrinsics setup to support this
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
```
资源的共享方式分为2种：

1. 独占式(Exclusive)
只有单个线程能够成功获取资源并执行，如ReentrantLock。

2. 共享式(Shared)
多个线程可成功获取资源并执行，如Semaphore/CountDownLatch等。

AQS将大部分的同步逻辑均已经实现好，继承的自定义同步器只需要实现state的获取(acquire)和释放(release)的逻辑代码就可以，主要包括下面方法：

* `tryAcquire(int)`：独占方式。尝试获取资源，成功则返回true，失败则返回false。</br>
* `tryRelease(int)`：独占方式。尝试释放资源，成功则返回true，失败则返回false。</br>
* `tryAcquireShared(int)`：共享方式。尝试获取资源。负数表示失败；0表示成功，但没有剩余可用资源；正数表示成功，且有剩余资源。</br>
* `tryReleaseShared(int)`：共享方式。尝试释放资源，如果释放后允许唤醒后续等待结点返回true，否则返回false。</br>
* `isHeldExclusively()`：该线程是否正在独占资源。只有用到condition才需要去实现它。</br>

AQS需要子类复写的方法均没有声明为abstract，目的是避免子类需要强制性覆写多个方法，因为一般自定义同步器要么是独占方法，要么是共享方法，只需实现tryAcquire-tryRelease、tryAcquireShared-tryReleaseShared中的一种即可。

当然，AQS也支持子类同时实现独占和共享两种模式，如ReentrantReadWriteLock。

#### 1.2. CLH队列(FIFO)

AQS是通过内部类Node来实现FIFO队列的，源代码解析如下：
```
static final class Node {
    
    // 表明节点在共享模式下等待的标记
    static final Node SHARED = new Node();
    // 表明节点在独占模式下等待的标记
    static final Node EXCLUSIVE = null;

    // 表征等待线程已取消的
    static final int CANCELLED =  1;
    // 表征需要唤醒后续线程
    static final int SIGNAL    = -1;
    // 表征线程正在等待触发条件(condition)
    static final int CONDITION = -2;
    // 表征下一个acquireShared应无条件传播
    static final int PROPAGATE = -3;

    /**
     *   SIGNAL: 当前节点释放state或者取消后，将通知后续节点竞争state。
     *   CANCELLED: 线程因timeout和interrupt而放弃竞争state，当前节点将与state彻底拜拜
     *   CONDITION: 表征当前节点处于条件队列中，它将不能用作同步队列节点，直到其waitStatus被重置为0
     *   PROPAGATE: 表征下一个acquireShared应无条件传播
     *   0: None of the above
     */
    volatile int waitStatus;
    
    // 前继节点
    volatile Node prev;
    // 后继节点
    volatile Node next;
    // 持有的线程
    volatile Thread thread;
    // 链接下一个等待条件触发的节点
    Node nextWaiter;

    // 返回节点是否处于Shared状态下
    final boolean isShared() {
        return nextWaiter == SHARED;
    }

    // 返回前继节点
    final Node predecessor() throws NullPointerException {
        Node p = prev;
        if (p == null)
            throw new NullPointerException();
        else
            return p;
    }
    
    // Shared模式下的Node构造函数
    Node() {  
    }

    // 用于addWaiter
    Node(Thread thread, Node mode) {  
        this.nextWaiter = mode;
        this.thread = thread;
    }
    
    // 用于Condition
    Node(Thread thread, int waitStatus) {
        this.waitStatus = waitStatus;
        this.thread = thread;
    }
}
```

可以看到，waitStatus非负的时候，表征不可用，正数代表处于等待状态，所以waitStatus只需要检查其正负符号即可，不用太多关注特定值。

### 2. CountDownLatch

CountDownLatch使一个线程等待其他线程各自执行完毕后再执行。
是通过一个计数器来实现的，计数器的初始值是线程的数量。每当一个线程执行完毕后，计数器的值就-1，当计数器的值为0时，表示所有线程都执行完毕，然后在闭锁上等待的线程就可以恢复工作了。

构造器：
```
public CountDownLatch(int count) { };  
```

重要方法：
```
public void await() throws InterruptedException { };   
public boolean await(long timeout, TimeUnit unit) throws InterruptedException { };  
public void countDown() { };  
```

模拟并发示例：
```
public class UseCountDownLatch {

    public static void main(String[] args) {
        CountDownLatch countDownLatch = new CountDownLatch(2);
        Thread t1 = new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println("进入t1线程。。。");
                try {
                    TimeUnit.SECONDS.sleep(3);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

                System.out.println("t1线程初始化完毕，通知t3线程继续操作！");
                countDownLatch.countDown();
            }
        }, "t1");

        Thread t2 = new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println("进入t2线程。。。");
                try {
                    TimeUnit.SECONDS.sleep(4);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

                System.out.println("t2线程初始化完毕，通知t3线程继续操作！");
                countDownLatch.countDown();
            }
        }, "t2");

        Thread t3 = new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println("进入t3 线程，并且等待...");
                try {
                    countDownLatch.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

                System.out.println("t3线程进行后续的执行操作...");
            }
        }, "t3");

        t1.start();
        t2.start();
        t3.start();
    }
}
```
```
进入t1线程。。。
进入t3 线程，并且等待...
进入t2线程。。。
t1线程初始化完毕，通知t3线程继续操作！
t2线程初始化完毕，通知t3线程继续操作！
t3线程进行后续的执行操作...
```

### 3. CyclicBarrier

CyclicBarrier允许一组线程在到达某个栅栏点(common barrier point)互相等待，直到最后一个线程到达栅栏点，栅栏才会打开，处于阻塞状态的线程恢复继续执行。

构造器：
```
public CyclicBarrier(int parties)
public CyclicBarrier(int parties, Runnable barrierAction) 
```

重要方法：
```
public int await() throws InterruptedException, BrokenBarrierException
public int await(long timeout, TimeUnit unit) throws InterruptedException, BrokenBarrierException, TimeoutException 
```

模拟并发示例：
```
public class UseCyclicBarrier {

    // 模拟运动员
    static class Runner implements Runnable {

        private String name;

        private CyclicBarrier cyclicBarrier;

        @Override
        public void run() {
            try {
                System.out.println("运动员：" + this.name + "进行准备工作！");
                TimeUnit.SECONDS.sleep((new Random().nextInt(5)));
                System.out.println("运动员：" + this.name + "准备完成！");
                this.cyclicBarrier.await();
            } catch (Exception e) {
                e.printStackTrace();
            }

            System.out.println("运动员" + this.name + "开始起跑！！！");
        }

        public Runner(String name, CyclicBarrier cyclicBarrier) {
            this.name = name;
            this.cyclicBarrier = cyclicBarrier;
        }
    }

    public static void main(String[] args) {
        CyclicBarrier cyclicBarrier = new CyclicBarrier(3);
        ExecutorService executorPools = Executors.newFixedThreadPool(3);

        executorPools.submit(new Thread(new Runner("张三", cyclicBarrier)));
        executorPools.submit(new Thread(new Runner("李四", cyclicBarrier)));
        executorPools.submit(new Thread(new Runner("王五", cyclicBarrier)));

        executorPools.shutdown();
    }
}
```
```
运动员：张三进行准备工作！
运动员：李四进行准备工作！
运动员：王五进行准备工作！
运动员：张三准备完成！
运动员：王五准备完成！
运动员：李四准备完成！
运动员李四开始起跑！！！
运动员张三开始起跑！！！
运动员王五开始起跑！！！
```

> **CountDownLatch和CyclicBarrier区别**：
> 1. CountDownLatch简单的说就是一个线程等待，直到他所等待的其他线程都执行完成并且调用countDown()方法发出通知后，当前线程才可以继续执行。
> 2. CyclicBarrier是所有线程都进行等待，直到所有线程都准备好进入await()方法之后，所有线程同时开始执行。
> 3. CountDownLatch的计数器只能使用一次。而CyclicBarrier的计数器可以使用reset() 方法重置。所以CyclicBarrier能处理更为复杂的业务场景，比如如果计算发生错误，可以重置计数器，并让线程们重新执行一次。

### 4. Semaphore

Semaphore（信号量）是用来控制同时访问特定资源的线程数量，它通过协调各个线程，以保证合理的使用公共资源。Semaphore 跟锁（synchronized、Lock）有点相似，不同的地方是，锁同一时刻只允许一个线程访问某一资源，而 Semaphore 则可以控制同一时刻多个线程访问某一资源。

信号量模型比较简单，可以概括为：一个计数器、一个队列、三个方法。

**计数器**：记录当前还可以运行多少个资源访问资源。

**队列**：待访问资源的线程。

**三个方法**：

* `init()`：初始化计数器的值，可就是允许多少线程同时访问资源。
* `up()`：计数器加1，有线程归还资源时，如果计数器的值大于或者等于 0 时，从等待队列中唤醒一个线程
* `down()`：计数器减 1，有线程占用资源时，如果此时计数器的值小于 0 ，线程将被阻塞。

模拟并发示例：
```
public class UseSemaphore {

    public static void main(String[] args) {
        ExecutorService executorService = Executors.newCachedThreadPool();
        
        // 信号量，只允许 3个线程同时访问
        Semaphore semaphore = new Semaphore(3);

        for (int i = 0; i < 10; i++){
            final long num = i;
            executorService.submit(new Runnable() {
                @Override
                public void run() {
                    try {
                        // 获取许可
                        semaphore.acquire();
                        // 执行
                        System.out.println("Accessing: " + num);
                        // 模拟随机执行时长
                        Thread.sleep(new Random().nextInt(5000)); 
                        // 释放
                        semaphore.release();
                        System.out.println("Release..." + num);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            });
        }

        executorService.shutdown();
    }
```

### 5. Exchanger

Exchanger 是 JDK 1.5 开始提供的一个用于两个工作线程之间交换数据的封装工具类，简单说就是一个线程在完成一定的事务后想与另一个线程交换数据，则第一个先拿出数据的线程会一直等待第二个线程，直到第二个线程拿着数据到来时才能彼此交换对应数据。

Exchanger定义为 `Exchanger<V>` 泛型类型，其中 V 表示可交换的数据类型，对外提供的接口很简单，具体如下：

* `Exchanger()`：无参构造方法。
* `V exchange(V v)`：等待另一个线程到达此交换点（除非当前线程被中断），然后将给定的对象传送给该线程，并接收该线程的对象。
* `V exchange(V v, long timeout, TimeUnit unit)`：等待另一个线程到达此交换点（除非当前线程被中断或超出了指定的等待时间），然后将给定的对象传送给该线程，并接收该线程的对象。

当一个线程到达 exchange 调用点时，如果其他线程此前已经调用了此方法，则其他线程会被调度唤醒并与之进行对象交换，然后各自返回；如果其他线程还没到达交换点，则当前线程会被挂起，直至其他线程到达才会完成交换并正常返回，或者当前线程被中断或超时返回。

模拟并发示例：
```
public class UseExchanger {

    static class Producer extends Thread {
        private Exchanger<Integer> exchanger;
        private static int data = 0;
        Producer(String name, Exchanger<Integer> exchanger) {
            super("Producer-" + name);
            this.exchanger = exchanger;
        }

        @Override
        public void run() {
            for (int i=1; i<5; i++) {
                try {
                    TimeUnit.SECONDS.sleep(1);
                    data = i;
                    System.out.println(getName()+" 交换前:" + data);
                    data = exchanger.exchange(data);
                    System.out.println(getName()+" 交换后:" + data);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    static class Consumer extends Thread {
        private Exchanger<Integer> exchanger;
        private static int data = 0;
        Consumer(String name, Exchanger<Integer> exchanger) {
            super("Consumer-" + name);
            this.exchanger = exchanger;
        }

        @Override
        public void run() {
            while (true) {
                data = 0;
                System.out.println(getName()+" 交换前:" + data);
                try {
                    TimeUnit.SECONDS.sleep(1);
                    data = exchanger.exchange(data);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(getName()+" 交换后:" + data);
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        Exchanger<Integer> exchanger = new Exchanger<Integer>();
        new Producer("", exchanger).start();
        new Consumer("", exchanger).start();
        TimeUnit.SECONDS.sleep(7);
        System.exit(-1);
    }
}
```
```
Consumer- 交换前:0
Producer- 交换前:1
Consumer- 交换后:1
Consumer- 交换前:0
Producer- 交换后:0
Producer- 交换前:2
Producer- 交换后:0
Consumer- 交换后:2
Consumer- 交换前:0
Producer- 交换前:3
Producer- 交换后:0
Consumer- 交换后:3
Consumer- 交换前:0
Producer- 交换前:4
Producer- 交换后:0
Consumer- 交换后:4
Consumer- 交换前:0
```

> [Java工程师的进阶之路 并发篇（一）](https://juejin.im/post/6844904192658636814)<br>
> [Java工程师的进阶之路 并发篇（二）](https://juejin.im/post/6844904192738328589)<br>
> [Java工程师的进阶之路 并发篇（三）](https://juejin.im/post/6844904193065484301)<br>
> [Java工程师的进阶之路 并发篇（四）](https://juejin.im/post/6844904193174536199)<br>
> [Java工程师的进阶之路 并发篇（五）](https://juejin.im/post/6844904193182924808)<br>