---
layout: post
title: 实现基于 Maltab 的 Hackrf FM收音机
author: Xuxiang Liu
date: 2023-07-08 23:18 +0800
tags: [SDR,CN]
---

用matlab 实现 Hackrf 收音机.
{: .message }

距离上次写 blog 好像已经一年了...

Capri 芯片项目也做失败了，这段时间终于有一些时间来继续我的hackrf 博客了～ 闲话少说，这次先来看一个 hackrf 最简单最普遍的应用 - FM 收音机。我相信所有玩过 hackrf 的朋友都是从这里起步的，不过也有大部分朋友也就止步于此了... 

Anyway，我们来看看吧，这篇 blog 不会像网络上能搜到的绝大部分 blog 一样只是讲一讲如何用 hackrf 实现 FM 收音机，而希望走信号处理的角度来聊聊如何实现 FM 收音机。众所周知，FM 调频收音机的本质是将音频信号调制到载波再发出去，那么第一个问题来了，FM广播信号的带宽是多少呢？

根据这篇连接的描述（https://www.aaronscher.com/wireless_com_SDR/RTL_SDR_AM_spectrum_demod.html），中国的FM广播频率范围为 87～108MHz，频道带宽为 200kHz，那么这个音频信号为什么会占据200kHz 带宽呢？实际上在这200kHz 带宽存在多种不同的数据传输类型：根据这篇连接的描述（https://www.aaronscher.com/wireless_com_SDR/RTL_SDR_AM_spectrum_demod.html），FM 信号频谱如下图所示：

![image](https://github.com/xuxiang-liu/xuxiang-liu.github.io/assets/40487487/9972d2fb-cc2f-4180-82b4-64fba5289a09)

可以看到广播的单声道音频主要集中15kHz 范围以内，而立体声音频则在 23-53kHz 范围以内，在 19kHz 的位置还有一个立体声伴音信号，用于辅助接收机解调立体声信号；单音+立体声广播信号也称之为 FM 立体声广播系统。我们在后面的 blog 中再对此专门进行介绍。

在 57kHz 位置还有一个 RDS/RBDS 信号，全称 radio data system/radio broadcast data system，是一种标准的数字通信协议用于传输少量的数字信息（1187.5bps）。典型的应用场景是在收音机接收时，同步显示相应的歌曲名、歌词、时间或广播站的名称（在国内似乎用的很少），如下图所示：这块儿我们在后面的blog中再来介绍。

![image](https://github.com/xuxiang-liu/xuxiang-liu.github.io/assets/40487487/72a2bddf-385c-49c0-bd30-393738379a3e)

回到对 FM 信号的解调中，现在我们可以更准确的表述是对 FM 单声道信号的解调。首先我们用基于 Matlab 的 Hackrf 采集一段 FM 调频信号，例如中心频率 102.6MHz，2Msps，采集 5s，通过 FFT 变换得到其频谱如下图所示：

![image](https://github.com/xuxiang-liu/xuxiang-liu.github.io/assets/40487487/97eb8e84-0ee4-4e4d-8714-8ec403436d2d)

接下来我们对信号通过一个低通滤波并做8倍降采样，得到带宽 250kHz 的信号，如下图所示。这也是我们要从中提取音频信号的原始信号了。

![image](https://github.com/xuxiang-liu/xuxiang-liu.github.io/assets/40487487/676db63c-b61c-4fa3-bdf3-47c60c0e9260)

接下来我们来做 demodulation，这里其实和蓝牙的 demodulation 是一样的原理，即相位等于频率的积分，因此我们可以通过 IQ 信号获取到相位信息，然后对相位做微分得到频率。



做微分其实就是用相位差除以间隔时间，因此我们可以用简单的这段代码实现：

FM_Phase = unwrap(angle(IQdata_Downsampled));

FM_Demod = (diff(FM_Phase))/(2*pi)*Fs_Downsampled;

FM_Demod = FM_Demod./max(abs(FM_Demod));

注意在求相位时需要对相位做 unwrap，否则相位将只在 (0,2*pi) 变化，无法求到准确的相位差，unwrap 前后的相位信息对比如下图所示：

![image](https://github.com/xuxiang-liu/xuxiang-liu.github.io/assets/40487487/41ae83bd-3a9a-4044-9775-c8d951a9c238)

通过上式即可求得解调出来的频率信号，我们对这个频率信号做 FFT 变化，Zoom-In之后可以看到在 15kHz 内的单声道信号，19kHz 的 stereo pilot signal，不过看起来 38kHz 两边的立体声信号很弱，可能并没有立体声信号。

![image](https://github.com/xuxiang-liu/xuxiang-liu.github.io/assets/40487487/dedb77ba-be95-4eda-98c6-b8898a26d7d9)

这是，如果直接把整个信号灌到 Audio，可以听到很明显的“叮”的单音信号，也就是这里的 19kHz stereo pilot 信号。为了有更好的效果，我们需要再次降采样，只保留 15kHz 内的单声道信号，这里我们可以直接对解调出来的 FM 信号进行8倍降采样，得到如下图所示的频谱：

这个信号就可以通过简单的一行 matlab 命令灌给 PC 的声卡，并听到相应的广播了！当然我们需要注意这里的 Fs_Audio 已经从最初的 2MHz 变到了 31.25kHz,　如果我们改变这个Fs_Audio （与 Audio_Signal 实际采样频率不同）会发现广播的声调发生了变化，会从原始的女声变为男声，这应该也是很多变音设备常用的方式了。

![image](https://github.com/xuxiang-liu/xuxiang-liu.github.io/assets/40487487/9e5f5cb5-9b8d-4a74-993d-320848bd4565)

sound(Audio_Signal,Fs_Audio)

以上就是通过 Maltab 解调获取 FM 广播的全部过程，如果希望能实时的通过 Matlab 实现基于 Hackrf 的收音机，我们也可以通过 Simulink 来搭建这样一个过程，整个 Simulink 和前文描述的方法完全一致：

![image](https://github.com/xuxiang-liu/xuxiang-liu.github.io/assets/40487487/77e52819-9966-4a60-838e-3a68e1c3a103)
