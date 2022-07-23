---
layout: post
title: Simple guide to understand the Simulink-hackrf source code
author: Xuxiang Liu
date: 2022-07-23 22:40 +0800
tags: [SDR,EN]
---

A simple guide to understand the S-func in simulink, mask and source code of simulink hackrf
{: .message }

After trying the _simulink-hackrf_, I get very interested in how it is realized. I found it is using the _S-func_ block in Simulink to call those functions in _hackrf.c_ and control the HackRF board. So I think this is a good opportunity to learn something about the _S-func_ in Matlab and also the _hackrf.c_. However, the true understanding of the _simulink-hackrf_ source code is based on the full understanding of those functions in _hackrf.c_. We will remain this part for later. You could regard this blog as an entrance for studying the source code of _simulink-hackrf_ and _hackrf.c_

The main contents of this blog include:

1. An introduction to the source code of _simulink-hackrf_ and how it is realized

2. An introduction to the S-func block in Simulink, based on one model (_hackrf_source_) in _simulink-hackrf_

3. Change the mask of the _hackrf_source_ in Simulink for a new GUI

4. Change the code in _hackrf_source_ to achieve more functions

## An introduction to the source code of _simulink-hackrf_ and how it is realized

The key of _simulink-hackrf_ is using the S-func to call functions in the _hackrf.c_. It has three important source codes which are _hackrf_find_devices.c_, _hackrf_source.c_ and _hackrf_sink.c_. They are in the path _simulink-hackrf/src/_. They are all written in C and executed as _mex_ _(MATLAB_ _Executable)_.

Finally, these files are complied using _simulink-hackrf/make.m_ and generate corresponding _.mexw64_ file. Then they are called by _S-func_ in Simulink. The _hackrf_find_devices.c_ is used to connect the HackRF device, _hackrf_source.c_ is used for receiving and _hackrf_sink.c_ is used for transmitting.

## An introduction to the S-func block in Simulink

_S-func_ is a very important tool in _Simulink_. It allows the user to create self-defined function in C, C# and etc. There are lots of blogs introducing the S-func. Here I will introduce the S-func based on _hackrf_source.c_. If you are interested, you could also refer to <https://blog.smileland.me/2020/02/12/%E4%BD%BF%E7%94%A8C%E8%AF%AD%E8%A8%80%E5%86%99%E7%AE%80%E5%8D%95S-Function/> to write a simple S-func by yourself. This will help you understand the S-func more thoroughly.

Using _hackrf_source.c_ as an example, a _S-func_ is constructed as shown in the figure below: You could also refer to this manual https://ww2.mathworks.cn/help/simulink/sfg/how-the-simulink-engine-interacts-with-c-s-functions.html> to know more about the workflow of Simulink Engine.

### mdlInitializeSizes

_mdlInitializeSizes_ is used at the beginning of the S-func. It is used for setting input and output port, sampling time, port type and etc. You could refer to <https://ww2.mathworks.cn/help/simulink/sfg/mdlinitializesizes.html> for more information.

### mdlStarts

_mdlStart_ is executed after _mdlInitializeSizes_. It will call _startHackRF_ function to turn on the recevier of HackRF. You could refer to <https://ww2.mathworks.cn/help/simulink/sfg/mdlstart.html> for more information.

### mdlOutputs

_mdlOutput_ is executed after each step. It will calculate the output of the S-func in current time step. The detailed output format is defined in _mdlInitializeSizes_. And you could refer to <https://ww2.mathworks.cn/help/simulink/sfg/mdloutputs.html> for more information.

### mdlSimStatusChange

_mdlSimStatusChange_ will be called when _Pause_ or _Continue_ button is pressed. It will execute _stopHackRF_ when _Pause_ is pressed, and execute _startHackRF_ when _Continue_ is pressed. You could refer to <https://ww2.mathworks.cn/help/simulink/sfg/mdlsimstatuschange.html> for more information.

### mdlTerminate

_mdlTerminate_ is called when simulation is stop. It will execute the _stopHackRF_. You could refer to <https://ww2.mathworks.cn/help/simulink/sfg/mdlterminate.html> for more information

### mdlCheckParameters & mdlProcessParameters

_mdlCheckParameters_ will be called when input parameter is changed. The change of parameter will happen at the begining or each time step of the simulation. _mdlCheckParameters_ will chekc whether or not each input parameter is satisfied the requirement. _mdlProcessParameters_ will be executed after _mdlCheckParameters_ verify the input parameters. It is used for processing the new input parameter. In _hackrf_source.c_, it is mainly call _Hackrf_Set_Param_ function to write the new parameter into the HackRF. You could refer to <https://ww2.mathworks.cn/help/simulink/sfg/mdlcheckparameters.html> and <https://ww2.mathworks.cn/help/simulink/sfg/mdlprocessparameters.html> for more information.

This is the all for introducing the S-func of _hacrf_source.c_. Acutally if you look at the S-func construcntion figure, it will be quite straightfoward. The whole code is not difficult to understand except those relats to _pthread_mutex_ in _mdlOutputs_ block. But we don't need to care too much about these codes, just know that they are used for generating a reliable output. 

## Change the UI screen of hackrf_source.c in simulink-hackrf 
  
After we realize the S-func using C languages, we could add a mask to make a simple UI. This UI can be used for accepting input and processing the changes in the input. There are two mask files for _hackrf_source.c_ and _hackrf_sink.c_ in _blockset/hackrf_library.slx_, and it will copy this .slx file to _build/_ folder in _simulink-hackrf/make.m_ So lets have a look at how the mask in _blockset/hackrf_library.slx_ corresponds to the _hackrf_source.c_

Double click to open the _HackRF_Source_ block in _hackrf_library.slx_, it is shown as below:
 
 ​![​image​](https://user-images.githubusercontent.com/40487487/170820285-a0a7b9a8-e65e-4030-bc0b-4b127999ba44.png) 
   
You could see that there are several settings including _center frequency, sample rate, bandwidth, output frame length, output data type, gain, lna and vga_. These are actually the same as the parameters mentioned in _hackrf_source.c -> mdlCheckParameters_. We could try to add more input/output ports to realize more self-defined functions. What we have seen now is known as the _mask_, it will transform the input port in .c file into GUI. After we select the block, right click to _Mask -> Edit Mask_ to open the Mask editor, as shown below:
  
 ​![​image​](https://user-images.githubusercontent.com/40487487/170820291-66882826-498e-4e5e-ab7a-fa33484dda1a.png) 
  

Lets see an example of changing the GUI of the  input parameter _center frequency_. Open the tab for _Parameters & Dialog_ and select the _Center Frequency_. Now you could change the type of this parameter from _editor_ to _slider_, ans set the max and min value for it, as shown below. Now we reopen the _Hackrf_Source_ block and we can find the GUI of _center frequency_ has changed to the _slider_. Moreover, we could set more features of this _Center Frequency_ in  _Parameters & Dialog_, for example like _Tunable_, layout and etc. To summarize, the _Mask_ editor us very user friendly.
  
 ​![​image​](https://user-images.githubusercontent.com/40487487/170820329-cc8efe61-b74f-444e-92df-cbb1c2c9dd95.png) 
  
 We could have a deeper look at the tab _Initialization_, as shown below. **NOTICE**: The parameters here are the one links to the parameters in _mask_ and _hackrf_source.c_. If you look closely, you could find that the name of parameters in _hackrf_source.c_ is different from the name in _Mask_ -> _Parameters&Dialog_. For example, the parameter about center frequency setting in _hackrf_source.c mdlCheckParameters_ function is _FREQUENCY_, but in _mask_ it is called _center frequency_. So how are the connected ? 
  
 ​![​image​](https://user-images.githubusercontent.com/40487487/170820546-83fbcfbe-cb02-48d3-a3df-cfd0aef14e38.png) 
   
 Actually, the first step of _mex_ complie from _hackrf_source.c_ to _S func_ is through the _Block Parameters_ tab, as shown below
  
 ​![​image​](https://user-images.githubusercontent.com/40487487/170820345-d5076575-35ee-4b02-b864-6152d8f6c3f7.png) 
 
 We could use right click to open this _Block Parameters_ tab (Or we will automatically start from this tab if we create a new _S-func_). The _S-func_ name corresponds to the name of _hackrf_source.c_ file. And parameters in _S-func_ are corresponds to the input parameters in _mdlCheckParameters_ func. For example, the second item _hackrf_frequency_rounded_ is corresponding to the second item _FREQUENCY_ in _mdlCheckParameters_.
 
 The second step is to relate the variable in _S-func_ parameters with the variable in _Mask_. For example, we could see there we use:
 
 ​>​ hackrf_frequency_rounded = round(hackrf_frequency);  
 
 This is to take the rounded value from _hackrf_frequency_ in _Mask_ and give it to _hackrf_frequency_rounded_, which is _FREQUENCY_ in _hackrf_source.c_. In this way, we create the link between parameters in _MASK_ and .c file. The benefits of doing this is significant. For example, the required center frequency is _uint64t_ in _hackrf.c_, but in _MASK_ UI we could type in any frequency in float type. This will make no mistake. 
 
## Change simulink-hackrf source.c to realize more functions 
 
Finally, lets try add some new input/output in _hackrf_source.c_ to realize some new features. The original purpose is to add a software switch to turn on the _biasTee_ in hackrf. But I find I always measure a 2.7V voltage at the antenna port whenever I turn on the biasTee or not. I also try to use software like _SDRange_ or _Airspy_ to turn on this function, but the result is the same. So maybe the _biasTee_ function on my board is damaged. So I can only add a non relative feature in the _hacrf_source.c_ code, just for the purpose to introduce the method. If you are interested, you could add some new features which are really related to the hackrf. 
  
My new feature is very boring. It is just read a input, and when this value is higher than 10, the output will be 1, otherwise it will be 0.

### Step 1.
   
Firstly, we need to add a input, which is named _ANT_PORT_. We add following code in _mdlInitializeSizes_. Here the _SS_PRM_SIM_ONLY_TUNABLE_ setting indicates that this input is allowed to change during running.
  
​>​ ssSetSFcnParamTunable(S, ANT_PORT,  SS_PRM_SIM_ONLY_TUNABLE)； 
 
Secondly, we add code to check the type of the input to make sure it is num.
  
>​ Assert_is_numeric(S, ANT_PORT); 
 
### Step 2.
 
Lets add a output. We make the following changes in _mdlInitializeSizes_: the code _ssSetNumOutputPorts(S,2)_ sets the number of output ports to 2.
  
>​ if (!ssSetNumInputPorts(S, 0) || !ssSetNumOutputPorts(S, 2)) return; 

Then we set the bit width to 1:
  
>​ ssSetOutputPortWidth(S, 1, 1); 
  
###​ ​Step 3. 
  
Finally add the functions in _mdlOutputs_ to achieve what we want: The function _ssGetOutputPortSignal_ get the value of _Output Port_ and we could manipulate on it.
  
> real_T       *y2 = ssGetOutputPortSignal(S,1);
> 
> if(GetParam(ANT_PORT)>10){
> 
>    y2[0] = GetParam(ANT_PORT)+1;
>    
> }
> 
> else{
> 
>    y2[0] = 0;
>    
> }
  
 ###​ ​Step 4. 
  
Finally we save our new code to _Mine_hackrf_source.c_ and _mex_ complie it. After it is successfully complied, we get _Mine_hackrf_source.mexw64_.
 
Then we create a new _S-func_ in simulink. In _Block Parameters -> S-function name_ we fill in _hackrf_source_v2_. And we fill in the name of the nine input parameters in _Block Parameters -> S-function parameters_. Then we can find the symbol of this _S-func_ changes from one input and output to two outputs, as shown in the figure below. Then we could define the corresponding _Mask_ for it.
  
![​image​](https://user-images.githubusercontent.com/40487487/170820388-0b2afc98-92b5-45ec-9446-340c31100391.png) 
 
**NOTICE**, becuase we define the type of all parameters should be numeric in _mdlCheckParameters_, and we also define the _frame_size_ should pow of 2. So we need to follow these rules here to fill in the _Block Parameters -> S-function parameters_, for example, we could fill in 1，2，3，4，5，6，8，7，9. Then it can successfully link the _S-func_ to _hackrf_source_v2_. After we create the link, we can chenge the _Block Parameters -> S-function parameters_ to a more easy understand name. And we could define these parameters in _mask_editor_.

The final output is shown as below. We see that the hackrf is receiving and another output port is changing according to the value of _ANT_PORT_. 
  
![​mask4-R2019a​](https://user-images.githubusercontent.com/40487487/170820437-d2101b71-6abf-411b-87d2-d56aeab97951.PNG)
