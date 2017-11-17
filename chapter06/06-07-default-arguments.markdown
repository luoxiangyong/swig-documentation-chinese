# 6.7 默认参数

SWIG可以处理各种类型的带默认参数的函数。例如成员函数：

```c++
class Foo {
public:
	void bar(int x, int y = 3, int z = 4);
};
```

SWIG通过对每个默认参数生成额外的重载方法来处理默认参数。SWIG对带有默认参数的方法的处理与对重载方法的处理一样。因此对于上面的例子，就好像我们有一下的定义一样：

```c++
class Foo {
public:
  void bar(int x, int y, int z);
  void bar(int x, int y);
  void bar(int x);
};
```

关于此的细节在后面[包装重载函数与方法](#swing-wrapping-overloaded-functions-and-methods)节会详细介绍。这种方式允许SWIG包装任何可能的参数，但很啰嗦。比如，如果方法有10个默认参数，就会生成11个包装函数。

请查看[特征与默认参数](#swig-features-and-default-arguments)，了解在带有默认参数的函数上使用`%feature`的详细信息。[模糊解析与重命名](#swig-ambiguity-resolution-and-renaming)节有关于在带有默认参数的函数上使用`%rename`和`%ignore`的介绍。如果你正在为带有默认参数的函数写自己的typemap，你可能需要看看[Typemap与重载](#swig-typemaps-and-overloading)这一节，或则使用下面将要介绍的`compactdefaultargs`特性。

> 兼容性注释：SWIG-1.3.23之前的版本对默认参数的包装略有不同。只包装一个方法，默认值被拷贝到C++包装函数中，调用全参数的函数。如果关心包装函数的大小，可以通过使用`compactdefaultargs`特性重新激活该特性：
>
> ```c++
> %feature("compactdefaultargs") Foo::bar;
> class Foo {
> public:
> 	void bar(int x, int y = 3, int z = 4);
> };
> ```
>
> 这将大大减少包装函数的大小，但需要警告你的是，对于像C#、Java这样的静态语言来说，这不会工作，这些语言没有可选参数，这个特性的另外一个限制是，它不能处理非公有的参数。下面的示例可以证明：
>
> ```c++
> class Foo {
> private:
>   static const int spam;
> public:
>   void bar(int x, int y = spam); // Won't work with %feature("compactdefaultargs") -
>   // private default value
> };
> ```
>
> 这会生成不兼容的包装代码，因为在C++中默认值的计算与成员函数在同一个作用域中，然而SWIG在包装函数的作用域中使用它们（这就要求值必须是公有的）。
>
> `compactdefaultargs`特性在包装带有默认参数的C代码时是自动打开的。有些目标语言支持关键字参数（kwargs）也会自动开启这个特性，不管是C还是C++语言。关键字参数时一些脚本语言的语言特性，比Python和Ruby。当包装重载方法时，SWIG不能支持kwargs，不能使用默认的方式去处理。

