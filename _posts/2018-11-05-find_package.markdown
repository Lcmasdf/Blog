---
layout: post
title: "永怀希望的find_package"
date: 2018-11-04 23:00:00
categories: CMAKE
excerpt: 瞧一瞧cmake是如何搞定复杂的包含关系的~~
published: true
---

# CMAKE之find_package

如果编译软件使用了外部库，且事先并不知道头文件和链接库的位置，需要在编译命令增加查找库的相关指令，cmake使用find_package()来解决这个问题。
```cmake
find_package(OpenCV QUIET)
```
如果能够找到OpenCV的库，那么接下来就可以使用OpenCV_INCLUDE_DIRS和OpenCV_LIBRARIES这两个变量

find_package本身不提供任何搜索库的简便方法，所有的搜索以及变量赋值的过程均需要cmake代码完成。

例如 : opencv的安装包中包含OpenCVXXX.cmake文件， make install后会放在/lib/x86_64-linux-gnu/share/OpenCV下。

### find_package采用两种模式搜索库
+ Module模式
搜索CMAKE_MODULE_PATH指定路径之下的FindXXX.cmake文件，执行该文件从而找到XXX库。查找具体库并给XXX_INCLUDE_DIRS和XXX_LIBRARIES赋值的工作由FindXXX.cmake模块完成。
	+ CMAKE_MODULE_PATH会在cmake install时被创建，默认为/usr/share/cmake-xx/Module。
+ Config模式
Config模式会搜索XXXConfig.cmake文件，执行从而找到XXX库。查找具体库并给XXX_INCLUDE_DIRS和XXX_LIBRARIES赋值的工作由XXXConfig.make完成。

无论使用Module还是使用Config模式，这一步都是要找到xxxx.cmake，后续的find的工作会由xxxx.cmake来完成，接下来通过解析FindIce.cmake来详细了解一下。
### FindIce.cmake 
find_package之后可以使用Ice_INCLUDE_DIRS的原因是在FindIce.cmake中定义了这个变量。所以find_package的功能是由各个.cmake文件遵守共同的规范来实现的

在FindIce.cmake中定义了以下的变量:
+ Ice_VERSION
+ Ice_Found
+ Ice_LIBRARIES
+ Ice_INCLUDE_DIRS
+ Ice_SLICE_DIRS
....

#### Ice_INCLUDE_DIRS定义的过程
```cmake
find_path(Ice_INCLUDE_DIR
		  NAMES "Ice/Ice.h"
		  HINTS ${ice_roots}
		  PATH_SUFFIXES ${ice_include_suffixes}
		  DOC "Ice include directory")
```
cmake将Ice.h所在的路径赋给Ice_INCLUDE_DIR，cmake在ice_roots的路径中寻找Ice。
ice_roots是怎么来的呢？
```cmake
if(Ice_HOME)
	list(APPEND ice_roots "${Ice_HOME}")
else()
	if (NOT "$ENV{ICE_HOME}" STREQUAL "")
		file(TO_CMAKE_PATH "$ENV{ICE_HOME}" NATIVE_PATH)
		list(APPEND ice_roots "${NATIVE_PATH}")
		set(Ice_HOME "${NATIVE_PATH}"
			CACHE PATH "Location of the Ice installation" FORCE)
	endif()
endif()
```
ice_roots就是环境变量中的ICE_HOME

Ice_INCLUDE_DIR 与 Ice_INCLUDE_DIRS又有什么关系呢
```cmake
if(Ice_FOUND)
	set(Ice_INCLUDE_DIRS "${Ice_INCLUDE_DIR}")
	...
endif()
```
Ice_INCLUDE_DIRS就是Ice_INCLUDE_DIR，那么Ice_FOUND又是咋么来的
```cmake
FIND_PACKAGE_HANDLE_STANDARD_ARGS(Ice
								  FOUND_VAR Ice_FOUND
								  REQUIRED_VARS Ice_SLICE2CPP_EXECUTABLE
												Ice_INCLUDE_DIR
												Ice_SLICE_DIR
												Ice_LIBRARY
												_Ice_REQUIRED_LIBS_FOUND
								  VERSION_VAR Ice_VERSION
								  FAIL_MESSAGE "Failed to find all Ice components")
```
find_package_handle_standard_args会检查REQUIRED_VARS后的变量（这些变量都会在次cmake文件中定义），requeire的条件被满足后，Ice_FOUND就会设置为True。required变量均类似Ice_INCLUDE_DIR会在之前被定义。

综上所述，Ice_INCLUDE_DIRS 就是在环境变量ICE_HOME的目录下寻找ice.h，然后将这个目录经过一些列的check赋值给Ice_INCLUDE_DIRS。

需要cmake，ice，使用者三方共同遵守调用规范。cmake要提供FindIce.cmake，ice要符合FindIce.cmake文中所描述的方式构建目录结构，使用者需要正确安装ice（包括但不限于设置环境变量ICE_HOME）。看起来似乎很复杂，但是其实只要按照Ice的README安装，make; make install。让Ice搞定上面的条件。