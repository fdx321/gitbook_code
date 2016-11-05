# 可重入锁
上一节主要看了一下 *Lock* 和 *Condition* 两个接口的定义。这节我看 *ReentrantLock.java* 的代码边记录自己的理解。

**ReentrantLock** 除了实现 *Lock* 接口指定的几个方法，还提供了 *isLocked()*, *getOwner()* 等方法用来查看锁的一些状态。

它提供了 *公平锁* 和 *非公平锁* 两种锁，可以通过构造函数传递一个boolean型的变量来指定类型。
它两唯一的区别就是公平锁在遇到竞争是，会按照先后顺序让等待最久的线程优先获得锁。但是公平不见得性能就好，但是它可以避免一些线程饥饿的情况。

**ReentrantLock** 是借助同步器 *AbstractQueuedSynchronizer* （后面简称AQS） 来实现的，内部有三个重要的内部类，*Snyc*, *NonfairSync*, *FairSnyc*. 后两者是Snyc的子类。ReentrantLock 本身的 lock， unlock等方法实现很简单，都是代理给sync去做的。

下面通过分析各种情况来看下同步器的工作原理，默认都是非公平锁

### 1. 获得一次锁
先看一下NonfairSync类的lock方法，
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
第一的逻辑见（1），这里分析下当前线程已经获得锁的情况下，第二次lock的逻辑。
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
将Node添加到列表后，进行acquireQueued
```java
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
