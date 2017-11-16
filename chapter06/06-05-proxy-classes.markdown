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

