---
title: AspectJ(五) 高级语法--Annotation
copyright: true
tags: AspectJ
date: 2017-07-26 14:36:00
categories: android
---

### Annotation
>这篇文章主讲 ` Annitation ` 作为切入点  

前几篇文章讲的都是通过具体方法路径作为切入点` pointcut `，这次我们通过Annotation来作为切入点` pointcut `

```java

@Retention(RetentionPolicy.CLASS)
@Target({ElementType.FIELD,ElementType.METHOD,ElementType.TYPE})
public @interface Runtime {
    String value() ;
}
```
注解相关知识这里就不多说了[Annotation入门](http://www.jianshu.com/p/fed18201cb12)
<!-- more -->
来个例子🌰简单说明
```java
@Pointcut("get(@com.github.zdongcoding.aspectjdemo.Runtime * *)")
void fieldAnnotition(){}
@Pointcut("execution(@com.github.zdongcoding.aspectjdemo.Runtime !synthetic * *(..))")
void methodAnnotition(){}
@Pointcut("within(@com.github.zdongcoding.aspectjdemo.Runtime *)")
void  withinclass(){}
```
上面三个切入点拥有` Runtime `注解的` Field ` ` Method ` ` class ` 

### 与普通的通过路径表达式区别在于前面多了一个注解类 
+ 注解切入点表达式 ： advice (@Annotation 访问访问 返回类型 切入点路径)
- 路径切入点表达式 ： advice（            访问访问 返回类型 切入点路径）  
   >访问访问 可以忽略

> 因为通过注解我们就不需要知道每个切入点的具体路径 
 
**区别: 前面多一个@Annotation**

[synthetic](http://blog.csdn.net/wonengxing/article/details/32711915)这个不是第一次见到了。

```java
 @After("methodAnnotition()&&withinclass()")
public void MethodAspect(JoinPoint joinPoint) throws Throwable {
        MethodSignature signature = (MethodSignature) joinPoint.getSignature();
        StringBuffer stringBuffer = new StringBuffer();
        stringBuffer.append("-----------------")
                .append("\n"+signature.getDeclaringType())
                .append("\n")
                .append(signature.getReturnType().getSimpleName())
                .append("  ")
                .append(signature.getName()).append("(");
        for (int i = 0; i < signature.getParameterTypes().length; i++) {
            stringBuffer.append(signature.getParameterTypes()[i].getName())
                    .append("  ")
                    .append(signature.getParameterNames()[i])
                    .append("=")
                    .append(joinPoint.getArgs()[i])
                    .append(",");
        }
        stringBuffer.deleteCharAt(stringBuffer.length() - 1)
                .append(")")
                .append("\n-----------------\n");
        Log.e("zoudong", stringBuffer.toString());
}
```
#### JoinPoint
```java
    Object getThis();   //当前切入点类

    Object getTarget(); //被切入的类（MainActivity对象）

    Object[] getArgs(); // 被切入的method ，Field-set参数

    Signature getSignature(); 

    SourceLocation getSourceLocation(); //被切入的行号位置

    String getKind();   //???
```
#### Signature
```java
public interface Signature {
    String getName();  //方法名，变量名

    int getModifiers();

    Class getDeclaringType(); //被切入的类 

    String getDeclaringTypeName();//被切入的类名
}
```
常用有几种子类
- FieldSignature
- MethodSignature
- ConstructorSignature
- TypeSignature

![](../assest/1787010-ad867955b97996e0.png)