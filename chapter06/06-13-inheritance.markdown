# 6.13 继承

SWIG支持C++类的继承，允许单个或多重继承，只受目标语言所限。SWIG的类型检查器知道基类与派生类之间的关系，允许指向派生类对象的指针在函数使用。类型检查器可以正确处理指针的强制转换，并且可以安全用于多重继承。

SWIG处理私有和保护继承的方式与C++的精神是相近的，只受目标语言能力的限制。多数情况下，这就意味着，SWIG会解释非公有的继承声明，除针对构造函数与析构函数的隐式策略外，它对代码生成没什么作用。

下面的雷子显式了SWIG如何处理继承。为了清晰，给出了全部的C++代码：

```c++
// shapes.i
%module shapes
%{
#include "shapes.h"
%}

class Shape {
public:
  double x,y;
  virtual double area() = 0;
  virtual double perimeter() = 0;
  void set_location(double x, double y);
};

class Circle : public Shape {
public:
  Circle(double radius);
  ~Circle();
  double area();
  double perimeter();
};

class Square : public Shape {
public:
  Square(double size);
  ~Square();
  double area();
  double perimeter();
}
```

当包装成Python后，我们可以执行线面的操作(用底层的Python访问函数演示):

```python
$ python
>>> import shapes
>>> circle = shapes.new_Circle(7)
>>> square = shapes.new_Square(10)
>>> print shapes.Circle_area(circle)
153.93804004599999757
>>> print shapes.Shape_area(circle)
153.93804004599999757
>>> print shapes.Shape_area(square)
100.00000000000000000
>>> shapes.Shape_set_location(square,2,-3)
>>> print shapes.Shape_perimeter(square)
40.00000000000000000
>>>
```

在这个例子中，创建了`Circle`和`Square`的对象。可在对象上调用成员函数：`Circle_area` ,`Square_area`等。但是在每个对象上只使用`Shape_area`完成同样的任务。

关于继承一个非常重要的一点是，底层的访问函数的生成只对那些实际声明的对象有效，在上面的例子中，`set_location()`函数只能通过`Shape_set_location()`访问，而不能通过`Circle_set_location()`或`Square_set_location()`访问。当然了，`Shape_set_location()`函数接受从`Shape`继承的类的任何对象。同样地，属性`x`和`y`的访问函数为`Shape_x_get()`,`Shape_x_set()`,`Shape_y_get()`和`Shape_y_set()`。函数`Circle_x_get()`不存在——可以使用`Shape_x_get()`代替。

请注意，底层的访问函数与代理类的方法一一对应，因此C++类方法与生成的代理类也是一一对应的。

> **注意：**为得到最好的结果，SWIG需要所有的基类都要在接口文件中定义。否则，你会得到如下的警告提示：
>
> ```shell
> example.i:18: Warning 401: Nothing known about base class 'Foo'. Ignored.
> ```
>
> 如果任何一个基类没有定义，SWIG还是会生成正确的类型关系。例如，接受`Foo*`类型的函数将会接受任何从`Foo`继承的类的对象，无论SWIG是否包装了`Foo`类。如果你真的不想为基类生成包装，但是你想关掉警告，可以考虑使用`%import`指令包含定义了`Foo`的文件。`%import`简单地收集类型信息，但不生成包装代码。另外一种方式是，你可以只定义`Foo`为空类或使用[警告抑制](#swig-warning-suppression)。

> **注意：**Typedef的名字可以被用作基类。比如：
>
> ```c++
> class Foo {
> ...
> };
> typedef Foo FooObj;
> class Bar : public FooObj { // Ok. Base class is Foo
> 	...
> };
> ```
>
> 同样地，typedef允许为命名的结构体用作基类。比如：
>
> ```c++
> typedef struct {
> ...
> } Foo;
>
> class Bar : public Foo { // Ok.
> ...
> };
> ```

> **兼容性注释：**从SWIG 1.3.7开始，SWIG只对在类中实际定义的声明生成包装代码。这与SWIG 1.1不同，它继承所有在基类中的声明，生成特定的访问函数，比如：`Circle_x_get()`,`Square_x_get()`,`Circle_set_location()`和`Square_set_location()`。这种行为将导致在那些大的类继承体系中生成了大量重复的代码，并且使构建跨多个模块的应用程序变得很麻烦（因为由于访问函数在每一个模块都有重复）。当使用代理类等高级特性时，也不需要有这样的包装器。注意，当开启-fvirtual选项后，将启用进一步的优化，这将避免对定义在基类中的虚拟函数再生成包装代码了。