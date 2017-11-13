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

`unsigned char` 和`signed char`是特例，它们被当做小的8位整数处理。一般情况下，`char`数据类型被映射为一个字符的ASCII的字符串。

`bool`数据类型被双向转换为0和1，除非目标语言提供了特定的布尔类型。

当处理大的整数时需要特别注意。大部分的脚步语言使用32位的整数，所以映射64位长整形可能导致截断错误。同样的问题发生在32位无符号整数上，会出现大的负数。作为经验法则，`int`和所有的`char` `short`数据类型都可以安全使用。对于`unsigned int`和`long`数据类型，被SWIG包装后，你需要谨慎检查，确保操作正确。

SWIG识别下面的浮点类型：

```c
float
double
```

浮点数据类型可以在两种语言间以自然格式安全映射。多数情况下用C语言的double表示。SWIG不支持很少被使用的`long double`类型。

`char`类型被映射成`NULL`结束的单个字符ASCII字符串。当在脚本语言中使用时，它显示为一个包含字符值的小字符串。当转换回C预言时，SWIG从目标语言中提取出一个字符的字符串，剥掉第一个字符作为`char`的值。因此，字符”foo“串赋值给`char`数据类型时，将获得值`f`。

`char *`类型作为NULL结束的ASCII字符串。SWIG将它们映射为目标语言中8位的字符串。当转换为C/C++时，SWIG将目标语言的字符串转换成NULL结束的字符串。默认的处理方式不允许这些字符串中包含嵌入的NULL字节。因此，`char *`类型不适用传递二进制数据。但是，可以通过定义SWIG typemap改变这个行为。参考[Typemaps](#swig-typemaps)章节了解详细信息。

现在，SWIG对Unicode和宽字符(_wchar\_t_)串提供了有限的支持。有些语言为_wchar\_t_提供了支持，但是请记住，这可能导致在不同操作系统间能以移植。这是一个微妙的话题，很多程序员都不理解，也没有以一致的方式跨语言实现。对那些提供Unicode支持的脚本语言，Unicode字符串可以以8位表示的方式访问，如UTF-8可以映射为`char *`类型（这种情况下SWIG接口可能会工作）。如果你要包装的程序使用Unicode，不能保证目标语言中使用的Unicode使用的就是操作系统的内置类型(如：UCS-2、UCS-4)。你可能需要自己写一些特殊的转换函数。

## 5.2.2 全局变量

只要可能，SWIG就会将C/C++的全局变量映射到脚本语言中。例如：

```c
%module example
double foo;
```

在脚本语言中可以如下访问：

```tcl
# Tcl
set foo [3.5] ;# Set foo to 3.5
puts $foo ;# Print the value of foo
```

```python
# Python
cvar.foo = 3.5 # Set foo to 3.5
print cvar.foo # Print value of foo
```

```perl
# Perl
$foo = 3.5; # Set foo to 3.5
print $foo,"\n"; # Print value of foo
```

```ruby
# Ruby
Module.foo = 3.5 # Set foo to 3.5
print Module.foo, "\n" # Print value of foo
```

不管什么使用在脚本语言中使用这个变量，它访问的都是底层的C变量。尽管SWIG尽最大努力让全局变量工作的像脚本语言的原生变量，但也不总能这样。例如，在Python语言中，所有的全局变量都需要通过一个叫`cvar`的特殊变量访问。在Ruby语言中，变量作为模块的属性访问。其他的语言肯能将全局变量转换为一对访问函数。例如，Java语言模块会生成一对函数：`double get_foo()`和`set_foo(double val)`来操作该变量。

最后，如果全局变量被声明为`const`，那它就只支持只读访问。

> 注意：这个行为从SWIG-1.3中引入。早期版本对`const`的处理不正确，只是将它表示为常量。



## 5.2.3 常量

可以使用`#define` 、枚举或特殊的指令`%constant`创建常量。下面的接口文件展示了一些有效的常量声明：

```c
#define I_CONST 5 // An integer constant
#define PI 3.14159 // A Floating point constant
#define S_CONST "hello world" // A string constant
#define NEWLINE '\n' // Character constant
enum boolean {NO=0, YES=1};
enum months {JAN, FEB, MAR, APR, MAY, JUN, JUL, AUG,
SEP, OCT, NOV, DEC};
%constant double BLAH = 42.37;
#define PI_4 PI/4
#define FLAGS 0x04 | 0x08 | 0x40
```

在`#define`声明中，常量的类型是通过语法推断出来的。例如，一个小数点的数字被假定为浮点。除此之外，SWIG必须能够解析使用`#define`定义的所有符号，从而定义常量。这个约束是必要的，因为`#define`还用来定义预处理器宏，而这些宏对脚本语言来说又没太大意义。例如：

```c
#define EXTERN extern
EXTERN void foo();
```

这种情况下，你可能不想重建一个叫`EXTERN`的常量（不知道它的值是多少）。一般情况下，SWIG不会对宏创建常量，除非它的值可以被预处理器完全确定。例如上面的例子中的声明:

```c++
#define F_CONST (double) 5 // A floating point constant with cast
```

使用常量表达式是可以的，但是SWIG不会评估它们，直接将其传递到输出文件中，让C编译器执行最终的评估(但SWIG还是会执行有限的类型检查)。

对于枚举，重要的是原始的枚举定义需要包含在接口文件中（通过头文件或`%{ }%`块）。SWIG只会讲需要的枚举装换到目标语言中。它需要原始的枚举定义，从而通过C编译器得到它们的正确数值。

`%constant`指令可用于更精确地根据C语言数据类型创建常量。尽管对简单数值一般不适用它，但在处理指针和其他更复杂的类型时，常常会用到它。典型情况是，当你想将原始C文件中没有定义但却要添加的常量添加到脚本语言中。

## 5.2.4 关于const的简短说明

C语言编程的一个常见混淆是声明中常量限定符的语义含义——特别是它和指针及其他类型限定符一起混合适用的时候。事实上，早期版本的SWIG对const的处理时不正确的，SWIG-1.3.7之后的版本修正了这个问题。

从SWIG-1.3开始，不管使用了多少const，所有的变量声明都被包装成全局变量。如果一个声明被声明为const，那它就包装成只读变量。为弄清楚变量是常量与否，你需要看它最右边出现的const限定符出现的位置（出现在变量名前）。如果最右边const在其他类型的限定符之后（如，指针），则该变量就是const的。否则不是。

这里有一些const声明的例子：

```c
const char a; // A constant character
char const b; // A constant character (the same)
char *const c; // A constant pointer to a character
const char *const d; // A constant pointer to a constant character
```

下面是一些非const声明的例子：

```c
const char *e; // A pointer to a constant character. The pointer
			  // may be modified.
```

这种情况下，`e`可以被改变——只有它指向的值是只读的。

请注意，对于函数中的常量参数会返回值，SWIG会忽略它们的const事实，参考[const的正确性](#swig-const-correctness)获得更对信息。

> 兼容性注释：SWIG将对const声明处理成只读变量的一个原因是因为，const变量的值在很多情况下会被修改。例如，一个库可能在它的API导出一个不鼓励修改的常量符号，当依然允许其他种类的内部机制更改。因此，程序员经常忽略这样一个事实，即像`char *const`这样的常量声明，所指向的底层数据也是可以被修改的——只有指针本省不能被修改。在嵌入式系统中，`const`声明可能引用一个只读的内存地址，如I/O设备的端口被映射的地址（值可以修改，但是硬件不支持写入端口）。比起对const限定符做一大堆特殊的限制，新的将const解释为只读的方式更简单，更匹配C/C++对const所作的实际语义的要求。如果真的像在旧版本的SWIG中创建常量，请使用`%constant`指令代替。例如:
>
> ```c
> %constant double PI = 3.14159;
> ```
>
> 或：
>
> ```c
> #ifdef SWIG
> #define const %constant
> #endif
> const double foo = 3.4;
> const double bar = 23.4;
> const int spam = 42;
> #ifdef SWIG
> #undef const
> #endif
> ...
> ```

## 5.2.5 关于`char *`的提醒

在进一步讨论之前，有一点关于`char *`的警告，现在必须提到。当将字符创从脚本语言转换到C的`char *`时，指针实际指向解析器内部的字符串数据。修改这些数据通常是不推荐的。因此，一些语言显式禁止这么做。例如，在Python中，字符串是不可修改的。如果你违反了这一点，当在现实中发布你的模块后，你可能会收到大家的指责。

问题的主要源头就是就地修改字符串。经典的例子就像下面的函数这样：

```c
char *strcat(char *s, const char *t)
```

尽管SWIG确实可以为其生成包装代码，但它的行为是未知的。事实上，这可能会导致你的程序崩溃，出现段错误或其他内存相关的问题。因为`s`引用到了目标语言的内部数据了，这些数据你是不能修改的。

最终建议：除了用作输入数据，不要依赖`char*`。但是，也要注意：你可以使用[typemaps](#swig-typemap)改变它的行为。