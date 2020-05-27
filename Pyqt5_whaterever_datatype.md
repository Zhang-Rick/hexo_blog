---
title: Pyqt5 textbox中识别整数与小数无报错办法
date: 2019-08-02 13:59:00
categories: Python Regex
tags: 
- Python
- 正则表达式
- Pyqt5
---
要求对如下计算机进行编程。可以是任何输入，但只有整数或者小数，结果才有值。
![][image-1]
要求无论什么输入都不允许有报错。
所以我就用了正则表达式来做。

在查看了calculator.io 转换而来的.py文件的时候

```python
`self.edtNumber1 = QtWidgets.QLineEdit(self.centralwidget)
self.edtNumber1.setGeometry(QtCore.QRect(70, 50, 146, 27))
self.edtNumber1.setObjectName("edtNumber1")
self.edtNumber2 = QtWidgets.QLineEdit(self.centralwidget)
self.edtNumber2.setGeometry(QtCore.QRect(360, 50, 146, 27))
self.edtNumber2.setObjectName("edtNumber2")
self.lblNumber1 = QtWidgets.QLabel(self.centralwidget)
self.lblNumber1.setGeometry(QtCore.QRect(110, 30, 71, 17))
self.lblNumber1.setObjectName("lblNumber1")
self.lblNumber2 = QtWidgets.QLabel(self.centralwidget)
self.lblNumber2.setGeometry(QtCore.QRect(400, 30, 71, 17))
self.lblNumber2.setObjectName("lblNumber2")
```
\`得知self.edtNumber1为box1的输入，self.edtNumber2为box2的输入。
以下是实现计算器的最终代码
```python
`pattern ='(?P\<number\>[+-] ?[0-9] *[.] [0-9] +)'
match=re.search(pattern,self.edtNumber2.text())
if match !=None and match["number"]  == self.edtNumber2.text():
number2 = float(match["number"] )
pattern1 ='(?P\<number\>[+-] ?[0-9] +)'
match1=re.search(pattern1,self.edtNumber2.text())
if match == None and match1 != None and match1["number"]  == self.edtNumber2.text():
number2 = int(match1["number"] )
```
## 一开始的想法
因为要判断小数还是整数，所以先判断数据类型。但是因为用正则表达式的话，满足整数的表达式也能在小数中找到。所以要改变查找整数和小数的顺序。
### **举例**：
**输入**：12.78

```python
`pattern ='(?P\<number\>[+-] ?[0-9] *[.] [0-9] +)'
match=re.search(pattern,self.edtNumber2.text())
```
\`用着整数的正则先查找，匹配如下。
**输出**：==12==.==78==
12, 78
显然，这是一个小数而不是整数。
所以先判断小数，再根据非小数的结果判断是否有整数。
所以就有了原先的判断语句。

` if match == None and match1 != None：`

\`来找到不符合小数却符合整数的类型。
## **但是出现了另一个问题**
就是该方法无法读取正确大的结果
如果输入如下
### **举例1**：
**输入**：12.78De
**输出**：==12.78==
这不是我们想要的结果。
于是就做了如下修改。


`if match !=None and match["number"]== self.edtNumber2.text()`

\`除了满足正则表达式的例子外，不应该有其他任何字符。
### **举例2**：
**输入**：12.78De
match["number"][1] == 12.78
self.edtNumber2.text() == 12.78De
(match["number"][2] == self.edtNumber2.text()) == False
运用了这个方法后，就不再需要整数与小数先后的办法了。因为不是正确的数据类型的话，在第二个if条件就会出错。
最后下面是完整代码
```python
`import re  
import sys  
from PyQt5 import QtCore, QtGui  
from PyQt5.QtWidgets import QMainWindow, QApplication,QFileDialog
from calculator import * 
class MathConsumer(QMainWindow, Ui_MainWindow):

def __init__(self, parent=None):
 super(MathConsumer, self).__init__(parent)
 self.setupUi(self)
 self.btnCalculate.clicked.connect(self.calculate)
# self.btnCalculate.clicked.connect(performOperation())

# self.lblNumber1 =
def calculate(self):
number1 = 'E'
number2 = 'E'
# print(float(self.edtNumber1.text()))
self.edtNumber1.text()
pattern ='(?P\<number\>[+-]() ?[0-9]() *[.]() [0-9]() +)'
match=re.search(pattern,self.edtNumber1.text())
if match !=None and match["number"]()  == self.edtNumber1.text():
number1 = float(match["number"]() )
pattern1 ='(?P\<number\>[+-]() ?[0-9]() +)'
match1=re.search(pattern1,self.edtNumber1.text())
if match == None and match1 != None and match1["number"]()  == self.edtNumber1.text():
number1 = int(match1["number"]() )

pattern ='(?P\<number\>[+-]()?[0-9]() *[.]() [0-9]() +)'
match=re.search(pattern,self.edtNumber2.text())
if match !=None and match["number"]()  == self.edtNumber2.text():
number2 = float(match["number"]() )
pattern1 ='(?P\<number\>[+-]() ?[0-9]() +)'
match1=re.search(pattern1,self.edtNumber2.text())
if match == None and match1 != None and match1["number"]()  == self.edtNumber2.text():
number2 = int(match1["number"]() )

if number1 == 'E' or number2 == 'E':
self.edtResult.setText("E")
else:
if self.cboOperation.currentText() == "+":
self.edtResult.setText(str(number1 + number2))
elif self.cboOperation.currentText() == "*":
self.edtResult.setText(str(number1 * number2))
elif self.cboOperation.currentText() == "-":
self.edtResult.setText(str(number1 - number2))
elif self.cboOperation.currentText() == "/":
if number2 != 0 or number2 != 0.0:
self.edtResult.setText(str(number1 / number2))
else:
self.edtResult.setText("E")

if __name__ == "__main__":
currentApp = QApplication(sys.argv)
currentForm = MathConsumer()
currentForm.show()
currentApp.exec_()
```
`

[1]:	#
[2]:	#


[image-1]:	https://img-blog.csdnimg.cn/20190406100216180.png