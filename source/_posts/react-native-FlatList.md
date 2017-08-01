---
title: react-native FlatList
date: 2017-07-06
copyright: true
tags: react-native
categories: react-native
---

### 使用简介

```js
 <FlatList style={styles.container}
      data={this.state.mData}
      renderItem={this._renderItem}

    this.state.mData 就是你需要渲染的数据
    this._renderItem item 的布局

   _renderItem=({item,index})=>{
        return (
            <View style={styles.item} >
                 <Text style={{color:'#1e1e1e', fontSize:15}}>
                     {item.title +item.id+'---'+index+'--->'}
                 </Text>
            </View>
        );
    }
/>
```
<!-- more -->
以上代码 一个最简单的FlatList 列表了，
 + 问题一： VirtualizedList: missing keys for items, make sure to specify a key property on each item or provide a custom keyExtractor.
   -  解决办法 : FlatList 加入 
``` js
   keyExtractor={this._keyExtractor}
   这个默认情况下 会取 data 中的 key 没有
```

### 方法说明
   - ` data ?Array<ItemT> `  数据 上文已经用到
   - ` renderItem  (info: {item: ItemT, index: number}) => ?React.Element<any> ` 渲染布局
        两个参数  iten,index  retrun  React.Element
   - `  keyExtractor (item: ItemT, index: number) => string  `  此函数用于为给定的item生成一个不重复的key。Key的作用是使React能够区分同类元素的不同个体，以便在刷新时能够确定其变化的位置，减少重新渲染的开销。若不指定此函数，则默认抽取item.key作为key值。若item.key也不存在，则使用数组下标。
   - ` ListHeaderComponent  ?ReactClass<any>`   顾名思义   return  <Component/>
   - ` ListFooterComponent  ?ReactClass<any>`   顾名思义   return  <Component/>
   - ` ItemSeparatorComponent?: ?ReactClass<any>  `  行与行之间的分隔线组件。不会出现在第一行之前和最后一行之后。
   - `getItem?: ,getItemCount?:  ` **报错 FlatList does not support custom data formats.**

   - ` onRefresh?: ?() => void `
        如果设置了此选项，则会在列表头部添加一个标准的RefreshControl控件，以便实现“下拉刷新”的功能。**同时你需要正确设置refreshing属性**。
   - ` refreshing?: ?boolean  ` 如果想下拉刷新 这个属性必须加 否则**报错` refreshing ` prop must be set as a boolean in order to use ` onRefresh `, but got ` undefined `**
   - ` onEndReached?: ?(info: {distanceFromEnd: number}) => void  `
      当所有的数据都已经渲染过，并且列表被滚动到距离最底部不足onEndReachedThreshold个像素的距离时调用。

   - ` onEndReachedThreshold?:  ?number `

   - ` onViewableItemsChanged?:  ?(info: {viewableItems: Array<ViewToken>, changed: Array<ViewToken>}) => void `

   - ` _ListEmptyComponent  ?ReactClass<any> | React.Element<any> ` 列表为空时渲染该组件。可以是React Component, 也可以是一个render函数， 或者渲染好的element。