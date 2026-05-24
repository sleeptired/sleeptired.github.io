---
title: "2026-05-24 TIL (Weekend)"
date: 2026-05-24 08:00:00 +0900
categories: ["TIL(Today I Learned)", "2026-05-24"]
tags: [TIL, "2026", 내일배움캠프, C++, Unreal, DigitalTwin, 팀프로젝트, Weekend]
math: true
---

# [Trouble Shooting] 자율주행 차량 정차 시 브레이크 등 미점등 버그 및 데드존(Deadzone) 함정 해결

## 1. 문제 발생 (Issue)
팀원이 구현한 자율주행 NPC 차량이 앞차를 발견하고 충돌 없이 잘 멈추긴 하지만, **완전히 정차했을 때 브레이크 등이 들어오지 않는 현상**에 대한 디버깅 및 해결을 요청받았다.

---

##  2. 1차 시도
처음에는 단순하게 "브레이크 값이 들어오면 불을 켜면 되겠다"고 생각하여 아래와 같이 물리 기반 브레이크 수치(`ActualBrake`)를 읽어와 점등하도록 코드를 수정했다.

```cpp
if (ChaosVehicleMovement)
{
    float ActualBrake = ChaosVehicleMovement->GetBrakeInput();
    if (ActualBrake > 0.0f || bIsInTunnel)
    {
        BrakeLights(true);
    }
    else
    {
        BrakeLights(false);
    }
}
```

### [문제 발생]

* **터널 연동 해제:** 상시 켜져 있어야 할 터널 안에서도 브레이크를 밟지 않으면 불이 꺼지는 현상 발생.
* **여전히 켜지지 않는 정차 브레이크 등:** 목적지에 도착해서 정차했을 때, 화면의 디버그 메시지에는 분명히 `Brake: 1.00`이라고 찍혀있는데 실제 차량의 불은 켜지지 않는 현상이 발생했다.

![버그 화면](/assets/img/Brake_Bug.jpg)

---

## 3. 원인 분석 (
도대체 왜 브레이크 값이 1.0인데 불이 안 켜지는지 원인을 추적하기 시작했다.

### 1) 디버그 메시지의 함정 (화면 덮어쓰기 현상)
화면에 찍힌 `Brake: 1.00`의 정체는 내 차(플레이어)가 아니라 앞서 정차해 있던 파란색 NPC 차량의 데이터였다.

* `GEngine->AddOnScreenDebugMessage(1, ...)` 처럼 고유 키(Key) 값을 1로 고정해 두었기 때문에, 맵에 존재하는 여러 대의 차량이 매 프레임 같은 줄에 데이터를 덮어쓰고 있었다.
* 즉, 가장 마지막에 연산된 NPC 차량의 수치만 화면에 남았던 것이다.

 **해결:** `if (GEngine && ChaosVehicleMovement && IsPlayerControlled())` 조건을 추가하여 플레이어 차량의 진짜 브레이크 값을 확인해보니, 놀랍게도 `ActualBrake` 값은 `0.0`이었다. 코드는 100% 정상 작동 중이었고, 애초에 차가 브레이크를 안 밟고 있던 것이다.

![해결 사진](/assets/img/Brake_Bug_Solve.png)

### 2) 자율주행 제어 로직의 허점: "관성 주행(Deadzone)의 함정"
왜 차가 멈췄는데 브레이크 값이 0.0일까? 팀원의 자율주행 로직(`CheckVehicleAhead`)과 내가 작성한 가감속 제어 로직(`ApplySpeedCommand`)이 맞물리면서 발생한 논리적 오류였다.

* **강제 0 반환:** 앞차와의 거리가 안전 마진(`StopGap`) 이하로 가까워지면 AI는 목표 속도(`TargetSpeed`)를 `0.f`로 설정한다.
* **속도 동기화:** 차가 서서히 감속하여 실제 내 차의 속도(`CurrentSpeed`)도 0에 도달한다.
* **데드존 진입:** 명령값(`Cmd`) = 목표 속도(0) - 실제 속도(0) = 0이 된다.

> **주행로직의 오류:** "목표 속도가 0인데 내 차도 지금 0이네? 이제 엑셀도 브레이크도 밟을 필요 없이 페달에서 발을 떼야지!"

결과적으로 AI는 정차 상태 유지를 위한 브레이크를 밟지 않고, 관성 주행(데드존) 상태로 빠져버려 페달에서 발을 뗀 것이다. 브레이크를 밟지 않았으니 당연히 브레이크 등도 켜지지 않았다.

---

## 💡 4. 문제 해결 (Resolution)
팀원의 `CheckVehicleAhead` 로직을 임의로 변경하지 않고 문제를 해결하기 위해, 차량에 제어 명령을 내리는 최종 컨트롤 타워인 `ApplySpeedCommand` 최상단에서 정차 신호(`0.f`)를 가로채는(Intercept) 예외 처리를 추가했다.

목표 속도가 0에 수렴하면 아예 데드존 계산 로직을 타지 못하게 막고, 강제로 물리 브레이크를 100% 밟도록 수정했다.

### 최종 수정 코드 (SplineFollowerComponent.cpp)

```cpp
void USplineFollowerComponent::ApplySpeedCommand(float TargetSpeed, float CurrentSpeed)
{
	ATeam24VehiclePawn* Pawn = OwnerPawn.Get();
	if (!Pawn) return;

	// [관성 주행 차단 예외 처리] 
	// CheckVehicleAhead 등에서 앞차나 목적지 때문에 목표 속도를 0(10.f 미만)으로 만들었다면,
	// 속도 차이(Cmd) 계산을 생략하고 즉시 물리 브레이크를 100% 밟아서 차를 고정합니다.
	if (TargetSpeed < 10.f) 
	{
		Pawn->DoThrottle(0.f);  // 엑셀 떼기
		Pawn->DoBrake(1.f);     // 물리 브레이크 100% 명시적 작동!
		Pawn->DoBrakeStart();   // 브레이크 등 점등
		return;                 // 아래의 데드존 로직을 타지 않고 즉시 종료
	}

	// ... 이하 기존 가감속 및 데드존(Deadzone) 주행 로직 정상 수행 ...
}
```

## 결론 및 요약
* **데이터 시각화의 오류 인지:** 디버그 메시지가 항상 내가 원하는 타겟의 데이터만을 보여준다는 편견을 깨고, `IsPlayerControlled()`를 통해 정확한 객체의 데이터를 추적하는 습관의 중요성을 깨달았다.
* **PID 제어(가감속) 로직의 이해:** 목표값과 현재값의 오차가 0이 될 때, 시스템이 '완전 정지'가 아니라 '입력 없음(관성)' 상태로 빠질 수 있다는 점을 파악하고, 하드코딩된 예외 처리(Intercept)를 통해 차가 밀리거나 브레이크 등이 꺼지는 현상을 해결했다.
