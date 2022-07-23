---
layout: post
title: Simulink-hackrf 源码初识
author: Xuxiang Liu
date: 2022-05-28 23:18 +0800
tags: [SDR]
---

探究 simulink hackrf 的源码，以及 Matlab S-func.
{: .message }

尝试跑了 simulink-hackrf 之后，我对它具体的实现方式十分感兴趣，于是简单的研究了一下：发现他主要是通过在 simulink 中的 S 函数来调用 hackrf.c 中的函数来实现的，于是也借此机会对 simulink 中的 S 函数做了一个了解。当然更深入的理解 simulink-hackrf 其实主要是理解 hackrf.c. 这部分内容会留到后面再去深入研究，这篇 blog 就作为一个通过 S 函数来学习 simulink-hackrf 的入口吧

这篇 blog 的主要内容有：

1. simulink-hackrf 源码及实现原理简要介绍

2. simulink S函数简要介绍

3. 更改 simulink-hackrf source mask 实现界面修改

4. 更改 simulink-hackrf source.c 实现更多功能

## simulink-hackrf 源码及实现原理简要介绍

simulink-hackrf 核心是通过 S 函数来调用 hackrf.c 中各种函数，从而实现对 hackrf 的操作。它一共有三个重要的源码文件，分别是 hackrf_find_devices.c, hackrf_source.c 和 hackrf_sink.c 这些都放在 simulink-hackrf/src/ 目录下。他们都是通过 c 语言实现的 S 函数，并以 MEX （Matlab Executable）作为可执行文件。

最终，这些源文件通过 simulink-hackrf/make.m 文件进行编译，生成对应的 .mexw64 文件，然后就可以在 Simulink S 函数中进行调用了。其中，hackrf_find_devices.c 是用于连接 hackrf devices，hackrf_source.c 用于实现 hackrf 接收功能，hackrf_sink.c 用于实现 hackrf 发射功能。

## simulink S 函数简要介绍

Simulink S 函数是一个很有用的工具，允许用户以 C，C# 等语言创建自定义的函数。关于 S 函数在很多 blog 中都有介绍，这里我就基于 hackrf_source.c 这个文件来简要介绍一下 S 函数的构造。如果有兴趣，也可以参考 <https://blog.smileland.me/2020/02/12/%E4%BD%BF%E7%94%A8C%E8%AF%AD%E8%A8%80%E5%86%99%E7%AE%80%E5%8D%95S-Function/> 这篇 blog 来自己写一个简单的 S 函数，这样对 S 函数就有更深入的理解了

以 hackrf_souce.c 这个函数为例，一个 S 函数的构造如下图所示，另外也可以参考 <https://ww2.mathworks.cn/help/simulink/sfg/how-the-simulink-engine-interacts-with-c-s-functions.html> 这篇文档来了解 Simulink Engine 的工作流程

![S-func-第 1 页](https://user-images.githubusercontent.com/40487487/170820271-68d8311e-4ec4-4244-8218-906d5eacb291.jpg)

### mdlInitializeSizes

mdlInitializeSizes 函数在程序开始时被调用，进行输入输出端口设置、仿真采样0时间设置、端口数据类型设置等等，更具体的可以参考 <https://ww2.mathworks.cn/help/simulink/sfg/mdlinitializesizes.html> 

### mdlStarts

mdlStart 在 mdlInitializeSizes 后执行，它会调用 startHackRF 函数，开启 Hackrf 的 receiver，更具体的可以参考 <https://ww2.mathworks.cn/help/simulink/sfg/mdlstart.html>

### mdlOutputs

mdlOutput 在每一个 step 都会被调用，它会计算 S 函数在当前 time step 下的输出，具体的输出数据格式在 mdlInitializeSizes 中进行了定义。更具体的可以参考 <https://ww2.mathworks.cn/help/simulink/sfg/mdloutputs.html>

### mdlSimStatusChange

mdlSimStatusChange 在每次按下暂停或恢复时被调用，它主要执行的就是当暂停时调用 stopHackrfRX 函数，而在恢复时调用 startHackRF 函数。更具体的可以参考 <https://ww2.mathworks.cn/help/simulink/sfg/mdlsimstatuschange.html>

### mdlTerminate

mdlTerminate 在停止仿真时被调用，它主要执行的就是调用 stopHackRF 函数。更具体的可以参考 <https://ww2.mathworks.cn/help/simulink/sfg/mdlterminate.html>

### mdlCheckParameters & mdlProcessParameters

mdlCheckParameters 在每次参数发生改变时被调用，参数发生改变可以发生在仿真开始时，或每个 time step 中。mdlCheckParameters 会去检查每个 Parameter 是否满足设定的数据类型要求等。mdlProcessParameters 在每次 mdlCheckParameters 被调用并验证通过后被调用，它主要是将更新后的参数进行处理，在这里主要就是调用 Hackrf_set_param 将新的参数写入到 hackrf 中。更具体的可以参考 <https://ww2.mathworks.cn/help/simulink/sfg/mdlcheckparameters.html> 和 <https://ww2.mathworks.cn/help/simulink/sfg/mdlprocessparameters.html>

以上就是结合 hackrf_source.c 对 S 函数进行简要介绍的全部内容，其实只要对照构造图，这些代码并不难理解。只是在 mdlOutputs 中有一些关于 pthread_mutex 的内容比较复杂，但是如果我们从整体出发，也可以不用太关注线程互锁这部分的内容。

## 更改 simulink-hackrf hackrf_source 的界面

S 函数通过 C 实现后，可以增加 Mask 来构造一个简单的 UI 界面，实现参数输入或改变的过程。在 simulink-hackrf 中，已经建好了 hackrf_source.c 和 hackrf_sink.c 的 mask 文件 (blockset/hackrf_library.slx)，并在 simulink-hackrf/make.m 中直接将建立好的 hackrf_library.slx 复制到了 build 文件中。因此我们可以打开 blockset/hackrf_library.slx 来看一下 hackrf_source.c 的 mask 与 hackrf_source.c 是如何对应起来的：

双击打开 hackrf_library.slx 中的 HackRF_Source 模块，如下图所示：

![image](https://user-images.githubusercontent.com/40487487/170820285-a0a7b9a8-e65e-4030-bc0b-4b127999ba44.png)

可以看到，界面上可以设置的包括 center frequency, Sample Rate, Bandwidth, Output frame Length, Output data type, Gain, LNA and VGA. 这些其实和 hackrf_source.c mdlCheckParameters 中指定的参数是一样的，这个在后面我们可以尝试增加一下输入输出端口来实现更多自定义的功能。目前我们看到的这个界面就称之为 mask，它将 .c 文件里面的输入端口以图形化的方式进行了展现，选中模块后右键 Mask -> Edit Mask 就可以打开 Mask 的编辑界面了，如下图所示：

![image](https://user-images.githubusercontent.com/40487487/170820291-66882826-498e-4e5e-ab7a-fa33484dda1a.png)

以改变 center frequency 这个输入参数的显示为例，在 Parameters & Dialog 选项卡中，选中 Center Frequency，在左侧的 Property Editor 中就可以将这个参数的 type 从 editor 改为 slider，并设置 minimum 和 maximu 值，如下图所示。保存后我们再双击打开 Hackrf_Source 模块，可以看到 Center Frequency 的界面已经变成了 slider 的样式。另外在这个 Property Editor 中还可以设置该输入参数的各种特征，比如是否允许在仿真中修改参数值（Tunable）等，以及整个界面的布局 Layout。总之，整个 Mask 的编辑界面十分友善易操作。

![image](https://user-images.githubusercontent.com/40487487/170820329-cc8efe61-b74f-444e-92df-cbb1c2c9dd95.png)

另外我们还可以再看一下 Initialization 这个选项卡,如下图所示 **这里其实有个很重要的一点**，即这里才是真正关联了 Mask 中的输入参数和 hackrf_source.c 中的参数！如果大家仔细的话会发现，在 hackrf_source.c 中，定义的输入参数名称其实与在 Mask -> Parameters & Dialog 中看见的并不一样，以中心频率设置为例，在 hackrf_source.c mdlCheckParameters 函数中它定义的名字叫做　FREQUENCY，而 mask 中是叫做 center frequency，那他们是如何关联起来的呢？

![image](https://user-images.githubusercontent.com/40487487/170820546-83fbcfbe-cb02-48d3-a3df-cfd0aef14e38.png)

实际上，从 hackrf_source.c 这样的源代码经过 mex 编译后再到 S 函数的第一步是通过 Block Parameters 这个选项卡来实现的，如下图所示。

![image](https://user-images.githubusercontent.com/40487487/170820345-d5076575-35ee-4b02-b864-6152d8f6c3f7.png)

我们可以 右键 ->　Block Parameters (S-function) 来打开这个对话框（如果是新建一个 S 函数的话也会从这个对话框开始)，其中 S-function name 对应是 hackrf_source.c　这个文件的名字，而　S-function parameters 则一一对应 .c 文件 mdlCheckParameters 函数中的输入参数，比如第二项 hackrf_frequency_rounded 其实就对应 mdlCheckParameters 中的第二项 FREQUENCY。

第二步是在 Mask Editor -> Initialization 选项卡中将 S-function parameters 中的变量与 Mask 中显示的变量一一对应，比如我们可以看到有 

> hackrf_frequency_rounded = round(hackrf_frequency); 

即这里是将从 Mask 界面获取到的 hackrf_frequency 取整赋予到 hackrf_frequency_rounded，也即是 .c 文件中的 FREQUENCY 里面去的。

这样就完成了变量从 Mask 界面到 .c 文件的映射。这样的好处也是不言而喻的，比如 hackrf.c 函数中，需要输入的中心频率要求是无符号整数 (uint64_t)，但在 Mask 的 UI 界面我们可以输入任意小数位的频率值，或者是用 slider 操作时可以拖到任意小数频率的数值，这样传递进 hackrf 中数据类型也不会出错。

## 更改 simulink-hackrf source.c 实现更多功能

最后，我们来尝试在 hackrf_source.c　中增加一些输入输出接口以实现新的功能。本来我是期望在　hackrf_source.c　中增加打开 hackrf biasTee 的这个开关，但是我发现我的 hackrf 无论是否打开 biasTee，天线端口都是量到 2.7V。我用 SDRangel 或 Airspy 这样成熟的软件去尝试打开 biasTee，得到的结果也是一样的，所以我怀疑可能是我的 hackrf 这个功能损坏掉了，因此无法验证修改后的代码是否能正确，所以就只新增了一个与 hackrf 完全无关的功能，只是用来介绍一下修改的流程。有兴趣的朋友可以看看 hackrf.c 这个文件，尝试增加一些真正和 hackrf　相关的功能。

我增加的功能很无趣，就是增加了一个输入变量, 然后当这个输入值大于 10 的时候，输出等于输入 +1，否则输出等于 0. 

### Step 1.

首先我们需要增加一个输入变量，名称为 ANT_PORT：在 mdlInitializeSizes 中增加如下代码：这里 SS_PRM_SIM_ONLY_TUNABLE 将该变量设置为允许在运行中调整

> ssSetSFcnParamTunable(S, ANT_PORT,  SS_PRM_SIM_ONLY_TUNABLE)；

然后在 mdlCheckParameters 中增加对该变量类型的检查，确定该变量为数值：

> Assert_is_numeric(S, ANT_PORT);

### Step 2.

再增加一个输出端口，在 mdlInitializeSizes 中做如下修改：其中 ssSetNumOutputPorts(S,2) 将输出端口设置为 2.

> if (!ssSetNumInputPorts(S, 0) || !ssSetNumOutputPorts(S, 2)) return;

然后将该端口的数据宽度设置为 1：

> ssSetOutputPortWidth(S, 1, 1);

### Step 3.

最后在 mdlOutputs 中增加输出变量的函数以实现我们期望的功能：其中 ssGetOutputPortSignal 函数获取了 Output Port 的值，使我们可以对输出信号进行操作。

> real_T       *y2 = ssGetOutputPortSignal(S,1);
> 
> if(GetParam(ANT_PORT)>10){
> 
>    y2[0] = GetParam(ANT_PORT)+1;
>    
> }
> 
> else{
> 
>    y2[0] = 0;
>    
> }

### Step 4.

我们将更改的 hackrf_source.c 另存为 Mine_hackrf_source.c，并对其进行 mex 编译。编译成功后生成 Mine_hackrf_source.mexw64。

然后我们在 simulink 中新建一个 S 函数，在 Block Parameters -> S-function name 填入 hackrf_source_v2; 在 Block Parameters -> S-function parameters 中依次填入 9个输入变量的名称（对应 mdlInitializeSizes 中的 9个输入变量），之后我们就可以发现 S 函数从原始的一个输入端口和一个输出端口的图例变为了仅有两个输出端口的图例，如下图所示。之后我们就可以为这个 S函数定义它对应的 Mask 了。

![image](https://user-images.githubusercontent.com/40487487/170820388-0b2afc98-92b5-45ec-9446-340c31100391.png)

**需要注意的是**，因为在 mdlCheckParameters 中定义了需要检查所有参数的类型为 numeric，以及第七个参数 frame_size 要为 2 的整数次方，所以在第一次创建时要按照规定的参数格式填写 Block Parameters -> S-function parameters，比如 1，2，3，4，5，6，8，7，9 就是满足 mdlCheckParameters 要求的格式。这样才可以正确的关联 S 函数和 hackrf_source_v2.c. 关联之后就可以将这些参数名称改为其他的名称，并在 mask editor 中来定义这些参数的数值，注意这些数值也需要满足 mdlCheckParameters 的要求

最终的输出结果如下图所示，可以看到 hackrf 的接收功能在正常工作，同时另一个端口根据输入参数 ANT_PORT 的变化进行相应的输出。

![mask4-R2019a](https://user-images.githubusercontent.com/40487487/170820437-d2101b71-6abf-411b-87d2-d56aeab97951.PNG)








