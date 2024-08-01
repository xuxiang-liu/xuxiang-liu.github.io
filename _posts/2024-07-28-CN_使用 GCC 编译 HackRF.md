---
layout: post
title: 使用 GCC 编译 HackRF
date: 2022-04-26 23:18 +0800
author: Xuxiang Liu
tags: [SDR,CN]
---

最近在 Linux 下终于弄明白了如何用 GCC 来编译 HackRF 环境了，终于可以不依赖于复杂的 CMake 脚本了。

简单的 Hackrf C 语言 Demo 如下：

“
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
“

这个简单的 C demo 实现了 hackrf 初始化和 ID、firmware version 读取
