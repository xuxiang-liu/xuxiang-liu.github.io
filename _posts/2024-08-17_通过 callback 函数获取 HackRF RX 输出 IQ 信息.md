---
layout: post
title: 通过 callback 函数获取 HackRF RX 输出 IQ 信息
author: Xuxiang Liu
date: 2024-08-16 23:18 +0800
tags: [SDR,CN]
---

获取 HackRF RX IQ 数据信息
{: .message }

下面我们看一下如何在 Linux 环境下如何通过 HackRF 获取到采到的 IQ 数据，这将是通过 HackRF 做各种数据处理的第一步：

不得不说 ChatGPT 确实让 programming 简单了太多，下面的很多内容都来自于 GhatGPT，特此致敬

在前几篇 Blog 中，我们写了一个简单的 HackRF 工程来获取 Hackrf 的 ID 和各种版本信息。下一步我们来看一下如何获取通过 HackRF 收到的 IQ 数据信息。

## 接收机参数配置

首先我们需要对 HackRF 的接收机参数进行配置，HackRF library 中通过以下几个函数进行 RX 配置：

1. hackrf_set_freq：该函数用于配置 HackRF 的工作频率，理论的配置范围是 1-6000MHz，单位是 Hz，Default 是 900MHz

2. hackrf_set_sample_rate：该函数用于配置 HackRF 的采样率，理论的配置范围是 2-20MHz，单位是 Hz，Default 是 10MHz

3. hackrf_set_vga_gain：该函数用于配置 HackRF 收发器芯片的 BB Gain，理论配置范围是 0-62dB，单位是 dB，步进是 2dB

4. hackrf_set_lna_gain：该函数用于配置 HackRF 收发器芯片的 LNA Gain，理论配置范围是 0-40dB，单位是 dB，步进是 8dB

5. hackrf_set_amp_enable：该函数用于使能/去使能 HackRF 收发器芯片前端的射频放大器，可以提供 14dB 的增益，理论配置范围是 0 或 1

为方便调用，我们可以创建一个 hackrf_rx_config 函数来进行上述的 RX 配置，如下图所示：

```C
int hackrf_rx_config(hackrf_device* dev,uint64_t freq_Hz,double sample_rate,uint32_t vga_gain_value, uint32_t lna_gain_value){

	int result = HACKRF_SUCCESS;

	// Set frequency
	result = hackrf_set_freq(dev, freq_Hz);
	if(result != HACKRF_SUCCESS){
		printf("HackRF Set Frequency Fail \n");
	}
	else{
		printf("HackRF Frequency Set to %lu \n", freq_Hz);
	}

	// Set Sample Rate
	result = hackrf_set_sample_rate(dev, sample_rate);
	if(result != HACKRF_SUCCESS){
		printf("HackRF Set Sample rate Fail \n");
	}
	else{
		printf("HackRF Frequency Set to %f \n", sample_rate);
	}

	// Set VGA Gain
	result = hackrf_set_vga_gain(dev, vga_gain_value);
	if(result != HACKRF_SUCCESS){
		printf("HackRF Set VGA Fail \n");
	}
	else{
		printf("HackRF VGA GAIN Set to %u \n", vga_gain_value);
	}

	// Set LNA Gain
	result = hackrf_set_lna_gain(dev, lna_gain_value);
	if(result != HACKRF_SUCCESS){
		printf("HackRF Set LNA Fail \n");
	}
	else{
		printf("HackRF LNA GAIN Set to %u \n", lna_gain_value);
	}

	// Disable Amplifier by Default
	result = hackrf_set_amp_enable(dev, 0);
	if(result != HACKRF_SUCCESS){
		printf("HackRF Disable Amp Fail \n");
	}
	else{
		printf("HackRF Disable Amp \n");
	}

	return result;
 }
```

## RX Callback 函数

接下来我们定义一个 callback 函数，用于当 HackRF RX 每次采集 Buffer 装满后调用，我们也可以在这个 callback 函数中将 HackRF 接收到的 IQ 数据采集出来，然后进行后续的数字信号处理。callback 函数本身会返回一个 int，注意返回值为 0 的时候会继续下一次的 rx；返回值为 1 就会在此次 callback 后停止接收。

callback 函数通过一个 hackrf_transfer 结构体来进行 USB 数据传输，hackrf_transfer 结构体包括了:

1. hackrf_device* device: 用于定义发起此次 USB 传输的 HackRF device

2. uint8_t* buffer: 这个就是 USB 传输的 Data，以交织的 8bit I 和 Q 的形式进行传输。注意，HackRF 在 2014.08.01 version 开始改为 signed 数据，即有符号数。因此大家可以通过查询自己 HackRF 的版本信息来判断是否采用 无符号数 或 有符号数。关于这一改动的记录在这里：

![图片](https://github.com/user-attachments/assets/265a2ab9-8e09-465c-9e59-b3be7f0bdd39)

3. int buffer_length：返回 Buffer 长度

4. int valid_length: 这个用于返回此次 callback 中有效的传输的 bytes 数量，注意这个数量除以 2 才是 IQ 数据的数量

5. void* rx_ctx: 是用户可以往 callback 函数中传输的内容，基本没有用到过。
 
因此我们可以写一个简单的 callback 函数来打印出 callback 函数的频率与每次调用后输出 Buffer 的有效数据长度（也就是采集到的 IQ 数据数量）。这样可以与我们设置的 Sample Rate 做对比。

一开始我尝试使用 clock_t current_time = clock() 的方式进行时间记录，但是发现这种方式在 callback 函数中无法正确的记录 callback 的频率，于是改为采用 timespec 来记录绝对的时间。实话说我不知道为什么 clock_t 的方法无法正确记录 callback 函数的频率，我猜测与 clock_t 只记录 CPU 的时间有关，因为我通过 sleep 的方式去使 hackrf 处于一段时间的 RX 状态，因此可能在这个 sleep 中间程序并没有使用 CPU 所以导致计时的错误。Anyway，通过 timespec 获取绝对时间的方式，可以完成计算，因此整个 rx callback 函数如下所示：

```C
int rx_callback(hackrf_transfer* transfer){

    // Used to record time consumption
    static struct timespec current_time;
    static struct timespec  last_time;
    static double execution_time;

    //Used to record how many times the callback is used
    static int num_callback = 0;

    // Error in streaming handle:
    if(transfer == NULL || transfer->buffer == NULL){
        fprintf(stderr,"Invalid transfer data \n");
        return -1;
    }

    // Time calculate
    clock_gettime(CLOCK_MONOTONIC,&current_time);
    execution_time = (current_time.tv_sec-last_time.tv_sec)*1000.0+(current_time.tv_nsec - last_time.tv_nsec)/1000000.0;
    num_callback++;
    last_time = current_time;

    // Print Output
    printf("The %i transfer Buffer length is: %i\n",num_callback,transfer->valid_length);
    printf("The execution time (ms) is: %f\n",execution_time);

    return 0;
}
```

## Main 函数

最后，我们在 main 函数中通过 hackrf_start_rx() 的方式启动 RX 流程，并将上面定义的 rx_callback 函数写入 hackrf_start_rx 的 callback 中。接下来我们设置 sleep(5）让程序休眠 5s，在这个期间当 RX Buffer 填满之后就会调用 callback 函数将每次有效的 Buffer 长度和 callback 调用频率打印出来。最后在休眠 5s 后通过 hackrf_stop_rx() 关闭 hackrf 接收，并通过 hackrf_close() 关闭 hackrf 设备。

整个 Main 函数如下所示：

```C
int main(int argc, char** argv){

	// Parameters
	void* rf_dev;
	int result = HACKRF_SUCCESS;

	// Initiate HackRF
	hackrf_init_and_checkID(&rf_dev);

    // Config HackRF RX
    hackrf_rx_config(rf_dev,FREQ,SAMPLE_RATE,VGA_GAIN,LNA_GAIN);

    // Start HackRF Receive
    hackrf_start_rx(rf_dev,rx_callback,NULL);

	// Execute RX for 5s
    sleep(5);

    // Stop HackRF Rx
    result = hackrf_stop_rx(rf_dev);
    if(result != HACKRF_SUCCESS){
        fprintf(stderr,"Hackrf STOP RX Fail \n");
    }

	// Close HackRF
	result = hackrf_close(rf_dev);
	if(result != HACKRF_SUCCESS){
		fprintf(stderr,"Hackrf not closed\n");
	}
	else{
		printf("Hackrf closed \n");
	}
}
```

## 输出

当我们设置采样率为 8MHz 时，从下图可以看到在 5s 内调用了 305次 callback 函数，也就是 16.39ms 每次。同时我们从记录的 time interval 可以看到实时每一次 callback 与上一次的时间间隔，大约也是 16.3ms 附近。每次 Buffer 的有效长度都是 262144 个 bytes，也就是有 131072 个 IQ 数据。我们可以计算 8MHz 采样率，每次采 131072 个数，理论需要的时间为 1/8e6*131072 = 16.384ms，因此所有的数据都能够一一对应。注意实际每次 callback 的调用频率从下图中可以看到并不是完全一样的，这可能是与 USB 传输延迟有关。

![图片](https://github.com/user-attachments/assets/5c9d8bd0-2e9d-4ad9-9acf-6a5df41354c3)


如果把采样调整为 20MHz，可以看到 5s 内调用了 762次 callback，每次实测的 callback 时间间隔大约是 6.6ms，与理论计算的 6.55ms 对齐。

![图片](https://github.com/user-attachments/assets/85b18239-8f5f-4ec1-80fb-fcb63a735f24)











