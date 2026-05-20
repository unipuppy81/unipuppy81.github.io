---
title: "행렬"
date: 2026-05-20 00:00:00 +0900
categories: [Game Development, DirectX]
tags: [DirectX, Game Math, Render Pipeline]
---

# 행렬

## 1. 행렬을 쓰는 이유

3D 그래픽스에서 행렬은 정점의 좌표계를 변환하기 위해 사용한다.

```txt
Local Space
 → World Space
 → View Space
 → Clip Space
```

각 단계의 역할은 다음과 같다.

```txt
World Matrix      : Local → World
View Matrix       : World → Camera/View
Projection Matrix : View → Clip
```

DirectX / HLSL에서 자주 쓰는 행벡터 기준 계산은 다음과 같다.

```hlsl
clipPos = mul(localPos, World);
clipPos = mul(clipPos, View);
clipPos = mul(clipPos, Projection);
```

또는 수식으로 쓰면 다음과 같다.

```cpp
clipPos = localPos * World * View * Projection;
```

---

## 2. 좌표를 float4로 확장하는 이유

3D 위치는 원래 `float3`이다.

```cpp
float3 position = float3(x, y, z);
```

하지만 행렬로 이동까지 처리하려면 `w`를 포함한 `float4`로 확장한다.

```cpp
float4 localPos = float4(x, y, z, 1.0f);
```

`w = 1`이면 위치 좌표다.

```txt
w = 1 : 위치, Translation 영향 받음
w = 0 : 방향, Translation 영향 안 받음
```

그래서 Normal, Direction 같은 방향 벡터는 보통 `w = 0`으로 계산한다.

```hlsl
float4 normal = float4(input.normal, 0.0f);
```

---

## 3. 행렬 곱셈 방식

행벡터 기준에서 벡터와 4x4 행렬의 곱은 다음 형태다.

```txt
[x y z w] * Matrix
```

행렬이 다음과 같을 때:

```txt
[ m00 m01 m02 m03 ]
[ m10 m11 m12 m13 ]
[ m20 m21 m22 m23 ]
[ m30 m31 m32 m33 ]
```

결과 좌표는 다음과 같다.

```cpp
result.x = x * m00 + y * m10 + z * m20 + w * m30;
result.y = x * m01 + y * m11 + z * m21 + w * m31;
result.z = x * m02 + y * m12 + z * m22 + w * m32;
result.w = x * m03 + y * m13 + z * m23 + w * m33;
```

행벡터 기준에서는 Translation 값이 행렬의 마지막 행에 들어간다.

```txt
[ 1  0  0  0 ]
[ 0  1  0  0 ]
[ 0  0  1  0 ]
[ tx ty tz 1 ]
```

---

## 4. World Matrix

World Matrix는 오브젝트의 로컬 좌표를 월드 좌표로 변환한다.

```txt
Local Space → World Space
```

적용되는 요소는 다음 3가지다.

```txt
Scale       : 크기
Rotation    : 회전
Translation : 위치
```

행벡터 기준 World Matrix의 일반 형태는 다음과 같다.

```txt
World =
[ xx  xy  xz  0 ]
[ yx  yy  yz  0 ]
[ zx  zy  zz  0 ]
[ tx  ty  tz  1 ]
```

`xx ~ zz`는 회전과 스케일이 섞인 3x3 영역이고, `tx ty tz`는 월드 위치다.

---

## 5. Scale Matrix

```txt
Scale =
[ sx  0   0   0 ]
[ 0   sy  0   0 ]
[ 0   0   sz  0 ]
[ 0   0   0   1 ]
```

예:

```cpp
localPos = [1, 1, 1, 1]
Scale    = scale(2, 3, 4)

result = [2, 3, 4, 1]
```

---

## 6. Rotation Matrix

### X축 회전

```txt
RotationX =
[ 1   0      0     0 ]
[ 0   cosθ   sinθ  0 ]
[ 0  -sinθ   cosθ  0 ]
[ 0   0      0     1 ]
```

### Y축 회전

```txt
RotationY =
[ cosθ  0  -sinθ  0 ]
[ 0     1   0     0 ]
[ sinθ  0   cosθ  0 ]
[ 0     0   0     1 ]
```

### Z축 회전

```txt
RotationZ =
[ cosθ   sinθ  0  0 ]
[-sinθ   cosθ  0  0 ]
[ 0      0     1  0 ]
[ 0      0     0  1 ]
```

---

## 7. Translation Matrix

```txt
Translation =
[ 1   0   0   0 ]
[ 0   1   0   0 ]
[ 0   0   1   0 ]
[ tx  ty  tz  1 ]
```

예:

```cpp
localPos = [1, 2, 3, 1]
Translation = move(10, 0, 5)

result = [11, 2, 8, 1]
```

---

## 8. World Matrix 조합 순서

행벡터 기준에서는 보통 다음 순서를 쓴다.

```cpp
World = Scale * Rotation * Translation;
```

정점 기준으로는 다음 순서로 적용된다.

```cpp
worldPos = localPos * Scale * Rotation * Translation;
```

즉 실제 적용 순서는 다음과 같다.

```txt
1. Scale
2. Rotation
3. Translation
```

주의할 점은 행렬 곱셈은 교환법칙이 성립하지 않는다는 것이다.

```cpp
Scale * Rotation * Translation != Translation * Rotation * Scale
```

---

## 9. View Matrix

View Matrix는 월드 좌표를 카메라 기준 좌표로 변환한다.

```txt
World Space → View Space
```

핵심은 다음이다.

```cpp
View = inverse(CameraWorldMatrix);
```

즉, 카메라를 움직이는 대신 월드 전체를 카메라의 반대 방향으로 변환한다.

---

## 10. LookAt View Matrix

View Matrix는 보통 카메라 위치, 바라보는 지점, 위쪽 방향으로 만든다.

```cpp
Eye    = camera position
Target = look target
Up     = camera up direction
```

DirectX Left-Handed 기준 축 계산은 다음과 같다.

```cpp
ZAxis = normalize(Target - Eye);      // Forward
XAxis = normalize(cross(Up, ZAxis));  // Right
YAxis = cross(ZAxis, XAxis);          // Up
```

행벡터 기준 View Matrix는 다음 형태다.

```txt
View =
[ XAxis.x   YAxis.x   ZAxis.x   0 ]
[ XAxis.y   YAxis.y   ZAxis.y   0 ]
[ XAxis.z   YAxis.z   ZAxis.z   0 ]
[-dot(XAxis, Eye)  -dot(YAxis, Eye)  -dot(ZAxis, Eye)  1 ]
```

의미:

```txt
XAxis : 카메라 오른쪽 방향
YAxis : 카메라 위쪽 방향
ZAxis : 카메라 전방 방향
Eye   : 카메라 위치
```

---

## 11. Projection Matrix

Projection Matrix는 View Space 좌표를 Clip Space로 변환한다.

```txt
View Space → Clip Space
```

원근 투영에서 필요한 값은 다음이다.

```txt
FovY   : 세로 시야각
Aspect : 화면 비율, width / height
NearZ  : 가까운 절단면
FarZ   : 먼 절단면
```

DirectX Left-Handed, 행벡터 기준 Perspective Matrix는 다음과 같다.

```cpp
yScale = 1 / tan(FovY / 2);
xScale = yScale / Aspect;
```

```txt
Projection =
[ xScale  0       0                          0 ]
[ 0       yScale  0                          0 ]
[ 0       0       FarZ / (FarZ - NearZ)      1 ]
[ 0       0      -NearZ * FarZ / (FarZ - NearZ)  0 ]
```

Projection 이후 좌표는 `clipPos = (x, y, z, w)`가 된다.

---

## 12. Perspective Divide

Projection Matrix 이후 GPU는 `w`로 나누어 NDC 좌표를 만든다.

```cpp
ndc.x = clip.x / clip.w;
ndc.y = clip.y / clip.w;
ndc.z = clip.z / clip.w;
```

DirectX 기준 NDC 범위는 다음과 같다.

```txt
x : -1 ~ 1
y : -1 ~ 1
z :  0 ~ 1
```

---

## 13. Viewport Transform

NDC 좌표는 실제 화면 픽셀 좌표로 변환된다.

```cpp
screen.x = (ndc.x + 1.0f) * 0.5f * screenWidth;
screen.y = (1.0f - ndc.y) * 0.5f * screenHeight;
```

이후 삼각형은 화면 공간에서 래스터라이제이션 단계로 넘어간다.

---

## 14. HLSL 예시

```hlsl
cbuffer TransformBuffer : register(b0)
{
    float4x4 World;
    float4x4 View;
    float4x4 Projection;
    float4x4 WorldInverseTranspose;
};

struct VS_IN
{
    float3 position : POSITION;
    float3 normal   : NORMAL;
    float2 uv       : TEXCOORD;
};

struct VS_OUT
{
    float4 position  : SV_POSITION;
    float3 worldPos  : POSITION;
    float3 worldNorm : NORMAL;
    float2 uv        : TEXCOORD;
};

VS_OUT VSMain(VS_IN input)
{
    VS_OUT output;

    float4 localPos = float4(input.position, 1.0f);

    float4 worldPos = mul(localPos, World);
    float4 viewPos  = mul(worldPos, View);
    float4 clipPos  = mul(viewPos, Projection);

    output.position = clipPos;
    output.worldPos = worldPos.xyz;

    // Normal은 방향 벡터이므로 w = 0이다.
    output.worldNorm = normalize(mul(float4(input.normal, 0.0f), WorldInverseTranspose).xyz);
    output.uv = input.uv;

    return output;
}
```

---

## 15. 핵심 요약

```txt
World Matrix
- 오브젝트를 월드에 배치한다.
- Scale, Rotation, Translation을 포함한다.

View Matrix
- 월드를 카메라 기준 좌표로 바꾼다.
- CameraWorldMatrix의 역행렬이다.

Projection Matrix
- View Space를 Clip Space로 변환한다.
- 원근감과 시야 범위를 결정한다.

Perspective Divide
- clip.xyz / clip.w
- Clip Space를 NDC로 변환한다.

Viewport Transform
- NDC를 실제 화면 픽셀 좌표로 변환한다.
```
