# 4.3 创建脚本语言扩展

将C/C++语言与脚本语言集成的最后一步是将你的扩展添加到脚本语言中。这里有两种主要方法去完成。比较好的技术是创建共享库构建可动态加载的扩展。另外一种方法是，将你的扩展代码添加到脚本语言的解析器中，然后重新编译脚本语言解析器。

## 4.3.1 共享库和动态加载

为创建动态库或DLL，你一般需要查看编译器和链接器的手册。部分平台的构建步骤一般如下：

```shell
# Build a shared library for Solaris
gcc -fpic -c example.c example_wrap.c -I/usr/local/include
ld -G example.o example_wrap.o -o example.so
# Build a shared library for Linux
gcc -fpic -c example.c example_wrap.c -I/usr/local/include
gcc -shared example.o example_wrap.o -o example.so
```

为使用你的共享库，可以简单地使用脚本语言相应的命令（load,import,use等）。它将导入你的模块，并允许你开始使用。例如：

```tcl
% load ./example.so
% fact 4
24
%
```

当操作C++代码时，构建共享库的过程可能更复杂点，主要是因为C++模块需要额外的代码才能正确工作。在多数机器上，你可以安装上面的过程构建共享库，但需要更改链接部分：

```shell
c++ -shared example.o example_wrap.o -o example.so
```

## 4.3.2 链接共享库

当构建共享库形式的扩展时，可能它还以来你机器上的其它共享库。为了让扩展正常工作，需要在运行时找到它们。否则你可能会得到像下面这样的错误提示：

```python
>>> import graph
Traceback (innermost last):
File "<stdin>", line 1, in ?
File "/home/sci/data1/beazley/graph/graph.py", line 2, in ?
import graphc
ImportError: 1101:/home/sci/data1/beazley/bin/python: rld: Fatal Error: cannot
successfully map soname 'libgraph.so' under any of the filenames /usr/lib/libgraph.so:/
lib/libgraph.so:/lib/cmplrs/cc/libgraph.so:/usr/lib/cmplrs/cc/libgraph.so:
>>>
```

这个错误说明SWIG创建的扩展模块依赖libgraph.so共享库，但运行时系统找不到它。为解决这个问题，你可以使用以下几种方式：

+ 链接你的扩展库，并显式告诉链接器它需要的共享库的位置。多数时间，可以通过制定特殊的链接器标志如_-R 或－rpath_等。这些都不时标注实现，因此你需要阅读链接器的man手册，查找如何设置共享库的搜索路径。
+ 将共享库和可执行文件放到共同的目录。这种技术在非UNIX平台上正确工作的必须方法。
+ 在Python程序运行前，设置UNIX环境变量_LD_LIBRARY_PATH_为共享库存放的目录。尽管这是一种简单的解决方案，但不推荐。考虑设置链接器的路径标志。

## 4.3.3 静态库

为静态链接，需要将扩展和脚本语言的解析器一起重新编译。这个过程一般包括编译一小段main程序，添加你的自定义命令到解析器，然后运行解析器。这种方法可以让你生成一个自定义的脚本语言解析器。

尽管所有的平台都支持静态链接，但这不是创建脚本语言扩展最好的技术。事实上，实际不推荐这么做，应该考虑使用共享库的方式代替。