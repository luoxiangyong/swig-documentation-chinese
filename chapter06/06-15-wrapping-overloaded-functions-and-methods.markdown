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

遗憾的是，SWIG并不能支持C++重载的所有可能的形式。考虑如下的代码：

```c
void foo(int x);
void foo(long x);
```

在C++中，这是合法的。但是，在脚本语言中，通常只有一种整形对象。因此，选择哪一个函数呢？很显然， 仅仅通过查看整数本身的值就没有办法真正地区别开来（int和long可能是相同的精度）。因此，当SWIG遇到这样的情况，它会生成像下面这样的警告消息：

example.i:4: Warning 509: Overloaded method foo(long) effectively ignored,
example.i:3: Warning 509: as it is shadowed by foo(int).

或者，像Java这样的静态语言：

example.i:4: Warning 516: Overloaded method foo(long) ignored,
example.i:3: Warning 516: using foo(int) instead.
at example.i:3 used.

这意味着第二个重载函数不能从接口中访问，或者这个方法根本就没被包装。之所以这样是因为，SWIG从前面的方法中不知道如何消除它的歧义。

在下列情况下会产生歧义的问题：

+ 整形转换。类似`int`,`long`和`short`这样的数据类型在一些语言中无法消除歧义。
+ 浮点转换。`float`和`double`在一些语言中无法辨识。
+ 指针与引用。比如: `Foo*`与`Foo&`。
+ 指针与数组。比如：`Foo*`与`Foo[4]`。
+ 指针与实例。比如,`Foo`与`Foo*`。注意：SWIG将所有的实例转换成指针。
+ 限定符。比如：`const Foo*` 与`Foo*`
+ 默认与非默认参数。比如：`foo(int a, int b)`与`foo(int a, int b = 3)`

当出现歧义时，方法以与接口文件中显示的顺序相同的方式进行检查。因此，前面的方法将隐藏出现在后面的方法。

当包装重载函数时，有可能得到这样的警告消息：

example.i:3: Warning 467: Overloaded foo(int) not supported (incomplete type checking rule -
no precedence level in typecheck typemap for 'int').

此错误意味着目标语言模块支持重载，但由于某种原因，没有可用于生成调度函数的类型检查规则。由此产生的行为是未定义的。如果是SWIG提供的typemap导致的问题，你应该向[SWIG bug tracking database](http://www.swig.org/bugs.html)报告这样的bug。

如果你得到如下的错误：

foo.i:6. Overloaded declaration ignored. Spam::foo(double )
foo.i:5. Previous declaration is Spam::foo(int )
foo.i:7. Overloaded declaration ignored. Spam::foo(Bar *,Spam *,int )
foo.i:5. Previous declaration is Spam::foo(int )

这意味着目标语言模块还没有实现对重载函数和方法的支持。解决问题的唯一方法是阅读下一节。

## 6.15.3 歧义消解与重命名

如果在重载解析中出现歧义，或者如果模块不允许重载，那么有几个解决问题的策略。首先，可以告诉SWIG忽略其中的方法。这很简单，使用`%ignore`指令就可以。比如：

```c++
%ignore foo(long);
void foo(int);
void foo(long); // Ignored. Oh well.
```

另一种方法是重命名其中一种方法。使用`%rename`指令，比如：

```c++
%rename("foo_short") foo(short);
%rename(foo_long) foo(long);

void foo(int);
void foo(short); // Accessed as foo_short()
void foo(long);  // Accessed as foo_long()
```

注意，新名称周围的引号是可选的，但是，如果新名称是C/C++的关键字，那么为了避免分析错误，引号将是必不可少的。`%ignore`和`%rename`指令都很强大，它们都有能力匹配任何声明。当使用简单形式时，它们将应用于全局函数和方法。例如：

```c++
/* Forward renaming declarations */
%rename(foo_i) foo(int);
%rename(foo_d) foo(double);
...
void foo(int); // Becomes 'foo_i'
void foo(char *c); // Stays 'foo' (not renamed)
class Spam {
public:
  void foo(int); // Becomes 'foo_i'
  void foo(double); // Becomes 'foo_d'
  ...
};
```

如果你只想将重命名应用于某个作用域，可以使用C++作用域解析操作符(::)。如：

```c++
%rename(foo_i) ::foo(int); // Only rename foo(int) in the global scope.
// (will not rename class members)
%rename(foo_i) Spam::foo(int); // Only rename foo(int) in class Spam
```

当重命名操作符应用到类似`Spam::foo(int)`的类上时，它将在所有的`Spam`类及派生类上起作用。这可以用于在只有一个声明的整个类层次结构上应用一致的重命名方案。如：

```c++
%rename(foo_i) Spam::foo(int);
%rename(foo_d) Spam::foo(double);

class Spam {
public:
  virtual void foo(int); // Renamed to foo_i
  virtual void foo(double); // Renamed to foo_d
  ...
};
class Bar : public Spam {
public:
  virtual void foo(int); // Renamed to foo_i
  virtual void foo(double); // Renamed to foo_d
  ...
};

class Grok : public Bar {
public:
  virtual void foo(int); // Renamed to foo_i
  virtual void foo(double); // Renamed to foo_d
...
};
```

还可以在类定义中包含`%rename`规则说明。比如：

```c++
class Spam {
%rename(foo_i) foo(int);
%rename(foo_d) foo(double);
public:
  virtual void foo(int); // Renamed to foo_i
  virtual void foo(double); // Renamed to foo_d
  ...
};

class Bar : public Spam {
public:
  virtual void foo(int); // Renamed to foo_i
  virtual void foo(double); // Renamed to foo_d
  ...
};
```

这种情况下，`%rename`指令依然应用到整个继承体系中，但是这样就不必再显式指定类前缀`Spam::`了。

可以将`%rename`指令的特殊形式只应用到类的成员（对所有的类）:

```c++
%rename(foo_i) *::foo(int); // Only rename foo(int) if it appears in a class.
```

> **注意：**`*::`语法不是标准的C++语法，但是，*号表示展开，可以匹配任何类的名字(我们想不到其他的替换方法，所以如果你有更好的主意，请给[swig-devel mailing list](http://www.swig.org/mail.html)发邮件)。

尽管我们在这里主要讨论`%rename`，但所有的规则都适用于`%ignore`。比如：

```c++
%ignore foo(double);		// Ignore all foo(double)
%ignore Spam::foo; 			// Ignore foo in class Spam
%ignore Spam::foo(double); 	 // Ignore foo(double) in class Spam
%ignore *::foo(double); 	// Ignore foo(double) in all classes
```

当应用到基类上时，`%ignore`将使得派生类中所有定义都将消失。比如，`%ignore Spam::foo(double)`将排除类`Spam`和所有派生自`Spam`类中的`foo(double)`函数。



**关于`%rename`和`ignore`需要注意的地方：**

+ 因为`%rename`声明用于对重命名的预先声明，它可以放在接口文件的开始位置。这使得应用统一、一致地命名方案变得可能，还不用修改头文件。比如：

  ```c++
  %module foo
  /* Rename these overloaded functions */
  %rename(foo_i) foo(int);
  %rename(foo_d) foo(double);

  %include "header.h"

  ```

+  范围限定符(::)也可以用到简单的名字上。比如：

   ```c++
  %rename(bar) ::foo; // Rename foo to bar in global scope only
  %rename(bar) Spam::foo; // Rename foo to bar in class Spam only
  %rename(bar) *::foo; // Rename foo in classes only
   ```

+ 名称匹配试图找到定义的最匹配的匹配项。像`Spam::foo`这样的限定名总是拥有比非限定名`foo`更高的优先级。`Spam::foo`优先级最高、其次是`*::foo`，然后是`foo`。在同一作用域范围内，参数化名字比非参数化名字的优先级更高。但是，带范围限定符的非参数化名字比在全局作用域中的参数化名字的优先级要高（比如：`Spam::foo`比`foo(int)`的优先级要高）。

+ 只要它在要命名的声明前，`%rename`指令出现的位置顺序不重要。因此，下面这样没区别：

  ```c++
  %rename(bar) foo;
  %rename(foo_i) Spam::foo(int);
  %rename(Foo) Spam::foo;
  ```

  和：

  ```c++
  %rename(Foo) Spam::foo;
  %rename(bar) foo;
  %rename(foo_i) Spam::foo(int);
  ```

  （这些声明没有存储在链表中，顺序不重要）。当然，如果指定了精确的名字、范围和参数，，重复的`%rename`指令会改变先前的指令。

+ 对于多个继承，重命名规则又在多个基类中定义，使用深度优先规则在类层次结构中找到的第一个重命名规则将被使用。

+ 名字匹配规则严格遵循成员限定规则。比如，如果有这样的类：

  ```c++
  class Spam {
  public:
    ...
    void bar() const;
    ...
  };
  ```

  声明：

  ```c++
  %rename(name) Spam::bar();
  ```

  将不会应用到未限定的成员`bar()`上。下面的将会被正确应用，因为匹配限定符：

  ```c++
  %rename(name) Spam::bar() const;
  ```

  通常被忽略的C++特性是类定义了两个不同的重载成员，但它们只是限定符中有所不同，如下所示：

  ```c++
  class Spam {
  public:
    ...
    void bar(); // Unqualified member
    void bar() const; // Qualified member
    ...
  };
  ```

  可以使用`%rename`分别给它们命名。比如：

  ```c++
  %rename(name1) Spam::bar();
  %rename(name2) Spam::bar() const;
  ```

  同样，如果你非常想忽略其中之一，使用`%ignore`并制定全限定符。比如，下面的指令会告诉SWIG忽略const版本的`bar()`函数：

  ```c++
  %ignore Spam::bar() const; // Ignore bar() const, but leave other bar() alone
  ```

+ 当前，没有对函数的参数进行分辨。这就意味着函数的参数类型必须精确匹配。比如，命名空间限定符和typedef不能工作。下面使用typedef的代码演示了这种情况：

  ```c++
  typedef int Integer;

  %rename(foo_i) foo(int);

  class Spam {
  public:
  	void foo(Integer); // Stays 'foo' (not renamed)
  };

  class Ham {
  public:
  	void foo(int); // Renamed to foo_i
  };
  ```

+ 在包装具有默认参数的方法时，名称匹配规则还使用默认参数进行更精细的控制。回想一下，默认参数的方法被包装起来，就像等效重载方法已经被解析一样。让我们考虑下面的示例类：

  ```c++
  class Spam {
  public:
    ...
    void bar(int i=-1, double d=0.0);
    ...
  };
  ```

  下面的`%rename`方法将会精确匹配并应用到所有目标语言的重载函数上，因为带有默认参数的声明精确匹配了包装方法：

  ```c++
  %rename(newbar) Spam::bar(int i=-1, double d=0.0);
  ```

  可以在目标语言中调用C++方法，而不管提供了多少参数，比如：`newbar(2, 2.0)`，`newbar(2)` 或` newbar()`。但是，如果`%rename`不包含默认参数，将只适用于单一等效的目标语言重载方法。所以，如果我们使用这样的代码：

  ```c++
  %rename(newbar) Spam::bar(int i, double d);
  ```

  C++的方法必须在目标语言中使用新的名字`newbar(2, 2.0)`，提供两个参数；或者使用原始名字`bar(2)`(一个参数)；或者`bar()`（没有参数）。事实上，可以在等价的重载方法上使用`%rename`，像所有等价的重载函数都重新命名一遍：

  ```c++
  %rename(bar_2args) Spam::bar(int i, double d);
  %rename(bar_1arg) Spam::bar(int i);
  %rename(bar_default) Spam::bar();
  ```

  同样，额外的重载方法可以使用`%ignore`指令有选择的忽略。

  > **兼容性注释：**`%rename`指令引入默认参数的匹配规则在SWIG-1.3.23时被添加进来，与此同时，还引入了对默认参数方法的包装。

## 6.15.4 关于重载的注释

对重载的支持从SWIG-1.3.14开始被第一次添加进来。但实现方式与其他类似工具不太一样。比如，声明出现的顺序无关紧要。因此，SWIG不依赖实验性执行代码或异常处理来找出到底要执行哪个方法。

在内部，重载机制完全在目标语言的模块中配置。因此，对重载的支持可能不同语言间差别很大。作为一个通用的规则，静态类型的语言如Java比动态的类型语言如Perl、Python、Ruby和Tcl提供了更多的支持。