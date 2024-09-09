---
layout: post
title: HackRF 频谱仪之 RingBuffer(二)
author: Xuxiang Liu
date: 2024-09-08 23:16 +0800
tags: [SDR,CN]
---

HackRF 频谱仪 - 多个 RingBuffer 的使用（传地址还是变量?)
{: .message }

在上一个 RingBuffer 介绍中，我们最后提到在使用 ringbuffer 的 push 和 pop 函数时，都是需要传入 ringbuffer 的地址，而并非变量。其原因是在 C 语言中传入**结构体** 变量时，实际上都是传入的这个变量的副本，因此我们修改的是这个副本，而非变量本身。所以这样就会遇到一个问题，当多次调用这个 ringbuffer 函数时，结构体都被重新 copy 了一份，也就是说这个 ringbuffer 都是全新的，也就失去了 ringbuffer 的作用了。

下面我们以两个例子进行说明，首先我们重新写一个 ringbuffer 的 push 功能，与上一篇 blog 中的区别在于将传递的参数 ringbuffer 的地址改为 ringbuffer 变量，如下所示：

```C
void ring_buffer_push_test(RING_BUFFER ringBuffer, int data_num, int *data){
    pthread_mutex_lock(&ringBuffer.ring_mutex);
    // Process the data in buffer one by one
    for (int i=0;i<data_num;i++){
        int previous_head = ringBuffer.head;
        // Add data
        ringBuffer.data[ringBuffer.head] = data[i];
        // Add Header
        ringBuffer.head = ((ringBuffer.head) + 1) % BUFFER_SIZE;
        // Count ++, if not full
        if(ringBuffer.count < BUFFER_SIZE){
            ringBuffer.count = ringBuffer.count+1;
        }
            // Deal with Tail when BUFFER is FULL. Keep counts the same
        else{
            ringBuffer.tail = (ringBuffer.tail+1)%BUFFER_SIZE;
        }
        printf("Ring Buffer No. %i Push-In, head = %i, tail = %i, inputData = %i, DataToRead = %i \n",ringBuffer.buffer_No,ringBuffer.head,ringBuffer.tail, ringBuffer.data[previous_head],ringBuffer.count);
    }
    pthread_cond_signal(&ringBuffer.ring_cond);
    pthread_mutex_unlock(&ringBuffer.ring_mutex);
}
```
接下来，我们分别创建两个 ringbuffer 的实例，一个的 bufferNo.= 1, 另一个的 = 2.

```C
static RING_BUFFER ringbuffer = {
        .buffer_No = 1,
        .head = 0,
        .tail = 0,
        .count = 0,
        .ring_mutex = PTHREAD_MUTEX_INITIALIZER,
        .ring_cond = PTHREAD_COND_INITIALIZER
};

static RING_BUFFER ringbuffer1 = {
        .buffer_No = 2,
        .head = 0,
        .tail = 0,
        .count = 0,
        .ring_mutex = PTHREAD_MUTEX_INITIALIZER,
        .ring_cond = PTHREAD_COND_INITIALIZER
};
```

然后我们在 Producer 线程中对两个 ringbuffer 分别写入数据，我们先用上一篇 blog 中的方法，也就是传入 ringbuffer 地址的方式进行写入：

```C
void* Producer_Pthread() {
    while (keep_running) {
        for (int i = 0; i < WRITE_NUM; i++) {
            a[i] = a[i] + i;
        }
        // VERY IMPORTANT HERE TO USE THE ADDRESS OF RINGBUFFER !!!
        ring_buffer_push(&ringbuffer, WRITE_NUM, a);
        sleep(1);

        for (int i=0; i< WRITE_NUM1;i++){
            b[i] = b[i]+i;
        }
        ring_buffer_push(&ringbuffer1,WRITE_NUM1,b);
        sleep(1);
    }
    return 0;
}
```

运行后可以看到，当从一个 ringbuffer 切换到另一个 ringbuffer 写入的时候，head，count 都会从上一次结束的位置开始计数，这就是符合我们预期的：

![图片](https://github.com/user-attachments/assets/44d6189f-e353-4008-8dcb-ac52355d65a9)

接下来，我们再看一下用新定义的 push 方式进行写入，即传入的是 ringbuffer 变量：

```C
void* Producer_Pthread() {
    while (keep_running) {
        for (int i = 0; i < WRITE_NUM; i++) {
            a[i] = a[i] + i;
        }
        // VERY IMPORTANT HERE TO USE THE ADDRESS OF RINGBUFFER !!!
        ring_buffer_push_test(ringbuffer, WRITE_NUM, a);
        sleep(1);

        for (int i=0; i< WRITE_NUM1;i++){
            b[i] = b[i]+i;
        }
        ring_buffer_push_test(ringbuffer1,WRITE_NUM1,b);
        sleep(1);
    }
    return 0;
}
```

运行后可以看到输出结果不符合预期，每次从一个 ringbuffer 切到另一个 ringbuffer 写入的时候，head，count 都会从 0 开始，也就失去了 ringbuffer 的意义。**因此这也就是为什么多次调用时，我们需要传入地址，而非变量。**

![图片](https://github.com/user-attachments/assets/710b3237-09a6-406e-b634-5133672bfa6d)

