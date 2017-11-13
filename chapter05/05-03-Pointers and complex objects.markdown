# 5.3 指针和复杂对象

大部分的C语言程序都要操作数组、结构体及其他类型的对象。本节讨论这些数据类型的处理。

## 5.3.1 简单指针

像下面这样的原生C语言数据类型的指针:

```c
int *
double ***
char **
```

SWIG全面支持。SWIG并不是将指针指向的数据转换成目标语言的表现形式，而是包装指针本身，使它包含指针所指的实际数据和一个类型标签(type-tag)。因此，上面指针在Tcl中的表示如下：

```tcl
_10081012_p_int
_1008e124_ppp_double
_f8ac_pp_char
```

空指针用`NULL`或`0`表示，且带有类型信息。

所有指针通过SWIG的处理都变成了透明的对象。因此，根据需要指针可以通过函数返回、传递给其他C函数。唯一的不同就是没有解引用的机制，因为这需要目标语言理解底层对象的内存布局。

指针的脚本语言表示的值永远都不应该直接操作。即使这些值看起来像是十六进制的地址，实际使用时根据实际机器地址的不同数值时不一样的(例如，在little-endian小端机器上，数值表示就是反向的)。此外，SWIG一般不会将指针映射为高级对象，如关联数组或列表( 如，将`int*`转换为整形列表)。不这么做的几个原因如下：

+ C声明没有包含足够的信息将其映射为更高级的构造。例如，`int*`可能是整形数组，但如果它包含了1千万个元素的，将其转换成列表的话真不是好主意。
+ 指针所指内容关联的语义SWIG并不知道。例如，`int*`可能根本就不是只想数组的指针——可能它用来输出值。
+ 以一致的方式处理所有的指针，SWIG的实现大大简化，不易出错。




## 5.3.2 运行时指针类型检查

通过允许从脚本语言中操纵指针，扩展模块有效地绕过了C/C++编译器中的编译时类型检查。为了防止错误，类型签名被编码到所有指针值中，并用于执行运行时类型检查。这种类型检查过程是SWIG的组成部分，不能被禁用或修改（后面的章节讲到的typemap可以）。

与C一样，空指针匹配任何指针。此外，空指针可以传递给任何希望接收指针的函数。尽管空指针可能会造成崩溃，但有时也用来作为警戒值或表示失踪/空值。因此，SWIG将空指针的检查留给应用程序。



## 5.3.3 派生类型、结构体和类

对于其他的类型（结构体、类、数组等），SWIG使用如下的简单规则：

<div align="center">**其他的一切都是指针**</div>

换句话说，SWIG通过引用操作其他类型。这个模型在情理之中，因为多数的C/C++程序大量舒勇指针，SWIG可以使用已经存在的基础数据类型指针的类型检查机制。

尽管这听起来可能很复杂，但实际上很简单。假设你有像下面的接口文件：

```c++
%module fileio
FILE *fopen(char *, char *);
int fclose(FILE *);
unsigned fread(void *ptr, unsigned size, unsigned nobj, FILE *);
unsigned fwrite(void *ptr, unsigned size, unsigned nobj, FILE *);
void *malloc(int nbytes);
void free(void *);
```

在这个文件中，SWIG不知道`FILE`是什么，但是因为它使用指针表示，所以它具体是什么不重要。如果你包装这个为Python模块，你可以像下面这样使用函数：

```python
# Copy a file
def filecopy(source,target):
    f1 = fopen(source,"r")
    f2 = fopen(target,"w")
    buffer = malloc(8192)
    nbytes = fread(buffer,8192,1,f1)
    while (nbytes > 0):
        fwrite(buffer,8192,1,f2)
        nbytes = fread(buffer,8192,1,f1)
	free(buffer)
```

在这个例子中，`f1` `f2` 和`buffer`都是包含C语言指针的透明对象。它们包含什么值并不重要——-没有这方面的知识我们的程序也运行良好。

## 5.3.4 未定义类型

当SWIG遇到未定义类型是，它自动假设它是一个结构体或指针。例如，假设接口文件中包含下面的函数:

```c++
void matrix_multiply(Matrix *a, Matrix *b, Matrix *c);
```

SWIG不知道`Matrix`是什么。但是，很明显它是一个指针，所以SWIG使用通用的指针处理代码生成包装代码。

不像C/C++语言，SWIG不需要关心`Matrix`是否已经在接口文件的前面定义与否。这允许SWIG从部分或受限的信息中生成接口。只要你能传递一个透明引用，你可以不用关心`Matrix`到底是什么。

一个重要的细节需要提及，SWIG将会为接口文件中未定义的类型名字生成包装代码。但是，**所有的未指定类型内部都作为结构体或类的指针处理**。例如，考虑如下的代码:

```c
void foo(size_t num);
```

如果`size_t`为声明，SWIG会生成期望接受`size_t*`指针的包装代码（这个映射很快就会讲述了）。结果，在脚本语言中使用的使用可能就表现的比较奇怪。例如：

```python
foo(40);
TypeError: expected a _p_size_t.
```

修正这个问题唯一地方法就是使用`typedef`真确定义你的类型。



## 5.3.5 Typedef

就像在C中一样，`typedef`可以用来在SWIG中定义新的类型名字。例如：

```c
typedef unsigned int size_t;
```

在SWIG接口文件中出现的`typedef`定义不会传递到生成的包装代码中。因此，它们需要同时在包含的头文件或声明段出现：

```c
%{
/* Include in the generated wrapper file */
typedef unsigned int size_t;
%}
/* Tell SWIG about it */
typedef unsigned int size_t;
```

或者:

```c
%inline %{
typedef unsigned int size_t;
%}
```

某些情况下，你可能需要包含其他的文件来收集类型信息。例如：

```c
%module example
%import "sys/types.h"
```

在这种情况下，需要这样运行SWIG：

```shell
$ swig -I/usr/include -includeall example.i
```

需要注意的是，最终结果可能跟你想象的不同。大家都知道，系统头文件都非常复杂，可能对包含其他一些非标准的C代码扩展（例如，GCC的特殊指令）。除非您确切地指定了正确的包含目录和预处理器符号，否则可能无法正常工作（你可以试验一下）。

SWIG会跟踪`typedef`声明，并使用这些信息为运行时类型检查服务。比如，如果你使用上面的`typedef`，有下面的函数声明：

```c
void foo(unsigned int *ptr);
```

响应的包装函数可以接受`unsigned int *`和`size_t *`类型的参数。