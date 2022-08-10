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

언리얼은 보통 <span style = "color:orange;">__"Sever-Client"의__</span> 구조를 사용한다. 서버는 권위적이며 모든 데이터는 클라이언트에서 서버로 먼저 전송되어야 한다. 그리고 서버는 데이터를 입증하고, 개발자의 코드에 따라 반응한다.

__Example :__

  멀티플레이어 매치에서 클라이언트로서 캐릭터를 이동할 때 실제로 캐릭터를 직접 이동하지 않고 서버에 캐릭터를 이동시킬 것을 알립니다. 그런 다음 서버는 사용자를 포함한 다른 모든 사용자에게 캐릭터의 위치를 업데이트합니다.
  
  _노트 :_ 로컬 클라이언트의 "지연" 느낌을 방지하기 위해 서버가 여전히 캐릭터의 위치를 재정의할 수 있지만, 로컬 클라이언트의 "지연" 느낌을 방지하기 위해 플레이어가 직접 로컬로 캐릭터를 제어하도록 하는 경우가 많습니다. 클라이언트가 부정행위를 시작할 때 서버는 여전히 캐릭터의 위치를 재정의할 수 있습니다! __즉, 클라이언트는 다른 클라이언트와 직접 '대화'하지 않습니다.__

__Another Example :__

  대화 메시지를 다른 클라이언트에 발송할 때, 실제로 서버로 발송한 다음 연결하려는 클라이언트에 전달합니다. 팀, 길드, 그룹 등이 될 수도 있습니다.

### 중요한 점

- 클라이언트를 절대 신뢰하지 마십시오! 클라이언트를 신뢰한다는 것은 클라이언트 작업을 실행하기 전에 테스트하지 않는다는 의미입니다. 이것은 부정행위를 허용합니다.
- 서버에서 클라이언트가 실제로 탄약(실행 권한)을 가지고 있고, 직접 샷을 처리하는 대신 다시 샷을 할 수 있는지 테스트해야 합니다!

## Framework & Network

서버-클라이언트 구조에 대한 프레임워크를 아래와 같이 4개의 구역으로 나눌 수 있습니다. 이때 __"Owning Client"란__ 액터를 소유한 플레이어/클라이언트입니다. 당신이 당신의 컴퓨터를 소유하는 한다고 볼 수 있습니다. Ownership(소유권)은 나중에 __"RPCs"에__ 중요해집니다.

|Architecture|Content|
|:--:|--|
|__Server Only__|개체는 서버에만 존재|
|__Server & Clients__|개체는 서버와 모든 클라이언트들에 존재|
|__Server & Owning Client__|개체는 오직 서버와 소유권이 있는 클라이언트에만 존재|
|__Owning Client Only__|객체는 오직 클라이언트에만 존재|

<img src="https://raw.githubusercontent.com/Goaway-1/goaway-1.github.io/master/_posts/images/UE5/Network/Network_Framework.png" height="350" title="Network_Framework">

위 그림은 네트워크 프레임워크에서 가장 중요한 클래스 중 일부가 배치되는 방법이다.

<img src="https://raw.githubusercontent.com/Goaway-1/goaway-1.github.io/master/_posts/images/UE5/Network/Network_Framework_Diagram.png" height="350" title="Network_Framework_Diagram">

위 그림은 개체들이 네트워크 프레임워크에서 어떻게 나뉘어 지는지 보여준다. 클라이언트1과 클라이언트2 사이에는 개체가 존재하는 않는데, 자신들만이 알고 있는 사물들을 실제로 공유하지 않기 때문이다.

### Common Classes

다음 페이지 부터는 가장 일반적인 클래스에 대해 설명합니다. 또한 이러한 클래스가 어떻게 사용되는지에 대한 작은 예시를 제공합니다.

#### Game Mode

GameMode Class는 GameModeBase와 GameMode로 분할되었습니다. 일부 게임에서는 GameMode 클래스의 전체 기능 목록이 필요하지 않을 수 있기 때문에 GameModeBase를 사용하며, 이는 기능이 더 적습니다.

이는 게임의 규칙을 정의하는 데 사용됩니다. 여기에는 Apawn, APlayer Controller, APlayerState 등과 같은 사용된 클래스가 포함되며, 서버에서만 사용할 수 있습니다. 클라이언트는 게임 모드의 개체가 없으며 게임 모드를 검색할 때는 nullptr이 표시됩니다.

__Example :__

  게임 모드는 일반적인 모드를 통해 데스매치, 팀 데스매치, 깃발 캡처 등에 사용하며, 아래와 같은 조건들을 선언할 수 있다.

  - 개인전인지 팀전인지를 구분
  - 몇번을 킬해야하는지, 우승 조건은 무엇인지
  - 점수가 오르는 기준. (킬, 점령)
  - 사용이 허용되는 무기들에 대한..

> Usage Example

<img src="https://raw.githubusercontent.com/Goaway-1/goaway-1.github.io/master/_posts/images/UE5/Network/OverrideFunction.png" height="350" title="OverrideFunction">

멀티플레이어에서 게임모드에는 플레이어 및/또는 일반적인 경기 흐름을 관리하는 데 도움이 되는 몇 가지 흥미로운 기능도 있습니다. Blueprint의 "Override Function"을 참고하면 위와 같은 함수들이 존재합니다.

  <details><summary><span style = "color:green;">Event OnPostLogin Code</span></summary> 

  ```cpp
  /* Header file of our GameMode Child Class inside of the Class declaration */ 
  // List of PlayerControllers
  TArray<class APlayerController*> PlayerControllerList; 

  // Overriding the PostLogin function
  virtual void PostLogin(APlayerController* NewPlayer) override;
  ```
  ```cpp
  /* CPP file of our GameMode Child Class */
  void ATestGameMode::PostLogin(APlayerController* NewPlayer) { 
    Super::PostLogin(NewPlayer);
    PlayerControllerList.Add(NewPlayer);
  }
  ```
  </details>

이때 게임 플레이 내내 특정 사항에 반응하는 특정 이벤트들도 존재한다. 좋은 예시로는 "Event OnPostLogin"이 존재하는데, 이는 새로운 플레이어가 게임에 참여할 때마다 호출됩니다. 플레이어와 이미 상호 작용하거나, 플레이어를 위해 새 폰을 생성하거나, 나중에 사용할 수 있도록 플레이어 컨트롤러를 배열에 저장하는 데 사용할 수 있습니다.

<img src="https://raw.githubusercontent.com/Goaway-1/goaway-1.github.io/master/_posts/images/UE5/Network/OverrideFunction_Flows.png" height="350" title="OverrideFunction_Flows">

  <details><summary><span style = "color:green;">Ready to start Match Code</span></summary> 

  ```cpp
  /* Header file of our GameMode Child Class inside of the Class declaration */ 
  // Maximum Number of Players needed/allowed during this Match
  int32 MaxNumPlayers; 

  // Override Implementation of ReadyToStartMatch 
  virtual bool ReadyToStartMatch_Implementation() override;
  ```
  ```cpp
  /* CPP file of our GameMode Child Class */
  bool ATestGameMode::ReadyToStartMatch_Implementation() { Super::ReadyToStartMatch();
  return MaxNumPlayers == NumPlayers; }
  ```
  </details>

이 게임모드를 사용하여 일반 매치 를로우를 관리할 수 있는데, "Ready to start Match"와 같이 재정의를 할 수 있는 기능과도 연결됩니다. 이는 True로 반환될 때 자동으로 호출되지만 수동으로도 사용할 수 있습니다. 이때 "작업이 GameState에서 처리되지 않는다"고 생각할 수 있지만, 게임 모드 기능을 실제로 게임 상태와 함께 작동합니다. 하지만 게임 모드는 서버에만 존재하기 대문에 클라이언트로부터 멀리 떨어진 상태를 관리할 수 있는 포인트를 제공합니다.


<img src="https://raw.githubusercontent.com/Goaway-1/goaway-1.github.io/master/_posts/images/UE5/Network/GameMode_OptionsString.png" height="150" title="GameMode_OptionsString">

  <details><summary><span style = "color:green;">Options String Code</span></summary> 

  ```cpp
  /* Header file of our GameMode Child Class inside of the Class declaration */ 
  // Maximum Number of Players needed/allowed during this Match
  nt32 MaxNumPlayers; 

  // Override BeginPlay, since we need that to recreate the BP version 
  virtual void BeginPlay() override;
  ```
  ```cpp
  /* CPP file of our GameMode Child Class */ 
  void ATestGameMode::BeginPlay() {
    Super::BeginPlay();
    // 'FCString::Atoi' converts 'FString' to 'int32' and we use the static 'ParseOption' function of the 
    // 'UGameplayStatics' Class to get the correct Key from the 'OptionsString'
    MaxNumPlayers = FCString::Atoi( *(UGameplayStatics::ParseOption(OptionsString, “MaxNumPlayers”)) ); 
  }
  ```
  </details>

게임 모드에서는 중요한 변수들이 이미 존재하며, 값을 설정할 수 있습니다. __"Default Player Name"은__ 플레이어 상태 클래스를 통해 액세스할 수 있는 기본 플레이어 이름을 제공합니다. __"bDelayed Start"를__ 선택하면 __"Ready to Start Match"가__ 다른 모든 기준을 충족하더라도 게임이 시작되지 않습니다. 더 중요한 변수 중 하나는 소위 <span style = "color:orange;">"Options String"이다.</span> 이러한 옵션은 '?'로 구분되며, 'OpenLevel' 기능을 통해 전달하거나 'ServerTravel'을 콘솔 명령으로 호출할 때 전달할 수 있습니다. 'Parse Option'을 사용하여 'MaxNumPlayers'와 같은 전달된 옵션을 추출할 수 있습니다.

#### Game State