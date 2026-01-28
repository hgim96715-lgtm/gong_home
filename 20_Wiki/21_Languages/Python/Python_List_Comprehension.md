---
aliases:
  - 파이썬 필터링
  - List Comprehension,
  - 리스트 컴프리헨션
tags:
  - Python
related:
  - "[[Python_Lists_Tuples#List Comprehension (리스트 컴프리헨션)]]"
---
---
## 개념 한 줄 요약

**"리스트를 만드는 공장 (`for`문 + `if`문)을 한 줄로 압축한 문법."**

- **일반 `for`문:** 빈 리스트 생성 -> 반복 -> 검사 -> 추가 (4줄 이상)
- **컴프리헨션:** `[ 넣을것 for 꺼낼것 in 통 if 조건 ]` (1줄 끝)

---
## 핵심: 다중 조건 필터링 (`and`) 

"파일명에 2024가 있고, `data_`로 시작하고, 확장자는 `csv`여야 해." 이렇게 조건이 많을 때, 
리스트를 여러 번 만들지 말고 **`and`** 로 연결하면 됩니다.

```python
files = ["data_202401.csv", "log_2023.txt", "data_202402.json"]

# [Good] 검문소 3개를 한 줄로 통합 (One-Pass)
target = [
    f for f in files
    if '2024' in f                     # 조건 1
    and f.startswith("data_")          # 조건 2
    and f.endswith((".csv", ".json"))  # 조건 3
]
```
---
## 초보자가 자주 하는 실수 (Mistakes)

### ① `endswith` 에는 튜플 `()`을 써야 한다! (중요 )

확장자를 여러 개 검사할 때 리스트 `[]`를 넣으면 에러가 납니다. 꼭 **튜플 `()`** 로 묶어주세요.

- ❌ `f.endswith([".csv", ".json"])` -> **Error!**
- ✅ `f.endswith((".csv", ".json"))` -> **OK!

### ② 줄 바꿈을 두려워하지 마라

조건이 길어지면 한 줄에 억지로 쓰지 말고, 엔터(Enter)를 쳐서 가독성을 높이세요. 
파이썬은 대괄호 `[]` 안에서의 줄 바꿈을 허용합니다.

---
## 언제 쓰는가? (Practical Context)

- **파일 골라내기:** 수천 개의 로그 파일 중 특정 날짜/확장자만 추출할 때.    
- **데이터 정제:** `None` 값을 날리거나, 문자열 공백을 제거(`strip`)하며 가져올 때.
