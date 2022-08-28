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

멀티플레이어 매치에서 클라이언트로서 캐릭터를 이동할 때 실제로 캐릭터를 직접 이동하지 않고 서버에 캐릭터를 이동시킬 것을 알립니다. 그런 다음 서버는 사용자를 포함한 다른 모든 사용자에게 캐릭터의 위치를 업데이트합니다.
  
_노트 :_ 로컬 클라이언트의 "지연" 느낌을 방지하기 위해 서버가 여전히 캐릭터의 위치를 재정의할 수 있지만, 로컬 클라이언트의 "지연" 느낌을 방지하기 위해 플레이어가 직접 로컬로 캐릭터를 제어하도록 하는 경우가 많습니다. 클라이언트가 부정행위를 시작할 때 서버는 여전히 캐릭터의 위치를 재정의할 수 있습니다! __즉, 클라이언트는 다른 클라이언트와 직접 '대화'하지 않습니다.__

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

게임 모드는 일반적인 모드를 통해 데스매치, 팀 데스매치, 깃발 캡처 등에 사용하며, 아래와 같은 조건들을 선언할 수 있다.

  - 개인전인지 팀전인지를 구분
  - 몇번을 킬해야하는지, 우승 조건은 무엇인지
  - 점수가 오르는 기준. (킬, 점령)
  - 사용이 허용되는 무기들에 대한..

---

> __Example__

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


이 게임모드를 사용하여 일반 매치의 흐름을 관리할 수 있는데, "Ready to start Match"와 같이 재정의를 할 수 있는 기능과도 연결됩니다. 이는 True로 반환될 때 자동으로 호출되지만 수동으로도 사용할 수 있습니다. 이때 "작업이 GameState에서 처리되지 않는다"고 생각할 수 있지만, 게임 모드 기능을 실제로 게임 상태와 함께 작동합니다. 하지만 게임 모드는 서버에만 존재하기 대문에 클라이언트로부터 멀리 떨어진 상태를 관리할 수 있는 포인트를 제공합니다.

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

> __Example__

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

---

> __Another Example__

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

---

> __Example__

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

Pawn의 자식 클래스인 "ACharacter"는 종종 사용되는데, 이는 위치, 회전등을 복제처리하는 네트워크화된 "MovementComponent"라는 구성요소가 존재하기 때문입니다. 항상 말하지만 클라이언트는 캐릭터를 움직이게 하지 않고, 서버가 클라이언트로부터 움직임을 입력받아 움직인후 캐릭터를 복제하는 형식입니다. 즉, Pawn은 대부분 모든 클라이언트들에게 복제됩니다.

---

> __Example__
 
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

---

> __Another Example__

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

> __How to Get PlayerController?__

  <ins>__"GetPlayerController(0)"__ 또는 __"UGameplayStatics::GetPlayerController(GetWorld()), 0;"은__</ins> 서버와 클라이언트에서 다르게 작동하지만, 실제로는 어렵지 않습니다.

  |Mode|Content|
  |:--:|:--|
  |__Listen-Server에서의 호출__|Listen-Server의 PlayerController를 반환|
  |__Client에서의 호출__|Client의 PlayerController를 반환|
  |__Dedicated Server에서의 호출__|첫 번째 클라이어트의 PlayerController를 반환|

> __Example__

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

> __Set Replication__

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

> __Replicating Properties (Replicated)__

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

> __Replicating Properties (RepNotify)__

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

복제(Replication)를 위한 다른 방법을 <span style = "color:orange;">"RPC"</span>라고 합니다. 이는 "Remote Porcedure Calls"(원격 프로시저 호출)의 약자로 다른 경우에 무언가를 호출할 때 사용됩니다. UE4는 이를 사용하여 "클라이언트에서 서버", "서버에서 클라이언트", "서버에서 특정 그룹"으로 함수를 호출할 수 있습니다. 이 RPC는 반환 값을 가질 수 없으며, 반환하기 위해서는 두 번째 RPC를 사용해야 합니다. 

|Rules|Content|
|:--:|:--|
|Run on Server|액터의 서버 인스턴스에서 실행되도록 되어 있습니다.|
|Run on OwningServer|소유자에게 실행되도록 되어 있습니다.|
|NetMulticast|모든 인스턴스에 대해 실행되도록 되어 있습니다.|

<img src="https://raw.githubusercontent.com/Goaway-1/goaway-1.github.io/master/_posts/images/UE5/Network/SetReplicated.png" height="250" title="SetReplicated">

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

> __요구 사항 및 주의 사항__

  "RPC"은 특정한 규칙에서만 작동하며, 아래와 같은 규칙과 상황이 존재합니다.

  1. Actor에서 호출되어야 합니다.
  2. Actor는 복제되어야 합니다.
  3. 클라이언트에서 RPC가 실행되도록 서버에서 호출되는 경우, __해당 Actor를 실제로 소유한 클라이언트만 해당 기능을 실행합니다.__
  4. 서버에서 RPC가 실행되도록 클라이언트에서 호출되는 경우, __클라이언트는 RPC가 호출되는 Actor를 소유해야 합니다.__
  5. 멀티캐스트 RPC는 예외입니다.
      - 서버에서 호출된 경우, 로컬에서 실행할 뿐만 아니라 현재 연결된 모든 클라이언트에서 실행할 수 있습니다.
      - 클라이언트에서 호출된 경우 로컬에서만 실행되며, 서버에서는 실행되지 않습니다. 
      - 우리는 멀티캐스트 이벤트를 위한 간단한 조절 메커니즘을 가지고 있습니다.     
        - 멀티캐스트는 주어진 시간에 두 번 이상 복제하지 않습니디.

---

> __서버에서 호출된 RPC__

  |Actor Onwership|Not Replicated|NetMulticast|Server|Client|
  |:--:|:--:|:--:|:--:|:--:|
  |Client-Owner Actor|서버에서 실행|서버와 모든 클라이언트에서 실행|서버에서 실행|액터를 소유한 클라이언트에서 실행|
  |Server-Owner Actor|서버에서 실행|서버와 모든 클라이언트에서 실행|서버에서 실행|서버에서 실행|
  |Unowned Actor|서버에서 실행|서버와 모든 클라이언트에서 실행|서버에서 실행|서버에서 실행|

---

> __클라이언트에서 호출된 RPC__

  |Actor Onwership|Not Replicated|NetMulticast|Server|Client|
  |:--:|:--:|:--:|:--:|:--:|
  |Owned by invoking Client|호출한 클라이언트에서 실행|호출한 클라이언트에서 실행|서버에서 실행|호출한 클라이언트에서 실행|
  |Owned by different Client|호출한 클라이언트에서 실행|호출한 클라이언트에서 실행|실행 X|호출한 클라이언트에서 실행|
  |Server-Owned Actor|호출한 클라이언트에서 실행|호출한 클라이언트에서 실행|실행 X|호출한 클라이언트에서 실행|
  |Unowned Actor|호출한 클라이언트에서 실행|호출한 클라이언트에서 실행|실행 X|호출한 클라이언트에서 실행|

## Onwership

"Onwership"이란 액터의 소유권을 나타냅니다. 위에서 이미 "Client-owned Actor"와 같은 표를 보았을 것입니다. 서버 또는 클라이언트들은 액터를 소유할 수 있죠. 

예를 들어 "Player Controller"를 소유하는 클라이언트가 있고, 서버가 소유하는 "문"이 월드에 배치되어 있습니다. 이때 클라이언트가 소유하지 않은 액터에서 서버 RPC를 호출할 경우 서버 RPC가 실패하게 됩니다. <ins>__즉 클라이언트는 서버가 소유한 "문"의 "Server_OpenDoor"를 호출할 수 없게 됩니다.__</ins>

<img src="https://raw.githubusercontent.com/Goaway-1/goaway-1.github.io/master/_posts/images/UE5/Network/OwnerShip.png" height="250" title="OwnerShip">

이를 해결하기 위해서 우리는 클라이언트가 실제로 소유하고 있는 클래스/액터를 사용합니다. <ins>__바로 여기서 플레이어 컨트롤러가 빛을 발하기 시작합니다.__</ins> "문"에서 입력을 활성화하고, __서버 RPC를 호출하는 대신 "Player Controller"에서 "서버 RPC"를 만들고 서버가 문에서 "인터페이스 기능"을 호출하도록 합니다.__ "문"의 [인터페이스](https://docs.unrealengine.com/4.27/ko/ProgrammingAndScripting/GameplayArchitecture/Interfaces/)를 구현하면 정의된 로직을 호출하여 Open/Closed 상태를 올바르게 복제합니다. 

### Actors & Their Owning Connections

이미 언급했듯이 "Player Controller"는 실제로 플레이어가 '소유'하는 클래스입니다. 만들어진 "Player Controller"는 해당 Connection에서 소유합니다. 따라서 액터가 누군가의 소유인지 결정할 때, 가장 바깥쪽 소유자에게 질문하고, 이것이 "PlayerController"인 경우 이를 소유한 Connection도 해당 액터를 소유합니다.

__<ins>Pawn/Character는 "PlayerController"에 의해 소유되며, 동시에 "PlayerController"는 소유된 Pawn의 소유자입니다.</ins>__

이 정보는 아래와 같은 경우에 있어 중요한 정보입니다.

  - RPC는 'Run-On-Client RPC'를 실행할 클라이언트를 결정하는데 필요합니다.
  - 액터 __복제(Replication)와 연결 관련성(Relevancy)__
  - Actor 속성 소유자가 관여할 때 복제 조건

RPC가 소유한 연결에 따라 클라이언트/서버가 호출할 때 RPC가 다르게 반응한다는 것을 이미 읽었습니다. 또한 (C++에서) 변수가 특정 조건에서만 복제되는 조건부 복제에 대해 읽었습니다. 다음 항목에서는 목록의 관련성 부분에 대해 설명합니다.

## Actor Relevancy & Priority

### Relevancy(관련성)

  그렇다면 "Relevancy"이란 무엇이며 왜 필요한가요? 플레이어가 실제로 다른 사람들에게 <ins>__'중요하지 않을'__</ins> 수 있을 만큼 충분히 큰 레벨/맵이 있는 게임이 있다고 상상해 보세요. 플레이어 A가 서로 멀리 떨어져 있는 경우 플레이어 B를 볼 필요는 없을 것입니다. 언리얼 네트워크 코드의 <ins>상당한 대역폭 최적화</ins>는, __서버가 클라이언트에게 연관성이 있는 액터 세트에 대해서만 알려주는 것으로 이루어집니다.__

  플레이어에 대한 관련 액터의 세트를 결정하기 위해 다음 규칙을 순서대로 적용합니다. 이러한 테스트는 가상 함수 __"AActor::IsNetRelevantFor()"에서__ 구현됩니다.

  1. 액터가 bAlwaysRelevant (항상 연관성이 있)거나, Pawn 이거나, 폰이 노이즈나 대미지같은 동작의 Instigator (유발자)인 경우, 연관성이 있습니다.
  2. Actor가 'bNetUserOwnerRelevancy'이고 Owner가 있는 경우, Owner의 연관성을 사용합니다.
  3. Actor가 'bOnlyRelevantToOwner'이고 첫 번째 검사를 통과하지 못하는 경우, 연관성이 없습니다.
  4. 액터가 다른 액터의 스켈레톤에 부착된 경우, 관련성은 상위 조상의 연관성에 의해 결정됩니다.
  5. 액터가 숨겨져 있고('bHidden == true') 루트 구성 요소가 충돌하지 않으면, 액터는 연관성이 없습니다.
      - 루트 구성 요소가 없는 경우 'AActor::IsNetRelevantFor()'는 경고를 출력하고 Actor를 'bAlwaysRelevant = true'로 설정해야 하는지 물어봅니다.
  6. 'AGameNetworkManager'가 거리 기반 관련성을 사용하도록 설정된 경우 액터가 "컬 거리(Cull Distance)"보다 가까우면 관련성이 있습니다.

  참고: Pawn 및 PlayerController는 'AActor::IsNetRelevantFor()'를 재정의하고 결과적으로 관련성 조건이 다릅니다.

### Prioritization(우선순위)

  언리얼에서는 모든 액터의 우선순위를 부여하는 __"부하 균형 기법"을__ 사용하여, 각 액터의 게임플레이 중요도에 따라 대역폭을 적절히 분배합니다.

  각 액터에는 NetPriority (우선권)이라는 플로트 변수가 있습니다. <ins>__수치가 큰 액터일수록 다른 액터에 비해 더욱 많은 대역폭을 받게 됩니다.__</ins> 우선권이 2.0 인 액터는 1.0 인 것보다 정확히 두 배의 빈도로 업데이트될 것입니다. 우선권 관련해서 중요한 유일한 것은 그 비율인데, 모든 우선권을 높인다고 언리얼의 네트워크 퍼포먼스가 향상되지는 않습니다. 저희 퍼포먼스 튜닝시 할당해 둔 NetPriority 값은 다음과 같습니다:

  - Actor = 1.0
  - Matinee = 2.7
  - Pawn = 3.0
  - PlayerController = 3.0

  액터의 현재 우선권은 "AActor::GetNetPriority()" 가상 함수를 사용해 계산합니다. 업데이트받지 못하는 경우를 피하기 위해 AActor::GetNetPriority() 는 NetPriority 에 액터의 지난 번 리플리케이션 이후 경과시간을 곱합니다. GetNetPriority 함수 역시 액터와 관찰자 사이의 거리와 상대적 위치를 고려합니다.

  <img src="https://raw.githubusercontent.com/Goaway-1/goaway-1.github.io/master/_posts/images/UE5/Network/Prioritization.png" height="250" title="Prioritization">

  <details><summary><span style = "color:green;">Set Prioritization Code</span></summary> 

  {% highlight cpp %}
  bOnlyRelevantToOwner = false; 
  bAlwaysRelevant = false; 
  bReplicateMovement = true; 
  bNetLoadOnClient = true; 
  bNetUseOwnerRelevancy = false; 
  bReplicates = true;

  NetUpdateFrequency = 100.0f;
  NetCullDistanceSquared = 225000000.0f;
  NetPriority = 1.0f;
  {% endhighlight %}
  </details>
  
  이러한 설정의 대부분은 Blueprint의 클래스 기본값에서 찾을 수 있으며, 물론 각 Actor의 자식 C++ 클래스 내에서 설정할 수 있습니다.

## Actor Role & RemoteRole

액터가 서버와 클라이언트 모두에서 존재하는 경우 Role 값은 일반적으로 서버에서 "ROLE_Authority"이고, 클라이언트에서는 액터가 넷플레이(netplay)에서 플레이할 수 있는 기본적인 3가지 다른 역할에 상응하는 "ROLE_DumbProxy", "ROLE_SimulatedProxy", "ROLE_AutonomousProxy" 중 하나입니다. 클라이언트에서 액터의 Role 은 액터의 기본 속성에서 RemoteRole 의 값으로 지정되는 것이 일반적이지만 게임을 하는 동안 아무 문제 없이 동적으로 값을 자유롭게 변경할 수 있습니다. Role 값은 현 Role 필드에서 설정되고 RemoteRole 은 연결 상대측에는 어떤 Role이 설정되어 있는지를 나타냅니다. 

"Actor Replication"에 대한 두 가지 중요한 속성이 있습니다. 이 두 가지 속성은 다음과 같습니다.

  - Actor에 대한 권한을 가진 사람이 누구인지
  - Actor의 복제 여부가 어떤지
  - 복제 모드가 어떤지

우리가 가장 먼저 확인하고 싶은 것은 누가 특정 Actor에 대한 <ins>권한(Authority)을</ins> 가지고 있는지 입니다. 현재 실행 중인 인스턴스에 Authority가 있는지 확인하려면 __"Role 속성이 ROLE_Authority"인지__ 확인하십시오. 만약 그렇다면, 이 인스턴스는 복제 여부와 상관없이 이 액터를 담당합니다. 이는 소유권과 동일하지 않습니다!

### Role/RemoteRole Reversal

이러한 값을 검사하는 사용자에 따라 "Role"과 "RemoteRole"이 반대로 적용될 수 있습니다. 

예를 들어, <ins>서버</ins>에 다음과 같은 구성이 있는 경우

  - Role == ROLE_Authority
  - RemoteRole = ROLE_SimulatedProxy
  
<ins>클라이언트</ins>는 다음과 같이 볼 수 있습니다.
  
  - Role == ROLE_SimulatedProxy
  - RemoteRole == ROLE_Authority

서버가 행위자를 관리하고 이를 클라이언트에 복제하고, 단지 클라이언트는 업데이트를 수신하고 업데이트 간에 Actor를 시뮬레이션해야 하면됩니다.

### Mode of Replication

서버가 Actor를 매번 업데이트하지는 않습니다. 이렇게 하면 대역폭과 CPU 리소스가 너무 많이 소모되기 때문입니다. 대신, 서버는 'AActor::'의 'NetUpdateFrequency' 속성에 지정된 빈도로 Actors를 복제합니다. 

즉, 행위자 업데이트 사이에 시간이 좀 걸립니다. 이로 인해 Actor들의 움직임이 분산되어 보이거나 흐트러질 수 있습니다. 이를 보완하기 위해 <ins>__클라이언트는 업데이트 간에 Actor를 시뮬레이션합니다.__</ins> 현재 발생하는 시뮬레이션에는 "ROLE_SimulatedProxy" 와 "ROLE_AutonomousProxy" 두 가지 유형이 있습니다.

  - __"ROLE_Authority"는__ 액터를 제어하는 서버 또는 컴퓨터에 사용됩니다.
  - __"ROLE_None"은__ 액터가 전혀 관련되지 않아야 한다는 것을 표시하는데 사용됩니다.
  - __"ROLE_DumbProxy"는__ 클라이언트측 시뮬레이션 또는 행동이 필요없는 액터에 사용됩니다. 실제로는 거의 사용되지 않습니다.
  - __"ROLE_SimulatedProxy"는__ 초기 회전, 위치, 속도 설정에 기초한 클라이언트측 시뮬레이션을 필요로 하는 모든 액터에 사용됩니다. 시뮬레이션된 폰은 Simulated Proxies 얻는 모든 정보 이외에 시간 경과에 따라 이 3가지 변수를 얻습니다.
  - __"ROLE_AutonomousProxy"는__ 소유자와는 별도로 취급될 필요가 있는 클라이언트 제어 액터에 사용됩니다. 이것은 클라이언트에서 액터를 시뮬레이션하는데 Simulated Proxy를 사용하는 게임에서의 다른 플레이어에 대한 클라이언트에서 서버로의 이동 정보를 제공합니다.

---
> __"ROLE_DumbProxy"__

  이 역할은 실제 클라이언트측 예측이 필요없는 간단한 액터에 사용됩니다. 
  
  이런 간단한 액터에는 어떠한 Physics 예측도 필요 없지만 액터의 변수가 복제되기 위해서는 해당 클라이언트에 관련될 필요가 있습니다. 이것은 높은 대역폭 Simulated Proxy(시뮬레이트된 프록시)와 같은 것을 방지하는 지상에 위치한 모든 Inventory 액터에 대해 사용되지만 인벤토리 액터에 필요한 모든 것을 볼 수 있도록 하여줍니다.
  
  이 액터를 수동으로 이동시키려고 하는 경우 클라이언트는 보통 약 매 0.5초마다 정상 연결 상태에서 주기적으로 업데이트를 받습니다. 특정 Physics 유형의 Dumb Proxy를 사용하는 경우 위치가 1초의 몇 분의 1동안마다 업데이트 되어지고, 시뮬레이션 또는 예측이 실행되지 않기에 일정하지 않고 변하는 효과를 만들 수 있습니다. 한 장소의 위치 업데이트이기 때문에 일정하지 않은 주기의 위치 업데이트도 상관 없는, 다시 스폰하기처럼 액터를 레벨 전반에 걸쳐 이동시킬 때 위치 업데이트는 유용하게 사용됩니다. 주기적 변경이 종종 클라이언트로 전송되는 회전 업데이트도 마찬가지입니다. 이 역할은 또한 클라이언트가 구체적으로 bClientAnim 을 통해 전송하지 말라고 명령하지 않은 한 애니메이션 업데이트를 서버에서 클라이언트로 전송되게 유발합니다. 
  
  요약하면 이 역할의 액터는 서버에서 강제로 공급되는 데이터입니다. <ins>이 역할은 움직이지 않는(수송, 재스폰 등과 같은 점프를 제외함), 하지만 여전히 관련되어져야 하는 필요가 있는 액터를 대상으로 합니다. 이 역할 유형은 실제로는 그리 자주 사용되지 않습니다.</ins> 대부분의 경우 일부 코드와 Simulated Proxy를 사용하여 필요한 모든 경우를 처리할 수 있습니다.

---

> __"ROLE_SimulatedProxy"__

  이 역할은 예측을 통행 네트워크상에서 시뮬레이션되는 모든 것에 대해 사용됩니다. 
  
  서버가 특정 Actor에 대한 업데이트를 보낼 때, 클라이언트는 새 위치를 향해 위치를 조정한 다음 업데이트 사이에 클라이언트는 서버에서 보낸 최신 속도에 따라 Actor를 계속 이동시킵니다. __<ins>즉, 서버가 Actor에 대한 업데이트를 보내면, 클라이언트는 내역대로 업데이트한 후 속도에 따라 업데이트합니다.</ins>__

  로켓을 그 예로 들어보겠습니다. 로켓은 항상 직선으로 이동하고 Simulated Proxy에 대한 이상적인 후보입니다. 같은 이유로 게임은 클라이언트측 낙하 물리를 예상할 수 있고, 그렇기에 수류탄과 낙하하는 사람들은 이 역할에 대한 이상적인 후보입니다. 물리 또는 클라이언트측 물리는 반드시 이 역할을 사용해야 하기 때문에 클라이언트측에서는 모든 것이 예상될 수 있습니다. 플레이어가 달리고 있는 경우 계속해서 달릴 것을 예상할 수 있기 때문에 모든 Controllers도 또한 이 역할을 사용하여 시뮬레이션의 이점을 활용할 수 있습니다. 
  
  이것은 예측할 수 없는 플레이어에 사용할 수 있는 최상의 예상 방법입니다. 플레이어에 대한 이 지침의 유일한 예외는 사용자 자신의 동작을 사용자가 시뮬레이션하고 싶지 않기 때문에 자기 자신이 플레이하고 있는 Pawn입니다.

  마지막으로 알려진 속도를 사용하여 시뮬레이션하는 것은 일반적인 시뮬레이션이 작동하는 방법의 한 예시이며, "서버 업데이트 간의 추론을 위한 정보"를 사용하기 위해 사용자 지정 코드를 작성하는 것을 막을 수 있는 방법은 없습니다.

  실제로 "Simulated Proxy" 액터에는 "Simulated Pawn"과 "Simulated Non-Pawn"의 2가지 유형이 있습니다. 이 2가지 유형은 다르지만 유사한 행동 패턴을 가집니다.

---

> __"ROLE_AutonomousProxy"__
  
  이것은 사용자를 위한 것입니다. 온라인 게임을 플레이할 때 자신은 다른 PlayerControllers와는 다르게 취급되야 합니다. __<ins>본인 자신이 서버의 예상대로 예측되어지는 것을 원하지 않을 것입니다. 오히여 본인 자신을 조작하고 서버에게 자신의 진행 위치를 알리고 싶을 것입니다.</ins>__ 이것이 이 역할의 목적입니다. 이 
  
  역할은 클라이언트가 직접 컨트롤하는 사물에 사용됩니다. 대부분의 경우 PlayerControllers에만 의해서 사용됩니다. 엔진이 "Autonomous Proxy" 액터를 볼 때 액터의 상위 소유자(owner->owner->owner...)를 찾습니다. 이것은 "PlayerController"이어야 합니다. 만약에 Top Owner가 현재 복제되어지는 플레이어가 아닌 경우 일시적으로 RemoteRole 을 "ROLE_SimulatedProxy" 로 변경합니다. "UActorChannel::ReplicateActor" 는 Autonomous Proxy 액터가 다른 모든 사람한테 Simulated Proxy 액터로 본인한테는 Autonomous Proxy 액터로 취급되게 하여줍니다. 액터의 "RemoteRole == ROLE_AutonomousProxy" 로 설정되면 액터의 소유자를 제외한 모든 사람에게는 Simulated Proxy로 나타납니다. 이것은 다른 사람과 비교하여 사용자가 차별화되게 행동하도록 하는데 필요한 것입니다.

  일반적으로 "Player Controller"가 소유한 Actor에서만 사용됩니다. 이것은 이 행위자가 사용자로부터 입력을 받고 있다는 것을 의미하기 때문에, 우리가 추론할 때, 우리는 __조금 더 많은 정보를 가지고 있고, (마지막으로 알려진 속도를 기반으로 추정하는 것보다) 누락된 정보를 채우기 위해 실제 사용자의 입력을 사용할 수 있다.__

## [Traveling in Multiplayer](https://docs.unrealengine.com/5.0/ko/travelling-in-multiplayer-in-unreal-engine/)

한 레벨에서 다른 레벨로 맵을 이동하는 것을 __"Travel"이라고__ 한다. 로그인 버튼을 누르면 서버측(GameMode)에서 로그인 순서대로 플레이어에게 순번을 부여하고, 클라이언트 측의 SaveGame이나 GameInstance에 저장했다가 2명이 모두 로그인을 완료하면 서버측(GameMode)에서는 Travel명령을 실행하여 모두 클라이언트가 다른 맵으로 전환하도록 한다.

Travel후에 실행되는 맵의 한 함수에서 "SaveGame/GameInstance"로부터 플레이어의 순번을 참조하여 해당 번호의 캐릭터를 스폰하여 3인칭 맵 위에 표시한다.

Game Mode에서 bUseSeamlessTravel를 true로 하고 Transition map을 만든다.
[Project Settings] > [Project] > [Maps & Modes] > [Advanced] > [Transition map] 에서 만든 맵을 설정하고, 다음과 같은 [과정](https://unrealengine.tistory.com/298)을 참고한다.

### Non-/Seamless travel

> __비/원활한 여행__

"Seamless"와 "Non-Seamless" travel의 사이의 차이는 간단하다. "Seamless travel"은 비차단 호출이고, "Non-Seamless travel"은 차단 호출이다. 

클라이언트에게 "Non-Seamless Travel"은 클라이언트가 서버에서 연결을 끊었다가 동일한 서버에 다시 연결하여, 새 맵을 로드합니다. 때문에 끊김현상이 발생합니다. 

엔진은 "Seamless Trabel"을 자주 사용할 것을 권장합니다. 이름 그대로 원활한 이동이며, "Transition map"이라는 중간에 거쳐가는 맵을 사용한다. 즉, A맵에서 B맵을 로딩할 때 __"A -> Transition map -> B"가__ 된다. 이렇게 하면 더 부드러운 경험을 할 수 있고 재연결 과정에서 발생할 수 있는 문제를 피할 수 있기 때문입니다.

<ins>blocking은 어떤 연산을 실행했을 때, 그 연산이 끝날때까지 기다리고,  non-blocking는 연산해달라고 요청만하고, 원래 실행 흐름을 그대로 실행한다.</ins>

"Non-seamless travel"을 사용하는 세 가지 방법이 있습니다.

  - 맵을 처음 로드할 때
  - 서버에 클라이언트로 처음 접속하는 경우
  - 멀티 플레이 게임을 종료하고 새로운 게임을 시작하려는 경우

### Main Traveling Functions

이동을 위해 쓰이는 함수는 크게 __"UEngine::Browse", "UWorld::ServerTravel", "APlayerController::ClientTravel"__ 세 가지입니다. 

---
> __"UEngine::Browse"__

  - 새로운 맵 로드 시 "하드 리셋"같은 것입니다.
  - 항상 매끄럽지 않은 이동을 합니다.
  - 목적 맵으로 이동하기 전 서버가 현재 클라이언트 접속을 끊습니다.
  - 클라이언트는 현재 서버에서 접속이 끊깁니다.
  - 데디케이티드 서버는 다른 서버로 이동할 수 없으므로, 맵은 반드시 로컬이어야합니다. (URL X)

---
> __"UWorld::ServerTravel"__

  - 서버 전용입니다.
  - 서버를 새 월드/레벨로 점프시킵니다.
  - 접속된 모든 클라이언트도 따라갑니다.
  - 멀티플레이어 게임에서 맵 사이 이동을 할 때 쓰는 방식으로, 서버에서 이 함수 호출을 담당합니다.
  - 서버는 접속된 __모든 클라이언트 플레이어에 대해 APlayerController::ClientTravel 을 호출합니다.__

---
> __"APlayerController::ClientTravel"__
  
  - 클라이언트에서 호출되면, 새 서버로 이동합니다.
  - 서버에서 호출되면, 특정 클라이언트더러 새 맵으로 이동하라 이릅니다. (현재 서버에 접속은 유지)

### Enabling Seamless Travel

매끄러운 이동을 활성화시키려면, __"Transition Map(트랜지션 맵)"을__ 구성해야 합니다. 이는 UGameMapsSettings::TransitionMap프로퍼티를 통해 이루어집니다. 기본적으로 이 프로퍼티는 비어있는데, 게임에서 이 프로퍼티가 비어있으면 트랜지션 맵에 기본 맵이 생성됩니다.

트랜지션 맵의 존재 이유는 __<ins>항상 (맵이 저장된) 월드가 로드되어 있으므로, 새 맵을 로드하기 전에 이전 맵을 해제시킬 수 없기 때문입니다.</ins>__ 맵이 매우 클 수가 있기 때문에, 이전 맵과 새 맵을 동시에 메모리에 두는 것은 좋지 않으니, 트랜지션 맵이 생긴 것입니다.

이제 현재 맵에서 트랜지션 맵으로 이동한 다음, 거기서 최종 맵으로 이동하면 됩니다. 트랜지션 맵은 매우 작으므로, 현재 맵을 최종 맵으로 대체시키는 와중에 그다지 큰 부하가 걸리지 않습니다.

<img src="https://raw.githubusercontent.com/Goaway-1/goaway-1.github.io/master/_posts/images/UE5/Network/Travel.png" height="250" title="Travel">

트랜지션 맵 구성이 완료된 후 "AGameModeBase::bUseSeamlessTravel"을 true으로 설정하면, 매끄러운 이동이 작동합니다.

### Persisting Actors / Seamless Travel

> __Seamless Travel__

매끄러운(Seamless) 이동 실행시의 일반적인 흐름은 이렇습니다.

  1. 트랜지션 레벨에 지속되는 액터를 마킹합니다.
  2. 트랜지션 레벨로 이동합니다.
  3. 최종 레벨에 지속되는 액터를 마킹합니다.
  4. 최종 레벨로 이동합니다.

---

> __Persisting Actors (지속되는 액터들)__

매끄러운 이동 사용시 현재 레벨에서 새 레벨로 (지속) 액터를 가지고 가는 것이 가능합니다. 인벤토리 아이템, 플레이어와 같은 특정 액터에 좋습니다.

기본적으로 자동 지속되는 액터는 다음과 같습니다.

  - GameMode 액터 (서버만)
      - AGameModeBase::GetSeamlessTravelActorList 를 통해 추가된 액터
  - 유효한 PlayerState 가 있는 모든 Controller (서버만)
  - 모든 PlayerController (서버만)
  - 모든 로컬 PlayerController (서버 및 클라이언트)
  - 로컬 PlayerController 에서 호출된   
      - APlayerController::GetSeamlessTravelActorList 를 통해 추가된 액터


## [Online Subsystem Overview](https://docs.unrealengine.com/5.0/ko/online-subsystem-in-unreal-engine/)

> __"Online Subsystem (온라인 서비스)" 기능을 사용하여 서버에 들어가는 것을 의미합니다.__

"Online Subsystem" 및 그 인터페이스는 Steam, Xbox Live, Facebook 등의 __<ins>온라인 서비스 기능을 공통된 방식으로 액세스할 수 있는 방법을 제공합니다.</ins>__ 여러 플랫폼에 출시하거나 여러 온라인 서비스를 지원하는 게임을 만들 때, 온라인 서브시스템을 사용하면 개발자는 각 지원 서비스에 맞게 구성만 조정해 주면 됩니다.

당신은 기본적으로 "SubsystemNULL"을 사용할 것입니다. 이를 통해 LAN 세션을 호스팅하거나 (서버 목록을 통해 세션을 찾고 LAN 네트워크에서 세션에 가입할 수 있음) IP를 통해 직접 가입할 수 있습니다. 그것이 당신에게 허락하지 않는 것은, 인터넷에서 그러한 세션을 진행하는 것입니다. __왜냐하면 클라이언트에게 서버/세션 목록을 제공하는 마스터 서버가 없기 때문입니다.__ 예를 들어, 스팀과 같은 하위 시스템에서는 인터넷을 통해 볼 수 있는 서버/세션도 호스팅할 수 있습니다. 자체 서브시스템/마스터 서버를 만들 수도 있지만, UE4 외부에서 많은 코딩이 필요합니다.

### Online Subsystem Module

---
> __Design Philosophy (디자인 철학)__

  "Online Subsystem"은 근본적으로 다양한 온라인 서비스와 비동기 통신을 처리하도록 설계되었습니다. 네트워크 연결 속도, 서버 지연시간, 백엔드 서비스 실행 시간 모두 로컬 머신이 알 수 없으므로, 이 시스템과의 상호작용에 시간이 얼마나 걸릴지도 예측할 수 없습니다. 
  
  이를 처리하기 위해 온라인 서브시스템은 모든 원격 작업에 __"Delegates(델리게이트)"를__ 사용하여 지원되는 비동기 기능이 사용될 때마다 해당 델리게이트가 호출되도록 합니다. 요청이 완료되면 응답하는 기능을 제공하는 것은 물론, 처리 중인(in-flight) 요청을 쿼리합니다. 또 델리게이트는 준수할 단일 코드 패스를 제공하므로, 개발자가 성공 또는 실패 조건을 잡아내는데 필요한 커스텀 코드를 작성할 필요가 없습니다.

  모듈식 서비스별 인터페이스는 지원 기능끼리 그룹으로 묶습니다. 예를 들어, Friends Interface (친구 인터페이스)는 친구 목록 관련 모든 것을 처리하고, Achievements Interface (업적 인터페이스)는 업적 나열, 검사, 획득 등을 처리합니다. 지원하는 각 온라인 서비스의 각 기능 그룹에 대해 인터페이스가 존재하지만, 서비스가 지원되지 않는 특정 함수에 대해서는 단순히 false 가 반환될 수 있습니다. 이 디자인을 통해 개발자는 모든 온라인 서비스에 대해 동일한 코드를 작성할 수 있습니다.

  하이 레벨에서 보다 복잡한 작업은 Online Asynchronous Task Manager (온라인 비동기 작업 관리자)를 사용하여 순차적 작업이나 다른 스레스에서 실행되는 작업을 지원합니다. 비동기 작업은 종속성을 설명하여, 관련이 없는 작업은 독립적으로 병렬 실행시키고, 순차적 작업은 순서대로 실행시킬 수 있습니다. 온라인 서브시스템 내 모든 인터페이스는 작업 일관성을 유지하기 위해 이러한 방식으로 작업을 예약합니다.

---
> __Use of Delegates__

  <details><summary><span style = "color:green;">Set Delegates Code</span></summary> 

  {% highlight cpp %}
  // Define!!
  DECLARE_DELEGATE(FDele_Single);
	FDele_Single Fuc_DeleSingle;

  // Bind & UnBind & Excute !!
  Function->Fuc_DeleSingle.BindUFunction(this, FName("CallDeleFunc_Single"));
  Fuc_DeleSingle.Unbind();
  Fuc_DeleSingle.Execute();
  {% endhighlight %}
  </details>

  위에서 설명한 것처럼 "Online Subsystem"은  __"Delegates"를__ 많이 사용합니다. "Delegates"를 존중하고 적절한 "Delegates"가 호명될 때까지 기다린 후 기능을 더 아래로 호출하는 것이 중요합니다.
  
  비동기 작업을 기다리지 않으면 충돌이 발생하고 예기치 않은 정의되지 않은 동작이 발생할 수 있습니다. 연결 해제 이벤트와 같은 연결 실패 시 "Delegates"를 기다리는 것이 특히 중요합니다. 작업이 완료되는 데 걸리는 시간은 이상적인 경우 순간적으로 보일 수 있지만, __시간 초과의 경우 거의 1분 이상 걸릴 수 있습니다.__

  "Delegates Interface"는 매우 간단하며, 각 "Delegates"는 각 인터페이스 헤더의 맨 위에 명확하게 정의됩니다. 모든 대리자는 추가, 지우기 및 트리거 기능을 가집니다. (단, 대리인을 수동으로 트리거하는 것은 권장되지 않음) 

  일반적으로 적절한 함수를 호출하기 직전에 "Delegates"를 add()한 다음 내부에서 "Delegates"를 Clear()합니다.

---
> __Basic Design__

  기본 모듈인 OnlineSubsystem 은 엔진에 서비스별 모듈을 정의하고 등록합니다. 초기화 도중 온라인 서브시스템은 Engine.ini 파일에 정의된 기본 온라인 서비스 모듈을 로드 시도합니다. 온라인 서비스로의 모든 액세스는 이 모듈을 통해 이루어집니다.

  {% highlight cpp %}
  [OnlineSubsystem]
  DefaultPlatformService=<Default Platform Identifier>
  {% endhighlight %}

  성공하면, 파라미터가 지정되지 않은 경우 정적 접근자를 통해 기본 온라인 서브시스템을 사용할 수 있습니다.

  {% highlight cpp %}
  static IOnlineSubsystem* Get(const FName& SubsystemName = NAME_None)
  {% endhighlight %}

  이 함수에 대한 호출이 요청하면 부가 서비스를 로드합니다. 잘못된 식별자나 모듈 로드 실패는 관대히 null 을 반환합니다.

---
> __[Interfaces](https://docs.unrealengine.com/5.0/en-US/online-subsystem-in-unreal-engine/)__

  모든 플랫폼이 모든 인터페이스를 구현하지는 않으며, 각 서비스가 지원하는 기능에 따라 다릅니다. 이 인터페이스에는 __"프로필/업적/외부 UI/친구/리더보드/현재상태/구매"등과__ 같은 것들이 존재합니다. 자세한 내용은 위 [UE4 링크](https://docs.unrealengine.com/5.0/en-US/online-subsystem-in-unreal-engine/)를 통해 확인할 수 있습니다.

### Sessions and Matchmaking

## Basic Life-Time of a Session