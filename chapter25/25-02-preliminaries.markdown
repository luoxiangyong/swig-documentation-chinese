# 25.2 先决条件

SWIG 1.1工作与JDK 1.1至1.4(java 2 SDK 1.4)，还可以工作与之后的版本。如果可以选择，你应该使用最近版本的JDK。SWIG的java模块可被用于Solris、Linux和各种Windows上的Sun公司的JVM上，包括Cygwin。在Kaffee JVM上使用可能会有问题，在本书写作之时，它的JVM的JNI支持还是不太全面。生成的代码在vxWorks上使用WindRiver的PJava 3.1也是可以工作的。想知道生成代码是否工作最好的方式是，在你使用的操作系统和JDK上运行SWIG自带的测试程序。在Unix系统上，安装完SWIG后，从SWIG根目录运行`make -k check`即可。

Java语言模块需要你的系统支持共享库和动态链接。这是加载JNI代码到你的系统的通用方法，应该大部分的系统都支持。

Android使用了JNI，也支持SWIG生成的代码。如果你对此感兴趣，请[Android](#swig-android)相关章了解详情。

## 25.2.1 运行SWIG

假设你像下面这样定义了一个SWIG模块：

```c++
/* File: example.i */
%module test
%{
#include "stuff.h"
%}
int fact(int n);
```

为创建java模块，带`-java`选项运行SWIG:

```shell
%swig -java example.i
```

如果使用的是C++，还要添加`-c++`选项：

```shell
$ swig -c++ -java example.i
```

这将创建两种不同的文件，一个名为`example_wrap.c`或``example_wrap.cxx`的C/C++源码文件和一些Java文件。生成的C/C++源文件包含JNI包装代码，需要和剩下的C/C++代码一起编译。

包装文件的名字从输入文件中得来。例如，如果输入文件是example.i，包装文件的名字就是`example_wrap.c`。像更改的话，可使用`-o`选项。还可以使用`-outdir`选项更改生成的java文件的输出目录。

使用`%module`指令指定的模块名字决定了生成的类的名字，后面详述。注意，模块名字默认情况下不定义java的包，生成的类没有指定java包。`-package`选项可以用来指定java的包名。

接下来的小节还有一些实践样例，详细描述了你可以怎样去编译和使用生成的代码。

## 25.2.2 额外的命令行选项

下表列出了java模块可以使用的额外的命令行选项。可以键入如下指令：

```shell
swig -java -help
```

**Java特定的选项**

| 选项                | 解释                 |
| ----------------- | ------------------ |
| -noproxy          | 抑制过早的垃圾收集预防参数      |
| -noproxy          | 生成底层的函数接口而不是代理类    |
| -package \<name\> | 设置java包的名字\<name\> |

它们的用法在你读完本节之后就会明白。

## 25.2.3 获取正确的头文件

为了编译C/C++的包装代码，编译器需要知道JDK中的`jni.h`和`jni_md.h`文件。它们一般发在类似这样的目录中：

```shell
/usr/java/include
/usr/java/include/<operating_system>
```

确切的位置根据机器不同可能大不一样，但上面的目录位置很典型。

## 25.2.4 编译动态模块

