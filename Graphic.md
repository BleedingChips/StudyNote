这里记载一些图形学的技巧
===

* 玄学随机函数

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

* WorleyNoise

    Worley噪声首先产生一系列独立的随机点，然后对于特定像素，计算其与所有随机点的距离，然后选取其第N短的值。
    
    对于HLSL，一般情况下，其所使用的随机点需要从外部单独生成，然后每个像素均要遍历所有的随机点，选取所需要的值。
    
    若没有要求其随机点的分布类型的话也可以通过玄学随机函数在Shader中实时生成。

    在二维下，其代码如下：
    ```hlsl
    float worley_noise(float2 n)
    {
        float dis = 2.0;
        for (uint count = 0; count < 9; ++count)
        {
            float2 dir = float2((count / 3) % 3, count % 3) - 1.0;
            float2 p = floor(n / 5.0) + dir;
            //rand 是一个输入二维向量并输出随机二维向量的随机函数。
            float2 rate = rand(p);
            float2 pre_rate = rate + dir - frac(n / 5.0);
            float d = length(pre_rate);
            dis = min(dis, d);
        }
        return 1.0 - dis;
    }
    ```

    其基本思路是，将所有的区域分格，然后假设每个格子内均有一个随机点，通过格子左上的坐标生成一个K维向量，该K维向量即表示随机点与左上点的相对偏移。对于纹理中任意一点，需要计算其周围3^K-1个格子的随机点之间的距离，然后取其中第N短的值，作为其输出。

* PerlinNoise

    Perlin噪声是一种渐变式的噪声，其实现原理为将区域分块，然后再块的定点上产生随机值，最后通过衰减函数进行插值。
    
    例如在二维中，首先将区域按正方形分块，对于区域中的像素，先寻找该区域的四个顶点，然后获取其随机值，然后通过衰减函数，对其进行线性插值。最终得到的噪声，既是Perlin噪声。

    该噪声对于其随机值的来源又分成梯度噪声和值噪声，其区别仅在于随机值的获取方式不同。值噪声的随机值直接由白噪声获取，梯度噪声的随机值需要从白噪声构造一个单位向量，然后通过单位向量与当前像素与顶点的偏移向量的点乘得到其随机值。

    一般的，比较常用的衰减函数为 (t * t * t * (t * (t * 6 - 15) + 10))。

* SimpleXNoise

    SimpleX噪声是一种优化的Perlin噪声。其基本原理与Perlin噪声一样，也是将区域分块，计算顶点随机值，然后进行插值。

    不同的是Perlin一般讲区域分成正方形，正方体等，而SimpleX噪声分成的是单形。在二维下是正三角形，在三维下是四面体。在高维下，其所需要进行插值的顶点数要更少。

    其基本思路是，将区域分成可填充整个空间的单体，通过投射变换，将单体各边变成与坐标轴相平行的图形，然后计算其单体顶点的随机值，然后通过以距离为参考的衰减函数进行插值。

    其变换要求如下：
    
    *   经过变换后的单形必须保持相似并且相等。
    *   经过变换后的单形必须能填充整个空间。 

    若要使效果比较好，则要保证：

    *   单形内的任意点到达顶点的距离的标准差最小。

* fbm

    随机漫步噪声，即使通过几个不同频率的噪声进行叠加出来的噪声，其实现如下：

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

* 误差纹

    在进行步进光线追踪的时候，由于计算精度有限，会在渲染出来的效果中出现一圈圈的光纹。其解决方案即使将每次步进光线的前进值加一个微小的随机偏移。
