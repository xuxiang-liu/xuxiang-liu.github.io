---
layout: post
title: 解决 hackrf 在 MATLAB R2019a，R2021b 等高版本上编译问题
date: 2022-05-08 23:18 +0800
tags: [SDR]
---

在高版本的 Matlab 中编译 simulink hackrf 
{: .message }

在上一篇 blog 中有提到在新版本的 MATLAB (> R2016a) 中无法用 mex 进行编译的问题。终于通过这份 issue 的回复 <https://github.com/kit-cel/simulink-hackrf/issues/11> 及新版 mex 的帮助文档找到了解决方案。目前我在 R2019a 和 R2021b 上都进行了验证，是可以正常使用的。

另外我 fork 了 simulink-hackrf 原始的 Github 项目；与原版的项目相比，fork 的新项目 simulink-hackrf_xuxiang 兼容新老版本的 Matlab，无需多余的配置步骤：按照 <https://xuxiang-liu.github.io/2022/04/16/firstpost.html> 所描述的进行，在第12步时直接用 Matlab 进行 make 即可。后续我也希望在这个 fork 的项目中增加一些新的功能。

下面记录了基于原始的 simulink-hackrf 项目在高版本 Matlab 上编译问题的具体解决方案，对于希望使用原始项目的朋友可以参考进行解决。

## 解决步骤

### 安装 MINGW

在新版的 Matlab 中，需要通过 matlab 的附件功能管理包进行安装。这一步就不做过多说明。比在 Matlab R2016a 中的安装会简单许多

### 生成新的 libhackrf.lib

这步可以参考 AustrianCodeWriter 在个回复中的内容 <https://github.com/kit-cel/simulink-hackrf/issues/11> 需要准备的环境是 VS2019 的 MSVC，关于 MSVC 的安装可以参考这片文档 <https://developercommunity.visualstudio.com/t/all-4-vc-command-prompt-batch-files-eg-vcvars64bat/536112> 安装 MSVC 完成后，即可在 VS2019 的安装路径下找到 "...\Microsoft Visual Studio\2019\Community\VC\Auxiliary\Build\vcvars64.bat" 

后续的步骤 AustrianCodeWriter 已经做了完整的描述：

1. Call a new MSVC console (e.g. for "Visual Studio 2019": Start a new command line window and call the batch-script from "C:\Program Files (x86)\Microsoft Visual Studio\2019\Community\VC\Auxiliary\Build\vcvars64.bat")

2. Change directory to "deps\bin" and run:
> dumpbin /EXPORTS libhackrf.dll > libhackrf.exports

3. Create a new file called "libhackrf.def" and insert the following lines:

>  LIBRARY "libhackrf.dll"
   
>  EXPORTS
   
>  hackrf_board_id_name
   
>  hackrf_xxx
   
For _hackrf_xxx_ insert each "**hackrf_**" element from your previously created
   "libhackrf.exports" file

4. In your MSVC console type the command:
lib /def:libhackrf.def /machine:x64 /out:libhackrf.lib

完成后会生成 libhackrf.exp, libhackrf.exports, libhackrf.def 和 libhackrf.lib　四个文件。将 libhackrf.lib 拷贝到 simulink-hackrf-1.0/deps/bin 中

### 修改 make 

打开 make file，将 %% Configuration 下的 ['-L' HACKRF_LIB_DIR]; '-lhackrf' ... 改为 ['-L' HACKRF_LIB_DIR]; '-l**lib**hackrf'... 再进行 make 就可以了

