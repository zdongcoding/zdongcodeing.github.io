---
title: AspectJ(一) 一些该了解的概念
date: 2017-07-17
copyright: true
tags: AspectJ
categories: android
---

# AspectJ 一些该了解的概念

> AspectJ就是AOP，只不过是面向java的。AOP里面有一些重要基本的概念

### 什么是AOP

AOP是Aspect Oriented Programming的缩写，即『面向切面编程』。它和我们平时接触到的OOP都是编程的不同思想，OOP，即『面向对象编程』，它提倡的是将功能模块化，对象化，而AOP的思想，则不太一样，它提倡的是针对同一类问题的统一处理，当然，我们在实际编程过程中，不可能单纯的安装AOP或者OOP的思想来编程，很多时候，可能会混合多种编程思想，大家也不必要纠结该使用哪种思想，取百家之长，才是正道。
那么AOP这种编程思想有什么用呢，一般来说，主要用于不想侵入原有代码的场景中，例如SDK需要无侵入的在宿主中插入一些代码，做日志埋点、性能监控、动态权限控制、甚至是代码调试等等。

### Aspect  (切面) 
 >实现了cross-cutting功能，是针对切面的模块。最常见的是logging模块、方法执行耗时模块，这样，程序按功能被分为好几层，如果按传统的继承的话，商业模型继承日志模块的话需要插入修改的地方太多，而通过创建一个切面就可以使用AOP来实现相同的功能了，我们可以针对不同的需求做出不同的切面。

 ### jointpoint（连接点)
 >连接点是切面插入应用程序的地方，该点能被方法调用，而且也会被抛出意外。连接点是应用程序提供给切面插入的地方，在插入地建立AspectJ程序与源程序的连接。例如，构造方法调用、调用方法、方法执行、异常等等，这些都是Join Points，实际上，也就是你想把新的代码插在程序的哪个地方，是插在构造方法中，还是插在某个方法调用前，或者是插在某个方法中，这个地方就是Join Points
 
 ![jointpoint](http://img.blog.csdn.net/20160523093513387)


 ### pointcut（切点）
 >pointcut可以控制你把哪些advice应用于jointpoint上去，通常你使用pointcuts通过正则表达式来把明显的名字和模式进行匹配应用。决定了那个jointpoint会获得通知。分为call、execution、target、this、within等关键字。与joinPoint相比，pointcut就是一个具体的切点（需要插入代码的地方）


 ### <span id="advice"> advice（处理逻辑）>
 >advice是我们切面功能的实现，它是切点的真正执行的地方。比如像写日志到一个文件中，advice（包括：before、after、around等）在jointpoint处插入代码到应用程序中。我们来看一看原AspectJ程序和反编译过后的程序。看完下面的图我们就大概明白了AspectJ是如何达到监控源程序的信息了。


### 举个例子
 
 ```java
@Before("execution(* android.app.Activity.on**(..))")
    public void onActivityMethodBefore(JoinPoint joinPoint) throws Throwable {
}
 ```
  + @Before  这是一个[advice](#advice)
  + execution  这是一个Join Point
  + (* android.app.Activity.on\*\*(..)" 这是一个正则表达式 
     -  第一个**\***表示返回值(任意类型) - 方法的路径(通过正则匹配) - ()表示方法的参数，可以指定类型
  +  onActivityMethodBefore 表示切入点的方法


# Method 参数规则

|  表达式 |   含义  |
|----|:-------|
|java.lang.String	| 匹配String类型
|java.*.String	    |匹配java包下的任何“一级子包”下的String类型，如匹配java.lang.String，但不匹配java.lang.ss.String
|   java..*	        |匹配java包及任何子包下的任何类型,如匹配java.lang.String、java.lang.annotation.Annotation
|java.lang.*ing	    |匹配任何java.lang包下的以ing结尾的类型
|java.lang.Number+	|匹配java.lang包下的任何Number的自类型，如匹配java.lang.Integer，也匹配java.math.BigInteger

|   参数	| 含义  |
|----|:-------|
|()	|表示方法没有任何参数
|(..)	|表示匹配接受任意个参数的方法
|(..,java.lang.String)	|表示匹配接受java.lang.String类型的参数结束，且其前边可以接受有任意个参数的方法
|(java.lang.String,..)	|表示匹配接受java.lang.String类型的参数开始，且其后边可以接受任意个参数的方法
|(*,java.lang.String)	|表示匹配接受java.lang.String类型的参数结束，且其前边接受有一个任意类型参数的方法





 [参考](http://blog.csdn.net/eclipsexys/article/details/54425414)