# 6.12 传递和返回值类型

C++程序偶尔也会传递和返回对象的值类型。例如，可能有这样的函数:

```c++
Vector cross_product(Vector a, Vector b);
```

如果没有`Vector`的信息，SWIG将会生成类似下面这样的包装函数：

```c++
Vector *wrap_cross_product(Vector *a, Vector *b) {
  Vector x = *a;
  Vector y = *b;
  Vector r = cross_product(x,y);
  return new Vector(r);
}
```

为了使包装代码得以编译通过，`Vector`必须定义拷贝构造函数也默认构造函数。

如果`Vector`在接口文件中定义了，但是不支持默认构造函数，SWIG会将用特殊的C++模板包装器封装参数，生成包装代码，这个过程称之为"富尔顿变换（Fulton Transform）"。生成的包装看起来像这样：

```c++
Vector cross_product(Vector *a, Vector *b) {
  SwigValueWrapper<Vector> x = *a;
  SwigValueWrapper<Vector> y = *b;
  SwigValueWrapper<Vector> r = cross_product(x,y);
  return new Vector(r);
}
```

> 译者注释：这段代码有点问题。我想应该是这样的：
>
> ```c++
> Vector *cross_product(Vector *a, Vector *b) {
>   SwigValueWrapper<Vector> x = *a;
>   SwigValueWrapper<Vector> y = *b;
>   SwigValueWrapper<Vector> r = cross_product(x,y);
>   return new Vector(r);
> }
> ```

这种转变有点鬼鬼祟祟，但是它支持值的传递，即使类没有提供默认构造函数，它让其他的SWIG自定特性得到合适的支持。`SwigValueWrapper`可以通过阅读SWIG生成的包装代码了解。这个类就是对指针做了一层薄的封装，其他什么都没做。

尽管SWIG一般探测类是否要这样应用富尔顿变换，但是在某些情况下有必要覆盖这个特性。可使用`%feature("valuewrapper")`和`%feature("novaluewrapper")`开启或关闭此特性:

```c++
%feature("novaluewrapper") A;
class A;
%feature("valuewrapper") B;
struct B {
  B();
  // ....
};
```

 可以考虑为了那些有默认构造函数的类开启本特征。它将移除包装代码中在不变量声明处多余的构造函数调用，这将对大的对象和类生成更高性能的代码。另外一种方式是考虑返回引用或指针。

> **注意：**这个转换对typemap或SWIG的其他部分没有作用——它应该是透明的，除非你看了SWIG的输出文件外。

> **注意：**模板转换是SWIG-1.3.11新添加的，在将来的版本中可能重新定义。实践中，只有对没有定义默认构造函数的类才有决定对必要使用它。

> **注意：**对这种模板的使用只有当对象通过传值传递参数或返回值时才发生。使用C++引用或指针的代码不会使用它。