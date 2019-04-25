---
title: "Python 自定义包的引用"    
author:     
date: February 26, 2018    

toc:    
  depth_from: 1    
  depth_to: 6    
  ordered: true    
---

# <center>Python 自定义包的引用 </center>  


新建一个项目 importDemo,含3个python包，foldA,foldB,foldC,项目文件结构如下：   
```html
importDemo   
-- foldA
---- __init__.py
---- classA.py
-- foldB
---- __init__.py
---- classB.py
-- foldC
---- __init__.py
---- funC.py
-- __init__.py
-- Main.py
```

classA.py 文件内容如下：    
```python
class FooClassA():
    version=1
    def __init__(self,nm='classA'):
        self.name=nm
        print 'Created a class instance for',self.name

    def show_name(self):
        print 'your name is',self.name
        print 'my name is',self.__class__.__name__

    def show_ver(self):
        print self.version

    def add2me(self,x):
        return x+x
```

classB.py 文件内容如下：  
```python
class FooClassB():
    version=2
    def __init__(self,nm='classB'):
        self.name=nm
        print 'Created a class instance for',self.name

    def show_name(self):
        print 'your name is',self.name
        print 'my name is',self.__class__.__name__

    def show_ver(self):
        print self.version

    def add2me(self,x):
        return x+x
```

## foldC 中的文件引用foldA和foldB中的文件   
只有一种方式，即在foldC的文件的开头导入整个项目的路径，示例如下：    
funC.py 
```python
import sys
sys.path.append("../../importDemo")

from foldA.classA import *
from foldB.classB import *

if __name__=='__main__':
    a = FooClassA()
    a.show_name()
    b = FooClassB()
    b.show_name()
```  

## 主项目中的文件引用其他包的文件   

### 方式一   
在文件的开头导入整个项目的路径，示例如下：
Main.py
```python
import sys
sys.path.append("../../importDemo")

from foldA.classA import *
from foldB.classB import *

if __name__=='__main__':
    a = FooClassA()
    a.show_name()
    b = FooClassB()
    b.show_name()
```


### 方式二    
在每个包的__init__.py文件添加包的信息。

在 foldA 和foldB 的__init__.py 中添加
```python
 from . import *
``` 
在  importDemo 的__init__.py中添加
```python
from . import foldA
from . import foldB
```

此时，在 Main.py 中去掉导入路径的语句，文件也正常执行   
Main.py   
```python
from foldA.classA import *
from foldB.classB import *

if __name__=='__main__':
    a = FooClassA()
    a.show_name()
    b = FooClassB()
    b.show_name()
```