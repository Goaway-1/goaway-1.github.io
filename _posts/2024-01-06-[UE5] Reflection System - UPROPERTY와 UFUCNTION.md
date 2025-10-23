---
layout: post
title: "[UE5] Reflection System - UPROPERTY와 UFUCNTION"
description: 언리얼에서의 Reflection System에 대해서 소개
date: 2024-01-06 13:00:00 +0900
categories: [Unreal Engine, Analysis]
tags: [Unreal Engine, Reflection, UFUNCTION, UPROPERTY]
---
## Reflection System 란?
[이전 포스트](https://goaway-1.github.io/posts/UE5-언리얼-Garbage-Collection의-동작과-규칙/)에서 언리얼 엔진의 메모리 관리 "GC와 SmartPoint"에 대해서 설명한 적이 있었는데요. 이번 포스트에서는 GC를 사용하기 위해서 사용했던 __<ins>Reflection 기능(프로퍼티 시스템)</ins>__ 에 대해서 깊이 알아보려고 합니다.

<br>

`Reflection`이란, 언리얼 엔진이 **런타임 중 객체의 메타정보를 조회하고 조작할 수 있도록** 하는 기능입니다.  모든 객체를 자동으로 분석하는 건 아니고, 개발자가 **특정 매크로(`UCLASS()`, `UPROPERTY()`, `UFUNCTION()` 등)** 를 사용해 엔진에게 "**이 멤버에 대한 정보를 노출해줘**"라고 명시해야 합니다. 

이 정보를 기반으로, 언리얼의 빌드 파이프라인은 **UHT(Unreal Header Tool)** 를 통해 관련 메타데이터를 생성하며, 이 데이터는 `FileName.generated.h` 파일에 저장됩니다.  

<br>

```cpp
// 실제 Character 클래스 구조
UCLASS(config=Game, BlueprintType, meta=(ShortTooltip="A character is a type of Pawn that includes the ability to walk around."), MinimalAPI)  
class ACharacter : public APawn  
{  
  GENERATED_BODY()
  ...
}
```

클래스를 만들 때, 항상 보이는 친구가 있죠? `GENERATED_BODY()` 매크로는 해당 정보를 실제 클래스에 삽입할 위치를 지정하는 역할을 합니다. 이외에도 타입에 따라 `GENERATED_USTRUCT_BODY()`, `GENERATED_UCLASS_BODY()` 로 구분됩니다.


<br>


> 특히, 포인터를 사용하는 오브젝트 관리 시 리플렉션을 이용하여 가비지 컬렉션에서 안전하게 참조할 수 있도록 하는 것이 중요합니다. 이는 아래에서 마저 다루겠습니다.
{: .prompt-tip }

<br>

## 매크로의 종류
- 멤버 변수 : [UPROPERTY](https://docs.unrealengine.com/4.27/ko/ProgrammingAndScripting/GameplayArchitecture/Properties/)
- 멤버 함수 : [UFUNCTION](https://docs.unrealengine.com/4.27/ko/ProgrammingAndScripting/GameplayArchitecture/Functions/)
- 인터페이스 :  [UInterface](https://docs.unrealengine.com/4.26/ko/ProgrammingAndScripting/GameplayArchitecture/Interfaces/)
- 클래스 : [UClass](https://docs.unrealengine.com/4.27/ko/ProgrammingAndScripting/GameplayArchitecture/Classes/Specifiers/)
- 열거형 : [UEnum](https://docs.unrealengine.com/4.27/en-US/API/Runtime/CoreUObject/UObject/UEnum/)

언리얼에서는 함수나 변수, 모든 선언부에 매크로를 지정할 수 있는데, 각 상황에 사용되는 대표적인 매크로는 위와 같습니다. 인터페이스, 클래스와 같은 매크로는 C++ 파일을 생성할 때, 자동으로 생성되는 것을 알 수 있죠. 

<br>

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

이 매크로에 대한 정의 `ObjectMacros.h`에 선언되어 있고, 'EditDefaultsOnly, BlueprintReadWrite'와 같이 추가할 수 있는 메타 데이터들도 존재하니까 궁금하시다면 찾아봐도 좋겠습니다.

<br>

## Reflection System 기능
이 **리플렉션 시스템(Reflection System)**을 사용하면 언리얼 엔진이 자체적으로 오브젝트들을 관리해 준다고 했는데요. 특히 이전에 설명했던 GC와 관련하여 <ins>**메모리 관리**</ins> 측면에서 큰 역할을 합니다. 게임은 수많은 오브젝트가 생성되고 소멸되는 복잡한 시스템이기 때문에, 프로그래머가 일일이 메모리를 직접 관리하기엔 한계가 있습니다. 이때 <ins>언리얼 엔진이 대신 오브젝트의 **유효성 검사**, **자동 초기화**, **메모리 해제(GC)** 등을 처리해주며, 안정성과 개발 효율을 동시에 확보</ins>할 수 있죠.

<br>

> 1. 프로퍼티 초기화 및 기본값 관리
> 2. 직렬화 (Serialization)
> 3. 에디터 통합 및 메타데이터 기반 제어
> 4. 가비지 컬렉션 (GC) 및 레퍼런스 무효화

위는 리플렉션 시스템이 제공하는 주요 기능들입니다.

<br>

### 1. 프로퍼티 초기화 및 기본값 관리
#### CDO 란?
일단 이 기능에 대해서 정확히 이해하기 위해서는 `CDO` 에 대해서 확실하게 이해하고 넘어가는게 좋겠습니다. CDO는 **Class Default Object**의 약어로 <ins>언리얼 객체가 가진 기본 값을 보관하는 템플릿 객체</ins>로 클래스 정보에 포함되어 있습니다. 하나의 클래스를 인스턴스화하여 레벨에 배치하면 일관성 있게 기본값을 저장해 주고, 만약 값이 변경되어도 레벨에 배치된 모든 인스턴스들의 값을 업데이트해 주는 효자죠. 

흔히 C++에서 정의한 클래스를 월드에 인스턴스화 해서 배치해보면, 기본 값을 저장하고 있는데 바로 이것입니다. **이 CDO 존재의 가장 큰 이유는 오브젝트 생성 시 매번 초기화하지 않고, 미리 기본 인스턴스를 만들어 놓고 복제함으로써 조금 더 메모리를 효과적으로 사용하기 위해서입니다.**

<br>

> 이 CDO는 런타임 초기화와 동시에 메모리에 올라갑니다. 그렇기 때문에 데이터를 외부에서 로드하도록 하는 것이 좋겠습니다. [참고 자료](https://forums.unrealengine.com/t/questions-about-cdo-in-the-documentation/2509955?utm_source=chatgpt.com)
{: .prompt-warning }


<br>

---
#### CDO 생성 시점과 정보 활용
컴파일 빌드 전에는 CDO가 생성되어 있지 않고, 빌드와 동시에 에디터가 실행될 때, 매크로로 지정된 파라미터들에 대해서 UHT(Unreal Header Tool)이 파싱해서 `.generated.h` 파일을 생성합니다. 

```cpp
UPROPERTY()
int8 Health = 100;
```

만약에 액터에 Float타입의 Health라는 파라미터를 추가하고 100으로 지정해주었다고 해봅시다.

<br>

```cpp
void ACoreActor::BeginPlay()  
{  
  Health = 80;  

	// UClass의 정보 가져오기
  UClass* ClassRunTime = this->GetClass();  
  UClass* ClassComplieTime = ACoreActor::StaticClass();  
  
  // 기존 CDO 액터  
  ACoreActor* DefaultActor = ClassRunTime->GetDefaultObject<ACoreActor>();  
  
  // CDO에 저장된 Health 값  
  int8 CDOHealth = ClassRunTime->GetDefaultObject<ACoreActor>()->Health;  
} 
```

이를 런타임에서 한번 확인해 봅시다. CDO의 정보를 플레이 중에 얻어오기 위해서는 `UTypeName::Static/StructClass()` 나 `Instance->GetClass()` 함수를 통해서 `UClass`의 정보를 받아오고, `GetDefaultObject()` 함수를 통해서 얻을 수 있어요.

<br>


![CDO 데이터](/assets/img/post/Reflection/CDO_Diff.png){: width="598" height="144"}
*CDO 데이터*

이처럼 가져온 `CDO` 정보를 바탕으로 Health에 변화가 생겼는지 확인해보면, CDO의 값은 기본 값으로 저장되어 있는 모습을 볼 수 있습니다.

<br>

> 어떤 멤버 변수가 "디폴트 값"인지 아닌지는 `IsDefaultSubobject()` 혹은 `IsTemplate()` 등의 함수를 통해 판단할 수 있습니다.
{: .prompt-tip }

<br>

---
#### CDO의 존재 이유
이제 `CDO`를 이해했으니, 왜 <ins>"객체 생성 시 해당 기본값을 자동 적용"</ins>하는지 알 수 있을 겁니다. C++로 만든 클래스를 BP로 인스턴스화하면, 자동으로 값이 지정되어 있죠? 이게 바로 CDO 데이터를 통해서 지정하는 것입니다. 

CDO는 기본 값인데, 누군가 `Health = 100` → `Health = 90`으로 **수정**했다면 어찌될까요? 

<br>

1. CDO 기준 `Health = 100`
2. 레벨 인스턴스에서 `Health = 90`
3. 언리얼은 이걸 "사용자가 바꿨구나"라고 인식해서 **`Health = 90`만 저장**

언리얼은 변경된 값만 디스크에 저장하기 때문에 저장 용량이 줄고, 정확한 상태 복원이 가능합니다.

<br>

### 2. 직렬화 (Serialization)
직렬화란 뭘까요? 간단하게 말하면, 데이터를 기계가 쉽게 읽고, 쓰도록 나타낸 형태를 말합니다. 그렇게 되면 데이터의 손실을 줄일 수 있겠죠?

<br>

언리얼 엔진에서 <ins>`UPROPERTY`로 선언된 멤버는 기본적으로 **자동 직렬화 대상**이 됩니다. 이는 저장/복원뿐 아니라, **블루프린트 노출, 네트워크 리플리케이션, Garbage Collection, 디테일 패널 바인딩** 등 다양한 시스템에서 사용</ins>됩니다. 직렬화 로직을 커스터마이징해야 하는 경우에는 `Serialize()` 함수를 오버라이드하면 됩니다. 예를 들어, 기본 직렬 방식이 원하는 구조가 아니거나, **버전 마이그레이션, 압축, 암호화** 같은 고유 처리 로직이 필요할 경우에 사용하죠.

```c++
virtual void Serialize(FArchive& Ar) override 
{     
	Super::Serialize(Ar);          
	Ar << CustomData1;     
	Ar << CustomData2; 
}
```

<br>

#### 커스텀 직렬화 경험
제가 직접 경험한 사례로, <ins>세이브 데이터 구조를 변경하면서도 기존 데이터를 유지해야 했던 상황</ins>이 있었습니다. 기존 세이브 데이터에 Level이라는 데이터가 있었는데, 이후 업데이트에서 `Level`이라는 네이밍이 명확하지 않아, `Power`로 변수명을 바꾸기로 했습니다.  하지만 기존 유저들의 저장된 세이브 데이터를 버릴 순 없기 때문에, **마이그레이션이 필요**했습니다.

```c++
// 이전 세이브 데이터
USTRUCT()
struct FMySaveData
{
  UPROPERTY()
  int8 Level;
    
	friend FArchive& operator<<(FArchive& Ar, FMySaveData& Data) 
	{ 
		Ar << Data.Level; // 필요에 따라 다른 멤버들도 추가 
		return Ar; 
	}
};

// 직렬화된 데이터
UPROPERTY() 
TArray<uint8> SerializedData;
```

변경이 필요한 `Level` 데이터를 직렬화하여 `SerializedData`에 저장해둡니다. 물론 어느 것이 필요할지 모르니, 모두 직렬화해두는 것도 하나의 방법입니다.

<br>

```c++
if (SerializedData.Num() > 0) 
{ 
	// 안전한 역직렬화를 위해 FMemoryReader 사용 
	FMemoryReader Reader(SerializedData, true); 
	Reader.ArIsSaveGame = true; 
	
	// 기존 데이터를 읽어온다. 
	FMySaveData OldData; 
	Reader << OldData; 
	
	// 새로운 데이터로 변환 	
	FMySaveData NewData; 
	NewData.SaveDataVersion = SaveVersion; 
	NewData.LevelClearDatas[0] = OldData.LastClearedLevelDataTableIndex * 2;
	
	SerializedData.Empty(); 
	FMemoryWriter Writer(SerializedData, true); 
	Writer.ArIsSaveGame = true; 
	Writer << NewData; 
	
	// 변환된 데이터를 세이브 슬롯에 저장
		UGameplayStatics::SaveGameToSlot(TargetSaveGameInstance,SaveSlotTypeToString(SaveSlotType), SlotIndex); 
	} 
}
```

이후에는 저 직렬화 데이터에 접근해서 Power로 역직렬화 해주면 됩니다. 

> 기존 세이브 관련 구조체를 없애고, 신규 세이브 구조체만 남긴다는 생각으로 진행했었는데, 여러 이슈로 진행되지 않았습니다. 실제로는 위 방법을 사용해서 적용하지 않았습니다. 예시일뿐...!
{: .prompt-warning }

<br>


### 3. 에디터 통합 및 메타데이터 기반 제어
너무나도 잘 알고 있는 기능이죠? `UPROPERTY`, `UFUNCTION` 등에 `VisibleAnywhere`, `EditDefaultsOnly`, `BlueprintReadWrite`과 같은 메타 데이터를 추가해 변수나 함수를 에디터에서 제어 가능하도록 합니다. 

<br>

```cpp  
UPROPERTY(EditAnywhere, BlueprintReadWrite, Category=Actor, AdvancedDisplay)  
uint8 bFindCameraComponentWhenViewTarget:1;
```

- **EditAnywhere** : **디테일 패널** 어디서든 편집 가능하다는 의미로, 블루프린트 클래스, 레벨에 배치된 클래스든지 상관없이 편집이 된다는 의미
- **BlueprintReadWrite** : 블루프린트 노드에서 읽고 쓰는 게 가능하다는 의미
- **Category** : 파라미터가 노출되는 카테고리를 지정
- **AdvancedDisplay** : 변수는 기본적으로 숨김 처리되고,  에디터에서 “Advanced”를 눌러야 보이게 됌

<br>

```cpp
UFUNCTION(BlueprintCallable, meta=(DisplayName = "Set Actor Location", 
	ScriptName = "SetActorLocation", Keywords="position"), 
	Category="Transformation")  
ENGINE_API bool K2_SetActorLocation(FVector NewLocation, bool bSweep, FHitResult& SweepHitResult, bool bTeleport);
```

- **BlueprintCallable** : 블루프린트에서 호출 가능
- **DisplayName** : 블루프린트에서 노드의 이름으로 보여질 표시 이름
- **ScriptName** : 스크립트 내부 이름 (이는 블루프린트 전용 함수이기에)
- **Keywords** : 함수 검색 시 나오는 키워드
- **Category** : 함수가 노출되는 카테고리를 지정

이번에도 Actor 클래스에서 UFUNCTION의 예시를 가져와 봤는데, 다들 잘 알고 계실꺼라고 믿고 간단하게 넘어가겠습니다.

<br>

> 더 많은 자료는 [UPROPERTY](https://dev.epicgames.com/documentation/ko-kr/unreal-engine/unreal-engine-uproperties)와 [UFUNCTION](https://dev.epicgames.com/documentation/ko-kr/unreal-engine/ufunctions-in-unreal-engine) 에서 확인하실 수 있습니다.
{: .prompt-tip }

<br>

### 4. GC
[GC는 이전 포스트](https://goaway-1.github.io/posts/UE5-언리얼-Garbage-Collection의-동작과-규칙/)에서 진행했었죠. 간단하게 말하자면 언리얼 엔진은 `Garbage Collection`이라고 프로그램에서 더 이상 참조되지 않아 사용되지 않는 오브젝트를 자동으로 감지하여 메모리를 회수해 주는 시스템을 지원합니다. 또한 아래와 같은 일을 하죠.

1. **메모리 누수 (Leak)** :  참조되지 않거나 수명 종료된 객체는 자동 소멸되며, 이후 포인터는 자동 `nullptr` 처리하여 메모리를 관리
2. **허상 포인터 (Dangling)** : 해당 오브젝트가 유효한지 확인하는 함수 제공 (`IsValid`)
3. **와일드 포인터 (Wild)** : `UPROPERTY` 매크로를 통해 자동으로 `nullptr`로 초기화

<br>

## 마무리
언리얼 엔진의 `Reflection System`은 런타임에서 오브젝트를 관리하고 조사할 수 있는 시스템으로, C++의 부족한 메모리 관리와 객체 관리 기능을 보완합니다. `UPROPERTY, UFUNCTION` 매크로 알고 사용합시다!

> 즉, <ins>메모리를 관리하려면 GC, GC 쓰려면 리플렉션, 리플렉션을 위해서는 지정된 매크로</ins>를 사용합시다!