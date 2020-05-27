---
title: python douyu 礼物统计
date: 2019-12-20 10:28:20
categories: python text
tags:
- Python
	- project
	- text
---
## 介绍
最近看有主播进行礼物的手动统计，感觉实在是有点费功夫于是尝试提升效率。首先在下实在不熟悉API，并且TCP了解的也比较肤浅，所以打算更具每次直播聊天的文本进行提取信息来统计礼物数。
## 具体实践顺序
### 1. 查看文本格式
![][image-1]
格式如上所示。由于直播方正好开了礼物感谢的方式，每一次收礼物都能进行文本抓取。
但是里面有一些比较特殊的礼物格式，例如下图。
![][image-2]
由于送礼物会出现连击的现象，在短时间内送出多个相同的礼物会出现连击的效果。其实只有六次而不是1+2+…+6次。但是又不能根据每一次连击效果进行简单的礼物数加一，因为存在一次性送礼物，并且不同的礼物类型会有不同的选项。
#### 举例
如果能赠送办卡会有如下的选项。
![][image-3]
x10，x66, x366,x520,x1314等选项。
所以存在一次叠加超过一次的情况。
### 2. 查看需要抓取的格式
根据前面几幅图，礼物送出后有如下体现。
【主播 ID】： 感谢。【送礼者ID】  送出的 【礼物类型】x【数量】
就可以根据简单的正则表达式进行抓取。但是由于是中文的原因，折腾了一个小时，包括文本log文件的转码（因为在mac上显示全是乱码），包括正则表达式中的乱码，最后还包括送礼者ID的群魔乱舞，可能包含各种特殊符号或文字。

### 3. 完整编码
完整代码如下
```python
`	import re
	import os
	import time
	import codecs
	def find\_liwu():
	filename = os.listdir('logfile')
	filepath = []
	for element in filename:
	    if len(element.split('.txt')) >= 2:
	        #print element
	        filepath.append(element)
	feiji = '飞机'
	banka = '办卡'
	i = 0
	bankadic = {}
	feijidic = {}
	g = 0
	num = [0]
	patternbanka = unicode('感谢 ','utf8',errors='ignore')+u'[\u4e00-\u9fa5\d\w]+'+unicode(' 送出的 ','utf8',errors='ignore')+unicode(banka,'utf8',errors='ignore')+'[x][\d]+'
	patternfeiji = unicode('感谢 ', 'utf8', errors='ignore') + u'[\u4e00-\u9fa5\d\w]+' + unicode(' 送出的 ', 'utf8',errors='ignore') + unicode(feiji, 'utf8', errors='ignore') + '[x][\d]+'
	while i < len(filepath):
	    path = 'logfile/' + filepath[i]
	    with open(path) as f:
	        loglines = f.read().splitlines()
	    j = 0
	
	    while j < len(loglines):
	        line = unicode(loglines[j], "utf8", errors="ignore")
	        answer = re.findall(patternbanka,line)
	        answer1 = re.findall(patternfeiji,line)
	        #print(answer)
	        if answer:
	            #print len(answer)
	            g += 1
	            #print(answer[0])
	            prev = int(num[0])
	            patternName = u'(?P<name>[\u4e00-\u9fa5\d\w]+)'+unicode(' 送出的 ','utf8',errors='ignore')
	            name = re.findall(patternName,answer[0])
	            patternNum = unicode(banka,'utf8',errors='ignore')+'[x](?P<num>[\d]+)'
	            num = re.findall(patternNum,answer[0])
	            num1 = int(num[0])
	            #print name[0], num1, prev
	            if name[0] in bankadic.keys():
	                if num1 == 1:
	                    bankadic[name[0]] = bankadic[name[0]] + 1
	                elif num1 == 10 and prev != 9:
	                    bankadic[name[0]] = bankadic[name[0]] + 10
	                elif num1 == 66 and prev != 65:
	                    bankadic[name[0]] = bankadic[name[0]] + 66
	                elif num1 == 366 and prev != 365:
	                    bankadic[name[0]] = bankadic[name[0]] + 366
	                elif num1 == 520 and prev != 519:
	                    bankadic[name[0]] = bankadic[name[0]] + 520
	                elif num1 == 1314 and prev != 1313:
	                    bankadic[name[0]] = bankadic[name[0]] + 1314
	                else:
	                    bankadic[name[0]] = bankadic[name[0]] + 1
	            else:
	                bankadic[name[0]] = num1
	        if answer1:
	            #print len(answer)
	            g += 1
	            #print(answer[0])
	            prev = int(num[0])
	            patternName = u'(?P<name>[\u4e00-\u9fa5\d\w]+)'+unicode(' 送出的 ','utf8',errors='ignore')
	            name = re.findall(patternName,answer1[0])
	            patternNum = unicode(feiji,'utf8',errors='ignore')+'[x](?P<num>[\d]+)'
	            num = re.findall(patternNum,answer1[0])
	            num1 = int(num[0])
	            #print name[0], num1, prev
	            if name[0] in feijidic.keys():
	                if num1 == 1:
	                    feijidic[name[0]] = feijidic[name[0]] + 1
	                elif num1 == 10 and prev != 9:
	                    feijidic[name[0]] = feijidic[name[0]] + 10
	                elif num1 == 66 and prev != 65:
	                    feijidic[name[0]] = feijidic[name[0]] + 66
	                elif num1 == 366 and prev != 365:
	                    feijidic[name[0]] = feijidic[name[0]] + 366
	                elif num1 == 520 and prev != 519:
	                    feijidic[name[0]] = feijidic[name[0]] + 520
	                elif num1 == 1314 and prev != 1313:
	                    feijidic[name[0]] = feijidic[name[0]] + 1314
	                else:
	                    feijidic[name[0]] = feijidic[name[0]] + 1
	            else:
	                feijidic[name[0]] = num1
	        j+= 1
	    i += 1
	#print bankadic.keys()[1], bankadic[bankadic.keys()[1]]
	#print feijidic
	f = codecs.open('liwu.txt','w','utf-8')
	#print bankadic.keys()[0]
	i = 0
	while i < (len(bankadic)):
	    if bankadic.keys()[i] in feijidic.keys():
	        f.write(bankadic.keys()[i] + ' '+ str(bankadic[bankadic.keys()[i]])+unicode(' 办卡 ','utf8',errors='ignore')+ str(feijidic[bankadic.keys()[i]])+ unicode(' 飞机','utf8',errors='ignore') +unicode(' 价值 ','utf8',errors='ignore')+ str(bankadic[bankadic.keys()[i]]*6+feijidic[bankadic.keys()[i]]*100)+ ' ''\n')
	    else:
	        f.write(bankadic.keys()[i] + ' '+ str(bankadic[bankadic.keys()[i]])+unicode(' 办卡','utf8',errors='ignore')+  unicode(' 0  飞机','utf8',errors='ignore')+unicode(' 价值 ','utf8',errors='ignore')+ str(bankadic[bankadic.keys()[i]]*6) +'\n')
	    i += 1
	i = 0
	while i < (len(feijidic)):
	    if feijidic.keys()[i] in bankadic.keys():
	        f.write(feijidic.keys()[i] + ' '+ str(bankadic[feijidic.keys()[i]])+ unicode(' 办卡 ','utf8',errors='ignore')+ str(feijidic[feijidic.keys()[i]])+unicode(' 飞机 ','utf8',errors='ignore') +unicode(' 价值 ','utf8',errors='ignore')+ str(bankadic[feijidic.keys()[i]]*6+feijidic[feijidic.keys()[i]]*100)+'\n')
	    else:
	        f.write(feijidic.keys()[i] + ' '+ unicode(' 0 办卡 ','utf8',errors='ignore')+ str(feijidic[feijidic.keys()[i]])+unicode(' 飞机','utf8',errors='ignore')  +unicode(' 价值 ','utf8',errors='ignore')+ str(feijidic[feijidic.keys()[i]]*100) +'\n')
	    i += 1
	 if __name__ == "__main__":
	start_time = time.time()
	a = find_liwu()
	#a = 'x5'
	#pattern = r'[x][\d]'
	#a2=re.findall(pattern,a)
	
	
	
	
	print("--- %s seconds ---" % (time.time() - start_time))
	filename = os.listdir('logfile')
```
`



#### 4. 结果输出
![][image-4]
这篇文本中的所有ID都能抓取到，连 xxx丶xx格式的ID也能抓取所以抓取的面应该是比较广的。

#### 5. GUI 界面
##### 介绍
由于是给新手用的，所以打算建立一个GUI互动界面。结果做完了才想起来，新手甚至没有python和GUI的库，所以应该运行不了。。。
在pyqt4中进行了UI的绘制，再通过pyqt5 BasicUI.ui -o BasicUI.py 进行转换。
##### 代码
UI源文件
```python
`	# -*- coding: utf-8 -*-
	
	# Form implementation generated from reading ui file 'liwu.ui'
	\#
	# Created by: PyQt5 UI code generator 5.7.1
	\#
	# WARNING! All changes made in this file will be lost!
	
	from PyQt5 import QtCore, QtGui, QtWidgets
	
	class Ui\_MainWindow(object):
	
	def setupUi(self, MainWindow):
	    MainWindow.setObjectName("MainWindow")
	    MainWindow.resize(800, 600)
	    self.centralwidget = QtWidgets.QWidget(MainWindow)
	    self.centralwidget.setObjectName("centralwidget")
	    self.pushButton = QtWidgets.QPushButton(self.centralwidget)
	    self.pushButton.setGeometry(QtCore.QRect(350, 240, 92, 27))
	    self.pushButton.setObjectName("pushButton")
	    MainWindow.setCentralWidget(self.centralwidget)
	    self.menubar = QtWidgets.QMenuBar(MainWindow)
	    self.menubar.setGeometry(QtCore.QRect(0, 0, 800, 25))
	    self.menubar.setObjectName("menubar")
	    MainWindow.setMenuBar(self.menubar)
	    self.statusbar = QtWidgets.QStatusBar(MainWindow)
	    self.statusbar.setObjectName("statusbar")
	    MainWindow.setStatusBar(self.statusbar)
	
	    self.retranslateUi(MainWindow)
	    QtCore.QMetaObject.connectSlotsByName(MainWindow)
	
	def retranslateUi(self, MainWindow):
	    _translate = QtCore.QCoreApplication.translate
	    MainWindow.setWindowTitle(_translate("MainWindow", "MainWindow"))
	    self.pushButton.setText(_translate("MainWindow", "liwu"))
```
`

以及UI的引用文件
```python

#coding:utf8
import re
import os
import codecs

import sys
from PyQt5.QtWidgets import QMainWindow, QApplication, QFileDialog
from liwu import *

#from pyparsing import unicode
from numpy import unicode




#######################################################
#   Author:     <Your Full Name>
#   email:      <Your Email>
#   ID:         <Your course ID, e.g. ee364j20>
#   Date:       <Start Date>
#######################################################



class Consumer(QMainWindow, Ui_MainWindow):

    def __init__(self, parent=None):
        super(Consumer, self).__init__(parent)
        self.setupUi(self)
        self.pushButton.clicked.connect(self.find_liwu)

    def find_liwu(self):
        filename = os.listdir('logfile')
        filepath = []
        for element in filename:
            if len(element.split('.txt')) >= 2:
                #print element
                filepath.append(element)
        feiji = '飞机'
        banka = '办卡'
        i = 0
        bankadic = {}
        feijidic = {}
        g = 0
        num = [0]
        patternbanka = str('感谢 ')+u'[\u4e00-\u9fa5\d\w]+'+str(' 送出的 ')+str(banka)+'[x][\d]+'
        patternfeiji = str('感谢 ') + u'[\u4e00-\u9fa5\d\w]+' + str(' 送出的 ') + str(feiji) + '[x][\d]+'
        while i < len(filepath):
            path = 'logfile/' + filepath[i]
            with open(path) as f:
                loglines = f.read().splitlines()
            j = 0

            while j < len(loglines):
                line = str(loglines[j])
                answer = re.findall(patternbanka,line)
                answer1 = re.findall(patternfeiji,line)
                #print(answer)
                if answer:
                #print len(answer)
                    g += 1
                    #print(answer[0])
                    prev = int(num[0])
                    patternName = u'(?P<name>[\u4e00-\u9fa5\d\w]+)'+str(' 送出的 ')
                    name = re.findall(patternName,answer[0])
                    patternNum = str(banka)+'[x](?P<num>[\d]+)'
                    num = re.findall(patternNum,answer[0])
                    num1 = int(num[0])
                    #print name[0], num1, prev
                    if name[0] in bankadic.keys():
                        if num1 == 1:
                            bankadic[name[0]] = bankadic[name[0]] + 1
                        elif num1 == 10 and prev != 9:
                            bankadic[name[0]] = bankadic[name[0]] + 10
                        elif num1 == 66 and prev != 65:
                            bankadic[name[0]] = bankadic[name[0]] + 66
                        elif num1 == 366 and prev != 365:
                            bankadic[name[0]] = bankadic[name[0]] + 366
                        elif num1 == 520 and prev != 519:
                            bankadic[name[0]] = bankadic[name[0]] + 520
                        elif num1 == 1314 and prev != 1313:
                            bankadic[name[0]] = bankadic[name[0]] + 1314
                        else:
                            bankadic[name[0]] = bankadic[name[0]] + 1
                    else:
                        bankadic[name[0]] = num1
                if answer1:
                    #print len(answer)
                    g += 1
                    #print(answer[0])
                    prev = int(num[0])
                    patternName = u'(?P<name>[\u4e00-\u9fa5\d\w]+)'+str(' 送出的 ')
                    name = re.findall(patternName,answer1[0])
                    patternNum = str(feiji)+'[x](?P<num>[\d]+)'
                    num = re.findall(patternNum,answer1[0])
                    num1 = int(num[0])
                    #print(name,num)
                    #print name[0], num1, prev
                    if name[0] in feijidic.keys():
                        if num1 == 1:
                            feijidic[name[0]] = feijidic[name[0]] + 1
                        elif num1 == 10 and prev != 9:
                            feijidic[name[0]] = feijidic[name[0]] + 10
                        elif num1 == 66 and prev != 65:
                            feijidic[name[0]] = feijidic[name[0]] + 66
                        elif num1 == 366 and prev != 365:
                            feijidic[name[0]] = feijidic[name[0]] + 366
                        elif num1 == 520 and prev != 519:
                            feijidic[name[0]] = feijidic[name[0]] + 520
                        elif num1 == 1314 and prev != 1313:
                            feijidic[name[0]] = feijidic[name[0]] + 1314
                        else:
                            feijidic[name[0]] = feijidic[name[0]] + 1
                    else:
                        feijidic[name[0]] = num1
                j+= 1
            i += 1
        #print(bankadic.keys()[1], bankadic[bankadic.keys()[1]])
        print(feijidic)
        f = codecs.open('liwu.txt','w','utf-8')
        #print bankadic.keys()[0]
        i = 0
        while i < (len(bankadic)):
            if list(bankadic.keys())[i] in feijidic.keys():
                f.write(list(bankadic.keys())[i] + ' '+ str(bankadic[list(bankadic.keys())[i]])+str(' 办卡 ')+ str(feijidic[list(bankadic.keys())[i]])+ str(' 飞机') +'\n')
            else:
                f.write(list(bankadic.keys())[i] + ' '+ str(bankadic[list(bankadic.keys())[i]])+str(' 办卡')+  str(' 0  飞机') +'\n')
            i += 1
        i = 0
        while i < (len(feijidic)):
            if list(feijidic.keys())[i] in bankadic.keys():
                f.write(list(feijidic.keys())[i] + ' '+ str(bankadic[list(feijidic.keys())[i]])+ str(' 办卡 ')+ str(feijidic[list(feijidic.keys())[i]])+str(' 飞机 ') +'\n')
            else:
                f.write(list(feijidic.keys())[i] + ' '+ str(' 0 办卡 ')+ str(feijidic[list(feijidic.keys())[i]])+str(' 飞机')  +'\n')
            i += 1

if __name__ == "__main__":
    currentApp = QApplication(sys.argv)
    currentForm = Consumer()

    currentForm.show()
    currentApp.exec_()

```


### 总结与问题
这个程序大概上能实现功能但是会遇到一下问题。
#### 1.礼物数量
如果送了9张办卡后 再送了10张办卡，可能只统计10张。因为程序猜测后面的x10为连击，因为x10前面为x9。这个问题可以用文本读取的方式不能有效解决。

可能可以用连击的时间来减少误算，但是再连击时间内发生还是无法避免。
#### 2.送礼者ID
如果送礼者ID不为汉子英文和数字的话，有大几率会出现遗漏。
#### 3.程序使用
该程序还是只能在搭建了环境下才能运行，较为麻烦。

#### 4.不同版本的python下转码的函数不通用
在3.7中只能支持str（）进行转码。encode（）好像不支持。
#### 5.时间
在处理一个文本花的时间为0.0143s，所以时间应该问题不是很大。
#### 6.市面上已经有免费的支持API的网站进行查询
所以本程序在得知这个消息的时候就不怎么打算更新了。

[image-1]:	https://res.cloudinary.com/djyodckal/image/upload/v1576809787/31576809495_.pic_hd_nhzean.png
[image-2]:	https://res.cloudinary.com/djyodckal/image/upload/v1576810007/41576809989_.pic_hd_izzax5.jpg
[image-3]:	https://res.cloudinary.com/djyodckal/image/upload/v1576810240/61576810220_.pic_xt0x06.jpg
[image-4]:	https://res.cloudinary.com/djyodckal/image/upload/v1576811179/71576811153_.pic_hd_lp9y00.jpg