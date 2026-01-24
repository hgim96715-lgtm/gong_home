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
---
## 개념 한 줄 요약

**흐름 제어(Control Flow)** 는 코드의 실행 순서를 내 마음대로 지휘하는 거야.
위에서 아래로만 흐르는 물길을 **"갈림길(If)"** 을 만들어 나누거나, **"물레방아(Loop)"** 처럼 뱅글뱅글 돌게 만드는 기술이지.

---
## 왜 필요한가 (Why)

**문제점:**
- 제어문이 없으면 코드를 1번부터 100번 줄까지 무조건 순서대로만 실행해야 해. 
- "주말엔 실행하지 마", "파일 100개 다 처리해" 같은 걸 못 해.

**해결책:**
- **If (조건):** 상황에 따라 다른 코드를 실행해. (판단)
- **For (반복):** 정해진 횟수만큼 반복해. (순회)
- **While (반복):** 조건이 맞을 때까지 계속 반복해. (대기)

---
##  Practical Context (Data Engineering)

데이터 엔지니어링에서 어떻게 쓰이는지 볼까?
1.  **If:** "오늘이 휴일이면(If holiday) 크롤링 스킵해."
2.  **For:** "S3 버킷에 있는 파일 리스트를 하나씩 꺼내서(For file in files) DB에 넣어."
3.  **While:** "API 응답이 올 때까지(While status != 200) 5초마다 재시도해."

---
##  Code Core Points

### A. If 문 (조건문)

파이썬은 괄호 `{ }` 대신 **들여쓰기(Indentation)** 가 생명이야!

```python
status = "failed"

if status == "success":
    print("성공! 다음 태스크 진행시켜.")
elif status == "failed":
    print("실패! 관리자에게 알림 보내.")
else:
    print("대기 중...")
```

### B. For 문 (순회 반복)

리스트, 딕셔너리 같은 **덩어리(Iterable)** 에서 하나씩 꺼낼 때 써. 
Airflow 동적 태스크 만들 때 제일 많이 쓰는 놈이야.

```python
files = ["data_v1.csv", "data_v2.csv", "data_v3.csv"]

# 파일 리스트를 하나씩 돌면서 처리
for file in files:
    print(f"Processing {file}...")
    # 여기에 전처리 로직 들어감
```

### C. While 문 (조건 반복)

끝나는 시점이 명확하지 않을 때, 특정 조건이 될 때까지 기다릴 때 써.

```python
import time
retry_count = 0

# 5번까지만 재시도 하겠다 (무한 루프 방지)
while retry_count < 5:
    print(f"{retry_count + 1}번째 연결 시도 중...")
    retry_count += 1
    time.sleep(1) # 1초 대기

print("연결 종료")
```

---
## Detailed Analysis (Break & Continue)

반복문을 더 세밀하게 제어하는 브레이크 페달들이야.

- **`break`**: "야, 그만해! 탈출!" (반복문 즉시 종료)
    - 예: 파일 찾다가 찾았으면 뒤에 건 볼 필요 없이 종료.

- **`continue`**: "이번 건 스킵하고 다음 거 해!" (이번 턴만 넘기기)
    - 예: 파일 처리하다가 에러 나면, 멈추지 말고 다음 파일로 넘어가기.

```python
for i in range(10):
    if i == 3:
        continue # 3은 건너뛰고 4로 감
    if i == 5:
        break    # 5가 되면 아예 반복문 끝!
    print(i)
# 결과: 0, 1, 2, 4 (5부터는 안 찍힘)
```

---
## 초보자가 자주 착각하는 포인트

1. "들여쓰기(Indentation) 에러"
	- 파이썬은 공백(Space) 4칸에 목숨 걸어야 해. 
	- `if`문 다음 줄이 들여쓰기가 안 되어 있으면 에러 나거나 엉뚱하게 동작해.

2. "While문 무한 루프"
	- `while True:`라고 써놓고 안에 `break`를 안 만들어두면? 컴퓨터 멈출 때까지 영원히 돌아. (서버 비용 폭탄의 주범 💣)
	- `while` 쓸 땐 항상 **"탈출 조건"** 이 있는지 두 번 확인해!

3. "For문에서 리스트 수정하기"
	- `for item in my_list:` 돌리면서 `my_list.remove(item)` 하면 인덱스가 꼬여서 대참사 일어남.
	- 리스트를 수정해야 하면 복사본(`my_list[:]`)을 돌리거나, 새 리스트를 만드는 게 안전해.

>**Obsidian 연결:** 이 내용은 Airflow의 **[[Dynamic_Tasks]]** (For문 활용)와 **[[Airflow_Sensors]]** (While문 개념)를 이해하는 기초가 돼.