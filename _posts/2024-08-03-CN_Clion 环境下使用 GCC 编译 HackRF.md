---
layout: post
title: Clion 环境下使用 Cmake 编译 HackRF.
author: Xuxiang Liu
date: 2024-08-03 23:18 +0800
tags: [SDR,CN]
---

Linux Clion Platform 编译 HackRF
{: .message }

终于在 linux 下找到一个超级好用的 IDE 平台 - CLION

具体的安装方案就略过了，之所以觉得好用，也许是和之前用了太多的 PyCharm 有关吧... 不过确实这是一个很强大工具，下面我们看一下如何在 Linux CLion 平台上通过 CmakeList.txt 来实现我们的对 hackrf.h 库的引用吧：

关于工程的建立就略过了，建立好一个 C 语言 executable 工程后，可以看见 Project 栏会自动生成 main.c 和 CmakeList.txt；而在 External Libraries 中也可以看到所有的库文件。那最重要的就是如何编写 CmakeList.txt 来实现对 hackrf 库的调用了

CmakeList file 的编写如下所示（其实这里结合了前面“基于 CMakeList 编译” 和 “基于 GCC 编译” 两篇博文的内容）：

```
cmake_minimum_required(VERSION 3.28)

project(HackRF_SA C)
set(CMAKE_C_STANDARD 11)

add_executable(HackRF_SA hackrf_sa_main.c)

# Add include file directories
target_include_directories(HackRF_SA PRIVATE /usr/local/include/libhackrf)

# Add Library directories
target_link_directories(HackRF_SA PRIVATE /usr/local/lib)

# Link TsetHackRF.o with libhackrf.so. It will automatically add lib and .so
target_link_libraries(HackRF_SA PRIVATE hackrf)
```

其中最重要的，分别为：

1. target_include_directories： 指定 include 文件夹路径

2. target_link_directories：指定 库文件夹路径

3. target_link_libraries: 关联可执行文件与指定的库

完成之后，点击 Load Cmake Changes，就可以看见在 hackrf_sa_main.c 中，#include <hackrf.h> 变为了绿色（如下图所示），同时所有与 hackrf 相关的函数都可以直接跳转啦！如下图所示，这个是之前在 Geany 平台上一直没有实现和困扰我的地方.

![图片](https://github.com/user-attachments/assets/9e7c5ae7-ec03-44d0-ac20-61b40c94f52c)

同时采用 CLion 这个编译器，还可以方便的增加断点进行调试:如我们在 hackrf_init_and_checkID 函数中加入断点，就可以通过 debug 方式看到当前读出来的变量格式，确实是太方便了！

![图片](https://github.com/user-attachments/assets/7c7ba5cf-f630-4fdd-80d4-3ec6271ed65c)


