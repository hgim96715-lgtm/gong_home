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
---
## 개념 한 줄 요약

데이터에 **이름표(Key)** 를 붙여서 저장하는 **"물품 보관함"** 같은 자료구조야.

* **리스트(`[]`):** "첫 번째 서랍 열어봐." (순서가 중요)
* **딕셔너리(`{}`):** "**'owner'** 라고 적힌 서랍 열어봐." (이름이 중요)

---
## 왜 필요한가 (Why) 중요!

**Airflow는 딕셔너리로 대화해:**
- DAG 설정(`default_args`), XCom 데이터, 커넥션 정보(`Extra` 필드) 전부 다 딕셔너리야.

**빠른 검색:**
- 리스트는 데이터를 찾으려면 처음부터 뒤져야 하는데, 딕셔너리는 **이름만 알면 0.001초 만에** 찾아내. (Hash Table 원리)

---
## Core Points & Syntax

### A. 기본 문법 (생성, 조회, 확인)

**1. 만들기 (Create)**

`{ }` (중괄호)를 쓰고, **콜론(`:`)** 으로 `이름표(Key)`와 `내용물(Value)`을 연결해.

```python
user = { "name": "airflow", "age": 10 }
```

**2. 조회하기 (Read)** 

대괄호 `['키 이름']`을 사용해서 값을 꺼내.

- **주의:** 이름표(Key)의 철자가 띄어쓰기 하나라도 틀리면 에러(KeyError)가 나니 조심!

```python
print(user['name'])  # 결과: 'airflow'
print(user['age'])   # 결과: 10
```

**3. 있는지 확인하기 

**(`in` 연산자)**  값을 꺼내기 전에 **"이 이름표가 진짜 있어?"** 라고 물어보는 거야.

- 결과는 `True`(있음) 또는 `False`(없음)로 나와.
- **꿀팁:** `if`문이랑 찰떡궁합이야!

```python
# 'email'이라는 키가 있는지 확인
if 'email' in user:
    print(user['email']) # 있으면 꺼내고
else:
    print("이메일 정보가 없어요!") # 없으면 넘어감
```

**주의:** 
`in` 연산자는 **이름표(Key) 만 검사해! 
내용물(Value)인 'airflow'가 있는지(`'airflow' in user`) 물어보면 `False`라고 대답해.

>**`in` 연산자**는 나중에 이런 로직 짤 때 무조건 쓰게 됩니다.
>"만약(`if`) 설정에 'retries'라는 키가 들어있으면(`in`) 그걸 쓰고, 없으면 기본값으로 1번만 재시도해!"*

### B. 안전하게 꺼내기 (`.get()`) - 실무 필수!

대괄호 `[]`로 없는 키를 찾으면 에러(`KeyError`)가 나면서 프로그램이 뻗어버려. 
이때 **`.get()`** 을 쓰면 에러 대신 `None`이나 기본값을 줘.

```python
# ❌ 위험한 방법 (KeyError 발생 가능)
# print(dag_config["start_date"])  <-- 펑! 💥 (없는 키)

# ⭕️ 안전한 방법 (.get)
# start_date가 없으면 '2024-01-01'을 대신 내놔라.
start_date = dag_config.get("start_date", "2024-01-01") 
print(start_date)
```

### 키/값 따로 보기

```python
print(dag_config.keys())   # 이름표만 쫘르륵: ['dag_id', 'schedule', ...]
print(dag_config.values()) # 내용물만 쫘르륵: ['my_first_dag', '@daily', ...]
print(dag_config.items())  # (키, 값) 쌍으로: [('dag_id', 'my_first_dag'), ...]
```

---
## Practical Context (Airflow Code)

Airflow DAG 짤 때 무조건 보는 코드지?

```python
# 1. 딕셔너리로 공통 설정 만들기
default_args = {
    'owner': 'data_team',
    'email_on_failure': True,
    'retries': 1
}

# 2. 함수 인자로 딕셔너리 풀어서 전달 (**kwargs)
with DAG(..., default_args=default_args) as dag:
    ...
```

---
## 초보자가 자주 착각하는 포인트

1. "순서가 있나요?"
	- 파이썬 3.7부터는 순서를 기억해주긴 하지만, 기본적으로 딕셔너리는 **"순서 따위 중요하지 않아! 이름표가 중요해!"** 라는 마인드로 써야 해.
	- 순서가 필요하면 `List`를 써야지.

2. "Key에 아무거나 써도 되나요?"
	- 아니! **변하지 않는 것(Immutable)** 만 쓸 수 있어.
	- 문자열(`"name"`), 숫자(`1`), 튜플(`(1,2)`)은 OK.
	- 리스트(`['a']`)나 딕셔너리는 Key로 못 써. (Key 자체가 변하면 서랍을 못 찾으니까)

3. **"딕셔너리랑 JSON이랑 똑같은 거 아닌가요?"**
	- 비슷하지만 달라!
	- **딕셔너리:** 파이썬 메모리에 있는 **객체**.
	- **JSON:** 파일이나 네트워크로 주고받기 위해 글자로 적어놓은 **문자열(String)** 포맷.
	- 이 둘을 변환해 주는 게 바로 `json.loads`(글자->딕셔너리)와 `json.dumps`(딕셔너리->글자)야.

4. **"점(.)으로 값 꺼내면 안 되나요?" (`my_dict.key`)**
	- **절대 안 돼!** 그건 자바스크립트나 파이썬 **객체(Object)** 에서 쓰는 방식이야.
	- 파이썬 **딕셔너리**는 무조건 대괄호 **`['key']`** 나 **`.get('key')`** 를 써야 해.

> `my_dict['key']`: 딕셔너리 (서랍장에 이름표 붙인 것)
> `my_obj.key`: 객체 (설계도로 만든 로봇의 팔다리)


