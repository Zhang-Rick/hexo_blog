---
title: Python正则表达式两者取一的表达
date: 2019-08-02 13:59:00
categories: Python Regex
tags: 
- 正则表达式
- Python
---
# 问题描述
曾经要用正则表达式从.txt或者.dat文本中寻找mac的地址的办法
目标数据 2a:3v:5g:3k:3d:6h或者2a-3v-5g-3k-3d-6h
可能是由于是16进制，字母不允许大于f。英文字符无视大小写。
#### 一开始的错误代码
```python
`"[a-fA-F0-9](#)2[:-](#)[a-fA-F0-9](#)2[:-](#)[a-fA-F0-9](#)2[:-](#)[a-fA-F0-9](#)2[:-](#)[a-fA-F0-9](#)2[:-](#)[a-fA-F0-9](#)2"
```
\`由于这个正则表达式能抓取一些错误的例子。
##### 举例
 **输入**: 2a-3v:5g-3k:3d:6h 
**输出**: 2a-3v:5g-3k:3d:6h 
会出现":"与"-"混用的例子，这些是错误的例子。由于输出的顺序必须和文本出现的顺序一致，所以不能分开来查找，于是就做了如下的修改。
```python
`"(?:[a-fA-F0-9](#)2[:](#)[a-fA-F0-9](#)[:](#)[a-fA-F0-9](#)2[:](#)[a-fA-F0-9](#)2[:](#)[a-fA-F0-9](#)2[:](#)[a-fA-F0-9](#)2|[a-fA-F0-9](#)2[-](#)[a-fA-F0-9](#)[-](#)[a-fA-F0-9](#)2[-](#)[a-fA-F0-9](#)2[-](#)[a-fA-F0-9](#)2[-](#)[a-fA-F0-9](#)2)"
```
\`其中(? |-)表示的是两种只能有一种。要不就是全部：衔接的，要不就是全部-衔接的mac地址表达式。