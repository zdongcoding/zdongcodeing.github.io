---
title: Hexo搭建个人博客(一)
copyright: true
categories: Hexo
date: 2017-07-30 13:10:32
tags: hexo
---

# 环境配置
>[Hexo](https://github.com/hexojs/hexo)是一款基于Node.js的静态博客框架, hexo github或者conding的pages

## 必须安装
  + npm,[Node](https://nodejs.org/en/)  
  + [Git]()  
  + [GitHub账号](https://github.com)  

## 安装HEXO
 ```
    npm install -g hexo   //安装全局hexo
 ```

## 开始搞事情
###  1、创建一个Hexo
```
hexo  init  'name'  //初始化成功，提示：INFO  Start blogging with Hexo!  
``` 
 继续进入hexo工程目录下执行如下命令 

###  2、安装依赖库
``` 
npm insatll
```
###  2、生成静态页面
```
hexo generate （hexo g  也可以）   
```
### 3、本地启动
    启动本地服务，进行文章预览调试，命令：
```
hexo server  (hexo s  也可以)
```
以上您还可以 命令联合操作 输入 ：` hexo s -g ` 将第2、3 步同步执行

浏览器输入http://localhost:4000 
