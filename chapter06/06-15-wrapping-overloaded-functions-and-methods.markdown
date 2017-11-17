# 6.15 包装重载函数与方法

对于多数的语言模块，SWIG提供对重载函数、方法和构造函数的部分支持。例如，如果你提供给SWIG的重载函数像这样：

```C++
void foo(int x) {
	printf("x is %d\n", x);
}
void foo(char *x) {
	printf("x is '%s'\n", x);
}
```

这个函数以完全自然的方式被使用。如：

```python
>>> foo(3)
x is 3
>>> foo("hello")
x is 'hello'
>>>
```

重载对方法和构造函数使用类似的方式工作。例如，如果您有此代码:

```c++
class Foo {
public:
  Foo();
  Foo(const Foo &); // Copy constructor
  void bar(int x);
  void bar(char *s, int y);
};
```

它可能会像这样使用:

```python
>>> f = Foo() # Create a Foo
>>> f.bar(3)
>>> g = Foo(f) # Copy Foo
>>> f.bar("hello",2)
```

## 6.15.1 调度函数的生成

重载函数和方法的实现因脚本语言的动态性质变得有些复杂。不像C++在编译阶段绑定重载函数，SWIG必须在运行时为脚本语言目标选择合适使用的方法。这种检查因为某些脚本语言的无类型性质进一步变得复杂。例如，在Tcl中，所有的类型都是简单字符串。因此，如果你有两个像这样的重载函数：

```c++
void foo(char *x);
void foo(int x);
```

对参数顺序的检查起着相当关键的作用。

对于静态类型的语言，SWIG使用该语言自己提供的方法重载机制。为了实现对脚本语言的重载机制，SWIG会生成调度函数，检查传递的参数和它们的类型。为了创建这个函数，SWIG首先检查所有的重载函数，根据以下的规则赋予一定的等级：

1. **需要的参数个数。**方法是通过增加所需参数的数量来排序的。
2. **参数类型的优先级。**所有的C++数据类型都被赋予数字类型的优先级值（在语言模块文件中定义）。


|       类型       | 优先级  |  方向  |
| :------------: | :--: | :--: |
|     TYPE *     |  0   |  高   |
|     void *     |  20  |      |
|    Integers    |  40  |      |
| Floating point |  60  |      |
|      char      |  80  |      |
|    Strings     | 100  |  低   |

通过这些优先级的值，相同数目参数的重载函数在根据优先级计算得来的值进行排序。

这看着好像挺让人困惑的，但通过例子的解释可能更有帮助。考虑如下的一组重载函数：

```c++
void foo(double);
void foo(int);
void foo(Bar *);
void foo();
void foo(int x, int y, int z, int w);
void foo(int x, int y, int z = 3);
void foo(double x, double y);
void foo(double x, Bar *z);
```

第一条规则简单地通过参数个数划定等级。这将产生如下的列表：

```shell
rank
-----
[0] foo()
[1] foo(double);
[2] foo(int);
[3] foo(Bar *);
[4] foo(int x, int y, int z = 3);
[5] foo(double x, double y)
[6] foo(double x, Bar *z)
[7] foo(int x, int y, int z, int w);
```

第二个规则，简单地通过查看类型的优先级数值提炼等级，得到：

```shell
rank
-----
[0] foo()
[1] foo(Bar *);
[2] foo(int);
[3] foo(double);
[4] foo(int x, int y, int z = 3);
[5] foo(double x, Bar *z)
[6] foo(double x, double y)
[7] foo(int x, int y, int z, int w);
```

最后，生成调度函数，传递给重载方法的参数只需按照与此级别相同的顺序进行检查。

如果你仍然困惑，不要担心，SWIG会正确处理它。

## 6.15.2 重载中的歧义