# 2.3 一个示例

演示SWIG的最好方法是提供一个简单的例子。考虑如下代码：

```c
/* File : example.c */
double My_variable = 3.0;
/* Compute factorial of n */
int fact(int n) {
  if (n <= 1) return 1;
  else return n*fact(n-1);
}
/* Compute n mod m */
int my_mod(int n, int m) {
    return(n % m);
}
```

假设你想从Tcl中访问这些函数和全局变量。你可以想下面这样，从编写一个SWIG接口文件开始（按照惯例，这些文件以.i为后缀）。

## 2.3.1 SWIG接口文件

```c
/* File : example.i */
%module example
%{
/* Put headers and other declarations here */
extern double My_variable;
extern int fact(int);
extern int my_mod(int n, int m);
%}
extern double My_variable;
extern int fact(int);
extern int my_mod(int n, int m);
```

接口文件中包含了ANSI C 函数原型和变量声明。`%module`指令为SWIG生成的模块定义了模块名。 `%{ %}`块提供插入代码的位置，可以插入C语言头文件或其他C声明到生成的包装代码中。

## 2.3.2 swig命令

SWIG提供了swig命令行程序。我们可以使用它像下面这样构建一个Tcl模块\(Linux系统下\):

```shell
unix > swig -tcl example.i
unix > gcc -c -fpic example.c example_wrap.c -I/usr/local/include
unix > gcc -shared example.o example_wrap.o -o example.so
unix > tclsh
% load ./example.so
% fact 4
24
% my_mod 23 7
2
% expr $My_variable + 4.5
7.5
%
```

swig命令产生一个新的文件example\_wrap.c，他需要和example.c文件一起编译。绝大多数的操作系统和脚本语言现在都提供动态库支持。在我们的这个例子中，Tcl模块被编译成一个共享库。当加载后，Tcl可以访问SWIG接口文件中声明的函数及全局变量。看看生成的example\_wrap.c，会发现里面乱七八糟的。但是不用担心。

## 2.3.3 构建Perl5模块

现在，让我们让这些功能用于Perl5模块。不作任何更改，请键入以下内容\(Solaris系统\)：

```shell
unix > swig -perl5 example.i
unix > gcc -c example.c example_wrap.c \
-I/usr/local/lib/perl5/sun4-solaris/5.003/CORE
unix > ld -G example.o example_wrap.o -o example.so # This is for Solaris
unix > perl5.003
use example;
print example::fact(4), "\n";
print example::my_mod(23,7), "\n";
print $example::My_variable + 4.5, "\n";
<ctrl-d>
24
2
7.5
unix >
```

## 2.3.4 构建Python模块

最后，让我们构建一个Python模块\(Irix系统\):

```shell
unix > swig -python example.i
unix > gcc -c -fpic example.c example_wrap.c -I/usr/local/include/python2.0
unix > gcc -shared example.o example_wrap.o -o _example.so
unix > python
Python 2.0 (#6, Feb 21 2001, 13:29:45)
[GCC egcs-2.91.66 19990314/Linux (egcs-1.1.2 release)] on linux2
Type "copyright", "credits" or "license" for more information.
>>> import example
>>> example.fact(4)
24
>>> example.my_mod(23,7)
2
>>> example.cvar.My_variable + 4.5
7.5
```

## 2.3.5 快捷方法

对那些比较懒的程序员，他们想知道为什么我们需要提供额外的接口文件呢。结果可以证明，一般你可以不需要提供它。例如，为构建一个Perl5模块，你可以直接使用C头文件，像下面这样提供一个模块名字：

```shell
unix > swig -perl5 -module example example.h
unix > gcc -c example.c example_wrap.c \
-I/usr/local/lib/perl5/sun4-solaris/5.003/CORE
unix > ld -G example.o example_wrap.o -o example.so
unix > perl5.003
use example;
print example::fact(4), "\n";
print example::my_mod(23,7), "\n";
print $example::My_variable + 4.5, "\n";
<ctrl-d>
24
2
7.5
```



