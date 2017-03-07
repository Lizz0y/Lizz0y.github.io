---
layout:     post
title:      "Broadcast广播分析"
date:       2017-03-07
author:     "Liz"
header-img: "img/2017-02-01-binder/pic.jpeg"
tags:
    - Android 
    - SourceCode
---

### BroadCast广播

继续分析广播。

#### 背景知识

消息传递: 其实跟观察者模式类似,一方订阅消息,当消息发布时接收处理事件。

* `J2EE`:消息驱动`Bean`来实现应用程序各个组件之间的消息传递
* `COM`:连接点实现消息传递
* `Android`:广播,现在更多的是Rx模式。

与`startService`类似,广播依旧要与AMS交互。程序将`Receiver`注册到AMS中,当相应的`Sender`发出广播时,AMS通知相应的`Receiver`去处理事件。

#### 源码分析

有了`service`的铺垫,看起源码来已经可以理解很多了。

首先看如何进行注册,在app层,依然使用AMS代理进行注册

* `mMainThread`:熟悉的`activityThread`
* `filter`:`IntentFilter`
* `rd`:不熟悉的对象`ReceiverDispather`

```java
return ActivityManagerNative.getDefault().registerReceiver(mMainThread.getApplicationThread(), mBasePackageName,rd, filter, broadcastPermission, userId);
```

引入一个最关键的类`ReceiverDispather`。

```java
rd = mPackageInfo.getReceiverDispatcher(receiver, context, scheduler,mMainThread.getInstrumentation(), true);
```

`InnerReceiver`是一个Binder实体对象,它负责真正的与AMS进行交互,在构造时外面包了一层`ReceiverDispatcher`。

看两者的构造函数:

```java
ReceiverDispatcher(BroadcastReceiver receiver, Context context,
                Handler activityThread, Instrumentation instrumentation,
                boolean registered) {
    if (activityThread == null) {
        throw new NullPointerException("Handler must not be null");
    }
    mIIntentReceiver = new InnerReceiver(this, !registered);
    mReceiver = receiver;
    mContext = context;
    mActivityThread = activityThread;
    mInstrumentation = instrumentation;
    mRegistered = registered;
    mLocation = new IntentReceiverLeaked(null);
    mLocation.fillInStackTrace();
}

final static class InnerReceiver extends IIntentReceiver.Stub {
    final WeakReference<LoadedApk.ReceiverDispatcher> mDispatcher;
    final LoadedApk.ReceiverDispatcher mStrongRef;
    InnerReceiver(LoadedApk.ReceiverDispatcher rd, boolean strong) {
        mDispatcher = new WeakReference<LoadedApk.ReceiverDispatcher>(rd);
        mStrongRef = strong ? rd : null;
    }
```

那么`rd`是如何与receiver绑定的呢:

```java
rd = mPackageInfo.getReceiverDispatcher(receiver, context, scheduler,mMainThread.getInstrumentation(), true);
```
这里的scheduler是mActivityThread中熟悉的handler,ReceiverDispathcer也保存了这个handler,作用后面会看到。

![](/img/2017-03-07-broadcast/14888180481880.jpg)



在`LoadApk`中:

```java
private final ArrayMap<Context, ArrayMap<BroadcastReceiver, ReceiverDispatcher>> mReceivers= new ArrayMap<Context, ArrayMap<BroadcastReceiver, LoadedApk.ReceiverDispatcher>>();

LoadedApk.ReceiverDispatcher rd = null;
ArrayMap<BroadcastReceiver, LoadedApk.ReceiverDispatcher> map = null;
if (registered) {
    map = mReceivers.get(context);
    if (map != null) {
        rd = map.get(r);
    }
}
if (rd == null) {
    rd = new ReceiverDispatcher(r, context, handler,instrumentation, registered);
    if (registered) {
        if (map == null) {
            map = new ArrayMap<BroadcastReceiver, LoadedApk.ReceiverDispatcher>();
            mReceivers.put(context, map);
        }
        map.put(r, rd);
    }
} else {
    rd.validate(context, handler);
}
```
在`loadapk`中,有一个`map`记录`receiver`与`dispather`的对应,而另一个map记录`context`与该`map`的对应。

![](/img/2017-03-07-broadcast/14888184255170.jpg)

进入AMS真正注册`receiver`:

在AMS中,有一个`map:IBinder->ReceiverList`,`IBinder`即前面的`IIntentReceiver`,`ReceiverList`也是一个很有趣的东西,一个`reciver`可以注册多个广播`filter`,所以`ReceiverList`继承于`ArrayList<BroadcastFilter>`,对应多个`IntentFilter`

```java
registerReceiver(sdStateReceiver, sdCarMonitorFilter);
```

```java
/**
 * A receiver object that has registered for one or more broadcasts.
 * The ArrayList holds BroadcastFilter objects.
 */
final class ReceiverList extends ArrayList<BroadcastFilter>
        implements IBinder.DeathRecipient {
```


```java
/**
* Keeps track of all IIntentReceivers that have been registered for
* broadcasts.  Hash keys are the receiver IBinder, hash value is
* a ReceiverList.
*/
final HashMap<IBinder, ReceiverList> mRegisteredReceivers = new HashMap<IBinder, ReceiverList>();
              
public Intent registerReceiver(IApplicationThread caller, String callerPackage,IIntentReceiver receiver, IntentFilter filter, String permission, int userId) {
    synchronized(this) {
        ProcessRecord callerApp = null;
        if (caller != null) {
            callerApp = getRecordForAppLocked(caller);
            ....
            ....
            ReceiverList rl = (ReceiverList)mRegisteredReceivers.get(receiver.asBinder());
            if (rl == null) {
                rl = new ReceiverList(this, callerApp, callingPid, callingUid,
                        userId, receiver);
                if (rl.app != null) {
                    rl.app.receivers.add(rl);//app代表ProcessRecord
                } else {
                    try {
                        receiver.asBinder().linkToDeath(rl, 0);
                    } catch (RemoteException e) {
                        return sticky;
                    }
                    rl.linkedToDeath = true;
                }
                mRegisteredReceivers.put(receiver.asBinder(), rl);
            } 
            BroadcastFilter bf = new BroadcastFilter(filter, rl, callerPackage,
                    permission, callingUid, userId);
            rl.add(bf);//在arraylist中加入了该filter
            mReceiverResolver.addFilter(bf);
            ...
        }
    }
```

![](/img/2017-03-07-broadcast/14888186491306.jpg)

再看`sendBroadcast`发送广播吧

```java
ActivityManagerNative.getDefault().broadcastIntent(
                mMainThread.getApplicationThread(), intent, resolvedType, null,
                Activity.RESULT_OK, null, null, null, AppOpsManager.OP_NONE, false, false,getUserId());
```

首先,通过`intentResolver`获取属于该`intent`的`BroadFilter list registeredReceivers`

```java
List receivers = null;
List<BroadcastFilter> registeredReceivers = null;
// Need to resolve the intent to interested receivers...
if ((intent.getFlags()&Intent.FLAG_RECEIVER_REGISTERED_ONLY)== 0) {
    receivers = collectReceiverComponents(intent, resolvedType, users);
}
if (intent.getComponent() == null) {
    registeredReceivers = mReceiverResolver.queryIntent(intent,resolvedType, false, userId);
}
        
int NR = registeredReceivers != null ? registeredReceivers.size() : 0;
if (!ordered && NR > 0) {
            // If we are not serializing this broadcast, then send the
            // registered receivers separately so they don't wait for the
            // components to be launched.
final BroadcastQueue queue = broadcastQueueForIntent(intent);
BroadcastRecord r = new BroadcastRecord(queue, intent, callerApp,
        callerPackage, callingPid, callingUid, resolvedType, requiredPermission,
                    appOp, registeredReceivers, resultTo, resultCode, resultData, map,ordered, sticky, false, userId);
                    
...                                                                                                                                                                                                                                                                                                                                          
final boolean replaced = replacePending &&queue.replaceParallelBroadcastLocked(r);
if (!replaced) {
    queue.enqueueParallelBroadcastLocked(r);
    queue.scheduleBroadcastsLocked();
}
registeredReceivers = null;
NR = 0;
}

```


根据`intent`找到`filters`,传递给`handler`,在`handler`中:

```java
final void processNextBroadcast(boolean fromMsg) {
while (mParallelBroadcasts.size() > 0) {
    r = mParallelBroadcasts.remove(0);
    r.dispatchTime = SystemClock.uptimeMillis();
    r.dispatchClockTime = System.currentTimeMillis();
    final int N = r.receivers.size();
    if (DEBUG_BROADCAST_LIGHT) Slog.v(TAG, "Processing parallel broadcast [" + mQueueName + "] " + r);
    for (int i=0; i<N; i++) {
        Object target = r.receivers.get(i);
        if (DEBUG_BROADCAST)  Slog.v(TAG,
            "Delivering non-ordered on [" + mQueueName + "] to registered "
                            + target + ": " + r);
            deliverToRegisteredReceiverLocked(r, (BroadcastFilter)target, false);
        }
        addBroadcastToHistoryLocked(r);
    }
}
```

r代表广播信息,`target`是广播的对象`filter`

```java
performReceiveLocked(filter.receiverList.app, filter.receiverList.receiver,new Intent(r.intent), r.resultCode, r.resultData,r.resultExtras, r.ordered, r.initialSticky, r.userId);
```

![](/img/2017-03-07-broadcast/14888189716968.jpg)

```java
                    
private static void performReceiveLocked(ProcessRecord app, IIntentReceiver receiver,Intent intent, int resultCode, String data, Bundle extras,boolean ordered, boolean sticky, int sendingUser) throws RemoteException {
        // Send the intent to the receiver asynchronously using one-way binder calls.
        if (app != null && app.thread != null) {
            // If we have an app thread, do the call through that so it is
            // correctly ordered with other one-way calls.
            app.thread.scheduleRegisteredReceiver(receiver, intent, resultCode,
                    data, extras, ordered, sticky, sendingUser, app.repProcState);
        } else {
            receiver.performReceive(intent, resultCode, data, extras, ordered,
                    sticky, sendingUser);
        }
    }
```


IPC通过receiver传递给调用者:

```java
        // This function exists to make sure all receiver dispatching is
        // correctly ordered, since these are one-way calls and the binder driver
        // applies transaction ordering per object for such calls.
public void scheduleRegisteredReceiver(IIntentReceiver receiver, Intent intent,int resultCode, String dataStr, Bundle extras, boolean ordered,boolean sticky, int sendingUser, int processState) throws RemoteException {
    updateProcessState(processState, false);
    receiver.performReceive(intent, resultCode, dataStr, extras, ordered,sticky, sendingUser);
}
```

先进入`IIntentReceiver.performReceive`,再抛到外层的`rd.performReceive`

```java
 public void performReceive(Intent intent, int resultCode, String data,Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
    LoadedApk.ReceiverDispatcher rd = mDispatcher.get();
    if (ActivityThread.DEBUG_BROADCAST) {
        int seq = intent.getIntExtra("seq", -1);
        Slog.i(ActivityThread.TAG, "Receiving broadcast " + intent.getAction() + " seq=" + seq + " to " + (rd != null ? rd.mReceiver : null));
    }
    if (rd != null) {
        rd.performReceive(intent, resultCode, data, extras,ordered, sticky, sendingUser);
    } 
    ...
}
        
public void performReceive(Intent intent, int resultCode, String data,Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
    if (ActivityThread.DEBUG_BROADCAST) {
        int seq = intent.getIntExtra("seq", -1);
        Args args = new Args(intent, resultCode, data, extras, ordered,sticky, sendingUser);
        if (!mActivityThread.post(args)) {
            if (mRegistered && ordered) {
                IActivityManager mgr = ActivityManagerNative.getDefault();
                if (ActivityThread.DEBUG_BROADCAST) Slog.i(ActivityThread.TAG,"Finishing sync broadcast to " + mReceiver);
                args.sendFinished(mgr);
            }
        }
    }
}

//最后进入Args.run()   
public Args(Intent intent, int resultCode, String resultData, Bundle resultExtras,
                    boolean ordered, boolean sticky, int sendingUser) {
    super(resultCode, resultData, resultExtras,mRegistered ? TYPE_REGISTERED : TYPE_UNREGISTERED,ordered, sticky, mIIntentReceiver.asBinder(), sendingUser);
    mCurIntent = intent;
    mOrdered = ordered;
}
            
    public void run() {
        final BroadcastReceiver receiver = mReceiver;
        final boolean ordered = mOrdered;
                
        final IActivityManager mgr = ActivityManagerNative.getDefault();
        final Intent intent = mCurIntent;
        mCurIntent = null;
        try {
            ClassLoader cl =  mReceiver.getClass().getClassLoader();
            intent.setExtrasClassLoader(cl);
            setExtrasClassLoader(cl);
            receiver.setPendingResult(this);
            receiver.onReceive(mContext, intent);
        } 
    ...
    }

```
与`service`类似,先生成`mReceiver`,再调用`onReceive`

根据图做个总结:

* 根据`filter`注册`receiver`时,创建对应的`dispatcher`和`InnerReceiver`用作`IPC`通信,一个`receiver`对应一个`dispatcher`,可以对应多个`filter`
* 在AMS中根据`innerReceiver`注册`map`,对应多个`Filter`,形成`ReceiverList`。即`ReceiverList`包含一个`receiver`对应的`filters`。
* send广播时,会对该广播形成一个`record`代表广播信息。然后通过该intent找到多个`filter`,`filter`找到对应的`receiverList`,因为`list`中包含该`filter`对应的`processRecord`和`InnerReceiver`
* 循环,通过`InnerReceiver`返回到`process`,注册的进程中,因为该`innterReceiver`中有相应的`BroadcastReceiver`,反射生成,调用`onReceive`即可。



