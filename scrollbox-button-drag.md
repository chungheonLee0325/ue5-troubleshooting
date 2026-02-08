# UScrollBox 안에 버튼을 배치했을 때 드래그 스크롤이 안 되는 이유

## 발단

인벤토리 UI를 만들다가 이상한 현상을 발견했다. UScrollBox 안에 아이템 슬롯들을 배치했는데, 드래그 스크롤이 제대로 작동하지 않는 것이다.

각 아이템 슬롯은 UButton을 포함한 커스텀 위젯이었다. 버튼 클릭은 정상 작동했고, 마우스 휠로 스크롤도 잘 되었다. 문제는 마우스로 드래그해서 스크롤하려고 할 때 발생했다.


> [증상]
>- 빈 공간 드래그: 스크롤 정상 작동 ✅
>- 버튼 위 드래그: 스크롤 안 됨 ❌
>- 마우스 휠: 항상 정상 작동 ✅


처음엔 "버튼 위에서도 드래그 스크롤이 되어야 하는 게 정상 아닌가?" 싶어서 뭔가 설정을 잘못한 줄 알았다. ScrollBox의 프로퍼티를 이것저것 만져봤지만 소용없었다.

---

## 해결책 발견

검색해보니 같은 문제를 겪은 사람들이 꽤 있었다. 해결 방법은 간단했다.

**Button의 Details 패널에서 Click Method를 `PreciseClick`으로 변경**

```
Button Details → Advanced → Click Method
기본값: Down And Up
변경: Precise Click
```

이렇게 하니까 버튼 위에서도 드래그 스크롤이 정상 작동했다. 문제는 해결됐지만, 궁금증이 생겼다.

"왜 기본 설정에서는 안 되는 거지? PreciseClick이 뭐가 다른데?"

---

## 배경: Slate 이벤트 버블링

본격적으로 원인을 파헤치기 전에, 언리얼의 이벤트 처리 구조를 이해할 필요가 있었다.

언리얼의 UI 시스템(Slate)은 이벤트를 계층 구조를 따라 전파한다. 대부분의 마우스 이벤트는 **버블링(Bubbling)** 방식으로 처리된다. 즉, 자식 위젯이 먼저 이벤트를 받고, 처리하지 않으면 부모로 올라간다.

```
[위젯 계층]
UScrollBox (부모)
  └─ UButton (자식)

[이벤트 흐름 - 버블링]
1. 버튼 (자식)이 먼저 받음
2. 버튼이 FReply::Unhandled() 반환하면
3. 스크롤박스 (부모)가 받음
```

FReply는 이벤트를 어떻게 처리했는지 알려주는 구조체다.

```cpp
FReply::Handled()    // "내가 처리했다. 부모는 받을 필요 없다"
FReply::Unhandled()  // "나는 관심 없다. 부모가 처리해라"
```

버블링은 엔진의 `FEventRouter`에 정의된 `FBubblePolicy`에 따라 동작한다.

```cpp
// Engine/Source/Runtime/Slate/Private/Framework/Application/SlateApplication.cpp
class FEventRouter::FBubblePolicy
{
public:
    FBubblePolicy( const FWidgetPath& InRoutingPath )
    : WidgetIndex( InRoutingPath.Widgets.Num()-1 )  // 마지막부터 시작 (자식)
    , RoutingPath (InRoutingPath)
    {
    }

    bool ShouldKeepGoing() const
    {
        return WidgetIndex >= 0;  // 0번(최상위 부모)까지 계속
    }

    void Next()
    {
        --WidgetIndex;  // 인덱스 감소 = 부모로 이동
    }
    // ...
};
```

이벤트는 자식(끝 인덱스)에서 시작해서 부모(0번 인덱스)로 올라가며, `FReply::Handled()`를 만나면 전파가 중단된다.

---

## 원인 추적: 버튼이 이벤트를 가로챈다

이제 버튼이 어떻게 이벤트를 처리하는지 확인할 차례였다. 엔진 소스를 열어봤다.

### SButton::OnMouseButtonDown

```cpp
// Engine/Source/Runtime/Slate/Private/Widgets/Input/SButton.cpp

FReply SButton::OnMouseButtonDown( const FGeometry& MyGeometry, const FPointerEvent& MouseEvent )
{
    FReply Reply = FReply::Unhandled();
    if (IsEnabled() && (MouseEvent.GetEffectingButton() == EKeys::LeftMouseButton || MouseEvent.IsTouchEvent()))
    {
        Press();  // 버튼 눌림 상태로 전환
        PressedScreenSpacePosition = MouseEvent.GetScreenSpacePosition();

        EButtonClickMethod::Type InputClickMethod = GetClickMethodFromInputType(MouseEvent);
        
        if(InputClickMethod == EButtonClickMethod::MouseDown)
        {
            Reply = ExecuteOnClick();
        }
        else if (InputClickMethod == EButtonClickMethod::PreciseClick)
        {
            // do not capture the pointer for precise taps or clicks
            Reply = FReply::Handled();
        }
        else  // DownAndUp (기본값)
        {
            // we need to capture the mouse for MouseUp events
            Reply = FReply::Handled().CaptureMouse( AsShared() );
        }
    }

    return Reply;
}
```

**핵심 발견: 버튼이 `OnMouseButtonDown`에서 무조건 `FReply::Handled()`를 반환한다.**
>CaptureMouse: 마우스 버튼을 놓을 때까지 모든 마우스 이벤트를 
이 위젯이 독점. 다른 위젯 위로 마우스가 이동해도 이 위젯이 계속 받음.

DownAndUp 모드(기본값)든 PreciseClick 모드든, 활성화된 버튼은 항상 Handled를 반환한다. 이게 무슨 의미인가?

```
[이벤트 흐름]
사용자가 버튼 위에서 마우스를 누름
  ↓
1. 버튼::OnMouseButtonDown (자식이 먼저)
   → Press() 호출
   → return FReply::Handled()
  ↓
2. 스크롤박스::OnMouseButtonDown
   → 호출되지 않음! ❌ (버튼이 Handled 했으므로)
```

버튼이 이벤트를 가로채서, 스크롤박스는 이 이벤트를 받지 못한다.

### SScrollBox는 OnMouseButtonDown에서 뭘 하는가?

```cpp
// Engine/Source/Runtime/Slate/Private/Widgets/Layout/SScrollBox.cpp

FReply SScrollBox::OnMouseButtonDown( const FGeometry& MyGeometry, const FPointerEvent& MouseEvent )
{
    if ( !bFingerOwningTouchInteraction.IsSet() )
    {
        EndInertialScrolling();  // 관성 스크롤 종료
    }

    if ( MouseEvent.IsTouchEvent() )
    {
        return FReply::Handled();
    }
    else
    {
        if ( MouseEvent.GetEffectingButton() == EKeys::RightMouseButton && ScrollBar->IsNeeded() && bAllowsRightClickDragScrolling)
        {
            AmountScrolledWhileRightMouseDown = 0;  // 초기화!
            Invalidate(EInvalidateWidget::Layout);
            return FReply::Handled();
        }
    }

    return FReply::Unhandled();    
}
```

스크롤박스는 `OnMouseButtonDown`에서 드래그 스크롤을 위한 **초기화 작업**을 한다. `AmountScrolledWhileRightMouseDown` 변수를 0으로 리셋한다.

참고로 기본 SScrollBox는 **오른쪽 마우스 버튼 드래그만** 지원한다. 왼쪽 버튼은 `Unhandled`를 반환한다.

### SScrollBox::OnMouseMove에서의 문제

```cpp
FReply SScrollBox::OnMouseMove( const FGeometry& MyGeometry, const FPointerEvent& MouseEvent )
{
    // ... (터치 처리 생략)
    
    if ( MouseEvent.IsMouseButtonDown(EKeys::RightMouseButton) && bAllowsRightClickDragScrolling)
    {
        // 드래그한 양 누적
        AmountScrolledWhileRightMouseDown += FMath::Abs(ScrollByAmountScreen);

        // 임계값을 넘었는지 확인
        if ( IsRightClickScrolling() )
        {
            InertialScrollManager.AddScrollSample(-ScrollByAmountScreen, FPlatformTime::Seconds());
            const bool bDidScroll = ScrollBy(MyGeometry, -ScrollByAmountLocal, AllowOverscroll, false);

            FReply Reply = FReply::Handled();

            if ( HasMouseCapture() == false )
            {
                Reply.CaptureMouse(AsShared()).UseHighPrecisionMouseMovement(AsShared());
                // ...
            }
            // ...
            return Reply;
        }
    }

    return FReply::Unhandled();
}
```

`IsRightClickScrolling()`은 `AmountScrolledWhileRightMouseDown`이 임계값을 넘었는지 확인하는 함수다. 이 변수는 `OnMouseButtonDown`에서 초기화된다.

**문제의 핵심:**

```
OnMouseButtonDown을 못 받음 
  → AmountScrolledWhileRightMouseDown 초기화 안 됨
  → OnMouseMove에서 IsRightClickScrolling() 체크 실패
  → 드래그 스크롤 안 됨 ❌
```

---

## PreciseClick이 해결하는 방법

그럼 `EButtonClickMethod::PreciseClick`으로 설정하면 뭐가 달라지는가?

### Enum 정의부터 확인

```cpp
// Engine/Source/Runtime/SlateCore/Public/Types/SlateEnums.h

UENUM(BlueprintType)
namespace EButtonClickMethod
{
    enum Type : int
    {
        /**
         * User must press the button, then release while over the button to trigger the click.
         * This is the most common type of button.
         */
        DownAndUp,

        /** Click will be triggered immediately on mouse down, and mouse will not be captured. */
        MouseDown,

        /** Click will always be triggered when mouse button is released over the button,
         * even if the button wasn't pressed down over it. */
        MouseUp,

        /**
         * Inside a list, buttons can only be clicked with precise tap.
         * Moving the pointer will scroll the list, also allows drag-droppable buttons.
         */
         PreciseClick
    };
}
```

주석을 보면 **"Moving the pointer will scroll the list"** - 포인터가 움직이면 리스트가 스크롤된다!

### SButton::OnMouseMove에서의 처리

```cpp
FReply SButton::OnMouseMove( const FGeometry& MyGeometry, const FPointerEvent& MouseEvent )
{
    if ( IsPressed() && IsPreciseTapOrClick(MouseEvent) && 
         FSlateApplication::Get().HasTraveledFarEnoughToTriggerDrag(MouseEvent, PressedScreenSpacePosition) )
    {
        Release();  // 버튼 눌림 상태 해제!
    }

    return FReply::Unhandled();
}
```

```cpp
bool SButton::IsPreciseTapOrClick(const FPointerEvent& MouseEvent) const
{
    return GetClickMethodFromInputType(MouseEvent) == EButtonClickMethod::PreciseClick;
}
```

**핵심: PreciseClick 모드에서는 마우스가 임계값 이상 움직이면 `Release()`를 호출해서 버튼의 눌림 상태를 해제한다.**

`HasTraveledFarEnoughToTriggerDrag()`는 Slate 시스템의 드래그 감지 거리(보통 5~10픽셀)를 체크한다.

### Release()는 무엇을 하는가?

```cpp
void SButton::Release()
{
    if ( bIsPressed )
    {
        bIsPressed = false;
        OnReleased.ExecuteIfBound();
        UpdatePressStateChanged();
    }
}
```

단순히 `bIsPressed` 플래그를 false로 만들고 델리게이트를 호출한다.

---

## 동작 시나리오 비교

### DownAndUp 모드 (기본값) - 스크롤 안 됨

```
T=0ms: 버튼 위에서 마우스 다운
  → 버튼::OnMouseButtonDown
     → Press() → bIsPressed = true
     → return Handled + CaptureMouse
  → 스크롤박스::OnMouseButtonDown 호출 안 됨 ❌

T=50ms: 마우스 10픽셀 이동
  → 버튼::OnMouseMove
     → DownAndUp 모드는 Release() 안 함
     → return Unhandled
  → 스크롤박스::OnMouseMove
     → AmountScrolledWhileRightMouseDown이 초기화 안 됨
     → IsRightClickScrolling() == false
     → 스크롤 안 됨 ❌

T=200ms: 마우스 계속 이동
  → 여전히 스크롤 안 됨 ❌
```

### PreciseClick 모드 - 스크롤 됨

```
T=0ms: 버튼 위에서 마우스 다운
  → 버튼::OnMouseButtonDown
     → Press() → bIsPressed = true
     → return Handled (CaptureMouse는 안 함!)
  → 스크롤박스::OnMouseButtonDown 호출 안 됨 ❌
     (여전히 못 받음)

T=50ms: 마우스 10픽셀 이동 (드래그 임계값 초과)
  → 버튼::OnMouseMove
     → IsPreciseTapOrClick() == true
     → HasTraveledFarEnoughToTriggerDrag() == true
     → Release() 호출! → bIsPressed = false
     → return Unhandled
  → 스크롤박스::OnMouseMove
     → MouseEvent.IsMouseButtonDown(EKeys::RightMouseButton) 체크
     → 오른쪽 버튼이면 스크롤 시작 ✅

T=100ms: 마우스 계속 이동
  → 버튼은 이미 Release 됨
  → 스크롤박스가 정상 동작 ✅
```

**핵심 차이:**

1. **DownAndUp**: 버튼이 끝까지 눌린 상태 유지 → 다른 위젯 방해
2. **PreciseClick**: 조금만 움직이면 빠르게 포기 → 스크롤 가능

PreciseClick은 OnMouseButtonDown을 여전히 가로채지만, OnMouseMove에서 빠르게 Release()하므로 이후 움직임은 스크롤박스가 처리할 수 있다.

---

## 엔진이 왜 이렇게 설계했을까

처음엔 "왜 기본값이 DownAndUp이지? PreciseClick이 더 나은 거 같은데?" 싶었다.

하지만 생각해보면 두 모드 모두 존재 이유가 있다.

### DownAndUp (기본값)
- **일반적인 버튼에 적합**: 메뉴, 다이얼로그, HUD 버튼 등
- 사용자가 실수로 누르고 바로 마우스를 움직여도 클릭이 취소되지 않음
- 누르고 있는 동안 버튼에서 벗어났다가 다시 돌아오면 클릭 가능
- **더 관대한 UX** - 정확한 클릭을 요구하지 않음

### PreciseClick
- **리스트/스크롤 컨테이너 안의 버튼에 적합**: 인벤토리, 상점, 채팅 로그 등
- 드래그 스크롤과 버튼 클릭을 명확히 구분
- 조금이라도 움직이면 클릭 취소 → 스크롤 우선
- **정밀한 클릭만 인정, 드래그 방해 안 함**

엔진이 기본값을 DownAndUp으로 한 건, 대부분의 버튼이 스크롤 컨테이너 밖에 있기 때문일 것이다. 스크롤 안에 넣는 특수한 경우에만 PreciseClick으로 바꾸라는 의도로 보인다.

---

## 프로젝트에 적용

결국 인벤토리 UI의 모든 아이템 슬롯 버튼에 PreciseClick을 적용했다.

UMG 에디터에서 일일이 바꾸기는 귀찮아서, C++로 일괄 설정하는 함수를 만들었다.

```cpp
void UInventorySlot::NativeConstruct()
{
    Super::NativeConstruct();
    
    if (ItemButton)
    {
        // 스크롤박스 안에 있으므로 PreciseClick 사용
        ItemButton->SetClickMethod(EButtonClickMethod::PreciseClick);
    }
}
```

블루프린트에서도 가능하다:

```
Event Construct
  → Get Item Button
  → Set Click Method (Precise Click)
```

---

## 심화: 터치와 마우스의 차이

엔진 코드를 더 보다 보니 `OnPreviewMouseButtonDown`이라는 함수도 발견했다. 버블링과 반대로 **부모가 먼저** 이벤트를 받는 단계(Tunnel)다.

SScrollBox는 이 함수에서 **터치 입력에 대해서만** 초기화를 수행한다. 마우스와 터치의 처리 방식이 꽤 다른데, 이 부분은 별도 문서로 정리했다.

→ [UScrollBox의 터치와 마우스 입력 처리 차이]((./scrollbox-touch-vs-mouse.md)

---

## 정리

UScrollBox 안에 UButton이 있으면, 버튼이 `OnMouseButtonDown` 이벤트를 가로채서(`FReply::Handled()`) 스크롤박스가 드래그 초기화를 못 한다. 

Button의 Click Method를 `PreciseClick`으로 변경하면, 마우스가 드래그 임계값 이상 움직일 때 버튼이 `Release()`를 호출해서 눌림 상태를 빠르게 포기하므로, 이후 마우스 움직임은 스크롤박스가 처리할 수 있다.

엔진은 대부분의 버튼이 스크롤 컨테이너 밖에 있다고 가정하고 DownAndUp을 기본값으로 설정했다. 스크롤 안에 버튼을 넣는 특수한 경우에는 PreciseClick으로 변경해야 한다.

---

## 참고

- `SButton` - `Engine/Source/Runtime/Slate/Private/Widgets/Input/SButton.cpp`
- `SScrollBox` - `Engine/Source/Runtime/Slate/Private/Widgets/Layout/SScrollBox.cpp`
- `EButtonClickMethod` - `Engine/Source/Runtime/SlateCore/Public/Types/SlateEnums.h`
- `FReply` - `Engine/Source/Runtime/SlateCore/Public/Input/Reply.h`
- `FEventRouter`, `FBubblePolicy` - `Engine/Source/Runtime/Slate/Private/Framework/Application/SlateApplication.cpp`
