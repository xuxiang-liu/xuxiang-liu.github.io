---
layout: post
title: HackRF 频谱仪之 Pthread 用法简介（二）
author: Xuxiang Liu
date: 2024-09-01 23:18 +0800
tags: [SDR,CN]
---

HackRF 频谱仪 - Pthread Cond 的使用
{: .message }

接下来我们看一下如何使用 pthread_cond 这个工具，实现多个异步线程之间的相互等待。

这个功能很有必要，因为在使用 HackRF 的过程中，HackRF 的 RX 往往是一个线程，不断的吐出采样的数据，而我们为了能够最高效的使用这些数据，往往需要另一个线程来读取这些数据。由于 HackRF 采样并回传受限于 USB 的带宽，因此往往 HackRF RX 的数据传输速率并不是固定的。在之前的 Blog 中也可以看到，以 8MHz 的采样率为例，HackRF RX callback 函数的调用频率大约在 16ms 附近，但每次各不相同。

因此，我们很难用固定的 delay 去平衡两个线程之间的相互关系，也就是说必然存在数据产生者（HackRF）与数据消耗这（Plotting）之间的异步。 在后面我们可以看到，我们可以通过 Ring Buffer 的方式来尽可能的解决这种异步问题。但在此之前，我们先解决一个简单的问题，即如何让一个线程等待另一个线程，在满足某些条件下，等待结束并启动运行。

同样的，我们用上一篇 blog 中的例子来进行 Demo 以理解 pthread_cond 的用法：假设 Producer 仍然每次 sleep 1s，而 Consumer 则每次只 sleep 0.6s，直接运行可以发现在第一次 Producer 休眠之后，几乎只有 Consumer 在运行，两个异步的线程完全混乱了，这也是因为 Consumer 线程休眠时间短，会因为快速唤醒而阻止 Producer 的运行，如下图所示：

![图片](https://github.com/user-attachments/assets/7989b6b3-86a0-4637-8866-c2eb91cb341e)

为了避免 Producer 无法执行，我们定义当 Consumer 将 a 的值减到 0 的时候，就停下来等 Producer 生成新的数据。因此可以将 Producer 与 Consumer 代码写为：

```C
void* Producer_Pthread(){pthread_cond_signal(&cond);
    while(keep_running){
        pthread_mutex_lock(&mutex);
        for(int i=0;i<20;i++){
            a = a+1;
            printf("In Producer Thread, a is: %i\n",a);
        }
        pthread_mutex_unlock(&mutex);
        sleep(1);
    }
    return 0;
}
```

```C
void* Consumer_Pthread(){
    while(keep_running){
        pthread_mutex_lock(&mutex);
        while(a<0){
        }
        a=a-1;
        printf("In Consumer Thread, a is: %i\n",a);
        pthread_mutex_unlock(&mutex);
        sleep(0.6);
    }
    return 0;
}
```

这个时候，我们就会发现一个问题，由于 pthread_mutex_lock 加了锁，一旦在 Consumer 进程中进入了 while(a<0) 的循环后，就无法退出了，因为 Producer 进程再也无法修改 a 的值了。因此我们通过 pthread_cond 可以有效的解决这个问题，关于 pthread_cond 的解释可以 Google 到，这里我简单理解为，pthread_cond 会阻止当前进程，并等待条件信号 cond 后释放。我们重点看 pthread_cond 如何使用：

## Pthread_Cond

Step 1. 定义一个 pthread_cond 变量并初始化：

```C
static pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
```

Step 2. 在等待中通过 pthread_cond_wait 函数实现阻止并等待的功能，注意如很多 pthread 文档所描述，pthread_cond 必须配合 mutex 进行使用，因此 Consumer 的代码可以改写为：

```C
void* Consumer_Pthread(){
    while(keep_running){
        pthread_mutex_lock(&mutex);
        while(a<0){
            pthread_cond_wait(&cond,&mutex); // Block the thread and wait for another pthread to signal it !
        }
        a=a-1;
        printf("In Consumer Thread, a is: %i\n",a);
        pthread_mutex_unlock(&mutex);
        sleep(0.6);
    }
    return 0;
}
```

Step 3. 实现 cond 信号释放，因为 pthrad_cond_wait 是等待 cond 的触发，因此这步需要在另一线程，即 Producer 中触发，否则 Consumer 线程将一直陷入在等待中，只剩下 Producer 线程会执行。因此 Producer 的代码可以写为：即我们在每次 Producer 完成新的 20次 a 累加后触发 cond

```C
void* Producer_Pthread(){pthread_cond_signal(&cond);
    while(keep_running){
        pthread_mutex_lock(&mutex);
        for(int i=0;i<20;i++){
            a = a+1;
            printf("In Producer Thread, a is: %i\n",a);
        }
        pthread_cond_signal(&cond);
        pthread_mutex_unlock(&mutex);
        sleep(1);
    }
    return 0;
}
```

这样，最终的结果如下图所示：可以看到，当 Consumer 将 a 值减少到 -1 的时候，就会停下来等待 Producer 代码的执行。而 Producer 代码会一次将 a 累加 20次后进入休眠，接下来就会触发 Consumer 线程退出等待并持续将 a 减小到 -1。两个异步线程就实现了有序的 等待-执行 的过程。反过来，如果 Producer 比 Consumer 生成的更快，那么就需要在 Producer 中使用 pthread_cond 来等待，等待 Consumer 将 a 的值减到一定的时候，再进行 Producer 线程。

![图片](https://github.com/user-attachments/assets/8b9f1133-6ff6-42f4-9d26-741cfe1b4077)

