---
layout: post
title: 在 Windows 环境下使用 Matlab + Hackrf
author: Xuxiang Liu
date: 2022-04-16 23:18 +0800
tags: [SDR,CN]
---

在 Windows 下通过 MATLAB 使用 Hackrf.
{: .message }

HackRF 一直是我认为最好的学习或者玩 SDR 的工具，常用的开发或者应用方式一般是在 linux 下进行，或配合 Octave 进行数据处理，更初级的可能就是在 windows 环境下用诸如 SDRsharp 这样的 SDR 软件来收听各种频道。随着工作越来越多的应用 Matlab 进行通信建模，我很希望能在 windows 环境下用 matlab 来进行开发. 终于，在 github 上找到这样一个开源的好项目<https://github.com/kit-cel/simulink-hackrf> 能支持我们在 windows 下通过 simulink 对 Hackrf 进行操作. 另外 Youtube 上也有一个视频进行了安装演示<https://www.youtube.com/watch?v=7dtikuo3BSw>

但是因为时间久远，这个项目看起来已经没有人在维护了，网上关于这块儿的讨论也很少，甚至连安装成功的教程也少之又少，大部分安装问题都没有人进行回复了. 因此，我花了一个周末按照 Youtube 上的演示进行了安装，经过无数踩坑后终于安装成功. 因此在这里做个记录，也为有兴趣的朋友提供一些避坑指南. **Hope u are enjoying**

## 软件及库版本要求
根据我的安装历程，我发现这个安装对新版本的支持度很差，**很多重要的 step 都需要采用和视频中一样的安装版本来进行，甚至连 Matlab 也不例外**。当然对 Matlab 版本的要求主要是因为对 Matlab 中 MinGW64 版本的要求，而不同的 Matlab 版本所支持的 MinGW64 版本也是不同的。**不过当安装编译完成后，我们可以用新版的 Matlab 进行操作,甚至从我看到的结果 Matlab R2019a 能更快速的处理数据，在 10Msps 下 Hackrf 不会报出 OOOOOO 的错误**

在文末我有放上所有需要的软件的安装包。可以一次性全部下载。

## 详细步骤
### 1. 安装 HackRF 驱动

1.1 这一步就不做详细介绍了，相信在 Windows 下用过 SDRsharp 的朋友都有安装，主要就是安装一个 zadig 进行驱动安装。

### 2. 安装 Matlab R2016a

2.1 这一步就不做详细介绍了。目前我尝试过用 Matlab R2019a 进行过安装，但是没有成功，问题主要是出在 Matlab 所需要的 Mingw complier 的版本上，看起来只有 Matlab R2016a 支持的 tdm64-gcc-4.9.2 版本才可以正确的在 Matlab 中进行 make；我在 github 项目的 Issue 里也看到有朋友用不同的 Matlab 版本进行过安装遇到问题，**因此强烈建议还是用 R2016a 进行安装，后续用其他高版本进行使用都是 OK 的**。有能力的朋友可帮忙 debug 一下为什么用高版本的 mingw 在 matlab 中无法正确进行 make

### 3. 在 matlab 中安装 MinGW64 附加功能 (tdm64-gcc-4.9.2)

3.1 可惜由于时间久远，Matlab 自带的附加功能管理包已经不支持直接安装 MinGW64 了，需要手动进行安装，参考这篇 blog <https://blog.csdn.net/weixin_44217573/article/details/105951236>. 首先我们下载 tdm64-gcc-4.9.2　的安装文件 <https://sourceforge.net/projects/tdm-gcc/files/TDM-GCC%20Installer/Previous/1.1309.0/>  直接运行安装即可，注意**不要**勾选下方的 Check for updated files... 选项。然后安装完成后在系统环境变量中添加 【变量名：WM_MINGW64_LOC 变量值：D:\TDM-GCC-64 (换成你的路径）】。安装完成后在 Matlab 中运行 mex -setup 看到以下界面即安装完成：

![TDM-GCC-64](https://user-images.githubusercontent.com/40487487/163658277-4e00672f-fa99-4998-b9ce-cbe1bdb0c006.PNG)

### 4. 安装 mingw64-posix-6.4.0

4.1 这一步也无需详细介绍，网上有很多安装教程，唯一需要注意的是，一定要安装 posix 版本 （版本号我没有试过，也许新的版本也行，如果有试过的朋友可以留言确认一下）！如果在线安装不方便的话可以选择离线安装，可以参考这篇 blog <https://blog.csdn.net/ZHAOJUNWEI08/article/details/86602120> 。安装完成后添加系统环境变量，【变量名：PATH 变量值：C:\Program Files\mingw64\bin (换成你的路径）】 安装完成后可以在 cmd 中执行 gcc -v，看到以下界面即安装完成

![gcc](https://user-images.githubusercontent.com/40487487/163658046-4d176592-6d96-4d87-ad72-70ecb60dc815.PNG)

### 5. 安装 cmake 3.6.2

5.1 这一步主要是注意正确的版本号,直接下载 Run 就可以了。虽然我没有验证过安装其他版本会不会出问题，但是这篇 blog 中 <https://blog.csdn.net/angu5630/article/details/112726326> 提到一定要用 3.6 版本的 cmake。Anyway，就按照他说的来吧，可惜这篇 blog 看起来也半途而废了。安装包可以在这里找 <https://download.csdn.net/download/icbm/9646285>，我在文后也会附上所有需要的安装包

### 6. 安装 cygwin64

6.1 这个安装也是在网上有很多教程，我遇到的唯一问题就是始终在一台电脑上无法找到镜像源，后来我只能换了一台电脑下载离线安装包，然后进行离线安装。如果有碰到类似问题的朋友，我在文后有留下离线安装包的路径，可以参考。这里不需要下载 cygwin64 额外的插件，就最基础的 cygwin64 即可。

### 7. 下载 libusb

7.1 直接在 github 上下载即可 <https://github.com/libusb/libusb/releases> ，这个对版本没有要求，下载后解压先放在某个路径下即可。

7.2 将 D:\libusb\include\libusb-1.0 (换成你的路径）中的 libusb.h copy 到 complier 的 include 中，即之前安装的 C:\Program Files\mingw64\x86_64-w64-mingw32\include 中。

### 8. 下载 libhackrf

8.1 直接在 github 上下载即可 <https://github.com/greatscottgadgets/hackrf> ，这个只需要 hackrf-master -> host 中的 libhackrf。同样下载后解压先放在某个路径下即可。

8.2 将整个 libusb 文件夹 copy 到解压后的 libhackrf 文件夹中

### 9. 下载 simulink-hackrf-1.0

9.1 直接在 github 上下载即可 <https://github.com/kit-cel/simulink-hackrf/releases/tag/v1.0> ，解压后再在 D:\simulink-hackrf-1.0\ 路径中(换成你的路径）创建一个名为 deps 的新文件夹

### 10. Run cmake

10.1 在 cmake 的安装路径中 D:\cmake-3.6.2-win64-x64\cmake-3.6.2-win64-x64\bin (换成你的路径）打开 cmake_gui.exe

10.2 Where is the source code 选择 D:/libhacrf/src (换成你的路径), Where to build the binaries 选择 D:/libhackrf/build

10.3 点击 Add Entry，**手动一条一条的加入如下图所示内容**。**注意** 如果某一个 Name 对应的 Value 是路径，则 Add Entry 中 Type 选择 Path; 如果 Value 对应具体的文件，则选择 FILEPATH；Value 是选项框的选择 BOOL; Value 是数值的选择 STRING; Value 为空的 Name 以 Path 结尾的选择 PATH，其余的选择 STRING.

![cmake](https://user-images.githubusercontent.com/40487487/163658079-9885a4d8-a39c-4ef0-9862-7301b007bc42.png)

10.4 点击 Configure，在弹出的选项框中 Complier 选择 MINGW，执行完成后会提示 Configuring done。**注意** 这里可能会提示一个 Warninng，不用管它

10.5 再点击 Generate，执行完成后提示 Generating done，如 10.3 图所示

### 11. cygwin64 make libhackrf

11.1 打开 cygwin64 terminal，进入到 libhackrf/build 文件夹中，在 terminal 执行 mingw32-make

11.2 再执行 mingw32-make install。这两步执行后的结果如图所示，并且在 simulink-hackrf-1.0 的 deps 文件夹中会出现 bin　和　include　两个子文件夹

![make](https://user-images.githubusercontent.com/40487487/163658115-cd38b392-f7b1-48d1-bb92-a06ee30bc37e.png)

### 12. 在 Matlab 下进行 make

12.1 将 D:\simulink-hackrf-1.0\deps\include\Project (换成你的路径) 中的 hackrf.h copy 到 D:\simulink-hackrf-1.0\src (换成你的路径) 

12.2 打开 matlab，路径选择到 simulink-hackrf-1.0 中，直接执行 make

12.3 这个时候可能会报如下图所示的错误，这个时候可以首先在 D:\simulink-hackrf-1.0\deps\bin (换成你的路径) 中新建一个空的 libhackrf.lib （直接右键->新建 txt 文档，再将 .txt 改为 .lib），然后将同样路径下的 libhackrf.dll copy 到 D:\simulink-hackrf-1.0\build (换成你的路径)，最后在 matlab 中将 build 也添加到路径里 （addpath('build')），再执行 make

![error](https://user-images.githubusercontent.com/40487487/163658158-a9d29fbf-28f2-47ff-9870-c2719b22644d.PNG)

12.4 好了，到这里应该就可以看到如下界面，编译成功。

![success](https://user-images.githubusercontent.com/40487487/163658215-c92a94d3-509c-48e1-9269-048fe1f41052.PNG)

### 13. Run Demo

13.1 最后可以通过自带的 demo 看一下效果。**注意** 要将 demos,build 都添加到 matlab 路径中去。Hackrf 插上后先执行 hackrf_find_devices 找到设备，然后打开 demos 中的 scope_demo，修改 Hackrf Sink 模块的采样率、增益、中心频点等参数，即可看到漂亮的频谱了。下图是用 Airpsy 和 Simulink 捕获到的中心频点在 102.6MHz 的对比，可以看到频谱基本上是一样的。

![demo](https://user-images.githubusercontent.com/40487487/163658230-0ad57941-9047-4329-81e8-d6b052d0bbff.png)

13.2 BTW，1-12 完成后，就可以用更高版本的 Matlab 去打开使用了，进入到 simulink-hackrf-1.0 路径下，添加 demos 和 build 到路径中即可。实际上上图就是用 Matlab R2019a 所截取出来的。

## 安装包链接

这里有所有需要的安装包，当然 matlab 的就算了...太大了... 另外再抵制一下限速收费的百度网盘！支持开源！

transfer link: <https://cowtransfer.com/s/2c0178772e934a> or access [ cowtransfer.com ] and input extract code: 066ef3 to download;

---- The End ---
