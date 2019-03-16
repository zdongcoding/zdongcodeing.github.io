---
title: OkHttp源码分析 Interceptor
copyright: true
categories: android
date: 2018-08-20 10:11:24
tags: 源码解析
---


# OkHttp源码分析 Interceptor


## 用法
```java

OkHttpClient client = new OkHttpClient.Builder()
    .addInterceptor(new LoggingInterceptor())
    .build();
    
class LoggingInterceptor implements Interceptor {
  @Override public Response intercept(Interceptor.Chain chain) throws IOException {
    Request request = chain.request();

    long t1 = System.nanoTime();
    logger.info(String.format("Sending request %s on %s%n%s",
        request.url(), chain.connection(), request.headers()));

    Response response = chain.proceed(request);

    long t2 = System.nanoTime();
    logger.info(String.format("Received response for %s in %.1fms%n%s",
        response.request().url(), (t2 - t1) / 1e6d, response.headers()));

    return response;
  }
}

```
两个重要的方法
 ```java
  Request request = chain.request();
  Response response = chain.proceed(request);
 ```
`OkHttpClient.Builder` 提供了两个 不同的**拦截器组** 
  + Interceptor（Application Interceptor）
  + NetworkInterceptor（Network Interceptor）
  
![okhttp-interceptor](https://ws1.sinaimg.cn/large/882b6a2aly1g14rd19tl1j20go0fa3zg.jpg)

  
![](https://ws1.sinaimg.cn/large/882b6a2aly1g14rc8lfoij218a0km7ie.jpg)

## Interceptor的始发站 ` getResponseWithInterceptorChain `
```java
  Response getResponseWithInterceptorChain() throws IOException {
    // Build a full stack of interceptors.
    List<Interceptor> interceptors = new ArrayList<>();
    // AppInterceptor
    interceptors.addAll(client.interceptors());
    interceptors.add(retryAndFollowUpInterceptor);
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    interceptors.add(new CacheInterceptor(client.internalCache()));
    interceptors.add(new ConnectInterceptor(client));
    if (!forWebSocket) {
     // NetWorkInterceptor
      interceptors.addAll(client.networkInterceptors());
    }
    interceptors.add(new CallServerInterceptor(forWebSocket));

    Interceptor.Chain chain = new RealInterceptorChain(interceptors, null, null, null, 0,
        originalRequest, this, eventListener, client.connectTimeoutMillis(),
        client.readTimeoutMillis(), client.writeTimeoutMillis());

    return chain.proceed(originalRequest);
  }
```
Interceptor 列表  `AppInterceptor` ,`RetryAndFollowUpInterceptor`,`BridgeInterceptor`,`CacheInterceptor`,`ConnectInterceptor`,`NetworkInterceptor`,`CallServerInterceptor`

## RealInterceptorChain 执行链式调用
```java
// chain is RealInterceptorChain
chain.proceed(originalRequest);
class RealInterceptorChain implements Interceptor.Chain{
}
```
继续往下走 
```java
Interceptor.Chain chain = new RealInterceptorChain(interceptors, null, null, null, 0,originalRequest, this, eventListener, client.connectTimeoutMillis(),
        client.readTimeoutMillis(), client.writeTimeoutMillis());
        
public Response proceed(Request request, StreamAllocation streamAllocation, HttpCodec httpCodec,
      RealConnection connection) throws IOException {
    if (index >= interceptors.size()) throw new AssertionError();
        calls++; // 后面 判断当前chain 只能调用一次
     // If we already have a stream, confirm that the incoming request will use it. 
     //默认情况 this.httpCodec=null
    if (this.httpCodec != null && !this.connection.supportsUrl(request.url())) {
      throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
          + " must retain the same host and port");
    }

    // If we already have a stream, confirm that this is the only call to chain.proceed(). 
    //默认情况 this.httpCodec=null
    if (this.httpCodec != null && calls > 1) {
      throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
          + " must call proceed() exactly once");
    }
    
    // Call the next interceptor in the chain. 
    //执行下一次的interceptor
    RealInterceptorChain next = new RealInterceptorChain(interceptors, streamAllocation, httpCodec,
        connection, index + 1, request, call, eventListener, connectTimeout, readTimeout,
        writeTimeout);
    Interceptor interceptor = interceptors.get(index);
    Response response = interceptor.intercept(next);
    ...
    return response;
}
```
源码可知  `RealInterceptorChain` 通过递归调用 `okhttp3.Interceptor.Chain#proceed(request)` 有的同学会说 上面源码中 只调用了一次这个方法,在 `getResponseWithInterceptorChain` 没错，表面上只掉了一次，那他们怎么实现递归的？？？ 
答案在`interceptor.intercept(next);` 我们在实现一个Interceptor接口时,返回值是Response,我们必须调用 `Response okhttp3.Interceptor.Chain#proceed(request)` 才能得到Response
```
public interface Interceptor {
  Response intercept(Chain chain) throws IOException;
}
```
    