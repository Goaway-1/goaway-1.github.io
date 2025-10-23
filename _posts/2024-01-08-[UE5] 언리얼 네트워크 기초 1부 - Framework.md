---
layout: post
title: "[UE5] 언리얼 네트워크 기초 1부 - Framework"
description: Unreal Engine Network Framework
date: 2024-01-08 06:00:00 +0900
categories: [Unreal Engine, Analysis]
tags: [Unreal Engine, Network, Framework]
---
## 멀티는 어려워

게임 개발을 시작한 지 얼마 안 된 초보자분들은 싱글플레이가 아닌 멀티플레이 게임을 개발하기에는 조금 두려운 마음이 있으실 겁니다. 네트워크에 대한 기본적인 개념도 알고 있어야 하고, 언리얼 네트워크의 프레임워크등 언리얼에서의 사용법 또한 알고 있어야 하기 때문이죠.

<br>

![※ Udemy 언리얼 강의](/assets/img/post/Network-1/Udemy.png){: width="704" height="254"}
*Udemy에서 들었던 강의들*

저도 막 게임 개발을 시작했을 때에는 공식 문서를 봐도 하나도 이해가 안돼서 ‘Udemy’와 같은 강의 사이트에서 네트워크 강의를 결제하고 공부했던 기억이 있네요. 근데 문제는 제가 그리 총명하지 않습니다. 잘 이해가 안 되더라고요. 결국에는 시간으로 밀어 붙여서 이해할 수 있었습니다. 여러분은 제가 설명하는 것을 쉽게 이해하실 수 있을 것이라고 믿어 의심치 않습니다.

<br>

## 언리얼 네트워크 프레임워크

플레이에 있어 싱글과 달리 멀티에서 가장 중요한 것은 무엇일까요? 바로 **정보를 공유**하는 것입니다. 싱글에서는 모든 정보가 하나의 클라이언트에 저장되면 끝이지만, 멀티플레이의 경우 플레이어 간에 정보를 공유해서 각 클라이언트의 정보가 모두 일치해야 합니다. 이 과정은 매우 복잡하지만, 언리얼 엔진에서 제공하는 네트워크 프레임워크를 통해서 비교적 편리하게 사용할 수 있어요.

<br>

### 네트워크 모델

![※ 클래스의 구분](/assets/img/post/Network-1/ServerClass.png){: width="630" height="420"}
*멀티플레이에서의 클래스*

위에 존재하는 이름들이 익숙하죠? 바로 언리얼에서 게임을 구성하기 위해 필요한 기초적인 클래스들입니다. 이 클래스들은 네트워크 모델에서 구분되어 존재합니다.

1. `AGameMode`는 서버에만 존재하며, 레벨 내의 게임 룰이나 정보 같은 것을 저장하는 중요한 역할입니다. 서버에만 존재하니 클라이언트에서는 직접 접근할 수 없죠.
2. `AGameState, APlayerState, APawn, ACharacter` 클래스들은 서버와 클라이언트 모두에서 접근이 가능합니다. 위에서 `AGameMode`에 클라이언트가 직접 접근할 수 없다고 했죠? 그래서 각 클라이언트는 `AGameState`에 접근하여 정보를 제공하거나 받고, 업데이트하는 식으로 사용하고는 합니다.
3. `APlayerController`는 서버와 소유 클라이언트에만 존재하며, 컨트롤러는 캐릭터를 조종하는 즉, 사용자의 입력을 받는 곳이죠. 그렇기 때문에 다른 클라이언트에 복제될 필요가 없습니다. 입력은 나만 하니까요.
4. `AHUD, UUSerWidget`는 소유 클라이언트에만 존재하며, 플레이어의 디스플레이에 표시되는 ‘체력, 시간’등의 정보를 표현하는 곳이죠. 그러니 역시 서버에 존재할 필요도 복제될 필요도 없습니다.

<br>

### 서버-클라이언트 모델

![※ 서버-클라이언트 모델](/assets/img/post/Network-1/Server-Client.png){: width="628" height="324"}
*서버-클라이언트 모델*


언리얼 엔진은 “서버-클라이언트” 아키텍처를 사용하는데요. 이는 서버가 권한을 가지고 있으며, 모든 데이터를 서버에서 우선시되어야 한다는 것입니다. 그런 다음 서버에서 데이터를 확인하고 각 클라이언트에서 반응하는 것이죠. 쉬운 예를 들자면, 클라이언트에서 캐릭터를 이동할 때, 직접 이동하는 게 아니라 서버에서 이동하고 이를 각 클라이언트에서 복제하는 것이죠. (기능에 따라 아닌 경우도 있습니다.)

싱글플레이에서는 하나의 컴퓨터에서 입력과 액터들을 제어하지만, 멀티플레이에서는 네트워크에 있는 한 대의 컴퓨터가 서버 역할을 하여 세션을 호스팅 합니다. 나머지는 클라이언트로 서버 컴퓨터에 연결되며, 이때 서버는 호스팅뿐만 아니라 게임이 실제로 진행되는 곳이고, 각 클라이언트들에게 복제해 주기 위해서 동기화 통신을 제공합니다. 위 그림과 같이 서버에서 생성된 것들을 각 클라이언트에 복제하며, 이 서버로부터 복제된 허상을 `Proxy`라고 합니다.

게임플레이에 영향을 미치는 모든 것을 진행한다고 서버에서 생각하면 되겠습니다. 아까 말씀드린 것처럼 개인 HUD에 보이는 것은 클라이언트에서 진행해도 상관없지만, 움직임이나 투사체의 이동 등은 게임에 큰 영향을 미치기 때문에 서버에서 진행되어야만 하죠. 물론 프로그래머의 선택에 따라 모든 것이 달라질 수 있습니다만, 네트워크 대역폭을 최대한 적게 사용하는 것이 좋은 퀄리티를 제공할 수 있겠죠.

<br>

## 원격 액터의 초기화

그렇다면 서버에 존재하는 허상을 물려받는 클라이언트의 액터에서는 어떤 작업이 진행될까요? 기존 액터의 초기화 과정에서는 `PostInitializeComponents` 함수에서 게임 플레이와 무관한 설정을 초기화해 주고, `BeginPlay`에서는 게임 플레이와 관련된 것들을 초기화해 줍니다. 하지만 원격 액터는 `PostInitializeComponents`를 통해 설정된 값들이 클라이언트에게 전달되지 않습니다. 이때 사용되는 함수가 바로 `PostNetInit`인데요. 원격 클라이언트에게 네트워크 관련된 설정을 초기화해 줍니다.

아래 코드를 보면 클라이언트가 아니라면 경고를 띄워주고, 해당 액터가 `BeginPlay`를 호출하지 않았다면 추가로 네트워크 관련 작업과 `BeginPlay`를 호출하는 것을 볼 수 있습니다.

```cpp
void AActor::PostNetInit()
{
	if(RemoteRole != ROLE_Authority)
	{
		UE_LOG(LogActor, Warning, TEXT("AActor::PostNetInit %s Remoterole: %d"), *GetName(), (int)RemoteRole);
	}
			
	if (!HasActorBegunPlay())
	{
		const UWorld* MyWorld = GetWorld();
		if (MyWorld && MyWorld->HasBegunPlay())
		{
			SCOPE_CYCLE_COUNTER(STAT_ActorBeginPlay);
			DispatchBeginPlay();
		}
	}
}
    
void AActor::DispatchBeginPlay(bool bFromLevelStreaming)
{
	UWorld* World = (!HasActorBegunPlay() && IsValidChecked(this) ? GetWorld() : nullptr);
			
	if (World)
	{
		ensureMsgf(ActorHasBegunPlay == EActorBeginPlayState::HasNotBegunPlay, TEXT("BeginPlay was called on actor %s which was in state %d"), *GetPathName(), (int32)ActorHasBegunPlay);
			
		const uint32 CurrentCallDepth = BeginPlayCallDepth++;
			
		bActorBeginningPlayFromLevelStreaming = bFromLevelStreaming;
		ActorHasBegunPlay = EActorBeginPlayState::BeginningPlay;
			
		BuildReplicatedComponentsInfo();
			
		#if UE_WITH_IRIS
		BeginReplication();
		#endif // UE_WITH_IRIS
			
		BeginPlay();
			
		ensure(BeginPlayCallDepth - 1 == CurrentCallDepth);
		BeginPlayCallDepth = CurrentCallDepth;
			
		if (bActorWantsDestroyDuringBeginPlay)
		{
			// Pass true for bNetForce as either it doesn't matter or it was true the first time to even 
			// get to the point we set bActorWantsDestroyDuringBeginPlay to true
			World->DestroyActor(this, true); 
		}
					
		if (IsValidChecked(this))
		{
			// Initialize overlap state
			UpdateInitialOverlaps(bFromLevelStreaming);
		}
			
		bActorBeginningPlayFromLevelStreaming = false;
	}
}
```
    
> 처음에는 중요하지 않지만, 개발하다 보면 가끔 문제가 생기더라고요. 알고만 있다가 생각나면 찾아보시는 것도 나쁘지 않습니다.
{: .prompt-tip }

<br>

## 액터의 역할 및 종류

멀티플레이에서의 액터는 현재 네트워크 상태에 따라 역할이 구분됩니다. 위에서 설명했듯이 서버에 존재하는 액터를 각 클라이언트에 복제한 것을 `Proxy`라고 했는데요. 그렇다면 서버 액터는 뭐라고 할까요? 바로 `Authority`라고 부릅니다. 쉽게 지휘자, 대리자라고 생각하면 되겠네요. 종류에는 무엇이 있을까요?

| 종류 | 내용 |
| --- | --- |
| None | 액터가 존재하지 않음 |
| Authority | 서비스를 대표하는 신뢰할 수 있는 역할로 게임의 로직을 수행 |
| AutonomousProxy | Authority를 가지는 오브젝트의 복제품으로 일부 게임 로직을 수행. 클라이언트의 입력 정보를 서버에 보낼 수 있기 때문에 '폰, 컨트롤러’가 적합 |
| SimulatedProxy | Authority를 가지는 오브젝트의 복제품으로 게임 로직을 수행하지 않음. 일방적으로 서버로부터 데이터를 수신하고 반영하는 배경에 적합 |

이처럼 액터를 신뢰할 수 있는지 없는지 아니면, 단방향 통신이 가능한지등의 역할들을 구분하기 위해서 언리얼 엔진은 `AActor::HasAuthority`를 통해서 `Authority`의 소유권을 판단할 수 있고, `AController::IsLocalController, APawn::IsLocallyControlled`등의 함수를 통해서 `AutonomousProxy`를 판단할 수 있습니다. 이를 구분할 수 있어야만, 각 상황에 따른 액터의 코드 실행을 분리할 수 있겠죠? 특히 리슨서버에서 동작하는 게임을 만든다면 특히나 필요할 것입니다.

> 추가로 리슨서버에서 현재 동작하는 게임의 역할을 ‘Local Role’이라고 하고, 연결된 게임의 역할을 ‘Remote Role’이라고 합니다.
{: .prompt-tip }

<br>

## 네트워크 모드

|Net Mode|Desciption|
| --- | --- |
| Standalone | 싱글플레이 또는 로컬 멀티플레이 게임에서 사용되며, 하나의 컴퓨터에서 진행 |
| Client | 네트워크 멀티플레이 세션에서 서버에 연결된 클라이언트로 실행 |
| Listen Server | 네트워크 멀티플레이 세션을 호스팅하는 서버로 클라이언트들의 연결을 수락 |
| Dedicated Server | Listen Server와 동일하게 클라이언트들의 연결을 허용하지만, 로컬 플레이어는 없이 오직 서버의 역할만 진행 |

게임을 플레이할 때, 네트워크 상황에 따른 모드는 총 4가지로 구분되며, 네트워크 멀티플레이어 세션의 관계를 설정합니다. Listen Server와 Dedicated Server에 대해서는 조금 더 설명을 드리겠습니다.

먼저 `Listen Server`는 게임 사본이 있는 사용자라면, 누구나 리슨 서버를 시작하고 호스팅하여 플레이할 수 있습니다. 하지만 호스트는 서버에서 직접 플레이를 진행하기 때문에 네트워크 지연에 있어 문제가 없고, 공정성 및 부정행위를 할 수 있는 우려가 존재하죠. 그렇기에 경쟁이 치열하거나 네트워크 부하가 높은 게임에는 적합하지 않습니다.

다음으로 `Dedicated Server`는 자체 네트워크가 필요하기 때문에 비싸고 구성에 있어 어려움이 존재합니다. 하지만 모든 플레이어에게 동일한 연결 및 동기화를 지원하기 때문에 공정성이 보장되고, 해당 서버에서는 시각적인 렌더링이나 로직을 수행하지 않기 때문에 네트워크를 효율적으로 사용할 수 있다는 장점이 있습니다.

<br>

### 주요 함수

이 네트워크 접속 과정에서 중요한 함수가 몇가지 있는데요. 바로 GameMode와 GameState에서의 함수입니다. 하나씩 보시면서 이해하시면, 개발에 있어 도움이 될거 같네요.

|GameMode 함수| 내용 |
| --- | --- |
| PreLogin | 클라이언트의 접속 요청을 처리하는 함수 (받아드리는 여부로 서버에서는 진행 X) |
| Login | 접속을 허용한 클라이언트에 대응하는 플레이어 컨트롤러를 만드는 함수 |
| PostLogin | 플레이어 입장을 위해 플레이어에 필요한 기본 설정을 모두 마무리하는 함수 (캐릭터 세팅) |
| StartPlay | 게임의 시작을 지시하는 함수 |
| BeginPlay | 게임 모드의 StartPlay를 통해 모든 액터에서 호출되는 함수 |

|GameState 함수 | 내용 |
| --- | --- |
| HandleBeginPlay | 월드에 있는 모든 액터들에게 BeginPlay를 진행하도록하는 함수 (서버) |
| OnRep_ReplicatedHasBegunPlay | 위와 동일하게 특정 프로퍼티가 변경되면, 클라이언트에게 알려줌 |

<br>

## 마무리

여기까지 기본적인 프레임워크나 클라이언트-서버 모델에 대해서 설명해봤는데요. 내용을 외우기 보다는 엔진 내부의 코드를 살펴보거나 로그를 찍어보는 등의 실습을 해보면서 학습하셨으면 좋겠습니다!