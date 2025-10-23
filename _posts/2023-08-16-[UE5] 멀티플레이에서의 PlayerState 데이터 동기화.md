---
layout: post
title: "[UE5] 멀티플레이에서의 PlayerState 데이터 동기화"
description: 멀티플레이 환경에서 PlayerState 데이터 동기화하는 방법에 대해 소개
date: 2023-08-16 16:00:00 +0900
categories: [UnrealEngine, Analysis]
tags: [UnrealEngine, Mutli, PlayerState, Sync]
---
## PlayerState란?
`PlayerState`는 게임 중에 필요한 '이름, 점수'와 같은 플레이어의 정보를 저장하는 목적으로 사용합니다. 특히 <ins>멀티플레이에서는 해당 정보가 자동으로 동기화</ins>되기 때문에 클라이언트에서 다른 클라이언트의 정보를 참고하는데도 아주 유용하죠. 또 레벨이 이동해도 유지되기 때문에 이름과 같은 정보는 정말 유용하게 사용됩니다.

물론 모든 정보가 레벨 이동 시 유지되지도 않고, 기본 제공하는 정보들에 한해서만 동기화되지만, 단순히 변수를 추가한다고 해서 동기화되는 것은 아닙니다. 어느 정도 **PlayerState**에 대해서 알고 있다는 전제하에 설명을 해보겠습니다. 모르시는 분은 [공식 Docs](https://docs.unrealengine.com/4.27/en-US/API/Runtime/Engine/GameFramework/APlayerState/)를 보고 와주시면 되겠습니다.

<br>

## 새로운 PlayerState

1. 저장하고자하는 데이터 변수 추가 및 동기화 작업
2. 데이터 복제와 초기화 (레벨 이동 시)

위 작업이 끝입니다. 레벨을 이동하지 않는다면 위 작업에서 1번 작업만 진행하면 끝이에요. 근데 만약 레벨을 이동하면서 데이터를 유지하고 싶거나, 특별히 초기화하고 싶은 데이터가 있다면 추가적으로 아래와 같은 작업을 진행해 줘야 합니다.

<br>

### 1. 새로운 데이터 추가

일단 **PlayerState**의 소스코드를 보면 기본적으로 제공하는 변수인 '점수, 아이디, 이름' 등이 저장되어 있고, `Replicate` 관련해서 무언가가 되어 있는 것을 볼 수 있습니다. 

```cpp
// Header
UE_DEPRECATED(4.25, "This member will be made private. Use GetScore or SetScore instead.")
UPROPERTY(ReplicatedUsing=OnRep_Score, Category=PlayerState, BlueprintGetter=GetScore)
float Score;

/** Unique net id number. Actual value varies based on current online subsystem, use it only as a guaranteed unique number per player. */
UE_DEPRECATED(4.25, "This member will be made private. Use GetPlayerId or SetPlayerId instead.")
UPROPERTY(ReplicatedUsing=OnRep_PlayerId, Category=PlayerState, BlueprintGetter=GetPlayerId)
int32 PlayerId;

/** Player name, or blank if none. */
UPROPERTY(ReplicatedUsing = OnRep_PlayerName)
FString PlayerNamePrivate;

UFUNCTION()
ENGINE_API virtual void OnRep_Score();

UFUNCTION()
ENGINE_API virtual void OnRep_PlayerName();

UFUNCTION()
ENGINE_API virtual void OnRep_PlayerId();

// Cpp
void APlayerState::GetLifetimeReplicatedProps(TArray< FLifetimeProperty > & OutLifetimeProps) const
{
	Super::GetLifetimeReplicatedProps(OutLifetimeProps);

	FDoRepLifetimeParams SharedParams;
	SharedParams.bIsPushBased = true;

	DOREPLIFETIME_WITH_PARAMS_FAST(APlayerState, Score, SharedParams);
	DOREPLIFETIME_WITH_PARAMS_FAST(APlayerState, PlayerNamePrivate, SharedParams);

	SharedParams.Condition = COND_SkipOwner;
	DOREPLIFETIME_WITH_PARAMS_FAST(APlayerState, CompressedPing, SharedParams);
	...
}
```

여기서 `OnRep_`함수는 리플리케이션(Replication) 시스템에서 사용되는 콜백 함수로 특정 변수가 서버에서 변경되었을 때, 클라이언트에서 이를 감지하고 해당 로직을 실행할 수 있도록 해주는 역할을 합니다. 즉, 동기화된 이후에 호출되는 것이기 때문에 동기화 자체에 관련해서는 영향을 미치지는 않죠. 그럼 이 변수들 모두 동기화 처리가 되어있다고 생각하면 되겠습니다.

<br>

```cpp
/** Header */
UPROPERTY(ReplicatedUsing=OnRep_IsDead, Category=PlayerState, BlueprintGetter=GetIsDead)
uint8 bIsDead:1;
	
UFUNCTION()
virtual void OnRep_IsDead();

/** Cpp */
void AMyPlayerState::GetLifetimeReplicatedProps(TArray< FLifetimeProperty > & OutLifetimeProps) const
{
	Super::GetLifetimeReplicatedProps(OutLifetimeProps);

	FDoRepLifetimeParams SharedParams;
	SharedParams.bIsPushBased = true;

	DOREPLIFETIME_WITH_PARAMS_FAST(AMyPlayerState, bIsDead, SharedParams);
}
```

그렇다면 저희가 할 일은 위 코드랑 그냥 똑같이 만들어 확장만 해버리면 됩니다. 예를 들어서 캐릭터가 죽은 이후에 상태를 저장하고 싶다면 아래와 같이 `bIsDead` 을 작성하면 되겠죠? 동기화 처리도 해주고요. 이제 이 데이터에 <ins>Getter/Setter</ins> 함수를 만들어서 접근하면 되겠습니다.

<br>

### 2. 데이터 유지 (레벨 이동 시)

위에서 말했듯이 레벨을 이동하면서 데이터를 유지하지 않아도 된다면, 이 작업이 필요 없지만, 만약에 레벨을 이동하면서 이 데이터를 유지하고 싶다면, 진행해야 합니다. 

```cpp
/** Cpp */
void APlayerState::SeamlessTravelTo(APlayerState* NewPlayerState)
{
	DispatchCopyProperties(NewPlayerState);
	NewPlayerState->SetIsOnlyASpectator(IsOnlyASpectator());
}
void APlayerState::DispatchCopyProperties(APlayerState* PlayerState)
{
	CopyProperties(PlayerState);
	ReceiveCopyProperties(PlayerState);
}
void APlayerState::CopyProperties(APlayerState* PlayerState)
{
	PlayerState->SetScore(GetScore());
	PlayerState->SetCompressedPing(GetCompressedPing());
	PlayerState->ExactPing = ExactPing;
	PlayerState->SetPlayerId(GetPlayerId());
	PlayerState->SetUniqueId(GetUniqueId());
	PlayerState->SetPlayerNameInternal(GetPlayerName());
	PlayerState->SetStartTime(GetStartTime());
	PlayerState->SavedNetworkAddress = SavedNetworkAddress;
}
```

이것도 먼저 코드를 살펴봅시다. **SeamlessTravel** 을 하려고하면, `SeamlessTravelTo` 함수가 연쇄적으로 호출되어서 결국에는 `CopyProperties` 함수가 호출되죠. 이는 이전 정보를 현재의 **PlayerState**에 저장하는 것으로 볼 수 있겠습니다.

<br>

그러면 이 또한 어렵지 않겠습니다. 해당 함수를 확장하여 저장하고자 하는 데이터를 복제 해주면 되겠죠.

```cpp
void AMyPlayerState::CopyProperties(APlayerState* PlayerState)
{
	Super::CopyProperties(PlayerState);

	if (AMyPlayerState* NewPlayerState = Cast<AMyPlayerState>(PlayerState))
  {
		NewPlayerState->SetIsDead(GetIsDead());
  }
}
```

어 왜 함수명이 **SeamlessTravel** 이죠? **Non-SeamlessTravel**이면 적용이 안 되는 거 아닌가요? 라고 생각하실 수 있는데, 저도 확인은 안 해봤지만, **Non-SeamlessTravel**이면, 연결을 끊고 진행하니까 애초에 **PlayerState**의 데이터를 지속할 수 없기 때문에 필요 자체가 없을 듯합니다.

그리고 언리얼에서는 공식적으로 재연결 시 이슈나 시각적으로 있어서 안정적인 **SeamlessTravel** 의 사용을 지양하니까 웬만해서는 이거 써주세요.

<br>

### 3. 데이터 초기화

```cpp
void APlayerState::Reset()
{
	Super::Reset();
	SetScore(0);
	ForceNetUpdate();
}
```

이는 반대로 레벨 이동 시 데이터를 초기화하고 싶을 때 사용합니다. 기본 제공하는 데이터에서는 점수를 0으로 초기화하고, 강제로 정보를 업데이트하네요. 킬 수나 점령 횟수 등에 사용하면 좋을 듯하네요.

```cpp
void AMyPlayerState::Reset()
{
	Super::Reset();
	
	SetIsDead(false);
}
```

똑같아요. 끝!

<br>

## 마무리

이렇게 `PlayerState`에서의 <ins>새로운 데이터 추가, 레벨 이동 시 데이터 유지 및 초기화하는 방법</ins>에 대해서 작성해 봤습니다. 데이터 동기화는 어렵지 않았죠? 

위에서 `DOREPLIFETIME_WITH_PARAMS_FAST`을 사용했지만, 설명은 없이 넘어갔는데요. 이는 동기화를 설정하는 매크로 중 하나로 최적화되어 빠른 성능을 제공하는 매크로라고 합니다. 기본적으로는 `DOREPLIFETIME`를 사용해서 모든 클라이언트에 동기화하는데, 조금 느린가 봅니다. 이외에도 특정 조건에 맞는 곳으로만 동기화해주는 `DOREPLIFETIME_CONDITION`를 사용한다면 최적화가 가능하겠죠? 이는 [공식 문서](https://dev.epicgames.com/documentation/ko-kr/unreal-engine/conditional-property-replication?application_version=4.27)에서 확인할 수 있습니다.