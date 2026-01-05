# BindWidget vs GetWidgetFromName: 거의 안 쓰는 위젯도 전부 바인딩해야 할까

## 발단

UMG로 UI를 만들다 보니 BindWidget을 습관처럼 쓰게 됐다. 인벤토리, HUD, 창고, 제작 UI 등 여러 위젯을 만들면서 C++에서 접근할 위젯은 전부 BindWidget으로 선언했다.

```cpp
UPROPERTY(meta = (BindWidget))
TObjectPtr<UTextBlock> Txt_ItemCount;

UPROPERTY(meta = (BindWidget))
TObjectPtr<UButton> Btn_Sort;

UPROPERTY(meta = (BindWidget))
TObjectPtr<UTextBlock> Txt_Warning;
```

그러다 문득 고민이 생겼다. `Txt_ItemCount`는 아이템 개수 바뀔 때마다 업데이트하니까 자주 쓴다. 근데 `Txt_Warning` 같은 건? 재료가 부족할 때만 표시하는 건데, 20번 중에 두세 번 정도. 이런 것까지 전부 멤버 변수로 들고 있어야 하나?

"자주 안 쓰는 건 필요할 때 `GetWidgetFromName`으로 찾으면 되지 않나?" 싶었다.

```cpp
void UMyWidget::OnCraftFailed()
{
    // 필요할 때만 찾기
    if (auto* Warning = Cast<UTextBlock>(GetWidgetFromName(TEXT("Txt_Warning"))))
    {
        Warning->SetVisibility(ESlateVisibility::Visible);
    }
}
```

얼핏 보면 이게 더 효율적으로 보인다. 안 쓰는 건 안 찾으니까. 근데 실제로 그런가? 엔진 코드를 찾아봤다.

---

## GetWidgetFromName의 실제 동작

먼저 `GetWidgetFromName`이 어떻게 동작하는지 추적했다.

```cpp
// Engine/Source/Runtime/UMG/Public/Blueprint/UserWidget.h
UWidget* UUserWidget::GetWidgetFromName(const FName& Name) const
{
    return WidgetTree ? WidgetTree->FindWidget(Name) : nullptr;
}
```

`FindWidget`을 따라가 봤다.

```cpp
// Engine/Source/Runtime/UMG/Private/Blueprint/WidgetTree.cpp
UWidget* UWidgetTree::FindWidget(const FName& Name) const
{
    UWidget* FoundWidget = nullptr;

    ForEachWidget([&] (UWidget* Widget) {
       if ( Widget->GetFName() == Name )
       {
          FoundWidget = Widget;
       }
    });

    return FoundWidget;
}
```

여기서 의외의 사실을 발견했다. **TMap이나 해시 테이블을 안 쓴다.** `ForEachWidget`으로 전체 위젯 트리를 순회하면서 이름을 하나씩 비교한다. 심지어 찾아도 break 없이 끝까지 순회한다.

그러니까 `GetWidgetFromName`을 호출할 때마다 **O(N) 선형 탐색**이 발생한다. 위젯이 100개면 100번 비교.

물론 찾은 결과를 TMap에 캐싱해두면 이후 접근은 O(1)이 되지만, 첫 번째 검색은 무조건 전체 트리를 순회한다.

---

## BindWidget은 어떻게 다른가

그럼 `BindWidget`은? UserWidget 초기화 과정에서 호출되는 `InitializeWidgetStatic` 함수를 찾아봤다.

```cpp
// Engine/Source/Runtime/UMG/Private/Blueprint/WidgetBlueprintGeneratedClass.cpp (간추림)
void UWidgetBlueprintGeneratedClass::InitializeWidgetStatic(...)
{
    // 1. 위젯 트리 복제 (BindWidget 유무와 무관하게 항상 발생)
    if ( CreatedWidgetTree == nullptr )
    {
        UserWidget->DuplicateAndInitializeFromWidgetTree(InWidgetTree, ...);
        CreatedWidgetTree = UserWidget->WidgetTree;
    }

    if (CreatedWidgetTree)
    {
        // 2. 프로퍼티 맵 구축 - 클래스의 모든 Object 프로퍼티를 TMap에 캐싱
        TMap<FName, FObjectPropertyBase*> ObjectPropertiesMap;
        for (TFieldIterator<FObjectPropertyBase> It(WidgetBlueprintClass, ...); It; ++It)
        {
            ObjectPropertiesMap.Add(It->GetFName(), *It);
        }

        // 3. 위젯 트리 순회 + 바인딩
        CreatedWidgetTree->ForEachWidget([&](UWidget* Widget) {
            // 위젯 이름으로 프로퍼티 맵에서 O(1) 검색
            if (FObjectPropertyBase** PropPtr = ObjectPropertiesMap.Find(Widget->GetFName()))
            {
                FObjectPropertyBase* Prop = *PropPtr;
                Prop->SetObjectPropertyValue_InContainer(UserWidget, Widget);
            }
        });
    }
}
```

핵심을 정리하면:

1. **위젯 트리 순회는 딱 1회** - `ForEachWidget`이 한 번만 호출됨
2. **프로퍼티 검색은 O(1)** - TMap 해시 기반
3. **BindWidget 개수와 무관** - 0개든 100개든 트리 순회 횟수는 똑같음

---

## 비용 비교

위젯 트리에 100개의 위젯이 있고, 그중 30개에 접근해야 하는 상황을 가정해봤다.

**GetWidgetFromName (30번 호출)**
```
첫 번째 호출: 100개 순회
두 번째 호출: 100개 순회
...
30번째 호출: 100개 순회

총: 100 × 30 = 3,000번 비교
```

**BindWidget (30개 선언)**
```
초기화 시 ForEachWidget 1회: 100개 순회
각 위젯마다 TMap lookup: O(1)

총: 100번 순회
```

BindWidget을 50개로 늘려도? 여전히 100번 순회. 트리 순회는 BindWidget 개수와 완전히 무관하다.

---

## "거의 안 쓰는 위젯" 문제

원래 고민으로 돌아가보자. 거의 사용하지 않는 위젯을 BindWidget으로 추가 선언하면 어떤 비용이 발생하나?

트리 순회 비용은 변하지 않는다. 추가되는 건:
- 프로퍼티 맵에 엔트리 1개 추가: O(1)
- TMap lookup 1회: O(1)  
- 포인터 대입 1회: O(1)
- **메모리: 64비트 환경에서 인스턴스당 8바이트** (포인터 크기)

리플렉션 메타데이터 비용도 있지만, 이건 `UPROPERTY` 선언 자체의 비용이고 `UClass`에 저장되어 모든 인스턴스가 공유한다. 인스턴스가 100개든 1000개든 메타데이터는 클래스당 1세트.

결국 유일한 인스턴스별 비용은 포인터 8바이트다.

반면 `GetWidgetFromName`은? "거의 안 씀"이 "한 번도 안 씀"은 아니다. 사용할 때마다 전체 트리를 순회한다.

---

## 메모리 비용

BindWidget은 메모리 효율도 더 좋다. (64비트 환경 기준)

**BindWidget (N개 선언)**
- 인스턴스당: N × 8 bytes (포인터)
- 리플렉션 메타데이터는 클래스당 1세트, 모든 인스턴스가 공유

**GetWidgetFromName + TMap 캐싱 (K개)**
- 인스턴스당: TMap 오버헤드 ~64 bytes + K × ~24 bytes (FName + 포인터 + 해시)

위젯 30개 기준으로 BindWidget은 240 bytes, TMap 캐싱은 ~784 bytes.

---

## 바인딩이 실제로 하는 일

엔진은 언리얼 리플렉션 시스템을 활용해서 바인딩한다. `UPROPERTY`로 선언하면 이름, 타입, 메모리 오프셋 같은 메타데이터가 생성되고, 이걸로 런타임에 포인터를 직접 대입한다.

```
UMyWidget 인스턴스 메모리:
┌──────────────────────────────────┐
│ UObject 기본 데이터   (0~64)        │
├──────────────────────────────────┤
│ UUserWidget 데이터    (64~120) 	   │
├──────────────────────────────────┤
│ Txt_ItemCount        (offset 128)│ ← 포인터 저장
├──────────────────────────────────┤
│ Btn_Sort             (offset 136)│
└──────────────────────────────────┘

Container 주소 + 오프셋 = 실제 주소 → 포인터 대입
```

복잡해 보이지만 결과적으로 `UserWidget->Txt_ItemCount = Widget;`과 동일하다.

---

## 정리

`GetWidgetFromName`은 호출할 때마다 전체 위젯 트리를 순회한다. `BindWidget`은 초기화 시 1회만 순회하고, 선언 개수와 무관하다.

"거의 안 쓰는 위젯"도 BindWidget으로 선언하는 게 맞다. 추가 비용은 인스턴스당 포인터 하나(64비트 환경에서 8바이트)뿐이고, 나중에 접근할 때 트리 순회를 피할 수 있다.

`GetWidgetFromName`이 필요한 경우는 런타임에 위젯 이름이 동적으로 결정되는 경우 정도다.

---

## 참고

- `UWidgetBlueprintGeneratedClass::InitializeWidgetStatic` - `Engine/Source/Runtime/UMG/Private/Blueprint/WidgetBlueprintGeneratedClass.cpp`
- `UWidgetTree::FindWidget` - `Engine/Source/Runtime/UMG/Private/Blueprint/WidgetTree.cpp`
- `UUserWidget::GetWidgetFromName` - `Engine/Source/Runtime/UMG/Private/Blueprint/UserWidget.cpp`
