---
title: ButterKnife源码分析
copyright: true
categories: android
date: 2018-10-05 10:25:56
tags: 源码解析
---


# ButterKnife源码分析
> 基于 ButterKnife 8.8.0版本进行分析

## ButterKnife 使用

### 依赖说明
```
dependencies {
  implementation 'com.jakewharton:butterknife:8.8.1'
  annotationProcessor 'com.jakewharton:butterknife-compiler:8.8.1'
}
```
### 简单使用
```java
class ExampleActivity extends Activity {
  @BindView(R.id.user) EditText username;
  @BindView(R.id.pass) EditText password;

  @BindString(R.string.login_error) String loginErrorMessage;

  @OnClick(R.id.submit) void submit() {
    // TODO call server...
  }

  @Override public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.simple_activity);
    ButterKnife.bind(this);
    // TODO Use fields...
  }
}
```
### 多模块使用
> 在Library 中使用ButterKnife 以上操作是无法使用的 ，我们还需要借助`Butterknife插件`来实现功能

在您的**root build.gradle**文件中buildscript:
```
buildscript {
  repositories {
    mavenCentral()
    google()
   }
  dependencies {
    classpath 'com.jakewharton:butterknife-gradle-plugin:8.8.1'
  }
}
```

#### 添加插件  apply
```
apply plugin: 'com.android.library'
apply plugin: 'com.jakewharton.butterknife'
```
#### 插件使用Butterknife 

```
class ExampleActivity extends Activity {
  @BindView(R2.id.user) EditText username;
  @BindView(R2.id.pass) EditText password;
    ...
}
```

### 问题说明
   * 在Library中无法使用在注解中无法使用R的资源
     >  原因很简单，因为Lib模块中 java注解无法使用变量 然而lib生成的R 文件资源都是 `public static` 不是 `public static final `  简单的说lib中生成的R文件不是常量
      
      编译器将会报错：Attribute value must be constant ，如下图：
      ![](https://ws1.sinaimg.cn/large/882b6a2aly1g0h9wcdpzcj20d002c755.jpg)

[Butterknife 官网简单使用说明](http://jakewharton.github.io/butterknife/)

## ButterKnife源码分析

### [源码地址](https://github.com/JakeWharton/butterknife)
### 源码module结构 
#### 组件依赖关系
> ButterKnife 共7个组件，他们的依赖关系如下图所示 butterknife-integration-test；该项目的测试用例--不做介绍


![](http://ww1.sinaimg.cn/large/882b6a2aly1fznagq9al6j20iv0bwt8n.jpg)

+ butterknife：这个工程提供了 ButterKnife.bind(this)，这是 ButterKnife 对外提供的门面。也是运行时，触发 Activity 中 View 控件绑定的时机,提供android使用的API。
+ butterknife-compiler：java－model 编译期间将使用该工程，他的作用是解析注解，并且生成 Activity 中 View 绑定的 Java 文件。
+ butterknife-annotations：java－model 将所有自定义的注解放在此工程下， 确保职责的单一。
+ butterknife-gradle-plugin：gradle 插件，这是8.2.0版本起为了支持 library 工程而新增的一个插件工程。
+ butterknife-lint：针对 butterknife-gradle-plugin 而做的静态代码检查工具，非常有态度的一种做法，在下文做详细介绍。

### 编译 生成Java代码
> 前提以上基本用法已经加入工程


##### 简单使用
```java
public class MainActivity extends AppCompatActivity {
    @BindView(R.id.btn1)
    Button btn;
    @BindView(R.id.tv1)
    TextView tv;

    @OnClick(R.id.btn1)
    public void btnClick() {
        Toast.makeText(this, "Butterknfie 简单使用", Toast.LENGTH_SHORT).show();
    }
}
```

#### Butterknife 之 编译期
 > android-apt(Annotation Processing Tool) ，在Java代码的编译时期，javac 会调用java注解处理器来生成辅助代码。生成的代码就在 `build/generated/source/apt`

```java
public class MainActivity_ViewBinding<T extends MainActivity> implements Unbinder {
  protected T target;

  private View view2131427416;

  @UiThread
  public MainActivity_ViewBinding(final T target, View source) {
    this.target = target;

    View view;
    view = Utils.findRequiredView(source, R.id.btn1, "field 'btn' and method 'btnClick'");
    //这里的 target 其实就是我们的 Activity
    //这个castView就是将得到的View转化成具体的子View
    target.btn = Utils.castView(view, R.id.btn1, "field 'btn'", Button.class);
    view2131427416 = view;
    //为按钮设置点击事件
    view.setOnClickListener(new DebouncingOnClickListener() {
      @Override
      public void doClick(View p0) {
        target.btnClick();
      }
    });
    target.tv = Utils.findRequiredViewAsType(source, R.id.tv1, "field 'tv'", TextView.class);
  }

  @Override
  @CallSuper
  public void unbind() {
    T target = this.target;
    if (target == null) throw new IllegalStateException("Bindings already cleared.");

    target.btn = null;
    target.tv = null;

    view2131427416.setOnClickListener(null);
    view2131427416 = null;

    this.target = null;
  }
```
* Utils.findRequiredView 方法的封装

```java
// Utils.findRequiredView
 View view = source.findViewById(id);
    if (view != null) {
      return view;
    }
```

### Butterknife 之 android-apt(Annotation Processing Tool)
>  APT(Annotation Processing Tool)，即注解处理工具。在该方案中，通常有个必备的三件套，分别是注解处理器 Processor，注册注解处理器 AutoService 和代码生成工具 JavaPoet。

#### 注解处理器 Processor
> ButterKnife 一切皆注解，因此首先需要个处理器来解析注解。 ButterKnifeProcessor 充当了该角色，其中 process 方法是触发注解解析的入口，所有的神奇的事情从这里发生。
process 方法中主要做两件事情，分别是：

+ 解析所有包含了 ButterKnife 注解的类
+ 根据解析结果，使用 JavaPoet 生成相应的Java文件
![ButterKnifeProcessor#process源码](https://ws1.sinaimg.cn/large/882b6a2aly1g0hbw37sx4j214k0j8n0q.jpg)

`findAndParseTargets(env)` 中解析注解的代码非常冗长，依次对 `@BindArray` 、`@BindColor`、`@BindString`、`@BindView` 等注解进行解析，解析结果存放在  bindingMap  中。

这里重点关注下 bindingMap 的键值对。key 值为 TypeElement 对象 ，可以简单的理解为被解析的类本身，而 value 值为 BindingSet 对象，该对象存放了解析结果，根据该结果，JavaPoet 将生成不同的 Java 文件，


以官方 sample 为例，其映射关系如下：

|  key	| value	|JavaPoet 根据 value 生成的文件 |
| ------| ----- |  ------|
|SimpleActivity	 |BindingSet	|SimpleActivity_ViewBinding.java
|SimpleAdapter	 |BindingSet	|SimpleAdapter$ViewHolder_ViewBinding.java


![](https://ws1.sinaimg.cn/large/882b6a2aly1g0hc6tnm3rj21340n4n7y.jpg)


#### 注册注解处理器 [AutoService](https://link.jianshu.com/?t=https://github.com/google/auto/tree/master/service)
> 定义完注解处理器后，还需要告诉编译器该注解处理器的信息，需在 `src/main/resource/META-INF/service` 目录下增加   `javax.annotation.processing.Processor` 文件，并将注解处理器的类名配置在该文件中。

整个过程比较繁琐，Google 为我们提供了更便利的工具，叫 `AutoService`，此时只需要为注解处理器增加 `@AutoService` 注解就可以了，如下：

```java
@AutoService(Processor.class)
public final class ButterKnifeProcessor extends AbstractProcessor {

}

```
[注] [AutoService使用的是`java.util.ServiceLoader`]


#### Java编写器 [JavaPoet](https://github.com/square/javapoet)
> 了解 JavaPoet ，最好的方式便是看[官方文档](https://github.com/square/javapoet)。简而言之，当我们写一个类时，其实是有固定结构的，JavaPoet 提供了生成这些结构的 api

举例如下：

+ 类：TypeSpec.classBuilder()

+ 构造器：MethodSpec.constructorBuilder()

+ 方法：MethodSpec.methodBuilder()

+ 参数：ParameterSpec.builder()

+ 属性：FieldSpec.builder()

+ 程序片段：CodeBlock.builder()

 以 ButterKnife 而言，他做的事情便是将注解处理器解析后的结果（实际上就是上文提到的 BindingSet 对象）生成 Activity_ViewBinding.java，该对象负责绑定 Activity 中的 View 控件以及设置监听器等。
 
 那么 JavaPoet 是如何处理的？实际上 ButterKnife 会将上文提到的 BindingSet 转换成类似于下文所示的代码：
 示例如下：

 ```java
// 创建类
TypeSpec typeSpec = TypeSpec.classBuilder("TestActivity_ViewBinding")
        .addModifiers(PUBLIC) // 类为public
        .addSuperinterface(UNBINDER) // 类为Unbinder的实现类
        .addField(targetField) // 生成属性 private TestActivity target
        .addMethod(constructorForActivity) // 生成构造器1
        .addMethod(otherConstructor) // 生成构造器2
        .addMethod(unBindeMethod) // 生成unbind()方法
        .build();
// 生成 Java 文件
JavaFile javaFile = JavaFile.builder("com.zdg", typeSpec)//包名和类
        .addFileComment("Generated code from Butter Knife. Do not modify!")
        .build();
javaFile.writeTo(System.out);
```


最后总结下这三件套的协作流程，如下图:

![](https://ws1.sinaimg.cn/large/882b6a2aly1g0hcpb81duj20bd0it41b.jpg)


### Butterknife 之 运行期
> 接下来我们来分析下运行期间发生的事情，相比于编译期间，运行期间的逻辑简单了许多。继续使用上面的Demo例子

运行时的入口在于 `ButterKnife.bind(this)`，追溯源码发现，最终将会执行以下逻辑：
```java
// 最终将找到 SimpleActivity_ViewBinding 的构造器，并实例化
Constructor<? extends Unbinder> constructor = findBindingConstructorForClass(targetClass);
constructor.newInstance(target, source);
```

也就是说 ButterKnife.bind(this) 等价于如下代码：

```java
View sourceView = activity.getWindow().getDecorView();
new SimpleActivity_ViewBinding(activity,sourceView);
```

注：虽然这里使用了反射，但源码中将 `Class.forName` 的结果缓存起来后再通过 `newInstance` 创建实例，避免重复加载类，提升性能。

编译期间和运行期间相辅相成，这便是 android-apt 的普遍套路。

### Library 
> 编译时和运行时的问题解决了，还有最后一个问题：由 R 生成 R2 的意义是什么？

如果你细心的话会发现在官方的 sample-library 中，注解的值均是由 R2 来引用的，如下图：
![](https://ws1.sinaimg.cn/large/882b6a2aly1g0hd2kvbtsj20ik041q62.jpg)

如果非 library 工程，则仍然引用系统生成的 R 文件。所以可以猜测：R2 的诞生是为 library 工程量身打造的。

`在上面我说过R文件的问题，library中生成的R文件资源文件不是常量 无法使用注解`

`JakeWharton大神`他是怎么解决这个问题呢？？？

> 既然 R 不能满足要求，那就自己构建一个 R2，由 R 复制而来，并且将其属性都修改为 public static final 来修饰的常量。为了让使用者对整个过程无感知，因此使用 gradle 插件来解决这个需求，这也是 `butterknife-gradle-plugin` 工程的由来。

#### butterknife-gradle-plugin

`butterknife-gradle-plugin` 有两个重要的第三方依赖，分别是 `javaparser` 和 `javapoet` ，前者用于解析 Java 文件，也就是解析 R 文件，后者用于将解析结果生成 R2 文件。
整个插件工程的源码并不难理解，在生成 R2 文件时，要将属性定义成 public static final ，在源码中我们可以看到此逻辑，在 FinalRClassBuilder.addResourceField() 中 ：

```java
FieldSpec.Builder fieldSpecBuilder = FieldSpec.builder(int.class, fieldName)
        .addModifiers(PUBLIC, STATIC, FINAL)
        .initializer(fieldValue);
```
`butterknife` 插件在 `processResources` 的 Task 中执行，该任务通常用来完成文件的 copy。

有关插件的编写 大家可以查看其他插件编写教程

#### JakeWharton 给ButterKnife 的情怀和态度------butterknife-lint 
>一个静态代码检查工具，用来验证非法的 R2 引用。一旦在我们的业务项目里不小心引用了 R2 文件，

当执行 Lint 后，将会有如下图的提示信息：

![](https://ws1.sinaimg.cn/large/882b6a2aly1g0hdkrf91gj21mu0bswmk.jpg)


 

[参考文章](https://www.jianshu.com/p/b8b59fb80554)