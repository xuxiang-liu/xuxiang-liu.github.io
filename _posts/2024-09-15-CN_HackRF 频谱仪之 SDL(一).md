---
layout: post
title: HackRF 频谱仪之 SDL(一)
author: Xuxiang Liu
date: 2024-09-15 23:16 +0800
tags: [SDR,CN]
---

HackRF 频谱仪 - 自定义静态库 和 SDL CMakeFile
{: .message }

今天我们继续来看一下构成 Hackrf 频谱仪的另一个重要库：SDL. 一开始我本来是准备用 GnuPlot 来进行频谱绘制，也考虑用 Qt5 进行图形化的开发。但是考虑到希望能用更底层，更灵活，更轻量化的方式进行，于是选择了用 SDL 的方式。

下面我们就来介绍一下如何用 SDL 库搭建我们的频谱仪界面。为了更聚焦与项目本身，我直接按照实际频谱仪界面开发的需要来介绍 SDL 中的各个模块。对 SDL 更感兴趣的朋友可以直接查阅 SDL 的官网或 SDL 相关的教学网站来对 SDL 有更系统的理解。

由于内容比较多，关于 SDL 的内容我们分多个 blog 进行。

# SDL CMakefile

我们会用到 SDL.h 和 SDL_ttf.h，前者是通用的 SDL 库，后者是处理文字渲染的 True Text Font 库，首先我们需要在 Cmakefile 中添加 SDL 和 SDL_TTF 库的链接：如之前博文所描述的，我们只需要找到 SDL 的 library 即可. 一般安装后 SDL 和 SDL_TTF 的库在 /usr/local/lib 中，因此我们在 Cmake 文件中添加以下内容即可:

```Cmake
{% highlight Cmake %}
# Library File Path
target_link_directories(BasicFunctions PRIVATE /usr/local/lib)
# Link Library
target_link_libraries(BasicFunctions PRIVATE SDL2)
target_link_libraries(BasicFunctions PRIVATE SDL2_ttf)
{% endhighlight %}
```
可以看到，target_link_directories 指定了 library 的路径，而 target_link_libraries 则关联了需要的库文件

# 自定义静态库编译和使用

为了让 code 看起来更有层次，我们创建了一个 SDL_GUI.c 和 SDL_GUI.h 来放置我们自己写的 SDL 相关的函数，在 Main 函数中我们直接调用 SDL_GUI.c 中的函数即可。

在 Cmake 中关联自己的 .c 方式有很多，我们在这里把 SDL_GUI.c 编译为一个静态库，然后通过 SDL_GUI.h 去实现调用，因此我们在 Cmakefile 中添加以下内容：

```Cmake
{% highlight Cmake %}
# Generate a static library
add_library(sdl_gui sdl_gui.c)
# Include File Path
target_include_directories(BasicFunctions PRIVATE /home/liuxuxiang/CLionProjects/BasicFunctions/include)
# Link Library
target_link_libraries(BasicFunctions PRIVATE sdl_gui)
{% endhighlight %}
```
对应的几个步骤详细描述如下：

Step1. 通过 add_library 将 SDL_GUI.c 编译成静态库, 生成的静态库会崔在 cmake-build-debug 文件夹中。

Step2. 通过 target_include_directories 将 SDL_GUI.h 的路径增加到 Cmakefile 中，方便我们调用静态库中的函数

Step3. 通过 target_link_libraries 关联 SDL_GUI.c 对应的静态库。这里不需要像上面 SDL 使用时指定库的路径，因为生成的 sdl_gui 静态库就在默认的路径下

完成上述后，我们就可以在 Main 函数中 include sdl_gui.h，并调用 SDL_GUI.c 中的函数了
