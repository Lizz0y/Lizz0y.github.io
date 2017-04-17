---
layout:     post
title:      "AbstractQueuedSynchronizer"
date:       2017-04-17
author:     "Liz"
header-img: "img/2017-02-01-binder/pic.jpeg"
tags:
    - Java
    - Concurrent
---

AQS在concurrent中是最重要的一个类,先分析一下源码。

>AQS的功能可以分为两类：独占功能和共享功能，它的所有子类中，要么实现并使用了它独占功能的API，要么使用了共享锁的功能，而不会同时使用两套API，即便是它最有名的子类ReentrantReadWriteLock，也是通过两个内部类：读锁和写锁，分别实现的两套


>公平锁和非公平锁，唯一的区别是在获取锁的时候是直接去获取锁，还是进入队列排队的问题

其实AQS最重要的作用

先看一些基本的结构:

#### Node

因为会进入队列排队,所以自然有`Node`结构
```java
static final class Node {
    /** 共享锁 */
    static final Node SHARED = new Node();
    /** 独占锁 */
    static final Node EXCLUSIVE = null;
    static final int CANCELLED =  1;
    static final int SIGNAL    = -1;
    static final int CONDITION = -2;
    static final int PROPAGATE = -3;
        /**
        SIGNAL:该节点的后继节点被blocked,所以当前节点释放锁或被取消时要唤醒后者,后面会看到几乎所有地方都会遍历两次,第一次设置SIGNAL;第二次发现时SIGNAL再进行唤醒
        CANCELLED:该节点因为超时或中断被取消
        CONDITION:在等待条件,它只有转移后才会进入sync queue,那时它的值被设为0
        PROPAGATE:释放共享锁时需要往后传递状态
        */
    volatile int waitStatus;
        ...
}

```

#### state

```java
/**
* The synchronization state.
*/
private volatile int state;
```


### ReentrantLock 独占锁代表

#### 公平锁

>AQS.java

```java

public final void acquire(long arg) {

    //尝试获取锁,无法获得就入队列;
    //acquireQueued返回true，说明当前线程被中断唤醒后获取到锁，
    // 重置其interrupt status为true。
    if (!tryAcquire(arg) &&
    acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
    selfInterrupt();
}
protected boolean tryAcquire(int arg) {
    throw new UnsupportedOperationException();
}
```

AQS有个很有趣的现象,所有的try...都交给子类去实现。

>一旦tryAcquire成功则立即返回，否则线程会加入队列 线程可能会反复的被阻塞和唤醒直到tryAcquire成功，这是因为线程可能被中断， 而acquireQueued方法中会保证忽视中断，只有tryAcquire成功了才返回。中断版本的独占获取是acquireInterruptibly这个方法， doAcquireInterruptibly这个方法中如果线程被中断则acquireInterruptibly会抛出InterruptedException异常。

##### tryAcquire

>ReentrantLock.java

```java
 protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    //ReentrantLock的state就是有多少xx占用锁(可重入)
    if (c == 0) {
        //没人占用,赶紧尝试占用
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
//是否是第一个节点
public final boolean hasQueuedPredecessors() {
    Node t = tail; // Read fields in reverse initialization order
    Node h = head;
    Node s;
    return h != t &&
            ((s = h.next) == null || s.thread != Thread.currentThread());
}
```

#### 加入队列
>AQS.java

AQS的队列有头有尾,当加入第一个节点时,会独立创造一个空的头节点,然后再在尾部插入第一个节点。

```java
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    //注意,这里是循环设置
    enq(node);
    return node;
}
//重要
private Node enq(final Node node) {
    //注意是个循环
    for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
            //这里没退出,所以第一个一定是个空节点
                tail = head;
            } else {
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                return t;
            }
         }
    }
}

//尝试加入队列 todo
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            //前面的节点就是头节点,说明没人有锁了,尽情的去获取吧
            if (p == head && tryAcquire(arg)) {
                //说明头结点永远是当前拥有锁的节点
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            // 线程会阻塞在parkAndCheckInterrupt方法中。
            // parkAndCheckInterrupt返回可能是前继unpark或线程被中断。
            if (shouldParkAfterFailedAcquire(p, node) &&//使用LockSupport.park()中断自己
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
         if (failed)
            cancelAcquire(node);
    }
}
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this)
    //好好的不park了,1.unpark2.interrupt
    return Thread.interrupted();
}
//如果异常了,就取消获取吧
private void cancelAcquire(Node node) {
        // Ignore if node doesn't exist
        if (node == null)
            return;

        node.thread = null;

        // 跳过状态是cancell的节点
        Node pred = node.prev;
        while (pred.waitStatus > 0)
            node.prev = pred = pred.prev;
        Node predNext = pred.next;
        node.waitStatus = Node.CANCELLED;
        // 1.我们是尾部节点,移除从尾到头的所有cancell的节点
        if (node == tail && compareAndSetTail(node, pred)) {
            compareAndSetNext(pred, predNext, null);
        } else {
            //2.如果不是头节点的后继节点,则将前节点设置为SIGNAL(代表其后继节点需要被唤醒)
            int ws;
            if (pred != head &&
                ((ws = pred.waitStatus) == Node.SIGNAL ||
                 (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
                pred.thread != null) {
                Node next = node.next;
                if (next != null && next.waitStatus <= 0)
                    compareAndSetNext(pred, predNext, next);
            } else {
                //直接唤醒吧后继节点吧
                unparkSuccessor(node);
            }

            node.next = node; // help GC
        }
    }
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL)
            /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
            return true;
        if (ws > 0) {
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
```



#### 释放锁

```java

public final boolean release(int arg) {
    if (tryRelease(arg)) {
         Node h = head;
        if (h != null && h.waitStatus != 0)
        //释放锁成功了,唤醒它的后继节点吧
            unparkSuccessor(h);
        return true;
    }
    return false;
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

![](/img/2017-04-17-concurrent-abs/14924406850306.jpg)
#### 非公平锁
```java
/**
* Sync object for non-fair locks
*/
static final class NonfairSync extends Sync {
    private static final long serialVersionUID = 7316153563782823691L;
        /**
         * Performs lock.  Try immediate barge, backing up to normal
         * acquire on failure.
         */
    final void lock() {
    //唯一的区别,如果能获取锁直接获取吧,否则尝试获取
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else
            acquire(1);
    }

    protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
    }
    final boolean nonfairTryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
        //没有hasQueuedPredecessors()
            if (compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0) // overflow
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
}
```

#### 限时
```java

 private boolean doAcquireNanos(int arg, long nanosTimeout)
        throws InterruptedException {
    long lastTime = System.nanoTime();
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return true;
            }
            if (nanosTimeout <= 0)
                return false;
            if (shouldParkAfterFailedAcquire(p, node) &&
                    nanosTimeout > spinForTimeoutThreshold)
                    LockSupport.parkNanos(this, nanosTimeout);
                long now = System.nanoTime();
            nanosTimeout -= now - lastTime;
            lastTime = now;
            if (Thread.interrupted())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

这里会抛出InterruptedException:  要注意,LockSupport.park()会被Interrupt但不会抛出Exception

##### 自旋spinForTimeoutThreshold

如果剩下的时间小于这个时间,就循环吧,省的进入阻塞状态发生上下文切换


#### countDownLatch 共享锁
```java
public void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
}
public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (tryAcquireShared(arg) < 0)
        doAcquireSharedInterruptibly(arg);
}
//实现节点自身获取共享锁成功后，唤醒下一个共享类型结点的操作，实现共享状态的向后传递。
private void doAcquireSharedInterruptibly(int arg)
        throws InterruptedException {
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
    
protected int tryAcquireShared(int acquires) {
    return (getState() == 0) ? 1 : -1;
}

private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head; // Record old head for check below
    setHead(node);
    //尝试signal下一个node
    if (propagate > 0 || h == null || h.waitStatus < 0) {
        Node s = node.next;
        if (s == null || s.isShared())
            doReleaseShared();
    }
}
private void doReleaseShared() {
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h);
            }
            else if (ws == 0 &&
                    !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            break;
    }
}  
private void unparkSuccessor(Node node) {
        /*
         * If status is negative (i.e., possibly needing signal) try
         * to clear in anticipation of signalling.  It is OK if this
         * fails or if status is changed by waiting thread.
         */
    int ws = node.waitStatus;
    if (ws < 0)
    compareAndSetWaitStatus(node, ws, 0);

        /*
         * Thread to unpark is held in successor, which is normally
         * just the next node.  But if cancelled or apparently null,
         * traverse backwards from tail to find the actual
         * non-cancelled successor.
         */
    Node s = node.next;
    //巧妙的做法,如果直接后继是null或者已经被取消了,就从尾部向前遍历。。虽然我也不知道为什么要这么做。。
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    //唤醒后面第一个状态不是cancell的节点
    if (s != null)
        LockSupport.unpark(s.thread);
}
    
```   

其实比起独占锁,就多了一步`setHeadAndPropagate`,主要分析一下这个函数:

* 首先把当前获取到锁的节点A设置为头部 
* 开始释放后面所有是共享属性的节点,先释放该节点的后继节点
* 我们看到,当后继节点B被unpark后，前面`doAcquireSharedInterruptibly`中会再一次循环再把该节点B设置为头节点,然后继续释放后继节点,所以整个状态就传递开了
* 那么前面的A呢,

![](/img/2017-04-17-concurrent-abs/14921858338931.jpg)

#### 释放

```java
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
protected boolean tryReleaseShared(int releases) {
    // Decrement count; signal when transition to zero
    for (;;) {
        int c = getState();
        if (c == 0)
            return false;
        int nextc = c-1;
        if (compareAndSetState(c, nextc))
            return nextc == 0;
    }
}
private void doReleaseShared() {
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h);
            }
            else if (ws == 0 &&
                        !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            break;
    }
}
```

>AQS来说它只是实现了一系列的用于判断“资源”是否可以访问的API,并且封装了在“访问资源”受限时将请求访问的线程的加入队列、挂起、唤醒等操作， AQS只关心“资源不可以访问时，怎么处理？”、“资源是可以被同时访问，还是在同一时间只能被一个线程访问？”、“如果有线程等不及资源了，怎么从AQS的队列中退出？”等一系列围绕资源访问的问题，而至于“资源是否可以被访问？”这个问题则交给AQS的子类去实现。

所以,当unpark一个node时,该node会继续调用unparkSuccessor唤醒下一个节点,然后下个再唤醒下一个。。。

#### FutureTask


看完前两个,Ft其实很简单。get的时候如果还没有就阻塞,set完了就进行唤醒,直接注释分析吧

```java
public class FutureTask<V> implements RunnableFuture<V> {
/*
 * Possible state transitions:
     * NEW -> COMPLETING -> NORMAL
     * NEW -> COMPLETING -> EXCEPTIONAL
     * NEW -> CANCELLED
     * NEW -> INTERRUPTING -> INTERRUPTED
     */
     //正在wait的队列
     private volatile WaitNode waiters;
      /** The thread running the callable; CASed during run() */
    private volatile Thread runner;
     public void run() {
        if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                         null, Thread.currentThread()))
            return;
        try {
            Callable<V> c = callable;
            if (c != null && state == NEW) {
                V result;
                boolean ran;
                try {
                    result = c.call();
                    ran = true;
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    setException(ex);
                }
                //call玩以后set
                if (ran)
                    set(result);
            }
        } finally {
            // runner must be non-null until state is settled to
            // prevent concurrent calls to run()
            runner = null;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts
            int s = state;
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
    }
    protected void set(V v) {
    //先设置为正在结束状态
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
            outcome = v;
            //然后设置为NORMAL状态
            UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
            finishCompletion();
        }
    }
    
    private void finishCompletion() {
        // assert state > COMPLETING;
        for (WaitNode q; (q = waiters) != null;) {
            if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
                for (;;) {
                    Thread t = q.thread;
                    if (t != null) {
                        q.thread = null;
                        //循环唤醒正在等待的队列上的节点thread
                        LockSupport.unpark(t);
                    }
                    WaitNode next = q.next;
                    if (next == null)
                        break;
                    q.next = null; // unlink to help gc
                    q = next;
                }
                break;
            }
        }

        done();

        callable = null;        // to reduce footprint
    }
    public boolean cancel(boolean mayInterruptIfRunning) {
        if (state != NEW)
            return false;
        //正在执行是否可以打断
        if (mayInterruptIfRunning) {
            if (!UNSAFE.compareAndSwapInt(this, stateOffset, NEW, INTERRUPTING))
                return false;
            Thread t = runner;
            if (t != null)
                t.interrupt();
            UNSAFE.putOrderedInt(this, stateOffset, INTERRUPTED); // final state
        }
        else if (!UNSAFE.compareAndSwapInt(this, stateOffset, NEW, CANCELLED))
            return false;
        //所以就算取消了,也会唤醒在get中沉睡的线程
        finishCompletion();
        return true;
    }

```
 
#### get()
 
 ```java
 public V get() throws InterruptedException, ExecutionException {
        int s = state;
        //如果比COMPLETING小,说明还没复制,await吧
        if (s <= COMPLETING)
            s = awaitDone(false, 0L);
        return report(s);
}
public V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException {
        if (unit == null)
            throw new NullPointerException();
        int s = state;
        if (s <= COMPLETING &&
            (s = awaitDone(true, unit.toNanos(timeout))) <= COMPLETING)
            throw new TimeoutException();
        return report(s);
}
private int awaitDone(boolean timed, long nanos）
        throws InterruptedException {
        // false,0
        final long deadline = timed ? System.nanoTime() + nanos : 0L;
        WaitNode q = null;
        boolean queued = false;
        for (;;) {
            if (Thread.interrupted()) {
                removeWaiter(q);
                throw new InterruptedException();
            }

            int s = state;
            //这个状态说明已经能赋值完了
            if (s > COMPLETING) {
                if (q != null)
                    q.thread = null;
                return s;
            }
            else if (s == COMPLETING) // cannot time out yet
            //让步给正在赋值的线程吧
                Thread.yield();
            else if (q == null)
                q = new WaitNode();
            else if (!queued)
            //加入到等待队列中区
                queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                     q.next = waiters, q);
            else if (timed) {
                nanos = deadline - System.nanoTime();
                if (nanos <= 0L) {
                    removeWaiter(q);
                    return state;
                }
                //如果有时间限制,那就睡一段时间
                LockSupport.parkNanos(this, nanos);
            }
            else
            //一直睡。。直到被unpart或者Interrupt
                LockSupport.park(this);
        }
    }
```    
     


