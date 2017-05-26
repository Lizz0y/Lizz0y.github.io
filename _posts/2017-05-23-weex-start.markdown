---
layout:     post
title:      "Weex启动过程"
date:       2017-05-23
author:     "Liz"
header-img: "img/cat.jpg"
tags:
    - Weex
    - Js
    - 源码分析
---

本文先分析`android sdk`中的启动过程,参考了以下文章:
[来自郑海波的Weex Android SDK源码分析](http://www.jianshu.com/p/3160a0297345)

[Weex SDK Android 源码解析](https://zhuanlan.zhihu.com/p/25326775)

### WXApplication

一切从`WXApplication`出发:

```java
public class WXApplication extends Application {
    @Override
    public void onCreate() {
        super.onCreate();
        InitConfig config = new InitConfig.Builder().setImgAdapter(new  ImageAdapter()).build();//配置初始参数
        WXSDKEngine.initialize(this, config);//关键初始化代码
        try {
            WXSDKEngine.registerModule("poneInfo", PhoneInfoModule.class);//注册自定义module和component
            WXSDKEngine.registerComponent("rich", RichText.class, false);
        } catch (WXException e) {
            e.printStackTrace();
        }
    }
}
```

#### init

```java
private static void doInitInternal(final Application application,final InitConfig config){
    WXEnvironment.sApplication = application;
    WXEnvironment.JsFrameworkInit = false;
    WXBridgeManager.getInstance().post(new Runnable() {
      @Override
      public void run() {
        WXSDKManager sm = WXSDKManager.getInstance();
        if(config != null ) {
          //为sdkManager设置初始化配置
          sm.setInitConfig(config);
          if(config.getDebugAdapter()!=null){
            config.getDebugAdapter().initDebug(application);
          }
        }
        WXSoInstallMgrSdk.init(application);
        boolean isSoInitSuccess = WXSoInstallMgrSdk.initSo(V8_SO_NAME, 1, config!=null?config.getUtAdapter():null);
        if (!isSoInitSuccess) {
          return;
        }
        sm.initScriptsFramework(config!=null?config.getFramework():null);
      }
    });
    register();
  }
  ```
#### post

首先研究一下`WXBridgeManager`中的`post`:

```java
@Override
public void post(Runnable r){
    if(mInterceptor != null && mInterceptor.take(r)){
      //task is token by the interceptor
      return;
    }
    if (mJSHandler == null){
      return;
    }
    mJSHandler.post(WXThread.secure(r));
}
```

`WXThread.secure(r)`会将`runnable`中的`exception`进行捕获，`JSHandler`是`HandlerThread`中创建的`handler`

#### register
```js
private static void register() {
    BatchOperationHelper batchHelper = new BatchOperationHelper(WXBridgeManager.getInstance());
    try {
        registerComponent(new SimpleComponentHolder(WXDiv.class,new WXDiv.Ceator()),false,
        WXBasicComponentType.CONTAINER,
        WXBasicComponentType.DIV,
        WXBasicComponentType.HEADER,
        WXBasicComponentType.FOOTER
    );
        ...
        registerModule("modal", WXModalUIModule.class, false);
        ...
        registerDomObject(WXBasicComponentType.TEXT, WXTextDomObject.class);
    } catch (WXException e) {
        WXLogUtils.e("[WXSDKEngine] register:", e);
    }
    batchHelper.flush();
}
```

初始化注册时,会注册`Component`,`Module`与`DomObject`。下面从这三类来介绍:

#### registerComponent

```java  
public SimpleComponentHolder(Class<? extends WXComponent> clz,ComponentCreator customCreator) {
    this.mClz = clz;
    this.mCreator = customCreator;
}
public static boolean registerComponent(IFComponentHolder holder, boolean appendTree, String ... names) throws WXException {
    boolean result =  true;
    for(String name:names) {
        Map<String, Object> componentInfo = new HashMap<>();
        if (appendTree) {
            componentInfo.put("append", "tree");
        }
        //name,(WXBasicComponentType):holder,isAppendExist
        result  = result && WXComponentRegistry.registerComponent(name, holder, componentInfo);
    }
    return result;
}
//以WXDIV为例,name为container,div,footer,header,使用WXComponentRegistry分别对他们注册
public static boolean registerComponent(final String type, final IFComponentHolder holder, final Map<String, Object> componentInfo) throws WXException {
    if (holder == null || TextUtils.isEmpty(type)) {
      return false;
    }
    //execute task in js thread to make sure register order is same as the order invoke register method.
    WXBridgeManager.getInstance().post(new Runnable() {
        @Override
        public void run() {
            try {
                Map<String, Object> registerInfo = componentInfo;
                if (registerInfo == null){
                    registerInfo = new HashMap<>();
                }
                registerInfo.put("type",type);
                registerInfo.put("methods",holder.getMethods());
                registerNativeComponent(type, holder);
                registerJSComponent(registerInfo);
                sComponentInfos.add(registerInfo);
            } catch (WXException e) {
                WXLogUtils.e("register component error:", e);
            }
        }
    });
    return true;
}
```

在`holder.getMethods`中,主要看内部的`clz`的`annotation`:

```java
if (anno instanceof WXComponentProp) {
    String name = ((WXComponentProp) anno).name();
    methods.put(name, new MethodInvoker(method,true));
    break;
}else if(anno instanceof JSMethod){
    JSMethod methodAnno = (JSMethod)anno;
    String name = methodAnno.alias();
    if(JSMethod.NOT_SET.equals(name)){
        name = method.getName();
    }
    mInvokers.put(name, new MethodInvoker(method,methodAnno.uiThread()));
    break;
}
```

最后由`registerNativeComponent`与`registerJSComponent`进行注册

##### registerNativeComponent

>native 部分注册：记录component 的 type 与 holder 的 map 映射。holder 默认为 SimpleComponentHolder，主要用来：如果不为懒加载（lazyLoad），会提前解析好注解 @WXComponentProp 和 @JSMethod，否则会等到使用的时候才会去解析注解，可以提高 weex 的初始化效率。

```java
private static boolean registerNativeComponent(String type, IFComponentHolder holder) throws WXException {
    try {
        holder.loadIfNonLazy();//clz中是否有lazyLoad的注解
        sTypeComponentMap.put(type, holder);
    }catch (ArrayStoreException e){
        e.printStackTrace();
    }
    return true;
}
 ``` 
 
##### registerJSComponent
```java
private static boolean registerJSComponent(Map<String, Object> componentInfo) throws WXException {
    ArrayList<Map<String, Object>> coms = new ArrayList<>();
    coms.add(componentInfo);
    WXSDKManager.getInstance().registerComponents(coms);
    return true;
  }
```
    
```java
public void registerComponents(final List<Map<String, Object>> components) {
    if ( mJSHandler == null || components == null
        || components.size() == 0) {
      return;
    }
    post(new Runnable() {
        @Override
        public void run() {
            invokeRegisterComponents(components);
        }
    }, null);
}
public void post(Runnable r, Object token) {
    if (mJSHandler == null) {
      return;
    }
    Message m = Message.obtain(mJSHandler, WXThread.secure(r));
    m.obj = token;
    m.sendToTarget();
}
```

最终,`mWXBridge.execJS("", null, METHOD_REGISTER_COMPONENTS, args);`调用js代码注册`component`。

#### registerModule


`registerModule("modal", WXModalUIModule.class, false);`


```java
public static <T extends WXModule> boolean registerModule(String moduleName, Class<T> moduleClass,boolean global) throws WXException {
    return moduleClass != null && registerModule(moduleName, new TypeModuleFactory<>(moduleClass), global);
}

public static boolean registerModule(final String moduleName, final ModuleFactory factory, final boolean global) throws WXException {
    if (moduleName == null || factory == null) {
      return false;
    }
    if (TextUtils.equals(moduleName,WXDomModule.WXDOM)) {
      WXLogUtils.e("Cannot registered module with name 'dom'.");
      return false;
    }
    WXBridgeManager.getInstance()
        .post(new Runnable() {
        @Override
        public void run() {
        if (sModuleFactoryMap.containsKey(moduleName)) {
            WXLogUtils.w("WXComponentRegistry Duplicate the Module name: " + moduleName);
        }
        if (global) {
            try {
                WXModule wxModule = factory.buildInstance();
                wxModule.setModuleName(moduleName);
                sGlobalModuleMap.put(moduleName, wxModule);
            } catch (Exception e) {
                WXLogUtils.e(moduleName + " class must have a default constructor without params. ", e);
            }
        }
        try {
            registerNativeModule(moduleName, factory);
        } catch (WXException e) {
            WXLogUtils.e("", e);
        }
        registerJSModule(moduleName, factory);
    }
    });
    return true;
}

```

和`component`一样,主要分为`registerNativeModule`与`registerJSModule`

##### registerNativeModule

一样,将`module`放入`sModuleFactoryMap`
```java
static boolean registerNativeModule(String moduleName, ModuleFactory factory) throws WXException {
    if (factory == null) {
        return false;
    }
    try {
        sModuleFactoryMap.put(moduleName, factory);
    }catch (ArrayStoreException e){
    }
    return true;
}
```

##### registerJSModule

```java
static boolean registerJSModule(String moduleName, ModuleFactory factory) {
    Map<String, Object> modules = new HashMap<>();
    modules.put(moduleName, factory.getMethods());
    WXSDKManager.getInstance().registerModules(modules);
    return true;
}
```
  
同样,在factory中`getMethods`:遍历查看是否有`JSMethod`和`WXModuleAnno`的注解:

![](/img/2017-05-23-weex-begin/14955415003014.jpg)

```java
public void registerModules(final Map<String, Object> modules) {
    if (modules != null && modules.size() != 0) {
        if(isJSThread()){
            invokeRegisterModules(modules);
        }
        else{
            post(new Runnable() {
            @Override
                public void run() {
                    invokeRegisterModules(modules);
                }
            }, null);
        }
    }
}

//一样,调用js方法进行register_modules

private void invokeRegisterModules(Map<String, Object> modules) {
    if (modules == null || !isJSFrameworkInit()) {
        if (!isJSFrameworkInit()) {
            WXLogUtils.e("[WXBridgeManager] invokeCallJSBatch: framework.js uninitialized.");
        }
        mRegisterModuleFailList.add(modules);
        return;
    }
    WXJSObject[] args = {new WXJSObject(WXJSObject.JSON, WXJsonUtils.fromObjectToJSONString(modules))};
    try {
        mWXBridge.execJS("", null, METHOD_REGISTER_MODULES, args);
    } catch (Throwable e) {
    }
}
```

#### registerDom

`registerDomObject(WXBasicComponentType.TEXTAREA,TextAreaEditTextDomObject.class)`


```java
public static boolean registerDomObject(String type, Class<? extends WXDomObject> clazz) throws WXException {
    return WXDomRegistry.registerDomObject(type, clazz);
}
public static boolean registerDomObject(String type, Class<? extends WXDomObject> clazz) throws WXException {
    if (clazz == null || TextUtils.isEmpty(type)) {
        return false;
    }
    if (sDom.containsKey(type)) {
        if (WXEnvironment.isApkDebugable()) {
            throw new WXException("WXDomRegistry had duplicate Dom:" + type);
        } else {
            WXLogUtils.e("WXDomRegistry had duplicate Dom: " + type);
            return false;
        }
    }
    sDom.put(type, clazz);
    return true;
}
```

`registerDom`比较简单,将dom的类型与类放到一个`sDom map`中即可。


### flush

```java
public void flush(){
    isCollecting = false;
    mExecutor.post(new Runnable() {
        @Override
        public void run() {
            Iterator<Runnable> iterator = sRegisterTasks.iterator();
            while(iterator.hasNext()){
                Runnable item = iterator.next();
                item.run();
                iterator.remove();
            }
        }
    });
    mExecutor.setInterceptor(null);
}
@Override
public void post(Runnable r){
    if(mInterceptor != null && mInterceptor.take(r)){
      //task is token by the interceptor
      return;
    }
    if (mJSHandler == null){
      return;
    }
    mJSHandler.post(WXThread.secure(r));
}
```

而`mInterceptor`为`BatchOperationHelper`,

```java
@Override
public boolean take(Runnable runnable) {
    if(isCollecting){
        sRegisterTasks.add(runnable);
        return true;
    }
    return false;
}
```

可以看到,前面在`register`时`isCollecting`是`true`,因此会将任务加入`sRegisterTasks`中,而当注册结束后`flush`时,`isCollecting`变成`false`,因此会从`mRegisterTask`中拿取任务

### initSo

```java
WXSoInstallMgrSdk.init(application);
boolean isSoInitSuccess = WXSoInstallMgrSdk.initSo(V8_SO_NAME, 1, config!=null?config.getUtAdapter():null);
if (!isSoInitSuccess) {
    return;
}
sm.initScriptsFramework(config!=null?config.getFramework():null);
```

调用`js`函数`initFrameWork`

有点乱,画个图总结一下:


![](/img/2017-05-23-weex-begin/14955511136700.jpg)



