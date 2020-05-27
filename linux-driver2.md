---
title: Linux内核驱动介绍(2) 
date: 2019-08-09 13:30:26
categories: Linux_driver
tags: 
- Linux
- Driver
---
# 详情回顾
上篇讲述了用动态，静态申请设备号以及注销，今天记录剩下的部分。

# 创建设备文件
- 手工创建
- 自动创建
## 手工创建
利用mknod 函数进行手工创建
```bash
`mknod filename type major minor
```
\`
- filename  设备文件名字
- type         设备文件类型
- major       主设备号
- minor       次设备号
*例子* mknod s3c241sel c 231 0
# 重要的数据结构
在linux驱动定义里有几个比较重要额度数据结构
- Struct file
- Struct inode
- Struct file\_operations
## Struct file
 这个数据结构代表的是一个打开的文件。系统中每个打开的文件在内核空间都有一个关联的struct file 。它由内核在打开文件时创建，在文件关闭后释放。 
## Struct inode
用来记录文件的物理上的信息，例如文件大小，文件归属，文件权限等等。
## Struct file\_operations
一个函数指针的集合，定义能在设备上进行的操作。结构中的成员指向驱动中的函数，这些函数实现一个特别的操作，对于不支持的操作保留为NULL。 
### 定义如下
```c
`static struct file_operations shumg_fops = {
  .owner = THIS_MODULE,
  .ioctl  =  shumg_ioctl,//读写外其他操作
  .write  =  shumg_write,//写
  .read  =  shumg_read,//读
  .open  =  shumg_open,//打开
  .release =  shumg_release,//关闭
}；
```
\`
# 各个函数介绍
- open
- release
- read
- write
- ioctl

## open函数
```c
` int (*open) (struct inode *，struct file *)；
```
\`尽管这常常是对设备文件进行的第一个操作， 不要求驱动声明一个对应的方法。 如果这个项是 NULL，设备打开一直成功，但是你的驱动不会得到通知。

Open的作用通常初始化了设备，并表明次设备号
## release/close函数
```c
`int (*release) (struct inode *， struct file *)；
```
\`尽管这常常是对设备文件进行的第一个操作， 不要求驱动声明一个对应的方法。 如果这个项是 NULL，设备打开一直成功，但是你的驱动不会得到通知。

Release的作用通常与Open相反。
## read函数
```c
`ssize_t write(int fd,  void *buf, size_t count);
```
\`
### 具体作用
从设备中读取数据。
### 参数
ssize\_t在位数不同的操作系统下位数没有变化。
- fd：要读取的文件
- buff： 指向数据缓存
- size\_t：大小
\*read和write的buff参数是用户空间指针。因此，它不能被内核代码直接引用，理由是:用户空间指针在内核空间时可能根本是无效的-----没有那个地址的映射
\* ### 返回值
-  返回值等于传递给 read 系统调用的count 参数，表明请求的数据传输成功。
- 返回值大于 0，但小于传递给read 系统调用的count 参数，表明部分数据传输成功，根据设备的不同，导致这个问题的原因也不同，一般采取再次读取的方法。
- 返回值＝0，表示到达文件的末尾。
- 返回值为负数，表示出现错误，并且指明是何种错误。
- 在阻塞型 io 中，read 调用会出现阻塞。

## write函数
```c
`ssize_t write(int fd, const void *buf, size_t count);
```
\`
### 具体作用
向设备发送数据。
### 返回值
- 返回值等于传递给 write 系统调用的count 参数，表明请求的数据传输成功。
- 返回值大于 0，但小于传递给write 系统调用的count 参数，表明部分数据传输成功，根据设备的不同，导致这个问题的原因也不同，一般采取再次读取的方法。
- 返回值＝0，表示没有写入任何数据。标准库在调用write 时，出现这种情况会重复调用write。
- 返回值为负数，表示出现错误，并且指明是何种错误。错误号的定义参见\<linux/ errno.h\>
- 在阻塞型 io 中，write 调用会出现阻塞。
## ioctl函数
```c
`int(*ioctl) (struct inode *inode,struct file *file,unsigned int cmd,unsigned long arg ) 
```
\`
### 参数
- inode：文件物理信息
- file： 内核中的文件
- cmd：需要执行的命令
- arg：带入的参数
### 具体作用
控制设备。