---
layout: post
title: 初识 pyhackrf2 和 python-hackrf.
author: Xuxiang Liu
date: 2026-07-11 23:16 +0800
tags: [HACKRF,CN]
---

基于 python 玩 hackrf
{: .message }

# 初识 pyhackrf2 + linux python venv 环境安装 pyhackrf2

之前发现了 pyhackrf2 这个基于 python 的 hackrf 库，十分好用，基于这个库不仅可以完成 FM Radio，ADS-B 追踪等项目的开发，而且 GUI 制作也十分简单。因此特意用这个文档系列来记录 pyhackrf2 的使用，后续也会基于此来进一步介绍 FM Radio，ADS-B 追踪项目


## 在 CLION IDE 创建 PYTHON PROJECT
按照下述步骤在 CLION 中可以直接创建 PYTHON PROJECT：

step1. 新建一个 C Executable 项目

step2. 新创建的 C Executable 项目一般就只有一个 main.c，在项目根目录下新建一个 python file，只需要在 python file 中写例如 # 或 include 等 python 格式内容，IDE 会自动提示没有 python interpretor，右上角会自动弹出 python config interpretor 的配置选项，点击即可创建 python 的 venv 环境；当然也可以在菜单栏 Settings 中去配置 python 编译器

step3. 为更好的管理 python 安装包环境，我们可以创建一个 venv 环境，在 interpretor 配置中选择 show all，然后创建一个 venv 环境

step4. 创建好虚拟环境后，可以在左边 Project 栏看到多了 venv 路径，我们后续安装各类 package 都需要到 venv 环境中安装

step5. 最后我们跑一个最简单的 print('hello world\n')，可以看到在下方的控制台中可以打印出 hello world，至此在 CLION IDE 中创建 python project 就完成了

## 安装 pyhackrf2

pyhackrf2 的官方文档见这里：https://pypi.org/project/pyhackrf2/ 

在新建好的 python project 中，我们输入 from pyhackrf2 import HackRF，可以看到会报错没有这个库，我们按下述方法来安装：

step1. 进入 path/venv/bin, 执行 source activate 进入到 venv 环境

step2. 直接输入 pip install pyhackrf2 即可完成安装; 安装完成后，在 venv/lib/python3.12/site-packages 下就可以看到 pyhackrf2 和 pyhackrf2-1.0.3.dist-info 两个文件夹了

step3. 最后连接 hackrf，通过下面简单的脚本即可判断是否成功，如果能返回 hackrf 的序列号，则说明整个环境已搭建成功

...

from pyhackrf2 import HackRF

device = HackRF()

print(device.enumerate())

...

## 安装 pyython-hackrf

不过就在写这个 blog 的时候，发现其实维护更好的一个库是 python_hackrf，由 GvozdevLeonid 所维护，官方文档：https://pysdr.org/content/hackrf.html 所以让我们也安装一个这个库来试试吧（之前我的 ADS-B， FM 等都用的 pyhackrf2，所以在写这个 blog 的时候也想干脆用 python_hackrf 库来改写吧，顺便也理清一下之前项目逻辑，看能否提高运行的效率）

step1. 同样进入 path/venv/bin, 执行 source activate 进入到 venv 环境

step2. 直接从 git 拉取即可，也可以用 pip install hackrf_python, 安装完成后可以看到同样 site-packages 路径下多了 python_hackrf 和 python_hackrf-1.5.0.dist-info 

step3. 最后连接 hackrf，通过下面简单的脚本即可判断是否成功，如果能返回 hackrf 的序列号，则说明整个环境已搭建成功

...

from python_hackrf import pyhackrf


pyhackrf.pyhackrf_init()

device = pyhackrf.pyhackrf_open()

print(pyhackrf.pyhackrf_library_version())

print(device.pyhackrf_board_partid_serialno_read())

...

## python_hackrf RX 数据接收

我们第一个基于 python_hackrf 的 demo，就是简单的用 python_hackrf 来打开 HackRF 的 RX 通道，收取每一帧数据并返回每一帧数据的有效长度；这也是很多 hackrf 有趣应用的基础，一旦 HackRF 硬件层的原始数据传到了 python 当中，我们就可以用 python 做很多的算法及应用层处理了

python_hackrf 这个库本质上是对 C 语言的 libhackrf 做的一层 Python 封装（通常基于 ctypes 或 cffi/Cython）。HackRF 硬件在底层是通过 C 语言的函数指针回调机制工作的：因此我们首先需要有一个 C 语言的 callback 回调函数，可以理解成 HACKRF 硬件驱动每次填满一个 USB 传输缓冲区，就会在C 层调用我们注册的回调函数。

当然因为我们是基于 python 的开发，因此 python_hackrf 库会把这个 C 层的回调，包装成一个"胶水层"函数：pyhackrf.__rx_callback；在 C语言层实际调用的是这个函数，然后我们自己定义的 python 的 callback 函数在这个 pyhackrf.__rx_callback
中再被调用，就实现了从 C 到 python 的转换

因此我们需要按照 C 语言 libhackrf 中的参数要求来构造 python 的这个 callback 函数，pyhackrf.__rx_callback 中，对于 callback 函数的输入要求是：__rx_callback(PyHackrfDevice, ndarray,int,int)，因此我们的 callback 函数也需要保留同样的格式，所以我们的 callback 函数需要写成这样：

```python
def rx_callback(device, buffer: np.ndarray, buffer_length: int, valid_length: int):
```


其中 buffer 就是收到的数据，buffer_length 和 valid_length 都是数据长度，由于 HackRF 的数据是按 I 和 Q 交替的形式返回的，I 和 Q 都是 8bit 的 ADC 采样数据，因此收到的总的 IQ sample = buffer_length/2

所以我们最终可以这样来在每次 rx_callback 中打印出收到的数据长度：

``` python
def rx_callback(device, buffer: np.ndarray, buffer_length: int, valid_length: int):

    """
    device: PyHackrfDevice 设备对象
    buffer: numpy.ndarray，接收到的 IQ 数据
    buffer_length: 缓冲区总长度
    valid_length: 本次实际有效的数据长度
    """
    accepted = valid_length // 2 # I 8bits, Q 8bits
    accepted_samples = buffer[:valid_length].astype(np.int8) # -128 to 127
    accepted_samples = accepted_samples[0::2] + 1j * accepted_samples[1::2]  # Convert to complex type (de-interleave the IQ)
    accepted_samples /= 128 # -1 to +1


    print(len(accepted_samples))
    return 0  # 返回0表示继续接收
```

最后我们在主函数中通过 set_rx_callback(rx_callback) 来注册回调函数，并通过 pyhackrf_start_rx() 来启动 RX 即可

```python
    sdr.set_rx_callback(rx_callback)

    print("开始接收数据... 按 Ctrl+C 停止")
    sdr.pyhackrf_start_rx()
```

最终打印出来的数据长度应该是 = 262144/2，其中 262144 直接返回的最原始的 byte 数量，他也是每次 USB 传输的固定的大小 256KB，对应的 sample 数就是 131072 个 IQ sample











