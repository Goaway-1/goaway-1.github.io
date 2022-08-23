---
layout: post
title: UE5 | Network Compendium
subtitle: Network in Unreal Engine
categories: UE5
tags: [UE5, UnrealEngine, C++]
---
## Introduction

이 포스트는 __"Cedric 'eXi' Neukirchen"이__ 제작하신 'Unreal Engine 4' Network Compendium을 읽고, 학습을 위해 한글로 번역한 것입니다. 저작권 및 각종 권한은 지은이에게 있음을 알립니다.

This post reads the "Unreal Engine 4 Network Compendium" produced by __"Cedric 'eXi' Neukirchen"__ and translates it into Korean for learning. We inform you that the copyright and various rights belong to the author.

## Network in Unreal

언리얼은 보통 <span style = "color:orange;">__"Sever-Client"의__</span> 구조를 사용합니다. 서버는 권위적이며 모든 데이터는 클라이언트에서 서버로 먼저 전송되어야 합니다. 그리고 서버는 데이터를 입증하고, 개발자의 코드에 따라 반응합니다.

__Example :__

  멀티플레이어 매치에서 클라이언트로서 캐릭터를 이동할 때 실제로 캐릭터를 직접 이동하지 않고 서버에 캐릭터를 이동시킬 것을 알립니다. 그런 다음 서버는 사용자를 포함한 다른 모든 사용자에게 캐릭터의 위치를 업데이트합니다.
  
  _노트 :_ 로컬 클라이언트의 "지연" 느낌을 방지하기 위해 서버가 여전히 캐릭터의 위치를 재정의할 수 있지만, 로컬 클라이언트의 "지연" 느낌을 방지하기 위해 플레이어가 직접 로컬로 캐릭터를 제어하도록 하는 경우가 많습니다. 클라이언트가 부정행위를 시작할 때 서버는 여전히 캐릭터의 위치를 재정의할 수 있습니다! __즉, 클라이언트는 다른 클라이언트와 직접 '대화'하지 않습니다.__

__Another Example :__

  대화 메시지를 다른 클라이언트에 발송할 때, 실제로 서버로 발송한 다음 연결하려는 클라이언트에 전달합니다. 팀, 길드, 그룹 등이 될 수도 있습니다.

### 중요한 점

- 클라이언트를 절대 신뢰하지 마십시오! 클라이언트를 신뢰한다는 것은 클라이언트 작업을 실행하기 전에 테스트하지 않는다는 의미입니다. 이것은 부정행위를 허용합니다.
- 서버에서 클라이언트가 실제로 탄약(실행 권한)을 가지고 있고, 직접 샷을 처리하는 대신 다시 샷을 할 수 있는지 테스트해야 합니다!

## Framework & Network

서버-클라이언트 구조에 대한 프레임워크를 아래와 같이 4개의 구역으로 나눌 수 있습니다. 이때 __"Owning Client"란__ 액터를 소유한 플레이어/클라이언트입니다. 당신이 당신의 컴퓨터를 소유한다고 볼 수 있습니다. Ownership(소유권)은 나중에 __"RPCs"에서__ 중요해집니다.

|Architecture|Content|
|:--:|--|
|__Server Only__|개체는 서버에만 존재|
|__Server & Clients__|개체는 서버와 모든 클라이언트들에 존재|
|__Server & Owning Client__|개체는 오직 서버와 소유권이 있는 클라이언트에만 존재|
|__Owning Client Only__|객체는 오직 클라이언트에만 존재|

<img src="https://raw.githubusercontent.com/Goaway-1/goaway-1.github.io/master/_posts/images/UE5/Network/Network_Framework.png" height="350" title="Network_Framework">

위 그림은 네트워크 프레임워크에서 가장 중요한 클래스 중 일부가 배치되는 방법입니다.

<img src="https://raw.githubusercontent.com/Goaway-1/goaway-1.github.io/master/_posts/images/UE5/Network/Network_Framework_Diagram.png" height="350" title="Network_Framework_Diagram">

위 그림은 개체들이 네트워크 프레임워크에서 어떻게 나뉘어 지는지 보여줍니다. 클라이언트1과 클라이언트2 사이에는 개체가 존재하는 않는데, 자신들만이 알고 있는 사물들을 실제로 공유하지 않기 때문입니다.

### Common Classes

다음 페이지 부터는 가장 일반적인 클래스에 대해 설명합니다. 또한 이러한 클래스가 어떻게 사용되는지에 대한 작은 예시를 제공합니다.

#### [Game Mode](https://docs.unrealengine.com/5.0/en-US/API/Runtime/Engine/GameFramework/AGameMode/)

GameMode Class는 GameModeBase와 GameMode로 분할되어 있습니다. 일부 게임에서는 GameMode 클래스의 전체 기능 목록이 필요하지 않을 수 있기 때문에 GameModeBase를 사용하며, 이는 기능이 더 적습니다.

이는 게임의 규칙을 정의하는 데 사용됩니다. 여기에는 Apawn, APlayer Controller, APlayerState 등과 같은 사용된 클래스가 포함되며, 서버에서만 사용할 수 있습니다. 클라이언트는 게임 모드의 개체가 없으며 게임 모드를 검색할 때는 nullptr이 표시됩니다.

__Example :__

  게임 모드는 일반적인 모드를 통해 데스매치, 팀 데스매치, 깃발 캡처 등에 사용하며, 아래와 같은 조건들을 선언할 수 있다.

  - 개인전인지 팀전인지를 구분
  - 몇번을 킬해야하는지, 우승 조건은 무엇인지
  - 점수가 오르는 기준. (킬, 점령)
  - 사용이 허용되는 무기들에 대한..

> Usage Example

<img src="https://raw.githubusercontent.com/Goaway-1/goaway-1.github.io/master/_posts/images/UE5/Network/OverrideFunction.png" height="350" title="OverrideFunction">

멀티플레이에서 게임모드에는 플레이어 및/또는 __일반적인 경기 흐름을 관리하는 데 도움이 되는 몇 가지 기능이__ 있습니다. Blueprint의 "Override Function"을 참고하면 위와 같은 함수들이 존재합니다.

  <details><summary><span style = "color:green;">Event OnPostLogin Code</span></summary> 

  {% highlight cpp %}
  /* GameMode의 파생 클래스 Header */ 
  // List of PlayerControllers
  TArray<class APlayerController*> PlayerControllerList; 

  //  PostLogin함수의 오버라이딩
  virtual void PostLogin(APlayerController* NewPlayer) override;
  {% endhighlight %}

  {% highlight cpp %}
  /* GameMode의 파생 클래스 CPP */
  void ATestGameMode::PostLogin(APlayerController* NewPlayer) { 
    Super::PostLogin(NewPlayer);
    PlayerControllerList.Add(NewPlayer);
  }
  {% endhighlight %}
  
  </details>


이때 게임 플레이 내내 특정 사항에 반응하는 특정 이벤트들도 존재한다. 좋은 예시로는 "Event OnPostLogin"이 존재하는데, 이는 새로운 플레이어가 게임에 참여할 때마다 호출됩니다. 플레이어와 이미 상호 작용하거나, 플레이어를 위해 새 폰을 생성하거나, 나중에 사용할 수 있도록 플레이어 컨트롤러를 배열에 저장하는 데 사용할 수 있습니다.

<img src="https://raw.githubusercontent.com/Goaway-1/goaway-1.github.io/master/_posts/images/UE5/Network/OverrideFunction_Flows.png" height="250" title="OverrideFunction_Flows">

  <details><summary><span style = "color:green;">Ready to start Match Code</span></summary> 


  {% highlight cpp %}
  /* GameMode의 파생 클래스 Header */ 
  // 매치 중에 필요하거나 허용할 최대 사용자의 수
  int32 MaxNumPlayers; 

  // ReadyToStartMatch의 Implementation 오버라이드
  virtual bool ReadyToStartMatch_Implementation() override;
  {% endhighlight %}
  {% highlight cpp %}
  /* GameMode의 파생 클래스 CPP */
  bool ATestGameMode::ReadyToStartMatch_Implementation() { 
    Super::ReadyToStartMatch();
    return MaxNumPlayers == NumPlayers; 
  }
  {% endhighlight %}
  
  </details>


이 게임모드를 사용하여 일반 매치 를로우를 관리할 수 있는데, "Ready to start Match"와 같이 재정의를 할 수 있는 기능과도 연결됩니다. 이는 True로 반환될 때 자동으로 호출되지만 수동으로도 사용할 수 있습니다. 이때 "작업이 GameState에서 처리되지 않는다"고 생각할 수 있지만, 게임 모드 기능을 실제로 게임 상태와 함께 작동합니다. 하지만 게임 모드는 서버에만 존재하기 대문에 클라이언트로부터 멀리 떨어진 상태를 관리할 수 있는 포인트를 제공합니다.

<img src="https://raw.githubusercontent.com/Goaway-1/goaway-1.github.io/master/_posts/images/UE5/Network/GameMode_OptionsString.png" height="120" title="GameMode_OptionsString">

  <details><summary><span style = "color:green;">Options String Code</span></summary> 

  {% highlight cpp %}
  /* GameMode의 파생 클래스 Header */ 
  // 매치 중에 필요하거나 허용할 최대 사용자의 수
  nt32 MaxNumPlayers; 

  // BeginPlay의 오버라이드 : BP 버전을 재생성하기 위해
  virtual void BeginPlay() override;
  {% endhighlight %}
  {% highlight cpp %}
  /* GameMode의 파생 클래스 CPP */ 
  void ATestGameMode::BeginPlay() {
    Super::BeginPlay();
    // 'FCString::Atoi'는 'FString'을 'int32'로 변환하고 static 'ParseOption' 기능을 사용
    // 'UGameplayStatistics' 클래스에서 'OptionsString'에서 올바른 키를 가져옴
    MaxNumPlayers = FCString::Atoi( *(UGameplayStatics::ParseOption(OptionsString, “MaxNumPlayers”)) ); 
  }
  {% endhighlight %}

  </details>


게임 모드에서는 중요한 변수들이 이미 존재하며, 값을 설정할 수 있습니다. __"Default Player Name"은__ 플레이어 상태 클래스를 통해 액세스할 수 있는 기본 플레이어 이름을 제공합니다. __"bDelayed Start"를__ 선택하면 __"Ready to Start Match"가__ 다른 모든 기준을 충족하더라도 게임이 시작되지 않습니다. 더 중요한 변수 중 하나는 소위 <span style = "color:orange;">"Options String"이다.</span> 이러한 옵션은 '?'로 구분되며, 'OpenLevel' 기능을 통해 전달하거나 'ServerTravel'을 콘솔 명령으로 호출할 때 전달할 수 있습니다. 'Parse Option'을 사용하여 'MaxNumPlayers'와 같은 전달된 옵션을 추출할 수 있습니다.

#### [GameState](https://docs.unrealengine.com/5.0/en-US/API/Runtime/Engine/GameFramework/AGameState/)

"GameState"는 __서버와 클라이언트들 사이에서 정보를 공유하는 가장 중요한 클래스이다.__ 이는 게임의 현재 상태를 추적하는데 사용되는데, 연결된 플레이어들의 정보 (APlayerState)가 포함되어 있습니다. __또한 모든 클라이언트들에게 복제되며,__ 모두가 접근할 수 있게하므로써 멀티플레이 게임을 위한 가장 중심적인 클래스가 되었습니다. (서버와 클라이언트 모두)

"GameMode"는 승리하는 데 필요한 킬 수를 알려주는 반면, "GameState"는 각 플레이어 및/또는 팀의 현재 킬 수를 추적합니다. 여기에 어떤 정보를 저장하느냐는 전적으로 사용자에게 달려 있습니다. 점수 배열 또는 그룹 및 길드를 추적하는 데 사용하는 사용자 지정 구조 배열일 수 있습니다.

__Example :__

멀티플레이어에서 "GameState" 클래스는 플레이어와 플레이어의 상태를 포함하여 __게임의 현재 상태를 추적하는 데__ 사용됩니다. 게임 모드는 "GameState"의 Match State 함수가 호출되는지 확인하고 "GameState" 자체가 클라이언트에서도 사용할 수 있는 기회를 제공합니다.

"GameMode"에 비해 "GameState"는 우리에게 많은 것을 주지 않습니다. 그러나 이 방법을 통해 모든 클라이언들에게 정보를 전파하기 위해 시도하는 로직을 만들 수 있습니다.

<img src="https://raw.githubusercontent.com/Goaway-1/goaway-1.github.io/master/_posts/images/UE5/Network/GameState_Valiable.png" height="350" title="GameState_Valiable">

  <details><summary><span style = "color:green;">PlayerArray* Code</span></summary> 

  {% highlight cpp %}
  // PlayerState 클래스 자체 내부
  void APlayerState::PostInitializeComponents() { 
    […]
    UWorld* World = GetWorld();

    // "GameState"에 이 "PlayerState"를 등록
    if(World->GameState != NULL) World->GameState->AddPlayerState(this); 
    […] 
  }
  {% endhighlight %}
  {% highlight cpp %}
  // GameState 클래스 자체 내부:
  void AGameState::PostInitializeComponents() { […]
    for(TActorIterator<APlayerState> It(World); It; ++It) AddPlayerState(*It);
  }
  void AGameState::AddPlayerState(APlayerState* PlayerState) { 
    if(!PlayerState->bIsInactive) PlayerArray.AddUnique(PlayerState);
  }
  {% endhighlight %}

  </details>

위 그림은 "GameState"클래스에서 사용할 수 있는 몇 가지 변수입니다. __"PlayerArray*", "MatchState", "ElapsedTime"들은__ 복제되므로 클라이언트도 접근할 수 있습니다. 이것은 "Authority Game Mode"에는 해당되지 않습니다. "GameMode"는 서버에만 존재하므로 서버만 액세스할 수 있습니다. __"PlayerArray*"은__ 직접 복제되지 않지만 모든 플레이어 상태는 복제되며 생성 시 플레이어 배열에 추가됩니다. 동시에 "GameState"는 생성될 때 한 번 수집합니다.

__Another Example :__

  <img src="https://raw.githubusercontent.com/Goaway-1/goaway-1.github.io/master/_posts/images/UE5/Network/GameState_CustomEvent.png" height="250" title="GameState_CustomEvent">

  <details><summary><span style = "color:green;">Custom Event Code</span></summary> 

  {% highlight cpp %}
  /* GameState 클래스의 Header */
  // Replication의 동작을 위해서는 헤어에 "#include “UnrealNetwork.h”"를 포함해야함
  UPROPERTY(Replicated)
  int32 TeamAScore; 
  UPROPERTY(Replicated)
  int32 TeamBScore;

  // 팀의 점수를 올리는 기능
  void AddScore(bool TeamAScored);
  {% endhighlight %}

  {% highlight cpp %}
  /* GameState 클래스의 CPP */
  void ATestGameState::AddScore(bool TeamAScored) {
    if(TeamAScored) TeamAScore++;
    else TeamBScore++;
  }

  // 이 함수는 UPROPERTY 매크로의 Replicated 지정자를 사용했다면 지정해주어야한다.
  void ATestGameState::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const {
    Super::GetLifetimeReplicatedProps(OutLifetimeProps); DOREPLIFETIME(ATestGameState, TeamAScore);
    DOREPLIFETIME(ATestGameState, TeamBScore); 
  }
  {% endhighlight %}

  </details>

예를 들어 두 팀 'A'와 'B'의 점수를 추적하는 것이 있습니다. 팀이 점수를 획득할 때 호출되는 사용자 "Custom Event"가 있다고 가정해 보겠습니다. 위 그림처럼 Boolean을 통해서 어느 팀이 득점했는지 알 수 있습니다. 나중에 "Replicated" 파트에서 서버만 변수를 복제할 수 있으므로, 해당 서버만 이 이벤트를 호출할 수 있다는 규칙을 읽을 수 있습니다. 다른 클래스(예: 누군가를 죽인 무기)에서 호출되며, 서버(항상!)에서 이 작업이 수행되어야 하므로 RPC는 여기에 필요하지 않습니다. 이러한 변수와 "GameState"가 복제되므로 이 두 변수를 사용할 수 있습니다. 위젯에 표시할 다른 클래스에서 가져올 수 있습니다.

#### [Player State](https://docs.unrealengine.com/5.0/en-US/API/Runtime/Engine/GameFramework/APlayerState/)

"Player State" 클래스는 특별한 플레이어에게 가장 중요한 클래스입니다. 이것은 플레이어의 현재 정보를 가지고 있음을 의미하며, 각 플레이어는 자신의 "Player State"를 가지고 있습니다. 또한 모두에게 복제되고 다른 클라이언트의 데이터를 검색하여 표시할 수 있습니다. 모든 "Player state"에 접근하는 쉬운 방법은 "GameState"에 있는 "PlayerArray"에 접근하는 것입니다.

__Example :__

  |Example Information|Content|
  |:--:|--|
  |__PlayerName__|현재 연결된 플레이어의 이름|
  |__Score__|현재 연결된 플레이어의 점수|
  |__Ping__|현재 연결된 플레이어의 핑|
  |__GuildID__|플레이어가 속한 길드 ID|
  |__etc__|과 같이 다른 플레이어에서 얻어야하는 Replicated된 정보..|

  <img src="https://raw.githubusercontent.com/Goaway-1/goaway-1.github.io/master/_posts/images/UE5/Network/PlayerState.png" height="250" title="PlayerState">

  위의 표는 PlayerState를 구성하는 변수들의 예시입니다. 이 변수들은 모두 복제되기 때문에, 모든 클라이언트에서 동기화된 상태로 유지됩니다. 만약 PlayerName을 수정하려고 한다면, "GameMode"의 "ChangeName"을 호출하여 플레이어의 컨트롤러를 전달하여 수정합니다. 

  <img src="https://raw.githubusercontent.com/Goaway-1/goaway-1.github.io/master/_posts/images/UE5/Network/PlayerState_Copy.png" height="200" title="PlayerState_Copy">

  <details><summary><span style = "color:green;">CopyProperties/OverrideWith Event Code</span></summary> 
  {% highlight cpp %}
  /* Header file of our PlayerState Child Class inside of the Class declaration */ 
  // Used to copy properties from the current PlayerState to the passed one
  virtual void CopyProperties(class APlayerState* PlayerState);
  
  // Used to override the current PlayerState with the properties of the passed one 
  virtual void OverrideWith(class APlayerState* PlayerState)
  {% endhighlight %}

  {% highlight cpp %}
  /* CPP file of our PlayerState Child Class */
  void ATestPlayerState::CopyProperties(class APlayerState* PlayerState) { 
    Super::CopyProperties(PlayerState);
    if(PlayerState) {
      ATestPlayerState* TestPlayerState = Cast<ATestPlayerState>(PlayerState); 
      if(TestPlayerState) TestPlayerState->SomeVariable = SomeVariable;
    } 
  }
  void ATestPlayerState::OverrideWith(class APlayerState* PlayerState) { 
    Super::OverrideWith(PlayerState);
    if(PlayerState) {
      ATestPlayerState* TestPlayerState = Cast<ATestPlayerState>(PlayerState); if(TestPlayerState)
      SomeVariable = TestPlayerState->SomeVariable;
    } 
  }
  {% endhighlight %}
  </details>

  또한 "PlayerState"는 __Level 변경, 예기치 않은 연경 문제 발생 시 데이터가 지속되도록하는데 사용됩니다.__ 즉, 플레이어를 다시 연결하거나 서버와 함께 새 맵으로 이동하는 기능이 기능이 있습니다. 또한 이미 보유하고 있는 정보를 새 플레이어 상태로 복사하는 작업을 수행합니다. Level 변경하거나 플레이어가 다시 연결될때 생성됩니다.

#### [Pawn](https://docs.unrealengine.com/5.0/en-US/API/Runtime/Engine/GameFramework/APawn/)

"Pawn" 클래스는 플레이어가 실제로 조작하는 액터입니다. 주로 인간형 캐릭터에 사용되지만, 고양이, 책, 비행기, 배와 같은 것도 가능합니다. 플레이어는 한번에 하나의 "Pawn"를 소유할 수 있지만, "Pawn"를 소유하거나 재소유하는 등 쉽게 변경할 수 있습니다.

__Pawn은 대부분 모든 클라이언트들에게 복제 :__

  Pawn의 자식 클래스인 "ACharacter"는 종종 사용되는데, 이는 위치, 회전등을 복제처리하는 네트워크화된 "MovementComponent"라는 구성요소가 존재하기 때문입니다. 항상 말하지만 클라이언트는 캐릭터를 움직이게 하지 않고, 서버가 클라이언트로부터 움직임을 입력받아 움직인후 캐릭터를 복제하는 형식입니다.

__Example :__
 
  멀티플레이에서 우리는 주로 "Pawn"가 캐릭터를 화면에 표시하고, 다른이와 몇가지 정보를 공유하기 위해 복제된 부분을 사용합니다. 간단한 예가 바로 캐릭터의 "Health"입니다. 하지만 우리는 오직 '체력'을 다른 사람에게만 보이게 하기 위해 복제할뿐만 아니라, 서버가 그것에 대한 권한을 가지고 클라이언트가 치트를 쓸 수 없도록 복제합니다.

  <img src="https://raw.githubusercontent.com/Goaway-1/goaway-1.github.io/master/_posts/images/UE5/Network/32p.png" height="200" title="32p">

  <details><summary><span style = "color:green;">Un/Possessed Event Code</span></summary> 
 
  {% highlight cpp %}
  virtual void PossessedBy(AController* NewController); 
  
  virtual void UnPossessed();
  {% endhighlight %}

  {% highlight cpp %}
  // Header file of our Pawn Child Class, inside of the Class declaration 
  // The 'UnPossessed' Event does not pass the old PlayerController though.
  // SkeletalMesh Component, so we have something to hide
  class USkeletalMeshComponent* SkeletalMesh; 
  
  // Overriding the UnPossessed Event
  virtual void UnPossessed() override;
  {% endhighlight %}

  - "MulticastRPCFunction"가 필요한 경우 아래와 같이 만들 수 있다.

  {% highlight cpp %}
  /* Header file of our Pawn Child Class, inside of the Class declaration */ 
  UFUNCTION(NetMulticast, unreliable)
  void Multicast_HideMesh();
  {% endhighlight %}

  {% highlight cpp %}
  /* CPP file of our Pawn Child Class */ 
  void ATestPawn::UnPossessed() {
    Super::UnPossessed(); Multicast_HideMesh();
  }
  
  // You will read later about RPC's and why that '_Implementation' is a thing 
  void ATestPawn::Multicast_HideMesh_Implementation() {
    SkeletalMesh->SetVisibility(false); 
  }
  {% endhighlight %}
  </details>

  표준 재정의 함수임에도 불구하고 "Pawn"은 "Possessed"와 "Unpossessed" 2가지 기능을 가지고 있습니다. 그것은 "Pawn"이 "UnPossessed"되지 않았을때, 숨기기 위해서 사용할 수 있습니다. (소유가 서버에서 발생하기 때문에, 이러한 이벤트는 서버에서만 호출됩니다.)

__Another Example :__

  <img src="https://raw.githubusercontent.com/Goaway-1/goaway-1.github.io/master/_posts/images/UE5/Network/34p.png" height="150" title="34p">

  <details><summary><span style = "color:green;">Health Event Code</span></summary> 

  {% highlight cpp %}
  /* Header file of our Pawn Child Class, inside of the Class declaration */ 
  // Replicated Health Variable
  UPROPERTY(Replicated)
  int32 Health;
  
  // Overriding the Damage Event
  virtual float TakeDamage(float Damage, struct FDamageEvent const& DamageEvent, AController* EventInstigator, AActor* DamageCauser) override;
  {% endhighlight %}

  {% highlight cpp %}
  /* CPP file of our Pawn Child Class */
  // This function is required through the Replicated specifier in the UPROPERTY Macro and is declared by it 
  void ATestPawn::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const {
    Super::GetLifetimeReplicatedProps(OutLifetimeProps); // This actually takes care of replicating the Variable
    DOREPLIFETIME(ATestPawn, Health); 
  }
  float ATestPawn::TakeDamage(float Damage, struct FDamageEvent const& DamageEvent, AController* EventInstigator, AActor* DamageCauser) {
    float ActualDamage = Super::TakeDamage(Damage, DamageEvent, EventInstigator, DamageCauser);
    Health -= ActualDamage;             // Lower the Health of the Player 
    if(Health <= 0.0) return ActualDamage;          // And destroy it if the Health is less or equal 0 Destroy();
  }
  {% endhighlight %}
  </details>

  두 번째 예시로는 "Health"의 값에 대한 것입니다. 사진을 보면 "EventAnyDamage"와 복제된 "체력"의 값이 플레이어의 체력을 낮추는 방법을 볼 수 있습니다. 이는 서버에서만 호출되며 클라이언트에서는 발생하지 않습니다.

  서버에서 호출할 경우 "Pawn"을 복제해야 하므로 'DestroyActor' 노드가 액터의 클라이언트 버전도 삭제합니다. 클라이언트에서 HUD 또는 모든 사람의 머리 위에 있는 헬스 바에 대해 복제된 "Health" 변수를 사용할 수 있습니다. 진행률 표시줄과 폰의 참조가 있는 위젯을 만들어 쉽게 이 작업을 수행할 수 있습니다.

  <img src="https://raw.githubusercontent.com/Goaway-1/goaway-1.github.io/master/_posts/images/UE5/Network/35_1p.png" height="170" title="35_1p">
  
  "TestPawn" 클래스에 "Health" 및 'MaxHealth' 변수가 있으며 모두 복제되도록 설정되어 있다고 가정해 보겠습니다. 이제 위젯 내부에 "TestPawn" 참조 변수와 진행률 표시줄을 만든 후 해당 막대의 백분율을 다음 함수에 바인딩할 수 있습니다.

  <img src="https://raw.githubusercontent.com/Goaway-1/goaway-1.github.io/master/_posts/images/UE5/Network/35_2p.png" height="170" title="35_2p">

  이제 위젯의 구성 요소를 설정한 후 ,BeginPlay에서 "Widget class to use"를 "HealthBar"로 설정할 수 있습니다.

  <img src="https://raw.githubusercontent.com/Goaway-1/goaway-1.github.io/master/_posts/images/UE5/Network/36p.png" height="200" title="36p">

  "BeginPlay"는 서버와 모든 클라이언트에서 의미하는 "Pawn"의 모든 인스턴스에서 호출됩니다. 그래서 모든 인스턴스가 스스로를 위젯의 "Pawn"의 참조로 설정됩니다. 

#### [Player Controller](https://docs.unrealengine.com/5.0/en-US/API/Runtime/Engine/GameFramework/APlayerController)

"APlayerController"클래스는 우리가 접하는 클래스 중 아마 가장 흥미있을 것이고 복잡한 클래스일 것입니다. 이 <span style = "color:orange;">__클래스는 클라이언트가 실제로 소유하는__</span> 첫 번째 클래스이기 때문에, 많은 클라이언트 관련 작업을 위한 센터입니다. 

"Player Controller"는 플레이어의 '입력'으로 볼 수 있습니다. 플레이어와 서버의 연결입니다. 즉, 모든 클라이언트에는 하나의 플레이어 컨트롤러가 있습니다. <ins>클라이언트의 "Player Controller"는 자신과 서버에만 존재하지만, 다른 클라이언트는 다른 "Player Controller"에 대해 알지 못합니다.</ins>

그 결과 서버는 모든 클라이언트의 "Player Controller"에 대한 참조를 가지게 됩니다. 이때 모든 입력(버튼 누름, 마우스 이동, 컨트롤러 축 등)을 "Player Controller"에 배치할 필요는 없습니다. 입력을 각 Character/Pawn 클래스에 배치하고, "Player Controller"를 사용하여 호출하는 것이 좋습니다.

__How to Get PlayerController?__

  <ins>__"GetPlayerController(0)"__ 또는 __"UGameplayStatics::GetPlayerController(GetWorld()), 0;"은__</ins> 서버와 클라이언트에서 다르게 작동하지만, 실제로는 어렵지 않습니다.

  |Mode|Content|
  |:--:|:--|
  |__Listen-Server에서의 호출__|Listen-Server의 PlayerController를 반환|
  |__Client에서의 호출__|Client의 PlayerController를 반환|
  |__Dedicated Server에서의 호출__|첫 번째 클라이어트의 PlayerController를 반환|

__Example :__

  "Player Controller"는 네트워킹에 대한 가장 중요한 클래스 중 하나이지만 기본적으로는 많이 존재하지 않습니다. 그래서 우리는 왜 그것이 필요한지 명확히 하기 위해 작은 예를 만들 것입니다. 소유권(Ownership)에 대한 장에서는 "Player Controller"가 RPC에 중요한 이유에 대해 설명합니다. 다음 예제에서는 "Player Controller"를 사용하여 위젯 단추를 눌러 게임 상태에서 복제된 변수를 늘리는 방법을 보여 줍니다.

  먼저 "Player Controller"가 필요한 이유에 대해서 설명하고자 합니다. 
  
  <ins>위젯은 Client/Listen-Server에만 존재하며, 클라이언트가 위젯을 소유하더라도 서버의 RPC에는 실행할 인스턴스가 존재하지 않습니다.</ins> 그것은 단순하게 복제되지 않습니다. 즉, 위젯은 클라이언트에만 존재하며, 이를 서버에서 실행할 수 없습니다. 그렇기 때문에 우리는 버튼을 눌렀을때 서버로 넘겨서 컨트롤할 수 있는 방법이 필요합니다. 이 방법이 바로 서버 소유인 GameState의 RPC를 사용하는 방법입니다. <span style = "color:orange;">"Player Controller"의 Server함수에서 GameState의 일반 함수를 호출하면 됩니다. </span>

---

  <img src="https://raw.githubusercontent.com/Goaway-1/goaway-1.github.io/master/_posts/images/UE5/Network/GameState_IncreaseVariable.png" height="200" title="GameState_IncreaseVariable">
  
  <details><summary><span style = "color:green;">GameState Event Code</span></summary> 

  {% highlight cpp %}
  /* CPP file of our GameState Child Class */
  // This function is required through the Replicated specifier in the UPROPERTY Macro and is declared by it 
  void ATestGameState::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const {
    Super::GetLifetimeReplicatedProps(OutLifetimeProps); 
    
    // This actually takes care of replicating the Variable
    DOREPLIFETIME(ATestGameState, OurVariable);
  }
  void ATestGameState::IncreaseVariable() {
    OurVariable++; 
  }
  {% endhighlight %}

  {% highlight cpp %}
  /* Header file of our GameState Child Class, inside of the Class declaration */ 
  // Replicated Integer Variable
  UPROPERTY(Replicated) 
    int32 OurVariable;
  public:
    // Function to Increment the Variable 
    void IncreaseVariable();
  {% endhighlight %}
  </details>

  버튼을 눌렀을때 Our Variable이 증가하는 함수입니다. 이 이벤트는 로컬에서 실행되는 함수입니다. (로컬 전용)

  <img src="https://raw.githubusercontent.com/Goaway-1/goaway-1.github.io/master/_posts/images/UE5/Network/PlayerController_Server_IncreaseVariable.png" height="200" title="PlayerController_Server_IncreaseVariable">

  <details><summary><span style = "color:green;">PlayerController Event Code</span></summary> 

  {% highlight cpp %}
  /* CPP file of our PlayerController Child Class */
  // Otherwise we can't access the GameState functions 
  #include “TestGameState.h”

  // You will read later about RPC's and why that '_Validate' is a thing 
  bool ATestPlayerController::Server_IncreaseVariable_Validate() {
    return true; 
  }

  // You will read later about RPC's and why that '_Implementation' is a thing 
  void ATestPlayerController::Server_IncreaseVariable_Implementation() {
    ATestGameState* GameState = Cast<ATestGameState>(UGameplayStatics::GetGameState(GetWorld())); 
    GameState->IncreaseVariable();
  }

  void ATestPlayerController::BeginPlay() { 
    Super::BeginPlay();

    // Make sure only the Client Version of this PlayerController calls the ServerRPC 
    if(Role < ROLE_Authority) Server_IncreaseVariable();
  }
  {% endhighlight %}

  {% highlight cpp %}
  /* Header file of our PlayerController Child Class, inside of the Class declaration */
  // Server RPC. You will read more about this in the RPC Chapter 
  UFUNCTION(Server, unreliable, WithValidation)
  void Server_IncreaseVariable();

  // Also overriding the BeginPlay function for this example 
  virtual void BeginPlay() override
  {% endhighlight %}
  </details>

  "GameState"를 통해서 복제된 정수 변수를 증가시키는 일반 함수(Increase Variable)를 가져옵니다. 이 이벤트는 "Player Controller"의 서버 RPC 내부에 있는 서버 측에서 호출됩니다. (서버 전용)

  <img src="https://raw.githubusercontent.com/Goaway-1/goaway-1.github.io/master/_posts/images/UE5/Network/Character_Button.png" height="150" title="Character_Button">

  버튼(클라이언트 측)을 클릭하면, "Player Controller"의 ServerRPC를 사용하여 서버 측으로 이동한 다음 게임 상태의 'Increase Variable' 이벤트를 호출하여 복제된 정수를 증가시킵니다. 이 정수는 서버에 의해 복제되고 설정되므로 이제 GameState의 모든 인스턴스에서 업데이트되며 클라이언트도 업데이트를 볼 수 있습니다.

#### [HUD](https://docs.unrealengine.com/5.0/en-US/API/Runtime/Engine/GameFramework/AHUD/)

"AHUD"클래스는 각 클라이언트에서만 사용할 수 있는 클래스이며 플레이어 컨트롤러를 통해 액세스할 수 있고, 자동으로 생성됩니다. UMG(Unreal Motion Graphics)가 출시되기 전에 HUD 클래스는 클라이언트의 뷰포트에서 텍스트, 텍스처 등을 그리는 데 사용되었습니다. "HUD"클래스를 사용하여 디버깅하거나 분리된 영역을 사용하여 위젯의 생성/표시/숨기기 및 삭제를 관리할 수 있습니다. "HUD"는 네트워크에 직접 연결되지 않기 때문에 싱글 플레이어만 영향을 미칩니다.

#### [Widgets (UMG)](https://docs.unrealengine.com/5.0/en-US/API/Runtime/UMG/)

"Widgets"은 "Unreal Motion Graphics"라고 불리는 에픽게임즈의 새로운 UI 시스템입니다. 이는 C++ 내에서 UI를 만드는 데 사용되며 ["Slate"](https://docs.unrealengine.com/4.27/ko/ProgrammingAndScripting/Slate)로부터 상속됩니다. 위젯은 클라이언트(수신 서버)에서만 로컬로 사용할 수 있습니다. 또한 복제되지 않으며, 버튼 누름과 같이 복제된 작업을 수행하려면 항상 별도의 복제된 클래스가 필요합니다. 이에 대한 예시는 Pawn에서 이미 진행하였습니다.

## Dedicated vs Listen Server

### Dedicated Server

"Dedicated Server"는 클라이언트를 필요로 하지 않는 독립 실행형 서버입니다. 게임 클라이언트와 별도로 실행되며 플레이어가 항상 참여/탈퇴할 수 있는 서버를 실행하는 데 주로 사용됩니다. 또한 시각적 부분이 없으므로 UI도 필요하지 않으며 플레이어 컨트롤러도 없습니다. 그들은 또한 게임에서 그들을 대표하는 캐릭터나 캐릭터와 유사한 것들을 가지고 있지 않습니다. Windows 및 Linux용으로 컴파일할 수 있으며, 고정 IP 주소를 통해 플레이어가 연결할 수 있는 가상 서버에서 실행할 수 있습니다. 

즉, 서버 그 자체를 의미합니다.

### Listen Server

"Listen Server"는 클라이언트이기도 한 서버입니다. 즉, 서버는 항상 적어도 하나의 클라이언트를 연결합니다. 이 클라이언트는 "Listen Server"라고 하며 이 연결이 끊어지면 서버가 종료됩니다. 클라이언트이기 때문에 "Listen Server"에는 UI가 필요하며 클라이언트 부분을 나타내는 "PlayerController"가 있습니다. 이때 "PlayerController(0)"를 가져오면 바로 해당 클라이언트의 "PlayerController"가 반환됩니다. "Listen Server"는 클라이언트 자체에서 실행되므로 다른 사용자가 연결해야 하는 IP는 클라이언트의 IP입니다. 전용 서버와 비교했을 때, 이것은 종종 고정 IP를 가지고 있지 않은 인터넷 사용자의 문제와 함께 발생합니다. 그러나 "Online Subsystem"을 사용하면 변화하는 IP 문제를 해결할 수 있습니다.

즉, 플레이어가 서버가 되는 것을 의미합니다.

## [Replication](https://docs.unrealengine.com/4.27/ko/InteractiveExperiences/Networking/Actors/Properties/)

"Replication"(복제)는 서버가 정보/데이터를 클라이언트에 전달하는 작업입니다. 이것은 특정 개념 및 그룹으로 제한될 수 있습니다. Blueprint는 대부분 영향을 받는 "Actor"의 설정에 따라 복제를 수행합니다. 속성을 복제할 수 있는 첫 번째 클래스는 "Actor"입니다. 앞서 언급한 모든 클래스는 "Actor"에서 상속되므로 필요한 경우 속성을 복제할 수 있습니다. 물론 "Actor"를 상속받지 않는 "GameMode"와 같이 전혀 복제되지 않고 서버에만 존재하는 경우도 있습니다.

---

__Set Replication__

  <img src="https://raw.githubusercontent.com/Goaway-1/goaway-1.github.io/master/_posts/images/UE5/Network/Replication.png" height="250" title="Replication">

  <details><summary><span style = "color:green;">Set Replication Code</span></summary> 

  {% highlight cpp %}
  ATestCharacter::ATestCharacter() {
    // Network game, so let's setup Replication 
    bReplicates = true;
    bReplicateMovement = true; 
  }
  {% endhighlight %}
  </details>

  "Replication"는 Actor 하위의 Class Defaults/Constructor에서 활성화할 수 있습니다. "bReplicates"가 TRUE로 설정된 "Actor"는 <span style = "color:yellow;"><ins>서버에 의해 생성될 때 모든 클라이언트에서 생성되고 복제됩니다.</ins></span> 클라이언트가 이 액터를 생성하면 액터는 바로 이 클라이언트에만 존재합니다.

---

__Replicating Properties (Replicated)__

  <img src="https://raw.githubusercontent.com/Goaway-1/goaway-1.github.io/master/_posts/images/UE5/Network/Replicated.png" height="300" title="Replicated">
  
  <details><summary><span style = "color:green;">Set Replicated Code</span></summary> 

  {% highlight cpp %}
  /* Header file inside of the Classes declaration */ 
  // Create replicated health variable 
  UPROPERTY(Replicated)
    float Health;
  {% endhighlight %}
  {% highlight cpp %}
  void ATestPlayerCharacter::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const { 
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    // Here we list the variables we want to replicate + a condition if wanted 
    DOREPLIFETIME(ATestPlayerCharacter, Health);

    // Replicates the Variable only to the Owner of this Object/Class
    DOREPLIFETIME_CONDITION(ATestPlayerCharacter, Health, COND_OwnerOnly);
  }
  {% endhighlight %}
  </details>

  "Replication"를 사용하도록 설정하면 변수를 복제할 수 있습니다. 변수의 "Replication"를 "Replicated"로 설정하고, 이 변수가 변경되면 그 변수 값이 클라이언트에게 공유됩니다. 물론 이것은 복제하도록 설정된 배우에게만 작동합니다. 전체 "Replication" 프로세스는 서버에서 클라이언트로만 작동하며, 그 반대는 작동하지 않습니다. 나중에 서버가 클라이언트가 다른 사용자와 공유하려는 내용을 복제하도록 하는 방법에 대해 알아보겠습니다

  이때 "DOREPLIFETIME_CONDITION"은 조건이 있는 복제 기준입니다. 이에 대한 규정은 아래 표를 확인해주세요.

  |Condition|Description|
  |:--|:--|
  |COND_InitialOnly|초기에만 전송을 시도합니다.|
  |COND_OwnerOnly|Actor의 소유자에게만 전송됩니다.|
  |COND_SkipOwner|소유자를 제외한 모든 연결에 전송됩니다.|
  |COND_SimulatedOnly|시뮬레이션된 Actor에게만 전송됩니다.|
  |COND_AutonomousOnly|자율 Actor에게만 전송됩니다.|
  |COND_SimulatedOrPhysics|시뮬레이션되는 또는 bRepPhysics 액터에 전송합니다.|
  |COND_InitialOrOwner|초기 패킷시, 또는 액터의 오너에 전송합니다.|
  |COND_Custom|별다른 조건이 없지만, SetCustomIsActiveOverride 를 통해 껐다 켰다 토글 기능을 원합니다.|

---

__Replicating Properties (RepNotify)__

  <img src="https://raw.githubusercontent.com/Goaway-1/goaway-1.github.io/master/_posts/images/UE5/Network/RepNotify.png" height="300" title="RepNotify">

  <details><summary><span style = "color:green;">Set RepNotify Code</span></summary> 

  {% highlight cpp %}
  /* Header file inside of the Classes declaration */ 
  // Create RepNotify Health variable 
  UPROPERTY(ReplicatedUsing=OnRep_Health) 
  float Health;

  // Create OnRep Function | UFUNCTION() Macro is important! | Doesn't need to be virtual though 
  UFUNCTION()
  virtual void OnRep_Health();
  {% endhighlight %}
  {% highlight cpp %}
  /* CPP file of the Class */
  void ATestCharacter::OnRep_Health() { 
    if(Health < 0.0f) PlayDeathAnimation(); 
  }
  {% endhighlight %}
  </details>

  "Replication"의 다른 방법은 "Replicated"대신 "RepNotify"을 사용하는 것입니다. 이는 업데이트된 값을 수신할 때, 모든 인스턴스에서 자동으로 함수를 호출하도록한다. 이렇게 하면 값이 복제된 후에 호출해야 하는 로직을 자동으로 호출할 수 있습니다.

## [Remote Procedure Calls](https://ko.wikipedia.org/wiki/%EC%9B%90%EA%B2%A9_%ED%94%84%EB%A1%9C%EC%8B%9C%EC%A0%80_%ED%98%B8%EC%B6%9C)

복제(Replication)를 위한 다른 방법을 <span style = "color:orange;">"RPC"</span>라고 합니다. 이는 "Remote Porcedure Calls"(원격 프로시저 호출)의 약자로 다른 경우에 무언가를 호출할 때 사용됩니다. UE4는 이를 사용하여 "클라이언트에서 서버", "서버에서 클라이언트", "서버에서 특정 그룹"으로 함수를 호출합니다. 이 RPC는 반환 값을 가질 수 없으며, 반환하기 위해서는 두 번째 RPC를 사용해야 합니다. 

|Rules|Content|
|:--:|:--|
|Run on Server|액터의 서버 인스턴스에서 실행되도록 되어 있습니다.|
|Run on OwningServer|소유자에게 실행되도록 되어 있습니다.|
|NetMulticast|모든 인스턴스에 대해 실행되도록 되어 있습니다.|

<img src="https://raw.githubusercontent.com/Goaway-1/goaway-1.github.io/master/_posts/images/UE5/Network/SetReplicated.png" height="300" title="SetReplicated">

<details><summary><span style = "color:green;">Set RPC Code</span></summary> 

{% highlight cpp %}
// This is the actual implementation (Not Server_PlaceBomb). But when calling it, we use "Server_PlaceBomb" 
void ATestPlayerCharacter::Server_PlaceBomb_Implementation() {
  // BOOM!
}
bool ATestPlayerCharacter::Server_PlaceBomb_Validate() {
  return true; 
}
{% endhighlight %}
{% highlight cpp %}
// This is a Server RPC, marked as unreliable and WithValidation (is needed!)
UFUNCTION(Server, unreliable, WithValidation)
void Server_PlaceBomb()
{% endhighlight %}
</details>

기본적으로 RPC는 비신뢰성이기 때문에 "Reliable"를 추가하여 원격 머신에서 확실히 실행되게 만들 수 있습니다. 하지만 그렇다고 모든 RPC를 신뢰할 수 있는 것으로 만들면 안됩니다.

<details><summary><span style = "color:green;">Set Validation Code</span></summary> 

{% highlight cpp %}
UFUNCTION(Server, unreliable, WithValidation) 
void SomeRPCFunction(int32 AddHealth);
{% endhighlight %}
{% highlight cpp %}
bool ATestPlayerCharacter::SomeRPCFunction_Validate(int32 AddHealth) { 
  if(AddHealth > MAX_ADD_HEALTH) {
    return false;                                // This will disconnect the caller! 
  }
  return true;                                   // This will allow the RPC to be called! 
}
{% endhighlight %}
</details>

"Validation"의 개념은, RPC에 대한 검증 함수가 매개 변수 중 하나라도 잘못된 것을 감지할 경우, RPC 호출을 시작한 클라이언트/서버의 연결을 끊도록 시스템에 알릴 수 있다는 것입니다. 현재 모든 ServerRPC 함수에 대해 유효성 검사가 필요한 경우 UFUNCTION 매크로의 'With Validation' 키워드를 사용합니다.

---

__요구 사항 및 주의 사항__
  하지만 "RPC"은 특정한 규칙에서만 작동하며, 아래와 같은 규칙과 상황이 존재합니다.

  1. ACtor에서 호출되어야 합니다.
  2. Actor는 복제되어야 합니다.
  3. 클라이언트에서 RPC가 실행되도록 서버에서 호출되는 경우, 해당 Actor를 실제로 소유한 클라이언트만 해당 기능을 실행합니다.
  4. 서버에서 RPC가 실행되도록 클라이언트에서 호출되는 경우, 클라이언트는 RPC가 호출되는 Actor를 소유해야 합니다.
  5. 멀티캐스트 RPC는 예외입니다.
      - 서버에서 호출된 경우, 서버는 로컬에서 실행할 뿐만 아니라 모든 서버에서 실행할 수 있습니다. 현재 연결된 클라이언트입니다.
      - 클라이언트에서 호출된 경우 로컬에서만 실행되며 서버에서 실행되지 않습니다. 
      - 현재, 우리는 멀티캐스트 이벤트를 위한 간단한 조절 메커니즘을 가지고 있다.     
        - 멀티캐스트 기능은 주어진 시간에 두 번 이상 복제하지 않는다.
        - 배우의 네트워크 업데이트 기간입니다. 장기적으로, 우리는 이것에 대해 개선하기를 기대한다.

---

__서버에서 호출된 RPC__

  |Actor Onwership|Not Replicated|NetMulticast|Server|Client|
  |:--:|:--:|:--:|:--:|:--:|
  |Client-Owner Actor|서버에서 실행|서버와 모든 클라이언트에서 실행|서버에서 실행|액터를 소유한 클라이언트에서 실행|
  |Server-Owner Actor|서버에서 실행|서버와 모든 클라이언트에서 실행|서버에서 실행|서버에서 실행|
  |Unowned Actor|서버에서 실행|서버와 모든 클라이언트에서 실행|서버에서 실행|서버에서 실행|

---

__클라이언트에서 호출된 RPC__
  |Actor Onwership|Not Replicated|NetMulticast|Server|Client|
  |:--:|:--:|:--:|:--:|:--:|
  |Owned by invoking Client|호출한 클라이언트에서 실행|호출한 클라이언트에서 실행|서버에서 실행|호출한 클라이언트에서 실행|
  |Owned by different Client|호출한 클라이언트에서 실행|호출한 클라이언트에서 실행|실행 X|호출한 클라이언트에서 실행|
  |Server-Owned Actor|호출한 클라이언트에서 실행|호출한 클라이언트에서 실행|실행 X|호출한 클라이언트에서 실행|
  |Unowned Actor|호출한 클라이언트에서 실행|호출한 클라이언트에서 실행|실행 X|호출한 클라이언트에서 실행|

## Onwership