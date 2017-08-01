---
title: react-native TabNavigator
date: 2017-07-05
copyright: true
tags: react-native
categories: react-native
---

> 上文已经说过StackNavigator,这篇文章说说TabNavigator，TabNavigator 与 StackNavigator  用法类似 语法相似


### TabNavigator平台差异 
 > Android IOS 默认显示效果不一样， Android  tab默认在Top而IOS 默认在Buttom 不过我们可以设置参数改变这些默认效果
``` js
    const Presets = {
    iOSBottomTabs: {
        tabBarComponent: TabBarBottom,
        tabBarPosition: 'bottom',
        swipeEnabled: false,
        animationEnabled: false,
        lazy: false,
    },
    AndroidTopTabs: {
        tabBarComponent: TabBarTop,
        tabBarPosition: 'top',
        swipeEnabled: true,
        animationEnabled: true,
        lazy: false,
    },
};
```
<!-- more -->
### 参数说明
```js

export default (
    routeConfigMap: NavigationRouteConfigMap,
    stackConfig: StackNavigatorConfig = {}
    )
    TabNavigator = (
    routeConfigs: NavigationRouteConfigMap,  //和StackNacigator 一样
    config: TabNavigatorConfig = {}
    )
   export type NavigationRouteConfigMap = {
  [routeName: string]: NavigationRouteConfig<*>,
};
```

#### routeConfigs （第一个参数）
> 都是基于 ` NavigationRouteConfigMap `    但不同点在于  navigationOptions  里面的参数不一致

上文已经说过了StackNavigator 这里就不累述了

在 ` navigationOptions ` 中 ` tabBar*系列 `都是设置TabNavigator的
  - ` title `
  - ` tabBarLabel `
  - ` tabBarIcon  `
  - ` tabBarVisible  `

```js
export type NavigationScreenOptions = {
    title?: string,
    };
export type NavigationTabScreenOptions = NavigationScreenOptions & {
    tabBarIcon?:
        | React.Element<*>
        | ((options: { tintColor: ?string, focused: boolean }) => ?React.Element<
        *
        >),
    tabBarLabel?:
        | string
        | React.Element<*>
        | ((options: { tintColor: ?string, focused: boolean }) => ?React.Element<
        *
        >),
    tabBarVisible?: boolean,
    };

export type TabViewConfig = {
    tabBarComponent?: ReactClass<*>,
    tabBarPosition?: 'top' | 'bottom',
    tabBarOptions?: {},
    swipeEnabled?: boolean,
    animationEnabled?: boolean,
    lazy?: boolean,
};
```
通过上面源码可以知道:
 - ` title ` 是一个` string `类型的   会同时设置StackNavigator的title(当全局没有设置title属性时)
 - ` tabBarLabel ` 是既可以是` string `又可以是` function(tintColor: ?string, focused: boolean) `
 - ` tabBarIcon ` 是既可以是` string `又可以是` function(tintColor: ?string, focused: boolean) `
 - ` tabBarVisible ` boolean 类型 是否显示tabbar 默认显示


#### TabNavigatorConfig （第二个参数）

直接上源码
```js
export type TabNavigatorConfig = {
    containerOptions?: void,
    } & NavigationTabRouterConfig &
    TabViewConfig;
    export type NavigationTabRouterConfig = {
        initialRouteName?: string,
        paths?: NavigationPathsConfig,
        navigationOptions?: NavigationScreenConfig<NavigationTabScreenOptions>,
        //以下是TabBar特有的
        order?: Array<string>, // todo: type these as the real route names rather than 'string'
        // Does the back button cause the router to switch to the initial tab
        backBehavior?: 'none' | 'initialRoute', // defaults `initialRoute`

export type TabViewConfig = {
        tabBarComponent?: ReactClass<*>,
        tabBarPosition?: 'top' | 'bottom',
        tabBarOptions?: {},
        swipeEnabled?: boolean,
        animationEnabled?: boolean,
        lazy?: boolean,
    };
};
```
{
 - ` initialRouteName `    设置默认的页面组件
 - ` paths ` 
 - ` navigationOptions ` 
 - ` order `            routerName的数组，顺序决定着tab的顺序。
 - ` backBehavior `     按 back 键是否跳转到第一个Tab(首页)， none 为不跳转
 - ` tabBarPosition `  设置tabbar的位置，iOS默认在底部，安卓默认在顶部。（属性值：'top'，'bottom'）
 - ` swipeEnabled `     是否允许在标签之间进行滑动。
 - ` animationEnabled `  是否在更改标签时显示动画。
 - ` lazy `      
 - ` tabBarOptions `     配置标签栏的一些属性

}
  
由以上源码可知
与` StackNavigator ` 区别多了 ` order ` , ` backBehavior `,` tabBarPosition `,` swipeEnabled `,` animationEnabled `,` lazy `,` backBehavior ` 还有 ` navigationOptions `,` tabBarOptions ` 的区别

#### tabBarOptions   配置标签栏的一些属性
```js
ios  TabBarBottom.js
type DefaultProps = {
        activeTintColor: string,  // label和icon的前景色 活跃状态下（选中）。
        activeBackgroundColor: string,  //：label和icon的背景色 活跃状态下（选中） 。
        inactiveTintColor: string,    //label和icon的前景色 不活跃状态下(未选中)。
        inactiveBackgroundColor: string,  //label和icon的背景色 不活跃状态下（未选中）。
        showLabel: boolean,  // 是否显示label，默认开启。
    };

type Props = {
        activeTintColor: string,  // label和icon的前景色 活跃状态下（选中）。
        activeBackgroundColor: string,  //：label和icon的背景色 活跃状态下（选中） 。
        inactiveTintColor: string,      //label和icon的前景色 不活跃状态下(未选中)。
        inactiveBackgroundColor: string,  //label和icon的背景色 不活跃状态下（未选中）。
        position: Animated.Value,
        navigation: NavigationScreenProp<NavigationState, NavigationAction>,
        jumpToIndex: (index: number) => void,
        getLabel: (scene: TabScene) => ?(React.Element<*> | string),
        renderIcon: (scene: TabScene) => React.Element<*>,
        showLabel: boolean,   // 是否显示label，默认开启。
        style?: Style,       //tabbar的样式。
        labelStyle?: Style,   //label的样式。
        showIcon: boolean,      //是否显示图标，默认关闭。
    };

android  TabBarTop.js
type DefaultProps = {
        activeTintColor: string,  //label和icon的前景色 活跃状态下（选中） 。
        inactiveTintColor: string,  //label和icon的前景色 不活跃状态下(未选中)。
        showIcon: boolean,          //是否显示图标，默认关闭。
        showLabel: boolean,         // 是否显示label，默认开启。
        upperCaseLabel: boolean,   //是否使标签大写，默认为true。
    };
     indicatorStyle：标签指示器的样式对象（选项卡底部的行）。安卓底部会多出一条线，可以将height设置为0来暂时解决这个问题。
type Props = {
        activeTintColor: string,//label和icon的前景色 活跃状态下（选中） 。
        inactiveTintColor: string, //label和icon的前景色 不活跃状态下(未选中)。
        showIcon: boolean,          //是否显示图标，默认关闭。
        showLabel: boolean,          // 是否显示label，默认开启。
        upperCaseLabel: boolean, //是否使标签大写，默认为true。
        position: Animated.Value,
        navigation: NavigationScreenProp<NavigationState, NavigationAction>,
        getLabel: (scene: TabScene) => ?(React.Element<*> | string),
        renderIcon: (scene: TabScene) => React.Element<*>,
        labelStyle?: Style,     //label的样式。
        iconStyle?: Style,      //图标的样式。
    };
```
```js
android  默认的 iconStyle 和 labelStyle
   icon: {
    height: 24,
    width: 24,
  },
  label: {
    textAlign: 'center',
    fontSize: 13,
    margin: 8,
    backgroundColor: 'transparent',
  },
  ios  默认的 iconStyle 和 labelStyle
   icon: {
    flexGrow: 1,
  },
  label: {
    textAlign: 'center',
    fontSize: 10,
    marginBottom: 1.5,
    backgroundColor: 'transparent',
  },
```
上述可知   Android  特有的属性 ： ` indicatorStyle `,` iconStyle `,` upperCaseLabel `