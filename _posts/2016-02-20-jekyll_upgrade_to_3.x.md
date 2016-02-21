---
layout: post
title: jekyll升级到3.0
fullview: true
category: WEB
tags: jekyll
---


jekyll目前已升级到3.0，代码高亮也从pygments换成了rouge，github在build page时提示原来使用的pygments代码高亮无效了，所以对博客进行了升级

首先升级jekyll版本

```sh
gem update jekyll
```

然后修改配置文件`_config.yml`, 将原来的`highlighter: pygments`去掉:

```
gems: [jekyll-paginate, jekyll-gist] #增加这一行
paginate: 2 # pagination based on number of posts
paginate_path: "page:num"
exclude: ["README.md"] # files to exclude
#highlighter: rouge #删除
#safe: true  #如果为safe将导致paginate不可用
markdown: kramdown #增加这一行

#增加配置kramdown
kramdown:
  input: GFM 
  syntax_highlighter: rouge
```

设置`input: GFM` 允许使用和markdown一致的代码块语法，而不用使用`{\% highlight %}`，例如，现在代码块可以这样写：

```

`​`` html
<a href="#">Hello world</a>
`​``

```
`html`是代码块的语言，rouge支持的语言在这里[Rouge's demo site](http://rouge.jayferd.us/demo)



