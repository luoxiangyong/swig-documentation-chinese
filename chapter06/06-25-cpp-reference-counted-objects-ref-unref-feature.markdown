# 6.25 C++引用计数对象——ref/unref特征

另外一个C++中相似的概念是引用计数对象。考虑如下的代码：

```c++
class RCObj {
  // implement the ref counting mechanism
  int add_ref();
  int del_ref();
  int ref_count();
  
public:
  virtual ~RCObj() = 0;
  
  int ref() const {
  	return add_ref();
  }
  
  int unref() const {
    if (ref_count() == 0 || del_ref() == 0 ) {
    	delete this;
    	return 0;
  	}
	return ref_count();
  }
};

class A : RCObj {
public:
  A();
  int foo();
};

class B {
	A *_a;
public:
  B(A *a) : _a(a) {
  	a->ref();
  }
  
  ~B() {
  	a->unref();
  }
};

int main() {
  A *a = new A(); 	 // (count: 0)
  a->ref(); 		// 'a' ref here (count: 1)
  B *b1 = new B(a);  // 'a' ref here (count: 2)
  if (1 + 1 == 2) {
    B *b2 = new B(a); // 'a' ref here (count: 3)
    delete b2;		 // 'a' unref, but not deleted (count: 2)
  }
  delete b1; 		// 'a' unref, but not deleted (count: 1)
  a->unref(); 		// 'a' unref and deleted (count: 0)
}
```

上面的例子中，类`A`的实例`a`是一个引用计数对象，不能随便删除，因为它在对象`b1`和`b2`间引用。`A`从引用计数对象（***Reference Counted Object***）`RCObj`继承，`RCObj`实现了`ref()和unref()`惯用方法。

为了告诉SWIG，`RCObj`和所有从它继承的类是引用计数对象，使用`ref`和`ubref`特征。还可以使用1`%refobject`和`%unrefobject`。例如：

```c++
%module example
...
%feature("ref") RCObj "$this->ref();"
%feature("unref") RCObj "$this->unref();"
%include "rcobj.h"
%include "A.h"
...
```

当新对象传递到Python时，在`ref`和`unref`特征处指定的代码就会执行，当Python试图释放代理对象的实例时也一样。

在Python这一边，对引用计数对象的使用个与对其他对象的使用没什么两样：

```python
def create_A():
    a = A() # SWIG ref 'a' - new object is passed to python (count: 1)
    b1 = B(a) # C++ ref 'a (count: 2)
    if 1 + 1 == 2:
    b2 = B(a) # C++ ref 'a' (count: 3)
    return a # 'b1' and 'b2' are released and deleted, C++ unref 'a' twice (count: 1)

a = create_A() # (count: 1)
exit # 'a' is released, SWIG unref 'a' called in the destructor wrapper (count: 0)
```

注意，用户不必要显式地调用`a->ref()`或`a->unref()`（也没必要调用`delete a`）。取而代之的是，SWIG在需要的时候自动处理对它们的调用。如果用户不给类型指定***ref/unref***特征，SWIG会生成类似下面的代码：

```c++
%feature("ref") ""
%feature("unref") "delete $this;"
```

换句话说就是，当心对象传递到Python时，SWIG不会做任何特殊的事情，当Python释放代理实例是总会`delete`底层对象。

[%newobject特征](#swig-newobject-feature)被设计用来指导目标语言获取返回对象的所有权。当与带***ref***特征的类型一起使用时，额外会增加对***ref***特性的支持。考虑包装如下的工厂函数：

```c++
%newobject AFactory;
A *AFactory() {
	return new A();
}
```

`AFactory`函数现在表现的像调用`A`的构造函数一样来处理内存:

```python
a = AFactory() # SWIG ref 'a' due to %newobject (count: 1)
exit # 'a' is released, SWIG unref 'a' called in the destructor wrapper (count: 0)
```

