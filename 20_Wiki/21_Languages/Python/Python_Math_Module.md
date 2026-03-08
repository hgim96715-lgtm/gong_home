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
# Python_Math_Module

## 개념 한 줄 요약

> "기본 연산자(+, -, /)만으로 부족한 정밀한 계산이나 누적 연산을 처리하는 표준 라이브러리."

```python
import math  # 먼저 선언 필수
```

---

---

# ① 누적 곱 — math.prod ✖️

> 리스트나 튜플 안에 있는 모든 숫자를 곱한다. (Python 3.8+)

```python
math.prod(리스트)
```

```python
import math

rates = [0.9, 0.8, 0.95]
total_efficiency = math.prod(rates)
# 결과: 0.684  (0.9 * 0.8 * 0.95)
```

```
활용:
- 장비 가동률 누적 계산
- 확률 누적 계산
```

---

---

# ② 올림 / 내림 / 버림 — ceil, floor, trunc

> 소수점 데이터를 정수로 변환할 때 사용. 데이터 전처리 필수.

```python
math.ceil(x)   # 올림 — x 보다 크거나 같은 최소 정수
math.floor(x)  # 내림 — x 보다 작거나 같은 최대 정수
math.trunc(x)  # 버림 — 소수점 제거, 0 쪽으로 붙임
```

```python
x = 3.7

math.ceil(x)   # → 4   (무조건 올림)
math.floor(x)  # → 3   (무조건 내림)
math.trunc(x)  # → 3   (소수점 버림)

x = -3.7
math.ceil(x)   # → -3  (0 방향으로 올림)
math.floor(x)  # → -4  (0 반대 방향으로 내림)
math.trunc(x)  # → -3  (0 쪽으로 버림)
```

```
ceil  vs  trunc:
양수일 땐 동일하게 보임
음수일 때 차이가 생김 → -3.7 기준
  ceil  = -3  (0 방향)
  trunc = -3  (동일)
  floor = -4  (0 반대 방향)
```

---

---

# ③ 최대공약수 / 최소공배수 — gcd, lcm

> 배치 작업 주기 맞추기, 데이터를 균등하게 나눌 때 활용.

```python
math.gcd(a, b)  # 최대공약수 (Greatest Common Divisor)
math.lcm(a, b)  # 최소공배수 (Least Common Multiple, Python 3.9+)
```

```python
math.gcd(12, 8)  # → 4
math.lcm(4, 6)   # → 12
```

## 활용 — 분수 덧셈 직접 구현

```python
import math

def add_fractions(a_num, a_den, b_num, b_den):
    # ① 공통 분모 = 두 분모의 최소공배수
    common_den = math.lcm(a_den, b_den)

    # ② 분자를 공통 분모 기준으로 변환 후 합산
    total_num = (a_num * (common_den // a_den)) + (b_num * (common_den // b_den))

    # ③ 결과 약분: 분자·분모의 최대공약수로 나눔
    g = math.gcd(total_num, common_den)
    return total_num // g, common_den // g

result = add_fractions(1, 3, 1, 4)  # 1/3 + 1/4
print(result)  # → (7, 12)  즉 7/12
```

> `fractions.Fraction` 을 쓰면 자동 처리되지만, `gcd` / `lcm` 으로 직접 구현하면 내부 동작 원리와 성능을 직접 제어할 수 있다. → [[Python_Fractions_Module]] 참고

---

---

# ④ 팩토리얼 — math.factorial

> n! = 1 × 2 × 3 × ... × n 순열/조합 계산의 기반.

```python
math.factorial(n)
```

```python
math.factorial(5)  # → 120  (5 * 4 * 3 * 2 * 1)
math.factorial(0)  # → 1    (0! = 1 by definition)
math.factorial(1)  # → 1
```

```
활용:
- 순열 계산: nPr = n! / (n-r)!
- 조합 계산: nCr = n! / (r! * (n-r)!)
- 확률 계산에서 경우의 수
```

---

---

# ⑤ 조합 — math.comb

> nCr = n개 중 r개를 **순서 없이** 뽑는 경우의 수. `factorial` 로 직접 계산하는 것보다 빠르고 오버플로우 안전.

```python
math.comb(n, r)
```

```python
math.comb(5, 2)   # → 10   (5개 중 2개 선택)
math.comb(10, 3)  # → 120
math.comb(5, 0)   # → 1
math.comb(5, 5)   # → 1
```

```python
# 로또 당첨 확률 분모 (45개 중 6개 선택)
math.comb(45, 6)  # → 8,145,060
```

```
math.comb vs math.factorial 로 직접 계산:
math.comb(10, 3)                              # 간단 ✅
math.factorial(10) // (math.factorial(3) * math.factorial(7))  # 장황 ❌
```

---

---

# ⑥ 순열 — math.perm

> nPr = n개 중 r개를 **순서 있게** 뽑는 경우의 수. (Python 3.8+)

```python
math.perm(n, r)
```

```python
math.perm(5, 2)   # → 20   (5개 중 2개를 순서 있게)
math.perm(5, 5)   # → 120  (= 5!)
math.perm(5)      # → 120  (r 생략 시 n! 과 동일)
```

```
조합 vs 순열:
math.comb(5, 2) = 10   → {A,B} 와 {B,A} 를 같은 것으로 봄
math.perm(5, 2) = 20   → {A,B} 와 {B,A} 를 다른 것으로 봄
```

---

---

# ⑦ 제곱근 — math.sqrt

```python
math.sqrt(x)
```

```python
math.sqrt(16)   # → 4.0
math.sqrt(2)    # → 1.4142135623730951
```

```
주의: 결과가 float 로 반환됨
정수로 필요하면 int(math.sqrt(x)) 또는 x ** 0.5 사용
```

---

---

# ⑧ 부동소수점 비교 — math.isclose

> 실수 연산에서 발생하는 미세한 오차를 무시하고 두 값이 같은지 비교.

```python
math.isclose(a, b, rel_tol=1e-9)
```

```python
0.1 + 0.2 == 0.3        # → False  ← 부동소수점 오차
math.isclose(0.1 + 0.2, 0.3)  # → True  ✅
```

---

---

# 전체 치트시트

|함수|설명|비고|
|---|---|---|
|`prod(list)`|리스트 내 모든 수의 곱|Python 3.8+|
|`ceil(x)`|올림||
|`floor(x)`|내림||
|`trunc(x)`|소수점 버림 (0 방향)||
|`gcd(a, b)`|최대공약수||
|`lcm(a, b)`|최소공배수|Python 3.9+|
|`factorial(n)`|n! 팩토리얼||
|`comb(n, r)`|조합 nCr (순서 무관)|Python 3.8+|
|`perm(n, r)`|순열 nPr (순서 있음)|Python 3.8+|
|`sqrt(x)`|제곱근|float 반환|
|`isclose(a, b)`|부동소수점 근사 비교||