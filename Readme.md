主要记载着在使用/学习DX11过程中的一些注意事项和坑。
---

<h2 id = "JUMP_POINT_MENU">目录</h2>
---

* [性能](#JUMP_POINT_PERFORMANCE)
* [对齐](#JUMP_POINT_ALIGN)
* [限制](#JUMP_POINT_LIMITATION)
* [调试](#JUMP_POINT_DEBUG)

<h3 id = "JUMP_POINT_PERFORMANCE">性能</h3>
---

* numthreads(32, 32, 1) 的性能在跑满的情况下比 numthreads(1, 1, 1)好很多。
* [显存使用](http://blog.csdn.net/toughbro/article/details/8854962)， 先马克一下。
* [返回目录](#JUMP_POINT_MENU)

<h3 id = "JUMP_POINT_ALIGN">对齐</h3>
---

* 详见资料：《[MSDN对齐规则](https://msdn.microsoft.com/en-us/library/windows/desktop/bb509632(v=vs.85).aspx)》
* GPU与CPU的对齐方式不一致！
* HLSL默认对齐为4字节，就是一个uint32\_t或者一个float的值。若其大小小于4字节，则自动扩充容量至4个字节，也就是说一个bool与一个uint32_t大小相等。
* 对于一个复合类型，例如说float2或者一个struct {float; int;}，则按以下规则进行对齐：
    * 空间按16个字节进行分段，也就是一个float4。
        * 0~15 段A， 16~31 段B。
    * 若上一个类型占据了当前段的头部，则尝试将当前类型在上一个类型的尾部进行拼接。
        * 上一个类型大小为8，占据了0~7，当前类型大小为32， 占据了 8 ~ 39。
    * 若当前类型跨越了一个段，则将当前整体移至下一个段的开头。
        * 因为占据了8~39,分段是在15，则么将当前整体移至16 ~ 47。
            * 0~7 (上一类型)， 8~15(空)
            * 16 ~ 31 (当前类型)
            * 32 ~ 47 (当前类型)
    * 若当前类型位于段的开头，则不做调整。
* 对于constant_buffer中的数组，则每个元素的开头必须位于段的开头。
    * 对于float\[100\]
        * 0~3 (float\[0\]), 4 ~15 (空)
        * 16~19 (float\[1\]), 20 ~31 (空)
        * 32~35 (float\[2\]), 36 ~47 (空)
        * ...
* 对于structured_buffer，数组不需要位于段的开头。
* [返回目录](#JUMP_POINT_MENU)

<h3 id = "JUMP_POINT_LIMITATION">限制</h3>
---

* constant buffer大小有限制，应该为4096个字节，约1028个float4或者默认对齐的1028个float。
* [返回目录](#JUMP_POINT_MENU)

<h3 id = "JUMP_POINT_DEBUG">调试</h3>
---

* 当shader使用了SV\_InstanceID, SV\_PrimitiveID, or SV\_VertexID三种系统值时，在VS的graphic debugger中无法对单个图元中的shader进行调试，会提示“此绘图调用使用影响像素历史纪录计算的系统值语义”("This draw call is using system-value semantics and interferes with pixel history computation")，这是因为这些值都是在GPU中生成的。
* [返回目录](#JUMP_POINT_MENU)