# AnimNotifyState 인스턴스 공유 문제: 왜 마지막 몬스터만 정상 동작하는가

## 발단

테스트맵에서 근접 공격 테스트를 하고 있었다. 같은 공격을 반복하는 몬스터를 여러 마리 배치해두고 debug draw로 공격 궤적을 확인하는 중이었는데, 뭔가 이상했다.

한 몬스터만 공격 궤적이 정상적으로 쭉 그려지고, 나머지 몬스터들은 시작 지점만 찍히다 마는 것이다.

```
[정상 - 몬스터 A]
시작 ────────────────────── 끝
     ████████████████████████  (전체 궤적)

[비정상 - 몬스터 B, C, D]
시작 ──
     ███  (시작만 찍히고 끊김)
```

처음엔 개별 몬스터의 애니메이션 타이밍 문제인가 싶었는데, 같은 AnimBP, 같은 Montage를 쓰는데 그럴 리가 없다. 뭔가 공유되는 게 있나?

---

## 배경: 일반적인 근접 공격 구현 패턴

본격적인 원인 추적 전에, 프로젝트에서 어떤 구조를 쓰고 있었는지 설명이 필요할 것 같다.

언리얼에서 근접 공격을 구현할 때 흔히 쓰는 패턴이 `AnimNotifyState`다. Animation Montage에 NotifyState를 배치하고, NotifyBegin~NotifyEnd 구간 동안 히트 판정을 수행한다.

```cpp
UCLASS()
class UMeleeAttackNotifyState : public UAnimNotifyState
{
    GENERATED_BODY()

private:
    // 이미 맞은 액터 추적 (중복 타격 방지)
    TSet<AActor*> HitActors;

public:
    virtual void NotifyBegin(...) override
    {
        HitActors.Empty();  // 공격 시작 시 초기화
    }

    virtual void NotifyTick(...) override
    {
        // 매 프레임 히트 판정
        TArray<FHitResult> Hits;
        PerformTrace(Hits);
        
        for (const FHitResult& Hit : Hits)
        {
            if (!HitActors.Contains(Hit.GetActor()))
            {
                ApplyDamage(Hit.GetActor());
                HitActors.Add(Hit.GetActor());  // 중복 방지
            }
        }
    }

    virtual void NotifyEnd(...) override
    {
        // 공격 종료
    }
};
```

이 패턴 자체는 문제가 없어 보인다. 공격 구간 진입 시 HitActors를 비우고, Tick마다 판정하면서 이미 맞은 대상은 스킵한다. 단일 캐릭터에서는 완벽하게 동작한다.

문제는 **같은 Montage를 사용하는 여러 캐릭터가 동시에 공격할 때** 발생한다.

---

## 원인 추적

의심 가는 부분이 있어서 NotifyState에 로그를 찍어봤다.

```cpp
void UMeleeAttackNotifyState::NotifyTick(USkeletalMeshComponent* MeshComp, ...)
{
    UE_LOG(LogTemp, Warning, TEXT("this=%p | Owner=%s | Mesh=%p"), 
        this, 
        *MeshComp->GetOwner()->GetName(),
        MeshComp);
}
```

결과:
```
this=0x7FF12345 | Owner=Monster_A | Mesh=0xAAA111
this=0x7FF12345 | Owner=Monster_B | Mesh=0xBBB222
this=0x7FF12345 | Owner=Monster_C | Mesh=0xCCC333
this=0x7FF12345 | Owner=Monster_A | Mesh=0xAAA111
this=0x7FF12345 | Owner=Monster_B | Mesh=0xBBB222
...
```

`MeshComp`는 각 몬스터마다 다른데, **`this` 포인터가 전부 동일**하다. 모든 몬스터가 같은 NotifyState 인스턴스를 공유하고 있었다.

---

## 엔진 구조 파악

왜 이런 일이 벌어지는지 엔진 코드를 까봤다. 아래는 핵심 부분만 간추린 코드다.

```cpp
// Engine/Source/Runtime/Engine/Classes/Animation/AnimSequenceBase.h (간추림)
UCLASS(abstract, MinimalAPI, BlueprintType)
class UAnimSequenceBase : public UAnimationAsset
{
    GENERATED_UCLASS_BODY()

public:
    UPROPERTY()
    TArray<FAnimNotifyEvent> Notifies;  // 여기에 NotifyState가 저장됨
};
```

```cpp
// Engine/Source/Runtime/Engine/Classes/Animation/AnimTypes.h (간추림)
USTRUCT()
struct FAnimNotifyEvent : public FAnimLinkableElement
{
    GENERATED_USTRUCT_BODY()

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Instanced, Category=AnimNotifyEvent)
    TObjectPtr<UAnimNotifyState> NotifyStateClass;  // 이 포인터가 공유됨
};
```

NotifyState 인스턴스는 **Animation Asset(Montage/Sequence)에 귀속**된다. `Instanced` 지정자가 있어도 런타임에 캐릭터별로 복제되지 않는다.

```
[메모리 구조]

UAnimMontage "AM_Attack"
└── Notifies[0].NotifyStateClass = 0x7FF12345

Monster_A → AnimInstance → AM_Attack ─┐
Monster_B → AnimInstance → AM_Attack ─┼─→ 같은 NotifyState 0x7FF12345
Monster_C → AnimInstance → AM_Attack ─┘
```

그러니까 내가 NotifyState 멤버 변수에 저장한 `HitActors` 같은 것들이 전부 덮어씌워지고 있었던 것이다.

```
시간순:
T+0.00: Monster_A NotifyBegin → this->HitActors.Empty()
T+0.01: Monster_B NotifyBegin → this->HitActors.Empty()  // A 것도 날아감!
T+0.02: Monster_A NotifyTick  → this->HitActors는 이미 B가 리셋함
...
```

첫 번째 몬스터가 공격을 시작하고 HitActors에 뭔가 쌓이는 중에, 두 번째 몬스터가 NotifyBegin을 호출하면 Empty()가 불려서 첫 번째 몬스터의 데이터도 날아간다. 그래서 마지막 몬스터만 정상 동작하는 것처럼 보였던 거다.

---

## 엔진이 왜 이렇게 설계했을까

처음엔 "이게 버그 아냐?" 싶었는데, 생각해보면 의도된 설계인 것 같다.

1. **메모리 효율**: 같은 애니메이션을 쓰는 캐릭터가 100개면, 각각 NotifyState 인스턴스를 가질 필요가 있을까? 설정값만 같으면 하나로 충분하다.

2. **Stateless 설계 철학**: 엔진에 기본 제공되는 NotifyState들을 보면 (`AnimNotifyState_Trail`, `AnimNotifyState_TimedParticleEffect` 등) 멤버 변수에 런타임 상태를 저장하지 않는다. 설정값만 있고, 실행 시 MeshComp에서 필요한 정보를 가져다 쓴다.

NotifyState의 멤버 변수는 "에디터에서 설정하는 값"을 위한 거지, "런타임에 바뀌는 상태"를 저장하는 용도가 아니었다.

```cpp
// 엔진이 의도한 사용 패턴 (상태 없음)
UCLASS()
class UAnimNotifyState_Sound : public UAnimNotifyState
{
    UPROPERTY(EditAnywhere)
    USoundBase* SoundToPlay;  // 설정값

    virtual void NotifyBegin(USkeletalMeshComponent* MeshComp, ...) override
    {
        // MeshComp에서 위치 가져와서 사운드 재생
        // 멤버 변수에 상태 저장 안 함
        UGameplayStatics::PlaySoundAtLocation(
            MeshComp->GetWorld(),
            SoundToPlay,
            MeshComp->GetComponentLocation()
        );
    }
};
```

---

## 해결책 탐색

몇 가지 방법을 찾아봤다.

### 방법 1: MeshComp를 키로 Map 사용

NotifyState 내부에서 MeshComp 포인터를 키로 상태를 분리하는 방법.

```cpp
UCLASS()
class UMeleeAttackNotifyState : public UAnimNotifyState
{
    TMap<TWeakObjectPtr<USkeletalMeshComponent>, FAttackState> StateMap;
    
    virtual void NotifyBegin(USkeletalMeshComponent* MeshComp, ...) override
    {
        FAttackState& State = StateMap.FindOrAdd(MeshComp);
        State.HitActors.Empty();
    }
    
    virtual void NotifyEnd(USkeletalMeshComponent* MeshComp, ...) override
    {
        StateMap.Remove(MeshComp);
    }
};
```

동작은 하겠지만, NotifyEnd가 호출 안 되는 경우(몽타주 강제 중단 등)에 메모리 정리가 애매해진다. 그리고 NotifyState가 해야 할 일이 아닌 것 같다.

### 방법 2: 외부 객체로 로직 위임

NotifyState는 트리거 역할만 하고, 실제 공격 로직은 캐릭터가 가진 별도 객체에서 처리하는 방법.

```cpp
// NotifyState - 트리거만
virtual void NotifyBegin(USkeletalMeshComponent* MeshComp, ...) override
{
    if (AActor* Owner = MeshComp->GetOwner())
    {
        if (auto* Combat = Owner->FindComponentByClass<UCombatComponent>())
        {
            Combat->OnAttackBegin(AttackIndex);
        }
    }
}
```

---

## 최종 선택

우리 프로젝트에는 자체 Skill 시스템이 있었다. 공격, 스킬 등 동작을 담당하는 객체인데, 이걸 상속받은 `MeleeAttack` 클래스가 근접 공격을 담당한다. 몬스터마다 별도의 Skill 인스턴스를 가지고 있으니까 상태 공유 문제가 없다.

결국 **방법 2**를 선택했다. NotifyState에서 MeshComp → Owner → Skill로 찾아 들어가서 공격 시작/중간/끝 이벤트만 전달하고, 실제 로직(히트 판정, HitActors 관리 등)은 Skill에서 처리하도록 변경했다.

```cpp
// NotifyState - 이제 상태를 저장하지 않음
UCLASS()
class UMeleeAttackNotifyState : public UAnimNotifyState
{
    GENERATED_BODY()

public:
    UPROPERTY(EditAnywhere)
    int32 AttackDataIndex = 0;  // 설정값만 (런타임에 안 바뀜)

    virtual void NotifyBegin(USkeletalMeshComponent* MeshComp, 
        UAnimSequenceBase* Animation, float TotalDuration,
        const FAnimNotifyEventReference& EventReference) override
    {
        if (UMeleeAttack* Skill = FindMeleeAttackSkill(MeshComp))
        {
            Skill->OnNotifyBegin(AttackDataIndex);
        }
    }

    virtual void NotifyTick(USkeletalMeshComponent* MeshComp,
        UAnimSequenceBase* Animation, float FrameDeltaTime,
        const FAnimNotifyEventReference& EventReference) override
    {
        if (UMeleeAttack* Skill = FindMeleeAttackSkill(MeshComp))
        {
            Skill->OnNotifyTick(AttackDataIndex, FrameDeltaTime);
        }
    }

    virtual void NotifyEnd(USkeletalMeshComponent* MeshComp,
        UAnimSequenceBase* Animation,
        const FAnimNotifyEventReference& EventReference) override
    {
        if (UMeleeAttack* Skill = FindMeleeAttackSkill(MeshComp))
        {
            Skill->OnNotifyEnd(AttackDataIndex);
        }
    }

private:
    UMeleeAttack* FindMeleeAttackSkill(USkeletalMeshComponent* MeshComp)
    {
        if (AActor* Owner = MeshComp->GetOwner())
        {
            // 프로젝트에 맞게 Skill을 찾는 로직
            return Owner->FindComponentByClass<UMeleeAttack>();
        }
        return nullptr;
    }
};
```

```cpp
// MeleeAttack (Skill) - 상태 관리
UCLASS()
class UMeleeAttack : public USkillBase  // 또는 UActorComponent
{
    GENERATED_BODY()

private:
    TMap<int32, FMeleeAttackState> AttackStateMap;

    struct FMeleeAttackState
    {
        TSet<AActor*> HitActors;
        bool bIsActive = false;
    };

public:
    void OnNotifyBegin(int32 AttackDataIndex)
    {
        FMeleeAttackState& State = AttackStateMap.FindOrAdd(AttackDataIndex);
        State.HitActors.Empty();
        State.bIsActive = true;
    }

    void OnNotifyTick(int32 AttackDataIndex, float DeltaTime)
    {
        if (FMeleeAttackState* State = AttackStateMap.Find(AttackDataIndex))
        {
            PerformHitDetection(*State);
        }
    }

    void OnNotifyEnd(int32 AttackDataIndex)
    {
        if (FMeleeAttackState* State = AttackStateMap.Find(AttackDataIndex))
        {
            State->bIsActive = false;
        }
    }

private:
    void PerformHitDetection(FMeleeAttackState& State)
    {
        TArray<FHitResult> Hits;
        // Trace 수행...
        
        for (const FHitResult& Hit : Hits)
        {
            if (!State.HitActors.Contains(Hit.GetActor()))
            {
                ApplyDamage(Hit.GetActor());
                State.HitActors.Add(Hit.GetActor());
            }
        }
    }
};
```

---

## 왜 이 구조가 맞다고 생각하는가

단순히 버그를 피하려고 이렇게 한 건 아니다. 생각해보면 역할 분리 측면에서도 이게 맞다.

- **NotifyState**: 애니메이션의 특정 타이밍을 알려주는 역할. "지금 공격 구간이다"라는 신호만 보내면 됨.
- **Skill/Combat Component**: 실제 공격을 수행하는 역할. 히트 판정, 중복 타격 방지, 데미지 계산 등.

NotifyState에 공격 로직을 넣는 건 애초에 책임이 잘못 할당된 거였다. 엔진이 NotifyState를 Asset 레벨에서 공유하도록 설계한 것도 이런 의도가 아닐까 싶다. 상태를 가지지 않는 순수한 트리거로 쓰라는 것.

---

## 정리

AnimNotifyState는 Animation Asset당 하나의 인스턴스만 존재하고, 같은 애니메이션을 사용하는 모든 캐릭터가 이를 공유한다. 멤버 변수에 런타임 상태(HitActors, ElapsedTime 등)를 저장하면 여러 캐릭터가 동시에 사용할 때 덮어씌워진다.

해결 방법은 NotifyState를 트리거로만 사용하고, 실제 로직은 캐릭터별로 존재하는 객체(Component, Skill 등)에서 처리하는 것이다.

---

## 참고

- `UAnimSequenceBase`, `FAnimNotifyEvent` 관련 소스는 `Engine/Source/Runtime/Engine/Classes/Animation/` 디렉토리
- 공식 문서의 `UAnimNotifyState`, `FAnimNotifyEvent` API 참조
