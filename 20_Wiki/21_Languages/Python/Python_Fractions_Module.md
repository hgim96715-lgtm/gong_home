---
aliases:
  - Python
  - Fraction
  - 분수
  - 유리수
  - 부동소수점 오차방지
tags:
  - Python
related:
  - "[[00_Python_HomePage]]"
  - "[[Python_Math_Module]]"
---
## Concept Summary (개념 한 문장 요약)

`fractions` 모듈은 부동소수점 오차 없이 유리수(분수)를 정확하게 표현하고 계산할 수 있게 해주는 파이썬 내장 라이브러리이다.

---
## Why It Is Needed (왜 필요한지)

파이썬에서 기본 `float` 자료형은 이진 부동소수점 방식을 사용하기 때문에 `0.1 + 0.2 == 0.3`이 `False`가 되는 등 미세한 계산 오차가 발생한다. 
데이터의 정확한 비율(ratio)을 유지하거나 정밀한 수학적 연산이 필요할 때, 오차를 원천적으로 차단하기 위해 필요하다.

---
## When It Is Used In Practice (실무에서 언제 쓰는지)

데이터 엔지니어링 파이프라인에서 스케일링이나 정규화 작업을 할 때 손실 없는 비율 데이터를 다루거나, 부동소수점 오차가 누적되면 안 되는 정밀한 통계 및 지표 계산을 수행할 때 사용한다. (단, 화폐 단위와 같은 금융 데이터 계산에는 주로 `decimal` 모듈을 사용한다.)

---
## Core Code Explanation (핵심 코드 설명)

## Creating Fractions (분수 생성)

가장 기본적으로 `Fraction(분자, 분모)` 형태로 객체를 뚝딱 만들어낸다.
정수, 문자열 등 다양한 타입을 인자로 받아 자동으로 기약분수 형태로 변환하여 저장한다.
문자열을 입력값으로 사용하면 초기 부동소수점 할당 시 발생하는 오차를 방지할 수 있다.

>소수를 넣을 때는 반드시 따옴표(`''`)로 감싸서 문자열로 넣어야 한다!

```python
from fractions import Fraction

# 1. 정수 두 개로 생성 (분자, 분모)
f1 = Fraction(3, 4)

# 2. 문자열로 생성
f2 = Fraction('1.5')
f3 = Fraction('-5/2')

print(f1) # 3/4
print(f2) # 3/2
print(f3) # -5/2
```

---
##  Arithmetic Operations (사칙연산) >(자동 통분 & 기약분수 약분!)

사칙연산을 수행하면 내부적으로 최소공배수를 찾아 계산한 뒤, 항상 최약분수(기약분수) 형태로 결과를 반환한다. 

```python
from fractions import Fraction

f1 = Fraction(1, 6)
f2 = Fraction(1, 3)

# 분수 간의 연산 및 정수와의 연산
print(f1 + f2)
print(f1 * 2)
```

>💡 **`math.gcd`와의 연결고리** `Fraction`이 자동으로 약분해주는 내부 원리가 바로 `math.gcd`다. 직접 구현하면 이렇게 된다.
>`Fraction`은 이 과정을 자동으로 처리해주는 포장지다. → 관련 문서: [[Python_Math_Module#③ 최대공약수와 최소공배수 (`gcd`, `lcm`)|최대공약수와 최소공배수]] 참조 

```python
import math

# Fraction(6, 9)의 내부 동작 원리
numerator, denominator = 6, 9
g = math.gcd(numerator, denominator)  # gcd(6, 9) = 3
print(numerator // g, denominator // g)  # → 2 3  (즉, 2/3)
```

---
## Limit Denominator (분모 제한)

기존의 무리수나 복잡한 `float` 값을 `Fraction`에 넣으면 파이썬 메모리에 저장된 이진수 근삿값을 그대로 분수로 치환하여 매우 큰 분자와 분모가 나온다
이때 `.limit_denominator(max_denominator)` 메서드를 사용하면 지정한 분모의 최댓값(위 예시에서는 100) 내에서 원래 값에 가장 가까운 유리수 근삿값을 찾아준다.

```python
import math
from fractions import Fraction

# 파이(pi) 값을 분수로 변환
f = Fraction(math.pi)

print(f)
print(f.limit_denominator(100))
```

---
## Common Mistakes (자주 하는 실수)

소수점이 있는 숫자를 `float` 형태로 그냥 넣을 때 발생하는 오차가 가장 잦은 실수이다.

`Fraction(0.1)`을 실행하면 직관적으로 `1/10`이 나올 것 같지만, 실제로는 `3602879701896397/36028797018963970`라는 전혀 예상치 못한 값이 출력된다. 이는 `0.1`이라는 `float` 값이 컴퓨터 내부에서 이미 미세한 오차를 가진 이진수로 저장되었기 때문이다.

정확한 `1/10`을 얻으려면 **반드시 `Fraction('0.1')`처럼 문자열로 넘기거나, `Fraction(1, 10)`처럼 정수로 직접 분자와 분모를 입력**해야 한다.
