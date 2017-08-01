---
title: react-native StackNavigator
date: 2017-07-04
copyright: true
tags: react-native
categories: react-native
---

>  npm install --save react-navigation

> 控件主要分为三种：
   1. StackNavigator ：类似于普通的Navigator，屏幕上方导航栏
   2. TabNavigator：obviously, 相当于iOS里面的TabBarController，屏幕下方标签栏
   3. DrawerNavigator：抽屉效果，左侧滑出这种效果。
<!-- more -->
###  StackNavigator

#### 使用方法

   **以下数据可以全局设置也可以单独配置**

   1. 初始化
      * StarkNavigator(routeConfigMap,stackConfig)
      routeConfigMap所有跳转的routeName 都在这配置 默认取第一个route
      stackConfig 使用页面的配置（TitleBar 都可以在这里统一配置）
   2. 跳转
      * 通过 this.props.navigation.navigate 获取  navigator
      * 调用 navigator(routName,params)进行跳转

 例子：
 ```
    const Nav = StackNavigator({
        Launch: {
            screen: HomeScreen
        },
        NavTitle: {
            screen: NavTitle
        },
        TabNav: {
            screen: MainScreenNavigator
        },
        Touchable: {screen: Touchable}
    }, {
        initialRouteName: 'Launch',
        initialRouteParams: { //Launch 初始化参数
            param: '初始化',
            user: 'zoudong'
        },
        navigationOptions: {
            headerTintColor: 'red', //返回键颜色，title 颜色
            headerRight: <Text onPress={() => {
                alert('点击了title Right')
            }}>Right</Text>,
            headerTitle: <Text
                style={{
                    backgroundColor: 'red',
                    alignSelf: 'center',
                    textAlignVertical: 'center'
                }}
                onPress={() => {
                    alert('点击了title Title')
                }}> 全局Title</Text>,
            headerStyle: { /
                backgroundColor: "white"
            },
            headerTitleStyle: {
                alignSelf: 'center'
            }
        },
        mode: 'card'
    });

    export default Nav;
 ```
### 参数说明

> StackNavigator(RouteConfigs, StackNavigatorConfig)

#### RouteConfigs（第一个参数）
```js
    export type NavigationRouteConfigMap = {
    [routeName: string]: NavigationRouteConfig<*>,

    export type NavigationRouteConfig<T> = T & {
         navigationOptions?: NavigationScreenConfig<*>,
         path?: string,
    };
};

```
 1. ` RouteConfigs `  类似 Android中的Mainfest.xml
       - {routeName: NavigationRouteConfig}
    * routeName   //跳转使用的  routeName
    * <spac id='navigationOptions1'/>NavigationRouteConfig ： 页面的配置
           {
         * ` path ` :'app:web', //optional   当深层次关联或者在web app中使用React Navigation,使用路径

         * `navigationOptions`:{
               //此处设置了, 会覆盖组件内的`static navigationOptions`设置
              //设置个标题
              title:({state}) => `${state.params.username}'s Profile'`
    } **这个可以在Component 里面单独写，它会覆盖这个**

 
#### StackNavigatorConfig（第二个参数）
2.  StackNavigatorConfig
```js
 export type StackNavigatorConfig = {
     containerOptions?: void,
    } & NavigationStackViewConfig &
        NavigationStackRouterConfig;
    
    export type NavigationStackViewConfig = {
        mode?: 'card' | 'modal',
        headerMode?: HeaderMode,
        cardStyle?: Style,
        transitionConfig?: () => TransitionConfig,
        onTransitionStart?: () => void,
        onTransitionEnd?: () => void,
    };
    export type NavigationStackRouterConfig = {
        initialRouteName?: string,
        initialRouteParams?: NavigationParams,
        paths?: NavigationPathsConfig,
        navigationOptions?: NavigationScreenConfig<NavigationStackScreenOptions>,
    };
    export type NavigationStackScreenOptions = NavigationScreenOptions & {
        header?: ?(React.Element<*> | (HeaderProps => React.Element<*>)),
        headerTitle?: string | React.Element<*>,
        headerTitleStyle?: Style,
        headerTintColor?: string,
        headerLeft?: React.Element<*>,
        headerBackTitle?: string,
        headerTruncatedBackTitle?: string,
        headerBackTitleStyle?: Style,
        headerPressColorAndroid?: string,
        headerRight?: React.Element<*>,
        headerStyle?: Style,
        gesturesEnabled?: boolean,
    };
```
以上可知  StackNavigatorConfig中有的字段名

   * <spac id='navigationOptions2'/>option for the route(路由选项):
       - ` initialRouteName ` -为stack设置默认的界面，必须和route configs里面的一个key匹配。
       - ` initialRouteParams ` - 初始路由的参数。
       - ` navigationOptions `- 屏幕导航的默认选项。
       - ` paths `-route config里面路径设置的映射。


   * Visual Option(视觉选项):
       - ` mode `- 定义渲染(rendering)和转换(transitions)的模式,两种选项(给字符串即可)：
           - ` card `  使用标准的iOS和Android的界面切换，这是默认的。
           - ` modal ` 仅在iOS端有用，即模态出该视图。
       - ` headerMode ` 指定header应该如何被渲染,选项：
           -  ` float `    共用一个header 意思就是有title文字渐变效果。
           -  ` screen `   各用各的header 意思就是没有title文字渐变效果。
           -  ` none `     没有header。
       - ` cardStyle `  使用该属性继承或者重载一个在stack中的card的样式。
       - ` onTransitionStart `  一个函数，在换场动画开始的时候被激活。
       - ` onTransitionEnd `   一个函数，在换场动画结束的时候被激活。


### navigationOptions

> 以上有两种方式设置[配置单独页面设置](#navigationOptions1), [全局统一设置](#navigationOptions2) 还有一种方式 Component 中设置
   优先级    routeConfigMap  > Component  >  stackConfig
```js
  static navigationOptions = {
    .....
  }
```

- title- 界面的标题(string)
- header- header bar设置对象
   1) visible - bool值，header是否可见。
   2) title-标题 String或者是一个react 节点
   3) backTitle-返回按钮在iOS平台上，默认是title的值
   4) right- react 节点显示在header右边，例如右按钮
   5) left- react 节点显示在header左边，例如左按钮
   6) style-header的style
   7) titleStyle- header的title的style (^__^) 嘻嘻……
   8) tintColor- header的前景色
- cardStack- 配置card stack
1)gesturesEnabled- 是否允许通过手势关闭该界面，在iOS上默认为true，在Android上默认为false

### 不同
1. Navigator 用法有些类似 语法有点不同

例如：
```js
 // Navigator
 const  navor=this.props.navigator;
 //StackNavigator
 const  navor=this.props.navigation.navigate;
```

2. 跳转不一样

例如：
```js
 // Navigator
   navor.push(route); //跳转
   1. 如果带参数:route={
        //component，params 是关键字
            component:xxx,
             params:{
                      id:xxx,
                }
             }
   2. 获取跳转带的参数 ： this.props.id;
 //StackNavigator
   1. navor(routeName,params)
      routeName: 名称
      params:参数
      navor（'Login',{userid:'xxx'}）
```
