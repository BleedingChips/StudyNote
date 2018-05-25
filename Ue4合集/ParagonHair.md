关于Paragon模型的导出问题：
选中模型编辑器，点击window -> clothing，然后把ClothingData全部删除，即可导出
见 https://www.youtube.com/watch?v=49CH1NVd-PQ

头发上的模型一共有三层UV。

参考材质：Content\ParagonShinbi\Characters\Global\Hair\Materials\M_HairSheet_Master2.uasset
在材质有使用注释划分的功能区，现按各属性划分对齐有影响的功能区：
* BaseColor : 
    * Variantion
    * Color
    * Noise
    * BottomLayer
    * HairColor
    * UV2ColorBlend
    * UseTextureForColor
    * DyeMask
    * DyeColors

* Metallic:
    * DyeMask
    * Scatter

* Specular:
    * BottomLayer

* Roughness
    * DyeMask
    * Roughness

* EmissiveColor
    * Flame
    * Emissive
    * AO
    * DyeMask

* Opacity

* OpacityMask
    * ReduceAliasingOnLowQualityMaterial
    * OpacityMask
    * EdgeMask
    * ContrastDepthValue
    * BottomLayer

* Normal
    * Trangent
    * BottomLayer

* WorldPositionOffset
    * HideHairParam,Scale verts down if hidden
    * HairWind
    * BottomLayer

* ClearCoat
    * BottomLayer
    * BlackLightOcclusion

* PixelDepthOffset
    * AnimatedDither
    * PDO
    * ReduceAliasingOnLowQualityMaterials


