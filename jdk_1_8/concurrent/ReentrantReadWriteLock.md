# ReentrantReadWriteLock
可重入读写锁，读锁是基于AbstractQueuedSynchronizer（AQS）的共享模式实现的，写锁是基于AQS的独占模式实现的。
有几个重要的内部类：
* Sync
* NonFairSync
* FairSync
* ReadLock
* WriteLock
