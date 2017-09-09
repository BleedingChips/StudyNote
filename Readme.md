主要记载着在使用/学习DX11过程中的一些注意事项和坑。
---

<h2 id = "JUMP_POINT_MENU">目录</h2>

* [性能](#JUMP_POINT_PERFORMANCE)
* [对齐](#JUMP_POINT_ALIGN)
* [限制](#JUMP_POINT_LIMITATION)
* [调试](#JUMP_POINT_DEBUG)
* [格式](#JUMP_POINT_FORMAT)
* [渲染](#JUMP_POINT_RENDER)

<h3 id = "JUMP_POINT_PERFORMANCE">性能</h3>

* numthreads(32, 32, 1) 的性能在跑满的情况下比 numthreads(1, 1, 1)好很多。
* [显存使用](http://blog.csdn.net/toughbro/article/details/8854962)， 先马克一下。
* [返回目录](#JUMP_POINT_MENU)

<h3 id = "JUMP_POINT_ALIGN">对齐</h3>

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
* 对于`structured_buffer`，数组不需要位于段的开头。

* 关于Map，一般用于 `D3D11_USAGE_DYNAMIC` 的都只能用 `D3D11_MAP_WRITE_DISCARD` 或者 `D3D11_MAP_WRITE_NO_OVERWRITE`。 前者会让硬件重新创建一个相同大小的`buffer`，后者要保证当前写的内存不能被用于渲染中。而且一般渲染的时候GPU都会延迟1到3帧，Dx11有`Event`的方式进行控制，直到Dx12才有正式的控制方式。所以用到的 `D3D11_MAP_WRITE_NO_OVERWRITE` 一般会创建比原本大三到四倍的内存空间。所以还不如在用的时候直接创建。

* [返回目录](#JUMP_POINT_MENU)

<h3 id = "JUMP_POINT_LIMITATION">限制</h3>

* constant buffer大小有限制，应该为4096个字节，约1028个float4或者默认对齐的1028个float。
* [返回目录](#JUMP_POINT_MENU)

<h3 id = "JUMP_POINT_DEBUG">调试</h3>

* 当shader使用了SV\_InstanceID, SV\_PrimitiveID, or SV\_VertexID三种系统值时，在VS的graphic debugger中无法对单个图元中的shader进行调试，会提示“此绘图调用使用影响像素历史纪录计算的系统值语义”("This draw call is using system-value semantics and interferes with pixel history computation")，这是因为这些值都是在GPU中生成的。
* [返回目录](#JUMP_POINT_MENU)

<h3 id = "JUMP_POINT_FORMAT">格式</h3>

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

* [返回目录](#JUMP_POINT_MENU)

<h3 id = "JUMP_POINT_RENDER">渲染</h3>

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

    >   Zn = (-(f + n) / (f - n) * Ze + -2 * f * n / (f - n)) / Ze;
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

* [返回目录](#JUMP_POINT_MENU)