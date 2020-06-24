这里记载一些图形学的技巧

---

# 目录

* [玄学](#玄学)
    * [玄学随机函数](#玄学随机函数)
* [噪声](#噪声) 
    * [fbm噪声](#fbm噪声)
    * [无缝与平铺噪声](#无缝与平铺噪声)
    * [Perlin噪声](#perlin噪声)
    * [Simplex噪声](#simplex噪声)
    * [Worley噪声](#worley噪声)
* [瑕疵](#瑕疵)
    * [误差纹](#误差纹)
* [场](#场)
    * [带符号距离场](#带符号距离场)
* [杂项](#杂项)
---

# 玄学

* [玄学随机函数](#玄学随机函数)
* [目录](#目录)

## 玄学随机函数
    
```hlsl
float rand(float c)
{
    return frac(sin(dot(float2(c, 11.1 * c), float2(12.9898, 78.233))) * 43758.5453);
}

float rand(float2 co)
{
    return frac(sin(dot(co.xy, float2(12.9898, 78.233))) * 43758.5453);
}

float rand(float3 co)
{
    return frac(sin(dot(co.xyz, float3(12.9898, 78.233, 42.1897))) * 43758.5453);
}

float rand2(float n)
{
    return frac(cos(n * 89.42) * 343.42);
}

float2 rand2(float2 n)
{
    return float2(rand2(n.x * 23.62 - 300.0 + n.y * 34.35), rand2(n.x * 45.13 + 256.0 + n.y * 38.89));
}
```

[额外参考](https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/028_Voronoi_Noise/Random.cginc)

* [返回上一级](#玄学)

---

# 噪声

* [fbm噪声](#fbm噪声)
* [无缝与平铺噪声贴图](#无缝与平铺噪声贴图)
* [Perlin噪声](#perlin噪声)
* [Simplex噪声](#simplex噪声)
* [Worley噪声](#worley噪声)
* [目录](#目录)

## fbm噪声

随机漫步噪声，即使通过同一个噪声按不同周期和不同幅度进行叠加出来的噪声，其实现如下：

```hlsl
float fbm_XXX_noise(in float2 n, in uint octaves, in float frequency, in float lacunarity, in float gain)
{
    float amplitude = gain;
    float total = 0.0;
    for (uint i = 0; i < octaves; ++i)
    {
        total += XXX_noise(n * frequency) * amplitude;
        frequency *= lacunarity;
        amplitude *= gain;
    }
    return total;
}
```

* [返回上一级](#噪声)

## 无缝与平铺噪声贴图

平铺(Tiling)噪声贴图与无缝(Seamless)噪声贴图。

> 若某两个贴图中，其中一个贴图的某个维度的终点与另一张贴图该维的起点的值和导数相同，那么称这两张贴图在该维上是无缝的。

> 若某两个贴图完全一致，其在所有轴上无缝，那么该贴图是可平铺的。

若现有噪声函数`n2(x,y)`，`n3(x,y,z)`，`n4(x,y,z,w)`均连续。设贴图中各维度定义域为`[N,N + 1]`，并设有变换`z = f(x)`，则两张二维贴图`n3(x, y, z), z = f(x)，x in [N， N+1]`在定义域连续时，在Y轴上是无缝的。若有连续变换使`f(X+1) = f(x)`，那么对于贴图`n4(x,y,z,w), z = f(x), w = f(y), (x,y) in [N,N+1]`则是平铺的。

即若有生成一张N维下的平铺噪声贴图，需要一个2*N维的噪声函数和一个闭合曲线变换函数。

* [返回上一级](#噪声)

## Perlin噪声

Perlin噪声是一种渐变式的噪声。其实现原理为将空间按坐标轴划分晶格。晶格的顶点为随机值，并且如两个相邻的晶格共用一个顶点，那么该顶点的值相等。晶格内部的点则将通过与坐标轴的距离，通过衰减函数得出加权值，然后再进行线性插值。一般的，比较常用的衰减函数为`(t * t * t * (t * (t * 6 - 15) + 10))`。

该噪声对于其随机值的来源又分成梯度噪声和值噪声，其区别仅在于随机值的获取方式不同。值噪声的随机值直接由白噪声获取，梯度噪声的随机值需要从白噪声构造一个单位向量，然后通过单位向量与当前像素与顶点的偏移单位向量的点乘得到其随机值。

若要求Perlin的平铺贴图，除了直接算2*N维噪声函数外，还可以通过将`[t, 1 + p]`的晶格映射到`[1 - t, p]`的晶格处来进行计算。

* [返回上一级](#噪声)

## Simplex噪声

SimpleX噪声是一种优化的Perlin噪声。其基本原理与Perlin噪声一样，也是将空间划分成晶格，计算顶点随机值，然后进行插值。

不同的是Perlin噪声将空间按坐标轴划分，而Simplex则是通过按单形来进行划分。若在N维下，Perlin噪声需要进行`2^N-1`次的插值，而Simplex则只需要进行`N+1`次插值。Simplex在高纬度的噪声中性能更好。

对于其选取的单形需要具有以下性质：

> 单形需要能填充整个空间
> 单形内部的点到达各顶点的距离的标准差要最小。

在进行插值的时候，需要使用与距离相关的衰减函数进行线性插值，并且要保证当点位于某个面时，与不在这个面上的顶点的距离的衰减系数必须为0。其常用的衰减函数为 `(1-t^2)^3`。

通常的，为了能更快得找到像素所在晶格的顶点，通常会经过一个投射变换，让单形的大部分边与坐标轴平行，并且其变换要求如下：
    
> 经过变换后的单形必须保持相似并且相等。
> 经过变换后的单形必须能填充整个空间。 

所以，通常在二维下，会选用正三角形，并且会变换成等腰直角三角形来计算顶点。在三维下，因为正四面体无法填充整个空间，通常会用一个4条棱相等的四面体来计算区块，该四面体经过变换成其中4条棱垂直且相等的四面体后，可以组成一个立方体。

* [返回上一级](#噪声)

## Worley噪声

Worley噪声首先产生一系列独立的随机点，然后对于特定像素，计算其与所有随机点的距离，然后选取其第N短的值。
    
一般情况下，其所使用的随机点需要从外部单独生成，然后每个像素均要遍历所有的随机点，选取所需要的值。若没有要求其随机点的分布类型的话也可以通过玄学随机函数在Shader中实时生成。

在二维下，其代码如下：

```hlsl
float worley_noise(float2 n)
{
    float dis = 2.0;
    for (uint count = 0; count < 9; ++count)
    {
        float2 dir = float2((count / 3) % 3, count % 3) - 1.0;
        float2 p = floor(n) + dir;
        //rand 是一个输入二维向量并输出随机二维向量的随机函数。
        float2 rate = rand(p);
        float2 pre_rate = rate + dir - frac(n);
        float d = length(pre_rate);
        dis = min(dis, d);
    }
    return 1.0 - dis;
}
```

其基本思路是，将所有的区域分格，然后假设每个格子内均有一个随机点，通过格子左上的坐标生成一个K维向量，该K维向量即表示随机点与左上点的相对偏移。对于纹理中任意一点，需要计算其周围`3^K-1`个格子的随机点之间的距离，然后取其中第N短的值，作为其输出。

若是使用2N维噪声函数生成的N维可平铺噪声贴图，则可能使结果的贴图丢失形状。例如在二维下求`1th`噪声贴图，若是不可平铺的，则是一个又一个的圆形光斑，若是可平铺的，则光斑可能变形成扁圆或者全白。

* [返回上一级](#噪声)

---

# 瑕疵
* [误差纹](#误差纹)
* [目录](#目录)

## 误差纹

在进行步进光线追踪的时候，由于计算精度有限，会在渲染出来的效果中出现一圈圈的光纹。其解决方案即使将每次步进光线的前进值加一个微小的随机偏移。或者减少每一次的误差累积。

* [返回上一级](#瑕疵)

---

# 场
* [带符号距离场](#带符号距离场)
* [目录](#目录)

## 带符号距离场

    EDT - Euclidean Distance   
    MDT - Manhattan Distance
    CDT - Chessboard Distance

带符号距离场一般用来进行一些与边框有关的渲染，例如说字体或者一些简单的单色图案，它能够用很小的分辨率，很小的储存通道，运用GPU采样器的自动插值功能来描述物体的边框，其在放大一定倍数之内表现都良好。

其基本原理，既计算边界与当前像素的最短距离，若在形状内则为正

* [返回上一级](#场)

---

# 抗锯齿(Anti-Aliasing)

* SSAA(Super Sampling AA)
    * 渲染到2x2倍大的RT，然后再采样到目标贴图上
        * 简单粗暴，最完美，但是性能消耗大。
* MSAA(Multi-Sampling AA)
    * 只在光栅化阶段运行，有硬件支持。在像素内部，会平均分成数个覆盖样本(Coverage sample)，然后计算三角形覆盖的像素样本的的个数，得到三角形的属性对该像素的贡献权重，生成shader，然后重新计算像素的值。例如说4xMSAA就是将一个像素分成四个样本。
        * SSAA的提升版，因为多了几步的操作，所以有效率有所影响，但是不需要2x2的RT，减缓宽带压力。但是与延迟渲染不怎么兼容（因为延迟渲染的操作是在最后面来着）。
* FXAA(Fast Approximate anti-aliasing)
    * 后处理，查找深度，若边界有突变，或者颜色值有突变，那么也就是说这是边界上的点，之后根据像素被标记为边界的程度计算最终颜色。
        * 性能高，但是会糊。
* TemporalAA(TXAA) [UE4详见](https://de45xmedrsdbp.cloudfront.net/Resources/files/TemporalAA_small-59732822.pdf)
    对于每一帧中的物体：
    * 对帧上的每一个Object，计算出每一个动态属性的改变函数。
    * 在过滤区间上确定物体覆盖的区间
    对于每一个像素：
    * 去顶
    



# 杂项

* 在射线上，线性化后的深度差不等于射线上两像素的世界坐标距离。

* numthreads(32, 32, 1) 的性能在跑满的情况下比 numthreads(1, 1, 1)好很多。

* 一个ThreadGroup共用一块临时寄存器，所以如果临时变量太多，将 numthreads(1, 1, 1) 改成 numthreads(32, 32, 1) 在性能上并不会有太大提升。

* [显存使用](http://blog.csdn.net/toughbro/article/details/8854962)， 先马克一下。

* 详见资料：《[MSDN对齐规则](https://msdn.microsoft.com/en-us/library/windows/desktop/bb509632(v=vs.85).aspx)》
* GPU与CPU的对齐方式不一致！
* HLSL默认对齐为4字节，就是一个uint32\_t或者一个float的值。若其大小小于4字节，则自动扩充容量至4个字节，也就是说一个bool与一个uint32_t大小相等。
* 对于一个复合类型，例如说float2或者一个struct {float; int;}，则按以下规则进行对齐：
    * 空间按16个字节进行分段，也就是一个float4。
        * 0 \~ 15 段A， 16 \~ 31 段B。
    * 若上一个类型占据了当前段的头部，则尝试将当前类型在上一个类型的尾部进行拼接。
        * 上一个类型大小为8，占据了0~7，当前类型大小为32， 占据了 8 ~ 39。
    * 若当前类型跨越了一个段，则将当前整体移至下一个段的开头。
        * 因为占据了8~39,分段是在15，则么将当前整体移至16 ~ 47。
            * 0 \~ 7 (上一类型), 8 \~ 15(空)
            * 16 \~ 31 (当前类型)
            * 32 \~ 47 (当前类型)
    * 若当前类型位于段的开头，则不做调整。
* 对于`constant_buffer`中的数组，则每个元素的开头必须位于段的开头。
    * 对于float\[100\]
        * 0 \~ 3 (float\[0\]), 4 \~ 15 (空)
        * 16 \~ 19 (float\[1\]), 20 \~ 31 (空)
        * 32 \~ 35 (float\[2\]), 36 \~ 47 (空)
        * ...
* 对于`structured_buffer`，数组不需要位于段的开头，不过如果元素不是4字节对齐的话，会有20%左右的性能损耗。

* `Append/Consume Structured buffer` 与创建UAV的BUFFER一样，不过在创建UAV的时候，需要将`UAV_MISC_APPEND`传入，并且在SetUAV...的时候，有个InitCount，设置弹出和压入的初始值，最后用`Context :: CopyStructuredCount`将这个Count复制到一个BUFFER上。注意，Count是与UAV相关的。同一个BUFFER创建的不同的UAV的Count不一样。

* 关于Map，一般用于 `D3D11_USAGE_DYNAMIC` 的都只能用 `D3D11_MAP_WRITE_DISCARD` 或者 `D3D11_MAP_WRITE_NO_OVERWRITE`。 前者会让硬件重新创建一个相同大小的`buffer`，后者要保证当前写的内存不能被用于渲染中。而且一般渲染的时候GPU都会延迟1到3帧，Dx11有`Event`的方式进行控制，直到Dx12才有正式的控制方式。所以用到的 `D3D11_MAP_WRITE_NO_OVERWRITE` 一般会创建比原本大三到四倍的内存空间。所以还不如在用的时候直接创建。

* 当shader使用了SV\_InstanceID, SV\_PrimitiveID, or SV\_VertexID三种系统值时，在VS的graphic debugger中无法对单个图元中的shader进行调试，会提示“此绘图调用使用影响像素历史纪录计算的系统值语义”("This draw call is using system-value semantics and interferes with pixel history computation")，这是因为这些值都是在GPU中生成的。

* Depth Stencil View(DSV) 只支持以下几种格式:

    * `DXGI_FORMAT_D16_UNORM`
    * `DXGI_FORMAT_D24_UNORM_S8_UINT`
    * `DXGI_FORMAT_D32_FLOAT`
    * `DXGI_FORMAT_D32_FLOAT_S8X24_UINT`
    
    但是这几种格式均不支持转换成Shader Resource View(SRV)，所以无法直接在Shader中读。

    但是对于一个Depth Stencil Texture(DST)，其创建的格式与DSV和SRV的格式不必严格一致。所以我们可以按一种格式创建DST，然后按照另一种格式创建DSV，最后再按第三种格式创建SRV：

    |DST|DSV|SRV|
    |---:|---:|---:|
    DXGI\_FORMAT\_R16\_TYPELESS|DXGI\_FORMAT\_D16\_UNORM|DXGI\_FORMAT\_R16\_UNORM
    DXGI\_FORMAT\_R24G8\_TYPELESS|DXGI\_FORMAT\_D24\_UNORM\_S8\_UINT| DXGI\_FORMAT\_R24\_UNORM\_X8\_TYPELESS
    DXGI\_FORMAT\_R32\_FLOAT|DXGI\_FORMAT\_D32\_FLOAT| DXGI\_FORMAT\_R32\_FLOAT
    DXGI\_FORMAT\_R32G8X24\_TYPELESS | DXGI\_FORMAT\_D32\_FLOAT\_S8X24\_UINT |DXGI\_FORMAT\_R32\_FLOAT\_X8X24\_TYPELESS

* TYPELESS 类型

    对于 Compute Shader 来讲，并不支持格式为 `R8G8B8A8_UNORM` 的 UAV ，反而在 `cs 5_0` 的时候支持 `R8G8B8A8_UINT` 但是格式为 `R8G8B8A8_UINT` 的 SRV 并不支持 Sample 。所以必须创建格式为  `R8G8B8A8_TYPELESS` 的纹理，然后以 `R8G8B8A8_UINT` 的格式创建 UAV ，然后以 `R8G8B8A8_UNORM` 的格式创建 SRV 。

    另外如果是用 DirectX 的 SaveToWICFile 保存 `R8G8B8A8_TYPELESS` 格式的纹理时，在 CaptureTexture 之后，可以用 ScratchImage::OverrideFormat 强制转换类型为 `R8G8B8A8_UNORM` ，然后再用 SaveToWICFile 保存。

* DirectX::Image::rowPitch 含义
    
    指的是输入数据里边单行的 `字节` 大小。

* 矩阵变换

    矩阵变换一般是以下路径：Local -> World -> View -> Projection，但是一般还有一个ViewPorts的变换，这个变换的矩阵从RSSetViewPorts中进行设置。

* SV\_Position

    VertexShader中SV\_Position不可读，也就是说如果你将一个变量放进SV\_Position中后，再取出，其值已经改变了。
    另外PixelShader中读取的SV_Postion是经过ViewPorts变换过的值，与VS中输出的值不一样。

* 透视矫正

    当进行透视变换后，其结果值w不为1.0，此时应该直接将w不为1.0的值直接传给SV\_Position，该值的w会对其余的属性进行透视矫正。如果先xyzw/w之后再赋值，那么所有的属性均不进行透视矫正。

* Z-Fighting。

    这种现象是由于两个物体在 Z-Buffer 中的数值太过接近，从而无法使其区分开来。
    
    由于在投影变换中，深度与距离值的比是非线性的，在距离比较大时，其深度之差会很小。而定点数由于存在最小精度，也就是可能造成相差很大的深度用同一个数值表示，也就会造成 Z-Fighting 。
    缓解方法是加大 far - near plane的比值，或者加大深度缓冲的精度。

* Z-Buffer的线性化

    一个位于相机坐标系下的坐标点要绘制到屏幕上，还要经过投影变换和视口变换。
    
    在投影变换中：

    >   Zn = (-(f + n) / (f - n) * Ze + -2 * f * n / (f - n)) / -Ze;
    >
    >   其中，Zn表示经过投影变换后的深度，f表示远剪裁面的距离，n表示近剪裁面的距离，Ze表示点在相机坐标系下的距离。

    可得出：

    >   Ze = 2 * f * n / ((f-n) * Zn - (f + n));

    在视口变换中：

    >   Zv = (fv - nv) / 2.0 * Zn + (fv + nv) / 2.0;
    >
    >   其中，Zn表示投影变换前的深度，Zv表示视口变换后的深度，fv表示视口的最大深度，nv表示视口的最小深度。

    可得出:

    >   Zn = (2.0 * Zv - (fv + nv)) / (fv - nv);

    综上：

    >   Ze = 2 * f * n / ((f-n) * (2.0 * Zv - (fv + nv)) / (fv - nv) - (f + n))

    当然，最后得到的结果是个负数，所以一般手动加上一个-号。

* 溢出问题

    HLSL中的float最多能保持6~7位有效数字，而当深度缓冲中的值为1.0，就是没有任何物体时，其值需要除以 `(f-n)/(f+n)`，若f与n相差过大，有可能导致溢出除零。

*  `early-z`

    在通常的流水线内，深度测试执行在PS之后。因为可以在PS里边进行深度的偏移。但是在在光栅化之后，在执行PS之前，会对光栅化之后的像素进行的一个预先的深度测试。这个测试会让被遮挡的像素跳过PS直接被抛弃。这个测试是GPU提供的功能，对应用透明，也就是说开不开启完全由GPU决定。而决定这个是否开启的主要因素是是否开启深度测试和是否在PS中是否进行深度偏移。

    所以对于在不透明渲染阶段内，优先渲染靠近视点的物体，能减少GPU开销。

*   2D模拟3D纹理。

    在用2D模拟3D纹理时，需要在XY平面上添加一个边框，以防止进行采样时采样到其他平面上。根据采样模式，需要在该边框上填入对应采样的位置。
    另外，在使用Wrap线性采样时。0.0是第一个像素与最后一个像素的中间值，权重为0.5。

*   wapChain 与 Context 是分开来的，所以可以多个SwapChain对应一个Context，但是通常地，需要从Context -> IDXGIDevice -> IDXGIAdapter -> IDXGIFactory2-> IDXGISwapChain 这个流程生成交换链。

*   DirecX::CaptureTexture 需要 immediately context 作为参数。

*   从 swapChain 创建地texture创建地uav获取到的resource会在swapchain::present之后丢失。需要重新获取。

*   Swap Chain：：Present的调用需要与绘制命令的调用在同一个线程下，不然会崩。

*   在运行ComputeShader的时候，有两组参数
    Context->Dispatch(x, y, z); // 启用多少个线程组
    [numberthreads(a,b,c)] // 单个线程组中的线程个数

*   在Shader上，则几个系统参数的值为：
    SV_GroupThreadID : float3(d,e,f) 本组内的线程组的ID，总共有 （x+y+z）个线程共有这个ID。
    SV_GroupIndex ： d * a + b * e + f * c，同上。
    SV_GroupID ： 在线程组中，其值唯一，与 abc无关。
    SV_DispatchThreadID ： 所有线程唯一，表现为 SV_GroupID * SV_GroupThreadID 。