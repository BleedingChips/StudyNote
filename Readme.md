这个项目主要记载着在使用/学习DX11过程中的一些注意事项和坑。

1. [numthreads(32, 32, 1)] 的性能在跑满的情况下比 [numthreads(1, 1, 1)]好很多。
2. 当使用std::array\<float,？\>之类的数据结构传constant buffer的时候，在HLSL里边应该写float4来一次保存4个float，否者数据将会有丢失。
3. constant buffer大小有限制，应该为4096个字节，约1028个float4或者默认对齐的1028个float。
4. 传std::array\<float,？\>类似的数据时应该使用structured buffer。