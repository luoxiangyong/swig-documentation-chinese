# 6.20 在命名空间中重命名模板类型

曾经提过，当`%rename`包含参数时，参数类型必须准确匹配（不执行typedef或命名空间解析）。SWIG对待模板类型的方式稍微不同，不像非模板类型，对模板类型有额外的匹配规则，不总是需要准确匹配。如果指定了全限定模板类型，它将比通用模板类型拥有更高的优先级。在下面的例子中，通用模板类型被用于重命名到`bbb`，全限定类型被用于重命名到`ccc`。

```c++
%rename(bbb) Space::ABC::aaa(T t); // will match but with lower precedence than ccc
%rename(ccc) Space::ABC<Space::XYZ>::aaa(Space::XYZ t);// will match but with higher precedence
// than bbb
namespace Space {
  class XYZ {};
   
  template<typename T> struct ABC {
    void aaa(T t) {}
  };
}
%template(ABCXYZ) Space::ABC<Space::XYZ>;
```

现在很明白，通过`%rename`有几种方式可达到重命名的效果。可以通过下面两个例子进行演示，同时也显式了`%rename`指令可以放在命名空间中：

```c++
namespace Space {
  %rename(bbb) ABC::aaa(T t); // will match but with lower precedence than ccc
  %rename(ccc) ABC<Space::XYZ>::aaa(Space::XYZ t);// will match but with higher precedence than bbb
  %rename(ddd) ABC<Space::XYZ>::aaa(XYZ t); // will not match
}
namespace Space {
  class XYZ {};
  template<typename T> struct ABC {
  	void aaa(T t) {}
  };
}
%template(ABCXYZ) Space::ABC<Space::XYZ>;
```

注意，`bbb`不会被匹配，因为没针对参数类型的命名空间解析，对模板类型的展开必须制定全限定类型。下面的例子显示了`%rename`是如何被用在`%extend`指令之中的：

```c++
namespace Space {
  %extend ABC {
  	%rename(bbb) aaa(T t); // will match but with lower precedence than ccc
  }
  
  %extend ABC<Space::XYZ> {
    %rename(ccc) aaa(Space::XYZ t);// will match but with higher precedence than bbb
    %rename(ddd) aaa(XYZ t); // will not match
  }
}
namespace Space {
  class XYZ {};
    template<typename T> struct ABC {
    	void aaa(T t) {}
    };
}
%template(ABCXYZ) Space::ABC<Space::XYZ>;
```

