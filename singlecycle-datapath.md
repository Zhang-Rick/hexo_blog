---
title: singlecycle\_ 设计及实现4(储存需求单元)
date: 2020-03-27 01:09:02
categories: mips CPU desig
tags:
- FPGA
- verilog
- CPU
---
### 简介
这篇博文主要介绍的是基于mips的 singlecycle CPU设计及实现的第四，五个单元memory\_control和request\_control储存控制和需求控制单元。
### RTL 图像
![][image-1]
接着上次博客对control\_unit的介绍，这次要讨论的是对memory储存的控制。因为一开始并没有编写缓存，所以对储存需要处理。储存器中是根据data hit(数据命中)和instrcution hit (指令命中)来进行读取数据对。不同的命中有不同的优先级。
例如：
ihit不能先于dhit，不然

[image-1]:	https://res.cloudinary.com/djyodckal/image/upload/v1584759605/datapath_jchv0c.png