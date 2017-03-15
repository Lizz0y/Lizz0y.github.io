---
layout:     post
title:      "DroidPlugin(一)"
date:       2017-03-12
author:     "Liz"
header-img: "img/2017-02-01-binder/pic.jpeg"
tags:
    - Android
    - 插件
    - DroidPlugin
---

目前的市场让我觉得应该静下心研究一下插件,热补丁等热门的东西,虽然`apple`刚宣布禁止`JPatch`等,但`android`还是如此自由浪漫。

对于插件,该阶段的目标是`DroidPlugin`,下阶段是`small`,共勉。

`DroidPlugin`把`weishu`的文章都看了一遍,受益匪浅。

本篇先讲个大体框架,细节后续补充

#### 基本框架

以官方`demo`介绍一下整体框架。

`pluginApplication`作为入口,在`onCreate`时,初始化各个`Manager`

* `PluginPatchManager`:处理插件异常情况
* `installHook`: 注册所有hook点
* `PluginHelper`:实现ServiceConnection
* `PluginManager`:实现ServiceConnection,带上PluginHelper这个`connection`,同时绑定`PluginManagerService`服务,该全局服务提供插件管理。
* `PluginManagerService`:初始化`IPluginManagerImpl`binder实体对象,该对象模拟系统的`PackageManagerService`，对插件进行简单的管理服务。


其实上述不了解也无所谓的,只是了解了更加清晰一点而已

![](/img/2017-03-15-droidplugin-1/14893024502061.jpg)

##### 插件注册启动

`在IPluginManagerImpl`中进行`install`



```java
PluginPackageParser parser = new PluginPackageParser(mContext, new File(apkfile));
parser.collectCertificates(0);
PackageInfo pkgInfo = parser.getPackageInfo(PackageManager.GET_PERMISSIONS | PackageManager.GET_SIGNATURES);
if (pkgInfo != null && pkgInfo.requestedPermissions != null && pkgInfo.requestedPermissions.length > 0) {
    for (String requestedPermission :pkgInfo.requestedPermissions) {
        boolean b = false;
        try {
            b = pm.getPermissionInfo(requestedPermission, 0) != null;
        } catch (NameNotFoundException e) {
    }
    if (!mHostRequestedPermission.contains(requestedPermission) && b{
        Log.e(TAG, "No Permission %s", requestedPermission);
        new File(apkfile).delete();
        return PluginManager.INSTALL_FAILED_NO_REQUESTEDPERMISSION;
    }
}
```

关键代码:

```java
PluginPackageParser parser = new PluginPackageParser(mContext, new File(apkfile));
```

一步步分析:

```java
 public PluginPackageParser(Context hostContext, File pluginFile) throws Exception {
        mHostContext = hostContext;
        mPluginFile = pluginFile;
        mParser = PackageParser.newPluginParser(hostContext);
        mParser.parsePackage(pluginFile, 0);
        mPackageName = mParser.getPackageName();
        mHostPackageInfo = mHostContext.getPackageManager().getPackageInfo(mHostContext.getPackageName(), 0);
```

* 根据api版本获取自定义的PackageParser,这其实挺恶心的,每出一个版本就要新写一个parser = =,然后调用parsePackage进行处理。

![](/img/2017-03-15-droidplugin-1/14893033591097.jpg)


可以看到主要是对`androidManifest`进行处理,分析除了四大组件和一些基本信息。

* 对前面分析出的组件进行处理,将`activity`,`ActivityInfo`,`intenetFilter`以`componentName`为`key`放入三个`cache`中

![](/img/2017-03-15-droidplugin-1/14893036084724.jpg)


其余`service`,`provider`,`broadcast`同理。

还附加了`mPermissionsObjCache,mPermissionsGroupCache,mRequestedPermissionsCache` 这个具体干啥还不清楚。。。。

回到最初的`install`:

![](/img/2017-03-15-droidplugin-1/14893039437996.jpg)

这个主要是看插件申请的`Permmission`是否在宿主`permission`范围内,如果超过了就要抛异常啦


总结:

![](/img/2017-03-15-droidplugin-1/14893027914542.jpg)


#### 替换LoadedApk
这是因为`classLoader`加载`class`时,如果是宿主的`classLoader`是找不到插件类的

```java
ActivityClientRecord r = (ActivityClientRecord)msg.obj;
r.packageInfo = getPackageInfoNoCheck(r.activityInfo.applicationInfo, r.compatInfo);
handleLaunchActivity(r, null);
java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
```

所以我们要替换`r.packageInfo`,干脆直接替换`packageInfo`吧。。

```java
WeakReference<LoadedApk> ref;
if (includeCode) {
    ref = mPackages.get(aInfo.packageName);
}
```

老样子,直接替换`mPackage`这个`map`吧


```java
public static void preLoadApk(Context hostContext, ComponentInfo pluginInfo) throws IOException, NoSuchMethodException, IllegalAccessException, InvocationTargetException, PackageManager.NameNotFoundException, ClassNotFoundException {
    if (pluginInfo == null && hostContext == null) {
        return;
    }
    if (pluginInfo != null && getPluginContext(pluginInfo.packageName) != null) {
        return;
    }
    /*添加插件的LoadedApk对象到ActivityThread.mPackages*/

    boolean found = false;
    synchronized (sPluginLoadedApkCache) {
        Object object = ActivityThreadCompat.currentActivityThread();
        if (object != null) {
            Object mPackagesObj = FieldUtils.readField(object, "mPackages");
            Object containsKeyObj = MethodUtils.invokeMethod(mPackagesObj, "containsKey", pluginInfo.packageName);
            //如果不包括packageName这个LoadedApk
            if (containsKeyObj instanceof Boolean && !(Boolean) containsKeyObj) {
                final Object loadedApk;
                if (VERSION.SDK_INT >= VERSION_CODES.HONEYCOMB) {
                    loadedApk = MethodUtils.invokeMethod(object, "getPackageInfoNoCheck", pluginInfo.applicationInfo, CompatibilityInfoCompat.DEFAULT_COMPATIBILITY_INFO());
                } else {
                    loadedApk = MethodUtils.invokeMethod(object, "getPackageInfoNoCheck", pluginInfo.applicationInfo);
                }
                sPluginLoadedApkCache.put(pluginInfo.packageName, loadedApk);
            /*添加ClassLoader LoadedApk.mClassLoader*/
                String optimizedDirectory = PluginDirHelper.getPluginDalvikCacheDir(hostContext, pluginInfo.packageName);
                String libraryPath = PluginDirHelper.getPluginNativeLibraryDir(hostContext, pluginInfo.packageName);
                String apk = pluginInfo.applicationInfo.publicSourceDir;
                if (TextUtils.isEmpty(apk)) {
                    pluginInfo.applicationInfo.publicSourceDir = PluginDirHelper.getPluginApkFile(hostContext, pluginInfo.packageName);
                    apk = pluginInfo.applicationInfo.publicSourceDir;
                }
                if (apk != null) {
                //new 一个classLoader
                    ClassLoader classloader = null;
                    try {
                        classloader = new PluginClassLoader(apk, optimizedDirectory, libraryPath, hostContext.getClassLoader().getParent());
                    } catch (Exception e) {
                }
                if(classloader==null){
                    PluginDirHelper.cleanOptimizedDirectory(optimizedDirectory);
                    classloader = new PluginClassLoader(apk, optimizedDirectory, libraryPath, hostContext.getClassLoader().getParent());
                }
                //loadedApk.mClassLoader = new classLoader
                synchronized (loadedApk) {
                    FieldUtils.writeDeclaredField(loadedApk, "mClassLoader", classloader);
                }
                sPluginClassLoaderCache.put(pluginInfo.packageName, classloader);
                Thread.currentThread().setContextClassLoader(classloader);
                found = true;
            }
                ProcessCompat.setArgV0(pluginInfo.processName);
            }
        }
    }
    if (found) {
        PluginProcessManager.preMakeApplication(hostContext, pluginInfo);
    }
}
//这里还不是很懂,待定分析
private static void preMakeApplication(Context hostContext, ComponentInfo pluginInfo) {
    try {
        final Object loadedApk = sPluginLoadedApkCache.get(pluginInfo.packageName);
        if (loadedApk != null) {
            Object mApplication = FieldUtils.readField(loadedApk, "mApplication");
            if (mApplication != null) {
                    return;
            }
            if (Looper.getMainLooper() != Looper.myLooper()) {
                final Object lock = new Object();
                mExec.set(false);
                sHandle.post(new Runnable() {
                    @Override
                    public void run() {
                        try {
                            MethodUtils.invokeMethod(loadedApk, "makeApplication", false, ActivityThreadCompat.getInstrumentation());
                        } catch (Exception e) {
                            Log.e(TAG, "preMakeApplication FAIL", e);
                        } finally {
                            mExec.set(true);
                            synchronized (lock) {
                                lock.notifyAll();
                            }
                        }
                     }
                });
                if (!mExec.get()) {
                    synchronized (lock) {
                        try {
                            lock.wait();
                        } catch (InterruptedException e) {
                    }
                }
            }
        } else {
             MethodUtils.invokeMethod(loadedApk, "makeApplication", false, ActivityThreadCompat.getInstrumentation());
        }
        }
    } catch (Exception e) {
        Log.e(TAG, "preMakeApplication FAIL", e);
    }
    ```

#### stub
为了完美的运行插件,宿主进程需要先定义一些`stub`。
什么是`stub`:
以activity为例,因为之前提过四大组件都需要在`manifest`中注册,但我们事先又不知道插件会用什么组件,所以为了瞒天过海,我们先在宿主进程中定义超多组件,

![](/img/2017-03-15-droidplugin-1/14893091122357.jpg)


如上图,预定义在`PluginP01`进程中的`ActivityStub,p01`进程,`SingleTop`的`mode`。

这样在运行插件时,可以用对应的activitystub去ams中注册,然后启动时瞒天过海替换成`realActivity`

在前面提到的`MyActivityManagerService`启动时,会初始化`StaticProcessList`。

初始化即遍历`manifest`中的四大组件,根据`process`初始化`ProcessItem`

![](/img/2017-03-15-droidplugin-1/14895015269438.jpg)


注意的是,为了不把宿主的activity加入,作者在`stub activity`中加入了`meta-data`做为标记:
`List<ResolveInfo> activities = pm.queryIntentActivities(intent, PackageManager.GET_META_DATA);`


#### IActivityManagerHook

插件中最关键的一个`hook`。

Hook不多说了,和之前[代理机制](http://lizz0y.me/2017/02/26/Proxy/)原理一样,使用动态代理进行hook。

`DroidPlugin`对接口的每一个方法都定义了一个handler
##### onInstall
```java
Class cls = ActivityManagerNativeCompat.Class();
Object obj = FieldUtils.readStaticField(cls, "gDefault");
if (obj == null) {
    ActivityManagerNativeCompat.getDefault();
    obj = FieldUtils.readStaticField(cls, "gDefault");
}
if (IActivityManagerCompat.isIActivityManager(obj)) {
    setOldObj(obj);
    Class<?> objClass = mOldObj.getClass();
    List<Class<?>> interfaces = Utils.getAllInterfaces(objClass);
    Class[] ifs = interfaces != null && interfaces.size() > 0 ? interfaces.toArray(new Class[interfaces.size()]) : new Class[0];
    Object proxiedActivityManager = MyProxy.newProxyInstance(objClass.getClassLoader(), ifs, this);
    FieldUtils.writeStaticField(cls, "gDefault", proxiedActivityManager);
    Log.i(TAG, "Install ActivityManager Hook 1 old=%s,new=%s", mOldObj, proxiedActivityManager);
```
分两步,`android`源码中旧的`sdk`中`IActivityManager`直接通过`gDefault`就能拿到;在高版本`sdk`
中保存在`Singleton`中,所以要做个判断
```java
else if (SingletonCompat.isSingleton(obj)) {
    Object obj1 = FieldUtils.readField(obj, "mInstance");
    if (obj1 == null) {
        SingletonCompat.get(obj);
        obj1 = FieldUtils.readField(obj, "mInstance");
    }
```


##### startService

这时候过去分析源码的好处就出来了,怒摔！

```java
private static class startService extends ReplaceCallingPackageHookedMethodHandler {

    public startService(Context hostContext) {
        super(hostContext);
    }

    private ServiceInfo info = null;
    @Override
    protected boolean beforeInvoke(Object receiver, Method method, Object[] args) throws Throwable {
            //API 2.3, 15, 16
        /*    public ComponentName startService(IApplicationThread caller, Intent service,
            String resolvedType) throws RemoteException;*/

            //API 17, 18, 19, 21
        /*public ComponentName startService(IApplicationThread caller, Intent service,
            String resolvedType, int userId) throws RemoteException;*/
        info = replaceFirstServiceIntentOfArgs(args);
        return super.beforeInvoke(receiver, method, args);
    }

    @Override
    protected void afterInvoke(Object receiver, Method method, Object[] args, Object invokeResult) throws Throwable{
        if (invokeResult instanceof ComponentName) {
            if (info != null) {
                setFakedResult(new ComponentName(info.packageName, info.name));
            }
        }
        info = null;
        super.afterInvoke(receiver, method, args, invokeResult);
    }
}
```
其实也不复杂,在`startService`之前先进行`before`预处理,那就先来看看做了哪些预处理的努力:

* `ServiceInfo serviceInfo = resolveService(intent);`

```java
public ServiceInfo getServiceInfo(ComponentName className, int flags) throws Exception {
    Object data;
    synchronized (mServiceObjCache) {
        data = mServiceObjCache.get(className);
    }
    if (data != null) {
        ServiceInfo serviceInfo = mParser.generateServiceInfo(data, flags);
        fixApplicationInfo(serviceInfo.applicationInfo);
        if (TextUtils.isEmpty(serviceInfo.processName)) {
            serviceInfo.processName = serviceInfo.packageName;}
        return serviceInfo;
    }
    return null;
}
```
从前面`parse`时填入的`Cache`中取出`ServiceInfo`



* `ServiceInfo proxyService = selectProxyService(intent);`

最重要的一步。

1. `runProcessGC();`
不懂为什么要先GC一次,先放着。。

2. 看正在运行的进程是否有符合条件的进程 

![](/img/2017-03-15-droidplugin-1/14893076033789.jpg)


3. 使用`stub`

在宿主进程中,定义了无数的`activity,service`和`provider`。

![](/img/2017-03-15-droidplugin-1/14893087825153.jpg)


* 遍历Process,即`:PluginP04...`
* 如果进程正在运行,并且没有任何插件包,则查找未被占用的`stubinfo`,并设置该`processItem`的`targetProcess`为`targetInfo.processName`
* 如果进程正在运行，并且有插件包了,若已有插件与`target`的`process`相同,并且`checkSignatures`通过,则返回可以在当前插件进程运行。

检查`signature`应该是看是不是同一个`app`吧。。。

```java
for (String pkg : item.pkgs) {
    if (PluginManager.getInstance().checkSignatures(packageName, pkg) == PackageManager.SIGNATURE_MATCH) {
        signed = true;
        break;
    }
}
if (signed) {
    return true;
}
```

* 替换`intent`

```java
newIntent.setAction(proxyService.name + new Random().nextInt());            
newIntent.setClassName(proxyService.packageName, proxyService.name);
newIntent.putExtra(Env.EXTRA_TARGET_INTENT, intent);
newIntent.setFlags(intent.getFlags());
args[intentOfArgIndex] = newIntent;
```

将原`intent`放入`EXTRA_TARGET_INTENT中`,方便后续替换回来

#### 返璞归真

![](/img/2017-03-15-droidplugin-1/14895033224570.jpg)

![](/img/2017-03-15-droidplugin-1/14895034440931.jpg)


如果`service == null,handleCreateServiceOne`
否则直接`handleOnStartOne`

##### `handleCreateServiceOne`关键函数 

在源码中,在`handleMessage`这一步`createService`时:
```java
public final void scheduleCreateService(IBinder token,
                ServiceInfo info, CompatibilityInfo compatInfo, int processState){
    updateProcessState(processState, false);
    CreateServiceData s = new CreateServiceData();
    s.token = token;
    s.info = info;
    s.compatInfo = compatInfo;
    sendMessage(H.CREATE_SERVICE, s);
}
```

所以在替换回去时我们也要这么`create`,利用反射生成`service`,注意要先`preLoadApk`替换`LoadedApk`,最后因为源码中将生成的`service`放在了`mServices`中,所以我们可以读取出来拿到service
![](/img/2017-03-15-droidplugin-1/14895035598358.jpg)

最后加入`runningProcess`中:

```java
void addServiceInfo(int pid, int uid, ServiceInfo stubInfo, ServiceInfo targetInfo) {
    ProcessItem item = items.get(pid);
    if (TextUtils.isEmpty(targetInfo.processName)) {
        targetInfo.processName = targetInfo.packageName;
    }
    if (item == null) {
        item = new ProcessItem();
        item.pid = pid;
        item.uid = uid;
        items.put(pid, item);
    }
    item.stubProcessName = stubInfo.processName;
    if (!item.pkgs.contains(targetInfo.packageName)) {
        item.pkgs.add(targetInfo.packageName);
    }
    item.targetProcessName = targetInfo.processName;
    item.addServiceInfo(stubInfo.name, targetInfo);
}
```

### 总结

总结一下

* 全局使用动态代理作`hook`
* `hook`了`AMS`来对四大组件进行插件化
* 预先注册多个组件和进程,采用替身的方式解决`security`检查
* 使用替换`LoadedApk`来解决`classLoader`加载插件问题





