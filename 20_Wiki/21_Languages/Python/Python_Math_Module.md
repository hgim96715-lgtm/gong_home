---
aliases:
  - Python Math
  - math.prod
  - 수학 함수
tags:
  - Python
related:
  - "[[Python_Random_Seed|랜덤 수 생성 (random)]]"
  - "[[Python_Builtin_Functions|기본 내장 함수 (sum, max, min)]]"
  - "[[00_Python_HomePage]]"
---
## 개요: 복잡한 산술 연산을 위한 `math` 모듈 

기본 연산자(`+`, `-`, `*`, `/`)만으로 부족한 정밀한 계산이나 누적 연산을 처리합니다.

- **문법**: `{python}import math` 를 먼저 선언해야 합니다.

---
##  주요 함수 (실무 활용)

### ① 누적 곱 (`math.prod`) ✖️

리스트나 튜플 안에 있는 모든 숫자를 싹 다 곱합니다. (Python 3.8+)

- **문법**: `{python}math.prod( <리스트> )`
- **활용**: 장비 가동률 계산, 확률 누적 계산

```python
import math

rates = [0.9, 0.8, 0.95]
total_efficiency = math.prod(rates) 
# 결과: 0.684 (0.9 * 0.8 * 0.95)
```

### ② 올림과 내림 (`ceil`, `floor`) 

소수점 데이터를 정수로 변환할 때 사용합니다. (데이터 전처리 필수)

- **`math.ceil(x)`**: 올림 (천장)
- **`math.floor(x)`**: 내림 (바닥)
- **`math.trunc(x)`**: 소수점 버림 (0 쪽으로 붙임)

### ③ 최대공약수와 최소공배수 (`gcd`, `lcm`)

배치 작업의 주기를 맞추거나 데이터를 균등하게 나눌 때 씁니다.

- **`math.gcd(a, b)`**: 최대공약수 (Greatest Common Divisor)
- **`math.lcm(a, b)`**: 최소공배수 (Least Common Multiple, Python 3.9+)

---
## 요약 표 (Cheat Sheet)

| **함수**              | **설명**           | **비유**             |
| ------------------- | ---------------- | ------------------ |
| **`prod(list)`**    | 리스트 내 모든 수의 곱    | **누적 곱하기**         |
| **`ceil(x)`**       | x보다 크거나 같은 최소 정수 | **무조건 올림**         |
| **`floor(x)`**      | x보다 작거나 같은 최대 정수 | **무조건 내림**         |
| **`sqrt(x)`**       | x의 제곱근           | **루트 씌우기**         |
| **`isclose(a, b)`** | 두 실수가 근접한지 확인    | **부동소수점 오차 무시 비교** |