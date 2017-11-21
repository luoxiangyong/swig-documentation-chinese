# 6.21 异常的规格说明

当C++程序使用异常时，异常的行为有时候作为函数或方法的一部分被指定。例如：

```c++
class Error { };
class Foo {
public:
  ...
  void blah() throw(Error);
  ...
};
```

当以这种方式指定异常时，SWIG会自动生成包装代码，截获指定的异常，如果可能的话，重新抛出到目标语言中，或者转换成目标语言中的错误。比如，在Python中，可以这样编写代码：

```python
f = Foo()
try:
	f.blah()
except Error,e:
	# e is a wrapped instance of "Error"
```

关于如何裁剪代码处理C++异常，转换到目标语言中的异常/错误的处理机制等内容可参考["throws" typemap](#swig-throws-typemap)节。

因为有时候异常的处理非常保守，这些内容还不足以处理C++异常，还有一些特殊的SWIG指令可用此目的。参考[使用%exception指令处理异常](#swig-exception-handling-with-%exception)节获取更多信息。下一节介绍模拟异常规格或替换异常的方法。