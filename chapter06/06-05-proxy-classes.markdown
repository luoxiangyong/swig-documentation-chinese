# 6.5 代理类

为了以更自然的方式将C++类映射到目标语言中，SWIG支持的目标语言多数情况下都会通过代理类来包装C++类。这些代理类典型情况下都使用目标语言实现。例如，如果你正在创建Python模块，那么每个C++类都会用一个相关的Python类来代理。或者，如果你正在创建一个Java模块，每个C++类也会使用一个相关的Java代理类来实现。

## 6.5.1 代理类的构建

代理类一般总是通过一层额外的包装层实现，该层使用底层的访问函数。为示意，假设你有一个C++类如下：

```c++
class Foo {
public:
  Foo();
  ~Foo();
  int bar(int x);
  int x;
};
```

使用C++伪代码，代理类可能像这样：

```c++
class FooProxy {
private:
	Foo *self;
public:
  FooProxy() {
  	self = new_Foo();
  }
  ~FooProxy() {
    delete_Foo(self);
  }
  int bar(int x) {
  	return Foo_bar(self,x);
  }
  int x_get() {
  	return Foo_x_get(self);
  }
  void x_set(int x) {
  	Foo_x_set(self,x);
  }
};
```

当然，一定要记住：代理类是使用目标语言实现的。例如，在Python 中，代理类长的可能想下面这样：

```python
class Foo:
    def __init__(self):
    	self.this = new_Foo()
        
    def __del__(self):
    	delete_Foo(self.this)
        
    def bar(self,x):
    	return Foo_bar(self.this,x)
    def __getattr__(self,name):
    	if name == 'x':
   	 		return Foo_x_get(self.this)
    	...
    
    def __setattr__(self,name,value):
   	 	if name == 'x':
    		Foo_x_set(self.this,value)
    		...
```

再说一遍，重点强调一点底层的访问函数总是被代理类使用。无论什么时候，代理类都尽量使用与C++语言特性相似的的高级语言特性。这包括操作符重载、异常处理和其他特性。

## 6.5.2 代理类中的资源管理

代理类中主要问题是被包装对象的内存管理。有如下的C++代码：

```c++
class Foo {
public:
  Foo();
  ~Foo();
  int bar(int x);
  int x;
};

class Spam {
  public:
  Foo *value;
  ...
};
```

下面是脚本语言使用它们的方式：

```python
f = Foo() # Creates a new Foo
s = Spam() # Creates a new Spam
s.value = f # Stores a reference to f inside s
g = s.value # Returns stored reference
g = 4 # Reassign g to some other value
del f # Destroy f
```

现在，思考产生的内存管理问题。当对象在脚本中创建时，对象被新创建的代理类包装起来。也就是说，有一个新的代理类实例和一个底层C类的新实例。在这个例子中，f和s都是用这种方式创建的。然而，声明的`s.value`值得关注——执行时，指向f的指针存储在另一个对象中。这意味着脚本代理类和另一个C++类共享对同一对象的引用。为了使事情更有趣，考虑语句`g = s.value`。在执行时，这将创建一个新的代理类g，它对存储在`s.value`中的C++对象提供了一层包装。通常情况下，没有办法知道对象从哪儿来——它可能从脚本中创建，但也可能是内部生成的。在这个特殊的例子中，对g的赋值是f的第二个代理类的结果。换句话说，对f的引用现在由两个代理类和一个C类共享。

最后，考虑当对象被释放时会发生什么。在语句`g=4`中，变量g被重新赋值。在多数语言中，这使得旧的`g`变量被垃圾回收。当然，仍然有一个对存储在另一个C++对象中的原始对象的引用。发生什么事了？对象仍然有效吗？

为了处理内存管理问题，代理类提供了一个控制对象所有权的的API。使用 C++伪代码，所有权控制可能看起来是这样的:

```c++
class FooProxy {
public:
  Foo *self;
  
  int thisown;
  
  FooProxy() {
    self = new_Foo();
    thisown = 1; // Newly created object
  }
  
  ~FooProxy() {
      if (thisown) delete_Foo(self);
  }
  ...
  // Ownership control API
  void disown() {
  	thisown = 0;
  }
  void acquire() {
  	thisown = 1;
  }
};

class FooPtrProxy: public FooProxy {
public:
  FooPtrProxy(Foo *s) {
    self = s;
    thisown = 0;
  }
};

class SpamProxy {
...
  FooProxy *value_get() {
  	return FooPtrProxy(Spam_value_get(self));
  }
  
  void value_set(FooProxy *v) {
    Spam_value_set(self,v->self);
    v->disown();
  }
  ...
};
```

分析这段代码，有以下几个中心特征：

+ 每个代理类保有标示对象拥有权的额外标志。只有当所有权标志设置时才能释放C++对象。
+ 当在目标语言中创建了新的对象，所有权标志就会被设置。
+ 当返回内部C++对象的引用是，它被包装成代理类，但代理类没有该引用的所有权。
+ 在某些情况下，所有权被调整。例如，当值被赋值给类的成员时，失去所有权。
+ 可使用`disown()`和`acquire()`方法手动调整所有权。

考虑到C内存管理的棘手特性，代理类不可能自动处理所有可能的内存管理问题。然而，代理确实提供了一种手动控制机制，可以用于（如果需要的话）解决一些更棘手的内存管理问题。

> 译者注解：上面这个例子需要大家仔细揣摩、细细评味，几乎所有的脚本语言、高级静态语言如Go、C#、Java等都提供自动垃圾回收机制，了解对象所有权很重要！



## 6.5.3 语言特定的细节

关于代理类的特定细节在每个目标语言相关章节有详细介绍。本章只是用比较通用的方式大致介绍了代理类的相关主题。