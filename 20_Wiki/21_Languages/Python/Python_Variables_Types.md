---
aliases:
  - 변수
  - 자료형
  - String
  - Integer
  - Boolean
  - TypeCasting
  - f-string
tags:
  - Python
related:
  - "[[Python_Control_Flow]]"
  - "[[Python_String_Methods]]"
---
## 개념 한 줄 요약

**"데이터를 담는 '그릇(Variable)'과 그릇에 담긴 내용물의 '종류(Type)'."**

* **변수(Variable):** 데이터를 저장하는 이름표. (`name = "Spark"`)
* **자료형(Type):** 데이터의 생김새. (글자, 정수, 소수, 참/거짓)
* **Dynamic Typing:** 파이썬은 그릇에 라벨을 안 붙여도 알아서 눈치껏 타입을 정해줍니다. (C/Java랑 다른 점!)

---
##  필수 자료형 4대장 

데이터 엔지니어가 매일 마주치는 4가지 타입입니다.

### ① 문자열 (String, `str`)

따옴표(`"`, `'`)로 감싸면 무조건 문자열입니다.

```python
# 작은 따옴표, 큰 따옴표 상관 없음
tool = "Airflow"
version = '2.5.1'  # 숫자로 보이지만 따옴표가 있으므로 문자열!

# ⭐️ [꿀팁] f-string (변수 섞어 쓰기)
# 문자열 앞에 'f'를 붙이고 {변수}를 넣으면 마법같이 합쳐짐. (강추!)
msg = f"현재 {tool} 버전은 {version} 입니다."
print(msg) 
# 출력: 현재 Airflow 버전은 2.5.1 입니다.
```

### ② 정수 (Integer, `int`)

소수점이 없는 숫자입니다. 개수를 세거나 인덱스를 다룰 때 씁니다.

```python
count = 100
workers = 4

# 사칙연산 가능
total = count * workers  # 400
```

### ③ 실수 (Float, `float`)

소수점이 있는 숫자입니다.

```python
accuracy = 98.5
pi = 3.14159

# ⚠️ 주의: 부동소수점 오차
# 컴퓨터는 0.1 + 0.2를 0.30000000000000004 로 계산할 때가 있음.
# 돈 계산할 때는 decimal 모듈을 써야 함.
```

### ④ 불린 (Boolean, `bool`)

**참(`True`)** 또는 **거짓(`False`)** 딱 두 가지 값만 가집니다. 
대문자로 시작해야 한다는 점을 절대 잊지 마세요! (`true` ❌ -> `True` ⭕️)

```python
is_active = True
has_error = False

# 주로 if 문이랑 짝꿍
if is_active:
    print("서버 가동 중!")
```

---
## 형 변환 (Type Casting) 

**"숫자처럼 생긴 문자"** 때문에 에러가 가장 많이 납니다. 
강제로 타입을 바꿔줘야 합니다.

```python
# 상황: 웹사이트나 파일에서 읽어오면 무조건 '문자열'로 옴
port = "8080" 

# (X) 에러 발생: 문자열에 숫자를 더하려고 함
# next_port = port + 1  -> TypeError!

# (O) 해결: 문자를 숫자로 바꿈 (int())
next_port = int(port) + 1 
print(next_port) # 8081
```

### 자주 쓰는 변환 함수

- `int(값)`: 정수로 변환 ("3.5" 넣으면 에러 남! 소수는 `float`로 먼저!)
- `str(값)`: 문자열로 변환 (숫자와 글자를 합칠 때 필수)
- `bool(값)`: 참/거짓으로 변환 (`0`, `""`, `[]` 같은 빈 값은 `False`가 됨)

---
## 데이터 엔지니어의 실수 노트 (Common Pitfalls)

### ① Airflow 변수나 환경변수는 다 `str`이다!

Airflow UI에서 Variable을 `100`이라고 저장해도, 파이썬 코드로 가져오면 `"100"`(문자열)으로 옵니다. 
반드시 **`int()`로 감싸서** 써야 합니다.

### ② `None` 타입

"값이 없음"을 나타내는 특별한 타입입니다. `0`이나 `False`랑은 다릅니다!

```python
result = None

if result is None:
    print("데이터가 아직 안 왔어요.")
```


---
### Tip

"파이썬하다가 **`TypeError`** 가 떴다? 90%는 숫자랑 문자를 더하려고 했거나, `None`값에다가 뭘 하려고 했을 때야. 
그럴 땐 `print(type(변수명))`을 찍어봐. 내 눈엔 숫자로 보여도 컴퓨터 눈엔 문자열(`str`)인 경우가 허다하거든!"