---
title: ReactNative基础之Text
copyright: true
tags: react-native
categories: react-native
date: 2018-06-08 15:40:19
---


# ReactNative基础之Text
> Text 只是一个显示文本控件， 继承于View

## View
>  [样式（style）和属性(props)](https://reactnative.cn/docs/view-style-props/)  这些就不说了 用到就查文档

## Text
> 除了上面的样式和属性之外 还有一个控件的特殊属性 

#### Style样式
* textShadowOffset: object: {width: number,height: number}

* color: color    //字体颜色

* fontSize: number  //字体大小
 
* fontStyle: enum('normal', 'italic') //字体样式

* fontWeight: enum('normal', 'bold', '100', '200', '300', '400', '500', '600', '700', '800', '900') //字体粗细 
    指定字体的粗细。大多数字体都支持'normal'和'bold'值。并非所有字体都支持所有的数字值。如果某个值不支持，则会自动选择最接近的值。
* lineHeight: number

* textAlign: enum('auto', 'left', 'right', 'center', 'justify')
        指定文本的对齐方式。其中'justify'值仅iOS支持，在Android上会变为left。

* textDecorationLine: enum('none', 'underline', 'line-through', 'underline line-through')

* textShadowColor: color

* fontFamily: string

* textShadowRadius: number

* includeFontPadding: bool (Android)
    Android在默认情况下会为文字额外保留一些padding，以便留出空间摆放上标或是下标的文字。对于某些字体来说，这些额外的padding可能会导致文字难以垂直居中。如果你把textAlignVertical设置为center之后，文字看起来依然不在正中间，那么可以尝试将本属性设置为false。默认值为true。

#### 常用的属性
* allowFontScaling    控制字体是否要根据系统的“字体大小”辅助选项来进行缩放。默认值为true。

* onLongPress    这个Text 特有 点击事件，其他View 需要点击事件需要Touch* 控件包裹着

* onPress        这个Text 特有 点击事件  需要点击事件需要Touch* 控件包裹着

* adjustsFontSizeToFit  `iOS特有`  指定字体是否随着给定样式的限制而自动缩放。

#### 上手操作
我们来修改创建的AwesomeProject 程序的入口App.js 功能
```JS
import React, {Component} from 'react';
import {Platform, StyleSheet, Text, View} from 'react-native';

const instructions = Platform.select({
  ios: 'Press Cmd+R to reload,\n' + 'Cmd+D or shake for dev menu',
  android:
    'Double tap R on your keyboard to reload,\n' +
    'Shake or press menu button for dev menu',
});

//1.创建一个组件
export default class App extends Component {
//2.组件的渲染 render()方法 必须返回 View / null;
  render() {
//3.组件渲染返回的View ,只有返回View才能显示出来
    return (
//4. style样式
      <View style={styles.container}>
        <Text style={styles.welcome}>Welcome to React Native!</Text>
        <Text style={styles.instructions}>To get started, edit App.js</Text>
        <Text style={styles.instructions}>{instructions}</Text>
      </View>
    );
  }
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    backgroundColor: '#F5FCFF',
  },
  welcome: {
    fontSize: 20,
    textAlign: 'center',
    margin: 10,
  },
  instructions: {
    textAlign: 'center',
    color: '#333333',
    marginBottom: 5,
  },
});
```

### 用法

```JS
  <Text style={style} {this.props}> </Text>
```
#### 嵌套文本
下面介绍 Text的特殊用法
> 在iOS和Android中显示格式化文本的方法类似，都是提供你想显示的文本内容，然后使用范围标注来指定一些格式（在iOS上是用NSAttributedString，Android上则是SpannableString）。这种用法非常繁琐。在React Native中，我们决定采用和Web一致的设计，这样你可以把相同格式的文本嵌套包裹起来：


```JS
 <Text style={styles.welcome}>Welcome to 
      <Text style={styles.welcome}> React Native!</Text>
 </Text>

```

#### 嵌套视图（仅限iOS）
+  Android
没错， 就是Text 嵌套 Text 但是不能嵌套View 否则报错 `Invariant Violation:Nesting of <Text> is not currently supported`
![](media/15450566409131/15452028957866.jpg)

还有一个小知识点 `Text` 使用表达式 需要`{ 表达式 }` 包括换行符: {'我要换行\n'} 

+ iOS

  ```JS
  <Text>
        There is a blue square
        <View style={{width: 50, height: 50, backgroundColor: 'steelblue'}} />
        in between my text.
      </Text>
  ```
  ![](media/15450566409131/15452033622860.jpg)
