# 11.3 模式匹配规则

本节描述C/C++类型关联typemap的模式匹配规则。实践中使用调试选项监测模式匹配规则也将会讲述。



### 11.3.1 基础匹配规则

Typemap同时使用类型和名称（参数名）进行匹配。对于给定`TYPE NAME`对，使用如下规则查找匹配。第一个找到的typemap的先被使用。

- 精确匹配 *TYPE*和*NAME*的typemap
- 仅仅精确匹配*TYPE*E*的typemap
- 如果*TYPE*是C++模板类型*T < TPARMS >*,*TPARMS *为模板参数，类型从模板参数中剔出来，使用如下的规则：
  - 精确匹配 *TYPE*和*NAME*的typemap
  - 仅仅精确匹配*TYPE*E*的typemap

如果*TYPE*包含修饰符（const、volatile等），一次去掉(strip)一个修饰符形成新的类型，使用上面的规则进行匹配。左边的修饰符最先被去掉，最右边的最后被去掉。例如`int const*const`第一次被去除修饰符后变成`int *const`，接下来变成`int *`。

如果*TYPE*是数组，使用如下的转换：

- 将所有的维度都替换为[ANY]，得到通用的数组typemap

  ​

为说明问题，假设有下面的代码：

```c
int foo(const char *s);
```

为了给`const char *s`找到合适的typemap，SWIG将搜索如下的typemap:

```c
const char *s // Exact type and name match
const char *  // Exact type match
char *s 	 // Type and name match (qualifier stripped)
char * 		 // Type match (qualifier stripped)
```

当找到多于一个的typemap时，只使用第一个匹配。下面这个例子展示了一些应用基础匹配规则的例子：

```c
%typemap(in) int *x {
	... typemap 1
}
%typemap(in) int * {
	... typemap 2
}
%typemap(in) const int *z {
	... typemap 3
}
%typemap(in) int [4] {
	... typemap 4
}
%typemap(in) int [ANY] {
	... typemap 5
}
void A(int *x); 			// int *x rule (typemap 1)
void B(int *y); 			// int * rule (typemap 2)
void C(const int *x); 		// int *x rule (typemap 1)
void D(const int *z); 		// const int *z rule (typemap 3)
void E(int x[4]); 			// int [4] rule (typemap 4)
void F(int x[1000]); 		// int [ANY] rule (typemap 5)
```

> 兼容性注释：SWIG-2.0.0引入一次剔除一个修饰符的规则。先前的版本一次将所有的修饰符都剔除了。



### 11.3.2 Typedef匹配规约(reduction)

如果使用前面的规则没有找到任何匹配，SWIG应用typedef匹配规约，然后在规约后的类型上继续使用一样的规则重复查找。为演示，假设有如下代码：

```c
%typemap(in) int {
	... typemap 1
}
typedef int Integer;
void blah(Integer x);
```

为找到`Integer x`的typemap，SWIG首先查找如下typemap:

```c
Integer x
Integer
```

没找到的话，使用`Integer -> int`规约，然后重复匹配：

```bash
int x
int --> match: typemap 1
```

即使通过typedef，两个类型是一样的，SWIG还是允许为它们分别定义不同的typemap。这个特性允许你对自己感兴趣的类型自定义单独的typemap。例如你写了如下代码：

```c
typedef double pdouble; // Positive double
// typemap 1
%typemap(in) double {
	... get a double ...
}
// typemap 2
%typemap(in) pdouble {
	... get a positive double ...
}
double sin(double x); 		// typemap 1
pdouble sqrt(pdouble x); 	// typemap 2
```

当规约类型时，一次应用一次typedef规约。匹配过程会一直进行下去，除非找到活没有更多的规约可用。

对于复杂类型，规约过程可能会生成一长串模式。考虑如下：

```c
typedef int Integer;
typedef Integer Row4[4];
void foo(Row4 rows[10]);
```

为匹配`Row4 rows[10]`参数，SWIG可能检查如下模式，直到它找到合适的匹配:

```ruby
Row4 rows[10]
Row4 [10]
Row4 rows[ANY]
Row4 [ANY]
# Reduce Row4 --> Integer[4]
Integer rows[10][4]
Integer [10][4]
Integer rows[ANY][ANY]
Integer [ANY][ANY]
# Reduce Integer --> int
int rows[10][4]
int [10][4]
int rows[ANY][ANY]
int [ANY][ANY]
```

对于像模板这样的参数化类型，情况更复杂。假设有如下的声明：

```c
typedef int Integer;
typedef foo<Integer,Integer> fooii;
void blah(fooii *x);
```

如下的typemap模式将会被搜索，用于匹配参数`fooii *x`：

```ruby
fooii *x
fooii *
# Reduce fooii --> foo<Integer,Integer>
foo<Integer,Integer> *x
foo<Integer,Integer> *
# Reduce Integer -> int
foo<int, Integer> *x
foo<int, Integer> *
# Reduce Integer -> int
foo<int, int> *x
foo<int, int> *
```

Typemap规约一般总是应用于最左边的类型。只有最左边的的不能匹配了才会向右规约。这种行为意味着你可以为`foo<int,Integer>`定义一个typemap,这样的话`foo<Integer,int>`typemap将永远不会匹配。这个技巧很少有人了解，实践中也很少有人这么干。当然，你可以使用这个技巧迷惑你的同事。

作为澄清，值得强调的是typedef匹配仅仅是typedef规约的过程，SWIG并不会搜索每一个可能的typedef。==假设声明了一个类型，它只会规约类型，不会在查找它的typedef定义==[^luoxiangyong-note-type-reduction]。例如，对于类型`Struct`，下面的typemap不会用于`aStruct`参数，因为`Struct`已经全部规约了。

```c
struct Struct {...};
typedef Struct StructTypedef;
%typemap(in) StructTypedef {
...
}
void go(Struct aStruct);
```

[^luoxiangyong-note-type-reduction]: 这段翻译的可能有点问题，大致的意思是说，一旦找到它的类型，关于该类型的其他typedef就不会再搜索了，针对这些typedef类型的定义的typemap也不会使用。



### 11.3.3 默认的typemap匹配规则

如果即使使用typedef规约，基础匹配规则也没有找到合适的匹配，SWIG就会使用默认的匹配规则查找合适的typemap。这些通用typemap基于`SWIGTYPE`基础类型。例如，指针使用`SWIGTYPE *`，参考使用`SWIGTYPE *`。更确切地说应该是，这些规则基于C++模板偏特化（template partial specialization）匹配规则，C++编译器使用这种规则查找合适的偏特化模板。这意味着匹配从一般typemap中选择最特化的版本使用。例如，当查找`int const *`时，根据规则，会在匹配`SWIGTYPE *`之前先匹配`SWIGTYPE const *`,会在匹配`SWIGTYP`之前先匹配`SWIGTYPE *`。

大多数的SWIG语言模块针对C语言的原始类型(primitive types)都定义了默认的typemap。这些定义全部都比较直接。例如，针对原始类型的值或const引用的列集可能如下编写：

```c
%typemap(in) int "... convert to int ...";
%typemap(in) short "... convert to short ...";
%typemap(in) float "... convert to float ...";
...
%typemap(in) const int & "... convert ...";
%typemap(in) const short & "... convert ...";
%typemap(in) const float & "... convert ...";
...
```

因为typemap匹配所有的typedef声明，通过值或const引用定义的任何原始类型的typedef都可以使用这些定义。绝大部分的目标语言模块同时还为char指针和char数组定义了typemap用以处理字符串，所以这样的非默认的类型也同样可以像原始类型一样使用基础的typemap，它们提供了比默认typemap更好的匹配规则。

下面是一组语言模块提供典型的默认类型，%typemap("in")的定义：

```c
%typemap(in) SWIGTYPE & { ... default reference handling ... };
%typemap(in) SWIGTYPE * { ... default pointer handling ... };
%typemap(in) SWIGTYPE *const { ... default pointer const handling ... };
%typemap(in) SWIGTYPE *const& { ... default pointer const reference handling ... };
%typemap(in) SWIGTYPE[ANY] { ... 1D fixed size arrays handlling ... };
%typemap(in) SWIGTYPE [] { ... unknown sized array handling ... };
%typemap(in) enum SWIGTYPE { ... default handling for enum values ... };
%typemap(in) const enum SWIGTYPE & { ... default handling for const enum reference values ... };
%typemap(in) SWIGTYPE (CLASS::*) { ... default pointer member handling ... };
%typemap(in) SWIGTYPE { ... simple default handling ... };
```

如果你想更改SWIG对简单指针的处理方式，你可以重新定义`SWIGTYPE *`。需要注意的是，简单默认的typemap规则用于匹配简单类型，不用用于匹配其他规则：

```c
%typemap(in) SWIGTYPE { ... simple default handling ... }
```

这个typemap非常重要，因为当调用或放回值类型时就会触发它。例如，如果你有如下声明：

```c
double dot_product(Vector a, Vector b);
```

`Vector`类型就会匹配`SWIGTYPE`。`SWIGTYPE`的默认实现就是讲值类型转换为指针类型（前面的章节讲过）。

通过重新定义`SWIGTYPE`类型，可以实现其他的行为。例如，如果你清除了所有针对`SWIGTYPE`的typemap，SWIG将不能包装未知的数据类型（这些类型可能对调试来说比较重要）了。然而，你可以修改`SWIGTYPE`,将对象列集为对象而不是转换为指针。

考虑如下的typemap定义，SWIG会为enum查找最佳的匹配，代码如下：

```c
%typemap(in) const Hello & { ... }
%typemap(in) const enum SWIGTYPE & { ... }
%typemap(in) enum SWIGTYPE & { ... }
%typemap(in) SWIGTYPE & { ... }
%typemap(in) SWIGTYPE { ... }

enum Hello {};
const Hello &hi;
```

那么最上面的typemap将会被选择，不因为它最先被定义，而是因为它是被包装类型的最贴切的匹配。如果上面的类型没有定义，就会选择使用接下来的类型。

探究默认typemap的最佳方式就是查看相关目标语言的模块定义。这些typemap的定义一般放在SWIG库路径下的java.swg，csharp.swg等文件中。但是，对许多目标语言来说，这些typemap定义都隐藏在复杂的宏定义中，因此，查看这些默认typemap的比较好地方式是查看预处理器的输出，可以通过在接口文件上运行`swig -E`命令达此目的。实践中，最好的方式是通过[调试typemap的模式匹配](#debugging-typemap-pattern-matching)选项，后面会讲到。

> 兼容性注释：默认的typemap匹配规则在SWIG-2.0.0版本做了调整，将简单的匹配方案调整为当前的使用C++的类模板偏特化匹配规则。

### 11.3.4 多个参数的typemap

当指定多个参数的typemap时，它的优先级要高于但给参数的typemap。例如：

```c
%typemap(in) (char *buffer, int len) {
	// typemap 1
}

%typemap(in) char *buffer {
	// typemap 2
}

void foo(char *buffer, int len, int count);  // (char *buffer, int len)
void bar(char *buffer, int blah); 			// char *buffer
```

多个参数的typemap写匹配限制更多。所有的类型和参数都必须匹配。

### 11.3.5 同C++模板的匹配方式的对比





### 11.3.6 调试typemap的模式匹配

<span id="debugging-typemap-pattern-matching" />

有两个用于调试的命令行参数可用于调试typemap，`-debug-tmsearch`和`-debug-tmused`。