---
layout: post
title: 使用 GCC 编译 HackRF
date: 2022-04-26 23:18 +0800
author: Xuxiang Liu
tags: [SDR,CN]
---

最近在 Linux 下终于弄明白了如何用 GCC 来编译 HackRF 环境了，终于可以不依赖于 CMake 脚本了。

简单的 Hackrf C 语言 Demo 如下：

``` C
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <hackrf.h>

// This function initiate hackrf, and checks its connection and ID

int hackrf_init_and_checkID(void **rf_dev){
	
	hackrf_device *dev = NULL;
	int result = HACKRF_SUCCESS;
	
	// Init Hackrf
	result = hackrf_init();
	if(result != HACKRF_SUCCESS){
		printf("HackRF Init Fail \n");
	}
	else{
		printf("HackRF Init Success \n");
	}
	
	// Open Hackrf
	result = hackrf_open(&dev);
	if(result != HACKRF_SUCCESS){
		printf("Hackrf not found \n");
	}
	else{
		printf("Hackrf Found \n");			
	}
	
	// Read ID
	uint8_t board_id;
	result = hackrf_board_id_read(dev,&board_id);
	if(result != HACKRF_SUCCESS){
		printf("Read HackRF Board ID FAIL \n");
	}
	else{
		printf("HackRF Board ID is: %u \n",board_id);			
	}
	
	// Read Board Part ID and Serail Number
	read_partid_serialno_t serialno_partid;
	result = hackrf_board_partid_serialno_read(dev,&serialno_partid);
	if(result != HACKRF_SUCCESS){
		printf("Read Serail ID FAIL \n");
	}
	else{
		printf("HackRF Board Part ID is: 0x%08x 0x%08x\n",serialno_partid.part_id[0],serialno_partid.part_id[1]);			
		printf("HackRF Board Serial Number is: 0x%08x 0x%08x 0x%08x 0x%08x\n",serialno_partid.serial_no[0],serialno_partid.serial_no[1],serialno_partid.serial_no[2],serialno_partid.serial_no[3]);
	}
	
	// Read HACKRF firmware Version
	uint8_t hackrf_version_length = 255;
	char hackrf_version[hackrf_version_length];
	result = hackrf_version_string_read(dev,&hackrf_version[0],hackrf_version_length);
	if(result != HACKRF_SUCCESS){
		printf("Read Firmware Version FAIL \n");
	}
	else{
		printf("HACKRF_Version is: %s \n",hackrf_version);
	}
	
	
	(*rf_dev) = dev;
	
	return 1;
}

int main(void){
	
	// Parameters
	void* rf_dev;
	int result = HACKRF_SUCCESS;
	
	// Initiate HackRF
	hackrf_init_and_checkID(&rf_dev);
	
	// Close HackRF
	result = hackrf_close(rf_dev);
	if(result != HACKRF_SUCCESS){
		printf("Hackrf not closed\n");
	}
	else{
		printf("Hackrf closed \n");			
	}
	
	
}
```

这个简单的 C demo 实现了 hackrf 初始化和 ID、firmware version 读取，其中 hackrf_init 等函数都是来自于 hackrf.h 中定义。在前面的 blog 中讲到用 cmake file 的编译方法，但是需要创建较为复杂的 cmakelist 文件，而且像 hackrf 的 library 还需要通过一个 FindLIBHACKRF.cmake file 来帮助找到系统中 libhackrf 的位置。

那么现在，我们通过两行简单的 gcc 命令的方式，也可以完成编译：
```
gcc -c -I/usr/local/include/libhackrf HackRF_SA_main.c -o HackRF_SA_main.o

gcc HackRF_SA_main.o -L/usr/local/lib -lhackrf -o HackRF_SA
```

上述 gcc 指令的解释如下：

1. gcc-c -I 用于指定 hackrf.h 的位置。那么一般安装 make hackrf 之后，都会在本地 usr/local/include 中找到一个 libhackrf 文件，hackrf.h 就在这里

2. 然后通过 -o 将 main.c 文件变为可以执行的 main.o 文件

3. 然后，通过 gcc -L 命令将 libhackrf.so 动态库与 main.o 可执行文件进行关联，同样的，安装完成后 libhackrf.so 一般都会放在本地的 usr/local/lib 中，-lhackrf 则会自动增加 lib 和 .so 字符。

4. 最后同样的通过 -o 生成可执行的 HackRF_SA 可执行程序，直接 ./HackRF_SA 即可运行

可以看到上述 GCC 方式更直接与易理解。另外，如果在本地 usr/ 路径下没有找到 hackrf.h 和 libhackrf.so，我们也可以在 hackrf-master/host 和 hackrf-master/host/build 中找到相应的内容。

最后，插上 Hackrf，执行 HackRF_SA 之后，应该就可以看到 HackRF 的信息了～




