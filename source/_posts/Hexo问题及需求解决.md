---
title: Hexo问题及需求解决
copyright: true
categories: Hexo
date: 2017-07-31 20:50:23
tags: hexo
---
> 遇到的一些问题， 和解决办法

<!-- more -->

#### 如何让文章想只显示一部分和一个 阅读全文 的按钮？ 
答：在文章中加一个` <!--more--> `，` <!--more--> `后面的内容就不会显示出来了。
#### 个人网址将pages放到子目录下需求
 > * 例如：[blog.zoudongq123.cn](blog.zoudongq123.cn)

- 1.修改_config.yml

```
    # URL
    ## If your site is put in a subdirectory, set url as 'http://yoursite.com/child' and root as '/child/'
    url: http://blog.zoudongq123.cn
    root: /blog/

    # Directory
    public_dir: public/blog
```

- 2.修改hexo-deployer-git源码
   > 路径 ：/node_modules/hexo-deployer-git/lib/deployer.js

修改` 第21行 `左右 修改如下：
```
  // var publicDir = this.public_dir;
  var publicDir = pathFn.join(baseDir, 'public');
```

完成以上步骤差不多就OK了，开始编译
 + hexo clean   //最好先clean 因为你以前已经编译后，最后把 ` .deploy_git `文件夹删除
 + hexo g  //编译生成public 
 + hexo d  //上传文件

这样大功告成！！！！！

#### 个人网站 **每次` hexo d `都会覆盖CNAME文件** 
  >背景:通过github 手动配置` Custom domain` 每次` hexo d `之后覆盖resp， 本地每次都会覆盖resp

解决办法：首先hexo g 生成静态文件后在public 目录下创建一个文件（CNAME）然后再` hexo d ` 


#### 个人网站配置子目录后 直接访问网址 报404错误
  >因为配置了子路经  直接访问网址访问的路径是根目录 ，而根目录没有index.html文件

解决办法：在public文件夹下面创建一个` index.html ` 我的默认做法让它重定向到pages页面
```
<script language="javascript" type="text/javascript"> 
    window.location.href='/blog/';
</script>
```

