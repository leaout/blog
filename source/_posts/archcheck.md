---
title: 如何判断CPU 架构
date: 2024-01-07
tags: C++
---
# C++代码中判断CPU 架构，操作系统类型，cmake 中判断CPU 架构，操作系统类型

## c++代码中判断

```
#include<iostream>

int main(){

#if defined __linux__

std::cout<<"linux system"<<std::endl;

#elif defined __ANDROID__

std::cout<<"android system"<<std::endl;

#endif

#if defined __aarch64__

std::cout<<"this is arm cpu"<<std::endl;

#elif defined __x86_64__

std::cout<<"this id x86 cpu"<<std::endl;

#endif

return 0;

}
```



## CMAKE 中判断操作系统类型和 cpu 架构

需要利用 cmake中的变量来判断。

CMAKE_HOST_SYSTEM_NAME：操作系统类型

CMAKE_HOST_SYSTEM_PROCESSOR：cpu 指令集

CMakeLists.txt：

```
cmake_minimum_required(VERSION 3.10.0)

message(${CMAKE_HOST_SYSTEM_NAME})

message(${CMAKE_HOST_SYSTEM_PROCESSOR})

if(CMAKE_HOST_SYSTEM_NAME MATCHES "Linux")

message("this is Linux")

elseif(CMAKE_HOST_SYSTEM_NAME MATCHES "Android")

 message("this is Android")

endif()

if(CMAKE_HOST_SYSTEM_PROCESSOR MATCHES "aarch64")

message("this is aarch64 cpu")

elseif(CMAKE_HOST_SYSTEM_PROCESSOR MATCHES "x86_64")

 message("this is x86_64 cpu")

endif()
```



## 常用宏定义

操作系统预定义宏：

操作系统 公共定义 64位系统定义

Windows _WIN32 _WIN64

macOS __APPLE__ __LP64__

Linux __linux__ __LP64__

Android __ANDROID__ __LP64__

编译器预定义宏：

编译器 编译器定义 x86指令集 AMD64指令集 ARM32指令集 Thumb指令集 ARM64指令集

MSVC _MSC_VER _M_IX86 _M_X64 _M_ARM _M_THUMB _M_ARM64

GCC __GNUC__ __i386__ __x86_64__ __arm__ __thumb__ __aarch64__

Clang __clang__ __i386__ __x86_64__ __arm__ __thumb__ __aarch64__