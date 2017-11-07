# 11.2 Typemap规范

本章主要描述%typemap指令的行为。

### 11.2.1 定义typemap

新的typemap使用%typemap指令定义。一般格式如下（以[...]包含的部分是可选的）：

```c
%typemap(method [, modifiers]) typelist code ;
```

*method*用于指定定义什么样的typemap，它是个简单的名字。这些名字通常像这样："in"，"out"， 或
"argout"。这些方法的目的后面会解释。

*modifiers*是一个可选的以逗号分隔，`name="value"`形式的列表。它们给typemap提供额外的信息，一般都是目标语言特有的，被称为typemap的属性(attibutes)。

*typelist*是typemap要匹配的C++数据类型模式的列表。一般形式描述如下:

```makefile
typelist : typepattern [, typepattern, typepattern, ... ] ;
typepattern : type [ (parms) ]
			| type name [ (parms) ]
			| ( typelist ) [ (parms) ]
```

每种类型模式可以是：简单类型、简单类型和参数名、多参数typemap类型的列表。除此之外，每个类型模式都可以参数化成暂存变量列表(params)。其用途后面也会简短描述。

*code*指定typemap的代码段。通常都是C/C++代码，像C\#和Java这样的静态类型目标语言，代码段可能包含目标语言代码。可以采用以下几种形式：

```makefile
code : 	 { ... }
		| " ... "
		| %{ ... %}
```

注意，预处理器会扩展{}界定符号里面的代码，另外两种格式的界定符号不做扩展，参考[预处理和typemap](#preprocessor-and-typemap)了解细节。下面是一些有效的typemap写法：

```c
/* Typemap with extra argument name */
%typemap(in) int nonnegative {
	...
}
/* Multiple types in one typemap */
%typemap(in) int, short, long {
	$1 = SvIV($input);
}
/* Typemap with modifiers */
%typemap(in,doc="integer") int "$1 = scm_to_int($input);";
/* Typemap applied to patterns of multiple arguments */
%typemap(in) (char *str, int len),
(char *buffer, int size)
{
  $1 = PyString_AsString($input);
  $2 = PyString_Size($input);
}
/* Typemap with extra pattern parameters */
%typemap(in, numinputs=0) int *output (int temp),
long *output (long temp)
{
	$1 = &temp;
}
```



### 11.2.2 Typemap作用域

Typemap一旦定义，跟在后面的所有声明都将使用这些规则。你可以在输入文件的需要的地方重新定义typemap。例如：

```c
// typemap1
%typemap(in) int {
	...
}
int fact(int); // typemap1
int gcd(int x, int y); // typemap1
// typemap2
%typemap(in) int {
	...
}
int isprime(int); // typemap2
```

对%extend特征指令，typemap的作用域规则不太一样。%extend用来给结构或类定义定义新的声明。因为如此，它使用在结构或类定义处定义的 typemap。举个例子：

```c
class Foo {
	...
};
%typemap(in) int {
	...
}
%extend Foo {
  int blah(int x); 	  // typemap has no effect. Declaration is attached to Foo which
  					// appears before the %typemap declaration.
};
```



### 11.2.3 拷贝typemap

使用赋值操作可以拷贝typemap。例如：

```c
%typemap(in) Integer = int;
```

或则：

```c
%typemap(in) Integer, Number, int32_t = int;
```

一种类型一般会有一组不同的typemap来控制。例如：

```c
%typemap(in) int { ... }
%typemap(out) int { ... }
%typemap(varin) int { ... }
%typemap(varout) int { ... }
```

为了拷贝这些typemap到新的类型，可以使用`%apply`指令。例如：

```c
%apply int { Integer }; 		// Copy all int typemaps to Integer
%apply int { Integer, Number };  // Copy all int typemaps to both Integer and Number
```

`%apply`使用与`%typemap`一样的规则，例如：

```c
%apply int *output { Integer *output }; // Typemap with name
%apply (char *buf, int len) { (char *buffer, int size) }; // Multiple arguments
```

### 11.2.4 删除typemap

要删除一个typemap，可以简单地将其代码段设为空，例如：

```c
%typemap(in) int; 				// Clears typemap for int
%typemap(in) int, long, short; 	 // Clears typemap for int, long, short
%typemap(in) int *output;
```

`%clear`指令可以清除指定类型的所有typemap，例如：

```c
%clear int; 					// Removes all types for int
%clear int *output, long *output;
```

> 因为SWIG的默认行为是使用typemap定义的，清除基础数据类型如`int`将会使该类型不可用，除非你在清除了后立马再定义一组新的typemap。

### 11.2.5 放置typemap

Typemap可以在全局作用域声明，也可以在C++命名空间、类声明等处。例如：

```c
%typemap(in) int {
	...
}

namespace std {
  class string;
      %typemap(in) string {
      ...
  }
}

class Bar {
	public:
    typedef const int & const_reference;
    	%typemap(out) const_reference {
    ...
    }
};
```

当typemap出现在命名空间或类中时，它的影响一直作用到输入文件结尾。但是，typemap的作用域是局部的。因此，这段代码：

```c
namespace std {
  class string;
    %typemap(in) string {
    ...
  }
}
```

就为`std::string`定义了typemap。你可能有如下代码：

```c
namespace std {
  class string;
  	%typemap(in) string { /* std::string */
  	...
  }
}

namespace Foo {
  class string;
    %typemap(in) string { /* Foo::string */
    ...
  }
}
```

在这个例子里，有两个完全不同的typemap应用于不同的类型(`std::string`和`Foo::string`)。

为了让作用域工作，SWIG需要知道`string`定义在特殊的命名空间。在这个例子中，可以使用`class string`前置声明达此目的。