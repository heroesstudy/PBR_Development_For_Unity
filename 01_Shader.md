# Shader

## 셰이더?

1. 표면(Surface)에서 일어나는 일을 코드화한 시뮬레이션
2. GPU에서 동작하는 코드



## 표면(Surface)에서 일어나는 일을 코드화한 시뮬레이션

사람이 볼 수 있는 것은 **빛이 눈으로 들어오기** 때문.

빛이 어떤 표면에 부딪치면 진행하던 매질과 다른 매질의 만나게 되고 그 결과 진행방향이 변하게 되는데 원래의 매질로 되돌아가는 경우를 **반사**라고 함.

빛이 표면에서 어떻게 반사되는지는 많은 요인이 작용.

- 빛이 들어오는 각도 : 빛의 입사각 = 빛의 반사각
  ![][01]

- 표면의 색상 : 흡수
  ![][02]

- 표면의 거친 정도 : 거친 표면이면 빛이 여러방향으로 반사
  ![][03]

- 반투명한 층의 존재여부 : 빛의 세기가 감쇄

  ![][04]
  

여러 요인들을 고려하여 표면에서 **빛이 얼마나 어느 방향으로 반사되는지를 시뮬레이션하는 것**이 셰이더의 주된 목적.

셰이더에서 시뮬레이션하고자 하는 계산을 **렌더링 방정식**으로 표현할 수 있음.

![][05]

이 방정식은 빛이 표면에 작용하는 특성을 설명.

한 표면에서 반사된 빛은 에너지가 남아있다면 한 번의 반사로 끝나지 않고 계속해서 다른 표면과 부딪혀 반사되는데 이를 **전역 조명(global illumination)**이라 함.

![][06]



## 빛의 종류

자연에서 모든 빛은 3D 표면에서 발산하는데 렌더링 과정에서는 근사법을 이용해서 필요 연산량을 줄임.

유니티에서는 실제 빛을 3가지 종류로 근사해 구분함.

- 점 광원 (Point light)
  ![][07]
- 방향 광원 (Directional light)
  ![][08]
- 면 광원 (Area light) : 유니티에서는 베이킹 라이트맵을 통해서만 지원.
  ![][09]



## 셰이더의 종류

셰이더에는 여러 종류가 존재.

- **정점 셰이더** : 모든 정점마다 한번씩 수행되는 셰이더.
- **프레그먼트 셰이더** : 최종 픽셀이 될만한 후보 픽셀( =프레그먼트 ) 마다 한번씩 수행되는 셰이더.
- **계산 셰이더** : 렌더링뿐만 아니라 다른 목적을 위한 연산( =General-Purpose computing on Graphics Processing Units  )에 사용되는 셰이더.

유니티에서 셰이더는 다음과 같은 종류로 구분됨.

- **언릿(Unlit) 셰이더** : 정점과 프레그먼트 셰이더를 한 파일로 묶은 셰이더.
- **표면(Surface ) 셰이더** : 일반적으로 라이팅 셰이더에서 사용하는 몇 가지 코드들을 자동화한 셰이더.
- **이미지 효과 셰이더** : 블러( Blur ), 블룸( Bloom ), 피사계심도(Depth of Field), 색조정( Color grading ) 등과 같은 효과를 적용하는데 사용하는 셰이더.



## 연습(빨간 셰이더)

표면의 색이 빨갛게 보이게하는 셰이더를 언릿 셰이더로 제작해 보자.

![][12]

언릿 셰이더는 프로젝트 윈도우에서 마우스 오른쪽 클릭 -\> Create -\> Shader -\> Unlit Shader 를 선택하여 생성할 수 있음.

![][10]

생성된 셰이더를 열어보면 기본적인 코드가 작성되어 있음.

```ShaderLab
Shader "Unlit/RedShader"
{
    Properties
    {
        _MainTex ("Texture", 2D) = "white" {}
    }
    SubShader
    {
        Tags { "RenderType"="Opaque" }
        LOD 100

        Pass
        {
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            // make fog work
            #pragma multi_compile_fog

            #include "UnityCG.cginc"

            struct appdata
            {
                float4 vertex : POSITION;
                float2 uv : TEXCOORD0;
            };

            struct v2f
            {
                float2 uv : TEXCOORD0;
                UNITY_FOG_COORDS(1)
                float4 vertex : SV_POSITION;
            };

            sampler2D _MainTex;
            float4 _MainTex_ST;

            v2f vert (appdata v)
            {
                v2f o;
                o.vertex = UnityObjectToClipPos(v.vertex);
                o.uv = TRANSFORM_TEX(v.uv, _MainTex);
                UNITY_TRANSFER_FOG(o,o.vertex);
                return o;
            }

            fixed4 frag (v2f i) : SV_Target
            {
                // sample the texture
                fixed4 col = tex2D(_MainTex, i.uv);
                // apply fog
                UNITY_APPLY_FOG(i.fogCoord, col);
                return col;
            }
            ENDCG
        }
    }
}
```



### 셰이더의 경로와 이름

```ShaderLab
Shader "Unlit/RedShader"
```

위의 경로와 이름을 변경하면 셰이더를 재질에 할당하기 위해 선택해야 하는 경로가 변경됨.

![][11]



### 태그

```ShaderLab
 Tags { "RenderType"="Opaque" }
```

태그는 정보를 표현하는 키와 값의 쌍으로 구성돼 있음. 위 구문은 어떤 랜더링 큐를 사용할지 지정하는 구문.



### Pragma 선언

```ShaderLab
#pragma vertex vert		// 버택스 셰이더 Entry point 함수는 vert이다.
#pragma fragment frag	// 프레그먼트 셰이더 Entry point 함수는 frag이다.
// make fog work
#pragma multi_compile_fog	// 안개 효과가 동작하도록 한다.
```

pragma는 셰이더 컴파일러에 정보를 전달하는 방법.



### 입력과 출력 구조체

```ShaderLab
struct appdata
{
    float4 vertex : POSITION;
    float2 uv : TEXCOORD0;
};

struct v2f
{
    float2 uv : TEXCOORD0;
    UNITY_FOG_COORDS(1)
    float4 vertex : SV_POSITION;
};
```

appdata는 입력 구조체를 통해 특정 정보를 받아오는 구조체.

v2f 는 정점 셰이더에서 프레그먼트 셰이더로 정보를 전달하기 위한 구조체.

> 구조체의 이름은 정해진 것이 아니므로 별도의 이름을 사용해도 됨.

구조체의 각 멤버 변수 선언에 세미콜론 다음으로 오는 단어를 시멘틱(Semantics)이라고 함.

시멘틱은 구조체 내 특정한 멤버에 어떤 종류의 정보를 저장하고자 하는지 표시하는 것.

시멘틱 중에도 SV 접두어가 붙은 시멘틱은 시스템 값(System Value)을 의미하는 것으로 특정한 목적을 위해서 미리 정의된 시멘틱임.



### 정점 함수와 프레그먼트 함수

```ShaderLab
v2f vert (appdata v)
{
	v2f o;
	o.vertex = UnityObjectToClipPos(v.vertex);
	o.uv = TRANSFORM_TEX(v.uv, _MainTex);
	UNITY_TRANSFER_FOG(o,o.vertex);
	return o;
}

fixed4 frag (v2f i) : SV_Target
{
	// sample the texture
	fixed4 col = tex2D(_MainTex, i.uv);
	// apply fog
	UNITY_APPLY_FOG(i.fogCoord, col);
	return col;
}
```



### 빨간 셰이더

frag 함수가 무조건 빨간색을 반환하도록 수정하면 빨간 셰이더가 완성.

텍스쳐 샘플링과 같은 필요 없는 기능을 모두 제외하면 최종 코드는 다음과 같음.

```ShaderLab
Shader "Unlit/RedShader"
{
    SubShader
    {
        Tags { "RenderType"="Opaque" }
        LOD 100

        Pass
        {
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag

            #include "UnityCG.cginc"

            struct appdata
            {
                float4 vertex : POSITION;
            };

            struct v2f
            {
                float4 vertex : SV_POSITION;
            };

            v2f vert (appdata v)
            {
                v2f o;
                o.vertex = UnityObjectToClipPos(v.vertex);
                return o;
            }

            fixed4 frag (v2f i) : SV_Target
            {
				return fixed4( 1.f, 0.f, 0.f, 1.f );
            }
            ENDCG
        }
    }
}
```



## 연습(단색 셰이더)

표시할 색상을 속성 값으로 하여 머티리얼에서 지정한 색상을 표시하는 단색 셰이더를 제작해 보자.

![][13]

다음과 같이 색상을 속성으로 추가.

```ShaderLab
Properties
{
	_Color( "Color", Color ) = ( 1, 1, 1, 1 )
}
```

추가한 속성은 Pass 내부에서 변수로 선언해야 사용할 수 있음.

```ShaderLab
fixed4 _Color;
```

프레그먼트 셰이더에서 속성 변수의 값을 그대로 반환하면 완성.

```ShaderLab
fixed4 frag (v2f i) : SV_Target
{
	return _Color;
}
```
[01]: ./Img/01/01.png
[02]: ./Img/01/02.png
[03]: ./Img/01/03.png
[04]: ./Img/01/04.png
[05]: ./Img/01/05.png
[06]: ./Img/01/06.png
[07]: ./Img/01/07.png
[08]: ./Img/01/08.png
[09]: ./Img/01/09.png
[10]: ./Img/01/10.png
[11]: ./Img/01/11.png
[12]: ./Img/01/12.png
[13]: ./Img/01/13.png