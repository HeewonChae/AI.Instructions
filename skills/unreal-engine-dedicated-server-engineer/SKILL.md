---
name: unreal-engine-dedicated-server-engineer
description: >
  Unreal Engine 5 데디케이티드 서버 리플리케이션 및 RPC 코드를 작성할 때 반드시 사용하는 스킬.
  다음 상황에서 항상 이 스킬을 참조할 것:
  - UPROPERTY(Replicated), RepNotify, DOREPLIFETIME 코드 작성
  - Server/Client/NetMulticast RPC 함수 선언 및 구현
  - 소유권(Ownership), HasAuthority(), ENetRole 관련 로직
  - 리플리케이션 변수 설계, 조건부 리플리케이션(COND_OwnerOnly 등)
  - 멀티플레이어 캐릭터/액터/무기/함정 등 네트워크 동기화 코드
  - "서버에서만 실행", "클라이언트에서만 실행", "모든 클라이언트에 브로드캐스트" 같은 요구사항
  UE 네트워크 코드가 조금이라도 포함된 요청이라면 이 스킬을 사용한다.
---

# UE5 Replication & RPC 코드 작성 가이드

## 코드 작성 전 필수 확인 3가지

1. **이 로직은 어느 머신에서 실행되어야 하는가?** (서버 전용 / 클라이언트 전용 / 양쪽)
2. **이 액터의 소유권 체인은 어디로 연결되는가?** (PlayerController까지 도달 가능한가?)
3. **데이터인가, 이벤트인가?** (데이터 → Replicated 변수 / 이벤트 → RPC)

---

## 클래스별 존재 위치 규칙

| 클래스 | 서버 | 클라이언트 |
|---|---|---|
| `AGameMode` | 존재 | NULL (접근 불가) |
| `AGameState` | 존재 | 존재 (복제됨) |
| `APlayerState` | 존재 | 존재 (모든 플레이어 것 포함) |
| `APlayerController` | 모든 PC 존재 | 자신의 PC만 존재 |
| `APawn/ACharacter` | 존재 | 존재 (복제됨) |
| `AHUD`, `UUserWidget` | 미생성 | 존재 |

```cpp
// 클라이언트에서 GameMode 접근 시 항상 NULL 체크
AMyGameMode* GM = GetWorld()->GetAuthGameMode<AMyGameMode>();
if (!GM) return; // 클라이언트에서는 NULL
```

---

## 변수 리플리케이션 패턴

### 패턴 1: Replicated (값만 동기화)

```cpp
// .h
UPROPERTY(Replicated)
int32 TeamId;

virtual void GetLifetimeReplicatedProps(
    TArray<FLifetimeProperty>& OutLifetimeProps) const override;

// .cpp
void AMyActor::GetLifetimeReplicatedProps(
    TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps); // 절대 빠뜨리지 않는다
    DOREPLIFETIME(AMyActor, TeamId);
}
```

사용 시점: 값만 있으면 되고 변경 시 추가 로직이 없을 때 (위치, 팀 번호, 점수 등)

### 패턴 2: RepNotify (값 동기화 + 콜백)

```cpp
// .h
UPROPERTY(ReplicatedUsing = OnRep_Health)
float Health;

UFUNCTION()
void OnRep_Health(); // 반드시 UFUNCTION() 붙일 것

// .cpp
void AMyCharacter::OnRep_Health()
{
    // 클라이언트에서만 자동 호출 (서버는 수동 호출 필요)
    UpdateHealthBarWidget(Health);
    PlayHitReactionMontage();
}
```

**C++ vs 블루프린트 OnRep 실행 위치:**

| | C++ OnRep | BP RepNotify |
|---|---|---|
| 서버에서 자동 호출 | X (수동 호출 필요) | O (자동) |
| 클라이언트에서 자동 호출 | O | O |

### 패턴 3: 조건부 리플리케이션

```cpp
void AMyCharacter::GetLifetimeReplicatedProps(
    TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    DOREPLIFETIME_CONDITION(AMyCharacter, Stamina,      COND_OwnerOnly);   // 나만
    DOREPLIFETIME_CONDITION(AMyCharacter, SkinId,       COND_InitialOnly); // 최초 1회
    DOREPLIFETIME_CONDITION(AMyCharacter, SimPhysState, COND_SkipOwner);   // 나 제외
    DOREPLIFETIME(AMyCharacter, Health); // 조건 없음 (항상 모두에게)
}
```

| 매크로 | 대상 | 대표 사용처 |
|---|---|---|
| `COND_OwnerOnly` | 소유 클라이언트만 | 개인 스태미나, 탄약, 쿨타임 |
| `COND_SkipOwner` | 소유 클라이언트 제외 | 타인이 봐야 할 애니메이션 상태 |
| `COND_InitialOnly` | 스폰 시 1회 | 외형 ID, 고정 스탯 |
| `COND_SimulatedOnly` | SimulatedProxy만 | 보간용 물리 데이터 |
| `COND_AutonomousOnly` | AutonomousProxy만 | 내 캐릭터 전용 피드백 |

---

## RPC 패턴

### 선언 규칙

```cpp
// Server RPC: 클라이언트 -> 서버 (소유권 있는 액터에서만 작동)
UFUNCTION(Server, Reliable, WithValidation)
void Server_Attack(FVector Direction);

// Client RPC: 서버 -> 특정 클라이언트 (PlayerController에서 호출)
UFUNCTION(Client, Reliable)
void Client_ShowNotification(FText Message);

// NetMulticast RPC: 서버 -> 모든 클라이언트 (서버에서 호출해야 전파됨)
UFUNCTION(NetMulticast, Unreliable)
void Multicast_PlayEffect(FVector Location);
```

### 구현 규칙

```cpp
// Server RPC: Validate + Implementation 두 함수 필수 구현
bool AMyCharacter::Server_Attack_Validate(FVector Direction)
{
    return Direction.IsNormalized() && !bIsDead && AttackCooldown <= 0.f;
}

void AMyCharacter::Server_Attack_Implementation(FVector Direction)
{
    // HasAuthority() == true 보장
    ApplyDamageToTarget(Direction);
    Multicast_PlayAttackEffect(GetActorLocation());
}

// Client RPC
void AMyPlayerController::Client_ShowNotification_Implementation(FText Message)
{
    if (HUDWidget)
        HUDWidget->ShowNotification(Message);
}

// NetMulticast RPC
void AMyCharacter::Multicast_PlayEffect_Implementation(FVector Location)
{
    UGameplayStatics::SpawnEmitterAtLocation(GetWorld(), ExplosionFX, Location);
}
```

### RPC 실행 위치 결정 테이블

**클라이언트에서 호출 시:**

| 액터 소유권 | Server RPC | NetMulticast | Client RPC |
|---|---|---|---|
| 클라이언트 소유 | 서버에서 실행 | 로컬만 실행 | 로컬 실행 |
| 서버 소유 | 실행 안됨 | 로컬만 실행 | 로컬 실행 |
| 소유 없음 | 실행 안됨 | 로컬만 실행 | 로컬 실행 |

**서버에서 호출 시:**

| 액터 소유권 | Server RPC | NetMulticast | Client RPC |
|---|---|---|---|
| 클라이언트 소유 | 서버 실행 | 서버+전체 | 소유 클라이언트 |
| 서버 소유 | 서버 실행 | 서버+전체 | 서버 실행 |
| 소유 없음 | 서버 실행 | 서버+전체 | 서버 실행 |

---

## 소유권(Ownership)과 HasAuthority()

### HasAuthority() 사용 패턴

```cpp
void AMyActor::ApplyDamage(float Amount)
{
    if (!HasAuthority()) return; // 서버에서만 실행

    Health -= Amount;
    // Replicated 변수이므로 자동으로 클라이언트에 동기화됨
}
```

### 소유권 체인 설정

```cpp
// Possess() 는 자동으로 소유권 체인 연결
PlayerController->Possess(Pawn);

// 부착 액터는 SetOwner()로 체인 연결
WeaponActor->SetOwner(OwningPawn); // WeaponActor -> Pawn -> PlayerController 체인
```

### 소유 없는 액터(Unowned Actor) 처리

소유 없는 액터 = 맵 배치 함정, 서버 스폰 아이템 박스 등 PlayerController 체인이 없는 액터.

```cpp
// 상황 A: 클라이언트가 트리거 -> PlayerController 경유
void ATrapActor::OnPlayerEnter(APlayerController* InstigatingPC)
{
    // ATrapActor 자체에서 Server RPC 불가 (소유 없음)
    if (InstigatingPC && InstigatingPC->IsLocalController())
    {
        InstigatingPC->Server_RequestActivateTrap(this);
    }
}

// 상황 B: 서버 OnOverlap -> HasAuthority()로 직접 처리
void ATrapActor::OnOverlapBegin(UPrimitiveComponent* OverlappedComp, AActor* OtherActor, ...)
{
    if (!HasAuthority()) return;
    ActivateTrap();
    Multicast_PlayTrapEffect(); // 전체 브로드캐스트
}
```

---

## 체력 시스템 완성 예시

```cpp
// AMyCharacter.h
UCLASS()
class AMyCharacter : public ACharacter
{
    GENERATED_BODY()

public:
    UPROPERTY(ReplicatedUsing = OnRep_Health)
    float Health = 100.f;

    UPROPERTY(Replicated)
    int32 TeamId = 0;

    UPROPERTY(Replicated) // GetLifetimeReplicatedProps에서 COND_OwnerOnly 설정
    float Stamina = 100.f;

    UFUNCTION() void OnRep_Health();

    UFUNCTION(Server, Reliable, WithValidation)
    void Server_Attack(FVector Direction);

    UFUNCTION(NetMulticast, Unreliable)
    void Multicast_PlayAttackEffect(FVector HitLocation);

    virtual void GetLifetimeReplicatedProps(
        TArray<FLifetimeProperty>& OutLifetimeProps) const override;

    void SetHealth(float NewHealth); // 서버 전용 세터
};

// AMyCharacter.cpp
void AMyCharacter::GetLifetimeReplicatedProps(
    TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);
    DOREPLIFETIME(AMyCharacter, Health);
    DOREPLIFETIME(AMyCharacter, TeamId);
    DOREPLIFETIME_CONDITION(AMyCharacter, Stamina, COND_OwnerOnly);
}

void AMyCharacter::SetHealth(float NewHealth)
{
    if (!HasAuthority()) return;
    Health = NewHealth;
    OnRep_Health(); // 서버에서는 C++ OnRep 수동 호출 필요
}

void AMyCharacter::OnRep_Health()
{
    if (HealthBarWidget)
        HealthBarWidget->SetHealthPercent(Health / MaxHealth);
    if (Health <= 0.f)
        PlayDeathMontage();
}

bool AMyCharacter::Server_Attack_Validate(FVector Direction)
{
    return !bIsDead && Direction.IsNormalized();
}

void AMyCharacter::Server_Attack_Implementation(FVector Direction)
{
    FHitResult Hit;
    PerformAttackTrace(Direction, Hit);
    if (AMyCharacter* Target = Cast<AMyCharacter>(Hit.GetActor()))
    {
        Target->SetHealth(Target->Health - AttackDamage);
    }
    Multicast_PlayAttackEffect(Hit.ImpactPoint);
}

void AMyCharacter::Multicast_PlayAttackEffect_Implementation(FVector HitLocation)
{
    UGameplayStatics::SpawnEmitterAtLocation(GetWorld(), HitFX, HitLocation);
    UGameplayStatics::PlaySoundAtLocation(GetWorld(), HitSound, HitLocation);
}
```

---

## 안티패턴 체크리스트

코드 작성 후 반드시 확인:

- [ ] `GetLifetimeReplicatedProps`에서 `Super::` 호출했는가?
- [ ] Replicated 변수를 클라이언트에서 직접 수정하지 않았는가?
- [ ] NetMulticast를 서버에서 호출하는가? (클라이언트 호출 = 로컬만 실행)
- [ ] Server RPC를 소유권 있는 액터에서 호출하는가?
- [ ] Tick 같은 고빈도 호출에 Reliable RPC를 쓰지 않는가?
- [ ] RPC 파라미터에 대용량 배열을 통째로 넘기지 않는가?
- [ ] Server RPC에 `WithValidation` + Validate 함수 구현했는가?
- [ ] C++ OnRep를 서버에서도 실행해야 한다면 수동 호출했는가?
- [ ] 소유 없는 액터에서 Server RPC를 직접 호출하지 않았는가?

---

## Reliable vs Unreliable 선택 기준

```
Reliable  -> 게임 플레이에 영향을 주는 1회성 이벤트
             (공격 판정, 아이템 획득, 사망, UI 상태 변경)

Unreliable -> 손실되어도 곧 갱신되는 시각/청각 피드백
              (파티클 이펙트, 풋스텝 사운드, 위치 보간 힌트)
```
