今天做了一个天空层与星空的合并到一个模型

主要先使用3D软件进行UV拉伸成一个球体，避免星空的2D贴图直接上去导致天顶星星明显增加，而地平线附近星星数目稀少而且出现被强行拉伸的痕迹

接下来就是用Texcoord1方式去得到这个新的球体的第二个UV坐标，让星空的Color在这个UV坐标进行展开，使得星星变回正常状态

此外，还有一个需求是需要让星星能够有闪烁效果，实现是依靠引入一张噪声图的方式，让噪声的UV随着时间进行偏移，然后与原来的星空相乘，调大Tiling（否则是一种像素刷新感），就可以出现有闪烁效果

后面就是优化的问题，我在星空图和闪烁图上各用了一个贴图，现在的新需求是把星星的闪烁贴图合并到星星贴图的G通道内，减少储存

具体的操作方式是利用Photoshop，将两个原来都是灰度的图像的一个切换成RGB，而把另外的一个放到其中的G通道上进行重新的保存

读取的时候只要读tex2D(...,...).g就可以了，减少了多一张贴图的麻烦事，下面是shader代码：

```glsl
Shader "XXX Shaders/Skybox/SkyboxProcedural"
{
	Properties
	{
		_RayleighMap("RayleighMap", 2D) = "black" {}

		_StarMap("Star,Moon,Jitter",2D) = "white" {}

		//_FlashStar("FlashStar",2D) = "white"{}
		_FlashSpeed("FlashSpeed",Range(0,10)) = 5

		_SunColor("_SunColor", Color) = (1, 1, 1, 1)
		_PartialRayleighInScattering("_PartialRayleighInScattering", Color) = (1, 1, 1, 0.1)
		_NightSkyColBase("_NightSkyColBase", Color) = (0, 0.7, 1, 1)
		_NightSkyColDelta("_NightSkyColDelta", Color) = (0.6, 0.75, 0.82, 0.4)

		_SunIntensity("_SunIntensity", Range(0, 10)) = 1
		_SunScale("_SunScale", Range(1000, 9000)) = 2000
		_SunHaloIntensity("_SunHaloIntensity", Range(0, 2)) = 0.5

		[Header(Moon)]
		_StarTransparency("_StarTransparency",Range(0, 1)) = 0.5
		_StarIntensity("_StarIntensity", Range(0, 1)) = 1

		[Toggle] _HasSun("HasSun?",float) = 0
	}
	SubShader
		{
			Tags { "RenderType" = "Background" "Queue" = "Background" "PreviewType" = "Skybox" }
			Cull Off
			LOD 100

			Pass
			{
				CGPROGRAM
				#pragma vertex vert
				#pragma fragment frag
				#pragma shader_feature  _HASCLOUD_OFF _HASCLOUD_ON
				#pragma shader_feature  _HASSUN_OFF _HASSUN_ON

				#include "UnityCG.cginc"

				struct appdata
				{
					float4 vertex : POSITION;
					fixed2 uv : TEXCOORD0;
					fixed2 uv2 : TEXCOORD1;//读取第二套UV
				};

				struct v2f
				{
					float4 vertex : SV_POSITION;
					fixed2 uv : TEXCOORD0;
					fixed4 uv2 : TEXCOORD1;//储存第二套uv，同时储存闪烁uv
					float3 localVertex : TEXCOORD2;
				};

				uniform fixed4 _GlobalSunColor;

				sampler2D _RayleighMap;
				fixed4 _PartialRayleighInScattering;
				fixed4 _NightSkyColBase;
				fixed4 _NightSkyColDelta;

				half _SunIntensity;
				half _SunScale;
				half _SunHaloIntensity;

				sampler2D _StarMap;
				float4 _StarMap_ST;
				half _StarTransparency;
				half _StarIntensity;

				//sampler2D _FlashStar;
				//float4 _FlashStar_ST;
				float _FlashSpeed;

				v2f vert(appdata v)
				{
					v2f o;
					o.vertex = UnityObjectToClipPos(v.vertex);
					o.uv = v.uv;
					o.uv2.xy = v.uv2.xy;
					//o.uv2.zw = TRANSFORM_TEX(v.uv2, _FlashStar);
					o.uv2.zw = TRANSFORM_TEX(v.uv2, _StarMap);
					o.localVertex = v.vertex.xyz;
					return o;
				}

				half4 frag(v2f i) : SV_Target
				{
					half4 color;
					color.a = 1.0;
					fixed4 rayleigh = tex2D(_RayleighMap, i.uv); //地平线贴图

					rayleigh.rgb = rayleigh.rgb * (rayleigh.a* rayleigh.a* _PartialRayleighInScattering.a);
					fixed3 viewDir = normalize(i.localVertex);

					fixed VdotL = -dot(-_WorldSpaceLightPos0.xyz, viewDir);
					fixed VdotL2 = VdotL * VdotL;

					half3 horizonCol = _PartialRayleighInScattering.rgb * rayleigh.rgb *((VdotL2 + 1.0) * 0.75);  //地平线变亮颜色 * 太阳前后两方向   *_SunIntensity   //VdotL2Sun

					fixed viewY = saturate(viewDir.y * _NightSkyColBase.a - _NightSkyColDelta.a);
					half3 skyCol = horizonCol + lerp(_NightSkyColBase.rgb, _NightSkyColDelta.rgb, viewY * (2.0 - viewY));

					//Sun
					half3 sunCol = pow((-VdotL + 1) / 2, _SunScale) * _SunIntensity * _GlobalSunColor.rgb; //horizonMulti  //   * lerp( 1.0 , horizonCol, saturate(rayleigh.r *((VdotL2 + 1.0)) * 20) 
					half3 sunHalo = pow((-VdotL + 1) / 2, _SunScale / 10) * _SunIntensity * _SunHaloIntensity * _GlobalSunColor.rgb; //horizonMulti *

					half4 starTexCol = tex2D(_StarMap,TRANSFORM_TEX(i.uv2, _StarMap));//TRANSFORM_TEX(i.uv, _StarMap)
					half starCol = starTexCol.r * _StarTransparency * pow(saturate(i.localVertex.y), 0.5) * _StarIntensity;
					float2 temp = i.uv2.zw;
					temp += _Time.y * _FlashSpeed;
					starCol *= tex2D(_StarMap, temp).g;

					#if _HASSUN_ON
						color.rgb = skyCol + (sunCol + sunHalo + starCol);
					#else
						color.rgb = skyCol + sunHalo + starCol;
					#endif

					return color;
				}
				ENDCG
			}
		}
}
```