---
title: Hexo搭建个人博客三
copyright: true
categories: Hexo
date: 2017-07-31 15:44:31
tags: hexo
---
> 默认是 landscape 主题 

<!-- more -->

主题的配置，主要在_config.yml

### _config.yml
> 每个主题 都要自己的配置，但是还是有些共同的特性
```
menu:
  home: /
  categories: /categories/    #分类
  archives: /archives/        #归档
  tags: /tags/                #tag
  about: /about/  
```
以上菜单按钮， 

问题 ：Cannot GET /xxxx/

menu 中每个item 都是一个page ，我们看看` source文件夹中是否有对应名字的文件夹 ` 如果没有的话，我们需要通过如下命令操作
```
hexo new page about
```

### 推荐主题
[NexT](http://theme-next.iissnan.com/)
