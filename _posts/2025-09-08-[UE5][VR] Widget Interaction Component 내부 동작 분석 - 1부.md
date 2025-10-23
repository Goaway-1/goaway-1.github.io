---
layout: post
title: "[UE5][VR] Widget Interaction Component 내부 동작 분석 - 1부"
description: VR에서 UI와 상호작용하기 위해 사용하는 Widget Interaction Component의 내부 동작 원리를 분석
date: 2025-09-08 09:00:00 +0900
categories: [Unreal Engine, VR]
tags: [Unreal Engine, VR, XR, Oculus, MetaQuest, WidgetComponent, UI, Slate]
---
## VR에서의 위젯
PC에서는 마우스와 키보드를 통해 UI를 조작합니다. "옵션 창, 메인 화면, 캐릭터 선택 창" 등 대부분은 마우스로 클릭하는 방식이죠. 그렇다면 VR에서는 어떻게 UI를 조작할까요?

![Meta Quest의 UI](/assets/img/post/WidgetInteractionComponent/MetaQuest_Lobby.jpg){: width="499" height="280"}
*Meta Quest의 UI*

PC에서는 UI를 화면 평면에 그리지만, VR에서는 보통 UI를 월드 공간에 배치합니다. 눈 바로 앞에 고정된 UI는 컨트롤러로 조작하기 불편하고, HMD를 쓰고 고개를 돌릴 때 UI가 계속 눈앞을 따라오면 시야가 가려져 답답함을 느끼게 됩니다. Meta Quest의 인터페이스처럼, 일정 거리 앞의 월드 공간에 UI를 띄우는 방식은 이런 불편을 줄이기 위해 고안된 설계이지 않을까 싶습니다.

<br>

## WidgetInteractionComponent

언리얼 엔진에서는 3D 공간에서 2D UI와 상호작용할 수 있도록 `WidgetInteractionComponent`를 제공합니다. 이 컴포넌트는 VR 환경에서 마우스 포인터 역할을 대신하는 장치라고 이해하면 쉽습니다. 컨트롤러 위치에서 **라인 트레이스(LineTrace)** 를 쏘아 UI 위젯을 찾아내고, 그 결과를 Slate 이벤트로 변환하여 전달합니다. 덕분에 VR에서도 기존의 UMG 버튼, 슬라이더 같은 위젯들이 그대로 동작할 수 있죠.

이제 동작 원리를 이해하기 위해서 코드 내부를 살짝 살펴봅시다.

<br>

## 동작 원리

### Hover 동작 원리

#### 1. 입력 전달
```cpp
void UWidgetInteractionComponent::SimulatePointerMovement()
{
	...	
	
	FWidgetPath WidgetPathUnderFinger = DetermineWidgetUnderPointer();

	ensure(PointerIndex >= 0);
	FPointerEvent PointerEvent(VirtualUser->GetUserIndex(), (uint32)PointerIndex, 
	  LocalHitLocation,	LastLocalHitLocation,	PressedKeys, FKey(),	0.0f,	ModifierKeys);
	
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

`UWidgetInteractionComponent`는 매 프레임 `TickComponent`에서 `SimulatePointerMovement`를 호출하는데요. 여기서 현재 포인터 아래에 있는 위젯의 경로인 `FWidgetPath`를 기반으로 `FPointerEvent`를 생성해 Slate에 전달합니다.

<br>

#### 2. 데이터의 검사 및 처리

```cpp
// FSlateApplication::RoutePointerMoveEvent()
FEventRouter::Route<FNoReply>(this, FEventRouter::FBubblePolicy(WidgetsUnderPointer), TransformedPointerEvent, [&MouseCaptorPath, &LastWidgetsUnderPointer](const FArrangedWidget& WidgetUnderCursor, const FPointerEvent& Event)
{
	if (!LastWidgetsUnderPointer.ContainsWidget(WidgetUnderCursor.GetWidgetPtr()))
	{
		if (MouseCaptorPath.ContainsWidget(WidgetUnderCursor.GetWidgetPtr()))
		{
			WidgetUnderCursor.Widget->OnMouseEnter(WidgetUnderCursor.Geometry, Event);
#if WITH_SLATE_DEBUGGING
			FSlateDebugging::BroadcastNoReplyInputEvent(ESlateDebuggingInputEvent::MouseEnter, &Event, WidgetUnderCursor.Widget);
#endif
		}
	}
	return FNoReply();
}, ESlateDebuggingInputEvent::MouseEnter);
```

`FSlateApplication::RoutePointerMoveEvent`는 아까의 정보를 받아서 현재 포인터 아래에 어떤 위젯들이 있는지 확인합니다. 그리고 이전 프레임에 Hover 중이던 위젯과 비교하여, 새로 들어간 위젯에는 OnMouseEnter, 빠져나간 위젯에는 OnMouseLeave와 같은 이벤트를 보냅니다. 이 과정이 저희가 흔히 알고 있는 위젯 이벤트가 호출되는 과정입니다. 

<br>

### Click 동작 원리

```cpp
// UWidgetInteractionComponent.cpp
void UWidgetInteractionComponent::PressPointerKey(FKey Key)
{
	...
	FReply Reply = FSlateApplication::Get().RoutePointerDownEvent(WidgetPathUnderFinger, PointerEvent);
	...
}

// SlateApplication.cpp
FReply FSlateApplication::RoutePointerDownEvent(const FWidgetPath& WidgetsUnderPointer, const FPointerEvent& PointerEvent)
{
	...
	Reply = FEventRouter::Route<FReply>( this, FEventRouter::FBubblePolicy( WidgetsUnderPointer ), TransformedPointerEvent, [this]( const FArrangedWidget TargetWidget, const FPointerEvent& Event )
	{
		FReply TempReply = FReply::Unhandled();
		if( !TempReply.IsEventHandled() )
		{
			if( Event.IsTouchEvent() )
			{
				TempReply = TargetWidget.Widget->OnTouchStarted( TargetWidget.Geometry, Event );
#if WITH_SLATE_DEBUGGING
				FSlateDebugging::BroadcastInputEvent(ESlateDebuggingInputEvent::TouchStart, &Event, TempReply, TargetWidget.Widget);
#endif
			}
			if( !Event.IsTouchEvent() || ( !TempReply.IsEventHandled() && this->bTouchFallbackToMouse ) )
			{
				TempReply = TargetWidget.Widget->OnMouseButtonDown( TargetWidget.Geometry, Event );
#if WITH_SLATE_DEBUGGING
				FSlateDebugging::BroadcastInputEvent(ESlateDebuggingInputEvent::MouseButtonDown, &Event, TempReply, TargetWidget.Widget);
#endif
			}
		}
		return TempReply;
	}, ESlateDebuggingInputEvent::MouseButtonDown);
	...
}
```

`Click`이벤트는 `Hover` 이벤트와 굉장히 유사합니다. 컨트롤러의 입력을 통해서 `PressPointerKey`가 호출되면 현재 포인터 아래 위젯의 경로와 입력 정보를 가져와 `RoutePointerDownEvent`로 전달합니다. 이후 `"MouseButtonDown"`과 같은 이벤트를 전달해줍니다.

<br>

### 위젯 탐색

```cpp
UWidgetInteractionComponent::FWidgetTraceResult UWidgetInteractionComponent::PerformTrace() const
{
	FWidgetTraceResult TraceResult;
	TArray<FHitResult> MultiHits;

	switch( InteractionSource )
	{
		case EWidgetInteractionSource::World:
		{
			const FVector WorldLocation = GetComponentLocation();
			const FTransform WorldTransform = GetComponentTransform();
			WorldDirection = WorldTransform.GetUnitAxis(EAxis::X);

			TArray<UPrimitiveComponent*> PrimitiveChildren;
			GetRelatedComponentsToIgnoreInAutomaticHitTesting(PrimitiveChildren);

			FCollisionQueryParams Params(SCENE_QUERY_STAT(WidgetInteractionComponentTrace));
			Params.AddIgnoredComponents(PrimitiveChildren);

			TraceResult.LineStartLocation = WorldLocation;
			TraceResult.LineEndLocation = WorldLocation + (WorldDirection * InteractionDistance);

			GetWorld()->LineTraceMultiByChannel(MultiHits, TraceResult.LineStartLocation, TraceResult.LineEndLocation, TraceChannel, Params);
			break;
		}
	}

	...
}
```

그렇다면 어떻게 현재 포인터 아래에 어떤 위젯이 있는지 찾을까요? `PerformTrace`를 보면, 컴포넌트 위치에서부터 InteractionDistance만큼 라인트레이스를 쏘아 충돌한 `WidgetComponent`를 찾습니다. 그리고 충돌 지점을 기준으로 로컬 좌표나 위젯의 경로등을 얻습니다. 

<br>

### 요약
`WidgetInteractionComponent`컴포넌트를 이용한 VR 환경에서의 UI 입력은 정리해보자면 아래와 같은 단계로 이루어집니다.

1. 컴포넌트 위치에서부터 InteractionDistance만큼 라인 트레이스 실행
2. 충돌한 WidgetComponent를 찾고, 로컬 좌표나 WidgetPath 계산
3. Slate에 이벤트(RoutePointerMoveEvent, RoutePointerDownEvent 등) 전달
4. Slate에서 OnMouseEnter, OnMouseButtonDown 같은 이벤트 호출

이 과정을 통해 VR에서도 별도의 UI 시스템을 새로 만들 필요 없이, 기존 언리얼 UMG 시스템을 그대로 사용할 수 있게 됩니다.

<br>

## 마무리
앞서 WidgetInteractionComponent가 3D 월드 공간에서 2D 위젯과 상호작용할 수 있도록 해주는 원리를 살펴봤습니다. 그런데 이 컴포넌트에는 VR에서 꽤 치명적이라고 할 수 있는 단점이 하나 있는데요. 
이건 다음 포스트로 작성해보겠습니다!