---
title: AspectJ(三) 语法
copyright: true
tags: AspectJ
date: 2017-07-20 10:00:00
categories: android
---



### Advice
#### Before  与  After
> 这两个Advice最常用的， 顾名思义在PointCut之前或者之后插入代码

例子
```java
@Before("execution (* com.github.zdongcoding.aspectjdemo.MainActivity.onCreate(..))")
    public void adviceOnCreateBefore(JoinPoint joinPoint) {
        MethodSignature signature = (MethodSignature) joinPoint.getSignature();
        Toast.makeText((Context) joinPoint.getTarget(), "Before" + signature.toShortString(), Toast.LENGTH_SHORT).show();
        Log.e(TAG, "" + joinPoint.getSourceLocation() + "Before" + signature.toShortString());
}

@After("execution (* com.github.zdongcoding.aspectjdemo.MainActivity.onCreate(..))")
    public void adviceOnCreateAfter(JoinPoint joinPoint) {
        MethodSignature signature = (MethodSignature) joinPoint.getSignature();
        Toast.makeText((Context) joinPoint.getTarget(), "After" + signature.toShortString(), Toast.LENGTH_SHORT).show();
        Log.e(TAG, "" + joinPoint.getSourceLocation() + "After" + signature.toShortString());
}
```
  编译后 可以看到onCreate 方法，可以明显的看到` adviceOnCreateBefore `， ` adviceOnCreateAfter ` 的痕迹
```java
protected void onCreate(Bundle savedInstanceState) {
        JoinPoint var4 = Factory.makeJP(ajc$tjp_0, this, this, savedInstanceState);

        try {
            ActivityLifeAspect.aspectOf().adviceOnCreateBefore(var4);
            super.onCreate(savedInstanceState);
            this.setContentView(2130968603);
        } catch (Throwable var7) {
            ActivityLifeAspect.aspectOf().adviceOnCreateAfter(var4);
            throw var7;
        }

        ActivityLifeAspect.aspectOf().adviceOnCreateAfter(var4);
}
```
#### Around
>Before和After其实还是很好理解的，也就是在Pointcuts之前和之后，插入代码，那么Around呢，从字面含义上来讲，也就是在方法前后各插入代码，是的，他包含了Before和After的全部功能

```java
@Around("execution (* com.github.zdongcoding.aspectjdemo.MainActivity.onDdd())")
public void adviceOnDDAround(ProceedingJoinPoint joinPoint) throws Throwable {
    Log.e("zoudong", "adviceOnDDAround====" + "joinPoint1 = [" + joinPoint.getSignature() + "]");
    joinPoint.proceed();
    Log.e("zoudong", "adviceOnDDAround====" + "joinPoint2 = [" + joinPoint.getSignature() + "]");
}
MainActivity.class onDdd()
protected  void onDdd(){
    Log.e("zoudong", "onDdd====" + "");
}
```
其中，joinPoint.proceed()代表执行原始的方法，在这之前、之后，都可以进行各种逻辑处理。也就是说只有调用了 joinPoint.proceed() 原始方法才会被调用 。我们可以通过 ` Around ` 来移花接木，偷工减料。

**我们可以发现，Around确实实现了Before和After的功能，但是要注意的是，Around和After是不能同时作用在同一个方法上的，会产生重复切入的问题**

#### AfterThrowing  异常处理
>AfterThrowing是一个比较少见的Advice，他用于处理程序中未处理的异常，记住，**这点很重要，是未处理的异常**

话不多说，反手一套代码
```java
 private void throwNullPoint(){
        String s=null;
        s.equals("null");
}
```
大家都知道上面的代码，调用之后肯定报错（NullPointerException）

```java
@AfterThrowing(pointcut = "execution(* com.github.zdongcoding.aspectjdemo.MainActivity.throwNullPoint(..))",throwing="exception")
public void  throwNullPoint(Exception exception){
        Log.e("zoudong", "throwNullPoint====" + " exception = [" + exception.toString() + "]");
}

private void throwNullPoint() {
        try {
            String s = null;
            ((String)s).equals("null");
        } catch (Exception var3) {
            ActivityLifeAspect.aspectOf().throwNullPoint(var3);
            throw var3;
        }
}
```
由以上可知，被插入了我们切入的代码，但是最后，他依然会throw var3，也就是说，这个异常已经会被抛出去，崩溃依旧是会发生的。
#### AfterReturning
```java
MainActivity.java
private String AfterReturning(){
        return "asglsas";
}
ActivityLifeAspentj.java
@AfterReturning(pointcut = "execution(* com.github.zdongcoding.aspectjdemo.MainActivity.AfterReturning(..))",returning="name")
    public void   AfterReturning(String name){
        Log.e("zoudong", "AfterReturning====" + "joinPoint = [" + name + "]");
}
```
注意 returing="name" 必须是最后一个参数
如果 将returining="name1"  报错--> **the last parameter of this advice must be named 'name1' to bind the returning value**
  

|关键词  | 说明   | 示例 |
|-----|:------:|:------:|
|before() | before advice |表示在JPoint执行之前，需要干的事情
|after()  | after advice  |表示JPoint自己执行完了后，需要干的事情。
|after():returning(返回值类型)<br>after():throwing(异常类型) | returning和throwing后面都可以指定具体的类型，如果不指定的话则匹配的时候不限定类型  |  假设JPoint是一个函数调用的话，那么函数调用执行完有两种方式退出，一个是正常的return，另外一个是抛异常。注意，after()默认包括returning和throwing两种情况
| 返回值类型 around()   | before和around是指JPoint执行前或执行后备触发，而around就替代了原JPoint   |around是替代了原JPoint，如果要执行原JPoint的话，需要调用proceed
## JoinPoint
>在AspectJ的切入点表达式中，我们前面都是使用的execution，实际上，还有一种类型——call，那么这两种语法有什么区别呢
####  execution ，call
   + execution(执行) 切入点是` method内部 `
   + call(调用)   切入点是` 调用该method的地方 `
   + withincode  通常用于切入点条件过滤

废话不多说，开始搞
```java
MainActivity.java
protected  void onDdd(){
        test01();
        Log.e("zoudong", "onDdd====" + "");
}
private  void   test(){
        test01();
        Log.e("zoudong", "test====" + "");
}
private  void test01(){
        Log.e("zoudong", "test01====" + "");
}
```

如上所写，一共有两个地方调用了 test01() ,但是需求 我只需要在onDdd()方法中插入代码

```java
public static final String invokeonDdd="withincode(com.github.zdongcoding.aspectjdemo.MainActivity.onDdd(..))";

public static final String test01="call(com.github.zdongcoding.aspectjdemo.MainActivity.test01(..))";
@Pointcut(invokeonDdd)
public void invokeOnDdd(){}

@Pointcut(test01)
public void invoketest01(){}

@Pointcut("invokeOnDdd()&&invoketest01()")
public void invoketest01OnlyOnDdd(){}

@Before("invoketest01OnlyOnDdd()")   //方式一
public void  invoketest01OnlyOnDddBefore(JoinPoint joinPoint){
    Log.e("zoudong", "invoketest01OnlyOnDddBefore====" + "joinPoint = [" joinPoint.getSourceLocation() + "]");
}

@After("invoketest01()&&invokeOnDdd()") //方式二
public void  invoketest01OnlyOnDddAfter(JoinPoint joinPoint){
    Log.e("zoudong", "invoketest01OnlyOnDddAfter====" + "joinPoint = [" joinPoint.getSourceLocation() + "]");
}
```
编译之后

```java
protected void onDdd() {
        JoinPoint var4 = Factory.makeJP(ajc$tjp_2, this, this);
        ActivityLifeAspect var10000 = ActivityLifeAspect.aspectOf();
        Object[] var5 = new Object[]{this, var4};
        var10000.adviceOnDDAround((new MainActivity$AjcClosure1(var5)).linkClosureAndJoinPoint(69648));
}
private void test() {
        this.test01();
        Log.e("zoudong", "test====");
}

private void test01() {
        Log.e("zoudong", "test01====");
}
```
可以看到只有onDdd()方法中调用了test01()才插入代码  test()调用了，但并没有插入代码


Test Github:<https://github.com/zdongcoding/AspectjDemo>