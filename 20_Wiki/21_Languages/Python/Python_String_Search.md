---
aliases:
  - Python 문자열 위치 찾기
  - find
  - index
  - rfind
  - rindex
tags:
  - Python
related:
  - "[[Python_String_Methods]]"
  - "[[Python_Lists_Tuples]]"
---
## 개념 한 줄 요약

**"문자열 안에서 '내가 찾는 글자'가 몇 번째 칸(Index)에 있는지 탐정처럼 찾아내는 함수들."**

---
## 왜 필요한가? (Why)

**문제점:**

- "이메일 주소(`user@gmail.com`)에서 `@` 앞부분만 아이디로 쓰고 싶은데, `@`가 몇 번째에 있는지 모르면 자를 수가 없다."
- "로그 파일 경로에서 가장 마지막 슬래시(`/`) 뒤에 있는 파일명만 가져오고 싶다."

**해결책:**

- **`find` / `index`**: 앞에서부터 찾아서 위치(숫자)를 알려준다.
- **`rfind` / `rindex`**: **뒤에서부터(Right)** 찾아서 위치를 알려준다.

----
## 실무 적용 사례 (Practical Context)

1. **아이디 추출:** 이메일(`@`)이나 구분자(`:`)의 위치를 찾아서 그 앞까지만 슬라이싱(`[:idx]`)할 때.
2. **파일 확장자 확인:** 파일명에서 가장 마지막 점(`.`)의 위치를 찾을 때 (`rfind` 사용).
3. **데이터 검증:** 특정 필수 키워드가 문장에 포함되어 있는지 위치를 확인해서 검증할 때.

---
## Code Core Points: 문법 공식

> **핵심 차이 1: 못 찾았을 때 반응**

> **`find` 계열:** 없으면 **`-1`** 을 반환 (안전함, 프로그램 안 죽음)
>  **`index` 계열:** 없으면 **`ValueError` 에러** 발생 (위험함, 확실히 있을 때만 써야 함)
>   

> **핵심 차이 2: 찾는 방향**
>
>  **`find` / `index`:** 왼쪽(0번) → 오른쪽
>  **`rfind` / `rindex`:** 오른쪽(끝) → 왼쪽 (**R**ight)

### ① 기본 탐색 (`find`, `index`)

```python
text = "hello world"

# 1. find: 찾으면 인덱스 반환
print(text.find("o"))   # 결과: 4 ('hello'의 o)

# 2. find: 못 찾으면 -1 (에러 안 남! ⭐️)
print(text.find("z"))   # 결과: -1

# 3. index: 찾으면 인덱스 반환
print(text.index("o"))  # 결과: 4

# 4. index: 못 찾으면 에러 발생! (멈춤 🚨)
# print(text.index("z")) # ValueError: substring not found
```

### ② 뒤에서부터 찾기 (`rfind`, `rindex`)

**파일 경로**나 **URL** 처리할 때 "가장 뒤에 있는 구분자"를 찾기 위해 정말 많이 씁니다.

```python
path = "/usr/local/bin/python"

# 1. 그냥 find: 앞에서부터 찾음
print(path.find("/"))   # 결과: 0 (맨 앞의 /)

# 2. rfind: 뒤에서부터 찾음 (Right find) ⭐️
# 마지막 슬래시의 위치를 찾음 -> 이 뒤가 진짜 '파일명'이니까!
last_slash_idx = path.rfind("/") 
print(last_slash_idx)   # 결과: 14

# 활용: 파일명만 쏙 뽑아내기
filename = path[last_slash_idx + 1:] 
print(filename)         # 결과: python
```

---
## 상세 분석: 위치 범위 지정 (Start, End)

검색할 범위를 제한할 수도 있습니다. `{python}text.find(찾을거, 시작위치, 끝위치)`

```python
data = "apple apple apple"

# 첫 번째 apple 패스하고, 그 다음부터 찾고 싶을 때
# 인덱스 5번부터 끝까지 중에서 찾아라
print(data.find("apple", 5))  # 결과: 6 (두 번째 apple의 시작 위치)
```

---
## 초보자가 자주 하는 실수 (Misconceptions)

### ① "`rfind`를 쓰면 인덱스도 거꾸로(음수) 나오나요?"

- **아니요!** 찾는 순서만 뒤에서부터일 뿐, **인덱스 번호는 항상 앞에서부터 0, 1, 2... 양수**로 나옵니다.
- 헷갈리지 마세요. 위치 번호(주소)는 변하지 않습니다.

### ② "리스트(List)에서도 `find`를 쓸 수 있나요?"

- **절대 불가 (X)**. `find`와 `rfind`는 **문자열(String) 전용**입니다.
- 리스트에서는 오직 **`index()`** 만 쓸 수 있습니다.
    - `['a', 'b', 'c'].index('b')` (O)
    - `['a', 'b', 'c'].find('b')` (X) -> `AttributeError` 발생!


### ③ "무조건 `index`가 `find`보다 나쁜가요?"

- 아닙니다. "이 값이 무조건 있어야만 로직이 돌아간다"는 상황에서는 `index`를 써서 에러를 터뜨리는 게 낫습니다.
- 반면, "있으면 처리하고 없으면 말지 뭐" 같은 상황에서는 `-1`을 주는 `find`가 훨씬 코드가 깔끔해집니다 (`if result != -1:`).