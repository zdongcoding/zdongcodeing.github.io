---
title: Dagger2的初试
date: 2017-09-21 14:40:57
copyright: true
tags: google
categories: android
---

####  一些概念：
   [依赖注入(Dependency Injection简称DI)](https://baike.baidu.com/item/%E6%8E%A7%E5%88%B6%E5%8F%8D%E8%BD%AC/1158025?fr=aladdin&fromid=5177233&fromtitle=%E4%BE%9D%E8%B5%96%E6%B3%A8%E5%85%A5)
>组件不做定位查询，只提供普通的Java方法让容器去决定依赖关系。容器全权负责的组件的装配，它会把符合依赖关系的对象通过JavaBean属性或者构造函数传递给需要的对象。通过JavaBean属性注射依赖关系的做法称为设值方法注入(Setter Injection)；将依赖关系作为构造函数参数传入的做法称为构造器注入（Constructor Injection）
<!-- more -->
<!-- ![举个例子](http://ow9n8vqns.bkt.clouddn.com/timg.jpeg?imageMogr2/thumbnail/!60p) -->

<div align=center><img src="https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1505997536102&di=048f8891b3d6eb9b649c5decc3d0c0c8&imgtype=0&src=http%3A%2F%2Fpic5.duowan.com%2Fwow%2F0902%2F99573894948%2F99574471488.jpg"  />
</div>

####  接入方式：

```
compile 'com.google.dagger:dagger:2.11'
annotationProcessor 'com.google.dagger:dagger-compiler:2.11'
```

####  Dagger2基本语法：
 + @Inject        
   - 用注解(Annotation)来标注目标类中所**依赖的其他类**，同样用注解来标注所**依赖的其他类的构造函数**
 + @Component     
   - 将被注解（@Inject）的**其他类**和**其他类的构造函数** 关联起来

 以上是Dagger2最基本的配套注解

举个例子：
```java
class A{
    @Inject
    public A{}
}
class B{
    @Inject
    A a;   //这里不能使用private
    public B{
        DaggerBComponent.create().inject(this);
    }
}
@Component
interface BComponent{
   inject(B b);
}
```

#####  问题：
1.DaggerBComponent是什么鬼？？

这个是根据@Component注解的Class ,Dagger2通过apt生成的类，类名(Dagger+被注解的类名)

2.依赖类构造函数不能被修改（第三方库中的类），@Inject不能注解被依赖类的构造函数了，那怎么弄？？

这个Dagger2已经考虑到了，这种情况我们就要使用另外两个注解来解决了   @Module   @Provides

####  Module的用途
> 上面的问题2我们只用@Inject是无法解决这个问题的，那我们怎么弄呢？？我们可以把这些第三方类，无法修改的构造函数的类封装起来，这就是Module
 + @Module   
   - 被注解的Module类,其实是一个简单工厂模式，Module里面的方法基本都是创建类实例的方法

@Component怎么样连接@Inject和@Module的呢？
```java
@Component(modules={ModuleClass.class})
interface BComponent{
   inject(B b);
}
```
` @Component `是注入器，它一端连接目标类，另一端连接目标类依赖实例，它把目标类依赖实例注入到目标类中。 ` Module(modules={ModuleClass.class}) `是一个提供类实例的类，所以Module应该是属于Component的实例端的（连接各种目标类依赖实例的端）。

那问题又来了， 怎么把Moudle中的各种创建类的实例函数与目标类中用` @Inject `注解标注的依赖产生关联关系呢？接下来我们就要用到` @Provides `
 + @Provides
   - 有了这个注解 Component 就可以在@Module类中搜索被@Provides注解的创建类的实例方法，这样就能解决第三方库使用Dagger的问题了
##### 注解迷失问题
>` @Provides ` 直接创建类实例方法返回类型是一个接口
解决方式官方已经提供：使用` @Named ` 或者` @Qulifier ` 
 + @Named    直接和` @Provides `配合使用  使用方式  @Named("name“)
 + @Qulifier  和@Named 的去区别是只是创建一个新的注解使用
```java
@Qualifier
@Retention(RetentionPolicy.RUNTIME)
public @interface AA {}
//然后使用方式
@Provides
@AA
//或者
@Provides
@Named("xxx")
```



遇到的问题：
1.当 ` @Inject `直接的依赖类型是一个接口
    编译无法通过:报错：xxx xcannot be provided without an @Provides- or @Produces-annotated method.

2.直接迷失问题` @Provides `的方法返回的类型是一个接口
 编译无法通过：报错 xxx  is bound multiple times:



使用过程的遇到的疑问：
1.构造方法带有基本参数，怎么弄？？
  下期预告
2.全局对象的传递，怎么弄？
  下期预告