---
layout: post
title: HackRF 频谱仪之 SDL(三)
author: Xuxiang Liu
date: 2024-09-21 23:16 +0800
tags: [SDR,CN]
---

HackRF 频谱仪 - 通过 SDL 增加文字、存储静态背景
{: .message }

接下来我们看一下如何通过 SDL 增加文本。实际上通过 SDL 增加文本是很麻烦的事情，需要用到 TTF 库，如上面所描述的，SDL 增加 Text 的流程是这样的：

Window -> Render -> Texture -> Surface -> TTF_Font 

除了前面绘制基础界面用到的 Window 和 Render，我们还需要用到 Texture, Surface 和 TTF_Font. 其相对关系是，TTF_Font 定义了字体相关的信息，包括字体类型和大小；然后创建一个 Surface 来承载这个字体，然后再用 Texture（硬件）的方式在渲染器（Render）上加载这个 Surface，最后渲染器在定义的 Window 中进行绘制显示；当然整个过程中还需要用到 SDL_Rect 来定义我们显示文字的区域。

所以首先我们需要与 TTF_Font 打交道。

## TTF_Font Library

如前所述，TTF 是 SDL 的一个库，因此为了使用他，我们同样需要在 CMake File 中进行添加，TTF 的库文件与 SDL 一样都在这个路径下 /usr/local/lib，所以只需要增加 target_link_libraries 即可:

{% highlight CMake %}
```CMake
# Library File Path
target_link_directories(BasicFunctions PRIVATE /usr/local/lib)
# Link Libraries
target_link_libraries(BasicFunctions PRIVATE SDL2_ttf)
```
{% endhighlight %}

在对应的 .c 文件中，我们同样增加这个头文件来使用 TTF 库

{% highlight C %}
```C
#include <SDL2/SDL_ttf.h>
```
{% endhighlight %}

## Surface & TTF_Font

接下来，我们看一下如何在 Surface 上承载一个 TTF_Font ，注意 Surface 除了可以承载文字，也可以承载各种图形文件，这部分在后面如果用上会展开相应的介绍。

Surface 承载 TTF_Font 对象分为两步：

Step1. 使用 TTF_OpenFont 从指定存放 font 的路径创建一个 TTF_Font 指针来加载文字字体和大小

Step2. 使用 TTF_RenderText_Solid 创建一个 Surface 来承载需要显示的文字和颜色，并使用之前定义的 TTF_Font 指针来定义文字的字体和大小

为方便我们后续添加不同的文字，我们创建一个函数来返回承载文字的 Surface ，这个函数的输入就是我们需要的 字体、大小、颜色和文字内容:

{% highlight C %}
```C
SDL_Surface* GUI_Text_Surface_Create(char* font_path,int font_size, char* text, SDL_Color text_color){
    TTF_Font* font = TTF_OpenFont(font_path,font_size);
    SDL_Surface *surface = TTF_RenderText_Solid(font,text,text_color);
    return surface;
}
```
{% endhighlight %}

## Texture with Rect

有了 Surface，我们就需要一个 Texture 来加载这个 Surface，注意由于这个 Texture 要放在 Render 上面，就需要指定我们希望放置在 Render 上的 X，Y 坐标，以及加载字体需要的长宽，所以我们定义一个结构体来存储：

{% highlight C %}
```C
typedef struct{
    SDL_Texture *texture;
    SDL_Rect text_region;
}SDL_Text_Texture_wRegion;
```
{% endhighlight %}

这个结构体中，除了 Texture 以外，还有一个矩形 Rect，这个 Rect 的位置和长宽就是我们要显示的字体的位置和长宽。接下来我们定一个函数来返回这个结构体：

{% highlight C %}
```C
SDL_Text_Texture_wRegion* GUI_text_Texture_Create(SDL_Renderer *render, SDL_Surface *surface, int text_x_pos, int text_y_pos){
    SDL_Text_Texture_wRegion *textTextureWRegion;
    
    // Create Texture
    textTextureWRegion->texture = SDL_CreateTextureFromSurface(render, surface);
    // Create region Rect
    textTextureWRegion->text_region.x = text_x_pos;
    textTextureWRegion->text_region.y = text_y_pos;
    SDL_QueryTexture(textTextureWRegion->texture,NULL,NULL,&textTextureWRegion->text_region.w,&textTextureWRegion->text_region.h);
    
    return textTextureWRegion;
}
```
{% endhighlight %}

在这个函数中，主要分为两步:

Step1. 通过 SDL_CreateTextureFromSurface 函数，将指定的 surface 加载到 Texture 上

Step2. Rect 的 x 和 y 定义为我们希望放置文字的位置；再通过 SDL_QueryTexture 函数返回这个加载了文字 Surface 的 Texture 的长和宽，也就是对应 Rect 的长和宽。这样就构建了我们需要的 Texture 和 Rect，并将他们放置于结构体 textTextureWRegion 中

## Add Text On Render

最后，我们有了 Texture，只需要将其放在 Render 上即可，SDL 中通过 SDL_RenderCopy 的方式将 Texture 放置在 Render 中，因此我们定一个简单的函数来实现该功能:

{% highlight C %}
```C
void GUI_Add_Text(SDL_Renderer *render,SDL_Text_Texture_wRegion *textTextureWRegion){
    int result = SDL_RenderCopy(render,textTextureWRegion->texture,NULL,&textTextureWRegion->text_region);
    if(result<0){
        fprintf(stderr,"Render Add Text Fail \n");
    }
}
```
{% endhighlight %}

最后我们看一下在坐标轴上添加 X-Label 后的效果，我们将上述我们的频谱仪中心是指定的中心频率，每个网格是间隔的频率刻度，我们将字体设置为红色，可以得到如下图所示的文字展示了。

![图片](https://github.com/user-attachments/assets/087a48b4-c77d-47ae-94f3-4cfa32f15490)

## More About Texture

当我们需要快速刷新背景时，会用到 SDL_RenderCLear，这样所有的 Render 上的内容就被清掉了；当下次需要显示静态背景时又需要重新去加载所有的内容到 Render 上，会影响显示的刷新率。因此我们可以通过 Texture 的方式，来将静态的背景做成 Texture（纹理），这样在下次加载时就不需要重新生成 Render，只需要将这个 Texture Copy 到空白的 Render 上即可。

所以我们回顾之前使用 Texture 来加载字体的时候，实际上也是将带有字体的 Texture（纹理）Copy 到 Render 上，所以其本质原理是一样的，我们之前画的两个途径其实本质上是可以合并成同一个途径的：

Window -> Render -> Texture -> Surface -> TTF_Font
                            -> Surface -> Bmp(Figure)
                            -> Rect, Line and etc (using RenderDraw)

或者用下图来表示：

![图片](https://github.com/user-attachments/assets/557cc5bc-3a96-4346-b27f-e1fc7b642308)

所以我们可以将当前的内容全部做为频谱仪显示的静态背景，修改代码如下图所示（理论上坐标轴文字是会随 HackRF 使用的采样率变化，这里暂时先把他也作为静态背景）

{% highlight C %}
```C
// Create Window
SDL_Window *main_window = GUI_Window_Create("HackRF SA",WINDOW_WIDTH,WINDOW_HEIGHT);

// Create Render and BackGround Texture
SDL_Renderer *main_render = GUI_Render_Create(main_window);
SDL_Texture *texture_bg = SDL_CreateTexture(main_render,SDL_PIXELFORMAT_RGB888,SDL_TEXTUREACCESS_TARGET,WINDOW_WIDTH,WINDOW_HEIGHT);

// Draw on BackGround BG
SDL_SetRenderTarget(main_render,texture_bg);

// Create FOR HACKRF SA and USER GUI
SDL_Rect_wLineWidth *Rect_SA = GUI_Create_Rect_with_LineWidth(sa_pos_x,sa_pos_y,sa_width,sa_height,region_lw,COLOR_BLACK,COLOR_WHITE);
SDL_Rect_wLineWidth *Rect_USER = GUI_Create_Rect_with_LineWidth(usr_pos_x,usr_pos_y,usr_width,usr_height,region_lw,COLOR_BLACK,COLOR_WHITE);

// Plot BG Color + SA GUI
SDL_SetRenderDrawColor(main_render,255,255,255,255);
SDL_RenderClear(main_render);
GUI_Create_HackRF_SA_Components(main_render,Rect_SA,Rect_USER);

// STOP Draw On BackGround BG
SDL_SetRenderTarget(main_render,NULL);

// Copy BG texture on Render
SDL_RenderCopy(main_render,texture_bg,NULL,NULL);

// Show Render
SDL_RenderPresent(main_render);
```
{% endhighlight %}

对比上一篇 Blog 可以看到，改动的几步分别是：

Step 1. 通过 SDL_CreateTexture 创建一个背景的 Texture（纹理），其中定义了像素格式是 RGB888，大小就是 Window 的尺寸，因为它是作为背景

Step 2. 通过 SDL_SetRenderTarget，将后续的关于 Render 的操作全部作用于 Texture 上；换句话说，这个时候去绘制 Render 看不到任何东西，因为所有的东西全部是在 texture 上面

Step 3. 绘制完成后，再次通过 SDL_SetRenderTarget 将 Target 还原到 Render 上，后续所有关于 Render 的操作将之间作用于 Render；也就是说我们的背景已经绘制完毕，并且存储在 texture_bg 这个纹理上

Step 4. 通过 SDL_RenderCopy 将 texture_bg 复制到我们的 Render 上，这个时候相当于 Render 上就粘贴了这个背景的纹理

最后调用 SDL_RenderPresent, 我们就可以看到和之前一样的区域、坐标轴 和 坐标轴文字了。虽然看起来都是一样的，但实际上我们已经将这些显示内容存在了 texture 上，Render 上只是粘贴了这个 texture 而已，和之前直接放在 Render 上还是有本质的区别。
