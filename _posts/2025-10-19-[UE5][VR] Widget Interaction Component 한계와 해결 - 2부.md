---
layout: post
title: "[UE5][VR] Widget Interaction Component 한계와 해결 - 2부"
description: Widget Interaction Component를 사용했을 때의 문제점과 이에 대한 개선 방법
date: 2025-10-19 09:00:00 +0900
categories: [Unreal Engine, VR]
tags: [Unreal Engine, VR, XR, Oculus, MetaQuest, WidgetComponent, UI]
---
## Widget Interaction Component의 단점
이전 포스트에서 `WidgetInteractionComponent`를 사용하여 3D 월드 공간에서 2D 위젯과 상호작용할 수 있도록 해주는 원리를 살펴봤습니다. 그런데 이 컴포넌트에는 VR에서 꽤 치명적이라고 할 수 있는 단점이 하나 있습니다. 바로 어떤 컨트롤러가 위젯과 상호작용했는지 구분할 수 없다는 점입니다. 왼쪽 컨트롤러인지 오른쪽 컨트롤러인지 알 수 없죠. 이건 아래에서 더욱 자세하게 설명하겠습니다.

<br>

키보드나 마우스 입력은 단순히 UI 클릭과 같은 동작만 전달하면 됩니다. 하지만 VR 컨트롤러처럼 물리적으로 분리된 입력 장치에는 공통적으로 `진동` 이라는게 있습니다. 이는 유저의 입력이 정상적으로 처리되었음을 즉각적으로 느낄 수 있게 해주는 중요한 수단입니다. 

만약 내가 왼손 컨트롤러로 위젯을 클릭했는데, UI 쪽에서 컨트롤러 구분을 알 수 없어 진동을 주지 못한다면 유저는 상호작용이 제대로 되었는지 체감하기 어렵고, 이는 **몰임감이 저하**될 겁니다.

<br>

> 정확히 말하면, VR에 국한된 문제라기 보다는 2개 이상의 컨트롤러로 UI와 상호작용할 때 발생하는 문제라고 할 수 있습니다.
{: .prompt-tip }

<br>

## 왜 알 수 없을까?
사실 이건 `WidgetInteractionComponent`의 문제라기 보다는 언리얼 엔진의 위젯 `Slate`의 고질적인 문제라고 생각합니다. 고질적이라는게 맞는 표현인지는 모르겠지만, 언리얼 엔진에서 보통 PC 게임이나 모바일 게임에 대해서 고려하고 엔진을 설계했을텐데요. 아시다싶이 PC나 모바일은 딱히 피드백이라는게 없죠? 그렇기에 버튼과 같은 위젯과 상호작용한 컨트롤러의 데이터 또한 전달할 필요가 없습니다. PC는 해봤자 마우스 또는 키보드 일테고, 모바일은 터치일뿐일테니까요.

하지만 Meta Quest와 같은 VR 기기들은 보통 컨트롤러가 2개 존재하기 때문에 버튼과 상호작용 했을 때, 어떤 컨트롤러가 상호작용했는지 알 수 없습니다. 버튼을 누른 경우에는 입력을 통해서 어찌저찌 구분할 수 있겠지만, Hover와 같은 이벤트는 글쎄요... 쉽게 구분할 수 없을 듯하네요. 

<p>
	<img src="./assets/img/post/WidgetInteractionComponent/Button_Events.png" alt="Floor" style="width:550px;">
	<figcaption style="text-align: center; font-size: 0.8em; color: #555;">
		파라미터가 없는 Button의 Event들
	</figcaption>
</p>

위는 버튼 위젯의 이벤트들인데요. 모두 전달되는 파라미터가 없는 것을 볼 수 있죠.

```cpp
// SButton.h
SLATE_EVENT( FOnClicked, OnClicked )
SLATE_EVENT( FSimpleDelegate, OnPressed )
SLATE_EVENT( FSimpleDelegate, OnReleased )
SLATE_EVENT( FSimpleDelegate, OnHovered )
SLATE_EVENT( FSimpleDelegate, OnUnhovered )

// SlateDelegates.h
DECLARE_DELEGATE_RetVal(FReply, FOnClicked)

// Delegate.h
DECLARE_DELEGATE( FSimpleDelegate );
```

혹시나 slate에는 구현되어 있을까요? 코드는 버튼에서 사용하는 이벤트들로 역시 파라미터가 없는 것을 볼 수 있습니다.

<br>


## 다른 게임에서는 어떻게 풀어낼까?

그래서 제가 다른 게임들을 해보면서 비교해봤는데, 보통 아래와 같은 전략을 사용하는 것을 확인했습니다.

1. 동시에 두 컨트롤러로 UI 조작을 하지 못하게 제한
2. UI를 특정 컨트롤러(왼손 혹은 오른손)에 고정시켜서, 진동을 줄 대상 지정

이런 방식은 유저의 편의성은 조금 떨어지더라도, 진동 피드백의 일관성을 보장할 수 있습니다. **하지만, 이게 싫어서 제가 했던 방법들을 공유해보겠습니다!**

<br>

## 개선 방법들

### 1. 위젯 깊이 탐색

가장 먼저 생각해낸 방법으로, 현재 __Hover__ 하고 있는 위젯을 가져와서 계층 구조에 있는 모든 위젯을 찾아서 내가 원하는 위젯이 있다면, 피드백을 주는 방법입니다. 

```cpp
const FWeakWidgetPath& WeakWidgetPath = WidgetInteractionComponent->GetHoveredWidgetPath();
const FWidgetPath& WidgetPath = WeakWidgetPath.ToWidgetPath();
UWidgetComponent* WidgetComp = HandWidgetInteractionComponent->GetHoveredWidgetComponent();
if(!WidgetComp)
{
	return;
}
UUserWidget* RootWidget = WidgetComp->GetWidget(); // 루트 위젯을 참조
for (int32 i = 0; i < WidgetPath.Widgets.Num(); i++)
{
	FArrangedWidget ArrangedWidget = WidgetPath.Widgets[i];
       
	if (ArrangedWidget.Geometry.IsUnderLocation(HandWidgetInteractionComponent->Get2DHitLocation()))
	{
		TArray<UWidget*> AllWidgets;
		RootWidget->WidgetTree->GetAllWidgets(AllWidgets);
		
		for (UWidget* Widget : AllWidgets)
		{
			if(Widget)
			{
				// 내가 원하는 위젯을 찾고, 진동 피드백 진행
			}
		}
	}
}
```

입력과 동시에 `WidgetInteractionComponent`와 __Hover__ 하고 있는 위젯의 경로 (`FWidgetPath`)를 통해서 최상위 위젯을 가져오고, 모든 위젯을 순회하면서 탐색합니다. 모든 위젯에서 내가 원하는 위젯이 있다면 (여기서는 버튼으로 하겠습니다.) 해당 컨트롤러에 진동과 같은 피드백을 제공하면 됩니다. 

<br>

모든 위젯을 순회하면서 찾는다는게 단순하지만, `WidgetTree`의 모든 위젯을 순회하므로 계산량이 많다는 게 가장 큰 단점이죠. 위젯 구조가 복잡할수록 성능에 더 큰 부담이 생길 수 밖에 없습니다. 만약 이게 `Click`이 아니라 `Hover` 이벤트를 처리한다고 생각해보면, Tick마다 진행되어야겠죠? 아찔합니다. 

버튼뿐만 아니라 다른 위젯에도 피드백을 추가했다고 가정해봅시다. 만약 현재 위젯 경로에 버튼도 있고, 다른 위젯도 있는 경우에는 어떤 위젯에 대해서 피드백을 우선적으로 제공해야할까요? 그리고 버튼이라고 해서 진동을 주고 싶지 않은 경우는 또 어떨까요? 이런식으로 많은 예외가 있고, 고려되어야 할게 많은 방법 중 하나입니다.

<br>

### 2. ModifierKey 응용

다음은 `WidgetInteractionComponent`를 확장하고, 기존 위젯 이벤트에 대해서 파라미터를 추가하여 확장하는 방법입니다.

#### 1. WidgetInteractionComponent의 PointerIndex 수정
```cpp
void UWidgetInteractionComponent::SimulatePointerMovement()
{
	if(!bEnableHitTesting) return;
	if(!CanSendInput()) return;
	
	FWidgetPath WidgetPathUnderFinger = DetermineWidgetUnderPointer();
	ensure(PointerIndex >= 0);
	FPointerEvent PointerEvent(VirtualUser->GetUserIndex(), (uint32)PointerIndex, LocalHitLocation, LastLocalHitLocation,	PressedKeys,	FKey(),	0.0f,	ModifierKeys);
	
	if (WidgetPathUnderFinger.IsValid())
	{
		check(HoveredWidgetComponent);
		LastWidgetPath = WidgetPathUnderFinger;		
		FSlateApplication::Get().RoutePointerMoveEvent(WidgetPathUnderFinger, PointerEvent, false);
	}
	else
	{
		FWidgetPath EmptyWidgetPath;
		FSlateApplication::Get().RoutePointerMoveEvent(EmptyWidgetPath, PointerEvent, false);
		LastWidgetPath = FWeakWidgetPath();
	}
}
```

Tick 마다 포인터의 움직임을 시뮬레이트하는 함수인 `UWidgetInteractionComponent::SimulatePointerMovement` 를 보면, 상호작용 하는 위젯에 대하여 `FPointEvent`를 전달하는 것을 볼 수 있는데요. 이는 현재 입력 정보들인  Alt, Shift를 눌렀는지, 충돌이 발생한 지점은 어디인지 등등 말이죠. 여기서 눈여겨 봐야할 것은 `PointerIndex`입니다. 주석을 보면, "<u>Each user virtual controller or virtual finger tips being simulated should use a different pointer index.</u>"라고 적혀있는데, 가상 컨트롤러를 구분하기 위해서 사용하는 것을 알 수 있습니다.

이 값을 응용해서 왼쪽 컨트롤러는 1, 오른쪽 컨트롤러는 0을 전달한다고 규칙을 세우고, 2개의 `WidgetInteractionComponent`의 `PointerIndex`에 각각 값을 지정해줍니다.

<br>

#### 2. 델리게이트 이벤트 확장

```cpp
// 기존 Button.h
DECLARE_DYNAMIC_MULTICAST_DELEGATE(FOnButtonClickedEvent);
DECLARE_DYNAMIC_MULTICAST_DELEGATE(FOnButtonPressedEvent);
DECLARE_DYNAMIC_MULTICAST_DELEGATE(FOnButtonReleasedEvent);
DECLARE_DYNAMIC_MULTICAST_DELEGATE(FOnButtonHoverEvent);

// 신규 ExtendedButton.h
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnButtonClickedEvent, EControllerHand, HandType);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnButtonPressedEvent, EControllerHand, HandType);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnButtonReleasedEvent, EControllerHand, HandType);
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnButtonHoverEvent,EControllerHand, HandType);
```

기존 버튼을 확장해서 `EControllerHand`를 파라미터로 받는 이벤트로 만들어줍니다.

<br>

#### 3. Slate에서 FPointEvent를 통한 이벤트 호출

```cpp
// 기존 SButton.cpp
FReply SButton::ExecuteOnClick()
{
	if (OnClicked.IsBound())
	{
		FReply Reply = OnClicked.Execute();
#if WITH_ACCESSIBILITY
		// @TODOAccessibility: This should pass the Id of the user that clicked the button but we don't want to change the regular Slate API just yet
		FSlateApplicationBase::Get().GetAccessibleMessageHandler()->OnWidgetEventRaised(
			FSlateAccessibleMessageHandler::FSlateWidgetAccessibleEventArgs(AsShared(), EAccessibleEvent::Activate));
#endif
		return Reply;
	}
}

// 기존 SExtendedButton.cpp
FReply SExtendedButton::ExecuteOnHandClick(const FPointerEvent& MouseEvent)
{
	if (OnHandClicked.IsBound())
	{
		FReply Reply = OnHandClicked.Execute(GetHandType(MouseEvent));
#if WITH_ACCESSIBILITY
		// @TODOAccessibility: This should pass the Id of the user that clicked the button but we don't want to change the regular Slate API just yet
		FSlateApplicationBase::Get().GetAccessibleMessageHandler()->OnWidgetEventRaised(
			FSlateAccessibleMessageHandler::FSlateWidgetAccessibleEventArgs(AsShared(), EAccessibleEvent::Activate));
#endif
		return Reply;
	}
}

EControllerHand SExtendedButton::GetHandType(const FPointerEvent& MouseEvent)
{
	EControllerHand HandType = MouseEvent.GetPointerIndex() ? EControllerHand::Left : EControllerHand::Right;
}
```

입력이 들어오면, `FPointEvent`의 `PointerIndex`를 활용하여 컨트롤러를 구분하고, 이벤트를 호출하면 끝입니다.

<br>


#### 4. Widget에서의 이벤트 호출

```cpp
// 기존 Button.cpp
FReply UButton::SlateHandleClicked()
{
	OnClicked.Broadcast();

	return FReply::Handled();
}

// 신규 ExtendedButton.h
FReply ExtendedButton::SlateHandleClicked(EControllerHand HandType)
{
	OnClicked.Broadcast(HandType);

	return FReply::Handled();
}
```

이제 `Slate`가 아닌, `Widget`에서도 동일하게 작업해줍니다.

<br>

> 작업에 필요한 "Button, SButton, ButtonSlot"는 직접 만드는 것을 추천 드립니다. 기존 엔진 코드는 수정하지 않는게 최고입니다.
{: .prompt-tip }


#### 5. 결과
<p>
	<img src="./assets/img/post/WidgetInteractionComponent/ExtendedButton_Events.png" alt="Floor" style="width:550px;">
	<figcaption style="text-align: center; font-size: 0.8em; color: #555;">
		확장한 ExtendedButton의 Event들
	</figcaption>
</p>

이제 기존 `Button`이 아닌, `ExtendedButton`을 사용하면 이벤트를 받는 부분에서 컨트롤러를 구분할 수 있습니다. 이렇게 사용하면, 각 버튼마다 원하는 컨트롤러에 대해서 피드백을 잘 전달할 수 있죠.

<br>

## 마무리

이렇게 각 컨트롤러에 대한 피드백을 효과적으로 전달하기 위해서 `WidgetInteractionComponent`와 `Button`을 확장해 새로운 `ExtendedButton`을 만들어보았습니다. 에픽게임즈에서 VR 환경도 고려해서 모든 위젯을 확장해줬으면 좋겠다는 생각이 듭니다.