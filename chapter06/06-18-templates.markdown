# 6.18 模板

模板类型名称可能出现在接口文件中期望类型的任何位置。例如：

```c++
void foo(vector<int> *a, int n);
void bar(list<int,100> *x);
```

非类型参数的使用有些限制。SWIG支持简单的字面量和一些常量表达式。但是，在常量表达式中包含`<`和`>`目前还不支持（`<=`和`>=`也一样）。例如：

```c++
void bar(list<int,100> *x); // OK
void bar(list<int,2*50> *x); // OK
void bar(list<int,(2>1 ? 100 : 50)> *x) // Not supported
```

SWIG的类型系统很强大，能够识别typedef。比如：

```c++
typedef int Integer;
void foo(vector<int> *x, vector<Integer> *y);
```

这种情况下，`vector<Integer>`等同于`vector<int>`。对`foo`的包装支持这样的变体。

从SWIG-1.3.7开始，SWIG支持简单的 C++模板声明。SWIG-1.3.12大幅扩展了早期的实现。在深入讨论之前，有些关于模板包装的事儿你需要了解。首先，C++模板并不定义任何可运行的对象代码，SWIG并不能为其创建包装。因此，为了包装模板，你需要告诉要实例化的模板的信息（如`vector<int`，`array<double>`等）。其次，实例化的名字如`vector<int>`一般在目标语言中无效。因此，当创建包装时你需要给模板的实例一个更合适的名字，比如`intvector`。

为演示，考虑如下的模板定义：

```c++
template<class T> class List {
private:
  T *data;
  int nitems;
  int maxitems;
  
public:
  List(int max) {
    data = new T [max];
    nitems = 0;
    maxitems = max;
  }
  
  ~List() {
  	delete [] data;
  };
  
  void append(T obj) {
    if (nitems < maxitems) {
      data[nitems++] = obj;
    }
  }
  
  int length() {
  	return nitems;
  }
  T get(int n) {
  	return data[n];
  }
};
```

对SWIG来说，这个模板的声明是无效的，会被简单忽略，因为SWIG不知道怎样为其生成包装代码，除非提供了`T`的实际类型。

创建特定模板实例的包装的一种方式是提供类的扩展版本，如：

```c++
%rename(intList) List<int>; // Rename to a suitable identifier
class List<int> {
private:
  int *data;
  int nitems;
  int maxitems;
public:
  List(int max);
  ~List();
  void append(int obj);
  int length();
  int get(int n);
};
```

`%rename`指令给模板类在目标语言中提供了一个合适的标识符名字（绝大多数的语言不识别C++模板语法，认为模板类的名字是无效的）。接下来的代码和常规类定义是一样的。

由于模板的手动扩展很快就过时了，`%template` 指令用于创建模板类的实例。语义上说，`%template`是个简单的快捷方式——它以上面解释的方式一样扩展模板代码。这里有一些例子：

```c++
/* Instantiate a few different versions of the template */
%template(intList) List<int>;
%template(doubleList) List<double>;
```

`%template`指令的参数是目标语言中要使用的实例化名字。你选择的名字不能与接口文件中的其他名字冲突，但有一个例外——用typedef声明的同样类型的名称可以一样。比如：

```c++
%template(intList) List<int>;
...
typedef List<int> intList; // OK
```

SWIG还可以为函数模板生成包装代码，使用的是一样的技术。例如：

```c++
// Function template
template<class T> T max(T a, T b) { return a > b ? a : b; }
// Make some different versions of this function
%template(maxint) max<int>;
%template(maxdouble) max<double>;
```

这种情况下，`maxint`和`maxdouble`编程特定实例化函数的唯一名字。

提供该`%template`指令的参数个数与原始模板定义的参数一样。支持默认参数的模板。比如：

```c++
template vector<typename T, int max=100> 
class vector {
	...
};

%template(intvec) vector<int>;	 	// OK
%template(vec1000) vector<int,1000>; // OK
```

在同一个作用域内，`%template`指令能同时包装对相同的模板进行实例化，要不然就会报错。比如：

```c++
%template(intList) List<int>;
%template(Listint) List<int>; // Error. Template already wrapped.
```

之所以会提示错误是因为，用同样的名字对模板进行展开得到了同样的类。这导致符号表冲突。除此之外，应该对特定的模板实例化一次，减少潜在的代码膨胀。

因为类型系统知道如何处理typedef，没必要对相同的类型实例化不同版本的模板。比如，考虑如下的代码：

```c++
%template(intList) vector<int>;
typedef int Integer;
...
void foo(vector<Integer> *x);
```

这种情况下，`vector<Integer>`和`vector<int>`是一样的类型。使用`vector<Integer>`时就会将其映射到`vector<int>`上。因此，没有必要再为`Integer`类型实例化模板了（这样做是多余的，并且对导致代码膨胀）。

当使用`%template`指令实例化模板时，该类的信息就会被SWIG保存起来，在程序的其他地方使用。例如，如果你写了这样的代码：

```c++
...
%template(intList) List<int>;
...
class UltraList : public List<int> {
...
};
```

则，SWIG知道`List<int>`已经被`intList`类包装了，并且以此信息正确处理继承关系。从另一方面说，如果不知道`List<int>`，你会得到类似下面一样的警告消息：

example.h:42: Warning 401. Nothing known about class 'List<int >'. Ignored.
example.h:42: Warning 401. Maybe you forgot to instantiate 'List<int >' using %template.

如果一个模板类从另外一个模板类继承，你需要确保基类在派生类之前已经实例化了。例如：

```c++
template<class T> class Foo {
...
};
template<class T> class Bar : public Foo<T> {
...
};

// Instantiate base classes first
%template(intFoo) Foo<int>;
%template(doubleFoo) Foo<double>;
// Now instantiate derived classes
%template(intBar) Bar<int>;
%template(doubleBar) Bar<double>;
```

这里的顺序非常重要，因为SWIG在包装代码中使用实例化名字去正确处理继承层级（基类需要在派生列之前被包装）。不要担心——如果写错了顺序，SWIG会给你提示警告消息的。

偶尔情况下，你可能需要告诉SWIG通过模板定义的基类，但是基类又不想被包装。因为SWIG这种情况下不能自动实例化模板，你必须手动调整。为达此目的，简单地使用空的模板实例化，也就是不给`%template`指令提供名字。比如：

```c++
// Instantiate traits<double,double>, but don't wrap it.
%template() traits<double,double>;
```

如果你不得不为不同类型实例化不同的类，可以考虑写一个SWIG宏。例如:

```c++
%define TEMPLATE_WRAP(prefix, T...)
%template(prefix ## Foo) Foo<T >;
%template(prefix ## Bar) Bar<T >;
...
%enddef

TEMPLATE_WRAP(int, int)
TEMPLATE_WRAP(double, double)
TEMPLATE_WRAP(String, char *)
TEMPLATE_WRAP(PairStringInt, std::pair<string, int>)
```

注意这里对类型T使用了可变参数的宏。如果不这样做的话，最后一个例子中的模板类型中的逗号就会导致错误发生。

SWIG模板机制的确支持特化。比如，你定义了像这样的类：

```c++
template<> class List<int> {
private:
  int *data;
  int nitems;
  int maxitems;
public:
  List(int max);
  ~List();
  void append(int obj);
  int length();
  int get(int n);
};
```

无论何时用户展开`List<int>`时，SWIG都会使用这段代码。实践中，这对底层的包装代码没什么影响，因为特化一般被用于提供稍微修改的方法体（这些又被SWIG忽略了）。但是，特殊的SWIG指令如`%typemap`、`%extend`等能将特化版本使用起来，用于提供对特定类型的自定义。

SWIG支持部分的模板偏特化。例如，下面的代码定义了一个模板，其参数是一个指针：

```c++
template<class T> class List<T*> {
private:
  T *data;
  int nitems;
  int maxitems;
public:
  List(int max);
  ~List();
  void append(int obj);
  int length();
  T get(int n);
};
```

SWIG同时支持显式特化和偏特化。如：

```c++
template<class T1, class T2> class Foo { }; // (1) primary template
template<> class Foo<double *, int *> { }; // (2) explicit specialization
template<class T1, class T2> class Foo<T1, T2 *> { }; // (3) partial specialization
```

SWIG有能力正确地匹配显式实例化：

```c++
Foo<double *, int *> // explicit specialization matching (2)
```

SWIG实现了模板参数的推导，所有下面的偏特化例子的工作方式和它们在C++编译器中的工作方式是一样的：

```c++
Foo<int *, int *> // partial specialization matching (3)
Foo<int *, const int *> // partial specialization matching (3)
Foo<int *, int **> // partial specialization matching (3)
```

成员函数模板也支持。后面的原理与一般模板的技术是一样的——SWIG不能穿件包装代码，除非你提供类型信息。例如，一个带有模板成员的类：

```c++
class Foo {
public:
  template<class T> void bar(T x, T y) { ... };
  ...
};
```

为了展开模板，简单地在类中使用`%template`:

```c++
class Foo {
public:
  template<class T> void bar(T x, T y) { ... };
  ...
  %template(barint) bar<int>;
  %template(bardouble) bar<double>;
};
```

或者，如果你想在原始外之外定义，这样做：

```c++
class Foo {
public:
  template<class T> void bar(T x, T y) { ... };
  ...
};
...
%extend Foo {
%template(barint) bar<int>;
%template(bardouble) bar<double>;
};
```

再者：

 ```c++
class Foo {
public:
  template<class T> void bar(T x, T y) { ... };
  ...
};
...
%template(bari) Foo::bar<int>;
%template(bard) Foo::bar<double>;
 ```

这种情况下，就不需要`%extend`指令了，`%template`做了这些工作，比如向`Foo`类添加了两个新方法。

> **注意：**因为模板的处理方式，`%template`指令必须总是出现在要展开的模板定义的后面。

现在，如果你的目标语言支持重载，你甚至可以这样：

```c++
%template(bar) Foo::bar<int>;
%template(bar) Foo::bar<double>;
```

并且，因为两个包装方法的名字`bar`相同，它们就被重载了，当被调用时会根据参数类型不同通过函数分发机制调用到正确的函数。