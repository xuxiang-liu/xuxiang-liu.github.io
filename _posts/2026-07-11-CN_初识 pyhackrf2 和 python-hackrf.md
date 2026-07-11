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





