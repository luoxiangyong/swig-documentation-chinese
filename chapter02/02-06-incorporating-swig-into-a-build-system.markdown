# 2.6 SWIG与自动构建系统的结合

SWIG是一个命令行工具，因而如此，它可以和任何支持调用外部工具/编译器的构建系统结合。多数情况下一班从Makefile中调用SWIG，但也可以从一些知名的IDE如Microsoft Visual Studio中调用。

如果你在项目中使用GNU的Autotools（Autoconf/Automake/Libtool）来配置SWIG，可以使用Autoonf的宏定义。主要的宏是`ax_pkg_swig`，从<http://www.gnu.org/software/autoconf-archive/ax_pkg_swig.html#ax_pkg_swig>可以了解详情。`x_python_devel`宏可用于生成Python扩展。查看[Autoconf归档](http://www.gnu.org/software/autoconf-archive/)可以了解这些宏定义的更多信息。

有一些构建工具也正在提供对SWIG的支持，例如[CMake](www.cmake.org)，一个跨平台、开源的构建管理系统，就提供了对SWIG的内建支持。CMake可以检测SWIG的可执行程序、目标语言的运行时库。CMake知道如何在很多不同类型的操作系统上创建共享库和可加载模块。这允许你容易地进行跨平台SWIG开发。它还可以自定命令，生成驱动IDE或makefile工作的相关文件。所有这些都可以在一个跨平台的输入文件中完成。下面这个例子就是一个CMake的输入文件，用来从SWIG接口文件example.i创建Python语言的包装代码：

```cmake
# This is a CMake example for Python
FIND_PACKAGE(SWIG REQUIRED)
INCLUDE(${SWIG_USE_FILE})
FIND_PACKAGE(PythonLibs)
INCLUDE_DIRECTORIES(${PYTHON_INCLUDE_PATH})
INCLUDE_DIRECTORIES(${CMAKE_CURRENT_SOURCE_DIR})
SET(CMAKE_SWIG_FLAGS "")
SET_SOURCE_FILES_PROPERTIES(example.i PROPERTIES CPLUSPLUS ON)
SET_SOURCE_FILES_PROPERTIES(example.i PROPERTIES SWIG_FLAGS "-includeall")
SWIG_ADD_MODULE(example python example.i example.cxx)
SWIG_LINK_LIBRARIES(example ${PYTHON_LIBRARIES})
```

> 新版本中_SWIG_ADD_MODULE_已经过时，改用_SWIG_ADD_LIBRARY_。

上面这个例子将会生成本机构建文件，如_makefiles_、_nmake files_或_Visual Studio_工程文件，调用SWIG并编译生成的C++ 文件，最终生成_example.so_（UNIX）或__example.pyd_[Windows]文件。其他的目标语言，生成Windows DLL文件而不是_.pyd_。