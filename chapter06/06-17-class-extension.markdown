# 6.17 类扩展

使用`%extend`指令可以为类添加新方法。这个指令主要被与代理类连接，用于给存在的类添加额外的功能。例如：

```c++
%module vector
%{
#include "vector.h"
%}

class Vector {
public:
	double x,y,z;
  	Vector();
    ~Vector();
  
    ... bunch of C++ methods ...
      
    %extend {
      char *__str__() {
        static char temp[256];
        sprintf(temp,"[ %g, %g, %g ]", $self->x,$self->y,$self->z);
        return &temp[0];
       }
   }
};
```

这段代码给我们的类添加了一个`__add__`方法，生成对象的字符串表示。在Python中，这样的方法允许我们使用`print`名字打印对象的值。

```python
>>>
>>> v = Vector();
>>> v.x = 3
>>> v.y = 4
>>> v.z = 0
>>> print(v)
[ 3.0, 4.0, 0.0 ]
>>>
```

C++的`this`指针经常被用于访问成员变量、方法等。特殊标量`$self`应该用在你需要`this`的地方。上面的例子演示了使用这种方式访问成员变量的方法了。注意，通过`$self`引用的变量必须是公有的成员，因为这些代码最终会生成全局函数，所以不能访问任何非公有的成员。C++中隐式指针`this`在C++方法中可用，但在`%extend`指令中不可用。为了在扩展类或其基类中访问所有成员，必须使用显式的`this`。下面的例子演示了如何访问基类的成员：

```c++
struct Base {
  virtual void method(int v) {
  ...
  }
  int value;
};

struct Derived : Base {
};

%extend Derived {
  virtual void method(int v) {
    $self->Base::method(v);  // akin to this->Base::method(v);
    $self->value = v; 		// akin to this->value = v;
    ...
  }
}
```

在`%extend`指令块中，下面的特使变量将被扩展：`$name`、`$sysname`、`overname`、`$decl`、`$fulldecl`、`parentclassname`和`$parentclasssymname`。[特殊变量](#swig-special-varials)这一节对每个特殊变量都做了详细介绍。

`%extend`指令与对C结构体处理方法是一样的。请参考[给C结构体添加成员函数](#sw-g-adding-member-functions-to-c-structures)获得等多信息。

> **兼容性注释：**`%extend`指令是 SWIG 1.1中`%addmethods`指令的新的名字。因为`%addmethods`不仅仅只是给结构体扩展方法，所以我们给它选择了另外一个更合适的指令名。