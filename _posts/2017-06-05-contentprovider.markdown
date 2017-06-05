---
layout:     post
title:      "ContentProvider分析"
date:       2017-06-05
author:     "Liz"
header-img: "img/school.jpg"
tags:
    - 四大组件
    - Android
    - 源码分析
---

转眼要毕业了,再过几天就要毕设答辩了,铭记我十多年的求学时光,感谢所有帮助过我的老师,砥砺前行吧

乘着答辩间隙分析一下`CP`的源码,之前实习时同事一直讨论不同`sdk`后`CursorWindow`泄露方的问题,借此机会研究一下。事实证明低版本和高版本确实不同,但也只是究竟在服务方还是调用方创建`CursorWindow`的区别,所以本质没什么区别。

[很清晰的介绍](http://blog.csdn.net/sinat_22657459/article/details/53323659)

### ContentProvider获取

我们看一个简单的使用例子:

```java
public void fetchAllContacts() {  
    ContentResolver contentResolver = this.getContentResolver();  
    Cursor cursor = contentResolver.query(android.provider.ContactsContract.Contacts.CONTENT_URI,  null, null, null, null);  
    cursor.getCount();  
    while(cursor.moveToNext()){        
    System.out.println(cursor.getString(cursor.getColumnIndex(android.provider.ContactsContract.Contacts._ID)));  
        System.out.println(cursor.getString(cursor.getColumnIndex(android.provider.ContactsContract.Contacts.DISPLAY_NAME)));  
    }  
    cursor.close();  
}  
```
在ContextImpl初始化时,

![](/img/2017-06-05-contentprovider/14965637004854.jpg)

![](/img/2017-06-05-contentprovider/14965635212328.jpg)

![](/img/2017-06-05-contentprovider/14965637994922.jpg)

![](/img/2017-06-05-contentprovider/14965639121555.jpg)


#### AMS.getContentProvider

最终使用`AMS`去拿到`ContentProvider`时, 这个函数首先会通过`getExistingProvider`函数来检查本地是否已经存在这个要获取的`ContentProvider`接口，如果存在，就直接返回了。本地已经存在的`ContextProvider`接口保存在`ActivityThread`类的`mProviderMap`成员变量中，以`ContentProvider`对应的`URI`的`authority`为键值保存。

>AMS.java

![](/img/2017-06-05-contentprovider/14965642020755.jpg)

注意`cpr,cpi`是关键变量。首先检查`mProviderMap`中是否有对应`authority`的`cpr`
![](/img/2017-06-05-contentprovider/14965646849696.jpg)

![](/img/2017-06-05-contentprovider/14965647885129.jpg)

如果找不到,通过`PMS`解析`CP`:

>PMS.java

![](/img/2017-06-05-contentprovider/14965649232019.jpg)

进入`PackageParser`去生成`ProviderInfo`,我们知道,在应用启动时,`PackageParser`会解析配置文件等,获取四大组件的信息,所以这时候`provider`是有内容的。

>AMS.java


同得到`cpi`后,构建`cpr`
![](/img/2017-06-05-contentprovider/14965655740385.jpg)

检查这个`provider`是否允许在客户进程中加载,还是需要新启进程
![](/img/2017-06-05-contentprovider/14965656162346.jpg)

如果需要新启进程,系统中正在加载的`CP`在`mLaunchingProvider`列表中

![](/img/2017-06-05-contentprovider/14965656898886.jpg)


使用`proc.thread`(调用方的`binderProxy`)来`installProvider`:看这个`CP`对应的进程是否已经启动,如果没有就先启动进程,如果启动了,就将该`CP`进行注册。通过获取`ProcessRecord`判断进程是否启动

![](/img/2017-06-05-contentprovider/14965657672527.jpg)

最后,因为在新进程启动,所以需要等待:直到新进程中设置了`cpr.provider`即可。


![](/img/2017-06-05-contentprovider/14965662266698.jpg)

#### 新进程启动

* ActivityManagerService.startProcessLocked
* Process.start (新进程)
* ActivityThread.main
* ActivityThread.attach
* ActivityManagerService.attachApplication (AMS做一些进程启动之后的微小工作。。)

在`attachAppliction`时,会去启动对应的`CP`:

>AMS.java
![](/img/2017-06-05-contentprovider/14965671671795.jpg)

![](/img/2017-06-05-contentprovider/14965671473241.jpg)

在`bindApplication`中:

>ActivityThread.java

![](/img/2017-06-05-contentprovider/14965674592374.jpg)

构造`AppBindData`,将该塞的信息都塞进去包括`provider`,然后`installProvider`

>Handler处理
![](/img/2017-06-05-contentprovider/14965676070828.jpg)

最后在新进程中,`installProvider`启动`CP`,同时通知`AMS`即可。又见到了熟悉的`classLoader.loadClass`,最后通过`publishContentProviders`通知`AMS bind Complete`

![](/img/2017-06-05-contentprovider/14965676661054.jpg)
![](/img/2017-06-05-contentprovider/14965676168519.jpg)



有一个地方值得关注,`provider = localProvider.getIcontentProvider()`,`localProvider`是新生成的`CP`

![](/img/2017-06-05-contentprovider/14965728382854.jpg)

返回的`mTransport`继承于`CPNative`,是一个本地`binder`:

![](/img/2017-06-05-contentprovider/14965728916994.jpg)

>ContentProvider类和Transport类的关系就类似于ActivityThread和ApplicationThread的关系，其它应用程序不是直接调用ContentProvider接口来访问它的数据，而是通过调用它的内部对象mTransport来间接调用ContentProvider的接口


最后调用真正子类的`onCreate`,而`AMS`唤醒之前睡的线程:

![](/img/2017-06-05-contentprovider/14965731340325.jpg)

最后返回给上层的是`mTransport`,用它来进行后续的查询插入等

### query

![](/img/2017-06-05-contentprovider/14966445410100.jpg)

在拿到`CP`接口后,我们正式查询

#### ContentProviderProxy

![](/img/2017-06-05-contentprovider/14965739442683.jpg)

进入真正的`Transport`(`ContentProviderNative`)进行查询:先检查这个包是否有读的权限,进入自定义的`CP`查询:

![](/img/2017-06-05-contentprovider/14965761496122.jpg)


正常套路:
![](/img/2017-06-05-contentprovider/14965762009913.jpg)

![](/img/2017-06-05-contentprovider/14965762478726.jpg)

![](/img/2017-06-05-contentprovider/14965762881173.jpg)

一步步深入,最终进入`SQLiteCursorDriver`中查询:


![](/img/2017-06-05-contentprovider/14965763534665.jpg)
可见服务端对应的为`SQLiteCursor`

这一步执行完成之后，就把这个`SQLiteCursor`对象返回给上层
我们看,在服务端拿到这个`SQLiteCursor`后,封装给`CursorToBulkCursorAdaptor`,同时将`BulkCursorDescriptor`返回

![](/img/2017-06-05-contentprovider/14965764384409.jpg)

在客户端:

![](/img/2017-06-05-contentprovider/14965765281873.jpg)

这个`BulkCursorDescriptor`其实是一个`Parceable`

![](/img/2017-06-05-contentprovider/14965766559477.jpg)

有`setWindow`,在服务端一定有`getWindow`:

![](/img/2017-06-05-contentprovider/14965767523881.jpg)


再看
![](/img/2017-06-05-contentprovider/14965772016818.jpg)
所以最终应用层拿到的是一个`CursorWrapperInner`,而`qCursor`是` BulkCursorToCursorAdaptor`

![](/img/2017-06-05-contentprovider/14965775897653.jpg)

![](/img/2017-06-05-contentprovider/14965776602844.jpg)
很关键的函数:

![](/img/2017-06-05-contentprovider/14965777082999.jpg)
`setWindow(mBulkCursor.getWindow(newPosition));`获取`window`

>在这里我们重点说一下，在Provider通信过程中，binder通信直接对应的两端的对象：客户端是BulkCursorToCursorAdaptor，服务端是CursorToBulkCursorAdaptor。往外退一层，应用层访问的Cursor实质是封装的一个CursorWrapperInner对象，服务端实际构造好的是一个SQLiteCursor对象。


>Content Provider首先会创建一个SQLiteCursor对象，即SQLite数据库游标对象，它继承了AbstractWindowedCursor类，后者又继承了AbstractCursor类，而AbstractCursor类又实现了CrossProcessCursor和Cursor接口。其中，最重要的是在AbstractWindowedCursor类中，有一个成员变量mWindow，它的类型为CursorWindow，这个成员变量是通过AbstractWindowedCursor的子类SQLiteCursor的setWindow成员函数来设置的。这个SQLiteCursor对象设置好了父类AbstractWindowedCursor类的mWindow成员变量之后，它就具有传输数据的能力了，因为这个mWindow对象内部包含一块匿名共享内存。
     
`CursorWindow`是一个`Parcelable`对象，可跨进程传输。所以就从服务端传递到了客户端，`CursorWindow`中`mWindowPtr`就是匿名共享内存指针


`CursorToBulkCursorAdaptor`是一个本地`binder`:

![](/img/2017-06-05-contentprovider/14965779049868.jpg)
#### new CursorWindow

CursorWindow对象，它在内部创建了一块匿名共享内存，同时，它实现了Parcel接口，因此它可以在进程间传输。

![](/img/2017-06-05-contentprovider/14965779750380.jpg)
![](/img/2017-06-05-contentprovider/14966485828871.jpg)
C++层创建匿名共享内存
#### fillWindow

在服务端,`SQLiteCursor.fillWindow`

>SQLiteCursor对象通过调用成员变量mQuery的fillWindow成员函数来把从SQLite数据库中查询得到的数据保存其父类AbstractWindowedCursor的成员变量mWindow中去，即保存到第三方应用程序创建的这块匿名共享内存中去。

其实就是通过`fillWindow`填充`window`,把查询得到的结果放到匿名共享内存中去。在`SQLiteQuery`中`fillWindow`->`getSession().executeForCursorWindow`->`mConnection.executeForCursorWindow`->`nativeExecuteForCursorWindow`->`CopyRowResult cpr = copyRow(env, window, statement, numColumns, startPos, addedRows);`

反正通过`sqlite`引擎获取查询结果,然后填充`window`:

![](/img/2017-06-05-contentprovider/14966662552709.jpg)
这里`window->putLong`...等就是填充`window`,之后查询时上层通过`getLong`也是从`window`中获取。

### ContentObserver

跟广播差不多,使用`ContentService`来做中间层转发。

![](/img/2017-06-05-contentprovider/14966384107480.jpg)

![](/img/2017-06-05-contentprovider/14966385122217.jpg)
返回的就是binder接口:

>ContentObserver类的成员变量mTransport是一个Binder对象，它是要传递给ContentService服务的，以便当ContentObserver所监控的数据发生变化时，ContentService服务可以通过这个Binder对象通知相应的ContentObserver它监控的数据发生变化了。

##### register

![](/img/2017-06-05-contentprovider/14966510528888.jpg)
关键注册函数,

![](/img/2017-06-05-contentprovider/14966511469638.jpg)
整个注册是一个树形结构,`getUriSegment`获取`uri`中的第`index`个部分,其实就是按`/`分割后一步步创建子节点罢了(递归)

#### resolver.notifyChange(newUri, null);

继续使用`ContentService.nn`


![](/img/2017-06-05-contentprovider/14966512745357.jpg)
`collectObserversLocked`获取`uri`对应的`observer`,然后通过`IContentObserver`接口通知改变,在`Transport`中分发至真正的`observer`的`onChange即可`


