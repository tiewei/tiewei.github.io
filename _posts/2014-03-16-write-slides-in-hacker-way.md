---
layout: post
title: "Write Slides in Hacker Way"
tagline : "像黑客一样写 PPT"
description: "Write Slides in Hacker Way"
category: "log"
tags: ["hacker"]
---
{% include JB/setup %}

## 像黑客一样写 PPT

> OK，又是周末晚上，没有约会，只有一大瓶Shasta汽水和全是快节奏的音乐...那就研究一下程序吧。

我显然没有那个[反编译出 D-Link 后门的 geek](http://blog.jobbole.com/49959/) 那么悲催，
但是一直以来各种做 PPT 深感用 power point 做 slides 绝大多数时间都浪费在如何正确使用软件上。
近些年的胶片风格越发的简单和突出重点，并且考虑到导出 pdf 各种酷炫的动作也越来越少。在微软的工具下
关注点总是很难集中到文字本身<更多的在排版之类的无意义的操作上>，近两年从 blog 网站到纯粹通过
markdown 来写 blog 深深感受到纯文字写作的魅力，于是这个周末尝试写一个 markdown 的解释器能转换成
胶片 -  为了借《像黑客一样写博客》炒作，于是就有了这个 Blog 的题目

### 实现部分

灵感来源于 github 上关注的一个 geek 的项目 [revelator](https://github.com/mpdehaan/revelator), 利用一个
非常酷的前端项目 [reveal.js](https://github.com/hakimel/reveal.js/), 将 yaml 格式的文档转换成
像[这里示例的](http://lab.hakim.se/reveal-js/#/)一样的 web 展示。

既然 yaml 可以，作为更适合写作的 markdown ，当然也可以，于是我创建了 [md_revelator](https://github.com/TieWei/md_revelator)项目。

仔细读了一下 reveal.js 的文档，发现其在每个 section 上是支持原生的 markdown 的，于是就有了 `simple_revelator`, 也就是 **version1**

version 1 
---

* 支持从 markdown 文件生成 reveal.js 的 slides
* 用3个或多个"+"或"="来隔开每个 slide, 分隔标记的前一行和后一行必须为空行
* 在第一个 slide 设定一些用于生成页面的 metadata，如 theme, author 等
* 支持垂直切换. 由于 `reveal.js` 的胶片可以水平或垂直切换，所以当分隔符与上一个分隔符不同时，当做出现垂直切换的子页面处理

第一个版本是一个晚上写出来的，采用最简单的做法，选取在 markdown 中没有定义的"==="和"+++"来区别页面，页面用 erb 来建立模板，Metadata 的加载
实现了一个简单版本的带检查的 `k:v` loader.


v1实现了基本需要，对于每页上的内容直接将 markdown 的内容交给 reveal.js 去渲染了，只是实现了分页和一些配置信息。
好像没什么成就感，何况 ppt 里的动作好像只有一个换页保留了。于是觉得继续做 **version2**


version 2
---

* 支持从 markdown 文件生成 reveal.js 的 slides 
* 用 markdown 中会翻译成 `<hr/>` 的语法标签来分隔页面
* 在文件头设定 metadata，就和 jekyll 的处理方法一样，以 yaml 的格式书写
* 支持每页上的动作，也就是 reveal.js 中的 fragment。这里采用 `$+<style>+空格` 的标记

第二个版本需要 makrdown 解释器, 纯粹是因为 github 采用的是 redcarpet 于是也用了。花时间研究了一下 redcarpet 的 C 扩展,
发现继承`Redcarpet::Render::HTML`之后不能通过 `super` 来包装 override 的方法，于是原本打算利用 `define_method`来统一处理
fragment 的念头打消了. 于是重新实现了`:paragraph`,`:list_item`和`:header`来支持 fragment，这3个基本标签已经能满足需要。
重写`:hrule`并且通过 `:doc_header` 和 `:doc_footer` 来完成 slides 分割，再通过`preprocess`和`postprocess`就可以加载
metadata 和生成 html 了。

v2版本主要是想解决 v1版本太粗糙的原因 :(.后来发现依然没有从分析器角度上重新处理 markdown，只是从 render 上做了点小手脚.

没有实现垂直切换是因为好像适用场景不多(其实是因为 `hrule`函数没有任何传入参数能让我分析是水平还是垂直..)


### 使用部分

从<https://github.com/TieWei/md_revelator>下载下整个项目之后看看 README 文件基本就会用了。常用的命令就一个

``` bash
bin/md2reveal <input-markdown-file> <output-dir> <version>
```

在 test 下有两个版本的实例，熟悉 markdown 的应该再熟悉不过了。

[test/slide-v1.md](https://github.com/TieWei/md_revelator/blob/master/test/slide-v1.md)查看<a href="/codes/md_revelator/slide-v1.html" target="_blank">输出效果</a>

[test/slide-v2.md](https://github.com/TieWei/md_revelator/blob/master/test/slide-v2.md)查看<a href="/codes/md_revelator/slide-v2.html" target="_blank">输出效果</a>

2014-3-17 Update - 加入CDN支持，默认开启，HOST 位于 <http://cdn.staticfile.org/reveal.js/2.6/> 


### 结束语

反正我用了这个东西之后，普通难度/场合的 slides 都可以完全用 markdown 写作了，简单且高效，有兴趣的同学可以试试看.
如果有 bug 什么的还请告诉我。
