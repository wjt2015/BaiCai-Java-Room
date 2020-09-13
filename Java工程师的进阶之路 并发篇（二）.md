> 白菜Java自习室 涵盖核心知识

> [Java工程师的进阶之路 并发篇（一）](https://juejin.im/post/6844904192658636814)<br>
> [Java工程师的进阶之路 并发篇（二）](https://juejin.im/post/6844904192738328589)<br>
> [Java工程师的进阶之路 并发篇（三）](https://juejin.im/post/6844904193065484301)<br>
> [Java工程师的进阶之路 并发篇（四）](https://juejin.im/post/6844904193174536199)<br>
> [Java工程师的进阶之路 并发篇（五）](https://juejin.im/post/6844904193182924808)<br>

## Java并发-Executor

Java的线程既是工作单元，也是执行机制。从JDK 5开始，把工作单元与执行机制分离开来。工作单元包括Runnable和Callable，而执行机制由Executor框架提供。

在上层，Java多线程程序通常把应用分解为若干个任务，然后使用用户级的调度器（Executor框架）将这些任务映射为固定数量的线程；在底层，操作系统内核将这些线程映射到硬件处理器上。

### 1. Executor框架的结构

**任务**：包括被执行任务需要实现的接口：Runnable接口或Callable接口。

**任务的执行**：包括任务执行机制的核心接口Executor，以及继承自Executor的ExecutorService接口。Executor框架有两个关键类实现了ExecutorService接口（ThreadPoolExecutor和ScheduledThreadPoolExecutor）。

**异步计算的结果**：包括接口Future和实现Future接口的FutureTask类。

> 通过execute()方法提交的任务都是没有返回值类型的，通过submit()提交的任务的返回值类型是Future,通过Future获取线程执行结果。

Executor是一个接口，它是Executor框架的基础，它将任务的提交与任务的执行分离开来。

ThreadPoolExecutor是线程池的核心实现类，用来执行被提交的任务。

ScheduledThreadPoolExecutor是一个实现类，可以在给定的延迟后运行命令，或者定期执行命令。ScheduledThreadPoolExecutor比Timer更灵活，功能更强大。

Future接口和实现Future接口的FutureTask类，代表异步计算的结果。

Runnable接口和Callable接口的实现类，都可以被ThreadPoolExecutor或ScheduledThreadPoolExecutor执行。

### 2. Executor类和接口示意图

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/750b927dd9a746d39c48f9d02f2febbb~tplv-k3u1fbpfcp-zoom-1.image)

### 3. Executor框架的使用示意图

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2b6b1fea3aed47089986250c5c593421~tplv-k3u1fbpfcp-zoom-1.image)

### 4. ThreadPoolExecutor

ThreadPoolExecutor通常使用工厂类Executors来创建。

Executors可以创建3种类型的ThreadPoolExecutor：SingleThreadExecutor、FixedThreadPool和CachedThreadPool。

**SingleThreadExecutor**：
适用于需要保证顺序地执行各个任务；并且在任意时间点，不会有多个线程是活动的应用场景。

```
public static ExecutorService newSingleThreadExecutor()

public static ExecutorService newSingleThreadExecutor(ThreadFactory threadFactory)
```

**FixedThreadPool**：
适用于为了满足资源管理的需求，而需要限制当前线程数量的应用场景，它适用于负载比较重的服务器。

```
public static ExecutorService newFixedThreadPool(int nThreads)

public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactoty)
```

**CachedThreadPool**：
是大小无界的线程池，适用于执行很多的短期异步任务的小程序，或者是负载较轻的服务器。

```
public static ExecutorService newCachedThreadPool()

public static ExecutorService newCachedThreadPool(ThreadFactory threadFactory)
```

### 5. ScheduledThreadPoolExecutor

ScheduledThreadPoolExecutor：通常用来创建定时线程任务的线程池，例如定时轮询数据库中的表的数据。

ScheduledThreadPoolExecutor通常使用工厂类Executors来创建。Executors可以创建2种类型的ScheduledThreadPoolExecutor，如下：

**ScheduledThreadPoolExecutor**：
适用于需要多个后台线程执行周期任务，同时为了满足资源管理的需求而需要限制后台线程的数量的应用场景。

```
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize)

public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize, ThreadFactory threadFactory)
```

**SingleThreadScheduledExecutor**：
适用于需要单个后台线程执行周期任务，同时需要保证顺序地执行各个任务的应用场景。

```
public static ScheduledExecutorService newSingleThreadScheduledExecutor()

public static ScheduledExecutorService newSingleThreadScheduledExecutor(ThreadFactory threadFactory)
```

### 6. Future接口

Future接口和实现Future接口的FutureTask类用来表示异步计算的结果。当我们把Runnable接口或Callable接口的实现类提交（submit）给ThreadPoolExecutor或ScheduledThreadPoolExecutor时，ThreadPoolExecutor或ScheduledThreadPoolExecutor会向我们返回一个FutureTask对象。

```
Future submit(Callable task)

Future submit(Runnable task, T result)

Future<> submit(Runnable task)
```

### 7. Runnable接口和Callable接口

Runnable接口和Callable接口的实现类，都可以被ThreadPoolExecutor或ScheduledThreadPoolExecutor执行。它们之间的区别是Runnable不会返回结果，而Callable可以返回结果。除了可以自己创建实现Callable接口的对象外，还可以使用工厂类Executors来把一个Runnable包装成一个Callable。

```
public static Callable<Object> callable(Runnable task)

public static <T> Callable<T> callable(Runnable task, T result)
```

提交给ThreadPoolExecutor或ScheduledThreadPoolExecutor执行时，submit()会向我们返回一个FutureTask对象。我们可以执行FutureTask.get()方法来等待任务执行完成。当任务成功完成后FutureTask.get()将返回该任务的结果。例如，如果提交的是callable(Runnable task)，FutureTask.get()方法将返回null；如果提交的是callable(Runnable task, T result)，FutureTask.get()方法将返回result对象。

> [Java工程师的进阶之路 并发篇（一）](https://juejin.im/post/6844904192658636814)<br>
> [Java工程师的进阶之路 并发篇（二）](https://juejin.im/post/6844904192738328589)<br>
> [Java工程师的进阶之路 并发篇（三）](https://juejin.im/post/6844904193065484301)<br>
> [Java工程师的进阶之路 并发篇（四）](https://juejin.im/post/6844904193174536199)<br>
> [Java工程师的进阶之路 并发篇（五）](https://juejin.im/post/6844904193182924808)<br>