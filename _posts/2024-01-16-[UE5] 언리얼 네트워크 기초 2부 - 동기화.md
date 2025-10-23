---
layout: post
title: "[UE5] 언리얼 네트워크 기초 2부 - 동기화"
description: Unreal Engine Network Synchronization
date: 2024-01-16 06:00:00 +0900
categories: [Unreal Engine, Analysis]
tags: [Unreal Engine, Network, RPC, Property Replication]
---
## 시작
지난번에 이어 2부입니다! [1부](https://goaway-1.github.io/posts/UE5-언리얼-네트워크-기초-2부/)에서는 전체적인 프레임워크에 대해서 배웠다면, 2부에서는 실제로 프로그래밍에 있어 딱 필요한 정보인 `RPC`와 `Property Replication`에 대해서 준비해 봤습니다. 

<br>

## RPC

가장 먼저 `Remote Procedure Call`의 약자인 RPC로 원격 컴퓨터에 있는 함수를 호출할 수 있도록 만든 통신 프로토콜입니다. 엔진 독스에는 "로컬에서 호출되지만 다른 머신에서 원격 실행되는 함수"라고 설명되어 있는데요. 이 설명이 최고의 설명입니다. 쉽게 풀어쓰자면 "***멀티플레이에서 서버와 클라이언트 간에 빠른 통신을 통해서 행동을 명령하고 정보를 주고받으며, 클라이언트에서 서버로 통신하는 유일한 수단***"이라고 말할 수 있을 것 같습니다.

일정 시간마다 정보를 동기화하는 `Property Replication`는 다르게, `RPC`는 즉각적으로 처리하기 때문에 비교적 빠른 속도로 진행되는데요. 그렇기 때문에 주로 사운드, 파티클등 핵심적인 기능과는 무관한 일시적 효과에 주로 사용됩니다.

<br>

예를 들어 봅시다. 총알은 게임플레이에 있어 아주 중요한 요소이죠? 만약 클라이언트 1에서 총알을 발사한다고 가정한다면, 클라이언트에서 총알을 생성하는 것이 아닌, 서버에서 생성과 검증 작업이 필요합니다. 그렇기에 클라이언트에서는 서버에서 총알을 생성하도록 요청하고, 동기화하는 과정을 거치게 되죠. 바로 이게 `RPC`입니다. 만약 리슨 서버라면, 서버에서 각 클라이언트에게 요청하는 RPC가 있을 수도 있죠.

<br>

### 조건

RPC를 실행하기 위해서는 `Actor`에서 함수가 호출되어야 하고, 해당 `Actor`가 `Replicated` 설정되어 있어야 합니다. 만약 움직임을 동기화한다면 `ReplicatesMovement`도 참으로 설정되어 있어야 하죠.

<br>

### 종류

RPC의 종류는 `Server, Client, NetMulticast` 3가지로 구분됩니다. `Server`는 말 그대로 서버로 요청하는 함수이고, `Client`는 클라이언트에서, `NetMulticast`는 서버와 모든 클라이언트에서 실행되는 함수입니다. 

| 액터 소유권 | NetMulticast | 서버 | 클라이언트 |
| --- | --- | --- | --- |
| 클라이언트 | 서버와 모든 클라이언트에서 실행 | 실행 | 액터의 소유 클라이언트에서 실행 |
| 서버 | 서버와 클라이언트에서 실행 | 실행 | 서버에서 실행 |
| 미소유 | 서버와 모든 클라이언트에서 실행 | 실행 | 서버에서 실행 |

| 액터 소유권 | NetMulticast | 서버 | 클라이언트 |
| --- | --- | --- | --- |
| 호출하는 클라이언트 | 호출하는 클라이언트에서 실행 | 실행 | 호출하는 클라이언트에서 실행 |
| 다른 클라이언트 | 호출하는 클라이언트에서 실행 | X | 호출하는 클라이언트에서 실행 |
| 서버 | 호출하는 클라이언트에서 실행 | X | 호출하는 클라이언트에서 실행 |
| 미소유 | 호출하는 클라이언트에서 실행 | X | 호출하는 클라이언트에서 실행 |

하지만 위 표를 보시면, 호출된 네트워크의 상태나 액터의 소유권에 따라 실행되는 위치가 달라지게되니까 참고해주세요.

<br>

### 사용방법

```cpp
UFUNCTION( Client )
void ClientRPCFunction();

UFUNCTION( Server )
void ServerRPCFunction();
```

그렇다면 RPC로 함수를 호출해 봅시다. 함수를 정의하는 헤더 파일에 `UFUNCTION`이라는 매크로와 함께 `Server, Client, NetMulticast` 키워드들 중 하나를 붙여서 사용하면 됩니다. 이때 언리얼에서는 RPC를 사용하는 함수 앞에도 키워드를 작성하여 구분하는 것을 추천합니다.

<br>

```cpp
UFUNCTION( Client, Reliable )
void ClientRPCFunction();
```

부가적인 키워드로는 `reliable, unreliable`이 존재하는데요. 말 그대로 신뢰성에 대해서 명시하는 것입니다. `reliable`은 매우 중요하지만, 자주 호출되지 않는 함수, 충돌, 무기 발사 시작과 종료, 액터 스폰에 사용하고, `unreliable`은 서버에서 클라이언트의 함수를 호출하는 경우, 움직임과 같이 매 프레임마다 변경이 필요한 경우에 주로 사용합니다. 하지만 <span style="text-decoration: underline;">**검증이 필요한 ‘reliable’은 느리기 때문에 되도록이면 ‘unrealiable’을 사용하는 것을 추천**</span> 드립니다.
<br>

```cpp
UFUNCTION( Server, WithValidation )
void ServerRPCFunction( int32 AddHealth );

bool SomeRPCFunction_Validate( int32 AddHealth )
{
    if ( AddHealth > MAX_ADD_HEALTH )
    {
        return false;                       // This will disconnect the caller
    }
    return true;                              // This will allow the RPC to be called
}

void SomeRPCFunction_Implementation( int32 AddHealth )
{
    Health += AddHealth;
}
```

또한 인증과 관련된 키워드인 `WithValidation`이 존재하는데요. RPC를 진행하기 전에 악성 파라미터를 감지한 후에 이상이 있다면 연결을 중지하고 시스템에게 알려주는 역할을 합니다. 이 인증을 하는 내용을 구분하기 위해서 함수 구현부에서 위와 같이 `Implementation, Validate` 2개의 함수를 추가로 구현해야 합니다.

<br>

## Property Replication

프로퍼티 리플리케이션이란 액터의 정보를 네트워크 내에 있는 다른 클라이언트에게 전달 및 동기화하는 작업을 의미하는데요. 게임 도중에 변경된 값에 대해 동기화를 진행할 때 사용합니다.

RPC와 같이 데이터를 동기화하는 방법 중 한 가지이지만, RPC는 함수를 실행하는 것이 초점이라면, 이는 변수를 동기화하기 위해서 값을 전달하는 것에 주로 사용합니다. 추가로 빈도나 연관성, 우선권이라는 개념이 존재하는데, 이는 추후에 알아보도록 하겠습니다.

<br>

### 사용방법

```cpp
// Header
UPROPERTY( replicated )
AActor* Owner;

// Cpp
void 클래스명::GetLifetimeReplicatedProps( TArray< FLifetimeProperty > & OutLifetimeProps ) const
{
  // 변경 시 동기화
  DOREPLIFETIME(클래스명, Owner);

  // 조건 달성 여부에 따른 동기화
  DOREPLIFETIME_CONDITION( 클래스명, ReplicatedMovement, COND_SimulatedOnly );
}
```

사용 방법은 간단합니다. 변수를 정의하는 헤더 파일에 ‘UPROPERTY’라는 매크로와 함께 ‘replicated’라는 키워드를 붙여서 사용하고, 본문 ‘GetLifetimeReplicatedProps()’함수 내부에 명시해 주면 됩니다.

명시할 때 사용하는 매크로는 ‘DOREPLIFETIME’로 값이 변경되면 항상 동기화를 처리합니다. 만약 조건에 따른 동기화가 필요하다면, ‘[DOREPLIFETIME_CONDITION](https://docs.unrealengine.com/5.3/ko/conditional-property-replication-in-unreal-engine/)’을 사용할 수 있습니다.

<br>

```cpp
// Header
UPROPERTY(ReplicatedUsing = OnRep_RotChanged)
float ServerRotationYaw;

UFUNCTION()
void OnRep_RotChanged();

// Cpp
void 클래스명::OnRep_RotChanged()
{
  // 부가적으로 실행하고 싶은 내용
}
```

만약 값이 변경되었을 때, 부가적으로 실행하고 싶은 내용이 있다면, ‘ReplicatedUsing’이라는 추가 키워드를 사용하면 됩니다. 이때 함수의 이름에는 ‘OnRep_’이라는 접두사를 붙이는 규칙을 가지고 있습니다.

<br>

## Property Replication VS NetMulticast RPC

여기까지 했을 때, `Property Replication(PR)` 과 `NetMulticase RPC`이 굉장히 유사하다는 것을 알 수 있는데요. 서버와 모든 클라이언트가 지정함 함수를 호출할 수 있고, 지정한 데이터 전송을 보장합니다. 하지만 PR으로 설정한 데이터는 클라이언트에 반드시 동기화되지만, `RPC`는 호출한 타이밍에 클라이언트가 없으면 데이터를 받을 수 없다는 차이점이 있겠죠.

정리해 보자면, `PR`은 게임에 영향을 미치는 데이터 동기화에 사용하고, `RPC`는 게임과 무관한 휘발성 데이터에 사용하는 것을 추천드립니다.

<br>

## 마무리

살짝의 팁을 드리자면, RPC는 요청이 들어오는 즉시 처리된다고 말씀드렸죠? 그렇기 때문에 너무 많이 사용하게 되면, 많은 트래픽을 요구하게 됩니다. 그렇기에 일정 주기마다 처리하는 `roperty Replication`을 사용하는 것을 추천드립니다! 물론 상황에 맞는 동기화 방식이 꼭 필요합니다.

여기까지 프로그래밍하는 데 있어 사용하는 `RPC`와 `Property Replication`에 대해서 학습했는데요. 정말 간단한 것만 작성해 봤습니다. 자세한 것들은 추후 속편에서 설명해 보겠습니다. 그럼 오늘도 화이팅입니다!