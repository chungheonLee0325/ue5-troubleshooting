# Listen Server에서 클라이언트가 호스트와 멀어지면 소켓 기반 공격이 실패하는 이유

## 발단

Listen Server 환경에서 근접 공격을 테스트하던 중 이상한 현상을 발견했다.

지금 진행하는 프로젝트는 서버 권위 방식으로 히트 판정을 구현했다. 클라이언트가 공격하면 서버에서 소켓 위치를 기준으로 Trace를 쏴서 판정한다.

문제는 클라이언트 캐릭터가 호스트에서 멀리 떨어진 곳으로 이동했을 때 발생했다. 몬스터 사냥, 채광, 벌목 같은 근접 판정이 딱 붙어야만 동작한다. 분명 닿는 거리인데 히트가 안 뜨고, 더 가까이 가야 맞았다.

호스트 근처에서는 정상이고, 멀어지면 문제가 생긴다. 뭔가 거리 관련 시스템이 영향을 주는 것 같았다.

---

## 배경: 소켓 기반 히트 판정

구체적으로 어떤 구조인지 보면, 데이터로 지정한 소켓을 기준으로 Trace를 쏜다. 무기의 시작점 소켓과 끝점 소켓 사이를 Sphere Trace나 Line Trace를 생성하는 방식이다. 이 코드는 서버에서 실행된다.

```cpp
// 서버에서 실행되는 히트 판정 코드
void UCombatComponent::PerformMeleeTrace()
{
    USkeletalMeshComponent* Mesh = GetOwnerMesh();
    
    FVector Start = Mesh->GetSocketLocation(TEXT("Weapon_Start"));
    FVector End = Mesh->GetSocketLocation(TEXT("Weapon_End"));
    
    FHitResult HitResult;
    GetWorld()->SweepSingleByChannel(
        HitResult,
        Start,
        End,
        FQuat::Identity,
        ECC_Pawn,
        FCollisionShape::MakeSphere(TraceRadius)
    );
    
    if (HitResult.bBlockingHit)
    {
        ApplyDamage(HitResult.GetActor());
    }
}
```

딱 붙어야만 동작한다는 건 이 Start, End 위치가 이상하다는 거다. 내가 만든 시스템이니까 뭘 기반으로 계산하는지 안다. 소켓 위치가 틀어지는 거라면... LOD?

---

## 원인: LOD가 소켓 위치에 미치는 영향

호스트에서 멀리 떨어진 클라이언트 캐릭터는 호스트 화면에 작게 보이니까 낮은 LOD가 적용된다. LOD가 낮아지면 본 개수가 줄고 위치도 달라질 수 있다.

Skeletal Mesh의 LOD는 단순히 폴리곤만 줄이는 게 아니다. 본도 줄인다.

```
LOD 0 (최고 품질 - 65개 본):
Root
├── Pelvis
├── Spine_01 ─ Spine_02 ─ Spine_03
├── Neck ─ Head
├── Clavicle_L ─ UpperArm_L ─ LowerArm_L ─ Hand_L
│   └── Finger bones (15개)
├── Clavicle_R ─ UpperArm_R ─ LowerArm_R ─ Hand_R
│   └── Finger bones (15개)
└── ...

LOD 2 (낮은 품질 - 20개 본):
Root
├── Pelvis
├── Spine_02 (Spine_01, 03 병합)
├── Head (Neck 병합)
├── UpperArm_L ─ Hand_L (LowerArm, 손가락 제거)
├── UpperArm_R ─ Hand_R
└── ...
```

소켓은 특정 본에 붙어있다. 그 본이 병합되거나 제거되면, 소켓 위치 계산 결과가 달라진다.

엔진 코드를 확인해봤다. 아래는 소켓 위치 계산 흐름을 이해하기 쉽게 간추린 코드다.

```cpp
// 소켓 위치 계산 흐름 (간추림)
// 실제로는 USceneComponent::GetSocketLocation → USkinnedMeshComponent::GetSocketTransform 등을 거침
FVector GetSocketLocation(FName InSocketName) const
{
    if (USkeletalMeshSocket* Socket = GetSocketByName(InSocketName))
    {
        // 소켓의 부모 본 찾기
        int32 BoneIndex = GetBoneIndex(Socket->BoneName);
        
        if (BoneIndex != INDEX_NONE)
        {
            // 현재 LOD의 본 트랜스폼 가져오기 ← 여기가 핵심
            FTransform BoneTransform = GetBoneTransform(BoneIndex);
            
            // 소켓 오프셋 적용
            FTransform SocketTransform = Socket->GetSocketLocalTransform();
            FTransform FinalTransform = SocketTransform * BoneTransform;
            
            return FinalTransform.GetLocation();
        }
    }
    return FVector::ZeroVector;
}
```

`GetBoneTransform`이 **현재 렌더링되는 LOD 레벨의 본 트랜스폼**을 반환한다.

```
[LOD 0 - 정확한 위치]
UpperArm ─ LowerArm ─ Hand ─ [Weapon_Socket]
                              ↑ 정확한 위치

[LOD 2 - 부정확한 위치]
UpperArm ─────────── Hand ─ [Weapon_Socket]
          (LowerArm 없음)     ↑ 위치 틀어짐!
```

---

## Listen Server의 특수성

Listen Server와 Dedicated Server의 핵심 차이가 여기 있다.

**Dedicated Server**는 렌더링을 하지 않는다. 화면에 뭔가 그릴 필요가 없으니까 LOD 시스템 자체가 꺼져있다. `GetSocketLocation`을 호출하면 항상 LOD 0 기준으로 정확한 값을 반환한다.

**Listen Server**는 호스트 플레이어의 화면을 렌더링한다. 그래서 렌더링 최적화 시스템(LOD 포함)이 전부 돌아간다. 문제는 서버 로직도 이 렌더링된 메시 데이터를 그대로 사용한다는 것이다.

```
┌─────────────────────────────────────────────────┐
│              Listen Server                      │
│                                                 │
│  호스트 카메라 ──────────────────────────      	  │ 
│       │                                         │
│       ▼ 가까움 (화면에 크게 보임)               	  │
│  [호스트 캐릭터] LOD 0                  			  │
│                                                 │
│       ▼ 멀음 (화면에 작게 보임)                		  │
│  [클라이언트 캐릭터] LOD 2~3                   	  │
│       │                                         │
│       └─→ 이 캐릭터가 공격할 때             		  │
│           서버는 LOD 2~3 기준으로           		  │
│           소켓 위치를 계산함               		  │
└─────────────────────────────────────────────────┘
```

호스트 카메라에서 멀리 떨어진 클라이언트 캐릭터가 공격하면, 서버는 LOD 2~3 기준의 틀어진 소켓 위치로 히트 판정을 한다. 그래서 "딱 붙어야만" 맞는 것처럼 느껴졌던 거다.

---

## 해결책 검토

몇 가지 방법을 생각해봤다.

### MinLOD 설정

메시 에셋 자체에 최소 LOD를 설정해서 너무 낮은 LOD로 안 내려가게 하는 방법. 근데 이건 전역 설정이라 최적화를 포기하는 셈이다. 소켓을 쓰는 시점이 명확하니까 코드로 제어하는 게 나아 보였다.

### 고정 범위 기반 판정

소켓 위치를 런타임에 가져오지 않고, 캐릭터 위치 기준 고정 범위로 판정하는 방법. FPS 게임의 근접 공격처럼 "캐릭터 앞 90도 부채꼴" 같은 방식이면 가능하다.

근데 기본 근접 공격은 소울라이크 스타일로 구현했다. 애니메이션과 판정이 일치해야 하는 방식이고, 무기를 들고 있으면 **무기 메시의 소켓**도 참조한다. 고정 범위로는 안 맞는다.

### 결론: SetForcedLOD

결국 `SetForcedLOD`가 가장 적합해 보였다. 소켓을 사용하는 시점(근접 공격 판정)이 명확하고, 그 타이밍에만 LOD를 올렸다가 끝나면 복원하면 된다. 기존 시스템 구조를 건드리지 않고 문제를 해결할 수 있다.

## 구현

`SetForcedLOD`로 LOD를 강제 고정할 수 있다.

```cpp
// USkinnedMeshComponent
void SetForcedLOD(int32 InNewForcedLOD);
// 0 = 자동 (거리 기반)
// 1 = LOD 0 강제
// 2 = LOD 1 강제
// ...
```

근접 스킬이 발동할 때 LOD를 최상으로 올리고, 끝나면 다시 자동으로 돌리는 방식을 사용했다. 서버에서만 필요한 처리니까 `HasAuthority` 체크를 넣었다.

```cpp
void UMeleeAttack::OnSkillBegin()
{
    if (GetOwner()->HasAuthority())
    {
        if (USkeletalMeshComponent* Mesh = GetOwnerMesh())
        {
            PrevForcedLOD = Mesh->GetForcedLOD();
            Mesh->SetForcedLOD(1);  // LOD 0 강제
        }
    }
}

void UMeleeAttack::OnSkillEnd()
{
    if (GetOwner()->HasAuthority())
    {
        if (USkeletalMeshComponent* Mesh = GetOwnerMesh())
        {
            Mesh->SetForcedLOD(PrevForcedLOD);  // 복원
        }
    }
}
```

공격 판정이 필요한 순간에만 정확한 본 위치를 쓰고, 평소에는 LOD 최적화가 돌아가게 했다.

---

## Dedicated Server는 어떨까?

궁금해서 찾아봤다. Dedicated Server는 어떻게 다른지. 아래는 핵심 흐름만 간추린 코드다.

```cpp
// Dedicated Server의 LOD 처리 흐름 (간추림)
void USkinnedMeshComponent::TickComponent(float DeltaTime, ...)
{
    if (GetWorld()->IsNetMode(NM_DedicatedServer))
    {
        // 렌더링 관련 업데이트 스킵
        // LOD 계산 안 함 - 항상 LOD 0 기준으로 동작
        return;
    }
    
    UpdateLODStatus();  // Listen Server, Client에서만 실행
}
```

Dedicated Server에서는 이 문제가 아예 발생하지 않는다. LOD 시스템이 비활성화되어 있어서 `GetSocketLocation`이 항상 LOD 0 기준으로 정확한 값을 반환한다.

우리 프로젝트는 Listen Server를 유지할 거라서 위의 해결책을 적용했지만, Dedicated Server로 전환한다면 이 코드는 필요 없어진다. 물론 있어도 해가 되진 않는다. `HasAuthority` 체크가 있고, Dedicated에서는 `SetForcedLOD`를 호출해도 LOD 시스템 자체가 비활성화니까 영향이 없다.

---

## 정리

Listen Server는 호스트 화면을 렌더링하기 때문에 LOD 시스템이 활성화된다. 호스트 카메라에서 먼 클라이언트 캐릭터는 낮은 LOD가 적용되고, 서버 권위 방식에서 이게 히트 판정에 영향을 준다. 소켓 기반으로 위치를 계산하는 시스템이라면 LOD에 따라 위치가 틀어질 수 있다.

해결책은 히트 판정이 필요한 타이밍에 `SetForcedLOD`로 LOD를 강제 고정하는 것이다. 판정이 끝나면 원래대로 돌려서 최적화가 계속 작동하게 한다.

Dedicated Server는 렌더링을 안 하니까 이 문제가 없다.

---

## 참고

- `USkinnedMeshComponent` 관련 소스: `Engine/Source/Runtime/Engine/Private/Components/SkinnedMeshComponent.cpp`
- `SetForcedLOD`, `GetBoneTransform` 등 API는 공식 문서 참조
