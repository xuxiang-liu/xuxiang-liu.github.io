---
layout: post
title: Matlab PLL 仿真系列-环路滤波器之传递函数仿真
author: Xuxiang Liu
date: 2023-12-31 23:18 +0800
tags: [PLL,MATLAB,CN]
---

Matlab PLL 仿真系列-环路滤波器之传递函数仿真.
{: .message }

环路滤波器是 PLL 里面非常重要的一环，在 Matlab 模型里面也至关重要，在 Loop Filter 的仿真里面涉及到从模拟滤波器到离散滤波器的转换，这一步也很容易导致 Matlab 模型无法正确的收敛。

首先我们看一下 PLL 的环路滤波器架构，这里我们主要考虑常用的 2阶、3阶 和 4阶环路滤波器。关于环路滤波器的理论知识，以及如何设计环路滤波器可以参考 Dean Banerjee 的 “PLL Performance, Simulation and Design” 这本小书。

参考 Matlab 的 Help 文档，可以看到不同阶数环路滤波器系数推到公式如下：

二阶环路滤波器及其传递函数：
  
![image](https://github.com/xuxiang-liu/xuxiang-liu.github.io/assets/40487487/b26d05c2-7e42-45b5-b68c-7ae137cfa84c)

三阶环路滤波器及其传递函数：

![image](https://github.com/xuxiang-liu/xuxiang-liu.github.io/assets/40487487/b907ed41-80c9-4442-9517-e1bb44afc831)

四阶环路滤波器及其传递函数：

![image](https://github.com/xuxiang-liu/xuxiang-liu.github.io/assets/40487487/2a4e0629-2130-45cb-820e-2160e63c975c)

系数和 R,C 的关系如下所示：

![image](https://github.com/xuxiang-liu/xuxiang-liu.github.io/assets/40487487/c82d74a2-791c-4bc7-8581-4e67ae6d6014)

在实际设计中，需要根据所需的环路滤波器带宽、相位稳定裕度等来选择合适的 R,C；而在我们的 PLL 仿真中，我们仅考虑如何将已知的 R,C 转成相应的滤波器系数，并选择正确的环路滤波器增益以保证仿真的收敛性。**尤其需要注意的是，这里公式给出的环路滤波器都是在 s 域的，即连续域里面，而我们需要将连续域的滤波器转成离散滤波器，才能进行仿真**。

我们先通过 Simulink 来设计一个简单的二阶环路滤波器，采用 Simulink 默认的参数，环路滤波器带宽是 500kHz，相位裕度是 45度，KVCO = 100MHz/V, Icp = 1mA, divN = 70； 那么Simulink 给出的 R,C 设计值是：

![image](https://github.com/xuxiang-liu/xuxiang-liu.github.io/assets/40487487/321c2454-9276-4a43-8e7e-8800972d33a0)

我们来看一下通过 .m 仿真出来的环路滤波器传递函数是否满足设计时提出的带宽与相位裕度的要求。

首先我们需要创建一个 .m 文件将 R,C 按照上述公式转换为相应的模拟滤波器系数。同时因为后续的仿真都是在离散域中进行的，我们把从模拟滤波器到离散滤波器的转换也在这个 .m 文件中一并完成。

在从连续到离散滤波器的转换过程中，我们可以通过两种不同的方式进行，一种是通过 Bilinear 方法来进行转换，另一种是通过Impulse invariance 的方法来进行转换。关于这两种方式的原理和差异可以参考 Matlab 相应的帮助文档。

完整的环路滤波器离散系数计算的 .m 文件如下：

![image](https://github.com/xuxiang-liu/xuxiang-liu.github.io/assets/40487487/f630d58d-658a-49ab-bed7-ff40071595cd)
![image](https://github.com/xuxiang-liu/xuxiang-liu.github.io/assets/40487487/72d9044d-164b-4e99-8c0a-0734cd61d55b)

接下来我们看一下 Simulink 生成的这个环路滤波器的性能。首先我们列出整个 PLL 的传递函数：

![image](https://github.com/xuxiang-liu/xuxiang-liu.github.io/assets/40487487/fd77e0e3-50de-45e1-844a-a55f91cd18c0)

其中 CP 的传递函数可以写为：

H_CP=I_CP/2π

VCO 的传递函数可以写为：

H_VCO=2π*K_VCO/s

因此 CP, LoopFilter 和 VCO 组成的开环传递函数可以写为：

G(s)=H_CP*Z(s)*H_VCO

整个 PLL 的开环传递函数可以写为：

H_open (s)=G(s)/N

整个 PLL 的闭环传递函数可以写为：

H_close (s)=G(s)/(1+N*G(s) )

其中 N 是分频系数的倒数。因此，画出开环传递函数的波特图的代码如下所示，**注意在分析环路滤波器及整个系统的传递函数时，我们都是采用连续域的滤波器系数来进行计算的，只有后面我们进行 PLL 仿真时，才会用到离散域的滤波器系数**

![image](https://github.com/xuxiang-liu/xuxiang-liu.github.io/assets/40487487/9b29e7ed-82ff-4665-8684-6e19078ff4e3)
![image](https://github.com/xuxiang-liu/xuxiang-liu.github.io/assets/40487487/631a4d0d-afd2-451c-aa4a-03f2b4198016)

开环环路滤波器的波特图如下所示，可以看到相位裕度确实是 45度（Pm = 180+(-135)），0dB增益的环路带宽是500kHz，与设计是一致的

![image](https://github.com/xuxiang-liu/xuxiang-liu.github.io/assets/40487487/831da6d3-83fc-4f87-8d72-cb944ebeefc4)

同样，闭环波特图如下所示：可以看到 3dB 的环路带宽约为 830kHz

![image](https://github.com/xuxiang-liu/xuxiang-liu.github.io/assets/40487487/a7697f01-fbe4-431d-aec1-9c58e36fa278)

该闭环传递函数对应的冲击响应和阶跃响应 如下图所示：

![image](https://github.com/xuxiang-liu/xuxiang-liu.github.io/assets/40487487/6b095277-d6ec-43fa-9b23-3fc48a363020)

我们也可以在 Simulink 的 PLL Block Parameters 对话框中直接进行闭环传递函数的分析，步骤和结果如下图所示：可以看到，与我们 .m 文件计算的结果完全一致

![image](https://github.com/xuxiang-liu/xuxiang-liu.github.io/assets/40487487/75215a15-7850-4c54-a623-f5403d4a3c2c)

![image](https://github.com/xuxiang-liu/xuxiang-liu.github.io/assets/40487487/3f04e374-ed8d-4d0e-be34-41f26d7fcb94)

在环路滤波器的下一部分，我们将看如何仿真 Charge Pump 脉冲电流经过环路滤波器的过程。



