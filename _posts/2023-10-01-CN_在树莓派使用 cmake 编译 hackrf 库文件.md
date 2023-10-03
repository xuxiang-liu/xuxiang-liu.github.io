---
layout: post
title: 在树莓派使用 cmake 编译 hackrf 库文件
author: Xuxiang Liu
date: 2023-10-01 23:18 +0800
tags: [SDR,CN]
--- 

在树莓派下通过 C 直接使用 Hackrf 系列 - cmake 编译.
{: .message }

搭建好树莓派上的 hackrf 环境后，我们开始来用 C 直接操作 hackrf 了。 当然我们第一步是将 hackrf 的库和我们的 project 关联起来。对于初学者，我觉得树莓派自带的 Geany  就是一个很方便的轻量级编译器工具，但是在使用 Geany 之前，我们还是用 cmake 命令行来进行编译和库的添加。下面是详细的参考步骤：

1. 首先我们需要创建一个 cmake list 文件来将 hackrf library 包括到这个工程中。当然这里可以参考很多 github 上的 CMakeLists.txt。 参考 BTLE 这个开源工程 （https://github.com/JiaoXianjun/BTLE），我们裁剪出一个最简单的将 hackrf library 加入到工程中的 CMakeLists.txt 文件，如下所示：

~~~C
cmake_minimum_required(VERSION 2.8)

project(HACKRFCAPTURE_XUXIANG C)
set(CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake/modules)

find_package(LIBHACKRF REQUIRED)
add_executable(HackRFCapture_Xuxiang main.c)

include_directories(${LIBHACKRF_INCLUDE_DIR})
LIST(APPEND TOOLS_LINK_LIBS ${LIBHACKRF_LIBRARIES})
target_link_libraries(HackRFCapture_Xuxiang ${TOOLS_LINK_LIBS} m)
~~~

我们逐一对每行代码进行解释：

> cmake_minimum_required(VERSION 2.8)

这句指定了本地安装的 cmake 的最低版本，应该目前安装的 cmake 都可以满足

> project(HACKRFCAPTURE_XUXIANG C)

接下来这句指定了 project 的名字和语言，这个 project 名字可以任意设置。

> set(CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake/modules)

接下来这句很重要，他指定了 CMAKE_MODULE_PATH, 而这个 path 用于后面 find_package 命令中寻找名为 LIBHACKRF 的 package 文件 （注意这里的 CMAKE_MODULE_PATH 都是 cmake 中定义好的）。 可以看到如果屏蔽掉在这句，则 cmake 会报错说无法找到一个名为 LIBHACKRF 的 package, 如下图所示

![image](https://github.com/xuxiang-liu/xuxiang-liu.github.io/assets/40487487/68b593bc-4353-411e-b006-027d38169ba8)

那么在这句话指定的路径中，我们会放入一个名为 FindLIBHACKRF.cmake 的 package configuration 文件，如上图提醒的，这个文件的命名也都是 cmake 中定义好的，要么是 FindLIBHACKRF.cmake 要么是 libhackrf-config.cmake

这里我们就不详细展开讲 FindLIBHACKRF.cmake 文件了，这个文件也取自 github 上开源的 BTLE 项目 （https://github.com/JiaoXianjun/BTLE/tree/master/host/cmake/modules）

> find_package(LIBHACKRF REQUIRED)

接下来这句就是通过 find_package 命令去系统路径中寻找 LIBHACKRF package

> add_executable(HackRFCapture_Xuxiang main.c)

接下来这句是添加可执行文件，括号中第一个参数是可执行文件的文件名，第二个参数是编译的 .c 文件, 这里我们把这个示例工程的源文件命名为 main.c

> include_directories(${LIBHACKRF_INCLUDE_DIR})

接下来这句把 LIBHACKRF INCLUDE DIR 加入到路径中，这个 DIR 是在 find_package 中找到的。

> LIST(APPEND TOOLS_LINK_LIBS ${LIBHACKRF_LIBRARIES})

接下来这句把  LIBHACKRF_LIBRARIS 加入 TOOLS_LINK_LIBS 路径中, 这里用 LIST 的原因是如果还需要加入更多的库，就可以把这些库都 append 到 TOOLS_LINK_LIBS 路径中，以后的项目都会用上。

> target_link_libraries(HackRFCapture_Xuxiang ${TOOLS_LINK_LIBS} m)

最后通过target_link_libraries 把 TOOLS_LINK_LIBS 和这个可执行文件关联。

这样就可以把 hackrf 的库文件和我们的 project 关联起来了。

2. 接下来，我们看一下如何进行 cmake 编译，整个工程的文件夹架构如下：

CMakeList file 放在根目录下面，同时创建 cmake 和 src 两个文件夹：其中 cmake 下创建 modules 子文件夹用于存放 FindLIBHACKRF.cmake，而 src 则用于存放 .c 源文件。当然这个文件夹架构并非必需的，只是按照这种方式会更清晰直观。另一个 build 文件夹是在后面编译过程中创建的：

![image](https://github.com/xuxiang-liu/xuxiang-liu.github.io/assets/40487487/11f788fd-9acc-4efc-a483-01844d1efc4a)

为了验证整个编译环境及是否把 hackrf 库正确做了关联，我们创建如下所示的一个简单的 main.c：

~~~C
#include <stdio.h>
#include <hackrf.h>
int main(void){
  int result;
 	result = hackrf_init();
 	if(result != HACKRF_SUCCESS){
 		printf("HackRF Init Fail \n");
 	}
 	else{
 		printf("HackRF Init Success \n");
 	}
 }
~~~

这个 main.c 会 include hackrf 的库，并且通过 hackrf_init 函数检查 hackrf 库环境是否 ready。 接下来，我们在 terminal 中通过 cd 指令进入到根目录下，并通过 mkdir 创建一个名为  build 的子文件夹，这个文件夹用于存放编译好的可执行文件。

进入到新创建的 build 文件夹中，通过 cmake ../ 指令，会在 build 的上一级目录中找到 CMAKEList 文件并执行，如果本地 hackrf 已安装，则应该返回下面的 cmake 成功的指令

![image](https://github.com/xuxiang-liu/xuxiang-liu.github.io/assets/40487487/7f06ea69-8c5d-451e-82fa-4bc8202e1fbe)

接下来我们还是在 build 文件夹中，执行 make，即可完成编译，如下图所示：

![image](https://github.com/xuxiang-liu/xuxiang-liu.github.io/assets/40487487/640ae2a2-b7fd-4f54-b513-f1a77f921f87)

这个时候在 build 文件夹中会出现一个绿色的名为 HackRFCapture_Xuxiang 的可执行文件，在 terminal 中执行可以得到以下结果：

![image](https://github.com/xuxiang-liu/xuxiang-liu.github.io/assets/40487487/f83429f2-b042-48f4-9f61-a27606fb10c1)

这样说明通过 cmake 编译，关联 hackrf 库就完成了。



