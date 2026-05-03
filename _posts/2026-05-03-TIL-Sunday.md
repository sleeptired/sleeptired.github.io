---
title: "2026-05-03 TIL (Weekend)"
date: 2026-05-03 08:00:00 +0900
categories: ["TIL(Today I Learned)", "2026-05-03"]
tags: [TIL, "2026", 내일배움캠프, Unreal, Weekend]
math: true
---

# 언리얼 엔진 데미지 시스템 및 UDamageType 활용

* **ApplyDamage 와 TakeDamage 관계**

두 함수의 관계는 **"데미지 발송인(가해자)"**과 **"데미지 수령인(피격자)"**의 관계입니다. 언리얼 엔진이 중간에서 중개자 역할을 하여 이 둘을 연결해 줍니다.

---

## 데미지 송수신 함수 분류
데미지를 주는 방식에 따라 호출하는 함수와 전달되는 이벤트 구조체가 다릅니다. 상황에 맞게 선택하여 사용합니다.

| 구분 | 송신 함수 (가해자) | 수신 이벤트 (피격자) | 데이터 구조체 (C++) | 주요 활용 사례 |
| :--- | :--- | :--- | :--- | :--- |
| **Any** | `ApplyDamage` | `TakeDamage` | `FDamageEvent` | 독, 화상, 단순 체력 감소 |
| **Point** | `ApplyPointDamage` | `TakeDamage` | `FPointDamageEvent` | 총기(헤드샷), 칼부림 (타격 위치 및 방향 필요) |
| **Radial** | `ApplyRadialDamage` | `TakeDamage` | `FRadialDamageEvent` | 폭발, 충격파 (거리에 따른 데미지 감쇄 필요) |

<br>

### `UGameplayStatics::ApplyDamage` 매개변수 분석

```cpp
float UGameplayStatics::ApplyDamage(
    AActor* DamagedActor, 
    float BaseDamage, 
    AController* EventInstigator, 
    AActor* DamageCauser, 
    TSubclassOf<UDamageType> DamageTypeClass
);
```

| 매개변수 이름 | 타입 | 의미 (역할) | (예시) |
| :--- | :--- | :--- | :--- |
| **`DamagedActor`** | `AActor*` | **누구를 때릴 것인가? (피격자)**<br>데미지를 받을 대상입니다. | 트레이스에 맞은 대상(`HitActor`)이나, 충돌체에 닿은 대상(`OtherActor`)을 그대로 넣습니다. |
| **`BaseDamage`** | `float` | **얼마나 아프게 때릴 것인가? (피해량)**<br>깎을 데미지의 기본 수치입니다. | `10.f`, `50.f` 등 무기나 스킬의 데미지 수치를 넣습니다. |
| **`EventInstigator`** | `AController*` | **누가 지시했는가? (배후/조종자)**<br>이 공격을 지시한 플레이어(PC)나 AI 컨트롤러입니다. 킬 로그를 띄우거나 어그로(타겟팅)를 끌 때 핵심이 됩니다. | 공격을 실행한 캐릭터의 컨트롤러(`GetController()`, `GetInstigatorController()`)를 넣습니다. 함정(Trap)처럼 주인이 없다면 `nullptr`을 넣기도 합니다. |
| **`DamageCauser`** | `AActor*` | **무엇으로 때렸는가? (직접적인 흉기)**<br>피격자와 실제로 물리적인 접촉을 한 액터입니다. | 총알 액터 그 자체, 휘두른 칼 액터, 또는 타격 판정을 가진 캐릭터 자신(`this`)을 넣습니다. |
| **`DamageTypeClass`** | `TSubclassOf<UDamageType>` | **어떤 속성으로 때렸는가? (데미지 타입)**<br>불, 독, 물리 타격 등 데미지의 성격을 규정합니다. 피격자 측에서 분기 처리를 할 때 쓰입니다. | 기본 데미지라면 `UDamageType::StaticClass()`를 넣고, 화염 속성이라면 직접 만든 `UFireDamageType::StaticClass()`를 넣습니다. |

* DamageTypeClass는 언리얼 엔진에서 C++ 클래스를 생성할 때 부모 클래스로 선택할 수 있는 `UDamageType`으로 생성해 사용 가능하다.

<br>

---

<br>

## UDamageType 엔진 기본 헤더

## 클래스 설계 철학 (분석)

엔진 헤더 상단의 주석을 보면 아주 중요한 아키텍처 규칙이 적혀 있습니다.

> *"DamageTypes are **never instanced** and should be treated as **immutable data holders**"*

* **해석:** 데미지 타입은 게임 중에 `NewObject` 등으로 매번 새롭게 찍어내서(Instanced) 쓰는 것이 아닙니다. 런타임 중에 상태나 값이 변하지 않는(Immutable) **'순수한 데이터 보관소(CDO)'**로만 다루어야 합니다.
* **이유:** 앞서 데미지를 받을 때 `GetDefaultObject()`를 써서 원본만 참조했던 이유가 바로 엔진 개발자들의 이 설계 철학 때문입니다. 메모리에 원본 딱 하나만 올려두고 모두가 그것을 돌려보며 참조하는 것이 가장 빠르고 효율적이기 때문입니다.

<br>

## 멤버 변수(Property)

이 클래스는 단순한 데미지 수치(HP 깎기) 외에, **'물리 엔진(RigidBody)'**과 **'파괴 시스템(Destruction)'**에 특화된 설정들을 기본적으로 제공합니다.

| 변수명 | 타입 | 설명 및 실무 활용 포인트 |
| :--- | :--- | :--- |
| **`bCausedByWorld`** | `uint32 : 1` | **월드(환경) 데미지 여부**<br>용암에 빠지거나, 낙사하거나, 맵의 가시에 찔렸을 때 `true`로 씁니다. AI가 피격을 당했을 때 "누가 날 때렸어!?" 하고 범인을 찾는 어그로 로직을 실행하지 않도록 차단할 때 유용합니다. |
| **`bScaleMomentumByMass`** | `uint32 : 1` | **질량 비례 넉백 여부**<br>맞고 밀려날 때, 뚱뚱한 캐릭터(Mass가 큼)는 조금만 밀려나고, 가벼운 캐릭터는 멀리 날아가게끔 물리 엔진이 무게를 계산할지 결정합니다. |
| **`bRadialDamageVelChange`** | `uint32 : 1` | **폭발 밀림 방식을 '속도 변화'로 할지 여부**<br>수류탄 같은 폭발(Radial) 데미지를 줄 때, 밀어내는 힘을 '가속도(질량 비례)'로 줄지, 아니면 아예 강제로 '속도(Velocity)'를 즉시 바꿔버릴지 결정합니다. (`true`면 코끼리나 쥐나 똑같은 속도로 강제로 튕겨 나갑니다.) |
| **`DamageImpulse`** | `float` | **타격 넉백(충격량) 세기**<br>물리엔진이 켜진 물체나 래그돌(Ragdoll) 상태로 쓰러진 캐릭터를 얼마나 세게 퍽! 하고 밀어낼지 정하는 절대적인 수치입니다. |
| **`DestructibleImpulse`** | `float` | **파괴 가능 물체(Destructible) 파편 충격량**<br>유리창이나 나무 상자 같은 파괴 가능한 메쉬를 부수었을 때, 부서진 파편 조각들이 폭발하듯 얼마나 멀리 튀어나갈지 결정합니다. |
| **`DestructibleDamageSpreadScale`** | `float` | **파괴 가능 메쉬 데미지 전파 범위**<br>유리창의 한가운데를 쐈을 때, 데미지가 주변으로 얼마나 쫙 퍼져서(Spread) 전체적으로 금이 가게 할지 정하는 전파 비율입니다. |
| **`DamageFalloff`** | `float` | **폭발 데미지 거리별 감소 곡선(지수)**<br>수류탄의 폭심지에서 멀어질수록 데미지가 어떻게 줄어들지 정합니다. (예: `1.0` = 거리에 따라 선형적으로 일정하게 줄어듦, `2.0` = 거리가 멀어지면 데미지가 제곱으로 급격히 뚝 떨어짐) |

<br>

> **C++: `uint32 변수명 : 1` 이란 무엇일까요? (비트 필드)**
> `bool` 타입을 쓰지 않고 왜 저렇게 썼을까요? C++의 극한의 메모리 최적화 기법인 **'비트 필드(Bit Field)'**입니다. 
> 원래 `bool` 타입 하나를 선언하면 메모리에서 최소 1바이트(8비트)를 차지합니다. 하지만 저렇게 선언하면 `uint32`(32비트) 공간 딱 하나를 쪼개서 **"이 변수는 1비트만 쓸게!"**라고 선언하는 것입니다. 변수 3개가 모여있어도 3바이트가 아니라 3비트만 차지하게 됩니다. 
> 게임 엔진처럼 0.001초라도 아껴야 하는 코어 시스템에서는 이런 세심한 최적화가 필수적입니다!
