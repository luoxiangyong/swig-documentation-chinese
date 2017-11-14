# 5.6 代码插入

有时候有必要像SWIG生成的包装代码中插入特殊的代码。比如，你可能想包含额外的C代码，执行初始化或其他操作。有4种通用方式用来插入代码，但在此之前首先了解SWIG输出内容的构成也是很有帮助的。

## 5.6.1 SWIG的输出

当SWIG创建它的输出文件时，输出内容被分隔成5个部分，分别是：开始部分、运行时代码、头部、包装函数和初始化代码。

+ 开始部分

  在C/C++包装代码文件的开始处，用户放置代码的占位符。它通常用来定义预处理器宏，以被后用。

+ 运行时代码

  这些代码用于SWIG内部，包含类型检查和其他的模块支持函数。

+ 头部

  这里是用户自定义的支持代码，通过`%{ %}`块直接包含进来。通常这里放一些包含文件和其他的帮助函数。

+ 包装代码

  这里就是SWIG自动生成的包装代码。

+ 模块初始化

  SWIG生成的函数，用于初始化模块的加载。

## 5.6.2 代码插入块

可以使用如下的插入指令将合适的代码插入特定的部分。包装文件中它们的顺序如下：

```c
%begin %{
... code in begin section ...
%}
%runtime %{
... code in runtime section ...
%}
%header %{
... code in header section ...
%}
%wrapper %{
... code in wrapper section ...
%}
%init %{
... code in init section ...
%}
```

空的`%{ ... %}`块指令是`%header %{ ... %}`的简写方式。

默认情况下`%begin`段初包含SWIG的标题外是空的。这个部分提供了在包装文件中，在其他代码生成前插入让用户的代码。代码插入块中的所有内容都被逐字复制到输出文件中，SWIG不会解析它们。多数的SWIG输入文件至少有这样一个块，用来插入头文件和支持的C代码。其他代码根据需要可以在SWIG文件的任何位置插入。

```c
%module mymodule
%{
#include "my_header.h"
%}
... Declare functions here
%{
void some_extra_function() {
...
}
%}
```

代码段的一般用法是编写帮助函数。这些函数用于特殊目的的接口绑定，但一般C程序里又没有。比如：

```c
%{
/* Create a new vector */
static Vector *new_Vector() {
return (Vector *) malloc(sizeof(Vector));
}
%}
// Now wrap it
Vector *new_Vector();
```

## 5.6.3 内联代码块

因为编写帮助函数很普遍，那就可以使用特定的内联代码块格式：

```c
%inline %{
/* Create a new vector */
Vector *new_Vector() {
	return (Vector *) malloc(sizeof(Vector));
}
%}
```

接口文件中的`%inline`指令将其中的所有代码逐字拷贝到包装代码文件的头部，这些代码然后会被SWIG预处理器和解释器处理。因此，上面的示例用一个声明创建了一个新的命令`new_Vector`。因为`%inline %{ ... %}`块中的代码同时传递给SWIG和C编译器，在其中再插入`%{ ... %}`指令就不合法了。

## 5.6.4 初始化代码

当代码包含在`%init`段中时，它直接传递给模块初始化函数。例如，如果你需要执行一些额外的模块加载时的初始化，可以这样写代码：

```c
%init %{
	init_variables();
%}
```

