---
title: OkHttp源码分析(二) Dispatcher
copyright: true
categories: android
date: 2018-08-19 13:11:24
tags: 源码解析
---


# OkHttp源码分析(二) Dispatcher
> okhttp发起请求的分发器


## 直接撸源码
```java
// okhttp3.OkHttpClient.Builder#dispatcher
public Builder dispatcher(Dispatcher dispatcher) {
      if (dispatcher == null) throw new IllegalArgumentException("dispatcher == null");
      this.dispatcher = dispatcher;
      return this;
    }
    //final 无法继承
public final class Dispatcher {
  private int maxRequests = 64;   //最大请求数量 ，可以单独设置
  private int maxRequestsPerHost = 5; //最大请求Host，可以单独设置
数量
  // 以上是发出请求的两个限制 限制runningAsyncCalls 队列
  
  private @Nullable Runnable idleCallback; //空闲时调用
  private @Nullable ExecutorService executorService; // 异步请求的线程池
  private final Deque<AsyncCall> readyAsyncCalls = new ArrayDeque<>(); // 异步队列
  private final Deque<AsyncCall> runningAsyncCalls = new ArrayDeque<>(); // 正在执行异步队列
  private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>();// 正在执行的同步队列
   
  ...
}

```
但是 okhttp3.Dispatcher 提供了两个构造方法
```java
public Dispatcher(ExecutorService executorService) {
    this.executorService = executorService;
  }
//Dispatcher 默认线程池
public synchronized ExecutorService executorService() {
    if (executorService == null) {
      executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
          new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp Dispatcher", false));
    }
    return executorService;
  } 
// 默认使用 无参构造   
public Dispatcher() {
}

//okhttp3.OkHttpClient.Builder#Builder() 
public Builder() {
   dispatcher = new Dispatcher();
   ...
}
```
**知识点: ExecutorService 线程池相关学习**

### 与发起请求的直接相关的方法：
```java
// 同步请求 调用
synchronized void executed(RealCall call) {
    runningSyncCalls.add(call);
}
// 异步请求 调用
void enqueue(AsyncCall call) {
    synchronized (this) {
      readyAsyncCalls.add(call);
    }
    promoteAndExecute();
}
```
**问题: 以上代码 两个 synchronized 同步的 有什么区别， 为什么这样写？？？**

由上面可见 同步和异步都加入相应的队列中，不同的是 异步会执行 `promoteAndExecute();`
```java
  private boolean promoteAndExecute() {
    assert (!Thread.holdsLock(this));

    List<AsyncCall> executableCalls = new ArrayList<>();
    boolean isRunning;
    synchronized (this) {
      for (Iterator<AsyncCall> i = readyAsyncCalls.iterator(); i.hasNext(); ) {
        AsyncCall asyncCall = i.next();

        if (runningAsyncCalls.size() >= maxRequests) break; // Max capacity.
        if (runningCallsForHost(asyncCall) >= maxRequestsPerHost) continue; // Host max capacity.

        i.remove();
        executableCalls.add(asyncCall);
        runningAsyncCalls.add(asyncCall);
      }
      isRunning = runningCallsCount() > 0;
    }
  //以上是限制请求数量 ，下面开始加入 线程池中 等待执行
    for (int i = 0, size = executableCalls.size(); i < size; i++) {
      AsyncCall asyncCall = executableCalls.get(i);
      asyncCall.executeOn(executorService());
    }
    //这个返回值 在 okhttp3.Dispatcher#finished() 用到，如果 isRunning=false 就会调用  idleCallback.run()
    return isRunning;
  }
```

我们从 同步流程源码[OkHttp 源码(一)  请求的大致流程](mweblib://15465027736165#execute`) ，

异步流程源码[OkHttp 源码(一)  请求的大致流程](mweblib://15465027736165#enqueue)



![](https://ws1.sinaimg.cn/large/882b6a2aly1g14re4ig7kj21860es7cx.jpg)
 

![](https://ws1.sinaimg.cn/large/882b6a2aly1g14rezuemsj218i0k0wql.jpg)