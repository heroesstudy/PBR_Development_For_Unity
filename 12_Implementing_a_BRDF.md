# BRDF 구현하기

## 들어가기 전에

상세 구현을 살펴보기 전에 기본 셰이더 코드의 구조를 살펴보도록 하자.

두 가지 새로운 속성이 다음과 같이 추가되었다.

```c++
Properties
{
    _Color ("Color", Color) = (1,1,1,1)
    _SpecColor ("SpecColor", Color) = (1,1,1,1)
    _MainTex ("Albedo (RGB)", 2D) = "white" {}
    _BumpMap( "BumpMap", 2D ) = "bump" {}
	_Roughness ("Roughness", Range(0,1)) = 0.0
	_Subsurface ("Subsurface", Range(0,1)) = 0.0
}
```

`_Roughness`는 표면의 거칠기를 나타내는 속성이다. 이 속성은 디퓨즈, 스펙큘러 요소 계산에 모두 사용될 것이다.

`_Subsurface`는 하부표면 근사에 사용되는 속성이다. 디퓨즈 요소 계산에만 사용된다.

다음으로 작성할 커스텀 라이팅 함수를 위한 `SurfaceOutputCustom` 구조체가 추가되었다.

```C++
struct SurfaceOutputCustom
{
	float3 Albedo;
	float3 Normal;
	float Alpha;
};

void surf (Input IN, inout SurfaceOutputCustom o)
{
    // Albedo comes from a texture tinted by color
    fixed4 c = tex2D (_MainTex, IN.uv_MainTex) * _Color;
    o.Albedo = c.rgb;
	o.Normal = UnpackNormal( tex2D( _BumpMap, IN.uv_MainTex ) );
    o.Alpha = c.a;
}
```

커스텀 라이팅 함수의 기본 구조는 다음과 같다.

```C++
void LightingCookTorrance_GI( SurfaceOutputCustom s, UnityGIInput data, inout UnityGI gi )
{
	gi = UnityGlobalIllumination( data, 1.0, s.Normal );
}

float4 LightingCookTorrance( SurfaceOutputCustom s, float3 viewDir, UnityGI gi )
{
	UnityLight light = gi.light;

	viewDir = normalize( viewDir );
	float3 lightDir = normalize( light.dir );
	s.Normal = normalize( s.Normal );

	float3 halfV = normalize( viewDir + lightDir );
	float NdotL = saturate( dot( s.Normal, lightDir ) );
	float NdotH = saturate( dot( s.Normal, halfV ) );
	float NdotV = saturate( dot( s.Normal, viewDir ) );
	float VdotH = saturate( dot( viewDir, halfV ) );
	float LdotH = saturate( dot( lightDir, halfV ) );

	float3 diff = 0; // 디퓨즈 요소 계산
	float3 spec = 0; // 스펙큘러 요소 계산

	float3 firstLayer = ( diff + spec * _SpecColor ) * _LightColor0.rgb;
	float4 c = float4( firstLayer, s.Alpha );
#ifdef UNITY_LIGHT_FUNCTION_APPLY_INDIRECT
	c.rgb += s.Albedo * gi.indirect.diffuse;
#endif
	return c;
}
```

조명 계산에 필요한 중요한 값( NdotL, NdotH 등... )을 미리 계산하고 디퓨즈와 스펙큘러 요소를 계산하는 함수에 이를 넘겨주도록 기본 구조를 잡았다.



## 쿡토렌스 구현

여기서 구현할 쿡 토렌스 구현은 시그라프 2013에서 발표된 [Real Shading in Unreal Engine 4](https://cdn2.unrealengine.com/Resources/files/2013SiggraphPresentationsNotes-26915738.pdf) 의 구현을 기본으로 한다.

일반적인 쿡 토렌스의 미세면 스펙큘러 셰이딩 모델은 다음과 같다.

![][01]

언리얼의 미세면 스펙큘러 BRDF의 각 항은 **디즈니의 모델을 기반**으로 하여 다른 대안들과 비교하며 각 항의 중요성을 평가하여 결정했다.



### Specular D

법선 분포 함수(Normal Distribution) 함수로 디즈니에서 선택한 GGX함수를 그대로 도입하였다. 

![][02]

이는 GGX의 긴 Tail이 언리얼 엔진의 아티스트들의 관심을 끌었기 때문이다.

![][03]

> 왼쪽 : 각도에 대한 스펙큘러 피크의 로그스케일 그래프
>
> black : 크롬
>
> red : GGX
>
> green : Beckmann
>
> blue : Blinn Phong
>
> 오른쪽 : 크롬, GGX, Beckmann의 점광원 반응들

다만 디즈니에서는 GGX 모델만을 단독으로 사용하지 않았다. GGX 모델은 Trowbridge-Reitz( 1975 ) 분산과 같은데 이 분산은 많은 재질을 위한 긴 테일을 제공하지 못하는 문제가 있다.



Trowbridge 와 Reitz는 젖빛 유리( ground glass ) 의 측정값에 대해서 자신들의 분산 함수와 다른 분산을 비교하였다. 그중에서 Berry ( 1923 )의 분산 함수는 매우 유사한 형태지만 지수가 2인 대신 1을 사용하여 좀 더 긴테일을 가진다.

![][04]

이를 지수 변수를 통해서 나타낸 것이 더 일반적인 분산인 Generalized-Trowbridge-Reitz, GTR이다.

![][05]

각 분산에서 `c` 는 비례상수이고 `a`는 0 ~ 1의 범위를 가지는 거칠기 매개 변수이다. 

![][06]

여기서 디즈니 모델은 거칠기를 a 를 roughness^2 로 매핑하는 것이 좀 더 선형적인 변화를 산출한다는 것을 발견하여 `a = roughness^2`로 재매핑하여 사용하고 있다. 언리얼의 구현도 이를 따른다.

디즈니의 구현은 두 개의 고정된 스펙쿨러 로브를 사용하는데 주 로브는 r = 2 을 사용하며 보조 로브는 r = 1을 사용한다.



#### 수정된 GGX 분포항

최종적으로 코드로 구현할 GGX 함수는 다음과 같다.

![][07]

 첫눈에 봤을 때는 앞에서 살펴본 GGX 함수와 많이 달라 보인다.

 미세면 분산 함수가 물리적으로 그럴듯하려면 반드시 정규화되어야 한다. 즉 다음을 만족해야 한다.

![][08]

따라서 GGX 함수에 정규화 인자가 곱해져야 한다. 정규화된 GGX함수는 아래와 같다.

![][09]

코드로 구현하면 다음과 같다. `a`를 `roughness^2`로 재매핑해서 사용하고 있다는 것을 잊지 말자.

```C++
float sqr( float value )
{
	return value * value;
}

float alpha = sqr( roughness );

// D
float alphaSqr = sqr( alpha );
float denom = sqr( NdotH ) * ( alphaSqr - 1.0f ) + 1.0f;
D = alphaSqr / ( PI * sqr( denom ) );
```



### Specular G

기하항으로는 디즈니는 GGX를 위해서 Walter( 2007 )가 유도한 모델을 사용하였다.

![][10]

반면 언리얼은 Schlick 모델을 사용하였다.

![][11]

다만 GGX를 위한 Smith 모델과 잘 맞도록 `k = a / 2`로 하였다. 이로 인해서 a = 1 일 때 GGX를 위한 Smith 모델과 정확하게 일치하고 0 ~ 1 범위에는 꽤 가깝게 근사된다.

![][12]

디즈니 모델은 여기에 더해 빛나는 표면에서 과하게 빛나는 것( gain )을 줄이기 위해서 **거칠기를 0.5 ~ 1 의 범위로 제한**하였다.

![][13]

여기서도 `a`를 `roughness^2`로 매핑했기 때문에 최종 k는

![][14]

이며 코드 구현은 다음과 같다.

```C++
float G1( float k, float x )
{
	return x / ( x * ( 1 - k ) + k );
}

float r = roughness + 1;
float k = sqr( r ) / 8;
float g1L = G1( k, NdotL );
float g1V = G1( k, NdotV );
G = g1L * g1V;
```

이는 Smith 모델의 G1 함수로 최종 기하항은

![][15]

이 된다. 코드 구현은 다음과 같다.

```C++
G = g1L * g1V;
```



### Specular F

프레넬 계산을 위해서 디즈니는 Schlick 근사를 그대로 사용하였다. 근사 계산으로 인한 에러는 다른 팩터들 때문에 발생하는 에러에 비해 무시할 수 있을 만한 수준이라고 한다.

![][16]

언리얼의 경우 여기에 약간의 수정을 가했는데 Spherical Gaussian 근사를 적용하여 식을 수정하였다. 이 수정으로 좀 더 효율적으로 계산할 수 있으며 결과는 크게 차이가 나지 않는다고 한다.

![][17]

언리얼의 구현을 따랐던 D, G 항과 다르게 F항의 구현은 디즈니의 Schlick 근사를 구현하였다. 코드 구현은 다음과 같다.

```C++
float SchlickFresnel(float value)
{
    float m = clamp(1 - value, 0, 1);
    return pow(m, 5);
}

float LdotH5 = SchlickFresnel(LdotH);
F = F0 + ( 1.0f - F0 ) * LdotH5;
```



### 전체 코드

전체 쿡 토렌스 코드는 다음과 같다.

```C++
float3 CookTorranceSpec( float NdotL, float LdotH, float NdotH,
						float NdotV, float roughness, float F0 )
{
	float alpha = sqr( roughness );
	float F, D, G;

	// D
	float alphaSqr = sqr( alpha );
	float denom = sqr( NdotH ) * ( alphaSqr - 1.0f ) + 1.0f;
	D = alphaSqr / ( PI * sqr( denom ) );

	// F
	float LdotH5 = SchlickFresnel( LdotH );
	F = F0 + ( 1.0f - F0 ) * LdotH5;

	// G
	float r = roughness + 1;
	float k = sqr( r ) / 8;
	float g1L = G1( k, NdotL );
	float g1V = G1( k, NdotV );
	G = g1L * g1V;

	float specular = NdotL * D * F * G / (4 * NdotL * NdotV + 0.000001f);
	return specular;
}
```



## 디즈니 디퓨즈 구현

몇몇 모델은 다음과 같은 디퓨즈 프레넬 요소를 포함한다.

![][18]

여기서 `F`는 반사에 대한 프레넬 요소이다.

2번의 프레넬 요소가 고려되는 것을 볼 수 있는데 1. 표면으로 들어갈 때 2. 표면에서 나올 때 두 가지 경우를 고려하기 때문이다.

과거의 경험에 의거 했을 때 램버트 모델은 가장 자리가 너무 어둡기 때문에 프레넬 요소를 도입하여 물리적으로 그럴듯하게 수정하였다.

![][19]

디퓨즈 프레넬 요소의 계산은 Schlick 근사를 사용하며 roughness 로 부터 결정된 값에 따라서 다음과 같이 계산된다.

![][20]

여기서 

![][21]

이다.

이것은 디퓨즈 프레넬 그림자를 생성하는데 부드러운 표면의 경우는 지표각에서 디퓨즈 반사율을 0.5만큼 줄이고 거친 표면에서는 디퓨즈 반사율을 2.5 만큼 증가시킨다.

셰이더에 추가된 `_Subsurface` 파라메터는 기본 디퓨즈 모델과 Hanrahan - Krueger subsurface BRDF 에서 영감을 받은 모델 블렌딩하는데 이는 멀리있는 물체와 평균 산란 거리가 작은 물체의 표면하 산란을 표현하는데 유용하다.

하지만 이 방식이 완전한 표면하 산란을 대체하지는 못한다.

```C++
float3 DisneyDiff( float3 albedo, float NdotL, float NdotV, float LdotH, float roughness, float subsurface )
{
	if ( NdotV == 0 || NdotL == 0 )
	{
		return float3( 0, 0, 0 );
	}

	float fresnelL = SchlickFresnel( NdotL );
	float fresnelV = SchlickFresnel( NdotV );

	float Fd90 = 0.5 + 2 * sqr( LdotH ) * roughness;

	float fresnelDiffuse = lerp( 1.0f, Fd90, fresnelL ) * lerp( 1.0f, Fd90, fresnelV );

	float Fss90 = LdotH * LdotH * roughness;
	float fresnelSS = lerp( 1.0f, Fss90, fresnelL ) * lerp( 1.0f, Fss90, fresnelV );
	float ss = 1.25 * ( fresnelSS * ( 1 / ( NdotL + NdotV ) - 0.5f ) + 0.5f );

	return saturate( lerp( fresnelDiffuse, ss, subsurface ) * ( 1 / PI ) * albedo );
}
```

디즈니 디퓨즈 모델은 에너지 보존을 지키지 못하는 경우가 있다.

![][22]

표면이 매끄럽고 빛의 입사각이 얕을수록 1을 넘어버리는 것이 관찰된다.

EA의 게임 엔진 Frostbite는 디즈니 디퓨즈 모델이 에너지 보존을 지키도록 정규화하여 사용하고 있다.

```C++
float3 DisneyFrostbiteDiff( float3 albedo, float NdotL, float NdotV, float LdotH, float roughness )
{
	if ( NdotV == 0 || NdotL == 0 )
	{
		return float3( 0, 0, 0 );
	}

	float energyBias = lerp( 0, 0.5, roughness );
	
	float Fd90 = energyBias + 2.0 * sqr( LdotH ) * roughness;
	float3 F0 = float3( 1, 1, 1 );

	float lightScatter = FresnelSchlickFrostbite( F0, Fd90, NdotL ).r;
	float viewScatter = FresnelSchlickFrostbite( F0, Fd90, NdotV ).r;
	
	float energyFactor = lerp( 1.0f, 1.0f / 1.51f, roughness );
	return lightScatter * viewScatter * energyFactor * ( 1 / PI ) * albedo;
}
```

![][23]

최대치가 1로 정규화된 것을 확인할 수 있다.





[01]: ./Img/12/01.png
[02]: ./Img/12/02.png
[03]: ./Img/12/03.png
[04]: ./Img/12/04.png
[05]: ./Img/12/05.png
[06]: ./Img/12/06.png
[07]: ./Img/12/07.png
[08]: ./Img/12/08.png
[09]: ./Img/12/09.png
[10]: ./Img/12/10.png
[11]: ./Img/12/11.png
[12]: ./Img/12/12.png
[13]: ./Img/12/13.png
[14]: ./Img/12/14.png
[15]: ./Img/12/15.png
[16]: ./Img/12/16.png
[17]: ./Img/12/17.png
[18]: ./Img/12/18.png
[19]: ./Img/12/19.png
[20]: ./Img/12/20.png
[21]: ./Img/12/21.png
[22]: ./Img/12/22.png
[23]: ./Img/12/23.png
[24]: ./Img/12/24.png
[25]: ./Img/12/25.png
[26]: ./Img/12/26.png
[27]: ./Img/12/27.png
[28]: ./Img/12/28.png
[29]: ./Img/12/29.png
[30]: ./Img/12/30.png