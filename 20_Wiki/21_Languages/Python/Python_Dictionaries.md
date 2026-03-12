---
aliases:
  - Dictionary
  - Dict
  - 딕셔너리
  - 해시맵
  - HashMap
  - Key-Value
  - JSON객체
  - fromkeys
  - get()
  - 체이닝
  - 중첩 딕셔너리
tags:
  - Python
related:
  - "[[Python_Lists_Tuples]]"
  - "[[Python_JSON]]"
  - "[[Python_Classes_Objects]]"
  - "[[Python_Sorting_Logic]]"
  - "[[Python_Membership_In]]"
---
# Python_Dictionaries

## 개념 한 줄 요약

> **"데이터에 이름표(Key) 를 붙여서 저장하는 물품 보관함."**

|구분|List `[]`|Dict `{}`|
|---|---|---|
|**찾는 법**|"첫 번째 서랍 열어봐" (순서)|"'owner' 라고 적힌 서랍 열어봐" (이름)|
|**비유**|아파트 우편함 (101호, 102호...)|주소록 (홍길동: 010-XXXX...)|
|**속도**|데이터 많으면 느려짐|데이터 100만 개여도 즉시 찾음 (Hash Table)|

---

---

# ① 만들기 / 조회 / 추가 / 수정 / 삭제

```python
# 만들기
user = {"name": "airflow", "age": 10}

# 조회
user["name"]   # "airflow"
user["age"]    # 10
# user["email"]  <- 없는 키 -> KeyError 발생!

# 추가: 없는 키에 값을 넣으면 새로 생김
user["email"] = "admin@test.com"

# 수정: 있는 키에 값을 넣으면 덮어쓰기
user["age"] = 11

# 삭제
del user["age"]
```

---

---

# ② in 연산자 — 키 존재 확인

```python
# in 은 Key 만 검사한다 (Value 검사 아님!)
if "email" in user:
    print(user["email"])
else:
    print("이메일 정보가 없어요!")

# Value 인 "airflow" 가 있는지 물어보면?
"airflow" in user   # False  <- Key 에 없으니까
"name" in user      # True   <- Key 에 있으니까
```

---

---

# ③ .get() — 안전한 조회 ⭐️

> 실무에서는 대괄호 `[]` 보다 `.get()` 을 훨씬 많이 쓴다.
>  없는 키를 조회해도 에러 없이 기본값을 반환해주기 때문.

```python
config = {"env": "prod"}

# 대괄호: 없는 키 -> KeyError 발생
# config["retries"]  <- 펑!

# .get(): 없는 키 -> 기본값 반환
retry_count = config.get("retries", 3)
print(retry_count)  # 3  <- 없으니까 기본값 반환

# 기본값 생략하면 None 반환
config.get("retries")   # None (에러는 안 남)
```

## .get() 체이닝 — 중첩 딕셔너리 안전하게 파고들기 ⭐️

> 공공데이터 API 응답처럼 딕셔너리가 여러 겹으로 중첩된 구조에서 중간에 키가 없어도 에러 없이 끝까지 통과하는 패턴.

```python
# 공공데이터 API 응답 구조
# response -> body -> items -> item (리스트)

# 대괄호 방식: 중간에 하나라도 없으면 KeyError 대참사
items = data["response"]["body"]["items"]["item"]

# .get() 체이닝 방식 (안전 / 권장)
items = (
    data
    .get("response", {})   # "response" 없으면 {} 인 척
    .get("body", {})       # "body" 없으면 {} 인 척
    .get("items", {})      # "items" 없으면 {} 인 척
    .get("item", [])       # "item" 없으면 [] 반환
)

    # item 이 결과가 1개일 때 -> API 가 딕셔너리 하나로 반환 (리스트 아님)
    # item 이 결과가 여러 개일 때 -> API 가 리스트로 반환
    # isinstance 로 타입 확인 후 항상 리스트로 통일
    # -> [[Python_Lists_Tuples#isinstance — 타입 확인 ⭐️|instance란?]] 참고
    return items if isinstance(items, list) else [items]
```

```
동작 원리 (양파 껍질 까기):

data
  └── .get("response", {})   없으면 {} 반환
        └── .get("body", {})     없으면 {} 반환
              └── .get("items", {})  없으면 {} 반환
                    └── .get("item", [])   없으면 [] 반환

중간에 "body" 가 없어도?
  -> "아, 없구나. 그럼 {} 인 척해줄게!"
  -> 다음 .get() 은 {} 에서 찾으니까 역시 없음
  -> 최종적으로 안전하게 빈 리스트 [] 반환 ✅
  -> KeyError 없이 스무스하게 통과
```

```python
# 실전: 공공데이터 API 응답 파싱
def parse_train_data(data: dict) -> list:
    items = (
        data
        .get("response", {})
        .get("body", {})
        .get("items", {})
        .get("item", [])
    )

    # item 이 비어있으면 빈 리스트 반환 (에러 없이 안전)
    if not items:
        print("데이터 없음")
        return []

    return items
```

---

---

# ④ fromkeys() — 한 방에 초기화

```python
keys = ["name", "age", "email"]

# 기본값 없이 -> None
user = dict.fromkeys(keys)
# {"name": None, "age": None, "email": None}

# 기본값 지정
user = dict.fromkeys(keys, "미입력")
# {"name": "미입력", "age": "미입력", "email": "미입력"}
```

## 리스트를 기본값으로 쓰면 대참사

```python
# 모든 키가 같은 리스트 객체를 공유
d = dict.fromkeys(["a", "b"], [])
d["a"].append(1)
print(d)  # {"a": [1], "b": [1]}  <- b 도 같이 바뀜!

# 기본값엔 0, "", None 같은 단순 값을 써야 안전
d = dict.fromkeys(["a", "b"], 0)
```

## 응용 — 순서 유지하며 중복 제거

```text
dict 의 두 가지 특성을 동시에 활용:
  1. 키는 중복 불가     → 중복 자동 제거
  2. Python 3.7+ 부터 삽입 순서 유지 → 원래 순서 보존
```

```python
# 리스트 중복 제거 (순서 유지)
items = [3, 1, 2, 1, 3, 4]
result = list(dict.fromkeys(items))
# [3, 1, 2, 4]   ← 첫 등장 순서 유지

# 문자열 중복 제거 (순서 유지)
my_string = "banana"
result = ''.join(dict.fromkeys(my_string))
# "ban"   ← b, a, n 순서 유지

# set 으로 중복 제거하면 순서 보장 안 됨
set("banana")  # {'a', 'b', 'n'}  순서 제각각 ❌
```

```text
fromkeys vs set 비교:
  set(items)              → 중복 제거 O / 순서 보장 X
  dict.fromkeys(items)    → 중복 제거 O / 순서 보장 O ✅

  실무 활용 예시:
    컬럼 목록에서 중복 제거할 때
    API 응답에서 중복 ID 제거할 때
    로그에서 중복 이벤트 걸러낼 때
```

>[[Python_Sets#⑨ 중복 제거 패턴]] 참고

---

---

# ⑤ items / keys / values — 전체 스캔

## items() — 키와 값을 튜플로 ⭐️

```python
dag_config = {"owner": "team_A", "schedule": "@daily"}

# 타입 확인
type(dag_config.items())  # <class 'dict_items'>

# 출력하면 튜플로 묶여서 나옴
dag_config.items()
# dict_items([("owner", "team_A"), ("schedule", "@daily")])
# 원래 : (콜론) 이었던 게 () (튜플) 로 변신

# 반복문: 키와 값 동시에
for k, v in dag_config.items():
    print(f"{k} : {v}")

# 인덱싱: 불가 -> list() 로 변환 필요
# dag_config.items()[0]  <- TypeError!
list(dag_config.items())[0]  # ("owner", "team_A")
```

```
단골 착각 주의:
for k, v in dag_config.items() 에서
k 를 인덱스(0, 1, 2...) 로 착각하기 쉬움

딕셔너리에는 순서 번호 개념이 없다.
k = Key 이름 / v = Value 값  <- 이것만 기억
```

## keys() — 키만 가져오기

```python
dag_config.keys()
# dict_keys(["owner", "schedule"])

list(dag_config.keys())[0]  # "owner"  <- 인덱싱은 list() 변환 후
```

## values() — 값만 가져오기

```python
dag_config.values()
# dict_values(["team_A", "@daily"])

for val in dag_config.values():
    print(val)
```

> items / keys / values 세 개 모두 인덱싱 불가. 순서대로 꺼내야 하면 반드시 `list()` 로 감싸야 한다.

---

---

# 주의사항

|착각|실제|
|---|---|
|리스트를 Key 로 쓸 수 있다|불가. Key 는 변하면 안 됨 (mutable 불가). 튜플은 가능|
|`dict_items` 가 리스트다|뷰(View) 객체. 인덱싱하려면 `list()` 변환 필요|
|`in` 이 Value 도 검사한다|Key 만 검사. Value 검사는 `in dict.values()`|
|딕셔너리에 순서가 없다|Python 3.7+ 부터 삽입 순서 유지. 하지만 Key 로 찾는 게 목적|