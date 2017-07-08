这个项目主要记载着在使用/学习DX11过程中的一些注意事项和坑。

1. [numthreads(32, 32, 1)] 的性能在跑满的情况下比 [numthreads(1, 1, 1)]好很多。
2. 当使用std::array\<float,？\>之类的数据结构传constant buffer的时候，在HLSL里边应该写float4来一次保存4个float，否者数据将会有丢失。
3. constant buffer大小有限制，应该为4096个字节，约1028个float4或者默认对齐的1028个float。
4. 传std::array\<float,？\>类似的数据时应该使用structured buffer。
5. HLSL里边最小数据长度是4个字节，与CPU上的对齐策略不同。

    例如说struct {bool b1; bool b2};在cpu上b2和b1头部位置差一个字节，但是在GPU上差4个字节。

    另，GPU上的最齐策略是：最小4字节，尽量凑够16字节，但是如果一个数据结构跨越16个字节时，会从下一个16字节开始排列。例：
        
        ```cpp
        struct{ 
            bool b1; //从第0个字节开始，占一个字节
            float2 b2; //从第4个字节开始，占8个字节 4 + 8 < 16
            float3 b3;//从第16个字节开始, 占12个字节 12 + 12 > 16
            float4 b4; //从第32个字节开始，占16个字节 12 + 16 > 16
            };
        ```

    见[MSDN对齐规则](https://msdn.microsoft.com/en-us/library/windows/desktop/bb509632(v=vs.85).aspx)。