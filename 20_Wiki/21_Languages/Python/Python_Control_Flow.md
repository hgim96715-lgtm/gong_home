---
aliases:
  - 제어문
  - 조건문
  - 반복문
  - If
  - For
  - While
  - 흐름 제어
tags:
  - Python
  - Logic
related:
  - "[[Dynamic_Tasks]]"
  - "[[Airflow_Sensors]]"
  - "[[Python_Looping_Helpers]]"
  - "[[Python_Variables_Types]]"
  - "[[Spark_Iterating_Data]]"
  - "[[00_Python_HomePage]]"
  - "[[Python_List_Comprehension]]"
---
## 개념 한 줄 요약

**"코드의 실행 순서를 내 마음대로 지휘하는 지휘봉."**
위에서 아래로만 흐르는 물길을 **"갈림길(If)"** 을 만들어 나누거나, **"물레방아(Loop)"** 처럼 뱅글뱅글 돌게 만드는 기술입니다.

---

##  왜 필요한가? (Why & Context)

###  문제점
* 제어문이 없으면 코드를 1번 줄부터 100번 줄까지 무조건 순서대로만 실행해야 합니다.
* **불가능한 미션:** "주말엔 크롤링 하지 마", "파일 100개 다 처리해", "에러 나면 재시도해".

###  해결책 (Data Engineering Context)
| 도구 | 역할 | 데이터 엔지니어링 활용 예시 |
| :--- | :--- | :--- |
| **If (조건)** | 판단 | "오늘이 휴일이면(**If**) 크롤링 스킵해." |
| **For (반복)** | 순회 | "S3 버킷의 파일 리스트를 하나씩 꺼내서(**For**) DB에 적재해." |
| **While (반복)** | 대기 | "API 응답이 200(성공)이 될 때까지(**While**) 5초마다 재시도해." |

---
##  If 문 (조건문)

상황에 따라 다른 코드를 실행합니다. 
파이썬은 `{ }` 괄호 대신 **들여쓰기(Indentation, 공백 4칸)** 가 생명입니다.

### 문법 구조 (Syntax)

```python
if [조건 A]:
    # 조건 A가 참(True)일 때 실행
elif [조건 B]:
    # A는 아니고, B가 참일 때 실행
else:
    # 위의 모든 조건이 거짓(False)일 때 실행
```

### 실전 코드

```python
status = "failed"

if status == "success":
    print("성공! 다음 태스크 진행시켜.")
elif status == "failed":
    print("실패! 관리자에게 알림 보내.")
else:
    print("대기 중...")
```

### Truthy & Falsy (참과 거짓)

파이썬에서는 숫자 그 자체를 조건문으로 쓸 수 있습니다.

- **0** : `False`로 취급
- **0이 아닌 모든 수** (1, -1, 100...) : `True`로 취급

**활용 예시:**

```python
n = 3
if n % 2:  # 나머지가 1이면 True (홀수)
    print("홀수입니다")
else:      # 나머지가 0이면 False (짝수)
    print("짝수입니다")
```

**[[Python_List_Comprehension]]** : `[i*i for i in range(2, n+1, 2)]` 처럼 짝수 제곱을 한 줄로 만드는 법 참고.

---
## For 문 (순회 반복) 

리스트, 딕셔너리 같은 **덩어리(Iterable)** 에서 데이터를 하나씩 꺼낼 때 씁니다. 
Airflow 동적 태스크 생성의 핵심입니다.

### 문법 구조 (Syntax)

```python
for [변수명] in [반복_가능한_대상]:
    # 꺼낸 데이터를 [변수명]으로 요리함
```

### 실전 코드

```python
files = ["data_v1.csv", "data_v2.csv", "data_v3.csv"]

# 파일 리스트를 하나씩 돌면서 처리
for file in files:
    print(f"Processing {file}...")
    # 여기에 전처리 로직 들어감
```

### [심화] Python은 `foreach`가 없다?

다른 언어(Java, C) 개발자들이 가장 많이 묻는 질문입니다.

- **Q.** "파이썬에는 `foreach` 문이 없나요?"
- **A.** **네, 없습니다. `for` 문이 곧 `foreach` 이기 때문입니다.**
    - 전통적인 `for(i=0; i<N; i++)` 방식이 아니라, 주머니에서 알맹이를 직접 꺼내는 방식입니다.

### `in` 뒤에는 누가 올 수 있나? (Iterable)

`for 변수 in [여기]:` 의 `[여기]` 자리에는 반드시 **Iterable(반복 가능한 객체)** 만 올 수 있습니다.

- **(O) 가능한 애들:** 리스트(`[]`), 튜플(`()`), 문자열(`"abc"`), 딕셔너리(`{}`), `range()`
- **(X) 불가능한 애들:** 정수(`10`), 실수(`3.14`) -> 꺼낼 게 없어서 에러(`TypeError`) 발생!

```python
# [해결책] 숫자로 반복하고 싶으면 range()를 써라!
# 0부터 9까지(10번) 반복
for i in range(10):
    print(i)
```


----
## While 문 (조건 반복) 

끝나는 횟수가 정해져 있지 않고, **특정 조건이 만족될 때까지** 계속 기다릴 때 씁니다.

### 문법 구조 (Syntax)

```python
while [조건]:
    # 조건이 True인 동안 계속 무한 반복
```

### 실전 코드 (재시도 로직)

```python
import time
retry_count = 0

# 5번까지만 재시도 하겠다 (무한 루프 방지 조건)
while retry_count < 5:
    print(f"{retry_count + 1}번째 연결 시도 중...")
    retry_count += 1
    time.sleep(1) # 1초 대기

print("연결 종료")
```

---
## 제어 장치 (Break & Continue) 

반복문을 더 세밀하게 조종하는 브레이크/액셀 페달입니다.

- **`break`:** "야, 그만해! 탈출!" (반복문 **즉시 종료**)
    - _예: 파일 찾다가 찾았으면 뒤에 건 볼 필요 없이 종료._

- **`continue`:** "이번 건 스킵하고 다음 거 해!" (이번 턴만 넘기고 **계속 진행**)
    - _예: 파일 처리하다가 에러 나면, 멈추지 말고 다음 파일로 넘어가기._

```python
for i in range(10):
    if i == 3:
        continue # 3은 건너뛰고 4로 감 (Skip)
    if i == 5:
        break    # 5가 되면 아예 반복문 끝! (Stop)
    print(i)
# 결과: 0, 1, 2, 4 (5부터는 안 찍힘)
```

---
## 초보자가 자주 착각하는 포인트 

### ① 들여쓰기(Indentation) 에러

파이썬은 공백(Space) 4칸에 목숨 걸어야 합니다.
`if`나 `for` 다음 줄이 들여쓰기가 안 되어 있으면 에러(`IndentationError`)가 나거나 로직이 꼬입니다.

### ② While문 무한 루프 (Infinite Loop)

`while True:`라고 써놓고 안에 `break`를 안 만들어두면? 컴퓨터가 멈출 때까지 영원히 돕니다.
(클라우드 서버 비용 폭탄의 주범 💣) -> 항상 **탈출 조건**을 확인하세요.

### ③ For문에서 리스트 수정하기

`for item in my_list:` 로 돌리면서 `my_list.remove(item)` 처럼 리스트 개수를 줄이면 인덱스가 꼬여서 대참사가 일어납니다. 
원본을 건드리지 말고 **새 리스트**를 만드세요.

>**Obsidian 연결**
> 이 내용은 Airflow의 **[[Dynamic_Tasks]]** (For문 활용)와 **[[Airflow_Sensors]]** (While문 개념)를 이해하는 기초가 됩니다.