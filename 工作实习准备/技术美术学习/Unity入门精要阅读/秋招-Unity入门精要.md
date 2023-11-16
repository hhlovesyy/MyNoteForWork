# 秋招-Unity入门精要

VS使用工具下载：

[(12条消息) 解决在VS中编写Unity Shader代码高亮显示、代码补全、自动缩进_山纹鱼的博客-CSDN博客_vs的unity插件](https://blog.csdn.net/weixin_50617270/article/details/123719076)

这样可以方便我们Shader的编写。

# 第二章 渲染管线

## 1.RTR中的描述

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220531213115019.png" alt="image-20220531213115019" style="zoom:80%;" />

### （1）应用阶段（通常CPU）

开发者的任务：

- 准备场景位置，光源数据；
- 粗粒度剔除culling；
- 设置每个模型的渲染状态，比如材质，纹理，Shader等。

输出内容：

**渲染图元**（点，线，三角面等）

### （2）几何阶段（GPU）

负责和每个渲染图元打交道。几何阶段的一个重要任务就是**把顶点坐标变换到屏幕空间中，再交给光栅器进行处理。**这一阶段将会输出屏幕空间的二维顶点坐标、每个顶点对应的深度值、着色等相关信息，并传递给下一个阶段。

### （3）光栅化阶段（GPU）

需要对上一个阶段得到的逐顶点数据（例如纹理坐标、顶点颜色等）进行插值，然后再进行逐像素处理。

> 这三个流水线阶段与GPU流水线阶段要做区分，这些是概念流水线，而GPU流水线将在接下来进行介绍。

<br/>

## 2.CPU和GPU之间的通信

对应上述的应用阶段，这个阶段会：

（1）数据加载到显存中；

渲染所需要的数据从硬盘加载到系统内存（RAM），然后网格和纹理等数据被加载到显卡的存储空间（显存）中。**这是因为显卡对于显存的访问速度更快，而且大多数显卡对于RAM没有直接的访问权利。**

加载到显存中的还有顶点位置，颜色，纹理坐标等。

理论上加载到显存之后，RAM对应的就可以删除了（不过有时还要做碰撞检测等，所以不删除，因为硬盘加载到RAM很慢）

（2）设置渲染状态

（3）调用Draw Call

**渲染优化的一个思路，减少Draw Call**

实际上，Draw Call就是一个命令，它的发起方是CPU，接收方是GPU。这个命令仅仅会指向一个需要被渲染的图元（primitives）列表，而不会再包含任何材质信息——这是因为我们已经在上一个阶段中完成了。

当给定了一个Draw Call时，GPU就会根据渲染状态（例如材质、纹理、着色器等）和所有输入的顶点数据来进行计算，最终输出成屏幕上显示的那些漂亮的像素，这就是GPU流水线，接下来会介绍。

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220531214409437.png" alt="image-20220531214409437" style="zoom:80%;" />

<br/>

## 3.GPU流水线

在上述提到的应用阶段，几何阶段，光栅化阶段，后面两个阶段由GPU实现，但对应GPU也会提供一些可配置性和可编程性。

![image-20220531214637979](E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220531214637979.png)

可以看到：

（1）**屏幕映射，三角形设置，三角形遍历**由GPU固定实现，开发者没有控制权；

（2）**顶点着色器，曲面细分着色器，几何着色器，片元着色器**可以编程控制，而且除了顶点着色器必须由开发者编程实现，其他的都是可选的。

（3）**裁剪和后面的逐片元操作**可以选择是否配置（可配置是指比如可以选择剔除正面或背面，选择是否开启深度测试等），但是不可编程。

<br/>

各个模块的作用如下：

### （1）顶点着色器

常用于实现顶点的空间变换，顶点着色等。输入来自CPU，处理单位是顶点（也就是输入进来的每个顶点都会调用一次顶点着色器）。

顶点着色器不能创建或销毁顶点，且无法得知两个点是否属于同一个三角形（这就意味着可以并行化处理，速度快）。

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220531215411112.png" alt="image-20220531215411112" style="zoom:67%;" />

**坐标变换可以用来制作顶点动画，同时也可以做水面，布料等。顶点着色器必须把顶点坐标从模型空间转移到齐次裁剪空间。**

顶点坐标-> 齐次裁剪坐标系-> 做透视除法，得到归一化的设备坐标NDC。Unity和OpenGL使用同一套NDC,z分量会被控制在[-1,1]。

<br/>

### （2）裁剪

当我们知道了在NDC下的顶点位置（此时顶点位置在一个立方体内），接下来就可以直接在NDC中做裁剪操作。

![image-20220531215901880](E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220531215901880.png)

<br/>

### （3）屏幕映射

此时的输入仍然是三维坐标系下的坐标（范围在单位立方体里）。**屏幕映射任务是把图元的x和y坐标转换到屏幕坐标系（二维坐标系，和显示分辨率有关系）**

直观理解，这就是一个缩放过程，**但是屏幕映射过程不会对z处理，屏幕坐标系和Z坐标共同组成窗口坐标系**，这些值会被一起传递到光栅化阶段。

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220531220140734.png" alt="image-20220531220140734" style="zoom:67%;" />

值得注意的是，OpenGL采用左下角作为原点，DirectX采用左上角作为原点（微软喜欢的一套：从上到下，从左到右）。

<br/>

### （4）三角形设置&三角形遍历

从上个阶段输出的是屏幕坐标系下的顶点位置和对应深度值z，以及比如法线方向，视角方向等。

> 光栅化有两个重要目标：
>
> - 计算每个图元覆盖了哪些像素；
> - 计算这些像素的颜色。

（a）**三角形设置**会计算光栅化一个三角网格所需的信息，比如边界上的像素坐标。

（b）**三角形遍历**会检查每个像素是否被一个三角网格覆盖，如果被覆盖就会生成一个**片元**，这个过程也被称为扫描变换。

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220531220834913.png" alt="image-20220531220834913" style="zoom: 67%;" />

（注意，插值的时候很多时候采用重心插值，且输出的片元序列包含了很多像素的状态，比如屏幕坐标，深度，法线，纹理坐标等。）

<br/>

### （5）片元着色器

前面的光栅化阶段实际上并不会影响屏幕上每个像素的颜色值，而是会产生一系列的数据信息，用来表述一个三角网格是怎样覆盖每个像素的。而每个片元就负责存储这样一系列数据。真正会对像素产生影响的阶段是下一个流水线阶段——**逐片元操作（Per-Fragment Operations）** 。

片元着色器的输入是上一个阶段对顶点信息插值得到的结果，更具体来说，是根据那些从顶点着色器中输出的数据插值得到的。而它的输出是一个或者多个颜色值。

**这个阶段可以完成很多重要的渲染技术，比如说纹理采样。**我们通常会在顶点着色器阶段输出每个顶点对应的纹理坐标，然后经过光栅化阶段对三角网格的3个顶点对应的纹理坐标进行插值后，就可以得到其覆盖的片元的纹理坐标了。

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220531221134670.png" alt="image-20220531221134670" style="zoom:67%;" />

片元着色器只能影响单个片元，**但是可以访问到导数信息。**

<br/>

### （6）逐片元操作

这一阶段有几个主要任务。

（1）决定每个片元的可见性。这涉及了很多测试工作，例如深度测试、模板测试等。

（2）如果一个片元通过了所有的测试，就需要把这个片元的颜色值和已经存储在颜色缓冲区中的颜色进行合并，或者说是混合。

需要指明的是，逐片元操作阶段是高度可配置性的，即我们可以设置每一步的操作细节。

![image-20220531221412627](E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220531221412627.png)



#### （a）模板测试和深度测试

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220531222354039.png" alt="image-20220531222354039" style="zoom:67%;" />

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220531223726190.png" alt="image-20220531223726190" style="zoom: 67%;" />

#### （b）合并操作

**为什么要进行合并操作?**

 因为这里的渲染过程是一个物体接着一个物体画到屏幕上的。而**每个像素的颜色信息被存储在一个名为颜色缓冲的地方**。因此，当我们执行这次渲染时，**颜色缓冲中往往已经有了上次渲染之后的颜色结果**，此时合并解决的问题是是否要使用这次渲染得到的颜色完全覆盖掉之前的颜色.

比如说，在渲染不透明物体的时候可以**关闭混合操作，这样片元着色器计算得到的颜色值就会直接覆盖掉颜色缓冲区中的像素值**；但对于半透明物体，就要用混合操作来使得这个物体看起来是透明的。

![image-20220601112140077](E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220601112140077.png)

（可以看到，混合操作也是可以高度配置的，也可以指定混合的函数，类似于PhotoShop）

#### （c）Early-Z技术

Unity给出的渲染流水线中，我们发现其给出的深度测试是在片元着色器之前，这就是Early-Z技术。后续还会详细介绍，

> 但是，如果将这些测试提前的话，其检验结果可能会与片元着色器中的一些操作冲突。例如，如果我们在片元着色器进行了透明度测试（我们将在8.3节中具体讲到），而这个片元没有通过透明度测试，我们会在着色器中调用API（例如clip函数）来手动将其舍弃掉。这就导致GPU无法提前执行各种测试。**因此，现代的GPU会判断片元着色器中的操作是否和提前测试发生冲突，如果有冲突，就会禁用提前测试。**但是，这样也会造成性能上的下降，因为有更多片元需要被处理了。这也是透明度测试会导致性能下降的原因。

经过重重计算和测试后，图元的颜色就会显示在屏幕上，为了避免我们看见正在光栅化的图元，GPU采用**双重缓冲（前置缓冲区和后置缓冲区，GPU会交换显示两者。）**

<br/>

## 4.一些常见问题

### （1）OpenGL&DirectX

> 概括来说，我们的应用程序运行在CPU上。应用程序可以通过调用OpenGL或DirectX的图形接口将渲染所需的数据，如顶点数据、纹理数据、材质参数等数据存储在显存中的特定区域。随后，开发者可以通过图像编程接口发出渲染命令，**这些渲染命令也被称为Draw Call，它们将会被显卡驱动翻译成GPU能够理解的代码，进行真正的绘制。**
>
> 一个显卡除了有图像处理单元GPU外，还拥有自己的内存，这个内存通常被称为**显存（Video Random Access Memory，VRAM）** 。GPU可以在显存中存储任何数据，但对于渲染来说一些数据类型是必需的，例如用于屏幕显示的图像缓冲、深度缓冲等。



### （2）DrawCall

关于这部分的剩余内容，可以参考之前博客当中整理的内容：

[《Unity Shader入门精要》阅读笔记(1)-渲染管线介绍 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/428939964)

<br/>

# 第三章 Unity Shader基础

如何创建一个材质,并赋予shader的步骤在这里略去.

在Unity 2021版本当中,有如下几种Shader:

![image-20220601110031818](E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220601110031818.png)

- Standard Surface Shader会产生一个包含了标准光照模型的Shader
- Unlit Shader产生一个不包含光照(但包含雾效)的基本的顶点/片元着色器.
- Image Effect Shader为各种屏幕后处理效果提供模板;
- Compute Shader会产生一种特殊的Shader文件，这类Shader旨在利用GPU的并行性来进行一些与常规渲染流水线无关的计算,不在本书的讨论范围里.

**目前来说,我们使用Unlit Shader来产生基本的顶点/片元着色器.**

<br/>

## 1.Unity Shader的结构

### （1）Properties属性

```glsl
//Properties里的东西会出现在材质面板上
//Name ("display name", PropertyType) = DefaultValue
Shader "Custom/ShaderLabProperties" {
    Properties {
        // Numbers and Sliders
        _Int ("Int", Int) = 2
        _Float ("Float", Float) = 1.5
        _Range("Range", Range(0.0, 5.0)) = 3.0
        // Colors and Vectors
        _Color ("Color", Color) = (1,1,1,1)
        _Vector ("Vector", Vector) = (2, 3, 6, 1)
        // Textures
        _2D ("2D", 2D) = "" {}
        _Cube ("Cube", Cube) = "white" {}
        _3D ("3D", 3D) = "black" {}
    }

    FallBack "Diffuse"
}
```

其中Name是Shader中访问的变量，display name是材质面板上可以拖动调整的属性。

如果要使用bool变量等，Unity允许我们重载默认的材质编辑面板（后面有时间再进行整理）

> 需要说明的是，即使我们不在*Properties* 语义块中声明这些属性，也可以直接在Cg代码片中定义变量。此时，我们可以通过脚本向Shader中传递这些属性。因此，*Properties* 语义块的作用仅仅是为了让这些属性可以出现在材质面板中。也就是说，**Properties语义块中定义的变量如果想要在Cg语句块中继续使用的话需要在Cg语句块中重新定义。**

### （2）SubShader

Shader中可以包含多个SubShader，但至少要有一个。Unity会扫描所有的SubShader，然后选择第一个能在目标平台运行的SubShader，如果都不支持的话会采用Fallback提供的。

Subshader当中往往定义如下：

```glsl
SubShader {
    // 可选的
    [Tags]

    // 可选的
    [RenderSetup]

    Pass {
    }
    // Other Passes
}
```

每个Pass中定义了一次完整的渲染流程，但如果Pass过多会造成渲染性能的下降，因此要尽量使用小数目的Pass，Pass里面也可以定义状态和标签，不同的是，*SubShader* 中的一些标签设置是特定的。

也就是说，**SubShader和Pass中定义的状态语法是相同的，但是标签有所不同。如果在SubShader中采用了某种设置，会应用于所有的Pass。**

常用渲染状态设置如下：

![image-20220601112140077](E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220601112140077.png)

可以在SubShader中定义这些渲染状态，会应用于所有的Pass，也可以为某个Pass单独设置。

<br/>

对于标签来说，SubShader的标签是一个键值对，键和值都是字符串类型，用来告诉Unity怎么样及何时渲染这个对象。

```glsl
Tags { "TagName1" = "Value1" "TagName2" = "Value2" }
```

**SubShader**支持的标签类型如下（注意，这些不可支持于Pass的标签）：

![image-20220601112724276](E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220601112724276.png)

### （3）Pass

```gl
Pass { 
    [Name]
    [Tags] 
    [RenderSetup] 
    // Other code
}
```

可以指定Pass的名字：

```c
Name "MyPassName"
```

这样就可以使用ShaderLab的UsePass命令来引用这个Pass：

```c#
UsePass "MyShader/MYPASSNAME"
```

注意，Unity内部会把所有Pass的名称转换为大写字母表示，所以在引用的时候也要采用全大写字母来引用。

Pass中也可以设置一些标签，如下：

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220601113423706.png" alt="image-20220601113423706" style="zoom: 67%;" />

### （4）留后路：Fallback

跟在所有的SubShader后面，如果所有的SubShader都不能执行，就用这个。比如：

```c#
Fallback "name"
// 或者
Fallback Off
```

还有一些其他的Shader语义，有需要的时候再进行介绍。

<br/>

## 2.Unity Shader的形式

```c#
Shader "MyShader" {
    Properties {
        // 所需的各种属性
    }
    SubShader {
        // 真正意义上的Shader代码会出现在这里
        // 表面着色器（Surface Shader）或者
        // 顶点/片元着色器（Vertex/Fragment Shader）或者
        // 固定函数着色器（Fixed Function Shader）
    }
    SubShader {
        // 和上一个SubShader类似
    }
}
```

### （1）表面着色器

> 当给Unity提供一个表面着色器的时候，它在背后仍旧把它转换成对应的顶点/片元着色器。我们可以理解成，表面着色器是Unity对顶点/片元着色器的更高一层的抽象。它存在的价值在于，Unity为我们处理了很多光照细节，使得我们不需要再操心这些“烦人的事情”。

（可以参考原书3.4章节，有需要再补充表面着色器的相关知识）

### （2）顶点/片元着色器

相比于表面着色器，他们更加复杂，但灵活性也更强。

```glsl
Shader "Custom/Simple VertexFragment Shader" {
    SubShader {
        Pass {
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag

            float4 vert(float4 v : POSITION) : SV_POSITION {
                return mul (UNITY_MATRIX_MVP, v);
            }

            fixed4 frag() : SV_Target {
                return fixed4(1.0,0.0,0.0,1.0);
            }

            ENDCG
        }
    }
}

```

与表面着色器不同之处在于，这种着色器是写在Pass内部的，而表面着色器是写在SubShader里的，同样，这里的*CGPROGRAM* 和*ENDCG* 之间的代码也是使用Cg/HLSL编写的。

<br/>

另外，还有固定函数着色器，在这里不做展开了。

## 3.一些常见问题

### （1）Unity Shader和Cg/HLSL之间的关系

> Unity Shader是用ShaderLab语言编写的，但对于表面着色器和顶点/片元着色器，我们可以在ShaderLab内部嵌套Cg/HLSL语言来编写这些着色器代码。这些Cg/HLSL代码是嵌套在*CGPROGRAM* 和*ENDCG* 之间的，正如我们之前看到的示例代码一样。由于Cg和DX9风格的HLSL从写法上来说几乎是同一种语言，因此在Unity里Cg和HLSL是等价的。我们可以说，Cg/HLSL代码是区别于ShaderLab的另一个世界。

<br/>

# 第四章 学习Shader所需要的数学基础

> 由于这里所需的数学知识与计算机图形学是类似的，因此在这里只列举标题和部分很重要的内容，其他内容如果忘记了可以直接去原书查看。

## 1.Unity使用的坐标系

对于模型空间和世界空间来说，Unity**采用左手坐标系**，可以从Scene窗口中的物体坐标系看出来：

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220601115615082.png" alt="image-20220601115615082" style="zoom: 67%;" />

可以看到，确实是左手坐标系。

但对于观察空间来说，Unity使用的是**右手坐标系**。观察空间，通俗来讲就是以摄像机为原点的坐标系。在这个坐标系中，摄像机的前向是*z* 轴的负方向，这与在模型空间和世界空间中的定义相反。也就是说，*z* 轴坐标的减少意味着场景深度的增加。

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220601115824911.png" alt="image-20220601115824911" style="zoom: 50%;" />

## 2.点乘和叉乘

对于归一化的两个向量，其有如下性质：
$$
cos\theta=\vec{a}·\vec{b}
$$
对叉乘来说，其不满足交换律和结合律，计算的结果的模为两个向量组成的平行四边形的面积。
$$
|\vec{a}×\vec{b}|=|\vec{a}||\vec{b}|sin\theta
$$
值得注意的是，$\vec{a}×\vec{b}$的方向由左手坐标系还是右手坐标系来决定。

关于点乘和叉乘的应用，可以参考之前在知乎发的文章：

[《Unity Shader入门精要》阅读笔记(3)-数学篇(1) - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/430122626)

<br/>

## 3.矩阵

### （1）矩阵的性质

- 矩阵不满足交换律，但是满足结合律。

- 如果一个矩阵有对应的逆矩阵，则称这个矩阵可逆，或者非奇异矩阵，否则如果没有逆矩阵则成为奇异矩阵。（判断行列式=0，即为不可逆矩阵，否则是可逆矩阵）

### （2）正交矩阵

正交矩阵的特点为：$MM^T=M^TM=I$

所以说对于正交矩阵，有：$M^T=M^{-1}$,这样的好处在于有时我们在做逆变换的时候，需要乘某个矩阵的逆矩阵，而逆矩阵的计算量较大；所以如果这个矩阵是正交矩阵的话，可以直接用转置矩阵表示逆矩阵，使得运算方便。

判断一个矩阵是否是正交矩阵，除了从定义出发，还有一种方案是**直接根据矩阵的构造过程来判断是否为正交矩阵。**

正交矩阵的特点推导如下：

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220601161955041.png" alt="image-20220601161955041" style="zoom: 50%;" />

从上图可以看出，三个向量的模都是1并且彼此要垂直，**这样的矩阵就是正交矩阵**。所以说，一组标准正交基组成的矩阵就是正交矩阵。（正交基不要求长度一定是1，但标准正交基要求模为1）

### （3）Unity向量变换

Unity采用列向量，左乘变换矩阵，来实现对位置，旋转方向等的变换。

<br/>

## 4.矩阵的几何意义——变换

### （1）线性变换

线性变换指的是那些可以保留矢量加和标量乘的变换，用数学公式表示就是：
$$
f(x)+f(y)=f(x+y) \\\\
kf(x)=f(kx)
$$
常见的线性变换有：旋转，缩放，错切，镜像，正交投影。**注意平移变换并不是线性变换。**

### （2）仿射变换

仿射变换就是合并线性变换和平移变换的变换类型，使用4×4的矩阵来表示，需要扩展到**齐次坐标空间。**

常见的变换种类和对应的类型：

![image-20220601163512447](E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220601163512447.png)

（注：需要熟悉旋转矩阵和镜像矩阵是正交矩阵。）

### （3）齐次坐标

对于一个点，三维坐标转换为齐次坐标会把w分量设为1；对于方向矢量来说w分量则是0（这是因为如果对点进行缩放，旋转，平移的话都会施加于该点，而对向量来说平移会被忽略。）

### （4）分解基础变换矩阵

一个基础的变换矩阵可以进行如下分解：

$$
\begin{bmatrix}
M_{3×3}& t_{3×1} \\
0_{1×3}& 1
\end{bmatrix}
$$
左上角的矩阵用于旋转缩放，右上角是$t_{3×1}$矩阵用于平移，左下角为[0,0,0]矩阵。



### （5）各种矩阵

这里涉及平移矩阵，缩放矩阵，旋转矩阵，不再做展开（在图形学当中已经有所学习），但做出几点说明：

- 对于平移矩阵，其逆矩阵比较好写，直接平移回来就行；
- 旋转矩阵是正交矩阵，并且多个旋转矩阵之间的串联也是正交的。
- 在复合变换的时候，**正确的顺序是先做缩放，再做旋转，最后做平移，这样大小和位置才是正确的。**

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220601165014512.png" alt="image-20220601165014512" style="zoom:67%;" />



### （6）四元数和欧拉角，旋转顺序

参考文章：

[(12条消息) 【Unity技巧】四元数（Quaternion）和旋转_妈妈说女孩子要自立自强的博客-CSDN博客_quaternion.euler](https://blog.csdn.net/candycat1992/article/details/41254799)

> 在Unity里，欧拉旋转的旋转顺序是Z、X、Y，这在相关的API文档中都有说明，例如Transform.Rotate。其实文档中说得不是非常详细，还有一个细节我们需要明白。如果你仔细想想，就会发现有一个非常重要的东西我们没有说明白，那就是旋转时使用的坐标系。给定一个旋转顺序（例如这里的Z、X、Y），以及它们对应的旋转角度（α，β，r），有两种坐标系可以选择：
>
> - 绕坐标系E下的Z轴旋转α，绕坐标系E下的Y轴旋转β，绕坐标系E下的X轴旋转r，即进行一次旋转时不一起旋转当前坐标系；
> - 绕坐标系E下的Z轴旋转α，绕坐标系E在绕Z轴旋转α后的新坐标系E'下的Y轴旋转β，绕坐标系E'在绕Y轴旋转β后的新坐标系E''下的X轴旋转r， 即在旋转时，把坐标系一起转动；
>
> 很容易知道，这两种选择的结果是不一样的。但如果把它们的旋转顺序颠倒一下，其实结果就会一样。
>
> **说得明白点，在第一种情况下、按ZXY顺序旋转和在第二种情况下、按YXZ顺序旋转是一样的。**证明方法可以参考如下这篇文章：
>
> [欧拉角与万向节死锁 - 魔のkyo的工作室 - IT博客 (cnitblog.com)](http://www.cnitblog.com/luckydmz/archive/2010/09/07/68674.html)
>
> 而Unity文档中说明的旋转顺序指的是在第一种情况下的顺序。
> ————————————————
> 版权声明：本文为CSDN博主「妈妈说女孩子要自立自强」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
> 原文链接：https://blog.csdn.net/candycat1992/article/details/41254799

（具体在作者大佬的博客当中也有进行深入一些的描述）

关于四元数的一些其他知识,后面会进行总结.

<br/>

## 5.坐标空间

### （1）坐标空间的变换

坐标空间会形成一个层次结构——每个坐标空间都是另一个坐标空间的子空间，反过来每个空间都有一个父坐标空间，所以坐标空间变换其实就是父空间和子空间之间点和矢量进行变换。

假设父空间为P，子空间为C，已知父空间中子坐标空间的原点位置以及三个单位坐标轴，此时一般有两种需求：将子坐标空间下表示的点或矢量转换到父坐标空间，或者反过来。接下来进行推导：

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220601174339342.png" alt="image-20220601174339342" style="zoom: 50%;" />

可以看到，下面那个矩阵就是$M_{c->p}$,此时如果想要求出$M_{P->C}$则只需要求解刚才的矩阵的逆矩阵即可。

![image-20220601174702307](E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220601174702307.png)

关于坐标空间的更为详细的介绍，可以参考之前在知乎上总结的博客：

[《Unity Shader入门精要》阅读笔记(5)-详解空间变换 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/434979675)

用书上的一张图来总结，方便复习的时候查看：

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220601180722971.png" alt="image-20220601180722971" style="zoom: 67%;" />

其中，只有观察空间是右手坐标系，其他坐标空间都是左手坐标系。

<br/>

## 6. 法线变换

一般来说，点和绝大部分方向向量都可以使用一个4×4或3×3的矩阵M使其从坐标空间A变到坐标空间B中。但在变化法线的时候，**如果使用同一个变换矩阵，不一定能保证法线的垂直性。**接下来解释一下为什么会有这种现象。

我们先来了解一下另一种方向矢量——**切线** **（tangent）** ，也被称为**切矢量** **（tangent vector）** 。与法线类似，切线往往也是模型顶点携带的一种信息。它通常与纹理空间对齐，而且与法线方向垂直，如图4.47所示。

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220601212512796-16540900374191.png" alt="image-20220601212512796" style="zoom:50%;" />

由于切线是两个顶点之间的差值计算得到的，因此可以直接用变换顶点的变换矩阵来变换切线。这种变化应该如下：
$$
T_B=M_{A->B}T_A
$$
但如果用这个矩阵直接变换法线的话，**可能新的法线方向就不会与表面垂直了。**

解决方案是先求解变换后的切线，再利用切线和法线垂直的性质求解法线。假设变换法线的矩阵为G，则推导过程如下(基于的原理:变换后的切线和变换后的法线也会保持垂直)：

![image-20220601213245998](E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220601213245998.png)

可以发现，如果变换矩阵是正交矩阵，就可以直接使用变换顶点的变换矩阵来变换法线（**此时如果变换只包含旋转变换则符合要求**）；

如果变换只包括旋转和统一缩放（不包括非统一缩放），则可以用统一缩放系数k得到变换矩阵的逆转置矩阵。
$$
(M_{A->B}^T)^{-1}=\frac{1}{k}M_{A->B}
$$
如果变换中包含了非统一变换，那么必须要求解逆矩阵来得到变换法线的矩阵。

<br/>

## 7.Unity Shader的内置变量（数学篇）

这里不做过多的展开，后续整理完后面的部分再回来做补充。

- 注意：CG对矩阵类型中元素的初始化是**按行进行填充的**，Unity Shader也是如此。

剩下的可以查看书中的描述，或者结合后面具体的实例来方便理解。

<br/>



## 8.关于屏幕坐标的解释:ComputeScreenPos/VPOS/WPOS

写Shader的过程中,有时我们希望能获得片元在屏幕上的像素位置.

(1)第一种方式是使用VPOS和WPOS语义,此时的片元着色器只需要这么写:

```glsl
fixed4 frag(float4 sp : VPOS) : SV_Target {
    // 用屏幕坐标除以屏幕分辨率_ScreenParams.xy，得到视口空间中的坐标
    return fixed4(sp.xy/_ScreenParams.xy,0.0,1.0);
}
```

得到的是一张渐变彩色图像:

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220827115325698.png" alt="image-20220827115325698" style="zoom:67%;" />

VPOS/WPOS语义定义的输入是一个float4类型的变量。xy值表示屏幕空间中的像素坐标,比如屏幕分辨率是400×300,那么x的范围就是[0.5,400.5],y同理.对于Unity来说,z分量范围是[0,1],其中近裁剪面的z值为0,远裁剪面z值为1,对于w来说要考虑相机的投影类型,如果是透视投影那么w的范围就是$[1/Near,1/Far]$,如果使用正交投影,那么w分量永远是1.这些值是通过对经过投影矩阵变换后的w分量取倒数得到的.

已知屏幕坐标的话,`sp.xy/_ScreenParams.xy`就是把屏幕坐标归一化,使其左下角是(0,0),右上角是(1,1).



(2)第二种方式则是用Unity提供的`ComputeScreenPos`函数,通常的做法如下:

```glsl
struct vertOut {
    float4 pos:SV_POSITION;
    float4 scrPos : TEXCOORD0;
};

vertOut vert(appdata_base v) {
    vertOut o;
    o.pos = mul (UNITY_MATRIX_MVP, v.vertex);
    // 第一步：把ComputeScreenPos的结果保存到scrPos中
    o.scrPos = ComputeScreenPos(o.pos);
    return o;
}

fixed4 frag(vertOut i) : SV_Target {
    // 第二步：用scrPos.xy除以scrPos.w得到视口空间中的坐标
    float2 wcoord = (i.scrPos.xy/i.scrPos.w);
    return fixed4(wcoord,0.0,1.0);
}
```

这种方法实际上是手动实现了屏幕映射的过程,具体的原理和过程可以参考书p92页左右.



# 第五章 开始Unity Shader学习之旅

## 1.顶点，片元着色器结构

首先复习一下Shader的结构：

```C#
Shader "MyShaderName" {
    Properties {
        // 属性
    }
    SubShader {
        // 针对显卡A的SubShader
        Pass {
            // 设置渲染状态和标签

            // 开始Cg代码片段
            CGPROGRAM
            // 该代码片段的编译指令，例如：
            #pragma vertex vert
            #pragma fragment frag

            // Cg代码写在这里

            ENDCG
            // 其他设置
        }
        // 其他需要的Pass
    }
    SubShader {
        // 针对显卡B的SubShader
    }

    // 上述SubShader都失败后用于回调的Unity Shader
    Fallback "VertexLit"
}
```

### （1）最简单的顶点，片元着色器

```glsl
// Upgrade NOTE: replaced 'mul(UNITY_MATRIX_MVP,*)' with 'UnityObjectToClipPos(*)'

Shader "Chapter 5/SimpleShader"
{
    SubShader
    {
        Pass
		{
			CGPROGRAM
			# pragma vertex vert   // 1
			# pragma fragment frag

			float4 vert(float4 v:POSITION):SV_POSITION
			{
				return UnityObjectToClipPos(v);
			}

			fixed4 frag() : SV_Target
			{
				return fixed4(1.0,1.0,1.0,1.0);
			}

			ENDCG
		}
    }
}

```

注：

- 1.上面注释1这里告诉Unity，哪个函数包含了顶点着色器的代码，哪个函数包含了片元着色器的代码；
- 2.顶点着色器逐顶点执行，输入v包含了这个顶点的位置（通过POSITION语义指定），返回值是一个float4类型变量，是该点在裁剪空间中的位置。POSITION和SV_POSITION不可省略。
- 3.对于片元着色器，SV_Target告诉渲染器，把用户的输出颜色存储到一个渲染目标（render target）中，这里输出到默认的帧缓存中。

<br/>

### （2）增加结构体，丰富Shader

```glsl
// Upgrade NOTE: replaced 'mul(UNITY_MATRIX_MVP,*)' with 'UnityObjectToClipPos(*)'

Shader "Chapter 5/SimpleShader"
{
	Properties
	{
		_Color("Color Tint",Color) = (1.0,1.0,1.0,1.0)
	}

		SubShader
	{
		Pass
		{
			CGPROGRAM
			# pragma vertex vert
			# pragma fragment frag

			//在CG代码中,我们需要定义一个与属性名称和类型都匹配的变量
			fixed4 _Color;

			// 1.使用一个结构体来定义顶点着色器的输入  
			struct a2v
			{
				float4 vertex:POSITION; //用模型空间顶点坐标填充vertex
				float3 normal:NORMAL; //用模型空间法线方向填充normal
				float4 texcoord:TEXCOORD0; //用模型的第一套纹理坐标填充texcoord
			};

			//2.顶点着色器和片元着色器通信
			struct v2f
			{
				float4 pos:SV_POSITION; //SV_POSITION告诉片元着色器,pos里包含了顶点在裁剪空间中的位置信息
				fixed3 color : COLOR0;//COLOR0可以用于存储颜色信息

			};

			//注意,这里输入变量去掉了POSITION语义,而是改成了刚才定义的a2v
			//同时去除了SV_POSITION语义(对于高版本来说),因为这里的参数是结构体所以不能带有顶点相关的语义
			v2f vert(a2v v)
			{
				v2f o;
				o.pos = UnityObjectToClipPos(v.vertex);
				o.color = v.normal*0.5 + fixed3(0.5, 0.5, 0.5);//使用技巧,把法线(-1,1)映射到颜色(0,1)
				return o;
			}

			fixed4 frag(v2f i) : SV_Target
			{
				fixed3 c = i.color;
				//使用_Color属性来控制输出颜色
				c *= _Color.rgb;
				return fixed4(c,1.0);
			}

			ENDCG
		}
    }
}

```

- 1.填充到POSITION,TANGENT,NORMAL这些语义中的数据是由材质的Mesh Renderer所提供的。在每帧调用Draw Call的时候，Mesh Renderer组件会把其负责渲染的模型数据发送给Unity Shader。一个模型包含一组三角面片（每个由3个顶点组成，每个顶点有坐标，法线，切线等数据），因此**顶点着色器可以访问顶点的这些数据。**

效果如下：

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220601221803053.png" alt="image-20220601221803053" style="zoom:67%;" />

（注意，此时旋转这个球体，会发现上面不完全是绿色的，会发生变化，这也验证了输入到顶点着色器的信息是模型空间的信息）

下面的表格比较重要，后面对于Cg变量的类型要掌握的比较熟练：

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220816213158890.png" alt="image-20220816213158890" style="zoom:67%;" />

**假如变量前有uniform是怎么回事?**

 uniform关键词是Cg中修饰变量和参数的一种修饰词,仅仅用于提供一些关于该变量的初始值是如何指定和存储的相关信息.在Unity Shader当中,**uniform关键词可以省略.**

<br/>

## 2.Unity提供的内置文件和变量

### （1）内置的包含文件

 **包含文件** **（include file）** ，是类似于C++中头文件的一种文件。在Unity中，它们的文件后缀是.cginc。在编写Shader时，我们可以使用#include指令把这些文件包含进来,示例如下:

```csharp
CGPROGRAM
//...
#include "UnityCG.cginc"
//...
ENDCG
```

### （2）关于内置的包含文件下载

 首先来到官网[Unity最新版本下载-Unity稳定版本 | Unity中国官网](https://link.zhihu.com/?target=https%3A//unity.cn/releases%23668d935442ef)

 选择一个版本并点击以下内容即可下载.

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220816213254228.png" alt="image-20220816213254228" style="zoom: 67%;" />

下载之后解压即可看到很多的cginc后缀文件,这些就是一些包含文件,我们**可以查看其源代码**.

 解压后的文件夹中有以下内容:

![image-20220816213645443](E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220816213645443.png)

其中:

- CGIncludes文件夹中包含了所有的内置包含文件;  DefaultResources文件夹中包含了一些内置组件或功能所需要的Unity Shader，例如一些GUI元素使用的Shader;
- DefaultResourcesExtra则包含了所有Unity中内置的Unity Shader；
- Editor文件夹用于定义Standard Shader所用的材质面板。

 具体的一些内容后续有机会会进一步进行介绍.

 **目前来说,我们只讨论CGIncludes当中的内容.**

 下表当中给出了CGIncludes文件夹中比较重要的内容:

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220601222442339.png" alt="image-20220601222442339" style="zoom:67%;" />

 **UnityCG.cginc是我们最常接触的一个包含文件。**该文件会提供很多的结构体和函数,大大简化了我们的代码量.以下给出了一些结构体名称和包含的变量.

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220601222645721.png" alt="image-20220601222645721" style="zoom:67%;" />

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220601222700944.png" alt="image-20220601222700944" style="zoom: 80%;" />

**后续或许会在文章中对于该文件的各种函数进行解读,这样会更有助于理解Unity Shader的一些逻辑.**在后面的Shader书写当中我们可能会频繁使用这些函数.

<br/>

## 3.Unity提供的CG/HLSL语义

语义实际上就是一个赋给Shader输入和输出的字符串，这个字符串表达了这个参数的含义。通俗地讲，这些语义可以让Shader知道从哪里读取数据，并把数据输出到哪里.例如上文所用到的**POSITION,SV_TARGET等**.

 通常情况下，这些**输入输出变量并不需要有特别的意义**，也就是说，我们可以自行决定这些变量的用途。例如在上面的代码中，顶点着色器的输出结构体中我们用*COLOR* 0语义去描述color变量。color变量本身存储了什么，Shader流水线并不关心。但为了方便起见,**有些语义会有特别的含义规定**.

 **需要注意的是，有的时候即便语义的名称一样，如果出现的位置不同，含义也不同。**

> 例如，TEXCOORD0既可以用于描述顶点着色器的输入结构体a2v，也可用于描述输出结构体v2f。
> 但在输入结构体a2v中，TEXCOORD0有特别的含义，即把模型的第一组纹理坐标存储在该变量中，
> 而在输出结构体v2f中，TEXCOORD0修饰的变量含义就可以由我们来决定。

<br/>

### （1）关于系统数值语义

 这类语义是以SV开头的，SV代表的含义就是**系统数值** **（system-value）** 。这些语义在渲染流水线中有**特殊的含义。**例如渲染引擎会把用**SV_POSITION** 修饰的变量经过光栅化后显示在屏幕上(用**SV_POSITION** 语义去修饰顶点着色器的输出变量pos，那么就表示pos包含了可用于光栅化的变换后的顶点坐标（即齐次裁剪空间中的坐标）)。对于某些平台来说SV并不是必须的,但是为了让我们的程序有更好的跨平台性**推荐加上SV**.

具体的语义还是要到对应的项目当中去加深记忆。

### （2）如何定义复杂的变量类型

 由于一个语义所能处理的浮点型数据数量有限,因此有的时候如果我们想要定义矩阵类型，如float3×3、float4×4等变量就需要使用更多的空间.此时一种可行的方案**就是用四个float4来代替float4×4**,将其进行适当的拆分.

<br/>

## 4.Debug！

### （1）利用Viusal Studio本身(主要)

 参考官网的文章:[Unity - Manual: Debugging shaders using Visual Studio (unity3d.com)](https://link.zhihu.com/?target=https%3A//docs.unity3d.com/Manual/SL-DebuggingD3D11ShadersWithVS.html)

(https://blog.csdn.net/weixin_46840974/article/details/123962713)

这里我们主要按这种方式进行介绍,首先按照上述博客的教程配置环境(看到配置环境部分即可),剩下的如何打包exe文件并用VS图形调试打开参考如下链接:

[(12条消息) VS2017下用Graphics Debugger调试UnityShader_带帯大师兄的博客-CSDN博客](https://blog.csdn.net/qq_29523119/article/details/78409811)

出现下面的画面说明环境已经配置成功了,后面的Shader可以基于此进行调试。

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220602202946364.png" alt="image-20220602202946364" style="zoom: 67%;" />

可以对单个像素进行调试观察，在后面的Shader学习当中会进行展开，这里只是简单介绍一下：

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220602203900980.png" alt="image-20220602203900980" style="zoom:80%;" />

### （2）使用RenderDoc

 首先,下载并安装RenderDoc.[RenderDoc](https://link.zhihu.com/?target=https%3A//renderdoc.org/)

 关于这个工具的使用可以参考如下文章:

 [Renderdoc快速入门 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/404576672)

[UnityTips 之 Shader调试工具RenderDoc - 简书 (jianshu.com)](https://link.zhihu.com/?target=https%3A//www.jianshu.com/p/7f621756347d)

 在后续的文章中会专门对这个工具进行介绍,敬请期待.

### （3）利用Unity内置的**Frame Debugger**

<br/>

## 5.小心：渲染平台的差异

### （1）渲染纹理差异

由于OpenGL和DirectX的坐标系有所不同，因此可能会出现渲染纹理翻转的情况。Unity正常会为我们处理反转的问题，除非我们开启了抗锯齿（Edit->Project Settings->Quality->Anti Aliasing中开启），有时就需要手动翻转一下纹理（特别多张纹理的时候，后面十二章会有介绍）

 <br/>

## 6.Shader整洁之道

### （1）float，half还是fixed

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220602204821706.png" alt="image-20220602204821706" style="zoom:67%;" />

- 大多数现代桌面GPU会把所有计算都按最高的浮点精度计算，所以上述三个基本是等价的；
- 移动平台的GPU上，会有不同的精度范围和运算速度，所以在移动平台上要验证shader
- 实际上，可以用fixed类型存储颜色和单位矢量，更大范围采用half，最后再考虑float



### （2）其他

- 避免无意义的大量计算，慎用分支和循环语句，如果可以的话尽量把计算向流水线上端移动（比如把放到片元着色器的计算放到顶点着色器中，或者直接在CPU中预计算，再把结果传给Shader），如果一定要用分支语句则建议：
  - 判断语句中的条件变量最好是常数
  - 分支中的操作指令数尽可能少
  - 分支的嵌套层数尽可能少

- 不要除以0

<br/>

# 第六章 Unity中的基础光照

## 1.一些基础概念

### （1）光源

光线照射在平面上的时候需要注意这个角度，辐照度要乘以法线方向n与入射光线l的夹角cosθ。（这是因为辐照度是和照射到物体表面时光线之间的距离d/cosθ成反比的）

参考下图：

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220602210632800.png" alt="image-20220602210632800" style="zoom:67%;" />

### （2）反射，折射

具体的漫反射，镜面反射等模型在这里就不做过多展开了（回忆Lambert 模型，Phong模型，Blinn-Phong模型），后面Shader里会实现。

### （3）BRDF

BRDF回答的问题是，当光线从某个方向照射到一个表面时，有多少光线被反射？反射的方向有哪些？

当给定模型的一个点时，BRDF包含了对该点外观的完整的描述。图形学中往往用一个数学公式来表示，并提供调整材质属性的参数。通俗来讲，当给定入射光线的方向和辐照度(对应games101的irrandiance)后，BRDF可以给出在某个出射方向上的光照能量分布。本章涉及的BRDF都是对真实场景进行理想化和简化后的模型，也就是说，它们并不能真实地反映物体和光线之间的交互，这些光照模型被称为是经验模型。（但在有些场合仍然比较实用）

（**后续将会讨论基于物理的BRDF模型**）

<br/>

## 2.标准光照模型

标准光照模型只关注直接光照（也就是那些直接从光源发射出来照射到物体表面后，经过物体表面的一次反射直接进入摄像机的光线）。包括：

- 自发光：可以直接由光源发射进入摄像机，而不需要经过任何物体的反射，直接使用该材质的自发光颜色即可。
- 高光反射

如果使用Phong模型的话，示意图如下：

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220602212629913.png" alt="image-20220602212629913" style="zoom:80%;" />

那么这里需要求出反射向量r：$r=2(n·l)n-l$

此时高光反射的部分为：$\large c_{specular}=(c_{light}·m_{specular})max(0,v·r)^{m_{gloss}}$

对于后面的高光系数，其值越大，材质上的”亮点“就越小。



如果采用Blinn-Phong模型的话，那么就要使用半程向量h，由v和l来进行计算，此时：

$\large c_{specular}=(c_{light}·m_{specular})max(0,n·h)^{m_{gloss}}$，其中$\large h=\frac{v+l}{|v+l|}$

硬件实现的时候，如果摄像机和光源距离模型足够远的话，Blinn要好于Phong，因为此时可以认为v和l都是定值，因此h是常量。**不过两种都是经验模型，不过一些情况下，Blinn更符合实验结果**



- 漫反射
  - 兰伯特定律：$c_{diffuse}=(c_{light}·m_{diffuse})max(0,n·l)$，其中clight是光源颜色，l是指向光源的单位矢量cdiffuse是反射光的强度；
- 环境光（描述其他所有的间接光照，在这个模型中往往直接使用一个全局变量来表示）

### （1）局限性

很多重要的物理现象无法用Blinn-Phong模型表现出来，比如菲涅尔反射。

并且，Blinn-Phong模型是各向同性的（也就是说，固定视角和光源转动这个表面时，反射不会有任何改变），但比如毛发这种各向异性的表面就难以处理了。

<br/>

## 3.逐像素还是逐顶点

逐顶点光照也叫做Gouraud shading，计算量要小于逐像素光照，但会产生比较明显的棱角现象（因为插值的时候有非线性的计算）。逐像素光照则是以每个像素为基础，得到它的法线，这种技术被称之为Phong着色。

<br/>

## 4.Unity中的环境光和自发光

在Unity中，场景中的**环境光**可以在*Window ->Rendering-> Lighting ->Environment->Ambient Source/Ambient Color/Ambient Intensity* 中控制，如图所示。在Shader中，我们只需要通过Unity的内置变量UNITY_LIGHTMODEL_AMBIENT就可以得到环境光的颜色和强度信息。

![image-20220602214242258](E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220602214242258.png)

绝大多数物体没有自发光属性,因此本书大部分Shader都没计算自发光。如果要计算的话只需要再片元着色器输出最后的颜色之前添加材质的自发光颜色到输出颜色即可。

<br/>

## 5.Unity Shader——漫反射光照模型

补充函数：`saturate(x)`

这个函数会把x截取到[0,1]范围里面，如果x是一个矢量则会对每一个分量都这样做。

### （1）逐顶点光照

这个得到的结果是类似Gauraud着色的效果，可能不会太好。需要保证所有的光照运算都是在同一个坐标系下进行的（这里是世界坐标系，所以需要把法线从模型坐标系转为世界坐标系（通过左乘$M_{A->B}$的逆转置矩阵来实现，前面有说，具体实现有一个巧妙方法，一会会说））

Shader代码如下：

```glsl
Shader "Chapter 6/DiffuseVertexLevel"
{
    Properties
    {
        _Diffuse("Diffuse",Color)=(1,1,1,1)
    }
    SubShader
    {
        Pass
        {
            Tags{"LightMode"="ForwardBase"}
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #include "Lighting.cginc"
            
            fixed4 _Diffuse;
            
            struct a2v
            {
                float4 vertex:POSITION;
                float3 normal:NORMAL;
            };
            
            struct v2f
            {
                float4 pos:SV_POSITION;
                fixed3 color:COLOR;
            };

            v2f vert(a2v v)
            {
                v2f o;
                o.pos=UnityObjectToClipPos(v.vertex);
                fixed3 ambient=UNITY_LIGHTMODEL_AMBIENT.xyz;
                
                fixed3 worldNormal=normalize(mul(v.normal,(float3x3)unity_WorldToObject));
                fixed3 worldLight=normalize(_WorldSpaceLightPos0.xyz);//如果有不止一个光源则不能这么简单获取,见后
                fixed3 diffuse=_LightColor0.rgb*_Diffuse.rgb*saturate(dot(worldNormal,worldLight));
                o.color=ambient+diffuse;
                return o;
            }   
            
            fixed4 frag(v2f i):SV_TARGET
            {
                return fixed4(i.color,1.0);
            }
            ENDCG
        }

    }
    FallBack "Diffuse"
}

```

此时,我们假设场景只有一个光源且光源的类型是平行光,所以才可以使用_WorldSpaceLightPos0,否则不可以使用。另外，关于以下的介绍：

```c#
fixed3 worldNormal=normalize(mul(v.normal,(float3x3)_World2Object));
```

首先来看mul函数的具体接口:

| mul(M, N) | 计算两个矩阵相乘，如果 M 为 AxB 阶矩阵，N 为 BxC 阶矩阵，则返回 AxC 阶矩阵。下面两个函数为其重载函数 |
| --------- | ------------------------------------------------------------ |
| mul(M, v) | 计算矩阵和向量相乘                                           |
| mul(v, M) | 计算向量和矩阵相乘                                           |

上述代码可以这样理解:

> 从前文我们得知了对法线的空间变换矩阵应该是$(M_{A->B}^T)^{-1}=(M_{A->B}^{-1})^T$(如果是正交矩阵的话就是$M_{A->B}$),而`_World2Object`矩阵就是模型空间到世界空间的逆矩阵,也就是$(M_{A->B}^{-1})$,又有:$(AB)^T=B^TA^T$,所以上述代码其实是实现了$A^T(M_{A->B}^{-1})$,在mul函数当中会被转换为正确的显示形式.(**一句话总结,就是行向量右乘矩阵A在mul函数中等同于列向量左乘A的转置**)

以上的这种技巧后面也会有相关应用。

效果如下：

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220603172718346.png" alt="image-20220603172718346" style="zoom:50%;" />

可以看到，背光面和向光面的交接处有明显的锯齿，因此我们要把模型修改为逐像素的漫反射效果。

### （2）逐像素光照

与上述不同之处在于，把漫反射的计算过程放在片元着色器当中去进行，顶点着色器只负责传输数据和做必要的运算。

```glsl
Shader "Chapter 6/DiffusePixelLevelMat"
{
    Properties
    {
        _Diffuse("Diffuse",Color)=(1,1,1,1)
    }
    SubShader
    {
        Pass
        {
            Tags{"LightMode"="ForwardBase"}
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #include "Lighting.cginc"
            
            fixed4 _Diffuse;
            
            struct a2v
            {
                float4 vertex:POSITION;
                float3 normal:NORMAL;
            };
            
            struct v2f
            {
                float4 pos:SV_POSITION;
                float3 worldNormal:TEXCOORD0;
            };

            v2f vert(a2v v)
            {
                v2f o;
                o.pos=UnityObjectToClipPos(v.vertex);
                o.worldNormal=normalize(mul(v.normal,(float3x3)unity_WorldToObject));
                return o;
            }   
            
            fixed4 frag(v2f i):SV_TARGET
            {
                fixed3 ambient=UNITY_LIGHTMODEL_AMBIENT.xyz;
                fixed3 worldNormal=i.worldNormal;
                fixed3 worldLightDir=normalize(_WorldSpaceLightPos0.xyz);

                fixed3 diffuse=_LightColor0.rgb*_Diffuse.rgb*saturate(dot(worldNormal,worldLightDir));
                fixed3 color=ambient+diffuse;
                return fixed4(color,1.0); 
            }
            ENDCG
        }

    }
    FallBack "Diffuse"
}

```

将逐像素光照和逐顶点光照进行对比，可以看到逐像素光照的效果提升很显著：

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220603174211374.png" alt="image-20220603174211374" style="zoom:80%;" />

逐像素光照可以得到更加平滑的光照效果。但是，即便使用了逐像素漫反射光照，有一个问题仍然存在。在光照无法到达的区域，模型的外观通常是全黑的，没有任何明暗变化，这会使模型的背光区域看起来就像一个平面一样，失去了模型细节表现。实际上我们可以通过添加环境光来得到非全黑的效果，但即便这样仍然无法解决背光面明暗一样的缺点。为此，有一种改善技术被提出来，这就是**半兰伯特** **（Half Lambert）** **光照模型** 。

### （3）半兰伯特模型

Valve公司在开发游戏《半条命》时提出了一种技术，由于该技术是在原兰伯特光照模型的基础上进行了一个简单的修改，因此被称为**半兰伯特光照模型** 。

广义上半兰伯特模型的公式如下：
$$
\large c_{diffuse}=(c_{light}·m_{diffuse})(\alpha(n,l)+\beta)
$$
这里没有用到传统的max，而是用α和β来控制漫反射的效果，通常来说两者设置为0.5即可，这样可以把点乘的结果从[-1,1]的范围映射为[0,1],此时就没必要去取max操作了。新建Shader，仅需对片元着色器进行如下修改即可：

```glsl
			fixed4 frag(v2f i):SV_TARGET
            {
                fixed3 ambient=UNITY_LIGHTMODEL_AMBIENT.xyz;
                fixed3 worldNormal=i.worldNormal;
                fixed3 worldLightDir=normalize(_WorldSpaceLightPos0.xyz);

                fixed3 diffuse=_LightColor0.rgb*_Diffuse.rgb*(0.5*dot(worldNormal,worldLightDir)+0.5);
                fixed3 color=ambient+diffuse;
                return fixed4(color,1.0); 
            }
```

三者对比如下：

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220603175111446.png" alt="image-20220603175111446" style="zoom:67%;" />

能明显感觉到，最左面的半兰伯特模型整体变明亮了许多，这对于游戏来说是很有意义的（避免模型纯黑色，影响观感）。

<br/>

## 6.Unity Shader——高光反射模型

相关函数:

CG提供了计算反射方向的函数`reflect(i,n)`

其中i为入射方向,n为法线方向,输入参数和返回值的关系如下图:

![image-20220603220049178](E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220603220049178.png)

### （1）逐顶点光照

还是和之前一样，现在顶点着色器里写逐顶点光照的逻辑，直接上代码（为了方便观察效果，这里可以把天空盒去掉）。在这里Diffuse项还是比较需要的，否则会高亮显示某个区域，其他地方可能都是黑的。

![image-20220603224200187](E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220603224200187.png)

此时我们发现，高光效果有较大的问题（不够平滑），这是因为高光反射部分的计算是非线性的(有一个高光系数的存在)，而在顶点着色器中计算光照再进行插值的过程则是线性的，这就破坏了原计算的非线性关系，因此我们采用逐像素的方法来计算高光反射。

逐顶点光照的代码如下(仅核心代码):

```glsl
v2f vert(a2v v)
{
    v2f o;
    o.pos=UnityObjectToClipPos(v.vertex);
    fixed3 ambient=UNITY_LIGHTMODEL_AMBIENT.xyz;
    fixed3 worldNormal=normalize(mul(v.normal,(float3x3)unity_WorldToObject));
    fixed3 worldLightDir=normalize(_WorldSpaceLightPos0.xyz);//如果有不止一个光源则不能这么简单获取,见后
    fixed3 diffuse=_LightColor0.rgb*_Diffuse.rgb*saturate(dot(worldNormal,worldLightDir));
                
    fixed3 reflectDir=normalize(reflect(-worldLightDir,worldNormal));
    fixed3 viewDir=normalize(_WorldSpaceCameraPos.xyz-mul(unity_ObjectToWorld,v.vertex).xyz);
                
    fixed3 specular=_LightColor0.rgb*_Specular.rgb*pow(saturate(dot(reflectDir,viewDir)),_Gloss);
    o.color=ambient+diffuse+specular;

    return o;
}

fixed4 frag(v2f i):SV_TARGET
{
     return fixed4(i.color,1.0);
}
```

<br/>

### （2）逐像素光照

这回在片元着色器当中去计算光照模型，修改Shader如下：

```glsl
Shader "Chapter 6/SpecularPixelLevel"
{
    Properties
    {
        _Diffuse("Diffuse",Color)=(1,1,1,1)
        _Specular("Specular",Color)=(1,1,1,1)
        _Gloss("Gloss",Range(8.0,256))=20
    }
    SubShader
    {
        Pass
        {
            Tags{"LightMode"="ForwardBase"}
            CGPROGRAM
            # pragma vertex vert
            # pragma fragment frag

            #include "Lighting.cginc"
            fixed4 _Diffuse;
            fixed4 _Specular;
            float _Gloss;
            
            struct a2v
            {
                float4 vertex:POSITION;
                float3 normal:NORMAL;
            };
            
            struct v2f
            {
                float4 pos:SV_POSITION;
                fixed3 worldNormal:TEXCOORD0;
                float3 worldPos:TEXCOORD1;
                
            };
            
            v2f vert(a2v v)
            {
                v2f o;
                o.pos=UnityObjectToClipPos(v.vertex);
                o.worldNormal=mul(v.normal,(float3x3)unity_WorldToObject);
                
                o.worldPos=mul(unity_ObjectToWorld,v.vertex).xyz; //后面计算视角方向的时候要用到物体的世界坐标

                return o;
            }

            fixed4 frag(v2f i):SV_TARGET
            {
                fixed3 ambient=UNITY_LIGHTMODEL_AMBIENT.xyz;
                fixed3 worldNormal=normalize(i.worldNormal);
                fixed3 worldLightDir=normalize(_WorldSpaceLightPos0.xyz);//如果有不止一个光源则不能这么简单获取,见后
                fixed3 diffuse=_LightColor0.rgb*_Diffuse.rgb*saturate(dot(worldNormal,worldLightDir));
                
                fixed3 reflectDir=normalize(reflect(-worldLightDir,worldNormal));
                fixed3 viewDir=normalize(_WorldSpaceCameraPos.xyz-i.worldPos.xyz);
                
                fixed3 specular=_LightColor0.rgb*_Specular.rgb*pow(saturate(dot(reflectDir,viewDir)),_Gloss);
                
                return fixed4(ambient+diffuse+specular,1.0);
            }
            ENDCG
        }

    }
    FallBack "Specular"
}

```

注意到渲染结果的高光部分就比较平滑，并且通过拖动右侧的_Gloss的数值我们也会发现高光系数越大，这种高光就越集中。

![image-20220603225506697](E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220603225506697.png)

### （3）Blinn-Phong模型

与Phong模型的区别在于，Blinn-Phong模型采用了半程向量，因此只需要修改片元着色器中的一部分即可：

```glsl
			fixed4 frag(v2f i):SV_TARGET
            {
                fixed3 ambient=UNITY_LIGHTMODEL_AMBIENT.xyz;
                fixed3 worldNormal=normalize(i.worldNormal);
                fixed3 worldLightDir=normalize(_WorldSpaceLightPos0.xyz);//如果有不止一个光源则不能这么简单获取,见后
                fixed3 diffuse=_LightColor0.rgb*_Diffuse.rgb*saturate(dot(worldNormal,worldLightDir));
                
                fixed3 viewDir=normalize(_WorldSpaceCameraPos.xyz-i.worldPos.xyz);
                
                fixed3 halfDir=normalize(worldLightDir+viewDir);
        
                fixed3 specular=_LightColor0.rgb*_Specular.rgb*pow(max(0,dot(worldNormal,halfDir)),_Gloss);
                
                return fixed4(ambient+diffuse+specular,1.0);
            }
```

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220603230214566.png" alt="image-20220603230214566" style="zoom:67%;" />

看上去，Blinn-Phong光照模型的高光反射部分看起来更大，更亮一些。实际渲染的时候我们大多数情况都会选择Blinn-Phong，但其和Phong都是经验模型。在一些情况下，Blinn-Phong模型更符合实验结果。

<br/>

## 7.使用Unity内置的函数

像之前的代码中,我们获取了光源方向和视角方向,但都有局限性(不能处理复杂光源),因此,我们要用到Unity的一些帮助计算的函数。

|                   函 数 名                    |                            描 述                             |
| :-------------------------------------------: | :----------------------------------------------------------: |
|      float3 WorldSpaceViewDir (float4 v)      | 输入一个模型空间中的顶点位置，返回世界空间中从该点到摄像机的观察方向。内部实现使用了UnityWorldSpaceViewDir函数 |
|   float3 UnityWorldSpaceViewDir (float4 v)    | 输入一个世界空间中的顶点位置，返回世界空间中从该点到摄像机的观察方向 |
|       float3 ObjSpaceViewDir (float4 v)       | 输入一个模型空间中的顶点位置，返回模型空间中从该点到摄像机的观察方向 |
|     float3 WorldSpaceLightDir (float4 v)      | **仅可用于前向渲染中** 。输入一个模型空间中的顶点位置，返回世界空间中从该点到光源的光照方向。内部实现使用了UnityWorldSpaceLightDir函数。没有被归一化 |
|   float3 UnityWorldSpaceLightDir (float4 v)   | **仅可用于前向渲染中** 。输入一个世界空间中的顶点位置，返回世界空间中从该点到光源的光照方向。没有被归一化 |
|      float3 ObjSpaceLightDir (float4 v)       | **仅可用于前向渲染中** 。输入一个模型空间中的顶点位置，返回模型空间中从该点到光源的光照方向。没有被归一化 |
| float3 UnityObjectToWorldNormal (float3 norm) |             把法线方向从模型空间转换到世界空间中             |
|   float3 UnityObjectToWorldDir (float3 dir)   |             把方向矢量从模型空间变换到世界空间中             |
|   float3 UnityWorldToObjectDir(float3 dir)    |             把方向矢量从世界空间变换到模型空间中             |

注意：

（1）Unity不保证为我们做了归一化，所以使用前最好进行一步归一化的操作；

（2）计算光源方向的3个函数：WorldSpaceLightDir、UnityWorldSpaceLightDir和ObjSpaceLightDir，稍微复杂一些，这是因为，Unity帮我们处理了不同种类光源的情况。**需要注意的是** ，**这3个函数仅可用于前向渲染**（关于什么是前向渲染会在9.1节中讲到）。这是因为只有在前向渲染时，这3个函数里使用的内置变量_WorldSpaceLightPos0等才会被正确赋值。

接下来，我们稍微使用一下内置的函数来改写Blinn-Phong模型，将顶点着色器和片元着色器修改如下：

 ```glsl
 			v2f vert(a2v v)
             {
                 v2f o;
                 o.pos=UnityObjectToClipPos(v.vertex);
                 o.worldNormal=UnityObjectToWorldNormal(v.normal); //使用内置的函数来解决问题
                 
                 o.worldPos=mul(unity_ObjectToWorld,v.vertex).xyz; //后面计算视角方向的时候要用到物体的世界坐标
 
                 return o;
             }
 
             fixed4 frag(v2f i):SV_TARGET
             {
                 fixed3 ambient=UNITY_LIGHTMODEL_AMBIENT.xyz;
                 fixed3 worldNormal=normalize(i.worldNormal);
                 fixed3 worldLightDir=normalize(UnityWorldSpaceLightDir(i.worldPos));//使用内置函数使得Shader可以适应各种光源
                 fixed3 diffuse=_LightColor0.rgb*_Diffuse.rgb*saturate(dot(worldNormal,worldLightDir));
                 
                 fixed3 viewDir=normalize(UnityWorldSpaceViewDir(i.worldPos));//使用内置函数
                 
                 fixed3 halfDir=normalize(worldLightDir+viewDir);
         
                 fixed3 specular=_LightColor0.rgb*_Specular.rgb*pow(max(0,dot(worldNormal,halfDir)),_Gloss);
                 
                 return fixed4(ambient+diffuse+specular,1.0);
             }
 ```

关于Unity中各种光源的使用，参考第九章。

<br/>

# 第七章 基础纹理

美术人员建模的时候，通常会在建模软件中利用纹理展开技术把**纹理映射坐标** **（texture-mapping coordinates）** 存储在每个顶点上。纹理映射坐标定义了该顶点在纹理中对应的2D坐标。通常，这些坐标使用一个二维变量(u, v)来表示，其中u是横向坐标，而v是纵向坐标。因此，纹理映射坐标也被称为UV坐标。

尽管纹理大小多样，但UV坐标的范围往往被归一化为[0,1]之间，注意采样的时候不一定要在[0,1]范围内采样，而这就涉及到后面会说的纹理平铺模式，他将决定引擎在遇到[0,1]范围外的纹理的时候会怎么做。

Unity使用的纹理空间是符合OpenGL传统的，所以以左下角作为原点。

需要说明的是，这章所实现的Shader纹理并不能用于实际的项目（因为直接使用的话会缺少阴影，光照衰减等效果），后面会有能够真正使用的Shader。

## 1. 单张纹理

通常会使用一张纹理来代替物体的漫反射颜色。代码如下：

```glsl
Shader "Chapter 7/SingleTexture"
{
    Properties
    {
        _Color("Color Tint",Color)=(1,1,1,1)
        _MainTex ("Texture", 2D) = "white" {}
        _Specular("Specular",Color)=(1,1,1,1)
        _Gloss("Gloss",Range(8.0,256))=20
    }
    SubShader
    {
        Pass
        {
            Tags{"LightMode"="ForwardBase"}
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            
            #include "Lighting.cginc"
            fixed4 _Color;
            sampler2D _MainTex;
            float4 _MainTex_ST; //不要忘了声明这个变量,ST表示缩放偏移
            fixed4 _Specular;
            float _Gloss;
            
            struct a2v
            {
                float4 vertex:POSITION;
                float3 normal:NORMAL;
                float4 texcoord:TEXCOORD0;
            };

            struct v2f
            {
                //在片元着色器当中去做光照计算
                float4 pos:SV_POSITION;
                float3 worldNormal:TEXCOORD0;
                float3 worldPos:TEXCOORD1;
                float2 uv : TEXCOORD2; 
            };

            v2f vert (a2v v)
            {
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);
                o.worldNormal=UnityObjectToWorldNormal(v.normal);
                o.worldPos=mul(unity_ObjectToWorld,v.vertex).xyz;
                o.uv = TRANSFORM_TEX(v.texcoord, _MainTex);
                //上一句等同于下面的写法:
                //o.uv = v.texcoord.xy * _MainTex_ST.xy + _MainTex_ST.zw;
                return o;
            }

            fixed4 frag (v2f i) : SV_Target
            {
                fixed3 worldNormal=normalize(i.worldNormal);
                fixed3 worldLightDir=normalize(UnityWorldSpaceLightDir(i.worldPos));//使用内置函数使得Shader可以适应各种光源
                fixed3 albedo=tex2D(_MainTex,i.uv).rgb*_Color.rgb;
                fixed3 ambient=UNITY_LIGHTMODEL_AMBIENT.xyz*albedo;
                
                fixed3 diffuse=_LightColor0.rgb*albedo*max(0,dot(worldNormal,worldLightDir));
                fixed3 viewDir=normalize(UnityWorldSpaceViewDir(i.worldPos));//使用内置函数
                
                fixed3 halfDir=normalize(worldLightDir+viewDir);
        
                fixed3 specular=_LightColor0.rgb*_Specular.rgb*pow(max(0,dot(worldNormal,halfDir)),_Gloss);
                
                return fixed4(ambient+diffuse+specular,1.0);
            }
            ENDCG
        }
    }
    FallBack "Specular"
}

```

其中，_MainTex_ST是一个float4变量，存放纹理的缩放值和偏移值，其中这个变量的xy存放缩放值，zw存放偏移值。可以在属性面板当中调整，如下图：

![image-20220604160514641](E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220604160514641.png)

使用作者提供的图片作为MainTex，最终效果如下：

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220604162510665.png" alt="image-20220604162510665" style="zoom:67%;" />

## 2. 纹理的属性

![image-20220604162738549](E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220604162738549.png)

当我们选中一张纹理图时，有一些属性需要注意，并且必要时要进行调节：

（1）Texture Type在高版本的Unity当中，只需要采用Default即可（如果后面用normal map，cubemap的话再去调整）；

（2）Alpha Source不要选择grayscale，这个后面会介绍；（如果选择的话那么透明通道的值将会由每个像素的灰度值生成）

（3）Wrap Mode非常重要，有两种主要模式（Repeat和Clamp），Repeat模式使得纹理不断重复，而在Clamp模式下，如果纹理坐标大于1，就会截取到1，如果小于0，就会截取到0，两者的对比如下：

![image-20220604163910402](E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220604163910402.png)

（4）Filter Mode属性决定了当纹理由于变换而产生拉伸的时候将会采用什么滤波模式。支持三种模式：Point，Bilinear和Trilinear，得到的滤波效果依次提升，但需要耗费的性能也依次增大。（其实就是图形学当中的最近邻插值，双线性插值和三线性插值）

效果如下：

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220604164139824.png" alt="image-20220604164139824" style="zoom: 50%;" />

（最近邻插值有一点像素风格的意味）

（5）Generate Mipmap是说是否要开启多级渐远纹理技术（图形学有学习相关内容）。我们还可以选择生成多级渐远纹理时是否要使用线性空间（用于伽马矫正），以及选择何种滤波器。

Mipmap是一种典型的用空间换时间的策略，一般会多占用33%左右的内存空间。

下图是三种插值（同时使用了Mipmap之后的观察结果）：

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220604164624657.png" alt="image-20220604164624657" style="zoom:67%;" />

（注：Trilinear滤波几乎是和Bilinear一样的，只是Trilinear还会在多级渐远纹理之间进行混合。如果一张纹理没有使用多级渐远纹理技术，那么Trilinear得到的结果是和Bilinear就一样的。通常，我们会选择Bilinear滤波模式。）

（6）纹理的最大尺寸和纹理模式：

Unity允许我们为不同目标平台选择不容的分辨率，如果导入的纹理大小超过了Max Size的设置值，那么Unity会把该纹理缩放为这个最大分辨率。理论上导入的纹理可以是非正方形的，但是长宽的大小应该是2的幂。（如果使用了非2的幂大小纹理，Non Power of Two，NPOT），就会占用更多的内存空间，不是很建议。

Format决定用什么格式来存储该纹理。使用的纹理格式精度越高，占用的内存越大，但得到的效果也越好。对于一些不需要很高精度的纹理（比如用于漫反射颜色的纹理），尽量用压缩格式。

<br/>

## 3.凹凸映射

原理：**使用一张纹理来修改模型表面的发现，以便为模型提供更多的细节。**不会真的改变模型的顶点位置，只是看起来凹凸不平而已。

（1）方法1：采用**高度纹理**模拟表面位移。

（2）方法2：采用一张**法线纹理**来直接存储表面法线，又叫法线映射。

### （1）高度纹理

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220604165559941.png" alt="image-20220604165559941" style="zoom:67%;" />

越黑的地方灰度越低，对应越向里凹。这种方式很直观，但是计算比较复杂，通常会和法线映射一起使用，用于给出表面凹凸的额外信息。（**通常用法线映射来修改光照**）

<br/>

### （2）法线纹理

法线纹理存储的就是表面的法线方向，但法线方向的分量范围为[-1,1],像素的分量范围为[0,1],所以要做一步转换：
$$
pixel=(normal+1)/2
$$
自然在Shader中对法线纹理采样之后,还要对结果进行一次反映射,映射回法线的方向:
$$
normal=2*pixel-1
$$
法线纹理当中存储的法线方向是在**切线空间当中的**(对于模型的每一个顶点,它都有一个属于自己的切线空间,这个切线空间的原点就是该点本身,而z轴是顶点的法线方向,x轴是顶点的切线方向,y轴则通过叉积来获得(也被称为副切线或者副法线)),此时的纹理被称之为**切线空间的法线纹理**,如下面两张图所示:

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220605170055544.png" alt="image-20220605170055544" style="zoom: 50%;" />

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220605170124750.png" alt="image-20220605170124750" style="zoom: 50%;" />

对于下面这张图,左面是模型空间下的法线纹理,我们可以看到是五颜六色的,这是因为模型各个点的法线方向有较大差异。而右侧的图则看起来几乎都是蓝色的。**这是因为每个法线所在的坐标空间都不同（是表面每点各自的切线空间）。这种法线纹理其实存储了每个点在各自的切线空间中的法线扰动方向，如果法线方向不变，则对应法线方向为（0，0，1），颜色映射为（0.5，0.5，1），也就是浅蓝色**。这些蓝色恰恰说明大部分法线和模型本身法线是一样的，不需要改变。

使用切线空间的好处如下：

（1）自由度很高，由于记录的是相对法线信息，因此其实可以应用于一个完全不同的网格上，也会得到合适的结果；

（2）可以很好地进行UV动画，做水或熔岩的时候会经常用到。

（3）可以重用法线纹理，比如一块砖六个面可以用一张法线纹理图；

（4）可压缩，因为切线空间下的法线纹理中的Z方向总是正方向，因此可以仅存储XY方向，从而推导得出Z方向。(法线贴图只有两个维度即可，代表xy方向，z方向通过叉乘获得，可以得到最后的法线方向)



### （3）实践

需要在计算光照模型的时候统一各个方向矢量所在的坐标空间，因此有两种选择：

- 在切线空间下进行光照计算，此时需要把视角方向，光照方向转换到切线空间；
- 都在世界空间下进行光照计算。

效率上来说第一种方法往往要优于第二种方法，因为顶点着色器中就可以完成光照方向和视角方向的转变，而第二种由于要对纹理采样，因此变换过程得放在片元着色器。所以片元着色器里要进行矩阵操作。

但**从通用性角度来看，第二种更好，因为有时要在世界空间下进行一些计算，比如后面的CubeMap环境映射（后面再介绍）**。接下来将分别实现两个空间的方法：

#### （a）在切线空间下计算

需要在顶点着色器当中把视角方向和光照方向从模型空间变换到切线空间当中。这就要求一下模型空间转到切线空间的矩阵。很容易求的是切线空间到模型空间的矩阵，只需要在顶点着色器中按切线（x轴），副切线（y轴），法线（z轴）的顺序按列排列即可（参考前面的数学章节），这个矩阵的转置矩阵就是我们所需要的。所以，我们把切线，副切线，法线按行排列即可得到我们需要的矩阵。代码如下：

```c#
Shader "Chapter 7/NormalMapTangentSpace"
{
    Properties
    {
        _Color("Color Tint",Color)=(1,1,1,1)
        _MainTex ("Main Tex", 2D) = "white" {}
        _BumpMap("Normal Map",2D)="bump" {}  //bump是unity内置的法线纹理,没有提供法线纹理时,bump对应模型自带的法线信息
        _BumpScale("Bump Scale",Float)=1.0
        _Specular("Specular",Color)=(1,1,1,1)
        _Gloss("Gloss",Range(8.0,256))=20
    }
    SubShader
    {
        
        Pass
        {
            Tags{"LightMode"="ForwardBase"}
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            

            #include "Lighting.cginc"
            
            fixed4 _Color;
            sampler2D _MainTex;
            float4 _MainTex_ST; //不要忘了声明这个变量,ST表示缩放偏移
            sampler2D _BumpMap;
            float4 _BumpMap_ST;
            float _BumpScale;
            fixed4 _Specular;
            float _Gloss;

            struct a2v
            {
                float4 vertex : POSITION;
                float3 normal:NORMAL;
                float4 tangent:TANGENT; //注意类型是float4,因为我们要用w分量来决定切线空间中副切线的方向性
                float4 texcoord:TEXCOORD0;
            };

            struct v2f
            {
                float4 pos:SV_POSITION;
                float4 uv : TEXCOORD0;
                float3 lightDir:TEXCOORD1;
                float3 viewDir:TEXCOORD2;
            };

         
            v2f vert (a2v v)
            {
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);
                o.uv.xy=v.texcoord.xy*_MainTex_ST.xy+_MainTex_ST.zw;
                o.uv.zw=v.texcoord.xy*_BumpMap_ST.xy+_BumpMap_ST.zw;
                //使用了两张纹理,因此需要存储两个纹理坐标,为此我们把v2f中的uv变量的类型定义为float4,分别存储
                //两个纹理的坐标(实际上,两个纹理通常会使用同一组纹理坐标,以减少插值寄存器的使用数目)
                TANGENT_SPACE_ROTATION; //Unity提供了一个内置宏,帮助我们直接计算得到rotation矩阵
               //(rotation就是转换到切线空间的矩阵)
                
                o.lightDir=mul(rotation,ObjSpaceLightDir(v.vertex)).xyz;
                o.viewDir=mul(rotation,ObjSpaceViewDir(v.vertex)).xyz;
                return o;
                //至此,我们将需要的量都转换到了切线空间
            }

            fixed4 frag (v2f i) : SV_Target
            {
                fixed3 tangentLightDir=normalize(i.lightDir);
                fixed3 tangentViewDir=normalize(i.viewDir);
                
                //从法线贴图中获得纹素
                fixed4 packedNormal=tex2D(_BumpMap,i.uv.zw);
                fixed3 tangentNormal;
                
                // If the texture is not marked as "Normal map"
                //  tangentNormal.xy = (packedNormal.xy * 2 - 1) * _BumpScale;//按照之前所写的公式
                //  tangentNormal.z = sqrt(1.0 - saturate(dot(tangentNormal.xy, tangentNormal.xy)));

                // Or mark the texture as "Normal map", and use the built-in funciton
                tangentNormal = UnpackNormal(packedNormal);
                tangentNormal.xy *= _BumpScale;
                tangentNormal.z = sqrt(1.0 - saturate(dot(tangentNormal.xy, tangentNormal.xy)));

                fixed3 albedo = tex2D(_MainTex, i.uv).rgb * _Color.rgb;

                fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz * albedo;

                //按照法线贴图来进行漫反射和高光项的计算
                fixed3 diffuse = _LightColor0.rgb * albedo * max(0, dot(tangentNormal, tangentLightDir));

                fixed3 halfDir = normalize(tangentLightDir + tangentViewDir);
                fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(max(0, dot(tangentNormal, halfDir)), _Gloss);

                return fixed4(ambient + diffuse + specular, 1.0);
            }
            ENDCG
        }
    }
}

```

对于上面Shader，需要做出几点说明：

- （1）在顶点着色器当中，我们将光照方向和视角方向都转移到了切线空间，这是为了片元着色器在切线空间计算光照等信息。
- （2）在片元着色器中，计算镜面反射（Blinn-Phong）和漫反射的时候，需要用到法线，这里的法线不能是模型原来的法线（这样就体现不出来法线贴图的作用了），而应该是经过法线贴图指导之后的切线空间下的法线方向，这就对应下面的一段代码：
- （3）通常来说，我们会在Unity当中标记法线贴图为Normal Map（属性当中可以看），此时就可以用Unity自带的函数UnpackNormal来获取法线贴图的法线xy方向了（其实就是一步简单的把颜色的【0，1】范围转换到【-1，1】范围）。

```glsl
			  //从法线贴图中获得纹素
                fixed4 packedNormal=tex2D(_BumpMap,i.uv.zw);
                fixed3 tangentNormal;
                
                // If the texture is not marked as "Normal map"
                //  tangentNormal.xy = (packedNormal.xy * 2 - 1) * _BumpScale;//按照之前所写的公式
                //  tangentNormal.z = sqrt(1.0 - saturate(dot(tangentNormal.xy, tangentNormal.xy)));

                // Or mark the texture as "Normal map", and use the built-in funciton
                tangentNormal = UnpackNormal(packedNormal);
                tangentNormal.xy *= _BumpScale;
                tangentNormal.z = sqrt(1.0 - saturate(dot(tangentNormal.xy, tangentNormal.xy)));

```

最后实现的效果如左图所示（从结果来看，是具有凹凸感的）：

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220611161520817.png" alt="image-20220611161520817" style="zoom:67%;" />

在不同的BumpScale下我们可以看到不同程度的凹凸感。

<br/>

#### （b）在世界空间下计算

在世界空间下计算光照模型，需要在片元着色器中把法线方向从切线空间变换到世界空间下，基本思想是**在顶点着色器当中计算从切线空间到世界空间的变换矩阵，然后传递给片元着色器。**变换矩阵的计算可以由顶点的切线、副切线和法线在世界空间下的表示来得到。最后，我们只需要在片元着色器中把法线纹理中的法线方向从切线空间变换到世界空间下即可。尽管这种方法需要更多的计算，但在需要使用Cubemap进行环境映射等情况下，我们就需要使用这种方法。

修改v2f，顶点着色器和片元着色器，Shader如下（注意看顶点着色器如何把变换矩阵传递给片元着色器以及片元着色器是如何处理的）：

```glsl
// Upgrade NOTE: replaced '_Object2World' with 'unity_ObjectToWorld'

Shader "Chapter 7/NormalMapWorldSpace"
{
    Properties
    {
        _Color("Color Tint",Color)=(1,1,1,1)
        _MainTex ("Main Tex", 2D) = "white" {}
        _BumpMap("Normal Map",2D)="bump" {}  //bump是unity内置的法线纹理,没有提供法线纹理时,bump对应模型自带的法线信息
        _BumpScale("Bump Scale",Float)=1.0
        _Specular("Specular",Color)=(1,1,1,1)
        _Gloss("Gloss",Range(8.0,256))=20
    }
    SubShader
    {
        
        Pass
        {
            Tags{"LightMode"="ForwardBase"}
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            

            #include "Lighting.cginc"
            
            fixed4 _Color;
            sampler2D _MainTex;
            float4 _MainTex_ST; //不要忘了声明这个变量,ST表示缩放偏移
            sampler2D _BumpMap;
            float4 _BumpMap_ST;
            float _BumpScale;
            fixed4 _Specular;
            float _Gloss;

            struct a2v
            {
                float4 vertex : POSITION;
                float3 normal:NORMAL;
                float4 tangent:TANGENT; //注意类型是float4,因为我们要用w分量来决定切线空间中副切线的方向性
                float4 texcoord:TEXCOORD0;
            };

            struct v2f
            {
                float4 pos:SV_POSITION;
                float4 uv : TEXCOORD0;
                float4 TtoW0:TEXCOORD1;
                float4 TtoW1:TEXCOORD2;
                float4 TtoW2:TEXCOORD3; //技巧:可以把矩阵进行拆解,理论上3x3的矩阵就够了,但我们顺便存一下世界空间下的顶点位置
            };

         
            v2f vert (a2v v)
            {
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);
                o.uv.xy=v.texcoord.xy*_MainTex_ST.xy+_MainTex_ST.zw;
                o.uv.zw=v.texcoord.xy*_BumpMap_ST.xy+_BumpMap_ST.zw;
                
                
                float3 worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;  
                fixed3 worldNormal = UnityObjectToWorldNormal(v.normal);  
                fixed3 worldTangent = UnityObjectToWorldDir(v.tangent.xyz);  
                fixed3 worldBinormal = cross(worldNormal, worldTangent) * v.tangent.w; 

                // Compute the matrix that transform directions from tangent space to world space
                // Put the world position in w component for optimization
                o.TtoW0 = float4(worldTangent.x, worldBinormal.x, worldNormal.x, worldPos.x);
                o.TtoW1 = float4(worldTangent.y, worldBinormal.y, worldNormal.y, worldPos.y);
                o.TtoW2 = float4(worldTangent.z, worldBinormal.z, worldNormal.z, worldPos.z);
                //注:矩阵默认是按行存储的,所以上面那三个其实是矩阵的三行,也对应着三个向量竖着放是坐标系转换矩阵
                //同时,世界空间下的顶点坐标被存在w分量中,节约空间
                return o;
            }

            fixed4 frag (v2f i) : SV_Target
            {
                //在世界空间下进行计算
                float3 worldPos=float3(i.TtoW0.w,i.TtoW1.w,i.TtoW2.w);
                fixed3 lightDir=normalize(UnityWorldSpaceLightDir(worldPos));
                fixed3 viewDir=normalize(UnityWorldSpaceViewDir(worldPos));

                fixed3 bump=UnpackNormal(tex2D(_BumpMap,i.uv.zw));
                bump.xy*=_BumpScale;
                bump.z=sqrt(1.0-saturate(dot(bump.xy,bump.xy)));

                //下面这一步可能需要理解一下,在下面会进行解释(将法线从切线空间转到世界空间下)
                bump=normalize(half3(dot(i.TtoW0.xyz,bump),dot(i.TtoW1.xyz,bump),dot(i.TtoW2.xyz,bump)));
               

                fixed3 albedo = tex2D(_MainTex, i.uv).rgb * _Color.rgb;

                fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz * albedo;
			   // bump是已经转换到世界空间下的
                fixed3 diffuse = _LightColor0.rgb * albedo * max(0, dot(bump, lightDir));

                fixed3 halfDir = normalize(lightDir + viewDir);
                fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(max(0, dot(bump, halfDir)), _Gloss);

                return fixed4(ambient + diffuse + specular, 1.0);
            }
            ENDCG
        }
    }
}

```

注:关于如何把切线空间下的法线转换到世界空间：

参考[数学：法线变换](# 6. 法线变换)一节，可以发现，如果变换矩阵是正交矩阵，就可以直接使用变换顶点的变换矩阵来变换法线（**此时如果变换只包含旋转变换则符合要求**），这里虽然矩阵不是正交矩阵，但我们提取前三列的话还是正交矩阵的，完全可以用。所以只需要用顶点着色器传给片元着色器的切线空间->世界空间矩阵左乘法线贴图采样并计算出的法线方向即可。

从视觉表现上来说，在切线空间和在世界空间下计算光照几乎没有任何差别，后续我们还是更习惯于**在世界空间下来计算光照**。

<br/>

## 4.Unity中的法线纹理类型

如果要使用Unity的UnpackNormal函数，则法线纹理图片的类型需要标注为Normal Map，接下来描述在我们改为Normal Map的时候发生了什么：

> 简单来说，这么做可以让Unity根据不同平台对纹理进行压缩（例如使用DXT5nm格式，具体的压缩细节可以参考：http://tech-artists.org/wiki/Normal_map_compression ），再通过UnpackNormal函数来针对不同的压缩格式对法线纹理进行正确的采样。我们可以在UnityCG.cginc里找到UnpackNormal函数的内部实现：

具体的细节可以参考书的第154页。

当我们把纹理类型设置成Normal map后，还有一个复选框选项是*Create from Grayscale*（前面也有提及过） ，那么它是做什么用的呢？读者应该还记得在本节开始我们提到过另一种凹凸映射的方法，即使用高度图，而这个复选框就是用于从高度图中生成法线纹理的。高度图本身记录的是相对高度，是一张灰度图，白色表示相对更高，黑色表示相对更低。当我们把一张高度图导入Unity后，除了需要把它的纹理类型设置成Normal map外，还需要勾选Create from Grayscale，这样就可以得到类似图7.17中的结果。然后，我们就可以把它和切线空间下的法线纹理同等对待了。

当勾选了Create from Grayscale后，还多出了两个选项—*Bumpiness* 和*Filtering* 。其中Bumpiness用于控制凹凸程度，而Filtering决定我们使用哪种方式来计算凹凸程度，它有两种选项：一种是*Smooth* ，这使得生成后的法线纹理会比较平滑；另一种是*Sharp* ，它会使用Sobel滤波（一种边缘检测时使用的滤波器）来生成法线。Sobel滤波的实现非常简单，我们只需要在一个3×3的滤波器中计算x和y方向上的导数，然后从中得到法线即可。具体方法是：对于高度图中的每个像素，我们考虑它与水平方向和竖直方向上的像素差，把它们的差当成该点对应的法线在x和y方向上的位移，然后使用之前提到的映射函数存储成到法线纹理的r和g分量即可。

用如下的方式来打开：

![image-20220611173856306](E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220611173856306.png)

（注：要先选择上面的Normal Map）

<br/>

## 5.渐变纹理

纹理其实可以用于存储任何表面属性，比如使用渐变纹理来控制漫反射光照的结果。之前介绍漫反射光照的时候，都是使用点积和材质的反射率相乘来得到表面的漫反射关照，但有时需要更加灵活地控制光照结果。（比如可以用来渲染游戏中插画风格的角色）。

这个技术最早在1998年发表，作者提出了一种基于冷到暖色调的着色技术，用来体现插画风格的渲染效果。**使用这种技术，可以保证物体的轮廓线相比于之前使用的传统漫反射光照更加明显，而且能够提供多种色调变化。**而现在，很多卡通风格的渲染中都使用了这种技术。我们在第十四章中会专门学习如何编写一个卡通风格的Unity Shader。

本节会使用一张渐变纹理来控制漫反射光照。目标实现下图所示的效果：

![image-20220611205253923](E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220611205253923.png)

相关的Shader如下：

```glsl
Shader "Chapter 7/RampTexture"
{
    Properties
    {
        _Color("Color Tint",Color)=(1,1,1,1)
        _RampTex("Ramp Tex",2D)="white"{} //渐变纹理,当然下面声明的变量要有一个_ST
        _Specular("Specular",Color)=(1,1,1,1)
        _Gloss("Gloss",Range(8.0,256))=20
    }
    SubShader
    {
        Pass
        {
            Tags{"LightMode"="ForwardBase"}
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
           
            #include "Lighting.cginc"
            fixed4 _Color;
            sampler2D _RampTex;
            float4 _RampTex_ST;
            fixed4 _Specular;
            float _Gloss;
            
            struct a2v
            {
                float4 vertex : POSITION;
                float3 normal: NORMAL;
                float4 texcoord: TEXCOORD0;
            };

            struct v2f
            {
                float4 pos : SV_POSITION;
                float3 worldNormal:TEXCOORD0;
                float3 worldPos:TEXCOORD1;
                float2 uv : TEXCOORD2;            
            };

            v2f vert (a2v v)
            {
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);
                o.worldNormal=UnityObjectToWorldNormal(v.normal);
                o.worldPos=mul(unity_ObjectToWorld,v.vertex).xyz;
                o.uv = TRANSFORM_TEX(v.texcoord, _RampTex);
               
                return o;
            }

            fixed4 frag (v2f i) : SV_Target
            {
                fixed3 worldNormal=normalize(i.worldNormal);
                fixed3 worldLightDir=normalize(UnityWorldSpaceLightDir(i.worldPos));//使用内置函数使得Shader可以适应各种光源
                fixed3 ambient=UNITY_LIGHTMODEL_AMBIENT.xyz;
                fixed halfLambert=0.5*dot(worldNormal,worldLightDir)+0.5;
                fixed3 diffuseColor=tex2D(_RampTex,fixed2(halfLambert,halfLambert)).rgb*_Color.rgb;
                // 注:上面按照对角线进行采样,是因为对于渐变纹理来说对角线相对来讲更长一些,有助于平滑过渡
                fixed3 diffuse=_LightColor0.rgb*diffuseColor;

                fixed3 viewDir = normalize(UnityWorldSpaceViewDir(i.worldPos));
                fixed3 halfDir = normalize(worldLightDir + viewDir);
                fixed3 specular=_LightColor0.rgb*_Specular.rgb*pow(max(0,dot(worldNormal,halfDir)),_Gloss);
                
                return fixed4(ambient+diffuse+specular,1.0);
            }
            ENDCG
        }
    }
    FallBack "Specular"
}
```

我们使用halfLambert模型来计算漫反射，使用halfLambert来构建一个纹理坐标，并用这个纹理坐标对渐变纹理_RampTex进行采样。由于 _RampTex就是一个一维纹理（在纵轴上颜色不变），因此纹理坐标的u和v方向都使用了halfLambert。然后，把从渐变纹理采样得到的颜色和材质 _Color相乘，得到最终的漫反射颜色。

最终效果如下：

![image-20220611212053322](E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220611212053322.png)

补充：为什么渐变纹理可以用来做采样，进而实现我们想要的效果？

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220611212357963.png" alt="image-20220611212357963" style="zoom: 80%;" />

（参考：[Shader入门精要笔记5(2)-渐变纹理和遮罩纹理 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/405183649)）

可以看到上面的纹理图，靠左的部分比较黑（对应点乘结果较小，也就是n和l夹角较大的情况（光源比较斜，本来也会比较暗）），另外也是同理的。（也就是，渐变纹理也是符合漫反射计算的结果大致规律的）。

**需要注意的是，要把渐变纹理的Wrap Mode设置为Clamp，否则纹理采样时可能会带来浮点数精度的问题，比如：如果我们使用Repeat模式而非Clamp模式：**

![image-20220611212834192](E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220611212834192.png)

产生这些不正确的区域斑点是因为在Repeat模式下，虽然理论上halfLambert计算的值在【0，1】之间，但可能会出现1.0001这种计算结果，如果是repeat就会舍弃1而保留小数部分，就会出现采样纹理上的突变，不符合要求，**所以要把渐变纹理的Wrap Mode设置为Clamp。**

<br/>

## 6.遮罩纹理

这种纹理非常重要，**可以允许我们保护某些区域，使他们免于某些修改**（比如我们希望模型表面的某些区域的反光强烈一些，某些区域弱一些。）为了得到更加细腻的效果，就可以使用一张遮罩纹理来控制光照。另一种常见的应用是在制作地形材质时需要混合多张照片，比如表现草地的纹理，表现石子的纹理等，使用遮罩纹理可以控制如何混合这些纹理。

一般流程为：**通过采样得到遮罩纹理的纹素值，然后使用其中某个或某几个通道的值（例如texel.r）来与某种表面属性进行相乘，这样，当通道的值为0时，可以保护表面不受该属性的影响**。（其实就是一张mask）

通过遮罩纹理，实现的效果以及对应的遮罩纹理如下：

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220611214026094.png" alt="image-20220611214026094" style="zoom: 80%;" />

相关代码如下：

```glsl
Shader "Chapter 7/MaskTexture"
{
    Properties
    {
        _Color("Color Tint",Color)=(1,1,1,1)
        _MainTex ("Main Tex", 2D) = "white" {}
        _BumpMap("Normal Map",2D)="bump" {}  //bump是unity内置的法线纹理,没有提供法线纹理时,bump对应模型自带的法线信息
        _BumpScale("Bump Scale",Float)=1.0
        //以下两个变量用来控制高光反射
        _SpecularMask("Specular Mask",2D)="white"{} //所需要使用的高光反射遮罩纹理
        _SpecularScale("Specular Scale",Float)=1.0  //控制遮罩影响度的系数
        _Specular("Specular",Color)=(1,1,1,1)
        _Gloss("Gloss",Range(8.0,256))=20
    }
    SubShader
    {
        Pass
        {
            Tags{"LightMode"="ForwardBase"}
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            
            #include "Lighting.cginc"
            
            fixed4 _Color;
            sampler2D _MainTex;
            float4 _MainTex_ST; //不要忘了声明这个变量,ST表示缩放偏移
            sampler2D _BumpMap;
            float _BumpScale;
            sampler2D _SpecularMask;
            float _SpecularScale;
            fixed4 _Specular;
            float _Gloss;
            
            //这里让主纹理,法线纹理和遮罩纹理共同使用_MainTex_ST,此时修改tiling和offset就会影响三种纹理的采样(这样的话就不需要额外写另外两个纹理的_ST了,具体要视需求而定)
            struct a2v
            {
                float4 vertex : POSITION;
                float3 normal:NORMAL;
                float4 tangent:TANGENT; //注意类型是float4,因为我们要用w分量来决定切线空间中副切线的方向性
                float4 texcoord:TEXCOORD0;
            };

            struct v2f
            {
                float4 pos:SV_POSITION;
                float2 uv : TEXCOORD0;
                float3 lightDir:TEXCOORD1;
                float3 viewDir:TEXCOORD2;
            };

            v2f vert (a2v v)
            {
                //在顶点着色器中我们对光照方向和视角方向进行了坐标空间的变换,使其从模型空间转到切线空间
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);
                o.uv.xy=v.texcoord.xy*_MainTex_ST.xy+_MainTex_ST.zw;
                
                TANGENT_SPACE_ROTATION; //Unity提供了一个内置宏,帮助我们直接计算得到rotation矩阵
               //(就是转换到切线空间的矩阵)
                
                o.lightDir=mul(rotation,ObjSpaceLightDir(v.vertex)).xyz;
                o.viewDir=mul(rotation,ObjSpaceViewDir(v.vertex)).xyz;
                return o;
                //至此,我们将需要的量都转换到了切线空间
            }

            fixed4 frag (v2f i) : SV_Target
            {
                fixed3 tangentLightDir=normalize(i.lightDir);
                fixed3 tangentViewDir=normalize(i.viewDir);
                
                //从法线贴图中获得纹素
                fixed4 packedNormal=tex2D(_BumpMap,i.uv);
                fixed3 tangentNormal;
              
                // Or mark the texture as "Normal map", and use the built-in function
                tangentNormal = UnpackNormal(packedNormal);
                tangentNormal.xy *= _BumpScale;
                tangentNormal.z = sqrt(1.0 - saturate(dot(tangentNormal.xy, tangentNormal.xy)));
                
                fixed3 albedo = tex2D(_MainTex, i.uv).rgb * _Color.rgb;

                fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz * albedo;

                fixed3 diffuse = _LightColor0.rgb * albedo * max(0, dot(tangentNormal, tangentLightDir));

                fixed3 halfDir = normalize(tangentLightDir + tangentViewDir);

                //重点:获取mask的值,进行逐像素相乘
                fixed specularMask=tex2D(_SpecularMask,i.uv).r*_SpecularScale;
                fixed3 specular=_LightColor0.rgb*_Specular.rgb*pow(max(0,dot(tangentNormal,halfDir)),_Gloss)*specularMask;

                return fixed4(ambient + diffuse + specular, 1.0);

            }
            ENDCG
        }
    }
}

```

使用遮罩纹理和不使用遮罩纹理的结果对比如下（左：使用，右：不使用）

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220611220717145.png" alt="image-20220611220717145" style="zoom:67%;" />

可以看到，使用遮罩纹理之后的高光反射效果更加细腻，这对游戏来说大有帮助。需要说明的是，我们使用的遮罩纹理其实有很多空间被浪费了——RGB分量其实可以存储不同的表面属性，比如说我们可以把高光反射的强度存在R通道，边缘光照的强度存在G，高光反射指数部分存在B，自发光强度存在A。比如DOTA2当中，开发人员为每个模型准备4张纹理（一张用于定义模型颜色，一张用于定义表面法线，另外两张则都是遮罩纹理。这样，两张遮罩纹理提供了共8种额外的表面属性，这使得游戏中的人物材质自由度很强，可以支持很多高级的模型属性。）

<br/>

# 第八章 透明效果

- 实现透明效果,通常会在渲染模型时控制它的**透明通道（Alpha Channel）**

当**开启透明混合**后，当一个物体被渲染到屏幕上时，每个片元除了颜色值和深度值之外，它还有另一个属性——**透明度**。

- 透明度为1:完全不透明
- 透明度为0:该像素完全不会显示

<br/>

在Unity中，我们通常使用两种方法来实现透明效果：第一种是使用**透明度测试（Alpha Test）** ，这种方法其实无法得到真正的半透明效果；另一种是**透明度混合（Alpha Blending）** 。

- 对于不透明（opaque）物体，**不考虑它们的渲染顺序也能得到正确的排序效果**，这是由于强大的深度缓冲（depth buffer，也被称为z-buffer）的存在。

  - 深度缓冲是用于**解决可见性（visibility）问题的**, 当渲染一个片元时，需要把它的深度值和已经存在于深度缓冲中的值进行比较（如果开启了**深度测试**），如果它的值距离摄像机更远，那么说明这个片元不应该被渲染到屏幕上（有物体挡住了它）；否则，这个片元应该覆盖掉此时颜色缓冲中的像素值，并把它的深度值更新到深度缓冲中（如果开启了**深度写入**）。
  - (**是否开启深度测试决定是否会对深度进行比较,而是否开启深度写入会决定新的深度值是否应该更新到深度缓冲区中**)

  - 使用深度缓冲，可以让我们不用关心不透明物体的渲染顺序.

- 对于透明物体,事情就不那么简单了,这是因为，**当使用透明度混合时，我们关闭了深度写入（ZWrite）。**



简单来说，透明度测试和透明度混合的基本原理如下(可以根据后面的效果来加深理解):

- **透明度测试** ：它采用一种“霸道极端”的机制，**只要一个片元的透明度不满足条件（通常是小于某个阈值），那么它对应的片元就会被舍弃。被舍弃的片元将不会再进行任何处理，也不会对颜色缓冲产生任何影响；**否则，就会按照普通的不透明物体的处理方式来处理它，即进行深度测试、深度写入等。也就是说，透明度测试是不需要关闭深度写入的，它和其他不透明物体最大的不同就是它会根据透明度来舍弃一些片元。虽然简单，但是它产生的效果也很极端，要么完全透明，即看不到，要么完全不透明，就像不透明物体那样。
- **透明度混合** ：这种方法可以得到真正的半透明效果。**它会使用当前片元的透明度作为混合因子，与已经存储在颜色缓冲中的颜色值进行混合，得到新的颜色。**但是，**透明度混合需要关闭深度写入**（我们下面会讲为什么需要关闭），这使得我们要非常小心物体的渲染顺序。需要注意的是，**透明度混合只关闭了深度写入，但没有关闭深度测试。**这意味着，当使用透明度混合渲染一个片元时，还是会比较它的深度值与当前深度缓冲中的深度值，**如果它的深度值距离摄像机更远，那么就不会再进行混合操作。**这一点决定了，当一个不透明物体出现在一个透明物体的前面，而我们先渲染了不透明物体，它仍然可以正常地遮挡住透明物体。(这就是深度测试的重要性). 也就是说，**对于透明度混合来说，深度缓冲是只读的。**

[为什么半透明模型的渲染要使用深度测试而关闭深度写入？ - 知乎 (zhihu.com)](https://www.zhihu.com/question/60898307)

https://www.jianshu.com/p/e953e937e210



## 8.1 为什么渲染顺序很重要

> 进行半透明渲染需要开启深度测试的原因是，如果半透明物体被不透明物体挡住了，那么该fragments是不需要写入到帧缓冲的，也就是说开启深度测试是为了保证不透明物体和半透明物体之间正常的遮挡关系；而关闭深度写入也很好理解，因为开启深度写入后我们会得到这样一种错误的结果，那就是近处的半透明fragments居然会挡住远处的半透明fragments，这和我们前面说到的我们可以透过半透明物体看到物体后面的东西是矛盾的！

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220817215628348.png" alt="image-20220817215628348" style="zoom:80%;" />

(上面的图8.1与下面的图8.2的AB的位置是一样的,因此这里统一用下面这张图)

- 不透明物体应该在半透明物体之前被渲染;
- 对于半透明物体之间,渲染顺序也非常重要,见下:
- ![image-20220817215752658](E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220817215752658.png)



### **结论(在开启深度测试但是关闭深度写入之后):**

- 先渲染所有的不透明物体,开启深度测试和深度写入;
- 把半透明物体按它们距离摄像机的远近进行排序，然后按照从后往前的顺序渲染这些半透明物体，并开启它们的深度测试，但关闭深度写入。
  - 值得注意的是,**这是物体级别的排序,所以下述情况难以处理:**
  - <img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220817220821856.png" alt="image-20220817220821856" style="zoom:67%;" />

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220817220850680.png" alt="image-20220817220850680" style="zoom:67%;" />

此时的问题是,如果我们选定物体上的一个特定点(比如中点)作为深度比较的依据,那么A会在B的前面(在上图中,不管我们选择物体上的哪个顶点都是这个结果,比如头部的点,中间的点,后面的点),但实际上A有一部分会被B遮挡.

- **这种问题的解决方法通常也是分割网格。**

尽管结论是，总是会有一些情况打乱我们的阵脚，**但由于上述方法足够有效并且容易实现，因此大多数游戏引擎都使用了这样的方法**.为了减少错误排序的情况，**我们可以尽可能让模型是凸面体**，并且**考虑将复杂的模型拆分成可以独立排序的多个子模型等**。其实就算排序错误结果有时也不会非常糟糕，如果我们不想分割网格，可以试着让透明通道更加柔和，使穿插看起来并不是那么明显。**我们也可以使用开启了深度写入的半透明效果来近似模拟物体的半透明（详见8.5节）。**





## 8.2 Unity Shader的渲染顺序

Unity为了解决渲染顺序的问题提供了**渲染队列** **（render queue）** 这一解决方案。我们可以使用`SubShader`的**Queue** 标签来决定我们的模型将归于哪个渲染队列。**Unity在内部使用一系列整数索引来表示每个渲染队列，且索引号越小表示越早被渲染。**在Unity 5中，Unity提前定义了5个渲染队列，**当然在每个队列中间我们可以使用其他队列**。表8.1给出了这5个提前定义的渲染队列以及它们的描述。

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220817221029493.png" alt="image-20220817221029493"  />

因此，如果我们想要通过透明度测试实现透明效果，代码中应该包含类似下面的代码：

```c#
SubShader {
    Tags { "Queue"="AlphaTest" }  //使用透明度测试
    Pass {
        ...
    }
}
```

如果我们想要通过**透明度混合**来实现透明效果，代码中应该包含类似下面的代码:

```c#
SubShader {
    Tags { "Queue"="Transparent" }
    Pass {
        ZWrite Off   //关闭透明度写入
        ...
    }
}
```

其中，**ZWrite Off** 用于关闭深度写入，在这里我们选择把它写在Pass中。我们也可以把它写在SubShader中，这意味着该SubShader下的所有Pass都会关闭深度写入。



## 8.3 透明度测试

- 在这种情况下,透明度测试得到的透明效果很“极端”——要么完全透明，要么完全不透明，它的效果往往像在一个不透明物体上挖了一个空洞。而且，得到的透明效果在边缘处往往参差不齐，有锯齿，这是因为在边界处纹理的透明度的变化精度问题。为了得到更加柔滑的透明效果，就可以使用透明度混合。



我们来看一下如何在Unity中实现透明度测试的效果。在上面我们已经知道了透明度测试的工作原理。

**透明度测试：** 只要一个片元的透明度不满足条件（通常是小于某个阈值），那么它对应的片元就会被舍弃。被舍弃的片元将不会再进行任何处理，也不会对颜色缓冲产生任何影响；否则，就会按照普通的不透明物体的处理方式来处理它。

通常，我们会在片元着色器中使用clip函数来进行透明度测试。clip是Cg中的一个函数，它的定义如下。

**函数** ：`void clip(float4 x); `

`void clip(float3 x); `

`void clip(float2 x);`

` void clip(float1 x);` 

` void clip(float x);`

**参数** ：裁剪时使用的标量或矢量条件。

**描述** ：如果给定参数的任何一个分量是负数，就会舍弃当前像素的输出颜色。它等同于下面的代码：

```c#
void clip(float4 x)
{
    if (any(x < 0))  // 如果给定参数的任何一个分量是负数，就会舍弃当前像素的输出颜色。
        discard;
}
```

在本节中，我们使用图8.5中的半透明纹理来实现透明度测试。在本书资源中，该纹理名为transparent_texture.psd。该透明纹理在不同区域的透明度也不同，我们通过它来查看透明度测试的效果。



## 本章目标1:

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220817222138875.png" alt="image-20220817222138875" style="zoom:80%;" />

1.新建shader,material等资源;

2.为了在材质面板中控制透明度测试时使用的阈值，我们在Properties语义块中声明一个范围在[0, 1]之间的属性_Cutoff：

```c#
	Properties
	{
		_Color("Color Tint",Color)=(1,1,1,1)
		_MainTex("Main Tex",2D) = "white"{}
		_Cutoff("Alpha Cutoff",Range(0,1))=0.5 //_Cutoff参数用于决定我们调用clip进行透明度测试时使用的判断条件。
			//它的范围是[0，1]，这是因为纹理像素的透明度就是在此范围内。
	}
```

然后，我们在SubShader语义块中定义了一个Pass语义块：

```c#
    SubShader
	{
		Tags { "Queue"="AlphaTest" "IgnoreProjector" = "True" "RenderType" = "TransparentCutout" }
		Pass
		{
			Tags {"LightMode"="ForwardBase"}
			CGPROGRAM
			#pragma vertex vert
			#pragma fragment frag
			#include "Lighting.cginc"

			ENDCG
		}
	}
```

我们在8.2节中已经知道渲染顺序的重要性，并且知道在**Unity中透明度测试使用的渲染队列是名为AlphaTest的队列**，因此我们需要把**Queue标签设置为AlphaTest**。

而RenderType标签可以让Unity把这个Shader归入到提前定义的组（这里就是TransparentCutout组）中，以指明该Shader是一个使用了透明度测试的Shader。

- RenderType标签通常被用于着色器替换功能。

我们还把IgnoreProjector设置为True，这意味着这个Shader不会受到投影器（Projectors）的影响。

**通常，使用了透明度测试的Shader都应该在SubShader中设置这三个标签。**最后，LightMode标签是Pass标签中的一种，它用于定义该Pass在Unity的光照流水线中的角色。只有定义了正确的LightMode，我们才能正确得到一些Unity的内置光照变量，例如_LightColor0。

<br/>

3.为了和Properties语义块中声明的属性建立联系，我们需要定义和各个属性类型相匹配的变量：

```c#
fixed4 _Color;
sampler2D _MainTex;
float4 _MainTex_ST; //对于纹理项来说,要注意这两个的类型,sampler2D以及float4,后面还要加_ST
fixed _Cutoff;
```

由于_Cutoff的范围在[0, 1]，因此我们可以使用fixed精度来存储它。

<br/>

4.然后，我们定义了顶点着色器的输入和输出结构体，接着定义顶点着色器：

```c#
struct a2v
{
	float4 vertex:POSITION;
	float3 normal:NORMAL;
	float4 texcoord:TEXCOORD0;
};
struct v2f
{
	float4 pos:SV_POSITION;
	float3 worldNormal:TEXCOORD0;
	float3 worldPos:TEXCOORD1; //worldNormal和worldPos是用于后续在片元着色器当中计算光照的diffuse项的
	float2 uv:TEXCOORD2;
};

v2f vert(a2v v)
{
	v2f o;
	o.pos = UnityObjectToClipPos(v.vertex); //输入顶点坐标通过MVP矩阵变换到裁剪空间
	o.worldNormal= UnityObjectToWorldNormal(v.normal); //世界坐标法线转换
	o.worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;  //顶点坐标世界空间
	o.uv = TRANSFORM_TEX(v.texcoord, _MainTex); //通过*tiling+offset将纹理图转换为uv
	return o;
}
```

补充内容:

[(7条消息) Unity_Shader 之TRANSFORM_TEX详解_lalate的专栏-CSDN博客_shader transform_tex](https://blog.csdn.net/lalate/article/details/91479715)

上面的代码我们已经见到过很多次了，我们在顶点着色器计算出世界空间的法线方向和顶点位置以及变换后的纹理坐标，再把它们传递给片元着色器。

5.**最重要的片元着色器**

```c#
fixed4 frag(v2f i):SV_Target
{
	fixed3 worldNormal = normalize(i.worldNormal);
	fixed3 worldLightDir = normalize(UnityWorldSpaceLightDir(i.worldPos)); // 光照方向,这两项可以用来计算漫反射项
	fixed4 texColor = tex2D(_MainTex, i.uv); //对纹理进行采样

	//进行透明度测试,不符合要求的直接舍弃掉
	clip(texColor.a - _Cutoff);
	fixed3 albedo = texColor.rgb*_Color.rgb;
	fixed3 diffuse = _LightColor0.rgb*albedo*max(dot(worldNormal, worldLightDir), 0);
	fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz * albedo;

	return fixed4(diffuse+ambient, 1.0);
}
```

前面我们已经提到过clip函数的定义，它会判断它的参数，即texColor.a - _ Cutoff是否为负数，如果是就会舍弃该片元的输出。也就是说，当texColor.a小于材质参数_Cutoff时，该片元就会产生完全透明的效果。使用clip函数等同于先判断参数是否小于零，如果是就使用discard指令来显式剔除该片元。后面的代码和之前使用过的完全一样，我们计算得到环境光照和漫反射光照，把它们相加后再进行输出。

<br/>

6.为Shader设置合适的Fallback

```c#
Fallback "Transparent/Cutout/VertexLit"
```

和之前使用的Diffuse和Specular不同，这次我们使用内置的Transparent/Cutout/VertexLit来作为回调Shader。这不仅能够保证在我们编写的SubShader无法在当前显卡上工作时可以有合适的代替Shader，还可以保证使用透明度测试的物体可以正确地向其他物体投射阴影，具体原理可以参见9.4.5节。

<br/>

总的代码:

```c#
Shader "MyShader/alpha/alpha1test"
{
	Properties
	{
		_Color("Color Tint",Color)=(1,1,1,1)
		_MainTex("Main Tex",2D) = "white"{}
		_Cutoff("Alpha Cutoff",Range(0,1)) = 0.5 //_Cutoff参数用于决定我们调用clip进行透明度测试时使用的判断条件。
			//它的范围是[0，1]，这是因为纹理像素的透明度就是在此范围内。
	}
		SubShader
		{
			Tags { "Queue" = "AlphaTest" "IgnoreProjector" = "True" "RenderType" = "TransparentCutout" }
			Pass
			{
				Tags {"LightMode" = "ForwardBase"}
				CGPROGRAM
				#pragma vertex vert
				#pragma fragment frag
				#include "Lighting.cginc"
				fixed4 _Color;
				sampler2D _MainTex;
				float4 _MainTex_ST;
				fixed _Cutoff;

				struct a2v
				{
					float4 vertex:POSITION;
					float3 normal:NORMAL;
					float4 texcoord:TEXCOORD0;
				};
				struct v2f
				{
					float4 pos:SV_POSITION;
					float3 worldNormal:TEXCOORD0;
					float3 worldPos:TEXCOORD1;
					float2 uv:TEXCOORD2;
				};

				v2f vert(a2v v)
				{
					v2f o;
					o.pos = UnityObjectToClipPos(v.vertex); //输入顶点坐标通过MVP矩阵变换到裁剪空间
					o.worldNormal= UnityObjectToWorldNormal(v.normal); //世界坐标法线转换
					o.worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;  //顶点坐标世界空间
					o.uv = TRANSFORM_TEX(v.texcoord, _MainTex); //通过*tiling+offset将纹理图转换为uv
					return o;
				}

				fixed4 frag(v2f i):SV_Target
				{
					fixed3 worldNormal = normalize(i.worldNormal);
					fixed3 worldLightDir = normalize(UnityWorldSpaceLightDir(i.worldPos)); // 光照方向,这两项可以用来计算漫反射项
					fixed4 texColor = tex2D(_MainTex, i.uv); //对纹理进行采样

					//进行透明度测试,不符合要求的直接舍弃掉
					clip(texColor.a - _Cutoff);
					fixed3 albedo = texColor.rgb*_Color.rgb;
					fixed3 diffuse = _LightColor0.rgb*albedo*max(dot(worldNormal, worldLightDir), 0);
					fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz * albedo;

					return fixed4(diffuse+ambient, 1.0);
				}

			ENDCG
					
		}
			// Fallback "Transparent/Cutout/VertexLit"
	}
}

```

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220817223851260.png" alt="image-20220817223851260" style="zoom: 80%;" />

<br/>

## 8.4 透明度混合

**透明度混合：** 这种方法可以得到真正的半透明效果。它会使用当前片元的透明度作为混合因子，与已经存储在颜色缓冲中的颜色值进行混合，得到新的颜色。但是，透明度混合需要关闭深度写入，这使得我们要非常小心物体的渲染顺序。

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220817224014093.png" alt="image-20220817224014093" style="zoom:67%;" />

在本节里，我们会使用第二种语义，即`Blend SrcFactor DstFactor`来进行混合。需要注意的是，**这个命令在设置混合因子的同时也开启了混合模式。**

> $DstColor_{new} = SrcAlpha × SrcColor + ( 1 − SrcAlpha ) × DstColor_{old}$



### 本章目标2:

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220820124530416.png" alt="image-20220820124530416" style="zoom:67%;" />

1. 创建一个平面和一个立方体,**最终实现的效果应该是可以通过立方体看到后面的平面.**

```c#
Shader "MyShader/alpha/alpha1test2"
{
    Properties
    {
		_Color("Main Tint",Color)=(1,1,1,1)
        _MainTex ("Texture", 2D) = "white" {}
		_AlphaScale("AlphaScale",Range(0,1))=0.5
    }
    SubShader
    {
		Tags {"Queue" = "Transparent" "IgnoreProjector" = "True" "RenderType" = "Transparent"} //注意这里面的队列Queue要改为Transparent
		//见注解1部分
        Pass
        {
			Tags {"LightMode"="ForwardBase"}
			ZWrite Off
			Blend SrcAlpha OneMinusSrcAlpha  //见注解2部分
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag

            #include "Lighting.cginc"

            struct a2v
            {
                float4 vertex : POSITION;
				float3 normal:NORMAL;
                float4 texcoord : TEXCOORD0;
            };

            struct v2f
            {
                float4 pos : SV_POSITION;
				float2 uv : TEXCOORD0;
				float3 worldNormal:TEXCOORD1;
				float3 worldPos:TEXCOORD2;
            };

            sampler2D _MainTex;
            float4 _MainTex_ST;  
			fixed4 _Color;
			fixed _AlphaScale;

			v2f vert(a2v v)
			{
				v2f o;
				o.pos = UnityObjectToClipPos(v.vertex); //输入顶点坐标通过MVP矩阵变换到裁剪空间
				o.worldNormal = UnityObjectToWorldNormal(v.normal); //世界坐标法线转换
				o.worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;  //顶点坐标世界空间
				o.uv = TRANSFORM_TEX(v.texcoord, _MainTex); //通过*tiling+offset将纹理图转换为uv
				return o;
			}

			fixed4 frag(v2f i) :SV_Target
			{
				fixed3 worldNormal = normalize(i.worldNormal);
				fixed3 worldLightDir = normalize(UnityWorldSpaceLightDir(i.worldPos)); // 光照方向,这两项可以用来计算漫反射项
				fixed4 texColor = tex2D(_MainTex, i.uv); //对纹理进行采样
				
				fixed3 albedo = texColor.rgb*_Color.rgb;
				fixed3 diffuse = _LightColor0.rgb*albedo*max(dot(worldNormal, worldLightDir), 0);
				fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz * albedo;

				return fixed4(diffuse + ambient, texColor.a*_AlphaScale); //见注解3
			}

            
            ENDCG
        }
    }
}

```

注解1:

在本章一开头，我们已经知道在Unity中**透明度混合使用的渲染队列是名为Transparent的队列**，因此我们需要把Queue标签设置为Transparent。`RenderType`标签可以让Unity把这个`Shader`归入到提前定义的组（这里就是Transparent组）中，用来指明该`Shader`是一个使用了透明度混合的`Shader`。`RenderType`标签通常被用于着色器替换功能。我们还把`IgnoreProjector`设置为True，这意味着这个`Shader`不会受到投影器（Projectors）的影响。通常，使用了透明度混合的`Shader`都应该在`SubShader`中设置这3个标签。

注解2:

Pass的标签仍和之前一样，即把`LightMode`设为`ForwardBase`，这是为了让Unity能够按前向渲染路径的方式为我们正确提供各个光照变量。除此之外，我们还把该Pass的深度写入（`ZWrite`）设置为关闭状态（Off），我们在之前已经讲过为什么要这样做了。这是非常重要的。然后，我们开启并设置了该Pass的混合模式。如在本节开头所讲的，我们将源颜色（该片元着色器产生的颜色）的混合因子设为`SrcAlpha`，把目标颜色（已经存在于颜色缓冲中的颜色）的混合因子设为`OneMinusSrcAlpha`，以得到合适的半透明效果。(**具体因子可以设置成什么值可以查看8.6节**)

注解3:

上述代码和8.3节中的几乎完全一样，只是移除了透明度测试的代码，并设置了该片元着色器返回值中的透明通道，它是纹理像素的透明通道和材质参数`_AlphaScale`的乘积。正如本节一开始所说的，只有使用Blend命令打开混合后，我们在这里设置透明通道才有意义，否则，这些透明度并不会对片元的透明效果有任何影响。

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220820124648519.png" alt="image-20220820124648519" style="zoom: 67%;" />

<br/>

##### 关闭深度写入带来的问题

> 对于模型网格之间有互相交叉的结构时，往往会得到错误的半透明效果:

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220820124724570.png" alt="image-20220820124724570" style="zoom:50%;" />

这都是由于我们关闭了深度写入造成的，因为这样我们就无法对模型进行像素级别的深度排序。**在8.1节中我们提到了一种解决方法是分割网格，从而可以得到一个“质量优等”的网格。但是很多情况下这往往是不切实际的。**这时，我们可以想办法重新利用深度写入，让模型可以像半透明物体一样进行淡入淡出。这就是我们下面要讲的内容。

<br/>

#### 8.5 开启深度写入的半透明效果

在8.4节最后，我们给出了一种由于关闭深度写入而造成的错误排序的情况。一种解决方法是**使用两个Pass** 来渲染模型：

- 第一个Pass开启深度写入，但不输出颜色，它的目的仅仅是为了把该模型的深度值写入深度缓冲中；
- 第二个Pass进行正常的透明度混合，**由于上一个Pass已经得到了逐像素的正确的深度信息，该Pass就可以按照像素级别的深度排序结果进行透明渲染。**但这种方法的缺点在于，多使用一个Pass会对性能造成一定的影响。

在本节最后，我们可以得到类似图8.11中的效果。可以看出，使用这种方法，我们仍然可以实现模型与它后面的背景混合的效果，但模型内部之间不会有任何真正的半透明效果。

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220820124811141.png" alt="image-20220820124811141" style="zoom:67%;" />

(对于每个像素来说,前面的会挡住后面的,因为我们开启了深度写入(但注意,我们不能改变缓冲区的颜色))

1.在原来的Shader的基础上,我们只需要加入一个Pass,放在原来Shader之前就可以了

```c#
Shader "Unity Shader Book/Chapter8-Alpha Blending ZWrite" {
    Properties {
        _Color ("Main Tint", Color) = (1,1,1,1)
        _MainTex ("Main Tex", 2D) = "white" {}
        _AlphaScale ("Alpha Scale", Range(0, 1)) = 1
    }
    SubShader {
        Tags {"Queue"="Transparent" "IgnoreProjector"="True" "RenderType"="Transparent"}

        // Extra pass that renders to depth buffer only
        Pass {
            ZWrite On
            ColorMask 0
        } //这个是新加入的Pass

        Pass {
            // 和8.4节同样的代码
        }
    } 
    Fallback "Diffuse"
}
```

**这个新添加的Pass的目的仅仅是为了把模型的深度信息写入深度缓冲中**，从而剔除模型中被自身遮挡的片元(所以后续对于这个模型来说,深度值小的会挡住深度值大的)。因此，Pass的第一行开启了深度写入。在第二行，我们使用了一个新的渲染命令——**ColorMask** 。在`ShaderLab`中，`ColorMask`用于设置颜色通道的写掩码（write mask）。它的语义如下：

`ColorMask RGB | A | 0 | 其他任何R、G、B、A的组合`

**当`ColorMask`设为0时，意味着该Pass不写入任何颜色通道，即不会输出任何颜色。这正是我们需要的——该Pass只需写入深度缓存即可。**

![image-20220223213944029](E:\各种总结指南\技术美术学习\技术美术相关\Unity Shader入门精要阅读\第八章 透明效果.assets\image-20220223213944029.png)

<br/>

#### 8.6 ShaderLab的混合命令

首先来看一下混合是如何实现的。当片元着色器产生一个颜色的时候，可以选择与颜色缓存中的颜色进行混合。这样一来，混合就和两个操作数有关：**源颜色** **（source color）** 和**目标颜色（destination color）** 。

- 源颜色，我们用**S** 表示，指的是由片元着色器产生的颜色值；
- 目标颜色，我们用**D** 表示，指的是从颜色缓冲中读取到的颜色值。
- 对它们进行混合后得到的输出颜色，我们用**O** 表示，它会重新写入到颜色缓冲中。需要注意的是，当我们谈及混合中的源颜色、目标颜色和输出颜色时，它们都包含了RGBA四个通道的值，而并非仅仅是RGB通道。

> 想要使用混合，我们必须首先开启它。在Unity中，当我们使用Blend（Blend Off命令除外）命令时，除了设置混合状态外也开启了混合。**但是，在其他图形API中我们是需要手动开启的。**例如在OpenGL中，我们需要使用glEnable(GL_BLEND)来开启混合。但在Unity中，它已经在背后为我们做了这些工作。

<br/>

### 8.6.1 混合等式和参数

混合是一个逐片元的操作，而且它不是可编程的，但却是高度可配置的。也就是说，我们可以设置混合时使用的运算操作、混合因子等来影响混合。

现在，我们已知两个操作数：源颜色S和目标颜色D，想要得到输出颜色O就必须使用一个等式来计算。我们把这个等式称为**混合等式（blend equation）** 。

当进行混合时，我们需要使用两个混合等式：一个用于混合RGB通道，一个用于混合A通道。当设置混合状态时，我们实际上设置的就是混合等式中的操作和**因子** 。在默认情况下，混合等式使用的操作都是加操作（我们也可以使用其他操作），我们只需要再设置一下混合因子即可。由于需要两个等式（分别用于混合RGB通道和A通道），每个等式有两个因子（一个用于和源颜色相乘，一个用于和目标颜色相乘），因此一共需要4个因子。表8.3给出了ShaderLab中设置混合因子的命令。

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220820123959155.png" alt="image-20220820123959155" style="zoom:80%;" />

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220820124031243.png" alt="image-20220820124031243" style="zoom:80%;" />

那么，这些混合因子可以有哪些值呢？表8.4给出了ShaderLab支持的几种混合因子。

|      参 数       |                            描 述                             |
| :--------------: | :----------------------------------------------------------: |
|       One        |                           因子为1                            |
|       Zero       |                           因子为0                            |
|     SrcColor     | 因子为源颜色值。当用于混合RGB的混合等式时，使用SrcColor的RGB分量作为混合因子；当用于混合A的混合等式时，使用SrcColor的A分量作为混合因子 |
|     SrcAlpha     |               因子为源颜色的透明度值（A通道）                |
|     DstColor     | 因子为目标颜色值。当用于混合RGB通道的混合等式时，使用DstColor的RGB分量作为混合因子；当用于混合A通道的混合等式时，使用DstColor的A分量作为混合因子。 |
|     DstAlpha     |              因子为目标颜色的透明度值（A通道）               |
| OneMinusSrcColor | 因子为(1-源颜色)。当用于混合RGB的混合等式时，使用结果的RGB分量作为混合因子；当用于混合A的混合等式时，使用结果的A分量作为混合因子 |
| OneMinusSrcAlpha |                  因子为(1-源颜色的透明度值)                  |
| OneMinusDstColor | 因子为(1-目标颜色)。当用于混合RGB的混合等式时，使用结果的RGB分量作为混合因子；当用于混合A的混合等式时，使用结果的A分量作为混合因子 |
| OneMinusDstAlpha |                 因子为(1-目标颜色的透明度值)                 |

使用上面的指令进行设置时，RGB通道的混合因子和A通道的混合因子都是一样的，有时我们希望可以使用不同的参数混合A通道，这时就可以利用**Blend SrcFactor DstFactor, SrcFactorA DstFactorA** 指令。例如，如果我们想要在混合后，输出颜色的透明度值就是源颜色的透明度，可以使用下面的命令：

```c++
Blend SrcAlpha OneMinusSrcAlpha, One Zero
```

<br/>

### 8.6.2 混合操作

在上面涉及的混合等式中，当把源颜色和目标颜色与它们对应的混合因子相乘后，我们都是把它们的结果加起来作为输出颜色的。那么可不可以选择不使用加法，而使用减法呢？答案是肯定的，我们可以使用ShaderLab的**BlendOp BlendOperation**(这里的BlendOperation替换为下表当中的Add,Sub等) 命令，即混合操作命令。表8.5给出了ShaderLab中支持的混合操作。

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220820124331286.png" alt="image-20220820124331286" style="zoom: 67%;" />

混合操作命令通常是与混合因子命令一起工作的。但需要注意的是，当使用**Min** 或**Max** 混合操作时，混合因子实际上是不起任何作用的，它们仅会判断原始的源颜色和目的颜色之间的比较结果。

<br/>

##### 8.6.3 常见的混合类型

通过混合操作和混合因子命令的组合，我们可以得到一些类似Photoshop混合模式中的混合效果：

```c++
// 正常（Normal），即透明度混合
Blend SrcAlpha OneMinusSrcAlpha

// 柔和相加（Soft Additive）
Blend OneMinusDstColor One

// 正片叠底（Multiply），即相乘
Blend DstColor Zero

// 两倍相乘（2x Multiply）
Blend DstColor SrcColor

// 变暗（Darken）
BlendOp Min
Blend One One

// 变亮（Lighten）
BlendOp Max
Blend One One

// 滤色（Screen）
Blend OneMinusDstColor One
// 等同于
Blend One OneMinusSrcColor

// 线性减淡（Linear Dodge）
Blend One One
```

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220820124214200.png" alt="image-20220820124214200" style="zoom:67%;" />



## 8.7 双面渲染的透明效果

在现实生活中，如果一个物体是透明的，意味着我们不仅可以透过它看到其他物体的样子，也可以看到它内部的结构。但在前面实现的透明效果中，无论是透明度测试还是透明度混合，我们都无法观察到正方体内部及其背面的形状，导致物体看起来就好像只有半个一样。这是因为，默认情况下渲染引擎剔除了物体背面（相对于摄像机的方向）的渲染图元，而只渲染了物体的正面。如果我们想要得到双面渲染的效果，可以使用**Cull** 指令来控制需要剔除哪个面的渲染图元。在Unity中，Cull指令的语法如下：

```
Cull Back | Front | Off
```

如果设置为Back，那么那些背对着摄像机的渲染图元就不会被渲染，这也是默认情况下的剔除状态；如果设置为Front，那么那些朝向摄像机的渲染图元就不会被渲染；如果设置为**Off** ，就会关闭剔除功能，那么所有的渲染图元都会被渲染，但由于这时需要渲染的图元数目会成倍增加，因此**除非是用于特殊效果，例如这里的双面渲染的透明效果，通常情况是不会关闭剔除功能的。**



### 8.7.1 透明度测试的双面渲染

```c#
Pass {
    Tags { "LightMode"="ForwardBase" }

    // Turn off culling
    Cull Off
```

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220820124403990.png" alt="image-20220820124403990" style="zoom:50%;" />

### 8.7.2　透明度混合的双面渲染

和透明度测试相比，**想要让透明度混合实现双面渲染会更复杂一些**，这是因为透明度混合需要关闭深度写入，而这是“一切混乱的开端”。我们知道，想要得到正确的透明效果，渲染顺序是非常重要的——**我们想要保证图元是从后往前渲染的。**

对于透明度测试来说，由于我们没有关闭深度写入，因此可以利用深度缓冲按逐像素的粒度进行深度排序，从而保证渲染的正确性。然而一旦关闭了深度写入，我们就需要小心地控制渲染顺序来得到正确的深度关系。如果我们仍然采用8.7.1节中的方法，直接关闭剔除功能，那么我们就无法保证同一个物体的正面和背面图元的渲染顺序，就有可能得到错误的半透明效果。

为此，**我们选择把双面渲染的工作分成两个Pass——第一个Pass只渲染背面，第二个Pass只渲染正面，由于Unity会顺序执行SubShader中的各个Pass，因此我们可以保证背面总是在正面被渲染之前渲染，从而可以保证正确的深度渲染关系。**

**核心:保证背面总是在正面之前被渲染,所以分成两波进行渲染(两个Pass)**

```c#
Shader "Chapter 8/AlphaBlendWithBothSide" {
    Properties {
        _Color ("Main Tint", Color) = (1,1,1,1)
        _MainTex ("Main Tex", 2D) = "white" {}
        _AlphaScale ("Alpha Scale", Range(0, 1)) = 1
    }
    SubShader {
        Tags {"Queue"="Transparent" "IgnoreProjector"="True" "RenderType"="Transparent"}

        Pass {
            Tags { "LightMode"="ForwardBase" }

            // First pass renders only back faces 
            Cull Front

            // 和之前一样的代码
        }

        Pass {
            Tags { "LightMode"="ForwardBase" }

            // Second pass renders only front faces 
            Cull Back

            // 和之前一样的代码
        }
    } 
    Fallback "Transparent/VertexLit"
}
```

最终的效果如下:

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220820125031813.png" alt="image-20220820125031813" style="zoom:67%;" />

<br/>

# 第九章 更复杂的光照

在前面的场景中,我们都仅有一个平行光。但实际上做游戏的时候，可能需要很多的光源和不同的类型。还有一点就是，我们希望能得到阴影效果。在此之前，我们有必要了解Unity如何处理这些光源，所以会先介绍渲染路径，然后会使用更多的不同类型光源（如点光源，聚光灯），接下来会介绍如何实现光照衰减和实现阴影效果。

## 1.Unity的渲染路径

> 渲染路径决定了光照是如何应用到Unity Shader当中的。Unity支持多种类型的渲染路径。**大多数情况下，一个项目只使用一种渲染路径，因此可以为整个项目设置渲染路径。**

方法如下：Unity->Edit->Project Settings->Graphics,选择渲染路径（默认情况下，选择的是前向渲染。）

![image-20220612122808455](E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220612122808455.png)

如果想要使用多个渲染路径，比如不同相机渲染的物品用不同的渲染路径，则可以在相机当中进行设置：

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220612205925541.png" alt="image-20220612205925541" style="zoom:80%;" />

注意，如果当前的显卡不支持所选择的渲染路径，则Unity会采用更低一级的渲染路径。

不同类型的渲染路径可以包含不同的标签，比如Pass中，使用前向渲染是ForwardBase，另外也可以是ForwardAdd，下表给了一个Pass中支持的LightMode标签：

|            标 签 名            |                            描 述                             |
| :----------------------------: | :----------------------------------------------------------: |
|             Always             | 不管使用哪种渲染路径，该Pass总是会被渲染，但不会计算任何光照 |
|          ForwardBase           | 用于**前向渲染** 。该Pass会计算环境光、最重要的平行光、逐顶点/SH光源和Lightmaps |
|           ForwardAdd           | 用于**前向渲染** 。该Pass会计算额外的逐像素光源，每个Pass对应一个光源 |
|            Deferred            |       用于**延迟渲染** 。该Pass会渲染G缓冲（G-buffer）       |
|          ShadowCaster          | 把物体的深度信息渲染到阴影映射纹理（shadowmap）或一张深度纹理中 |
|          PrepassBase           | 用于**遗留的延迟渲染** 。该Pass会渲染法线和高光反射的指数部分 |
|          PrepassFinal          | 用于**遗留的延迟渲染** 。该Pass通过合并纹理、光照和自发光来渲染得到最后的颜色 |
| Vertex、VertexLMRGBM和VertexLM |                  用于**遗留的顶点照明渲染**                  |

接下来将会介绍不同的渲染路径。

<br/>

## 2.前向渲染路径

这是我们最常使用的一种渲染路径。

### 2.1 前向渲染的原理

> 每进行一次完整的前向渲染，我们需要渲染该对象的渲染图元，并计算颜色和深度缓冲区的信息。（深度缓冲决定一个片元是否可见，如果可见就更新缓冲区中的颜色值）。伪代码如下：

```c#
Pass {
    for (each primitive in this model) {
        for (each fragment covered by this primitive) {
            if (failed in depth test) {
                // 如果没有通过深度测试，说明该片元是不可见的
                discard;
            } else {
                // 如果该片元可见
                // 就进行光照计算
                float4 color = Shading(materialInfo, pos, normal, lightDir, viewDir);
                // 更新帧缓冲
                writeFrameBuffer(fragment, color);
            }
        }
    }
}
```

对于每个逐像素光源，我们都需要进行上面一次完整的渲染流程。如果一个物体在多个逐像素光源的影响区域内，那么该物体就需要执行多个Pass，每个Pass计算一个逐像素光源的光照结果，然后在帧缓冲中把这些光照结果混合起来得到最终的颜色值。假设，场景中有*N* 个物体，每个物体受*M* 个光源的影响，那么要渲染整个场景一共需要N*M 个Pass。可以看出，如果有大量逐像素光照，那么需要执行的Pass数目也会很大。因此，渲染引擎通常会限制每个物体的逐像素光照的数目。

<br/>

### 2.2 Unity中的前向渲染

一个Pass不仅仅用来计算逐像素光照，也可以用来计算逐顶点等其他光照。而这取决于光照计算所处的流水线阶段和使用的数学模型。Unity中，前向渲染有三种处理光照（即照亮物体）的方式：

- 逐顶点处理
- 逐像素处理
- 球谐函数处理（SH）

决定一个光源使用哪种处理模式取决于它的类型（比如平行光或其他）和渲染模式（是否为Important，如果是的话就是希望Unity能够重视，把他当作一个逐像素光源来处理）。

在下图所示位置可以进行调整：

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220612211346489.png" alt="image-20220612211346489" style="zoom: 80%;" />

在前向渲染当中，**渲染时Unity会根据场景中各个光源的设置以及这些光源对物体的影响程度（例如，距离该物体的远近、光源强度等）对这些光源进行一个重要度排序。其中，一定数目的光源会按逐像素的方式处理，然后最多有4个光源按逐顶点的方式处理，剩下的光源可以按SH方式处理。Unity使用的判断规则如下。**

- 场景中最亮的平行光按逐像素处理的；
- 渲染模式设置成Important的，按照逐像素处理；
- Not Important的，按照逐顶点或者SH处理；
- 如果根据以上规则得到的逐像素光源数量小于Project Settings中Quality中的逐像素光源数量（Pixel Light Count），则会有更多的光源用逐像素的方式渲染。

在Pass中进行光照计算。前向渲染有两种Pass：Base Pass和Additional Pass。相关标签和渲染设置以及常规光照计算如下：

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220612212158426-16550402261981.png" alt="image-20220612212158426" style="zoom:80%;" />

有几点说明如下：

- （1）#pragma指令是需要的，使用这两个编译指令才能在相关的Pass中得到一些正确的光照变量，比如光照衰减等。

- （2）Base Pass中，我们可以访问光照纹理，这里面渲染的平行光默认是支持阴影的（如果开启了光源的阴影功能），而Additional Pass中渲染的光源默认情况下没有阴影，即使在Light组件中设置了有阴影的Shadow Type。可以如上图所示用#pragma multi_compile_ fwdadd_fullshadows来为点光源和聚光灯开启阴影效果，但这需要Unity在内部使用更多的Shader。
- （3）环境光和自发光在Base Pass中计算，因为这两个我们只希望计算一次；
- （4）Additional Pass的渲染设置中，我们还开启和设置了混合模式，这是因为，我们希望每个Additional Pass可以与上一次的光照结果在帧缓存中进行叠加，从而得到最终的有多个光照的渲染效果。如果我们没有开启和设置混合模式，那么Additional Pass的渲染结果会覆盖掉之前的渲染结果，看起来就好像该物体只受该光源的影响。通常情况下，我们选择的混合模式是**Blend One One** 。
- （5）对于前向渲染来说，一个Unity Shader通常会定义一个Base Pass（Base Pass也可以定义多次，例如需要双面渲染等情况）以及一个Additional Pass。**一个Base Pass仅会执行一次（定义了多个Base Pass的情况除外），而一个Additional Pass会根据影响该物体的其他逐像素光源的数目被多次调用，即每个逐像素光源会执行一次Additional Pass。**

上面的图只是通常情况下我们在每种Pass中进行的计算，实际上也可以自己写别的。

<br/>关于前向渲染中可以使用的内置光照函数和变量，可以查看书第185页的表格，在这里不做过多展开了。

**注：有一种顶点照明渲染应该不太用了，就不整理了。**

<br/>

## 3.延迟渲染路径（Deferred）

前向渲染会有一个问题，就是场景中包含大量实时光源的时候，前向渲染的性能会急速下降。比如我们放了一大堆的光源互相重叠，那么此时该区域里每个物体都要执行多个Pass来计算不同光源的影响结果，还要在颜色缓存中把这些结果混合起来得到最后结果。此时每执行一个Pass我们就要重新渲染一遍物体，但很多计算是重复的。

**延迟渲染中，除了前向渲染当中的颜色缓冲和深度缓冲外，还会额外利用一个G缓冲区（G是Geometry）**，G缓冲区存储了我们所关心的表面（通常指的是离摄像机最近的表面）的其他信息，比如该表面的法线，位置，用于光照计算的材质属性等。

### 3.1 延迟渲染原理

> 延迟渲染主要包含了两个Pass。**在第一个Pass中，我们不进行任何光照计算，而是仅仅计算哪些片元是可见的，这主要是通过深度缓冲技术来实现，当发现一个片元是可见的，我们就把它的相关信息存储到G缓冲区中。然后，在第二个Pass中，我们利用G缓冲区的各个片元信息，例如表面法线、视角方向、漫反射系数等，进行真正的光照计算。**
>
> 延迟渲染的过程大致可以用下面的伪代码来描述：

```glsl
Pass 1 {
    // 第一个Pass不进行真正的光照计算
    // 仅仅把光照计算需要的信息存储到G缓冲中

    for (each primitive in this model) {
        for (each fragment covered by this primitive) {
            if (failed in depth test) {
                // 如果没有通过深度测试，说明该片元是不可见的
                discard;
            } else {
                // 如果该片元可见
                // 就把需要的信息存储到G缓冲中
                writeGBuffer(materialInfo, pos, normal, lightDir, viewDir);
            }
        }
    }
}

Pass 2 {
    // 利用G缓冲中的信息进行真正的光照计算

    for (each pixel in the screen) {
        if (the pixel is valid) {
            // 如果该像素是有效的
            // 读取它对应的G缓冲中的信息
            readGBuffer(pixel, materialInfo, pos, normal, lightDir, viewDir);

            // 根据读取到的信息进行光照计算
            float4 color = Shading(materialInfo, pos, normal, lightDir, viewDir);
            // 更新帧缓冲
            writeFrameBuffer(pixel, color);
        }
    }
}
```

可以看到，延迟渲染使用的Pass数量基本就两个，跟场景光源数目没有关系。也就是说，延迟渲染的效率不依赖于场景的复杂度，而是和屏幕空间的大小有关。因为我们需要的信息都存储在缓冲区中，而这些缓冲区可以理解成一张张2D图像，计算实际上是在这些图像空间中进行的。

对于延迟渲染路径来说，**它最适合在场景中光源数目很多、如果使用前向渲染会造成性能瓶颈的情况下使用。而且，延迟渲染路径中的每个光源都可以按逐像素的方式处理。**但是，延迟渲染也有一些缺点。

- 不支持真正的抗锯齿（anti-aliasing）功能。
- 不能处理半透明物体。
- 对显卡有一定要求。如果要使用延迟渲染的话，显卡必须支持MRT（Multiple Render Targets）、Shader Mode 3.0及以上、深度渲染纹理以及双面的模板缓冲。

当使用延迟渲染时，**Unity要求我们提供两个Pass。**

（1）第一个Pass用于渲染G缓冲。在这个Pass中，我们会把物体的漫反射颜色、高光反射颜色、平滑度、法线、自发光和深度等信息渲染到屏幕空间的G缓冲区中。对于每个物体来说，这个Pass仅会执行一次。

（2）第二个Pass用于计算真正的光照模型。这个Pass会使用上一个Pass中渲染的数据来计算最终的光照颜色，再存储到帧缓冲中。

默认的G缓冲区（注意，不同Unity版本的渲染纹理存储内容会有所不同）包含了以下几个渲染纹理（Render Texture，RT）。

- RT0：格式是ARGB32，RGB通道用于存储漫反射颜色，A通道没有被使用。
- RT1：格式是ARGB32，RGB通道用于存储高光反射颜色，A通道用于存储高光反射的指数部分。
- RT2：格式是ARGB2101010，RGB通道用于存储法线，A通道没有被使用。
- RT3：格式是ARGB32（非HDR）或ARGBHalf（HDR），用于存储自发光+lightmap+反射探针（reflection probes）。
- 深度缓冲和模板缓冲。

当在第二个Pass中计算光照时，默认情况下仅可以使用Unity内置的Standard光照模型。如果我们想要使用其他的光照模型，就需要替换掉原有的Internal-DeferredShading.shader文件。更详细的信息可以访问官方文档（http://docs.unity3d.com/Manual/RenderTech-DeferredShading.html ）。

关于延迟渲染可以使用的变量，可以参考书的第188页，这本书后面主要使用的是**前向渲染路径**。

<br/>

## 4.Unity的光源类型

本节将会学习如何在Unity当中处理点光源和聚光灯。

注意：**面光源仅在烘焙的时候才可发挥作用，因此不再本节讨论范围。**

### 4.1 光源类型有什么影响

要考虑光源的位置，方向（到某点），颜色，强度以及衰减（到某点的衰减）。

#### （1）平行光

平行光可以照亮的范围没有限制，通常作为太阳这种。平行光之所以简单，是因为它可以放在场景中的任意位置，几何属性只有方向。（可以调整Transform的Rotation属性）。平行光到场景中所有点的方向都是一样的，也没有衰减的概念。

#### （2）点光源

由空间中的一个球体定义。可以表示由一个点发出的，向所有方向延申的光。对应属性如下：

![image-20220612220026723](E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220612220026723.png)

请注意，**如果我们想要在Scene窗口中看到光源的效果，记得打开Gizmo当中的光照。**球体半径可以用Range属性或者直接拖来修改。点光源有位置属性，方向是点光源的位置减去某点的位置，点光源的颜色和强度可以在Light组件中调整。**点光源会衰减，球心处强度最强，边界处最弱（为0）**.衰减效果可以用函数定义。

#### （3）聚光灯（Spot Light）

照亮空间不再是球体，而是由空间中一块锥形区域定义的。聚光灯可以用于表示从一个特定位置出发，向特定方向延伸的光。效果如下：

![image-20220612220634121](E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220612220634121.png)

Range属性或者拖拽可以控制锥形区域的半径，锥体的张开角度由Spot Angle属性决定。同样也可以控制位置和方向，同样也有衰减（**衰减函数相对于点光源更为复杂**）

<br/>

### 4.2 在前向渲染当中处理不同的光源类型

之前说了，光源要考虑光源的位置，方向（到某点），颜色，强度以及衰减（到某点的衰减），接下来的Shader会说明如何使用这几个属性：（**注意，建立在前向渲染之上,具体说明见代码后面**）

```glsl
Shader "Chapter 9/ForwardRendering"
{
   	Properties {
		_Diffuse ("Diffuse", Color) = (1, 1, 1, 1)
		_Specular ("Specular", Color) = (1, 1, 1, 1)
		_Gloss ("Gloss", Range(8.0, 256)) = 20
	}
	SubShader {
		Tags { "RenderType"="Opaque" }
		
		Pass {
			// Pass for ambient light & first pixel light (directional light)
			Tags { "LightMode"="ForwardBase" }
		
			CGPROGRAM
			
			// Apparently need to add this declaration 
			#pragma multi_compile_fwdbase	
			
			#pragma vertex vert
			#pragma fragment frag
			
			#include "Lighting.cginc"
			
			fixed4 _Diffuse;
			fixed4 _Specular;
			float _Gloss;
			
			struct a2v {
				float4 vertex : POSITION;
				float3 normal : NORMAL;
			};
			
			struct v2f {
				float4 pos : SV_POSITION;
				float3 worldNormal : TEXCOORD0;
				float3 worldPos : TEXCOORD1;
			};
			
			v2f vert(a2v v) {
				v2f o;
				o.pos = UnityObjectToClipPos(v.vertex);
				o.worldNormal = UnityObjectToWorldNormal(v.normal);
				o.worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;
				return o;
			}
			
			fixed4 frag(v2f i) : SV_Target {
				fixed3 worldNormal = normalize(i.worldNormal);
				fixed3 worldLightDir = normalize(_WorldSpaceLightPos0.xyz);
				
				fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;
				
			 	fixed3 diffuse = _LightColor0.rgb * _Diffuse.rgb * max(0, dot(worldNormal, worldLightDir));

			 	fixed3 viewDir = normalize(_WorldSpaceCameraPos.xyz - i.worldPos.xyz);
			 	fixed3 halfDir = normalize(worldLightDir + viewDir);
			 	fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(max(0, dot(worldNormal, halfDir)), _Gloss);

				fixed atten = 1.0;
				
				return fixed4(ambient + (diffuse + specular) * atten, 1.0);
			}
			
			ENDCG
		}
	
		Pass {
			// Pass for other pixel lights
			Tags { "LightMode"="ForwardAdd" }
			
			Blend One One  //如果没有blend，则additional pass会直接覆盖掉之前的光照结果，其他情况也可以用blend srcalpha one等
		
			CGPROGRAM
			
			// Apparently need to add this declaration
			#pragma multi_compile_fwdadd
			
			#pragma vertex vert
			#pragma fragment frag
			
			#include "Lighting.cginc"
			#include "AutoLight.cginc"
			
			fixed4 _Diffuse;
			fixed4 _Specular;
			float _Gloss;
			
			struct a2v {
				float4 vertex : POSITION;
				float3 normal : NORMAL;
			};
			
			struct v2f {
				float4 pos : SV_POSITION;
				float3 worldNormal : TEXCOORD0;
				float3 worldPos : TEXCOORD1;
			};
			
			v2f vert(a2v v) {
				v2f o;
				o.pos = UnityObjectToClipPos(v.vertex);
				
				o.worldNormal = UnityObjectToWorldNormal(v.normal);
				
				o.worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;
				
				return o;
			}
			
			fixed4 frag(v2f i) : SV_Target {
				fixed3 worldNormal = normalize(i.worldNormal);
				#ifdef USING_DIRECTIONAL_LIGHT
					fixed3 worldLightDir = normalize(_WorldSpaceLightPos0.xyz);
				#else
					fixed3 worldLightDir = normalize(_WorldSpaceLightPos0.xyz - i.worldPos.xyz);
				#endif
				
				fixed3 diffuse = _LightColor0.rgb * _Diffuse.rgb * max(0, dot(worldNormal, worldLightDir));
				
				fixed3 viewDir = normalize(_WorldSpaceCameraPos.xyz - i.worldPos.xyz);
				fixed3 halfDir = normalize(worldLightDir + viewDir);
				fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(max(0, dot(worldNormal, halfDir)), _Gloss);
				
				#ifdef USING_DIRECTIONAL_LIGHT
					fixed atten = 1.0;
				#else
					#if defined (POINT)
				        float3 lightCoord = mul(unity_WorldToLight, float4(i.worldPos, 1)).xyz;
				        fixed atten = tex2D(_LightTexture0, dot(lightCoord, lightCoord).rr).UNITY_ATTEN_CHANNEL;
				    #elif defined (SPOT)
				        float4 lightCoord = mul(unity_WorldToLight, float4(i.worldPos, 1));
				        fixed atten = (lightCoord.z > 0) * tex2D(_LightTexture0, lightCoord.xy / lightCoord.w + 0.5).w * tex2D(_LightTextureB0, dot(lightCoord, lightCoord).rr).UNITY_ATTEN_CHANNEL;
				    #else
				        fixed atten = 1.0;
				    #endif
				#endif

				return fixed4((diffuse + specular) * atten, 1.0);
			}
			
			ENDCG
		}
	}
	FallBack "Specular"
}

```

一些说明：

- （1）记得每个pass的#pragma指令不要忘了写；
- （2）base pass当中，我们计算了环境光（默认按没有自发光计算，理论上自发光也要算进来）。
- （3）在base pass当中处理了场景中最重要的平行光（如果有多个平行光，Unity**会选择最亮的平行光传递给base pass进行逐像素处理，其他平行光会逐顶点或者在additional pass中按逐像素的方式处理。**）如果场景中没有任何平行光，那么base pass会当成全黑的光源处理。
- （4）**对于Base Pass来说，它处理的逐像素光源类型一定是平行光。**我们可以使用  _WorldSpaceLightPos0来得到这个平行光的方向（位置对平行光来说没有意义），使用  _LightColor0来得到它的颜色和强度（ _LightColor0已经是颜色和强度相乘后的结果），由于平行光可以认为是没有衰减的，因此这里我们直接令衰减值为1.0（乘完1.0之后相当于没衰减）。
- （5）在Addtional Pass当中，我们同样通过判断是否定义了USING_DIRECTIONAL_LIGHT来决定当前处理的光源类型。如果是平行光的话，衰减值为1.0。如果是其他光源类型，那么处理更复杂一些。**尽管我们可以使用数学表达式来计算给定点相对于点光源和聚光灯的衰减，但这些计算往往涉及开根号、除法等计算量相对较大的操作，因此Unity选择了使用一张纹理作为查找表（Lookup Table，LUT），以在片元着色器中得到光源的衰减。**我们首先得到光源空间下的坐标，然后使用该坐标对衰减纹理进行采样得到衰减值。关于Unity中衰减纹理的细节可以参见后面的光照衰减部分。
- （6）unity_WorldToLight这个矩阵是从世界空间到光源空间的变换矩阵。可以用于采样cookie和光强衰减纹理，用于点光源和聚光灯（参考上面的代码）

可以进行实验，来观察最终的效果：

![image-20220612225134243](E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220612225134243.png)

在调试和测试运行的时候，可以通过给不同的光源不同的颜色来方便观察效果：

![image-20220612230457084](E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220612230457084.png)

<br/>

### 4.3 尝试：利用VS2019进行Debug观察绘制结果(后面介绍Renderdoc)

回顾一下之前所描述的VS调试Shader,如果忘记了可以参考教程:

[(16条消息) VS2017下用Graphics Debugger调试UnityShader_带帯大师兄的博客-CSDN博客](https://blog.csdn.net/qq_29523119/article/details/78409811)

**关于这个调试的一些细节,以及相关的信息,在这里由于能力关系暂时没法很好地进行总结,后续学会了有空再进行展开;**

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220619233958724.png" alt="image-20220619233958724" style="zoom:67%;" />

Unity处理这些点光源的顺序是按照重要度排序的.如果所有的点光源颜色强度相同,则重要度取决于距离胶囊体的远近.首先绘制的是离胶囊体最近的点光源.当然如果强度和颜色不相同,那么距离就不是唯一标准.(在Unity 5中还没有给出Unity如何进行排序,因此我们只知道和这几个有关.)

对于场景中一个物体,如果不在光源光照范围内,则Unity不会调用Pass来处理.如果把光源的Render Mode设置为Not Important,则不希望Unity把该光源当成逐像素处理.

<br/>

### 4.4 Unity的光照衰减

Unity会使用一张纹理作为查找表,来在片元着色器中计算逐像素光照的衰减.这样就不会依赖于数学公式的复杂性,而只需要一个参数区纹理中采样.不过也有一些弊端:

- 需要预处理,且纹理大小也会影响衰减精度;
- 不直观也不方便,不太好用其他数学公式计算衰减.

不过纹理查找可以一定程度上提升性能,且大多数情况下还不错,所以Unity默认用这种方式**来计算逐像素的点光源和聚光灯的衰减.**



#### 4.4.1 用于光照衰减的纹理

Unity内部使用_LightTexture0的纹理来计算光照衰减.

(如果对光源使用了cookie,那么衰减查找纹理是_LightTextureB0,在这里就不讨论了.)

一般只关注_LightTexture0对角线上的纹理颜色值,**而这些表明了在光源空间中不同位置的点的衰减值.**比如(0,0)表示与光源重合的点的衰减值,(1,1)表示最远的点的衰减.

为了对 _LightTexture0纹理采样得到给定点到该光源的衰减值，我们首先需要得到该点在光源空间中的位置，这是通过unity_WorldToLight变换矩阵得到的。

然后,我们可以利用这个得到的坐标的模的平方来对衰减纹理进行采样,从而得到衰减值,代码如下:

```glsl
float3 lightCoord = mul(unity_WorldToLight, float4(i.worldPos, 1)).xyz;
fixed atten = tex2D(_LightTexture0, dot(lightCoord, lightCoord).rr).UNITY_ATTEN_CHANNEL; //dot出来的是一个向量，rr中的每一个r是rgba中的第一个元素，shader中就是所得向量的第一元素了（如果是二维向量的话就只有r和g有值了），rr的话就是相当于v.vertex.xx这样，获得一个由两个第一个元素组成的二维数,比如(1,2,3).rr=(1,1)
```

在上面的代码中，我们使用了光源空间中顶点距离的平方（通过dot函数来得到）来对纹理采样，之所以没有使用距离值来采样是因为这种方法可以避免开方操作。然后，我们使用宏UNITY_ATTEN_CHANNEL来得到衰减纹理中衰减值所在的分量，以得到最终的衰减值。

(UNITY_ATTEN_CHANNEL是衰减值所在的纹理通道，可以在内置的HLSLSupport.cginc文件中查看。一般PC和主机平台的话UNITY_ATTEN_CHANNEL是r通道，移动平台的话是a通道)

<br/>

关于聚光灯,这段代码如下:

```glsl
float4 lightCoord = mul(unity_WorldToLight, float4(i.worldPos, 1));
fixed atten = (lightCoord.z > 0) * tex2D(_LightTexture0, lightCoord.xy / lightCoord.w + 0.5).w * tex2D(_LightTextureB0, dot(lightCoord, lightCoord).rr).UNITY_ATTEN_CHANNEL;
```

与点光源不同，由于聚光灯有更多的角度等要求，因此为了得到衰减值，除了需要对衰减纹理采样外，还需要对聚光灯的范围、张角和方向进行判断.此时衰减纹理存储到了 _LightTextureB0中，这张纹理和点光源中的 _LightTexture0是等价的.

聚光灯的_LightTexture0存储的不再是基于距离的衰减纹理，而是一张基于张角范围的衰减纹理



#### 4.4.2 自己设置衰减函数

这里举一个例子:

```glsl
float distance = length(_WorldSpaceLightPos0.xyz - i.worldPosition.xyz);
atten = 1.0 / distance; // linear attenuation
```

这里只是应用了一个简单的衰减函数,复杂一些的还得等后面学习了再做整理.

<br/>

## 5.Unity的阴影

### 5.1 阴影是如何实现的

shadow map的示意图如下(图源见水印):

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220820204454533.png" alt="image-20220820204454533" style="zoom: 67%;" />

实时渲染中,最常用的是shadow map技术.他会先把摄像机的位置放在与光源重合的位置上,那么直观理解阴影区就是"光源看不到的地方".在前向渲染路径中，如果场景中最重要的平行光开启了阴影，Unity就会为该光源计算它的阴影映射纹理（shadowmap）。这张阴影映射纹理本质上也是一张深度图，它记录了从该光源的位置出发、能看到的场景中距离它最近的表面位置（深度信息）。

Unity在计算Shadow Map的时候,Unity选择使用一个额外的Pass来专门更新光源的阴影映射纹理，这个Pass就是**LightMode** 标签被设置为**ShadowCaster** 的Pass,这个Pass的渲染目标就是shadow map(或叫深度纹理).

首先,Unity先把相机的位置放置在光源上,然后调用调用上述Pass,通过顶点变换得到**光源空间下的位置**,并据此输出深度信息到shadow map当中.因此,当开启光源的阴影效果后,引擎会首先去寻找这个标签被设置为ShadowCaster的Pass(如果找不到会到Fallback里去找),如果没有找到,则**该物体无法向其他物体投射阴影(但仍可接受来自其他物体的阴影)**.找到ShadowCaster之后,Unity会使用该Pass来更新光源的Shadow map.

传统的shadow map实现中,会在正常渲染时把顶点转换到光源空间下,从而得到其在光源空间的三维坐标信息,接下来使用x,y坐标对shadow map采样,得到shadow map中该点的深度信息,与该点转换后的深度值z进行比较,如果记录的z值<该点的z值,则这个点处于阴影当中.**在Unity 5当中有使用一种不同的,基于屏幕空间的shadow map技术**(screenspace shadow map,注意这种技术并不一定在早版本移动端支持.)

Screenspace Shadow Map的基本思路如下:

> Unity会首先调用LightMode被设置为ShadowCaster的Pass,从而得到可以投射阴影的光源的shadow map以及摄像机的深度纹理.然后,根据光源的shadow map和摄像机的深度纹理计算得到屏幕空间的阴影图.如果摄像机的深度图中记录的表面深度大于转换到shadow map中的深度值，就说明该表面虽然是可见的，但是却处于该光源的阴影中.这样,阴影图中就包含了屏幕空间中所有有阴影的区域.**如果我们想要一个物体接收来自其他物体的阴影,只需要对阴影图进行采样即可.**由于阴影图是屏幕空间下的，因此，我们首先需要把表面坐标从模型空间变换到屏幕空间中，然后使用这个坐标对阴影图进行采样。

总结一下一个物体向其他物体投射阴影和接收其他物体的阴影,是两个过程:

> - 如果我们想要一个物体接收来自其他物体的阴影，就必须在Shader中对阴影映射纹理（包括屏幕空间的阴影图）进行采样，把采样结果和最后的光照结果相乘来产生阴影效果。
> - 如果我们想要一个物体向其他物体投射阴影，就必须把该物体加入到光源的shadow map的计算中，从而让其他物体在对阴影映射纹理采样时可以得到该物体的相关信息。在Unity中，这个过程是通过为该物体执行**LightMode** 为**ShadowCaster** 的Pass来实现的。如果使用了屏幕空间的投影映射技术，Unity还会使用这个Pass产生一张摄像机的深度纹理。

接下来会在Unity当中实现.



### 5.2 不透明物体的阴影

#### (1)让物体投射阴影

新建一个场景,去掉天空盒,搭建如下:

![image-20220820190529108](E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220820190529108.png)

这里注意要把平面的Lighting属性改为Two Sided,否则可能无法向正方体方向投射阴影.可以看到阴影在Unity当中的物体的Lighting属性中,Cast Shadows是投射阴影,Receive Shadows则是接收阴影.同时我们还要点击光源,设置阴影属性为软阴影.

> 如果开启Cast Shadows,那么Unity就会把该物体加入到光源的Shadow Map中计算,这是通过执行**LightMode** 为**ShadowCaster** 的Pass来实现的.如果没有开启Receive Shadows,那么Unity在执行Shader的时候就不会执行计算阴影的宏和变量.

新建一个Shader,将之前的ForwardBase.shader赋值过来,发现虽然没有"LightMode"="ShadowCaster"仍然可以**投射阴影**(但观察到不能接受阴影,这个后面会说),这是因为Fallback指定的Shader当中有相关的LightMode标签.这样最后会回调到Fallback VertexLit当中(在NormalVertexLit.shader文件当中),通过查看这个文件可以看到shader:

```glsl
// Pass to render object as a shadow caster
Pass {
Name "ShadowCaster"
Tags { "LightMode" = "ShadowCaster" }

CGPROGRAM
#pragma vertex vert
#pragma fragment frag
#pragma target 2.0
#pragma multi_compile_shadowcaster
#pragma multi_compile_instancing // allow instanced shadow pass for most of the shaders
#include "UnityCG.cginc"

struct v2f {
    V2F_SHADOW_CASTER;
    UNITY_VERTEX_OUTPUT_STEREO
};

v2f vert( appdata_base v )
{
    v2f o;
    UNITY_SETUP_INSTANCE_ID(v);
    UNITY_INITIALIZE_VERTEX_OUTPUT_STEREO(o);
    TRANSFER_SHADOW_CASTER_NORMALOFFSET(o)
    return o;
}

float4 frag( v2f i ) : SV_Target
{
    SHADOW_CASTER_FRAGMENT(i)
}
ENDCG
}
```

上面这个Pass的渲染目标可以是光源的shadow map,或者摄像机的深度纹理.**今后书写shader的时候建议还是开启内置的一些fallback,可以方便我们的日常shader编写**.



#### (2)让物体接收阴影

在ForwardBase.shader的基础上,代码中新声明一个内置文件,并做出如下修改:

```glsl
#include "AutoLight.cginc"

struct v2f {
	float4 pos : SV_POSITION;
	float3 worldNormal : TEXCOORD0;
	float3 worldPos : TEXCOORD1;
	SHADOW_COORDS(2) //声明一个用于对纹理阴影采样的坐标,参数是下一个可用的插值寄存器的索引值,本例中是2
};
v2f vert(a2v v) {
    // ...
    //Pass shadow coordinates to pixel Shader
    TRANSFER_SHADOW(o);  //这个宏用于在顶点着色器中计算上面声明的阴影纹理坐标
	return o; 
}
fixed4 frag(v2f i) : SV_Target {
	//...
	fixed shadow = SHADOW_ATTENUATION(i); 
    //...
	return fixed4(ambient + (diffuse + specular) * atten * shadow, 1.0);
}
```

注:

> **SHADOW_COORDS** 、**TRANSFER_SHADOW** 和**SHADOW_ATTENUATION** 是计算阴影时的重要组成部分,具体的声明可以到`Autolight.cginc`文件里去看.该文件中Unity为不同的光源类型,不同平台定义了多个版本的宏.
>
> 在前向渲染中:
>
> - 宏**SHADOW_COORDS** 实际上就是声明了一个名为`_ShadowCoord`的阴影纹理坐标变量。
> - **TRANSFER_SHADOW** 的实现依赖于不同平台。
>   - 如果当前平台可以使用屏幕空间的阴影映射技术（通过判断是否定义了**UNITY_NO_SCREENSPACE_ SHADOWS** 来得到），**TRANSFER_SHADOW** 会调用内置的ComputeScreenPos函数来计算`_ShadowCoord`；
>   - 如果该平台不支持屏幕空间的阴影映射技术，就会使用传统的阴影映射技术，**TRANSFER_SHADOW** 会把顶点坐标从模型空间变换到光源空间后存储到`_ShadowCoord`中。然后，SHADOW_ATTENUATION负责使用_ShadowCoord对相关的纹理进行采样，得到阴影信息。
>
> 注意到，内置代码的最后定义了在关闭阴影时的处理代码。可以看出，当关闭了阴影后，**SHADOW_COORDS** 和**TRANSFER_SHADOW** 实际没有任何作用，而**SHADOW_ATTENUATION** 会直接等同于数值1。

注意,以上的宏会基于上下文变量来进行相关计算,所以为了保证运行的正确,我们尽量让自定义的变量名和宏中的变量名相匹配.比如v2f中的pos,a2v结构体中的vertex,以及顶点着色器的输出结构体v2f需要命名成v. 以上我们只对Base Pass进行修改,实际上Additional Pass的修改也是类似的.

使用帧调试器查看Shader渲染过程在后续的Renderdoc文档中进行介绍.



### 5.3 统一管理光照衰减和阴影

注意到上面片元着色器的最后一句:

```glsl
return fixed4(ambient + (diffuse + specular) * atten * shadow, 1.0);
```

可以看到,attenuation和shadow对渲染结果的影响是有共通之处的,Unity提供了内置的`UNITY_LIGHT_ATTENUATION`宏来将两者综合到一起.示例如下:

```glsl
#include "Lighting.cginc"
#include "Autolight.cginc"
struct v2f {
	float4 pos : SV_POSITION;
	float3 worldNormal : TEXCOORD0;
	float3 worldPos : TEXCOORD1;
	SHADOW_COORDS(2)
};
    
v2f vert(a2v v) {
    // ...
    //Pass shadow coordinates to pixel Shader
    TRANSFER_SHADOW(o);  //这个宏用于在顶点着色器中计算上面声明的阴影纹理坐标
	return o; 
}

fixed4 frag(v2f i) : SV_Target {
	//...
	UNITY_LIGHT_ATTENUATION(atten,i,i.worldPos);
    //...
	return fixed4(ambient + (diffuse + specular) * atten, 1.0);
}
```

`UNITY_LIGHT_ATTENUATION`是Unity内置的计算光照衰减和阴影的宏,同样可以在`AutoLight.cginc`中找到声明.该宏接收3个参数:

- 第一个参数用于接收光照衰减和阴影值相乘的结果.
  - 但在这里,我们并没有声明atten,这是因为UNITY_LIGHT_ATTENUATION会帮我们声明;
- 第二个参数是v2f,会传递给5.2(2)所说的`SHADOW_ATTENUATION`,从而用来计算阴影值;
- 第三个参数是世界空间的坐标,用于计算光源空间下的坐标,再对光照衰减纹理采样来得到光照衰减.

Unity会针对不同的光源类型,是否启用cookie来声明多个版本的`UNITY_LIGHT_ATTENUATION`.另外,如果我们想要在Addition

 Pass中添加阴影效果,需要使用`#pragma multi_compile_fwdadd_fullshadows`来代替`\#pragma multi_compile_fwdadd`.



### 5.4 透明度物体的阴影

对于透明物体来说,我们要小心设置Fallback,不能用简单的VertexLit进行调用.对于透明度测试的处理比较简单.但如果直接使用VertexLit,Specular等Fallback往往无法得到正确阴影(这是因为透明度测试需要在片元着色器中舍弃某些片元,但VertexLit中的阴影投射没有这一步操作.)不妨做一个测试,将透明度测试的代码拷贝到一个新的Shader当中,然后增加以下内容:

```glsl
#include "Lighting.cginc"
#include "Autolight.cginc"
struct v2f{
	float4 pos:SV_POSITION;
	float3 worldNormal:TEXCOORD0;
	float3 worldPos:TEXCOORD1; //worldNormal和worldPos是用于后续在片元着色器当中计算光照的diffuse项的
	float2 uv:TEXCOORD2;
	SHADOW_COORDS(3)  //这里是3,因为TEXCOORD0,1,2都用完了,接下来用3
};

    
v2f vert(a2v v) {
    // ...
    //Pass shadow coordinates to pixel Shader
    TRANSFER_SHADOW(o);  //这个宏用于在顶点着色器中计算上面声明的阴影纹理坐标
	return o; 
}

fixed4 frag(v2f i) : SV_Target {
	//...
	UNITY_LIGHT_ATTENUATION(atten,i,i.worldPos);
    //...
	return fixed4(diffuse*atten+ambient, 1.0);
}
Fallback "Transparent/Cutout/VertexLit"
```

两点说明:

- 1.如果Fallback是Vertexlit这种,则无法看到正确结果,因为这个Pass没有进行透明度测试的相关计算.所以要使用Transparent/Cutout/VertexLit,见后面的对比图;
- 2.如果使用了Transparent/Cutout/VertexLit还是没有效果,**请注意是否是因为Transparent/Cutout/VertexLit计算透明度测试的时候需要使用_Cutoff属性,这里命名需要保持正确.**

两种Fallback的对比如下:

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220820212207886.png" alt="image-20220820212207886" style="zoom: 67%;" />

这里还有问题,会出现一些不应该透过光的部分,这是因为默认情况下把物体渲染到深度图和shadow map中时只会考虑物体的正面,但本例的正方体有一些面完全背对光源,因此这些面的深度信息没有加入到shadow map的计算当中.解决方案是将cast shadows属性设置为**Two Sided即可**.下图给出了设置Two Sided前后的对比(左:设置后,右:设置前):

![image-20220820212759693](E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220820212759693.png)

补充:

>  如果是透明度混合添加阴影,会更加复杂一些.因为所有内置的透明度混合相关的Unity Shader都没有包含阴影投射的Pass,这些半透明物体不会参与深度图和阴影映射纹理的计算,既不会投射阴影,也不会接收其他物体投射的阴影.**这是由于透明度混合关闭了深度写入,因此带来的问题也影响阴影的生成.**要想为半透明物体产生正确阴影,需要在每个光源空间下仍然严格从后往前渲染,这就很复杂了.可以通过将Fallback设置为VertexLit,Diffuse这种来强制生成阴影,不过结果也不完全是正确的(见下图右侧结果).一些工业界的解决方法在后面的笔记中可能会进行整理.

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220820214248039.png" alt="image-20220820214248039" style="zoom: 200%;" />



# 第十章 高级纹理

## 1.立方体纹理CubeMap

cubemap是环境映射的一种实现方法.cube map一共包括了6张图像,对应一个立方体的六个面,每个面表示沿着世界空间下的轴向(上下左右前后)观察所得到的图像.**在采样的时候只需要提供一个三维的纹理坐标(表示在世界空间下的一个3D方向),该方向矢量从立方体中心出发,向外延申时和立方体的6个纹理之一相交,**这样采样得到的结果就由该交点计算而来,示意图如下:

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220821110724944.png" alt="image-20220821110724944" style="zoom: 67%;" />

cubemap的好处在于实现简单快速,但也有一些缺点,比如当场景引入了新的光源和物体的时候都要重新生成cube map,而且立方体纹理也仅可以反射环境,但不能反射使用了该cube map的物体本身.这是因为cube map无法模拟多次反射的结果(比如两个金属球互相反射,这种可能要用到全局关照).**在游戏设计中,我们应尽量对凸面体使用cube map而不要用凹面体(因为凹面体会反射自身)**.

cube map的常见场合是天空盒.

### 1.1 天空盒Skybox

作用是模拟天空,使用了cube map技术.制作如下:

- 首先,新建一个材质,Shader选择Unity自带的Skybox/6 Sided,该材质需要6张纹理.
- 使用书中资源中Assets/Textures/Chapter 10/Cubemaps下的6张纹理进行赋值,并且要设置wrap mode 为Clamp,防止接缝处出现不匹配现象.
- 注意到Shader还有3个属性:
  - Tint Color:用于控制该材质的整体颜色
  - Exposure:用于调整天空盒子的亮度
  - Rotation: 用于调整天空盒子沿+y方向的旋转角度
- 将刚才的材质赋予windows-Rendering->Lighting中的天空盒,同时为了让摄像机正常显示skybox,需要保证渲染场景的相机的Camera组件中的Clear Flags被设置为skybox.

整体赋值和结果如下:

![image-20220821113020550](E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220821113020550.png)

如果我们希望某些摄像机可以用不同的天空盒,可以通过向该摄像机添加skybox组件来覆盖掉之前的设置.

**在Unity当中,skybox是在所有不透明物体之后渲染的,而其背后使用的网格是一个立方体或一个细分后的球体.**



### 1.2 创建用于环境映射的cube map

除了天空盒,cubemap还常见于环境映射,通过cubemap我们可以模拟出金属质感的材质.创建用于环境映射的cubemap有三种方法:

- (1)由特殊布局的纹理创建,比如类似立方体展开图的交叉布局,全景布局等,将该纹理的Texture Type设置为Cubemap.基于PBR的渲染中,通常会使用一张HDR图像来生成高质量Cubemap;
- (2)手动创建Cubemap资源,并赋值6张图(我们刚才做的操作)
  - 不过官方推荐第一种,因为方便对纹理压缩,且支持边缘修正,光滑反射和HDR等功能;
- (3)用程序生成.如果我们希望根据物体在场景中位置的不同,生成各自不同的cubemap,此时就可以在Unity中用脚本创建(通过Unity的`Camera.RenderToCubemap`函数来实现,该函数可以把任意位置观察到的场景图像存储到6张图像中,从而创建该位置的cubemap)

这里我们尝试第三种方法,首先新建一个脚本`RenderCubemapWizard.cs`,注意由于我们要添加菜单栏条目,因此需要建一个Assets->Editor文件夹,并将脚本放进去,脚本书写如下:

```c#
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEditor;

public class RenderCubemapWizard : ScriptableWizard
{
	public Transform renderFromPosition; //由用户指定,在这个位置处创建一个摄像机,调用Camera.RenderToCubemap函数把从当前位置观察到的图像渲染到用户指定的了立方体纹理cubemap中,然后销毁临时摄像机.
	public Cubemap cubemap;

	void OnWizardUpdate()
	{
		helpString = "Select transform to render from and cubemap to render into";
		isValid = (renderFromPosition != null) && (cubemap != null);
	}

	void OnWizardCreate()
	{
		// create temporary camera for rendering
		GameObject go = new GameObject("CubemapCamera");
		go.AddComponent<Camera>();
		// place it on the object
		go.transform.position = renderFromPosition.position;
		// render into cubemap		
		go.GetComponent<Camera>().RenderToCubemap(cubemap);

		// destroy temporary camera
		DestroyImmediate(go);
	}

	[MenuItem("GameObject/Render into Cubemap")]
	static void RenderCubemap()
	{
		ScriptableWizard.DisplayWizard<RenderCubemapWizard>(
			"Render cubemap", "Render!");
	}
}

```

相关的说明可以参考Unity官方文档:

[UnityEditor.ScriptableWizard - Unity 脚本 API](https://docs.unity.cn/cn/current/ScriptReference/ScriptableWizard.html)

有了上述脚本后,我们来创建一个Cubemap:

- (1)创建一个空GameObject对象,使用该对象的位置信息渲染cubemap;
- (2)新建一个用于存储的cubemap,具体地在Project面板下右键->Create->Legacy->Cubemap来创建,命名为Cubemap_0,为了让脚本可以顺利将图像渲染到该cubemap中,我们需要勾选Readable;
  - 可以设置Cubemap的Faze Size选项,该值越大,渲染出来的立方体纹理分辨率更大,效果可能更好但需要更多内存,这可以通过下方显示的内存观察到.
- (3)选择刚才新增的Render into Cubemap选项卡

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220821120100967.png" alt="image-20220821120100967" style="zoom:67%;" />

可以看到,选择Render之后就成功渲染进去了.准备好纹理后就可以对物体使用环境映射技术,而其中最常见的应用就是反射和折射.



### 1.3 反射

只需要通过入射光线的方向和表面法线的方向来计算反射方向,再利用反射方向对立方体纹理采样即可.在这里我们使用刚才的Cubemap,并拖入一个teapot模型(与刚才新建的空Gameobject位置重合),然后赋予材质和Shader:

```glsl
Shader "Chapter 10/Reflection"
{
   Properties{
		_Color("Color Tint",Color)=(1,1,1,1)
		_ReflectColor("Reflection Color",Color)=(1,1,1,1)
		_ReflectAmount("Reflect Amount",Range(0,1))=1
		_Cubemap ("Reflection Cubemap",Cube) = "_Skybox" {}
	}
	SubShader{
		Tags{"RenderType" = "Opaque" "Queue" = "Geometry"} //opaque most of the shaders (Normal, Self Illuminated, Reflective, terrain shaders).
		Pass{
			Tags {"LightMode" = "ForwardBase"}
			CGPROGRAM
			#pragma multi_compile_fwdbase
			#pragma vertex vert
			#pragma fragment frag

			#include "Lighting.cginc"
			#include "AutoLight.cginc"
			
			fixed4 _Color;
			fixed4 _ReflectColor;
			fixed _ReflectAmount;
			samplerCUBE _Cubemap;

			struct a2v{
				float4 vertex:POSITION;
				float3 normal:NORMAL;
			};

			struct v2f{
				float4 pos:SV_POSITION;
				float3 worldPos:TEXCOORD0;
				fixed3 worldNormal:TEXCOORD1;
				fixed3 worldViewDir:TEXCOORD2;
				fixed3 worldRefl:TEXCOORD3;
				SHADOW_COORDS(4)
			};

			v2f vert(a2v v){
				v2f o;
				o.pos = UnityObjectToClipPos(v.vertex);
				o.worldNormal = UnityObjectToWorldNormal(v.normal);
				o.worldPos = mul(_Object2World,v.vertex).xyz;
				o.worldViewDir = UnityWorldSpaceViewDir(o.worldPos);

				o.worldRefl = reflect(-o.worldViewDir,o.worldNormal);
				TRANSFER_SHADOW(o);
				return o;
			}

			fixed4 frag(v2f i):SV_Target{
				fixed3 worldNormal = normalize(i.worldNormal);
				fixed3 worldLightDir = normalize(UnityWorldSpaceLightDir(i.worldPos));
				fixed3 worldViewDir = normalize(i.worldViewDir);
				fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;

				fixed3 diffuse = _LightColor0.rgb*_Color.rgb*max(0,dot(worldNormal,worldLightDir));
				
				fixed3 reflection = texCUBE(_Cubemap,i.worldRefl).rgb*_ReflectColor.rgb; //不用归一化,因为我们只需要关心方向而不需要关心大小
				
				UNITY_LIGHT_ATTENUATION(atten,i,i.worldPos);
				
				fixed3 color = ambient + lerp(diffuse,reflection,_ReflectAmount)*atten; //diffuse+_ReflectAmount(reflection-diffuse)
				return fixed4(color,1.0);	

			}
            ENDCG
		}
	}
    FallBack "Reflective/VertexLit"
}
```

上述代码中在顶点着色器当中计算反射方向,理论上也可以在片元着色器中,由于人眼观察的差别不大,因此这里就用顶点着色器了.将之前的_Cubemap_0拖入到shader的对应属性中,调整参数观察效果:

![image-20220821163304411](E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220821163304411.png)



### 1.4 折射

首先复习斯涅尔定律:

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220821163552493.png" alt="image-20220821163552493" style="zoom:67%;" />

有:$\eta_1sin\theta_1=\eta_2sin\theta_2$,其中η是折射率.通常来说得到折射方向之后就会使用其来对cube map进行采样,不过理论上一种更准确的模拟方法需要计算两次折射(一次是当光线进入内部时,一次是从内部射出时),不过在实时渲染中很难模拟处第二次折射方向,因此往往我们只模拟第一次折射.新建一个Shader,代码如下(与反射相同的部分不再说明):

```glsl
Shader "Chapter 10/Refraction"
{
    Properties{
		_Color("Color Tint",Color)=(1,1,1,1)
		_RefractColor("Refraction Color",Color)=(1,1,1,1)
		_RefractAmount("Refract Amount",Range(0,1))=1
		_RefractRatio("Refraction Ratio",Range(0.1,1))=0.5  //不同介质的透射比
		_Cubemap ("Refraction Cubemap",Cube) = "_Skybox" {}
	}
	SubShader{
		Tags{"RenderType" = "Opaque" "Queue" = "Geometry"} //opaque most of the shaders (Normal, Self Illuminated, Reflective, terrain shaders).
		Pass{
			Tags {"LightMode" = "ForwardBase"}
			CGPROGRAM
			#pragma multi_compile_fwdbase
			#pragma vertex vert
			#pragma fragment frag

			#include "Lighting.cginc"
			#include "AutoLight.cginc"
			
			fixed4 _Color;
			fixed4 _RefractColor;
			fixed _RefractAmount;
			fixed _RefractRatio;
			samplerCUBE _Cubemap;

			struct a2v{
				float4 vertex:POSITION;
				float3 normal:NORMAL;
			};

			struct v2f{
				float4 pos:SV_POSITION;
				float3 worldPos:TEXCOORD0;
				fixed3 worldNormal:TEXCOORD1;
				fixed3 worldViewDir:TEXCOORD2;
				fixed3 worldRefr:TEXCOORD3;
				SHADOW_COORDS(4)
			};

			v2f vert(a2v v){
				//...
				o.worldRefr = refract(-normalize(o.worldViewDir),normalize(o.worldNormal),_RefractRatio);
				TRANSFER_SHADOW(o);
				return o;
			}

			fixed4 frag(v2f i):SV_Target{
				//...
				
				fixed3 refraction = texCUBE(_Cubemap,i.worldRefr).rgb*_RefractColor.rgb;
				
				UNITY_LIGHT_ATTENUATION(atten,i,i.worldPos);
				
				fixed3 color = ambient + lerp(diffuse,refraction,_RefractAmount)*atten;
				return fixed4(color,1.0);	

			}
			ENDCG		
		}
	}
		FallBack "Reflective/VertexLit"
}

```

CG的Refract函数接收三个参数:

> 第一个参数是如何光线的方向(需要归一化),第二个参数是表面法线方向(需要归一化),第三个参数是介质比(入射介质比折射介质,比如空气到玻璃就是1/1.5)

返回值是计算得到的折射方向,它的模等于入射光线的模,最终效果如下(同样材质赋予刚才的skybox):

![image-20220821165346530](E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220821165346530.png)



### 1.5 菲涅尔反射

实时渲染中,往往会使用菲涅尔反射来根据视角方向控制反射程度(原理见games101),几乎任何物体都或多或少包含了菲涅尔效果,这也是基于PBR的渲染中重要的一项高光反射计算因子.在实际应用中我们往往使用**Schlick菲涅尔近似等式:**
$$
F_{Schilick}(v,n)=F_0+(1-F_0)(1-v·n)^5
$$
其中F0是一个反射系数,用于控制菲涅尔反射的强度,v是视角方向,n是表面法线.另一个近似等式是**Empricial菲涅尔近似等式:**
$$
F_{Empricial}(v,n)=max(0,min(1,bias+scale×(1-v·n)^{power}))
$$
上式当中的bias,scale和power是对应的控制项.以上的公式可以解决一些比如车漆,水面等材质的渲染.接下来我们在Shader当中写:

```glsl
Shader "Chapter 10/Fresnel" //使用Schlick菲涅尔近似等式
{
    Properties
    {
        _Color("Color Tint",Color)=(1,1,1,1)
        _Cubemap ("Refraction Cubemap",Cube) = "_Skybox" {}
        _FresnelScale("Fresnel Scale",Range(0,1))=0.5
        
    }
    SubShader{
		Tags{"RenderType" = "Opaque" "Queue" = "Geometry"} //opaque most of the shaders (Normal, Self Illuminated, Reflective, terrain shaders).
		Pass{
			Tags {"LightMode" = "ForwardBase"}
			CGPROGRAM
			#pragma multi_compile_fwdbase
			#pragma vertex vert
			#pragma fragment frag

			#include "Lighting.cginc"
			#include "AutoLight.cginc"
			
			fixed4 _Color;
			fixed _FresnelScale;
			samplerCUBE _Cubemap;

			struct a2v{
				float4 vertex:POSITION;
				float3 normal:NORMAL;
			};

			struct v2f{
				float4 pos:SV_POSITION;
				float3 worldPos:TEXCOORD0;
				fixed3 worldNormal:TEXCOORD1;
				fixed3 worldViewDir:TEXCOORD2;
				fixed3 worldRefl:TEXCOORD3;
				SHADOW_COORDS(4)
			};

			v2f vert(a2v v){
				v2f o;
				o.pos = UnityObjectToClipPos(v.vertex);
				o.worldNormal = UnityObjectToWorldNormal(v.normal);
				o.worldPos = mul(unity_ObjectToWorld,v.vertex).xyz;
				o.worldViewDir = UnityWorldSpaceViewDir(o.worldPos);

				o.worldRefl = reflect(-o.worldViewDir,o.worldNormal);
				TRANSFER_SHADOW(o);
				return o;
			}

			fixed4 frag(v2f i):SV_Target{
				fixed3 worldNormal = normalize(i.worldNormal);
				fixed3 worldLightDir = normalize(UnityWorldSpaceLightDir(i.worldPos));
				fixed3 worldViewDir = normalize(i.worldViewDir);
				UNITY_LIGHT_ATTENUATION(atten,i,i.worldPos);
				fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;

				fixed3 diffuse = _LightColor0.rgb*_Color.rgb*max(0,dot(worldNormal,worldLightDir));
				fixed3 reflection = texCUBE(_Cubemap,i.worldRefl).rgb;
				fixed fresnel = _FresnelScale + (1-_FresnelScale) * pow(1-dot(worldViewDir,worldNormal),5);
				
				fixed3 color = ambient + lerp(diffuse,reflection,saturate(fresnel))*atten;
				return fixed4(color,1.0);	

			}
			ENDCG		
		}
	}
		FallBack "Reflective/VertexLit"
        
}

```

注意,以上我们混合的是漫反射光照和反射光照,一些实现也会直接把fresnel和反射光照相乘后叠加到漫反射光照上,模拟边缘光照的效果.**在这里我们没有混合反射和折射光照,这个将在第15章的模拟水面效果部分来说明.**

最终效果如下:

![image-20220821172435543](E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220821172435543.png)



## 2.渲染纹理Render Texture

现代的GPU允许我们把整个三维场景渲染到一张中间缓冲当中(也就是**渲染目标纹理(Render Target Texture,RTT)**),而不是传统的帧缓冲或后备缓冲.与之相关的是**多重渲染目标(MRT)**,这种技术指的是GPU允许我们把场景同时渲染到多个渲染目标纹理中.延迟渲染就是使用多重渲染目标的一个应用.

Unity定义了一种专门的纹理类型(Render Texture,渲染纹理),在Unity当中通常有两种使用方式:

- (1)一种方式是在Project目录下创建一个渲染纹理，然后把某个摄像机的渲染目标设置成该渲染纹理，这样一来该摄像机的渲染结果就会实时更新到渲染纹理中，而不会显示在屏幕上。使用这种方法，我们还可以选择渲染纹理的分辨率、滤波模式等纹理属性。
- (2)另一种方式是在屏幕后处理时使用GrabPass命令或OnRenderImage函数来获取当前屏幕图像，Unity会把这个屏幕图像放到一张和屏幕分辨率等同的渲染纹理中，下面我们可以在自定义的Pass中把它们当成普通的纹理来处理，从而实现各种屏幕特效。我们将依次学习这两种方法在Unity中的实现（OnRenderImage函数会在第12章中讲到）。



### 2.1 镜子效果

新建一个场景,去掉天空盒,搭建场景如下:

- (1)用6个Cube搭建六面体,作为房间,将相机放入房间内,加入一定数量的点光源以照亮场景,为材质赋予基础光照Shader(在Shader->Common文件夹下);
- (2)创建3个球体和两个立方体,调整位置和大小,并赋予标准材质,作为房间饰品;
- (3)创建一个Quad作为镜子,我们接下来就要写赋予这个Quad的材质的Shader.
- (4)Project视图下创建一个渲染纹理,取名MirrorTexture,纹理设置采用默认即可.
- (5)新建一个摄像机.调整对应参数,使其显示图像是我们所希望的镜子图像(所以这个camera要和main camera放在mirror的两侧,方便起见可以把这个camera作为mirror的子节点来挂载).由于这个摄像机不需要直接显示在屏幕上,而是用于渲染到纹理,因此我们将MirrorTexture赋予相机的Target Texture.

![image-20220821181323156](E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220821181323156.png)

书写Shader如下:

```glsl
Shader "Chapter 10/Mirror"
{
    Properties
    {
        _MainTex ("Main Tex", 2D) = "white" {}
    }
    SubShader
    {
        Tags { "RenderType"="Opaque" }
        LOD 100

        Pass
        {
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag

            #include "UnityCG.cginc"

            struct a2v
            {
                float4 vertex : POSITION;
                float2 texcoord : TEXCOORD0;
            };

            struct v2f
            {
                float2 uv : TEXCOORD0;
                float4 pos : SV_POSITION;
            };

            sampler2D _MainTex;
            float4 _MainTex_ST;

            v2f vert (a2v v)
            {
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);
                //o.uv = TRANSFORM_TEX(v.texcoord, _MainTex);
                o.uv = v.texcoord;
                o.uv.x = 1-o.uv.x; //镜子里显示的图像左右相反
                return o;
            }

            fixed4 frag (v2f i) : SV_Target
            {
                return tex2D(_MainTex,i.uv);
            }
            ENDCG
        }
    }
}

```

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220821203022241.png" alt="image-20220821203022241" style="zoom:67%;" />

上图仅作为示例使用,可能画面有一些穿帮不过不做精细调整了.如果纹理没有及时刷新上来可以重新拖一下render texture.



### 2.2 玻璃效果

可以在Shader当中定义一种特殊的GrabPass来完成获取屏幕图像的目的.当定义了一个Grab Pass之后,Unity会把当前屏幕的图像绘制在一张纹理中,以便于我们在后续的Pass中访问.**Grab Pass可以用来实现诸如玻璃等透明材质的模拟**,可以让我们对该物体后面的图像进行更复杂的处理,比如用法线来模拟折射效果.

使用Grab Pass的时候必须要**小心物体的渲染队列设置**,由于其常用来渲染透明物体,因此往往要设置(`"Queue"="Transparent"`),这样才能保证渲染该物体时所有不透明物体已经被绘制在屏幕上了,从而获取正确的屏幕图像.玻璃效果的实现思路如下:

> (1)首先用一张法线纹理来修改模型的法线信息
>
> (2)用之前10.1介绍的反射方法,通过Cubemap模拟玻璃的反射
>
> (3)在模拟折射时,使用GrabPass获取玻璃后的屏幕图像,并使用切线空间下的法线对屏幕纹理坐标偏移,然后再对屏幕图像采样来模拟折射效果.

具体如下:

(1)新建一个场景并去掉天空盒.构建一个由6面墙围成的封闭房间,在房间中放一个正方体,内部放一个球体;

(2)新建一个材质,赋予房间中的立方体;

(3)通过前面的Render into Cubemap创建对应的Cube map,命名为Glass_Cubemap.

(4)新建一个Shader,书写如下:

```glsl
Shader "Chapter 10/GlassRefraction"
{
    Properties
    {
        _MainTex ("Texture", 2D) = "white" {}
        _BumpMap ("Normal Map",2D)="bump" {}
        _Cubemap ("Environment CubeMap",Cube) = "_SkyBox" {} //注意命名与下面保持一致
        _Distortion("Distortion",Range(-1000,1000))=10
        _RefractAmount("Refract Amount",Range(0.0,1.0))=1.0 //值为0时只包含反射效果,为1时只包括折射效果
    }
    SubShader
    {
        Tags { "Queue"="Transparent" "RenderType"="Opaque" } //见注解1
        GrabPass {"_RefractionTex"} //见注解2
        

        Pass
        {
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag

            #include "UnityCG.cginc"

            struct a2v {
				float4 vertex : POSITION;
				float3 normal : NORMAL;
				float4 tangent : TANGENT; 
				float2 texcoord: TEXCOORD0;
			};
			
			struct v2f {
				float4 pos : SV_POSITION;
				float4 scrPos : TEXCOORD0;
				float4 uv : TEXCOORD1;
				float4 TtoW0 : TEXCOORD2;  
			    float4 TtoW1 : TEXCOORD3;  
			    float4 TtoW2 : TEXCOORD4; 
			};

            sampler2D _MainTex;
            float4 _MainTex_ST;
            sampler2D _BumpMap;
            float4 _BumpMap_ST;
            samplerCUBE _Cubemap;
            float _Distortion;
            fixed _RefractAmount;
            sampler2D _RefractionTex;
            float4 _RefractionTex_TexelSize; //见注解3

            v2f vert (a2v v)
            {
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);
                o.scrPos = ComputeGrabScreenPos(o.pos); //见注解4
                o.uv.xy = TRANSFORM_TEX(v.texcoord, _MainTex);
                o.uv.zw = TRANSFORM_TEX(v.texcoord, _BumpMap);
                
                float3 worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;  
                fixed3 worldNormal = UnityObjectToWorldNormal(v.normal);  
                fixed3 worldTangent = UnityObjectToWorldDir(v.tangent.xyz);  
                fixed3 worldBinormal = cross(worldNormal, worldTangent) * v.tangent.w; 
                
                //由于要在片元着色器中把法线方向从切线空间变换到世界空间下,从而方便cubemap采样,与法线贴图步骤类似
                o.TtoW0 = float4(worldTangent.x, worldBinormal.x, worldNormal.x, worldPos.x);  
                o.TtoW1 = float4(worldTangent.y, worldBinormal.y, worldNormal.y, worldPos.y);  
                o.TtoW2 = float4(worldTangent.z, worldBinormal.z, worldNormal.z, worldPos.z);  
                return o;
            }

            fixed4 frag (v2f i) : SV_Target
            {
                float3 worldPos = float3(i.TtoW0.w, i.TtoW1.w, i.TtoW2.w);
                fixed3 worldViewDir = normalize(UnityWorldSpaceViewDir(worldPos));

                fixed3 bump = UnpackNormal(tex2D(_BumpMap, i.uv.zw));   

                // Compute the offset in tangent space
                float2 offset = bump.xy * _Distortion * _RefractionTex_TexelSize.xy;  //见注解5
                i.scrPos.xy = offset * i.scrPos.z + i.scrPos.xy; //书中的代码这里没有*scrPos.z
                fixed3 refrCol = tex2D(_RefractionTex, i.scrPos.xy/i.scrPos.w).rgb;  //见注解6

                // Convert the normal to world space
                bump = normalize(half3(dot(i.TtoW0.xyz, bump), dot(i.TtoW1.xyz, bump), dot(i.TtoW2.xyz, bump)));
                fixed3 reflDir = reflect(-worldViewDir, bump);
                fixed4 texColor = tex2D(_MainTex, i.uv.xy);
                fixed3 reflCol = texCUBE(_Cubemap, reflDir).rgb * texColor.rgb;

                fixed3 finalColor = reflCol * (1 - _RefractAmount) + refrCol * _RefractAmount;

                return fixed4(finalColor, 1);
            }
            ENDCG
        }
    }
}
```

相关注解如下(**有的地方有些没理解,后面再做补充**):

- 注解1:`Tags { "Queue"="Transparent" "RenderType"="Opaque" }`
  - 看似矛盾,但是却服务于不同的需求.Queue设置为Transparent可以确保该物体渲染时,所有的不透明物体都已经被渲染到屏幕上了.RenderType则是为了在使用着色器替换时,该物体可以在需要时被正确渲染(通常用于要得到摄像机的深度和法线纹理,13章会提到).
- 注解2:`GrabPass {"_RefractionTex"}`
  - 通过GrabPass定义了一个抓取屏幕图像的Pass,这个Pass中定义了一个字符串,该名称会决定抓取得到的屏幕图像会被存入哪个纹理当中.理论上可以省略声明该字符串,但声明会提高性能,后面会进行解释.
- 注解3:`float4 _RefractionTex_TexelSize;`
  - 这句话前面我们定义了`_RefractionTex`,也就是GrabPass时指定的纹理名称,而`_RefractionTex_TexelSize`则可以让我们得到该纹理的纹素大小.比如一个256×512的纹理,纹素大小就是(1/256,1/512),对屏幕图像采样的坐标偏移时需要用该变量.
- 注解4:关于顶点着色器,`ComputeGrabScreenPos`函数可以得到对应被抓取的屏幕图像的采样坐标.由于片元着色器中我们需要把法线方向从切线空间(法线纹理采样得到)转换到世界空间(以方便对Cubemap采样),因此如之前一样,需要存储一个从切线空间变换到世界空间的变换矩阵,这就是`o.TtoWx`系列.
- 注解5:在片元着色器中,我们对法线纹理进行采样,得到切线空间下的法线方向,并使用该值和_Distortion属性以及`_RefractionTex_TexelSize`来对屏幕图像的采样坐标进行偏移,模拟折射效果.在这里选择用切线空间下的法线方向来进行偏移,因为该空间下的法线可以反应顶点局部空间下的法线方向.
- 注解6:`fixed3 refrCol = tex2D(_RefractionTex, i.scrPos.xy/i.scrPos.w).rgb;`
  - **这里可以参考书上4.9.3节的关于屏幕坐标的解释.**这里对scrPos透视除法得到真正的屏幕坐标（原理可参见4.9.3节），再使用该坐标对抓取的屏幕图像_RefractionTex进行采样，得到模拟的折射颜色。至于反射项就是前面学习的Cubemap采样了.

最终的场景如下:

![image-20220827165621292](E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220827165621292.png)

注意,如果场景物体过大的时候,可能会出现拖动Distortion数值效果变动不明显的问题,后面深刻理解原理后再来补充,



### 2.3 两个问题

(1)关于GrabPass的使用

- 1.直接使用GrabPass { },在后续的Pass中直接使用_GrabTexture来访问屏幕图像,不过如果场景中多个物体都这样抓取屏幕的话,这种方法性能消耗较大,好处是每个物体都可以得到不同的屏幕图像,取决于渲染队列和当时的缓冲区颜色;
- 2.类似本节的实现:`GrabPass { "TextureName" }`,此时可以在后续的Pass中用TextureName获取到屏幕图像,此时Unity只会在每一帧时为第一个使用名为TextureName的纹理执行抓取屏幕的操作,获得的纹理同样可以在其他Pass中访问.此时Unity的一次抓取工作就可以让多个物体来使用,比较高效.

(2)关于渲染纹理 vs GrabPass

- GrabPass的实现方式更为简单,但渲染纹理的效率往往要高于GrabPass,因为渲染纹理可以自定义纹理大小,而GrabPass获取到的图像分辨率和显示屏幕一致,并且虽然GrabPass不会重新渲染场景,但往往需要CPU直接读取back buffer中的数据,从而破坏了CPU和GPU之间的并行性,相对耗时一些.
- Unity引入了命令缓冲功能,这个后面会进行介绍.



## 3.程序纹理

### 3.1 使用算法生成波点纹理

新建场景和对应的材质和Shader,首先我们希望能用脚本来创建程序纹理.对应的cs文件如下:
```c#
using System.Collections;
using System.Collections.Generic;
using Antlr.Runtime.Tree;
using UnityEditor;
using UnityEngine;

[ExecuteInEditMode]
public class ProceduralTextureGeneration : MonoBehaviour
{
    public Material material = null;

    #region Material properties

    [SerializeField, SetProperty("textureWidth")]
    private int m_textureWidth = 512;

    public int textureWidth
    {
        get
        {
            return m_textureWidth;
        }
        set
        {
            m_textureWidth = value;
            _UpdateMaterial();
        }
    }

    [SerializeField, SetProperty("backgroundColor")]
    private Color m_backgroundColor = Color.white;

    public Color backgroundColor
    {
        get
        {
            return m_backgroundColor;
        }
        set
        {
            m_backgroundColor = value;
            _UpdateMaterial();
        }
    }

    [SerializeField, SetProperty("CircleColor")]
    private Color m_circleColor = Color.yellow;

    public Color CircleColor
    {
        get
        {
            return m_circleColor;
        }
        set
        {
            m_circleColor = value;
            _UpdateMaterial();
        }
    }

    [SerializeField,SetProperty("blurFactor")]
    private float m_blurFactor = 2.0f;
    public float blurFactor
    {
        get
        {
            return m_blurFactor;
        }
        set
        {
            m_blurFactor = value;
            _UpdateMaterial();
        }
    }

    #endregion

    private Texture2D m_generatedTexture = null;

    void Start()
    {
        if (material == null)
        {
            Renderer renderer = gameObject.GetComponent<Renderer>();
            if (renderer == null)
            {
                Debug.LogWarning("Cannot find a renderer");
                return;
            }

            material = renderer.sharedMaterial;
        }
        _UpdateMaterial();
    }
    private void _UpdateMaterial()
    {
        if (material != null)
        {
            m_generatedTexture = _GenerateProceduralTexture();
            material.SetTexture("_MainTex", m_generatedTexture);
        }
    }

    private Color _MixColor(Color color0, Color color1, float mixFactor)
    {
        Color mixColor = Color.white;
        mixColor.r = Mathf.Lerp(color0.r, color1.r, mixFactor);
        mixColor.g = Mathf.Lerp(color0.g, color1.g, mixFactor);
        mixColor.b = Mathf.Lerp(color0.b, color1.b, mixFactor);
        mixColor.a = Mathf.Lerp(color0.a, color1.a, mixFactor);
        return mixColor;
    }

    private Texture2D _GenerateProceduralTexture()
    {
        Texture2D proceduralTexture = new Texture2D(textureWidth, textureWidth);
        //定义圆与圆之间的间距
        float circleInterval = textureWidth / 4.0f;
        float radius = textureWidth / 10.0f;
        //定义模糊系数
        float edgeBlur = 1.0f / blurFactor;

        for (int w = 0; w < textureWidth; w++)
        {
            for (int h = 0; h < textureWidth; h++)
            {
                //使用背景颜色进行初始化
                Color pixel = backgroundColor;
                for (int i = 0; i < 3; i++)
                {
                    for (int j = 0; j < 3; j++)
                    {
                        //计算当前所绘制的圆的圆心位置
                        Vector2 circleCenter = new Vector2(circleInterval * (i + 1), circleInterval * (j + 1));
                        //计算当前像素与圆心的距离
                        float dist = Vector2.Distance(new Vector2(w, h), circleCenter) - radius;
                        //模糊圆的边界
                        Color color = _MixColor(CircleColor, new Color(pixel.r, pixel.g, pixel.b, 0.0f),
                            Mathf.SmoothStep(0f, 1.0f, dist * edgeBlur));

                        //与之前得到的颜色进行混合
                        pixel = _MixColor(pixel, color, color.a);
                    }
                }
                proceduralTexture.SetPixel(w, h, pixel);
            }
        }

        proceduralTexture.Apply();
        return proceduralTexture;
    }
}

```

如果代码当中的SetProperty报错，可以按下面的文章来解决(或者在学习过程中,直接设置为public也可以)：

[【Unity插件】SetProperty -在Inspector面板上访问属性(get/set) - xudawang's blog](http://blog.xudawang.fun/2018/05/18/setproperty-cn/)

说明如下:

- **#region** 和**#endregion** 仅仅是为了组织代码，并没有其他作用。
- _GenerateProceduralTexture(),这个函数代码首先初始化一张二维纹理，并且提前计算了一些生成纹理时需要的变量。然后，使用了一个两层的嵌套循环遍历纹理中的每个像素，并在纹理上依次绘制9个圆形。最后，调用Texture2D.Apply函数来强制把像素值写入纹理中，并返回该程序纹理。

新建一个材质,赋予其第七章的Simple Texture纹理,然后给正方体绑定脚本和材质,如下:

![image-20220827204420710](E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220827204420710.png)



### 3.2 Unity的程序材质

可以用第三方软件导入对应的程序材质,具体地后续再进行展开.



# 第十一章 让画面动起来

 动画效果往往要把时间添加到一些变量地计算中,使得画面可以随时间变化,Unity Shader内置的时间变量如下:

|      名 称      | 类 型  |                            描 述                             |
| :-------------: | :----: | :----------------------------------------------------------: |
|      _Time      | float4 | t是自该场景加载开始所经过的时间，4个分量的值分别是(t/20, t, 2t, 3t)。 |
|    _SinTime     | float4 |     t是时间的正弦值，4个分量的值分别是(t/8, t/4, t/2, t)     |
|    _CosTime     | float4 |     t是时间的余弦值，4个分量的值分别是(t/8, t/4, t/2, t)     |
| unity_DeltaTime | float4 | dt是时间增量，4个分量的值分别是(dt, 1/dt, smoothDt, 1/smoothDt) |

## 11.1 纹理动画

### 1.序列帧动画

最常见的纹理动画之一就是序列帧动画,在某些受局限的移动平台上,往往会使用纹理动画来替代复杂的粒子系统等模拟动画效果.优点在于不用做大量计算,缺点在于美术人员的工作量较大.首先我们需要提供一张序列帧纹理,从上到下,从左到右地播放动画:

<img src="E:/prepareForWorkFinal/%E6%8A%80%E6%9C%AF%E7%BE%8E%E6%9C%AF/%E7%A7%8B%E6%8B%9B/%E7%A7%8B%E6%8B%9B-Unity%E5%85%A5%E9%97%A8%E7%B2%BE%E8%A6%81.assets/image-20220828123158273.png" alt="image-20220828123158273" style="zoom:67%;" />

具体操作如下:

(1)首先,新建一个场景,去掉天空盒,新建一个Quad并使其正面朝向摄像机,赋予材质和shader;

(2)在Shader中书写以下内容:

```glsl
Shader "Chapter 11/ImageSequenceAnimation"
{
    Properties
    {
        _Color("Color Tint",Color) = (1,1,1,1)
        _MainTex ("Image sequence", 2D) = "white" {}
        _HorizontalAmount("Horizontal Amount",Float) = 4
        _VerticalAmount("Vertical Amount",Float) = 4
        _Speed("Speed",Range(1,100)) = 30
    }
    SubShader
    {
        //序列帧图像通常是透明纹理,需要设置Pass相关状态,以渲染透明状态
        Tags{"Queue"="Transparent""IgnoreProjector"="True""RenderType"="Transparent"}  //见注解1
        Pass
        {
            Tags{"LightMode"="ForwardBase"}
            ZWrite Off
            Blend SrcAlpha OneMinusSrcAlpha
            
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #include "UnityCG.cginc"
            
            fixed4 _Color;
			sampler2D _MainTex;
			float4 _MainTex_ST;
			float _HorizontalAmount;
			float _VerticalAmount;
			float _Speed;
            
            struct a2v
            {
                float4 vertex:POSITION;
                float2 texcoord:TEXCOORD0;
            };

            struct v2f
            {
                float4 pos:SV_POSITION;
                float2 uv:TEXCOORD0;
            };   
            
            v2f vert(a2v v)
            {
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);    
                o.uv = TRANSFORM_TEX(v.texcoord,_MainTex);
                return o;
            }

            fixed4 frag(v2f i):SV_Target
            {
                float time = floor(_Time.y*_Speed); //_Time.y是自该场景加载后所经过的时间,与Speed相乘得到模拟的时间
                float row = floor(time/_HorizontalAmount);
                float column = time - row*_VerticalAmount; //前三行中计算了行列数
                //  half2 uv = float2(i.uv.x /_HorizontalAmount, i.uv.y / _VerticalAmount);
                //  uv.x += column / _HorizontalAmount;
                //  uv.y -= row / _VerticalAmount;
                half2 uv = i.uv + half2(column,-row); //见注解2
                uv.x /= _HorizontalAmount;
                uv.y /= _HorizontalAmount;

                fixed4 c = tex2D(_MainTex,uv);
                c.rgb*=_Color;
                return c;
                
            }
            
            ENDCG
        }
    }
}

```

其中,该shader的基本思路是计算对应时刻下应该播放的关键帧的位置,并对该关键帧进行纹理采样.

注解:

1. 由于序列帧图像通常是透明纹理,因此需要设置Pass的相关状态,需要设置Queue和RenderType为Transparent,将IgnoreProjector设定为True,并且在Pass当中设置混合模式,关闭深度写入.
2. 用我们设置的horizontalAmount和VerticalAmount,以及_Time的计算结果,可以找到对应采样纹理图的第几行第几列,此时的问题就在于当我们给出某行某列,要正确地采样到对应的图案上,此时一个做法是首先把原纹理坐标i.uv按行数和列数进行等分,得到每个子图像的纹理坐标范围,接下来就可以用计算出的几行几列的值来对纹理进行相应的偏移,得到当前子图像的纹理坐标.**注意,对竖直方向的坐标偏移需要用减法,这是因为在Unity中纹理坐标竖直方向的顺序(从下到上递增)和序列帧纹理中的顺序(从上到下递增)是相反的.**

(3)将Boom.png序列帧纹理的Alpha Is Transparency属性勾选上,因为该纹理是透明纹理.

**Q:如何实现只播放一次?能否有shader来实现.**:



### 2.滚动的背景

很多2D游戏会使用不断滚动的背景来模拟游戏角色在场景中穿梭,而这些背景往往包含了许多layers来模拟视差效果,背景的实现往往就是使用了纹理动画.本节将会实现一个包含两层的无限滚动的2D游戏背景.为此做出如下工作:

(1)新建场景,关闭skybox,将相机改为正交投影;

(2)创建一个Quad,调整位置和大小,使其充满摄像机视野范围,将目标材质赋给Quad,其会充当游戏背景显示板.

(3)新建Shader,将其赋给(2)的材质,Shader中书写以下内容:

```glsl
Shader "Chapter 11/ScollingBg"
{
    Properties
    {
        _MainTex ("Base Layer (RGB)", 2D) = "white" {} //较远背景纹理
        _DetailTex("2nd Layer(RGB)",2D) = "white"{}  //较近背景纹理
        _ScrollX ("Base layer scroll speed",Float) = 1.0
        _Scroll2X ("2nd layer scroll speed",Float) = 1.0
        _Multiplier("Layer Multiplier",Float)=1  //控制纹理的整体亮度
    }
    SubShader
    {
        Tags { "RenderType"="Opaque""Queue"="Geometry" }

        Pass
        {
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #include "UnityCG.cginc"

             sampler2D _MainTex;
			sampler2D _DetailTex;
			float4 _MainTex_ST;
			float4 _DetailTex_ST;
			float _ScrollX;
			float _Scroll2X;
			float _Multiplier;

            struct a2v
            {
                float4 vertex : POSITION;
                float2 texcoord : TEXCOORD0;
            };

            struct v2f
            {
                float4 uv : TEXCOORD0;
                float4 pos : SV_POSITION;
            };

            v2f vert (a2v v)
            {
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);
                o.uv.xy = TRANSFORM_TEX(v.texcoord,_MainTex) + frac(float2(_ScrollX, 0.0) * _Time.y); //frac取小数部分
                o.uv.zw = TRANSFORM_TEX(v.texcoord,_DetailTex) + frac(float2(_Scroll2X, 0.0) * _Time.y);
                return o;
            }

            fixed4 frag (v2f i) : SV_Target
            {
                fixed4 firstLayer = tex2D(_MainTex,i.uv.xy);
                fixed4 secondLayer = tex2D(_DetailTex,i.uv.zw);

                fixed4 c = lerp(firstLayer,secondLayer,secondLayer.a); //使用第二层纹理的透明通道来混合两张纹理
                c.rgb *= _Multiplier;   
                return c;
            }}
            ENDCG
        }
        FallBack "Reflective/VertexLit"
}
```



## 11.2 顶点动画

在游戏中,我们常常用顶点动画来模拟例如飘动的旗帜,水流动等.本节会学习两种常见顶点动画的应用:

(1)流动的河流

(2)广告牌技术

### 1.流动的河流

河流的模拟是顶点动画最常见的应用之一,通常会用正弦函数等来模拟水流的波动效果,具体操作如下:

(1)新建场景,去掉天空盒,将相机修改为正交投影;

(2)创建材质和shader,由于我们要模拟多层水流的效果,因此最后还需要两个WaterMat材质;

(3)在Water.shader文件中书写如下:

```glsl
```

注解:

1. 关闭批处理是因为批处理会合并所有相关的模型,这样这些模型各自的模型空间就会丢失;本例需要在物体的模型空间下对顶点位置进行偏移,所以就需要关闭批处理操作.

