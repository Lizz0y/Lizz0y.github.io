---
layout:     post
title:      "ExexutorService"
date:       2017-04-17
author:     "Liz"
header-img: "img/2017-02-01-binder/pic.jpeg"
tags:
    - Java
    - Concurrent
---

### 源码分析

#### 一些接口

##### ExecutorService

```java
public interface ExecutorService extends Executor {

     //关闭,保证之前提交的task都会被执行,但新的不会再被接受
    void shutdown();
    List<Runnable> shutdownNow();
    boolean isShutdown();
    boolean isTerminated();
    boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedException;
    
    <T> Future<T> submit(Callable<T> task);
    <T> Future<T> submit(Runnable task, T result);
    Future<?> submit(Runnable task);
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException;
    <T> T invokeAny(Collection<? extends Callable<T>> tasks,
                    long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
    ...
}
```

##### ExecutorCompletionSerivce

```java
public class ExecutorCompletionService<V> implements CompletionService<V> {
    private final Executor executor;
    private final AbstractExecutorService aes;
    private final BlockingQueue<Future<V>> completionQueue;
    /**
     * FutureTask extension to enqueue upon completion主要是扩充了FutureTask,在结束时把它加到BlockingQueue里面
     */
    private class QueueingFuture extends FutureTask<Void> {
        QueueingFuture(RunnableFuture<V> task) {
            super(task, null);
            this.task = task;
        }
        protected void done() { completionQueue.add(task); }
        private final Future<V> task;
    }
    public Future<V> take() throws InterruptedException {
        return completionQueue.take();
    }
}
```

其实就是加了一个`BlockingQueue`,并把结果放在里面,take()或是poll()都走的是BlockingQueue的api,也就是说如果没有完成的话你是拿不到结果的,一直阻塞在那。

##### AbstractExecutorService

* invokeAny: 只要有一个任务正常完成(不抛异常),就能正常返回,线程池中的其他任务都被取消。
* invokeAll: 等所有任务执行完才会返回,除非定时器到了或者被interrupt抛出异常了

```java
public abstract class AbstractExecutorService implements ExecutorService {
    protected <T> RunnableFuture<T> newTaskFor(Runnable runnable, T value) {
        return new FutureTask<T>(runnable, value);
    }
    protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
        return new FutureTask<T>(callable);
    }
    //submit runnable,将其转换为FutureTask,然后执行。
    public Future<?> submit(Runnable task) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<Void> ftask = newTaskFor(task, null);
        execute(ftask);
        return ftask;
    }
/**
一旦有1个任务正常完成(执行过程中没有抛异常)，线程池会终止其他未完成的任务。
可以看到在doInvokeAny中用到了completionService,也就是会等待第一个完成。
**/
private <T> T doInvokeAny(Collection<? extends Callable<T>> tasks,
                            boolean timed, long nanos)
        throws InterruptedException, ExecutionException, TimeoutException {
        if (tasks == null)
            throw new NullPointerException();
        int ntasks = tasks.size();
        if (ntasks == 0)
            throw new IllegalArgumentException();
        List<Future<T>> futures= new ArrayList<Future<T>>(ntasks);
        ExecutorCompletionService<T> ecs =
            new ExecutorCompletionService<T>(this);
        try {
            // Record exceptions so that if we fail to obtain any
            // result, we can throw the last exception we got.
            ExecutionException ee = null;
            long lastTime = timed ? System.nanoTime() : 0;
            Iterator<? extends Callable<T>> it = tasks.iterator();

            // Start one task for sure; the rest incrementally
            //invokeAny 有一个返回结果就行,那就先试着提交一个,万一返回了呢,后面的都不用提交啦。。。
            futures.add(ecs.submit(it.next()));
            --ntasks;
            int active = 1;

            for (;;) {
                Future<T> f = ecs.poll();
                //如果还没有执行完,会返回null,这是completionSerivce的特性,因为queue里还没有数据
                if (f == null) {
                //如果还有没提交的,那就提交吧,active代表正在执行的(还没执行完)
                    if (ntasks > 0) {
                        --ntasks;
                        futures.add(ecs.submit(it.next()));
                        ++active;
                    }
                    //都执行完了,也没意义了
                    else if (active == 0)
                        break;
                    else if (timed) {
                    //等待这么久还没执行完,超时了
                        f = ecs.poll(nanos, TimeUnit.NANOSECONDS);
                        if (f == null)
                            throw new TimeoutException();
                        long now = System.nanoTime();
                        nanos -= now - lastTime;
                        lastTime = now;
                    }
                    else
                    //这代表不限时
                        f = ecs.take();
                }
                if (f != null) {
                    --active;
                    try {
                        return f.get();
                    //oops,失败了。。再看下一个能不能拿到结果吧
                    } catch (ExecutionException eex) {
                        ee = eex;
                    } catch (RuntimeException rex) {
                        ee = new ExecutionException(rex);
                    }
                }
            }

            if (ee == null)
                ee = new ExecutionException();
            throw ee;

        } finally {
        //不管是成功或失败,都是要取消的
            for (Future<T> f : futures)
                f.cancel(true);
        }
    }
    //invokeAll future之间互不影响,并且会阻塞直到全部都完成
    public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException {
        if (tasks == null)
            throw new NullPointerException();
        List<Future<T>> futures = new ArrayList<Future<T>>(tasks.size());
        boolean done = false;
        try {
            for (Callable<T> t : tasks) {
                RunnableFuture<T> f = newTaskFor(t);
                futures.add(f);
                execute(f);
            }
            for (Future<T> f : futures) {
                if (!f.isDone()) {
                    //阻塞
                    try {
                        f.get();
                    } catch (CancellationException ignore) {
                    } catch (ExecutionException ignore) {
                    }
                }
            }
            done = true;
            return futures;
        } finally {
            if (!done)
                for (Future<T> f : futures)
                    f.cancel(true);
        }
    }
```



#### ThreadPoolExecutor

难啃的源码部分:

`ExecutorService`接口在`Executor`的基础上提供了对任务执行的生命周期的管理

* RUNNING状态：线程池正常运行，可以接受新的任务并处理队列中的任务；
* SHUTDOWN状态：不再接受新的任务，但是会执行队列中的任务；
* STOP状态：不再接受新任务，不处理队列中的任务

其实整个核心在于以下两点:

* 当前状态是否为RUNNING
* 当前状态是否为SHUTDOWN状态,如果是队列中是否还有未执行的任务。

理解这两点才能更好的理解代码

 ![](/img/2017-04-17-concurrent-threadPoolExecutor/14925935622113.jpg)


```java
public class ThreadPoolExecutor extends AbstractExecutorService {
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
    private static final int COUNT_BITS = Integer.SIZE - 3;
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;
    

     //当大于corePoolSize时,空闲超过该time的会被Kill
    private volatile long keepAliveTime;
    
    //实现了一个简单的非重入的互斥锁， 实现互斥锁主要目的是为了中断的时候判断线程是在空闲还是运行,不用ReentrantLock是为了避免任务执行的代码中修改线程池的变量，如setCorePoolSize，因为ReentrantLock是可重入的。
    /** worker实现了一个简单的不可重入的互斥锁，而不是用ReentrantLock可重入锁
     * 因为我们不想让在调用比如setCorePoolSize()这种线程池控制方法时可以再次获取锁(重入)
    * 解释：
    *   setCorePoolSize()时可能会interruptIdleWorkers()，在对一个线程interrupt时会要              w.tryLock()
     *  如果可重入，就可能会在对线程池操作的方法中中断线程，类似方法还有：
    *   setMaximumPoolSize()
    *   setKeppAliveTime()
    *   allowCoreThreadTimeOut()
    *   shutdown()
    * 此外，为了让线程真正开始后才可以中断，初始化lock状态为负值(-1)，在开始runWorker()时将state置为0，而state>=0才可以中断*/
    private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable
    {
        ....
        
        Worker(Runnable firstTask) {
            setState(-1); // inhibit interrupts until runWorker
            this.firstTask = firstTask;
            this.thread = getThreadFactory().newThread(this);
        }
        public void run() {
            runWorker(this);
        }

        protected boolean isHeldExclusively() {
            return getState() != 0;
        }

        protected boolean tryAcquire(int unused) {
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }

        protected boolean tryRelease(int unused) {
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }

        public void lock()        { acquire(1); }
        public boolean tryLock()  { return tryAcquire(1); }
        public void unlock()      { release(1); }
        public boolean isLocked() { return isHeldExclusively(); }

        void interruptIfStarted() {
            Thread t;
            if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                }
            }
        }
    }
    //1.如果workCount<corePoolSize,直接addWorker(core),即新建线程
        //1.1加入失败,因为线程池被shutDown并且任务队列中没有任务了
        //1.2加入失败,因为线程数大于corePoolSize
        //1.3加入成功,返回
    //2.如果前面失败,或者workCount>corePoolSize
        //2.1如果线程还活着,那就加入到任务队列中等待吧
            //2.1.1如果加完发现不再running了。。。直接从队列中退出,结束任务吧。。。
            //2.1.2加完
    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        /**
         1. 如果比corePoolSize小,那就尝试开启一个新的线程,并把当前command作为first t
         ask
         2. 如果一个task成功入队了,还需要double-check是否需要新加一个thread,因为可能又有
         老的已经执行结束了
         3. 如果不能成功入队,尝试新加一个thread?? 也就是线程数大于corePoolSize 小于maximumPoolSize 的情况。
         **/
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))//addWorker一个new thread来处理该任务(true的情况)，直接返回；如果addWork返回false(线程池被shutdown or shutdownNow;或者同时又别的客户端提交任务，并且此时线程数大于corePoolSize);继续往下执行
                return;
            c = ctl.get();
        }
        if (isRunning(c) && workQueue.offer(command)) {//对addWorker返回false的情况进行判断,当线程池还运行着，说明是因为thread number 大于corePoolSize的情况，则&&操作第二个表达式把任务添加到workQueue队列
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))//把任务加入队列的同时，pool被shutdown， 则从队列中删除task,并且调用rejectHandler的方法
                reject(command);
            else if (workerCountOf(recheck) == 0)
                //这是一句很有意思的代码addWorker(null,false)
                 //只保证有一个worker线程可以从queue中获取任务执行就行了？？
                 //因为只要还有活动的worker线程，就可以消费workerQueue中的任务
                addWorker(null, false);        }
        //如果线程池中的线程数量大于等于corePoolSize，且队列workQueue已满，但线程池中的线程数量小于maximumPoolSize，则会创建新的线程来处理被添加的任务
        else if (!addWorker(command, false))
            reject(command);
    }
    
    private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // 考虑这种情况,已经SHUTDOWN但是队列中任务还没有执行完
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))//当前的运行状态不为RUNNING，或者在SHUTDOWN状态下（不接受新任务但是会执行完队列中的任务）,否则返回false  
                return false;

            for (;;) {
                int wc = workerCountOf(c);
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                //尝试增加workerCount
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            final ReentrantLock mainLock = this.mainLock;
            //要往线程池提交任务了,加锁吧,这里这里mainLock已经要去获取锁了
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int c = ctl.get();
                    int rs = runStateOf(c);

                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }
```
![](/img/2017-04-17-concurrent-threadPoolExecutor/14925942478046.jpg)


`ThreadPoolExecutor`与线程相关的几个成员变量是：`keepAliveTime`、`allowCoreThreadTimeOut`、`poolSize`、`corePoolSize`、`maximumPoolSize`，它们共同负责线程的创建和销毁。

`corePoolSize`：
>线程池的基本大小，即在没有任务需要执行的时候线程池的大小，并且只有在工作队列满了的情况下才会创建超出这个数量的线程。这里需要注意的是：在刚刚创建`ThreadPoolExecutor`的时候，线程并不会立即启动，而是要等到有任务提交时才会启动，除非调用了`prestartCoreThread/prestartAllCoreThreads`事先启动核心线程。再考虑到`keepAliveTime`和`allowCoreThreadTimeOut`超时参数的影响，所以没有任务需要执行的时候，线程池的大小不一定是`corePoolSize`。

`maximumPoolSize`：
线程池中允许的最大线程数，线程池中的当前线程数目不会超过该值。如果队列中任务已满，并且当前线程个数小于`maximumPoolSize`，那么会创建新的线程来执行任务。这里值得一提的是`largestPoolSize`，该变量记录了线程池在整个生命周期中曾经出现的最大线程个数。为什么说是曾经呢？因为线程池创建之后，可以调用`setMaximumPoolSize()`改变运行的最大线程的数目。



##### 拒绝策略

* ThreadPoolExecutor.AbortPolicy:丢弃任务并抛出RejectedExecutionException异常。 
* ThreadPoolExecutor.DiscardPolicy：也是丢弃任务，但是不抛出异常。 
* ThreadPoolExecutor.DiscardOldestPolicy：丢弃队列最前面的任务，然后重新尝试执行任务（重复此过程）
* ThreadPoolExecutor.CallerRunsPolicy：由调用线程处理该任务 

##### runWorker()

```java
final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        //此处是调用Worker类的tryRelease()方法，将state置为0， 而interruptIfStarted()中只有state>=0才允许调用中断，说明只有真正进入runWorker()后才允许被中断
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            //先运行firstTask,然后再从池子里拿任务
            while (task != null || (task = getTask()) != null) {
                w.lock();//上锁，不是为了防止并发执行任务，为了在shutdown()时不终止正在运行的worker
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null;
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            processWorkerExit(w, completedAbruptly);
        }
    }
private Runnable getTask() {
        boolean timedOut = false; // Did the last poll() time out?

        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            // 1.rs > SHUTDOWN 所以rs至少等于STOP,这时不再处理队列中的任务
			   // 2.rs = SHUTDOWN 所以rs>=STOP肯定不成立，这时还需要处理队列中的任务除非队列为空
			   // 这两种情况都会返回null让runWoker退出while循环也就是当前线程结束了，所以必须要decrement wokerCount
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                decrementWorkerCount();//这个任务要go die了
                return null;
            }

            boolean timed;      // 是否需要定时从workQueue中获取

            for (;;) {
                int wc = workerCountOf(c);
                timed = allowCoreThreadTimeOut || wc > corePoolSize;

                if (wc <= maximumPoolSize && ! (timedOut && timed))
                    break;
                if (compareAndDecrementWorkerCount(c))
                    return null;
                c = ctl.get();  // Re-read ctl
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }

            try {
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) ://大于core,要有超时机制
                    workQueue.take();//小于就算了一直阻塞着,知道抛出异常
                if (r != null)
                    return r;
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }
private void processWorkerExit(Worker w, boolean completedAbruptly) {
        if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
            decrementWorkerCount();

        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            completedTaskCount += w.completedTasks;
            workers.remove(w);
        } finally {
            mainLock.unlock();
        }

        tryTerminate();

        int c = ctl.get();
        if (runStateLessThan(c, STOP)) {
        //当前线程终止了,如果是异常终止再加一个worker;
        //如果不是,看当前线程数目是否小于应该维护的数目
            if (!completedAbruptly) {
                int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
                if (min == 0 && ! workQueue.isEmpty())
                    min = 1;
                if (workerCountOf(c) >= min)
                    return; // replacement not needed
            }
            addWorker(null, false);
        }
    }
```

![](/img/2017-04-17-concurrent-threadPoolExecutor/14922461012911.png)

>Worker控制中断主要有以下几方面：
1、初始AQS状态为-1，此时不允许中断interrupt()，只有在worker线程启动了，执行了runWorker()，将state置为0，才能中断
    不允许中断体现在：
    A、shutdown()线程池时，会对每个worker tryLock()上锁，而Worker类这个AQS的tryAcquire()方法是固定将state从0->1，故初始状态state==-1时tryLock()失败，没发interrupt()
    B、shutdownNow()线程池时，不用tryLock()上锁，但调用worker.interruptIfStarted()终止worker，interruptIfStarted()也有state>0才能interrupt的逻辑
2、为了防止某种情况下，在运行中的worker被中断，runWorker()每次运行任务时都会lock()上锁，而shutdown()这类可能会终止worker的操作需要先获取worker的锁，这样就防止了中断正在运行的线程

**Worker实现的AQS为不可重入锁，为了是在防止获得worker锁的情况下再进入其它一些需要加锁的方法**

### 终止

```java
//  尝试终止,只是尝试啊！
//  判断线程池是否满足终止的状态
//A、如果状态满足，但还有线程池还有线程，尝试对其发出中断响应，使其能进入退出流程
//B、没有线程了，更新状态为tidying->terminated
final void tryTerminate() {
    for (;;) {
        int c = ctl.get();
        //如果正在运行,或者已经超过TIDYING,或者是SHUTDOWN但是队列未空,直接返回，很保守的策略啊
        if (isRunning(c) ||
            runStateAtLeast(c, TIDYING) ||
            (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
            return;
        if (workerCountOf(c) != 0) { // Eligible to terminate
            //中断正在运行的线程,当然如果真的正在运行时中断不了的
            interruptIdleWorkers(ONLY_ONE);
            return;
        }
        //如果workCount是0
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                try {
                    terminated();
                } finally {
                    ctl.set(ctlOf(TERMINATED, 0));
                    termination.signalAll();
                }
                return;
            }
        } finally {
            mainLock.unlock();
        }
            // else retry on failed CAS
    }
}
private void interruptIdleWorkers(boolean onlyOne) {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        for (Worker w : workers) {
            Thread t = w.thread;
            if (!t.isInterrupted() && w.tryLock()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                } finally {
                    w.unlock();
                }
            }
            if (onlyOne)
                break;
        }
    } finally {
        mainLock.unlock();
    }
}
```

```java
public void shutdown() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        checkShutdownAccess();
        advanceRunState(SHUTDOWN);
        interruptIdleWorkers();
        onShutdown(); // hook for ScheduledThreadPoolExecutor
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
}
```

看到在`shutdown`中,将state设为SHUTDOWN了,但在`tryTerminate`中还是可能因为队列中有未执行的任务就直接返回了,但后续新的任务是再也加不进来的

我们可以将worker划分为：

* 空闲worker：正在从workQueue阻塞队列中获取任务的worker
* 运行中worker：正在task.run()执行任务的worker

>某些情况下，interruptIdleWorkers()时多个worker正在运行，不会对其发出中断信号，假设此时workQueue也不为空。那么当多个worker运行结束后，会到workQueue阻塞获取任务，获取到的执行任务，没获取到的，如果还是核心线程，会一直workQueue.take()阻塞住，线程无法终止，因为workQueue已经空了，且shutdown后不会接收新任务了。这就需要在shutdown()后，还可以发出中断信号

这段话的意思是，如果worker都在运行状态,那么是会忽略中断信号的,但如果后来他们进入了阻塞获取任务额状态,就无法终止了因为队列已经空了,再也拿不到任务了。

这里我一直不能理解,尤其是`ONLY_ONE`才一个啊宝贝！后来看到所有worker执行完后在`processWorkerExit`都会再进行一次`tryTerminate`,所以不管怎样最后一个worker执行完后线程池都会进入`Terminate`的状态欧耶

```java
public List<Runnable> shutdownNow() {
        List<Runnable> tasks;
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();
            advanceRunState(STOP);
            interruptWorkers();
            //把队列中还没有被执行的任务拿出来,返回给调用者i
            tasks = drainQueue();
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
        return tasks;
    }
private void interruptWorkers() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        for (Worker w : workers)
            w.interruptIfStarted();
    } finally {
        mainLock.unlock();
    }
}
//不管它在不在执行,直接中断吧,不用tryLock了！
void interruptIfStarted() {
    Thread t;
    if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
        try {
            t.interrupt();
        } catch (SecurityException ignore) {
        }
    }
}
```

>需要注意的是，对于运行中的线程调用Thread.interrupt()并不能保证线程被终止，task.run()内部可能捕获了InterruptException，没有上抛，导致线程一直无法结束。真的这样就很恶心了。。


状态的转化主要是：

* RUNNING -> SHUTDOWN（调用shutdown()） On invocation of shutdown(), perhaps implicitly in finalize()
* (RUNNING or SHUTDOWN) -> STOP(调用shutdownNow())On invocation of shutdownNow()
* SHUTDOWN -> TIDYING（queue和pool均empty）
* STOP -> TIDYING（pool empty，此时queue已经为empty）
* TIDYING -> TERMINATED(调用terminated())




注意: shutdown()一般和awaitTermination()结合使用~

![](/img/2017-04-17-concurrent-threadPoolExecutor/14925975345513.jpg)


### callable & runnable

```java
public interface Callable<V> {
    /**
     * Computes a result, or throws an exception if unable to do so.
     *
     * @return computed result
     * @throws Exception if unable to compute a result
     */
    V call() throws Exception;
}
```

* Callable规定的方法是call(),Runnable规定的方法是run().
* Callable的任务执行后可返回值，而Runnable的任务是不能返回值得
* call方法可以抛出异常，run方法不可以
* 运行Callable任务可以拿到一个Future对象，Future 表示异步计算的结果。它提供了检查计算是否完成的方法，以等待计算的完成，并获取计算的结果。

### 四大线程池

先看一下threadPoolExecutor的构造函数

```java
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue) {
    this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             Executors.defaultThreadFactory(), defaultHandler);
}
```
 
#### FixedThreadPool

```java
 public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,0L, TimeUnit.MILLISECONDS,
    new LinkedBlockingQueue<Runnable>());
}
```
无界队列,corePoolSize = maximumPoolSize = nThreads 如果核心满了,后来的线程会一直在队列中排队等待。

     
#### singleThreadExecutor   

```java
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
}
``` 

其实和fixedThreadPool一样,只是永远只有一个线程。

#### CacheThreadPool
   
```java 
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
}
```    

SynchronousQueue作为主线程池的工作队列，它是一个没有容量的阻塞队列。每个插入操作必须等待另一个线程的对应移除操作。也就是说,先插,插得太快 如果主线程提交任务的速度高于线程池中处理任务的速度时，CachedThreadPool会不断创建新线程。




#### ScheduledThreadPool

```java
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
    return new ScheduledThreadPoolExecutor(corePoolSize);
}
```

这个后续章节继续讨论吧


