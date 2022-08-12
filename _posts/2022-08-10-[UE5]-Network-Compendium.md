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

#### Game Mode

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

#### GameState

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

#### Player State

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

#### Pawn
