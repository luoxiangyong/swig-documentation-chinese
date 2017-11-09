# 4.2 脚本语言与C语言间的对话

脚本语言的解释器知道如何执行命令和脚本。在这个解释器中，内置解释命令和访问变量的机制。通常，这用于实现语言的内置(builtin)特征。但是，通过扩展解析器，一般可能增加新的命令和变量。通过这样，多数的语言会定义特殊的API，用于增加命令。因此，一个特殊的外部函数接口定义了如何将这些新命令绑定到解析器中。

通常，当您向脚本解释器添加一个新命令时，您需要做两件事：首先，您需要编写一个特殊的“包装器”函数，充当解释器和底层C函数之间的粘合剂；然后，您需要向提供解释器关于包装器的信息如函数名称、参数等的详细信息。接下来的几节说明了这个过程。

## 4.2.1 包装函数

假设你的原始C函数如下：

```c
int fact(int n) {
  if (n <= 1) return 1;
  else return n*fact(n-1);
}
```

为了从脚本语言访问这个函数，需要些一个特殊的“包装（wrapper）”函数，充当脚本语言和底层C函数之间的粘合剂。这个包装函数必须做三件事：

+ 收集函数参数并使它们有效
+ 调用C函数
+ 转换C函数的返回值，构造成脚本语言识别的形式

作为示例，Tcl语言的`fact()`包装函数可能看起来像下面这个样子：

```c
int wrap_fact(ClientData clientData, Tcl_Interp *interp,int argc, char *argv[]) {
  int result;
  int arg0;
  
  if (argc != 2) {
    interp->result = "wrong # args";
    return TCL_ERROR;
  }
  
  arg0 = atoi(argv[1]);
  result = fact(arg0);
  sprintf(interp->result,"%d", result);
  
  return TCL_OK;
}
```

一旦你创建了一个包装函数，最后的一步就是告诉脚本语言关于这个新函数函数的信息。通常可以通过加载模块，在语言的初始化函数中完成。例如，将上面的函数添加到Tcl解释器中的代码如下：

```c
int Wrap_Init(Tcl_Interp *interp) {
  Tcl_CreateCommand(interp, "fact", wrap_fact, 
                    (ClientData) NULL,
                    (Tcl_CmdDeleteProc *) NULL);
  return TCL_OK;
}
```

当执行时，Tcl将会有一个叫`fact`的新命令，你可以像使用其他Tcl命令一样使用它。

尽管只介绍了向Tcl添加新函数的过程，但是扩展Perl和Python的过程大致也是这样的。都需要特殊的包装函数和额外的初始化代码。只有具体细节是不一样的。

## 4.2.2 变量的链接

变量的链接涉及到将C/C++的全局变量映射到脚本语言的问题。例如，假设你有如下的变量:

```c
double Foo = 3.5;
```

从脚本中可以像下面这样访问就漂亮了（Perl）:

```perl
$a   = $Foo * 2.3;  # Evaluation
$Foo = $a + 2.0; 	# Assignment
```

为提供这样的访问，变量通常使用一对get/set函数来操作。例如，不管何时变量被读，get函数被调用。同样，不管何时变量被更改，set函数被调用。

在一些语言中，对get/set函数的调用可以与求值(evaluation)、赋值(assignment)操作符关联起来。因此，计算变量`$Foo`可以隐式调用get函数。同样，键入`$Foo = 4`可以调用底层的set函数改变其值。

## 4.2.3 常量

多数情况下，C程序或库定义了大量的常量。例如：

```c
#define RED 0xff0000
#define BLUE 0x0000ff
#define GREEN 0x00ff00
```

为让常量可用，它们的值在脚本语言中存储为变量`$RED, $BLUE, $GREEN`。所有的脚本语言都提供创建变量的C函数，因此安装常量比较简单。

## 4.2.4 结构体与类

尽管脚本元访问简单函数和变量没问题，但访问C/C++结构体和类还是有困难的。这是因为结构体的实现很大程度上与数据的表现与布局相关。因而，某些语言特征很难映射到解析器中。例如，C/C++的继承在Perl意味着什么？

处理结构体最直接的技术是实现一组访问函数，隐藏底层结构题的数据呈现。例如：

```c
struct Vector {
  Vector();
  ~Vector();
  double x, y, z;
};
```

可以转换成以下一组函数：

```c
Vector *new_Vector();
void delete_Vector(Vector *v);
double Vector_x_get(Vector *v);
double Vector_y_get(Vector *v);
double Vector_z_get(Vector *v);
void Vector_x_set(Vector *v, double x);
void Vector_y_set(Vector *v, double y);
void Vector_z_set(Vector *v, double z);
```

现在，你可以从解析器中这样操作：

```tcl
% set v [new_Vector]
% Vector_x_set $v 3.5
% Vector_y_get $v
% delete_Vector $v
```

因为访问函数提供了访问对象内部的机制，解析器不需要知道`Vector`的实际内存呈现。

## 4.2.5 代理类

在某些情况下，可能需要使用底层的访问函数创建代理类，也称影子类。代理类是一种特殊的对象，它使用脚本语言实现，访问C/C++类（或结构体）,其表现得就像原始结构一样（它代理实际的C++类）。例如，你有如下的C++定义：

```c
class Vector {
  public:
  Vector();
  ~Vector();
  double x, y, z;
};
```

代理类机制允许从解析器中使用更自然的方式访问结构体。例如，在Python中，可以这么做：

```python
>>> v = Vector()
>>> v.x = 3
>>> v.y = 4
>>> v.z = -13
>>> ...
>>> del v
```

同样，在Perl5中可以这样：

```perl
$v = new Vector;
$v->{x} = 3;
$v->{y} = 4;
$v->{z} = -13;
```

最后，使用Tcl可以这样：

```tcl
Vector v
v configure -x 3 -y 4 -z -13
```

当使用代理类的时候，实际有两个对象一起工作，一个是脚本语言对象，一个是C++对象。如果你简单的操作了C/C++对象，该操作实际中将同时影响两个对象。