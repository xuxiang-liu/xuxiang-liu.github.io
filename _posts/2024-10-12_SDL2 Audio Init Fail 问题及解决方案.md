---
layout: post
title: SDL2 Audio Init Fail 问题及解决方案
author: Xuxiang Liu
date: 2024-10-12 23:16 +0800
tags: [SDR,CN]
---

记录 Ubuntu SDL2 模块初始化 Aduio 失败的解决方案
{: .message }

在准备用 SDL 做音频驱动的时候，发现 SDL_INIT 音频模块会报错：SDL_Init Fail: dsp: No such audio device，同时也无法找到设备中的任何音频设备。在 Google 了一番后，发现不少人遇到类似问题，因此在这里记录下解决方案 (我使用的平台是 Ubuntu Ubuntu 24.04 LTS)：

1. 补充安装缺少的 library：前两个是大部分平台提到的需要补充安装的依赖文件，后面是我一次性都安装上去的依赖库，建议大家可以先安装前两个试试，如果不行也可以像我一样全部安装：
   
1) libasound2-dev

2) libdbus-1-dev

3) libsdl2-mixer-2.0-0

4) libudev-dev

5) libdbus-1-dev

2. 接下来需要 rebuild SDL，进入到下载的 SDL 根目录下，依次执行：

./configure

make

sudo make install

3. 到此应该就可以解决问题了，我们通过下面的 code 来看一下，可以看到已经可以找到设备上的音频源了:


{% highlight C %}
```C
int res = SDL_Init(SDL_INIT_AUDIO);
if(res<0){
  fprintf(stderr,"SDL_Init Fail: %s\n", SDL_GetError());
}

int numAudioDevices = SDL_GetNumAudioDevices(0); // Get the number of available audio devices
printf("Number of audio devices: %d\n", numAudioDevices);

for (int i = 0; i < numAudioDevices; i++) {
  const char *deviceName = SDL_GetAudioDeviceName(i, 0); // Get the name of the audio device
  printf("Audio device %d: %s\n", i, deviceName);
}
```
{% endhighlight %}

![Uploading 图片.png…]()

