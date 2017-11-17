# 6.9 枚举与常量

对枚举与常量的处理不同语言模块处理方式不同，相关语言章节都有相机介绍。但是，多数语言将它们映射为类定义中的带类前缀常量。例如：

```c++
class Swig {
public:
	enum {ALE, LAGER, PORTER, STOUT};
};
```

在脚本语言中生成下面一组常量：

```c++
Swig_ALE = Swig::ALE
Swig_LAGER = Swig::LAGER
Swig_PORTER = Swig::PORTER
Swig_STOUT = Swig::STOUT
```

声明为const的成员被包装成只读的成员。