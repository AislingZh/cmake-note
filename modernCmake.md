
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [cmake](#cmake)
  - [cmake 入门](#cmake-入门)
  - [文件编译](#文件编译)
    - [单一源文件](#单一源文件)
      - [编译过程](#编译过程)
        - [vs + cmake + msbuild](#vs-cmake-msbuild)
        - [vs + cmake + nmake](#vs-cmake-nmake)
        - [vs + cmake + PreLoad.cmake + nmake](#vs-cmake-preloadcmake-nmake)
    - [一个目录 多个源文件](#一个目录-多个源文件)
    - [多个目录 多个源文件](#多个目录-多个源文件)
- [modern cmake](#modern-cmake)
  - [概 述](#概-述)
  - [Traditioncal CMake 与 modern cmake 区别](#traditioncal-cmake-与-modern-cmake-区别)
  - [实例](#实例)
- [maya cmake](#maya-cmake)

<!-- /code_chunk_output -->

# cmake

## cmake 入门

- [x]TODO: 添加cmake语法细节


## 文件编译
### 单一源文件
```
- Demo
    - main.cpp
    - CMakeList.txt
```

1. 编写 CMakeLists.txt

首先编写 CMakeLists.txt 文件

```bash
# CMake 最低版本号要求
cmake_minimum_required(VERSION 2.8)

# 项目信息
project(Demo)

# 指定生成目标
add_executable(Demo main.cc)
```

对于上文的三个命令，含义分别是

`cmake_minimum_required` 指定运行此配置文件所需的 CMake 的最低版本
`project` 参数值是 Demo，该命令表示项目的名称是 Demo
`add_executable` 将名为 main.cpp 的源文件编译成一个名称为 Demo 的可执行文件

#### 编译过程
##### vs + cmake + msbuild
```bash
>> cmake .

>> msbuild demo.sln
```
##### vs + cmake + nmake
```bash
>> cmake -G "NMake Makefiles" .

>> nmake
```
##### vs + cmake + PreLoad.cmake + nmake
上个命令需要用`"-G"`选项指定生成器。每次都输入感觉比较麻烦，可以将其配置在`"PreLoad.cmake"`文件中
```
- Demo
    - main.cpp
    - CMakeList.txt
    - PreLoad.cmake
```

在当前目录下新建文件`"PreLoad.cmake"`，输入：
```bash
set(CMAKE_GENERATOR "NMake Makefiles" CACHE INTERNAL "" FORCE)
```

### 一个目录 多个源文件
```
- Demo
    - CMakeLists.txt
    - mian.cpp
    - text.h
    - text.cpp
```
这个时候，可以单单更改CMakeLists.txt里的 **add_executable** 行
```bash
add_executable(Demo main.cc text.h text.cpp)
```
这么改动没什么问题，但是要增加的源文件过多，就显的很麻烦，更轻松的是使用 `aux_source_directory` 命令
> 该命令会查找指定目录下的所有源文件，然后将结果存进指定变量名

其语法如下：
```bash
aux_source_directory(<dir> <variable>)
```
因此，可以修改 CMakeLists.txt 如下：


```bash
cmake_minimum_required(VERSION 2.8)

project(Demo)

# 查找当前目录下的所有源文件
# 并将名称保存到 DIR_SRCS 变量
aux_source_directory(. DIR_SRCS)

add_executable(Demo ${DIR_SRCS})
```
CMake 会将当前目录所有源文件的文件名赋值给变量 `DIR_SRCS`，再指示变量 `DIR_SRCS` 中的源文件需要编译成一个名称为 `Demo` 的可执行文件。

### 多个目录 多个源文件
现在目录如下
```
- Demo
    - mian.cpp
    - text
        - text.h
        - text.cpp
```
对于这种情况，需要分别在项目 `Demo` 目录 和 `text` 目录里各编写一个 `CMakeLists.txt` 文件。为了方便，我们可以先将 `text` 目录里的文件编译成静态库再由 `main` 函数调用。
```bash
cmake_minimum_required (VERSION 2.8)

project (Demo)

# 查找当前目录下的所有源文件
# 并将名称保存到 DIR_SRCS 变量
aux_source_directory(. DIR_SRCS)

# 添加 text 子目录
add_subdirectory(text)

add_executable(Demo main.cc)

# 添加链接库
target_link_libraries(Demo Text)
```

`add_subdirectory` 指明本项目包含一个子目录 `text`，这样 `text` 目录下的` CMakeLists.txt` 文件和源代码也会被处理
`target_link_libraries` 指明可执行文件 `main` 需要连接一个名为 `text` 的链接库 

`text`目录中的` CMakeLists.txt`

```bash
# 查找当前目录下的所有源文件
# 并将名称保存到 DIR_LIB_SRCS 变量
aux_source_directory(. DIR_LIB_SRCS)

# 生成链接库
add_library (Text ${DIR_LIB_SRCS})
```
`add_library` 将 text 目录中的源文件编译为静态链接库。

# modern cmake

- [x]TODO: 添加modern cmake

## 概 述
现代化的CMake是围绕 `Target` 和 `Property` 来定义的，并且竭力避免出现变量variable的定义
Variable横行是典型CMake2.8时期的风格。现代版的CMake更像是在遵循OOP的规则，通过target来约束link、compile等相关属性的作用域。如果把一个Target想象成一个对象（Object），会发现两者的组织方式非常相似：
1. 构造函数：
`add_executable`
`add_library`
2. 成员函数：
`get_target_property()`
`set_target_properties()`
`get_property(TARGET)`
`set_property(TARGET)`
`target_compile_definitions()`
`target_compile_features()`
`target_compile_options()`
`target_include_directories()`
`target_link_libraries()`
`target_sources()`
3. 成员变量
`Target properties（太多）`

在Target中有两个概念非常重要：`Build-Requirement`s 和 `Usage-Requirements`
- `Build-Requirements`： 包含了所有构建Target必须的材料
- `Usage-Requirements`：包含了所有使用Target必须的材料，这些往往是当另一个Target需要使用当前target时，必须包含的依赖
> 源代码，include路径，预编译命令，链接依赖，编译/链接选项，编译/链接特性等

## Traditioncal CMake 与 modern cmake 区别
- Traditioncal CMake在设置`build-requirements`和`usage-requirements`上都依赖手动输入命令，并且人工维持其作用域（变量的作用域以目录为单位）
- 而Modern CMake在设置上述`requirement`均以target为单位，所以在传递target属性到其依赖的下游链条中更自动也更智能。

PRIVATE/INTERFACE/PUBLIC：定义了Target属性的传递范围

`PRIVATE`   表示Target的属性只定义在当前Target中，任何依赖当前Target的Target不共享PRIVATE关键字下定义的属性
`INTERFACE` 表示Target的属性不适用于其自身，而只适用于依赖其的Target
`PUBLIC`    表示Target的属性既是build-requirements也是usage-requirements。凡是依赖于当前Target的Target都会共享本属性。

## 实例

```
- HelloWorld
    - CMakeLists.txt
    - hello-exe
        - CMakeLists.txt
        -  main.cpp
    - hello-lib
        - CMakeLists.txt
        - hello.hpp
        - hello.cpp
```
以这样一个简单的HelloWorld开启有助于我们快速进入主题

这个项目结构很简单，包含两个子文件夹，**hello-exe**生成`executable(可执行文件)`，**hello-lib**生成链接库（动态）。

我们先看下顶层CMakeLists的内容：
```bash
# HelloWorld/CMakeLists.txt
cmake_minimum_required(VERSION 3.14)

project(HelloWorld VERSION 1.0.0)

add_subdirectory(hello-lib)
add_subdirectory(hello-exe)
```
这里没有什么值得多讨论的，与传统CMake一样的写法，定义project名称，版本号，添加子文件夹

我们接着看hello-lib。首先看源码

```cpp
// hello-lib/hello.hpp
#pragma once

#ifdef DLL_EXPORT
#def DLL_API _declspec(dllexport)
#else
#define DLL_API _declspec(dllimport)
#endif

class DLL_API hello_pointer
{
public:
    void print();
}
```
------------------
```cpp
// hello-lib/hello.cpp
#include <iostream>
#include "hello.hpp"

void hello_pointer::print()
{
    std::cout << "hello world" << std::endl;
}
```

源码比较简单，只是定义一个hello_printer类，并在其cpp中定义成员函数print

请注意头文件中的预编译命令。这在VS中是非常常用的预编译命令，用于导出动态库的符号。而当该库被其他Target调用时，需要使用dllimport导入符号(这条预编译命令刚好符合*build-requirement*和*usage-requirement*的定义)

> 如果library A依赖第三方库B，而executable E依赖于A，那么问题来了，E是否依赖于B?在传统CMake中，这个依赖关系是需要用户来手动维护，比如在E的target_link_libraries中添加相应的依赖项，在include_directories中添加依赖项的头文件路径。而在Modern CMake中，是靠A在target_link_libraries的时候将B放在public 还是private来区分，在e中只需要依赖A，是否需要依赖B，是由CMake自动推导完成。

对于hello-lib而言，定义DLL_EXPORT从而将`DLL_API`定义为`_declspec(dllexport)`是`build-requirement`，而对于该Target的调用者，需要的是不定义DLL_EXPORT。因而需要在定义`compile_definitions` 时将`Dll_EXPORT`放在PRIVATE关键词下

当其他Target使用hello-lib的时候，还需要知道hello.hpp的路径。传统的CMake写法是通过在调用者的CMakeLists.txt中添加includedirectory来实现。但这种写法会依赖库之间的相对路径，一旦调整路径，所有的CMakeLists都将需要更新。在Modern CMake中不必如此，你只需要通过target_include_directories指定hello.hpp的路径，将之纳入INTERFACE(当然PUBLIC)也行。则调用者就可以得到该include路径。

CMakeLists.txt 全文如下：
```bash
set(target_name "hello-lib")

add_library(${target_name}  SHARED
        hello.cpp
        hello.hpp
)

target_include_directories(${target_name} INTERFACE ${CMAKE_CURRENT_SOURCE_DIR})
target_compile_definitions(${target_name} PRIVATE DLL_EXPORT)
```
最后看下hello-exe。hello-exe中的CMakeLists.txt就可以比较简单了：
```bash
add_executable(hello-exe main.cpp)
target_link_libraries(hello-exe PUBLIC hello-lib)
```

































# maya cmake
- https://github.com/chadmv/cgcmake



























































































