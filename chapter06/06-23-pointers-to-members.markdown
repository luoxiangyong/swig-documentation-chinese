# 6.23 指向成员的指针

从SWIG-1.3.7开始，对C++的类成员指针提供了有限的支持。例如：

```c++
double do_op(Object *o, double (Object::*callback)(double,double));
extern double (Object::*fooptr)(double,double);
%constant double (Object::*FOO)(double,double) = &Object::foo;
```

尽管这种类型的指针能被解释，并能用SWIG类型系统表示，很少有语言模块知道如何处理他们，因为用标准C指针的方式实现它的方式不一样。强烈建议读者参考这方面的高级材料如《The Annotated C++ Manual》获取特定的细节。

当支持成员指针后，指针的值看起来比较特别：

```python
>>> print example.FOO
_ff0d54a800000000_m_Object__f_double_double__double
>>>
```

这种情况下，十六进制数字表示了指针的全部值，在多数机器上是一个小的C++结构的内容。

当使用成员指针时，SWIG的类型系统检查机制也非常受限。当检查类型时，一般SWIG会尽量跟踪继承结构。但是，对成员指针来说还不支持这样的功能。