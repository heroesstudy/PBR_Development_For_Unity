# 물리 기반 셰이더 제작하기

물리 기반 BRDF가 따라야 하는 3가지 규칙이 있다. 바로 양수성, 상반성, 에너지 보존이다. 여기서는 이전에 작성한 퐁 셰이더가 3가지 규칙을 만족하는지 살펴본다.



## 퐁 분석하기

다음 퐁 커스텀 라이팅 함수가 양수성, 상반성, 에너지 보존을 지키는지 살펴본다.

```C++
inline fixed4 LightingPhong( SurfaceOutput s, half3 viewDir, UnityGI gi )
{
    UnityLight light = gi.light;

    float nl = max( 0.f, dot( s.Normal, light.dir ) );

    float3 diffuseTerm = nl * s.Albedo.rgb * light.color;

    float3 reflectionDirection = reflect( -light.dir, s.Normal );
    float specularDot = max( 0.f, dot( reflectionDirection, viewDir ) );
    float specular = pow( specularDot, _Shininess );
    float3 specularTerm = specular * _SpecColor.rgb * light.color.rgb;

    float3 finalColor = diffuseTerm + specularTerm;

    fixed4 c;
    c.rgb = finalColor;
    c.a = s.Alpha;

#ifdef UNITY_LIGHT_FUNCTION_APPLY_INDIRECT
    c.rgb += s.Albedo * gi.indirect.diffuse;
#endif

    return c;
}
```



### 양수성

일반적으로 BRDF 함수가 수학적으로 양수임을 보장하는 것이 좋지만 퐁 커스텀 라이팅 함수도 다음과 같은 방법으로 양수성을 보장한다.

```C++
float specularDot = max( 0.f, dot( reflectionDirection, viewDir ) );
```



### 상반성

퐁 커스텀 라이팅 함수는 상반성을 보장하지 않는다. 퐁 커스텀 라이팅 함수는 수식으로 다음과 같이 표현할 수 있다.

![][01]

이를 brdf로 변경해보면 다음과 같다.

![][02]

여기서 입사 방향과 반사 방향을 바꿔보면 식의 분자 부분은 변함이 없지만 분모 부분은 입사 방향과 반사 방향이 바꼈을 때 변하기 때문에 상반성이 보장되지 않는다.



### 에너지 보존

작성된 퐁 커스텀 라이팅 함수는 에너지 보존을 지키지 않는다. 하지만 올바른 정규화 인자(Normalization factor)를 사용하면 에너지 보존을 지킬 수 있다.



## 개선된 퐁

퐁 커스텀 라이팅 함수가 물리 기반 BRDF를 따르도록 수정해보자. 양수성은 이미 지켜지고 있으므로 상반성과 에너지 보존을 지키도록 수정해야 한다.



### 상반성

상반성을 지키는 방법은 간단하다. 이전에 언급 했던 것 처럼 퐁 커스텀 라이팅 함수가 구현한 전통적인 퐁 모델의 BRDF에서 상반성이 위반되는 이유는 분모 부분 때문이였다.

![][02]

이를 제거하면 BRDF는 다음과 같이 되고 상반성을 지킬 수 있다. 이를 **Modified Phong BRDF**라 부른다.

![][03]

랜더링 방정식에 대입하면 최종적인 라이팅 함수의 수식은 다음과 같다.

![][04]



### 에너지 보존

BRDF가 에너지 보존을 지키기 위해서는 랜더링 방정식에 대입한 최종적인 라이팅 계산의 결과가 1을 넘어서는 안된다. 즉

![][05]

를 만족해야 한다는 이야기다. 스펙큘러가 최대인 경우에 이를 만족하면 그 외의 경우는 자동으로 에너지 보존을 만족하게 된다. 스펙큘러가 최대인 경우를 생각해보면 **입사 벡터와 법선이 이루는 각도가 0**이고 **시야 방향 벡터와 반사 벡터가 이루는 각도가 0**일 때 스펙큘러가 최대가 된다. 이 경우 식을 다음과 같이 간략화 할 수 있다.

![][06]

이 적분식을 풀면 다음과 같다.

![][07]

최종 라이팅 계산 결과가 1을 넘으면 안되므로 최종적으로 퐁 모델의 정규화 인자는 다음과 같다.

![][08]

퐁 커스텀 라이팅 함수가 물리 기반 BRDF를 따르도록 코드를 작성하면 다음과 같다.

```C++
Properties
{
	_Color( "Color", Color ) = ( 1,1,1,1 )
	_MainTex( "Albedo (RGB)", 2D ) = "white" {}
	_NormalTex( "Normal", 2D ) = "bump" {}
	_SpecColor( "Specular Material Color", Color ) = ( 1,1,1,1 )
	_Shininess( "Shininess", Range( 1, 1000 ) ) = 10
}

inline fixed4 LightingPhongModified( SurfaceOutput s, half3 viewDir, UnityGI gi )
{
    const float PI = 3.14159265358979323846;
    UnityLight light = gi.light;

    float nl = max( 0.f, dot( s.Normal, light.dir ) );

    float3 diffuseTerm = nl * s.Albedo.rgb * light.color;

    float norm = ( _Shininess + 2 ) / ( 2 * PI );
    float3 reflectionDirection = reflect( -light.dir, s.Normal );
    float specularDot = max( 0.f, dot( reflectionDirection, viewDir ) );
    float specular = norm * pow( specularDot, _Shininess );
    float3 specularTerm = specular * _SpecColor.rgb * light.color.rgb;

    float3 finalColor = diffuseTerm + specularTerm;

    fixed4 c;
    c.rgb = finalColor;
    c.a = s.Alpha;

#ifdef UNITY_LIGHT_FUNCTION_APPLY_INDIRECT
    c.rgb += s.Albedo * gi.indirect.diffuse;
#endif

    return c;
}
```



물리 기반 BRDF를 따르는 퐁 모델과 이전 퐁 모델의 차이점은 Shininess 수치를 조절해 보는 것으로 쉽게 비교할 수 있다.

![][09]

스펙큘러가 넓게 퍼졌을 때 Modified Phong 모델이 Phong 모델보다 밝기가 덜한 것을 알 수 있다. 반면 스펙큘러가 작은 영역을 차지할 때는 에너지 보존 법칙에 의해서 더 밝은 것을 확인할 수 있다.





[01]: ./Img/09/01.png
[02]: ./Img/09/02.png
[03]: ./Img/09/03.png
[04]: ./Img/09/04.png
[05]: ./Img/09/05.png
[06]: ./Img/09/06.png
[07]: ./Img/09/07.png
[08]: ./Img/09/08.png
[09]: ./Img/09/09.png