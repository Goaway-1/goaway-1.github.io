---
layout: post
title: "[UE5] IsValid vs IsValidLowLevel"
description: 유효성 검사에 사용하는 함수들의 차이 구분
date: 2025-08-05 09:00:00 +0900
categories: [Unreal Engine, Analysis]
tags: [Unreal Engine, IsValid, IsValidLowLevel]
---

## 들어가며
여러분들은 개발을 진행하면서 유효성 검사를 많이 사용하시나요? 보통 프로그램과는 다르게 게임은 많은 객체들이 생성되고 파괴되는 동적인 부분이 많죠. 그렇기에 접근하거나 관리하기 위해서는 객체의 상태, 즉 유효한 상태인지 검사하는 것은 필수가 됩니다.

언리얼에서는 대표적으로 `"IsValid, IsValidLowLevel"`과 같은 유효성 검사 함수나, `"Ensure, Check, Verify"`와 같은 `Assertion` 매크로를 사용하여 유효성 검사를 진행하고는 합니다. 특히 포인터를 다룰 때 자주 사용하게 되죠.

이번 포스트에서는 각각의 검사 방법이 `"무엇을 확인하는지, 언제 사용하는 것이 적절한지"`를 코드와 함께 살펴보고, 차이점을 정리해보겠습니다.

<br>

---

## 유효성 검사

### 1. IsValid

```c++
FORCEINLINE bool IsValid(const UObject *Test)
{
	return Test && FInternalUObjectBaseUtilityIsValidFlagsChecker::CheckObjectValidBasedOnItsFlags(Test);
}
```
가장 일반적으로 사용되는 유효성 검사인 `IsValid`입니다. 이 내부를 보면, 딱 2개의 조건만 검사하는 것을 볼 수 있죠. 일단 객체가 `nullptr`인지 판단하고, 객체가 플래그를 가지고 있는지 검사합니다. 어떤 플래그인지 함수를 확인해 봅시다.

<br>

```c++
struct FInternalUObjectBaseUtilityIsValidFlagsChecker
{
	FORCEINLINE static bool CheckObjectValidBasedOnItsFlags(const UObject* Test)
	{
		// Here we don't really check if the flags match but if the end result is the same
		checkSlow(GUObjectArray.IndexToObject(Test->InternalIndex)->HasAnyFlags(EInternalObjectFlags::Garbage) == Test->HasAnyFlags(RF_MirroredGarbage));
		return !Test->HasAnyFlags(RF_MirroredGarbage);
	}
};
```
전체적으로 보면, `RF_MirroredGarbage`플래그 여부를 확인하는 것을 볼 수 있습니다. GC에 의해 파괴 대상으로 지정된 객체를 식별하는 플래그이죠. 자세히 살펴봅시다.

가장 먼저 넘겨진 객체의 인덱스를 기반으로 `언리얼의 모든 객체 배열(GUObjectArray)`에서 해당 `UObject`를 찾고, `EInternalObjectFlags::Garbage`라는 플래그를 가지는지 확인합니다. 그리고 직접 플래그를 조사해서 서로 동기화된 상태인지 체크합니다.

그 후에 스스로를 조사해서 `RF_MirroredGarbage`비트 플래그를 소유하고 있는지 여부를 반환합니다.

<br>

> 근데 왜 스스로 조사하지 않고, 왜 GUObjectArray와 비교할까요? 왜 동기화 여부를 확인하려고 하는 걸까요?
{: .prompt-warning }

<br>

```c++
// ObjectMacros.h
enum class EInternalObjectFlags : int32
{
    Garbage = 1 << 21,
};

enum EObjectFlags
{
    RF_MirroredGarbage = 0x40000000,
};
```
이 외에도 `ObjectMacros.h`에서 `EInternalObjectFlags와 EObjectFlags`열거형을 통해서 플래그를 확인할 수 있으니 한번 확인해보는 것도 좋을 것 같네요.

<br>

---

### 2. IsValidChecked

```c++
FORCEINLINE bool IsValidChecked(const UObject* Test)
{
	check(Test);
	return FInternalUObjectBaseUtilityIsValidFlagsChecker::CheckObjectValidBasedOnItsFlags(Test);
}
```
`IsValid` 와 차이점은 Assert인 check밖에 없죠? 디버깅 시 **즉시 예외를 발생**시켜 문제를 조기에 포착하는 용도로 유용합니다.

<br>

---

### 3. IsValidLowLevel

```c++
bool UObjectBase::IsValidLowLevel() const
{
	return IsValidLowLevelForDestruction() && GUObjectArray.IsValid(this);
}
```
함수명만 봤을 때는 `IsValid`보다 더 꼼꼼히 따지는 함수인가? 했는데, `LowLevel`에서 검사하고, `GUObjectArray`에서 해당 오브젝트가 유효한지 검사를 진행하네요.


```c++
bool UObjectBase::IsValidLowLevelForDestruction() const
{
	if (this == nullptr)
	{
		UE_LOG(LogUObjectBase, Warning, TEXT("NULL object"));
		return false;
	}
	if (!ClassPrivate)
	{
		UE_LOG(LogUObjectBase, Warning, TEXT("Object is not registered"));
		return false;
	}
	return true;
}
```
객체가 `nullptr`인지 판단하고, 객체의 `ClassPrivate` 즉, 어떤 클래스의 인스턴스인지에 대한 정보를 확인합니다. 이전 함수들과는 다르게 `GC`와 관련된 부분이 보이지는 않는 것을 볼 수 있죠. 조금 더 볼까요?

<br>

```c++
bool FUObjectArray::IsValid(const UObjectBase* Object) const 
{ 
	int32 Index = Object->InternalIndex;
	if( Index == INDEX_NONE )
	{
		UE_LOG(LogUObjectArray, Warning, TEXT("Object is not in global object array") );
		return false;
	}
	if( !ObjObjects.IsValidIndex(Index))
	{
		UE_LOG(LogUObjectArray, Warning, TEXT("Invalid object index %i"), Index );
		return false;
	}
	const FUObjectItem& Slot = ObjObjects[Index];
	if( Slot.Object == NULL )
	{
		UE_LOG(LogUObjectArray, Warning, TEXT("Empty slot") );
		return false;
	}
	if( Slot.Object != Object )
	{
		UE_LOG(LogUObjectArray, Warning, TEXT("Other object in slot") );
		return false;
	}
	return true;
}
```
`IsValidLowLevelForDestruction` 코드와 비슷하네요. `GUObjectArray`에 해당 객체의 Index가 있는지 유효한지를 검사합니다. 하지만 이 역시 `IsValid`와 동일한 이름을 가지고 있지만, 여기서도 GC와 관련된 부분은 보이지 않다는 것이 중요한 부분입니다.

<br>

---

## 마무리
정리하자면, `IsValid()`는 `GC`상태를 판별하는 "게임 로직 관점의 유효성 검사"이고, `IsValidLowLevel()`는 해당 오브젝트의 내부 구조가 손상되지 않았는지를 확인하는 "저수준 안정성 검사"라고 정리할 수 있겠습니다.

보통의 경우에는 `IsValid()`만으로 충분하겠습니다. 혹시 `IsValidLowLevel()`을 실무에서 명확히 사용하는 상황에 대해서 알고 있다면 공유해주세요!