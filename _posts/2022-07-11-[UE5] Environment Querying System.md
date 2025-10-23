---
layout: post
title: "[UE5] Environment Querying System"
description: EQS 사용 방법에 대한 설명
date: 2022-07-11 09:12:00 +0900
categories: [Unreal Engine, Analysis]
tags: [Unreal Engine, Environment Querying System, EQS]
---

## EQS란?

EQS란 무엇일까요? 언리얼에서는 이를 `게임 속 환경에서 데이터를 수집하여, 질문을 생성하고 질문을 던지는 방법의 툴`이라고 정의하고 있는데요. 인공지능 시스템 내에서 환경으로부터 데이터를 수집하기 위해 사용되는 기능입니다. EQS 내에서 다양한 테스트를 통해 수집된 데이터에 대해 가장 적합한 행동을 생성할 수 있습니다.
  
EQS 쿼리는 AI 캐릭터에게 공격하기 위한 시야를 제공하는 최적의 위치, 가장 가까운 상태 또는 탄약 픽업 또는 가장 가까운 커버 포인트(다른 가능성 중)를 찾도록 지시하는 데 사용할 수 있습니다. 고도화된 AI를 만들기에 사용할 수 있는 도구라고 할 수 있죠.

저도 많이 사용해보지 않았고, 심도있게 공부한 부분이 아니니 "이런 기능이 있다!"의 차원에서 간단하게 설명해보겠습니다.

> 공식 문서는 [여기](https://dev.epicgames.com/documentation/ko-kr/unreal-engine/environment-query-system-in-unreal-engine)를 참고해주세요.
{: .prompt-tip }

<br>

### EQS의 핵심 개념

EQS는 AI가 환경을 평가하고 최적의 결정을 내린다고 했죠? 그 과정에 있어 필요한 핵심 개념은 아래와 같습니다.

1. __쿼리(Query)__

    AI가 환경에서 정보를 수집하는 과정으로, AI가 특정 조건을 만족하는 위치나 객체를 찾도록 설정하는 기본 단위입니다. 예를 들어, "플레이어 주변에서 가장 가까운 엄폐물을 찾아라" 같은 요청을 보낼 수 있겠죠?

2. __쿼리 컨텍스트(Query Context)__

    위 쿼리를 수행할 때 기준이 되는 위치나 객체를 의미합니다. 이를 통해 AI는 특정 지점을 기준으로 환경을 분석할 수 있습니다. 위에서는 플레이어 주변에서 가장 가까운 엄폐물을 찾으라고 했으니, "플레이어의 위치, AI의 현재 위치"이 컨텍스트가 되겠네요.

3. __쿼리 아이템(Query Item)__

    쿼리 실행 시 평가할 후보들을 의미하는데요. AI는 여러 후보를 비교한 후, 최적의 선택을 하게 됩니다. 위 예시에서는 월드 상에서 여러 개의 엄폐물이 쿼리 아이템이 되겠습니다.

<br>

### EQS의 동작 방식

간단하게 생각해서 <ins>EQS의 동작 방식은 **점수**를 기준으로 판단</ins> 합니다. EQS 쿼리를 통해서 여러개의 행동 선택지를 찾고, 각각에 대해서 점수를 부여합니다. 이 결과로 도출된 점수 중에서 가장 높은 점수의 아이템이 선택되는 것이죠. 해당 아이템으로 이동하거나 추적하는 등의 액션을 취하는 것입니다. 엄폐물을 찾는 것으로 예시를 들어보겠습니다.

1. __후보 아이템 생성__
  
    환경에서 AI가 평가할 여러 후보 아이템을 생성합니다. 현재 레벨에서 엄폐물로 사용할 수 있는 모든 객체를 후보 아이템으로 선택합니다.

2. __테스트 수행__

    EQS는 후보 아이템에 대해 다양한 테스트를 수행하여 적절한지를 평가합니다. 엄폐물은 "현재 거리, 플레이어에게 보이지 않는지"등의 조건을 검사할 수 있습니다.

3. __점수화 및 최적의 선택__

    테스트 결과를 기반으로 각 아이템에 점수를 부여하고, 가장 높은 점수를 받은 아이템이 최종적으로 선택합니다. 가장 가깝고, 플레이어에게 보이지 않는 위치가 가장 높은 점수를 받겠죠?

4. __결과 반환 및 행동 수행__

    최적의 결과가 결정되면, AI는 해당 위치나 대상을 목표로 삼아 행동을 수행합니다. 점수가 가장 높은 엄폐물로 이동하여 숨는 행동을 취할 수 있죠.

<br>

## EQS 사용 방법
  
![UE4_Setting](/assets/img/post/EQS/UE4_Setting.png){: width="762" height="465"}
*출처 : 언리얼 공식 문서*

EQS를 사용하기에 앞서 UE4버전에서는 위와 같이 `Edit` -> `Preferences/AI/Environment-Querying-System`을 활성화 해주어야 하고, UE5버전에서는 기본적으로 활성화되어 있다고 합니다.

<br>

### Environment Query

![EQSAsset](/assets/img/post/EQS/EQSAsset.png){: width="434" height="352"}
*Environment Query의 생성*
  
EQS의 기능을 다루기 위해서 필요한 Environment Query는 `Artificial` -> `Intelligence/Environment Query`에서 생성할 수 있습니다. 이는 AI 행동의 기반이 되는 환경 정보 값을 검색하는 과정을 의미하는데요. 

<br>

생성이 완료되면 아래 그림의 왼쪽과 같은 것을 볼 수 있는데, 이는 AI를 만들어봤다면 볼 수 있는 __Behavior Tree와__ 비슷하게 구성되는 것을 볼 수 있습니다.

<br>

![EQSAsset](/assets/img/post/EQS/EQS_Query.png){: width="530" height="194"}
*Environment Query 구조*

먼저 좌측 그림의 SimpleGrid는 질문자(Querier)를 중심으로 반경 내에 Grid를 생성하고요. 우측 그림의 'Distance'는 반경 내의 Actor를 찾기 위한 조건이고, 'Trace'는 질문자의 시점에서 벗어나는 지점을 알기 위한 조건입니다.
  
> 이때 기본적으로 제공하는 것이 아닌, EQS의 기능을 커스텀으로 따로 제작할 수 있는데 __EnvQueryContext_BlueprintBase를__ 사용하며, 이는 사전에 설정한 Grid내에서 사용자 액터를 찾아 반환하거나, 무언가에 대해 반환할때 사용합니다.
{: .prompt-tip }

<br>
   
### EQSTestingPawn

![EQS_Example](/assets/img/post/EQS/EQS_Example.gif){: width="606" height="417"}
*EQS Pawn 예시*

EQS가 실제로 하고 있는 역할에 대해서 확인을 돕는 특수한 Pawn 클래스로, 이전 작성한 Environment Query의 로직에 영향을 받도록 설정합니다. 위 그림과 같이 구체를 통해서 시각적으로 디버깅할 수 있게 도와주는 특수한 Pawn이라고 생각하시면 되겠습니다.

<br>

![EQS_TestingPawn_Query](/assets/img/post/EQS/EQS_TestingPawn_Query.png){: width="460" height="380"}

EQS_TestingPawn은 위 그림과 같이 생성할 수 있고, EQS의 __Query Template에서__ 이전에 만든 __Environment Query를__ 연결하여 사용해야만 합니다. 세부적인 파라미터 조정 값은 아래와 같습니다.

|구분|설명|
|:--:|:--|
|Query Template|Query Asset을 선택|
|Query Config|Query Params 를 대체해 생긴 프로퍼티|
|Time Limit Per Step|EQS Manager는 시분할 방식으로 Query를 처리하는데, 이때 주어진 시간에 얼마나 자주 Query를 던질지 결정|
|Step to Debug Draw|Time Limit Per Step에 설정된 시간마다 생성된 결과들을 화면상에 디버깅해 보여줄지 에 대한 설정|
|Draw Lables|모든 Item의 Debug Lables를 그림|
|Draw Failed Items|Test에 실패한 Item들을 어두운 파란색으로 표시|
|Re Run Query Only on Finish|마지막 변화 이후 Pawn을 Testing, 비활성화된 경우에는 Pawn이 움직일 때 마다 연속해서 Query를 발생|
|Should be Visible in Game|시뮬레이션 모드에서 Pawn을 이동 가능|
|Tick During Game|런타임에 Tick을 돌지에 대한 여부|
|Querying Mode|All Matching : EQS의 최종적인 Query들의 모든 값이 계산된 결과를 표현|

<br>

## EQS 실습

![EQS_Example](/assets/img/post/EQS/EQS_Example2.gif){: width="682" height="353"}
*새로운 EQS 예시*

이번에는 위 영상과 같이 실제로 AI가 사용자를 찾아 총을 발사하려고 할때의 최적 위치를 찾는 EQS를 만들어보려고 합니다. 조건은 아래와 같이 잡아볼게요.

1. AI의 일정 범위 내를 탐색하며, 가까운 거리일 수록 높은 점수를 갖는다.
2. Player의 시야에 보이는 위치는 낮은 점수를 가지게 된다. (벽으로 가려져있다면 높은 점수를 가지게 된다.)
3. Player의 시야에 보이는 위치가 높은 점수를 가지게 된다. 다만 일정 높이에서의 시야를 뜻하며, 쉽게말하면 방해물로 인해서 AI의 타격점이 적음을 의미한다.

<br>

### 1. AI 주변 범위 지정
  
![EQS_1](/assets/img/post/EQS/EQS_1.png){: width="492" height="389"}
    
  - Point : Donut을 사용하여 AI 주변에 일정 범위의 원형을 생성하며, 각 원형으로 보이는 곳은 포인트로 추후 캐릭터가 이동할 위치를 의미한다.
  - 위 그림과 같이 구성할 필요는 없고, 사용자에 맞게 범위, 포인트의 개수등을 설정해준다. 
  - 이때 AI는 Navigation을 활용하기 때문에 `Projection Data/Trace Mode`를 `Navigation`으로 설정하여 주었다.

<br>

### 2. 거리에 따른 점수 부여
  
![EQS_2](/assets/img/post/EQS/EQS_2.png){: width="492" height="389"}
    
  - AI로부터 가까운 거리에 대해 높은 점수를 부여받은 것을 볼 수 있다.
  - Distance를 사용하여 최소거리에 대해 높은 점수를 부여하였으며, `FilterType`은 `Mimimum`으로, `Scoring Factor`는 -2로 지정해주었다.

<br>

### 3. Player와의 시야에 따른 점수 부여

![EQS_3](/assets/img/post/EQS/EQS_3.png){: width="492" height="389"}
    
  - AI가 Player로부터 시야에 벗어난 경우에 높은 점수를 부여받은 것을 볼 수 있다.
  - Trace를 사용하여 Player로부터 시야에 벗어난 경우에 높은 점수를 부여하였으며, 이때 Context는 Player들을 반환한다. 아래 그림과 같다.

![AllPlayerFind](/assets/img/post/EQS/AllPlayerFind.png){: width="648" height="122"}
  
<br>

### 4. Player와의 시야(높이)에 따른 점수 부여

![EQS_4](/assets/img/post/EQS/EQS_4.png){: width="492" height="389"}
    
  - AI가 Player로부터 시야에 보이는 경우에 높은 점수를 부여받은 것을 볼 수 있다. 단 낮은 벽과 같이 AI를 일정 부분 가려야한다.
  - Trace를 사용하여 Player로부터 시야에 보이는 경우에 높은 점수를 부여하였으며, 이때 높이를 지정해야하기 때문에 Item(벽), Context(Player)의 Z-Offset을 100으로 설정해주었다.

<br>

### 5. 결과

![FindMoveTo](/assets/img/post/EQS/FindMoveTo.png){: width="492" height="389"}
  
이 모든 결과 EQS는 위 그림과 같이 구성됩니다.

## 마무리

이렇게 제가 알고 있는 EQS에 대해서 간단하게 설명해봤는데요. 아직 AI를 사용하는 경우가 많지 않아서 부족한 부분이 많다는 생각이 드네요. 이는 추후에 다른 포스트로 더욱 보완해보겠습니다! 나중에 Lyra 분석해보는 겸사겸사 말입니다!