---
title: Retrofit源码分析
copyright: true
categories: android
date: 2018-09-11 09:25:10
tags: 源码解析
---


# Retrofit
>  A type-safe HTTP client for Android and Java


开发Android App肯定会使用Http请求与服务器通信，上传或下载数据等。目前开源的Http请求工具也有很多，比如Google开发的`Volley`，loopj的`Android Async Http`，Square开源的`OkHttp`或者`Retrofit`等。

我觉得Retrofit 无疑是这几个当中最好用的一个，设计这个库的思路很特别而且巧妙。Retrofit的代码很少，花点时间读它的源码肯定会收获很多。

本文的源码分析基于Retrofit 2，和Retrofit 1.0的Api有较大的不同， 本文主要分为几部分：`Retrofit 是什么`，`Retrofit怎么用`，`Retrofit的原理是什么`


## Retorift 使用

### 官方使用简介
> 官方使用简介:(http://square.github.io/retrofit/)

### 引用依赖

```
// 最新版本 ：2.5.0
implementation 'com.squareup.retrofit2:retrofit:(insert latest version)'
```

### 生成一个ApiService

```java
public interface GitHubService {
  @GET("users/{user}/repos")
  Call<List<Repo>> listRepos(@Path("user") String user);
}
```

### 创建Retorfit 
```java
Retrofit retrofit = new Retrofit.Builder()
    .baseUrl("https://api.github.com/")
    .build();

GitHubService service = retrofit.create(GitHubService.class);

```

###  发起请求
```java
Call<List<Repo>> repos = service.listRepos("octocat");

//异步请求
// 请求数据，并且处理response
call.enqueue(new Callback<List<Repo>>() {
    @Override
    public void onResponse(Response<List<Repo>> response) {
        System.out.println(response);
    }
    @Override
    public void onFailure(Throwable t) {
    }
});
//同步请求
call.execute()
```
### ApiService 配置

#### Url配置

| 注解 | 描述 | 实例 |
| --- | --- | --- |
| @GET | get请求 | @GET("group/users") |
| @POST | post请求 | @POST("group/users") |
| @Uri | Uri全连接配合 其他请求方法 | @Uri("http://xxxx/group/users") |
| @PATH | get/post 请求的动态路径，用于方法参数注解 | @GET("group/{id}/users"    Call<List<User>> groupList(@Path("id") int groupId); |
| @Headers | 请求的 HEADER 作用于方法 | @Headers("Cache-Control: max-age=640000") |
| @Header | 请求的 HEADER 作用于方法参数 | @Header("Authorization") String authorization |
| @Query & @QueryMap | 用于方法参数注解,只能配合@GET | Call<List<User>> groupList(@Path("id") int groupId,@Query("sort") String sort); |
| @Body | 用于方法参数注解 | @POST("users/new")Call<User> createUser(@Body User user);  |
| @Field & @FieldMap | 用于方法参数注解,只能配合@POST | 类似@Query & @QueryMap |
| @Part & @PartMap |  用于POST文件上传  |Call<User> updateUser(@Part("photo") RequestBody photo, @Part("description") RequestBody description);|



### 其他高级用法
  
  官方使用简介:(http://square.github.io/retrofit/)

## Retrofit 源码分析
> retrofit的最大特点就是解耦，要解耦就需要大量的设计模式，假如一点设计模式都不懂的人，可能很难看懂retrofit。

### 基本流程(组装过程）

#### Retrofit#create

```java
ApiService service = retrofit.create(ApiService.class);
```
Retrofit是如何创建Api的，是通过`Retrofit#create()`的方法 ,通过[动态代理](https://www.cnblogs.com/gonjan-blog/p/6685611.html)方式。

关于动态代理 这里就不多讲了，楼下有动态代理链接，至于为什么用动态代理，大家想一想？？？

```java
public <T> T create(final Class<T> service) {
    Utils.validateServiceInterface(service);
    if (validateEagerly) {
      eagerlyValidateMethods(service);
    }
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
          private final Platform platform = Platform.get();

          @Override public Object invoke(Object proxy, Method method, Object... args)
              throws Throwable {
            // If the method is a method from Object then defer to normal invocation.
            if (method.getDeclaringClass() == Object.class) {
              return method.invoke(this, args);
            }
           //DefaultMethod 是 Java 8 的概念，是定义在 interface 当中的有实现的方法
            if (platform.isDefaultMethod(method)) {
              return platform.invokeDefaultMethod(method, service, proxy, args);
            }
             //每一个接口最终实例化成一个 ServiceMethod，并且会缓存
             
            ServiceMethod<Object, Object> serviceMethod =
            (ServiceMethod<Object, Object>) loadServiceMethod(method);
            //Retrofit 与 OkHttp 完全耦合
            OkHttpCall okHttpCall = new OkHttpCall<>(serviceMethod, args);
          //发起请求，并解析服务端返回的结果
            return serviceMethod.adapt(okHttpCall);
          }
        });
  }
```
 提取在create方法中的关键几行代码 `Service Proxy`

当大家了解了动态代理之后， 都知道我们调用ApiService中的`method`都会自动调用 `InvocationHandler()`.

真真开始执行的是 `serviceMethod.adapt(okHttpCall)==> calladapter#adapt(okhttpCall);`

 ```java
 ServiceMethod<Object, Object> serviceMethod =
             (ServiceMethod<Object, Object>) loadServiceMethod(method);
 OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
return serviceMethod.adapt(okHttpCall);
 ```

一个 ServiceMethod 对象对应于一个 API interface 的一个方法，loadServiceMethod(method) 方法负责加载 ServiceMethod：

```java
ServiceMethod<?, ?> loadServiceMethod(Method method) {
    ServiceMethod<?, ?> result = serviceMethodCache.get(method);
    if (result != null) return result;

    synchronized (serviceMethodCache) {
      result = serviceMethodCache.get(method);
      if (result == null) {
        result = new ServiceMethod.Builder<>(this, method).build();
        serviceMethodCache.put(method, result);
      }
    }
    return result;
}
```

这里实现了缓存逻辑，同一个 API 的同一个方法，只会创建一次。这里由于我们每次获取 API 实例都是传入的 class 对象，而 class 对象是进程内单例的，所以获取到它的同一个方法 Method 实例也是单例的，所以这里的缓存是有效的。

`ServiceMethod`是什么呢？？？里面包含什么 ???
大致就是想这个方法的method转化成 `ServiceMethod对象` 包含 各个注解（方法的注解，参数的注解，方法的返回值的东西）,那我们来看下具体有什么东西

我们看看 `ServiceMethod` 的构造函数：

```java
ServiceMethod(Builder<R, T> builder) {
    this.callFactory = builder.retrofit.callFactory();
    this.callAdapter = builder.callAdapter;
    this.baseUrl = builder.retrofit.baseUrl();
    this.responseConverter = builder.responseConverter;
    this.httpMethod = builder.httpMethod;
    this.relativeUrl = builder.relativeUrl;
    this.headers = builder.headers;
    this.contentType = builder.contentType;
    this.hasBody = builder.hasBody;
    this.isFormEncoded = builder.isFormEncoded;
    this.isMultipart = builder.isMultipart;
    this.parameterHandlers = builder.parameterHandlers;
}
```

这里用了buider 模式，这里很多参数，有的参数一看就明白 像baseUrl，headers ，contyentType等，讲解主要关注四个成员：callFactory，callAdapter，responseConverter 和 parameterHandlers。

+ callFactory 负责创建 HTTP 请求，HTTP 请求被抽象为了 okhttp3.Call 类，它表示一个已经准备好，可以随时执行的 HTTP 请求；
+ callAdapter 把 retrofit2.Call<T> 转为 T（注意和 okhttp3.Call 区分开来，retrofit2.Call<T> 表示的是对一个 Retrofit 方法的调用），这个过程会发送一个 HTTP 请求，拿到服务器返回的数据（通过 okhttp3.Call 实现），并把数据转换为声明的 T 类型对象（通过 Converter<F, T> 实现）；
+ responseConverter 是 Converter<ResponseBody, T> 类型，负责把服务器返回的数据（JSON、XML、二进制或者其他格式，由 ResponseBody 封装）转化为 T 类型的对象；
+ parameterHandlers 则负责解析 API 定义时每个方法的参数，并在构造 HTTP 请求时设置参数；

`retrofit2.ServiceMethod.Builder#build()`方法中生成了以上对象

```java
 public ServiceMethod build() {
    // 通过method 找到对应的retrofit2.CallAdapter
    callAdapter = createCallAdapter();
     //省略代码...
    // 通过method 找到对应的retrofit2.Converter
    responseConverter = createResponseConverter();
     //省略代码...
    for (Annotation annotation : methodAnnotations) {
      //解析 请求的方法
        parseMethodAnnotation(annotation);
    }
     int parameterCount = parameterAnnotationsArray.length;
      parameterHandlers = new ParameterHandler<?>[parameterCount];
      for (int p = 0; p < parameterCount; p++) {
        Type parameterType = parameterTypes[p];
         //省略代码...
        Annotation[] parameterAnnotations = parameterAnnotationsArray[p];
        ...
        //解析方法参数注解
        parameterHandlers[p] = parseParameter(p, parameterType, parameterAnnotations);
      }
    //省略代码...
    return new ServiceMethod<>(this);
 }
``` 

### CallAdapter
> retrofit2.Retrofit.Builder#addCallAdapterFactory(Factory)

默认 retrofit2.DefaultCallAdapterFactory 使用的是OkhttpCall
+ `retrofit2.ServiceMethod.Builder#createCallAdapter(...)`
    +  `retrofit2.Retrofit#callAdapter(...)`
    +  `retrofit2.Retrofit#nextCallAdapter(null,...)`

```java
 public CallAdapter<?, ?> nextCallAdapter(@Nullable CallAdapter.Factory skipPast, Type returnType,
      Annotation[] annotations) {
    checkNotNull(returnType, "returnType == null");
    checkNotNull(annotations, "annotations == null");

    int start = callAdapterFactories.indexOf(skipPast) + 1;
    for (int i = start, count = callAdapterFactories.size(); i < count; i++) {
      // 调用retrofit2.CallAdapter.Factory#get
      CallAdapter<?, ?> adapter = callAdapterFactories.get(i).get(returnType, annotations, this);
      if (adapter != null) {
        return adapter;
      }
    }
    //省略代码
    //throw new IllegalArgumentException(builder.toString());
  }
```
上面知道如果注册了多个`retrofit2.CallAdapter`,按照注册的顺序来执行，如果那个有返回值就会直接返回，不走for循环了


### Converter
> 通过Retrofit 添加ConverterFactory 
`Retrofit.Builder#addConverterFactory(Factory)`

 默认`retrofit2.BuiltInConverters`


#### ParameterHandlers or RequestConverter
 + `retrofit2.ServiceMethod.Builder#parseParameter(...)`
 + `retrofit2.ServiceMethod.Builder#parseParameterAnnotation(...)`
    +  `retrofit2.Retrofit#stringConverter(...)`
        + `retrofit2.Converter.Factory#stringConverter(...)`
        + OR `retrofit2.Converter.Factory#requestBodyConverter(...)`

```java
private ParameterHandler<?> parseParameter(
        int p, Type parameterType, Annotation[] annotations) {
      ParameterHandler<?> result = null;
      for (Annotation annotation : annotations) {
        //关键代码
        ParameterHandler<?> annotationAction = parseParameterAnnotation(
            p, parameterType, annotations, annotation);

        if (annotationAction == null) {
          continue;
        }
        //省略代码  ...

        result = annotationAction;
      }
         //省略代码 ...
      return result;
}
private ParameterHandler<?> parseParameterAnnotation(int p, Type type, Annotation[] annotations, Annotation annotation) {
  // 关键代码
  Converter<?, String> converter =
              retrofit.stringConverter(iterableType, annotations);

  // or
  Converter<?, RequestBody> converter =
                retrofit.requestBodyConverter(type, annotations, methodAnnotations);
    return  new ParameterHandler;
}

```

【总结】: 当执行`retrofit2.Retrofit#create(final Class<T> service)`的时候就会找到对应的 `retrofit2.Converter`

#### responseConverter

 + `retrofit2.ServiceMethod.Builder#createResponseConverter(...)`
    +  `retrofit2.Retrofit#responseBodyConverter(...)`
    +  `retrofit2.Retrofit#nextResponseBodyConverter(null,...)`

```java
public <T> Converter<ResponseBody, T> nextResponseBodyConverter(
      @Nullable Converter.Factory skipPast, Type type, Annotation[] annotations) {
    checkNotNull(type, "type == null");
    checkNotNull(annotations, "annotations == null");

    int start = converterFactories.indexOf(skipPast) + 1;
    for (int i = start, count = converterFactories.size(); i < count; i++) {
      // 调用 retrofit2.Converter.Factory#responseBodyConverter
      Converter<ResponseBody, ?> converter =
          converterFactories.get(i).responseBodyConverter(type, annotations, this);
      if (converter != null) {
        //noinspection unchecked
        return (Converter<ResponseBody, T>) converter;
      }
    }
    ...
    //省略代码
    // throw new IllegalArgumentException(...)
}
```

上面知道如果注册了多个Converter 按照注册的顺序来执行，如果那个有返回值就会直接返回，不走for循环了

【总结】: 当执行`retrofit2.Retrofit#create(final Class<T> service)`的时候就会找到对应的 `retrofit2.Converter`

**以上都是初始化操作，将所有的请求相关的东西都封装到`ServiceMethod`中**

### 执行过程

```java
serviceMethod.adapt(okHttpCall);
```
真正执行的开始点就是`serviceMethod.adapt` ,**ServiceMethod.class**执行过程中的 三个很重要，也是整个过程关键节点

```java
  // 简单粗俗的讲：返回包装okhttpCall后的对象
  T adapt(Call<R> call) {
    return callAdapter.adapt(call);
  }
  //retrofit2.OkHttpCall#createRawCall 调用
  okhttp3.Call toCall(@Nullable Object... args) throws IOException {
    // 返回 okhttp.Call
  }
  //retrofit2.OkHttpCall 请求结果之后 会调用 responseBody  convert
  R toResponse(ResponseBody body) throws IOException {
    return responseConverter.convert(body);
  }
```

**通过`serviceMethod`上面三个方法,将 `OkhttpCall`,`CallAdapter` ,`Converter` 关联起来**

![](https://ws1.sinaimg.cn/large/882b6a2aly1g0hirf70mqj20or0nen2d.jpg)



为什么要用动态代理？

因为对接口的所有方法的调用都会集中转发到 InvocationHandler#invoke函数中，我们可以集中进行处理，更方便了。你可能会想，我也可以手写这样的代理类，把所有接口的调用都转发到 InvocationHandler#invoke，当然可以，但是可靠地自动生成岂不更方便,可以方便为什么不方便。


[Retrofit分析-漂亮的解耦套路](https://www.jianshu.com/p/45cb536be2f4)

[Java动态代理](http://a.codekk.com/detail/Android/Caij/%E5%85%AC%E5%85%B1%E6%8A%80%E6%9C%AF%E7%82%B9%E4%B9%8B%20Java%20%E5%8A%A8%E6%80%81%E4%BB%A3%E7%90%86)