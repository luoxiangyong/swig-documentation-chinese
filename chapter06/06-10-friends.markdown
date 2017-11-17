# 6.10 友元

SWIG识别友元声明。例如，如果有这样的代码：

```c++
class Foo {
public:
  ...
  friend void blah(Foo *f);
  ...
};
```

则对友元声明生成的代码与下面声明的生成代码的方式一样：

```c++
class Foo {
public:
	...
};

void blah(Foo *f);
```

在C++中，友元声明的作用域与其声明的类的作用域相同，因此：

```c++
%ignore bar::blah(Foo *f);

namespace bar {
  class Foo {
  public:
    ...
    friend void blah(Foo *f);
    ...
  };
}
```

将忽略对`blah`的包装。