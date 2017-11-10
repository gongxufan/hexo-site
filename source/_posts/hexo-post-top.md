---
layout: post
title: "如何将hexo生成的文章置顶"
date: 2017-11-10 15:54
tags: hexo
category: 随笔
description: 使用hexo -g命令生成的博客首页文章列表默认是按照文章创建日期倒序排列的,如果我们想实现置顶功能则可以通过修改生成器代码来实现。

---
关于hexo的安装和配置直接参考官方的文档：[https://hexo.io/zh-cn/docs/](https://hexo.io/zh-cn/docs/)

## 首页代码代码如何生成？

hexo使用nodejs开发的各种插件程序来实现静态网站的动态化操作，首页是通过插件hexo-generator-index进行数据处理的，在项目的node_modules下可以找到该程序。
我这边的位置是：
/home/gongxufan/project/github/hexo-site/node_modules/hexo-generator-index/lib/generator.js。其代码如下：
```javascript
'use strict';

var pagination = require('hexo-pagination');

module.exports = function(locals) {
  var config = this.config;
  var posts = locals.posts.sort(config.index_generator.order_by);
  var paginationDir = config.pagination_dir || 'page';

  var path = config.index_generator.path || '';

  return pagination(path, posts, {
    perPage: config.index_generator.per_page,
    layout: ['index', 'archive'],
    format: paginationDir + '/%d/',
    data: {
      __index: true
    }
  });
};

```
这个脚本程序引入了分页的插件，从而实现在首页的文章列表分页。其中
`var posts = locals.posts.sort(config.index_generator.order_by);`
这行代码很关键，order_by字段就是首页文章生成的顺序，这个参数在_config.yml中有定义：
```xml
index_generator:
  path: ''
  per_page: 5
  order_by: -date
```
-date表示按事件倒序，因此如果需要置顶我们可以通过自定义参数top然后在生成器中修改排序规则。

## Front-matter
我们可以在Front-matter中定义top字段，设置该字段为数字类型并且数字越大则排名靠前，比如：top: 999。如果你还不知道Front-matter是什么，那就赶紧搜索一番。
> Front-matter is a block of YAML or JSON at the beginning of the file that is used to configure settings for your writings. Front-matter is terminated by three dashes when written in YAML or three semicolons when written in JSON.

简单的说就是我们在用md编辑器写文章的时候头部设置的meta信息。

## 修改排序
直接上代码：
```javascript
  //根据Front-matter定义的top值排序
     var posts = locals.posts.data.sort(function (a, b) {
         //两个post都定义了top
         if (a.top && b.top) {
             //按日期将降序
             if (a.top == b.top) return b.date - a.date;
             //按top排序
             else return b.top - a.top;
         }
         //定义了top的排前面
         else if (a.top && !b.top) {
             return -1;
         }
         else if (!a.top && b.top) {
             return 1;
         }
         //没有定义top就按照日期降序
         else return b.date - a.date;
     });
```
这里的逻辑也很简单，使用sort进行排序，根据文章的头部是否设置了top进行判断。如果没有设置top则默认按照发布日期进行排序。
## end
最后执行:`hexo d`重新部署即可查看效果