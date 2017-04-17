---
layout:     post
title:      "Java并发编程之细枝末节"
date:       2017-04-17
author:     "Liz"
header-img: "img/2017-02-01-binder/pic.jpeg"
tags:
    - Java
    - Concurrent
---

分析Java并发编程(主要是Concurrent包)是一个大工程,此篇是一些细节,之后会再具体上源码

### this引用逃逸

说实话看Java并发编程实战这本书看的比较心累的,很多东西细节很少还需要自己去找大量资料

[this引用很棒的分析](http://blog.csdn.net/flysqrlboy/article/details/10607295)

`Object obj = new Object()`

this引用逃逸其实非常简单。在创建一个类时,会有①初始化变量,②把对象赋值给引用obj的过程。因为重排序的问题,可能②先于①,导致obj有值了但对象的变量其实还在未定义的状态,导致后续程序出错。

解决方法:像上文所言,等对象完整构建完后再发布this

### interrupt

`interrupt`中断线程。这个和我们平时接触的中断不一样,它有以下两大含义:

* 对于object.wait,Thread.sleep等阻塞函数在遇到interrupt时会抛出InterruptedException并清除线程中断状态
* 注意LockSupport.park不会抛出InterruptedException,但会离开相应中断
* 对于其余情况,就置一个intterupt的标识位为1

#### 三大方法

* interrupt：置线程的中断状态
* isInterrupt：线程是否中断
* interrupted：返回线程的上次的中断状态，并清除中断状态

#### 处理Exception

对于InterruptedException的处理，可以有两种情况,但最关键的就是**不能swallow it(只捕获不处理)**:

* 外层代码可以处理这个异常，直接抛出这个异常即可
* 如果不能抛出这个异常，比如在run()方法内，因为在得到这个异常的同时，线程的中断状态已经被清除了，需要保留线程的中断状态，则需要调用Thread.currentThread().interrupt()


### 同步工具类

* 闭锁:命令一组线程在同一个时刻开始执行某个任务，或者等待一组相关的操作结束。尤其适合计算并发执行某个任务的耗时。

```java
public class CountDownLatchTest {  
  
    public void timeTasks(int nThreads, final Runnable task) throws InterruptedException{  
        final CountDownLatch startGate = new CountDownLatch(1);  
        final CountDownLatch endGate = new CountDownLatch(nThreads);  
          
        for(int i = 0; i < nThreads; i++){  
            Thread t = new Thread(){  
                public void run(){  
                    try{  
                        startGate.await();  
                        try{  
                            task.run();  
                        }finally{  
                            endGate.countDown();  
                        }  
                    }catch(InterruptedException ignored){  
                          
                    }  
                      
                }  
            };  
            t.start();  
        }  
          
        long start = System.nanoTime();  
        System.out.println("打开闭锁");  
        startGate.countDown();  
        endGate.await();  
        long end = System.nanoTime();  
        System.out.println("闭锁退出，共耗时" + (end-start));  
    }  
      
    public static void main(String[] args) throws InterruptedException{  
        CountDownLatchTest test = new CountDownLatchTest();  
        test.timeTasks(5, test.new RunnableTask());  
    }  
      
    class RunnableTask implements Runnable{  
  
        @Override  
        public void run() {  
            System.out.println("当前线程为：" + Thread.currentThread().getName());  
              
        }     
    }  
 
 ```
 
* 信号量:控制同时访问某个特定资源的操作数量
* 栅栏:所有线程到达某处时再进行

### Volatile与内存栅栏

Volatile初次觉得是一个很简单的东西,被设置成该属性的变量每次写完都会强制刷新缓存,使缓存内的变量无效,那么下次去读时必定读到是最新的。

但当它用内存栅栏去解释时,又比较迷茫了

[解释volatile非常好的一篇文章](http://www.infoq.com/cn/articles/java-memory-model-4?utm_source=infoq&utm_campaign=user_page&utm_medium=link)

关键总结为以下三点:

* 当第二个操作是volatile写时，不管第一个操作是什么，都不能重排序。**这个规则确保volatile写之前的操作不会被编译器重排序到volatile写之后。
*** 当第一个操作是volatile读时，不管第二个操作是什么，都不能重排序。**这个规则确保volatile读之后的操作不会被编译器重排序到volatile读之前。**
* 当第一个操作是volatile写，第二个操作是volatile读时，不能重排序。

JMM采取保守策略。下面是基于保守策略的JMM内存屏障插入策略：

* 在每个volatile写操作的前面插入一个StoreStore屏障。
* 在每个volatile写操作的后面插入一个StoreLoad屏障。
* 在每个volatile读操作的后面插入一个LoadLoad屏障。
* 在每个volatile读操作的后面插入一个LoadStore屏障。

在实际执行时，只要不改变volatile写-读的内存语义，编译器可以根据具体情况省略不必要的屏障。

![](/img/2017-04-17-concurrent-1/14924361712622.jpg)

### DCL


![](/img/2017-04-17-concurrent-1/14923501737279.jpg)

著名的单例模式,DCL是不安全的。

原因:
与前面的this引用逃逸差不多,这边在初始化resource已经有值了,B线程拿到的却可能是不完整的。

正确的方式:

#### 对resource加Volatile

看过前面volatile部分,其实比较好理解。

![](/img/2017-04-17-concurrent-1/14924365071616.jpg)

其实关键在于2,3的重排序,使用volatile之所以可以解决这个问题,因为**volatile写之前的操作不会被编译器重排序到volatile写之后。** 那么2一定在3之前,那么如果线程B看到instance已经有值了,等价于该对象已经被初始化了。

#### 外部初始化

外面包一层`Holder`,类加载过程中能保证`INSTANCE`初始化的完整性。
```java
 
public class SingletonKerriganF {     
      
    private static class SingletonHolder {     
        /**   
         * 单例对象实例   
         */    
        static final SingletonKerriganF INSTANCE = new SingletonKerriganF();     
    }     
      
    public static SingletonKerriganF getInstance() {     
        return SingletonHolder.INSTANCE;     
    }     
}    
```

#### 神奇的写法

看到CAS+Atomatic+无限for循环的强大之处

```java
public class Singleton {    
    private static final AtomicReference<Singleton> INSTANCE = new AtomicReference<Singleton>();    
    private Singleton (){}    
    public static  Singleton getInstance() {            
        for (;;) {            
            Singleton current = INSTANCE.get();                 
            if (current != null) {                
                return current;            
            }            
            current = new Singleton();            
            if (INSTANCE.compareAndSet(null, current)) {                
                return current;           
           }       
     }    
  }
}
```

#### 枚举

枚举据说是最安全的用法。


### 有趣的例子

下面是一些来自书上的例子,写的很有启发性,特此记录一下:

#### 例子1

这个例子把`future`放入一个`ConcurrentMap`中,当后续来的发现该任务已经存在时,就直接通过`get`阻塞等待。注意`putIfAbsent`,`ConcurrentMap`中有趣的函数。

![](/img/2017-04-17-concurrent-1/14923441291027.jpg)

#### 例子2

`CompletionService`的案例。开始把所有任务都放入线程池,然后等待结果,如果有一个返回了能立马处理返回的结果。

![](/img/2017-04-17-concurrent-1/14923457269263.jpg)

#### 例子3

`invokeAll`很好的案例。会等所有任务都执行完毕再返回。


![](/img/2017-04-17-concurrent-1/14923459992373.jpg)

### 小tip

* 在可能发生死锁时,使用`tryLock`

* 活锁指任务执行时因两者冲突双方不断回滚，再次重试时又冲突只能再回滚。所以需要在重试时增加随机性

* 当线程持有锁时间较长,使用公平锁。一个场景:A释放锁,B是队列中第一个准备苏醒去拿锁,C进来了,如果是非公平锁,C直接拿去执行,执行结束B刚刚要拿到锁,吞吐量大大提高。

* 只有在任务相互独立时,设置线程池界限才合理,否则很容易造成饿死,考虑场景:线程池界限为1,A等B执行，B等A结束再执行,导致死锁。此时应该使用无界线程池。




















