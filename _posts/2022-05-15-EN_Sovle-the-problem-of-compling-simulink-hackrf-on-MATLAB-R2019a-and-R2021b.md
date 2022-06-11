---
layout: post
title: Sovle the problem of compling simulink-hackrf on MATLAB R2019a and R2021b
date: 2022-05-15 23:18 +0800
tags: [SDR]
---

Complie the simulink-hackrf in higher Matlab version.
{: .message }
   
In my previous blog i mentioned that in higher Matlab version (>R2016a) there is a problem about compling simulink-hackrf using _mex_. After reading this reply <https://github.com/kit-cel/simulink-hackrf/issues/11> and the help document of _mex_, i find the way of compling the simulink-hackrf on Matlab R2019a and R2021b.
 
I also fork the original _simulink-hackrf_ Github project. The forked project _simulink-hackrf_xuxiang_ <https://github.com/xuxiang-liu/simulink-hackrf_xuxiang> is compatible for all Matlab versions and there is no extra installing or compling steps needed comparing to the original one. You can just follow my blog <https://xuxiang-liu.github.io/2022/04/16/firstpost.html> to process. And in the _step 12_, you only need one _make_ command. I also hope to add more interesting features in this forked project.
 
The following records how to use _mex_ to complie _simulink-hackrf_ on higher Matlab version. If you want to use the original _simulink-hackrf_ project, you could referto this.
   
## Steps
  
### Install MINGW

In higher Matlab version, we need to use Matlab Add-Ons to install the _MINGW_. This is much easier than installing in Matlab R2016a. Just click Adds-Ons and install it automatically

### Generate new libhackrf.lib
 
You could refer to <https://github.com/kit-cel/simulink-hackrf/issues/11> for more information about this step. The environment you need to prepare is the MSVC in VS2019. You could refer to this document for more information https://developercommunity.visualstudio.com/t/all-4-vc-command-prompt-batch-files-eg-vcvars64bat/536112>. When MSVC is installed, open it in this path "...​\M​icrosoft Visual Studio​\2​019​\C​ommunity​\V​C​\A​uxiliary​\B​uild​\v​cvars64.bat".

Then AustrianCodeWriter has given a complete description about how to generate libhackrf.lib based on MSVC environment.

1. Call a new MSVC console (e.g. for "Visual Studio 2019": Start a new command line window and call the batch-script from "C:\Program Files (x86)\Microsoft Visual Studio\2019\Community\VC\Auxiliary\Build\vcvars64.bat")

2. Change directory to "deps\bin" and run:
> dumpbin /EXPORTS libhackrf.dll > libhackrf.exports

3. Create a new file called "libhackrf.def" and insert the following lines:

>  LIBRARY "libhackrf.dll"
   
>  EXPORTS
   
>  hackrf_board_id_name
   
>  hackrf_xxx
   
   For _hackrf_xxx_ insert each "**hackrf_**" element from your previously created
   "libhackrf.exports" file

4. In your MSVC console type the command:
lib /def:libhackrf.def /machine:x64 /out:libhackrf.lib

After doing these steps, it will generate _libhackrf.exp, libhackrf.exports, libhackrf.def_ and _libhackrf.lib_. Copy _libhackrf.lib_ into _simulink-hackrf-1.0/deps/bin_. 
 
### Change _make_

Open the _make_ file in _Simulink-hackrf_. Then find _%% Cofiguration_, change _['-L' HACKRF_LIB_DIR]; '-lhackrf' ..._ to _['-L' HACKRF_LIB_DIR]; '-l​**lib**​hackrf'..._. Then run _make_. 

Now everything is fine. You should be able to successfully complie it.
