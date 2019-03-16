---
title: LeakCanary源码分析
copyright: true
categories: android
date: 2018-11-28 17:26:41
tags: 源码解析
---



# LeakCanary源码分析
> 基于版本  1.6.3

## 使用

### 依赖
```java
dependencies {
  debugImplementation 'com.squareup.leakcanary:leakcanary-android:1.6.3'
  releaseImplementation 'com.squareup.leakcanary:leakcanary-android-no-op:1.6.3'
  // Optional, if you use support library fragments:
  debugImplementation 'com.squareup.leakcanary:leakcanary-support-fragment:1.6.3'
}
```

### 初始化

Application 注入 初始化代码
```java
public class ExampleApplication extends Application {

  @Override public void onCreate() {
    super.onCreate();
    //LeakCanary.isInAnalyzerProcess(this)会调用 sInServiceProcess(context, HeapAnalyzerService.class)来判断是否跟HeapAnalyzerService在同一个进程中。
    if (LeakCanary.isInAnalyzerProcess(this)) {
      // This process is dedicated to LeakCanary for heap analysis.
      // You should not init your app in this process.
      return;
    }
    LeakCanary.install(this);
    // Normal app init code...
  }
}
```
aar 自带  以下不需要开发者编写
HeapAnalyzerService是一个IntentServcie，主要用来分析生成的hprof文件。我们看下HeapAnalyzerService的清单配置：
```xml
<service
        android:name=".internal.HeapAnalyzerService"
        android:process=":leakcanary"
        android:enabled="false"
/>
```
可以看到，这个服务运行在单独的“:leakcanary”进程中。如果你浏览下LeakCanary的其他android组件（DisplayLeakService，DisplayLeakActivity等）清单配置，会发现它们都是运行在这个进程中的。为什么HeapAnalyzerService要单独放在这个进程？因为内存泄露时，生成的hprof文件很大，解析的时候会耗费大量内存，如果放在app主进程中，可能会导致OOM。

另外，我们也注意到`android:enabled="false"`这个属性设置，那说明默认情况下，HeapAnalyzerService这个service是不可用的，**那它什么时候打开呢？** 带着这个疑问我们接着往下看：

```java
//com.squareup.leakcanary.internal.LeakCanaryInternals
public static boolean isInServiceProcess(Context context, Class<? extends Service> serviceClass) {
    PackageManager packageManager = context.getPackageManager();
    PackageInfo packageInfo;
    try {
      packageInfo = packageManager.getPackageInfo(context.getPackageName(), GET_SERVICES);
    } catch (Exception e) {
      CanaryLog.d(e, "Could not get package info for %s", context.getPackageName());
      return false;
    }
    String mainProcess = packageInfo.applicationInfo.processName;

    ComponentName component = new ComponentName(context, serviceClass);
    ServiceInfo serviceInfo;
    try {
      serviceInfo = packageManager.getServiceInfo(component, 0);
    } catch (PackageManager.NameNotFoundException ignored) {
      // Service is disabled.
      /****标记1*******.首次运行的时候，HeapAnalyzerService组件没有打开，所以到这里就返回了***/
      return false;
    }
    //HeapAnalyzerService激活后才会有下边的代码
    if (serviceInfo.processName.equals(mainProcess)) {/****标记2*******/
      CanaryLog.d("Did not expect service %s to run in main process %s", serviceClass, mainProcess);
      // Technically we are in the service process, but we're not in the service dedicated process.
      return false;
    }

    int myPid = android.os.Process.myPid();
    ActivityManager activityManager =
        (ActivityManager) context.getSystemService(Context.ACTIVITY_SERVICE);
    ActivityManager.RunningAppProcessInfo myProcess = null;
    List<ActivityManager.RunningAppProcessInfo> runningProcesses;
    try {
      runningProcesses = activityManager.getRunningAppProcesses();
    } catch (SecurityException exception) {
      // https://github.com/square/leakcanary/issues/948
      CanaryLog.d("Could not get running app processes %d", exception);
      return false;
    }
    if (runningProcesses != null) {
      for (ActivityManager.RunningAppProcessInfo process : runningProcesses) {
        if (process.pid == myPid) {
          myProcess = process;
          break;
        }
      }
    }
    if (myProcess == null) {
      CanaryLog.d("Could not find running process for %d", myPid);
      return false;
    }

    return myProcess.processName.equals(serviceInfo.processName);
  }
```

上边我们说过，HeapAnalyzerService组件默认是关闭的，所以一次执行的时候，到代码中1标识的位置就结束了。第二次执行的时候，才会往下走。此外，在2标记的的位置,如果你把HeapAnalyzerService的进程设置为跟主进程一样，LeakCanary依然可以工作，但要小心OOM。再往下就是判断HeapAnalyzerService服务进程是否跟主进程一样。

### 查看内存泄漏


## 原理

### 如何判断内存泄漏
> 弱引用探测内存泄露
WeakReference(T referent, ReferenceQueue<? super T> q)
referent被gc回收时，会将包裹它的弱引用注册到ReferenceQueue中，在gc后判断ReferenceQueue有没有referent包裹的WeakReference，就可以判断是否被gc正常回收。
 
### `LeakCanary.install`

```java
public static @NonNull RefWatcher install(@NonNull Application application) {
    return refWatcher(application).listenerServiceClass(DisplayLeakService.class)
        .excludedRefs(AndroidExcludedRefs.createAppDefaults().build())
        .buildAndInstall();
}
public static @NonNull AndroidRefWatcherBuilder refWatcher(@NonNull Context context) {
    return new AndroidRefWatcherBuilder(context);
}
```
intall方法中，生成了AndroidRefWatcherBuilder对象，设置了一些参数，并最终调用了buildAndInstall()。

### `AndroidRefWatcherBuilder`

|参数类型|默认实现|android平台实现类|含义|
| ---  | ---  |  ---|  ---  | 
|HeapDump.Listener	| HeapDump.Listener.NONE	|ServiceHeapDumpListener	 |执行分析HeapDump，在android中是在启动DisplayLeakService在一个新的的进程中，执行解析dump  |
|DebuggerControl 	|DebuggerControl.NONE	|AndroidDebuggerControl	|判断进程是否处于debug状态   |
|HeapDumper	 |HeapDumper.NONE	|AndroidHeapDumper	|dump heap到文件中  |
|GcTrigger	|GcTrigger.DEFAULT	|GcTrigger.DEFAULT	|主动调用GC|
|WatchExecutor	|WatchExecutor.NONE	|AndroidWatchExecutor	|延迟调用GC的执行器；在android平台是内部有一个HandThread，通过 handler来做延迟操作|
|HeapDump.Builder	|HeapDump.Builder	|HeapDump.Builder	|主要是构造执行dump时的一些参数|


注意两点：

+ AndroidRefWatcherBuilder与RefWatcherBuilder使用了builder模式，可以看下是怎么用泛型实现Builder模式的继承结构的。
+ 参数里边的默认实现，都是在接口定义中使用匿名内部类定义了一个默认的的实现，这种写法可以参考，比如DebuggerControl的定义
```java
public interface DebuggerControl {
  DebuggerControl NONE = new DebuggerControl() {
    @Override public boolean isDebuggerAttached() {
      return false;
    }
  };

  boolean isDebuggerAttached();
}
```

上边说的HeapDump.Builder（Builder模式），最终会构造一个HeapDump对象，看下他有哪些参数：

|参数名称	|类型	 |含义    |
|   ---  |  ---  |   ---  |
|heapDumpFile	|File	 |hprof文件路径  |
|referenceKey	|String	 |是一个UUID，用来唯一标识一个要监测的对象的|
|referenceName	|String	 |用户给监控对象自己定义的名字|
|excludedRefs	|ExcludedRefs	|要排除的类集合；因为有些内存泄露，不|是我们的程序导致的，而是系统的bug，这些bug我们无能为力，所以做了这样一个列|表，把这些泄露问题排除掉，不会展示给我们|
|watchDurationMs	|long	|执行RefWatcher.watch(）之后，到最终监测到内|存泄露花费的时间|
|gcDurationMs	|long	|手动执行gc花费的时间|
|heapDumpDurationMs	|long	|记录dump内存花费的时间|
|computeRetainedHeapSize	|boolean	|是否计算持有的堆的大小|
|reachabilityInspectorClasses	|List<Class<? extends Reachability.Inspector>>	| ？  |

+  AndroidRefWatcherBuilder#listenerServiceClass()

```java
/**
  * Sets a custom {@link AbstractAnalysisResultService} to listen to analysis results. This
  * overrides any call to {@link #heapDumpListener(HeapDump.Listener)}.
*/
public @NonNull AndroidRefWatcherBuilder listenerServiceClass(
      @NonNull Class<? extends AbstractAnalysisResultService> listenerServiceClass) {
    enableDisplayLeakActivity = DisplayLeakService.class.isAssignableFrom(listenerServiceClass);
    return heapDumpListener(new ServiceHeapDumpListener(context, listenerServiceClass));
  }
```

`ServiceHeapDumpListener`：先简单说下这个listsener的作用，当监测到内存泄露时，就会hprof文件回调给这个listener，在回调中它会启启动独立的进程，解析这这个文件，并将解析结果传给`AbstractAnalysisResultService`的实现类，也即install方法中传入的`DisplayLeakService`，`DisplayLeakService`会弹出系统通知提示发生了内存泄露，这个listener将整个流程串了起来。 `DisplayLeakService` 最后会讲到!
再往下看最后调用的`AndroidRefWatcherBuilder.buildAndInstall()`：

```java
/**
 * Creates a {@link RefWatcher} instance and makes it available through {@link
 * LeakCanary#installedRefWatcher()}.
 *
 * Also starts watching activity references if {@link #watchActivities(boolean)} was set to true.
 *
 * @throws UnsupportedOperationException if called more than once per Android process.
 */
public @NonNull RefWatcher buildAndInstall() {
  if (LeakCanaryInternals.installedRefWatcher != null) {
    throw new UnsupportedOperationException("buildAndInstall() should only be called once.");
  }
  RefWatcher refWatcher = build();
  if (refWatcher != DISABLED) {
    if (enableDisplayLeakActivity) {
      LeakCanaryInternals.setEnabledAsync(context, DisplayLeakActivity.class, true);
    }
    if (watchActivities) {
      ActivityRefWatcher.install(context, refWatcher);
    }
    if (watchFragments) {
      FragmentRefWatcher.Helper.install(context, refWatcher);
    }
  }
  LeakCanaryInternals.installedRefWatcher = refWatcher;
  return refWatcher;
}
```

build()是用之前传的参数构造了一个RefWatcher对象返回给我们，检测对象内存泄露的逻辑都是这个类处理的。
接下来异步激活组件DisplayLeakActivity，它是用来展示泄露列表信息的，激活组件调用的是 **packageManager.setComponentEnabledSetting()** 这个api，它可以实现一些特殊的功能，比如如果有系统权限，可以屏蔽应用开机监听广播，有兴趣的可以google一下。因为这个方法可能会阻塞IPC通信，所以放在了异步里边执行。

+ LeakCanaryInternals.setEnabledAsync()

```java
public static void setEnabledAsync(Context context, final Class<?> componentClass,
      final boolean enabled) {
    final Context appContext = context.getApplicationContext();
    AsyncTask.THREAD_POOL_EXECUTOR.execute(new Runnable() {
      @Override public void run() {
        setEnabledBlocking(appContext, componentClass, enabled);
      }
    });
}
public static void setEnabledBlocking(Context appContext, Class<?> componentClass,
      boolean enabled) {
    ComponentName component = new ComponentName(appContext, componentClass);
    PackageManager packageManager = appContext.getPackageManager();
    int newState = enabled ? COMPONENT_ENABLED_STATE_ENABLED : COMPONENT_ENABLED_STATE_DISABLED;
    // Blocks on IPC.
    packageManager.setComponentEnabledSetting(component, newState, DONT_KILL_APP);
}
```

激活之后，我们桌面上会多一个图标，为什么会多一个图标？谜底就在他的清单配置中的intent-filter

```xml
激活之后，我们桌面上会多一个图标，为什么会多一个图标？谜底就在他的清单配置中的intent-filter

<activity
        android:theme="@style/leak_canary_LeakCanary.Base"
        android:name=".internal.DisplayLeakActivity"
        android:process=":leakcanary"
        ....
        >
      <intent-filter>
        <action android:name="android.intent.action.MAIN"/>
        <category android:name="android.intent.category.LAUNCHER"/>
      </intent-filter>
</activity>

```

那具体什么时候触发检测的呢？这就是ActivityRefWatcher和FragmentRefWatcher要做的事儿。对于Activity和Fragment来说，当它的onDestory()被回调之后，就应该被系统回调掉。

### `ActivityRefWatcher`
 
+  com.squareup.leakcanary.ActivityRefWatcher#install()

```java
public static void install(@NonNull Context context, @NonNull RefWatcher refWatcher) {
    Application application = (Application) context.getApplicationContext();
    ActivityRefWatcher activityRefWatcher = new ActivityRefWatcher(application, refWatcher);

    application.registerActivityLifecycleCallbacks(activityRefWatcher.lifecycleCallbacks);
}
private final Application.ActivityLifecycleCallbacks lifecycleCallbacks =
      new ActivityLifecycleCallbacksAdapter() {
        @Override public void onActivityDestroyed(Activity activity) {
          refWatcher.watch(activity);
        }
};
```
到这里我们就能证实了当`Acitivity.onDestroy()` 监听内存泄露 `refWatcher.watch(activity);`

### `FragmentRefWatcher`

+ com.squareup.leakcanary.internal.FragmentRefWatcher.Helper#install()

```java
public static void install(Context context, RefWatcher refWatcher) {
  List<FragmentRefWatcher> fragmentRefWatchers = new ArrayList<>();
  if (SDK_INT >= O) {
    fragmentRefWatchers.add(new AndroidOFragmentRefWatcher(refWatcher));
  }
  try {
    Class<?> fragmentRefWatcherClass = Class.forName(SUPPORT_FRAGMENT_REF_WATCHER_CLASS_NAME);
    Constructor<?> constructor =
        fragmentRefWatcherClass.getDeclaredConstructor(RefWatcher.class);
    FragmentRefWatcher supportFragmentRefWatcher =
        (FragmentRefWatcher) constructor.newInstance(refWatcher);
    fragmentRefWatchers.add(supportFragmentRefWatcher);
  } catch (Exception ignored) {
  }
  if (fragmentRefWatchers.size() == 0) {
    return;
  }
  Helper helper = new Helper(fragmentRefWatchers);
  Application application = (Application) context.getApplicationContext();
  application.registerActivityLifecycleCallbacks(helper.activityLifecycleCallbacks);
}
private final Application.ActivityLifecycleCallbacks activityLifecycleCallbacks =
    new ActivityLifecycleCallbacksAdapter() {
      @Override public void onActivityCreated(Activity activity, Bundle savedInstanceState) {
        for (FragmentRefWatcher watcher : fragmentRefWatchers) {
          watcher.watchFragments(activity);
        }
      }
};
```
Fragment的onDestory() `监听的是Acitivity#onDestory()` 监听要麻烦一些，我们知道以前不管是原生和support库中的Fragment是没有统一的系统回调的，后来在Support库中和android O以上的版本增加了对fragment的生命周期回调`fragmentManager.registerFragmentLifecycleCallbacks()`。
我们只看下是如何对Support库中的fragment处理的。

+ com.squareup.leakcanary.internal.AndroidOFragmentRefWatcher

```java
@RequiresApi(Build.VERSION_CODES.O) //
class AndroidOFragmentRefWatcher implements FragmentRefWatcher {

  private final RefWatcher refWatcher;

  AndroidOFragmentRefWatcher(RefWatcher refWatcher) {
    this.refWatcher = refWatcher;
  }

  private final FragmentManager.FragmentLifecycleCallbacks fragmentLifecycleCallbacks =
      new FragmentManager.FragmentLifecycleCallbacks() {

        @Override public void onFragmentViewDestroyed(FragmentManager fm, Fragment fragment) {
          View view = fragment.getView();
          if (view != null) {
            refWatcher.watch(view);
          }
        }

        @Override
        public void onFragmentDestroyed(FragmentManager fm, Fragment fragment) {
          refWatcher.watch(fragment);
        }
      };

  @Override public void watchFragments(Activity activity) {
    FragmentManager fragmentManager = activity.getFragmentManager();
    fragmentManager.registerFragmentLifecycleCallbacks(fragmentLifecycleCallbacks, true);
  }
}
```

+ com.squareup.leakcanary.internal.SupportFragmentRefWatcher

```java
class SupportFragmentRefWatcher implements FragmentRefWatcher {
  private final RefWatcher refWatcher;

  SupportFragmentRefWatcher(RefWatcher refWatcher) {
    this.refWatcher = refWatcher;
  }

  private final FragmentManager.FragmentLifecycleCallbacks fragmentLifecycleCallbacks =
      new FragmentManager.FragmentLifecycleCallbacks() {

        @Override public void onFragmentViewDestroyed(FragmentManager fm, Fragment fragment) {
          View view = fragment.getView();
          if (view != null) {
            refWatcher.watch(view);
          }
        }

        @Override public void onFragmentDestroyed(FragmentManager fm, Fragment fragment) {
          refWatcher.watch(fragment);
        }
      };

  @Override public void watchFragments(Activity activity) {
    if (activity instanceof FragmentActivity) {
      FragmentManager supportFragmentManager =
          ((FragmentActivity) activity).getSupportFragmentManager();
      supportFragmentManager.registerFragmentLifecycleCallbacks(fragmentLifecycleCallbacks, true);
    }
  }
}
```
可以看到，对于Fragment来对，LeakCanary除了监控fragment又没有回收外，还会监测Fragment的view有没有被及时回收。AndroidOFragmentRefWatcher的代码跟上边的类似。
在android O及以上版本是通过`AndroidOFragmentRefWatcher`来处理，如果用的是最新的support库，是通过`SupportFragmentRefWatcher`来处理,他们都继承在FragmentRefWatcher。`AndroidOFragmentRefWatcher`和 `SupportFragmentRefWatcher` 代码基本一样为了适配API !

### `RefWatcher.watch()`
>从上可以看到，不管是`Fragment`,`Activity`还是`View`最终都调用了同一个`RefWatcher.watch()`方法，

以Activity：

```java
/**
 * Watches the provided references and checks if it can be GCed. This method is non blocking,
 * the check is done on the {@link WatchExecutor} this {@link RefWatcher} has been constructed
 * with.
 *
 * @param referenceName An logical identifier for the watched object.
 */
public void watch(Object watchedReference, String referenceName) {
  if (this == DISABLED) {
    return;
  }
  checkNotNull(watchedReference, "watchedReference");
  checkNotNull(referenceName, "referenceName");
  final long watchStartNanoTime = System.nanoTime();
  String key = UUID.randomUUID().toString();
  retainedKeys.add(key);
  final KeyedWeakReference reference =
      new KeyedWeakReference(watchedReference, key, referenceName, queue);
  ensureGoneAsync(watchStartNanoTime, reference);
}

private void ensureGoneAsync(final long watchStartNanoTime, final KeyedWeakReference reference) {
    // com.squareup.leakcanary.AndroidWatchExecutor   watchExecutor
    watchExecutor.execute(new Retryable() {
      @Override public Retryable.Result run() {
        return ensureGone(reference, watchStartNanoTime);
      }
    });
}
```

`KeyedWeakReference` 继承自WeakReference，增加的key和referenceName，这个key会同时放入Set集合中，方便gc回收对象后，做对比。
`watchExecutor` 在Android中实现是`AndroidWatchExecutor`
```java
// com.squareup.leakcanary.AndroidWatchExecutor#execute
@Override public void execute(@NonNull Retryable retryable) {
  if (Looper.getMainLooper().getThread() == Thread.currentThread()) {
    waitForIdle(retryable, 0);
  } else {
    postWaitForIdle(retryable, 0);
  }
}

```
`waitForIdle` 和  `postWaitForIdle` 最终还是调用 `waitForIdle`

```java
private void waitForIdle(final Retryable retryable, final int failedAttempts) {
    // This needs to be called from the main thread.
    Looper.myQueue().addIdleHandler(new MessageQueue.IdleHandler() {
      @Override public boolean queueIdle() {
        postToBackgroundWithDelay(retryable, failedAttempts);
        return false;
      }
    });
}
private void postToBackgroundWithDelay(final Retryable retryable, final int failedAttempts) {
    long exponentialBackoffFactor = (long) Math.min(Math.pow(2, failedAttempts), maxBackoffFactor);
    long delayMillis = initialDelayMillis * exponentialBackoffFactor;
    //  HandlerThread
    backgroundHandler.postDelayed(new Runnable() {
      @Override public void run() {
        Retryable.Result result = retryable.run();
        if (result == RETRY) {
          postWaitForIdle(retryable, failedAttempts + 1);
        }
      }
    }, delayMillis);
}
```
这个类会在主UI线程空闲的时候，向内部的HandlerThread发送一个消息，执行execute方法传给他的Retryable对象，
调用`retryable.run()`，如果返回RETRY会继续等待下一个主UI线程空闲回调，直至返回为Done，每次重试都会增大间隔时间。

**注意：是通过 Looper.myQueue().addIdleHandler（）来监听系统主线程空闲的**

到这里我们就应该知道`ensureGone`最开始这个方法被执行的 `HandlerThread`线程中

```java
  @SuppressWarnings("ReferenceEquality") // Explicitly checking for named null.
  Retryable.Result ensureGone(final KeyedWeakReference reference, final long watchStartNanoTime) {
    long gcStartNanoTime = System.nanoTime();
    long watchDurationMs = NANOSECONDS.toMillis(gcStartNanoTime - watchStartNanoTime);

    //第一次调用如果系统GC 减少引用
    removeWeaklyReachableReferences();

    if (debuggerControl.isDebuggerAttached()) {
      // The debugger can create false leaks.
      return RETRY;
    }
    if (gone(reference)) {
      return DONE;
    }
    gcTrigger.runGc();
    //第二次调用 主动GC 在判断引用 减少误报
    removeWeaklyReachableReferences();
    if (!gone(reference)) {
    //到这里就有可能出现内存泄露了
      long startDumpHeap = System.nanoTime();
      long gcDurationMs = NANOSECONDS.toMillis(startDumpHeap - gcStartNanoTime);
    // 关键时刻  heapDumper 实现是 com.squareup.leakcanary.AndroidHeapDumper
      File heapDumpFile = heapDumper.dumpHeap();
      //生成  dump  文件
      if (heapDumpFile == RETRY_LATER) {
        // Could not dump the heap.
        return RETRY;
      }
      long heapDumpDurationMs = NANOSECONDS.toMillis(System.nanoTime() - startDumpHeap);
      // 将dump文件构造成HeapDump 对象
      HeapDump heapDump = heapDumpBuilder.heapDumpFile(heapDumpFile).referenceKey(reference.key)
          .referenceName(reference.name)
          .watchDurationMs(watchDurationMs)
          .gcDurationMs(gcDurationMs)
          .heapDumpDurationMs(heapDumpDurationMs)
          .build();
      //HeapDump 对象 heapdumpListener对应的是ServiceHeapDumpListener的analyze（）
      heapdumpListener.analyze(heapDump);
    }
    return DONE;
  }

  private boolean gone(KeyedWeakReference reference) {
    return !retainedKeys.contains(reference.key);
  }

  private void removeWeaklyReachableReferences() {
    // WeakReferences are enqueued as soon as the object to which they point to becomes weakly
    // reachable. This is before finalization or garbage collection has actually happened.
    KeyedWeakReference ref;
    while ((ref = (KeyedWeakReference) queue.poll()) != null) {
      retainedKeys.remove(ref.key);
    }
  }
```

`removeWeaklyReachableReferences`调用了两次，第一次调用后，如果系统gc正常回收，retainedKeys就不会再包含`KeyedWeakReference` 的key；如果没有回收，会手动调用gc，再判断，减少误报；如果两次gc后，对象仍然没有被回收，可能就是真的内存泄露了。再调用heapDumper.dumpHeap()得到内存数据。heapDumper对应的是`AndroidHeapDumper`，`LeakDirectoryProvider` 指定了dumperHeap()的文件夹路径。
`heapDumper.dumpHeap();`生成文件后，并且生成HeapDump 对象,HeapDump 对象 heapdumpListener对应的是`ServiceHeapDumpListener`的analyze（）
+ com.squareup.leakcanary.AndroidHeapDumper

```java
@Override 
protected @NonNull HeapDumper defaultHeapDumper() {
    LeakDirectoryProvider leakDirectoryProvider =
        LeakCanaryInternals.getLeakDirectoryProvider(context);
    return new AndroidHeapDumper(context, leakDirectoryProvider);
}

@Override @Nullable
public File dumpHeap() {
    File heapDumpFile = leakDirectoryProvider.newHeapDumpFile();

    if (heapDumpFile == RETRY_LATER) {
      return RETRY_LATER;
    }

    FutureResult<Toast> waitingForToast = new FutureResult<>();
    showToast(waitingForToast);

    if (!waitingForToast.wait(5, SECONDS)) {
      CanaryLog.d("Did not dump heap, too much time waiting for Toast.");
      return RETRY_LATER;
    }

    Notification.Builder builder = new Notification.Builder(context)
        .setContentTitle(context.getString(R.string.leak_canary_notification_dumping));
    Notification notification = LeakCanaryInternals.buildNotification(context, builder);
    NotificationManager notificationManager =
        (NotificationManager) context.getSystemService(Context.NOTIFICATION_SERVICE);
    int notificationId = (int) SystemClock.uptimeMillis();
    notificationManager.notify(notificationId, notification);

    Toast toast = waitingForToast.get();
    try {
      Debug.dumpHprofData(heapDumpFile.getAbsolutePath());
      cancelToast(toast);
      notificationManager.cancel(notificationId);
      return heapDumpFile;
    } catch (Exception e) {
      CanaryLog.d(e, "Could not dump heap");
      // Abort heap dump
      return RETRY_LATER;
    }
  }
```
dumpHeap会弹出一个通知，显示Dumping，并调用`Debug.dumpHprofData(heapDumpFile.getAbsolutePath())`来保存内存，如果异常dumpHprofData异常会重试。

如果dump成功heapDumper.dumpHeap()，会构造HeapDump 对象，`ensureGone()   -->  heapdumpListener.analyze(heapDump);` 并传递给`heapdumpListener`。前边我们说过`heapdumpListener`对应的是`ServiceHeapDumpListener`的analyze（）
>  heapdumpListener 是在 `com.squareup.leakcanary.LeakCanary#install()`中调用 `listenerServiceClass`,`heapDumpListener()` 

```java
// com.squareup.leakcanary.ServiceHeapDumpListener#analyze
 @Override 
public void analyze(@NonNull HeapDump heapDump) {
    checkNotNull(heapDump, "heapDump");
    HeapAnalyzerService.runAnalysis(context, heapDump, listenerServiceClass);
}
// com.squareup.leakcanary.internal.HeapAnalyzerService#runAnalysis
public static void runAnalysis(Context context, HeapDump heapDump,
      Class<? extends AbstractAnalysisResultService> listenerServiceClass) {
    setEnabledBlocking(context, HeapAnalyzerService.class, true);
    setEnabledBlocking(context, listenerServiceClass, true);
    Intent intent = new Intent(context, HeapAnalyzerService.class);
    intent.putExtra(LISTENER_CLASS_EXTRA, listenerServiceClass.getName());
    intent.putExtra(HEAPDUMP_EXTRA, heapDump);
    ContextCompat.startForegroundService(context, intent);
}
```
**[注意]**:`DisplayLeakActivity`,已经调用pm.setComponentEnabledSetting()方法激活组件了，但listenerServiceClass对应的`DisplayLeakService`和`HeapAnalyzerService`还没有激活，所以在启动`HeapAnalyzerService`时，要先把这两个组件激活。
`HeapAnalyzerService`是一个前台IntentService，最终回调到他的`onHandleIntentInForeground`方法：

```java
@Override 
protected void onHandleIntentInForeground(@Nullable Intent intent) {
    if (intent == null) {
      CanaryLog.d("HeapAnalyzerService received a null intent, ignoring.");
      return;
    }
    String listenerClassName = intent.getStringExtra(LISTENER_CLASS_EXTRA);
    HeapDump heapDump = (HeapDump) intent.getSerializableExtra(HEAPDUMP_EXTRA);

    HeapAnalyzer heapAnalyzer =
        new HeapAnalyzer(heapDump.excludedRefs, this, heapDump.reachabilityInspectorClasses);
    //生成泄露对象引用链
    AnalysisResult result = heapAnalyzer.checkForLeak(heapDump.heapDumpFile, heapDump.referenceKey,
        heapDump.computeRetainedHeapSize);
      // listenerClassName=DisplayLeakService
    AbstractAnalysisResultService.sendResultToListener(this, listenerClassName, heapDump, result);
}
```

在onHandleIntentInForeground中根据Intent的参数，构造了heapAnalyzer ，调用`heapAnalyzer.checkForLeak()`得到泄露对象的引用链。

```java
public static void sendResultToListener(@NonNull Context context,
      @NonNull String listenerServiceClassName,
      @NonNull HeapDump heapDump,
      @NonNull AnalysisResult result) {
    Class<?> listenerServiceClass;
    // DisplayLeakService
    try {
      listenerServiceClass = Class.forName(listenerServiceClassName);
    } catch (ClassNotFoundException e) {
      throw new RuntimeException(e);
    }
    Intent intent = new Intent(context, listenerServiceClass);

    File analyzedHeapFile = AnalyzedHeap.save(heapDump, result);
    if (analyzedHeapFile != null) {
      intent.putExtra(ANALYZED_HEAP_PATH_EXTRA, analyzedHeapFile.getAbsolutePath());
    }
    ContextCompat.startForegroundService(context, intent);
}
```

返回的结果通过AnalyzedHeap.save(heapDump, result)保存到了一个文件冲，并返回给listenerServiceClassName代表的Sersvice，即DisplayLeakService，这个服务也是一个前台IntentService，最终回调到DisplayLeakService的onHeapAnalyzed方法：


```java
public class DisplayLeakService extends AbstractAnalysisResultService {
  @Override
  protected final void onHeapAnalyzed(@NonNull AnalyzedHeap analyzedHeap) {
    HeapDump heapDump = analyzedHeap.heapDump;
    AnalysisResult result = analyzedHeap.result;

    String leakInfo = leakInfo(this, heapDump, result, true);
    CanaryLog.d("%s", leakInfo);

    heapDump = renameHeapdump(heapDump);
    boolean resultSaved = saveResult(heapDump, result);

    String contentTitle;
    if (resultSaved) {
      PendingIntent pendingIntent =
          DisplayLeakActivity.createPendingIntent(this, heapDump.referenceKey);
      if (result.failure != null) {
        contentTitle = getString(R.string.leak_canary_analysis_failed);
      } else {
        String className = classSimpleName(result.className);
        if (result.leakFound) {
          if (result.retainedHeapSize == AnalysisResult.RETAINED_HEAP_SKIPPED) {
            if (result.excludedLeak) {
              contentTitle = getString(R.string.leak_canary_leak_excluded, className);
            } else {
              contentTitle = getString(R.string.leak_canary_class_has_leaked, className);
            }
          } else {
            String size = formatShortFileSize(this, result.retainedHeapSize);
            if (result.excludedLeak) {
              contentTitle =
                  getString(R.string.leak_canary_leak_excluded_retaining, className, size);
            } else {
              contentTitle =
                  getString(R.string.leak_canary_class_has_leaked_retaining, className, size);
            }
          }
        } else {
          contentTitle = getString(R.string.leak_canary_class_no_leak, className);
        }
      }
      String contentText = getString(R.string.leak_canary_notification_message);
      showNotification(pendingIntent, contentTitle, contentText);
    } else {
      onAnalysisResultFailure(getString(R.string.leak_canary_could_not_save_text));
    }

    afterDefaultHandling(heapDump, result, leakInfo);
  }
}
```
将泄露结果在系统通知栏上显示。


以上，LeakCanary从注册Activity和Fragment的生命周期回调，到`onDestory`中开始`RefWatcher.watch()`,然后dumpHeap，再分析内存泄露引用链，最后将结果展示到系统通知栏的整个过程就基本分析完毕了。

![](https://ws1.sinaimg.cn/large/882b6a2aly1g0hsp3z74ej219m0kfq5s.jpg)


## 总结

总结下看源码过程中学到的知识点：

+ 内存泄露如何判断（WeakReference）
+ Debug.dumpHprofData（）获取内存数据，Debug.isDebuggerConnected()判断是否debug
+ setComponentEnabledSetting()动态开关组件
+ Looper.myQueue().addIdleHandler（）注册UI线程空闲回调


[链接](https://www.jianshu.com/p/b857736d1461)
