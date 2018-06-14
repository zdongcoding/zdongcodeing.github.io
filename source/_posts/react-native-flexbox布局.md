---
title: react-native-flexbox布局
copyright: true
categories: android
date: 2018-06-07 14:27:19
tags: react-native
---

## 什么是Flexbox布局（“弹性布局”）

> React中引入了flexbox概念,flexbox是属于web前端领域CSS的一种布局方案，是2009年W3C提出了一种新的布局方案，可以简便、完整、响应式地实现各种页面布局。你可以简单的理解为flexbox是CSS领域类似Android中 `LinearLayout`的一种布局，但是要比 `LinearLayout`要强大的多!

##  flexbox布局的组成

  1.  首先使用Flexbox需要理解两个概念： 主轴（main axis）和侧轴（cross axis）主轴和侧轴是互相垂直的，在`react-native`中 主轴是**垂直方向**，
  2.  flexbox 布局由**伸缩容器**和**伸缩项目**组成。任何一个元素都可以指定为 flexbox 布局。
  3.  伸缩容器的子元素称为伸缩项目。


## 伸缩容器的相关属性说明
### `flexDirection`  控制主轴方向
 > **row**(水平方向) | **column**(垂直方向)

```js
flexDirection: 'column'  //row | column(react-native默认）
```
<center>
<img src="http://ow9n8vqns.bkt.clouddn.com/flexDirection_column.png" width="40%" height="40%" />
flexDirection_column
</center>

<center>
<img src="http://ow9n8vqns.bkt.clouddn.com/flexDirection_row.png" width="40%" height="40%" />
flexDirection_row
</center>

### `justifyContent` 伸缩项目在伸缩容器的主轴方向对齐方式
 > flex-start、center、flex-end、space-around、space-between

 ```js
 justifyContent: 'flex-start'   //默认方式
 ```
  -  `flex-start`（默认值）：伸缩项目向主轴线的起始位置靠齐。
  -  `center` ：  伸缩项目向主轴线的中间位置靠齐。
  -  `flex-end` ：伸缩项目向主轴线的结束位置靠齐。
  -  `space-between` ：伸缩项目会平均地分布在主轴线里。第一个项目在主轴线开始位置，最后一个项目在终点位置。
  -  `space-around` ：伸缩项目会平均地分布在主轴线里，两端保留一半的空间。

`space-around` 表示单个**元素左右**的间距 注意对比`space-between`

`space-between`表示两个**元素之间**的间距 注意对比`space-around`

<center>
<img src="http://ow9n8vqns.bkt.clouddn.com/justifyContent_flex_center.png" width="40%" height="40%" />
justifyContent_flex_center
<img src="http://ow9n8vqns.bkt.clouddn.com/justifyContent_flex_end.png" width="40%" height="40%" />
justifyContent_flex_end

<img src="http://ow9n8vqns.bkt.clouddn.com/justifyContent_space_between.png" width="40%" height="40%" />
justifyContent_space_between

<img src="http://ow9n8vqns.bkt.clouddn.com/justifyContent_space-around.png" width="40%" height="40%" />
justifyContent_space-around
</center>
 

### <span id="jump_alignItems">`alignItems`</span> 伸缩项目在伸缩容器的侧轴上的对齐方式
> flex-start、center、flex-end、stretch

- `flex-start`(默认值)：伸缩项目向侧轴的起始位置靠齐。
- `flex-end`：伸缩项目向侧轴的结束位置靠齐。
- `center`：伸缩项目向侧轴的中间位置靠齐。
- `stretch`（默认值）：伸缩项目在侧轴方向拉伸填充整个伸缩容器。
注意：这种情况下项目不能设置高度，否则看不到效果。

<center>
<img src="http://ow9n8vqns.bkt.clouddn.com/alignItems_flex-end.png" width="40%" height="40%" />
alignItems_flex-end

<img src="http://ow9n8vqns.bkt.clouddn.com/alignItems_stretch.png" width="40%" height="40%" />
alignItems_stretch
</center>
 

### `flexWrap` 伸缩项目在伸缩容器的主轴上显示不下的对齐方式
> nowrap | wrap   nowrap 不换行  ， wrap换行

### `flex` 伸缩项目的放大比例
> 默认值是 0，即表示如果存在剩余空间，也不放大。
如果将所有的伸缩项目的 `flex` 设置为 1，那么每个伸缩项目将设置为一个大小相等的剩余空间。
如果又将其中一个伸缩项目的 `flex` 设置为 2，那么这个伸缩项目所占的剩余空间是其他伸缩项目所占的剩余空间的两倍。

宽度 ＝ 弹性宽度 * ( flexGrow / sum( flexGorw ) )


###  `alignSelf` 设置单独的伸缩项目在侧轴上的对齐方式，会覆盖默认的对齐方式
> auto、flex-start、flex-end、center、stretch。

**使用场景：** 伸缩容器中设置了 [**alignItems**](#jump_alignItems) 对齐方式 ，`alignSelf`设置单个伸缩项目单独的对齐方式

<center>
<img src="http://ow9n8vqns.bkt.clouddn.com/alignSeft_flex-end.png" width="40%" height="40%" />
alignSeft_flex-end
</center>


-------------
参考：

[从零学React Native之flexbox布局](https://www.jianshu.com/p/2acddb3731a7)


