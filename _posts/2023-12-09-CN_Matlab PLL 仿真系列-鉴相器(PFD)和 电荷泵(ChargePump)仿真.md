---
layout: post
title: Matlab PLL 仿真系列-鉴相器(PFD)和 电荷泵(ChargePump)仿真
author: Xuxiang Liu
date: 2023-12-09 23:18 +0800
tags: [PLL,MATLAB,CN]
---

Matlab PLL 仿真系列-鉴相器(PFD)和 电荷泵(ChargePump)仿真.
{: .message }


PFD 鉴相器是 PLL 中的第一环，用于比较 reference clk 和 feedback clk 的相位偏差，然后驱动 Charge Pump 产生对应的控制电流。

如图 1 所示，常见的 PFD 由两个 D 触发器构成，同时通过 delay compensation 模块可以避免死区时间。

![image](https://github.com/xuxiang-liu/xuxiang-liu.github.io/assets/40487487/739ed28c-ed40-4b88-99d7-f1b500fc188e)

首先我们来构造 D触发器：D触发器在时钟上升沿进行判断，输出将变为新的 dataIn，在没有上升沿的时候，D 触发器输出保持与上次一致。同时 D触发器通过 CLR 信号来进行复位，CLR 信号从图中可以看到是低有效的。

因此整个 D 触发器的 .m 代码如下所示：

![image](https://github.com/xuxiang-liu/xuxiang-liu.github.io/assets/40487487/7a8ce2a0-e37a-4928-90b9-51967c333274)

接下来，我们来构造整个 PFD 模块，核心是如何实现 delay 模块：为了实现 delay 模块，我们需要创建一个数组，这个数组用来存储长度为 delay/Ts的 clear 信号，因为当 clear 信号产生之后，如果不去 clear 掉，则 clear 信号会一直存在，而这个存在时间就是 delay time，因此我们需要准备一个数组来存储这段的 clear 信号。

在每个仿真时刻，我们都会检测当前的时间点是否在存储 clear信号的数组中，如果是，则可以进行 clear 操作，如果不是，则正常通过 D-FlipFlop 进行输出。最终再更新所有的状态即可。

PFD 的整个 .m 代码如下：

![image](https://github.com/xuxiang-liu/xuxiang-liu.github.io/assets/40487487/78c5052a-2ce2-43ff-806f-020e7804b7ad)

最后我们再来准备一个测试 PFD 的代码：首先需要定义一个采样频率 Fs，这个频率（时间）是整个仿真的系统时间。接下来，我们要定义 PFD 的两个输入时钟信号频率 fRef 和 fIn，由于 Nyquist 限制，这两个频率一定是低于采样频率 Fs 的。然后我们定义 delay 时间，再将它转换为对应的采样个数，即对 t_dalay/Ts 取整。

完整的测试 .m 代码如下：

![image](https://github.com/xuxiang-liu/xuxiang-liu.github.io/assets/40487487/90b661e1-c3b1-446a-af81-155978d8113f)
![image](https://github.com/xuxiang-liu/xuxiang-liu.github.io/assets/40487487/638f1429-6f2b-457d-b746-03e564b0e4da)

最后我们来看一下 PFD 输入与输出的关系，我们展示前 6000个采样点的结果：

![image](https://github.com/xuxiang-liu/xuxiang-liu.github.io/assets/40487487/c5eb615d-48d6-4b18-95c2-a4420f6032be)

