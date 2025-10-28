---
layout: post
title: "[UE5] 머티리얼 Time 노드 정밀도 문제 분석"
description: 모바일 환경에서 발생하는 Float 정밀도 문제와 해결 방법
date: 2025-10-28 09:00:00 +0900
categories: [Unreal Engine, Analysis]
tags: [Unreal Engine, Material, Precision, Float]
---
## 머티리얼 Time 노드
언리얼 엔진에서 머티리얼을 활용하다 보면, 시간 기반 애니메이션을 위해 `Time` 노드를 자주 사용하게 됩니다. 하지만 모바일 환경에서 일정 시간이 지나면 머티리얼의 **애니메이션이 끊기거나, 텍스처가 깨져 보이는** 현상이 발생하고는 하죠.

이 포스트에서는 **왜 그런 문제가 발생하는지** 그리고 **정밀도 관련 설정을 어떻게 해야 하는지**에 대해 정리해보겠습니다.

<br>

## 문제 현상

<div style="display: flex; justify-content: center; gap: 10px;">
	<figure style="text-align: center; width: 50%;">
    <img src="/assets/img/post/Material-정밀도/Material_Normal.gif" style="width: 80%; border-radius: 8px;">
    <figcaption style="font-size: 14px; color: gray; margin-top: 4px;transform: translateX(-35px);">
      정상적인 Material
    </figcaption>
  </figure>
	<figure style="text-align: center; width: 50%;">
    <img src="/assets/img/post/Material-정밀도/Material_Error.gif" style="width: 80%; border-radius: 8px;">
    <figcaption style="font-size: 14px; color: gray; margin-top: 4px;transform: translateX(-40px);">
      비정상적인 Material
    </figcaption>
  </figure>
</div>

동적으로 움직이는 Material의 경우 Time 노드를 사용하는데, 이 시간의 흐름에 따라 텍스처가 빛나거나 움직이는 경우에 대해서 해당 문제가 발생합니다. 그렇다고 시작과 동시에 바로 발생하는 현상은 아니고, 일정 시간이 지나면 위 그림과 같이 부드럽게 이동하던 텍스처가 마침 프레임이 떨어진 것 처럼 보이거나, 텍스처가 일그러지는 등의 현상이 발생하는거죠. 

<br>

## 원인 - Float 정밀도 한계

Time 노드는 Float 타입 즉, 부동소수점이기 때문에 지수부의 값이 커짐에 따라 가수부의 값을 표현할 수 있는 비트의 개수가 줄어듭니다. 그러니 시작이 지날수록 지수부를 표현하는 비트 수가 커지고, 가수부를 표현하는 비트의 수가 줄어들어 정밀도 감소하는 게 원인이죠. 정밀도가 감소하니 뚝뚝 끊어지는 현상이 발생하는 겁니다.

![유니티 공식 문서](/assets/img/post/Material-정밀도/Unity_Docs.png){: width="512" height="326"}
*유니티 공식 문서*

그럼 PC는 왜 현상이 발생하지 않을까요? [유니티 공식 문서](https://docs.unity3d.com/2021.1/Documentation/Manual/SL-DataTypesAndPrecision.html)에서는 PC GPU는 항상 고정밀 `High-precision`을 사용하지만, 모바일 GPU에서는 전력과 성능 때문에 `half-precision`, 즉 16비트로 계산하는 경우가 있다고 하네요. 이게 핵심입니다. 모바일 환경에서는 <ins>지수부와 가수부를 표현하는 비트의 수가 PC에 비해 상대적으로 적기 때문에 발생</ins>하는거죠.

<br>

## 해결 방법

### 1. Precision 설정 (GPU)

| Precision 모드 | 사용되는 데이터 타입 | 비트 수 | 정밀도 | 성능 |
| --- | --- | --- | --- | --- |
| **Half Precision** | `half` | 16-bit | 낮음 | 빠름 |
| **Default Precision** | `float` or `half` (자동 결정) | 16 or 32-bit | 상황에 따라 다름 | 중간 |
| **Full Precision** | `float` | 32-bit (IEEE 754) | 높음 | 느림 |

제일 간단한 해결 방법으로는 `Precision` 모드를 `Half`에서 `Full` 로 변경하는 것인데요. PC에서와 동일하게 항상 고정밀도로 계산하도록 하는 방법입니다. 이렇게 하면 계산에 사용되는 float 의 비트 수 자체가 달라지겠죠. 
하지만 이는 개별 노드에만 적용되는 게 아니라, 해당 머티리얼 전체에 대한 precision 정책에 영향을 주기 때문에 조금 무거운 머티리얼에 대해서는 부담을 가질 수 있는데요. 

<br>

![유니티 공식 문서](/assets/img/post/Material-정밀도/Spec.png){: width="512" height="326"}
*유니티 공식 문서*

[위 표](https://chipsandcheese.com/p/inside-snapdragon-8-gen-1s-igpu-adreno-gets-big)를 보면, 모바일 GPU(Adreno 530)가 FP32일 때보다 FP16일 때, 훨씬 빠른 처리속도를 보이는 것을 확인할 수 있습니다. 물론 GPU마다 얼마나 차이가 존재하는지는 알 수 없지만,  통상적으로 모바일 GPU에 대해서는 FP16 처리 방식으로 진행하는 것이 덜 부담이 될 것 입니다.

<br>

### [2. Period 설정 (CPU)](https://dev.epicgames.com/documentation/en-us/unreal-engine/materials-for-mobile-platforms?application_version=4.27#troubleshootingmaterialsformobile)
![Period](/assets/img/post/Material-정밀도/Period.png){: width="758" height="235"}
*Period*

GPU 정밀도 문제를 CPU 연산으로 우회하는 방법입니다. `Time` 노드에는 `Period` 속성이 있는데, 이를 활성화하면 <ins>GPU 대신 CPU에서 주기 연산이 수행되도록 바뀌고, GPU의 half 정밀도 문제가 아닌, 풀 정밀도로 시간 계산</ins>이 이루어지게 됩니다. 즉, `Period`를 활성화하면 `Time Expression`의 누적 시간이 CPU 측에서 관리되고, GPU에는 보정된 주기 정보가 전달되어 정밀도 문제를 해결할 수 있는거죠.

이 옵션은 CPU가 일정 간격마다 `fmod()` 연산을 수행해 Time을 재정렬해주어 편리하지만, 이 주기를 잘못 설정하면 오히려 애니메이션 동기화가 틀어지는 문제가 생길 수 있습니다. 

만약 주기가 1로 지정을 했을 때, Material의 실제 애니메이션 주기와 동일하지 않다면 동기화가 틀어지는 문제가 발생하겠죠. 그래서 동기화 시간을 일치시켜주는 작업은 반드시 필요합니다.

<br>

### 3. MPC (CPU)
`Period` 옵션은 좋은 해결 방법이지만, CPU가 각 머티리얼별로 주기를 계산하는 방식이기 때문에 머티리얼 개수가 많을수록 CPU 연산량이 그대로 누적된다는 문제점이 있습니다. 그래서 저는 시간에 대해서 한번만 계산하고, 이를 공통적으로 사용할 수 있도록 **MPC(Material Parameter Collection)**를 통해서 보정된 값을 전달하는 방식을 사용했습니다.

<br>

```cpp
void UpdateMPCTime(float DeltaSeconds)
{
	if (!MPC) return;

	float FractionalValue = 0.f;
	MPC->GetScalarParameterValue(MPCName, FractionalValue);

	FractionalValue += DeltaSeconds;
	FractionalValue = FMath::Frac(FractionalValue); // 0~1 사이 반복

	MPC->SetScalarParameterValue(MPCName, FractionalValue);
}
```

위 코드처럼 `DeltaSeconds`를 누적하고 Frac()으로 가수부만 남기면, 항상 0~1 범위에서 안정적으로 반복되는 시간 값을 만들 수 있죠. 이 값을 MPC를 통해 머티리얼로 전달하면 각각의 머티리얼이 하나의 MPC로만 동작합니다.

다만 이 방법 또한 문제가 있습니다. '0 ~ 1' 까지만 반복되기 때문에 모든 머티리얼이 하나의 주기로 묶이고, 각 머티리얼 마다 다른 주기나 속도를 가질 수 없습니다.

<br>

## 마무리
이렇게 모바일에서 발생할 수 있는 머티리얼 정밀도 문제에 대해서 설명해봤는데요. 완벽한 해결 방법보다는 상황에 맞는 선택이 중요할 것 같네요.

각 머티리얼이 서로 다른 주기나 속도를 가져야 할 때는 `Period` 방식을 사용하는 것이 좋겠고, 여러 머티리얼이 하나의 시간 흐름을 공유해야 할 때는 `MPC` 방식을 사용하는 것이 CPU 부하와 드로우콜을 줄일 수 있는 방법이 되겠습니다. 그냥 간단하게 해치우려면 `Float Precision Mode`를 `Full`로 설정하는게 빠르겠습니다.