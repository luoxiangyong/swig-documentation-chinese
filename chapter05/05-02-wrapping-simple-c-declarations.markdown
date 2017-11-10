# 5.2 包装简单的C声明

SWIG包装简单C语言声明的方式与C语言程序中的声明很相似，写入创建的接口文件就行了。例如，考虑如下的代码:

```c
%module example
%inline %{
extern double sin(double x);
extern int strcmp(const char *, const char *);
extern int Foo;
%}
#define STATUS 50
#define VERSION "1.1"
```

在这个文件中，有两个函数`sin()`和`strcmp()`，一个全局变量，两个常量`STATUS` 和`VERSION`。当SWIG创建扩展模块是，这些声明可以分别通过脚本语言的函数、变量和常量。例如在Tcl中可以这样：

```tcl
% sin 3
5.2335956
% strcmp Dave Mike
-1
% puts $Foo
42
% puts $STATUS
50
% puts $VERSION
1.1
```

或者在Python中这样:

```python
>>> example.sin(3)
5.2335956
>>> example.strcmp('Dave','Mike')
-1
>>> print example.cvar.Foo
42
>>> print example.STATUS
50
>>> print example.VERSION
```

只要可能，SWIG就会创建和底层C/C++语言代码密切相配的接口。但是，因为语言间的微妙不同、运行时环境、语法等，不太可能总能这样。接下来的几节将会描述语言间映射时的诸多方面。

## 5.2.1 基础类型处理

为了创建接口，SWIG需要将C/C++语言的数据类型转化为目标语言等同的类型。一般情况下，脚本语言提供比C语言更受限制的原始类型。因此，这种转换过程涉及一定数量的类型强制。

多数的脚本语言都提供一个整形类型，它用C语言的数据类型`int`或`long`实现。下面的表显示了SWIG需要在C语言与目标语言间相互转换的所有数据类型。

```c
int
short
long
unsigned
signed
unsigned short
unsigned long
unsigned char
signed char
bool
```

当要从C中转换一个整形数值时，需要一个转换方式将它变为目标语言可识别的类型。因此，C语言的16位的短整形可能被提升为32位整形。当像反方向转换时，值又被转型变为原始的C语言类型。如果数据太大不适配，就会被默默的截断。

`unsigned char` 和`signed char`是特例，它们被当做小的8位正式处理。一般情况下，`char`数据类型被映射为一个字符的ASCII的字符串。

`bool`数据类型被双向转换为0和1，除非目标语言提供了特定的布尔类型。