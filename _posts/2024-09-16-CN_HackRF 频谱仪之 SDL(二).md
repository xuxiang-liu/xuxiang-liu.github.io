---
layout: post
title: HackRF 频谱仪之 SDL(二)
author: Xuxiang Liu
date: 2024-09-16 23:16 +0800
tags: [SDR,CN]
---

HackRF 频谱仪 - 通过 SDL 的矩形和线条，绘制频谱仪 GUI 静态界面
{: .message }

在上一篇 Blog 我们有实现 include SDL 库，下面我们通过 SDL 中的基础图形，矩形和线条来绘制频谱仪的 GUI 的静态界面

## Overview

我们在创建频谱仪可视化时，会用到 Window, Render, Surface, Texture, TTF_Font, 我们会依次介绍这些组件，他们的包含关系如下所示：

Window -> Render -> Texture -> Surface -> TTF_Font 

Window -> Render -> Line, Rect and etc

那我们的频谱仪界面，如下图所示，将会是这样的：

![img_3](https://github.com/user-attachments/assets/3f65a8ec-ac06-4be7-b6aa-4362d67b2896)


整个区域分为 SPECTRUM REGION 和 TEXT REGION，其中 SPECTRUM REGION 用于显示频谱，USER REGION 用于 HackRF 配置交互。当然在频谱显示区域中，我们还可以加入鼠标显示频率和幅度、Grid 功能等等，这些将在另外的 Blog 中进行描述。

对这些尺寸的定义，如下所示：

```C
{% highlight C %}
#define WINDOW_MARGIN 10
#define REGION_LINE_WIDTH 2
#define SPECTRUM_REGION_WIDTH 1024
#define SPECTRUM_REGION_HEIGHT 480
#define USER_REGION_WIDTH 400
#define USER_REGION_HEIGHT 480
#define WINDOW_WIDTH 4*WINDOW_MARGIN+4*REGION_LINE_WIDTH+SPECTRUM_REGION_WIDTH+USER_REGION_WIDTH
#define WINDOW_HEIGHT 2*WINDOW_MARGIN+2*REGION_LINE_WIDTH+SPECTRUM_REGION_HEIGHT
{% endhighlight %}
```

## SDL WINDOW

SDL WINDOW 是 SDL 中最基础的一个部件，他创建了一个窗口, 其代码很简单，我们可以将窗口函数写为：

```C
{% highlight C %}
SDL_Window* GUI_Window_Create(char *WINDOW_NAME, int WINDOW_WIDTH, int WINDOW_HEIGHT){
    SDL_Window *window = SDL_CreateWindow(WINDOW_NAME,SDL_WINDOWPOS_UNDEFINED,SDL_WINDOWPOS_UNDEFINED,WINDOW_WIDTH,WINDOW_HEIGHT,SDL_WINDOW_SHOWN);
    if (!window) {
        fprintf(stderr, "SDL_CreateWindow Error: %s\n", SDL_GetError());
    }
    return window;
}
{% endhighlight %}
```
## SDL RENDER

SDL RENDER 是绘图、显示文字等比不可少的，创建一个 Render 的代码很简单，如下所示：

```C
{% highlight C %}
SDL_Renderer* GUI_Render_Create(SDL_Window *window){
    SDL_Renderer *render= SDL_CreateRenderer(window, -1, SDL_RENDERER_ACCELERATED);
    if (!render) {
        fprintf(stderr, "SDL_CreateRenderer Error: %s\n", SDL_GetError());
    }
    return render;
}
{% endhighlight %}
```

## SDL Rect with Line Width

接下来，我们先来看如何**直接**使用 Render 进行绘制。如 Overview 中所描述的，Render 可以直接进行一些基础图形的绘制，例如线条和矩形。那我们首先看一下如何创建一个有线宽的矩形。在 SDL 函数中，其实并不支持直接绘制有线宽的矩形，所以我们通过绘制一大一小两个矩形来实现。

首先我们定义一个由两个矩形构成的结构体，命名为 SDL_Rect_wLineWidth，这样方便我们直接传递“带线宽的矩形”给到其他函数:

```C
{% highlight C %}
typedef struct{
    SDL_Rect OutLayerRect;
    SDL_Rect InnerLayerRect;
    SDL_Color LineColor;
    SDL_Color RectColor;
}SDL_Rect_wLineWidth;
{% endhighlight %}
```

可以看到这个结构体中包括了两个矩形，以及最终这个“带线宽矩形”的线条颜色和填充颜色；接下来我们定一个返回 SDL_Rect_wLineWidth 的函数来绘制这个带线宽的矩形：

```C
{% highlight C %}
SDL_Rect_wLineWidth* GUI_Create_Rect_with_LineWidth(int pos_x, int pos_y, int width, int height, int line_width, SDL_Color Line_Color, SDL_Color Rect_Color){
    if ((width < 2*line_width)|(height< 2*line_width)){
        fprintf(stderr,"Line Width needs to be smaller than size of Rect \n");
    }
    static SDL_Rect_wLineWidth *Rect;
    Rect = (SDL_Rect_wLineWidth*) malloc(1*sizeof(SDL_Rect_wLineWidth));
    
    Rect->OutLayerRect.x = (pos_x);
    Rect->OutLayerRect.y = (pos_y);
    Rect->OutLayerRect.w = (width)+2*line_width;
    Rect->OutLayerRect.h = (height)+2*line_width;

    Rect->InnerLayerRect.x = pos_x + (line_width);
    Rect->InnerLayerRect.y = pos_y + (line_width);
    Rect->InnerLayerRect.w = width;
    Rect->InnerLayerRect.h = height;

    Rect->LineColor = Line_Color;
    Rect->RectColor = Rect_Color;

    return Rect;
}
{% endhighlight %}
```

**需要十分注意的是，这里待返回的 SDL_Rect_wLineWidth 需要用指针的形式，否则每次调用这个函数创建不同的带线宽矩形时将永远得到最后一个矩形的数值，其原因与之前一篇创建多个 ringBuffer 的博文是一样的**

在这个函数中，我们指定的是外面大矩形的位置和长宽，因此在开始的时候需要判断一下大矩形的宽度和高度需要大于2倍的线宽；接下来就是直接定义两个矩形的位置和长宽；最后再定义线条颜色与矩形颜色。

有了两个矩形，接下来我们通过 Render 直接进行绘制。绘制的第一步是通过 SDL_SetRenderDrawColor 函数指定要绘制的矩形的颜色，第二步则通过 SDL_RenderFillRect 进行绘制。注意我们先画大矩形再画小矩形以实现线条颜色与填充颜色的不同。

```C
{% highlight C %}
void GUI_Plot_Rect_with_LineWidth(SDL_Renderer *render, SDL_Rect_wLineWidth *Rect){
    SDL_SetRenderDrawColor(render,Rect->LineColor.r,Rect->LineColor.g,Rect->LineColor.b,Rect->LineColor.a);
    SDL_RenderFillRect(render,&Rect->OutLayerRect);
    SDL_SetRenderDrawColor(render,Rect->RectColor.r,Rect->RectColor.g,Rect->RectColor.b,Rect->RectColor.a);
    SDL_RenderFillRect(render,&Rect->InnerLayerRect);
}
{% endhighlight %}
```

## SDL Line with Line Width

Render 除了绘制矩形，也可以直接绘制线条，同样的，SDL Render 默认只能进行无线宽的线条绘制，同样的，我们需要自定义函数实现带线宽的线条绘制。

SDL_Render 绘制线条的方式有两种，一种是通过指定起始点的方式（SDL_RenderDrawLine(render,x_start,y_start,x_end,y_end))，另一种则是连接多个 SDL_Points 的方式（SDL_RenderDrawLines(render,points,count))

当前我们主要通过带线宽的线条来绘制网格方便找到频谱的峰值，因此我们主要针对第一种方式进行带线宽线条的函数定义。首先，我们定义一个结构体 SDL_Line_wLineWidth 来描述这种带线宽的线条：

```C
{% highlight C %}
typedef struct{
    int line_width;
    int* pos_x_s;
    int* pos_x_e;
    int* pos_y_s;
    int* pos_y_e;
    SDL_Color LineColor;
}SDL_Line_wLineWidth;
{% endhighlight %}
```

在这个结构体中，我们定义了 line_width 用来表示线条的宽度，同时定义了指针 pos_x_s, pos_x_e, pos_y_s 和 pos_y_e 来表示 line_width 条线；由于 line_width 是一个用户指定的输入，因此在定义结构体时不能提前分配内存空间，只能通过指针的方式进行定义；最后我们定义了线条的颜色 LineColor。本质上就是通过画多条线来实现线宽的效果。

接下来我们来定一个函数生成对应的带线宽的线条：

```C
{% highlight C %}
SDL_Line_wLineWidth* GUI_Create_Line_with_LineWidth(int pos_x_s, int pos_y_s, int pos_x_e, int pos_y_e, int line_width,SDL_Color Line_Color){
    static SDL_Line_wLineWidth *Line;
    Line = (SDL_Line_wLineWidth*) malloc(sizeof(SDL_Line_wLineWidth)*1);
    
    Line->pos_x_s = (int*) malloc(line_width*sizeof(int));
    Line->pos_x_e = (int*) malloc(line_width*sizeof(int));
    Line->pos_y_s = (int*) malloc(line_width*sizeof(int));
    Line->pos_y_e = (int*) malloc(line_width*sizeof(int));

    if(pos_x_s == pos_x_e) {
        for (int i = 0; i < line_width; i++) {
            Line->pos_x_s[i] = pos_x_s + i;
            Line->pos_x_e[i] = pos_x_e + i;
            Line->pos_y_s[i] = pos_y_s;
            Line->pos_y_e[i] = pos_y_e;
        }
    }
    else if(pos_y_s == pos_y_e){
        for (int i = 0; i < line_width; i++) {
            Line->pos_x_s[i] = pos_x_s;
            Line->pos_x_e[i] = pos_x_e;
            Line->pos_y_s[i] = pos_y_s+i;
            Line->pos_y_e[i] = pos_y_e+i;
        }
    }
    else if ((pos_y_s == pos_y_e)&(pos_x_s == pos_x_e)){
        fprintf(stderr,"Cannot plot a points with linewidth\n");
    }
    else{
        float k = (float)(pos_y_e-pos_y_s)/(pos_x_e-pos_x_s);
        // k > 0
        if ((k>1)|(k<-1)){
            for (int i = 0; i < line_width; i++) {
                Line->pos_x_s[i] = pos_x_s+i;
                Line->pos_x_e[i] = pos_x_e+i;
                Line->pos_y_s[i] = pos_y_s;
                Line->pos_y_e[i] = pos_y_e;

            }
        }
        else{
            for(int i=0;i<line_width;i++){
                Line->pos_x_s[i] = pos_x_s;
                Line->pos_x_e[i] = pos_x_e;
                Line->pos_y_s[i] = pos_y_s+i;
                Line->pos_y_e[i] = pos_y_e+i;
            }
        }
    }
    Line->line_width = line_width;
    Line->LineColor = Line_Color;

    return Line;
}
{% endhighlight %}
```

在上面的代码中，我们首先需要对 SDL_Line_wLineWidth 结构体中的指针变量进行内存分配，这里分配的内存空间由 line_width 决定，**这对于定义为指针的变量来说是十分重要的**; 接下来我们分情况进行线条的增加:

1. 当线条水平或垂直时，我们只需要增加另一方向上的线条数量即可，因此当线条水平时（平行于 X 轴）,我们在 Y 轴上增加线条，所以将 y_pos 进行增加；同理当线条垂直时（平行于 Y 轴），我们在 X 轴上增加线条，将 x_pos 进行增加。

2. 当线条有一定斜率时，一种方式是直接对 y 轴做平移，x 轴保持不变（或 x 轴做平移，y 轴保持不变），但是这种方式会导致不同斜率的线条在同样宽度下出现视觉上的不一致，如下图所示。可以看到，同样是往 y 轴平移一个单元格，k>1 与 k<1 的平移结果视觉差异很大。
   
![img_5](https://github.com/user-attachments/assets/71153ac4-bb6b-4013-9a7e-b0f6a45cbaf1)

因此我们需要对有斜率的线条进行分情况的处理：当 k > 1 或 <-1 时，也就是 y 轴方向的增加比 x 轴增加更大时，我们对 x 轴进行增加，对 y 轴则不增加；反之当 -1 < k < 1 时，对 y 轴进行增加，对 x 轴不增加。这样我们可以得到视觉宽度类似的斜线:

![img_6](https://github.com/user-attachments/assets/30918cc5-daf5-459c-8edb-30865aff14a0)

最后我们同样通过 Render 来对 “有线宽” 的线条进行绘制，本质就是多次调用 SDL_RenderDrawLine 函数来画多条线:

```C
{% highlight C %}
void GUI_Plot_Line_with_LineWidth(SDL_Renderer *render,SDL_Line_wLineWidth *Line){
    SDL_SetRenderDrawColor(render,Line->LineColor.r,Line->LineColor.g,Line->LineColor.b,Line->LineColor.a);
    for(int i=0;i<Line->line_width;i++){
        SDL_RenderDrawLine(render,Line->pos_x_s[i],Line->pos_y_s[i],Line->pos_x_e[i],Line->pos_y_e[i]);
    }
}
{% endhighlight %}
```
## Dotted SDL Line with Line Width

考虑到我们在绘制 Grid 的时候会用到虚线，SDL 中默认也无法支持虚线，因此我们在之前定义的 GUI_Plot_Line_with_LineWidth 函数中增加绘制虚线的功能。

虚线的本质是将一条定义了起始点和终点坐标的直线分解成多个点，然后间隔进行连接，例如 pt1-pt2,pt3-pt4... 就可以形成虚线的形式了。考虑到虚线也需要有一定的线宽，我们直接在 GUI_Plot_Line_with_LineWidth 函数中增加这个功能。**需要注意的是，当前虚线只会用于横竖的网格绘制，因此我们当前仅支持 X 和 Y 方向上的虚线功能**

更新后的 GUI_Plot_Line_with_LineWidth 函数如下所示：

```C
{% highlight C %}
void GUI_Plot_Line_with_LineWidth(SDL_Renderer *render,SDL_Line_wLineWidth *Line,int dotted,int dotted_gap){
    SDL_SetRenderDrawColor(render,Line->LineColor.r,Line->LineColor.g,Line->LineColor.b,Line->LineColor.a);
    if(!dotted){
        for(int i=0;i<Line->line_width;i++){
            SDL_RenderDrawLine(render,Line->pos_x_s[i],Line->pos_y_s[i],Line->pos_x_e[i],Line->pos_y_e[i]);
        }
    }
    else{
        if((Line->pos_x_s[0] == Line->pos_x_e[0])|(Line->pos_y_s[0] == Line->pos_y_e[0])){
            // Plot each line in line width
            for(int i=0;i<Line->line_width;i++){
                if(Line->pos_x_s[i] == Line->pos_x_e[i]){
                    // Get Length
                    int length = Line->pos_y_e[i]-Line->pos_y_s[i];
                    if(length<0){
                        length = -length;
                    }
                    // Prepare the points
                    int gap = dotted_gap;
                    SDL_Point points[(int)(length/gap)];
                    // Do the dot work
                    int k=0;
                    for(int j=0;j<(int)(length/gap);j++){
                        points[j].x = Line->pos_x_s[i];
                        points[j].y = Line->pos_y_s[i]+j*gap;
                        if((j>0)&(!(k%2))){
                            SDL_RenderDrawLine(render,points[j-1].x,points[j-1].y,points[j].x,points[j].y);
                        }
                        k++;
                    }
                }
                else{
                    // Get Length
                    int length = Line->pos_x_e[i]-Line->pos_x_s[i];
                    if(length<0){
                        length = -length;
                    }
                    // Prepare the points
                    int gap = dotted_gap;
                    SDL_Point points[(int)(length/gap)];
                    // Do the dot work
                    int k=0;
                    for(int j=0;j<(int)(length/gap);j++){
                        points[j].x = Line->pos_x_s[i]+j*gap;
                        points[j].y = Line->pos_y_s[i];
                        if((j>0)&(k%2)){
                            SDL_RenderDrawLine(render,points[j-1].x,points[j-1].y,points[j].x,points[j].y);
                        }
                        k++;
                    }
                }
            }
        }
        else{
            printf("DOTTED LINE DOES NOT SUPPORTED\n");
        }
    }
{% endhighlight %}
```
其中，else 为新增的虚线的绘制内容，又分为平行于 X 轴的虚线绘制和平行于 Y 轴的虚线绘制；SDL_Point 用来创建包含 X 和 Y 坐标的一系列点，然后通过奇偶判断的方式来实现间隔的两点连接。

## Grid ON/OFF

为了方便看到频谱位置和幅度大小，我们增加一个网格（Grid）功能，采用灰色的虚线。以垂直的网格线为例，其本质就是多条带线宽的虚线；因此我们应该首先用 GUI_Create_Line_with_LineWidth 生成多条指定线宽的线条，然后通过 GUI_Plot_Line_with_LineWidth 来绘制这些虚线。整个函数如下所示:

```C
{% highlight C %}
void GUI_GridOn(SDL_Renderer* main_render,SDL_Rect_wLineWidth *Rect_Region, int GRID_NUM_X, int GRID_NUM_Y){
    static SDL_Rect_wLineWidth** Grid_X;
    static SDL_Rect_wLineWidth** Grid_Y;
    Grid_X = (SDL_Rect_wLineWidth**) malloc(GRID_NUM_X*sizeof(SDL_Rect_wLineWidth*));
    Grid_Y = (SDL_Rect_wLineWidth**) malloc(GRID_NUM_Y*sizeof(SDL_Rect_wLineWidth*));

    int Grid_Gap_X = (int)Rect_Region->InnerLayerRect.w/(GRID_NUM_X+1);
    int Grid_Gap_Y = (int)Rect_Region->InnerLayerRect.h/(GRID_NUM_Y+1);
    int Grid_Width = 2;
    int Dotted_Gap = 10;
    SDL_Color COLOR_GRAY = {192,192,192,255};

    int GRID_pos_x_s = Rect_Region->InnerLayerRect.x;
    int GRID_pos_x_e = Rect_Region->InnerLayerRect.x+Rect_Region->InnerLayerRect.w;
    int GRID_pos_y_s = Rect_Region->InnerLayerRect.y;
    int GRID_pos_y_e = Rect_Region->InnerLayerRect.y+Rect_Region->InnerLayerRect.h;

    for(int i=0;i<GRID_NUM_X;i++){
        Grid_X[i] = GUI_Create_Line_with_LineWidth(GRID_pos_x_s+(i+1)*Grid_Gap_X,GRID_pos_y_s,GRID_pos_x_s+(i+1)*Grid_Gap_X,GRID_pos_y_e,Grid_Width,COLOR_GRAY);
        GUI_Plot_Line_with_LineWidth(main_render,Grid_X[i],1,Dotted_Gap);
    }
    for(int i=0;i<GRID_NUM_Y;i++){
        Grid_Y[i] = GUI_Create_Line_with_LineWidth(GRID_pos_x_s,GRID_pos_y_s+(i+1)*Grid_Gap_Y,GRID_pos_x_e,GRID_pos_y_s+(i+1)*Grid_Gap_Y,Grid_Width,COLOR_GRAY);
        GUI_Plot_Line_with_LineWidth(main_render,Grid_Y[i],1,Dotted_Gap);
    }
}
{% endhighlight %}
```

Step1. 由于 GUI_Create_Line_with_LineWidth 函数是返回的一个 SDL_Line_wLineWidth 的**指针**，因此当需要调用这个函数生成多个线条时，我们是需要一个指针的数组;为此，我们创建一个**指针的指针**来实现该功能;与指针类似的，我们也需要通过 malloc 来为这个指针的指针分配内存空间.

Step2. 接下来我们计算出每条虚线的间隔，其方法是获取要绘制网格区域的宽度（或长度），然后通过定义的虚线条数进行分割计算，注意间隔的数量应该是虚线条数加 1.

Step3. 最后通过 For 循环依次生成对应的虚线，并通过 GUI_Plot_Line_with_LineWidth 进行绘制就可以得到网格了。注意这里 Grid_X/Y[i] 是对应一个 SDL_Line_wLineWidth 的指针。

## Build the Spectrum Analyzer GUI

最后，我们利用这些函数来设计我们的界面，为了方便调用，我们将静态的 GUI 界面（包括可能改变的坐标轴标题等）绘制放在 sdl_gui.c 的一个函数中，这样在每次刷新频谱后都可以方便的调用该函数调来恢复 GUI 界面（本质是重新绘制 render）。

该函数 GUI_Create_HackRF_SA_Components 如下所示，三个函数分别是: 1）生成一个 Spectrum Analyzer 的显示区域 2）生成一个用户交互区域 3) 开启网格显示，将 X 轴分为 8 等份，将 Y 轴分为 10 等份。

```C
{% highlight C %}
void GUI_Create_HackRF_SA_Components(SDL_Renderer* main_render,SDL_Rect_wLineWidth *SA_REGION,SDL_Rect_wLineWidth *USER_REGION){
    // Create SA REGION
    GUI_Plot_Rect_with_LineWidth(main_render,SA_REGION);
    // Create USER REGION
    GUI_Plot_Rect_with_LineWidth(main_render,USER_REGION);
    // Grid ON
    GUI_GridOn(main_render,SA_REGION,7,9);
}
{% endhighlight %}
```
最后，我们在 main.c 中，调用 sdl_gui.c 中的这些函数以实现 GUI 界面的创建。

```C
{% highlight C %}
// Create Window
SDL_Window *main_window = GUI_Window_Create("HACKRF_SA",WINDOW_WIDTH,WINDOW_HEIGHT);

// Create Render
SDL_Renderer *main_render = GUI_Render_Create(main_window);

// Create FOR HACKRF SA and USER GUI
SDL_Rect_wLineWidth *Rect_SA = GUI_Create_Rect_with_LineWidth(sa_pos_x,sa_pos_y,sa_width,sa_height,region_lw,COLOR_BLACK,COLOR_WHITE);
SDL_Rect_wLineWidth *Rect_USER = GUI_Create_Rect_with_LineWidth(usr_pos_x,usr_pos_y,usr_width,usr_height,region_lw,COLOR_BLACK,COLOR_WHITE);

// Plot BG Color + SA GUI
SDL_SetRenderDrawColor(main_render,255,255,255,255);
SDL_RenderClear(main_render);
GUI_Create_HackRF_SA_Components(main_render,Rect_SA,Rect_USER);

// Show
SDL_RenderPresent(main_render);
{% endhighlight %}
```
注意代码中的尺寸相关的变量都是按照 Overview 中所示的尺寸定义的，最终输出的结果如下图所示:

![img_7](https://github.com/user-attachments/assets/44260a32-249a-403c-a8df-b56c478c9a41)
