# 11 Typemaps

## 11.1 介绍

可能有两个原因，您需要阅读本章：要么想自定义SWIG的行为；要么你听某些人含含糊糊的说<b>"typemaps"</b>
非常难懂的傻话，这时你可能会想"typemaps"到底是个什么鬼东西呢？既然这样，在开始之前我们做个
简单的澄清："typemaps"是SWIG提供的一种高级自定义特征(feature),通过它们我们可以直接访问底
层的(low-level)代码生成器。不仅如此，它们也是SWIG C++类型系统（该类型系统本省也是一个非凡
的主题）的一部分。一般情况下使用SWIG并不需要你理解typemaps。因此，如果你对SWIG默认情况下到
底做了什么并不是很清楚的情况下，建议你还是重新读一下前面的章节。


### 11.1.1 类型转换(type conversion)

在编程语言之间做数据类型的装换(conversion)或列集(marshalling)是包装代码生成器(wrapper 
code generator)的一项中非常重要的工作。对每一种c/C++的类型声明(declaration)，SWIG必须以
某种方式生成包装代码用以在语言间来回传递数据的值（value）。由于每种编程语言表达数据的方式不
同，因此不能简单将代码使用C连接器连接在一起。SWIG需要知道每种语言是如何表达数据的，并且需要
知道怎样去操作它们。

为说明问题，假设你有下面一段简单的C代码：

	```
	int factorial(int n);
	```

为了能从Python中访问该函数，需要使用一对Python API函数转换整形数据。例如：

	```
	long PyInt_AsLong(PyObject *obj); /* Python --> C */
	PyObject *PyInt_FromLong(long x); /* C --> Python */
	```
第一个函数用来将输入参数Python整形对象转换为C语言的long类型。第二个函数用来将C语言的long型
数值转换回Python的整形对象。

在包装函数内部，你可能看到类似下面的下面的函数：

	```
	PyObject *wrap_factorial(PyObject *self, PyObject *args) {
		int arg1;
		int result;
		PyObject *obj1;
		PyObject *resultobj;
		if (!PyArg_ParseTuple("O:factorial", &obj1)) return NULL;
		arg1 = PyInt_AsLong(obj1);
		result = factorial(arg1);
		resultobj = PyInt_FromLong(result);
		return resultobj;
	}
	```

SWIG支持的每一种目标语言都有类似的转换函数。比如，在Perl中，使用如下的函数：

	``` 
	IV SvIV(SV *sv); /* Perl --> C */
	void sv_setiv(SV *sv, IV val); /* C --> Perl */
	```

在TCL中：

	```
	int Tcl_GetLongFromObj(Tcl_Interp *interp, Tcl_Obj *obj, long *value);
	Tcl_Obj *Tcl_NewIntObj(long value);
	```

在这里，具体细节不是那么重要。重要的是，所有的底层类型转换都使用类似的功能函数，如果你对语言
扩展感兴趣可以阅读各种语言相关的文档，这些实验就留给你们自己去练习吧。

### 11.1.2 Typemaps

因为类型转换时代码包装生成器的中心工作，SWIG允许用户完全定义（或重定义）。为此，你可以使用特
殊的%typemap指令。例如：

	```
	/* Convert from Python --> C */
	%typemap(in) int {
		$1 = PyInt_AsLong($input);
	}
	
	/* Convert from C --> Python */
	%typemap(out) int {
		$result = PyInt_FromLong($1);
	}
	```

第一次看到这样的代码，你肯定会迷迷糊糊地。但这真的是没什么大不了的。