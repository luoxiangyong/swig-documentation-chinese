# 6.26 using声明与继承

`using`声明有时候被用于调整对基类成员的访问。例如：

```c++
class Foo {
public:
	int blah(int x);
};

class Bar {
public:
	double blah(double x);
};

class FooBar : public Foo, public Bar {
public:
  using Foo::blah;
  using Bar::blah;
  
  char *blah(const char *x);
};
```

在这个例子中，`using`声明在派生类中引入了不同版本重载的`blah()`方法。例如：

```c++
FooBar *f;
f->blah(3); // Ok. Invokes Foo::blah(int)
f->blah(3.5); // Ok. Invokes Bar::blah(double)
f->blah("hello"); // Ok. Invokes FooBar::blah(const char *);
```

当这样的代码被包装时，SWIG也会模拟类似的功能。例如，如果你在Python中包装这段代码时，工作方式和你期望的方式一样：

```python
>>> import example
>>> f = example.FooBar()
>>> f.blah(3)
>>> f.blah(3.5)
>>> f.blah("hello")
```

`using`声明还能用于改变访问方式。例如：

```c++
class Foo {
protected:
  int x;
  int blah(int x);
};

class Bar : public Foo {
public:
  using Foo::x; 	// Make x public
  using Foo::blah;  // Make blah public
};
```

SWIG也支持这样的工作方式——包装后也能正常工作。

当`using`声明通过上面的方式使用时，基类中的声明被拷贝至派生类，然后正常包装。当拷贝时，这些声明依然保持使用`%rename`、`%ignore`或`%feature`指令关联的任何属性。因此，如果方法在基类中被忽略，即使使用了`using`声明也还是被忽略。

由于`using`声明不提供对导入声明的细粒度控制，对这些声明的管理可能很困难，要使用很多SWIG自定义特性。如果你不能让`using`正确的工作，你总能像下面这样做：

```c++
class FooBar : public Foo, public Bar {
public:
#ifndef SWIG
using Foo::blah;
using Bar::blah;
#else
int blah(int x); // explicitly tell SWIG about other declarations
double blah(double x);
#endif
char *blah(const char *x);
};
```

> **注意：**
>
> + 如果派生类重新定义了基类的方法，`using`声明也不会导致冲突。例如：
>
> ```c++
> class Foo {
> public:
>   int blah(int );
>   double blah(double);
> };
>
> class Bar : public Foo {
> public:
> 	using Foo::blah; // Only imports blah(double);
> 	int blah(int);
> };
> ```
>
> + 消除重载的歧义可以通过使用`using`导入声明来实现。例如：
>
> ```c++
> %rename(blah_long) Foo::blah(long);
> class Foo {
> public:
>   int blah(int);
>   long blah(long); // Renamed to blah_long
> };
>
> class Bar : public Foo {
> public:
>   using Foo::blah; // Only imports blah(int)
>   double blah(double x);
> };
> ```
>
> 