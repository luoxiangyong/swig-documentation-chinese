# 5.4 其他实用性建议

到目前为止，本章已经介绍了包装简单接口所需要学习的几乎所有的知识。但是，有些C程序使用一些难以映射到脚本语言接口的习惯用法。本节描述其中的一些问题。

## 5.4.1 通过结构体传递值

有时C函数接受由值传递的结构参数。例如，考虑以下函数：

```c
double dot_product(Vector a, Vector b);
```

为处理这个，SWIG创建一个包装函数，将原始函数转换成使用指针的方式：

```c++
double wrap_dot_product(Vector *a, Vector *b) {
	Vector x = *a;
 	Vector y = *b;
  
	return dot_product(x,y);
}
```

在目标语言中，`dot_product()`函数接受指向`Vector`的指针而不是`Vector`。大部分情况下，这种转换是透明地，不需要去关心。

## 5.4.2 返回的是值类型

返回结构体或类数据类型值的C语言函数更难处理。考虑如下函数：

```c++
Vector cross_product(Vector v1, Vector v2);
```

这个函数像返回`Vector`，但是SWIG只支持指针。结果，SWIG会创建这样的包装代码:

```c++
Vector *wrap_cross_product(Vector *v1, Vector *v2) {
  Vector x = *v1;
  Vector y = *v2;
  Vector *result;
  result = (Vector *) malloc(sizeof(Vector));
  *(result) = cross(x,y);
  
  return result;
}
```

如果使用了-C++选项：

```c++
Vector *wrap_cross(Vector *v1, Vector *v2) {
  Vector x = *v1;
  Vector y = *v2;
  Vector *result = new Vector(cross(x,y)); // Uses default copy constructor
  
  return result;
}
```

两种情况下，SWIG都会分配新的对象，并会返回它的引用。当不再使用时，需要用户删除它们。很显然，如果你没有意识到这个隐式的内存分配的话，就不会释放对象，从而带来内存泄露的问题。应该注意到一些语言模块现在可以自动跟踪新创建的对象，并为您恢复内存。请查阅每个语言模块的文档以获得更多的详细信息。

还应该注意的是，用C++处理值的传递/返回还有一些特殊情况。例如，如果`Vector`没有定义默认构造函数，上述代码片段就不能正常工作。SWIG与C++章节有关于这个情况的更多信息。

## 5.4.3 链接结构体变量

当全局变量或类的成员变量包含结构体时，SWIG将它们处理成指针。例如，像下面这样的全局变量:

```c++
Vector unit_i;
```

映射成底层的一对set/get函数：

```c++
Vector *unit_i_get() {
	return &unit_i;
}

void unit_i_set(Vector *value) {
	unit_i = *value;
}
```

以这种方式创建的全局变量将在目标语言中视为指针。释放这样的指针是不对的。同样，C++类必须提供合适的拷贝构造函数让赋值得以正常工作。

## 5.4.4 链接到`char*`

当遇到`char*`类型的全局变量时，SWIG使用`malloc()`或`new`函数给新值分配内存。特别是当你有如下的定义:

```c
char *foo;
```

SWIG将生成如下代码：

```c++
/* C mode */
void foo_set(char *value) {
  if (foo) free(foo);
  foo = (char *) malloc(strlen(value)+1);
  strcpy(foo,value);
}
/* C++ mode. When -c++ option is used */
void foo_set(char *value) {
  if (foo) delete [] foo;
  foo = new char[strlen(value)+1];
  strcpy(foo,value);
}
```

如果这不是你想要的行为，请考虑使用`%immutable`指令将其变为只读的。或者，您可以编写一个简短的帮助函数来设置您想要的值。例如：

```c
%inline %{
void set_foo(char *value) {
	strncpy(foo,value, 50);
}
%}
```

> 注意：如果你写了像这样一个帮助函数，你需要在目标语言中调用它(表现的不像是一个变量)。例如，在Python中，你可以这样写：
>
> ```python
> >>> set_foo("Hello World")
> ```

`char *`变量的一个比较常见的错误是像这样:

```c
char *VERSION = "1.0";
```

这种情况下，变量时可读的，但是尝试更改它的值将导致段错误或通用保护异常。这是因为，SWIG通过`free`或`delete`释放旧值，但当前字符串字面量又不是通过`malloc()`或`new`申请的。为修正这样的行为，你可以标记这个变量为只读的、自定义一个typemap(第六章介绍)、或者写一个特殊的函数。还可以将它声明为一个数组:

```c
char VERSION[64] = "1.0";
```

当声明`const char *`时，SWIG依然为其生成get/set函数。但是，默认的行为不是释放先前的内容（结果可能导致内存泄露）。事实上，像这样包装代码的话你会得到警告消息：

```shell
example.i:20. Typemap warning. Setting const char * variable may leak memory
```

之所以会这样是因为`const char *`变量一般用来指向字符串字面量。例如:

```c
const char *foo = "Hello World\n";
```

因此，在这样的指针上释放指针是个坏主意。另一方面，改变指针，使其指向其他值是合法的。当设置这种类型的变量时，SWIG将会分配新的字符串(通过`malloc()`或`new`)，改变指针指向新的值。但是，重复修改值将会导致内存泄露，因为旧的值没有被释放。



## 5.4.5 数组

SWIG全面支持数组，但是它们总是被处理成指针，而不是将其映射为特殊的数组对象或目标语言的列表类型。因此，如下的声明:

```c
int foobar(int a[40]);
void grok(char *argv[]);
void transpose(double a[20][20]);
```

处理后像这样：

```c
int foobar(int *a);
void grok(char **argv);
void transpose(double (*a)[20]);
```

像C一样，SWIG不执行数组边界检查。用户需要确保指针指向合适的内存位置。

多维数组将被转换成降了一维的数组的指针。例如：

```c
int [10]; 			 	// Maps to int *
int [10][20]; 		 	// Maps to int (*)[20]
int [10][20][30]; 	 	// Maps to int (*)[20][30]
```

在C语言的类型系统中，多维数组`a[][]`与单指针`*a`或双指针`char**`**不是等价的**，注意到这个非常重要！指向数组的指针的实际值是数组起始位置的内存位置。强烈建议读者弹去C语言数据上的灰尘，重读关于数组的章节，再在SWIG中使用它们。

SWIG支持数组变量，但默认情况下是只读的。例如：

```c
int a[100][200];
```

这种情况下，读取变量`a`将放回类型`int (*)[200]`,它指向数组的第一个元素的地址`&a[0][0]`。修改`a`会导致错误。这是因为SWIG不知道如何从目标语言中拷贝数据到数组中。为解决这样的限制，你可能需要写像下面这样的帮助函数:

```c
%inline %{
void a_set(int i, int j, int val) {
	a[i][j] = val;
}
int a_get(int i, int j) {
	return a[i][j];
}
%}
```

为可变的大小和形态的数组创建动态的绑定，可在接口文件中像下面这样编写帮助函数：

```c
// Some array helpers
%inline %{
/* Create any sort of [size] array */
int *int_array(int size) {
	return (int *) malloc(size*sizeof(int));
}

/* Create a two-dimension array [size][10] */
int (*int_array_10(int size))[10] {
	return (int (*)[10]) malloc(size*10*sizeof(int));
}
%}
```

SWIG对`char`类型的数组做了特殊处理。这种情况下，目标语言的字符串可以存储到数组中。例如，有如下的声明:

```c
char pathname[256];
```

SWIG将生成set/get函数:

```c
char *pathname_get() {
	return pathname;
}
void pathname_set(char *value) {
	strncpy(pathname,value,256);
}
```

在目标语言中使用它就像访问普通的变量。



## 5.4.6 创建只读变量

使用`%immutable`指令可以创建只读变量:

```c
// File : interface.i
int a; // Can read/write
%immutable;
int b,c,d // Read only variables
%mutable;
double x,y // read/write
```

`%immutable`指令开启只读模式，直到再使用`%mutable`指令禁止。还有一种方式是单独为每个声明指定只读类型。例如：

```c
%immutable x; // Make x read-only
...
double x; // Read-only (from earlier %immutable directive)
double y; // Read-write
...
```

`%immutable`和`%mutable`指令实际上是用[`%feature`指令](#swig-feature-dicrectives)定义的：

```c++
#define %immutable %feature("immutable")
#define %mutable %feature("immutable","")
```

如果你想让所有的变量都变成只读的，只有一两个是例外的话，可以这样做:

```c
%immutable; // Make all variables read-only
%feature("immutable","0") x; // except, make x read/write
...
double x;
double y;
double z;
```

当声明变量为const类型的时候，也会创建只读变量。例如：

```c
const int foo; 					/* Read only variable */
char * const version="1.0"; 	 /* Read only variable */
```

> 兼容性注释：只读访问以前使用一对指令`%readonly`和`%readwrite`来控制。尽管这些质量依然可以工作，但会生成警告消息。可以简单的将它们替换成`%immutable`和`%mutable`指令，就可以关闭警告。不要忘记了额外的分号！



## 5.4.7 重命名与忽略声明

### 5.4.7.1 特殊标识符的简单重命名

一般情况下，经过包装后，目标语言直接使用C语言中声明的名字。但是，有时候这可能与脚本语言的关键字或函数冲突。为解决命名冲突，你可以像下面这样使用`%rename`指令:

```c
// interface.i
%rename(my_print) print;
extern void print(const char *);
%rename(foo) a_really_long_and_annoying_name;
extern int a_really_long_and_annoying_name;
```

SWIG依然会调用正确的C函数，但是这种情况下，函数`print()`在目标语言中应该使用`my_print()`进行调用。

`%rename`指令可以放在任意的位置，只要它出现在要重命名的声明之前。通用方式是像下面这样编写接口代码:

```c
// interface.i
%rename(my_print) print;
%rename(foo) a_really_long_and_annoying_name;
%include "header.h"
```

`%rename`指令将其后出现的关联名字全部重新命名。它可以应用到函数、变量、类和结构体的名字、成员函数、数据成员。例如，如果有很多C++类，都有有一个函数为`print`(在Python中时关键字)，你可以将它们统一命名为`output`：

```c
%rename(output) print; // Rename all `print' functions to `output'
```

SWIG一般不会检查，看它包装的函数是否在目标语言中定义了没。但是，如果你小心处理命名空间和模块的名字，一般情况下都可以避免这些问题。

与`%rename`指定紧密相关的指令是`%ignore`指令。`%ignore`指示SWIG忽略指定标识符的声明。例如：

```c
%ignore print; // Ignore all declarations named print
%ignore MYMACRO; // Ignore a macro
...
#define MYMACRO 123
void print(const char *);
...
```

