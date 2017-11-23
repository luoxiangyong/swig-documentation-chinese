# 6.24 智能指针与operator->()

在一些C++程序中，通常使用智能指针和代理类来包装对象。有时候它们被用来实现自动内存管理（引用计数）或持久化。典型情况下智能指针用模板类定义，重载了操作符`operator->`。其他类被该类包装。比如：

```c++
// Smart-pointer class
template<class T> class SmartPtr {
  T *pointee;
public:
  SmartPtr(T *p) : pointee(p) { ... }
  T *operator->() {
  	return pointee;
    ...
  }
};
  
// Ordinary class
class Foo_Impl {
public:
  int x;
  virtual void bar();
  ...
};

// Smart-pointer wrapper
typedef SmartPtr<Foo_Impl> Foo;

// Create smart pointer Foo
Foo make_Foo() {
	return SmartPtr<Foo_Impl>(new Foo_Impl());
}

// Do something with smart pointer Foo
void do_something(Foo f) {
  printf("x = %d\n", f->x);
  f->bar();
}

// Call the wrapped smart pointer proxy class in the target language 'Foo'
%template(Foo) SmartPtr<Foo_Impl>;
```

这种方法的关键点是定义`operator->()`方法，被智能指针包装的对象属性的访问时透明的。例如，这样的表达式：

```c++
f->x
f->bar()
```

被透明的映射到：

```c++
(f.operator->())->x;
(f.operator->())->bar();
```

当生成包装时，SWIG尽量模拟这种扩展功能。当在类中无论何时遇到`operator->()`，SWIG查看它的返回值，通过它来访问底层对象的属性。例如，对上面代码的包装可能看起来像这样：

```c++
int Foo_x_get(Foo *f) {
	return (*f)->x;
}

void Foo_x_set(Foo *f, int value) {
	(*f)->x = value;
}

void Foo_bar(Foo *f) {
	(*f)->bar();
}
```

这些包装代码使用智能指针的实例作为参数，但是通过解引用获取对`operator->()`操作符返回的对象访问。你应该仔细对比这里的包装代码和本章前面的包装代码（它们稍许不同）。

结果是访问方式与在C++中相似。例如，你可以在Python中这样做：

```c++
>>> f = make_Foo()
>>> print f.x
0
>>> f.bar()
>>> 
```

当通过智能指针生成包装代码时，SWIG尽力为所有通过`operator->()`访问的方法和属性生成包装代码。这包括通过继承能够被访问的任何成员。但是，有一些限制：

+ 成员变量和方法通过智能指针被包装。枚举、构造器和析构器不被包装。

+ 如果智能指针类和底层的对象都定义了相同名字的方法了变量的话，则智能指针版优先。例如，如果有如下代码：

  ```c++
  class Foo {
  public:
  	int x;
  };

  class Bar {
  public:
    int x;
    Foo *operator->();
  };
  ```

  则，对`Bar->x`包装后访问的就是`Bar`中的x，而不是`Foo`中的`x`。

  ​



如果你的目的是仅仅在接口文件中暴露智能指针，就没必要包装智能指针类和底层对象的类了。但是，如果想让本节介绍的技术得以工作，你必须要告诉SWIG这两个类。只包装智能指针类的话，可以使用`%ignore`指令。例如：

  ```c++
  %ignore Foo;
  class Foo { // Ignored
  };

  class Bar {
  public:
    Foo *operator->();
    ...
  };
  ```

  还有一种方法是，你可以通过`%import`指令，导入独立的文件来导入`Foo`的定义。

> **注意:**当类定义了`operator->()`后，操作符本身被包装成`__deref__()`方法。例如：
>
> ```python
> f = Foo() # Smart-pointer
> p = f.__deref__() # Raw pointer from operator->
> ```

> **注意:**使用`%ignore`指令可以忽略`operator->()`。例如：
>
> ```c++
> %ignore Bar::operator->;
> ```

> **注意：**SWIG-1.3.14加入了对智能指针的支持。