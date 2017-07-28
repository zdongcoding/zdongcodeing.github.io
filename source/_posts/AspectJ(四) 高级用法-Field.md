---
title: AspectJ(四) 高级用法--Field
copyright: true
tags: AspectJ
---


# AspectJ(四) 高级用法--Field

## Field  <set/get>
> 变量的获取和设置,**不能是局部变量**

### get Field 的值
+ 想获取一个变量Field的值
```java
    MainActivity.java
    --> private  int data=1;
```
我们怎么获取这个 ` data ` 的值？？
```java
@Pointcut("get(int com.github.zdongcoding.aspectjdemo.MainActivity.data)")
public void fieldget() {}
@Around("fieldget()")
public int fieldget(ProceedingJoinPoint thisJoinPoint) throws Throwable {
    Object proceed = thisJoinPoint.proceed();
    System.out.println(thisJoinPoint.toLongString());
    System.out.println("  " + thisJoinPoint.getSignature().getName());
    System.out.println("  " +"--->"+proceed);
  return (int) proceed; //必须return 否则报错（applying to join point that doesn't return void: field-get）
}
@AfterReturning(pointcut = "fieldget()",returning = "data")
public void  fieldget2(JoinPoint joinPoint,int base){
        Log.e("zoudong", "fieldget2====" + "joinPoint = [" + joinPoint + "], data = [" + data + "]");
        // 不能使用return
}
```
以上两种方法都可以获取一个变量的值 
```java
@After(value = "fieldget()")
public void fieldget3(JoinPoint joinPoint) {
     Log.e("zoudong", "fieldget3====" + "joinPoint = [" + joinPoint + "]");
}
@Before(value = "fieldget()")
public void fieldget4(JoinPoint joinPoint) {
     Log.e("zoudong", "fieldget4====" + "joinPoint = [" + joinPoint + "]");
}
```
通过能收到调用但是无法获取变量值

### set Field 
> 两种情况 1.声明时  2.赋值时
```java
@Pointcut("set(int com.github.zdongcoding.aspectjdemo.MainActivity.base)")
public void fieldset() {}
@Around("fieldset()")
public void  fieldset(ProceedingJoinPoint joinPoint) throws Throwable {
        Log.e(TAG, "around->" + joinPoint.getTarget().toString() + "#" + joinPoint.getArgs()[0]);
        joinPoint.proceed(new Integer[]{100});
}
```
+ 声明时  (沿用上面的例子🌰)
  当MainActivity 对面创建时初始化变量调用一次

```java
public MainActivity() {
    byte var1 = 1;
    JoinPoint var3 = Factory.makeJP(ajc$tjp_0, this, this, Conversions.intObject(var1));
    ActivityLifeAspect var10000 = ActivityLifeAspect.aspectOf();
    Object[] var4 = new Object[]{this, this, Conversions.intObject(var1), var3};
    var10000.fieldset((new MainActivity$AjcClosure1(var4)).linkClosureAndJoinPoint(4112));
}
```

+ 赋值时
  随便一个地方修改data值 （我们再onCreate 方法中修改data）` date=10000; `
  编译之后onCreate 方法
```java
 protected void onCreate(Bundle savedInstanceState) {
       ....
       this.setContentView(2131361819);
       int var4 = 10000000;
       JoinPoint var6 = Factory.makeJP(ajc$tjp_1, this, this, Conversions.intObject(var4)) ;
       ActivityLifeAspect var10000 = ActivityLifeAspect.aspectOf();
       Object[] var7 = new Object[]{this, this, Conversions.intObject(var4), var6};
       var10000.fieldset((new MainActivity$AjcClosure3(var7)).linkClosureAndJoinPoint (4112));
       Log.e("zoudong", "data=" + this.data);
       ....
 }
```
打印的结果是 100 因为上面我们已经不管设置多少  返回的都是100，

- get   是取值  所以 只有通过 retrun 才能取到值  无法通过 arg  获取值
- set   是赋值  所以 可以通过 arg 取值   （advice（before after around）都可以）
        并且（默认情况下不拦截）赋值成功， 如果您需要改变赋值的值（拦截）则需要使用around 所以只有它能return

**结论 ： set/get  不管是哪里取值还是赋值都会被拦截**

遇到的问题：
- ApectJ类 中的方法 如果想return的话必须使用` around->joinPoint.proceed` ,否则报错 （Found @AspectJ annotation on a non around advice not returning）
- 使用 Around 报错 incompatible return type applying to field-set   ` set ` 不能使用 return 只能使用 joinPoint.proceed 
- 使用get 使用 around  必须return   否则报错 （applying to join point that doesn't return void: field-get）


### 总结
* @Around  在get/set 中  set不能使用return  但是get 必须使用return
* set 中 所有的advice  都不能使用 return
* advice 除了@Around 其他的都不能return

##### 新的理解
 - 使用` advice(Around) `时切入点是一个方法 ` Method ` 如果有返回值 那么 一定需要 ` return ` 否则报错 `  applying to join point that doesn't return void `