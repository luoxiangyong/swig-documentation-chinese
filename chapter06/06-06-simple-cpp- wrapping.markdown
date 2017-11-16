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

