---
title: Linux驱动练习
date: 2019-09-06 13:54:51
categories: linux\_driver
tags:
- driver
- linux
---
# 简介
时隔近一个月，准备吧实习的内容在总结一下。这一次上具体的练习的过程。具体准备的是打算写一个简单  的在内核打印hello\_world的驱动。

# 过程介绍
1. 完成source文件
2. 完成Makefile
3. 加载模组，并尝试测试

# source 文件
以下是练习写的code，按照基本的几个函数构建的驱动模版。在退出这个驱动的时候运行了printk，在内核中打印了出来。代码如
```c`
static int __init hello_init_module(void)
{
    return misc_register(&hello_miscdevice);
}

static void __exit hello_cleanup_module(void)
{    
    misc_deregister(&hello_miscdevice);
    printk(KERN_ALERT”Hello__world!n”);//在内核中打印function，用printk函数
    //return 0;
}

module_init(hello_init_module);
module_exit(hello_cleanup_module);_
```
 上述代码表现在在卸载模块的时候，会运行打印函数，最后打印在内核中。具体看下面的完成Makefile
在完成了source code以后，需要完成makefile进行编辑，更具需要运行的环境，选择相对应环境的编译工具

```c`
ifneq ($(KERNELRELEASE),)

obj-m:=hello_new.o

else

#generate the path

CURRENT_PATH:=$(shell pwd)

#the absolute path

LINUX_KERNEL_PATH:=/lib/modules/$(shell uname -r)/build
#complie object
default:
    make -C $(LINUX_KERNEL_PATH) M=$(CURRENT_PATH) modules 
clean:
    make -C $(LINUX_KERNEL_PATH) M=$(CURRENT_PATH) clean
endif
```
 以上的代码是编译内核模块的makefile代码。Linux\_Kernel的地址为运行环境的内核头地址。这次编译是在unbuntu文件运行，所以就如上所示。
然后需要在终端输入如下指令。

```c`
# include <linux/module.h>
# include <linux/types.h> 
# include <linux/fs.h>
# include <linux/version.h>
# include <linux/delay.h>
# include <linux/crc32.h>
# include <linux/interrupt.h>
# include <linux/init.h>
# include <linux/miscdevice.h>
# include <linux/cdev.h> 
# include <linux/uaccess.h>
# define DEV_DRIVER_NAME        "HelloDev"
char memory[50]();

int hello_open(struct inode * inode, struct file * file)
{
    return 0;
}

int hello_close(struct inode * inode, struct file * file)
{
    return 0;
}

ssize_t hello_read(struct file *filp, char __user *buff, size_t count, loff_t *pos)
{
    if (count > strlen(memory))
    {
        count = strlen(memory);
    }
  if (count < 0)
    {
        return -EINVAL;
    }
    if (copy_to_user(buff,memory,count)){
        return EFAULT;
    }

    return count;


}

ssize_t hello_write(struct file *filp, const char __user *buff, size_t count,  loff_t *pos)
{

if (count > strlen(memory))
    {
        count = strlen(memory);
    }
  if (count < 0)
    {
        return -EINVAL;
    }
    if (copy_from_user(buff,memory,count)){
        return EFAULT;
    }
    return count;

}


ssize_t hello_ioctl(struct inode *inode, struct file *file, unsigned int cmd, unsigned long arg)
{
    return 0;
}

ssize_t unlocked_hello_ioctl(struct file *file, unsigned int cmd, unsigned long arg)
{
    return hello_ioctl(NULL, file, cmd, arg);
}

static struct file_operations hello_fops = {
    .owner            = THIS_MODULE,
# if (LINUX_VERSION_CODE < KERNEL_VERSION( 2,6,35 ))
   .ioctl     = hello_ioctl,
# else
   .unlocked_ioctl = unlocked_hello_ioctl,
# endif//版本号决定ioct的不同么，但是为什么用编译条件 不直接用if
    .open        = hello_open,
    .release    = hello_close,
    .read       = hello_read,
    .write      = hello_write
};


static struct miscdevice hello_miscdevice = {
    MISC_DYNAMIC_MINOR,//次设备号
    DEV_DRIVER_NAME,
    &hello_fops,//这个为什么是地址的格式
};//.minor,.name，.fops 可以省略是吗

static int __init hello_init_module(void)
{
    return misc_register(&hello_miscdevice);
}

static void __exit hello_cleanup_module(void)
{    
    misc_deregister(&hello_miscdevice);
    printk(KERN_ALERT”Hello__world!n”);
    //return 0;
}

module_init(hello_init_module);
module_exit(hello_cleanup_module);_
```






















