---
layout:     post
title:      "Locks"
date:       2017-04-19
author:     "Liz"
header-img: "img/2017-02-01-binder/pic.jpeg"
tags:
    - Java
    - Concurrent
---

locks,锁。概念很好理解,其实都是用了介绍过得AQS来实现的。主要分析三个类:

### ReentrantLock

```java
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```
通过参数控制是公平锁还是非公平锁

```java
class X {
    private final ReentrantLock lock = new ReentrantLock();
    // ...
 
    public void m() {
      lock.lock();  // block until condition holds
      try {
        // ... method body
      } finally {
        lock.unlock()
      }
    }
}
```

通过`tryLock`防止死锁,`lockInterruptly`来相应中断。

### Condition
```java
/*
 * class BoundedBuffer {
 *   <b>final Lock lock = new ReentrantLock();</b>
 *   final Condition notFull  = <b>lock.newCondition(); </b>
 *   final Condition notEmpty = <b>lock.newCondition(); </b>
 *
 *   final Object[] items = new Object[100];
 *   int putptr, takeptr, count;
 *
 *   public void put(Object x) throws InterruptedException {
 *     <b>lock.lock();
 *     try {</b>
 *       while (count == items.length)
 *         <b>notFull.await();</b>
 *       items[putptr] = x;
 *       if (++putptr == items.length) putptr = 0;
 *       ++count;
 *       <b>notEmpty.signal();</b>
 *     <b>} finally {
 *       lock.unlock();
 *     }</b>
 *   }
 *
 *   public Object take() throws InterruptedException {
 *     <b>lock.lock();
 *     try {</b>
 *       while (count == 0)
 *         <b>notEmpty.await();</b>
 *       Object x = items[takeptr];
 *       if (++takeptr == items.length) takeptr = 0;
 *       --count;
 *       <b>notFull.signal();</b>
 *       return x;
 *     <b>} finally {
 *       lock.unlock();
 *     }</b>
 *   }
 * }
 * */
```

 看到condition是与lock绑定在一起的，主要有:
 
 * `await()` 默认会抛出interrupt异常,当发现interrupt时就抛异常吧
 * `signal()`
 * `signalAll()`
 
 三大类方法
 
 来看在AQS中的具体实现:
 
 ```java
 public class ConditionObject implements Condition, java.io.Serializable {
        private static final long serialVersionUID = 1173984872572414699L;
        /** First node of condition queue. */
        private transient Node firstWaiter;
        /** Last node of condition queue. */
        private transient Node lastWaiter;
                 public final void await() throws InterruptedException {
            if (Thread.interrupted())
                throw new InterruptedException();
            Node node = addConditionWaiter();
            //是的没有看错,这里在释放锁,因为await()前肯定是会调用lock()占用锁的，所以需要释放。
            int savedState = fullyRelease(node);
            int interruptMode = 0;
            //释放完毕后，遍历AQS的队列，看当前节点是否在队列中，
            while (!isOnSyncQueue(node)) {
            ///不在 说明它还没有竞争锁的资格，所以继续将自己沉睡。
                LockSupport.park(this);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            //在signal的时候入队
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }
        private Node addConditionWaiter() {
            Node t = lastWaiter;
            // 如果最后一个waiter已经被cancel那就把它unlink
            if (t != null && t.waitStatus != Node.CONDITION) {
                unlinkCancelledWaiters();
                t = lastWaiter;
            }
            Node node = new Node(Thread.currentThread(), Node.CONDITION);
            if (t == null)
                firstWaiter = node;
            else
                t.nextWaiter = node;
            //在队尾插入节点
            lastWaiter = node;
            return node;
        }
        public final void signal() {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            Node first = firstWaiter;
            if (first != null)
                doSignal(first);
        }
        private void doSignal(Node first) {
            do {
            //修改头节点
                if ( (firstWaiter = first.nextWaiter) == null)
                    lastWaiter = null;
                first.nextWaiter = null;
            } while (!transferForSignal(first) &&
                     (first = firstWaiter) != null);
        }
        final boolean transferForSignal(Node node) {
        /*
         * If cannot change waitStatus, the node has been cancelled.
         */
            if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
                return false;
            //将老的头结点，加入到AQS的等待队列中            
            Node p = enq(node);
            int ws = p.waitStatus;
            //如果该结点的状态为cancel 或者修改waitStatus失败，则直接唤醒。
            if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
                LockSupport.unpark(node.thread);
             return true;
    }
```




> 关键的就在于此，我们知道AQS自己维护的队列是当前等待资源的队列，AQS会在资源被释放后，依次唤醒队列中从前到后的所有节点，使他们对应的线程恢复执行。直到队列为空。而Condition自己也维护了一个队列，该队列的作用是维护一个等待signal信号的队列，两个队列的作用是不同，事实上，每个线程也仅仅会同时存在以上两个队列中的一个，流程是这样的：
1. 线程1调用reentrantLock.lock时，线程被加入到AQS的等待队列中。
2. 线程1调用await方法被调用时，该线程从AQS中移除，对应操作是锁的释放。
3. 接着马上被加入到Condition的等待队列中，意味着该线程需要signal信号。
4. 线程2，因为线程1释放锁的关系，被唤醒，并判断可以获取锁，于是线程2获取锁，并被加入到AQS的等待队列中。
5. 线程2调用signal方法，这个时候Condition的等待队列中只有线程1一个节点，于是它被取出来，并被加入到AQS的等待队列中。  注意，这个时候，线程1 并没有被唤醒。
6. signal方法执行完毕，线程2调用reentrantLock.unLock()方法，释放锁。这个时候因为AQS中只有线程1，于是，AQS释放锁后按从头到尾的顺序唤醒线程时，线程1被唤醒，于是线程1回复执行。
7. 直到释放所整个过程执行完毕。


**只有到发送signal信号的线程调用reentrantLock.unlock()后因为它已经被加到AQS的等待队列中，所以才会被唤醒。** 
 
就是说就算`signal`了因为没锁还是不能进行啊摔

### ReentrantReadWriteLock

```java
public class ReentrantReadWriteLock
        implements ReadWriteLock, java.io.Serializable {
    private static final long serialVersionUID = -6992448646407690164L;
    /** Inner class providing readlock */
    private final ReentrantReadWriteLock.ReadLock readerLock;
    /** Inner class providing writelock */
    private final ReentrantReadWriteLock.WriteLock writerLock;
    /** Performs all synchronization mechanics */
    final Sync sync;
    //读锁占用量
    static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
    //写锁占用量
    static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }
    //读锁的重入次数(threadLocal!)
    static final class ThreadLocalHoldCounter
            extends ThreadLocal<HoldCounter> {
        public HoldCounter initialValue() {
            return new HoldCounter();
        }
    }
    //threadLocal 当前线程占用的读锁
    private transient ThreadLocalHoldCounter readHolds;
    // firstReader 代表第一个获取读锁的线程（这里的第一个指的是：可能是首次获取读锁线程，可能是在写锁释放后的首次获取的读锁线程。更确切的说，指的是第一个把读锁的计数（shared count）从0改为1的线程。      
    //firstReaderHoldCount 是firstReader 线程的重入次数
    private transient Thread firstReader = null;
    private transient int firstReaderHoldCount;
    //不是threadLocal,最后一个读锁的holdCounter
    private transient HoldCounter cachedHoldCounter;
```
理解了上面的,下面的代码就好懂很多

```java
//如果没有读锁占用,因为当前线程马上要占用了,就设置first
if (r == 0) {  
    firstReader = current;  
    firstReaderHoldCount = 1;  
} else if (firstReader == current) {  
    firstReaderHoldCount++;  
} else {  
    HoldCounter rh = cachedHoldCounter;  
    if (rh == null || rh.tid != current.getId())  
        cachedHoldCounter = rh = readHolds.get();  
    else if (rh.count == 0)  
        readHolds.set(rh);  
    rh.count++;  
}  
```
看到定义了两个锁`readLock`和`writeLock`,一个读锁,一个写锁。

#### readLock

```java
public void lock() {
    sync.acquireShared(1);
}
```

照理看应用特有的`tryAcquire()`:
尝试获取共享锁,获取不了就排队等

```java
protected final int tryAcquireShared(int unused) {
    //如果写锁被人占用并且不是被自己占用,返回-1
    Thread current = Thread.currentThread();
    int c = getState();
    if (exclusiveCount(c) != 0 &&
        getExclusiveOwnerThread() != current)
        return -1;
    int r = sharedCount(c);
    /**
    readerShouldBlock是在读锁lock时候，需要首先判断是否要阻塞当前读锁的获取。
    readerShouldBlock有两种实现：
    公平锁：判断AQS同步等待队列的头结点是否有正在等待的线程，且等待线程非当前线程。如果存在，那么返回true，!true=false，顺序排队等待执行。
    非公平锁：判断当前头节点线程是否是writer（互斥模式），如果是，则返回true，!true=false，顺序排队等待执行
    **/
    if (!readerShouldBlock() &&r < MAX_COUNT && compareAndSetState(c, c + SHARED_UNIT)) {
        if (r == 0) {
        //更新之前提到的第一个节点
            firstReader = current;
            firstReaderHoldCount = 1;
        } else if (firstReader == current) {
            firstReaderHoldCount++;
        } else {
            HoldCounter rh = cachedHoldCounter;
            if (rh == null || rh.tid != current.getId())
                cachedHoldCounter = rh = readHolds.get();
            else if (rh.count == 0)
                readHolds.set(rh);
            rh.count++;
        }
        return 1;
    }
    //如果第一次获取失败了,进入下个函数继续获取
    return fullTryAcquireShared(current);
}
final int fullTryAcquireShared(Thread current) {
       HoldCounter rh = null;
       for (;;) {
           int c = getState();
           //如果当前写锁被人占用
            if (exclusiveCount(c) != 0) {
                if (getExclusiveOwnerThread() != current)
                    return -1;
                // else we hold the exclusive lock; blocking here
            // would cause deadlock.
            //如果当前读需要阻塞,比如目前写正排在第一位,那就做清空处理
            } else if (readerShouldBlock()) {
                // Make sure we're not acquiring read lock reentrantly
                if (firstReader == current) {
                        // assert firstReaderHoldCount > 0;
                } else {
                    if (rh == null) {
                        rh = cachedHoldCounter;
                        if (rh == null || rh.tid != current.getId()) {
                            rh = readHolds.get();
                            if (rh.count == 0)
                                readHolds.remove();
                        }
                    }
                    if (rh.count == 0)
                        return -1;
                }
            }
            if (sharedCount(c) == MAX_COUNT)
                throw new Error("Maximum lock count exceeded");
            
            if (compareAndSetState(c, c + SHARED_UNIT)) {
                if (sharedCount(c) == 0) {
                    firstReader = current;
                    firstReaderHoldCount = 1;
                } else if (firstReader == current) {
                    firstReaderHoldCount++;
                } else {
                     if (rh == null)
                        rh = cachedHoldCounter;
                    if (rh == null || rh.tid != current.getId())
                        rh = readHolds.get();
                    else if (rh.count == 0)
                        readHolds.set(rh);
                    rh.count++;
                    cachedHoldCounter = rh; // cache for release
                 }
                return 1;
            }
        }
    }
```

我们顺便看一下这里公平和非公平锁的含义,完全体现在`readerShouldBlock()`和`writeShouldBlock()`的不同写法上。

##### nonFairLock

```java
static final class NonfairSync extends Sync {
    final boolean readerShouldBlock() {
        return apparentlyFirstQueuedIsExclusive();
    }
}
//由于读锁不应该让写锁始终等待，因此apparentlyFirstQueuedIsExclusive判断等待队列中的第一个节点如果存在且是写锁（!s.isShared()）则readerShouldBlock返回true（表示请求读锁的线程请求失败）。
final boolean apparentlyFirstQueuedIsExclusive() {
    Node h, s;
    return (h = head) != null &&(s = h.next)  != null &&
!s.isShared() &&s.thread != null;
}
```

#### writeLock

```java
protected final boolean tryAcquire(int acquires) {
    Thread current = Thread.currentThread();
    int c = getState();
    ////这里先判断AQS的state是否等于0，如果不等于0，则有可能当前时间读锁或者写锁被持有
    int w = exclusiveCount(c);
    if (c != 0) {
       //写锁没有被占用或者写锁被占用了但不是当前线程占用的,返回
        if (w == 0 || current != getExclusiveOwnerThread())
            return false;
        //判断写锁重入次数是否大于65535
        if (w + exclusiveCount(acquires) > MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        // Reentrant acquire
        setState(c + acquires);
        return true;
    }
    if (writerShouldBlock() ||
            !compareAndSetState(c, c + acquires))
        return false;
    setExclusiveOwnerThread(current);
    return true;
}
```
这里和独占锁策略差不多,只有一个`writerShouldBlock()`

##### nonFairLock

```java
final boolean writerShouldBlock() {
    return false; // writers can always barge
}
```
##### FairLock

```java
final boolean writerShouldBlock() {
    return hasQueuedPredecessors();
}
```
看是不是队列中第一个元素。
#### policy总结

##### 写锁获取:

①如果有另一个线程获取写锁,插入队列
②如果读锁处于活动中，写锁退让等待。
③如果自己已经获取写锁,就重入

##### 读锁获取

①如果有另一个线程正在获取写锁,插入队列
②如果自己获取写锁,则可以进行锁降级
③如果读锁处于活动中,就重入

##### 锁释放

锁释放时,如果下一个获取的是读锁,会继续往后传播

##### 锁升级

读取锁是不能直接升级为写入锁的。因为获取一个写入锁需要释放所有读取锁，所以如果有两个读取锁视图获取写入锁而都不释放读取锁时就会发生死锁。





