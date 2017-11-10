# 5.1 运行SWIG

可使用swig命令运行SWIG，指定选项和文件名：

```shell
swig [ options ] filename
```

这里，filename是SWIG接口文件或C/C++的头文件。下面是一组可用的选项子集。每种目标语言还定义了额外的选项。通过键入`swig -help或 swig -lang -help`可以获取全部选项。

```shell
-allegrocl Generate ALLEGROCL wrappers
-chicken Generate CHICKEN wrappers
-clisp Generate CLISP wrappers
-cffi Generate CFFI wrappers
-csharp Generate C# wrappers
-go Generate Go wrappers
-guile Generate Guile wrappers
-java Generate Java wrappers
-lua Generate Lua wrappers
-modula3 Generate Modula 3 wrappers
-mzscheme Generate Mzscheme wrappers
-ocaml Generate Ocaml wrappers
-perl Generate Perl wrappers
-php Generate PHP wrappers
-pike Generate Pike wrappers
-python Generate Python wrappers
-r Generate R (aka GNU S) wrappers
-ruby Generate Ruby wrappers
-sexp Generate Lisp S-Expressions wrappers
-tcl Generate Tcl wrappers
-uffi Generate Common Lisp / UFFI wrappers
-xml Generate XML wrappers
-c++ Enable C++ parsing
-cppext ext Change file extension of C++ generated files to ext (default is cxx, except for PHP which -Dsymbol Define a preprocessor symbol
-Fstandard Display error/warning messages in commonly used format
-Fmicrosoft Display error/warning messages in Microsoft format
-help Display all options
-Idir Add a directory to the file include path
-lfile Include a SWIG library file.
-module name Set the name of the SWIG module
-o outfile Set name of C/C++ output file to <outfile>
-oh headfile Set name of C++ output header file for directors to <headfile>
-outcurrentdir Set default output dir to current dir instead of input file's path
-outdir dir Set language specific files output directory
-pcreversion Display PCRE version information
-swiglib Show location of SWIG library
-version Show SWIG version number
```



## 5.1.1 输入格式

SWIG的输入可以使包含ANSI C/C++的声明或包含特殊的SWIG指令。多数情况下，它是一个特殊的SWIG接口文件，文件后缀为.i或.swg。某些情况下，SWIG可以直接作用域原始的头文件或源文件上。但是，这种情况不常用，后面会解释为什么不要这么做。

SWIG接口文件常见格式如下：

```c
%module mymodule
%{
#include "myheader.h"
%}
// Now list ANSI C/C++ declarations
int foo;
int bar(int x);
...
```

`%module`指令指定模块的名字。模块会在[模块介绍](#modules-ntroduction)节介绍。

在`%{...%}`块之间的一切将原封不动的拷贝到SWIG结果生成的包装文件中。本节它主要用于包含头文件和用于使包装代码编译通过的其他声明。需要重点强调的是，需要你在SWIG的输入文件中包含了声明，这些声明不会自动出现在生成的包装代码中。因此，你需要确定在`%{...%}`中包含了合适的头文件。需要注意的是`%{...%}`包含的代码，SWIG是不会解释的。`%{...%}`的语法和语义类似yacc和bison工具中输入为文件中的声明。



## 5.1.2 SWIG输出

SWIG的输出是C/C++文件，这个文件包含了用于创建扩展的所有的包装代码。根据目标语言的不同，SWIG可能还会产生其他额外的文件。默认情况下，file.i或转换输入为file_wrap.c或file_wrap.cpp（依赖于是否指定了-C++选项)。使用-o选项可以改变输出文件的名字。在某些情况下，编译器使用文件后缀决定源代码使用的语言(C，C++等)。因此，如果你想更改默认输出文件的名字，你需要使用-o选项改变SWIG生成的包装文件的后缀。例如：

```shell
$ swig -c++ -python -o example_wrap.cpp example.i
```

SWIG的输出文件一般包含用于构建目标语言扩展的所有代码。SWIG不是存根编译器，通常也不需要需要修改输出文件（如果你看看输出文件，你可能就不想这样做了）。为构建最终的扩展模块，SWIG的输出文件需要与你的剩下的C/C++代码一起编译问共享库。

对多数目标语言来说，SWIG同时还会生成代理类(proxy class）的文件。这些语言独立的特别文件默认的输出文件夹和生成的C/C++包装文件是一样的。可以使用-outdir选项修改。例如:

```shell
$ swig -c++ -python -outdir pyfiles -o cppfiles/example_wrap.cpp example.i
```

如果目录cppfiles和pyfiles存在，下面将产生:

```shell
cppfiles/example_wrap.cpp
pyfiles/example.py
```

如果指定了-outcurrentdir选项(不指定-o)，SWIG的表现和典型的C/C++编译器一样，默认输出就是当前文件夹。没有这个选项，输出文件夹就是输入文件的路径。如果-o选项和-outcurrentdir选项一起使用，-outcurrentdir被忽略，如果不指定-outdir，语言特定的文件和生成的C/C++文件输出位置相同。



## 5.1.3 注释

C/C++样式的可以出现在接口文件的任何地方。在先前版本的SWIG中，注释被用于生成文档文件。但是，这个特征现在正被修复，将会在以后的发行中重新可用。



## 5.1.4 C预处理器

像C语言预处理器一样，SWIG通过增强的C语言预处理器处理所有的输入文件。所有的标准预处理特性它都支持，包括：文件包含、条件编译和宏。但是，`#include`语句是被忽略的，除非你指定了-includeall命令行参数。禁用包含的原因是因为SWIG有时候被用于处理原始的C语言头文件。在这种情况下，通常你只是想扩展那些包含在提供头文件中的函数到扩展模块中，而不是该头文件所包含的所有定义（比如，系统头文件、C语言库等)。

还应该注意的是，SWIG预处理器会跳过所有包含在`%{...%}`块中的代码。除此之外，预处理器还提供了一些增强宏，用起来比常规C语言预处理器更强大。这些扩展在[预处理](#preprocessor)章节会描述的。



## 5.1.5 SWIG指令

绝大多数的SWIG操作是通过特殊的指令控制的，它们总是带有_\$_前置符号，主要目的是为了和C语言的声明区分开来。这些指令被用于给SWIG提供线索或改变SWIG解释器的行为。

因为SWIG指令不是合法的C语言语法，通常不可能在包含文件中书写。但是，SWIG指令可通过使用C语言的条件编译写在包含文件中，比如：

```c
/* header.h --- Some header file */
/* SWIG directives -- only seen if SWIG is running */
#ifdef SWIG
%module foo
#endif
```

`SWIG`是一个特殊的预处理器符号，它在输入文件被解释时是SWIG程序定义的。

## 5.1.6 解释器的局限

尽管SWIG可以解释多数的C/C++声明，但它没有完全实现C/C++解释器。大部分的限制是关于非常复杂的类型声明和某些高级C++特性。特别是一下这些当前不支持的特性:

+ 非传统的类型声明。例如，SWIG不支持下面这样的声明(尽管在C中时合法的):

  ```c++
  /* Non-conventional placement of storage specifier (extern) */
  const int extern Number;
  /* Extra declarator grouping */
  Matrix (foo); // A global variable
  /* Extra declarator grouping in parameters */
  void bar(Spam (Grok)(Doh));
  ```

  实践中，很少（如果真有的话）有C程序员真得像这样写代码，因为这样的代码在编程书籍里几乎没被提及过。但是，如果你感到困惑，这样做了，你会发现SWIG不能正常工作（尽管我也不知道你为什么还是要这么做）。

+ 不推荐直接在C++源代码文件上运行SWIG。一般方法是给SWIG提供带有C++定义和声明的头文件。主要原因是，如果SWIG遇到了带作用域的声明或定义时（在C++源文件中很常见），它会直接忽略，除非对符号的声明进行了早期解析。例如：

  ```c++
  /* bar not wrapped unless foo has been defined and
  the declaration of bar within foo has already been parsed */
  int foo::bar(int) {
    ... whatever ...
  }
  ```

+ 某些C++高级特性如嵌套类也支持的不全面。请查看C++[嵌套类](#nested-classes)章节获取更多信息。

当遇到解释性错误时，可以使用条件编译跳过相关代码。例如：

```c
#ifndef SWIG
... some bad declarations ...
#endif
```

或者，你可以直接从接口文件中将其删除。

SWIG不提供对C++解释器的完全支持的原因之一是，它被设计用来支持不完全的规格说明、能处理宽泛的C/C++数据类型（如，即使没有类定义或数据类型透明，SWIG也可以生成接口）。不幸的是，这种方式让它很难实现C/C++解释器的某些特性，因为编译器使用类型信息帮助解释非常复杂的声明==(for the truly curious, the primary complication in the implementation is that the SWIG parser does not utilize a separate typedef-name terminal symbol as described on p. 234 of K&R)==。