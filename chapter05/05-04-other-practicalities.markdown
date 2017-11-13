# 5.4 其他实用性建议

到目前为止，本章已经介绍了包装简单接口所需要学习的几乎所有的知识。但是，有些C程序使用一些难以映射到脚本语言接口的习惯用法。本节描述其中的一些问题。

## 5.4.1 通过结构体传递值

有时C函数接受由值传递的结构参数。例如，考虑以下函数：

```c
double dot_product(Vector a, Vector b);
```

为处理这个，SWIG创建一个包装函数，将原始函数转换成使用指针的方式：

```c++
double wrap_dot_product(Vector *a, Vector *b) {
	Vector x = *a;
 	Vector y = *b;
  
	return dot_product(x,y);
}
```

在目标语言中，`dot_product()`函数接受指向`Vector`的指针而不是`Vector`。大部分情况下，这种转换是透明地，不需要去关心。

## 5.4.2 返回的是值类型

返回结构体或类数据类型值的C语言函数更难处理。考虑如下函数：

```c++
Vector cross_product(Vector v1, Vector v2);
```

这个函数像返回`Vector`，但是SWIG只支持指针。结果，SWIG会创建这样的包装代码:

```c++
Vector *wrap_cross_product(Vector *v1, Vector *v2) {
  Vector x = *v1;
  Vector y = *v2;
  Vector *result;
  result = (Vector *) malloc(sizeof(Vector));
  *(result) = cross(x,y);
  
  return result;
}
```

如果使用了-C++选项：

```c++
Vector *wrap_cross(Vector *v1, Vector *v2) {
  Vector x = *v1;
  Vector y = *v2;
  Vector *result = new Vector(cross(x,y)); // Uses default copy constructor
  
  return result;
}
```

两种情况下，SWIG都会分配新的对象，并会返回它的引用。当不再使用时，需要用户删除它们。很显然，如果你没有意识到这个隐式的内存分配的话，就不会释放对象，从而带来内存泄露的问题。应该注意到一些语言模块现在可以自动跟踪新创建的对象，并为您恢复内存。请查阅每个语言模块的文档以获得更多的详细信息。

还应该注意的是，用C++处理值的传递/返回还有一些特殊情况。例如，如果`Vector`没有定义默认构造函数，上述代码片段就不能正常工作。SWIG与C++章节有关于这个情况的更多信息。

## 5.4.3 链接结构体变量

