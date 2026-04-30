---
title: "Week_3_Assignment_Plan"
date: 2026-04-08 08:00:00 +0900
categories: ["Assignment", "Plan"]
tags: ["2026", 내일배움캠프, 과제, Unreal]
math: true
---

# **3주차 과제 기획**

<br>

## 구현 순서

핵심이 되는 '캐릭터 이동'을 우선으로 잡고, 이후 환경(퍼즐)과 시스템(웨이브)을 순차적으로 적용합니다.

1. **[7번 과제] 캐릭터 움직임 구현 (우선순위 1️⃣)**
2. **[6번 과제] 퍼즐 오브젝트 세팅 (우선순위 2️⃣)**
3. **[8번 과제] 웨이브 및 디버프 시스템 (우선순위 3️⃣)**

<br>

## [7번 과제] 주의사항 및 제약 조건

> **핵심 구현 제약 조건**
> 
> 1. **`Character` 클래스 사용 금지**
>    * 플레이어는 반드시 **`Pawn` 클래스**를 베이스로 생성해야 합니다. (즉, 엔진이 기본으로 제공하는 `CharacterMovementComponent`를 사용할 수 없음)
> 
> 2. **기본 카메라/이동 제어 함수 사용 금지**
>    * `AddControllerYawInput()`, `AddControllerPitchInput()` 등 언리얼 엔진이 기본 제공하는 컨트롤러 회전 함수를 사용해선 안 됩니다.

<br>

### 계획
조건에 따라, 키보드 및 마우스 입력값을 단순히 엔진 함수에 넘겨주는 것이 아니라, **DeltaTime과 벡터(Vector), 로테이터(Rotator)를 활용하여 트랜스폼(Transform)의 위치와 회전값을 직접 수학적으로 계산하고 갱신(SetActorLocation, SetActorRotation)하는 방식**으로 구현할 예정입니다.

### SpringArmComp->bUsePawnControlRotation = false; 하는이유

만약 이 옵션이 `true`로 되어 있으면, 드론이 아무리 앞뒤로 기울고(Pitch) 옆으로 누워도(Roll) 카메라는 세상의 수평을 유지하려고 빳빳하게 서 있게 됩니다. 
우리는 **"드론의 시점(Local)"**을 그대로 따라가며 박진감 넘치게 비행해야 하므로, 컨트롤러의 가상 회전을 무시(`false`)하고 드론 메쉬의 실제 회전에 카메라를 용접시키듯 고정하기 위함입니다.

### IA_Move과 IA_Look Axis3D로 설정

일반적인 인간형 캐릭터는 땅 위에서 앞뒤/좌우(2D)로만 걷지만, 6자유도 드론은 **앞뒤(X), 좌우(Y), 상하(Z) 3차원**으로 움직여야 합니다. 
회전 역시 마우스를 통한 Pitch, Yaw뿐만 아니라 몸체를 굴리는 Roll까지 3차원으로 돌아야 하므로, 입력값 자체가 3개의 좌표 값을 모두 담을 수 있는 **Vector3 (Axis3D)** 타입이어야 합니다.

<br>

### **Modifier 설정**
키보드 버튼 하나(Space, Shift, Q, E)는 기본적으로 X축(앞뒤)에 1.0이라는 값을 발생시킵니다. 이 1차원 값을 원하는 축으로 꺾어주기 위해 모디파이어(Modifier)를 사용합니다.

#### **상하 이동 (Z축)**
* **Space Bar (위)**: `Swizzle(ZYX)`
  * **이유**: 스페이스바를 누르면 기본적으로 X축 값이 들어옵니다. 상승은 Z축을 써야 하므로, **"X로 들어온 값을 Z축으로 돌려라!"** 라고 스위치를 바꿔준 것입니다.
* **Left Shift (아래)**: `Swizzle(ZYX)`, `Negate`
  * **이유**: 스페이스바와 마찬가지로 값을 Z축으로 보낸 뒤, 하강(아래) 방향으로 가야 하므로 부호를 뒤집어주는 **Negate(반전, -1)**를 걸어준 것입니다.

#### **회전 (Roll, Z축)**
`LookInput`의 X는 마우스 좌우(Yaw), Y는 마우스 상하(Pitch)로 이미 쓰고 있으므로 남은 것은 Z축(Roll)뿐입니다.
* **E 키 (우회전)**: `Swizzle(ZYX)`
  * **이유**: Q와 E를 누를 때 기본 X축으로 들어오는 값을 Roll에 해당하는 **Z축**으로 꽂아주기 위함입니다.
* **Q 키 (좌회전)**: `Swizzle(ZYX)`, `Negate`
  * **이유**: 값을 Z축으로 보낸 뒤, 우회전(E)과 반대 방향으로 회전해야 하므로 **Negate**를 추가한 것입니다.

<br>

### **키를 뗐을 때 멈추게 하는 `Completed` 함수 구현**
우리의 드론은 `CharacterMovement` 컴포넌트를 사용하지 못하므로(마찰력 없음), Tick마다 직접 위치값을 갱신하고 입력받은 값을 더해주는 과정을 거칩니다. 
따라서 `SetupPlayerInputComponent`에서 `ETriggerEvent::Completed`를 바인딩해 두어야 합니다. 키보드에서 손을 떼는 순간 **(0, 0, 0)**이라는 빈 벡터값이 `Move()` 함수로 들어오게 만들어서, `MoveInput`이 0이 되고 결국 물리 연산 곱하기가 0이 되면서 드론이 멈추게 됩니다.

<br>

### **Local(로컬) 좌표계로 계산하는 이유**
드론 자기 자신을 기준으로 한 앞, 뒤, 위, 아래를 정하기 위함입니다.
드론이 하늘을 향해 90도로 고개를 쳐들었다고 가정해 봅시다. 이때 'W(앞으로 가기)'를 누르면 드론은 **자기 머리 위(Local Forward)**로 날아가야 합니다. 만약 이걸 절대 좌표인 World로 계산했다면, 고개를 쳐들든 말든 무조건 북쪽(World X축)으로만 평행 이동했을 것입니다. 6자유도의 핵심은 Local입니다!

<br>

### **LookInput은 초기화하는데, MoveInput은 초기화 안 하는 이유 (핵심)**
* **`MoveInput` (키보드)**: WASD 키는 누르고 있으면 `Triggered`가 불리고, 손을 떼는 순간 `Completed`가 무조건 한 번 불려서 스스로 0으로 초기화해 줍니다.
* **`LookInput` (마우스)**: 마우스는 '현재 좌표'가 아니라 **'순간적인 이동 거리(Delta)'**를 뱉어냅니다. 마우스를 휙 움직이고 멈추면 손을 뗀 것이 아니기 때문에 `Completed`가 호출되지 않을 때가 많습니다. 
  * 따라서 `Tick`에서 이번 프레임 회전에 값을 써먹었으면, **"다 썼다!" 하고 수동으로 `LookInput = FVector::ZeroVector`로 비워주지 않으면** 남아있는 값 때문에 드론이 혼자서 팽이처럼 영원히 돌게 됩니다.

<br>

### **드론의 조작감을 위한 양력(Thrust)과 중력 구현**
단순한 깡통 비행체가 아니라 진짜 드론 같은 손맛을 내기 위해 중력과 양력을 커스텀 구현합니다.

* **상승 (Space)**: 강력한 UpSpeed가 더해져 중력 가속도를 씹어먹고 하늘로 솟구칩니다.
* **하강 (Shift)**: ShiftSpeed가 마이너스로 꽂히면서 기본 중력과 합쳐져 미친 듯이 아래로 다이브합니다.
* **호버링 (대기 상태)**: 공중에 가만히 있을 때는 중력의 힘을 받아 서서히 조금씩 아래로 떨어지게 하여 현실감을 줍니다.
* **에어 컨트롤 (마우스+W)**: 마우스로 기수를 위로 향하고 직진(W)할 때, Local 방향 이동 결괏값에 Z축 상승분이 포함되므로 중력을 이겨내고 비행기처럼 대각선 위로 부드럽게 날아갑니다. 반대로 아래를 향하면 중력을 타고 가속도가 붙어 빠르게 강하합니다.

## Custom Tick 함수 리팩토링: 클린 코드(Clean Code) 적용하기

게임 프로그래밍에서 `Tick` 함수는 방심하면 순식간에 수백 줄의 스파게티 코드가 되기 쉽습니다. 
이번에는 비대해진 틱 함수를 **'함수 추출(Extract Function)'** 기법을 통해 직관적이고 유지보수하기 좋게 리팩토링(Refactoring)해 보았습니다.

<br>

### 튜터님의 강의

> "짧은 함수는 **‘무엇을 하는지’를 명확히 보여주어** 코드를 쉽게 파악하게 해줍니다. 
> 주석이 필요하다고 느껴지는 부분은 따로 함수로 빼고, **의도가 드러나는 이름**을 붙이세요. 
> 협업을 해보면 아시겠지만, 실제로는 주석과 코드가 다를 때가 매우 많습니다!"

#### 나쁜 예시 (수백 줄짜리 만능 Tick)

과거에는 `Tick` 안에 모든 기능을 주석으로 구분해서 때려 넣었습니다. 코드가 길어지면 어떤 부분이 오류인지 찾기 매우 힘들어집니다.

```cpp
void AMyCharacter::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);
    // 1. 이동 처리 (수십 줄...)
    // 2. 점프 처리 (수십 줄...)
    // 3. 공격 처리 (수십 줄...)
    // 4. 버프/디버프 처리 (수십 줄...)
    // 5. 체력 체크 (수십 줄...)
    // 6. 애니메이션 업데이트 (수십 줄...)
    // ... (500줄이 넘어가요!)
}
```

<br>

#### 좋은 예시 (함수 추출 적용 후)

> 주석으로 설명하던 부분들을 아예 **명확한 동작을 나타내는 함수 이름**으로 만들어 직관성을 높였습니다.

**1. 깔끔해진 CustomTick 함수**  
이제 `CustomTick`의 4줄만 보고도  단번에 로직의 흐름을 이해할 수 있습니다.

```cpp
void AWeek3Drone::CustomTick(float FixedDeltaTime)
{
	UpdateRotation(FixedDeltaTime);               // 1. 회전 업데이트
	UpdateGroundDetection();                      // 2. 지면 감지
	UpdateGravityAndHovering(FixedDeltaTime);     // 3. 중력 및 호버링(Z축) 업데이트
	UpdateMovement(FixedDeltaTime);               // 4. 전후좌우(XY축) 이동 업데이트

	// 단발성 마우스 입력 초기화
	LookInput.X = 0.0f;
	LookInput.Y = 0.0f;
}
```

**2. 분리된 세부 구현 함수들**  
각 기능별 세부 로직은 별도의 함수 안으로 캡슐화하여, 필요한 기능만 찾아 수정할 수 있도록 분리했습니다.

```cpp
// =========================================================
// 분리된 세부 구현 함수들 (세부 로직 생략)
// =========================================================

// 1. 마우스 입력에 따른 회전 처리
void AWeek3Drone::UpdateRotation(float DeltaTime) { /* ... */ }

// 2. 라인트레이스를 이용한 바닥 감지
void AWeek3Drone::UpdateGroundDetection() { /* ... */ }

// 3. 상승, 하강 및 중력/호버링 적용
void AWeek3Drone::UpdateGravityAndHovering(float DeltaTime) { /* ... */ }

// 4. 지상 및 공중 상태에 따른 에어컨트롤 및 이동 속도 제어
void AWeek3Drone::UpdateMovement(float DeltaTime) { /* ... */ }
```

<br>

#### 리팩토링 결과

* **가독성 극대화** * 다른 프로그래머가 코드를 봤을 때, 뼈대가 되는 `Tick` 함수만 보고도 전체 로직의 흐름을 단번에 이해할 수 있습니다.
* **버그 추적 용이** * 만약 드론이 땅을 자꾸 파고드는 버그가 생겼다면? 수백 줄의 코드를 뒤질 필요 없이 딱 `UpdateMovement()` 함수 내부만 집중적으로 확인하면 됩니다.
* **재사용성** * 나중에 다른 기능을 추가할 때 (예: 점프 패드를 밟았을 때 등), 특정 로직(예: `UpdateGravityAndHovering`)만 따로 빼서 재사용하기가 훨씬 수월해집니다.

<br>

---

<br>

## [6번 과제] 동적 퍼즐 오브젝트 및 랜덤 스포너 시스템 구현

이번 과제에서는 단순한 액터 배치를 넘어, **프레임 독립성(DeltaTime)**을 보장하는 이동/회전 기믹과 성능 최적화를 고려한 **타이머 시스템**, 그리고 매번 새로운 경험을 주는 **동적 랜덤 스포너**를 구현했습니다.

<br>

### 1. 프레임 독립적인 동적 트랜스폼 제어 (Tick & DeltaTime)

게임은 유저의 PC 성능(FPS)에 따라 매 프레임 계산 주기가 달라집니다. 어떤 환경에서도 퍼즐 기믹이 일정한 속도로 움직이게 하려면 반드시 `DeltaTime`을 활용해야 합니다.

### 회전 로직 (`Week3Gear.cpp`)
기어(톱니바퀴)나 장애물은 제자리에서 회전해야 합니다. 절대 좌표가 아닌 액터 자신의 축을 기준으로 회전시키기 위해 `AddActorLocalRotation`을 사용했습니다.

```cpp
void AWeek3Gear::UpdateRotation(float DeltaTime)
{
    // RotationSpeed(회전 속도)에 프레임 지연 시간(DeltaTime)을 곱해 기기 성능과 무관하게 일정한 회전을 보장합니다.
    AddActorLocalRotation(RotationSpeed * DeltaTime);
}
```

<br>

### 2. 왕복 이동 로직
엘리베이터나 움직이는 발판은 지정된 범위를 넘어가지 않고 왕복해야 합니다. 시작 위치(`StartLocation`)를 기준으로 이동 거리를 계산하고, 최대 범위(`MaxRange`)에 도달하면 방향을 뒤집는 로직을 설계했습니다.

```cpp
void AWeek3MovePlatform::UpdateMovement(float DeltaTime)
{
    // 1. 현재 방향 * 속도 * 델타타임으로 이동량 계산
    FVector DeltaLocation = Settings.MoveDirection * Settings.MoveSpeed * DeltaTime;
    AddActorWorldOffset(DeltaLocation, true); // true: 스윕(Sweep) 충돌 검사 켬

    // 2. 기준점으로부터 얼마나 멀어졌는지 거리 계산
    float DistanceMoved = FVector::Distance(StartLocation, GetActorLocation());

    // 3. 설정한 범위를 벗어났다면?
    if (DistanceMoved >= Settings.MaxRange)
    {
        // 범위를 초과한 만큼 다시 뒤로 당겨주는 보정(Correction) 작업
        float Overshoot = DistanceMoved - Settings.MaxRange;
        FVector Correction = -Settings.MoveDirection * Overshoot;
        AddActorWorldOffset(Correction, true);

        // 이동 방향을 완전히 반대로 뒤집음 (*= -1.0f)
        Settings.MoveDirection *= -1.0f;

        // 새로운 왕복의 시작점이므로 현재 위치를 다시 기준점으로 갱신
        StartLocation = GetActorLocation();
    }
}
```

<br>

### 타이머 시스템 활용 (Tick 최적화)
"일정 시간 뒤에 폭발한다"는 기믹을 만들 때, `Tick`에서 매 프레임 시간을 빼는(`Time -= DeltaTime`) 방식은 비효율적입니다. 언리얼 엔진의 `TimerManager`를 사용하면 이벤트 기반으로 성능 낭비 없이 로직을 처리할 수 있습니다.

```cpp
// 1. Tick 함수를 아예 꺼버려서 불필요한 연산을 원천 차단합니다.
PrimaryActorTick.bCanEverTick = false; 

// 2. 구체(TriggerSphere)에 무언가 닿았을 때만 이벤트가 발생합니다.
void AWeek3DetectMine::OnOverlapBegin(...)
{
    if (!bIsTriggered && OtherActor && OtherActor != this)
    {
        bIsTriggered = true; // 중복 실행 방지
        
        // 3. 타이머 매니저를 통해 ExplosionDelay(설정된 대기 시간) 이후에 Explode() 함수를 딱 한 번(false) 실행하도록 예약합니다.
        GetWorld()->GetTimerManager().SetTimer(
            ExplosionTimerHandle, 
            this, 
            &AWeek3DetectMine::Explode, 
            Settings.ExplosionDelay, 
            false
        );
    }
}
```

<br>

### 동적 스폰 및 랜덤 속성 부여 
에디터에서 기믹을 하나씩 배치하는 수고를 덜고, 매 게임마다 다른 퍼즐 코스를 제공하기 위해 `SpawnActor`와 `FMath::RandRange`를 활용한 스포너를 제작했습니다.

#### 무작위 생성 로직
* **스폰 범위 제어:** `UBoxComponent`를 추가하여 스포너가 맵 어느 범위 안에서 퍼즐을 흩뿌릴지 눈으로 보고 설정할 수 있게 했습니다.
* **클래스 랜덤 추출:** 배열(`PlatformClassesToSpawn`)에 여러 기믹 클래스를 넣어두고, 인덱스를 무작위로 뽑아 스폰할 액터를 결정합니다.
* **독립적인 속성 부여:** 가장 중요한 부분입니다. 스폰된 액터가 '회전 기믹'인지 '이동 기믹'인지 캐스팅(`Cast`)으로 확인한 후, 각자 다른 속도와 방향을 무작위로 부여합니다.

```cpp
void AWeek3PuzzleSpawner::ApplyRandomSettings(AActor* SpawnedActor)
{
    // 만약 방금 스폰된 액터가 '이동 발판' 이라면?
    if (AWeek3MovePlatform* MovingPlatform = Cast<AWeek3MovePlatform>(SpawnedActor))
    {
        FMovePlatformSettings RandomSettings;
        RandomSettings.MoveSpeed = FMath::RandRange(100.0f, 800.0f); // 속도 랜덤
        RandomSettings.MaxRange = FMath::RandRange(500.0f, 1500.0f); // 이동 거리 랜덤
        RandomSettings.MoveDirection = FVector( // 이동 방향(XYZ) 완전 무작위
            FMath::RandRange(-1.0f, 1.0f),
            FMath::RandRange(-1.0f, 1.0f),
            FMath::RandRange(-1.0f, 1.0f)
        );
        MovingPlatform->SetPlatformSettings(RandomSettings);
    }
    // 만약 방금 스폰된 액터가 '회전 기어' 라면?
    else if (AWeek3Gear* Gear = Cast<AWeek3Gear>(SpawnedActor))
    {
        FRotator RandomSpin = FRotator(
            FMath::RandRange(-180.0f, 180.0f), // Pitch, Yaw, Roll 랜덤 회전
            FMath::RandRange(-180.0f, 180.0f),
            FMath::RandRange(-180.0f, 180.0f)
        );
        Gear->SetGearRotation(RandomSpin);
    }
}
```

<br>

### Struct를 활용한 설계 포인트
코드 내부를 보면 기믹의 속성들을 개별 변수로 흩어놓지 않고 `FMovePlatformSettings`나 `FMineTrapSettings` 같은 구조체 하나로 묶어서 관리했습니다. 

이렇게 설계하면 위 예시 코드처럼 **"설정값 통째로 넘겨주기 (`SetPlatformSettings`)"**가 가능해져, 스포너에서 무작위로 만든 세팅을 액터에게 주입할 때 코드가 매우 깔끔해지고 유지보수가 편리해집니다.


<br>

---

<br>


## 8번과제


### 흐름도

#### 핵심 목표 및 기본 규칙
* **게임 목표:** 제한 시간 동안 맵에 배치된 장애물과 지뢰를 피해 생존하며, **최대한 많은 점수(Score)를 획득**하는 서바이벌 퍼즐 아케이드입니다.
* **페널티:** 장애물 및 지뢰에 피격당할 경우 플레이어의 **체력(HP)이 감소**합니다.
* **진행 방식:** 총 3개의 레벨(Level)이 존재하며, 각 레벨은 레벨 전환 없이 연속된 3개의 웨이브(Wave)로 진행됩니다.
* **난이도 스케일링:** 웨이브가 진행될수록 **생성되는 전체 아이템과 장애물의 개수가 점점 많아져** 맵이 좁아지는 듯한 압박감을 줍니다.

<br>

### 웨이브별 제한 시간 
웨이브가 진행될수록 제한 시간이 짧아져, 더욱 빠르고 정확한 판단(밀도 높은 플레이)을 요구합니다.
* **Wave 1:** 60초
* **Wave 2:** 45초
* **Wave 3:** 30초

<br>

### 레벨 및 웨이브별 스폰 테이블
각 레벨과 웨이브에 따라 스폰되는 아이템 및 기믹의 종류가 누적되며 다채로워집니다.

| 구분 | Wave 1 (60초) | Wave 2 (45초) | Wave 3 (30초) |
| :--- | :--- | :--- | :--- |
| **Level 1** | 점수 배터리 | + 가시 함정 | + 힐링 배터리 |
| **Level 2** | 점수, 힐링, 기어(회전) | + 이동속도 저하(디버프) | + 가시 함정 |
| **Level 3** | 점수, 힐링, 기어, 슬로우, 가시, **조작 반전** | + **범위 지뢰 (AoE)** | Wave 2와 동일 (개수 최대치) |

<br>

#### 디버프 및 UI 시스템
플레이어를 방해하는 디버프 아이템(슬로우, 조작 반전)에 대한 처리는 다음과 같이 이루어집니다.

* **UI 시각화:** 플레이어가 현재 어떤 디버프에 걸려있는지, **남은 지속 시간은 얼마인지 HUD에 실시간 텍스트로 표시**하여 직관적인 피드백을 제공합니다.
* **디버프 중첩 (Stacking):** 디버프 지속 시간 도중 동일한 디버프 아이템을 또 획득할 경우, 페널티 수치가 커지는 것이 아니라 **남은 지속 시간에 새로운 시간이 누적(합산)**되어 페널티 시간이 길어지도록 설계되었습니다.

<br>

---

<br>

### 1. 멀티 웨이브 및 레벨 시스템 

레벨 전환 없이 한 맵에서 여러 웨이브를 처리하고, 조건 충족 시 다음 레벨로 넘어가는 완벽한 흐름을 구축했습니다.

* **동적 난이도 조절 (`Week3GameState.cpp`):**
  * `WaveDuration = BaseDuration - (CurrentWave - 1) * 15.0f;` 코드를 통해 웨이브가 진행될수록 제한 시간이 짧아지도록 설계했습니다.
* **스폰량 증가 (`Week3PuzzleSpawner.cpp`):**
  * `ObjectCount + (ArrayIndex * 10);` 코드를 적용하여 웨이브가 지날수록 아이템과 함정의 수가 늘어나 난이도가 점진적으로 상승합니다.
* **레벨 트랜지션 (`Week3GameInstance.cpp` & `GameState`):**
  * `UWeek3GameInstance`를 활용해 맵이 바뀌어도 `TotalScore`와 `CurrentLevelIndex`가 유지되도록 싱글톤(Singleton) 패턴의 이점을 살렸습니다. 3웨이브 클리어 시 자동으로 `Level_2`, `Level_3`로 넘어가며, 최종 클리어 시 UI를 띄워줍니다.

<br>

### 2. UI/UX 리뉴얼 및 기능 연결 

Tick을 남용하지 않고 타이머와 델리게이트 구조를 활용해 UI를 업데이트하는 최적화 기법이 적용되었습니다.

* **HUD 정보 통합 배치:**
  * `GameState::UpdateHUD()`를 통해 점수, 남은 시간, 체력, 현재 웨이브/레벨 수치를 한 번에 관리하고 위젯 텍스트를 갱신합니다.
* **입력 모드 및 커서 제어 (`Week3DroneController.cpp`):**
  * 게임 중에는 `SetInputMode(FInputModeGameOnly())`로 마우스를 숨기고 비행에 집중하며, 메인/결과 메뉴가 뜰 때는 `SetInputMode(FInputModeUIOnly())`와 `bShowMouseCursor = true`를 호출해 정확한 UI 인터랙션이 가능하게 만들었습니다.

<br>

### 3. 디버프 시스템 

아이템 획득 시 단순 점수 차감이 아닌, 플레이어의 조작을 방해하는 2가지 디버프를 구현했습니다.

* **SlowingItem (이동 속도 감소):** `UpdateMovement`에서 `bIsSlowed`가 참일 경우 `MoveSpeed * 0.5f` 연산을 통해 실시간으로 속도를 반감시킵니다.
* **ReverseControlItem (조작 반전):** `LocalInput *= -1.0f;` 단 한 줄의 깔끔한 수학적 논리로 W, A, S, D의 이동 방향을 완벽히 뒤집었습니다.
* **중첩(Stack) 시스템 적용:** 이미 디버프에 걸린 상태에서 또 먹었을 경우, `RemainingTime + Duration` 로직을 통해 타이머의 남은 시간에 새로운 시간을 누적(Stack)시키는 디테일을 챙겼습니다.
* **UI 시각화:** HUD 업데이트 로직에서 디버프 남은 시간이 0 이상일 때만 화면에 경고 텍스트(`Slow! : 5.0 s`)를 띄웁니다.

<br>

### 4. 웨이브별 환경 변화 및 동적 난이도 

맵의 환경이 계속 변하는 느낌을 데이터 테이블과 UI를 통해 훌륭하게 구현했습니다.

* **데이터 테이블 스와핑 (`Week3PuzzleSpawner.cpp`):**
  * 배열 `WaveDataTables`를 사용하여 1웨이브는 기본 아이템, 2웨이브는 스파이크, 3웨이브는 지뢰 테이블로 교체하며 스폰합니다. 하드코딩 없이 기획자가 데이터만 바꾸면 난이도가 조절되는 데이터 기반 아키텍처입니다.
* **지뢰 폭발 기믹 (`Week3DetectMine.cpp`):**
  * 밟는 순간 `ExplosionDelay` 타이머가 돌며, 붉은색 장판(`RangeIndicator`)을 표시하여 플레이어에게 회피할 기회(인지적 단서)를 제공합니다.
* **이벤트 알림 UI (`Week3GameState.cpp`):**
  * `GetWaveEventMessage`를 통해 각 레벨/웨이브 시작 시 "스파이크가 나타났습니다!", "조작 반전 아이템을 조심하세요!" 등의 문구를 4초간 화면 중앙에 띄워 목표를 환기시킵니다.

<br>

### 5. 3D 위젯 및 위제 애니메이션 활용

* **블루프린트 애니메이션 호출 (`Week3DroneController.cpp`):**
  * C++에서 블루프린트로 만든 위젯 애니메이션을 실행하기 위해 `FindFunction(FName("PlayGameOverAnim"))` 와 `ProcessEvent` 리플렉션 기능을 사용하여 멋지게 연동했습니다.
* **3D 위젯 적용 (`Week3Drone.cpp`):**
  * 캐릭터 머리 위에 띄우기 위해 `UWidgetComponent`를 부착했습니다. Space를 `Screen`으로 설정하여 카메라 거리에 상관없이 체력 가독성을 유지하게 만든 점이 돋보입니다. (`UpdateOverheadHP()`)

<br>


###  6. 확장성을 고려한 아이템 시스템 

모든 아이템의 공통 동작(충돌 감지, 태그 확인, 파괴)을 부모 클래스에서 처리하고, 개별 아이템은 '무슨 효과를 낼 것인가'만 집중하도록 캡슐화했습니다.

* **공통 로직 통합 (`AWeek3BaseItem`):**
  * `CapsuleComponent`를 루트로 생성하고 `OnComponentBeginOverlap` 이벤트를 부모 클래스에서 미리 바인딩해 두었습니다.
  * 플레이어(`Player` 태그)가 닿았을 때만 가상 함수인 `ActivateItem()`을 호출하도록 설계하여, 자식 클래스에서 충돌 검사 코드를 매번 짤 필요가 없게 만들었습니다.
* **긍정적 아이템 구현 (`Healing_Battery`, `Score_Battery`):**
  * 부모의 `ActivateItem()`을 오버라이딩하여, 힐링 배터리는 `AddHealth()`를, 스코어 배터리는 `GameState->Add_Score()`를 호출하고 자신을 파괴(`DestroyItem`)하는 깔끔한 구조를 가집니다.

<br>

### 디버프 아이템 구현

단순한 수치 변화를 넘어, 플레이어의 조작을 직접적으로 방해하는 상태 이상 아이템을 구현했습니다.

* **모듈화된 디버프 부여:**
  * 아이템 클래스 내부에서 복잡한 타이머를 돌리지 않고, 플레이어(`Week3Drone`) 객체에 구현된 `ApplySlow()`와 `ApplyReverse()` 함수에 **지속 시간(Duration)**만 전달하는 방식으로 책임 소재를 명확히 분리했습니다.
* **`ApplySlow` (이동 속도 50% 감소):** 드론의 속도를 반감시킵니다.
* **`ApplyReverse` (조작 반전):** 드론의 방향 입력 벡터에 `-1`을 곱해 W/A/S/D 입력을 완벽히 뒤집습니다.

<br>

### 7. 아키텍처: 함정 부모 클래스

함정 시스템에서 가장 돋보이는 부분은 **자식 클래스의 충돌체를 자동으로 인식하여 이벤트를 바인딩하는 동적 처리 로직**입니다.

```cpp
// Week3TrapBase.cpp 의 BeginPlay 내부
TArray<UShapeComponent*> AllCollisions;
GetComponents<UShapeComponent>(AllCollisions);

for (UShapeComponent* Collision : AllCollisions)
{
    if (Collision)
    {
        Collision->OnComponentBeginOverlap.AddDynamic(this, &AWeek3TrapBase::OnTrapOverlap);
    }
}
```

* **설계 포인트 (다형성 활용):** * 자식 클래스가 어떤 충돌체(Box, Sphere, Capsule)를 사용할지 부모는 알 수 없습니다.
  * 따라서 `BeginPlay`에서 `GetComponents<UShapeComponent>`를 통해 자신의 하위에 있는 **모든 형태의 충돌체를 싹 다 긁어모은 뒤, 자동으로 오버랩 이벤트를 연결**해 줍니다.
  * 이 덕분에 자식 클래스는 충돌체 모양만 선언하면 추가적인 이벤트 바인딩 없이 즉시 함정으로 작동하게 됩니다. (매우 세련된 최적화 기법입니다!)

<br>

### 8. 지뢰 폭발 및 시각화
단순히 닿으면 데미지를 입는 함정이 아니라, 밟는 순간 타이머가 작동하며 폭발을 예고하는 입체적인 기믹을 구현했습니다.

* **인지적 단서 (Telegraphing) 제공:**
  * 밟는 순간 감춰져 있던 빨간색 장판(`RangeIndicator`)의 `Visibility`를 켜서 플레이어에게 폭발 범위를 시각적으로 경고합니다.
* **타이머 기반 범위 피해 (AoE):**
  * `SetTimer`를 통해 `ExplosionDelay`(예: 3초) 뒤에 `Explode()` 함수가 호출됩니다.
  * 폭발 시점에 `TriggerSphere->GetOverlappingActors`를 호출하여, **도망치지 못하고 범위 안에 남아있는 모든 플레이어**에게 일괄적으로 `ApplyDamage`를 입히는 완벽한 광역 공격 로직을 구현했습니다.


### 9. 추가 구현

과제 요구사항을 넘어서, 실제 게임 출시에 필요한 디테일과 안정성 코드가 다수 포함되어 있습니다.

1. **무적 판정 (I-Frames) 구현:**
   * 지뢰나 스파이크에 연속으로 닿아 즉사하는 불합리함을 막기 위해 `TakeDamage` 시 `bIsInvincible = true;`를 켜고, 2초 뒤 타이머로 해제하는 정석적인 무적 프레임 기법을 적용했습니다.
2. **UI 최적화 (Tick 대체):**
   * UI 글자를 바꾸기 위해 `Tick`을 쓰지 않고, `SetTimer(HUDUpdateTimerHandle, ..., 0.1f, true)`를 사용하여 0.1초마다 UI를 갱신하게 만들어 불필요한 프레임 낭비를 막았습니다.
3. **확률 기반 가중치 랜덤 스폰:**
   * 스포너가 단순히 랜덤으로 아이템을 뽑는 것이 아니라, `SpawnChance` 가중치를 전부 합친 뒤 `FRandRange`를 통해 **확률에 기반한 아이템 뽑기 (가챠 시스템 로직)**를 직접 구현(`GetRandomItemFromTable`)한 점은 매우 수준 높은 코드입니다.

