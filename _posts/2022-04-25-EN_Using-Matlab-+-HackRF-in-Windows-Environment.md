---
layout: post
title: Using Matlab + HackRF in Windows Environment
date: 2022-04-25 23:18 +0800
author: Xuxiang Liu
tags: [SDR,EN]
---

Use Hackrf in Matlab under Windows Environemnt.
{: .message }

Hackrf is always the best tool for playing or learning the SDR in my opinion. The common way of using it is in Linux environment, together with Octave for data processing. A much easier way of using it is just listening to the radio in Windows environment through all kinds of SDR softwares like SDRsharp. However, i really would like to find a way to use the hackrf more sophisticatedly in Windows environment. Luckily, there is a very good project on the github to support the hackrf development in Windows using Matlab & Simulink. There is also a video on YouTube describing how to install/complie this project <https://www.youtube.com/watch?v=7dtikuo3BSw>

But it seeme there is no longer any maintaince on this project, the issues about the installation or complie on different versions of dependencies or Matlab have not been answered anymore. So i spent a weekend to figure out how to correctly install it follwoing the video on the Youtube. After lots of failed attempts, i finally made it work. I record the whole procedures here and hope it could help those who are interested in. **Hope u are enjoying**

This project has very strong requirements on the version of the supporting softwares and dependencies. **Even the Matlab version has to be R2016a**. However, this Matlab version requirement is actually requiring the version of the MinGW64 adds on of the Matlab. **When installation is done, we could switch back to a higher Matlab version to use it. And the speed of processing, from what i have seen, is actually better with the Matlab R2019a. It will not give OOOOO error report when using R2019a.**

## Installation Steps

### 1. Install drivers for HackRF
 
1.1 This step will not be discussed here. Once you have used SDRsharp in Windows, you must have installed this driver. Just download the _Zadig_ then everthing is done.

### 2. Install Matlab R2016a
 
2.1 This step will not be discusses there. I have failed to use Matlab R2019a for installing this project. The root cause is the version of the Mingw complier of the Matlab. It seems the _tdm64-gcc-4.9.2_ is the only version which can successfully _make_ this project in Matlab. I saw there are several issues on the installation using different versions of Matlab, so **I strongly suggest you to use R2016a for installation. You can switch to a higer version to use it after installation**. You are very welcome to debug why the _make_ procedure will give errors when using higher version of Mingw (higher version of Matlab). Thanks in advance.

### 3. Install MinGW64 (tdm64-gcc-4.9.2) in Matlab
 
3.1 The current Matlab R2016a adds-on manager does not support automatically install MinGW64. You need to manually install it. Firstly we need to download the _tdm64-gcc-4.9.2_ from here <https://sourceforge.net/projects/tdm-gcc/files/TDM-GCC%20Installer/Previous/1.1309.0/>. Then we run the _.exe_ file. 【**Do not click the _Check for updates files_ box**】After installation, add the path to system environment. The _Name_ is WM_MINGW64_LOC and the _Value_ is D:​\T​DM-GCC-64 (Your own path). Finally execute _mex-setup_ in Matlab and you should see following figure

 ​![​TDM-GCC-64​](https://user-images.githubusercontent.com/40487487/163658277-4e00672f-fa99-4998-b9ce-cbe1bdb0c006.PNG)

### 4. Install MinGW64-posix-6.4.0
 
4.1 There are many blogs talking about how to install this, so we are not going to repeat it. The only thing you need to notice is that only the _posix_ version is supported. I haven't tried different version numbers, you are welcome to have a try and leave a message to me. If the online installation is not available for you, the offline installation is also ok. 
 
 ![​gcc​](https://user-images.githubusercontent.com/40487487/163658046-4d176592-6d96-4d87-ad72-70ecb60dc815.PNG) 

### 5 Install cmake 3.6.2

5.1 The most important thing here is to use the correct version of the cmake. I haven't tried another versions but in one blog it mentions to use the exactly version 3.6. So lets just follow it. It might be difficult for you to find this old version, i put all necessary softwares and dependencies in the end of this blog. You could also find it in this link <https://download.csdn.net/download/icbm/9646285>

### 6. Install cygwin64

6.1 There are lots of blogs on this topic, the only problem i meet is that I could not find the online mirror source on one of my computer. So i have to use offline installation. In case you have the same problem, i leave the link of the offline installation package in the end of this blog. You only need to install the basic cygwin64.

### 7. Download libusb

7.1 You can download the libusb directly on github <https://github.com/libusb/libusb/releases> . There is no requirement for the version. Just download and unzip it.

7.2 Copy the _libusb.h_ from D:​\l​ibusb​\i​nclude​\l​ibusb-1.0 (Change to your own path) to the _include_ path of the complier, which is C:​\P​rogram Files​\m​ingw64​\x​86_64-w64-mingw32​\i​nclude (Change to your own path)

### 8. Download libhackrf

8.1 You can download the libhackrf directly from github <https://github.com/greatscottgadgets/hackrf>. We only need the _libhackrf_ in _hackrf-master -> host_. Just download and unzip it.

8.2 Copy the whole _libusb_ folder in previous step to the _libhackrf_ folder.

### 9. Download simulink-hackrf-1.0

9.1 You can download it directly from github  <https://github.com/kit-cel/simulink-hackrf/releases/tag/v1.0>. Just download and unzip it. Then create a folder named _deps_ in it.

### 10. Run cmake

10.1 Open _cmake_gui.exe_ in your cmake installation path.

10.2 Choose _D:/libhacrf/src (Change it to your own path)_ for _where is the source code_. Choose _D:/libhackrf/build (Change it to your own path)_ for _where to build the binaries_

10.3 Click _Add Entrt_, and **manually add those entries as the figure shown below**. If the _Value_ is a path, select _PATH_ for the Type, if the _Value_ is a file, select _FILEPATH_ for the Type, if the _Value_ is a box, select _BOOL_ for the Type and if the _Value_ is empty, select _PATH_ or _STRING_.

![​cmake​](https://user-images.githubusercontent.com/40487487/163658079-9885a4d8-a39c-4ef0-9862-7301b007bc42.png) 

10.4 Click _Configure_ and choose _MINGW_ for complier. It will show _Configure done_ if it is successfully executed.

10.5 Click _Generate_. It will show _Generate done_ if it is successfully executed, as shown in the figure of step 10.3

### 11. Make libhackrf in cygwin64

11.1 Open the cygwin terminal and _cd_ into the _libhackrf/build_. Then execute _mingw32-make_

11.2 Execute _mingw32-make install_ in the same path. The figure below shows the resutls after these two steps. You can also find there are two folders named _bin_ and _include_ created in _simulink-hackrf-1.0/deps_

![​make​](https://user-images.githubusercontent.com/40487487/163658115-cd38b392-f7b1-48d1-bb92-a06ee30bc37e.png) 

### 12. make in Matlab

12.1 Copy the hackrf.h from D:​\s​imulink-hackrf-1.0​\d​eps​\i​nclude​\P​roject (Changeb to your own path) to D:​\s​imulink-hackrf-1.0​\s​rc (Change to your own path)

12.2 Open Matlab R2016a and select the _simulink-hackrf-1.0_ folder, run _make_ command

12.3 Ideally it should successfully _make_. However, there might be an error as the figure shown below. To solve this, you could create an empty _libhackrd.lib_ (right click -> create new .txt file, then change _.txt_ to _.lib_) in D:​\s​imulink-hackrf-1.0​\d​eps​\b​in (Change to your own path). Then copy _libhackrf.dll_ from the same path to D:​\s​imulink-hackrf-1.0​\b​uild (Change to your own path). Finally add the _build_ path into Matlab and _make_ again.

![​error​](https://user-images.githubusercontent.com/40487487/163658158-a9d29fbf-28f2-47ff-9870-c2719b22644d.PNG) 

12.4 Now if you see the figure shown as below, the _make_ is successful.

 ​![​success​](https://user-images.githubusercontent.com/40487487/163658215-c92a94d3-509c-48e1-9269-048fe1f41052.PNG)

### 13. Run Demo

13.1 Let's have a look at the demo. **Notice we need to add _demos_ and _build_ into the Matlab path**. Plug in the HackRF and execute _hackrf_find_devices.m_ to.fins the device. Then open the _scope_demo_ in _demos_. After correctlt setting the _sampling rate_, _gain_, and _center frequency_, we could see the spectrum in Simulink. The figure below is the comparsion between spectrum from Airspy and Simulink. 

![​demo​](https://user-images.githubusercontent.com/40487487/163658230-0ad57941-9047-4329-81e8-d6b052d0bbff.png)

13.2 BTW, after _make_ in Matlab, we could use a higher version Maltab to run. Just add the _demos_ and _build_ into the Matlab path. 

## Link for installation package

Here I put all necessary softwares and dependencies in the link below (Except Matlab of course...).

transfer link: <https://cowtransfer.com/s/2c0178772e934a> or access [ cowtransfer.com ] and input extract code: 066ef3 to download; 

 ​---- The End ---
