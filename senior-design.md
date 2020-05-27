---
title: 毕业设计
date: 2020-05-23 02:02:26
categories: Project
tags: 
- Embedded\_C
	- Python
	- project
	- hardware
---
### 起因
这个项目主要是实现了一个电子名片的功能。这是个实体电子名片的设计，由一个硅谷工程师微型计算机名片得到灵感。由于设计难度过大，对计算机组成在当时还不熟悉，所以做了较大的改变。
### 介绍
设计最初打算用micro-python做一个USB接口的优盘，然后插入电脑时，能自动弹开一个像本博客一样的网页。但是由于太过简单，而且多数电脑对自动打开程序，类似病毒的脚本的行为会进行安全措施，这个毕业设计在一开始险些被砍掉。
最后改成了如下的需求才给予通过。

该电子名片在不接入电脑时，能显示所有者的姓名，地址，电话号码，邮箱，地址等文本内容，同时能显示个人头像或者微信二维码等图像。

当手机靠近该电子名片，并打开任意NFC app能接受预先储存于电子名片内的信息，比如个人简历的PDF或者个人网站域名。

当电子名片接入电脑时，能出现像正常U盘一样在电脑上访问的磁盘，里面的文件都是只读的权限，同时不能复制，保护用户的个人隐私，并且保证用户的所有权。只有用户或者访问者打开U盘中的UI程序输入正确的密码，才给予写和复制的权限。密码保存在原地，但是被加密。同时改变电子名片显示的信息和图片均可以通过该UI程序进行修改，同时NFC需要发送的内容也可以进行修改。当保存键按下时，信息会自动更新。
#### 示意图
![][image-1]
### 组成部分
大容量usb储存：实现U盘功能，同时需要足够储存大小。用到了安卓手机的Micro-SD 卡。
黑白墨迹屏幕：显示文本和图片。
NFC：发送信息给手机。
![][image-2]
#### 结构图
![][image-3]
### 选择这款屏幕的原因
因为当初设计的时候需要一个尽量小，且能持续显示的屏幕。所以显示了像kindle一样的墨迹屏，而且该屏幕不需要供电就能维持显示的状态，所以完美符合了需求，再者因为若干墨迹屏中只有这款屏幕的刷新时间在两秒以内，最终就选择了这款产品。
### 次级系统1：USB 大容量储存
![][image-4]
#### 过程
该部分用STM32cubeIDE 和STM32MUX 一起完成，打开了USB mass storage然后修改storage\_config\_if.c中的读写和获取状态方程即可。但是中间一直有个问题，USB和盘符一张无法正确读取，直到让写的方程等到能读到卡的状态后，运行指令才成功，但是需要大概一分钟电脑才能读到磁盘。
### 次级系统2：图像处理和Micro-SD卡读写
![][image-5]
![][image-6]
#### 过程
该部分需要读取已经保存在Micro-SD卡中解码的图像的数列及文本，然后改变大小，改变色彩显示模式，最后根据屏幕驱动把图像和文本在屏幕上显示出来。
### 次级系统3：UI和NFC模块
![][image-7]
![][image-8]
#### 过程
该部分需要在插入U盘后，有一个可以执行的UI程序，方便用户。同时NFC模块读取SD卡中的相应内容，储存在NFC模块的标签上。
### 次级系统4：图像解码
![][image-9]
#### 过程
该部分需要在插入U盘后，有一个可以执行的UI程序，方便用户。同时NFC模块读取SD卡中的相应内容，储存在NFC模块的标签上。
### 总结
最后nfc的步骤目前只能读取信息还不能写，和主单片机的交互的spi也没有实现。
目前就是这样，整张名片成本大概50美元，并且usb大容量储存的相应时间比较久，需要一两分钟，原因还未知。

### Github链接及英文描述
[https://github.com/Zhang-Rick/ECE49022_Senior_Design][1]
演示视频
[https://www.youtube.com/watch?v=P2ZDCQjS7qo][2]

[1]:	https://github.com/Zhang-Rick/ECE49022_Senior_Design
[2]:	https://www.youtube.com/watch?v=P2ZDCQjS7qo

[image-1]:	https://res.cloudinary.com/djyodckal/image/upload/v1590173169/sketchpng_up0mrk.png
[image-2]:	https://pic4.zhimg.com/80/v2-0d60706e016452b7901ca174755b186b_1440w.jpg
[image-3]:	https://res.cloudinary.com/djyodckal/image/upload/v1590173238/blockDia_rntrjc.png
[image-4]:	https://res.cloudinary.com/djyodckal/image/upload/v1590173358/mass_storage_ocpop6.png
[image-5]:	https://res.cloudinary.com/djyodckal/image/upload/v1590173357/ImagePro_vdfpdc.png
[image-6]:	https://res.cloudinary.com/djyodckal/image/upload/v1590173539/flowchart_jonb1v.png
[image-7]:	https://res.cloudinary.com/djyodckal/image/upload/v1590173539/UI_lepqzz.png
[image-8]:	https://res.cloudinary.com/djyodckal/image/upload/v1590173539/NFC_wab78v.png
[image-9]:	https://res.cloudinary.com/djyodckal/image/upload/v1590173537/Decoder_hdstmz.png