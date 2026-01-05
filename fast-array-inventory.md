# TMap은 Replicate가 안 된다: Fast Array로 인벤토리 동기화하기

## 발단

멀티플레이어 게임에서 인벤토리 시스템을 만들고 있었다. 자연스럽게 `TMap`을 선택했다. 아이템 ID를 키로, 아이템 정보를 값으로.

```cpp
UPROPERTY(Replicated)
TMap<int32, FInventoryItem> InventoryMap;
```

빌드가 안 된다. `TMap`은 리플리케이션을 지원하지 않아서 `Replicated` 지정자를 붙이면 컴파일 에러가 난다.

---

## 첫 번째 시도: TArray로 변경

일단 동작하게 만들어야 했다. `TArray<FInventoryItem>`으로 바꿨다.

```cpp
USTRUCT()
struct FInventoryItem
{
    GENERATED_BODY()

    UPROPERTY()
    int32 ItemID = 0;

    UPROPERTY()
    int32 Count = 0;

    UPROPERTY()
    int32 SlotIndex = 0;
    
    // ...
};

UPROPERTY(Replicated)
TArray<FInventoryItem> InventoryArray;
```

동기화는 됐다. 근데 마음에 안 드는 부분이 있었다.

---

## 문제: 전체 배열 동기화

`TArray`의 리플리케이션은 단순하다. 배열이 바뀌면 **전체 배열**을 다시 보낸다.

인벤토리에 아이템이 50개 있는데, 포션 하나 사용해서 Count가 99 → 98로 바뀌면? 50개 아이템 정보가 전부 날아간다.

```
[변경 전]
Slot 0: Sword x1
Slot 1: Shield x1
Slot 2: Potion x99  ← 이것만 바뀜
Slot 3: Gold x500
...
Slot 49: Arrow x200

[리플리케이션]
전체 50개 아이템 정보 전송  ← 비효율
```

테스트 환경에서야 체감이 안 되지만, 라이브 서비스에서 수백 명이 동시에 인벤토리 조작하면 대역폭 낭비가 신경 쓰였다.

---

## 해결책 탐색

예전에 C# 서버에서 인벤토리 시스템을 만들 때 dirty 플래그 패턴을 썼던 기억이 났다. 변경된 슬롯만 마킹해두고 그것만 동기화하는 방식. 언리얼에도 비슷한 게 있지 않을까?

### 방법 1: 델타 비교 직접 구현

이전 상태와 현재 상태를 비교해서 바뀐 것만 보내는 방식. 직접 구현할 수도 있다.

```cpp
// 의사 코드
void ReplicateInventoryDelta()
{
    for (int32 i = 0; i < InventoryArray.Num(); i++)
    {
        if (InventoryArray[i] != PrevInventoryArray[i])
        {
            // 이 슬롯만 RPC로 전송
            Client_UpdateSlot(i, InventoryArray[i]);
        }
    }
    PrevInventoryArray = InventoryArray;
}
```

가능은 한데, 위험해 보였다. 비교 로직 실수하면 동기화가 깨지고, 추가/삭제 처리도 복잡하고, 엔진의 리플리케이션 시스템과 따로 노는 코드가 된다.

### 방법 2: FFastArraySerializer

찾아보니 언리얼에 이미 이런 용도로 만들어진 게 있었다. `FFastArraySerializer`.

엔진에서 제공하는 **범용 델타 리플리케이션 시스템**이다. 배열의 **변경된 요소만** 리플리케이트한다. 내가 C# 서버에서 쓰던 dirty 패턴과 비슷한 개념이다. GAS 내부에서도 이걸 쓰고, 인벤토리나 장비 시스템 구현 예제에서도 자주 보인다. 정확히 원하던 거다.

---

## Fast Array 구현

### 구조

Fast Array는 두 개의 구조체로 구성된다.

1. **배열 요소** (`FFastArraySerializerItem` 상속)
2. **배열 컨테이너** (`FFastArraySerializer` 상속)

```cpp
// 1. 배열 요소 (예시 코드)
USTRUCT()
struct FInventoryEntry : public FFastArraySerializerItem
{
    GENERATED_BODY()

    UPROPERTY()
    int32 ItemID = 0;

    UPROPERTY()
    int32 Count = 0;

    UPROPERTY()
    int32 SlotIndex = 0;

    // 클라이언트에서 변경 감지 시 호출되는 콜백 (선택 사항)
    void PreReplicatedRemove(const struct FInventoryList& InArraySerializer);
    void PostReplicatedAdd(const struct FInventoryList& InArraySerializer);
    void PostReplicatedChange(const struct FInventoryList& InArraySerializer);
};

// 2. 배열 컨테이너 (예시 코드)
USTRUCT()
struct FInventoryList : public FFastArraySerializer
{
    GENERATED_BODY()

    UPROPERTY()
    TArray<FInventoryEntry> Items;

    // 소유자 참조 (콜백에서 사용)
    UPROPERTY(NotReplicated)
    TObjectPtr<UInventoryComponent> OwnerComponent = nullptr;

    // 필수: 리플리케이션을 위한 NetDeltaSerialize
    bool NetDeltaSerialize(FNetDeltaSerializeInfo& DeltaParms)
    {
        return FFastArraySerializer::FastArrayDeltaSerialize<FInventoryEntry, FInventoryList>(
            Items, DeltaParms, *this);
    }
};

// 필수: TStructOpsTypeTraits 정의
template<>
struct TStructOpsTypeTraits<FInventoryList> : public TStructOpsTypeTraitsBase2<FInventoryList>
{
    enum
    {
        WithNetDeltaSerializer = true,
    };
};
```

### 컴포넌트에서 사용

아래는 인벤토리 컴포넌트에서 Fast Array를 사용하는 예시 코드다.

```cpp
UCLASS()
class UInventoryComponent : public UActorComponent
{
    GENERATED_BODY()

public:
    UPROPERTY(Replicated)
    FInventoryList InventoryList;

    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override
    {
        Super::GetLifetimeReplicatedProps(OutLifetimeProps);
        DOREPLIFETIME(UInventoryComponent, InventoryList);
    }

    // 아이템 추가
    void AddItem(int32 ItemID, int32 Count)
    {
        if (GetOwner()->HasAuthority())
        {
            FInventoryEntry NewEntry;
            NewEntry.ItemID = ItemID;
            NewEntry.Count = Count;
            NewEntry.SlotIndex = FindEmptySlot();

            InventoryList.Items.Add(NewEntry);
            InventoryList.MarkItemDirty(InventoryList.Items.Last());  // 중요!
        }
    }

    // 아이템 수량 변경
    void UpdateItemCount(int32 SlotIndex, int32 NewCount)
    {
        if (GetOwner()->HasAuthority())
        {
            if (FInventoryEntry* Entry = FindEntryBySlot(SlotIndex))
            {
                Entry->Count = NewCount;
                InventoryList.MarkItemDirty(*Entry);  // 중요!
            }
        }
    }

    // 아이템 제거
    void RemoveItem(int32 SlotIndex)
    {
        if (GetOwner()->HasAuthority())
        {
            int32 Index = FindIndexBySlot(SlotIndex);
            if (Index != INDEX_NONE)
            {
                InventoryList.Items.RemoveAt(Index);
                InventoryList.MarkArrayDirty();  // 삭제 시에는 MarkArrayDirty
            }
        }
    }
};
```

### 핵심: MarkItemDirty

아이템을 변경한 후 반드시 `MarkItemDirty`를 호출해야 한다. 이게 없으면 리플리케이션 시스템이 변경을 감지하지 못한다.

```cpp
// 변경 후 반드시 호출
Entry->Count = NewCount;
InventoryList.MarkItemDirty(*Entry);  // 이 아이템이 바뀌었다고 알림

// 삭제나 대량 변경 시
InventoryList.MarkArrayDirty();  // 전체 배열 다시 체크하라고 알림
```

### 클라이언트 콜백

클라이언트에서 변경을 감지하고 UI 업데이트 등을 처리할 수 있다. 아래는 구현 예시다.

```cpp
void FInventoryEntry::PreReplicatedRemove(const FInventoryList& InArraySerializer)
{
    // 아이템이 제거되기 전 호출
    if (UInventoryComponent* Comp = InArraySerializer.OwnerComponent)
    {
        Comp->OnItemRemoved(SlotIndex);
    }
}

void FInventoryEntry::PostReplicatedAdd(const FInventoryList& InArraySerializer)
{
    // 새 아이템이 추가된 후 호출
    if (UInventoryComponent* Comp = InArraySerializer.OwnerComponent)
    {
        Comp->OnItemAdded(SlotIndex, ItemID, Count);
    }
}

void FInventoryEntry::PostReplicatedChange(const FInventoryList& InArraySerializer)
{
    // 기존 아이템이 변경된 후 호출
    if (UInventoryComponent* Comp = InArraySerializer.OwnerComponent)
    {
        Comp->OnItemChanged(SlotIndex, ItemID, Count);
    }
}
```

---

## 동작 원리

Fast Array가 어떻게 델타만 보내는지 간단히 보면, 아래는 핵심 흐름만 간추린 것이다.

```cpp
// FastArrayDeltaSerialize 핵심 흐름 (간추림)
template<typename Type, typename SerializerType>
static bool FastArrayDeltaSerialize(
    TArray<Type>& Items,
    FNetDeltaSerializeInfo& Parms,
    SerializerType& ArraySerializer)
{
    if (Parms.Writer)
    {
        // 서버: 변경된 아이템만 직렬화
        // 각 아이템의 ReplicationID와 ReplicationKey로 변경 추적
    }
    else
    {
        // 클라이언트: 받은 델타 적용
        // 추가/수정/삭제 콜백 호출
    }
}
```

각 `FFastArraySerializerItem`은 내부적으로 `ReplicationID`와 `ReplicationKey`를 가진다. 엔진이 이걸로 어떤 아이템이 추가/수정/삭제됐는지 추적한다. 델타 비교를 직접 구현할 필요 없이 엔진이 알아서 해준다.

---

## TArray vs Fast Array

```
[TArray - 포션 1개 사용]
서버: Potion 99 → 98
전송: 50개 아이템 전체 (약 2KB)

[Fast Array - 포션 1개 사용]
서버: Potion 99 → 98, MarkItemDirty
전송: 변경된 1개 아이템 (약 40B)
```

직접 프로파일링을 돌려보진 않았지만, 구조적으로 효율적인 건 분명하다. 그리고 델타 비교를 직접 구현하는 것보다 엔진에서 제공하는 검증된 시스템을 쓰는 게 안전하다.

---

## 정리

`TMap`은 리플리케이션을 지원하지 않는다. `TArray`는 되지만 전체 배열을 동기화해서 비효율적이다. 

인벤토리처럼 자주 변경되는 배열 데이터는 `FFastArraySerializer`를 사용하면 변경된 요소만 동기화할 수 있다. `MarkItemDirty` 호출만 잊지 않으면 된다.

---

## 참고

- 엔진 소스: `FastArraySerializer.h` (Runtime/NetCore 또는 Runtime/Engine 모듈)
- 공식 문서의 `FFastArraySerializer`, `FFastArraySerializerItem` API 참조
