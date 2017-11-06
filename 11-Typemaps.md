---
author:"罗祥勇"
email: "solo_lxy@126.com"
github:"https://github.com/luoxiangyong"
tags:
  - swig
  - C/C++
  - Java
  - Python
  - Ruby
---

[TOC]

# 11 Typemaps

[^luoxiangyong-swig]: 请不要随意拷贝本人的劳动成果 {{github}} 

## 11.1 介绍

可能有两个原因，您需要阅读本章：要么想自定义SWIG的行为；要么你听某些人含含糊糊的说"**typemaps**"
非常难懂的傻话，这时你可能会想"**typemaps**"到底是个什么鬼东西呢？既然这样，在开始之前我们做个
简单的澄清："**typemaps**"是SWIG提供的一种高级自定义特征(feature),通过它们我们可以直接访问底
层的(low-level)代码生成器。不仅如此，它们也是SWIG C++类型系统（该类型系统本省也是一个非凡
的主题）的一部分。一般情况下使用SWIG并不需要你理解typemaps。因此，如果你对SWIG默认情况下到
底做了什么并不是很清楚的情况下，建议你还是重新读一下前面的章节。


### 11.1.1 类型转换(Type Conversion)

在编程语言之间做数据类型的装换(conversion)或列集(marshalling)是包装代码生成器(wrapper 
code generator)的一项中非常重要的工作。对每一种c/C++的类型声明(declaration)，SWIG必须以
某种方式生成包装代码用以在语言间来回传递数据的值（value）。由于每种编程语言表达数据的方式不
同，因此不能简单将代码使用C连接器连接在一起。SWIG需要知道每种语言是如何表达数据的，并且需要
知道怎样去操作它们。

为说明问题，假设你有下面一段简单的C代码：

```c
int factorial(int n);
```

为了能从Python中访问该函数，需要使用一对Python API函数转换整形数据。例如：

```C
long PyInt_AsLong(PyObject *obj); /* Python --> C */
PyObject *PyInt_FromLong(long x); /* C --> Python */
```
第一个函数用来将输入参数Python整形对象转换为C语言的long类型。第二个函数用来将C语言的long型
数值转换回Python的整形对象。

在包装函数内部，你可能看到类似下面的下面的函数：

```c
PyObject *wrap_factorial(PyObject *self, PyObject *args) {
	int arg1;
	int result;
	PyObject *obj1;
	PyObject *resultobj;
	if (!PyArg_ParseTuple("O:factorial", &obj1)) return NULL;
	arg1 = PyInt_AsLong(obj1);
	result = factorial(arg1);
	resultobj = PyInt_FromLong(result);
	return resultobj;
}
```

SWIG支持的每一种目标语言都有类似的转换函数。比如，在Perl中，使用如下的函数：

```c
IV SvIV(SV *sv); /* Perl --> C */
void sv_setiv(SV *sv, IV val); /* C --> Perl */
```

在TCL中：

```c
int Tcl_GetLongFromObj(Tcl_Interp *interp, Tcl_Obj *obj, long *value);
Tcl_Obj *Tcl_NewIntObj(long value);
```

在这里，具体细节不是那么重要。重要的是，所有的底层类型转换都使用类似的功能函数，如果你对语言
扩展感兴趣可以阅读各种语言相关的文档，这些实验就留给你们自己去练习吧。



### 11.1.2 Typemaps

因为类型转换时代码包装生成器的中心工作，SWIG允许用户完全定义（或重定义）。为此，你可以使用特

殊的%typemap指令。例如：

```c
/* Convert from Python --> C */
%typemap(in) int {
	$1 = PyInt_AsLong($input);
}

/* Convert from C --> Python */
%typemap(out) int {
	$result = PyInt_FromLong($1);
}
```

第一次看到这样的代码，你肯定会迷迷糊糊地。但这真的是没什么大不了的。第一个typemap("in" typemap)用来从目标语言转换数值(value)到C语言。第二个typemap("out" typemap)用于从C语言转换数据到目标语言。每个typemap的内容都是一小段代码片段，直接插入SWIG生成的包装函数代码中。这些代码一般是C或C++代码，通过包装代码生成器处理后插入包装函数中。需要注意的是，某些目标语言的typemap也允许目标语言代码的插入。在这些代码中，带\$前缀的特殊变量会自动展开。这些变量只是C/C++语言变量的占位符号，经处理后会插入最终的包装代码中。\$input表示需要转换到C/C++的输入对象，$result表示包装函数要返回的对象。\$1表示C/C++变量，它的类型就是typemap申明中(这个例子中的`int`)指定的。



给一个简单的例子更好理解。如果你想包装如下的函数：

```{c}
int gcd(int x, int y);	
```

包装函数可能如下：

```{c}
PyObject *wrap_gcd(PyObject *self, PyObject *args) {
  int arg1;
  int arg2;
  int result;
  PyObject *obj1;
  PyObject *obj2;
  PyObject *resultobj;
  if (!PyArg_ParseTuple("OO:gcd", &obj1, &obj2)) return NULL;
  /* "in" typemap, argument 1 */
  {
  	arg1 = PyInt_AsLong(obj1);
  }
  /* "in" typemap, argument 2 */
  {
    arg2 = PyInt_AsLong(obj2);
  }
  result = gcd(arg1, arg2);
  /* "out" typemap, return value */
  {
    resultobj = PyInt_FromLong(result);
  }
  return resultobj;
}
```

在这段代码中，你能看到typemap是如何插入到生成的函数中的。你也可以看到特殊变量$是如何匹配特定变量名，并在包装函数中展开的。这就是typemap的全部思想，它们可以让你插入任意代码到生成的包装函数的不同地方。因为人意代码都可以插入，它可能完全改变值被改变的方式。



### 11.1.3 模式匹配(Pattern Matching)

正如名字所暗含的意思，typemap的目的就是在目标语言中映射("**map**")C语言数据类型。一旦某个C语言数据类型的typemap被定义，输入文件中所有出现的该类型都会应用其特征。例如：

```{c}
/* Convert from Perl --> C */
%typemap(in) int {
	$1 = SvIV($input);
}
...
int factorial(int n);
int gcd(int x, int y);
int count(char *s, char *t, int max);
```

匹配typemap到其相应的C语言数据类型不是简单的文本匹配。事实上，typemap全面内置(builtin)于底层的类型系统中。因此，typemap并不受typedef、namespace或其他可能会隐藏底层类型的声明的影响。例如，可能你有如下代码：

```{C}
/* Convert from Ruby--> C */
%typemap(in) int {
	$1 = NUM2INT($input);
}.
..
typedef int Integer;
namespace foo {
	typedef Integer Number;
};
int foo(int x);
int bar(Integer y);
int spam(foo::Number a, foo::Number b);
```

这种情况下，typemap依然可以应用到合适的参数上，即使typemap的类型名不总能匹配`int`。实际上，这种跟踪类型的能力是SWIG的重要部分，所有的目标语言模块都为基础类型定义了一组typemap。同时，没有必要为`typedef`定义新的typemap。



除了跟踪类型名字，typemap还可以特别指定去匹配特定的参数名。例如，你写了如下代码：

```{c}
%typemap(in) double nonnegative {
  $1 = PyFloat_AsDouble($input);
  if ($1 < 0) {
    PyErr_SetString(PyExc_ValueError, "argument must be nonnegative.");
    SWIG_fail;
  }
}
...
double sin(double x);
double cos(double x);
double sqrt(double nonnegative);
typedef double Real;
double log(Real nonnegative);
```

在对输入参数进行转换的情况下，typemap可以被定义成能处理连续参数的样式。例如：

```{c}
%typemap(in) (char *str, int len) {
  $1 = PyString_AsString($input); 	/* char *str */
  $2 = PyString_Size($input); 		/* int len */
}.
..
int count(char *str, int len, char c);
```

这种情况下，目标语言的一个输入对象被扩展成一对C语言参数。这个例子还展示了不常用的变量命名方案，$1、$2，诸如此类等。



### 11.1.4 重用typemap

Typemap一般用于指定的类型和参数名模式。但是，也可以被拷贝和复制。这样做的一种方式是想下面这样赋值：

```{c}
%typemap(in) Integer = int;
%typemap(in) (char *buffer, int size) = (char *str, int len);
```

更通用的拷贝形式是使用如下的`%apply`指令：

```{c}
%typemap(in) int {
/* Convert an integer argument */
...
}%
typemap(out) int {
/* Return an integer value */
...
}
/* Apply all of the integer typemaps to size_t */
%apply int { size_t };
```

`%apply`指令仅仅是把针对某种类型的所有typemap应用到另外一种类型上。

<!-- 注意：可以在%apply指令的{...}中使用逗号分隔的类型。 -->

需要注意是是，没有必要为某种类型的typedef拷贝新的typemap。例如，如果你有如下代码：

```{c}
typedef int size_t;
```

SWIG就已经知道应用`int`的typemap了，不需要再做其他工作。



### 11.1.5 使用typemap能干什么

使用typemap的主要目的就是为C/C++数据类型层面定义包装器的行为。当前typemap可以定位6种通用类别的问题：

- **参数处理(Argument Handing)**

  > int foo(**int x, double y, char *s**);

  + **输入参数转换("in" typemap)**
  + **重载(overloading)函数输入参数类型转换检查("typecheck" typemap)**
  + **输出产出处理("argout" typemap)**
  + **输入参数值的检查("check" typemap)**
  + **输入参数值的初始化("arginit" typemap)**
  + **默认参数("default" typemap)**
  + **输入参数资源管理("freearg" typemap)**

- **返回值处理（Return Value Handling）**

  > int **foo**(int x, double y, char *s);

  - **函数返回值转换（"out" typemap)**
  - **返回值资源管理("ret" typemap)**
  - **新分配对象的资源管理("newfree" typemap)**

- **异常处理(Exception Handling)**

  > **int** foo(int x, double y, char *s) throw(**MemoryError, IndexError**);

  + **处理C++异常说明("throw" typemap)**

- **全局变量(Global Variables)**

  > **int foo;**

  + **全局变量的赋值("varin" typemap)**
  + **返回全局变量("varout" typemap)**

- **成员变量(Member Variables)**

  > struct Foo {
  > ​	**int x[20];**
  > };


  + **对类或结构体的成员进行赋值("memberin" typemap)**

- **创建常量(Constant Creation)**

  > \#define FOO 3
  >
  > %constant int BAR = 42;
  > enum { ALE, LAGER, STOUT };

  + **常量的创建("consttab"或者"constcode" typemap)**

每个typemap我们都会做简短地描述。某些语言的模块也会定义额外的typemap。例如，Java模块就定义了一大堆typemap,用于控制Java绑定(binding)的各个方面。请参考各语言的特定文档了解细节。



### 11.1.6 Typemap不可以干什么？

Typemap不能用于给C/C++的声明定义属性(properties)。例如，比方说你有下面的声明：

```{c}
Foo *make_Foo(int n);
```

你想告诉SWIG`make_Foo(int n)`要返回一个新的分配对象（可能是为了提供更好的内存分配策略）。显然`make_Foo(int n)`不是类型`Foo *`关联的属性。因此，如果想达到这样的目的，需要使用SWIG提供的另外自定义机制(%feature)。请查看[自定义属性](#customizaton-features)章节连接详情。



Typemap也不能用于重新组织或装换参数的顺序。例如，你有如下的函数：

```{c}
void foo(int, char *);
```

你不能使用typemap来交换参数，达到如下的目的：

```python
foo("hello",3) # Reversed arguments
```

如果你想更改函数的调用规则，可以像下面一样写一个帮助函数(helper function)：

```C
%rename(foo) wrap_foo;
%inline %{
void wrap_foo(char *s, int x) {
	foo(x,s);
}
%}
```



### 11.1.7 面向切面编程的相似性

SWIG与面向切面编程（[Aspect Oriented Programming](https://en.wikipedia.org/wiki/Aspect-oriented_programming)）是相似的。Typemap中关于[AOP术语](https://en.wikipedia.org/wiki/Aspect-oriented_programming#Terminology)可以表述如下：

+ 横切关注点(cross-cutting concerns): 横切关注点就是typemap实现的功能模块，主要用于在目标语言和C/C++语言互相间列集类型。

+ 增强(Advice,也叫**通知**): typemap的代码体，只要列集需要就会执行。

+ 切点(Pointcut)：切点就是typemap生成的包装代码放置的位置。

+ 切面(Aspect)：切面就是切点和增强的组合，因此每个typemap都是切面。

  ​

SWIG的%feature也可以看做是切面。像%exception这样的特征也具备横切关注点的特征，因为它也可以包装用于添加日志或异常处理的功能。



### 11.1.8 本章剩下内容要讲什么

本章剩下的内容会给像了解如何编写typemap的人们提供详细信息。这些信息对那些想给新的目标语言编写模块的人特别重要。高级用户可以使用这些信息编写应用特定的类型转换规则。



因为typemap与底层的C++类型系统严格绑定，接下来的章节假设你熟悉一下C++基础概念：值、指针、引用、数组、类型修饰符（如：const）、结构体、命名空间、模板以及内存分配。如果你不了解的话建议去看看Kernighan and Ritchie的《The C Programming Language》和Stroustrup的《The C++ Programming Language》。



## 11.2 Typemap规则说明

本章主要描述%typemap指令的行为。

### 11.2.1 定义typemap

新的typemap使用%typemap指令定义。一般格式如下（以[...]包含的部分是可选的）：

```c
%typemap(method [, modifiers]) typelist code ;
```

*method*用于指定定义什么样的typemap，它是个简单的名字。这些名字通常像这样："in"，"out"， 或
"argout"。这些方法的目的后面会解释。

*modifiers*是一个可选的以逗号分隔，`name="value"`形式的列表。它们给typemap提供额外的信息，一般都是目标语言特有的，被称为typemap的属性(attibutes)。

*typelist*是typemap要匹配的C++数据类型模式的列表。一般形式描述如下:

```makefile
typelist : typepattern [, typepattern, typepattern, ... ] ;
typepattern : type [ (parms) ]
			| type name [ (parms) ]
			| ( typelist ) [ (parms) ]
```

每种类型模式可以是：简单类型、简单类型和参数名、多参数typemap类型的列表。除此之外，每个类型模式都可以参数化成暂存变量列表(params)。其用途后面也会简短描述。

*code*指定typemap的代码段。通常都是C/C++代码，像C\#和Java这样的静态类型目标语言，代码段可能包含目标语言代码。可以采用以下几种形式：

```makefile
code : 	 { ... }
		| " ... "
		| %{ ... %}
```

注意，预处理器会扩展{}界定符号里面的代码，另外两种格式的界定符号不做扩展，参考[预处理和typemap](#preprocessor-and-typemap)了解细节。下面是一些有效的typemap写法：

```c
/* Typemap with extra argument name */
%typemap(in) int nonnegative {
	...
}
/* Multiple types in one typemap */
%typemap(in) int, short, long {
	$1 = SvIV($input);
}
/* Typemap with modifiers */
%typemap(in,doc="integer") int "$1 = scm_to_int($input);";
/* Typemap applied to patterns of multiple arguments */
%typemap(in) (char *str, int len),
(char *buffer, int size)
{
  $1 = PyString_AsString($input);
  $2 = PyString_Size($input);
}
/* Typemap with extra pattern parameters */
%typemap(in, numinputs=0) int *output (int temp),
long *output (long temp)
{
	$1 = &temp;
}
```



### 11.2.2 Typemap作用域

Typemap一旦定义，跟在后面的所有声明都将使用这些规则。你可以在输入文件的需要的地方重新定义typemap。例如：

```c
// typemap1
%typemap(in) int {
	...
}
int fact(int); // typemap1
int gcd(int x, int y); // typemap1
// typemap2
%typemap(in) int {
	...
}
int isprime(int); // typemap2
```

对%extend特征指令，typemap的作用域规则不太一样。%extend用来给结构或类定义定义新的声明。因为如此，它使用在结构或类定义处定义的 typemap。举个例子：

```c
class Foo {
	...
};
%typemap(in) int {
	...
}
%extend Foo {
  int blah(int x); 	  // typemap has no effect. Declaration is attached to Foo which
  					// appears before the %typemap declaration.
};
```



### 11.2.3 拷贝typemap

使用赋值操作可以拷贝typemap。例如：

```c
%typemap(in) Integer = int;
```

或则：

```c
%typemap(in) Integer, Number, int32_t = int;
```

一种类型一般会有一组不同的typemap来控制。例如：

```c
%typemap(in) int { ... }
%typemap(out) int { ... }
%typemap(varin) int { ... }
%typemap(varout) int { ... }
```

为了拷贝这些typemap到新的类型，可以使用`%apply`指令。例如：

```c
%apply int { Integer }; 		// Copy all int typemaps to Integer
%apply int { Integer, Number };  // Copy all int typemaps to both Integer and Number
```

`%apply`使用与`%typemap`一样的规则，例如：

```c
%apply int *output { Integer *output }; // Typemap with name
%apply (char *buf, int len) { (char *buffer, int size) }; // Multiple arguments
```

### 11.2.4 删除typemap

要删除一个typemap，可以简单地将其代码段设为空，例如：

```c
%typemap(in) int; 				// Clears typemap for int
%typemap(in) int, long, short; 	 // Clears typemap for int, long, short
%typemap(in) int *output;
```

`%clear`指令可以清除指定类型的所有typemap，例如：

```c
%clear int; 					// Removes all types for int
%clear int *output, long *output;
```

> 因为SWIG的默认行为是使用typemap定义的，清除基础数据类型如`int`将会使该类型不可用，除非你在清除了后立马再定义一组新的typemap。

### 11.2.5 放置typemap

Typemap可以在全局作用域声明，也可以在C++命名空间、类声明等处。例如：

```c
%typemap(in) int {
	...
}

namespace std {
  class string;
      %typemap(in) string {
      ...
  }
}

class Bar {
	public:
    typedef const int & const_reference;
    	%typemap(out) const_reference {
    ...
    }
};
```

当typemap出现在命名空间或类中时，它的影响一直作用到输入文件结尾。但是，typemap的作用域是局部的。因此，这段代码：

```c
namespace std {
  class string;
    %typemap(in) string {
    ...
  }
}
```

就为`std::string`定义了typemap。你可能有如下代码：

```c
namespace std {
  class string;
  	%typemap(in) string { /* std::string */
  	...
  }
}

namespace Foo {
  class string;
    %typemap(in) string { /* Foo::string */
    ...
  }
}
```

在这个例子里，有两个完全不同的typemap应用于不同的类型(`std::string`和`Foo::string`)。

为了让作用域工作，SWIG需要知道`string`定义在特殊的命名空间。在这个例子中，可以使用`class string`前置声明达此目的。



## 11.3 模式匹配规则

本节描述C/C++类型关联typemap的模式匹配规则。实践中使用调试选项监测模式匹配规则也将会讲述。



### 11.3.1 基础匹配规则

Typemap同时使用类型和名称（参数名）进行匹配。对于给定`TYPE NAME`对，使用如下规则查找匹配。第一个找到的typemap的先被使用。

+ 精确匹配 *TYPE*和*NAME*的typemap
+ 仅仅精确匹配*TYPE*E*的typemap
+ 如果*TYPE*是C++模板类型*T < TPARMS >*,*TPARMS *为模板参数，类型从模板参数中剔出来，使用如下的规则：
  - 精确匹配 *TYPE*和*NAME*的typemap
  - 仅仅精确匹配*TYPE*E*的typemap

如果*TYPE*包含修饰符（const、volatile等），一次去掉(strip)一个修饰符形成新的类型，使用上面的规则进行匹配。左边的修饰符最先被去掉，最右边的最后被去掉。例如`int const*const`第一次被去除修饰符后变成`int *const`，接下来变成`int *`。

如果*TYPE*是数组，使用如下的转换：

+ 将所有的维度都替换为[ANY]，得到通用的数组typemap

  ​

为说明问题，假设有下面的代码：

```c
int foo(const char *s);
```

为了给`const char *s`找到合适的typemap，SWIG将搜索如下的typemap:

```c
const char *s // Exact type and name match
const char *  // Exact type match
char *s 	 // Type and name match (qualifier stripped)
char * 		 // Type match (qualifier stripped)
```

当找到多于一个的typemap时，只使用第一个匹配。下面这个例子展示了一些应用基础匹配规则的例子：

```c
%typemap(in) int *x {
	... typemap 1
}
%typemap(in) int * {
	... typemap 2
}
%typemap(in) const int *z {
	... typemap 3
}
%typemap(in) int [4] {
	... typemap 4
}
%typemap(in) int [ANY] {
	... typemap 5
}
void A(int *x); 			// int *x rule (typemap 1)
void B(int *y); 			// int * rule (typemap 2)
void C(const int *x); 		// int *x rule (typemap 1)
void D(const int *z); 		// const int *z rule (typemap 3)
void E(int x[4]); 			// int [4] rule (typemap 4)
void F(int x[1000]); 		// int [ANY] rule (typemap 5)
```

> 兼容性注释：SWIG-2.0.0引入一次剔除一个修饰符的规则。先前的版本一次将所有的修饰符都剔除了。



### 11.3.2 Typedef匹配规约(reduction)

如果使用前面的规则没有找到任何匹配，SWIG应用typedef匹配规约，然后在规约后的类型上继续使用一样的规则重复查找。为演示，假设有如下代码：

```c
%typemap(in) int {
	... typemap 1
}
typedef int Integer;
void blah(Integer x);
```

为找到`Integer x`的typemap，SWIG首先查找如下typemap:

```c
Integer x
Integer
```

没找到的话，使用`Integer -> int`规约，然后重复匹配：

```bash
int x
int --> match: typemap 1
```

即使通过typedef，两个类型是一样的，SWIG还是允许为它们分别定义不同的typemap。这个特性允许你对自己感兴趣的类型自定义单独的typemap。例如你写了如下代码：

```c
typedef double pdouble; // Positive double
// typemap 1
%typemap(in) double {
	... get a double ...
}
// typemap 2
%typemap(in) pdouble {
	... get a positive double ...
}
double sin(double x); 		// typemap 1
pdouble sqrt(pdouble x); 	// typemap 2
```

当规约类型时，一次应用一次typedef规约。匹配过程会一直进行下去，除非找到活没有更多的规约可用。

对于复杂类型，规约过程可能会生成一长串模式。考虑如下：

```c
typedef int Integer;
typedef Integer Row4[4];
void foo(Row4 rows[10]);
```

为匹配`Row4 rows[10]`参数，SWIG可能检查如下模式，直到它找到合适的匹配:

```ruby
Row4 rows[10]
Row4 [10]
Row4 rows[ANY]
Row4 [ANY]
# Reduce Row4 --> Integer[4]
Integer rows[10][4]
Integer [10][4]
Integer rows[ANY][ANY]
Integer [ANY][ANY]
# Reduce Integer --> int
int rows[10][4]
int [10][4]
int rows[ANY][ANY]
int [ANY][ANY]
```

对于像模板这样的参数化类型，情况更复杂。假设有如下的声明：

```c
typedef int Integer;
typedef foo<Integer,Integer> fooii;
void blah(fooii *x);
```

如下的typemap模式将会被搜索，用于匹配参数`fooii *x`：

```ruby
fooii *x
fooii *
# Reduce fooii --> foo<Integer,Integer>
foo<Integer,Integer> *x
foo<Integer,Integer> *
# Reduce Integer -> int
foo<int, Integer> *x
foo<int, Integer> *
# Reduce Integer -> int
foo<int, int> *x
foo<int, int> *
```

Typemap规约一般总是应用于最左边的类型。只有最左边的的不能匹配了才会向右规约。这种行为意味着你可以为`foo<int,Integer>`定义一个typemap,这样的话`foo<Integer,int>`typemap将永远不会匹配。这个技巧很少有人了解，实践中也很少有人这么干。当然，你可以使用这个技巧迷惑你的同事。

作为澄清，值得强调的是typedef匹配仅仅是typedef规约的过程，SWIG并不会搜索每一个可能的typedef。==假设声明了一个类型，它只会规约类型，不会在查找它的typedef定义==[^luoxiangyong-note-type-reduction]。例如，对于类型`Struct`，下面的typemap不会用于`aStruct`参数，因为`Struct`已经全部规约了。

```c
struct Struct {...};
typedef Struct StructTypedef;
%typemap(in) StructTypedef {
...
}
void go(Struct aStruct);
```

[^luoxiangyong-note-type-reduction]: 这段翻译的可能有点问题，大致的意思是说，一旦找到它的类型，关于该类型的其他typedef就不会再搜索了，针对这些typedef类型的定义的typemap也不会使用。



### 11.3.3 默认的typemap匹配规则

如果即使使用typedef规约，基础匹配规则也没有找到合适的匹配，SWIG就会使用默认的匹配规则查找合适的typemap。这些通用typemap基于`SWIGTYPE`基础类型。例如，指针使用`SWIGTYPE *`，参考使用`SWIGTYPE *`。更确切地说应该是，这些规则基于C++模板偏特化（template partial specialization）匹配规则，C++编译器使用这种规则查找合适的偏特化模板。这意味着匹配从一般typemap中选择最特化的版本使用。例如，当查找`int const *`时，根据规则，会在匹配`SWIGTYPE *`之前先匹配`SWIGTYPE const *`,会在匹配`SWIGTYP`之前先匹配`SWIGTYPE *`。

大多数的SWIG语言模块针对C语言的原始类型(primitive types)都定义了默认的typemap。这些定义全部都比较直接。例如，针对原始类型的值或const引用的列集可能如下编写：

```c
%typemap(in) int "... convert to int ...";
%typemap(in) short "... convert to short ...";
%typemap(in) float "... convert to float ...";
...
%typemap(in) const int & "... convert ...";
%typemap(in) const short & "... convert ...";
%typemap(in) const float & "... convert ...";
...
```

因为typemap匹配所有的typedef声明，通过值或const引用定义的任何原始类型的typedef都可以使用这些定义。绝大部分的目标语言模块同时还为char指针和char数组定义了typemap用以处理字符串，所以这样的非默认的类型也同样可以像原始类型一样使用基础的typemap，它们提供了比默认typemap更好的匹配规则。

下面是一组语言模块提供典型的默认类型，%typemap("in")的定义：

```c
%typemap(in) SWIGTYPE & { ... default reference handling ... };
%typemap(in) SWIGTYPE * { ... default pointer handling ... };
%typemap(in) SWIGTYPE *const { ... default pointer const handling ... };
%typemap(in) SWIGTYPE *const& { ... default pointer const reference handling ... };
%typemap(in) SWIGTYPE[ANY] { ... 1D fixed size arrays handlling ... };
%typemap(in) SWIGTYPE [] { ... unknown sized array handling ... };
%typemap(in) enum SWIGTYPE { ... default handling for enum values ... };
%typemap(in) const enum SWIGTYPE & { ... default handling for const enum reference values ... };
%typemap(in) SWIGTYPE (CLASS::*) { ... default pointer member handling ... };
%typemap(in) SWIGTYPE { ... simple default handling ... };
```

如果你想更改SWIG对简单指针的处理方式，你可以重新定义`SWIGTYPE *`。需要注意的是，简单默认的typemap规则用于匹配简单类型，不用用于匹配其他规则：

```c
%typemap(in) SWIGTYPE { ... simple default handling ... }
```

这个typemap非常重要，因为当调用或放回值类型时就会触发它。例如，如果你有如下声明：

```c
double dot_product(Vector a, Vector b);
```

`Vector`类型就会匹配`SWIGTYPE`。`SWIGTYPE`的默认实现就是讲值类型转换为指针类型（前面的章节讲过）。

通过重新定义`SWIGTYPE`类型，可以实现其他的行为。例如，如果你清除了所有针对`SWIGTYPE`的typemap，SWIG将不能包装未知的数据类型（这些类型可能对调试来说比较重要）了。然而，你可以修改`SWIGTYPE`,将对象列集为对象而不是转换为指针。

考虑如下的typemap定义，SWIG会为enum查找最佳的匹配，代码如下：

```c
%typemap(in) const Hello & { ... }
%typemap(in) const enum SWIGTYPE & { ... }
%typemap(in) enum SWIGTYPE & { ... }
%typemap(in) SWIGTYPE & { ... }
%typemap(in) SWIGTYPE { ... }

enum Hello {};
const Hello &hi;
```

那么最上面的typemap将会被选择，不因为它最先被定义，而是因为它是被包装类型的最贴切的匹配。如果上面的类型没有定义，就会选择使用接下来的类型。

探究默认typemap的最佳方式就是查看相关目标语言的模块定义。这些typemap的定义一般放在SWIG库路径下的java.swg，csharp.swg等文件中。但是，对许多目标语言来说，这些typemap定义都隐藏在复杂的宏定义中，因此，查看这些默认typemap的比较好地方式是查看预处理器的输出，可以通过在接口文件上运行`swig -E`命令达此目的。实践中，最好的方式是通过[调试typemap的模式匹配](#debugging-typemap-pattern-matching)选项，后面会讲到。

> 兼容性注释：默认的typemap匹配规则在SWIG-2.0.0版本做了调整，将简单的匹配方案调整为当前的使用C++的类模板偏特化匹配规则。

### 11.3.4 多个参数的typemap

当指定多个参数的typemap时，它的优先级要高于但给参数的typemap。例如：

```c
%typemap(in) (char *buffer, int len) {
	// typemap 1
}

%typemap(in) char *buffer {
	// typemap 2
}

void foo(char *buffer, int len, int count);  // (char *buffer, int len)
void bar(char *buffer, int blah); 			// char *buffer
```

多个参数的typemap写匹配限制更多。所有的类型和参数都必须匹配。

### 11.3.5 同C++模板的匹配方式的对比





### 11.3.6 调试typemap的模式匹配

<span id="debugging-typemap-pattern-matching" />

有两个用于调试的命令行参数可用于调试typemap，`-debug-tmsearch`和`-debug-tmused`。





## 11.4 代码生成规则

