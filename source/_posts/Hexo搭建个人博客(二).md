---
title: Hexo搭建个人博客二
copyright: true
categories: Hexo
date: 2017-07-31 15:03:48
tags: hexo
---

# 认识Hexo工程

## 目录结构
```
 theme  //主题
 scaffolds    //模板
 public   //生成静态页面目录
 source   //一般是您写的文章资源目录 page
 _config.yml  //hexo 配置文件
```
## theme
> Hexo 的[主题](https://hexo.io/themes/)文件 可以在_config.yml文件中修改

修改主题只需要修改 _config.yml文件中的` theme: xxx `  搜索theme 你就知道了

## source
> 每一个文件夹都是一个page

 _posts 是你的文章目录
```
  hexo new page  tags/categories/about          // 创建各个page
```
## _config.yml
> hexo 的配置文件 语法一般都是 ` key: 后面要有空格 `
这里不多说网上很多介绍

## 配置Github --->不同部署两个Repository
  * 建立Repository

   建立与你用户名对应的仓库，仓库名必须为`【your_user_name.github.io】`，固定写法 然后建立关联
   修改 `_config.yml` 文件` deploy ` 搜索 ` deploy `

```
deploy:
  type: git
  repository: xxxxx     (可以使用http或者 ssh)
  branch: branchname
如果想配置多个Repository
```
**如果想配置多个Repository**
```
deploy:
  type: git
  repo:
      github: xxxx,branchname
      coding: xxxx,branchname
```

  * 安装hexo提交resp依赖库
```
npm install hexo-deployer-git --save
```
## 部署步骤
每次部署的步骤，可按以下三步来进行。三部曲
```
hexo clean   
hexo generate  缩写 （hexo g）
hexo deploy    缩写 （hexo d）

//或者使用联合命令
hexo clean &&  hexo generate && hexo deploy
```
如果提交成功，你可以用访问你的博客 访问 ` https://your_name.github.io ` 就能访问！！！！
