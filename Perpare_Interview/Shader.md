我们给shader增加一个Pass

```shader
Pass {
    Tags {
        "LightMode" = "Deferred"
    }
    CGPROGRAM
    #pragma target 3.0

    #pragma exclude_renderers nomrt
    #pragma shader_feature _ _RENDERING_CUTOUT
    #pragma shader_feature _METALLIC_MAP
    #pragma shader_feature _ _SMOOTHNESS_ALBEDO _SMOOTHNESS_METALLIC
    #pragma shader_feature _NORMAL_MAP
    #pragma shader_feature _OCCLUSION_MAP
    #pragma shader_feature _EMISSION_MAP
    #pragma shader_feature _DETAIL_MASK
    #pragma shader_feature _DETAIL_ALBEDO_MAP
    #pragma shader_feature _DETAIL_NORMAL_MAP
    #pragma multi_compile _ UNITY_HDR_ON

    #pragma vertex MyVertexProgram
    #pragma fragment MyFragmentProgram

    #define DEFERRED_PASS
    #include "My Lighting.cginc"

    ENDCG
}
```

它最终会把着色的结果写入G缓冲（几何缓冲）中，而不是颜色缓冲

