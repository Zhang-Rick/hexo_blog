---
title: TCP & UDP 局域网应用
date: 2019-08-05 
categories: TCP & UDP 介绍
tags:
-  实习
-  计算机网络
-  TCP & UDP
---
# 简单介绍
一般而言，路由器的作用是把不同网段的内网外网进行连接，进行传输数据。因为绝大多数内网IP地址很可能是相同的。相对而言，一个IP地址其实是一个设备的一个标识。那如果那么多设备的IP地址一样，那服务器还怎么区分需要输送数据到那台设备呢。所以这时候就需要路由器，给需要访问外网的内网设备一个处理过的IP地址。
## 举例1
有些朋友可能在家里配置过家里的路由器，一般而言进入路由器地址进行配置的IP是192.168.1.1。所以这些地址确实会是重复的。
# 如何实现数据交互
通过今年暑假的实习，也粗略的了解了一些互联网的知识，然后搭建了一个类似一下的一个局域网网络。
![ WechatIMG5 ][image-1]
由途中可知，在LAN口这端的内网是可以直接连接外网的，可以通过ping测试是否可以从内网连接至外网。
而在WAN口这端的外网是无法直接连通内网的，其中有很多种办法。这里我用的是端口映射（TCP&UDP）来实现。
# 端口映射理解
对于本人而言，端口映射表现在外网访问一个可能不唯一的内网地址的时候（因为内网地址可能会有重复，在一个路由器下面可能还有路由器进行分配独立的内网地址，所有当只有一个ip地址的时候，可能未必能定位是哪个（自己理解可能不正确））会在输入内网ip的同时额外补充一个端口号，这个端口号具体体现在锁定那个需要信息交互的IP地址。
## 举例1
显示为192.168.1.254:4000
^                     ^
   IP 地址.           端口号
# 具体操作
## 1. 访问路由器地址进行路由器配置
### a）登陆路由器地址
一般而言，内网只有一个路由器的话，路由器IP地址一般为192.168.1.1
![][image-2]
在任意网页输入路由器IP地址进入配置路由器界面，首次访问可能需要输入路由器账号密码，默认为账号：admin，密码：admin。
### b）配置允许访问该路由器的设备
具体通过访问路由器设备的IP地址和MAC地址。（网络地址：访问网络的标签和物理地址：设备独有的标签）如下图所示，从上至下为mac的IP地址和Mac地址，在Mac上搭载的虚拟机的IP地址和Mac地址和在外网的Windows端的IP地址和Mac地址。如上所示，因为要建立路由而不是交换机，WAN口和LAN口的网段并不一样。192.168.1.x和192.168.232.x，其中232也是随机设的值。具体为外网访问静态路由器IP是输入的端口号将对于这里输入的内网IP。
![][image-3]
### c）设置外网WAN口网络设置
因为本人设置的是外网的静态IP所以需要额外设置，因为外网网段为192.168.232.x所以只需设置一个与外网设备不同但同网段的IP，如192.168.232.3而不是Windows段的192.168.232.2。当时本人设成同一个IP，然后在端口映射测试上花费了颇多时间。
![][image-4]
### d）设置端口转发
然后开始配置端口转发的功能，给出外网访问路由器的WAN口外网静态IP地址后输入相对应端口后跳转到内网哪个IP地址，然后同意TCP和UDP协议就OK了。
![][image-5]

至此，路由器上的端口转发设置就全部完成了！
## 2. 配置进行tcp数据传输
### a）选择工具
1. Windows我选择的是TCP&UDP测试工具 1.02版本）使用iperf 2.0 java版测试带宽
2. Mac我选择是app store的异米网络工具
3. linux我选择是iperf3（最后又借助telnet测试是否联通）
### b.1）进行 1 -\> 2 互联
Mac端异米网络工具
![][image-6]
Windows段TCP&UDP测试工具
![][image-7]
配置如上图所示
位于内网的mac端访问外网Windows端之间输入目标IP就可直接访问，（端口不一样好像也能访问，但是可能是只有两台设备的原因）而位于外网的设备访问内网就得输入路由器WAN口的IP地址也就是之前关于网络静态的设置IP再加上防火墙上设置的端口转移的端口号。然后在两端都有接收到信息。
### b.2）进行1-\> 3互联
这次用了iperf 2.0 java版本进行互联
![][image-8]
这次用了iperf3 ubuntu命令行
`iperf3 -c 192.169.232.2 -f -k`
上两图图是以windows外网作为服务端，所建立的连接
 *这里用iperf2和iperf3互联一定要打开jperf2中“Enable Compatibility Mode”，因为两个版本不同。*
![][image-9]
最后附上一张测试带宽的图，配上输出的文本，用python算出了最大带宽。
![][image-10]
实现代码如下：
```python 
import os      # List of  module  import  statements
import re
DataPath = os.path.expanduser('/Volumes/CENA_X64FRE')

def findMaxBandwidth(name):
filename1 = DataPath + name
with open(filename1) as f:
lines1 = f.read().splitlines()
patternIP = "[0-9](){1,3}[.]()[0-9](){1,3}[.]()[0-9](){1,3}[.]()[0-9](){1,3}"
IPs = re.findall(patternIP,lines1[5]()+lines1[6]())
patternBandWidth = "[0-9]()+[ \t]()Kbits/sec"
i = 0
bandwidthList =[]()
while i \< len(lines1):
bandwidth = re.findall(patternBandWidth,lines1[i]())
if bandwidth != []():
bandwidthList.append(int(bandwidth[0]().split(' ')[0]()))
i += 1
MaxBandwidth = max(bandwidthList)
return (IPs,MaxBandwidth)

if __name__  == "__main__":
answer = findMaxBandwidth("/TCP_WANtoLAN")
IPs = answer[0]()
MaxBandwidth = answer[1]()
print("TCP: ")
print("Local IP: " + IPs[0]() + " destinated IP: " + IPs[1]())
print("bandwidth: " + str(round(int(MaxBandwidth)/1024.0,2))+ 'M/s')
answer2 = findMaxBandwidth("/UDP_WANtoLAN")
IPs2 = answer2[0]()
MaxBandwidth2 = answer2[1]()
print("UDP: ")
print("Local IP: " + IPs2[0]() + " destinated IP: " + IPs2[1]())
print("bandwidth: " + str(round(int(MaxBandwidth2)/1024.0,2))+'M/s')

```
\`至此网络协议相关就结束了









[image-1]:	https://res.cloudinary.com/djyodckal/image/upload/v1564988402/WechatIMG5_tnqecw.jpg
[image-2]:	https://res.cloudinary.com/djyodckal/image/upload/v1564988405/WechatIMG7_njq2st.jpg
[image-3]:	https://res.cloudinary.com/djyodckal/image/upload/v1564988407/WechatIMG10_xb7gcf.jpg
[image-4]:	https://res.cloudinary.com/djyodckal/image/upload/v1564988406/WechatIMG9_ispyvc.jpg
[image-5]:	https://res.cloudinary.com/djyodckal/image/upload/v1564988413/WechatIMG11_ct8jbn.jpg
[image-6]:	https://res.cloudinary.com/djyodckal/image/upload/v1564988408/WechatIMG12_zcetsn.jpg
[image-7]:	https://res.cloudinary.com/djyodckal/image/upload/v1564988410/WechatIMG13_kg4ia3.jpg
[image-8]:	https://res.cloudinary.com/djyodckal/image/upload/v1564988412/WechatIMG16_msnbpa.jpg
[image-9]:	https://res.cloudinary.com/djyodckal/image/upload/v1564988412/WechatIMG17_cyaqh3.jpg
[image-10]:	https://res.cloudinary.com/djyodckal/image/upload/v1564988414/WechatIMG20_g6a2ov.jpg