---
layout:     post
title:      "Dom渲染过程"
date:       2017-05-27
author:     "Liz"
header-img: "img/party.jpg"
tags:
    - Weex
    - Js
    - 源码分析
---

本节,我们介绍`Dom`的渲染过程,在主界面中进行`render`,先引入`.we`文件:

```js
<template>
  <div>
    <text style="font-size:100px;">Hello World.</text>
  </div>
</template>
```

可以看到只有一个`div`包裹了`text`。

先介绍一下渲染的整体过程,页面最底部的框架使用`createBody`渲染,后续所有元素通过`addDom`添加绘制。

### 入口onCreate

```java
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);
    mWXSDKInstance = new WXSDKInstance(this);
    mWXSDKInstance.registerRenderListener(this);
    /**
     * WXSample 可以替换成自定义的字符串，针对埋点有效。
     * template 是.we transform 后的 js文件。
     * option 可以为空，或者通过option传入 js需要的参数。例如bundle js的地址等。
     * jsonInitData 可以为空。
     */
    Map<String,Object> options=new HashMap<>();
    options.put(WXSDKInstance.BUNDLE_URL,"file://hello.js");
    mWXSDKInstance.render("WXSample", WXFileUtils.loadAsset("hello.js", this), null, null, WXRenderStrategy.APPEND_ASYNC);
}
```

我们看到,每次`render`时都会生成一个新的`WXSDKInstance`,其`id`会递增
在`render`中,主要`createInstance`:
`WXSDKManager.getInstance().createInstance(this, template, renderOptions, jsonInitData);`

#### createInstance

```java
void createInstance(WXSDKInstance instance, String code, Map<String, Object> options, String jsonInitData) {
    mWXRenderManager.registerInstance(instance);
    mBridgeManager.createInstance(instance.getInstanceId(), code, options, jsonInitData);
}
```



其中,在`WXRenderManager`中,为`instance`生成`WXRenderStatement`,后续使用它来为`instance`进行渲染,这是关键,一个`instance`对应一个`statement`

```java
public void registerInstance(WXSDKInstance instance) {
    mRegistries.put(instance.getInstanceId(), new WXRenderStatement(instance));
}  
```
在`WXModuleManager`中,为`instance`生成`WXDomModule`
```java
public static void createDomModule(WXSDKInstance instance){
    if(instance != null) {
        sDomModuleMap.put(instance.getInstanceId(), new WXDomModule(instance));
    }
}
```

##### invokeCreateInstance

```java
post(new Runnable() {
    @Override
    public void run() {
        long start = System.currentTimeMillis();
        invokeCreateInstance(instance, template, options, data);
        final long totalTime = System.currentTimeMillis() - start;
        WXSDKManager.getInstance().postOnUiThread(new Runnable() {
            @Override
            public void run() {
                instance.createInstanceFinished(totalTime);
            }
        }, 0);
    }
}, instanceId);
```

```java
WXJSObject instanceIdObj = new WXJSObject(WXJSObject.String,instance.getInstanceId());
WXJSObject instanceObj = new WXJSObject(WXJSObject.String,template);//template是.js代码
WXJSObject optionsObj = new WXJSObject(WXJSObject.JSON,options == null ? "{}": WXJsonUtils.fromObjectToJSONString(options));
WXJSObject dataObj = new WXJSObject(WXJSObject.JSON,data == null ? "{}" : data);
WXJSObject[] args = {instanceIdObj, instanceObj, optionsObj,dataObj};
invokeExecJS(instance.getInstanceId(), null, METHOD_CREATE_INSTANCE, args,false);
```

组建`args`后调用`js`的`create_instance`方法,通知`js`该`instanceid`对应的`instance`需要创建。

到这里,我们看到`instaceId`对应:

* WXSDKInstance
* WXRenderStatement
* WXDomModule

### CallNative


调用`js`代码后`js`会回调`callNative`,具体怎么调用的之后再分析。我们现在直接跳到`callNative`方法分析。看到`METHOD_CREATE_INSTANCE`后的结果如上图，然后处理得到的`json`串:
![](/img/2017-05-27-weex-render/14958817363775.jpg)
`js`希望回调进行`createBody`的操作。

```java
public int callNative(String instanceId, String tasks, String callback) {
    int size = array.size();
    if (size > 0) {
        try {
            JSONObject task;
            for (int i = 0; i < size; ++i) {
                task = (JSONObject) array.get(i);
                if (task != null && WXSDKManager.getInstance().getSDKInstance(instanceId) != null) {
                    Object target = task.get(MODULE);
                    if(target != null){
                        if(WXDomModule.WXDOM.equals(target)){
                            WXDomModule dom = getDomModule(instanceId);
                            dom.callDomMethod(task);
                        }else {
                            WXModuleManager.callModuleMethod(instanceId, (String) target,(String) task.get(METHOD), (JSONArray) task.get(ARGS));
                        }
                    }else if(task.get(COMPONENT) != null){
                     //call component
                        WXDomModule dom = getDomModule(instanceId);
                        dom.invokeMethod((String) task.get(REF),(String) task.get(METHOD),(JSONArray) task.get(ARGS));
                    }else{
                        throw new IllegalArgumentException("unknown callNative");
                    }
                }
            }   
        }
    }
    getNextTick(instanceId, callback);//关键16ms
    return IWXBridge.INSTANCE_RENDERING;
```

我们以`createInstance`的`callNative`参数为例:
![](/img/2017-05-27-weex-render/14955424281733.jpg)
在`callDomMethod`中,会解析`method`与`args`,根据`method`调用相应的方法,例如这里为`createBody`,构造`Message`,发送给`WXDomhandler`。然后送入`WXDomManager`中处理

![](/img/2017-05-27-weex-render/14956387019442.jpg)
```java
void createBody(String instanceId, JSONObject element) {
    throwIfNotDomThread();
    WXDomStatement statement = new WXDomStatement(instanceId, mWXRenderManager);
    mDomRegistries.put(instanceId, statement);//important,存储了instanceId与statement的关系,在statement中有 mRegistry,它存储dom
    statement.createBody(element);
 }
```
  

#### addDomInternal

比较复杂的函数，单独分析一下:这个函数在`createBody`与`addDom`时都会用到,以`root`作为判断标准。

```java
private void addDomInternal(JSONObject dom,boolean isRoot, String parentRef, final int index){
    if (mDestroy) {
        return;
    }
    WXSDKInstance instance = WXSDKManager.getInstance().getSDKInstance(mInstanceId);
    if (instance == null) {
        return;
    }
    //判断是否为root
    WXErrorCode errCode = isRoot ? WXErrorCode.WX_ERR_DOM_CREATEBODY : WXErrorCode.WX_ERR_DOM_ADDELEMENT;
    if (dom == null) {
        instance.commitUTStab(IWXUserTrackAdapter.DOM_MODULE, errCode);
    }
    //only non-root has parent.
    WXDomObject parent;
    WXDomObject domObject = WXDomObject.parse(dom,instance);
```
##### WXDomObject.parse

`parse`主要处理`json`字符串。

```java
String type = (String) json.get(TYPE);//得到type,此处为`div`
WXDomObject domObject = WXDomObjectFactory.newInstance(type);
domObject.parseFromJson(json);
Object children = json.get(CHILDREN);
if (children != null && children instanceof JSONArray) {
    JSONArray childrenArray = (JSONArray) children;
    int count = childrenArray.size();
    for (int i = 0; i < count; ++i) {
        domObject.add(parse(childrenArray.getJSONObject(i),wxsdkInstance),-1);
    }
}
```

在`WXDomObjectFactory.newInstance`中:利用反射创建`WXDomObject`，对应的对象是根据之前注册的`registerDomObject`来创建的.

```java
if (WXDomObject.class.isAssignableFrom(clazz)) {
    WXDomObject domObject = clazz.getConstructor().newInstance();
    return domObject;
}
```

然后`parseJson`,主要处理:

* type
* ref
* style
* attr
* event


最后递归处理子节点。

#### 继续addDom

```java
if (isRoot) {
    WXDomObject.prepareRoot(domObject,
WXViewUtils.getWebPxByWidth(WXViewUtils.getWeexHeight(mInstanceId),WXSDKManager.getInstanceViewPortWidth(mInstanceId)),
WXViewUtils.getWebPxByWidth(WXViewUtils.getWeexWidth(mInstanceId),WXSDKManager.getInstanceViewPortWidth(mInstanceId)));
} else if ((parent = mRegistry.get(parentRef)) == null) {
      instance.commitUTStab(IWXUserTrackAdapter.DOM_MODULE, errCode);
      return;
} else {
      //non-root and parent exist
      parent.add(domObject, index);
}
```

如果是`root`,那就要进入`prepareRoot`:主要增加一些属性，比如背景色，宽高度等。

```java
public static void prepareRoot(WXDomObject domObj,float defaultHeight,float defaultWidth) {
    domObj.mRef = WXDomObject.ROOT;
    WXStyle domStyles = domObj.getStyles();
    Map<String, Object> style = new HashMap<>(5);
    if (!domStyles.containsKey(Constants.Name.FLEX_DIRECTION)) {
        style.put(Constants.Name.FLEX_DIRECTION, "column");
    }
    if (!domStyles.containsKey(Constants.Name.BACKGROUND_COLOR)) {
        style.put(Constants.Name.BACKGROUND_COLOR, "#ffffff");
    }
    style.put(Constants.Name.DEFAULT_WIDTH, defaultWidth);
    style.put(Constants.Name.DEFAULT_HEIGHT, defaultHeight);
    domObj.updateStyle(style);
}
```
在`updateStyle`中更新style,同时将该节点的`LayoutState`设为`dirty`

如果不是`root`且有`parent`,进入`parent.add`加入到`parent`的`children`数组中

```java
    domObject.traverseTree(mAddDOMConsumer,ApplyStyleConsumer.getInstance());

    //Create component in dom thread
    WXComponent component = isRoot ?mWXRenderManager.createBodyOnDomThread(mInstanceId, domObject) :mWXRenderManager.createComponentOnDomThread(mInstanceId, domObject, parentRef, index);
}
```

##### traverseTree

`traverseTree`会被调用很多次,会遍历以该节点为根的子树:

```java
  /** package **/ void traverseTree(Consumer...consumers){
if (consumers == null) {
    return;
}
for (Consumer consumer:consumers){
    consumer.accept(this);
}
int count = childCount();
    WXDomObject child;
    for (int i = 0; i < count; ++i) {
        child = getChild(i);
        child.traverseTree(consumers);
    }
}
```

有两个`consumer`:

```java
private static class AddDOMConsumer implements WXDomObject.Consumer {
    final ConcurrentHashMap<String, WXDomObject> mRegistry;
    AddDOMConsumer(ConcurrentHashMap<String, WXDomObject> r){
        mRegistry = r;
    }
    @Override
    public void accept(WXDomObject dom) {
        //register dom
        dom.young();
        mRegistry.put(dom.getRef(), dom);
        //find fixed node
        WXDomObject rootDom = mRegistry.get(WXDomObject.ROOT);
        if (rootDom != null && dom.isFixed()) {
            rootDom.add2FixedDomList(dom.getRef());
        }
    }
}
```
将该`domObject`设为`young`,同时看它的`root`节点是否固定。

```java
static class ApplyStyleConsumer implements WXDomObject.Consumer {
    static ApplyStyleConsumer sInstance;
    public static ApplyStyleConsumer getInstance(){
        if(sInstance == null){
            sInstance = new ApplyStyleConsumer();
        }
        return sInstance;
    }
    private ApplyStyleConsumer(){};
    @Override
    public void accept(WXDomObject dom) {
        WXStyle style = dom.getStyles();
        /** merge default styles **/
        Map<String, String> defaults = dom.getDefaultStyle();
        if (defaults != null) {
            for (Map.Entry<String, String> entry : defaults.entrySet()) {
                if (!style.containsKey(entry.getKey())) {
                style.put(entry.getKey(), entry.getValue());
            }
        }
    }
    if (dom.getStyles().size() > 0) {
        dom.applyStyleToNode();
    }
}
```

然后开始`applyStyle`:
![](/img/2017-05-27-weex-render/14956401025727.jpg)
反正就是为`cssStyle`的一系列属性赋值



##### createBodyOnDomThread

然后正式`createBody`:进入`renderManager`
```java
public WXComponent createBodyOnDomThread(String instanceId, WXDomObject domObject) {
    WXRenderStatement statement = mRegistries.get(instanceId);
    if (statement == null) {
        return null;
    }
    return statement.createBodyOnDomThread(domObject);
}
```

主要创建`ComponentTree`

```java
private WXComponent generateComponentTree(WXDomObject dom, WXVContainer parent) {
    if (dom == null ) {
        return null;
    }
    WXComponent component = WXComponentFactory.newInstance(mWXSDKInstance, dom,parent);
    mRegistry.put(dom.getRef(), component);
    if (component instanceof WXVContainer) {
        WXVContainer parentC = (WXVContainer) component;
        int count = dom.childCount();
        WXDomObject child = null;
        for (int i = 0; i < count; ++i) {
            child = dom.getChild(i);
            if (child != null) {
                parentC.addChild(generateComponentTree(child, parentC));
            }
        }
    }
    return component;
}
```

在`WXComponentFactory`中根据`dom`创建`WXComponent`,

```java
IFComponentHolder holder = WXComponentRegistry.getComponent(node.getType());
if (holder == null) {
      holder = WXComponentRegistry.getComponent(WXBasicComponentType.CONTAINER);//之前注册Component中的sTypeComponentMap
      if(holder == null){
        throw new WXRuntimeException("Container component not found.");
      }
}
return holder.createInstance(instance, node, parent);
    
```


这里的`Holder`是之前注册时生成的,`SimpleComponentHolder`中:

```java
@Override
public synchronized WXComponent createInstance(WXSDKInstance instance, WXDomObject node, WXVContainer parent) throws IllegalAccessException, InvocationTargetException, InstantiationException {
    WXComponent component = mCreator.createInstance(instance,node,parent);
    component.bindHolder(this);
    return component;
}
```

在生成`ComponentTree`时会对子节点递归生成。

#### 继续addInternal

```java
AddDomInfo addDomInfo = new AddDomInfo();
addDomInfo.component = component;
mAddDom.put(domObject.getRef(), addDomInfo);
IWXRenderTask task = isRoot ? new CreateBodyTask(component) : new AddDOMTask(component, parentRef, index);
mNormalTasks.add(task);
addAnimationForDomTree(domObject);
mDirty = true;
```

将这个将要被渲染的`Component`封装到`CreateBodyTask`中,即`renderTask`。到这里还没在做准备工作,并没有开始真正渲染真是心累。

### addDom

对于一个页面,真正要显示元素需要通过`AddDom`来添加节点。例如这里的`text`节点。可以看到,它和`root`的`instanceId`为同一个,`parent`是`root`

![](/img/2017-05-27-weex-render/14956410291875.jpg)

#### createBodyTask

```java
void createBody(WXComponent component) {
    component.createView();
    component.applyLayoutAndEvent(component);
    component.bindData(component);
    if (component instanceof WXScroller) {
        WXScroller scroller = (WXScroller) component;
        if (scroller.getInnerView() instanceof ScrollView) {
            mWXSDKInstance.setRootScrollView((ScrollView) scroller.getInnerView());
        }
    }
    mWXSDKInstance.onRootCreated(component);//mRenderContainer.addView(root.getHostView());
    if (mWXSDKInstance.getRenderStrategy() != WXRenderStrategy.APPEND_ONCE) {
        mWXSDKInstance.onCreateFinish();//mRenderListener.onViewCreated(WXSDKInstance.this, wxView);
    }
}
```

##### createView

创建`host`,例如`WXTextView`等

##### applyLayoutAndEvent

* setLayout(component.getDomObject());
* setPadding(component.getDomObject().getPadding(), component.getDomObject().getBorder());
* addEvents();

##### bindData

* updateProperties(component.getDomObject().getStyles());
* updateProperties(component.getDomObject().getAttrs());
* updateExtra(component.getDomObject().getExtra());

### 刷新

在`weex`中,使用16ms的刷新机制去进行UI渲染

```java
if (!mHasBatch) {
    mHasBatch = true;    mWXDomManager.sendEmptyMessageDelayed(WXDomHandler.MsgType.WX_DOM_BATCH, DELAY_TIME);
}
```

`DELAY_TIME`为16ms

```java
void batch() {
    throwIfNotDomThread();
    Iterator<Entry<String, WXDomStatement>> iterator = mDomRegistries.entrySet().iterator();
    while (iterator.hasNext()) {
        iterator.next().getValue().batch();
    }
}
```


### layout

在layout中真正渲染,使用`WXDomStatement`,与`instanceId`绑定。又是一个难啃的函数,继续分段研究:

```java
 void layout(WXDomObject rootDom) {
    if (rootDom == null) {
        return;
    }
    rebuildingFixedDomTree(rootDom);
    rootDom.traverseTree( new WXDomObject.Consumer() {
        @Override
        public void accept(WXDomObject dom) {
            if (!dom.hasUpdate() || mDestroy) { //在hasUpdate中看是否dirty或hasNewLayout,因为前面设置为dirty,所以结果为true
                return;
            }
            dom.layoutBefore();//每个domObject有自己的特性，比如text会设置一下内容
        }
    });
```


```java
rootDom.calculateLayout(mLayoutContext);
```
使用`CSS Layout`进行`layoutNode`,对一系列属性例如`width,height`进行赋值
```java
public void calculateLayout(CSSLayoutContext layoutContext) {
    csslayout.resetResult();
    LayoutEngine.layoutNode(layoutContext, this, CSSConstants.UNDEFINED, null);
}
```
 
结束后,进行`layoutAfter`收尾工作
```java
rootDom.traverseTree( new WXDomObject.Consumer() {
    @Override
    public void accept(WXDomObject dom) {
        if (!dom.hasUpdate() || mDestroy) {
            return;
        }
        dom.layoutAfter();
      }
});
```

`rootDom.traverseTree(new ApplyUpdateConsumer());`

然后真正更新属性,

```java
if (dom.hasUpdate()) {
    dom.markUpdateSeen();
    if (!dom.isYoung()) {
        final WXDomObject copy = dom.clone();
        if (copy == null) {
            return;
        }
        mNormalTasks.add(new IWXRenderTask() {
            @Override
            public void execute() {
                mWXRenderManager.setLayout(mInstanceId, copy.getRef(), copy);
                if(copy.getExtra() != null) {
                    mWXRenderManager.setExtra(mInstanceId, copy.getRef(), copy.getExtra());
                }
            }
        }
    }
}
```

`weex`考虑的还是比较多的,首先看该节点是否需要更新,因为前面在更新属性的时候已经标记节点是脏的了，所以需要更新,然后`markUpdateSeen`,说明已经确认这个更新了。然后只有在`dom`不是`young`的时候才会去真正`setLayout`，那接下来就会发现什么时候不再年轻了:
    
```java
    updateDomObj();
    int count = mNormalTasks.size();
    for (int i = 0; i < count && !mDestroy; ++i) {
        mWXRenderManager.runOnThread(mInstanceId, mNormalTasks.get(i));
    }
```

对于普通的节点,`updateDomObj`即更新为`old`...

至于为什么需要`ApplyUpdate`,一开始我也百思不得其解:

>为什么需要这个步骤？正常情况下在renderTask中对Component View setLayout就可以了，但是这要基于一个前提，那就是所有的 Dom 节点都已经被注册到 mRegistry 中了，只有这样最后 CSSLayout 计算出来的才是正确的。 由于 batch 驱动的不确定性（有可能分好几帧），非常有可能在本次 Layout 过程中 Dom 数量是不完整的，导致 CSSLayout 计算的结果肯定是不完整的。因此需要每次batch 重新计算完整 dom 树的时候把之前节点 Layout 再更新一遍，并且此时的节点必为 old（上一帧被标为old）

其实我们看到,`ApplyUpdate`做的和后面的`createBody`等一样,都是设置`layout`和属性等等,但是因为每次`batch`时整个`dom`树不一定完整,所以需要在第二次`batch`时更新,这也是为什么只有在`old`的时候才会走`setLayout`

#### ApplyUpdate

还记得在前面`ApplyUpdateConsumer`中我们加入队列的`execute`么:

```java
    @Override
    public void execute() {
        mWXRenderManager.setLayout(mInstanceId, copy.getRef(), copy);
        if(copy.getExtra() != null) {
            mWXRenderManager.setExtra(mInstanceId, copy.getRef(), copy.getExtra());
        }
    }
```

`component.setLayout(domObject);`

可以看到,`component`其实与`domObject`是绑定的:

```java
 //fixed style
if (mDomObj.isFixed()){    
    setFixedHostLayoutParams(mHost,realWidth,realHeight,realLeft,realRight,realTop,realBottom);
}else {
    setHostLayoutParams(mHost, realWidth, realHeight, realLeft, realRight, realTop, realBottom);
}
```

在这段代码中,真正设置了元素的宽高,`padding,margin`间距。




### 总结

在Weex中,主要有两种操作:

* `createBody`: 创建一个页面的根节点
* `addDom`: 增加根节点下的所有节点

对`dom`节点的属性进行解析后得到`dom`树的结构,`updateStyle`更新到最新的`style`,标记节点为`dirty`。利用`AddDOMConsumer`标记节点为`young`,使用`ApplyStyleConsumer`对属性进行赋值。最后通过`Task`真正渲染`UI`,创建`native view`并加入层次中后,进行`layout`,更新属性增加点击事件等。









