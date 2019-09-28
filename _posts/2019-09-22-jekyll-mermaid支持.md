---
layout: post
title:  "jekyll mermaid支持"
description: jekyll mermaid支持
date:   2019-09-22 12:16:52 +0530
categories: jekyll mermaid
tags: jekyll mermaid
---

### 摘要

最近想在jekyll上集成mermaid，参考了好几篇文章，发现mermaid的js文件很大，有1M多，一旦放在github page上这速度简直无法想象，只能暂时先把这想法搁置，等有时间再搞。

### 测试

```mermaid
graph TD;
    A-->B;
    A-->C;
    B-->D;
    C-->D;
```



### 参考文献

[Embed Mermaid Charts in Jekyll without Plugin](http://kkpattern.github.io/2015/05/15/Embed-Chart-in-Jekyll.html)

[mermaid github](https://github.com/knsv/mermaid)

[Jekyll 集成 mermaid](https://lkebin.com/2018/09/18/jekyll-with-mermaid.html)
