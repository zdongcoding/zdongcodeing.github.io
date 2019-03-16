---
title: Jetpack之Lifecycle的源码分析
copyright: true
categories: android
date: 2018-10-16 13:29:27
tags: [Jetpack,源码分析]
---

# Android Jetpack之Lifecycle的源码分析

Lifecycle组件中的类结构，`LifecycleOwner`表示拥有生命周期功能。
Lifecycle定义了Android中的生命周期的接口。而`LifecycleObserver`是生命周期的监听的接口。Lifecycle可以注册和反注册`LifecycleObserver`，二者为观察者模式。

![](http://ww1.sinaimg.cn/large/882b6a2aly1g0jm1iyj40j23l41boaoo.jpg)


## LifecycleOwner

```java
public interface LifecycleOwner {
    @NonNull
    Lifecycle getLifecycle();
}
```

`LifecycleOwner`只有一个简单的接口，获取`Lifecycle`。而support现在的库中的`Fragment`和`AppCompatActivity`均实现了此接口。

### Fragment

![](http://ww1.sinaimg.cn/large/882b6a2aly1g0jm1r5f1lj20rs0s6mxo.jpg)

现在的support库中的Fragment实现了LifecycleOwner，我们来观摩一下。

```java
@Override
public Lifecycle getLifecycle() {
    return mLifecycleRegistry;
}
```

`Fragment`直接返回变量`mLifecycleRegistry`，

```java
LifecycleRegistry mLifecycleRegistry = new LifecycleRegistry(this);
```

变量是LifecycleRegistry的对象。

`Fragment`实现了`LifecycleOwner`，并返回了`LifecycleRegistry`类的对象。
那`mLifecycleRegistry`怎么知道生命周期变化的呢？
> 在`performCreate`方法中，调用支持`onCreate`方法并使用`mLifecycleRegistry`来处理`Lifecycle.Event.ON_CREATE`。

```java
void performCreate(Bundle savedInstanceState) {
    ······
    onCreate(savedInstanceState);
    ······
    mLifecycleRegistry.handleLifecycleEvent(Lifecycle.Event.ON_CREATE);
}
```

同样的：
-   `performStart`调用`onStart`方法并使用`mLifecycleRegistry`处理`Lifecycle.Event.ON_START`
-   `performResume`调用`onResume`方法并使用`mLifecycleRegistry`处理`Lifecycle.Event.ON_RESUME`
-   `performStop`调用`onStop`方法并使用`mLifecycleRegistry`处理`Lifecycle.Event.ON_STOP`
-   `performDestroy`调用`onDestroy`并使用`mLifecycleRegistry`处理`Lifecycle.Event.ON_DESTROY`

调用生命周期的相关的方法的同时，也使用`mLifecycleRegistry`处理`Lifecycle.Event`中的相应事件。

`mLifecycleRegistry`通过此种方式来监听生命周期的变更的。（至于什么时候执行performXX方法可以自行分析）。


### AppCompatActivity
>   `AppCompatActivity`继承关系比较深，最终基础类为`SupportActivity`

```java
private LifecycleRegistry mLifecycleRegistry = new LifecycleRegistry(this);
@Override
public Lifecycle getLifecycle() {
        return mLifecycleRegistry;
}
```

`SupportActivity`也是返回`LifecycleRegistry`返回。但是使用的`Fragment`来监听的生命周期。

```java
protected void onCreate(@Nullable Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    ReportFragment.injectIfNeededIn(this);
}
```
其实委托给`ReportFragment`处理生命周期。

![](http://ww1.sinaimg.cn/large/882b6a2aly1g0jm93kubtj20rs0mdt9i.jpg)

```java
@Override
public void onActivityCreated(Bundle savedInstanceState) {
    super.onActivityCreated(savedInstanceState);
    dispatchCreate(mProcessListener);
    dispatch(Lifecycle.Event.ON_CREATE);
}
```

在`onActivityCreated`方法中调用`dispatch`方法分发`Lifecycle.Event.ON_CREATE`事件。


```java
private void dispatch(Lifecycle.Event event) {
    Activity activity = getActivity();
    if (activity instanceof LifecycleRegistryOwner) {
        ((LifecycleRegistryOwner) activity).getLifecycle().handleLifecycleEvent(event);
        return;
    }
    //SupportActivity实现了LifecycleOwner，会调用此代码
    if (activity instanceof LifecycleOwner) {
        Lifecycle lifecycle = ((LifecycleOwner) activity).getLifecycle();
        if (lifecycle instanceof LifecycleRegistry) {
            ((LifecycleRegistry) lifecycle).handleLifecycleEvent(event);
        }
    }
}
```

`SupportActivity`实现了`LifecycleOwner`方法，最终会调用到`SupportActivity`的`mLifecycleRegistry`处理事件。
类似:
-   `onStart`方法中发送`Lifecycle.Event.ON_CREATE`事件。
-   `onResume`方法中发送`Lifecycle.Event.ON_RESUME`事件。
-   `onPause`方法中发送`Lifecycle.Event.ON_PAUSE`事件。
-   `onStop`方法中发送`Lifecycle.Event.ON_STOP`事件。
-   `onDestroy`方法中发送`Lifecycle.Event.ON_DESTROY`事件。

`SupportActivity`通过`ReportFragment`间接处理生命周期的监听。

但还有个问题

```java
protected void onSaveInstanceState(Bundle outState) {
    mLifecycleRegistry.markState(Lifecycle.State.CREATED);
    super.onSaveInstanceState(outState);
}
```

通过`onSaveInstanceState()`保存`AppCompatActivity`的状态时，在调用ON_START事件之前，UI认为是不可变的。在`onSaveInstanceState()`之后才会调用`AppCompatActivity`的onStop()方法，在不允许UI状态更改，但是`Lifecycle`尚未移到CREATED状态(`AppCompatActivity`的`onStop()`方法尚未调用)的存在间隙。
为了防止UI变更，因此这里标记了状态为`Lifecycle.State.CREATED`。


### Lifecycle

```java
public abstract class Lifecycle {
    @MainThread
    public abstract void addObserver(@NonNull LifecycleObserver observer);

    @MainThread
    public abstract void removeObserver(@NonNull LifecycleObserver observer);

    @MainThread
    public abstract State getCurrentState();
}
```

`Lifecycle`定义了生命周期类。用于派发生命周期。包含三个方法`addObserver`添加观察者，`removeObserver`移除观察者，`getCurrentState`获取当前的状态。

`LifecycleRegistry`实现了此接口。可以直接使用它来定义自己的`LifecycleOwner`。


### LifecycleRegistry

![](http://ww1.sinaimg.cn/large/882b6a2aly1g0jmdy9c10j20rs0k6dgs.jpg)

我们先跟踪事件处理的逻辑。


```java
class LifeycycleFragment : Fragment() {

    private val mLifecycleObserver: LifecycleObserver = GenericLifecycleObserver { source, event ->
        Log.d("LifeycycleFragment", "source:$source event:$event")
    }
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        lifecycle.addObserver(mLifecycleObserver)
    }
    override fun onCreateView(inflater: LayoutInflater, container: ViewGroup?, savedInstanceState: Bundle?): View? {
        return inflater.inflate(R.layout.fragment_lifecycle, container, false)
    }
    override fun onDestroy() {
        lifecycle.removeObserver(mLifecycleObserver)
        super.onDestroy()
    }
}
```

我们只是简单的在`Fragment`的`onCreate`方法中添加了`LifecycleObserver`。

从`Fragment`的`perfromCreate`方法：


![](http://ww1.sinaimg.cn/large/882b6a2aly1g0jmgka5p5j20rs0jxwei.jpg)

```java
void performCreate(Bundle savedInstanceState) {
    ······
    onCreate(savedInstanceState);
    ······
    mLifecycleRegistry.handleLifecycleEvent(Lifecycle.Event.ON_CREATE);
}
```

`performCreate`有两个功能，一个是调用`onCreate`方法，一个处理生命周期的事件。

我们先分析`onCreate`方法。

我们的`LifeFragment`复写了`onCreate`并添加了生命周期的观察者。

我们来分析添加观察者的逻辑。


### LifecycleRegistry 添加观察者

LifecycleRegistry :

```java
@Override
public void addObserver(@NonNull LifecycleObserver observer) {
    State initialState = mState == DESTROYED ? DESTROYED : INITIALIZED;
    ObserverWithState statefulObserver = new ObserverWithState(observer, initialState);
    ObserverWithState previous = mObserverMap.putIfAbsent(observer, statefulObserver);

    if (previous != null) {
        return;
    }
    LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
    if (lifecycleOwner == null) {
        // it is null we should be destroyed. Fallback quickly
        return;
    }
    ......
}
```

-   首先根据当前的状态获取初始化的状态，创建`ObserverWithState`对象。因为还未处理状态，所以mState的状态为INITIALIZED，初始化的状态也为INITIALIZED。
-   `mObserverMap`调用putIfAbsent添加`ObserverWithState`对象。如果存在就返回原来的对象，因为这里是第一次添加，因此返回的值为null。
-   通过`mLifecycleOwner`（弱引用）获取`LifecycleOwner`。


```java
@Override
public void addObserver(@NonNull LifecycleObserver observer) {

    ......
    boolean isReentrance = mAddingObserverCounter != 0 || mHandlingEvent;
    State targetState = calculateTargetState(observer);
    mAddingObserverCounter++;
    while ((statefulObserver.mState.compareTo(targetState) < 0
            && mObserverMap.contains(observer))) {
        pushParentState(statefulObserver.mState);
        statefulObserver.dispatchEvent(lifecycleOwner, upEvent(statefulObserver.mState));
        popParentState();
        // mState / subling may have been changed recalculate
        targetState = calculateTargetState(observer);
    }

    if (!isReentrance) {
        // we do sync only on the top level.
        sync();
    }
    mAddingObserverCounter--;
}
```

`mAddingObserverCounter`表示有多少个正在添加，`mHandlingEvent`是否有事件正在处理。这两个变量很奇怪，看似是为了支持多线程，但是又没有同步同步锁，又没有`volatile`修饰，只能说明不支持多线程操作，只能在UI线程中操作，一次添加一个观察者，一次只能处理一个事件。因此这两个变量很奇怪。isReentrance肯定是false。

接着是调用`calculateTargetState`方法计算目标的状态，


```java
private State calculateTargetState(LifecycleObserver observer) {
    Entry<LifecycleObserver, ObserverWithState> previous = mObserverMap.ceil(observer);

    State siblingState = previous != null ? previous.getValue().mState : null;
    State parentState = !mParentStates.isEmpty() ? mParentStates.get(mParentStates.size() - 1)
            : null;
    return min(min(mState, siblingState), parentState);
}
```

从mObserverMap获取`ObserverWithState`，并获取`ObserverWithState`状态。前面说过初始化的状态为INITIALIZED。而mState也为INITIALIZED。没有操作mParentStates，其数据parentState为null。最终`calculateTargetState`计算的目标状态为INITIALIZED。

由于statefulObserver.mState的状态与targetState所以`addObserver`不会执行while语句。

由于isReentrance为false。因此会执行`sync`方法


```java
private void sync() {
    ......
    while (!isSynced()) {
        mNewEventOccurred = false;
        // no need to check eldest for nullability, because isSynced does it for us.
        if (mState.compareTo(mObserverMap.eldest().getValue().mState) < 0) {
            backwardPass(lifecycleOwner);
        }
        Entry<LifecycleObserver, ObserverWithState> newest = mObserverMap.newest();
        if (!mNewEventOccurred && newest != null
                && mState.compareTo(newest.getValue().mState) > 0) {
            forwardPass(lifecycleOwner);
        }
    }
    mNewEventOccurred = false;
}
```

通过`isSynced`判断是否需要进行处理。

```java
private boolean isSynced() {
    if (mObserverMap.size() == 0) {
        return true;
    }
    State eldestObserverState = mObserverMap.eldest().getValue().mState;
    State newestObserverState = mObserverMap.newest().getValue().mState;
    return eldestObserverState == newestObserverState && mState == newestObserverState;
}
```

我们已经添加了观察者，而且仅一个观察者，`mObserverMap.eldest()`与`mObserverMap.newest()`，并且状态与mState相同，因此此方法返回true。
因此呢`sync`方法的while语句也不会执行。

至此，我们的`addObserver`已经分析完成。

### LifecycleRegistry 事件处理

接下来我们分析一下`handleLifecycleEvent(Lifecycle.Event.ON_CREATE)`的流程。

```
public void handleLifecycleEvent(@NonNull Lifecycle.Event event) {
    State next = getStateAfter(event);
    moveToState(next);
}
```

首先调用`getStateAfter`方法获取状态，其次是调用`moveToState`方法。

```java
static State getStateAfter(Event event) {
        switch (event) {
            case ON_CREATE:
            case ON_STOP:
                return CREATED;
            case ON_START:
            case ON_PAUSE:
                return STARTED;
            case ON_RESUME:
                return RESUMED;
            case ON_DESTROY:
                return DESTROYED;
            case ON_ANY:
                break;
        }
        throw new IllegalArgumentException("Unexpected event value " + event);
    }
```

`getStateAfter`根据事件获取到状态，此时我们的event为ON_CREATE，因此我们获取的状态为CREATED。

接下来是`moveToState`方法。

```java
private void moveToState(State next) {
    if (mState == next) {
        return;
    }
    mState = next;
    if (mHandlingEvent || mAddingObserverCounter != 0) {
        mNewEventOccurred = true;
        // we will figure out what to do on upper level.
        return;
    }
    mHandlingEvent = true;
    sync();
    mHandlingEvent = false;
}

```

mState为INITIALIZED，next为CREATED，不会直接返回，接着我们修改当前状态变量mState。没有事件正在处理mHandlingEvent为false，也没有正在添加观察者，`mAddingObserverCounter为0`，不会直接返回。

接着调用`sync`方法。

```java
private void sync() {
    ······
    while (!isSynced()) {
        mNewEventOccurred = false;
        // no need to check eldest for nullability, because isSynced does it for us.
        if (mState.compareTo(mObserverMap.eldest().getValue().mState) < 0) {
            backwardPass(lifecycleOwner);
        }
        Entry<LifecycleObserver, ObserverWithState> newest = mObserverMap.newest();
        if (!mNewEventOccurred && newest != null
                && mState.compareTo(newest.getValue().mState) > 0) {
            forwardPass(lifecycleOwner);
        }
    }
    mNewEventOccurred = false;
}
```

接着调用isSynced，由于mState为INITIALIZED，但是呢mObserverMap中的ObserverWithState对象为INITIALIZED，因此返回false。接着执行while语句。由于INITIALIZED小于CREATED，因此会调用`forwardPass`方法。

```java
private void forwardPass(LifecycleOwner lifecycleOwner) {
    Iterator<Entry<LifecycleObserver, ObserverWithState>> ascendingIterator =
            mObserverMap.iteratorWithAdditions();
    while (ascendingIterator.hasNext() && !mNewEventOccurred) {
        Entry<LifecycleObserver, ObserverWithState> entry = ascendingIterator.next();
        ObserverWithState observer = entry.getValue();
        while ((observer.mState.compareTo(mState) < 0 && !mNewEventOccurred
                && mObserverMap.contains(entry.getKey()))) {
            pushParentState(observer.mState);
            observer.dispatchEvent(lifecycleOwner, upEvent(observer.mState));
            popParentState();
        }
    }
}
```

最重要的功能是获取观察者的Iterator，并遍历Iterator比较状态，调用`ObserverWithState`对象分发事件。

其中有一个关键的方法`upEvent`方法。

```java
private static Event upEvent(State state) {
    switch (state) {
        case INITIALIZED:
        case DESTROYED:
            return ON_CREATE;
        case CREATED:
            return ON_START;
        case STARTED:
            return ON_RESUME;
        case RESUMED:
            throw new IllegalArgumentException();
    }
    throw new IllegalArgumentException("Unexpected state value " + state);
}
```

根据状态获取事件，我们传递的是observer.mState为INITIALIZED。因此获取的事件为ON_CREATE。

`upEvent`方法对应还有`downEvent`方法。

```java
private static Event downEvent(State state) {
    switch (state) {
        case INITIALIZED:
            throw new IllegalArgumentException();
        case CREATED:
            return ON_DESTROY;
        case STARTED:
            return ON_STOP;
        case RESUMED:
            return ON_PAUSE;
        case DESTROYED:
            throw new IllegalArgumentException();
    }
    throw new IllegalArgumentException("Unexpected state value " + state);
}
```

这两个方法的逻辑总结为如下的状态图。
-   `upEvent`状态值变大发生的事件，比如INITIALIZED -> CREATED -> STARTED -> RESUMED或者DESTROYED -> CREATED -> STARTED -> RESUMED，这中间发生的事件。
-   `downEvent`状态值变小发生的事件。比如RESUMED -> STARTED -> CREATED -> DESTROYED这中间发生的事件。

![](http://ww1.sinaimg.cn/large/882b6a2aly1g0jmn8oya7j20rs0sl3yu.jpg)

对于`Fragment`的其他生命周期事件的处理可以自行分析，流程基本一致。
`SupportActivity`的生命周期也是类似的分析方式，只是生命周期的控制委托给了`ReportFragment`。原理也是一致。

**注意**: `SupportActivity`的`onSaveInstanceState`就会标记生命周期的状态为`State.CREATED`，并通知观察者的事件ON_STOP，不会等待onStop方法执行。

### LifecycleObserver

添加`LifecycleObserver`有下面几种方式
1.  实现`FullLifecycleObserver`，`GenericLifecycleObserver`类。
2.  实现`LifecycleObserver`并在方法上使用注解`OnLifecycleEvent`。
`LifecycleObserver`整体结构。
 
![](http://ww1.sinaimg.cn/large/882b6a2aly1g0jmoctvbdj20rs0bsq38.jpg)
 
 
`LifecycleObserver`只是一个简单的类，没有任何的接口。

```java
public interface LifecycleObserver {

}
```

有个疑问是，我们的`LifecycleObserver`没有任何的接口，事件是如何正确的分发事件的呢？注解`LifecycleObserver`又是如何起作用的呢？

上一节我们知道了有事件分发，会调用`ObserverWithState`的`dispatchEvent`方法。其实我们通知`LifecycleObserver`就是`ObserverWithState`来完成的。

实现`FullLifecycleObserver`监听生命周期

`FullLifecycleObserver`继承自`LifecycleObserver`。

```java
interface FullLifecycleObserver extends LifecycleObserver {
    void onCreate(LifecycleOwner owner);
    void onStart(LifecycleOwner owner);
    void onResume(LifecycleOwner owner);
    void onPause(LifecycleOwner owner);
    void onStop(LifecycleOwner owner);
    void onDestroy(LifecycleOwner owner);
}
```

`ObserverWithState`会适配`LifecycleObserver`到相应的`GenericLifecycleObserver`。我们来分析一下

```java
static class ObserverWithState {
    State mState;
    GenericLifecycleObserver mLifecycleObserver;
    ObserverWithState(LifecycleObserver observer, State initialState) {
        mLifecycleObserver = Lifecycling.getCallback(observer);
        mState = initialState;
    }
}
```

在构造方法中，会通过`Lifecycling.getCallback`的方法来适配`LifecycleObserver`对象到`GenericLifecycleObserver`对象。

那我们来分析一下`Lifecycling.getCallback`的方法。

```java
public class Lifecycling {
    ······
    @NonNull
    static GenericLifecycleObserver getCallback(Object object) {
        if (object instanceof FullLifecycleObserver) {
            return new FullLifecycleObserverAdapter((FullLifecycleObserver) object);
        }
        ······
    }
    ······
}
```

如果是`FullLifecycleObserver`对象，我们是直接通过`FullLifecycleObserverAdapter`来进行适配。

`FullLifecycleObserverAdapter`代码很简单，这里不做叙述了。

所以我们实现了`FullLifecycleObserver`监听生命周期时，`ObserverWithState`会适配此观察者为`FullLifecycleObserverAdapter`，分发事假时调用`FullLifecycleObserverAdapter`的`onStateChanged`方法进行分发事件。

实现`GenericLifecycleObserver`监听生命周期

```java
public class Lifecycling {
    @NonNull
    static GenericLifecycleObserver getCallback(Object object) {
        ······
        if (object instanceof GenericLifecycleObserver) {
            return (GenericLifecycleObserver) object;
        }
        ······
    }
}
```

同样的，在构造方法中，会通过`Lifecycling.getCallback`的方法来适配`LifecycleObserver`对象到`GenericLifecycleObserver`对象。因为传递的`LifecycleObserver`已经是`GenericLifecycleObserver`对象，所以呢`Lifecycling.getCallback`直接返回此对象。

使用`OnLifecycleEvent`注解监听生命周期

使用注解的方式如下：

```java
public class MyObserver implements LifecycleObserver {
    @OnLifecycleEvent(Lifecycle.Event.ON_RESUME)
    public void connectListener() {
        ...
    }
    @OnLifecycleEvent(Lifecycle.Event.ON_PAUSE)
    public void disconnectListener() {
        ...
    }
}
```

我们继续分析`Lifecycling.getCallback`方法。

```java
@NonNull
static GenericLifecycleObserver getCallback(Object object) {
    ......
    final Class<?> klass = object.getClass();
    int type = getObserverConstructorType(klass);
    if (type == GENERATED_CALLBACK) {
        List<Constructor<? extends GeneratedAdapter>> constructors =
                sClassToAdapters.get(klass);
        if (constructors.size() == 1) {
            GeneratedAdapter generatedAdapter = createGeneratedAdapter(
                    constructors.get(0), object);
            return new SingleGeneratedAdapterObserver(generatedAdapter);
        }
        GeneratedAdapter[] adapters = new GeneratedAdapter[constructors.size()];
        for (int i = 0; i < constructors.size(); i++) {
            adapters[i] = createGeneratedAdapter(constructors.get(i), object);
        }
        return new CompositeGeneratedAdaptersObserver(adapters);
    }
    return new ReflectiveGenericLifecycleObserver(object);
}
```

代码的`getCallback`的调用流程。
 
![](http://ww1.sinaimg.cn/large/882b6a2aly1g0jmoq6njmj20rs0g9q30.jpg)


通过`getObserverConstructorType`方法获取构造的类型，根据此类型来判断使用哪种Adapter来适配。

接下来分析`getObserverConstructorType`方法，

```java
private static int getObserverConstructorType(Class<?> klass) {
    if (sCallbackCache.containsKey(klass)) {
        return sCallbackCache.get(klass);
    }
    int type = resolveObserverCallbackType(klass);
    sCallbackCache.put(klass, type);
    return type;
}
```

首先判断缓存时候包含Class类型缓存。我们是首次进入此方法时不会包含。
接着通过`resolveObserverCallbackType`方法获取类型。

我们来一步步分析`resolveObserverCallbackType`方法。

```java
private static int resolveObserverCallbackType(Class<?> klass) {
    // anonymous class bug:35073837
    if (klass.getCanonicalName() == null) {
        return REFLECTIVE_CALLBACK;
    }
    ......
}
```

-   如果是匿名内部类，就直接返回`REFLECTIVE_CALLBACK`。

```java
private static int resolveObserverCallbackType(Class<?> klass) {
    ......
    Constructor<? extends GeneratedAdapter> constructor = generatedConstructor(klass);
    if (constructor != null) {
        sClassToAdapters.put(klass, Collections
                .<Constructor<? extends GeneratedAdapter>>singletonList(constructor));
        return GENERATED_CALLBACK;
    }
    ......
}
```

-   通过`generatedConstructor`获取构造方法。`generatedConstructor`根据Class获取相关类的构造方法，但是`GeneratedAdapter`接口没有暴露，我们传递的也不会是此接口相关的类，后续版本也许会暴露`GeneratedAdapter`接口。因此这里不需要分析了，直接返回为null。if语句也就不会执行了。

```java
private static int resolveObserverCallbackType(Class<?> klass) {
    ......
    boolean hasLifecycleMethods = ClassesInfoCache.sInstance.hasLifecycleMethods(klass);
    if (hasLifecycleMethods) {
        return REFLECTIVE_CALLBACK;
    }
    ......
}
```

-   通过`ClassesInfoCache.sInstance.hasLifecycleMethods`判断是否有生命周期的方法。接着分析此方法。

```java
boolean hasLifecycleMethods(Class klass) {
    if (mHasLifecycleMethods.containsKey(klass)) {
        return mHasLifecycleMethods.get(klass);
    }
    Method[] methods = getDeclaredMethods(klass);
    for (Method method : methods) {
        OnLifecycleEvent annotation = method.getAnnotation(OnLifecycleEvent.class);
        if (annotation != null) {
            createInfo(klass, methods);
            return true;
        }
    }
    mHasLifecycleMethods.put(klass, false);
    return false;
}
```

-   首先缓存里面没有相关的数据，接着通过`getDeclaredMethods`反射获取Class的方法。接着遍历方法，获取`OnLifecycleEvent`注解，因为我们是使用注解的方式来管理声明周期的，因此可以方法上可以获取到OnLifecycleEvent。因此会调用createInfo方法，并返回true。createInfo其实是根据Class获取声明OnLifecycleEvent注解的方法，并再次封装成MethodReference，并放入缓存中。这里就不分析代码了。
-   `ClassesInfoCache`的方法`hasLifecycleMethods`返回true -> Lifecycling的 `resolveObserverCallbackType`返回REFLECTIVE_CALLBACK -> Lifecycling的`getObserverConstructorType`返回REFLECTIVE_CALLBACK ，那Lifecycling的getCallback将返回`ReflectiveGenericLifecycleObserver`来进行适配。


### 总结
1.  `LifecycleOwner`只有一个简单的接口，获取`Lifecycle`，`Lifecycle`与`LifecycleObserver`采用的是观察者模式进行组织。
2.  `LifecycleRegistry`实现Lifecycle接口，可以使用此类来实现自己的生命周期。`LifecycleRegistry`内存采用了适配器模式，把`LifecycleObserver`是适配成`GenericLifecycleObserver`。
3.  `LifecycleObserver`没有任何接口，我们实现此接口时可以利用注解的方式，生命周期的变更执行响应的方法。

