* MaterialShader Debug

    ConsoleVariables.ini文件（通常位于Engine/Config/ConsoleVariables.ini）中输入以下代码：
    
    ```
    r.ShaderDevelopmentMode = 1
    r.DumpShaderDebugInfo = 1
    r.DumpShaderDebugShortNames = 1
    ```
    
    输入完后，修改`.ush`文件，然后再按住`ctrl`+`Shift`+逗号，即可重新编译Shader，生成的Debug信息位于相应项目的ShaderDebugInfo目录下的对应平台名字下，例如PC平台：
   
    ```
    \UE4\Samples\Games\TappyChicken\Saved\ShaderDebugInfo\PCD3D_SM5\
    ```

* Shader里读取SceneTexture

    ```hlsl
    uint StencilTextureIndex = 8; // SceneTexture的表示，具体ID可以看材质节点SceneTexture中的顺序
    float2 UV = ClampSceneTextureUV(ViewportUVToSceneTextureUV(TexCoord, StencilTextureIndex), StencilTextureIndex); // UV修正
    float4 SampleResult = SceneTextureLookup(UV, StencilTextureIndex, false); // 采样，最后一个参数表示是否使用就近点采样 
    ```
    
    有时候会提示找不到标识`SceneTextureLookup`，只需要在材质编辑器上加入`SceneTexture`并有效连线即可。

* Shader里读取非线性DepthTexture

    ```hlsl
    uint SceneTextureIndex = 1;
        float2 UV = ClampSceneTextureUV(ViewportUVToSceneTextureUV(TexCoord, SceneTextureIndex), SceneTextureIndex); // UV修正
    #if SCENE_TEXTURES_DISABLED
	    float SceneDepth = SCENE_TEXTURES_DISABLED_SCENE_DEPTH_VALUE;
    #else
        float SceneDepth = Texture2DSampleLevel(SceneTexturesStruct.SceneDepthTexture, SceneTexturesStruct.SceneDepthTextureSampler, UV, 0).r;
    #endif
    ```
    读取CustomDepth

    ```hlsl
    uint SceneTextureIndex = 13;
        float2 UV = ClampSceneTextureUV(ViewportUVToSceneTextureUV(TexCoord, SceneTextureIndex), SceneTextureIndex); // UV修正
    #if SCENE_TEXTURES_DISABLED
	    float SceneDepth = SCENE_TEXTURES_DISABLED_SCENE_DEPTH_VALUE;
    #else
        float SceneDepth = Texture2DSampleLevel(SceneTexturesStruct.CustomDepthTexture, SceneTexturesStruct.CustomDepthTextureSampler, UV, 0).r;
    #endif
    ```

    由非线性转成线性：

    ```hlsl
    #if SCENE_TEXTURES_DISABLED
	    return SCENE_TEXTURES_DISABLED_SCENE_DEPTH_VALUE;
    #else
	    return ConvertFromDeviceZ(SceneDepth);
    #endif
    ```

* LineColorTosRGB
    ```
    half3 sRGBToLinear( half3 Color ) 
    {
        Color = max(6.10352e-5, Color); // minimum positive non-denormal (fixes black problem on DX11 AMD and NV)
        return Color > 0.04045 ? pow( Color * (1.0 / 1.055) + 0.0521327, 2.4 ) : Color * (1.0 / 12.92);
    }
    ```
    在 GammaCorrentionCommon.ush

* AntiToneMapping
    ```
    return FilmToneMapInverse(Color);
    ```
    在 /Engine/Private/TonemapCommon.ush

    可直接使用版本：
    ```
    // ToneColor
    static const float3x3 AP0_2_AP1_MAT = //mul( AP0_2_XYZ_MAT, XYZ_2_AP1_MAT );
    {
        1.4514393161, -0.2365107469, -0.2149285693,
        -0.0765537734,  1.1762296998, -0.0996759264,
        0.0083161484, -0.0060324498,  0.9977163014,
    };

    static const float3x3 XYZ_2_AP1_MAT =
    {
        1.6410233797, -0.3248032942, -0.2364246952,
        -0.6636628587,  1.6153315917,  0.0167563477,
        0.0117218943, -0.0082844420,  0.9883948585,
    };

    static const float3x3 D65_2_D60_CAT =
    {
        1.01303,    0.00610531, -0.014971,
        0.00769823, 0.998165,   -0.00503203,
        -0.00284131, 0.00468516,  0.924507,
    };


    static const float3x3 sRGB_2_XYZ_MAT =
    {
        0.4124564, 0.3575761, 0.1804375,
        0.2126729, 0.7151522, 0.0721750,
        0.0193339, 0.1191920, 0.9503041,
    };

    static const float3x3 XYZ_2_sRGB_MAT =
    {
        3.2409699419, -1.5373831776, -0.4986107603,
        -0.9692436363,  1.8759675015,  0.0415550574,
        0.0556300797, -0.2039769589,  1.0569715142,
    };

    static const float3x3 D60_2_D65_CAT =
    {
        0.987224,   -0.00611327, 0.0159533,
        -0.00759836,  1.00186,    0.00533002,
        0.00307257, -0.00509595, 1.08168,
    };

    static const float3x3 AP1_2_XYZ_MAT = 
    {
        0.6624541811, 0.1340042065, 0.1561876870,
        0.2722287168, 0.6740817658, 0.0536895174,
        -0.0055746495, 0.0040607335, 1.0103391003,
    };

    static const float3 AP1_RGB2Y =
    {
        0.2722287168, //AP1_2_XYZ_MAT[0][1],
        0.6740817658, //AP1_2_XYZ_MAT[1][1],
        0.0536895174, //AP1_2_XYZ_MAT[2][1]
    };

    const float3x3 sRGB_2_AP1 = mul( XYZ_2_AP1_MAT, mul( D65_2_D60_CAT, sRGB_2_XYZ_MAT ) );
    const float3x3 AP1_2_sRGB = mul( XYZ_2_sRGB_MAT, mul( D60_2_D65_CAT, AP1_2_XYZ_MAT ) );
        
    // Use ACEScg primaries as working space
    half3 WorkingColor = mul( sRGB_2_AP1, saturate( ToneColor ) );

    WorkingColor = max( 0, WorkingColor );
        
    // Post desaturate
    WorkingColor = lerp( dot( WorkingColor, AP1_RGB2Y ), WorkingColor, 1.0 / 0.93 );

    half3 ToeColor		= 0.374816 * pow( 0.9 / min( WorkingColor, 0.8 ) - 1, -0.588729 );
    half3 ShoulderColor	= 0.227986 * pow( 1.56 / ( 1.04 - WorkingColor ) - 1, 1.02046 );

    half3 t = saturate( ( WorkingColor - 0.35 ) / ( 0.45 - 0.35 ) );
    t = (3-2*t)*t*t;
    half3 LinearColor = lerp( ToeColor, ShoulderColor, t );

    // Pre desaturate
    LinearColor = lerp( dot( LinearColor, AP1_RGB2Y ), LinearColor, 1.0 / 0.96 );

    LinearColor = mul( AP1_2_sRGB, LinearColor );

    // Returning positive sRGB values
    return max( 0, LinearColor );
    ```