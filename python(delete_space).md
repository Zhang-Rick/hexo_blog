---
title: Python读取文件去掉空格读取数据方法
date: 2019-08-02 13:59:00
categories: python text
tags: 
- 文档
- 去空格
- Python
---
读取.txt或者.dat文件然后转成list类型，方便读取数据。
![][image-1]
` list(filter(None,lines[i].split(' ')))[x]`
\`通过把空格替换成None来消除空格，x为数据的列数。

[image-1]:	https://img-blog.csdnimg.cn/20190406110524542.png