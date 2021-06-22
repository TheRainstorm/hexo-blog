---
title: Use dlib in C++
date: 2020-03-01 01:25:00
tags:
description: 同样是关于dlib库的安装。但由于dlib可以不用预编译，官网上也没有提供预编译的版本，本文主要是介绍如何在项目中直接使用dlib。在结尾总结了配置3d_face_reconstruction的过程（国创项目中需要用到，虽然并没有安装成功）
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

## 配置3d_face_reconstruction总结
### 1. OpenCV

1. windows下载预编译版(只含有msvc编译的版本)

2. CMake直接使用find_packages即可。（需添加环境变量，以便找到OpenCV的config文件）

### 2. dlib

1. 官方推荐将代码直接包含到项目中编译（好处是没有ABI一致性问题）

   CMake中add_subdirectory(/path/to/dlib/top/dir)即可。（甚至可以自动从网上下载）

### 3. boost

1. 分为只含有头文件的库和需要独立编译的库。
2. 官方编译方法为：
   1. 先build出Boost.Build（可以看作Boost的一个build工具）
   2. 然后调用b2编译指定模块。（可以添加参数指定编译的库的路径）
   3. 我的编译目录是stage/（库位于stage/lib/下）
3. CMake find_packages暂时是失败的。

### 4. eos

1. 只需要包含头文件即可(header only)，include和3rdparty
2. 顶层目录下的CMakeLists.txt默认勾选了编译example下的示例（需要boost的system, filesystem, program_options和openCV的core, higui, imgproc
3. CMake直接add_subdirectory(/path/to/eos)比较方便。

### 5. Qt

1. 官方介绍了如何使用CMake find_packages，自动tic, moc等。



### TODO

1. 学习Qt，能自己写界面。
2. 明白eos那个示例程序输入输出是什么
3. 最后，自己来改写eos的程序，自己写Qt程序，最后展示出来。