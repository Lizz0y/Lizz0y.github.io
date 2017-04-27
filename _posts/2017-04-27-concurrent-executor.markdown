---
layout:     post
title:      "Future总结"
date:       2017-04-27
author:     "Liz"
header-img: "img/2017-02-01-binder/pic.jpeg"
tags:
    - Java
    - Concurrent
---

首先分析一下`ScheduleThreadPoolExecutor`:

### ScheduleFuture

```java
public interface ScheduledFuture<V> extends Delayed, Future<V> {
}
```

### ScheduleThreadPoolExecutor

ScheduledThreadPoolExecutor的功能主要有两点：在固定的时间点执行（也可以认为是延迟执行），重复执行。入口多个,包括`execute`,`schedule`等等。

```java
public void execute(Runnable command) {
    schedule(command, 0, TimeUnit.NANOSECONDS);
}
 public ScheduledFuture<?> schedule(Runnable command,
                                       long delay,
                                       TimeUnit unit) {
    if (command == null || unit == null)
        throw new NullPointerException();
    RunnableScheduledFuture<?> t = decorateTask(command,
            new ScheduledFutureTask<Void>(command, null,
                                          triggerTime(delay, unit)));
    delayedExecute(t);
    return t;
}
    
```
>把任务包装成RunnableScheduledFuture对象，然后调用delayedExecute来实现延迟执行。任务包装类继承自ThreadPoolExecutor的包装类RunnableFuture，同时实现ScheduledFuture接口使包装类具有了延迟执行和重复执行这些功能以匹配ScheduledThreadPoolExecutor。


#### ScheduledFutureTask

```java
private class ScheduledFutureTask<V>
            extends FutureTask<V> implements RunnableScheduledFuture<V> {
        /** 距离任务开始执行的时间，纳秒为单位 */
        private long time;

        /**
         * 重复执行任务的间隔，即每隔多少时间执行一次任务
         */
        private final long period;
/**
         * Overrides FutureTask version so as to reset/requeue if periodic.
         */
        public void run() {
            boolean periodic = isPeriodic();
            if (!canRunInCurrentRunState(periodic))
                cancel(false);
            else if (!periodic)
            //不是周期的,直接跑吧
                ScheduledFutureTask.super.run();
            // 对于需要重复执行的任务，则执行一次，然后reset
            // 更新一下下次执行的时间，调用reExecutePeriodic更新任务在执行队列的
            // 位置（其实就是添加到队列的末尾
            else if (ScheduledFutureTask.super.runAndReset()) {
                setNextRunTime();
                reExecutePeriodic(outerTask);
            }
        }
}
    /**
     * 1.RUNNING返回true
     * 2.SHUTDOWN看标志位,翻译一下就知道了,就是字面意思
     */
    boolean canRunInCurrentRunState(boolean periodic) {
        return isRunningOrShutdown(periodic ?
                                   continueExistingPeriodicTasksAfterShutdown :
                                   executeExistingDelayedTasksAfterShutdown);
    }
```

#### DelayExecute

```java

public ScheduledThreadPoolExecutor(int corePoolSize) {
    super(corePoolSize, Integer.MAX_VALUE, 0, TimeUnit.NANOSECONDS,
            new DelayedWorkQueue());
}
    
 private void delayedExecute(RunnableScheduledFuture<?> task) {
        if (isShutdown())
            reject(task);
        else {
    
            super.getQueue().add(task);
            if (isShutdown() &&
                !canRunInCurrentRunState(task.isPeriodic()) &&
                remove(task))
                task.cancel(false);
            else
            //至少加入一个worker去runWorker执行task
                ensurePrestart();
        }
    }

```
我们可以看到,构造函数中是`DelayedWorkQueue`,而不是普通的`BlockQueue`,还要注意与之前提到过得`DelayedQueue`做区别。

##### DelayedWorkQueue

主要看一下offer函数 

```java
    public boolean offer(Runnable x) {
            if (x == null)
                throw new NullPointerException();
            RunnableScheduledFuture e = (RunnableScheduledFuture)x;
            final ReentrantLock lock = this.lock;
            lock.lock();
            try {
                int i = size;
                if (i >= queue.length)
                    grow();
                size = i + 1;
                if (i == 0) {
                    queue[0] = e;
                    setIndex(e, 0);
                } else {
                    siftUp(i, e);
                }
                if (queue[0] == e) {
                    // 因为有元素新添加了，第一个等待的线程可以结束等待了，因此这里
                    // 删除第一个等待线程
                    leader = null;
                    available.signal();
                }
            } finally {
                lock.unlock();
            }
            return true;
        }
    //重写compareTo,可以看到比较的都是time,也就是下次执行的时间
    public int compareTo(Delayed other) {
            if (other == this) // compare zero ONLY if same object
                return 0;
            if (other instanceof ScheduledFutureTask) {
                ScheduledFutureTask<?> x = (ScheduledFutureTask<?>)other;
                long diff = time - x.time;
                if (diff < 0)
                    return -1;
                else if (diff > 0)
                    return 1;
                else if (sequenceNumber < x.sequenceNumber)
                    return -1;
                else
                    return 1;
            }
            long d = (getDelay(TimeUnit.NANOSECONDS) -
                      other.getDelay(TimeUnit.NANOSECONDS));
            return (d == 0) ? 0 : ((d < 0) ? -1 : 1);
        }
    public RunnableScheduledFuture take() throws InterruptedException {
            final ReentrantLock lock = this.lock;
            lock.lockInterruptibly();
            try {
                for (;;) {
                    RunnableScheduledFuture first = queue[0];
                    if (first == null)
                        available.await();
                    else {
                        long delay = first.getDelay(TimeUnit.NANOSECONDS);
                        if (delay <= 0)
                            return finishPoll(first);
                        else if (leader != null)
                            available.await();
                        else {
                            Thread thisThread = Thread.currentThread();
                            leader = thisThread;
                            try {
                                available.awaitNanos(delay);
                            } finally {
                                if (leader == thisThread)
                                    leader = null;
                            }
                        }
                    }
                }
            } finally {
                if (leader == null && queue[0] != null)
                    available.signal();
                lock.unlock();
            }
        }
```

#### 总结

所以看到因为继承于`ThreadPoolExecutor`,整体只是变成了`DelayedWorkerQueue`,如果是周期性的,执行完就加入到队列中并设置`delay`是`time`,当`worker`去取时发现`delay`还大于0就`wait`




### 线程池总结

![](/img/2017-04-27-concurrent-executor/14932594538381.jpg)


所以,各种策略其实体现在,take的时候是否需要等待直到`getDelayed`小于0,增加任务的时候是否小于`coolPoolSize`使得可以直接往`worker`池加`worker`还是现在队列里排队。








