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

    * 离线方式

        计算随机3维点，然后计算1st，2nd... 距离，然后输出纹理。

    * 实时方式

        需要HLSL下的

* PerlinNoise

* fbm
