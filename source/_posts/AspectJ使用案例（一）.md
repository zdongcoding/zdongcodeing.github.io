---
title: Aspectj与androidM的Permission简单结合
date: 2017-08-18 11:12:50
tags: android
---



前言
这里我们开始使用AspectJ来实现权限的申请和相关操作


> 上几篇文章已经讲过了 ` AspectJ `的语法和配置和Android M 中的权限的申请

大致思路：1.找到权限申请的切点 >>> 2.插入权限申请的判断

#### 1.切点
>一般来说我们申请权限写在一个方法内方便我们Hook,我们使用注解（Annotation）方式来实现，如果我们把一个具体的方法当着切点扩展性差而且使用也不方便。

我们定义一个Annotation来当着申请权限的切入点
```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface CheckPermission {
    String[] value();  //权限申请
    int requestcode() default 10000; //后面讲
}
```
#### 2.开始切入
如果你有AspectJ切入的经验或者看过 [Annotation的切入][url] 那看客可以继续往下看了

看客们能到这一步，大概应该都知道我们需要Hook到一个方法的是否可以让它执行到，我们应该用` Around `环绕模式来PointCut这个点

```java
@Pointcut("execution(@com.zdg.aspectj.checkpermission.annotition.CheckPermission !synthetic * *(..))")
public void findCheckPermission() {}
@Around("findCheckPermission()")
public  void checkPermission(ProceedingJoinPoint proceedingJoinPoint) {}
```
这样我们大概也就切入了需要申请权限这个点了，接下来我们就来处理 申请权限的具体操作

Android M的Permission我们还需要了解一些基本知识[Android 6.0以上的Permission](http://blog.zoudongq123.cn/2017/08/16/M-%E6%9D%83%E9%99%90%E7%94%B3%E8%AF%B7/)。

接下来我们具体实现 ` checkPermission(ProceedingJoinPoint proceedingJoinPoint)`,话不多说，就是干。
```java
@Around("findCheckPermission()")
public  void checkPermission(ProceedingJoinPoint proceedingJoinPoint) {
    Object target = proceedingJoinPoint.getTarget();//调用使用CheckPermission注解的类对象
    MethodSignature signature = (MethodSignature) proceedingJoinPoint.getSignature();
    CheckPermission annotation = signature.getMethod().getAnnotation(CheckPermission.class);//获取该方法注解中的需要申请的权限
    if(PermissionChecker.checkSelfPermission((Context)target, Arrays.asList(annotation.vaule()))==PermissionChecker.PERMISSION_GRANTED/*判断权限是否同意了*/){
        try {
            proceedingJoinPoint.proceed(proceedingJoinPoint.getArgs());
        } catch (Throwable throwable) {
                throwable.printStackTrace();
        }
    }
}
```
以上只是实现了一个最简单的判断逻辑。 然而权限的申请往往不是这样简单？？？？？😎😎😎😎😎😎，我们知道申请权限的过程是一个异步的，申请的过程中会弹出系统对话框，用户选择操作后会回调方法onRequestPermissionsResult，第一次申请是异步的回调onRequestPermissionsResult后判断同意继续执行同意后的操作。

问题就来了， 一个异步的过程我们怎么切入到一个方法中的？？？？想出了几个方案。

#### 问题总结： 
    问题一：权限申请是异步操作，@CheckPermission 回调onRequestPermissionsResult后怎么样回到设有@CheckPermission执行
    问题二：开发者权限申请过程中为了方便肯定不愿意自己实现onRequestPermissionsResult，自己实现了，代码量和自己写权限申请的代码量差不多了， 为嘛还是用你这个SDK????

方案一：继续切入onRequestPermissionsResult
+ 放弃理由：1.如果切入的当前类没有复写改方法，AspectJ无法切入改方法，2.怎么样让两个切入点，执行同一套方法（一个是同步的， 一个是异步的）3.接入代码量和使用原始方法申请没有多大的区别

方案二：在方案一的基础上改变套路，申请权限启动内部的一个activity 在这个activity中实现onRequestPermissionsResult，并且切入 onRequestPermissionsResult ，这样解决了减少代码量。

以上两个方案解决了《问题二》的问题。

至于《问题一》暂时想到的方案是在Aspect类保存请求权限时（通过过注解切入权限的时）保存` JoinPoint `这个对象，当` onRequestPermissionsResult `切入点被执行时 权限通过时 获取保存的` JoinPoint `来继续执行 该（@CheckPermission）切入点的方法。


大概就是这个思路：
```java
@Aspect
public class CheckPermissionRuntime {
     ProceedingJoinPoint mPermissionJoinPoint; 
    @Around("findCheckPermission()")
    public  void checkPermission(ProceedingJoinPoint proceedingJoinPoint) {
        this.mPermissionJoinPoint = proceedingJoinPoint;  //解决《问题一》
        if（/*需要申请权限*/）{
            startActivity()
        }else{
            try {
                 proceedingJoinPoint.proceed(proceedingJoinPoint.getArgs());
            } catch (Throwable throwable) {
                throwable.printStackTrace();
            }
        }
            
    }
    @Before("onRequestPermissionsResult()")
    public void onRequestPermissionsResult(JoinPoint joinPoint){
        Object[] args = joinPoint.getArgs();
        int requestcode = (int) args[0];
        String[] permissions= (String[]) args[1];
        int[] grantResults= (int[]) args[2];
        if（/*判断所有权限已经都同意了*/）{
            try {
                 mPermissionJoinPoint.proceed(mPermissionJoinPoint.getArgs());
            } catch (Throwable throwable) {
                throwable.printStackTrace();
            }
        }
    }
}
```
以上就是大致的实现方式。

还有一些细节：

    1.因为处理 ` CheckPermissionRuntime `是一个单例 而proceedingJoinPoint会持有activity的对象，所以用不到的时候最好置空，
    2.activity与fragment申请权限的不同方式。


   [url]:http://blog.zoudongq123.cn/2017/07/26/AspectJ(%E4%BA%94)%20%E9%AB%98%E7%BA%A7%E8%AF%AD%E6%B3%95-Annotation/       

