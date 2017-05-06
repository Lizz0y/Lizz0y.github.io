---
layout:     post
title:      "DroidPlugin系列之二"
date:       2017-05-05
author:     "Liz"
header-img: "img/2017-02-01-binder/pic.jpeg"
tags:
    - 插件化
    - DroidPlugin
---

事实证明我已经忘了一写了啥了对我就是这么健忘,欧耶(大概是最近毕设做傻了

这次宏观上有了很充分的理解,所以也许会和第一篇有所重复

### build.gradle

```java
defaultConfig{
    // 建议改为自己的 packageName + .droidplugin_stub ，防止跟其它本插件使用者冲突
    def authorityName = "com.morgoo.droidplugin_stub"
    buildConfigField "String", "AUTHORITY_NAME", "\"${authorityName}\""
    manifestPlaceholders = [
        authorityName:"${authorityName}",
    ]
}
```
   
我们知道在这里设置的buildConfigField会在BuildConfig.java中生成相应变量, 
### androidManifest

#### 预申请权限

#### 四大组件stub

P00代表进程0。**注意meta-data**

![](/img/2017-05-05-droidplugin2/14936293919569.jpg)

![](/img/2017-05-05-droidplugin2/14936294934628.jpg)

![](/img/2017-05-05-droidplugin2/14936295424604.jpg)

注意这里`provider`的`authorities`是`${authorityName}`,即前面`build.gradle`定义的

### 初始化

初始化过程我记得在一中讲过了[DroidPlugin一](http://lizz0y.me/2017/03/12/Droidplugin-1/)

### ContentProvider

#### IContentProviderInvokeHandle

可以看到`proxy`的`beforeInvoke`中:

```java
@Override
protected boolean beforeInvoke(Object receiver, Method method, Object[] args) throws Throwable {
//local是真实的provider,stub是替代provider
    if (!mLocalProvider && mStubProvider != null) {
        //找到url
        final int index = indexFirstUri(args);
        if (index >= 0) {
            Uri uri = (Uri) args[index];
            //找到authority
            String authority = uri.getAuthority();
            if (!TextUtils.equals(authority, mStubProvider.authority)) {
                Uri.Builder b = new Builder();
                b.scheme(uri.getScheme());
                //把authority设为stub
                b.authority(mStubProvider.authority);
                b.path(uri.getPath());
                b.query(uri.getQuery());
                //把原本的authority作为参数传过去
                b.appendQueryParameter(Env.EXTRA_TARGET_AUTHORITY, authority);
                b.fragment(uri.getFragment());
                args[index] = b.build();
            }
        }
    }
    return super.beforeInvoke(receiver, method, args);
    }
}

```

#### AbstractContentProviderStub

以`query`为例:

```java

@Override
public Cursor query(Uri uri, String[] projection,
                        String selection, String[] selectionArgs, String sortOrder) {
    //真的要查时,把之前传入的参数拿出来
    String targetAuthority = uri.getQueryParameter(Env.EXTRA_TARGET_AUTHORITY);
    if (!TextUtils.isEmpty(targetAuthority) && !TextUtils.equals(targetAuthority, uri.getAuthority())) {
        ContentProviderClient client = getContentProviderClient(targetAuthority);
        try {
        //让真实的去query吧
            return client.query(buildNewUri(uri, targetAuthority), projection, selection, selectionArgs, sortOrder);
        } catch (RemoteException e) {
            handleExpcetion(e);
        }
    }
    return null;
}
private synchronized ContentProviderClient getContentProviderClient(final String targetAuthority) {
    ContentProviderClient client = sContentProviderClients.get(targetAuthority);
    if (client != null) {
        return client;
    }
    if (Looper.getMainLooper() != Looper.myLooper()) {
        PluginManager.getInstance().waitForConnected();
    }
    ProviderInfo stubInfo = null;
    ProviderInfo targetInfo = null;
    try {
        String authority = getMyAuthority();
        stubInfo = getContext().getPackageManager().resolveContentProvider(authority, 0);
        //得到真实的providerInfo
        targetInfo = PluginManager.getInstance().resolveContentProvider(targetAuthority, 0);
    } catch (Exception e) {
        Log.e(TAG, "Can not reportMyProcessName on ContentProvider");
    }
    if (stubInfo != null && targetInfo != null) {
        try {
            PluginManager.getInstance().reportMyProcessName(stubInfo.processName, targetInfo.processName, targetInfo.packageName);
        } catch (RemoteException e) {
            Log.e(TAG, "RemoteException on reportMyProcessName", e);
        }
    }
    try {
        if (targetInfo != null) {
            PluginProcessManager.preLoadApk(getContext(), targetInfo);
        }
    } catch (Exception e) {
        handleExpcetion(e);
    }
    //去创建真实的provider
    ContentProviderClient newClient = mContentResolver.acquireContentProviderClient(targetAuthority);
    sContentProviderClients.put(targetAuthority, newClient);
    try {
        if (stubInfo != null && targetInfo != null) {
            //加入到runningProvider中去
            PluginManager.getInstance().onProviderCreated(stubInfo, targetInfo);
        }
    } catch (Exception e) {
        Log.e(TAG, "Exception on report onProviderCreated", e);
    }
    return sContentProviderClients.get(targetAuthority);
}
```

看过`weishu`的文章都知道,CP的替换是 将url中的authority替换为stub然后最终在真正查询的地方替换回来即可。


### Service

#### AbstractServiceStub

因为service与Activity不同,可以共用:

```java @Override
public void onStart(Intent intent, int startId) {
    try {
        if (intent != null) {
            if (intent.getBooleanExtra("ActionKillSelf", false)) {
               ...
            } else {
                mCreator.onStart(this, intent, 0, startId);
            }
        }
    } catch (Throwable e) {
        handleException(e);
    }
    ...
}
```

#### ServcesManager

```java
public int onStart(Context context, Intent intent, int flags, int startId) throws Exception {
    Intent targetIntent = intent.getParcelableExtra(Env.EXTRA_TARGET_INTENT);
    if (targetIntent != null) {
        ServiceInfo targetInfo = PluginManager.getInstance().resolveServiceInfo(targetIntent, 0);
        if (targetInfo != null) {
            Service service = mNameService.get(targetInfo.name);
            if (service == null) {
                handleCreateServiceOne(context, intent, targetInfo);
            }
            handleOnStartOne(targetIntent, flags, startId);
        }
    }
    return -1;
}
```
##### handleCreateServiceOne

关键函数，其实和weishu介绍的一样 这里是第一次创建`service`

```java
private void handleCreateServiceOne(Context hostContext, Intent stubIntent, ServiceInfo info) throws Exception {
    //拿到真实ResolveInfo
    ResolveInfo resolveInfo = hostContext.getPackageManager().resolveService(stubIntent, 0);
    ServiceInfo stubInfo = resolveInfo != null ? resolveInfo.serviceInfo : null;
    PluginManager.getInstance().reportMyProcessName(stubInfo.processName, info.processName, info.packageName);
    //preLoad
    PluginProcessManager.preLoadApk(hostContext, info);
    Object activityThread = ActivityThreadCompat.currentActivityThread();
    //以下主要是在createService
    IBinder fakeToken = new MyFakeIBinder();
    Class CreateServiceData = Class.forName(ActivityThreadCompat.activityThreadClass().getName() + "$CreateServiceData");
    Constructor init = CreateServiceData.getDeclaredConstructor();
    if (!init.isAccessible()) {
        init.setAccessible(true);
    }
    Object data = init.newInstance();
    FieldUtils.writeField(data, "token", fakeToken);
    FieldUtils.writeField(data, "info", info);
    if (VERSION.SDK_INT >= VERSION_CODES.HONEYCOMB) {
        FieldUtils.writeField(data, "compatInfo", CompatibilityInfoCompat.DEFAULT_COMPATIBILITY_INFO());
    }
    Method method = activityThread.getClass().getDeclaredMethod("handleCreateService", CreateServiceData);
    if (!method.isAccessible()) {
        method.setAccessible(true);
    }
    method.invoke(activityThread, data);
    Object mService = FieldUtils.readField(activityThread, "mServices");
    Service service = (Service) MethodUtils.invokeMethod(mService, "get", fakeToken);
    MethodUtils.invokeMethod(mService, "remove", fakeToken);
    mTokenServices.put(fakeToken, service);
    mNameService.put(info.name, service);

    if (stubInfo != null) {
         PluginManager.getInstance().onServiceCreated(stubInfo, info);
    }

}
 ```
   
##### handleOnStartOne

这里是`start`
    
```java
private void handleOnStartOne(Intent intent, int flags, int startIds) throws Exception {
    ServiceInfo info = PluginManager.getInstance().resolveServiceInfo(intent, 0);
    if (info != null) {
        Service service = mNameService.get(info.name);
        if (service != null) {
            ClassLoader classLoader = getClassLoader(info.applicationInfo);
            intent.setExtrasClassLoader(classLoader);
            Object token = findTokenByService(service);
            Integer integer = mServiceTaskIds.get(token);
            if (integer == null) {
                integer = -1;
            }
            int startId = integer + 1;
            mServiceTaskIds.put(token, startId);
            int res = service.onStartCommand(intent, flags, startId);
            QueuedWorkCompat.waitToFinish();
        }
    }
}

```

我们以`IActivityManagerHookHandle`的`stopService`为例:
不去停止stub,而是直接停止自己

```java
protected boolean beforeInvoke(Object receiver, Method method, Object[] args) throws Throwable {
       
        int index = 1;
        if (args != null && args.length > index && args[index] instanceof Intent) {
            Intent intent = (Intent) args[index];
            ServiceInfo info = resolveService(intent);
            if (info != null && isPackagePlugin(info.packageName)) {
                int re = ServcesManager.getDefault().stopService(mHostContext, intent);
                setFakedResult(re);
                return true;
            }
        }
        return super.beforeInvoke(receiver, method, args);
    }
}
```

//todo setExtrasClassLoader是干啥的？？

### pm

我们看到PM能做的真的好多,罗列一下:

```java
ActivityInfo generateActivityInfo(ComponenetName,flag)
PackageInfo generatePackageInfo（PackageName,flag....)
List<PermissionInfo> generatePermissionInfo(ComponentName,flag)
generateApplicationInfo(flags)
getActivities() ->mActivities
permissions
```
* 注意这些得到以后,像ApplicationInfo的一些dir文件目录名就要换成插件自己的
* 注意Receiver也是ActivityInfo

### IntentMatcher

### hook

#### Instrumentation替换

![](/img/2017-05-05-droidplugin2/14937025759028.jpg)
#### handler替换

![](/img/2017-05-05-droidplugin2/14937025914875.jpg)
#### AMS替换

![](/img/2017-05-05-droidplugin2/14937026966584.jpg)

![](/img/2017-05-05-droidplugin2/14937027793235.jpg)
#### 各类Service替换

先替换getService拿到的`IBinder`
![](/img/2017-05-05-droidplugin2/14937095876949.jpg)
#### 再替换Ibinder.queryLocalInterface方法

![](/img/2017-05-05-droidplugin2/14937099396109.jpg)

### handle

各类真正实现`before/after Invoke`的地方

以AMS为关键:

大部分都是把包名换成宿主包,关键函数如下:

#### startActivities

```java


protected boolean doReplaceIntentForStartActivityAPIHigh(Object[] args) throws RemoteException {
    int intentOfArgIndex = findFirstIntentIndexInArgs(args);
    if (args != null && args.length > 1 && intentOfArgIndex >= 0) {
        Intent intent = (Intent) args[intentOfArgIndex];
        //XXX String callingPackage = (String) args[1];
        if (!PluginPatchManager.getInstance().canStartPluginActivity(intent)) {
            PluginPatchManager.getInstance().startPluginActivity(intent);
            return false;
        }
        ActivityInfo activityInfo = resolveActivity(intent);
        if (activityInfo != null && isPackagePlugin(activityInfo.packageName)) {
        //挑选stub
            ComponentName component = selectProxyActivity(intent);
            if (component != null) {
                Intent newIntent = new Intent();
                try {
                    //TODO
                    ClassLoader pluginClassLoader = PluginProcessManager.getPluginClassLoader(component.getPackageName());
                    setIntentClassLoader(newIntent, pluginClassLoader);
                } catch (Exception e) {
                    Log.w(TAG, "Set Class Loader to new Intent fail", e);
                }
                //插入target参数
                newIntent.setComponent(component);
                newIntent.putExtra(Env.EXTRA_TARGET_INTENT, intent);
                newIntent.setFlags(intent.getFlags());
                String callingPackage = (String) args[1];
                if (TextUtils.equals(mHostContext.getPackageName(), callingPackage)) {
                    newIntent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
                    args[intentOfArgIndex] = newIntent;
                    args[1] = mHostContext.getPackageName();
                } else {
                    Log.w(TAG, "startActivity,replace selectProxyActivity fail");
                }
            }
        }
        return true;
    }
//并不知道这个在干啥。。
private void setIntentClassLoader(Intent intent, ClassLoader classLoader) {
    try {
        Bundle mExtras = (Bundle) FieldUtils.readField(intent, "mExtras");
        if (mExtras != null) {
            mExtras.setClassLoader(classLoader);
        } else {
            Bundle value = new Bundle();
            value.setClassLoader(classLoader);
            FieldUtils.writeField(intent, "mExtras", value);
        }
    } catch (Exception e) {
    } finally {
        intent.setExtrasClassLoader(classLoader);
    }
}

protected boolean doReplaceIntentForStartActivityAPILow(Object[] args) throws RemoteException {
    int intentOfArgIndex = findFirstIntentIndexInArgs(args);
    if (args != null && args.length > 1 && intentOfArgIndex >= 0) {
        Intent intent = (Intent) args[intentOfArgIndex];
        ActivityInfo activityInfo = resolveActivity(intent);
        if (activityInfo != null && isPackagePlugin(activityInfo.packageName)) {
            ComponentName component = selectProxyActivity(intent);
            if (component != null) {
                Intent newIntent = new Intent();
                newIntent.setComponent(component);
                newIntent.putExtra(Env.EXTRA_TARGET_INTENT, intent);
                newIntent.setFlags(intent.getFlags());
                if (TextUtils.equals(mHostContext.getPackageName(), activityInfo.packageName)) {
                    newIntent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
                }
                args[intentOfArgIndex] = newIntent;
            } else {
                 Log.w(TAG, "startActivity,replace selectProxyActivity fail");
            }
        }
    }
    return true;
}

@Override
protected boolean beforeInvoke(Object receiver, Method method, Object[] args) throws Throwable {
    RunningActivities.beforeStartActivity();
    boolean bRet = true;
    if (Build.VERSION.SDK_INT < Build.VERSION_CODES.JELLY_BEAN_MR2) {
        bRet = doReplaceIntentForStartActivityAPILow(args);
    } else {    
        bRet = doReplaceIntentForStartActivityAPIHigh(args);
    }
    if (!bRet) {
        setFakedResult(Activity.RESULT_CANCELED);
        return true;
    }
    return super.beforeInvoke(receiver, method, args);
    }
}
```

#### getContentProvider

很有趣的函数，CP目前了解的太少后续补充

```java
@Override
//before比较好理解,就是更改authority为stub的
protected boolean beforeInvoke(Object receiver, Method method, Object[] args) throws Throwable {
    if (args != null) {
        final int index = 1;
        if (args.length > index && args[index] instanceof String) {
            String name = (String) args[index];
            mStubProvider = null;
            mTargetProvider = null;
            ProviderInfo info = mHostContext.getPackageManager().resolveContentProvider(name, 0);
            //targetProvider
            mTargetProvider = PluginManager.getInstance().resolveContentProvider(name, 0);
            if (mTargetProvider != null && info != null && TextUtils.equals(mTargetProvider.packageName, info.packageName)) {
                mStubProvider = PluginManager.getInstance().selectStubProviderInfo(name);
                if (mStubProvider != null) {
                    args[index] = mStubProvider.authority;
                } else {
                    Log.w(TAG, "getContentProvider,fake fail 1");
                }
            } else {
                mTargetProvider = null;
                Log.w(TAG, "getContentProvider,fake fail 2=%s", name);
            }
        }
    }
    return super.beforeInvoke(receiver, method, args);
}

//没怎么看懂 todo 等看完CP源码再回来

@Override
protected void afterInvoke(Object receiver, Method method, Object[] args, Object invokeResult) throws Throwable {
    if (invokeResult != null) {
         ProviderInfo stubProvider2 = (ProviderInfo) FieldUtils.readField(invokeResult, "info");
        if (mStubProvider != null && mTargetProvider != null && TextUtils.equals(stubProvider2.authority, mStubProvider.authority)) {
            //FIXME 其实这里写的并不好，需要适配各种机型。这里就先这样吧。
            Object fromObj = invokeResult;
            //生成新的provider??
            Object toObj = ContentProviderHolderCompat.newInstance(mTargetProvider);
            //toObj.provider = fromObj.provider;
            copyField(fromObj, toObj, "provider");
            if (VERSION.SDK_INT >= VERSION_CODES.JELLY_BEAN) {
                copyConnection(fromObj, toObj);
            }
            //toObj.noReleaseNeeded = fromObj.noReleaseNeeded;
            copyField(fromObj, toObj, "noReleaseNeeded");
            Object provider = FieldUtils.readField(invokeResult, "provider");
            if (provider != null) {
                boolean localProvider = FieldUtils.readField(toObj, "provider") == null;            //新建handler
                IContentProviderHook invocationHandler = new IContentProviderHook(mHostContext, provider, mStubProvider, mTargetProvider, localProvider);
                invocationHandler.setEnable(true);
                Class<?> clazz = provider.getClass();
                List<Class<?>> interfaces = Utils.getAllInterfaces(clazz);
                Class[] ifs = interfaces != null && interfaces.size() > 0 ? interfaces.toArray(new Class[interfaces.size()]) : new Class[0];
                Object proxyprovider = MyProxy.newProxyInstance(clazz.getClassLoader(), ifs, invocationHandler);
                FieldUtils.writeField(invokeResult, "provider", proxyprovider);
                FieldUtils.writeField(toObj, "provider", proxyprovider);
            }
            setFakedResult(toObj);
        } else if (VERSION.SDK_INT >= VERSION_CODES.JELLY_BEAN_MR2) {
            Object provider = FieldUtils.readField(invokeResult, "provider");
            if (provider != null) {
                boolean localProvider = FieldUtils.readField(invokeResult, "provider") == null;
                IContentProviderHook invocationHandler = new IContentProviderHook(mHostContext, provider, mStubProvider, mTargetProvider, localProvider);
                invocationHandler.setEnable(true);
                Class<?> clazz = provider.getClass();
                List<Class<?>> interfaces = Utils.getAllInterfaces(clazz);
                Class[] ifs = interfaces != null && interfaces.size() > 0 ? interfaces.toArray(new Class[interfaces.size()]) : new Class[0];
                Object proxyprovider = MyProxy.newProxyInstance(clazz.getClassLoader(), ifs, invocationHandler);
                FieldUtils.writeField(invokeResult, "provider", proxyprovider);
            }
        }
        mStubProvider = null;
        mTargetProvider = null;
    }
}

```




### PluginCallBack

只对Activity的Create做了处理,在handleMessage中替换成了targetActivity

### PluginInstrumentation

这个类都是一些奇奇怪怪的变量修正,跳过吧心累了。唯一就是ApplicationOnCreate()中注册了静态广播:
`PluginProcessManager.registerStaticReceiver(app, app.getApplicationInfo(), app.getClassLoader());
`
### preLoadApk

前面看到很多次了,每次新建Activity或Service等都要来一次,设置classLoader与Application等

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
            //不包含
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
                    ClassLoader classloader = null;
                    try {
                        classloader = new PluginClassLoader(apk, optimizedDirectory, libraryPath, hostContext.getClassLoader().getParent());
                    } catch (Exception e) {
                }
                if(classloader==null){
                    PluginDirHelper.cleanOptimizedDirectory(optimizedDirectory);
                        classloader = new PluginClassLoader(apk, optimizedDirectory, libraryPath, hostContext.getClassLoader().getParent());
                    }
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

//Application通过LoadedApk的makeApplication方法创建
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
}
```

### MyAms

```java
//先从正在运行的进程中查找看是否有符合条件的进程，如果有则直接使用之。如果进程名相同并且包名相同;或者包名不同但签名相同
String stubProcessName1 = mRunningProcessList.getStubProcessByTarget(targetInfo);
if (stubProcessName1 != null) {
    List<ActivityInfo> stubInfos = mStaticProcessList.getActivityInfoForProcessName(stubProcessName1, useDialogStyle);
    for (ActivityInfo stubInfo : stubInfos) {
        //对于stubInfo,如果启动模式相同
        if (stubInfo.launchMode == targetInfo.launchMode) {
            //如果支持多activity
            if (stubInfo.launchMode == ActivityInfo.LAUNCH_MULTIPLE) {
                //那么直接加入
                mRunningProcessList.setTargetProcessName(stubInfo, targetInfo);
                return stubInfo;
            //不然看一下是否已经在使用了
            } else if (!mRunningProcessList.isStubInfoUsed(stubInfo, targetInfo, stubProcessName1)) {
                mRunningProcessList.setTargetProcessName(stubInfo, targetInfo);
                return stubInfo;
            }
        }
    }
}
//运行的每一个行啊
List<String> stubProcessNames = mStaticProcessList.getProcessNames();
for (String stubProcessName : stubProcessNames) {
    List<ActivityInfo> stubInfos = mStaticProcessList.getActivityInfoForProcessName(stubProcessName, useDialogStyle);
    if (mRunningProcessList.isProcessRunning(stubProcessName)) {//该预定义的进程正在运行。
        if (mRunningProcessList.isPkgEmpty(stubProcessName)) {//空进程，没有运行任何插件包。
            for (ActivityInfo stubInfo : stubInfos) {
                if (stubInfo.launchMode == targetInfo.launchMode) {
                    if (stubInfo.launchMode == ActivityInfo.LAUNCH_MULTIPLE) {
                        mRunningProcessList.setTargetProcessName(stubInfo, targetInfo);
                        return stubInfo;
                    } else if (!mRunningProcessList.isStubInfoUsed(stubInfo, targetInfo, stubProcessName1)) {
                        mRunningProcessList.setTargetProcessName(stubInfo, targetInfo);
                        return stubInfo;
                    }
                }
            }
            throw throwException("没有找到合适的StubInfo");
            //不在运行，看是否能运行
        } else if 
        (mRunningProcessList.isPkgCanRunInProcess(targetInfo.packageName, stubProcessName, targetInfo.processName)) {
            for (ActivityInfo stubInfo : stubInfos) {
                if (stubInfo.launchMode == targetInfo.launchMode) {
                    if (stubInfo.launchMode == ActivityInfo.LAUNCH_MULTIPLE) {
                        mRunningProcessList.setTargetProcessName(stubInfo, targetInfo);
                        return stubInfo;
                    } else if (!mRunningProcessList.isStubInfoUsed(stubInfo, targetInfo, stubProcessName1)) {
                        mRunningProcessList.setTargetProcessName(stubInfo, targetInfo);
                        return stubInfo;
                    }
                }
            }
            throw throwException("没有找到合适的StubInfo");
        } else {
            //这里需要考虑签名一样的情况，多个插件公用一个进程。
        }
    } else { //该预定义的进程没有。
        for (ActivityInfo stubInfo : stubInfos) {
            if (stubInfo.launchMode == targetInfo.launchMode) {
                if (stubInfo.launchMode == ActivityInfo.LAUNCH_MULTIPLE) {
                    mRunningProcessList.setTargetProcessName(stubInfo, targetInfo);
                    return stubInfo;
                } else if (!mRunningProcessList.isStubInfoUsed(stubInfo, targetInfo, stubProcessName1)) {
                    mRunningProcessList.setTargetProcessName(stubInfo, targetInfo);
                    return stubInfo;
                }
            }
        }
        throw throwException("没有找到合适的StubInfo");
    }
}
```

Service/Provider与Activity最大的区别在于 `isStubInfoUsed`  

```java 
boolean isStubInfoUsed(ProviderInfo stubInfo) {
//TODO
    return false;
}

boolean isStubInfoUsed(ServiceInfo stubInfo) {
    //TODO
    return false;
}

```
所以就算已经有Service或Provider安家了,其他还是可以在同一个进程里公用的


### IPackageManagerHookHandle

#### getPackageInfo

```java
packageInfo = PluginManager.getInstance().getPackageInfo(packageName, flags);
```

因为没有安装，使用Parser去解析 包括所有拿到ActivityInfo等信息,都通过parser去中转。

### 总结

看完全部其实对插件化已经很清楚了。

>* Activity: 替换成stubActivity然后在handler回调中替换回来
* Service:使用一个大的Service替代,在start的时候去主动生成targetService并启动
* 广播:注册静态的即可
* Provider:这部分todo,但大体概念就是替换authority





