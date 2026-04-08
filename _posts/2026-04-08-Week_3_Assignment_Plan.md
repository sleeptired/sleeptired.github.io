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
> 1. **`Character` 클래스 사용 금지 🚫**
>    * 플레이어는 반드시 **`Pawn` 클래스**를 베이스로 생성해야 합니다. (즉, 엔진이 기본으로 제공하는 `CharacterMovementComponent`를 사용할 수 없음)
> 
> 2. **기본 카메라/이동 제어 함수 사용 금지**
>    * `AddControllerYawInput()`, `AddControllerPitchInput()` 등 언리얼 엔진이 기본 제공하는 컨트롤러 회전 함수를 사용해선 안 됩니다.

<br>

### 계획
조건에 따라, 키보드 및 마우스 입력값을 단순히 엔진 함수에 넘겨주는 것이 아니라, **DeltaTime과 벡터(Vector), 로테이터(Rotator)를 활용하여 트랜스폼(Transform)의 위치와 회전값을 직접 수학적으로 계산하고 갱신(SetActorLocation, SetActorRotation)하는 방식**으로 구현할 예정입니다.

## SpringArmComp->bUsePawnControlRotation = false; 하는이유

## IA_Move과 IA_Look Axis3D로 설정

# 이유 조사할 리스트

## Space Bar (위): Modifier -> Swizzle Input Axis Values (Order를 ZYX로 설정. X축 입력을 Z축으로 돌려줍니다) 이유?
## Left Shift (아래): Modifier -> Swizzle Input Axis Values (ZYX), Negate (Z축 -1) 이유?

## E 키 (Roll 우회전): Modifier -> Swizzle Input Axis Values (ZYX) 이유?
## Q 키 (Roll 좌회전): Modifier -> Swizzle Input Axis Values (ZYX), Negate 이유?

## CharacterMovement를 사용하지 못하므로 Tick마다 위치값을 갱신하고 입력 받은 값을 더하고 그런과정을 거쳐야한다 그래서 SetupPlayerInputComponent에서 키를 뗐을 때 멈추게 하는 함수를 구현해야함

