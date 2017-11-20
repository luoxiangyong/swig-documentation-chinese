# 6.16 包装重载操作符

SWIG可以包装C++重载操作符。例如，考虑如下的类：

```c++
class Complex {
private:
	double rpart, ipart;
public:
  Complex(double r = 0, double i = 0) : rpart(r), ipart(i){ }
  
  Complex(const Complex &c) : rpart(c.rpart), ipart(c.ipart) { }
  
  Complex &operator=(const Complex &c) {
    rpart = c.rpart;
    ipart = c.ipart;
    return *this;
  }
  
  Complex operator+(const Complex &c) const {
      return Complex(rpart+c.rpart, ipart+c.ipart);
  }

  Complex operator-(const Complex &c) const {
      return Complex(rpart-c.rpart, ipart-c.ipart);
  }

  Complex operator*(const Complex &c) const {
    return Complex(rpart*c.rpart - ipart*c.ipart,rpart*c.ipart + c.rpart*ipart);
  }

  Complex operator-() const {
  	return Complex(-rpart, -ipart);
  }
  
  double re() const { return rpart; }
  double im() const { return ipart; }
};
```

当出现操作符时，处理他们的方式与处理常规方法一致。但，这些方法的名字被设置成类似`operator +`或`operator -`的形式。这些名字的问题是，在脚本语言中不合法。比如，不能再Python中调用`operator +`。

有些语言模块知道如何自动处理某些操作符（将它们映射到目标语言的操作符）。但是，底层的实现使用非常通用的方法，利用`%rename`指令。例如，在Python中：

```c++
%rename(__add__) Complex::operator+;
```

将+操作符绑定到`__add__`方法（Python中实现+操作符的惯用方法）。在内部，包装操作符的包装代码使用如下的类似代码：

```c++
_wrap_Complex___add__(args) {
  ... get args ...
  obj->operator+(args);
  ...
}
```

当在目标语言中使用时，可以像常规方式一样使用重载操作符。比如：

```python
>>> a = Complex(3,4)
>>> b = Complex(5,2)
>>> c = a + b 		# Invokes __add__ method
```

这里没有奇迹发生。只是使用`%rename`指令，选择了一个有效的名字而已。如果你写成这样：

```c++
%rename(add) operator+;
```

在脚本中就需要这样调用接口：

```python
a = Complex(3,4)
b = Complex(5,2)
c = a.add(b) 		# Call a.operator+(b)
```

前面介绍的用于重载函数的技术同样适用于操作符。例如：

```c++
%ignore Complex::operator=; // Ignore = in class Complex
%ignore *::operator=; // Ignore = in all classes
%ignore operator=; // Ignore = everywhere.
%rename(__sub__) Complex::operator-;
%rename(__neg__) Complex::operator-(); // Unary -
```

这个例子的后面演示了如何处理对`operator-`方法的多个定义。

使用这种方式处理重载操作符多数情况下都很直接。但是，有一些小问题需要记住：

+ 在C++中，为不同类型定义不同版本的操作符是很普遍的。例如，类可能还包含下面这样的友元函数：

  ```c++
  class Complex {
  public:
  	friend Complex operator+(Complex &, double);
  };

  Complex operator+(Complex &, double);
  ```

  SWIG会简单忽略所有的友元声明。因此，就不知道如何在类中关联`operator+`了（因为它不是类的成员）。

  任然可能对这个操作符进行包装，但是不得不把它当做普通函数进行处理。例如：

  ```c++
  %rename(add_complex_double) operator+(Complex &, double);
  ```

+ 某些操作符默认情况下是被忽略的。比如，`new`和`delete`操作符都被忽略了。

+ 某些操作符的语义在目标语言中并没有匹配的表示方法。



