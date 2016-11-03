# ThreadPoolExecutor.java

## **1. 继承关系**

```
    ┌───────────────┐                                           
    │  (I)Executor  │                                           
    └───────────────┘                                           
            ▲                                                   
            │                                                   
            │                                                   
┌──────────────────────┐        ┌──────────────────────────────┐
│  (I)ExecutorService  │◀───────│  (C)AbstarctExecutorService  │
└──────────────────────┘        └──────────────────────────────┘
                                                ▲               
                                                │               
                                                │               
                                   ┌─────────────────────────┐  
                                   │  (C)ThreadPoolExecutor  │  
                                   └─────────────────────────┘

```

## **2. 介绍**

下面是ThreadPoolExecutor最主要的一个构造函数。

```
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler)

```

### **2.1 corePoolSize\/maximumPoolSize**

ThreadPoolExecutor会根据 _corePoolSize_ 和 _maximumPoolSize_ 动态的调整线程池大小。

提交一个task时：

* 如果线程池中在runnning的线程数小于 _corePoolSize_，则创建一个新线程来处理该 task，及时当前线程池中有空闲线程。
* 如果线程池中在running的线程数在 _corePoolSize_ 和 _maximumPoolSize_ 之间，且queue满了，则创建一个新线程来处理该 task.

默认情况下，创建线程池后，core 线程是在有task被提交时才被创建和初始化的。特殊情况（比如先在队列里塞一些task进去），可以通过手动调用 _prestartCoreThread_ 或者 _prestartAllCoreThreads_ 来预先启动线程。

### **2.2 keepAliveTime\/unit**

当线程池中线程数超过 _corePoolSize_时，如果有线程idle的时间超过 _keepAliveTime_,该线程会被终止。这是一种节省资源的方式。

### **2.3 workQueue**

* 线程数小于 _corePoolSize_, Executor倾向于创建新的Thread.
* 线程数大于 _corePoolSize_, Executor倾向于将task放进queue.
* 如果queue满了，且线程数超过 _maximumPoolSize_，不能创建新线程了，则reject该task.

三种queuing策略

* **Direct handoffs**：这个一种同步queue（一般的选择是 _SynchronousQueue_），直接把task传递给thread，不会保留他们。这种情况，如果没有线程立马可用，那么task放入queue就会失败，因此必要要新建一个线程。 这种策略可以在处理多个有依赖关系的请求时防止被锁住。Direct handoffs 通常需要一个无限大的maximumPoolSizes防止新的任务被拒绝。 反过来，也会出现线程无限增长的情况。 Executors的 _newCachedThreadPool_ 用的就是这种策略。

* **Unbounded queues**：无界队列（比如一个没有定义capacity的LinkedBlockingQueue, 或者capacity是Integer.MAX\_VALUE），当所有的 corePoolSize 线程都busy的时候，新提交的任务都会被添加到queque中，鉴于queue无界，所以设置的 _maximumPoolSize_也就没什么卵用了。task之间完全独立的时候比较适用。

* **Bounded queues**：有界队列（比如一个定义了capacity的ArrayBlockingQueue），采用的这种策略配合一个有限的 _maximumPoolSize_可以防止资源耗尽，当然参数的调优比较难控制。队列大小和 _maximumPoolSize_是反相关的，队列大\/maximumPoolSize小的情况下，可以减少CPU使用、上下文切换，但使得吞吐量也比较低。队列小\/maximumPoolSize大的情况下，CPU负载可能会较高，上下文切换比较频繁。


### **2.4 threadFactory**

除非特别指定，默认情况下使用 _DefaultThreadFactory_ 创建线程。

### **2.5 handler**

_RejectedExecutionHandler_，当task被reject的时候，调用该handler。默认的handler会throw一个exception. 两种情况下task会被reject:

* 调用了shutdown\/shutdownNow方法
* 使用了有指定capacity的queue和maximumPoolSize，且队列和线程数都满了

**ThreadPoolExecutor** 提供了4中默认的reject策略：

* **ThreadPoolExecutor.AbortPolicy**：throw一个_RejectedExecutionException_
* **ThreadPoolExecutor.CallerRunsPolicy**： 直接在caller的线程中执行被reject的task
* **ThreadPoolExecutor.DiscardPolicy**：默默的丢弃task, 啥都不做，日志也不打，异常也不throw
* **ThreadPoolExecutor.DiscardOldestPolicy**: 在Executor没有被shutdown的情况下，从队列中去除最老的task，然后执行新提交的这个task.

### **2.6 hook\/callback**

ThreadPoolExecutor还提供了一些钩子方法，用于在线程执行的某个阶段做些处理，比如打印日志、数据统计等。如：

```
beforeExecute(Thread, Runnable)
afterExecute(Runnable, Throwable)

```

这些方法默认什么都不做，子类可以通过Override来自定义它的行为。**如果钩子方法在执行过程中出现异常，可能会导致工作线程返回失败或意外退出。**

### **2.7 扩展样例**

一个可暂停的线程池

```
class PausableThreadPoolExecutor extends ThreadPoolExecutor {
  private boolean isPaused;
  private ReentrantLock pauseLock = new ReentrantLock();
  private Condition unpaused = pauseLock.newCondition();

  public PausableThreadPoolExecutor(...) { super(...); }

  protected void beforeExecute(Thread t, Runnable r) {
    super.beforeExecute(t, r);
    pauseLock.lock();
    try {
      while (isPaused) unpaused.await();
    } catch (InterruptedException ie) {
      t.interrupt();
    } finally {
      pauseLock.unlock();
    }
  }

  public void pause() {
    pauseLock.lock();
    try {
      isPaused = true;
    } finally {
      pauseLock.unlock();
    }
  }

  public void resume() {
    pauseLock.lock();
    try {
      isPaused = false;
      unpaused.signalAll();
    } finally {
      pauseLock.unlock();
    }
  }
}

```

## **3. Executors类的几个工厂方法分析**

分析完**ThreadPoolExecutor**的一些关键参数后，回过头来看一下**Executors**的几个工厂方法，就更清楚了。

### **3.1 newFixedThreadPool**

采用**ubounded queue**;

corePoolSize 和 maximumPoolSize 都是 nThreads\(因为用了ubounded queue，maximumPoolSize设再大也没有意义\);

keepAliveTime为0，表示idle线程不自动停止。

```
return new ThreadPoolExecutor(nThreads, nThreads,
                              0L, TimeUnit.MILLISECONDS,
                              new LinkedBlockingQueue<Runnable>());

```

### **3.2 newSingleThreadExecutor**

采用**ubounded queue**；

线程数为1；

keepAliveTime为0，表示idle线程不自动停止。

外层用了代理类，提供了finalize功能，不可配置，只暴露的一些接口，使返回的Executor不可配置。

```
return new FinalizableDelegatedExecutorService
           (new ThreadPoolExecutor(1, 1,
                                   0L, TimeUnit.MILLISECONDS,
                                   new LinkedBlockingQueue<Runnable>()));

```

### **3.3 newCachedThreadPool**

采用**Direct Handsoff**的**SynchronousQueue**（关于这个queque，后面会有专门的介绍）；

keepAliveTime为60S, idle线程超过这个时间后会被回收；

corePoolSize为0和maximumPoolSize为最大。

```
return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                              60L, TimeUnit.SECONDS,
                              new SynchronousQueue<Runnable>());

```

## **4. 详细的实现分析**

Doug Lea将workerCount（线程池当前有效线程数）和runState（线程池当前所处状态）放置到一个原子变量ctl（AtomicInteger）上，原子变量高三位保存runState，低29位保存workerCount。因此，ThreadPoolExecutor（JDK8）支持的最大线程数为2^29-1。

线程池状态有以下五种：

* RUNNING（正常运行，-1）: Accept new tasks and process queued tasks
* SHUTDOWN（关闭,0）: Don't accept new tasks, but process queued tasks
* STOP（停止,1）: Don't accept new tasks, don't process queued tasks, and interrupt in-progress tasks
* TIDYING（整理中,2）: All tasks have terminated, workerCount is zero, the thread transitioning to state TIDYING will run the terminated\(\) hook method
* TERMINATED（终结,3）: terminated\(\) has completed

线程池状态图，当前工作线程计数以及线程池的状态变迁，通过ctl原子变量的**CAS**操作完成。

```

                     On invocation of                                     
     ┌───────────┐  shutdown(), perhaps     ┌───────────┐                 
     │RUNNING(-1)│─────implicitly in ──────▶│SHUTDOWN(0)│────┐            
     └───────────┘      finalize()          └───────────┘    │            
           │                                      │          │            
           │                                      │ When both queue and   
           │                                      │ pool are empty        
           ├──────────────────────────────────────┘                       
           │                                                 │            
On invocation of shutdownNow()                               │            
           │                                                 │            
           ▼                                                 ▼            
       ┌───────┐              When pool                ┌──────────┐       
       │STOP(1)│──────────────is empty  ──────────────▶│TIDYING(2)│       
       └───────┘                                       └──────────┘       
                                                             │            
                                                             │            
                                                             │            
    ┌─────────────┐       When the terminated() hook method has completed.
    │TERMINATED(3)│◀──────Threads waiting in awaitTermination() will      
    └─────────────┘       return when the state reaches TERMINATED.
```

