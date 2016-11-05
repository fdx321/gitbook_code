# CountDownLatch
CountDownLatch是一个同步工具类，它允许一个或多个线程一直等待，直到其他线程的操作执行完后再执行.
CountDownLatch是基于*AbstractQueuedSynchronizer*（AQS）同步器的共享模式来实现的。有一个重要的内部类Snyc，它是AQS的子类。
为什么是共享模式呢？因为它允许多个线程等待一个条件，在CountDownLatch state为0的时候，多个线程都必须被唤起（相当于共享一个锁，这么说好像不太对）。

来一个简单的使用场景分析一下，就不贴代码了。
```
1. CountDownLatch latch = new CountDownLatch(2);
2. 然后有两个线程执行了 *latch.await()* 操作.
3. 然后又两个线程分别执行了 *latch.countDown()*
```
### 1. 构造函数分析
```java
public CountDownLatch(int count) {
  if (count < 0) throw new IllegalArgumentException("count < 0");
  this.sync = new Sync(count);
}

Sync(int count) {
  setState(count);
}
```
count不能小于0，构造函数的功能就是设置AQS的state为count.

### 2. await()实现分析
```java
public final void acquireSharedInterruptibly(int arg) throws InterruptedException {
    // await方法是可被interrupt的，所以，检查线程是否被打断，是的话直接抛异常
    if (Thread.interrupted())
        throw new InterruptedException();
    //尝试获得锁(暂时叫锁吧，好像不太合适)，如果state为0, 获取成功，返回. 否则，进入等待队列
    if (tryAcquireShared(arg) < 0)  
        doAcquireSharedInterruptibly(arg);
}

protected int tryAcquireShared(int acquires) {
    return (getState() == 0) ? 1 : -1;
}

private void doAcquireSharedInterruptibly(int arg) throws InterruptedException {
    // 构建Node队列, 注意，这里是共享模式 Node.SHARED
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor(); //获得前置节点
            if (p == head) {
                int r = tryAcquireShared(arg); //尝试获得锁
                if (r >= 0) {
                    //进入到这里，表示获得成功了
                    setHeadAndPropagate(node, r); // 将head指针往后移，并循环操作后面的节点
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
            }
            //第一次循环将waitStatus从0置为-1（SIGNAL,需要通知），
            //第二次循环将线程park
            if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```
两个线程分别进行完await操作后，等待队列的结构如下：
```
┌───────────────┐   ┌────────────────┐   ┌────────────────┐
│waitStatus=-1  │──▶│waitStatus=-1   │──▶│waitStatus=0    │
│thread=null    │   │thread=thread1  │   │thread=thread2  │
│nextWaiter=null│◀──│nextWaiter=NodeX│◀──│nextWaiter=NodeX│
└───────────────┘   └────────────────┘   └────────────────┘
       ▲                                        ▲       
       │                                        │       
     head                                      tail     
```

### 3. countDown的实现分析
```java
public final boolean releaseShared(int arg) {
    /**
     * 具体逻辑见下面tryReleaseShared的代码
     * 第一次countDown将state 从 2 置为 1
     * 第二次countDown将state 从 1 置为 0，并返回 true, 进入if条件
     **/
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}

//Sync 重写的
protected boolean tryReleaseShared(int releases) {
    for (;;) {
        int c = getState();
        if (c == 0)
            return false;
        int nextc = c-1;
        if (compareAndSetState(c, nextc))
            return nextc == 0;
    }
}

// 多次countDown后，state为0的情况下，执行该方法
private void doReleaseShared() {
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0)) //将当前node的waitStatus置为0
                    continue; // CAS操作失败，下次循环继续执行CAS
                unparkSuccessor(h); //唤醒第一个等待的线程
            }
            else if (ws == 0 && !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue; // CAS操作失败，下次循环继续执行CAS
        }
        if (h == head) // loop if head changed
            break;
    }
}

private void unparkSuccessor(Node node) {
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread);
}
```
### 4. state到达0后，等待线程被唤醒的过程
看一下doAcquireSharedInterruptibly中for循环里调用的setHeadAndPropagate方法

```java
/**
 * 总结一下，就是讲head指针往后移一个节点，继续执行doReleaseShared操作，唤醒后面的线程，依次这样
 **/
private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head;
    setHead(node);
    if (propagate > 0 || h == null || h.waitStatus < 0 || (h = head) == null || h.waitStatus < 0) {
        Node s = node.next;
        if (s == null || s.isShared())
            doReleaseShared();
    }
}
```

### Reference
[深度解析Java 8：AbstractQueuedSynchronizer的实现分析（下）](http://www.infoq.com/cn/articles/java8-abstractqueuedsynchronizer)
