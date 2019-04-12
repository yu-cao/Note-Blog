<h2>一些shader的常用函数与概念理解</h2>

将顶点从模型空间变换到裁剪空间：

```shaderLab
o.pos = mul(UNITY_MATRIX_MVP,v.vertex);
```

(注意，pos是一个float4的变量，为(x,y,z,1)，具体原因见https://en.wikibooks.org/wiki/Cg_Programming/Vertex_Transformations#Structure_of_the_Model_Matrix)
当然在现在我使用的Unity 2018.1.8这个版本中会把这个自动替换成

```shaderLab
UnityObjectToClipPos(v.vertex)
```

这个帮助函数

将_MainTex中的纹理变换到v.texcoord中：

```shaderLab
o.uv = TRANSFORM_TEX(v.texcoord, _MainTex);
```

获得每个顶点在世界空间中的法线(经过标准化之后的结构)：

```shaderLab
fixed3 worldNormal = normalize(mul(v.normal,(float3x3)_World2Object));
```

或者也可以是

```shaderLab
worldNormal = UntiyObjectToWorldNormal(v.normal);
```

(这个没有经过归一化，使用了帮助函数)

获得在世界空间中的光的路径(标准化之后的结果)：

```shaderLab
fixed3 worldLight = normalize(_WorldSpaceLightPos0.xyz);
```

获取模型空间下的视角位置：

```shaderLab
float3 viewer = mul(unity_WorldToObject,float4(_WorldSpaceCameraPos, 1));
```

`SV_POSITION`：裁剪空间的顶点坐标，在结构体中必须包含一个用该语义修饰的变量，通常为`float4`

获取在世界空间下的视角方向：(计算方式为相机的三维位置减去顶点的三维位置得到的一个从顶点指向相机的向量)

```shaderLab
fixed3 viewDir = normalize(_WorldSpaceCameraPos.xyz - mul(_Object2World,v.vertex).xyz);
```

UntiyCG.cginc中一些有用的帮助函数：(内置函数都是没有经过归一化的)：

|函数名|描述|
|:-:|:-:|
|`float3 WorldSpaceViewDir(float4 v)`|输入一个模型空间中的顶点位置，返回世界空间中从该点到摄像机的观察方向|
|`float3 WorldSpaceLightDir(float4 v)`|输入一个模型空间中的顶点位置，返回世界空间中从该点到光源的光照方向(没有被归一化)|
|`float3 UntiyObjectToWorldNormal(float3 norm)`|把法线方向从模型空间转换到世界空间之中|
|`float3 UnityObjectToWorldDir(float3 dir)`|把方向矢量从模型空间变换到世界空间中|
