---
layout: post
title: Matlab PLL 仿真系列-概述
author: Xuxiang Liu
date: 2023-11-19 23:18 +0800
tags: [PLL,MATLAB,CN]
---

Matlab PLL 仿真系列-概述.
{: .message }

## 前言
很久没有玩 Hackrf 了，因为最近在做 Matlab 两点调制的仿真，突然很想将整个过程记录下来，因为这是我认为对整个 RF 架构的理解中最有意思的一部分，后续有机会也会增加一些例如 SAR ADC 或 DCDC 等的 MATLAB 仿真。

我计划将整个 Matlab 两点调制过程分为两部分，一部分是 Matlab 进行一个标准的 Charge Pump PLL 仿真，这个部分就叫做 Matlab PLL 仿真系列吧；另一部分是基于此 PLL Matlab 模型添加两点调制相关内容及基于 Matlab 的 BLE RF PHY 测试代码（这部分主要还是改自 Matlab 自带的 Bluetooth Toolbox，有兴趣的朋友也可以直接在 Matlab Toolbox 搜到相应的内容）

其实第一部分在 Matlab Simulink Mixed Signal toolbox 中也有已经建好的模型，但是直接使用 Simulink 中模型实际上会让人错过很多精彩的细节，也不利于我们做的更深入，因此我们会通过 .m 文件来构建 PLL 模型，整个过程中也会参考 Simulink 中的 PLL 模型，并将结果与之对比。

## Charge Pump PLL 模型

Charge Pump PLL 模型主要的组成部分包括：鉴相器（PFD），电流泵（Charge Pump），环路滤波器（Loop Pass Filter），压控振荡器（VCO）和分频器（Divider），考虑小数分频的 PLL，因此这个 divider 也常常采用 Delta-Sigma Divider（DSM）。整个框图如下所示:

![image](https://github.com/xuxiang-liu/xuxiang-liu.github.io/assets/40487487/3faffd8a-b340-41d7-a92e-b059946259fe)

在Simulink 中，我们可以直观的看到最终的输出结果及中途每一步的过程量，这个模型本身采用了变步长的求解器来提高仿真的速度和精度。后面我们将把每个模块改写成 .m 文件，并最终通过定步长的求解方式来。

后面的 blog 将分别以鉴相器和电流泵、环路滤波器、VCO和分频器、Delta-Sigma Modulator 这几个模块为核心展开
