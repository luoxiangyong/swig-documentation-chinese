# 1.8 向后兼容性

如果你是SWIG早期用户，请不要期望SWIG提供了完全的向后兼容性。尽管开发者们尽了极大努力保持向后兼容，但最主要的目的还是想让SWIG变的更好，两者并不能兼得。潜在的不兼容在[发行说明](#release-note)做了详细说明。

如果你使用了不同版本的SWIG，向后兼容就是个问题，可以使用`SWIG_VERSION`(记录当前执行的SWIG的版本号)预处理符号。`SWIG_VERSION`是类似0x010311一样的十六进制数字(代表SWIG-1.3.11)。可以使用它在接口文件中定义不同的typemap，可以像下面一样利用这个特征：

```c
#if SWIG_VERSION >= 0x010311
/* Use some fancy new feature */
#endif
```

>注意：SWIG生成的文件中并没有定义该符号。从SWIG-1.3.11开始，SWIG定义了`SWIG_VERSION`。
>
>

