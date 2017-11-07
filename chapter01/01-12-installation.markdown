# 

# 1.12 安装

## 1.12.1 Windows安装

请参考为Windows专门准备的章节了解如何安装SWIG，运行示例程序。Windows发布版称为swigwin，根目录中包含了预编译的swig.exe可执行程序。除此之外，它和SWIG主分发版没有任何区别。不需要另行下载其他东西了。

## 1.12.2 UNIX安装

必须使用[GNU Make](http://www.gnu.org/software/make/)构建、安装SWIG。

为构建SWIG，[PCRE](http://www.pcre.org/)必须先安装，pcre-config程序必须可用。如果你已经有PCRE的头文件和库文件，但没有pcre-config，你可以覆盖编译器和连接器的参数：设置PCRE\_LIBS和PCRE\_CFLAGS变量。如果你根本就没有PCRE，SWIG的配置脚本会提供获取它的指令。

为构建、安装SWIG，简单使用如下指令：

```shell
$ ./configure
$ make
$ make install
```

SWIG默认安装在/usr/local。使用./configure脚本的--prefix选项可以更改安装位置。例如：

```shell
$ ./configure --prefix=/home/yourname/projects
$ make
$ make install
```

> 注意：提供给--prefix的路径必须是绝对路径。不要使用shell-escape字符~引用你的主目录。如果你这么做了，SWIG将不会正常工作。

SWIG源代码根目录的INSTALL文件详细描述了如何使用configure。使用如下命令:

```shell
$ ./configure --help.
```

configure构建脚本会搜索你机器上的的TCL、Perl5、Python等其他SWIG支持的语言的库。如果你看到'not found'的消息也不要头疼，SWIG不需要这些模块才能编译或运行。configure脚本查找这些库的主要是为了让你可以直接运行Examples目录下的示例程序，而不用手工改动makefiles。

> 注意：--without-xxx选项（其中xxx表示目标语言），作用很小。它们主要是为了减少'make check'时的测试时间。SWIG可执行文件、安装的库文件目前不能配置以支持一组目标语言的子集。

SWIG过去包含一组运行时库，用于支持某些语言与多个模块进行协作。现在，这些库不再在安装阶段构建了。但是，用户可以像构建模块一样的方式来构建它们。根目录中的CHANGES文件也描述了如何构建SWIG运行时。

> 如果通过Git获取了代码，你应该在运行./configure前执行./autogen.sh。除此之外，完全构建SWIG需要安装诸多软件包。详细指令请查考[SWIG bleeding edge](http://www.swig.org/svn.html)。



