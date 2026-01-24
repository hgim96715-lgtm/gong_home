---
aliases:
  - List
  - Tuple
  - 리스트
  - 튜플
  - Slicing
  - 슬라이싱
  - 배열
tags:
  - Python
  - DataStructure
  - Syntax
related:
  - "[[Python_Control_Flow]]"
  - "[[]]"
  - "[[Python_Dictionaries]]"
---
## 개념 한 줄 요약

데이터를 순서대로 담는 **기차** 같은 자료구조야.

* **List (`[]`):** 내용물을 뺐다 꼈다 할 수 있는 **화물 기차**. (수정 가능 - Mutable)
* **Tuple (`()`):** 한 번 짐을 실으면 용접해버리는 **밀봉 기차**. (수정 불가 - Immutable)

---
##  왜 필요한가 (Why)

 **Airflow 활용:**
-   DAG 안에서 태스크 순서 정할 때: `[task1, task2] >> task3` (**List**)
- 함수에서 값 여러 개 리턴할 때: `return "success", 200` (**Tuple**)

 **핵심:**
 -  데이터를 묶어서 관리하지 않으면 변수 이름 100개를 지어야 하는 지옥이 펼쳐짐.

---
## Core Points & Syntax

### A. List (리스트) - 만능 상자

```python
# 생성
tasks = ["extract", "transform", "load"]

# 추가 & 삭제
tasks.append("email_alert") # 맨 뒤에 추가
tasks.remove("transform")   # 삭제

# 조회 (인덱싱)
print(tasks[0])  # 'extract' (0부터 시작!)
```

### B. Tuple (튜플) - 안전 금고

```python
# 생성 (소괄호 사용)
db_info = ("localhost", 5432)

# 조회
print(db_info[0]) # 'localhost'

# 🚨 수정 불가 (에러 발생)
# db_info[0] = "127.0.0.1"  <-- TypeError!
```

---
## Deep Dive: Slicing (슬라이싱) 

리스트의 일부분을 잘라내거나 **복사**할 때 쓰는 핵심 기술. 
Airflow `for`문 돌릴 때 필수!

- **문법:** `리스트[시작:끝]` (끝 번호 앞까지만 잘림)
- **`[:]` (전체 복사):** 이게 제일 중요해!

```python
org_list = ["task_a", "task_b", "task_c"]

# 1. 부분 자르기
print(org_list[0:2]) # ['task_a', 'task_b']

# 2. 전체 복사 (Copy) ✨
# 그냥 `new = org` 하면 '참조'만 돼서 원본이 같이 바뀜.
# `new = org[:]` 해야 완전히 새로운 쌍둥이를 만듦.
copy_list = org_list[:] 

# 반복문 돌리면서 리스트 수정할 때 안전장치로 쓰임
for t in org_list[:]: 
    print(f"Processing {t}")
```

---
## 초보자가 자주 착각하는 포인트

1. "튜플은 괄호 `()` 없어도 되나요?"
	- 응! `a = 1, 2` 라고만 써도 파이썬은 자동으로 튜플로 인식해. (Packing)

2. "리스트 변수 그냥 복사하면 안 돼요?"
	- list_a = [1, 2]
	- `list_b = list_a` (이건 복사가 아니라 **별명 붙이기**야)
	- `list_b`를 건드리면 `list_a`도 바뀜! 그래서 **`[:]`** 나 `.copy()`를 써야 해.

3. **"항목이 하나일 때 튜플이 안 돼요!"**
	- 요소가 딱 하나만 있을 때는 **반드시 뒤에 콤마(,)** 를 찍어야 해.
	- `("hello")` ➡ 그냥 **문자열(String)** 로 인식됨. (수학 괄호랑 똑같이 취급)
	- `("hello",)` ➡ 비로소 **튜플(Tuple)**로 변신!
	- **"괄호가 본체가 아니라 콤마가 본체다"** 라고 기억해!