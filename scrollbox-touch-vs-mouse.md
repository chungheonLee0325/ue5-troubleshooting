# UScrollBox의 터치와 마우스 입력 처리 차이

## 발단

[UScrollBox 안에 버튼을 배치했을 때 드래그 스크롤이 안 되는 이유](../scrollbox-button-drag.md)를 분석하면서 SScrollBox 엔진 코드를 보다 보니 이상한 점이 눈에 띄었다.

```cpp
// OnPreviewMouseButtonDown - 터치만 처리
if (MouseEvent.IsTouchEvent() && !bFingerOwningTouchInteraction.IsSet())
{
    AmountScrolledWhileRightMouseDown = 0;
    PendingScrollTriggerAmount = 0;
    // ...
}
```

`OnPreviewMouseButtonDown`? Preview가 붙은 함수는 처음 봤다. 그리고 왜 터치 입력만 여기서 처리하는 걸까? 마우스는 `OnMouseButtonDown`에서 처리하는데.

같은 "버튼 누름" 동작인데 왜 코드가 다른 함수에 나뉘어 있는지 궁금해서 더 파봤다.

---

## 핵심 차이점

터치와 마우스는 초기화 위치부터 다르다.

터치는 `OnPreviewMouseButtonDown`에서 초기화한다. 이건 Tunnel 단계로, **부모가 먼저** 이벤트를 받는다. 반면 마우스는 `OnMouseButtonDown`에서 초기화한다. 이건 Bubble 단계로, **자식이 먼저** 받는다.

```cpp
// SScrollBox::OnPreviewMouseButtonDown - 터치 전용
FReply SScrollBox::OnPreviewMouseButtonDown(const FGeometry& MyGeometry, const FPointerEvent& MouseEvent)
{
    if (bEnableTouchScrolling && MouseEvent.IsTouchEvent() && !bFingerOwningTouchInteraction.IsSet())
    {
        InertialScrollManager.ClearScrollVelocity();
        AmountScrolledWhileRightMouseDown = 0;
        PendingScrollTriggerAmount = 0;
        bFingerOwningTouchInteraction = MouseEvent.GetPointerIndex();

        Invalidate(EInvalidateWidget::Layout);
    }
    return FReply::Unhandled();
}
```

주석을 보면: **"Someone put their finger down in this list, so they probably want to drag the list."**

터치 입력은 **"아마도 스크롤하려는 것"** 으로 가정한다. 그래서 자식 위젯(버튼)보다 먼저 스크롤박스가 받아서 준비한다.

```cpp
// SScrollBox::OnMouseButtonDown - 마우스 전용
FReply SScrollBox::OnMouseButtonDown( const FGeometry& MyGeometry, const FPointerEvent& MouseEvent )
{
    if ( !bFingerOwningTouchInteraction.IsSet() )
    {
        EndInertialScrolling();
    }

    if ( MouseEvent.IsTouchEvent() )
    {
        return FReply::Handled();
    }
    else
    {
        if ( MouseEvent.GetEffectingButton() == EKeys::RightMouseButton && ScrollBar->IsNeeded() && bAllowsRightClickDragScrolling)
        {
            AmountScrolledWhileRightMouseDown = 0;
            Invalidate(EInvalidateWidget::Layout);
            return FReply::Handled();
        }
    }

    return FReply::Unhandled();    
}
```

마우스 입력은 자식 위젯이 먼저 받고, 자식이 처리 안 하면 스크롤박스가 받는다. 즉, **"버튼 클릭이 우선"** 이다.

---

## 이벤트 라우팅: Tunnel vs Bubble

Slate는 이벤트를 2단계로 처리한다.

### Tunnel 단계 (OnPreview...)

루트에서 리프 방향으로 전파된다. 부모가 먼저 이벤트를 보고, 필요하면 준비할 수 있다.

```cpp
// Engine/Source/Runtime/Slate/Private/Framework/Application/SlateApplication.cpp
class FEventRouter::FTunnelPolicy
{
public:
    FTunnelPolicy( const FWidgetPath& InRoutingPath )
    : WidgetIndex(0)  // 0번부터 시작 (최상위 부모)
    , RoutingPath(InRoutingPath)
    {
    }

    bool ShouldKeepGoing() const
    {
        return WidgetIndex < RoutingPath.Widgets.Num();
    }

    void Next()
    {
        ++WidgetIndex;  // 인덱스 증가 = 자식으로 이동
    }
    // ...
};
```

Tunnel 단계에서 `Unhandled`를 반환해도 자식이 이벤트를 받는다. 가로채는 게 아니라 "먼저 보는 것"이다.

### Bubble 단계 (일반 On...)

리프에서 루트 방향으로 전파된다. 자식이 먼저 처리하고, `Handled`를 반환하면 부모는 못 받는다.

```cpp
class FEventRouter::FBubblePolicy
{
public:
    FBubblePolicy( const FWidgetPath& InRoutingPath )
    : WidgetIndex( InRoutingPath.Widgets.Num()-1 )  // 마지막부터 (자식)
    , RoutingPath(InRoutingPath)
    {
    }

    void Next()
    {
        --WidgetIndex;  // 인덱스 감소 = 부모로 이동
    }
    // ...
};
```

---

## 왜 터치는 Preview에서 처리하도록 설계했을까

터치 입력의 특성을 생각해보면 이해가 된다.

**손가락은 마우스보다 부정확하다.** 버튼을 정확히 누르기 어렵다. 리스트가 버튼으로 꽉 차 있으면 빈 공간을 터치하기가 거의 불가능하다.

**의도가 애매하다.** 버튼 클릭인지 스크롤인지 터치 시점에는 알 수 없다. 손가락을 대고 움직이는지 안 움직이는지 봐야 판단할 수 있다.

**약간의 떨림이 불가피하다.** 완벽히 고정된 터치는 거의 없다. 그래서 임계값(`DragTriggerDistance`)을 넘어야 "스크롤 의도"로 판단한다.

그래서 스크롤박스는 터치가 들어오면 **일단 준비부터 한다**. Preview 단계에서 먼저 받아서 추적을 시작하고(`bFingerOwningTouchInteraction`), 이후 움직임을 보고 "스크롤인지 클릭인지" 판단한다.

반면 마우스는 정확한 포인팅이 가능하고 클릭 의도가 명확하니까, 자식(버튼)이 먼저 받는 게 자연스럽다.

---

## 주요 변수들의 역할

| 변수 | 역할 | 설명 |
|-----|-----|------|
| `bFingerOwningTouchInteraction` | 터치 소유권 | 어떤 손가락이 이 스크롤박스와 상호작용 중인지 추적. 멀티터치 환경에서 다른 손가락의 움직임을 무시하기 위함 |
| `PendingScrollTriggerAmount` | 터치 이동 누적 | 터치 후 움직인 거리. 임계값(`DragTriggerDistance`)을 넘으면 스크롤 모드로 전환 |
| `bTouchPanningCapture` | 스크롤 모드 플래그 | true가 되면 손가락을 떼기 전까지 스크롤만 처리 |
| `AmountScrolledWhileRightMouseDown` | 마우스/터치 공용 | 버튼 누른 동안 스크롤한 양. 변수명은 RightMouse지만 터치도 사용 |

### bFingerOwningTouchInteraction이 필요한 이유

모바일에서는 멀티터치가 가능하다. 손가락 2개로 동시에 터치할 수 있다.

```
손가락 A: 스크롤박스 터치 → bFingerOwningTouchInteraction = A
손가락 B: 다른 버튼 터치

손가락 B가 움직임 → OnMouseMove 호출
  → GetPointerIndex() == B
  → bFingerOwningTouchInteraction (A)와 다름
  → 무시
```

이 스크롤박스를 "소유"한 손가락만 스크롤을 제어할 수 있다.

### PendingScrollTriggerAmount가 필요한 이유

터치는 항상 약간 떨린다. 정확히 고정된 터치는 거의 불가능하다.

```
사용자: "버튼 클릭하려고!"
실제: 손가락 약간 떨림 (2픽셀 이동)

PendingScrollTriggerAmount = 2픽셀
DragTriggerDistance = 5픽셀 (보통)

2 < 5 → 아직 클릭으로 간주
```

임계값 이상 움직여야 "스크롤하려는 거구나"라고 판단한다.

---

## OnMouseMove에서의 분기 처리

```cpp
FReply SScrollBox::OnMouseMove( const FGeometry& MyGeometry, const FPointerEvent& MouseEvent )
{
    const float ScrollByAmountScreen = GetScrollComponentFromVector(MouseEvent.GetCursorDelta());

    if ( MouseEvent.IsTouchEvent() )
    {
        // 터치 처리
        if ( !bTouchPanningCapture )
        {
            // 아직 스크롤 모드 아님 - 임계값 체크
            if ( bFingerOwningTouchInteraction.IsSet() && !HasMouseCapture() )
            {
                PendingScrollTriggerAmount += ScrollByAmountScreen;

                if ( FMath::Abs(PendingScrollTriggerAmount) > FSlateApplication::Get().GetDragTriggerDistance() )
                {
                    // 임계값 초과 → 스크롤 모드 전환
                    bTouchPanningCapture = true;
                    ScrollBar->BeginScrolling();
                    Reply = FReply::Handled().CaptureMouse(AsShared());
                }
            }
        }
        else
        {
            // 스크롤 모드 - 실제 스크롤 수행
            ScrollBy(MyGeometry, -ScrollByAmountLocal, EAllowOverscroll::Yes, false);
        }
    }
    else
    {
        // 마우스 처리 - 우클릭 드래그
        if ( MouseEvent.IsMouseButtonDown(EKeys::RightMouseButton) && bAllowsRightClickDragScrolling )
        {
            AmountScrolledWhileRightMouseDown += FMath::Abs(ScrollByAmountScreen);

            if ( IsRightClickScrolling() )
            {
                ScrollBy(MyGeometry, -ScrollByAmountLocal, AllowOverscroll, false);
                // ...
            }
        }
    }
}
```

터치는 `PendingScrollTriggerAmount`와 `bTouchPanningCapture`로 상태를 관리하고, 마우스는 `AmountScrolledWhileRightMouseDown`과 `IsRightClickScrolling()`으로 관리한다. 비슷한 역할이지만 분리되어 있다.

---

## TouchMethod와 ClickMethod의 매핑

버튼은 터치와 마우스를 별도로 설정할 수 있다.

```cpp
// SButton.h
TEnumAsByte<EButtonClickMethod::Type> ClickMethod;  // 마우스용
TEnumAsByte<EButtonTouchMethod::Type> TouchMethod;  // 터치용
```

런타임에 자동 매핑된다.

```cpp
TEnumAsByte<EButtonClickMethod::Type> SButton::GetClickMethodFromInputType(const FPointerEvent& MouseEvent) const
{
    if (MouseEvent.IsTouchEvent())
    {
        switch (TouchMethod)
        {
            case EButtonTouchMethod::Down:
                return EButtonClickMethod::MouseDown;
            case EButtonTouchMethod::DownAndUp:
                return EButtonClickMethod::DownAndUp;
            case EButtonTouchMethod::PreciseTap:
                return EButtonClickMethod::PreciseClick;  // PreciseTap → PreciseClick
        }
    }
    return ClickMethod;
}
```

`TouchMethod::PreciseTap`은 `ClickMethod::PreciseClick`과 동일하게 동작한다. 그래서 스크롤 컨테이너 안의 버튼에는 둘 다 설정하는 게 안전하다.

```cpp
ItemButton->SetClickMethod(EButtonClickMethod::PreciseClick);
ItemButton->SetTouchMethod(EButtonTouchMethod::PreciseTap);
```

---

## 동작 흐름 비교

### 터치 입력 (PreciseClick/PreciseTap 버튼)

```
OnPreviewMouseButtonDown (Tunnel - 부모 먼저)
  스크롤박스: bFingerOwningTouchInteraction = 손가락 ID
             PendingScrollTriggerAmount = 0
             → Unhandled (자식도 받게 함)

OnMouseButtonDown (Bubble - 자식 먼저)
  버튼: Press(), Handled

OnMouseMove (10픽셀 이동)
  버튼: PreciseTap → Release()
        → Unhandled
  스크롤박스: PendingScrollTriggerAmount += 10
             → 임계값 초과 → bTouchPanningCapture = true
             → CaptureMouse(), 스크롤 시작 ✅
```

### 마우스 입력 (PreciseClick 버튼)

```
OnPreviewMouseButtonDown (Tunnel)
  스크롤박스: 터치 아니므로 패스
             → Unhandled

OnMouseButtonDown (Bubble)
  버튼: Press(), Handled
  스크롤박스: 호출 안 됨 (버튼이 Handled)

OnMouseMove (10픽셀 이동)
  버튼: PreciseClick → Release()
        → Unhandled
  스크롤박스: IsMouseButtonDown(RightMouseButton) 체크
             → 우클릭이면 스크롤 ✅
```

핵심 차이는 **초기화 시점**이다. 터치는 Preview에서 미리 준비하고, 마우스는 버튼이 가로채면 준비 자체를 못 한다. 그래서 마우스는 PreciseClick이 없으면 스크롤이 안 되고, 터치는 Preview 덕분에 DownAndUp이어도 스크롤이 될 수 있다(단, 버튼이 CaptureMouse를 안 해야 함).

---

## 정리

터치와 마우스는 입력 특성이 다르기 때문에 엔진이 다르게 처리한다.

터치는 부정확하고 의도가 애매하다. 그래서 스크롤박스가 Preview 단계에서 먼저 받아서 추적을 시작하고, 움직임을 보고 나중에 "스크롤인지 클릭인지" 판단한다. `bFingerOwningTouchInteraction`, `PendingScrollTriggerAmount`, `bTouchPanningCapture` 같은 터치 전용 변수들이 이 판단을 위해 존재한다.

마우스는 정확하고 의도가 명확하다. 그래서 자식(버튼)이 먼저 받는 Bubble 방식으로 처리한다. 버튼 클릭이 우선이고, 빈 공간을 클릭해야 스크롤이 된다.

스크롤 컨테이너 안의 버튼에는 `ClickMethod::PreciseClick`과 `TouchMethod::PreciseTap`을 둘 다 설정하는 게 안전하다.

---

## 참고

- `SScrollBox::OnPreviewMouseButtonDown`, `OnMouseMove` - `Engine/Source/Runtime/Slate/Private/Widgets/Layout/SScrollBox.cpp`
- `SButton::GetClickMethodFromInputType` - `Engine/Source/Runtime/Slate/Private/Widgets/Input/SButton.cpp`
- `EButtonTouchMethod` - `Engine/Source/Runtime/SlateCore/Public/Types/SlateEnums.h`
- `FTunnelPolicy`, `FBubblePolicy` - `Engine/Source/Runtime/Slate/Private/Framework/Application/SlateApplication.cpp`
