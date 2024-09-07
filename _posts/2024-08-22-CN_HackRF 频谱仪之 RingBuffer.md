---
layout: post
title: HackRF 频谱仪之 RingBuffer.
author: Xuxiang Liu
date: 2024-09-07 23:18 +0800
tags: [SDR,CN]
---

HackRF 频谱仪 - RingBuffer 的使用
{: .message }

之前我们看了如何通过 Pthread 的方式对多个线程进行控制，可以用于 HackRF 频谱仪中的数据采集和绘制频谱两个进程的控制。下面我们看一下如何实现这两个进程中的数据传递。我们通过一个 RingBuffer 的形式来进行，RingBuffer 可以用于两个异步的进程间有效的存储与读取数据，其原理示意图如下所示：

![图片](https://github.com/user-attachments/assets/a4fee66a-f595-4b6d-aed4-b2e6d32135eb)

可以看到，RingBuffer 这个结构体中以下参数：

1. data_buffer: 这是用来存储数据 Buffer 本身，是一个自定义 BufferSize 长度的数组

2. head: 这是用来指示写入 buffer 位置的“指针”

3. tail: 这是用来指示读取 buffer 位置的“指针”

4. count: 这是用来指示 buffer 中有多少 data 待读取

5. pthread_mutex: 这是用来为读写 buffer 过程上锁，避免 buffer 的读写操作相互影响

6. pthread_cond: 这是用来实现读 buffer 等待功能，待 buffer 中有数据后再进行读取

对于 pthrad_mutex 与 pthread_cond 的使用和上两篇博文类似，只是在 RingBuffer 这个结构体中进行了实例化，这样当需要多个 RingBuffer 时，就可以以 RingBuffer 为单位去控制这个指定 RingBuffer 的读写线程了。

下面我们首先来定义这个 RingBuffer 的结构体，其代码很简单，即定义了上述的这些参数，如下所示:

```C
typedef struct {
    int buffer_No;
    int data[BUFFER_SIZE];
    int head;
    int tail;
    int count;
    pthread_mutex_t ring_mutex;
    pthread_cond_t ring_cond;
}RING_BUFFER;
```

然后我们来实现 RingBuffer 的数据写入功能，这个函数有三个传入的参数，分别是 RingBuffer 的指针、要写入的 data 的数量 和 要写入的 data 的指针。我们首先对整个写入 Buffer 的过程加锁，然后我们一个一个 data 进行处理：

Step 1. 首先将 data 写入到 RingBuffer 中 head 指针所指的位置（Add data）

Step 2. 然后将 head 指针 + 1，注意由于是一个循环 Buffer，当超过 BUFFER SIZE 之后我们要重新回到 0 的位置，因此用取余的方法实现 head 指针的递增 (Add header)

Step 3. 将可读取的 data 数量（count）+ 1，注意这里 count 最大就是 BUFFER SIZE (Count++)

Step 4. 当 count = BUFFER SIZE 时，即整个 BUFFER 都被写满了，需要继续写入 Buffer 就需要占据未读取的 BUFFER 的空间，因此需要将 tail 指针 + 1, 同样的，当超过 BUFFER SIZE 后需要重新回到 0 的位置，因此我们用取余的方法进行 (Deal with Tail)

Step 5. 最后当写入操作完成后，释放 cond 信号，这里用于告知读取进程此时 Buffer 中已经有可读的数据了，因此若读取进程处于等待过程时，就可以退出等待了。

最后一步是解锁 Buffer，这样就完成了 RingBuffer 的整个数据写入功能。

```C
void ring_buffer_push(RING_BUFFER *ringBuffer, int data_num, int *data){
    pthread_mutex_lock(&ringBuffer->ring_mutex);
    // Process the data in buffer one by one
    for (int i=0;i<data_num;i++){
        int previous_head = ringBuffer->head;
        // Add data
        ringBuffer->data[ringBuffer->head] = data[i];
        // Add Header
        ringBuffer->head = ((ringBuffer->head) + 1) % BUFFER_SIZE;
        // Count ++, if not full
        if(ringBuffer->count < BUFFER_SIZE){
            ringBuffer->count = ringBuffer->count+1;
        }
        // Deal with Tail when BUFFER is FULL. Keep counts the same
        else{
            ringBuffer->tail = (ringBuffer->tail+1)%BUFFER_SIZE;
        }
        printf("Ring Buffer No. %i Push-In, head = %i, tail = %i, inputData = %i, DataToRead = %i \n",ringBuffer->buffer_No,ringBuffer->head,ringBuffer->tail, ringBuffer->data[previous_head],ringBuffer->count);
    }
    pthread_cond_signal(&ringBuffer->ring_cond);
    pthread_mutex_unlock(&ringBuffer->ring_mutex);
}
```

然后我们再来实现从 Buffer 中读取的功能：这个函数同样有三个参数，分别是 RingBuffer 的指针、要读取的 data 的数量 和 要存储读出 data 的指针。同样我们首先对整个读取 Buffer 的过程加锁，然后我们一个一个 data 进行处理：

Step 1. 判断读取这个 data 时，Buffer 中是否还有可以读的 data，也就是检查 count 是否为 0，若为 0，则通过 pthread_cond_wait 阻碍当前读取的线程，并进行等待；否则就继续读取

Step 2. 从 Tail 指针位置开始读取 data (Read out 1 data)

Step 3. Tail 指针加 1, 同样的当超过 BUFFER SIZE 时回到 0 的位置，因此用取余的方法实现 （Move Tail)

Step 4. 可读取的 Data 数量，也就是 count 减 1 (Count--)

最后一步是解锁 Buffer，这样就完成了 RingBuffer 的整个数据读取功能。

```C
void ring_buffer_pop(RING_BUFFER *ringBuffer, int data_num, int *data){
    pthread_mutex_lock(&ringBuffer->ring_mutex);
    for(int i=0;i<data_num;i++){
        while(ringBuffer->count == 0){
            pthread_cond_wait(&ringBuffer->ring_cond,&ringBuffer->ring_mutex);
        }
        int tail_previous = ringBuffer->tail;
        // Read out 1 data
        data[i] = ringBuffer->data[ringBuffer->tail];
        // Move the tail
        ringBuffer->tail = (ringBuffer->tail+1)%BUFFER_SIZE;
        // Count--
        ringBuffer->count = ringBuffer->count-1;
        printf("Ring Buffer No. %i Pop Out head = %i, tail = %i, outData = %i, DataToRead = %i \n",ringBuffer->buffer_No,ringBuffer->head,ringBuffer->tail, ringBuffer->data[tail_previous],ringBuffer->count);
    }
    pthread_mutex_unlock(&ringBuffer->ring_mutex);
}
```

现在，我们对之前的 Producer 和 Consumer 两个线程进行改动，在 Producer 中往 RingBuffer 写入数据，在 Consumer 中从 RingBuffer 读取数据，如下所示

```C
void* Producer_Pthread() {
    while (keep_running) {
        for (int i = 0; i < WRITE_NUM; i++) {
            a[i] = a[i] + i;
        }
        ring_buffer_push(&ringbuffer, WRITE_NUM, a);
        sleep(1);
    }
    return 0;
}
```

```C
void* Consumer_Pthread(){
    while(keep_running){
        ring_buffer_pop(&ringbuffer,READ_NUM,readout);
        for(int i=0;i<READ_NUM;i++){
            printf("%i\t ",readout[i]);
        }
        printf("\n");
        sleep(1);
    }
    return 0;
}
```

接下来，我们看一下如果写入比读取慢的情况，定义 WRITE_NUM = 20, READ_NUM = 10，但 Consumer 的 sleep 时间改为 0.6s，结果如下所示。可以看到当 count = 0 之后，Consumer 进程会暂停，待 Producer 重新填入数据后再次开始读取。

![图片](https://github.com/user-attachments/assets/26100557-4c90-47f1-8651-1aac5a04d145)


另外，我们再看一下若写入比读取更快的情况，定义 WRITE_NUM = 20, READ_NUM = 10, 同时 Consumer 与 Producer 的 sleep 时间均为 1s，结果如下所示。可以看到当 head = tail 的时候，下一次对 Buffer 写入的时候，tail 的值也会加 1。而到读取的时候，tail 的值为 0,也就是从 Buffer 中第一个数进行读取，读出来的数是 30,那么我们可以从更早的结果看到（未在截图中），Buffer 中位置 0 最近一次写入的数就是 30.

![图片](https://github.com/user-attachments/assets/4f3c551c-98d0-49fc-be46-65d0d1bf49cc)

## IMPORTANT !

最后我们还需要额外注意的是：在 Producer 和 Consumer 中，调用 ringbuffer_push 或 ringbuffer_pop 时都是传入 ringbuffer 的地址！
