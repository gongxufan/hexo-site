---
layout: post
title: "hexo使用next主题统计文章字符和阅读时长"
date: 2016-11-10 21:20
tags: javascript
category: 随笔
description: 使用hexo-wordcount插件统计文章字数和阅读时长。

---
next主题已经集成了字数统计和阅读时长功能，只是默认没有开启。打开主题的配置文件_config.yml,更改以下配置：
```xml
post_wordcount:
  item_text: true
  wordcount: true
  min2read: true
  totalcount: true
  separated_meta: true
```
然后安装hexo-wordcount插件，注意这里不能安装默认的3.x版本，不然编译会报错：
```xml
ERROR Plugin load failed: hexo-wordcount
SyntaxError: Unexpected token {
    at Object.exports.runInThisContext (vm.js:53:16)
    at /home/gongxufan/project/github/hexo-site/node_modules/_hexo@3.4.0@hexo/lib/hexo/index.js:230:17
    at tryCatcher (/home/gongxufan/project/github/hexo-site/node_modules/_bluebird@3.5.1@bluebird/js/release/util.js:16:23)
    at Promise._settlePromiseFromHandler (/home/gongxufan/project/github/hexo-site/node_modules/_bluebird@3.5.1@bluebird/js/release/promise.js:512:31)
    at Promise._settlePromise (/home/gongxufan/project/github/hexo-site/node_modules/_bluebird@3.5.1@bluebird/js/release/promise.js:569:18)
    at Promise._settlePromise0 (/home/gongxufan/project/github/hexo-site/node_modules/_bluebird@3.5.1@bluebird/js/release/promise.js:614:10)
    at Promise._settlePromises (/home/gongxufan/project/github/hexo-site/node_modules/_bluebird@3.5.1@bluebird/js/release/promise.js:693:18)
    at Promise._fulfill (/home/gongxufan/project/github/hexo-site/node_modules/_bluebird@3.5.1@bluebird/js/release/promise.js:638:18)
    at Promise._resolveCallback (/home/gongxufan/project/github/hexo-site/node_modules/_bluebird@3.5.1@bluebird/js/release/promise.js:432:57)
    at Promise._settlePromiseFromHandler (/home/gongxufan/project/github/hexo-site/node_modules/_bluebird@3.5.1@bluebird/js/release/promise.js:524:17)
    at Promise._settlePromise (/home/gongxufan/project/github/hexo-site/node_modules/_bluebird@3.5.1@bluebird/js/release/promise.js:569:18)
    at Promise._settlePromise0 (/home/gongxufan/project/github/hexo-site/node_modules/_bluebird@3.5.1@bluebird/js/release/promise.js:614:10)
    at Promise._settlePromises (/home/gongxufan/project/github/hexo-site/node_modules/_bluebird@3.5.1@bluebird/js/release/promise.js:693:18)
    at Promise._fulfill (/home/gongxufan/project/github/hexo-site/node_modules/_bluebird@3.5.1@bluebird/js/release/promise.js:638:18)
    at /home/gongxufan/project/github/hexo-site/node_modules/_bluebird@3.5.1@bluebird/js/release/nodeback.js:42:21
    at /home/gongxufan/project/github/hexo-site/node_modules/_graceful-fs@4.1.11@graceful-fs/graceful-fs.js:78:16
    at FSReqWrap.readFileAfterClose [as oncomplete] (fs.js:380:3)

```
这应该是该插件和我的hexo版本不兼容，我的程序版本信息如下：
```mysql
gongxufan@gongxufan-ThinkPad-E460 ~/project/github/hexo-site $ hexo -v
hexo: 3.4.0
hexo-cli: 1.0.4
os: Linux 4.8.0-53-generic linux x64
http_parser: 2.5.0
node: 4.2.6
v8: 4.5.103.35
uv: 1.8.0
zlib: 1.2.8
ares: 1.10.1-DEV
icu: 55.1
modules: 46
openssl: 1.0.2g-fips
```
指定老的版本即可：`cnpm install hexo-wordcount@2 --save`,查看package.json：
```json
{
  "name": "hexo-site",
  "version": "0.0.0",
  "private": true,
  "hexo": {
    "version": "3.4.0"
  },
  "dependencies": {
    "hexo": "^3.2.0",
    "hexo-deployer-git": "^0.3.1",
    "hexo-deployer-heroku": "^0.1.2",
    "hexo-generator-archive": "^0.1.4",
    "hexo-generator-category": "^0.1.3",
    "hexo-generator-index": "^0.2.0",
    "hexo-generator-search": "^2.1.1",
    "hexo-generator-tag": "^0.2.0",
    "hexo-renderer-ejs": "^0.3.0",
    "hexo-renderer-marked": "^0.3.0",
    "hexo-renderer-stylus": "^0.3.1",
    "hexo-server": "^0.2.0",
    "hexo-wordcount": "^2.0.1"
  }
}

```
如果需要增加具体的描述可以修改themes/next/layout/_macro/post.swig文件，增加具体的字数和时长单位。
```html
 <span title="{{ __('post.wordcount') }}">
         {{ wordcount(post.content) }} 字
  </span>
{% if theme.post_wordcount.item_text %}
   <span class="post-meta-item-text">{{ __('post.min2read') }}≈</span>
     {% endif %}
    <span title="{{ __('post.min2read') }}">
     {{ min2read(post.content) }} 分钟
     </span>
```
最后重新编译部署即可，效果如下：
![img](/upload/images/hexo/count.png)
