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
