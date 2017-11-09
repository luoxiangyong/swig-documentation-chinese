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

