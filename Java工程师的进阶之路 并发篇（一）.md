> 白菜Java自习室 涵盖核心知识

> [Java工程师的进阶之路 并发篇（一）](https://juejin.im/post/6844904192658636814)<br>
> [Java工程师的进阶之路 并发篇（二）](https://juejin.im/post/6844904192738328589)<br>
> [Java工程师的进阶之路 并发篇（三）](https://juejin.im/post/6844904193065484301)<br>
> [Java工程师的进阶之路 并发篇（四）](https://juejin.im/post/6844904193174536199)<br>
> [Java工程师的进阶之路 并发篇（五）](https://juejin.im/post/6844904193182924808)<br>

## Java并发-线程池

### 1. 线程池的优势

1. 降低系统资源消耗，通过重用已存在的线程，降低线程创建和销毁造成的消耗；
2. 提高系统响应速度，当有任务到达时，通过复用已存在的线程，无需等待新线程的创建便能立即执行；
3. 方便线程并发数的管控。因为线程若是无限制的创建，可能会导致内存占用过多而产生OOM，并且会造成cpu过度切换（cpu切换线程是有时间成本的（需要保持当前执行线程的现场，并恢复要执行线程的现场））。
4. 提供更强大的功能，延时定时线程池。

### 2. 线程池的主要参数

```
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler)
```

1. **corePoolSize**（线程池基本大小）：当向线程池提交一个任务时，若线程池已创建的线程数小于corePoolSize，即便此时存在空闲线程，也会通过创建一个新线程来执行该任务，直到已创建的线程数大于或等于corePoolSize时，（除了利用提交新任务来创建和启动线程（按需构造），也可以通过 prestartCoreThread() 或 prestartAllCoreThreads() 方法来提前启动线程池中的基本线程。）
2. **maximumPoolSize**（线程池最大大小）：线程池所允许的最大线程个数。当队列满了，且已创建的线程数小于maximumPoolSize，则线程池会创建新的线程来执行任务。另外，对于无界队列，可忽略该参数。
3. **keepAliveTime**（线程存活保持时间）：当线程池中线程数大于核心线程数时，线程的空闲时间如果超过线程存活时间，那么这个线程就会被销毁，直到线程池中的线程数小于等于核心线程数。
4. **workQueue**（任务队列）：用于传输和保存等待执行任务的阻塞队列。
5. **threadFactory**（线程工厂）：用于创建新线程。threadFactory创建的线程也是采用new Thread()方式，threadFactory创建的线程名都具有统一的风格：pool-m-thread-n（m为线程池的编号，n为线程池内的线程编号）。
6. **handler**（线程饱和策略）：当线程池和队列都满了，再加入线程会执行此策略。

### 3. 线程池的执行流程

![](https://user-gold-cdn.xitu.io/2020/6/16/172bc5b53c2a4d5b?w=937&h=348&f=webp&s=16018)

1. 判断核心线程池是否已满，没满则创建一个新的工作线程来执行任务。已满则。
2. 判断任务队列是否已满，没满则将新提交的任务添加在工作队列，已满则。
3. 判断整个线程池是否已满，没满则创建一个新的工作线程来执行任务，已满则执行饱和策略。

### 4. 线程池的阻塞队列

1. 线程若是无限制的创建，可能会导致内存占用过多而产生OOM，并且会造成cpu过度切换。
2. 线程池创建线程需要获取mainlock这个全局锁，影响并发效率，阻塞队列可以很好的缓冲。
3. 阻塞队列可以保证任务队列中没有任务时阻塞获取任务的线程，使得线程进入wait状态，释放cpu资源。

### 5. 线程池的饱和策略

1. **AbortPolicy**：抛出 RejectedExecutionException来拒绝新任务的处理。
2. **CallerRunsPolicy**：调用执行自己的线程运行任务，也就是直接在调用execute方法的线程中运行(run)被拒绝的任务，如果执行程序已关闭，则会丢弃该任务。因此这种策略会降低对于新任务提交速度，影响程序的整体性能。如果您的应用程序可以承受此延迟并且你要求任何一个任务请求都要被执行的话，你可以选择这个策略。
3. **DiscardPolicy**： 不处理新任务，直接丢弃掉。
4. **DiscardOldestPolicy**： 此策略将丢弃最早的未处理的任务请求。

### 5. 线程池的配置选择

1. **CPU密集型任务**：
尽量使用较小的线程池，一般为CPU核心数+1。 因为CPU密集型任务使得CPU使用率很高，若开过多的线程数，会造成CPU过度切换。
2. **IO密集型任务**：
可以使用稍大的线程池，一般为2*CPU核心数。 IO密集型任务CPU使用率并不高，因此可以让CPU在等待IO的时候有其他线程去处理别的任务，充分利用CPU时间。
3. **混合型任务**：
可以将任务分成IO密集型和CPU密集型任务，然后分别用不同的线程池去处理。 只要分完之后两个任务的执行时间相差不大，那么就会比串行执行来的高效。

### 6. Java提供的线程池

1. **newCachedThreadPool**：用来创建一个可以无限扩大的线程池，适用于负载较轻的场景，执行短期异步任务。（可以使得任务快速得到执行，因为任务时间执行短，可以很快结束，也不会造成cpu过度切换）
2. **newFixedThreadPool**：创建一个固定大小的线程池，因为采用无界的阻塞队列，所以实际线程数量永远不会变化，适用于负载较重的场景，对当前线程数量进行限制。（保证线程数可控，不会造成线程过多，导致系统负载更为严重）
3. **newSingleThreadExecutor**：创建一个单线程的线程池，适用于需要保证顺序执行各个任务。
4. **newScheduledThreadPool**：适用于执行延时或者周期性任务。

### 7. execute()和submit()方法

1. **execute()**，执行一个任务，没有返回值。
2. **submit()**，提交一个线程任务，有返回值。

submit(Callable<T> task)能获取到它的返回值，通过future.get()获取（阻塞直到任务执行完）。一般使用FutureTask+Callable配合使用（IntentService中有体现）。<br>
submit(Runnable task, T result)能通过传入的载体result间接获得线程的返回值。
submit(Runnable task)则是没有返回值的，就算获取它的返回值也是null。

Future.get方法会使取结果的线程进入阻塞状态，知道线程执行完成之后，唤醒取结果的线程，然后返回结果。

> [Java工程师的进阶之路 并发篇（一）](https://juejin.im/post/6844904192658636814)<br>
> [Java工程师的进阶之路 并发篇（二）](https://juejin.im/post/6844904192738328589)<br>
> [Java工程师的进阶之路 并发篇（三）](https://juejin.im/post/6844904193065484301)<br>
> [Java工程师的进阶之路 并发篇（四）](https://juejin.im/post/6844904193174536199)<br>
> [Java工程师的进阶之路 并发篇（五）](https://juejin.im/post/6844904193182924808)<br>