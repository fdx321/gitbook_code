# **Executors.java**

这是一个工厂类，用来创建Executor, ExecutorService, ScheduledExecutorService. 还可以用来创建ThreadFactory, Callable任务。

## **1. 普通线程池**

### **newFixedThreadPool**

创建一个**固定线程数**的线程池，它使用一个**无界**\(队列长度Integer.MAX\_VALUE\)的queue. 最多有nThreads个线程在执行中，如果有额外的task进来且没有空闲的线程，这个task将被放入queue中，直到有空闲线程。池中的线程将一直存在，除非显示的调用shutdown\/shutdownNow停掉。

```
public static ExecutorService newFixedThreadPool(int nThreads)
public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory);

```

### **newSingleThreadExecutor**

只有**一个线程**， 使用一个**无界**queue, 能保证任务顺序执行。

```
public static ExecutorService newSingleThreadExecutor()
public static ExecutorService newSingleThreadExecutor(ThreadFactory threadFactory)

```

### **newCachedThreadPool**

创建一个按需的线程池，如果没有空闲的线程，就创建一个新线程并加入到pool中，如果一个线程已经空闲超过60S了，就终止它。

```
public static ExecutorService newCachedThreadPool();
public static ExecutorService newCachedThreadPool(ThreadFactory threadFactory);

```

## **2. 可调度的线程池**

### **newSingleThreadScheduledExecutor**

创建一个单线程的执行器，该执行器具有调度的功能，可以delay一段时间执行，或按一定的频率执行

```
public static ScheduledExecutorService newSingleThreadScheduledExecutor()
public static ScheduledExecutorService newSingleThreadScheduledExecutor(ThreadFactory threadFactory)

```

### **newScheduledThreadPool**

创建一个指定大小的线程池，具有调度的功能。_corePoolSize_: pool里的线程数量

```
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize)
public static ScheduledExecutorService newScheduledThreadPool(
            int corePoolSize, ThreadFactory threadFactory)

```

## **3. 不可配置的Executor**

通过DelegatedExecutorService\/DelegatedScheduledExecutorService代理一个Executor，代理后的Executor不能被修改配置，达到一个保护的效果。

```
public static ExecutorService unconfigurableExecutorService(ExecutorService executor)
public static ScheduledExecutorService unconfigurableScheduledExecutorService(ScheduledExecutorService executor)

```

## **4. ThreadFactory**

### **DefaultThreadFactory**

创建的线程都在一个ThreadGroup, 创建的先吃都是非daemon的，优先级是min\(Thread.NORM\_PRIORITY, ThreadGroup中允许的最大优先级\)。创建的线程的名字是_pool-N-thread-M_, N 是这个factory的sequence, M 是 这个facotory创建的线程的sequence.

### **PrivilegedThreadFactory**

其他的和DefaultThreadFactory一样，只是该factory创建的线程的**AccessControlContext**, **contextClassLoader**和调用PrivilegedThreadFactory\(\)的线程一样。

### **5.TODO**

学习 ForkJoinPool

