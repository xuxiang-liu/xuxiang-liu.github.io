---
layout: post
title: Hackrf-DC-滤波器
author: Xuxiang Liu
date: 2022-09-02 23:18 +0800
tags: [SDR,CN]
---

Simulink 构造 DC 滤波器并通过 Hackrf 进行验证.
{: .message }

## DC spike

初次拿到 hackrf 的朋友们都会发现在频谱的中央看起来会有一个很强的”信号”，如下图所示：实际上这并不是真实的信号，而是接收机本身所产生的直流偏置（DC offset），在 hackrf 的官网 FAQ中对它的产生有详细的介绍，有兴趣的朋友可以直接参考这里：https://hackrf.readthedocs.io/en/latest/faq.html

![image](https://user-images.githubusercontent.com/40487487/188273592-e8e3aae4-4226-4f00-b022-70da5315118d.png)

FAQ 中提到任何的 quadrature sampling system 都会存在这样的直流偏置，这里的 quadrature sampling system 应该就是我们最常用到的 IQ 架构的收发机了。那么我们该如何对待这样的直流泄漏呢？

在讲述下面三种方式之前，需要先明确一点，以下三种方式都是在已经产生 DC offset 的情况下的解决方式，从根本上来讲，我们可以通过对 IQ 通路进行校准来减少 DC offset。当然，这不属于我们目前讨论的范畴。

一般来说在实际应用中因为绝大部分信号都是有一定带宽的，因此中心的 DC offset 也许并不会对接收信号造成太大的影响，例如常见的 FM 解调，DC 的部分其实根本无法被人耳所听见，因此对接收解调并没有太大的影响。

另一种常见的方法是在一定 offset 的中心频率上进行接收，然后通过数学变换得到真正想要的信号，这样就完全不会受到 DC offset 的影响，如下面的公式所描述：

![image](https://user-images.githubusercontent.com/40487487/188273622-288da2fd-069b-4a78-a002-fda6a92cb9b2.png)

最后还有一种方式是直接进行滤波，将中间的 DC offset 滤掉，这也可以让我们真正在频谱上看到变化，也是很多 SDR 软件常用的一种方式，如 Airspy 中的 Correct IQ 功能。这篇 blog 的主要目的就是来实现类似 Airpsy 中 Correct IQ 这样功能的实时滤波器。

![image](https://user-images.githubusercontent.com/40487487/188273631-11191eb1-7bb4-4e91-a038-74e70b723a3b.png)

## DC 滤波器

最简单的实现 DC 滤波的方式就是对一定长度的 IQ 数据取平均，然后直接减去平均值即可做到 DC 滤波了。当然，显然这不是一种实时的滤波方法。

参考这篇 https://dsp.stackexchange.com/questions/40734/what-does-correcting-iq-do 博客，我们可以搭建这样一种数字的 DC 消除滤波器，结构如下图所示，其中 a 称之为滤波器增益系数，控制着滤波器的带宽，当 a 越趋近于1的时候，滤波器的带宽会越窄。初看到这里的时候，我们可能会好奇为了实现最好的 DC 滤波效果，a 是不是应该设置的越接近1越好？还是说这个增益的设置会带来别的影响？后面我们会通过 simulink 仿真来进行说明。

![image](https://user-images.githubusercontent.com/40487487/188273800-017c7376-9b78-4401-b0c4-229a497c81da.png)

其实在这篇博客 https://flylib.com/books/en/2.729.1/dc_removal.html 中有提到，上述的DC消除滤波器都可以用一种通用的 DC 消除滤波器的架构来表示，如下图所示。其中 a 是增益系数，同样a越接近于1滤波的效果越好。除去增益系数的表达方式不同以外，他们的 z 域模型都可以写为：

![image](https://user-images.githubusercontent.com/40487487/188273812-f43ea3a6-6e8c-4312-a99e-913420cd89af.png)

这里表示的是上一时刻的数据点。

![image](https://user-images.githubusercontent.com/40487487/188273835-27d521e7-c217-436f-aa73-3aeba54b94eb.png)

既然两种模型除了增益系数的表达不一样，本质都是一样的，我们则以第二种 DC 滤波器为例进行实现，当然有兴趣的朋友也可以参考这篇blog来实现第一种 DC 滤波器，来验证是否和第二种DC滤波器有同样的效果。

## DC 滤波器静态模型

DC 消除滤波器的静态模型可以直接通过 matlab 代码进行实现，实现方式十分简单：

Step1. 载入 hackrf 收到的原始数据，在matlab中是以 I+jQ 的复数形式表达的，因此可以先画出原始数据的频谱，如下图所示，可以看见中间有一个很明显的毛刺，这就是 DC offset。另外从 IQ 数据也可以看出，IQ 信号并不是关于0点对称的，这就是从时域上看到的 DC offset。

![image](https://user-images.githubusercontent.com/40487487/188273858-a8a4af80-efc0-4c1e-a2a6-e7981d8667ce.png)

Step2. 根据上述模型实现滤波器，即当前时刻点的滤波器输出  与当前时刻点的输入 及上一个时刻点的输入 有以下关系：

![image](https://user-images.githubusercontent.com/40487487/188273870-0f54e1f7-7574-4688-9da9-3a58dd22a09c.png)

对应的代码如下所示：其中filter_gain2 就是前面提到的滤波器增益系数我们可以先设置成0.99。之后我们可以画出滤波后的频谱与 IQ 数据，如下图所示：

> temp = zeros(length(IQdata),1);
> 
> temp(1) = 0;
> 
> for i=2:1:length(Input2)
> 
>    temp(i) = temp(i-1)*filter_gain2 + Input2(i);
>    
> end
> 
> temp_last = [0;temp(1:end-1)];
> 
> IQ_DCremove2 = temp-temp_last;

![image](https://user-images.githubusercontent.com/40487487/188274061-6afaa805-a10e-4478-b965-5f611a46fe53.png)

可以明显看到滤波后的频谱已经没有了直流分量，另外 IQ 数据也基本是关于0对称的了。到这里，看起来我们的 DC 消除滤波器已经能很好的实现它的功能了，可是如果你有足够的好奇心的话会发现，如果我们调整滤波器的增益系数会得到什么变化呢？另外到底这个 DC滤波器的带宽是什么样的呢？

## DC 滤波器的 Simulink 模型

为了更好的回答上述两个问题，我们通过 Simulink 建模来进行。现在我们可以通过 simulink 来获取 Hackrf 的数据，因此可以进行实时的分析。上图的框图模型在 matlab 中可以搭成如下图所示：

![image](https://user-images.githubusercontent.com/40487487/188273922-cdab7680-3c9d-4f5a-ba84-a89e16d9f530.png)

整个模型如下图所示，注意图中还保留了不经过 DC nulling filter 和 第一种 DC nulling filter 的架构。运行改 simulink，我们可以看到动态的 DC 消除的滤波效果（见最后的GIF）。

![image](https://user-images.githubusercontent.com/40487487/188273928-1bb9f38a-5e75-4a5e-a271-166241f856c1.png)

那么回到我们之前的问题，如果调整增益系数越趋近于1会有什么效果呢？以及这个 filter 的带宽是是多少呢？我们可以通过波特图来进行分析。波特图在 Simulink 中选择 Analysis -> Control Design -> Linear Analysis，打开后弹出将 Analysis I/Os 改为Root Level Inports and Outports，之后选择 Bode 来查看波特图，如下图所示：

![image](https://user-images.githubusercontent.com/40487487/188273937-95654db1-d3a9-47e8-be22-a5d76e1d1446.png)

我们可以看到，通过改变Filter Gain 当 Filter Gain 越接近于1，滤波器的带宽越窄。比如，下图中我们的采样率为 1kHz，因此分析的带宽为 500Hz，当Filter Gain = 0.999时-3dB带宽为0.000394Hz，而当 Filter Gain = 0.99时-3dB 带宽为0.00398Hz，有10倍的差异。

![image](https://user-images.githubusercontent.com/40487487/188273941-d02c81bf-9234-4d22-8c62-43fdc2346319.png)

但是如果我们将 Fillter Gain 越趋近于 1，我们可以从下面的动态滤波效果中看到，会花更长的时间来达到稳定状态。

Filter Gain = 0.95

![filterGain=0 95](https://user-images.githubusercontent.com/40487487/188274077-d47f5503-95fd-4c00-9f43-7415121ac170.gif)

Filter Gain = 0.9995

![filterGain=0 9995](https://user-images.githubusercontent.com/40487487/188274081-dd3e0fe6-8974-4338-995d-bf5983980fb9.gif)

## 总结

以上就是关于 DC 滤波器的所有内容，但我想再次说明的是，在实际无线接收机设计中，我们都是通过对 IQ 通路的校准来实现 DC 消除，这个过程一般称之为 DC offset calibration. 目前我们做的都是基于已经得到的 IQ 信号的操作，实际应用中我只需要增加一个 offset 也可以解决。




