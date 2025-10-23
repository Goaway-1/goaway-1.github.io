---
layout: post
title: "[UE5] 언리얼 네트워크 기초 3부 - Server Mode"
description: Unreal Engine Network Server Mode
date: 2024-02-07 06:00:00 +0900
categories: [Unreal Engine, Analysis]
tags: [Unreal Engine, Network, Server Mode, Dedicated, Listen]
---
## Dedicated Server VS Listen Server

`Dedicated Server`와 `Listen Server`에 대해서 부가적으로 설명드리겠습니다. Dedicated는 자체 네트워크가 필요하다고 말씀을 드렸죠? 그렇기 때문에 대부분의 초보 개발자분들은 게임의 규모가 작기도 하고, 상대적으로 편리한 리슨 서버를 사용하여 게임을 개발하고 계실 거라고 생각합니다. 물론 저도 그랬고요. 사실은 서버가 따로 필요하다는 말에 지레 겁먹고 안 한 이유가 큽니다. 

근데 알고 보면, <ins>`Dedicated Server`로 개발하는 것이 훨씬 많은 장점</ins>을 가지고 있습니다.

<br>

### Dedicated Server의 장점

1. **코드의 역할 분리**

    프로그래머의 입장에서 가장 큰 장점은 <ins>"코드의 역할 분리"</ins>인 것 같습니다. 서버에서는 물리적인 충돌과 같은 작업이, 클라이언트에서는 이를 복제하여 플레이어에게 그럴듯하게 보여주는 게 주요 포인트인데요. 만약 `Listen Server` 일 때, 호스팅 한 클라이언트는 네트워크와 동시에 비주얼적인 부분도 담당해야 하는 것입니다. 원래는 클라이언트에서만 해도 되는 건데 말이죠. 그렇기 때문에 코드를 작성할 때, 고려할 사항이 늘어나게 되죠.

    애니메이션은 클라이언트에서만 실행되면 됩니다. 하지만 `Listen Server`는 서버 겸 클라이언트가 있기 때문에 서버에서도 애니메이션이 실행돼야 하는 것이죠. 이렇게 되면 동기화도 해야 되기 때문에 작성해야 할 코드가 늘어나겠죠?

2. **네트워크 성능 향상**

    두 번째로는 네트워크의 품질이 향상한다는 것입니다. 뭐 실제로 네트워크의 품질이 향상되는 것은 아니지만, 생각해 봅시다. 클라이언트가 서버의 역할도 한다면..? 이미 클라이언트의 시각적인 부분을 위해서 CPU, GPU 다 돌리고 있는데, 여기서 서버도 돌린다고 하면 PC가 얼마나 힘들어할까요? 실질적으로는 게임의 속도로 느려지고, 혼자의 문제가 아니라 접속한 모든 클라이언트 또한 버벅거리는 현상을 보게 될 겁니다.

3. **디버깅**

    디버깅이 분리된다는 게 아주 좋습니다. 특히 중단점 걸고 디버깅할 때 이만한 게 없더라고요. 만약 `Listen Server` 면 중단점 걸릴 때마다 이게 서버에서 발생한 건지 클라이언트에서 발생한 건지 매우 헷갈리게 됩니다. 개발은 디버깅이 정말 중요한 것 같아요. 진짜.

<br>

## Dedicated Server를 켜보자

장점에 대해서 장황하게 설명했으니 실제로 키는 법까지 알려드려야겠죠? 프로젝트를 Dedicated Server로 켜 클라이언트들을 접속시키고 디버깅까지 해봅시다. `Dedicated Server` 이하 `DS`라고 하겠습니다.

<br>

### 1. 기초 설정

DS를 켜기 위해서는 GitHub에 있는 EpicGames의 UnrealEngine 레포지토리에서 현재 프로젝트 버전에 맞는 [UnrealEngine-release](https://docs.unrealengine.com/5.1/en-US/downloading-unreal-engine-source-code/)를 받아야 합니다. 그리고는 압축을 풀고 해당 폴더 내의 `Setup.bat, GenerateProjectFiles.bat`을 순서대로 실행시켜 줍니다. 맥북이라면 `command` 확장자를 리눅스라면 'sh' 확장자로 된 파일을 열어 주세요.

<br>

![※ 기존 프로젝트 확장 방법](/assets/img/post/Network-3/ProjectExpansion.png){: width="550" height="418"}
*빌드 방법*

이제 `UE5.sin` 파일이 생성될 텐데요. 열어서 `Solution Configurations`을 `Development Editor`로 지정한 후에 빌드해 주세요. (엄청나게 오래 걸리고 용량도 큽니다.) 빌드가 완료되었다면, `Debug/Start New Instance`를 통해서 기존 프로젝트를 열어주는데, 새로운 카피버전으로 열어줍니다. 같은 엔진이지만, 개발자로서의 기능이 보다 확장된 엔진으로 열기 때문에 카피버전으로 연다고 생각하시면 됩니다. 이제 다 되었으면 꺼도 상관없습니다.

<br>

![※ Source 폴더](/assets/img/post/Network-3/SourceFolder.png){: width="507" height="127"}
*생성된 빌드 설정 파일*
다음으로 복제된 프로젝트의 `Source` 폴더로 가면 `<프로젝트명>.Target.cs`와 `<프로젝트명>Editor.Target.cs` 파일이 있을텐데요. 이는 타겟이나 빌드할 때의 설정을 위해서 필요합니다. 서버와 클라이언트로 구성된, DS에서 동작할 수 있도록 `<프로젝트명>Server.Target.cs`와 `<프로젝트명>Client.Target.cs`파일을 만들어주세요.

<br>

![※ 소스코드 수정](/assets/img/post/Network-3/EditCode.png){: width="1419" height="387"}
*소스 폴더*
그 후에는 해당 파일의 소스코드를 수정하여 위 그림과 같이 타깃을 지정해주어야 합니다. 지정해 주고 다시 sin파일을 만들게 되면, `Solution Configurations`에서 서버, 클라이언트로 빌드를 지정할 수게 됩니다. 이제 서버와 클라이언트로 각각 빌드를 해주면, `Binaries` 폴더 내부에 실행할 수 있는 파일들이 생성되었을 겁니다.

<br>

### 2. 쿠킹

DS에서 바로 접속할 레벨을 설정해 주시고, `Server, Client` 빌드 타깃 모두 쿠킹 해주세요. 요건 쉽죠?

<br>

### 3. 실행 방법

실행 방법은 다양한데, cmd를 사용하는 방법과 바로가기를 만드는 2가지 방법을 설명해 드리겠습니다.

![※ CMD](/assets/img/post/Network-3/CMD.png){: width="992" height="146"}
*cmd*

CMD를 사용하는 방법입니다. 빌드된 파일들 뒤에 커맨드를 입력하기만 하면 끝입니다. 위 명령어는 데디케이티드 서버와 로그를 찍는 것이고, 하단은 해당 아이피(로컬)에 접근하는 클라이언트 되시겠습니다. 모르는 사람이 지나가면서 본다면 간지를 어필할 수 있다는 장점이 있지만, 저는 치는 거 귀찮아서 이 방식은 사용하지 않습니다. 

<br>

![※ 바로가기](/assets/img/post/Network-3/Shortcut.png){: width="504" height="358"}
*바로가기로 만들기*

저는 보통 바로가기 만들어두고 사용합니다. 바로가기에 원하는 커맨드를 입력해 둔 후에 그냥 더블클릭 해주면 간편하게 사용할 수 있습니다. 이전 깃허브에서 받은 언리얼의 <ins>'Engine\Binaries\Win64' 폴더에 있는 'UnrealEditor.exe'</ins>를 바탕화면에 바로가기로 만들어주고, 뒤에 추가로 본인 프로젝트의 경로와 원하는 커맨드를 입력해 줍니다. 서버를 실행하는 커맨드는 `-Server`, 클라는 `-Game`, 로그를 띄우는 것은 `-log`입니다. 이외에도 많은 커맨드는 [공식 홈페이지](https://docs.unrealengine.com/4.26/en-US/ProductionPipelines/CommandLineArguments/)에서 확인할 수 있습니다.

저는 DS서버를 열고 로그를 찍어보기 위해서 <ins>"깃허브 언리얼 경로\UnrealEditor.exe" "내 프로젝트경로\프로젝트명.uproject" -log -server</ins>와 같이 입력해 주었습니다. 이제 각각 실행시켜 주면 완료입니다!

- 서버 커맨드 : -log -server
- 클라 커맨드 : 127.0.0.1:7777 -windowed -resX=800 -resY=450 -game -log

<br>

## Dedicated Server Debugging

![SetDebugging.png](/assets/img/post/Network-3/SetDebugging.png){: width="628" height="441"}
*Debugging*

그렇다면 디버깅은 어찌할까요? 사용하는 에디터인 Visual Studio를 사용해야겠죠. 프로젝트를 마우스로 우측클릭한 뒤 `Properties/Command`에 빌드파일의 위치를 올려주면 됩니다. 뒤에 커맨드도 추가해도 되고요. 이렇게 하고 디버깅을 시작하면, 서버도 실행되고 디버깅도 할 수 있습니다.

단점이 있다면, 디버깅하려는 플랫폼마다 비쥬얼 스튜디오를 켜고 설정을 바꿔주어야 합니다. 어디 실행파일 딸깍 누르면 비쥬얼 켜주는 커맨드 없을까요?

<br>

## 자잘한 팁

### 게임 모드

![Network_Framework_Diagram](/assets/img/post/Network-3/Network_Framework_Diagram.png){: width="532" height="441"}
*Network Framework Diagram*

게임 모드는 서버에만 존재하기에 클라이언트에 복제가 불가능합니다. 만약 게임 모드의 정보를 가지고 오고 싶다면 어찌해야할까요? 게임 모드가 아닌, 게임 스테이트에서 필요한 정보를 저장하고 `NetMulticast`를 통해서 클라이언트로 전송되어 갱신해주면 되겠죠?

```c++
//header of GameStateBase
UFUNCTION(NetMulticast, Reliable)
void A();

//cpp of GameStateBase
void A_Implementation() {~~~~}
```

> 추가로 플레이어 컨트롤러는 서버/클라이언트 모두에 존재합니다.
{: .prompt-tip }

<br>

### 데이터 전송
  
`FVector_NetQuantize, TEnumAsByte`와 같은 타입을 사용하여 데이터를 양자화하고, 이 데이터를 클라이언트에 복제하는 데 필요한 크기를 크게 줄여 서버간 효율적으로 통신할 수 있다고 하는데요. 아직 사용해본적은 없지만, 이를 `ReplicatedUsing`과 혼합하여 사용하면 효과적이겠죠?

```c++
// Contains information of a single hitscan weapon linetrace
USTRUCT()
struct FHitScanTrace {
  GENERATED_BODY()

public:
  UPROPERTY()
  TEnumAsByte<EPhysicalSurface> SurfaceType;

  UPROPERTY()
  FVector_NetQuantize TraceTo;
};
```

<br>

### 서버 Tick 시간

```c++
//BeginPlay...
NetUpdateFrequency = 66.f;      
MinNetUpdateFrequency = 33.f;   //최소 프레임
```

`NetUpdateFrequency`는 액터가 자체 업데이트를 할 때 초당 최대 몇번까지 업데이트를 할지 나타내며, `MinNetUpdateFrequency`는 최소 몇 번까지 업데이트를 시도할지 나타내는데요. 이처럼 액터는 서버로 전송되는 프레임(업데이트 빈도)을 지정할 수 있습니다. 무조건 프레임을 높이는 것은 좋지 않지만, 게임 플레이에 큰 영향을 미치는 요소들은 프레임을 높게하여 자주 전송해주는 것이 좋겠죠?

- EX) 플레이어가 조종하는 액터 : 10프레임 (0.1초)
- EX) AI처럼 느리게 움직이는 액터 : 5프레임 (0.2초)
  
<br>

### 액터의 파괴

액터가 사용되고 `Destory`를 통해서 파괴되는 경우에는 서버에서 클라이언트에게 많은 시간을 남겨두지 않습니다. 그렇기 때문에 만약 액터가 파괴될때 효과음이나 이펙트를 사용할때 바로 `Destory`를 사용하게 되면 클라이언트에서는 실행되지 않거나 불안정하게 종료되는 경우가 발생할 수 있죠. 

이 문제를 해결하기 위해서는 액터의 가시성과 물리충돌을 없애고, 파괴까지 시간을 부여하면 됩니다. 시간으로 하는게 조금 모호한 경우에는 배열에 종료 후 진행되는 작업들을 기록하고, 모두 종료된 이후에 파괴하는 절차를 밟아도 되겠습니다!

```c++
// 바로삭제
Destroy();

// 클라이언트가 일정 작업을 처리하도록 몇 초뒤에 삭제
Component->SetVisibility(false, true);
Component->SetCollisionEnabled(ECollisionEnabled::NoCollision);
SetLifeSpan(2.f);
```

<br>

## 참고

DS서버와 흔히 말하는 Lobby 서버의 개념을 헷갈릴 수도 있는데, 이는 용도가 다릅니다. DS서버는 실질적으로 플레이어들이 하나의 레벨에서 게임을 진행할 때 사용하는 서버이고, Lobby서버는 게임 진행을 제외한 '로비, 방 정보' 등에서 사용합니다. 즉, Lobby서버는 서버 개발자 분들이 패킷을 통해서 DB의 정보를 주고 받는 작업이라고 생각하시면 되겠습니다.


## 마무리

네. 이렇게 DS의 장점부터 실행과 디버깅하는 법까지 해봤습니다. 사실 저도 혼자 개발하면서 DS를 키는 것과 디버깅하는 것은 처음 해봤습니다. 인턴 하면서는 해봤는데 말이죠. 아무쪼록 저의 게시글이 도움이 되었기를 바라며 마무리하겠습니다.

※ 참고자료 : [UE5 Docs](https://docs.unrealengine.com/4.27/en-US/InteractiveExperiences/Networking/HowTo/DedicatedServers/)