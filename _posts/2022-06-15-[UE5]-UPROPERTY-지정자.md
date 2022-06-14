---
layout: post
title: UE5 | UPROPERTY 지정자
subtitle: UPROPERTY
categories: UE5
tags: [UE5, UnrealEngine, Udemy, C++]
---

## UPROPERTY란

  먼저 UPROPERTY()란 변수/함수 앞에 붙는 __Reflection 매크로이다.__ 이는 실행시간에 자기 자신을 조사하라는 것을 명시하며, 아래 코드처럼 매크로에 인자 값을 넣어 용도에 맞게 활용한다. 

  ```cpp
  UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "AI")
	AActor* PointA;

  UPROPERTY(EditDefaultsOnly, Category="A | B")
	AActor* PointB;
  ```

  이렇게 매크로가 선언된 변수는 __Garbage Collection(GC)에__ 의해 생명주기가 자동으로 관리되며, Reflection을 통해서 런타임 중에 변수의 이름, 유형과 같은 정보를 확인할 수 있다.

## 구분
  이는 __"변수 공개/수정 권한, 블루프린트 공개/수정 권한, 카테고리, 메타"와__ 같이 크게 4가지의 타입으로 다음과 같이 구분할 수 있다.

  1. 변수 공개/수정 권한
    이는 말 그대로 변수에 대한 권한을 설정하는 것으로, __VisibleInstanceOnly와__ 같이 아래의 표와 __Visible, Edit를__ 섞어서 사용한다.

      |||
      |:--:|:--:|
      |DefaultsOnly|블루프린트 에디터 창의 디테일 패널에서...|
      |InstanceOnly|월드 상에 인스턴스화된 오브젝트의 디테일 패널에서..| 
      |Anywhere|위 2가지를 모두 만족| 


      ※ 이때 포인터 변수는 사용시 Visible로만 설정해야 하는데, 이는 포인터 변수를 Edit로 설정하면 참고자를 수정하려하기 때문이다.

  2. 블루프린트 공개/수정 권한
    이는 C++로 로직을 구상하는 것이 아닌 블루프린트에서 구상할 때, 변수에 대한 권한을 주는지에 대한 설정이다. 아래의 경우가 존재한다.

      |||
      |:--:|:--:|
      |BlueprintReadOnly|블루프린트에서 변수 읽기 가능|
      |BlueprintReadWrite|블루프린트에서 변수 읽기/쓰기 가능|
      |BlueprintGetter|해당 변수에 접근 가능한 함수를 지정하고 이를 통해 접근 (get을 커스텀)|
      |BlueprintSetter|해당 변수에 수정 가능한 함수를 지정하고 이를 통해 수정 (set을 커스텀)|
      |BlueprintImplementableEvent|C++에서 바디 구현이 불가능하며, 헤더파일에만 작성 된 함수로 일반적으로 블루프린트에서 작성|
      |BlueprintNativeEvent|C++로 함수 바디 구현이 가능하고, 블루프린트에서 오버라이드 가능|

  3. Category  
    디테일 패널의 항목 명칭한다. 

      ```cpp
      UPROPERTY(EditDefaultsOnly, Category="A | B")
      AActor* PointB;
      ```

  4. Meta  
    에디터 관련 다양한 기능을 구현한다.

      |||
      |:--:|:--:|
      |AllowPrivateAccess|private으로 선언된 변수를 에디터에서 접근 가능하도록 수정|

      ```cpp
      UPROPERTY(EditDefaultsOnly, meta = (AllowPrivateAccess="true"))
      AActor* PointB;
      ```

## 참조
  더 많은 정보는 [프로퍼티 공식 문서](#https://docs.unrealengine.com/4.26/ko/ProgrammingAndScripting/GameplayArchitecture/Properties/)를 참조하여 확인할 수 있다.