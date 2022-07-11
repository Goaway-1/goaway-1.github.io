---
layout: post
title: UE5 | Environment Querying System
subtitle: EQS (Environment Querying System), 게임 속 환경에 대한 질문을 생성하고 던지는 방법
categories: UE5
tags: [UE5, UnrealEngine, C++]
---
## EQS

  > 게임 속 환경에서 데이터를 수집하여, __질문을 생성하고 질문을 던지는 방법의 툴__

  인공지능 시스템 내에서 환경으로부터 데이터를 수집하기 위해 사용되는 기능입니다. EQS 내에서 다양한 테스트를 통해 수집된 데이터에 대해 질문할 수 있으며, 질문 유형에 가장 적합한 항목을 생성할 수 있습니다.
  
  EQS 쿼리는 AI 캐릭터에게 공격하기 위한 시야를 제공하는 최적의 위치, 가장 가까운 상태 또는 탄약 픽업 또는 가장 가까운 커버 포인트(다른 가능성 중)를 찾도록 지시하는 데 사용할 수 있습니다. 

  ※ 다만 아직 실험단계의 기능이기 때문에 출시 제품에 사용하는 것은 권장하지 않는다. [원본 링크](#https://docs.unrealengine.com/4.27/ko/InteractiveExperiences/ArtificialIntelligence/EQS)

## EQS의 사용
  
  EQS를 사용하기에 앞서 UE4버전에서는 Editor Preferences/AI/Environment-Querying-System을 활성화해주어야 하지만 UE5버전에서는 그럴 필요가 없다.

### Environment Query
  
  EQS의 기능을 다루기 위해서는 Environment Query를 생성해야하는데 이는 Artificial Intelligence/Environment Query에서 생성할 수 있다.

  <img src="https://raw.githubusercontent.com/Goaway-1/goaway-1.github.io/master/_posts/images/EQS/EQSAsset.png" height="300" title="EQSAsset">

  생성을 완료하면 아래 그림의 왼쪽과 같은 것을 볼 수 있는데, 이는 AI를 만들어봤다면 볼 수 있는 __Behavior Tree와__ 비슷하게 구성되는 것을 볼 수있다.

  먼저 SimpleGrid는 질문자(Querier)를 중심으로 반경 내에 Grid를 생성하며, Distance와 Trace는 각각 조건을 의미하는데 Distance는 반경 내의 Actor를 찾기 위함이고, Trace는 질문자의 시점에서 벗어나는 지점을 알기 위한 Task이다. 
  
  <img src="https://raw.githubusercontent.com/Goaway-1/goaway-1.github.io/master/_posts/images/EQS/EQS_Query.png" height="300" title="EQS_Query">
  
### EQSTestingPawn

  EQS가 실제로 하고 있는 역할에 대해서 확인을 돕는 특수한 Pawn 클래스로, 이전 작성한 Environment Query의 로직에 영향을 받는다.
  
  이는 구체를 통해서 표현하게 되는데 색상에 따라 결과를 표현한다.

  EQS_TestingPawn은 아래 그림과 같이 생성할 수 있고, EQS의 __Query Template에서__ 이전에 만든 __Environment Query를__ 연결하여 사용한다.

  <img src="https://raw.githubusercontent.com/Goaway-1/goaway-1.github.io/master/_posts/images/EQS/EQS_TestingPawn.png" height="300" title="EQS_TestingPawn">
  
  <img src="https://raw.githubusercontent.com/Goaway-1/goaway-1.github.io/master/_posts/images/EQS/EQS_TestingPawn_Query.png" height="300" title="EQS_TestingPawn_Query">

  |option|explanation|
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

### 결과

  실시간으로 테스트를 진행해본 결과는 아래와 같으며, 질문자의 시점내에 있는 곳은 파란색으로 표현되고, 그렇지 않은 곳은 녹색으로 표시되는 것을 볼 수있다.

  <img src="https://raw.githubusercontent.com/Goaway-1/goaway-1.github.io/master/_posts/images/EQS/EQS_Example.gif" height="300" title="EQS_Example">

  이처럼 EQS를 활용하여 AI가 환경요소에 따른 활동을 선택할 수 있으며, 멀티플레이의 GameMode에서도 활용될 수 있다.

  [참고한 자료](#https://bbagwang.com/unreal-engine/ue4-%EC%97%90%EC%84%9C%EC%9D%98-environment-querying-system/)