---
title: ReactNative基础之State和Props
copyright: true
tags: react-native
categories: react-native
date: 2018-06-06 14:27:19
---


# ReactNative基础之State和Props
> React Native 看起来很像 React，只不过其基础组件是原生组件而非 web 组件。要理解 React Native 应用的基本结构，首先需要了解一些基本的 React 的概念，比如 [JSX 语法](http://www.cnblogs.com/zourong/p/6043914.html)、组件、state状态以及props属性。如果你已经了解了 React，那么还需要掌握一些 React Native 特有的知识，比如原生组件的使用。这篇教程可以供任何基础的读者学习，不管你是否有 React 方面的经验。
> 


`在任何应用中，数据都是必不可少的。我们需要直接的改变页面上一块的区域来使得视图的刷新，或者间接地改变其他地方的数据。React的数据是自顶向下单向流动的，即从父组件到子组件中，组件的数据存储在props和state中，这两个属性有啥子区别呢？`

## Props
React的核心思想就是组件化思想，页面会被切分成一些独立的、可复用的组件。

组件从概念上看就是一个函数，可以接受一个参数作为输入值，这个参数就是props，所以可以把props理解为从外部传入组件内部的数据。由于React是`单向数据流`，所以props基本上也就是从父级组件向子组件传递的数据。

### 用法
假设我们现在需要实现一个列表，根据React组件化思想，我们可以把列表中的行当做一个组件，也就是有这样两个组件：<ItemList/>和<Item/>。

**父控件 <ItemList/>**
```js
 import Item from "./item";
 export default class ItemList extends React.Component{
      const itemList = data.map(item => <Item item=item />);
      render(){
        return (
          {itemList}
        )
      }
    }
```

列表的数据我们就暂时先假设是放在一个data变量中，然后通过map函数返回一个每一项都是<Item item='数据'/>的数组，也就是说这里其实包含了data.length个<Item/>组件，数据通过在组件上自定义一个参数传递。当然，这里想传递几个自定义参数都可以。

**子控件 <Item/>**
```js
 export default class Item extends React.Component{
  render(){
    return (
      <Text>{this.props.item}</Text>
    )
  }
}
```

在render函数中可以看出，组件内部是使用this.props来获取传递到该组件的所有数据，它是一个对象，包含了所有你对这个组件的配置，现在只包含了一个item属性，所以通过this.props.item来获取即可。

#### 只读性
> props经常被用作渲染组件和初始化状态，当一个组件被实例化之后，它的props是`只读的，不可改变的`。如果props在渲染过程中可以被改变，会导致这个组件显示的形态变得不可预测。只有通过父组件重新渲染的方式才可以把新的props传入组件中。
#### 默认参数
在组件中，我们最好为props中的参数设置一个defaultProps，并且制定它的类型。
比如，这样：
```js
     //定义默认参数
    static defaultProps = {
      item: 'Hello Props',
    };
    //定义Props类型
    static propTypes = {
      item: PropTypes.string,
    };
```

关于propTypes，可以声明为以下几种类型：

```js
    optionalArray:  PropTypes.array,
    optionalBool:   PropTypes.bool,
    optionalFunc:   PropTypes.func,
    optionalNumber: PropTypes.number,
    optionalObject: PropTypes.object,
    optionalString: PropTypes.string,
    optionalSymbol: PropTypes.symbol,
```

官方文档： https://facebook.github.io/react/docs/typechecking-with-proptypes.html

`总结：`props是一个从外部传进组件的参数，主要作为就是从父组件向子组件传递数据，它具有可读性和不变性，只能通过外部组件主动传入新的props来重新渲染子组件，否则子组件的props以及展现形式不会改变。

## state
> State is similar to props, but it is private and fully controlled by the component.
> 一个组件的显示形态可以由数据状态和外部参数所决定，外部参数也就是props，而数据状态就是state。

### 用法

```js
export default class ItemList extends React.Component{
  //初始化 state 有两种方式
  // state = {
 //    itemList:'一些数据',
 //   }
  constructor(){
    super();
    this.state = {
      itemList:'一些数据',
    }
  }
  render(){
    return (
      {this.state.itemList}
    )
  }
}
```
首先，在组件初始化的时候，通过this.state给组件必须设定一个初始的state，在第一次render的时候就会用这个数据来渲染组件。

### setState

`state`不同于`props`的一点是，state是可以被改变的。不过，不可以直接通过this.state=的方式来修改，而需要通过this.setState()方法来修改state。

比如，我们经常会通过异步操作来获取数据，我们需要在`componentDidMount`阶段来执行异步操作：

```js
componentDidMount(){
  fetch('url')
    .then(response => response.json())
    .then((data) => {
      this.setState({itemList:item});  
    }
}
```

当数据获取完成后，通过`this.setState`来修改数据状态。

当我们调用`this.setState`方法时，React会更新组件的数据状态state，并且重新调用`render`方法，也就是会对组件进行重新渲染。

注意：通过this.state=来初始化state，使用`this.setState`来修改state，constructor是唯一能够初始化的地方。

setState接受一个对象或者函数作为第一个参数，只需要传入需要更新的部分即可，不需要传入整个对象，比如：

```js
export default class ItemList extends React.Component{
  constructor(){
    super();
    //上面说过 有两种方式初始化 state 
    this.state = {
      name: '老黄',
      age: 25,
    }
  }
  componentDidMount(){
    this.setState({age:18})  
  }
}
```

在执行完`setState`之后的state应该是{name:'老黄',age:18}。

setState还可以接受第二个参数，它是一个函数，会在setState调用完成并且组件开始重新渲染时被调用，可以用来监听渲染是否完成：

```js
    this.setState({
      name:'被修改了'
    },()=>console.log('setState finished'))
```


### **区别**
`state`是组件自己管理数据，控制自己的状态，可变；
`props`是外部传入的数据参数，不可变；
没有`state`的叫做`无状态组件`，有`state`的叫做`有状态组件`；
多用props，少用state。也就是多写无状态组件。


**PS : componentDidMount在React 16时候 已经被废弃了, [新生命周期](https://segmentfault.com/a/1190000014456811?utm_source=channel-hottest)**