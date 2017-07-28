---
title: AspectJ(二) 入门
copyright: true
tags: AspectJ
---

# AspectJ 入门

## android studio  配置AspectJ 环境
  
### 1.新建model(Aspectj)
    在model(aspectj)/build.gradle文件中添加
    如下代码
    
```java
 import org.aspectj.bridge.IMessage
 import org.aspectj.bridge.MessageHandler
 import org.aspectj.tools.ajc.Main
 apply plugin: 'com.android.library'
 
 buildscript {
 
     repositories {
         jcenter()
     }
     dependencies {
         classpath 'com.android.tools.build:gradle:2.3.3'
         classpath 'org.aspectj:aspectjtools:1.8.9'
         classpath 'org.aspectj:aspectjweaver:1.8.9'
     }
 }
 
 
 
 android {
     compileSdkVersion 26
     buildToolsVersion "25.0.3"
     lintOptions {
         abortOnError false
     }
 }
 
 dependencies {
     compile 'org.aspectj:aspectjrt:1.8.9'
 }
 
 android.libraryVariants.all { variant ->
 //    LibraryPlugin plugin = project.plugins.getPlugin(LibraryPlugin)
     JavaCompile javaCompile = variant.javaCompile
     javaCompile.doLast {
         String[] args = ["-showWeaveInfo",
                          "-1.5",
                          "-inpath", javaCompile.destinationDir.toString(),
                          "-aspectpath", javaCompile.classpath.asPath,
                          "-d", javaCompile.destinationDir.toString(),
                          "-classpath", javaCompile.classpath.asPath,
                          "-bootclasspath", project.android.bootClasspath.join(
                 File.pathSeparator)]
         MessageHandler handler = new MessageHandler(true);
         new Main().run(args, handler)
 
         def log = project.logger
         for (IMessage message : handler.getMessages(null, true)) {
             switch (message.getKind()) {
                 case IMessage.ABORT:
                 case IMessage.ERROR:
                 case IMessage.FAIL:
                     log.error message.message, message.thrown
                     break;
                 case IMessage.WARNING:
                 case IMessage.INFO:
                     log.info message.message, message.thrown
                     break;
                 case IMessage.DEBUG:
                     log.debug message.message, message.thrown
                     break;
             }
         }
     }
 }
```
> 网上配置方法会报一个错误  No such property: project for class: com.android.build.gradle.LibraryPlugin
   修复办法：不使用 LibraryPlugin  直接使用 project
### 2.App(主工程)修改 build.gradle

>  dependencies{ compile project("aspectl")} //第一步新建的model

```java
import org.aspectj.bridge.IMessage
import org.aspectj.bridge.MessageHandler
import org.aspectj.tools.ajc.Main

buildscript {
    repositories {
        mavenCentral()
    }
    dependencies {
        classpath 'org.aspectj:aspectjtools:1.8.9'
    }
}

final def log = project.logger
final def variants = project.android.applicationVariants

variants.all { variant ->
    if (!variant.buildType.isDebuggable()) {
        log.debug("Skipping non-debuggable build type '${variant.buildType.name}'.")
        return;
    }

    JavaCompile javaCompile = variant.javaCompile
    javaCompile.doLast {
        String[] args = ["-showWeaveInfo",
                         "-1.5",
                         "-inpath", javaCompile.destinationDir.toString(),
                         "-aspectpath", javaCompile.classpath.asPath,
                         "-d", javaCompile.destinationDir.toString(),
                         "-classpath", javaCompile.classpath.asPath,
                         "-bootclasspath", project.android.bootClasspath.join(File.pathSeparator)]
        log.debug "ajc args: " + Arrays.toString(args)

        MessageHandler handler = new MessageHandler(true);
        new Main().run(args, handler);
        for (IMessage message : handler.getMessages(null, true)) {
            switch (message.getKind()) {
                case IMessage.ABORT:
                case IMessage.ERROR:
                case IMessage.FAIL:
                    log.error message.message, message.thrown
                    break;
                case IMessage.WARNING:
                    log.warn message.message, message.thrown
                    break;
                case IMessage.INFO:
                    log.info message.message, message.thrown
                    break;
                case IMessage.DEBUG:
                    log.debug message.message, message.thrown
                    break;
            }
        }
    }
}
```
### Demo
> 到了这一步，您就可以开始搞事情了， 

  在model(aspectj)(main/java/xxx)中新建class    ->  ActivityLifeAspect

```java
@Aspect
public class ActivityLifeAspect {

    private static final String TAG = ActivityLifeAspect.class.getSimpleName();

    @Before("execution (* com.github.zdongcoding.aspectjdemo.MainActivity.onCreate(..))")
    public void adviceOnCreate(JoinPoint joinPoint) {
        MethodSignature signature = (MethodSignature) joinPoint.getSignature();
        Toast.makeText((Context) joinPoint.getTarget(), "aspectj"+signature.toShortString(), Toast.LENGTH_SHORT).show();
        Log.e(TAG, ""+joinPoint.getTarget());
    }

}
```

运行一下，立马见效果， 

 ### 查看编译后的源码 
源码路径在 ： app/build/intermediates/classes/xxx/MainActivity.class  
你会看到多了 如下代码：
```java
     JoinPoint var4 = Factory.makeJP(ajc$tjp_0, this, this, savedInstanceState);
     ActivityLifeAspect.aspectOf().adviceOnCreate(var4);
```