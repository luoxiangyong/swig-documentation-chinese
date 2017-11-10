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

