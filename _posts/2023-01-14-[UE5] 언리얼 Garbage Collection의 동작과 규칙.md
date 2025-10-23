---
layout: post
title: "[UE5] 언리얼 Garbage Collection의 동작과 규칙"
description: 언리얼에서의 Garbage Collection 동작 원리와 이용 규칙에 대해서 소개
date: 2023-01-14 09:00:00 +0900
categories: [Unreal Engine, Analysis]
tags: [Unreal Engine, Garbage Collection, GC, UPROPERTY, SmartPointer]
---
## Garbage Collection (GC) 란?

최근의 게임은 예전과 비교할 수 없을 정도로 플레이 타임, 레벨의 규모, 그리고 게임 규칙의 복잡성이 증가했습니다. 게임 내에서 사용되는 수많은 리소스를 하나하나 수동으로 관리하는 것이 이상적일 수는 있지만, 현실적으로는 매우 어렵고 비효율적입니다. 


예를 들어, 플레이어가 수류탄을 던지면, '폭발 이펙트, 파편 조각, 충격파 오브젝트' 등이 동시에 생성되는 것을 가정해봅시다. __(실제로 이렇지는 않습니다.)__ 이런 오브젝트들은 몇 초 내에 사라지는 임시 객체들인데, 매번 new와 delete로 관리하려면 생성과 소멸 타이밍, 메모리 해제 등을 모두 직접 처리해야 합니다. 그 과정에서 메모리 누수, 허상 포인터등의 치명적인 상황이 발생할 수 있습니다. 

<br>

![GC의 역할](/assets/img/post/GC/GC.png){: width="350" height="350"}
*GC의 역할*

그래서 필요한 것이 바로 Garbage Collection, GC입니다. 이를 사용하면, 보통 다른 객체에서 참조되지 않거나 사용되고 있지 않는 개체들을 메모리에서 제거하여 메모리를 관리하는 기능을 지원합니다. 제가 따로 할 필요가 없죠. GC란 `Garbage Collection`의 약자로 풀어 쓰면, __"쓰레기 수집"__ 으로 말 그대로 사용하지 않는 쓰레기를 수집하는 역할이라고 생각하면 되겠죠. 

언리얼에서도 마찬가지로 메모리 관리를 위해서 <ins>GC(Garbage Collection)</ins>을 사용하는데요. 사실 GC는 C#에서는 지원하지만, C++에서는 지원하지 않기 때문에 언리얼에서 자체적으로 지원합니다. 언리얼에서는 이 GC를 사용하기 위해서 __<ins>Reflection 기능(프로퍼티 시스템)</ins>을__ 활용합니다. 

> 이처럼 직접 메모리를 관리하는 것은 프로그래머의 실수를 유도하기 때문에 C++ 이후의 언어는 포인터의 개념이 없다고 하네요.
{: .prompt-tip }

<br>

## GC 수집 방법
그렇다면 GC는 어떻게 미사용인 오브젝트를 선별할까요? 언리얼은 일반적으로 자주 사용하는 `Mark-Sweap` 방식을 사용하는데요. 아래와 같은 순서로 미사용 오브젝트를 찾아내 메모리를 회수하는 작업을 진행합니다.

1. 동적 오브젝트의 정보가 저장된 저장소에서 최초 탐색을 시작하는 루트 오브젝트를 표기
2. 루트 오브젝트가 참조하는 객체를 찾아 `마크(mark)`하며, 이 과정을 반복
3. 마크되지 않은 객체들의 메모리를 `회수(sweap)`

<br>

## GC의 호출 주기
GC는 일정 주기마다 자동으로 실행되어 객체들을 검사하고 수집합니다. 따라서 개발자가 딱히 관여할 일은 없지만, GC 동작 방식이 궁금하다거나 메모리가 부족한 경우에는 GC를 의도적으로 동작시킬 수 있습니다.

```cpp
bool UObject::ConditionalBeginDestroy()
{
	check(IsValidLowLevel());
	if( !HasAnyFlags(RF_BeginDestroyed) )
	{
		SetFlags(RF_BeginDestroyed);
#if !(UE_BUILD_SHIPPING || UE_BUILD_TEST)
		checkSlow(!DebugBeginDestroyed.Contains(this));
		DebugBeginDestroyed.Add(this);
#endif

#if PROFILE_ConditionalBeginDestroy
		double StartTime = FPlatformTime::Seconds();
#endif

		BeginDestroy();
  }
}
```

제일 쉬운 방법은 `UObject::ConditionalBeginDestroy()`나 `AActor::Destroy()` 함수를 호출하여 객체의 소멸 과정을 시작하는 방법입니다. 위 코드는 `UObject::ConditionalBeginDestroy()`의 일부분인데요. `SetFlags`를 통해서 파괴 시작이라는 마크를 남기고, 파괴를 시작하는 것을 볼 수 있습니다.

<br>

```cpp
void UEngine::ForceGarbageCollection(bool bForcePurge/*=false*/)
{
	TimeSinceLastPendingKillPurge = 1.0f + GetTimeBetweenGarbageCollectionPasses();
	bFullPurgeTriggered = bFullPurgeTriggered || bForcePurge;

#if UE_ENABLE_LOG_STACK_ON_FORCE_GC
	if (GLogStackOnForceGC)
	{
		UE_LOG(LogOutputDevice, Log, TEXT("ForceGarbageCollection called, logging stack"));
		FDebug::DumpStackTraceToLog(ELogVerbosity::Log);
	}
#endif
}
```

그게 아니고 GC를 강제로 수동 호출하고 싶을 때는 `ForceGarbageCollection(bool bFullPurge)` 함수를 사용하면 됩니다.

<br>


![GC 설정](/assets/img/post/GC/GC_Setting.png){: width="546" height="417"}
*GC 설정*

추가로 `Project Settings/Engine/Garbage Collection`에서 GC와 관련된 주기와 같은 설정을 조정할 수 있습니다.

<br>

## GC 사용 규칙
그렇다면 이 GC를 사용하기 위해서 우리가 해야할 일은 뭘까요? 언리얼 오브젝트를 관리할 때의 경우와 일반 오브젝트일 때의 경우 2가지로 나누어서 간단하게 설명하겠습니다.

<br>

### 1. 언리얼 클래스의 경우
```cpp
UPROPERTY()
TObjectPtr<class AActor> OwnerActor;
```

오브젝트(UObject)를 멤버 변수로 가질 경우, 그리고 이 멤버 변수가 소유자의 생명 주기와 동일해야한다면 `UPROPERTY` 매크로를 붙여주어야 합니다. GC에게 나를 등록하라고 요청하면 되죠. (이는 `Reflection System`과도 연결되어 있는데요. 이 내용은 추후에 다뤄보겠습니다.)

그 외에 `int, float` 와 같은 보통의 멤버 변수에 대해서는 별도로 관리해 줄 필요가 없습니다.

<br>

### 2. 일반 클래스의 경우
```cpp
// header
class 프로젝트_API FCustomActor : FGCObject
{
public:
	FCustomActor(class UCustomActor* InActor) : SafeActor(InActor){}

	virtual void AddReferencedObjects(FReferenceCollector& Collector) override;
	virtual FString GetReferencerName() const override {
		return TEXT("FCustomActor");
	}
	 
	const class UCustomActor* GetCustomActor() const {return SafeActor;}
  
private:
	class UCustomActor* SafeActor = nullptr;
};

// C++
void FCustomActor::AddReferencedObjects(FReferenceCollector& Collector)
{
	if (SafeActor->IsValidLowLevel())
	{
		Collector.AddReferencedObject(SafeActor);
	}
}
```

일반 C++ 클래스에서 언리얼 오브젝트를 멤버 변수로 사용할 때는 `UPROPERTY` 를 사용할 수 없습니다. 그렇기 때문에 GC를 사용하기 위해서는 `FGCObject`를 상속받고, `AddReferencedObjects`, `GetReferencerName`함수를 재정의 해야합니다.

- ***AddReferencedObjects*** : GC의 관리를 원하는 오브젝트를 추가
- ***GetReferencerName*** : 레퍼런스의 이름을 지정
    
<br>

## 마무리
언리얼 엔진의 GC는 `UObject` 기반 객체들의 생명 주기를 자동으로 관리해주며, 스마트 포인터는 일반 C++ 객체의 수명을 안전하게 관리합니다. `UObject`를 다룰 때는 반드시 GC가 추적할 수 있도록 `UPROPERTY()`와 동시에 `TObjectPtr, TWeakObjectPtr` 등과 같은 스마트 포인터를 사용하는 것이 좋겠습니다.