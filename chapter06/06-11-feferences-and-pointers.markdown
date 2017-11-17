# 6.11 引用与指针

SWIG支持C++的引用，但将其转换成指针。例如，像这样的声明：

```c++
class Foo {
public:
	double bar(double &a);
}
```

访问函数是这样的：

```c++
double Foo_bar(Foo *obj, double *a) {
	obj->bar(*a);
}
```

作为特例，多数的语言模块对传递const原生数据类型的引用(int、short、float等)的处理成传值，而不是指针。例如，有这样的函数：

```c++
void foo(const int &x);
```

在脚本语言中可以这样调用：

```python
foo(3) # Notice pass by value
```

返回引用的函数被映射为指针。例如：

```c++
class Bar {
public:
	Foo &spam();
};
```

生成的访问函数像这样：

```c++
Foo *Bar_spam(Bar *obj) {
  Foo &result = obj->spam();
  return &result;
}
```

但是，返回原生类型(int、short等)的const引用的函数一般通过值而不是指针。例如，下面的函数：

```c++
const int &bar();
```

在目标语言中可能返回类似37或42这样的值而不是指针。

不要返回在堆栈上分配的变量的引用！SWIG不会拷贝对象，所以这可能会导致你的程序崩溃。

> 对原生类型的特殊处理时必要的，这提供了对更多C++高级特性的无缝集成——特别是关于模板和STL。这个特性从SWIG-1.3.12开始引入。