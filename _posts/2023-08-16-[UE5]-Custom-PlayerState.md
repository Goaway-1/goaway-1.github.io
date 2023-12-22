---
layout: post
title: Custom PlayerState
subtitle: PlayerState
categories: UE5
tags: [UE5, UnrealEngine]
---
## PlayerState란?

이 PlayerState라는 놈은 게임 중에 필요한 이름, 점수‘와 같은 플레이어의 정보를 저장하는 목적으로 사용합니다. 특히 멀티플레이에서는 해당 정보가 자동으로 동기화되기 때문에 클라이언트에서 다른 클라이언트의 정보를 참고하는데도 아주 유용하죠. 또 레벨이 이동해도 유지되기 때문에 이름과 같은 정보는 정말로! 유용하게 사용됩니다.

물론 모든 정보가 레벨 이동 시 유지되지도 않고, 기본 제공하는 정보들에 한해서만 동기화되지만, 문제 없습니다. 제가 이제부터 알려줄꺼니까요. 일단 어느 정도 ‘***PlayerState***’에 대해서 알고 있다는 전제하에 설명을 해보겠습니다. 모르시는 분은 [공식 Docs](https://docs.unrealengine.com/4.27/en-US/API/Runtime/Engine/GameFramework/APlayerState/)나 [다른 블로그](https://goaway-1.github.io/ue5/2022/08/10/UE5-Network-Compendium.html#h-framework--network)들 보고 와주세요.

## CustomPlayerState를 만들어보자!

1. 저장하고자하는 데이터 변수 추가 및 동기화 작업
2. 데이터 복제와 초기화 (레벨 이동 시)

... 끝입니다. 레벨을 이동하지 않는다면 위 작업에서 1번 작업만 진행하면 끝이에요. 근데 만약 레벨을 이동하면서 데이터를 유지하고 싶거나, 특별히 초기화하고 싶은 데이터가 있다면 추가적으로 아래와 같은 작업을 진행해줘야 합니다. 

### 새로운 데이터 추가 방법

일단 ‘***PlayerState***’의 소스코드를 보면 기본적으로 제공하는 변수인 “***점수, 아이디, 이름***”등이 저장되어 있고, ‘***Replicate***’ 처리되어 있는 것을 볼 수 있습니다. 이 변수들 모두 동기화 처리가 되어있다고 생각하면 되겠네요.

```cpp
/** Header */
private:
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

/** Cpp */
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

그렇다면 저희가 할 일은 위 코드랑 그냥 똑같이 만들어 확장만 해버리면 됩니다. 예를들어서 캐릭터가 죽은 이후에 상태를 저장하고 싶다면 아래와 같이 ‘***bIsDead***’을 작성하면 되겠죠? 동기화처리도 해주고요. 이제 이 데이터에 Getter/Setter 함수를 만들어서 접근하면 되겠습니다.

```cpp
/** Header */
private:
	UPROPERTY(ReplicatedUsing=OnRep_IsDead, Category=PlayerState, BlueprintGetter=GetIsDead)
	uint8 bIsDead:1;

/** Cpp */
void AMyPlayerState::GetLifetimeReplicatedProps(TArray< FLifetimeProperty > & OutLifetimeProps) const
{
	Super::GetLifetimeReplicatedProps(OutLifetimeProps);

	FDoRepLifetimeParams SharedParams;
	SharedParams.bIsPushBased = true;

	DOREPLIFETIME_WITH_PARAMS_FAST(AMyPlayerState, bIsDead, SharedParams);
}
```

### 데이터 유지

위에서 말했듯이 레벨을 이동하면서 데이터를 유지하지 않아도 된다면, 이 작업이 필요없어요! 근데 만약에 레벨을 이동하면서 하… 이 데이터를 유지하고 싶은데…! 라는 생각이 든다면, 진행시켜야 합니다. 

이것도 먼저 코드를 살펴봅시다. ‘***SeamlessTravel***’을 하려고하면, ‘***SeamlessTravelTo***’ 함수가 연쇄적으로 호출되서 결국에는 ‘***CopyProperties***’ 함수가 호출되죠. 이는 이전 정보를 현재의 ‘***PlayerState***’에 저장하는 것으로 볼 수 있겠습니다.

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

그러면 이 또한 어렵지 않겠습니다. 해당 함수를 확장하여 저장하고자 하는 데이터를 복제해주면 되겟죠.

```cpp
void AMyPlayerState::CopyProperties(APlayerState* PlayerState)
{
	Super::CopyProperties(PlayerState);

	if (AMyPlayerState* NewPlayerState = Cast<AMyPlayerState>(PlayerState))
  {
		NewPlayerState ->SetIsDead(GetIsDead());
  }
}
```

어 왜 함수명이 ‘***SeamlessTravel***’이죠? ‘***Non-SeamlessTravel***’이면 적용이 안되는거 아닌가요? 라고 물어볼 수 있는데, 저도 확인은 안해봤지만, ***Non-SeamlessTravel***’이면, 연결을 끊고 진행하니까 아마 ‘***PlayerState***’의 데이터를 지속하기에는 무리가 아닐까? 생각해봅니다. (아닐 수도 있어요 ㅎ_ㅎ)

그리고 언리얼에서는 공식적으로 재연결 시 이슈나 시각적으로 있어서 안정적인 ‘***SeamlessTravel***’의 사용을 지양하니까 웬만해서는 이거 써주세요.

### 데이터 초기화

이는 반대로 레벨 이동 시 데이터를 초기화하고 싶을 때 사용합니다. 기본 제공하는 데이터에서는 점수를 0으로 초기화하고, 강제로 정보를 업데이트하네요. 킬 수나 점령 횟수등에 사용하면 좋을 듯하네요.

```cpp
void APlayerState::Reset()
{
	Super::Reset();
	SetScore(0);
	ForceNetUpdate();
}
```

똑같아요. 끝!

```cpp
void AMyPlayerState::Reset()
{
	Super::Reset();
	
	SetIsDead(false);
}
```

참고로 저는 Score가 초기화는지 모르고, ‘***CopyProperties***’에서 Score 복제하는데 왜 자꾸 초기화될까..? 라는 생각에 다른 코드를 많이 건들였지 뭡니까? 이럴꺼면 왜 복제하는거야 엔진놈들…

## 마무리

네. 이렇게 'PlayerState’에서의 ‘새로운 데이터 추가’, ‘레벨 이동 시 데이터 유지 및 초기화’하는 방법에 대해서 작성해봤습니다. 어떻게 보면 저의 지식을 공유한 첫 번째 게시글이 아닌가 싶네요. 많은 분들에게 도움이 되기를 바라며, 즐거운 개발되세요!