---
layout: post
title: HackRF 频谱仪之 Pthread 用法简介
author: Xuxiang Liu
date: 2024-08-22 23:18 +0800
tags: [SDR,CN]
---

HackRF 频谱仪 - Pthread 的使用
{: .message }

制作一个基于 HackRF 的频谱仪之前，先储备一些关于 Pthread 和 SDL 的基本知识，这些应该都会用于频谱仪的 Demo 中。

# Pthread

一开始，我以为可以直接在 hackrf 的 callback 中直接调用 SDL 进行绘图（关于 SDL 的内容在下一篇博文中介绍），但是很快发现这个方式有一个缺陷，即由于 SDL 绘图与 callback 之间存在时间差，因此两个操作很难同步，会导致 SDL 绘图出现卡顿。因此后来考虑通过线程的方式进行控制。下面以自己的理解，通过简单的 C demo 来说明在我们的 Hackrf 频谱仪中会用到的线程操作：

## 构建一个多线程的 demo

要使用线程，首先需要定一个线程标识(pthread identifier), 接下来我们定义两个线程标识分别是 producer 和 consumer 线程：

```C
static pthread_t produceer_pthread;
static pthread_t consumer_pthread;
```

接下来，我们创建两个线程中的操作，在 producer 线程中，我们每隔 1s 中进行 20次 连续的对一个变量 a 的 +1 操作，并且打印出来；在 consumer 线程中，我们每隔 1s 对同一个变量 a 进行 -1 操作，并打印出来：

```C
void* Producer_Pthread(){
    while(keep_running){
        for(int i=0;i<20;i++){
            a++;
            printf("In Producer Thread, a is: %i\n",a);
        }
        sleep(1);
    }
    return 0;
}

void* Consumer_Pthread(){
    while(keep_running){
        a--;
        printf("In Consumer Thread, a is: %i\n",a);
        sleep(1);
    }
    return 0;
}
```

之后，我们在 main 函数中将两个线程启动, pthread_create 函数第一个参数是我们之前定义的线程标识的地址，第三个参数则是我们的线程函数，第二和第四个参数暂时没有用到。

```C
pthread_create(&produceer_pthread,NULL,Producer_Pthread,NULL);
pthread_create(&consumer_pthread,NULL,Consumer_Pthread,NULL);
```

最后，我们通过一个简单的 SDL 事件方式（关于 SDL 的内容在下一篇博文中介绍）让主函数保持循环，直到我们关闭 SDL Window

```C
SDL_Init(SDL_INIT_EVERYTHING);
SDL_Window *window = SDL_CreateWindow("Test",SDL_WINDOWPOS_UNDEFINED,SDL_WINDOWPOS_UNDEFINED,400,400,SDL_WINDOW_SHOWN);
SDL_Event event;
while(keep_running){
	while(SDL_PollEvent(&event)){
		if(event.type == SDL_QUIT){
			keep_running = 0;
        }
    }
}
```

这样我们来看一下两个进程的输出: 可以看到，由于两个线程都会去修改到变量 a，因此输出中会看见 consumer 线程穿插在了 producer 的线程中，因此 producer 的 20次 连续输出被打断了。

![图片](https://github.com/user-attachments/assets/71c3b508-c39b-4094-952d-51cdb85323b2)


为了避免这种问题的出现，我们在通过线程锁(pthread_mutex) 来解决。
