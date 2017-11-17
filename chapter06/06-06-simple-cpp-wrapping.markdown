# 6.6 简单的C++包装

下面的例子展示了简单C++类被SWIG接口文件进行封装的方法：

```c++
%module list
%{
#include "list.h"
%}
// Very simple C++ example for linked list
class List {
public:
  List();
  ~List();
  int search(char *value);
  void insert(char *);
  void remove(char *);
  char *get(int n);
  int length;
  static void print(List *l);
};
```

为了包装这个类，SWIG首先将该类变换成一组底层的C风格的访问函数，这些函数之后被代理类使用。

## 6.6.1 构造函数与析构函数

C++构造函数与析构函数可使用下面的方法转换成访问函数：

```c++
List * new_List(void) {
	return new List;
}
void delete_List(List *l) {
	delete l;
}
```

## 6.6.2 默认构造函数，拷贝构造函数与隐式析构函数

下面的C++规则用于C++的隐式构造函数与析构函数：即使没在接口文件中显式声明，SWIG还是会自动假设它们存在。

一般情况下：

+ 如果C++类没有声明显式构造函数，SWIG将自动为其生成包装代码。
+ 如果C++类没有的声明显式的拷贝构造函数，如果使用了`%copyctor`指令，SWIG将自动为其生成包装代码。
+ 如果C++类没有声明显式析构函数，SWIG将自动为其生成包装代码。

同时，在C++中，还有一些规则可用来改变先前的行为：

+ 如果类已经定义了一个带有参数的构造函数的话，默认构造函数就不会被创建。
+ 对带有纯虚函数的类或从抽象类继承的类，不生成默认构造函数，对所有的纯虚方法不提供其定义。
+ 除非所有的积基类都提供默认构造函数，否则不生成默认构造函数。
+ 如果使用`private`或`protected`限定符修饰了构造函数和析构函数的话，不生成默认构造函数和隐式析构函数。
+ 如果任何基类定义了一个非公有的默认构造函数或析构函数的话，也不生成默认构造函数和隐式析构

符合上面的条件，SWIG再生成默认构造函数，拷贝构造函数或默认析构函数包装代码的话就是非法的。但是，在某些情况下，通过手动禁止生成隐式构造函数或析构函数可能也是必要的（如果上面规则违反了，并且在SWIG中没有提供类的完全声明）或期望的。

为手动禁止生成它们，可使用`%nodefaultctor`和`%nodefaultdtor`指令。注意，这些指令只影响隐式生成，并且对显式声明的默认/拷贝构造函数或析构函数不起作用。

例如：

```c++
%nodefaultctor Foo; // Disable the default constructor for class Foo.
class Foo { // No default constructor is generated, unless one is declared
...
};
class Bar { // A default constructor is generated, if possible
...
};
```

`%nodefaultctor`指令可应用到全局:

```c++
%nodefaultctor; // Disable creation of default constructors
class Foo { // No default constructor is generated, unless one is declared
...
};
class Bar {
public:
	Bar(); // The default constructor is generated, since one is declared
};
%clearnodefaultctor; // Enable the creation of default constructors again
```

响应地，如果需要，`%nodefaultdtor`指令可用于禁止生成默认或隐式的析构函数。但请注意，这可能导致目标语言内存泄露。因此，推荐只在你非常了解代码的情况下使用它。例如：

```c++
%nodefaultdtor Foo; // Disable the implicit/default destructor for class Foo.
class Foo { // No destructor is generated, unless one is declared
...
};
```

> 兼容性注释：从SWIG 1.3.7开始生成默认生成构造函数、隐式析构函数。这可能会破坏旧的模块，但是旧的行为可以简单的使用`%nodefault`指令或`-nodefault`命令行选项修复。因此，为了让SWIG正确的生成（或不生成）默认构造函数，必须能够从`private`和`protected`部分收集信息（特别是，需要知道是否定义了私有的或保护的构造函数/析构函数）。在旧的SWIG版本中，简单地删除或注释掉类私有的或保护的部分很平常。但是，删除它们现在将导致SWIG错误地为类生成构造函数。可以考虑在接口文件中修复这部分内容或使用`%nodefault`指令修复问题。

> 注意：上面描述的`%nodefault`指令，`-nodefault`选项，会禁用默认构造函数和隐式析构函数，可能导致内存泄露，所以强烈建议不要使用它们。

## 6.6.3 当针对构造函数的包装没有生成的时候

如果类定义了构造函数，SWIG一般会尽量生成对它的包装代码。但是，如果SWIG认为这会导致不合法的代码是，它就不会生成构造函数的代码。实际上有两种情况可能会出现。

首先，SWIG不会为保护的或私有的构造函数生成包装代码。例如：

```c++
class Foo {
protected:
  Foo(); // Not wrapped.
public:
  ...
};
```

其次，如果类是抽象的，SWIG也不会为其生成构造函数包装代码——也就是说，那些带有纯虚的未定义的函数。下面是一些示例：

```c++
class Bar {
public:
  Bar(); // Not wrapped. Bar is abstract.
  virtual void spam(void) = 0;
};
class Grok : public Bar {
public:
	Grok(); // Not wrapped. No implementation of abstract spam().
};
```

有些用户会惊讶(或迷惑)的发现生成的包装代码中没有构造函数。这种情况的发生多数是因为类被定义成了抽象的。为了看看是不是这样的，开启警告支持运行SWIG：

```shell
% swig -Wall -python module.i
```

在这个模式下， SWIG将对所有的抽象类提示警告消息。可以使用如下的方法强制让一个类变成非抽象的:

```c++
%feature("notabstract") Foo;
class Foo : public Bar {
public:
  Foo(); // Generated no matter what---not abstract.
  ...
};
```

关于`%feature`指令的更多信息可从[自定义特征](customization-features)这一章获取。

## 6.6.4 拷贝构造函数

如果类定义了不止一个构造函数，其在目标语言中的行为依赖于目标语言的能力。如果目标语言支持重载，就可以通过正常的构造函数访问拷贝构造器。例如，如果有一下代码：

```c++
class List {
public:
  List();
List(const List &); // Copy constructor
...
};
```

则，拷贝构造函数可以这么使用：

```python
x = List() # Create a list
y = List(x) # Copy list x
```

如果目标语言不支持重载，则拷贝构造函数通过像下面这样特殊的方式访问：

```c++
List *copy_List(List *f) {
	return new List(*f);
}
```

> **注意：**对于类X，只有构造函数接受类型X或X*，SWIG才会将构造函数用作拷贝构造函数。如果定义了不止一个拷贝构造函数，只有第一个定义的能被使用——其他的定义会导致命名冲突。SWIG将类似`X(const X &)`、`X(X &)`及`X(X *)`这样的构造函数都认识是拷贝构造函数。

> **注意：**除非你显式的在类中声明了，SWIG不会生成拷贝构造函数的包装代码。这与对构造函数和析构函数的处理方式不同。但是，如果使用`copyctor`特征标志，也能生成拷贝构造函数的包装。比如：
>
> ```c++
> %copyctor List;
> class List {
> public:
> 	List();
> };
> ```
>
> 将会为`List`生成拷贝构造函数的包装。

> **兼容性注释：**对拷贝构造函数的特殊支持只到SWIG-1.3.12才被加入。先前的版本中，拷贝构造函数能够生成，但它们需要被重命名。例如：
>
> ```c++
> class Foo {
> public:
>   Foo();
>   %name(CopyFoo) Foo(const Foo &);
> 	...
> };
> ```
>
> 为了向后兼容，如果构造函数已经被手动重新命名了，SWIG就不会执行对任何拷贝构造函数的处理。比如在上面的例子中，构造器就被设置为了`new_CopyFoo()`。这个行为和旧版本的一致。

## 6.6.5 成员函数

所有的成员函数都是大致翻译成这样的访问函数：

```c++
int List_search(List *obj, char *value) {
	return obj->search(value);
}
```

即使成员函数被声明为虚拟，这个转换也是一样的。值得注意的，SWIG实际上并没有真的在生成的代码中创建C风格的访问函数。却而代之的是，对类似`obj->search(value)`这样成员的访问，被直接内联到生成的包装函数之中。但是，底层过程包装函数的名字和调用约定和上面访问函数原型的描述是一致的。

## 6.6.6 静态成员

静态成员函数都被直接调用，不做任何特殊的处理。例如，静态成员函数`print(List *l)`，在生成的包装代码中可直接用`List::print(List *l)`调用。

## 6.6.7 数据成员

对数据成员的处理和对C结构体的处理时一样的。直接创建一对访问函数。比如：

```c++
int List_length_get(List *obj) {
	return obj->length;
}
int List_length_set(List *obj, int value) {
	obj->length = value;
return value;
}
```

可使用`%immutable`和`%mutable`指令创建只读成员。例如，我们可能不像用户修改列表的长度，所以可以像下面这样做，让变量可用，但是是只读的：

```c++
class List {
public:
  ...
  %immutable;
  int length;
  %mutable;
  ...
};
```

还可以这样指定：

```c++
%immutable List::length;
...
class List {
  ...
  int length; // Immutable by above directive
  ...
};
```

类似地，所有使用`const`修饰的数据属性也都被包装成只读的成员。

默认情况下，SWIG使用常量引用typemap对原生类型的成员进行处理。对于不是原生的数据类型的成员，包装它们的方式稍微有些不同，比如类。比如，你有另外一个像这样的类：

```c++
class Foo {
public:
  List items;
  ...
```

则底层对`items`的访问函数实际使用指针：

```c++
List *Foo_items_get(Foo *self) {
	return &self->items;
}
void Foo_items_set(Foo *self, List *value) {
  self->items = *value;
}
```

更多信息可从[SWIG基础](#swig-basics)这一章的[结构体的数据成员]()(#swig-structure-data-members)节了解到。

类的访问函数的包装代码的生成使用指针typemap。这对某些类型可能有些不自然。例如，用户可能期望STL的 `std:string`作为类的成员可从目标语言中作为字符串访问，而不是指向这个类的指针。常量引用typemap提供了对这种类型的列集(marshalling)，它高数SWIG使用常量引用typemap而不是指针typemap。这样更自然，可更有效的更改生成访问函数的方式：

```c++
const List &Foo_items_get(Foo *self) {
	return self->items;
}
void Foo_items_set(Foo *self, const List &value) {
	self->items = value;
}
```

`%naturalvar`指令是一个宏定义，它等同于`%feature("naturalvar")`。可以像线面这样使用：

```c++
// All List variables will use const List& typemaps
%naturalvar List;
// Only Foo::myList will use const List& typemaps
%naturalvar Foo::myList;
struct Foo {
	List myList;
};
// All non-primitive types will use const reference typemaps
%naturalvar;
```

细心的读者可能注意到`%naturalvar`指令和其他特征标志指令工作方式一样，但灵活性更大。第一个示例中使用`%naturalvar`关联到`List`类型的`myList`变量上。第二个示例使用`%naturalvar`指令直接关联到变量名上。因此`naturalvar`特征可在变量名和类型上。需要注意地是，作用到变量名上的`naturalvar`特征会覆盖作用到类型上的`naturalvar`特征。

`naturalvar`的行为可以通过`-naturalvar`命令行选项或者通过`%module(naturalvar=1)`进行去全局调整。但是，使用`%feature("naturalvar")`可以覆盖全局设置。

> **兼容性注释：**`%naturalvar`从SWIG-1.3.28开始引入，早期版本需要手动应用常量引用typemap，如`%apply const std::string & { std::string * }`，但这个例子同样可以应用于带有`std::string*`参数的方法上。

> **兼容性注释：**对只读访问的控制以前是通过`%readonly`和`%readwrite`指令。尽管这些指令现在还可以工作，但会有警告提示。简单将它们替换成`%immutable;`和`%mutable;`就可以了。**不要忘记了额外的分号。**

> **兼容性注释：**在SWIG-1.3.12之前，所有未知类型都被包装成使用指针的访问函数。例如，有下面的结构：
>
> ```c++
> struct Foo {
> 	size_t len;
> };
> ```
>
> 且不知道`size_t`的类型，则访问函数使用`size_t*`工作。从SWIG-1.3.12开始，这个行为被修改了。特殊情况下，只有SWIG知道数据类型是结构体或类的时候才会使用指针的方案。因此，上面的代码会包装成使用`size_t`类型的访问函数。这个改变很小，但是它是对一些问题的处理变得更平滑，比如结构体包装和SWIG的自定义特性。