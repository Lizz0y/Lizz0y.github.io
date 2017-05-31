---
layout:     post
title:      "dataBinding--应用层运行"
date:       2017-05-31
author:     "Liz"
header-img: "img/man.jpg"
tags:
    - MVVM
    - DataBinding
    - 源码分析
---

本章从应用层角度分析代码运行过程:QQ音乐团队的分析很完美哈

#### 入口


对于`activity_main.xml`,会生成`ActivityMainBinding`类:

```java
ActivityMainBinding binding = DataBindingUtil.setContentView(this, R.layout.activity_main);`
return bindToAddedViews(bindingComponent, contentView, 0, layoutId);
```

对于最普通的例子而言,这里的`bindingComponent`是null,`contentView`是`android.R.id.content ContentFrameLayout`,它的子`View`就是`layout`

最后进入

```java
static <T extends ViewDataBinding> T bind(DataBindingComponent bindingComponent, View root,int layoutId) {
    return (T) sMapper.getDataBinder(bindingComponent, root, layoutId);
}
```

在`DataBinderMapper`中,根据`layoutId`选择`ActivityMainBinding`并进行初始化

```java
public ActivityMainBinding(DataBindingComponent bindingComponent, View root) {
    super(bindingComponent, root, 1);
    Object[] bindings = mapBindings(bindingComponent, root, 4, sIncludes, sViewsWithIds);
    this.mboundView0 = (LinearLayout)bindings[0];
    this.mboundView0.setTag((Object)null);
    this.mboundView1 = (TextView)bindings[1];
    this.mboundView1.setTag((Object)null);
    this.mboundView2 = (TextView)bindings[2];
    this.mboundView2.setTag((Object)null);
    this.mboundView3 = (Button)bindings[3];
    this.mboundView3.setTag((Object)null);
    this.setRootTag(root);
    this.mCallback1 = new android.databinding.generated.callback.OnClickListener(this, 1);
    this.invalidateAll();
}
```

关键在于`mapBinding`,先看一下父类的构造函数吧！

#### ViewDataBinding

```java
protected ViewDataBinding(DataBindingComponent bindingComponent, View root, int localFieldCount) {
    this.mBindingComponent = bindingComponent;
    this.mLocalFieldObservers = new ViewDataBinding.WeakListener[localFieldCount];
    this.mRoot = root;
    if(Looper.myLooper() == null) {
        throw new IllegalStateException("DataBinding must be created in view\'s UI Thread");
    } else {
        if(USE_CHOREOGRAPHER) {
            this.mChoreographer = Choreographer.getInstance();
            this.mFrameCallback = new FrameCallback() {
                public void doFrame(long frameTimeNanos) {
                    ViewDataBinding.this.mRebindRunnable.run();
                }
            };
        } else {
            this.mFrameCallback = null;
            this.mUIThreadHandler = new Handler(Looper.myLooper());
        }
    }
}
```

#### mapBindings

```java
protected static Object[] mapBindings(DataBindingComponent bindingComponent, View root,int numBindings, IncludedLayouts includes, SparseIntArray viewsWithIds) {
    Object[] bindings = new Object[numBindings];
    mapBindings(bindingComponent, root, bindings, includes, viewsWithIds, true);
        return bindings;
    }
```
在编译后,所有`view`(除了根)之外都会变成`binding-1`,`bidning-2`等递增的名称,因此如下代码就按此处理:

```java
private static void mapBindings(DataBindingComponent bindingComponent, View view,Object[] bindings, IncludedLayouts includes, SparseIntArray viewsWithIds,boolean isRoot) {
    final int indexInIncludes;
    final ViewDataBinding existingBinding = getBinding(view);//即view.getTag("R.id.dataBinding")拿到绑定的ViewDataBinding,这里还是null
    if (existingBinding != null) {
        return;
    }
    Object objTag = view.getTag();
    final String tag = (objTag instanceof String) ? (String) objTag : null;//layout/activity_main_0
    boolean isBound = false;
    if (isRoot && tag != null && tag.startsWith("layout")) {
        final int underscoreIndex = tag.lastIndexOf('_');
        if (underscoreIndex > 0 && isNumeric(tag, underscoreIndex + 1)) {
            final int index = parseTagInt(tag, underscoreIndex + 1);
            if (bindings[index] == null) {
                bindings[index] = view; //bindings[0] = rootView
            }
            indexInIncludes = includes == null ? -1 : index;
            isBound = true;
        } else {
            indexInIncludes = -1;
        }
    } else if (tag != null && tag.startsWith(BINDING_TAG_PREFIX)) {
        int tagIndex = parseTagInt(tag, BINDING_NUMBER_START);
        if (bindings[tagIndex] == null) {
            bindings[tagIndex] = view;//bindings[1]= xxView
        }
        isBound = true;
        indexInIncludes = includes == null ? -1 : tagIndex;
    } else {
        // Not a bound view
        indexInIncludes = -1;
    }
    if (!isBound) {
        final int id = view.getId();
        if (id > 0) {
            int index;
            if (viewsWithIds != null && (index = viewsWithIds.get(id, -1)) >= 0 &&bindings[index] == null) {
                bindings[index] = view;
            }
        }
    }
    if (view instanceof  ViewGroup) {
        final ViewGroup viewGroup = (ViewGroup) view;
        final int count = viewGroup.getChildCount();
        int minInclude = 0;
        for (int i = 0; i < count; i++) {
            final View child = viewGroup.getChildAt(i);
            boolean isInclude = false;
            ...... //与include有关,暂不分析
            if (!isInclude) {
                mapBindings(bindingComponent, child, bindings, includes, viewsWithIds, false); //map子节点
            }
        }
    }
}
```

#### setRootTag

为根视图设置tag
```java
protected void setRootTag(View view) {
    if (USE_TAG_ID) {
        view.setTag(R.id.dataBinding, this);
    } else {
        view.setTag(this);
    }
}
```

#### invalidateAll

```java
@Override
public void invalidateAll() {
    synchronized(this) {
        mDirtyFlags = 0x40L;
    }
    requestRebind();
}
```
设置`flag`,同时要求重新绑定

```java
protected void requestRebind() {
    synchronized (this) {
        //如果正在pendingRebind,退出
        if (mPendingRebind) {
            return;
        }
        mPendingRebind = true;
    }
    if (USE_CHOREOGRAPHER) {
        mChoreographer.postFrameCallback(mFrameCallback);
    } else {
        mUIThreadHandler.post(mRebindRunnable);
    }
}
```

进入之前`Choreographer`设置的`CallBack`

```java
private final Runnable mRebindRunnable = new Runnable() {
    @Override
    public void run() {
        synchronized (this) {
            mPendingRebind = false;
        }
        if (VERSION.SDK_INT >= VERSION_CODES.KITKAT) {
            // Nested so that we don't get a lint warning in IntelliJ
            if (!mRoot.isAttachedToWindow()) {
                mRoot.removeOnAttachStateChangeListener(ROOT_REATTACHED_LISTENER);
                                       mRoot.addOnAttachStateChangeListener(ROOT_REATTACHED_LISTENER);
                return;
            }
        }
        executePendingBindings();
    }
};
```

>在API 19及以上的版本，检查下UI控件是否附加到了窗口上，如果没有附到窗口上，则设置监听器，以便在UI附加到窗口上的时候立即执行rebind操作，然后返回。当符合执行条件(API 19以下或UI控件已经附加到窗口上)的时候，则调用executePendingBindings执行binding逻辑。


##### executeBindings()

这个函数可以理解成数据更新后进行刷新,进入子类执行绑定:代码不完全列出,举例即可

```java
 @Override
protected void executeBindings() {
    long dirtyFlags = 0;
    synchronized(this) {
        dirtyFlags = mDirtyFlags;
        mDirtyFlags = 0;
    }
    java.lang.String firstNameUser = null;
    com.android.databindingtest.Task task = mTask;
    java.lang.String lastNameUser = null;
    com.android.databindingtest.Presenter presenter = mPresenter;
    com.android.databindingtest.User user = mUser;
    android.view.View.OnClickListener androidViewViewOnCli = null;
    com.android.databindingtest.MyHandler handlers = mHandlers;
    if ((dirtyFlags & 0x71L) != 0) {
        if ((dirtyFlags & 0x51L) != 0) {
            if (user != null) {
                firstNameUser = user.getFirstName();
            }
        }
        if ((dirtyFlags & 0x61L) != 0) {
            if (user != null) {
                    // read user.lastName
                lastNameUser = user.getLastName();
            }
        }
```

可以看到,根据`dirtyFlag`的脏的位来进行数据更新。接下来有两个问题,首先这个脏位是谁设置的,其次点击事件如何处理。

#### 脏位设置

在编译生成的`ActivityMainBinding`中,自动会生成`set`等函数,例如:

![](/img/2017-05-31-databinding-1/14962198889807.jpg)
```java
public boolean setVariable(int variableId, Object variable) {
    switch(variableId) {
        case BR.task :
            setTask((com.android.databindingtest.Task) variable);
            return true;
        case BR.presenter :
            setPresenter((com.android.databindingtest.Presenter) variable);
            return true;
        case BR.user :
            setUser((com.android.databindingtest.User) variable);
            return true;
        case BR.handlers :
            setHandlers((com.android.databindingtest.MyHandler) variable);
            return true;
    }
    return false;
}
```

我们看到,只要数据变了,在`BaseObservable`中:

```java
public void notifyPropertyChanged(int fieldId) {
    if (mCallbacks != null) {
        mCallbacks.notifyCallbacks(this, fieldId, null);
    }
}
```

同时会请求重新`rebind`

同时,在`setUser`中比较特殊,`updateRegistration(0, user);` 因为对于`User`,我们设置可能会更改`field`,因此需要绑定监听器:

```java
/**
 * Method object extracted out to attach a listener to a bound Observable object.
*/
private static final CreateWeakListener CREATE_PROPERTY_LISTENER = new CreateWeakListener() {
    @Override
    public WeakListener create(ViewDataBinding viewDataBinding, int localFieldId) {
        return new WeakPropertyListener(viewDataBinding, localFieldId).getListener();
    }
};
```

![](/img/2017-05-31-databinding-1/14962198889807.jpg)

当发生变化时,`CallBackRegistry`会发出通知:

```java
public synchronized void notifyCallbacks(T sender, int arg, A arg2) {
    mNotificationLevel++;
    notifyRecurse(sender, arg, arg2);
    mNotificationLevel--;
    if (mNotificationLevel == 0) {
        if (mRemainderRemoved != null) {
            for (int i = mRemainderRemoved.length - 1; i >= 0; i--) {
                final long removedBits = mRemainderRemoved[i];
                if (removedBits != 0) {
                    removeRemovedCallbacks((i + 1) * Long.SIZE, removedBits);
                    mRemainderRemoved[i] = 0;
                }
            }
        }
        if (mFirst64Removed != 0) {
            removeRemovedCallbacks(0, mFirst64Removed);
            mFirst64Removed = 0;
        }
    }
}
```
在`WeakListener`中收到属性变化的通知:

```java
@Override
public void onPropertyChanged(Observable sender, int propertyId) {
    ViewDataBinding binder = mListener.getBinder();
    if (binder == null) {
        return;
    }
    Observable obj = mListener.getTarget();
    if (obj != sender) {
        return; // notification from the wrong object?
    }
    binder.handleFieldChange(mListener.mLocalFieldId, sender, propertyId);
}
```

然后`ViewDataBinding`会收到变化的通知,并重新绑定。
```java
private void handleFieldChange(int mLocalFieldId, Object object, int fieldId) {
    boolean result = onFieldChange(mLocalFieldId, object, fieldId);
    if (result) {
        requestRebind();
    }
}
```

#### 点击事件

存在以下两个与点击事件有关的函数:

```java
public final void _internalCallbackOnClick(int sourceId , android.view.View callbackArg_0) {
    com.android.databindingtest.Task task = mTask;
    com.android.databindingtest.Presenter presenter = mPresenter;
    // presenter != null
    boolean presenterObjectnull = false;
    presenterObjectnull = (presenter) != (null);
    if (presenterObjectnull) {
        presenter.onSaveClick(task);
    }
}

// Listener Stub Implementations
public static class OnClickListenerImpl implements android.view.View.OnClickListener{
    private com.android.databindingtest.MyHandler value;
    public OnClickListenerImpl setValue(com.android.databindingtest.MyHandler value) {
        this.value = value;
        return value == null ? null : this;
    }
    @Override
    public void onClick(android.view.View arg0) {
        this.value.onClickFriend(arg0);
    }
}
```


在`layout`中,我们定义了两种点击事件:

```xml
<TextView android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:text="@{user.lastName}"
    android:onClick="@{handlers::onClickFriend}"/>
<Button android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:onClick="@{() -> presenter.onSaveClick(task)}" />
```

对于第一种,生成`OnClickListenerImpl`,`this.value.onClickFriend(arg0);`

对于第二种,`mCallback1 = new android.databinding.generated.callback.OnClickListener(this, 1);`

