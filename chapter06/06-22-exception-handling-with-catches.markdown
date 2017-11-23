# 6.22 用%catches处理异常

带异常规格说明的方法的异常处理时自动的。没指定异常规格说明的方法可以给通过`%catches`指令达到相似的效果。使用`%catches`特征可以替代任何申明的异常规格说明。事实上，`%catches`使用相同的`"throws" typemap`，用来处理异常的规格说明。`%catches`特征必须包含一个能被抛出的可能的类型。对在这个列表中的每个类型，SWIG将生成截获处理器，对在异常规格说明中指定类型的处理方式也是一样的。注意，列表中还可以包含截获所有规格的`...`。比如：

```c++
struct EBase { virtual ~EBase(); };
struct Error1 : EBase { };
struct Error2 : EBase { };
struct Error3 : EBase { };
struct Error4 : EBase { };
%catches(Error1,Error2,...) Foo::bar();
%catches(EBase) Foo::blah();
class Foo {
public:
  ...
  void bar();
  void blah() throw(Error1,Error2,Error3,Error4);
  ...
};
```

对`Foo::bar()`方法来说，它可以抛出任何类型，SWIG将会生成处理`Error1`、`Error2`及`...`任何类型的异常处理代码。每个处理器将截获的异常装换到目标语言的错误/异常。截获所有的异常处理器将截获的异常转换为未知的错误/异常。

`Foo::blah()`方法没有使用`%cacthes`特征，SWIG会为在方法中指定的`Error1`, `Error2`, `Error3`, `Error4`成成异常处理器。但是，通过上面的`%catches`特征指定，对基类来说只有一个异常处理器，`EBase`将被用于转换C++异常到目标语言的错误/异常。