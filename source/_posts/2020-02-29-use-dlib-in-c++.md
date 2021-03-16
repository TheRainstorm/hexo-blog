---
title: Use dlib in C++
date: 2020-03-01 01:25:00
tags:
description: 同样是关于dlib库的安装。但由于dlib可以不用预编译，官网上也没有提供预编译的版本，本文主要是介绍如何在项目中直接使用dlib。
---

## use dlib in c++

从方法1.1到1.3都不必预编译dlib库，而是在使用dlib的项目中编译。

方法1.4介绍了将dlib预编译成静态库然后使用时会遇到的一些问题。

### with CMake(officially recommend)

看`dlib/example/CMakeLists.txt`

```cmake
cmake_minimum_required(VERSION 2.8.12)

project(examples)

# Tell cmake we will need dlib.  This command will pull in dlib and compile it
# into your project.  Note that you don't need to compile or install dlib.  All
# cmake needs is the dlib source code folder and it will take care of everything.
add_subdirectory(./dlib dlib_build)

macro(add_example name)
   add_executable(${name} ${name}.cpp)
   target_link_libraries(${name} dlib::dlib )
endmacro()

# if an example requires GUI, call this macro to check DLIB_NO_GUI_SUPPORT to include or exclude
macro(add_gui_example name)
   if (DLIB_NO_GUI_SUPPORT)
      message("No GUI support, so we won't build the ${name} example.")
   else()
      add_example(${name})
   endif()
endmacro()

# 
add_gui_example(3d_point_cloud_ex)
```

命令行执行：

```bash
mkdir build
cd build
cmake .. -G "Visual Studio 14 2015 Win64" -T host=x64 #-T host=x64告诉CMake生成64bit的可执行文件。其实安装了最新的VisualStudio后，-G -T都不用指定，默认使用最新的VisualStudio，默认64位。
cmake --build . --config Release
```

使用CMake GUI可以设置一些选项，configure, generate之后可以使用命令行执行最后一步(不用打开VisualStudio)

### with GCC in terminal

```bash
g++ -std=c++11 -O3 -I.. ../dlib/all/source.cpp -lpthread -lX11 example_program_name.cpp
```

windows下还需`gdi32, comctl32, user32, winmm, ws2_32, or imm32 `库

### with VisualStudio

- 把dlib的**父目录**添加到include search path

  >  You should *NOT* add the dlib folder itself to your compiler's include path. 
  >   Doing so will cause the build to fail because of name collisions (such as 
  >   dlib/string.h and string.h from the standard library). Instead you should 
  >   add the folder that contains the dlib folder to your include search path 
  >   and then use include statements of the form #include <dlib/queue.h> or
  >   #include "dlib/queue.h".  This will ensure that everything builds correctly.

- 把 dlib/all/source.cpp 添加到源文件

  > If you are using Visual Studio you add .cpp files to your application using
  >   the solution explorer window.  Specifically, right click on Source Files,
  >   then select Add -> Existing Item and select the .cpp files you want to add.

- 如果需要libjpeg等，把dlib/external文件夹下的源文件添加到project，并 define the DLIB_PNG_SUPPORT and DLIB_JPEG_SUPPORT preprocessor directives. 

### Installing dlib as a precompiled library

dlib的CMake脚本包含INSTALL target，因此可以像其它C++库一样将dlib作为一个静态或动态库安装到系统上。

- **使用预编译的库时，必须保证项目中使用的所有库都是同一个版本的VisualStudio编译出来的。**

  说明：[Mixing Multiple Visual Studio Versions in a Program is Evil]( http://siomsystems.com/mixing-visual-studio-versions/)

- 调用dlib时出现USER_ERROR __inconsistent_build_configuration__ see_dlib_faq_2错误。
  需要将build/dlib/config.h文件拷贝到源码目录dlib-19.17/dlib进行覆盖。config.h文件内有其说明。