# 11.1 介绍

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

```c
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

```c
int gcd(int x, int y);	
```

包装函数可能如下：

```c
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

```c
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

```c
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

```c
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

```c
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

```c
%typemap(in) Integer = int;
%typemap(in) (char *buffer, int size) = (char *str, int len);
```

更通用的拷贝形式是使用如下的`%apply`指令：

```c
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


```c
typedef int size_t;
```

SWIG就已经知道应用`int`的typemap了，不需要再做其他工作。



### 11.1.5 使用typemap能干什么

使用typemap的主要目的就是为C/C++数据类型层面定义包装器的行为。当前typemap可以定位6种通用类别的问题：

- **参数处理(Argument Handing)**

  > int foo(**int x, double y, char *s**);

  - **输入参数转换("in" typemap)**
  - **重载(overloading)函数输入参数类型转换检查("typecheck" typemap)**
  - **输出产出处理("argout" typemap)**
  - **输入参数值的检查("check" typemap)**
  - **输入参数值的初始化("arginit" typemap)**
  - **默认参数("default" typemap)**
  - **输入参数资源管理("freearg" typemap)**

- **返回值处理（Return Value Handling）**

  > int **foo**(int x, double y, char *s);

  - **函数返回值转换（"out" typemap)**
  - **返回值资源管理("ret" typemap)**
  - **新分配对象的资源管理("newfree" typemap)**

- **异常处理(Exception Handling)**

  > **int** foo(int x, double y, char *s) throw(**MemoryError, IndexError**);

  - **处理C++异常说明("throw" typemap)**

- **全局变量(Global Variables)**

  > **int foo;**

  - **全局变量的赋值("varin" typemap)**
  - **返回全局变量("varout" typemap)**

- **成员变量(Member Variables)**

  > struct Foo {
  > ​	**int x[20];**
  > };


- **对类或结构体的成员进行赋值("memberin" typemap)**


- **创建常量(Constant Creation)**

  > \#define FOO 3
  >
  > %constant int BAR = 42;
  > enum { ALE, LAGER, STOUT };

  - **常量的创建("consttab"或者"constcode" typemap)**

每个typemap我们都会做简短地描述。某些语言的模块也会定义额外的typemap。例如，Java模块就定义了一大堆typemap,用于控制Java绑定(binding)的各个方面。请参考各语言的特定文档了解细节。



### 11.1.6 Typemap不可以干什么？

Typemap不能用于给C/C++的声明定义属性(properties)。例如，比方说你有下面的声明：

```c
Foo *make_Foo(int n);
```

你想告诉SWIG`make_Foo(int n)`要返回一个新的分配对象（可能是为了提供更好的内存分配策略）。显然`make_Foo(int n)`不是类型`Foo *`关联的属性。因此，如果想达到这样的目的，需要使用SWIG提供的另外自定义机制(%feature)。请查看[自定义属性](#customizaton-features)章节连接详情。



Typemap也不能用于重新组织或装换参数的顺序。例如，你有如下的函数：

```c
void foo(int, char *);
```

你不能使用typemap来交换参数，达到如下的目的：

```python
foo("hello",3) # Reversed arguments
```

如果你想更改函数的调用规则，可以像下面一样写一个帮助函数(helper function)：

```c
%rename(foo) wrap_foo;
%inline %{
void wrap_foo(char *s, int x) {
	foo(x,s);
}
%}
```



### 11.1.7 面向切面编程的相似性

SWIG与面向切面编程（[Aspect Oriented Programming](https://en.wikipedia.org/wiki/Aspect-oriented_programming)）是相似的。Typemap中关于[AOP术语](https://en.wikipedia.org/wiki/Aspect-oriented_programming#Terminology)可以表述如下：

- 横切关注点(cross-cutting concerns): 横切关注点就是typemap实现的功能模块，主要用于在目标语言和C/C++语言互相间列集类型。

- 增强(Advice,也叫**通知**): typemap的代码体，只要列集需要就会执行。

- 切点(Pointcut)：切点就是typemap生成的包装代码放置的位置。

- 切面(Aspect)：切面就是切点和增强的组合，因此每个typemap都是切面。

  ​

SWIG的%feature也可以看做是切面。像%exception这样的特征也具备横切关注点的特征，因为它也可以包装用于添加日志或异常处理的功能。



### 11.1.8 本章剩下内容要讲什么

本章剩下的内容会给像了解如何编写typemap的人们提供详细信息。这些信息对那些想给新的目标语言编写模块的人特别重要。高级用户可以使用这些信息编写应用特定的类型转换规则。



因为typemap与底层的C++类型系统严格绑定，接下来的章节假设你熟悉一下C++基础概念：值、指针、引用、数组、类型修饰符（如：const）、结构体、命名空间、模板以及内存分配。如果你不了解的话建议去看看Kernighan and Ritchie的《The C Programming Language》和Stroustrup的《The C++ Programming Language》。