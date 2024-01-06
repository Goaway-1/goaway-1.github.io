---
layout: post
title: Reflection System
subtitle: Reflection
categories: UE5
tags: [UE5, UnrealEngine]
---
## [Reflection System](https://docs.unrealengine.com/5.3/ko/unreal-object-handling-in-unreal-engine/)

### 정의

언리얼 엔진에서의 Reflection 이란, 언리얼 엔진이 실행되는 시간에 동작하는 오브젝트를 조사하는 것을 의미하는데, 쉽게 말해 언리얼 엔진에서 오브젝트를 관리한다고 생각해 주시면 되겠습니다. 

(※ 물론 표준 C++에서는 지원하지 않습니다.)

그렇다고 모든 오브젝트에 대해 조사를 진행하지도 않습니다. 이 기능을 사용하기 위해서는 오브젝트에 대한 정보를 제공해 달라고 특정 매크로(**`UPROPERTY()`**, **`UFUNCTION()`**)를 사용하여 엔진에 요청을 해두어야 합니다. 이 정보는 빌드 과정에서 UBT(Ubisoft Build Tool) 및 UHT(Ubisoft Header Tool)가 **`FileName.generated.h`, `GENERATED_BODY()`** 에 데이터가 저장되어 런타임에서 활용됩니다.

특히, 포인터를 사용하는 오브젝트 관리 시 리플렉션을 이용하여 가비지 컬렉션에서 안전하게 참조할 수 있도록 하는 것이 중요합니다. 이는 아래에서 마저 다루겠습니다.


### 종류 :

언리얼 오브젝트에는 프로퍼티와 함수를 지정할 수 있는데, 각 상황에 사용되는 프로퍼티는 아래와 같아요. 인터페이스, 클래스와 같은 매크로는 C++ 파일을 생성할 때, 자동으로 생성되는 것을 알 수 있죠. 

- 멤버 변수 : [UPROPERTY](https://docs.unrealengine.com/4.27/ko/ProgrammingAndScripting/GameplayArchitecture/Properties/)
- 멤버 함수 : [UFUNCTION](https://docs.unrealengine.com/4.27/ko/ProgrammingAndScripting/GameplayArchitecture/Functions/)
- 인터페이스 :  [UInterface](https://docs.unrealengine.com/4.26/ko/ProgrammingAndScripting/GameplayArchitecture/Interfaces/)
- 클래스 : [UClass](https://docs.unrealengine.com/4.27/ko/ProgrammingAndScripting/GameplayArchitecture/Classes/Specifiers/)
- 열거형 : [UEnum](https://docs.unrealengine.com/4.27/en-US/API/Runtime/CoreUObject/UObject/UEnum/)

이 매크로에 대한 정의 ‘ObjectMacros.h’에 선언되어 있고, ‘EditDefaultsOnly, BlueprintReadWrite’와 같이 추가할 수 있는 메타데이터들도 존재하니까 궁금하시다면 찾아봐도 좋겠습니다.

```cpp
// These macros wrap metadata parsed by the Unreal Header Tool, and are otherwise
// ignored when code containing them is compiled by the C++ compiler
#define UPROPERTY(...)
#define UFUNCTION(...)
#define USTRUCT(...)
#define UMETA(...)
#define UPARAM(...)
#define UENUM(...)
#define UDELEGATE(...)
#define RIGVM_METHOD(...)
```

## 기능

이 리플렉션을 사용하면 언리얼 엔진이 자체적으로 오브젝트들을 관리해 준다고 했는데요. 아래와 같이 다양한 기능들을 제공합니다. 

다양한 기능이 있지만, C++에서 제공하지 않는 메모리 관리 기능을 보완하기 위한 목적이 가장 크다고 생각해요. 게임이라는 방대한 시스템에서 모든 오브젝트의 메모리 관리를 프로그래머가 관리하기는 힘드니, 대신 엔진이 유효성을 검사해 주거나 초기화를 통해서 자동으로 메모리를 관리해 주는 것이죠.

1. **프로퍼티 업데이트 및 자동 초기화**
    
    CDO를 사용하여 오브젝트가 가진 기본값을 보관하며, 기본 값이 변동되면 갱신하여 저장
    
    값을 초기화하지 않아도 기본 값으로 자동 초기화
    
2. **직렬화 (Serialization)**
    
    객체를 지정한 포맷에 맞게 디스크에 저장/호출을 일괄적으로 진행
    
3. [**에디터 통합**](https://docs.unrealengine.com/4.26/en-US/ProgrammingAndScripting/GameplayArchitecture/Properties/Specifiers/)
    
    위에서 설명했듯 리플렉션 매크로에 메타 데이터를 추가하여 에디터에서 제어 및 노출이 가능
    
4. **런타임 유형 정보 및 형변환**
    
    오브젝트에 대해서 런타임에서 정보를 얻어 안전하게 형 변환이 가능하다. (실패 시 null 반환)
    
5. **가비지 컬렉션**
    
    표준 C++에서는 제공되지 않는 기능으로, 더 이상 참조되지 않거나 소멸 예약한 오브젝트에 대해서 메모리를 자동으로 관리해 주는 기능으로 NULL 검사 안정성이 높아 허상 참조 예방이 가능
    
6. **네트워크 리플리케이션**
    
    추가 설정 시 직렬화와 유사하게 네트워크상에서 서버-클라이언트 간의 통신 (전송/수신)을 자동으로 진행 
    

## 예시

위 기능 중에서 몇 가지만 살펴보도록 하겠습니다!

### CDO

CDO는 ***Class Default Object***의 약어로 언리얼 객체가 가진 기본값을 보관하는 템플릿 객체로 클래스 정보에 포함되어 있습니다. 하나의 클래스를 인스턴스화하여 레벨에 배치하면 일관성 있게 기본값을 저장해 주고, 만약 값이 변경되어도 레벨에 배치된 모든 인스턴스들의 값을 업데이트해 주는 효자죠. **이 CDO 존재의 가장 큰 이유는 오브젝트 생성 시 매번 초기화하지 않고, 미리 기본 인스턴스를 만들어 놓고 복제함으로써 조금 더 메모리를 효과적으로 사용하기 위해서입니다.**

CDO의 정보를 플레이 중에 얻어오기 위해서는 `UTypeName::Static/StructClass()` 나 `Instance->GetClass()` 함수를 통해서 `UClass`의 정보를 받아오고, `GetDefaultObject()` 함수를 통해서 얻을 수 있어요.

```cpp
UClass* ClassRuntime = GetClass();
UClass* ClassComplietime = U클래스::StaticClass();

// 검증 코드
check(ClassRuntime == ClassComplietime)

// 기존 CDO의 변수 값을 가져옴
ClssRuntime->GetDefaultObject<U클래스>()->변수명;
```

이렇게 가져온 정보를 바탕으로 기본값을 검증하여 변경된 값과의 차이를 비교하고자 할 때 사용할 수 있겠죠.

※ 물론 저도 아직 사용 안 해봤어요. ㅎ

### GC

GC는 다들 아시죠? C++는 메모리 관리를 직접해야 되잖아요…? 하지만 킹왕짱 언리얼 엔진은 `Garbage Collection`이라고 프로그램에서 더 이상 참조되지 않아 사용되지 않는 오브젝트를 자동으로 감지하여 메모리를 회수해 주는 시스템을 지원합니다. 또한 아래와 같은 일을 하죠.

1. ***메모리 누수 (Leak)*** : GC가 자동으로 메모리를 관리하여 해결
2. ***허상 포인터 (Dangling)*** : 해당 언리얼 오브젝트가 유효한지 확인하는 함수 제공 (`IsValid`)
3. ***와일드 포인터 (Wild)*** : `UPROPERTY` 매크로를 통해 자동으로 `nullptr`로 초기화

언리얼 엔진에서도 일반적으로 자주 사용하는 동작 방식인 `Mark-Sweap` 방식을 사용하며,  성능을 위해서 병렬 처리 및 클러스터링과 같은 기능을 탑재하고 있죠. GC와 관련된 설정을 하고 싶다면, 에디터에서 ‘`Project Settings/Engine/Garbage Collection`’ 를 확인할 수 있습니다.

## 오늘의 한줄

메모리를 관리하려면 GC를, GC 쓰려면 리플렉션을, 리플렉션을 위해서는 지정된 매크로를 쓰자!