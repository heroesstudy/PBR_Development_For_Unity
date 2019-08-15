# Orientation

## 교제

![][01]



## 유니티 셰이더

유니티 셰이더를 작성할 수 있는 방식은 크게 두가지

1. Shader Graph
   ![][02]
   
   - Unity 2018.1 버전부터 지원되는 시각적인 인터페이스를 사용한 셰이더 작성 방식.
   - Unity 2018.1 버전부터 추가된 **Scriptable Render Pipeline**(SRP) 에서 작동하기 때문에 프로젝트를 **HDRP**(High-Definition Render Pipeline) 내지 **LWRP**(Lightweight Render Pipeline)으로 생성해야 사용가능.
   
2. Shader Code
   ![][03]
   
   - 텍스트 명령의 나열로 셰이더를 작성하는 방식.
   
   - 셰이더 코드는 세가지 방식으로 작성할 수 있음.
   
     1. ShaderLab으로만 작성 (= 고정 파이프라인 셰이더)
   
        - Unity에서 제공하는 자체 셰이딩, 머티리얼 언어인 ShaderLab을 사용하여 모든 셰이더 코드를 작성하는 방법.
        - 이미 만들어진 기능만 사용해야하므로 확장성이 떨어짐.
   
     2. Surface Shader로 작성
   
        - 셰이더 코드내에 다음과 같은 구문이 보이면 Surface Shader
   
          ```ShaderLab
          #pragma surface surf Standard
          ```
   
        - ShaderLab 과 함께 일부분을 CG 셰이더 코드로 작성하는 방법.
   
        - CG 셰이더 코드는 **CGPROGRAM ~ ENDCG** 구문 사이에 작성
   
        - 버텍스 셰이더의 정점 변환등이 자동으로 처리되기 때문에 이름 그대로 표면(Surface)에만 집중하여 코드를 작성할 수 있음.
   
          > 기본적인 조명 코드도 제공 (Lambert, Blinn Phong, PBR)
   
     3. Vertex & Fragment Shader로 작성
   
        - 셰이더 코드내에 다음과 같은 구문이 보이면 Vertex & Fragment Shader
     
          ```ShaderLab
          #pragma vertex vert
        #pragma fragment frag
          ```
     
        - Surface Shader 보다 좀 더 많은 부분을 직접 작성하는 방법
        - 정점 변환 부터 조명 계산까지 직접 작성하며 HLSL의 Vertex & Pixel Shader, GLSL의 Vertex & Fragment Shader 코드 작성과 유사



## ShaderLab

셰이더 코드를 어떤 방식으로 작성하던 ShaderLab은 반드시 사용됨. 여러 문법 중에 가장 자주 사용하게 될 문법이 있는데 바로 Properties.

### Properties 

물체의 특성에 따라 셰이더에 특정 수치를 입력해 줘야하는 상황이 있음.

이 수치는 물체마다 다르게 지정해줄 수 있어야 하며 빠른 이터레이션을 위해서 실시간으로 쉽게 수정이 가능해야 하는데 유니티에서는 셰이더 코드에서 프로퍼티를 통해 해당 수치를 노출하고 설정할 수 있음.

![][04]

위와 같이 작성된 프로퍼티는 머티리얼 에셋에서 다음과 같이 노출 됨.

![][05]

프로퍼티는 '*이름 ( "머티리얼에 표시될 이름", 데이터형 ) = 초기값* ' 의 형태로 정의되며 다양한 데이터형을 제공함.

- 숫자

  - 정수
    ![][06]

    ```
    _Integer( "Integer", Int ) = 5
    ```

  - 실수
    ![][07]

    ```
    _Float( "Float", Float ) = 1.5
    ```

- 슬라이더
  ![][08]

  ```\
  _Slider( "Slider", Range( 0, 5 ) ) = 1
  ```

- 색상 
  ![][09]

  ```
  _Color ( "Color", Color ) = ( 1, 1, 1, 1 )
  ```

- 벡터
  ![][10]

  ```
  _Vector( "Vector", Vector ) = ( 1, 2, 3, 4 )
  ```

- 텍스쳐 : 기본 값으로 "white", "black", "gray", "bump", 빈문자열을 지정할 수 있음.
  
데이터형으로 2D, 3D, Cube 를 지정할 수 있음.
  ![][11]
  
  ```
_Texture( "Texture", 2D ) = "gray" {}
  ```
  
  

## Cg(C for Graphics)

Nvidia가 Microsoft 와 협력하여 개발한 고수준 셰이딩 언어. 이름처럼 C언어에 기반을 두고 개발되었으며 GPU 프로그래밍에 필요한 데이터형이 추가되었음.

2012년 이후 3.1버전을 마지막으로 Cg는 추가적인 개발이나 지원이 중단되었고 사용이 권장되지 않음.(**deprecated**) 

*하지만 유니티 셰이더 코드는 Cg로 작성되기 때문에 살펴보지 않을 수 없음.*

C 언어에 기반을 두었기에 **구조체의 선언, 함수의 선언이나 호출, 변수의 선언등 대부분의 문법은 C 언어와 동일하게 사용**할 수 있어 C 언어를 알고 있다면 어렵지 않게 사용할 수 있음.

GPU 프로그래밍에 필요한 데이터형과 내장 함수를 가볍게 살펴보도록 함.



#### 자료형

| 자료형                                                       | 비고                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| float                                                        | 32bit 실수                                                   |
| half                                                         | 16bit 실수<br />-60000 ~ 60000 소수점 이하 3자리까지 표현 가능 |
| fixed                                                        | 11bit 실수<br />-2.0 ~ 2.0 까지 표현 가능, 1 / 256 정밀도    |
| float<숫자>, half<숫자>, fixed<숫자><br />ex) float4, half2, fixed3 | 벡터형                                                       |
| float<숫자>x<숫자>, half<숫자>x<숫자>, fixed<숫자>x<숫자><br />ex) float4x4, half2x3, fixed4x3 | 행렬형                                                       |
| sampler1D, sampler2D, sampler3D, samplerCube                 | 텍스쳐와 같이 샘플링할 수 있는 외부 오브젝트를 참조하는 자료형 |



#### 내장 함수

| 함수                                          | 비고                                                      |
| --------------------------------------------- | --------------------------------------------------------- |
| abs( x )                                      | 절대 값을 반환                                            |
| cos( x ), sin( x )                            | 삼각함수                                                  |
| cross( v1, v2 )                               | 외적 벡터 반환                                            |
| dot( v1, v2 )                                 | 내적값 반환                                               |
| floor( x )                                    | 실수의 소수부분을 버린 값을 반환                          |
| lerp( a, b, f )                               | 선형보간 값을 반환                                        |
| saturate( x )                                 | 수치를 0 ~ 1 사이로 자른 값을 반환                        |
| step( y, x )                                  | y <= x 이면 1 y > x 이면 0을 반환                         |
| frac( x )                                     | 실수의 소수부분을 반환                                    |
| tex2D( sampler, uv )                          | 2D 텍스쳐를 샘플링                                        |
| mul( m, m )<br />mul( v, m )<br />mul( m, v ) | 행렬 대 행렬 곱<br />벡터 대 행렬 곱<br />행렬 대 벡터 곱 |

이외에도 [여러 내장 함수](<https://developer.download.nvidia.com/cg/index_stdlib.html>)를 제공



#### Swizzling & Masking

벡터형 자료를 다룰 때 사용되는 표기법. 다음과 같이 벡터의 요소를 임의의 순서로 접근할 수 있음.

```ShaderLab
float4 a;
float4 b;
a = b.yzxw;
```

위와 같이 요소의 값을 임의의 순서로 읽는 경우를 **스위즐링( Swizzling )**이라고 하고 다음과 같이 벡터의 요소의 특정 성분만 갱신되게 하는 표기법을 **마스킹( masking )**이라고 함.

```ShaderLab
a.yw = b;
```

위 코드는 a.y = b.y; a.w = b.w 와 동일함.



## 연습(Warped Diffuse기법)

![][12]

텍스쳐로 제작한 디퓨즈 조명의 음영을 참조하여 자유로운 음영표현이 가능하도록 하는 기법.

다음과 같이 제작한 Warp 텍스쳐를 사용해서

![][13]

다음과 같은 결과물을 얻을 수 있음.

![][14]

텍스쳐를 어떻게 칠하느냐에 따라 다양한 표현이 가능함.

1. 머티리얼 에셋을 생성.
   프로젝트 윈도우에서 마우스 오른쪽 클릭 -\> Create -\> Material
   ![][15]

2. 셰이더 에셋을 생성.

   프로젝트 윈도우에서 마우스 오른쪽 클릭 -\> Create -\> Shader -\> Standard Surface Shader
   ![][16]

3. 셰이더 에셋을 머티리얼로 드래그 & 드롭하여 적용.
   ![][17]



4. 좌측 하이라키 윈도우의 + 버튼 -\> 3D Object -\> Sphere 를 눌러 씬에 구를 추가하고 우측 인스펙터 윈도우에서 Position을 ( 0, 1, -5 ) 로 수정.
   ![][18]

   

5. 셰이더 에셋을 더블클릭하여 편집기를 띄움.

6. 기본적인 Surface Shader 코드가 이미 작성되어 있는 것을 확인 할 수 있음. 다음 부분만 남기고 모두 삭제.

   ```ShaderLab
   Shader "Custom/WarpedDiffuse"
   {
       Properties
       {
           _MainTex ("Albedo (RGB)", 2D) = "white" {}
       }
       SubShader
       {
           Tags { "RenderType"="Opaque" }
           LOD 200
   
           CGPROGRAM
           #pragma surface surf Standard fullforwardshadows
   
           #pragma target 3.0
   
           sampler2D _MainTex;
   
           struct Input
           {
               float2 uv_MainTex;
           };
   
           UNITY_INSTANCING_BUFFER_START(Props)
           UNITY_INSTANCING_BUFFER_END(Props)
   
           void surf (Input IN, inout SurfaceOutputStandard o)
           {
               fixed4 c = tex2D (_MainTex, IN.uv_MainTex);
               o.Albedo = c.rgb;
           }
           ENDCG
       }
       FallBack "Diffuse"
   }
   ```

7. 기존의 조명 함수를 사용하지 않을 것이므로 다음과 같이 사용자 정의 조명함수를 사용하도록 수정

   - pragma surface 부분 수정
     ![][19]

   - surf 함수 수정
     ![][20]

   - 사용자 정의 조명함수 추가
     ![][21]

     > 함수명은 Lighting<라이팅모델이름> 의 규칙을 따라야 함.

   

8. 사용자 정의 함수를 다음과 같이 수정

   ```ShaderLab
   float4 Lightingwarp( SurfaceOutput s, float3 lightDir, float atten )
   {
   	float ndotl = dot( s.Normal, lightDir );
   	return tex2D( _MainTex, float2( ndotl * 0.5 + 0.5, 0.5 ) );
   }
   ```

9. 머티리얼을 씬 뷰의 구에 드래그 앤 드롭하여 적용
   ![][22]

10. 프로젝트 윈도우에서 마우스 오른쪽 클릭 -\> Import New Asset을 선택하여 미리 제작한 warp 텍스쳐를 임포트
    ![][23]

11. 임포트한 텍스쳐를 클릭하여 인스펙터 윈도우에서 Wrap Mode를 Clamp로 설정
    ![][24]

12. 씬 뷰에서 구를 선택하고 우측 인스펙터 윈도우에서 머티리얼 항목의 텍스쳐 파라미터에 임포트한 warp 텍스쳐를 드래그 & 드롭
    ![][25]

    



[01]: ./Img/00/01.png

[02]: ./Img/00/02.png
[03]: ./Img/00/03.png
[04]: ./Img/00/04.png
[05]: ./Img/00/05.png
[06]: ./Img/00/06.png
[07]: ./Img/00/07.png
[08]: ./Img/00/08.png
[09]: ./Img/00/09.png
[10]: ./Img/00/10.png
[11]: ./Img/00/11.png
[12]: ./Img/00/12.png
[13]: ./Img/00/13.png
[14]: ./Img/00/14.png
[15]: ./Img/00/15.png
[16]: ./Img/00/16.png
[17]: ./Img/00/17.png
[18]: ./Img/00/18.png
[19]: ./Img/00/19.png
[20]: ./Img/00/20.png
[21]: ./Img/00/21.png
[22]: ./Img/00/22.png
[23]: ./Img/00/23.png
[24]: ./Img/00/24.png
[25]: ./Img/00/25.png

