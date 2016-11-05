# 可重入锁
上一节主要看了一下 *Lock* 和 *Condition* 两个接口的定义。接下来我边看 *ReentrantLock.java* 的代码边记录自己的理解。

**ReentrantLock** 除了实现 *Lock* 接口指定的几个方法外，还提供了 *isLocked()*, *getOwner()* 等方法用来查看锁的一些状态。

它提供了 *公平锁* 和 *非公平锁* 两种锁，可以通过构造函数传递一个boolean型的变量来指定类型。
它两唯一的区别就是公平锁在遇到竞争是，会按照先后顺序让等待最久的线程优先获得锁。公平不见得性能就好，但是它可以避免一些线程饥饿的情况。

**ReentrantLock** 是借助同步器 *AbstractQueuedSynchronizer* （后面简称AQS） 来实现的，内部有三个重要的内部类，*Snyc*, *NonfairSync*, *FairSnyc*. 后两者是Snyc的子类。ReentrantLock 的lock， unlock等方法都是代理给sync去做的。

下面通过分析非公平锁的几种情况来看下同步器的工作原理

### 1. 获得一次锁
NonfairSync类的lock方法，
```java
final void lock() {
  /**
   * 调用AQS 的 CAS 方法
   * 将 state 将 0（锁没有owner） 设置为 1 (锁有owner了)
   **/
  if (compareAndSetState(0, 1)) {
    //获得锁成功，设置属主后返回
    setExclusiveOwnerThread(Thread.currentThread());
  } else {
    /**
     * 两种情况会到这里，调用AQS的acquire方法
     * 1. 锁被另一个线程获得了
     * 2. 锁被当前线程已经获得了
     **/
    acquire(1);
  }
}
```
### 2. 同一个线程获得两次锁
这种情况也就是所谓的重入，第一次获得锁的逻辑见（1），这里分析下当前线程在已经获得锁的情况下，第二次lock的逻辑。
```java
//AQS的acquire方法
public final void acquire(int arg) {
  if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
    selfInterrupt();
}
```
```java
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    // 或的state，因为当前线程已经获得锁了，所以c为1
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        //当前线程已经获得锁了，增加锁的计数到state并返回
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

### 3. 线程1已经获得锁，线程2没有获得锁一直等待，线程1释放锁，线程2得到锁
#### 3.1 在线程1已经获得锁的情况下，线程2调用lock()
```java
//AQS的acquire方法
public final void acquire(int arg) {
  //这种情况下，线程2 tryAcquire返回false，执行addWaiter操作
  if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
    selfInterrupt();
}
```
addWaiter会构建一个Node双向列表，完成后的列表如下
```
┌───────────────┐      ┌───────────────┐
│waitStatus=0   │─────▶│waitStatus=0   │
│thread=null    │      │thread=thread2 │
│nextWaiter=null│◀─────│nextWaiter=null│
└───────────────┘      └───────────────┘
       ▲                       ▲       
       │                       │       
     head                     tail     
```
将Node添加到列表后，进行acquireQueued. 第一次循环在shouldParkAfterFailedAcquire中将p的waitStatus从0变成-1(SIGNAL)并返回false， 第二次循环就一直park该线程。
```java
//AQS acquireQueued
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```
线程2被park后，Node链表的结构如下：
```
┌───────────────┐      ┌───────────────┐
│waitStatus=-1  │─────▶│waitStatus=0   │
│thread=null    │      │thread=thread2 │
│nextWaiter=null│◀─────│nextWaiter=null│
└───────────────┘      └───────────────┘
       ▲                       ▲       
       │                       │       
     head                     tail     
```
#### 3.2 线程1 unlock()

```java
//AQS的release方法
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        //锁释放成功，unpark 后继者
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}

protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
      throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) {
      free = true;
      setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```

#### 3.3 线程1 unlock成功后，线程2获得锁
见AQS的acquireQueued方法，线程2 unpark后，继续循环，然后获得锁，然后将head从Node中去除。
```java
if (p == head && tryAcquire(arg)) {
    setHead(node);
    p.next = null; // help GC
    failed = false;
    return interrupted;
}
```
结束后的Node结构如下：
```
┌───────────────┐
│waitStatus=0   │
│thread=thread2 │
│nextWaiter=null│
└───────────────┘
      ▲  ▲       
      │  │       
   head tail
```
---
### 4. 公平锁/非公平锁的区别
两者的大部分操作都是基于Sync类来实现的，唯一的区别就是获得锁时候的区别，看下 NonfairSync 和 FairSync 的区别就可以了。
NonfairSync的lock方法，进来就compareAndSetState, 如果拿到锁就返回了，不用每次都进队列，达到一个抢占的效果。FairSync 的 lock每次都会进队列。两者的tryAcquire方法实现也有点小区别，这里不贴代码了。
```java
static final class NonfairSync extends Sync {
    final void lock() {
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else
            acquire(1);
    }
}

static final class FairSync extends Sync {
    final void lock() {
        acquire(1);
    }
}
```
### 5. lock 可interrupt/不可interrupt实现的区别
可以看到，一个是被interrupt的时候直接抛出异常，结束循环，取消acquire。一个是将interrupted设置为true，继续循环。
```java
private void doAcquireInterruptibly(int arg) throws InterruptedException {
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();//区别在这里
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}

final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;//区别在这里
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```
### 6. tryLock指定等待时间
实现也类似，就是在上面的循环里面，调用的是LockSupport带超时时间的park方法
```java
LockSupport.parkNanos(this, nanosTimeout);
```

---
差不多就是这样，AQS独占锁(排它锁)的原理就是：
* 通过一个Node双向队列来管理等待锁的线程，通过waitStatus来管理每个Node的状态，thread来管理Node所属的线程。
* Node链表中head、tail指针的移动，waitStatus状态的变更，AQS中state变量的维护等都是通过Unsafe的CAS来保证原子性的。
* 线程的挂起、唤醒是通过LockSupport的park, unpark来实现的。

AQS 同步器 只是两种类型的锁，独占锁和共享锁，后面我再学习并记录一下共享锁的原理。


### Reference
[深度解析Java 8：JDK1.8 AbstractQueuedSynchronizer的实现分析（上）](http://www.infoq.com/cn/articles/jdk1.8-abstractqueuedsynchronizer)
