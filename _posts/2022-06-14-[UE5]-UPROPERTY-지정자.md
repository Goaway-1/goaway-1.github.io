---
title: UPROPERTY 매크로
description: UPROPERTY 매크로의 사용 이유에 대해서 공유하고, 프로퍼티 지정자를 통한 기능들에 대해서 살펴봅니다.
date: 2022-06-14 16:44:20 +0900
categories: [UnrealEngine, Analysis]
tags: [UnrealEngine]
---

## UPROPERTY란

먼저 `UPROPERTY`란 클래스의 멤버 변수 앞에 붙는 __Reflection 매크로입니다.__ 이는 변수 상단에 추가해주면 런타임 때, 자기 자신을 조사하라는 것을 명시하며, 아래 코드처럼 매크로에 프로퍼티 지정자를 추가하여 다양한 기능들을 추가하여 용도에 맞게 활용할 수 있습니다.

```cpp
UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Main")
AActor* PointA;

UPROPERTY(EditDefaultsOnly, Category="Main | Sub")
float DistanceValue;
```

이렇게 매크로가 선언된 변수는 __Garbage Collection(GC)에__ 의해 생명주기가 자동으로 관리되며, Reflection을 통해서 런타임 중에 변수의 이름, 유형과 같은 정보를 확인할 수 있습니다. 

> 네이티브 C++가 아닌, `언리얼 스크립트`로 작성된 것이나, `포인터 변수`들은 매크로를 추가하여 GC를 통해 생명주기가 관리되는 것이 좋습니다.
{: .prompt-tip }

<br>

## 기능

해당 매크로는 다양한 기능들을 제공하는데요. 제 경험을 바탕으로 대표적으로 사용하는 `변수 공개/수정 권한`, `블루프린트 공개/수정 권한`, `카테고리`, `메타 데이터` 총 4가지에 대해서 간단하게 설명해보겠습니다.

<br>

### 변수 공개/수정 권한
 
말 그대로 변수에 대한 에디터에서의 접근 권한을 설정하는 것입니다. 가장 많이 사용하는 매크로가 아닌가 싶은데요. __"수정 권한"(Visible, Edit) + "접근 권한"(InstanceOnly, DefaultOnly, Anywhere)__ 과 같은 형태로 구성되어 있기 때문에 사용하고자 하는 형식대로 사용하면 되겠습니다. 

예를 들어서 나는 에디터에서 수정은 불가능하게 하고 인스턴스에서만 보고 싶다고 하면, `VisibleInstanceOnly`가 되겠죠? 또, 수정이 가능하게 하고 원본에서만 수정하고 싶어라고 하면, `EditDefaultOnly`로 지정하시면 되겠습니다.

```cpp
// 인스턴스 객체에 대해서만 접근 가능
UPROPERTY(VisibleInstanceOnly)
float DistanceValue;

// 객체에 대해서만 수정 가능
UPROPERTY(EditDefaultOnly)
FVector ForwardVector;
```

<br>

### 블루프린트 공개/수정 권한
  
이는 C++ 멤버 변수에 대해서 블루프린트에게 접근 권한을 설정하는 매크로입니다. 오직 C++로만 작성하겠다면, 필요없지만, 블루프린트로 로직을 작성하는 경우에 대해서는 필요하죠? 아니면, 간단하게 접근 권한 열어두고 BP에서 로그 찍을 때도 많이 사용하긴 합니다.

|기능|내용|
|:--:|:--|
|BlueprintReadOnly|블루프린트에서 변수 읽기 가능|
|BlueprintReadWrite|블루프린트에서 변수 읽기/쓰기 가능|
|BlueprintGetter|해당 변수에 접근 가능한 함수를 지정하고 이를 통해 접근 (get을 커스텀)|
|BlueprintSetter|해당 변수에 수정 가능한 함수를 지정하고 이를 통해 수정 (set을 커스텀)|

<br>

### Category

![Category](/assets/img/post/UPROPERTY_Macro/Category.png){: width="434" height="352"}

위 그림과 같이 블루프린트 디테일 패널에서 아래 프로퍼티 `Distance`에 대해서 찾아볼 수 있는데요. 카테고리 매크로를 추가하면 변수 이름이 아닌, 카테고리를 통해서도 검색이 가능해집니다. 형식은 `Category="MainCategory | SubCategory"` 이런식으로 작성할 수 있는데, Sub는 꼭 작성하지 않아도 됩니다. 

```cpp
UPROPERTY(VisibleAnywhere, Category="Main | Sub")
float Distance;	
```

<br>

### Meta  
  
이 메타라는 카테고리는 정말 다양한 기능들을 포함하고 있습니다. 그 중에서 자주 사용하는 기능들에서만 간단히 설명하도록 하겠습니다.

#### AllowPrivateAccess

가장 먼저 에디터에서의 접근 권한을 수정하는 기능입니다. 위에서 설명했듯이 UPROPERTY에 EditAnywhere와 같은 프로퍼티 지정자를 지정하면, 에디터에서 접근 및 수정이 가능하다고 했죠? 하지만 프로퍼티가 private으로 지정되어 있다면, 아무리 EditAnywhere여도 접근이 불가능합니다.
  
이에 `AllowPrivateAccess` 메타 데이터를 추가하면, private 프로퍼티에 대해서 접근할 수 있는 권한이 부여됩니다. 이를 통해서 디버깅할때, 유용하게 사용할 수 있겠죠?
  
```cpp
private:
  UPROPERTY(VisibleAnywhere, meta = (AllowPrivateAccess="true"))
  bool bIsActive;
```

> private을 BP에서 수정할 수 있게 된다면, 객체 구조 프로그래밍의 캡슐화에 위배되는 행위가 될 수 있으니, `Visible`만 지정하는 것이 좋겠네요.
{: .prompt-warning }

<br>

#### ClampMin / ClampMax

![Clamp](/assets/img/post/UPROPERTY_Macro/Clamp.png){: width="986" height="110"} 좌측 그림과 같이 `float`, `int`형식의 멤버 변수에 대해서 값의 최소/최대 범위를 제한합니다. 협업에 있어 나름 정말 유용한 기능이라고 생각하는데요. 기획자나 제 3자가 해당 프로퍼티의 값을 에디터에서 수정할 때, 개발자가 지정된 값을 넘지 않게 되기 때문에 제한이 어느정도인지 제 3자가 직관적으로 보기 쉽고, 최대 값을 넘어가는 등의 예외상황이 발생하지 않게 되죠.
  
```cpp
UPROPERTY(EditAnywhere, meta = (ClampMin = "0.0", ClampMax = "100.0"))
float MaxDistance;
```

<br>

#### EditCondition

![EditCondition](/assets/img/post/UPROPERTY_Macro/EditCondition.png){: width="866" height="134"} 특정 조건에 따라 에디터에서 해당 프로퍼티의 활성화 여부를 제어합니다. 아래에서는 bEnableFeature가 true인 경우에만 해당 프로퍼티가 수정가능하도록 변경됩니다. 프로퍼티들끼리 엮여있는 경우에 유용하게 사용할 수 있겠죠? 
  
예를 들어 공중에서의 이동 여부를 불리언으로 만들어 두고, 해당 불리언이 참일 때만, "속도, 중력"와 같은 속성들을 변경할 수 있게끔 할 수 있을 겁니다.

```cpp
UPROPERTY(EditAnywhere)
bool bEnableFeature;

UPROPERTY(EditAnywhere, meta = (EditCondition = "bEnableFeature"))
float FeatureIntensity;
```

<br>

## 참조

UPROPERTY 매크로와 자주 사용하는 프로퍼티 지정자에 대해서 작성해봤습니다. 이 외에도 정말 많은 프로퍼티 지정자가 있으니까요. 더 많은 정보는 프로퍼티 공식 문서를 참조하여 확인하시면 좋겠습니다.


> 우리가 보통 클래스에 포함된 변수를 `멤버 변수`라고 부르고는 하죠? 하지만 UPROPERTY와 같은 매크로가 추가된 멤버 변수는 `프로퍼티`라고 부릅니다.
{: .prompt-warning }