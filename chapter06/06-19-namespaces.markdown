# 6.19 命名空间

SWIG对C++命名空间的支持是全面的，但是默认情况下比较简单，有些目标语言能通过稍后描述的nspace特征开启命名空间的高级支持。在未命名的名称空间中的代码将被忽略，因为在未命名的命名空间中的符号声明不从外部访问。在了解命名空间的默认实现之前，值得注意的是，C++命名空间的语义非常不平凡——特别是关于C++的类型系统和类机制。名称空间最基础的应用是有时用于封装公共功能。例如：

```c++
namespace math {
  double sin(double);
  double cos(double);
  
  class Complex {
    double im,re;
    public:
    ...
  };
  ...
};
```

在C++中，命名空间的成员的访问通过前置命名空间前缀来引用名称。例如：

```c++
double x = math::sin(1.0);
double magnitude(math::Complex *c);
math::Complex c;
...
```

在这个级别上，名称空间相对来说比较容易管理。然而，当您采用其他方式使用命名空间时，事情就变得非常难看了。例如，可以使用`using`选择性的导出符号：

```c++
using math::Complex;
double magnitude(Complex *c); // Namespace prefix stripped
```

同样，整个名称空间可以这样全部导出：

```c++
using namespace math;
double x = sin(1.0);
double magnitude(Complex *c);
```

另外，命名空间可以有别名：

```c++
namespace M = math;
double x = M::sin(1.0);
double magnitude(M::Complex *c);
```

组合使用这些特征的话，可以写出让人头脑发麻的代码：

```c++
namespace A {
  class Foo {
  };
}

namespace B {
  namespace C {
  	using namespace A;
  }
	
  typedef C::Foo FooClass;
}

namespace BIGB = B;

namespace D {
  using BIGB::FooClass;
  class Bar : public FooClass {
  }
};

class Spam : public D::Bar {
};

void evil(A::Foo *a, B::FooClass *b, B::C::Foo *c, BIGB::FooClass *d,
          BIGB::C::Foo *e, D::FooClass *f);
```

考虑一下这些叠加的所有可能性，估计没有哪个C++程序员希望这样的代码被包装进目标语言。很明显这段代码定义了三个不同的类。但是，每个类都可以通过六种方式访问。

SWIG在它的内部类型系统和类处理代码中完全支持命名空间。如果你将上面的代码塞给了SWIG，它将会被正确地解释、生成兼容的包装代码、生成可以工作的语言模块。但是，默认的包装行为是在目标语言中将名称空间展开。这就意味着，所有的命名空间中的内容都会合并到脚本语言模块中去。例如，如下代码：

```c++
%module foo
namespace foo {
  void bar(int);
  void spam();
}
namespace bar {
  void blah();
}
```

SWIG会简单地为`bar()`、`spam()`和`blah()`生成包装代码。它不会为每个函数生成命名空间的前缀，也不会将这些函数打进任何类型的嵌套的包中。

采取这种做法是有道理的。因为C++命名空间一般用于定于C++模块，SWIG模块的内容与命名空间中的内容之间有中天然地相关关系。比如，不能想当然的认为程序员会将每个C++的命名空间都包装成一个独立的扩展模块。在这种情况下，当这些模块本身已经在目标语言中被当做名称空间使用后，为每个符号都加上额外的命名空间前缀就显得有些冗余。或者，换种说法，如果你想让SWIG保持名称空间独立，可以简单地将每个命名空间包装成一个独立的接口模块。

因为命名空间被展平了，就有可能出现命名冲突。比如：

```c++
namespace A {
	void foo(int);
}
namespace B {
	void foo(double);
}
```

当冲突发生时，您将得到类似于此的错误消息：

example.i:26. Error. 'foo' is multiply defined in the generated target language module.
example.i:23. Previous declaration of 'foo'

为了解决这个错误，简单地使用`%rename`指令可以消除歧义声明。比如：

```c++
%rename(B_foo) B::foo;
...
namespace A {
  void foo(int);
}
namespace B {
	void foo(double); // Gets renamed to B_foo
}
```

 同样，`%ignore`也能用于忽略声明。

`using`声明对生成的包装代码没有效果。它们被SWIG语言模块忽略，不产生任何代码。但是，这些声明被内部类型系统用来跟踪类型名。因此，如果你有这样的代码：

```c++
namespace A {
	typedef int Integer;
}

using namespace A;
void foo(Integer x);
```

SWIG就会知道`Integer`与`A::Integer`是一样的，都是`int`类型。

命名空间可能会与模板结合使用。如果必要的话，`%template`指令可用来不同命名空间中的模板定义。例如：

```c++
namespace foo {
	template<typename T> T max(T a, T b) { return a > b ? a : b; }
}
using foo::max;
%template(maxint) max<int>; // Okay.
%template(maxfloat) foo::max<float>; // Okay (qualified name).
namespace bar {
	using namespace foo;
	%template(maxdouble) max<double>; // Okay.
}
```



组合使用命名空间和其他的SWIG指令可能会引入些许作用域相关的问题。**需要记住的关键是，SWIG生成的所有包装都放在全局命名空间中。**其他命名空间中的符号总是通过全限定名称进行访问——这些名字绝不会导入进全局空间，除非接口中使用了`using`声明。多数情况下，SWIG会将类型名和符号调整成全限定名的形式。但是，对类似函数体、typemap、异常处理等代码片段，不会这么做。比如，如下代码：

```c++
namespace foo {
  typedef int Integer;
  
  class bar {
    public:
    ...
  };
}

%extend foo::bar {
  Integer add(Integer x, Integer y) {
    Integer r = x + y; // Error. Integer not defined in this scope
    return r;
  }
};
```

这种情况下，SWIG正确地解析了添加的方法的参数，并且返回了`Foo::Integer`类型。但是，因为函数体没被解释，这段代码直接植入全局命名空间中，产生了关于`Integer`的编译器错误。为修复这个错误，确保你使用了全限定名称。例如：

```c++
%extend foo::bar {
  Integer add(Integer x, Integer y) {
    foo::Integer r = x + y; // Ok.
    return r;
  }
};
```

> **注意：**SWIG不会将`using`声明传播到结果包装代码中。如果这些声明出现在了接口中，它们也必须出现在初始化代码块中的头文件中。换句话说就是，除非它们在底层C++代码中出现了，不要在SWIG接口文件中插入额外的`using`声明。

> **注意：** \{%...%}块或`%inline {%...%}`块中包含的代码不能放在命名空间的声明中。通过这些指令释处的代码将不会被包装进命名空间，并且你可能会得到非常奇怪的代码。如果你需要在这些指令中使用名称空间，考虑如下方式：
>
> ```c++
> // Good version
> %inline %{
> namespace foo {
>   void bar(int) { ... }
>   ...
> }
> %}
> // Bad version. Emitted code not placed in namespace.
> namespace foo {
>   %inline %{
>     void bar(int) { ... } /* I'm bad */
>     ...
>   %}
> }
> ```

> **注意：**当在命名空间中使用`%extend`指令时，生成的代码中会包含命名空间。例如，如下代码：
>
> ```c++
> namespace foo {
> class bar {
> public:
>   %extend {
>     int blah(int x);
>     };
>   };
> }
> ```
>
> 添加的函数`blah()`被映射为`int foo_bar_blah(foo::bar *self, int x)`。这个函数在全局命名空间中。

> **注意**：尽管命名空间在目标语言中被展平了，SWIG声明的代码与输入接口文件中的代码使用的都是相同的命名空间管理。因此，如果输入没有符号冲突，生成的代码中也不会有冲突。

> 注意：同样，因为没在参数上执行解析，转换操作符名称必须与它定义的名称完全匹配。也不要更改操作符的限定名。比如，接口文件中有如下代码：
>
> ```c++
> namespace foo {
>   class bar;
>   class spam {
>   public:
>     ...
>     operator bar(); // Conversion of spam -> bar
>     ...
>   };
> }
> ```
>
> 下面是为使其正确匹配应该使用的特征操作：
>
> ```c
> %rename(tofoo) foo::spam::operator bar();
> ```
>
> 下面的代码不能工作，因为改变了操作符的名称，命名空间的解析没被执行:
>
> ```c++
> %rename(tofoo) foo::spam::operator foo::bar();
> ```
>
> 但是，还要注意，如果操作符在它的名字中使用了限定符，则在特征指令中也的这样做：
>
> ```c++
> %rename(tofoo) foo::spam::operator bar(); // will not match
> %rename(tofoo) foo::spam::operator foo::bar(); // will match
> namespace foo {
>   class bar;
>   class spam {
>   public:
>     ...
>     operator foo::bar();
>     ...
>   };
> }
> ```

> **兼容性注释：**SWIG 1.3.32版之前，这种方法的表现不一致。一般情况下需要全限定名，但在某些情况下又不能正确工作。

> **注意：**命名空间的扁平化只是作为支持基本的名称空间来实现的。目前没有任何目标语言模块用任何名称空间的方式来编程。将来，语言模块可能会，也可能不会提供更高级的命名空间支持。

## 6.19.1 命名空间的nspace特征

有些语言提供了对nspace特征的支持。这个特征也被用于声明在命名空间中的任何类、结构体、联合体或枚举。这个特征将这些类型包装到目标语言中与命名空间概念相似的设施中，例如，Java包或C#的命名空间。请查看语言特定的章节了解你感兴趣的目标语言是否支持nspace特征。

下面使用C#演示了这个特征的使用方式：

```c#
%feature("nspace") MyWorld::Material::Color;
%nspace MyWorld::Wrapping::Color; // %nspace is a macro for %feature("nspace")
namespace MyWorld {
  namespace Material {
    class Color {
    	...
    };
  }
  
  namespace Wrapping {
    class Color {
    	...
    };
  }
}
```

没有上面的的`nspace`特征指令或`%rename`，你会得到类似下面的警告，只有一个`Color`类可以在目标语言中可用：

example.i:9: Error: 'Color' is multiply defined in the generated target language module.
example.i:5: Error: Previous declaration of 'Color'

使用了`nspace`指令后，两个类都可以在C#的命名空间中使用了。在C#中可以使用全限定名访问这两个类：

```c#
MyWorld.Material.Color materialColor = new MyWorld.Material.Color();
MyWorld.Wrapping.Color wrappingColor = new MyWorld.Wrapping.Color();
```

注意，`nspace`特征不能应用于在命名空间中简单声明的变量和函数。比如，下面的符号如果不重名命名就不能同时在目标语言中同时使用。这个缺陷可能在将来的版本中有改变。

```c#
namespace MyWorld {
  namespace Material {
    int quantity;
    void dispatch();
  }
  namespace Wrapping {
    int quantity;
    void dispatch();
  }
}
```

> **兼容性注释：**从SWIG-2.0.0开始才引入`nspace`特征。