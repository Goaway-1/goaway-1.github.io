---
layout: post
title: "[UE5] C++ vs BP BeginPlay 호출 순서의 차이"
description: C++와 BP에서의 BeginPlay의 호출에 대해서 발생할 수 있는 이슈에 대해서 분석
date: 2025-07-27 09:00:00 +0900
categories: [Unreal Engine]
tags: [Unreal Engine, Blueprint, C++, BeginPlay]
---

## BeginPlay란
`BeginPlay`는 게임이 실행될 때, Actor, Component와 같은 클래스들이 생명주기를 시작할 때 호출되는 함수입니다. `시작` 이라는 키워드와 연관이 있다 보니, 보통 초기화나 설정 작업을 수행하는 시점으로 사용됩니다.

<p>
  <img src="./assets/img/post/BP_Beginplay/Image.png" alt="C++ 과 BP의 BeginPlay">
  <figcaption style="text-align: center; font-size: 0.8em; color: #555;">
     C++ 과 BP의 BeginPlay
  </figcaption>
</p>

<br>

이 함수는 **C++과 Blueprint** 모두에서 오버라이드해서 사용할 수 있기 때문에 많은 개발자들이 자연스럽게 둘을 섞어서 사용합니다. 하지만 흔히 **"C++의 BeginPlay가 모두 실행된 이후에 그 자식 BP의 BeginPlay가 호출된다"** 고 생각할 수 있지만, 항상 그렇지는 않습니다. 어떤 상속 구조를 가지고 있는지에 따라 다른 결과가 나오게 되죠.

이번 포스트에서는 Bueprint와 C++에서의 `BeginPlay`의 차이에 대해서 설명해보겠습니다.

<br>

---

## BeginPlay의 호출
일단 엔진에서의 `BeginPlay`가 언제, 어떤 흐름으로 호출되는지 살펴봅시다. 단순히 `Actor`가 알아서 호출되는 거라고 생각할 수 있는데, 실제로는 게임 전체 시스템의 흐름에 따라 순차적으로 호출됩니다.

<p style="text-align: center;">
  <img src="./assets/img/post/BP_Beginplay/Actor_BeginPlay.png" alt="Actor::BeginPlay">
  <figcaption style="text-align: center; font-size: 0.8em; color: #555;">
    Actor::BeginPlay
  </figcaption>
</p>

`BeginPlay` 에 중단점을 걸고, 콜 스택을 확인해 봅시다. 모두 확인할 필요는 없고, 핵심은 `UWorld::BeginPlay` 부터 시작된다는 점입니다. 월드에서 시작해서 게임모드, 액터의 순서로 호출되는 것을 볼 수 있습니다. 
이 흐름은 언리얼에서 대부분의 생명주기 로직이 월드를 중심으로 흘러간다는 걸 보여주는 대표적인 예죠.

<br>

---

## 상속 구조에서의 BeginPlay 호출 순서

<p>
  <img src="./assets/img/post/BP_Beginplay/BeginPlay_Flow.png" alt="BeginPlay_Flow" style="width: 600px;">
  <figcaption style="text-align: center; font-size: 0.8em; color: #555;">
    BeginPlay에 대한 흐름도
  </figcaption>
</p>

상속 구조에서 `BeginPlay`의 호출 순서는 어떻게 될까요? 예시를 들어봅시다. `Actor` 클래스를 상속받은 `FirstActor`라는 C++ 클래스, `FirstActor`를 상속받은 `SecondActor`라는 C++ 클래스가 있고, 마지막으로 `SecondActor`를 상속받은 Blueprint인 `ThirdActor` 클래스가 있다고 가정해봅시다.

<br>

```
ThirdActor::BeginPlay (BP)
  → Super::BeginPlay() 
    → SecondActor::BeginPlay()
      → FirstActor::BeginPlay()
        → Actor::BeginPlay()
```

이럴 경우 대부분의 개발자들은 다음처럼 호출될 거라 예상합니다. `Super::BeginPlay`을 통해서 최상위 `Actor::BeginPlay`까지 올라갔다가 아래로 순차적으로 호출될 것이라고 생각할 것입니다. 과연 그럴까요?

<br>

<p style="text-align: center;">
  <img src="./assets/img/post/BP_Beginplay/BeginPlayFlowTest.png" alt="실제 호출 순서">
  <figcaption style="text-align: center; font-size: 0.8em; color: #555;">
    실제 호출 순서
  </figcaption>
</p>

모든 함수에서 `Super`를 먼저 호출하도록 하고 실제 로그를 찍어보면, 전혀 다른 호출 순서가 나타나는 것을 볼 수 있습니다. 블루프린트의 `Beingplay`의 모든 로직이 완료되고, 이후에 상위 클래스에 대한 `BeginPlay`가 호출됩니다. 왜 일까요?

<br>

---

### 원인

```cpp
/** Event when play begins for this actor. */
UFUNCTION(BlueprintImplementableEvent, meta=(DisplayName = "BeginPlay"))
ENGINE_API void ReceiveBeginPlay();

/** Overridable native event for when play begins for this actor. */
ENGINE_API virtual void BeginPlay();
```

원인은 바로 `BeginPlay`에 대한 C++ 과 BP의 차이점에 있습니다. C++의 `BeginPlay()`와 블루프린트의 `BeginPlay()`가 실제로는 서로 다른 함수이기 때문입니다.

실제로 Actor의 헤더를 열어보면 위와 같이 `ReceiveBeginPlay`와 `BeginPlay`가 존재하는데요. C++에서 호출되는 것은 아래에 있는 `BeginPlay`이고, BP에서 호출되는 것은 `ReceiveBeginPlay`입니다. `ReceiveBeginPlay` 의 Displayname을 보면 `BeginPlay` 로 되어 있죠? 블루프린트 상의 노출은 저 이름으로 나타난다는 겁니다.

<br>

```cpp
void AActor::BeginPlay()
{
	TRACE_OBJECT_LIFETIME_BEGIN(this);
	...

	ReceiveBeginPlay();
  ...
}
```

그럼 `ReceiveBeginPlay` 는 언제 호출될까요? Actor의 본문을 열어보면, `BeginPlay` 내부의 마지막 라인에서 `ReceiveBeginPlay`를 호출하고 있습니다. 이러면 위에서 발생한 현상을 명확하게 설명할 수 있습니다. 최상위 `Actor의 Beginplay` 이후, 최하위 클래스인 `ThirdActor의 BeginPlay`가 호출되는게 맞는 흐름인것이죠.

<br>

---

### 문제 및 해결 방안
만약 상위 클래스인 `C++`의 `BeginPlay`에서 필요한 데이터가 초기화 된다고 가정해봅시다. 이후에 상위 클래스를 상속받은 `Blueprint`에서 데이터가 초기화되었다는 가정하에 작업을 진행한다면 어떻게 될까요? 당연히 해당 데이터가 초기화되어 있지 않고, 이상한 결과 값을 반환하거나 충돌이 발생하는 등의 문제가 발생할 수 있죠. 

이처럼 호출 순서를 정확히 이해하지 못하면 초기화 로직이 어긋나거나, 예측 불가능한 문제가 발생할 수 있습니다.

<br>

#### 해결 방법 1 : `DelayUntilNextTick` 응용
<p style="text-align: center;">
  <img src="./assets/img/post/BP_Beginplay/Delay.png" alt="Delay">
  <figcaption style="text-align: center; font-size: 0.8em; color: #555;">
    방법 1 : Delay
  </figcaption>
</p>


가장 단순한 방식입니다. BP의 `BeginPlay` 내부에서 `DelayUntilNextTick`을 사용해서 실행 시점을 다음 프레임으로 미루는 거죠. 이렇게 하면 C++의 BeginPlay가 우선적으로 실행되고, 그 다음에 BP 로직이 실행되게 됩니다.
하지만, **이건 어디까지나 임시 방편일 뿐, 근복적인 해결책은 절대로 아닙니다. 계산을 시간에게 맡긴다는 건 프로그래머로써 불확실성을 코드에 적용한다는 것이죠.**

<br>

> 프레임 단에서 우연한 타이밍에 의존하는 건 테스트 할 때만 한정하는게 좋습니다.
{: .prompt-warning }

 <br>

#### 해결 방법 2: `BlueprintImplementableEvent` 응용

```cpp
// SecondActor.h
UFUNCTION(BlueprintImplementableEvent, Category = "Init")
void BlueprintBeginPlay();

// SecondActor.cpp
void ASecondActor::BeginPlay()
{
    Super::BeginPlay();

    BlueprintBeginPlay(); 
}
```

C++ 클래스에 BP용 Hook 함수를 명시적으로 만들어주는 방식이 가장 깔끔합니다. 이를 C++ 클래스 계층에서 최하단인 클래스인 `SecondActor::BeginPlay` 에서 호출해둡니다. 이렇게 하고 BP에서 `BeginPlay` 대신에 해당 클래스의 내부를 정의해주면 되겠죠.

<br>

<p style="text-align: center;">
  <img src="./assets/img/post/BP_Beginplay/CorrectWay.png" alt="올바른 호출 순서">
  <figcaption style="text-align: center; font-size: 0.8em; color: #555;">
    올바른 호출 순서
  </figcaption>
</p>

이러면 상위 C++ `BeginPlay` 이 모두 종료된 이후에 BP에 해당하는`BlueprintBeginPlay` 작업이 진행되겠죠. 기존 BP에서 `BeginPlay`에 사용하던 로직을 `BlueprintBeginPlay` 로 이전해주면 되겠습니다.

<br>


> 단, 반드시 C++ 클래스 계층의 ‘최하위’에서만 호출해야 합니다. 만약 상위 클래스에서 `BlueprintBeginPlay()`를 먼저 호출해버리면 그 이후에 진행될 하위 클래스의 C++ 초기화가 **BP보다 늦어지는 꼴**이 되기 때문이죠. 즉, **C++ 로직보다 BP가 먼저 움직이는 구조**가 또다시 발생합니다.
{: .prompt-warning }

<br>

---

## 마무리
사실 이런 호출 순서 문제는 `BeginPlay`에만 국한되지 않습니다. 혹시나 해서 Receive 접두사가 붙은 함수들을 살펴봤는데, `ReceiveEndPlay`, `ReceiveTick` , `ReceiveDestroyed` 가 있더라고요. 이런 함수들 역시 **블루프린트에서의 Event 노드로 연결되는 함수들**입니다.

위에서 말했듯이 `AActor`를 상속 받은 단일 블루프린트라면 큰 문제가 발생하지 않습니다. 하지만 C++ 계층 구조가 깊어지고, 초기화 순서가 중요하다면, 순서 보장을 신경 써야 합니다. 