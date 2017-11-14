# 5.5 结构体与联合体

本节描述SWIG处理ANSI C结构体和联合体声明相关的行为。对处理C++声明的处理在下一章讲述。

当SWIG遇到结构体或联合体的定义时，它会创建一组访问函数。尽管SWIG不需要结构体的定义去定义接口，提供这些定义让它可以访问结构体的成员。SWIG生成的访问函数简单的拥有一个指向该对象的指针，并且允许访问私有成员。例如，下面的声明：

```c
struct Vector {
	double x,y,z;
}
```

将转换成下面一组访问函数：

```c
double Vector_x_get(struct Vector *obj) {
	return obj->x;
}
double Vector_y_get(struct Vector *obj) {
	return obj->y;
}
double Vector_z_get(struct Vector *obj) {
	return obj->z;
}
void Vector_x_set(struct Vector *obj, double value) {
	obj->x = value;
}
void Vector_y_set(struct Vector *obj, double value) {
	obj->y = value;
}
void Vector_z_set(struct Vector *obj, double value) {
	obj->z = value;
}
```

除此之外，如果没在接口文件中定义的话，SWIG还会创建默认的构造函数和析构函数。如：

```c
struct Vector *new_Vector() {
	return (Vector *) calloc(1,sizeof(struct Vector));
}
void delete_Vector(struct Vector *obj) {
	free(obj);
}
```

使用这些底层的访问函数，可以从目标语言中这样访问一个对象：

```python
v = new_Vector()
Vector_x_set(v,2)
Vector_y_set(v,10)
Vector_z_set(v,-5)
...
delete_Vector(v)
```

但是，多数的SWIG语言模块同时也提供更高层次的接口，使用它们更便利。接着往下读。

## 5.5.1 Typedef与structures

SWIG支持下面这样的在C中非常普遍的构造：

```c
typedef struct {
	double x,y,z;
} Vector;
```

当遇到这样的情况时，SWIG假设对象的名字是`Vector`，并像前面描述的一样创建访问函数。唯一的不同是，使用typedef允许SWIG在生成的代码中丢掉`struct`关键字。如：

```c
double Vector_x_get(Vector *obj) {
	return obj->x;
}
```

如果像下面这样使用了不同的名字：

```c
typedef struct vector_struct {
	double x,y,z;
} Vector
```

就会使用名字`Vector`而不是`vector_struct`，因为这才是更经典的C语言编程风格。如果，后来在接口文件中使用了类型`struct vector_struct`，SWIG就会知道这与`Vector`是一样的，它会生成合适的类型检查代码。

## 5.5.2 字符串与结构体

包含字符串的结构体需要特别注意。SWIG假设所有的`char*`类型都使用`malloc()`函数动态申请，并且都是以NULL结束的ASCII字符串。当这样的成员被修改时，先前的内容将会被释放，新内容将重新申请。如：

```c
%module mymodule
...
struct Foo {
	char *name;
	...
}
```

会创建如下的访问函数：

```c
char *Foo_name_get(Foo *obj) {
	return Foo->name;
}
char *Foo_name_set(Foo *obj, char *c) {
  	if (obj->name) free(obj->name);
  	obj->name = (char *) malloc(strlen(c)+1);
  	strcpy(obj->name,c);
	return obj->name;
}
```

如果你的程序行为不是这样的，SWIG的typemap `memberin`可用于改变其行为。查看typemap相关章节了解更多信息。

> **注意：**如果使用了-C++选项，`new`和`delete`会被用于执行内存分配。

## 5.5.2 数组成员

数组可以作为结构体的成员，但它们将变成只读的。SWIG将会创建一个访问函数，返回指向第一个数组元素地址的指针，但不会创建改变数组内容的函数。当遇到这样的情况时，SWIG会生成一个像下面这样的警告消息:

```shell
interface.i:116. Warning. Array member will be read-only
```

可以使用typemap消除这样的警告，这会在后面的章节介绍。多数情况下，这些警告消息没什么害处。

## 5.5.3 结构体作为数据成员

偶尔，结构体也会包含结构体成员。如：

```c
typedef struct Foo {
	int x;
} Foo;
typedef struct Bar {
	int y;
	Foo f; /* struct member */
} Bar;
```

当结构体成员被包装时，它被处理成指针，当时用`%naturalvar`指令后，它被处理成C++的引用。访问成员变量指针的函数包装成如下形式：

```c
Foo *Bar_f_get(Bar *b) {
	return &b->f;
}
void Bar_f_set(Bar *b, Foo *value) {
	b->f = *value;
}
```

这样做的原因有些微妙，但这样做是为了解决访问和修改内部数据。例如，假设你想修改`Bar`对象的`f.x`的值：

```c
Bar *b;
b->f.x = 37;
```

在脚本语言接口文件中，将其转换成赋值函数调用：

```c
Bar *b;
Foo_x_set(Bar_f_get(b),37);
```

在这个代码中，如果`Bar_f_get()`函数返回`Foo`而不是`Foo*`的话，结果会导致只对`f`的拷贝进行了修改，结构体的成员`f`并没被修改。很明显，这不是你想要的结果！

需要注意的是，只有SWIG知道数据成员是结构体或类的时候才会做这样的转换。例如，如果有下面这样的结构体：

```c
struct Foo {
	WORD w;
};
```

不知道`WORD`是什么类型，这是SWIG将会生成更通用的访问函数：

```c
WORD Foo_w_get(Foo *f) {
	return f->w;
}
void Foo_w_set(FOO *f, WORD value) {
	f->w = value;
}
```

> 兼容性注释：SWIG-1.3.11和早期的发行版将所有的非原生数据类型都当做指针进行转换。从SWIG-1.2.12开始，只有当数据类型明确是结构体、类或联合体的时候才会转换。这不太可能破坏现有的代码。但是，如果你需要告诉SWIG一个为定义的数据类型真的是一个结果体的话，简单的使用前置结构体声明`struct Foo;`就行了。

## 5.5.5 C构造器与析构器

当包装结构体时，通常创建和删除对象的机制一般会非常有用。如果你没有这样做，SWIG将会使用`malloc`和`free`自动生成创建和删除对象的函数。需要注意的是，在C代码中使用`malloc`，C++代码中使用`new`。

如果你不想让SWIG生成默认的构造函数，可以使用`%nodefaultctor`指令或-nodefaultctor命令行选项。如：

```shell
swig -nodefaultctor example.i
```

或：

```c
%module foo
...
%nodefaultctor; // Don't create default constructors
... declarations ...
%clearnodefaultctor; // Re-enable default constructors
```

如果你想精确控制，`%nodefaultctor`可用来选择单独的结构体定义。如：

```c
%nodefaultctor Foo; // No default constructor for Foo
...
struct Foo { // No default constructor generated.
};
struct Bar { // Default constructor generated.
};
```

因为忽略隐式的或默认的析构函数多数情况下会导致内存泄露，SWIG总是会生成它们。但是，如果需要的话，你可以使用`%nodefaultctor`有选择性的禁止生成默认/隐式的析构函数。

> **兼容性注释：**SWIG-1.3.7版之前，SWIG不会自动生成默认的构造函数和析构函数，除非你显式地使用-make_default选项打开它。但是，大部分的用户想要这样的特性，所以现在它是SWIG的默认行为。

> **注意：**可以使用-nodefault命令行选项和`%nodefault`指令，禁止默认或隐式的析构函数。这会导致语言间相互调用后存在内存泄露，所以强烈建议你不要使用它们。

## 5.5.6 向C结构体天剑成员函数

多数语言都提供创建类和支持面向对象编程的机制。从C语言的观点看，面向对象编程其实可以归结为给结构体关联上函数。这些函数一般对结构体的实例（对象）进行操作。尽管使用C++的方案更自然，但在C语言中没有直接实现这些特性的机制。但是，SWIG提供了特殊的`%extend`指令，使用它就可以关联方法到C结构体上,从而构建面向对象的接口。假设你有如下声明的C语言头文件:

```c
/* file : vector.h */
...
typedef struct Vector {
	double x,y,z;
} Vector;
```

你可以在SWIG接口文件中这样写，让`Vector`看起来更像一个类：

```c
%module mymodule

%{
#include "vector.h"
%}

%include "vector.h" // Just grab original C header file
  
%extend Vector { // Attach these functions to struct Vector
  Vector(double x, double y, double z) {
    
    Vector *v;
    v = (Vector *) malloc(sizeof(Vector));
    v->x = x;
    v->y = y;
    v->z = z;
    return v;
  }
  
  ~Vector() {
      free($self);
  }
  
  double magnitude() {
      return sqrt($self->x*$self->x+$self->y*$self->y+$self->z*$self->z);
  }
  
  void print() {
      printf("Vector [%g, %g, %g]\n", $self->x,$self->y,$self->z);
  }
};
```

注意，`$self`特殊变量的使用。它和C++语言的this指针的用法是一样的，可用在访问结构体实例的任何地方。同时还要注意，即使是C代码也使用了C++语言的构造函数和析构函数的语法来模拟构造器与析构器。尽管使用了通常的C++构造函数，也与普通的C++构造函数实现由稍许区别，新的被构造出来的对象必须返回对象，在这个例子中就是`Vector*`。

现在，当在Python中使用代理类，可以这么做：

```python
>>> v = Vector(3,4,0) 	 # Create a new vector
>>> print v.magnitude()  # Print magnitude
5.0
>>> v.print() 			# Print it out
[ 3, 4, 0 ]
>>> del v 				# Destroy it
```

`%extend`指令还可以用在`Vector`结构体的定义中。如：

```c
// file : vector.i
%module mymodule
%{
#include "vector.h"
%}
typedef struct Vector {
  double x,y,z;
  %extend {
  Vector(double x, double y, double z) { ... }
  ~Vector() { ... }
  ...
  }
} Vector;
```

注意，通过提供特定命名规则的函数，`%extend`还可以访问外部提供的函数：

```c
/* File : vector.c */
/* Vector methods */
#include "vector.h"

Vector *new_Vector(double x, double y, double z) {
  Vector *v;
  v = (Vector *) malloc(sizeof(Vector));
  v->x = x;
  v->y = y;
  v->z = z;
  return v;
}
void delete_Vector(Vector *v) {
	free(v);
}
double Vector_magnitude(Vector *v) {
	return sqrt(v->x*v->x+v->y*v->y+v->z*v->z);
}
```

```c
// File : vector.i
// Interface file
%module mymodule
%{
#include "vector.h"
%}
typedef struct Vector {
  double x,y,z;
  %extend {
  Vector(int,int,int); // This calls new_Vector()
  ~Vector(); // This calls delete_Vector()
  double magnitude(); // This will call Vector_magnitude()
  ...
  }
} Vector;
```

`%extend`使用的名字必须是原始结构体的名字而不能是用typedef定义的别名，如：

```c
typedef struct Integer {
  int value;
} Int;

%extend Integer { ... } /* Correct name */
%extend Int { ... } /* Incorrect name */

struct Float {
	float value;
};
typedef struct Float FloatValue;
%extend Float { ... } /* Correct name */
%extend FloatValue { ... } /* Incorrect name */
```

本规则有一个例外，当结构体以匿名方式命名后，如:

```c
typedef struct {
	double value;
} Double;
%extend Double { ... } /* Okay */
```

`%extend`有一个鲜为人知的特征，它还可以用于对存在的数据属性添加同步属性或修改其行为。例如，建设你想让`Vector`的`magnitude`变成只读的方法，可以这么做：

```c
// Add a new attribute to Vector
%extend Vector {
	const double magnitude;
}
// Now supply the implementation of the Vector_magnitude_get function
%{
const double Vector_magnitude_get(Vector *v) {
	return (const double) sqrt(v->x*v->x+v->y*v->y+v->z*v->z);
}
%}
```

现在，`magnitude`将变成对象的一个属性。

同样的技术还可以用于你想处理的数据成员。例如，考虑如下接口：

```c
typedef struct Person {
  char name[50];
  ...
} Person;
```

如果你想让`name`是大写的，可以像下面这样重写接口，确保无论何时它被读或写都能达此目的：

```c
typedef struct Person {
	%extend {
		char name[50];
	}
...
} Person;
%{
#include <string.h>
#include <ctype.h>
void make_upper(char *name) {
  char *c;
  for (c = name; *c; ++c)
  *c = (char)toupper((int)*c);
}
/* Specific implementation of set/get functions forcing capitalization */
char *Person_name_get(Person *p) {
  make_upper(p->name);
  return p->name;
}
void Person_name_set(Person *p, char *val) {
  strncpy(p->name,val,50);
  make_upper(p->name);
}
%}
```

最后需要强调的是，即使`%extend`指令可以被用来添加数据成员，这个新添加的成员在对象分配额外的存储空间(如，它们的值必须完全从结构的现有属性或其他地方获得)。

> **兼容性注释：**`%extend`指令是`%addmethods`指令的新名字。因为`%addmethods`不仅可以添加方法来扩展结构体，还可以做其他事情，所以更给它换了个更合适的名字。

## 5.5.7 内嵌的结构体

偶尔情况下，C程序中可能会包含向下面这样的结构体：

```c
typedef struct Object {
  int objtype;
  union {
    int ivalue;
    double dvalue;
    char *strvalue;
    void *ptrvalue;
  } intRep;
} Object;
```

当SWIG遇到这种情况时，它执行结构拆分操作，将声明转换为等价的如下操作：

```c
typedef union {
  int ivalue;
  double dvalue;
  char *strvalue;
  void *ptrvalue;
} Object_intRep;

typedef struct Object {
  int objType;
  Object_intRep intRep;
} Object;
```

SWIG会在接口文件中创建一个`Object_intRep`结构。同时，两个结构体的访问函数也会被创建。这种情况下，下面这些函数将会创建：

```c
Object_intRep *Object_intRep_get(Object *o) {
	return (Object_intRep *) &o->intRep;
}
int Object_intRep_ivalue_get(Object_intRep *o) {
	return o->ivalue;
}
int Object_intRep_ivalue_set(Object_intRep *o, int value) {
	return (o->ivalue = value);
}
double Object_intRep_dvalue_get(Object_intRep *o) {
	return o->dvalue;
}
... etc ...
```

虽然这个过程有点毛毛糙糙的，但它在目标脚本语言中工作的与您在所期望的一样——特别是当时候代理类以后。例如，在Perl中：

```perl
# Perl5 script for accessing nested member
$o = CreateObject(); # Create an object somehow
$o->{intRep}->{ivalue} = 7 # Change value of o.intRep.ivalue
```

如果你有很多的嵌入结果体声明，运行SWIG后，建议最好再检查一遍。尽管很有可能它们会工作，但在某些情况下，您可能需要修改接口文件。

最后，请注意在C++模式下嵌套处理的方式不同，请查看[内嵌类](#swig-nested-classes)。

## 5.5.8 关于结构体包装的其他注意事项

SWIG不关心在.i后缀的接口文件中声明的结构体精确匹配C底层使用的代码（嵌套结构的情况除外）。因为这个原因，省略问题成员或者干脆省略结构定义是没有问题的。如果你愿意传递指针，可以不给出结构体的定义。

从SWIG 1.3开始，SWIG代码生成器做了很多改进。特别是，尽管结构体的访问被描述成高层次的访问函数，如:

```c++
double Vector_x_get(Vector *v) {
	return v->x;
}
```

但生成的包装代码本身却是内联的。因此，生成的包装代码中其实没有`Vector_x_get()`这个函数。例如，当创建Tcl模块是，下面的函数被生成：

```c
static int
_wrap_Vector_x_get(ClientData clientData, Tcl_Interp *interp,int objc, Tcl_Obj *CONST objv[]) {
  struct Vector *arg1 ;
  double result ;
  
  if (SWIG_GetArgs(interp, objc, objv,"p:Vector_x_get self ",&arg0,SWIGTYPE_p_Vector) 
  							== TCL_ERROR)
  	return TCL_ERROR;
  	
  result = (double ) (arg1->x);
  Tcl_SetObjResult(interp,Tcl_NewDoubleObj((double) result));
  
  return TCL_OK;
}
```

这个规则唯一例外是使用`%extend`指令定义的方法。这种情况下，添加的代码包含在一个单独的函数中。

最后，需要注意到的是，多数语言模块可以选择创建更多的高级接口，这很重要。尽管你可能从来不使用这里介绍的低级接口，但SWIG的语言模块在其他地方以另外的方式总在使用。