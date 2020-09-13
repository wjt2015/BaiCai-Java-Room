> 白菜Java自习室 涵盖核心知识

> [Java工程师的进阶之路 并发篇（一）](https://juejin.im/post/6844904192658636814)<br>
> [Java工程师的进阶之路 并发篇（二）](https://juejin.im/post/6844904192738328589)<br>
> [Java工程师的进阶之路 并发篇（三）](https://juejin.im/post/6844904193065484301)<br>
> [Java工程师的进阶之路 并发篇（四）](https://juejin.im/post/6844904193174536199)<br>
> [Java工程师的进阶之路 并发篇（五）](https://juejin.im/post/6844904193182924808)<br>

## Java并发-并发容器

### 1. Threadlocal

ThreadLocal顾名思义是线程私有的局部变量存储容器，可以理解成每个线程都有自己专属的存储容器，它用来存储线程私有变量，其实它只是一个外壳，内部真正存取是一个Map，后面会仔细讲解。每个线程可以通过set()和get()存取变量，多线程间无法访问各自的局部变量，相当于在每个线程间建立了一个隔板。只要线程处于活动状态，它所对应的ThreadLocal实例就是可访问的，线程被终止后，它的所有实例将被垃圾收集。总之记住一句话：**ThreadLocal存储的变量属于当前线程。**

#### 1.1. ThreadLocal简单使用

话不多说先看一下ThreadLocal的一个简单案例：
```
public class Test implements Runnable {
    private static AtomicInteger counter = new AtomicInteger(100);
    private static ThreadLocal<String> threadInfo = new ThreadLocal<String>() {
        @Override
        protected String initialValue() {
            return "[" + Thread.currentThread().getName() + "," + counter.getAndIncrement() + "]";
        }
    };

    @Override
    public void run() {
        System.out.println("threadInfo value:" + threadInfo.get());
    }

    public static void main(String[] args) throws InterruptedException {
        Thread thread1 = new Thread(new Test());
        Thread thread2 = new Thread(new Test());

        thread1.start();
        thread2.start();

        thread1.join();
        thread2.join();

        System.out.println("threadInfo value in main:" + threadInfo.get());
    }
}
```
输出结果：
```
threadInfo value:[Thread-0,100]
threadInfo value:[Thread-1,101]
threadInfo value in main:[main,102]
```

上述代码中我用ThreadLocal来存储线程的信息，其格式为[线程名，线程ID]，定义的变量是静态的。从运行结果可以看出来每个线程包括主线程访问到的threadInfo获取到的值都是不一样的，而且存放的信息就是本线程的信息，也应证了上面那句话ThreadLocal存储的变量属于当前线程。

#### 1.2. ThreadLocal原理

**1.2.1. ThreadLocal的存取过程**

解析原理先从源码开始，首先看一下ThreadLocal.set()方法
```
//  ThreadLocal中set方法
public void set(T value) {
    //  获取当前线程对象
    Thread t = Thread.currentThread();
    //  获取该线程的threadLocals属性（ThreadLocalMap对象）
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}

//  Thread类中的threadLocals定义
ThreadLocal.ThreadLocalMap threadLocals = null;

//  ThreadLocal中getMap方法
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}

//  ThreadLocal中createMap方法
//  为线程创建ThreadLocalMap对象并赋值给threadLocals
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```
从源码中看到在set方法里面就是把值存入ThreadLocalMap类中，这个类是属于ThreadLocal的内部类，但是在Thread类中也有定义threadLocals变量，get方法的操作对象也是ThreadLocalMap，也就是说关键的存储和获取实质上在于ThreadLocalMap类。其中是以ThreadLocal类为key，存入的值为value，而ThreadLocal又是定义在每个线程的属性中，这也就实现了“ThreadLocal线程私有化”的功能，**每次都是先从当前线程获取到threadLocals属性，也就是获得ThreadLocalMap对象，以ThreadLocal对象作为key存取对应的值**。

**1.2.2. 探究ThreadLocalMap对象**

ThreadLocalMap对象是ThreadLocal类的内部类，其中它就是一个简单的Map，每个存的值被封装成Entry进行存储，下面是Entry的源码
```
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```
Entry是ThreadLocalMap的内部类，仔细观察其源码发现，它是继承了一个ThreadLocal的弱引用。回忆一下Java中的四种引用：强引用、软引用、弱引用、幻象引用。强引用是new创建出来的对象，只要强引用存在，垃圾收集器永远不会回收该引用；软引用（SoftReference）是在内存即将被占满时被回收；弱引用（WeakReference）用来描述非必需对象的，但是它的强度比软引用更弱一些，被弱引用关联的对象只能生存到下一次GC发生之前，当垃圾收集器工作时，无论当前内存是否足够，都会回收掉该类对象；幻象引用又称虚引用或幽灵引用（Phantom Reference），它是最弱的一种引用关系。一个对象是否有虚引用的存在，完全不会对其生存时间构成影响，也无法通过虚引用来取得对象实例，任何时候都可能被回收。

**1.2.3. ThreadLocal对象的回收**

Entry是一个弱引用，是因为它不会影响到ThreadLocal的GC行为，如果是强引用的话，在线程运行过程中，我们不再使用ThreadLocal了，将ThreadLocal置为null，但ThreadLocal在线程的ThreadLocalMap里还有引用，导致其无法被GC回收。而Entry声明为WeakReference，ThreadLocal置为null后，线程的ThreadLocalMap就不算强引用了，ThreadLocal就可以被GC回收了。
```
//  ThreadLocal中的remove方法
public void remove() {
    ThreadLocalMap m = getMap(Thread.currentThread());
    if (m != null)
        m.remove(this);
}

//  ThreadLocalMap中的remove方法
private void remove(ThreadLocal<?> key) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);
    for (Entry e = tab[i];
            e != null;
            e = tab[i = nextIndex(i, len)]) {
        if (e.get() == key) {
            e.clear();
            expungeStaleEntry(i);
            return;
        }
    }
}
```
**可能存在的内存泄漏问题:**

ThreadLocalMap的生命周期是与线程一样的，但是ThreadLocal却不一定，可能ThreadLocal使用完了就想要被回收，但是此时线程可能不会立即终止，还会继续运行（比如线程池中线程重复利用），如果ThreadLocal对象只存在弱引用，那么在下一次垃圾回收的时候必然会被清理掉。

如果ThreadLocal没有被外部强引用的情况下，在垃圾回收的时候会被清理掉的，这样一来ThreadLocalMap中使用这个ThreadLocal的key也会被清理掉。但是，value 是强引用，不会被清理，这样一来就会出现key为null的value

在ThreadLocalMap中，调用 set()、get()、remove()方法的时候，会清理掉key为null的记录。在ThreadLocal设置为null之后，ThreadLocalMap中存在key为null的值，那么就可能发生内存泄漏，只有手动调用remove()方法来避免，**所以我们在使用完ThreadLocal变量时，尽量用threadLocal.remove()来清除，避免threadLocal=null的操作**。remove方法是彻底地回收该对象，而通过threadLocal=null只是释放掉了ThreadLocal的引用，但是在ThreadLocalMap中却还存在其Entry，后续还需处理。

#### 1.3. ThreadLocal应用场景

* 处理较为复杂的业务时，使用ThreadLocal代替参数的显示传递。
* ThreadLocal可以用来做数据库连接池保存Connection对象，这样就可以让线程内多次获取到的连接是同一个了（Spring中的DataSource就是使用的ThreadLocal）。
* 管理Session会话，将Session保存在ThreadLocal中，使线程处理多次处理会话时始终是一个Session。

### 2. 常见的并发容器

不考虑多线程并发的情况下，容器类一般使用ArrayList、HashMap等线程不安全的类，效率更高。在并发场景下，常会用到ConcurrentHashMap、ArrayBlockingQueue等线程安全的容器类，虽然牺牲了一些效率，但却得到了安全。

上面提到的线程安全容器都在java.util.concurrent包下，这个包下并发容器不少，下面全部翻出来介绍一下。

#### 2.1. ConcurrentHashMap：并发版HashMap

最常见的并发容器之一，可以用作并发场景下的缓存。底层依然是哈希表，但在Java 8中有了不小的改变，而Java 7和Java 8都是用的比较多的版本，因此经常会将这两个版本的实现方式做一些比较（比如面试中）。

一个比较大的差异就是，Java 7中采用分段锁来减少锁的竞争，Java 8中放弃了分段锁，采用CAS（一种乐观锁），同时为了防止哈希冲突严重时退化成链表（冲突时会在该位置生成一个链表，哈希值相同的对象就链在一起），会在链表长度达到阈值（8）后转换成红黑树（比起链表，树的查询效率更稳定）。

#### 2.2. CopyOnWriteArrayList：并发版ArrayList

并发版ArrayList，底层结构也是数组，和ArrayList不同之处在于：当新增和删除元素时会创建一个新的数组，在新的数组中增加或者排除指定对象，最后用新增数组替换原来的数组。

适用场景：由于读操作不加锁，写（增、删、改）操作加锁，因此适用于读多写少的场景。

局限场景：由于读的时候不会加锁（读的效率高，就和普通ArrayList一样），读取的当前副本，因此可能读取到脏数据。如果介意，建议不用。

#### 2.3. CopyOnWriteArraySet：并发Set

基于CopyOnWriteArrayList实现（内含一个CopyOnWriteArrayList成员变量），也就是说底层是一个数组，意味着每次add都要遍历整个集合才能知道是否存在，不存在时需要插入（加锁）。

适用场景：在CopyOnWriteArrayList适用场景下加一个，集合别太大（全部遍历伤不起）。

#### 2.4. ConcurrentLinkedQueue：并发队列(基于链表)

基于链表实现的并发队列，使用乐观锁(CAS)保证线程安全。因为数据结构是链表，所以理论上是没有队列大小限制的，也就是说添加数据一定能成功。

#### 2.5. ConcurrentLinkedDeque：并发队列(基于双向链表)

基于双向链表实现的并发队列，可以分别对头尾进行操作，因此除了先进先出(FIFO)，也可以先进后出（FILO），当然先进后出的话应该叫它栈了。

#### 2.6. ConcurrentSkipListMap：基于跳表的并发Map

SkipList即跳表，跳表是一种空间换时间的数据结构，通过冗余数据，将链表一层一层索引，达到类似二分查找的效果
![](https://user-gold-cdn.xitu.io/2020/6/17/172bdfa4fda228ee?w=600&h=196&f=jpeg&s=10261)

#### 2.7. ConcurrentSkipListSet：基于跳表的并发Set

类似HashSet和HashMap的关系，ConcurrentSkipListSet里面就是一个ConcurrentSkipListMap，就不细说了。

#### 2.8. ArrayBlockingQueue：阻塞队列(基于数组)

基于数组实现的可阻塞队列，构造时必须制定数组大小，往里面放东西时如果数组满了便会阻塞直到有位置（也支持直接返回和超时等待），通过一个锁ReentrantLock保证线程安全。

#### 2.9. LinkedBlockingQueue：阻塞队列(基于链表)

基于链表实现的阻塞队列，想比与不阻塞的ConcurrentLinkedQueue，它多了一个容量限制，如果不设置默认为int最大值。

#### 2.10. LinkedBlockingDeque：阻塞队列(基于双向链表)

类似LinkedBlockingQueue，但提供了双向链表特有的操作。

#### 2.11. PriorityBlockingQueue：线程安全的优先队列

构造时可以传入一个比较器，可以看做放进去的元素会被排序，然后读取的时候按顺序消费。某些低优先级的元素可能长期无法被消费，因为不断有更高优先级的元素进来。

#### 2.12. SynchronousQueue：读写成对的队列

一个虚假的队列，因为它实际上没有真正用于存储元素的空间，每个插入操作都必须有对应的取出操作，没取出时无法继续放入。
Java中一个使用场景就是Executors.newCachedThreadPool()，创建一个缓存线程池。

#### 2.13. LinkedTransferQueue：基于链表的数据交换队列

实现了接口TransferQueue，通过transfer方法放入元素时，如果发现有线程在阻塞在取元素，会直接把这个元素给等待线程。如果没有人等着消费，那么会把这个元素放到队列尾部，并且此方法阻塞直到有人读取这个元素。和SynchronousQueue有点像，但比它更强大。

#### 2.14. DelayQueue：延时队列

可以使放入队列的元素在指定的延时后才被消费者取出，元素需要实现Delayed接口。

> [Java工程师的进阶之路 并发篇（一）](https://juejin.im/post/6844904192658636814)<br>
> [Java工程师的进阶之路 并发篇（二）](https://juejin.im/post/6844904192738328589)<br>
> [Java工程师的进阶之路 并发篇（三）](https://juejin.im/post/6844904193065484301)<br>
> [Java工程师的进阶之路 并发篇（四）](https://juejin.im/post/6844904193174536199)<br>
> [Java工程师的进阶之路 并发篇（五）](https://juejin.im/post/6844904193182924808)<br>