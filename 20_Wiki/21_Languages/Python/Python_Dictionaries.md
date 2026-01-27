---
aliases:
  - Dictionary
  - Dict
  - 딕셔너리
  - 해시맵
  - HashMap
  - Key-Value
  - JSON객체
tags:
  - Python
  - DataStructure
related:
  - "[[Python_Lists_Tuples]]"
  - "[[Python_JSON]]"
  - "[[Python_Classes_Objects]]"
  - "[[Python_Sorting_Logic]]"
  - "[[Spark_Data_Aggregation]]"
---
## 개념 한 줄 요약 

데이터에 **이름표(Key)** 를 붙여서 저장하는 **"물품 보관함"** 같은 자료구조입니다.

| 구분 | **리스트 (List) `[]`** | **딕셔너리 (Dict) `{}`** |
| :--- | :--- | :--- |
| **찾는 법** | "첫 번째 서랍 열어봐." (순서) | "**'owner'** 라고 적힌 서랍 열어봐." (이름) |
| **비유** | 아파트 우편함 (101호, 102호...) | 주소록 (홍길동: 010-XXXX...) |

---
##  왜 필요한가? (Why) 

### ① Airflow는 딕셔너리로 대화한다

데이터 엔지니어링의 표준 도구인 Airflow의 설정은 99% 딕셔너리입니다.
* DAG 설정(`default_args`), 태스크 간 데이터 전달(XCom), DB 접속 정보(`Extra`)가 전부 딕셔너리 형태입니다.

### ② 엄청나게 빠른 검색 속도

* **리스트:** 데이터를 찾으려면 맨 앞부터 하나씩 뒤져야 합니다. (데이터 많으면 느려짐)
* **딕셔너리:** **이름(Key)만 알면 0.001초 만에** 바로 찾아냅니다. (Hash Table 원리)

---
## 3. 핵심 문법 (Core Syntax) 

### A. 만들기 (Create) & 조회하기 (Read)

`{ }` (중괄호)를 쓰고, **콜론(`:`)** 으로 `이름표(Key)`와 `내용물(Value)`을 연결합니다.

```python
# 1. 만들기
user = { "name": "airflow", "age": 10 }

# 2. 조회하기 (대괄호 사용)
print(user['name'])  # 결과: 'airflow'
print(user['age'])   # 결과: 10
```

> **⚠️ 주의:** 이름표(Key)의 철자가 띄어쓰기 하나라도 틀리면 에러(`KeyError`)가 납니다!


### B. 있는지 확인하기 (`in` 연산자)

값을 꺼내기 전에 **"이 이름표가 진짜 있어?"** 라고 물어보는 안전장치입니다.

```python
# 'email'이라는 키가 있는지 확인 (내용물 검사 아님!)
if 'email' in user:
    print(user['email']) # 있으면 꺼내고
else:
    print("이메일 정보가 없어요!") # 없으면 넘어감
```

- **주의:** `in`은 **Key만 검사합니다.**  내용물(Value)인 `'airflow'`가 있는지 물어보면 `False`라고 합니다.

---
## 데이터 엔지니어의 필수 무기: `.get()` 

실무에서는 대괄호 `[]`보다 **`.get()`** 을 훨씬 많이 씁니다. 
프로그램이 뻗는 것을 막아주기 때문입니다.

```python
config = {"env": "prod"}

# ❌ 위험한 방법 (KeyError 발생 가능)
# print(config["retries"])  <-- 펑!  (없는 키라서 에러)

# ⭕️ 안전한 방법 (.get)
# "retries"가 없으면 에러 내지 말고, 기본값(3)을 줘라.
retry_count = config.get("retries", 3)
print(retry_count) # 결과: 3
```

---
## 전체 스캔하기 (`items`, `keys`, `values`) 

딕셔너리 안의 내용을 반복문(`for`)으로 돌리고 싶을 때 씁니다.

### ① `items()`: 이름표랑 내용물 둘 다! (가장 중요 ⭐)

**용도:** 정렬(`sorted`)하거나, 키와 값을 동시에 `for`문 돌릴 때 필수.
**타입:** `<class 'dict_keys'>` 라는 **뷰(View) 객체**입니다. (가짜 리스트)

* **특징:**
    * **메모리 절약:** 데이터를 복사하지 않고 원본을 실시간으로 참조합니다.
    * **호환성:** `for` 문이나 `sorted()` 에는 바로 넣어도 알아서 잘 작동합니다. -> 리스트 인 척 한다. 
    * **제약:** 인덱싱(`items[0]`)은 안 됩니다. 순서대로 꺼내고 싶다면 `list()`로 변환해야 합니다.

```python
dag_config = {'owner': 'team_A', 'schedule': '@daily'}

# 1. 타입 확인: 리스트가 아니라 'dict_items'
print(type(dag_config.items())) 
# 결과: <class 'dict_items'>

# 2. 반복문 사용 (바로 사용 가능 O)
for k, v in dag_config.items():
    print(f"{k} : {v}")

# 3. 인덱싱 시도 (불가능 X)
# print(dag_config.items()[0]) 
# -> TypeError: 'dict_items' object is not subscriptable

# 4. 인덱싱 하려면? (리스트로 변환 O)
real_list = list(dag_config.items())
print(real_list[0]) 
# 결과: ('owner', 'team_A')
```

### ② keys() : 이름표(Key)만 가져오기

**타입:** `<class 'dict_keys'>`
**용도:** "어떤 설정값들이 있는지 목록만 볼 때" 사용.

```python
# 1. 그냥 찍으면 'dict_keys'라는 껍질에 싸여서 나옴
print(dag_config.keys())
# 결과: dict_keys(['owner', 'schedule'])

# 2. 0번째 키를 딱 집으려면? -> 리스트로 변환(list()) 필수!
key_list = list(dag_config.keys())
print(key_list[0])
# 결과: 'owner'
```

### ③ `values()`: 내용물(Value)만 가져오기

- **타입:** `<class 'dict_values'>`
- **용도:** "평균 점수 계산하게 점수만 다 줘봐" 할 때 사용.

```python
# 1. 내용물만 쏙 뽑아옴
print(dag_config.values()) 
# 결과: dict_values(['team_A', '@daily'])

# 2. 반복문에는 변환 없이 바로 사용 가능
for val in dag_config.values():
    print(val)
```

> [공통 주의사항]
> 이 3형제(`items`, `keys`, `values`)는 모두 **인덱싱(`[0]`)이 불가능**합니다. 
> 순서대로 딱 집어서 꺼내야 한다면 반드시 **`list()`** 로 감싸주세요!






