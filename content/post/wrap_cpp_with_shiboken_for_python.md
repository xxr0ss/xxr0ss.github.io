---
title: "用Shiboken封装简单的C++类，实现Python对C++的调用"
date: 2021-03-15T16:06:53+08:00
ShowToc: true
---


Shiboken是[Qt For Python](https://doc.qt.io/qtforpython/contents.html)实现的根基，它分析C++代码抽取其中的信息，产生CPython代码，可以直接编译为python的模块，使得C/C++代码能用在Python里面去。

本文将介绍在Windows平台上，使用Shiboken6封装C++的具体操作，以及刚开始接触Shiboken可能碰到的一些坑。

## Shiboken的基本使用逻辑
1. 把该C++项目编译成二进制文件（如Windows下是`dll`）
2. 将现有需要导出给Python的C++类声明在一个xml里，这个xml叫typesystem
3. 利用这个xml和C++项目的头文件，生成绑定代码，其中包含`模块名_module_wrapper`和导出要用的头文件（已在typesystem中定义好了）的`类_wrapper`
4. 编译生成的wrapper代码，与先前生成的`dll`链接，生成`.pyd`（本质上是dll，可直接在python中import）
5. 在Python中直接导入模块使用


下面介绍
## 具体操作细节
### 环境准备
按照PySide6安装目录下官方提供的示例`examples/samplebindings`（例如，在我机器上是：`D:\Python\Python39\Lib\site-packages\PySide6\examples\samplebinding`)中的使用习惯，我们
* 使用`bindings.h`保存要封装的C++类的头文件，用`binidngs.xml`保存typesystem信息。
* 使用CMake和NMake进行项目构建编译等自动控制，请确保你电脑上安装了VS2019。
	* 由于要使用nmake，所以需要使用x64 Native Tools Command Prompt for VS 2019，这个是随VS2019安装自动安装的，本质上就是执行了一个bat，让你的cmd环境变量里多了VS提供的一些开发工具（比如nmake

在这个示例的目录下，如你所见，官方提供了两个示例，一个是samplebindings，演示如何封装简单的C++类；另一个是widgetbinding，演示如何封装Qt库，需要你机器上装了Qt开发环境。
![在这里插入图片描述](/wrap_cpp_with_shiboken_for_python.assets/pyside6_examples.png)
本文使用我自己提供的一份示例代码，演示如何封装C++类，算是对官方示例的对照和补充吧。可从我的GitHub仓库下载： [shiboken6_demo](https://github.com/xxr0ss/shiboken6_demo)

文件结构：
```bash
shiboken_simple
 ├── bindings
 │   ├── bindings.h
 │   ├── bindings.xml
 │   ├── pyside_config.py
 │   └── wrap_src_gen.py
 ├── build
 ├── CMakeLists.txt
 └── math
     ├── CMakeLists.txt
     ├── mathematician.cpp
     └── mathematician.h
```

顶层CMakeLists.txt（shiboken_simple/CMakeLists.txt）负责大部分工作：
* 调用shiboken生成封装代码（wrapper）和编译封装代码
* 链接封装后的pyd模块和我们将C++代码直接生成的dll
* 将pyd和相关依赖复制到（install）shiboken_simple目录下，可以直接编写python代码调用

math是我们的C++项目，其中的CMakeLists.txt工作很简单，就是负责将C++代码构建dll
```cmake
set(${cppmath_library}_sources
    ${CMAKE_CURRENT_SOURCE_DIR}/mathematician.cpp
)

add_library(${cppmath_library} SHARED ${${cppmath_library}_sources})
add_compile_definitions(BINDINGS_BUILD)
```
其中的`${cppmath_library}`变量定义在顶层CMakeLists.txt

### C++项目代码
很简单，功能也比较单一的一个C++类，头文件mathematician.h定义：
```cpp
#if !defined(MYMATH_H)
#define MYMATH_H

#if defined _WIN32 || defined __CYGWIN__
    #if BINDINGS_BUILD
        #define BINDINGS_API __declspec(dllexport)
    #else
        #define BINDINGS_API __declspec(dllimport)
    #endif
#else
    #define BINDINGS_API
#endif

class BINDINGS_API Mathematician{
public:
    Mathematician();
    ~Mathematician();

    void inc();
    int getCount();

private:
    int m_count = 0;
};

#endif // MYMATH_H
```
值得注意的是`__declspec(dllimport)`，最终是类名前的`BINDINGS_API`定义，在Windows下，不声明一个类导出的话，MSVC编译是不产生`.lib`文件的，也将直接导致后面与shiboken生成的绑定代码链接时无法链接，最后nmake的时候就会报错
```bash
NMAKE : fatal error U1073: 不知道如何生成“math\libmath.lib”
```
这个头文件还有一个点是`#if BINDINGS_BUILD`这句，`BINDINGS_BUILD`是CMakeLists.txt里通过`add_compile_definitions(BINDINGS_BUILD)`给加上的。

然后是mathematician.cpp：
```cpp
#include "mathematician.h"

Mathematician::Mathematician(){
    m_count = 0;
}

Mathematician::~Mathematician() {

}

void Mathematician::inc() {
    m_count ++;
}

int Mathematician::getCount() {
    return m_count;
}
```


### 为shiboken准备必要信息
在bindings.h中包含我们要导出的C++类所在头文件（如果一个C++类不需要封装给Python，则在这里和typesystem的xml里都不需要把它写出来）
```cpp
#ifndef BINDINGS_H
#define BINDINGS_H

#include "../math/mathematician.h"

#endif // BINDINGS_H
```
然后是我们要导出的类`Mathematician`，由于没啥特殊修改，简单地在bindings.xml里面进行声明即可：
```xml
<?xml version="1.0"?>
<typesystem package="Cppmath">
    <primitive-type name="int"/>
    <object-type name="Mathematician"></object-type>
</typesystem>
```
`package`是最终Python里import的模块名，`object-type`里就是我们要导出的类了。我们使用了C++的数据类型`int`，在这里我们声明了一个`<primitive-type>`。否则在生成封装代码时会出现类似下面这样的报错：
```
skipping function 'int Mathematician::getCount()', unmatched return type 'int': Unable to translate type "int": Cannot find type entry for "int".
```


### CMake构建
然后是重头戏了，官方示例里CMakeLists.txt有一百多行，如果是第一次接触的话确实会有些不适用，但是仔细一行一行看下来，还是能看明白每一行都在干些什么的。本文提供的示例项目里，经过我的理解，直接将示例的CMakeLists.txt里大篇幅的代码少量修改，然后加上我自己的一些逻辑。不过最终还是维持在了一百多行。建议读者如果要进一步深入Shiboken，还是要理解这个CMakeLists.txt。它很烦，但是能帮我们实现很多工作的自动化。

具体细节不少都写在注释里了，还请仔细阅读。
```cmake
cmake_minimum_required(VERSION 3.14) # to support generator Visual Studio 16 2019

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED On)

project(shiboken_simple)

set(cppmath_library "libmath")  # C++项目生成的dll名；dll's name
set(bindings_library "Cppmath") # 和Python要导入的包名一样; same as the binding package name

# get python environment for this project
if (NOT python_interpreter)
    find_program(python_interpreter "python")
endif()
message(STATUS "Using python: ${python_interpreter}")


add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/math)

# =============================== Shiboken Configurations & build =============================
# 使用脚本wrap_src_gen.py来产生xxx_wrapper.cpp，暂时不理解的话，其实也可以模仿samplebindings示例代码手写
# use our wrap_src_gen.py to generate names like xxx_wrapper.cpp，you can manually write
# wrappers name, please refer to the samplebindings in pyside's examples
set(bindings_dir ${CMAKE_CURRENT_SOURCE_DIR}/bindings)

execute_process(COMMAND ${python_interpreter} ${bindings_dir}/wrap_src_gen.py 
                ${CMAKE_CURRENT_BINARY_DIR}         # output dir
                OUTPUT_VARIABLE generated_sources)  # save returned value in generated_sources
list(LENGTH generated_sources list_len) # list_len == 0 means we failed to generate wrapper names
message([*] "length: ${list_len}")
foreach(src ${generated_sources})
    message([*] ${src})
endforeach()

# PySide6目录下的一个辅助脚本，直接复制过来用了，其实就是用于读取一些路径之类的，保存在变量里用于后面使用
macro(pyside_config option output_var)
    if(${ARGC} GREATER 2)
        set(is_list ${ARGV2})
    else()
        set(is_list "")
    endif()

    execute_process(
      COMMAND ${python_interpreter} "${bindings_dir}/pyside_config.py"
              ${option}
      OUTPUT_VARIABLE ${output_var}
      OUTPUT_STRIP_TRAILING_WHITESPACE)

    if ("${${output_var}}" STREQUAL "")
        message(FATAL_ERROR "Error: Calling pyside_config.py ${option} returned no output.")
    endif()
    if(is_list)
        string (REPLACE " " ";" ${output_var} "${${output_var}}")
    endif()
endmacro()

pyside_config(--shiboken-module-path shiboken_module_path)
pyside_config(--shiboken-generator-path shiboken_generator_path)
pyside_config(--shiboken-generator-include-path shiboken_include_dir 1)
pyside_config(--shiboken-module-shared-libraries-cmake shiboken_shared_libraries 0)
pyside_config(--python-include-path python_include_dir)
pyside_config(--python-link-flags-cmake python_linking_data 0)
message(STATUS "shiboken_module_path: ${shiboken_module_path}")
message(STATUS "shiboken_generator_path: ${shiboken_generator_path}")
message(STATUS "python_include_dir: ${python_include_dir}")
message(STATUS "shiboken_include_dir: ${shiboken_include_dir}")
message(STATUS "shiboken_shared_libraries: ${shiboken_shared_libraries}")
message(STATUS "python_linking_data: ${python_linking_data}")


set(wrapped_header ${bindings_dir}/bindings.h)
set(typesystem_file ${bindings_dir}/bindings.xml)

set(shiboken_options --generator-set=shiboken --enable-parent-ctor-heuristic
    --enable-return-value-heuristic --use-isnull-as-nb_nonzero
    --avoid-protected-hack
    -I${bindings_dir}
    -I${CMAKE_CURRENT_SOURCE_DIR}/math   # Include paths used by the C++ parser
    -T${bindings_dir}   # Path used when searching for type system files
    --output-directory=${CMAKE_CURRENT_BINARY_DIR})
set(shiboken_path ${shiboken_generator_path}/shiboken6${CMAKE_EXECUTABLE_SUFFIX})
set(generated_sources_dependencies ${wrapped_header} ${typesystem_file})

add_custom_command(OUTPUT ${generated_sources}
                    COMMAND ${shiboken_path}
                    ${shiboken_options} ${wrapped_header} ${typesystem_file}
                    DEPENDS ${generated_sources_dependencies}
                    IMPLICIT_DEPENDS CXX ${wrapped_header}
                    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
                    COMMENT "Running generator for ${typesystem_file}.")


# =============================== CMake target - bindings_library =============================

# Set the cpp files which will be used for the bindings library.
set(${bindings_library}_sources ${generated_sources})

# Define and build the bindings library.
add_library(${bindings_library} MODULE ${${bindings_library}_sources})

# Apply relevant include and link flags.
target_include_directories(${bindings_library} PRIVATE ${python_include_dir})
target_include_directories(${bindings_library} PRIVATE ${shiboken_include_dir})
target_include_directories(${bindings_library} PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/math)

target_link_libraries(${bindings_library} PRIVATE ${shiboken_shared_libraries})
target_link_libraries(${bindings_library} PRIVATE ${cppmath_library})

# Adjust the name of generated module.
set_property(TARGET ${bindings_library} PROPERTY PREFIX "")
set_property(TARGET ${bindings_library} PROPERTY OUTPUT_NAME
             "${bindings_library}${PYTHON_EXTENSION_SUFFIX}")
if(WIN32)
    set_property(TARGET ${bindings_library} PROPERTY SUFFIX ".pyd")
endif()

# Make sure the linker doesn't complain about not finding Python symbols on macOS.
if(APPLE)
  set_target_properties(${bindings_library} PROPERTIES LINK_FLAGS "-undefined dynamic_lookup")
endif(APPLE)

# Find and link to the python import library only on Windows.
# On Linux and macOS, the undefined symbols will get resolved by the dynamic linker
# (the symbols will be picked up in the Python executable).
if (WIN32)
    message(STATUS "building on WIN32")
    list(GET python_linking_data 0 python_libdir)
    list(GET python_linking_data 1 python_lib)
    find_library(python_link_flags ${python_lib} PATHS ${python_libdir} HINTS ${python_libdir})
    target_link_libraries(${bindings_library} PRIVATE ${python_link_flags})
endif()


# ================================= Dubious deployment section ================================

set(windows_shiboken_shared_libraries)

if(WIN32)
    set(python_versions_list 3 36 37 38 39)
    set(python_additional_link_flags "")
    foreach(ver ${python_versions_list})
        set(python_additional_link_flags
            "${python_additional_link_flags} /NODEFAULTLIB:\"python${ver}_d.lib\"")
        set(python_additional_link_flags
            "${python_additional_link_flags} /NODEFAULTLIB:\"python${ver}.lib\"")
    endforeach()

    set_target_properties(${bindings_library}
                           PROPERTIES LINK_FLAGS "${python_additional_link_flags}")

    # Compile a list of shiboken shared libraries to be installed, so that
    # the user doesn't have to set the PATH manually to point to the PySide6 package.
    foreach(library_path ${shiboken_shared_libraries})
        string(REGEX REPLACE ".lib$" ".dll" library_path ${library_path})
        file(TO_CMAKE_PATH ${library_path} library_path)
        list(APPEND windows_shiboken_shared_libraries "${library_path}")
    endforeach()
endif()

install(TARGETS ${cppmath_library} ${bindings_library}    # Since CMake 3.13, install can use TARGETS in subdirectories
    LIBRARY DESTINATION ${CMAKE_CURRENT_SOURCE_DIR}
    RUNTIME DESTINATION ${CMAKE_CURRENT_SOURCE_DIR}
)
message(STATUS "Install files: ${windows_shiboken_shared_libraries}")
install(FILES ${windows_shiboken_shared_libraries} DESTINATION ${CMAKE_CURRENT_SOURCE_DIR})
```

里面用到的两个python脚本，一个是pyside_config.py，是在PySide6的examples/utils目录下提供的。另一个是一个用于产生shiboken封装后的代码文件名的，如果暂时不能理解，直接参照samplebindigs，手写xxx_wrapper.cpp的名字即可。

不理解的部分首先请参考CMake的官方文档和Shiboken的文档。

## 执行构建
打开x64 Native Tools Command Prompt for VS 2019，跳转到项目目录（shiboken_simple/)下，执行
```bash
cmake -B ./build -G "NMake Makefiles" -DCMAKE_BUILD_TYPE=Release
```
构建类型要是Release，不加上这个选项的话，编译过程会出问题，虽然也能想办法修复，不过节约头发我们先搞定Release编译。
然后到build目录下
```bash
nmake # 执行构建
nmake install # 执行安装过程，就是将必要的文件复制到项目根目录下
```
此时根目录下多出几个文件：
```txt
shiboken_simple
 ├── bindings
 ├── build
 ├── CMakeLists.txt
 ├── Cppmath.pyd
 ├── libmath.dll
 ├── math
 └── shiboken6.abi3.dll
```

## Python中使用测试
我们直接在当前目录下编写python代码进行测试：
![在这里插入图片描述](/wrap_cpp_with_shiboken_for_python.assets/usage.png)
## 写在最后
文章可能包含纰漏、错误和不足之处，欢迎大家提出来一起交流！有疑问欢迎评论！

参考资料：
[1] [https://doc.qt.io/qtforpython/shiboken6/gettingstarted.html](https://doc.qt.io/qtforpython/shiboken6/gettingstarted.html)
[2] [https://doc.qt.io/qtforpython/shiboken6/examples/samplebinding.html](https://doc.qt.io/qtforpython/shiboken6/examples/samplebinding.html)
